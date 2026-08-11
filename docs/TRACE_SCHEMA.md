# The trace schema — AURA's observability contract

Everything AURA claims about explainability rests on one idea: **every decision an
agent makes is a recorded span, and a completed run can be replayed from those spans
alone.** Not a log line. Not a post-hoc explanation asked of the model. A structured
record written as the decision happens.

This document is that contract. A real exported run is in
[`../examples/sample_trace.json`](../examples/sample_trace.json).

## Why spans and not logs

A log line tells you *something happened*. A span tells you **what was asked, what
came back, how long it took, what it cost, and what it was nested inside.** That last
part is the one that matters: parentage is what lets a run be reconstructed as a tree
instead of a flat stream, which is what makes replay possible.

The rule AURA holds to: **an explanation is reconstructed from recorded decisions,
never generated after the fact.** If you ask a model why it did something, it will
produce a plausible narrative that may have nothing to do with what actually
executed. Post-hoc explanations lie. Spans don't.

## The span

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Unique span id |
| `trace_id` | string | The run this span belongs to |
| `parent_id` | string \| null | Enclosing span. `null` marks the root — this is what makes the tree |
| `name` | string | What happened, e.g. `plan:decompose`, `agent:analyst_worker`, `llm:claude-haiku-4-5-20251001` |
| `kind` | string | Category — see below |
| `status` | `ok` \| `error` | Outcome |
| `started_at` / `ended_at` | float | Unix timestamps; their difference is the duration |
| `input` / `output` | string \| null | The actual payload, subject to the capture policy below |
| `tokens_in` / `tokens_out` | int | Token accounting, per span |
| `cached` | bool | Whether a cache hit served this call |
| `error` | string \| null | Failure detail when `status = error` |
| `meta` | JSON string | Per-kind extras — model name, cost, confidence, retrieval coverage, quarantine counts |
| `tenant` | string | Owning tenant; the trace store is queried tenant-scoped |

### Span kinds

| Kind | Emitted for |
|---|---|
| `orchestration` | Planning, decomposition, routing, replanning |
| `agent` | A worker executing its assigned step |
| `llm` | One model call — carries model, tokens, cost, cache status |
| `tool` | One tool invocation, with its input and result |
| `memory` | Recall, remember, compression, context assembly |
| `eval` | Verification, judging, groundedness checks |

## What a real trace looks like

From the sample export — a plan-and-execute run, 13 spans:

```
plan_execute                                    (orchestration, root)
├─ llm:claude-haiku-4-5-20251001                (llm)
├─ plan:decompose                               (orchestration)
│     in : {"task": "Profile the top customer"}
│     out: {"steps": ["Identify the top customer",
│                     "List that customer's orders",
│                     "Summarise the products bought"]}
├─ llm:claude-sonnet-4-6                        (llm)
├─ agent:analyst_worker                         (agent)
├─ plan:step:1                                  (orchestration)
├─ agent:analyst_worker                         (agent)
├─ plan:step:2                                  (orchestration)
└─ …
```

Two things are worth noticing. **A cheap model and a strong model both appear in the
same run** — routing per role rather than sending everything to the expensive one. And
**the decomposition is visible as data**: you can read the three steps the planner
chose, so when the answer is wrong you can see whether the plan was wrong or the
execution was.

## Content capture is a policy, not a default

Payloads are the sensitive part, so capture is explicitly tiered rather than always-on:

| Setting | Behaviour |
|---|---|
| `off` | Structure only — names, timings, tokens, status. No payloads. |
| `summary` | Truncated payloads. |
| `full` | Complete payloads. |

**Secrets are always redacted regardless of tier.** PII redaction is opt-in. The
sample trace in this repo was exported at `full` and then passed through an additional
redaction pass for absolute paths, key-shaped strings and internal IP ranges before
publication.

## What replay is built on

Because the tree, the timings and the payloads are all recorded, a run can be
reconstructed without re-executing it. That's what drives:

- the **2D timeline** — a waterfall of spans by duration
- the **3D Agent Theater** — the run classified into a scene (courtroom for
  judge/verify, round table for debate, knowledge lab for planning) and performed
  beat by beat
- the **live SSE feed** — spans streamed as they're written, via a cursor over the
  store, so both start and completion surface
- **explanation chains** — "why this answer" assembled from what actually ran
- **trajectory evaluation** — tool-sequence quality, loop detection and step-budget
  checks scored from the recorded run, with no API key needed

## Honest limits

- **Sampling is not implemented.** Every span is written. At high volume that's a
  storage cost you'd have to manage.
- **The store is SQLite by default** (WAL-enabled for concurrent sessions), with a
  Postgres backend behind the same interface. The Postgres path is not exercised by
  the keyless test gate.
- **`full` capture means payloads are at rest in the trace store.** Secret redaction
  always applies; PII redaction is opt-in and therefore off unless you turn it on.
  Treat the store with the same care as the data it observed.
- **Cost figures are config-driven.** They're computed from a price table you supply;
  with no table configured, cost reads zero rather than guessing.
