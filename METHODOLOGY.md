# AURA vs. alternatives — methodology (Sprint #87)

This document states exactly what the comparison in
[RESULTS_COMPARE.md](RESULTS_COMPARE.md) measures, what it deliberately does
not, how fairness is protected, and where bias could still hide. It follows
the repo's benchmark discipline (Sprint #67): **publish the number you can
honestly own — framework overhead — and leave model latency and answer
quality to the model and your key.**

## What is measured

**Orchestration overhead.** Wall milliseconds per multi-step scenario, with
every framework replaying **identical scripted LLM responses** through its own
documented mock-LLM idiom, plus a one-time setup cost and a one-shot
allocation peak (`tracemalloc`). Three scenarios exercise the three
multi-agent shapes every framework claims:

| scenario | shape | steps in the shared script |
|---|---|---|
| `tool_chain` | one agent, think→act→observe | 3 tool calls + final answer |
| `pipeline_3_agents` | sequential 3-stage handoff | 3 text stages |
| `router_dispatch` | manager routes to a worker | route choice + worker answer + synthesis |

A **correctness gate** (`ok`) asserts the framework actually produced the
scenario's expected answer from the script — a fast wrong run scores nothing.

**Capability coverage.** A sourced feature matrix (`features.py`), each cell
backed by a documentation link checked on the stated date — coverage is
*sourced, not scored*: the matrix records what each framework's docs say it
provides, it does not rate quality.

## What is NOT measured (honestly)

- **Answer quality.** No SWE-bench, no HumanEval, no LLM-judged comparison.
  Those need live models, keys, and hours of compute; running them is a
  deliberate operator action, not something a keyless gate can vouch for.
  The operator path, when someone funds it: hold the model constant (same
  provider/model/temperature across frameworks), run the same task set
  through each adapter, and publish per-task transcripts alongside scores.
  Until that exists, **this repo makes no comparative quality claim** —
  stated in the README section and in the paper (docs/AURA_PAPER.md §7).
- **Model latency.** Excluded by design (the #67 rule); it belongs to the
  provider.
- **Distributed throughput / production load.** Single-process, one machine.

## Fairness: design choices and residual bias

Protections:

1. **Same scenarios, same scripted turns** for every framework; a framework
   that makes *extra* LLM calls (e.g. richer prompt scaffolding) has the
   final text turn repeated (the exhaustion rule in `adapters/base.py`) —
   the extra calls are honestly part of its overhead, and the correctness
   gate still applies.
2. **Each framework runs its own documented idiom** — LangGraph
   `StateGraph`, CrewAI `Crew`/`Process`, AG2 `ConversableAgent`/`GroupChat`,
   AURA `ToolAgent`/`Supervisor` — built from each framework's own docs, not
   a lowest-common-denominator wrapper.
3. **Adapter code is published** under `benchmarks/compare/adapters/` for
   review; every workaround is commented in place.
4. **Setup is timed separately** from per-run execution.

Residual bias, stated:

- **This comparison was built by the AURA side.** Adapters for other
  frameworks were written from their public docs by people who know AURA
  best. We accepted every workaround that made a competitor pass (see
  per-adapter caveats below) and publish the code, but a maintainer of
  LangGraph/CrewAI/AG2 could likely write a leaner adapter. Corrections are
  welcome — that is why the adapters are small, separate files.
- **The frameworks do different work per step by design.** AURA's cells run
  its production posture (every step persists a trace span to SQLite, guard
  screens run, the semantic cache is consulted — a per-run nonce defeats
  exact-match caching so a cache hit never masquerades as orchestration).
  LangGraph and AG2 run bare graphs/chats — their observability is external
  tooling and is not running here. CrewAI's numbers include its heavier
  prompt scaffolding. **Lower is therefore not better across rows**; the
  table answers "what does each framework's idiomatic path cost", not
  "which framework is fastest at identical work".
- **Mock thickness differs.** Each framework's mock uses that framework's
  own custom-model seam, so the mock's plumbing depth follows the
  framework's design. AURA's mock is its real stub client running through
  its real LLM wrapper — the deepest of the four — which, if anything,
  overstates AURA's overhead. We accept the conservative direction.

## Per-adapter caveats (every workaround, named)

- **CrewAI** (`crewai_adapter.py`): hierarchical (router) crews attach
  delegation tools to the auto-created manager during `kickoff()` and then
  reject that manager on the next `kickoff()` ("Manager agent should not
  have tools") — the adapter resets `crew.manager_agent = None` per run so
  CrewAI rebuilds it, honestly timed. CrewAI's interactive trace prompt and
  telemetry are disabled via `CREWAI_TRACING_ENABLED=false`,
  `CREWAI_DISABLE_TELEMETRY=true`, `OTEL_SDK_DISABLED=true` (an interactive
  20-second prompt inside a benchmark loop is itself a finding, but not a
  number we publish).
- **AG2** (`autogen_adapter.py`): three integration quirks, all commented in
  code — (1) `register_for_llm` rebuilds the agent's client wrapper, so the
  custom model client must be re-registered *after* tool registration;
  (2) the response object must carry a `message_retrieval_function`
  attribute or AG2's extraction bypasses `message_retrieval` entirely;
  (3) "auto" group-chat speaker selection spawns internal agents whose
  custom clients cannot be activated pre-run, so the router uses AG2's
  documented custom `speaker_selection_method` callable — **AG2's router
  cell therefore spends no mock-LLM call on the routing decision, unlike
  the other three frameworks** (its router number is flattered by one call;
  stated here rather than papered over).
- **LangGraph** (`langgraph_adapter.py`): the mock follows the
  `langchain-core` fake-chat-model pattern; no workarounds were needed.

## Framework facts (verified 2026-07-11, links in features.py / RESULTS)

- **LangGraph** — PyPI `langgraph` 1.2.9 (released 2026-07-10), MIT,
  LangChain Inc.
- **CrewAI** — PyPI `crewai` 1.15.2 (2026-07-08), MIT, crewAIInc.
- **AG2 vs. AutoGen naming:** the AutoGen community split in late 2024.
  Microsoft's line (PyPI `autogen-agentchat`, last release 0.7.5,
  2025-09-30) is in maintenance mode, folded into the Microsoft Agent
  Framework; the actively developed community continuation is **AG2**
  (PyPI `ag2` 0.14.0, 2026-06-26, Apache-2.0), which kept the classic
  `import autogen` API. This comparison targets AG2 and labels it
  "ag2 (autogen)".

## Reproduce

```bash
# isolated venv — NEVER the main one (protects the 679-test keyless gate)
python -m venv .venv-compare
.venv-compare/Scripts/pip install -r requirements.txt \
    -r benchmarks/compare/requirements-compare.txt

CREWAI_TRACING_ENABLED=false CREWAI_DISABLE_TELEMETRY=true OTEL_SDK_DISABLED=true \
AURA_LLM=stub .venv-compare/Scripts/python benchmarks/compare/harness.py --write
```

`--quick` runs n=3 for a smoke pass; `--frameworks aura,langgraph` selects a
subset. With only AURA installed the harness still runs and reports the
others "not run here" with the reason — keyless degradation is a tested
property (`tests/test_compare_bench.py`).
