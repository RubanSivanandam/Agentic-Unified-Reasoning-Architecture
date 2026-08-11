<div align="center">

<img src="docs/assets/hero.svg" width="100%" alt="AURA — Agentic Unified Reasoning Architecture"/>

<br/>

<img src="docs/assets/tagline.svg" width="80%" alt="Drop your database tables in and an agentic pipeline builds the ontology; ask in natural language and watch every decision replay; runs on a laptop or an air-gapped box."/>

<br/><br/>

<img src="https://img.shields.io/badge/version-v2.7.0-56b6c2?style=flat-square" alt="version v2.7.0"/>
<img src="https://img.shields.io/badge/gated_sprints-149-e8a33d?style=flat-square" alt="149 gated sprints"/>
<img src="https://img.shields.io/badge/waves_shipped-18-bb9af7?style=flat-square" alt="18 waves shipped"/>
<img src="https://img.shields.io/badge/keyless_tests-1293_passing-9ece6a?style=flat-square" alt="1293 keyless tests passing"/>

<br/>

<img src="https://img.shields.io/badge/python-3.10_·_3.11_·_3.12-3776AB?style=flat-square&logo=python&logoColor=white"/>
<img src="https://img.shields.io/badge/GPU-not_required-7aa2f7?style=flat-square"/>
<img src="https://img.shields.io/badge/BYOK-any_model-bb9af7?style=flat-square" alt="bring your own key, any model"/>
<img src="https://img.shields.io/badge/source-private-6e7681?style=flat-square" alt="source is private"/>

<br/><br/>

<img src="docs/assets/stats.svg" width="100%" alt="AURA at a glance: 149 gated sprints, 18 waves, 1293+ keyless tests, 4 benchmark suites, 0 GPUs required"/>

</div>

---

> ### 📄 What this repository is
>
> This is the **public engineering record** for AURA — the architecture, the full
> sprint ledger, the measured results, and the methodology behind them.
>
> **The implementation lives in a private repository.** Nothing here is runnable; every
> document here is the evidence trail for something that is. The numbers on this page
> were measured, not estimated, and where a claim isn't proven I say so explicitly.
>
> Engineers and hiring teams who'd like to go through the internals: contact is at the
> bottom.

---

## The problem

Most agent frameworks are black boxes that assume a datacenter. When an agent returns a
wrong answer you get the wrong answer and nothing else — no way to see which step
broke, which tool misfired, or which retrieval came back empty.

I hit exactly that. I had data sitting in undocumented database tables, I wanted to ask
questions in plain English, and every framework I tried failed halfway with no
explanation — most of them wanting a GPU I didn't have.

## What AURA does

**Drop your database tables in, an agentic pipeline builds the ontology itself, you ask
in natural language, and every routing, planning, tool and verification decision becomes
a traced span you can replay.**

No GPU. No cloud bill beyond your own API key. Runs on a laptop or an air-gapped box.

<div align="center">
<img src="docs/assets/architecture.png" width="88%" alt="AURA architecture: ingest, knowledge, reasoning and tools layers, with observability, safety and evaluation across every layer"/>
</div>

## Watch the agents think

The fastest way to understand AURA is to watch a run replay itself. Every decision is a
recorded span, so a completed run plays back beat by beat — as a 2D timeline, or on a
cinematic 3D stage where the supervisor routes, debaters argue at a round table, and the
judge scores.

<div align="center">
<img src="docs/assets/theater.gif" width="62%" alt="Animated capture of the 3D Agent Theater replaying a real multi-agent debate: two debaters and a judge at a round table, live vote tally, per-agent model badges, and the execution graph"/>
<br/>
<em>A real multi-agent debate, replayed. The badges are the actual models used per agent —
a cheap one and a strong one — and the left panel is the real execution graph.<br/>
Captured from a live run; <a href="docs/assets/theater3d.png">full-resolution still here</a>.</em>
</div>

<br/>

<div align="center">
<img src="docs/assets/dashboard.png" width="88%" alt="The AURA dashboard showing the core reactor, live system telemetry, mission log of recent traces, and the knowledge graph"/>
<br/>
<em>Live telemetry from real runs: traces, model calls, cache-hit rate, token spend, tool-error rate.</em>
</div>

Visual replay is one half. The other is **[deterministic execution replay](REPLAY.md)** —
re-running the *same agent code* with the recorded model answers injected, so the
deterministic glue executes for real and any behaviour change fails loudly. A recorded
run becomes a regression test anyone can run offline, with no API key.

---

## The receipts

Every claim AURA makes has a document behind it. That's the point of this repo.

| Receipt | What it shows |
|---|---|
| **[RESULTS_BENCHMARKS.md](RESULTS_BENCHMARKS.md)** | Multi-model benchmark harness over τ²-bench / GAIA / SWE-bench / BFCL adapters — with an explicit statement of what is *not* proven without a key |
| **[RESULTS_RELIABILITY.md](RESULTS_RELIABILITY.md)** | Calibrated confidence, faithfulness gating, cost-aware routing, causal failure attribution |
| **[RESULTS_ONTOLOGY.md](RESULTS_ONTOLOGY.md)** | The ontology as a real OWL/RDF graph — 29 triples, 7/7 SPARQL competency questions, SHACL catching a dangling relation |
| **[RESULTS_MEMORY.md](RESULTS_MEMORY.md)** | Measured compression: fidelity 0.609 vs 0.0 on faithful vs vague summaries; intent-aware retrieval k=2 vs k=8 |
| **[METHODOLOGY.md](METHODOLOGY.md)** | How the framework comparison was run — including the fairness caveats that make the numbers honest |
| **[RESULTS_COMPARE.md](RESULTS_COMPARE.md)** | The comparison results themselves, caveats attached |
| **[REPLAY.md](REPLAY.md)** | Deterministic execution replay: the bundle format, the "a miss is the signal" contract, and an explicit list of what replay does **not** pin |
| **[ZERO_TRUST.md](ZERO_TRUST.md)** | Security posture: per-agent identity, capability authz, egress allow-list, memory integrity |
| **[SECURITY_ASI2026.md](SECURITY_ASI2026.md)** | Self-audit against OWASP's Top 10 for Agentic Applications (2026), ASI01–ASI10 |

## The ledger

**[ROADMAP.md](ROADMAP.md)** is the complete record: **149 bounded, individually-gated
sprints across 18 waves.** Each sprint is one reversible change behind an environment
flag, accepted only when three keyless gates stayed green, and reported as a measured
delta — never as "10x" or "100%".

It also records what I **declined to build**, and why. That half is the more useful half.

| Wave | Milestone | What it added |
|---|---|---|
| Foundation → Adoption | `v0.9.0` | Tool-use loop, self-building ontology, DAG planning, GraphRAG, packaging + CI |
| Measurably smarter | `v0.10.0` | A golden scoreboard *first*, then typed contracts, parallel fan-out, agentic retrieval |
| Self-learning | `v1.0.0` | Tenant isolation, runtime parity, memory compression, MCP trust enforcement |
| Ecosystem | `v1.1.0` | Five model backends, plugin registry, A2A v1.0, bidirectional MCP |
| Enterprise | `v1.2.0` | Distributed execution, durable workflows, compliance surface |
| Intelligence frontier | `v2.0.0` | Tool synthesis, cross-session learning, formal safety bounds, adversarial testing |
| Acts in the world | `v2.1.0` | Persistent scheduler, screened notifications, approval-gated connectors |
| Scale + Zero Trust | `v2.3.0` | WAL concurrency, exactly-once execution, per-agent identity, egress control |
| Orchestration + research | `v2.4.0` | Swarm topologies, benchmark leaderboard, autonomous researcher, eval-gated self-improvement |
| Agent tooling + MCP | `v2.5.0` | Sandboxed code interpreter, egress-gated web research, MCP registry |
| Brain 2.0 | `v2.6.0` | OWL/RDF export, SPARQL competency questions, SHACL validation |
| Memory 2.0 | `v2.7.0` | Measured compression fidelity, online semantic synthesis, intent-aware retrieval |
| **Determinism & Throughput** | *in progress* | **[Deterministic execution replay](REPLAY.md)** — re-execute a recorded run with its model answers injected, turning any recorded run into a keyless regression test |

## How it's built

**[HANDBOOK.md](HANDBOOK.md)** is the guided tour of the architecture, and
**[docs/TRACE_SCHEMA.md](docs/TRACE_SCHEMA.md)** documents the observability contract —
the span shape everything else is built on. A real exported run sits in
**[examples/sample_trace.json](examples/sample_trace.json)** if you'd rather read the
actual data structure than a description of it.

**[CHANGELOG.md](CHANGELOG.md)** has the release history.

---

## The engineering rules I held to

These are the reason the numbers above are worth anything.

**Measured, never claimed.** Every sprint reports a delta against a fixed golden set —
40 keyless fixtures across 7 categories. "It feels smarter" is banned.

**A flaky pass is not a pass.** A fixture that flip-flops across runs is held red until
it proves stable. One test that failed 11 times in 200 runs turned out to be a real
double-execution bug, not a flake.

**Three gates, every sprint, no exceptions.** Full test suite, a real end-to-end run,
and the golden-set report — all keyless, on Python 3.10, 3.11 and 3.12.

**Off by default.** Every capability ships behind an environment flag that defaults to
safe. With the flags unset, behaviour is byte-identical to the previous release — and
that's test-proven, not assumed.

**Attack your own work.** The sandbox, the egress allow-list and the skill audit were
each broken by a deliberate adversarial pass before shipping: a full-host escape in my
own sandbox, an SSRF-via-redirect around my own allow-list, and a security scanner I had
to honestly downgrade from "the boundary" to "one pre-filter" after it was defeated.
Those are written up in the ledger, not buried.

**Say what isn't proven.** Where a result needs an API key to demonstrate, the receipt
says so rather than implying the keyless gate covered it.

---

## Stack

Python · FastAPI · SQLite (WAL) · Kuzu · sqlite-vec · rdflib + pySHACL · three.js +
React Three Fiber · OpenTelemetry · Docker · Helm · GitHub Actions

Model-agnostic by design — five interchangeable backends, plus a keyless stub
backend that makes the entire test suite runnable with no API key at all.

## Contact

**Ruban S** — Software Engineer · Agentic AI · Chennai, India

Happy to walk through the architecture, the trace model, or any of the receipts above.
Source access can be discussed for a serious technical conversation.

[LinkedIn](https://www.linkedin.com/in/ruban-s-811964173/) · [GitHub](https://github.com/RubanSivanandam)

---

<sub>Documentation in this repository is licensed <strong>CC BY 4.0</strong> — see
<a href="LICENSE">LICENSE</a>. The AURA implementation is not included here and is not
covered by that licence.</sub>
