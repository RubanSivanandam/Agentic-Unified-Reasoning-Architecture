# AURA vs. alternatives — orchestration overhead (Sprint #87)

> keyless orchestration overhead — identical scripted mock-LLM responses per framework; model latency excluded by design.
> Python 3.11.0 · AMD64 · 2026-07-11 · n=15 repetitions per cell · numbers are machine-dependent — reproduce with `python benchmarks/compare/harness.py --write` (see METHODOLOGY.md).

| framework | scenario | ok | mean ms | p50 ms | p95 ms | setup ms | peak alloc KB |
|---|---|---|---|---|---|---|---|
| aura | tool_chain | yes | 44.398 | 43.368 | 54.686 | 0.278 | 15.0 |
| aura | pipeline_3_agents | yes | 42.97 | 42.169 | 51.002 | 0.047 | 10.1 |
| aura | router_dispatch | yes | 37.208 | 36.601 | 42.091 | 0.13 | 12.0 |
| langgraph | tool_chain | yes | 2.419 | 1.833 | 4.856 | 2.441 | 32.6 |
| langgraph | pipeline_3_agents | yes | 0.884 | 0.833 | 1.167 | 2.609 | 27.9 |
| langgraph | router_dispatch | yes | 1.021 | 0.871 | 1.423 | 3.525 | 29.1 |
| crewai | tool_chain | yes | 28.285 | 28.243 | 37.302 | 20.339 | 116.9 |
| crewai | pipeline_3_agents | yes | 76.427 | 77.766 | 81.46 | 11.915 | 179.5 |
| crewai | router_dispatch | yes | 50.571 | 48.808 | 60.817 | 8.602 | 492.4 |
| ag2 (autogen) | tool_chain | yes | 1.877 | 1.75 | 2.925 | 11.562 | 9.1 |
| ag2 (autogen) | pipeline_3_agents | yes | 0.963 | 0.917 | 1.539 | 3.558 | 6.3 |
| ag2 (autogen) | router_dispatch | yes | 0.371 | 0.361 | 0.416 | 3.889 | 5.7 |

`ok` = the framework produced the scenario's expected answer from the identical script (correctness gate, not a quality score). `setup ms` = one-time agent/graph construction. Adapter code is published for review under `benchmarks/compare/adapters/`.

**Reading the numbers honestly:** the frameworks do different amounts of work per step BY DESIGN. AURA's cells run its production posture — every step persists a trace span to SQLite, guard screens run, the semantic cache is consulted — because that is how AURA always runs; the LangGraph and AG2 adapters execute bare graphs/chats with no persistent observability (their tracing is external tooling), and CrewAI's cells include its richer prompt scaffolding. Lower is therefore NOT better across rows: the table answers "what does each framework's own idiomatic multi-step path cost, mock-model latency excluded" — not "which framework is fastest at the same work". See METHODOLOGY.md for the full fairness discussion and residual-bias statement.

## Capability coverage (sourced, not scored)

| capability | AURA | LangGraph | CrewAI | AG2 (AutoGen) |
|---|---|---|---|---|
| Multi-agent orchestration (supervisor / graph / crew / group chat) | yes | yes | yes | yes |
| Native tool-calling loop (think->act->observe) | yes | yes | yes | yes |
| Deterministic keyless mock LLM shipped in-framework (run the whole stack with no key, no network) | yes | partial | no | no |
| Built-in tracing UI (self-hosted, no external service) | yes | no | no | partial |
| Durable execution / checkpoint-resume | yes | yes | partial | partial |
| Human-in-the-loop approval gates | yes | yes | yes | yes |
| License / stewardship (checked on PyPI) | yes | yes | yes | yes |

Sources (checked 2026-07-11):

- **Multi-agent orchestration (supervisor / graph / crew / group chat)** — aura: aura/agents/supervisor.py, aura/agents/parallel_supervisor.py; langgraph: https://docs.langchain.com/oss/python/langgraph/graph-api; crewai: https://docs.crewai.com/en/concepts/crews; ag2: https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/orchestration/orchestrations/
- **Native tool-calling loop (think->act->observe)** — aura: aura/agents/tool_agent.py; langgraph: https://docs.langchain.com/oss/python/langchain/agents; crewai: https://docs.crewai.com/en/concepts/tools; ag2: https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/introducing-tools/
- **Deterministic keyless mock LLM shipped in-framework (run the whole stack with no key, no network)** — aura: aura/llm/stub.py (AURA_LLM=stub; the entire 679-test gate runs on it); langgraph: fake chat models live in langchain-core (GenericFakeChatModel), not langgraph itself: https://reference.langchain.com/python/langchain_core/language_models/; crewai: custom BaseLLM subclass required: https://docs.crewai.com/en/learn/custom-llm; ag2: custom model client required: https://docs.ag2.ai/latest/docs/blog/2024-01-26-Custom-Models/index/
- **Built-in tracing UI (self-hosted, no external service)** — aura: aura/dashboard + aura/tracing (dashboard at localhost:8000, spans in SQLite); langgraph: observability is LangSmith, a separate commercial platform: https://docs.langchain.com/langsmith/home; crewai: observability via external integrations (AgentOps, Langfuse, ...): https://docs.crewai.com/en/observability/overview; ag2: runtime logging to SQLite, no bundled UI: https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/runtime-logging/
- **Durable execution / checkpoint-resume** — aura: aura/workflows/engine.py, aura/agents/checkpoint.py; langgraph: checkpointers (SQLite/Postgres/memory): https://docs.langchain.com/oss/python/langgraph/persistence; crewai: task replay from a crew kickoff; memory persistence: https://docs.crewai.com/en/concepts/memory; ag2: resumable group chat via messages export: https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/resuming-group-chat/
- **Human-in-the-loop approval gates** — aura: aura/workflows/dsl.py checkpoint steps + approvals store; langgraph: interrupts: https://docs.langchain.com/oss/python/langgraph/interrupts; crewai: human_input on tasks: https://docs.crewai.com/en/learn/human-input-on-execution; ag2: human_input_mode on ConversableAgent: https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/human-in-the-loop/
- **License / stewardship (checked on PyPI)** — aura: MIT, this repo; langgraph: MIT, LangChain Inc — https://pypi.org/project/langgraph/ (1.2.9, 2026-07-10); crewai: MIT, crewAIInc — https://pypi.org/project/crewai/ (1.15.2, 2026-07-08); ag2: Apache-2.0, AG2AI (community, ex-AutoGen) — https://pypi.org/project/ag2/ (0.14.0, 2026-06-26); Microsoft's autogen-agentchat is in maintenance mode (0.7.5, 2025-09-30)
