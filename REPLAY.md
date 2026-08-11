# Deterministic execution replay

*Sprint #151, Wave 18. Record a run's model answers, then re-execute the same
agent code with those answers injected.*

AURA could already replay a run **visually** — the Agent Theater renders recorded
spans. It could not **re-execute** one. This closes that gap.

## Why

An agent run is mostly deterministic glue — routing, planning, tool dispatch,
memory, verification — wrapped around one irreducibly nondeterministic thing: the
model's answer. Pin that, and everything else becomes reproducible.

This matters more than it sounds. Agent accuracy is reported to vary by **up to
~15% between runs** even with deterministic settings configured, and in
multi-step workflows each step's output is the next step's input, so small
divergences compound. The record-replay discipline is the standard answer, adapted
here from the 2026 agent-debugging literature ([arXiv:2505.17716](https://arxiv.org/pdf/2505.17716),
[arXiv:2606.08275](https://arxiv.org/pdf/2606.08275)).

What you get:

- **Reproducible debugging.** Re-run a failure as many times as you like without
  paying for it or hoping the model repeats itself.
- **Keyless regression tests from real runs.** A production recording becomes a
  test that anyone can run offline, with no API key.
- **Change detection.** If a code change alters the sequence of model calls, the
  replay says so immediately.

## Usage

```bash
# 1. record — works with ANY backend, including the keyless stub
AURA_LLM_RECORD=run.replay.jsonl python my_agent.py

# 2. re-execute — no provider is contacted
AURA_LLM=replay AURA_REPLAY_BUNDLE=run.replay.jsonl python my_agent.py
```

The agent code runs **for real** on the replay: routing, planning, tools, memory
and verification all execute. Only the model's answers come from the recording.

## Flags

| Variable | Default | Meaning |
|---|---|---|
| `AURA_LLM_RECORD` | unset (off) | Path to write the bundle. Wraps whichever backend is active. |
| `AURA_LLM=replay` | — | Selects the replay backend. |
| `AURA_REPLAY_BUNDLE` | — | Bundle to replay. Required in replay mode. |
| `AURA_REPLAY_MATCH` | auto | `strict` = recorded order + fingerprint check. `key` = match by fingerprint, order-independent. **Auto-selected**: `strict` for a single-caller run, `key` when the recording had several callers (see below). |

With both unset, behaviour is byte-identical to before — the recording class is
never even constructed. That is test-proven, not assumed.

## A miss IS the signal

If the run asks for a model call the recording doesn't contain, `ReplayMiss` is
raised. **This is the feature, not a bug.** It means the deterministic glue took a
different path than when the bundle was recorded. Silently substituting an answer
would destroy the only thing replay exists to detect.

## Ordering: why multi-caller runs match by fingerprint

A single-caller run has a well-defined call order, so strict cursor order plus a
fingerprint check is used — the sharpest signal available.

A **fan-out does not**. A debate or parallel supervisor drives several `LLM`
instances (a strong worker and a cheap utility model), and which one calls first
depends on state *outside* the recording — context size, how much memory has
accumulated, routing decisions. This was found the honest way, by replaying a real
two-model debate: the recording read `[sonnet ×4, haiku]` and the replay asked for
the haiku call first, because the recording run had itself grown the memory that
the replay then read back.

So when more than one caller writes a bundle, the manifest is marked `concurrent`
and replay matches **by fingerprint instead of position**. Every recorded response
must still be accounted for — order-independence does not weaken the miss
contract, it just stops reporting a false divergence for an order that was never
reproducible in the first place.

Three ways divergence surfaces:

- **Different request** — the fingerprint at position *N* doesn't match.
- **More calls than recorded** — the run went further than the recording.
- **Fewer calls than recorded** — `verify_exhausted()` reports unused entries.

## How it fits the existing architecture

No rewrite. Three seams:

**Recording** is a decorator, not a backend. `RecordingClient` wraps whichever
client `_ensure_client` already chose and passes calls straight through. It sits
*below* the quota gate, circuit breaker, retry loop, cache and tracing spans — all
of which live in `LLM` — so it changes none of them.

**Replay** is just another backend. `ReplayClient` satisfies the same duck-type
`StubClient` already proves is sufficient, so it inherits the breaker, retries,
quotas, spans and cost accounting for free.

**One mandatory interaction:** recording and replay both disable the semantic
cache. A cache hit returns without calling `messages.create`, so a recording would
omit a call the replay still asks for — producing a false miss on every run. This
is enforced in `LLM._bypass_cache`.

## Bundle format

JSONL, append-only. First line is the manifest; one line per model call.

```json
{"kind":"manifest","version":1,"aura_version":"2.7.0","recorded_at":1754…,"model":"…","fidelity":"full","concurrent":false}
{"kind":"call","seq":0,"fingerprint":"sha256:9f2a…","request":{…},"response":{"content":[…],"stop_reason":"end_turn","usage":{…}},"latency_ms":812}
{"kind":"manifest_patch","concurrent":true}
```

Append-only is deliberate: the file is never rewritten, so a crashed run still
leaves a loadable bundle, and a torn final line is tolerated on load.

## ⚠️ Security

**A bundle contains raw prompts and raw model responses — unredacted and
untruncated.** That is required for fidelity: the trace store truncates and
redacts by policy, so it cannot serve as a replay source. Treat a bundle with the
same care as the data the run touched. `*.replay.jsonl` and `replay_bundles/` are
gitignored.

## What replay does NOT pin

Being explicit, because a partial guarantee that reads as total is worse than
none:

- **Tool side effects.** Tools re-execute for real. If a tool writes to a
  database or sends a request, replaying does it again.
- **Wall-clock time**, `uuid`, `random` — not intercepted.
- **Network performed by tools** — only *model* calls are served from the bundle.
- **Concurrent ordering.** When calls overlap, recorded order isn't reproducible;
  the manifest records `concurrent` and replay matches by fingerprint instead.

Replay pins the model's answers. Everything else is genuinely re-executed — which
is exactly why it catches behaviour changes.
