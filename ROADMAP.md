# AURA Roadmap — toward a high-level agentic agent

AURA today is a strong **reactive** agent: a real ReAct tool-use loop, reflection +
debate, hub-and-spoke routing, self-building ontology synthesis, memory + a Kuzu
graph, guardrails + circuit breaker, a golden-set gate, OTLP, A2A, and a
Plan-and-Execute planner. "High-level" means it also **plans, verifies itself,
remembers deeply, and measurably improves over time.**

Every item below is one **bounded, gated sprint** — CPU-only, free OSS, added
behind an existing interface + env flag, and accepted only if all three gates stay
green:

```bash
AURA_LLM=stub python -m pytest tests -q
AURA_LLM=stub python example_ingest.py
AURA_LLM=stub python -m aura.evaluation.golden_ingest --report
```

Honesty rule (Law 1): progress is reported as measured deltas (test count, golden
score), never as "100%" or "1000×".

---

## Status legend
✅ done · 🟡 partial · ⬜ planned

---

## Tier 1 — Reasoning & reliability (the capability jump)
- ✅ **Plan-and-Execute planner** — `aura/orchestration/planner.py`, opt-in via
  `AURA_PLANNER=enabled`. Decomposes a goal into ≤`AURA_PLAN_MAX_STEPS` steps,
  executes each (tool-using), then synthesises. *(Sprint #1 — done.)*
- ✅ **Structured output (Pydantic schema + repair loop)** — `aura/ingest/schemas.py`
  `OntologyProposal`; the synthesizer validates the model's reply against it and
  **repairs malformed replies** (up to `AURA_STRUCTURED_RETRIES`) before falling back
  to lenient parsing. Done with **Pydantic, not Instructor** — Instructor patches the
  Anthropic client and would bypass AURA's LLM wrapper (Law 4) + the keyless stub.
  Flag `AURA_STRUCTURED=off` reverts to legacy parsing. *(Sprint #2 — done.)*
- ✅ **Verifier / self-consistency agent** — `aura/orchestration/verifier.py`:
  `verify(question, answer, evidence, votes)` checks groundedness via the cheap
  utility model, with **majority voting** for self-consistency. Wired into the
  copilot behind `AURA_VERIFY=enabled` (caveats an unsupported answer; fails open
  on a flaky judge). No new dep. *(Sprint #3 — done.)*
- ✅ **Hierarchical / typed-DAG planning** — `aura/orchestration/dag_planner.py`
  `DAGPlanExecutor`, opt-in via `AURA_PLANNER=dag`. The planner emits steps with
  explicit `needs` (dependency ids); they run in **topological order** with
  **dependency-scoped context** (each step sees only its transitive ancestors, not
  the whole trail). **Re-plans on step failure** (the worker raises / returns an
  empty-or-error result) by grafting a model-proposed recovery sub-plan, bounded by
  `AURA_REPLAN_MAX` (default 2), then degrades gracefully instead of crashing.
  **Bounded sub-goal recursion** decomposes a `complex` step into its own sub-DAG
  down to `AURA_PLAN_MAX_DEPTH` (default 2). Cycles are broken (never looped on).
  Degrades to a single linear step under the keyless stub; the flat `planner.py`
  stays intact. Closes flaw #2 and delivers frontier item **T2**. *(Sprint #23 —
  done; +14 tests, gate 98→112.)*

## Tier 2 — Memory & retrieval (the intelligence)
- ✅ **Full hybrid GraphRAG** — `aura/knowledge/retrieval.py`: deterministic
  multi-query **rewrite** + embedding **re-rank** + guaranteed **graph depth**
  (NetworkX or Kuzu multi-hop), behind `AURA_RETRIEVAL=hybrid`. Keyless re-rank
  meter added to Gate 3 (`check_retrieval`). *(Sprint #4 — done.)* Future: an
  optional LLM query-rewrite and a cross-encoder re-ranker (both need a model/key).
- ✅ **Hierarchical memory (Mem0)** — `aura/memory/mem0_backend.py` `Mem0Memory`
  adapter; `LongTermMemory` delegates when `AURA_MEMORY=mem0` (graceful fallback,
  injectable for tests). Namespace → Mem0 `user_id` gives **tenant-scoped memory**.
  `mem0ai` is installed; live use needs a BYOK Anthropic-LLM config + a first-run
  FastEmbed model download (so it is wired + gate-safe but not exercised offline).
  *(Sprint #11)*
- ✅ **Scalable vector index (`sqlite-vec`)** — opt-in `AURA_VECTOR=sqlite-vec` ANN
  KNN backend in `aura/memory/long_term.py` (falls back to exact cosine scan; auto-
  detects if the extension can't load). *(Sprint #9)*
- ✅ **Multi-turn conversation memory** — session-scoped short-term memory wired
  into the copilot (`session` / `session_id`): the agent remembers the conversation
  across turns; sessions are isolated. *(Sprint #13)*

## Tier 3 — Self-improvement (the autonomy)
- ✅ **Evaluator → optimizer loop (metric-gated)** — `aura/learning/optimizer.py`
  `optimize_prompt` (keep a variant only if it beats the current by `min_gain`,
  else revert — never regresses) + `prompt_store.py` (accepted prompts persist and
  the synthesizer reads them) + `run_improvement_cycle(metric)`. Dependency-free
  core (the DSPy lesson without the dep); **DSPy/TextGrad can later supply richer
  candidate proposers**. The live metric = golden-set recovery (needs a key);
  injected so the loop is tested offline. *(Sprint #5 — done.)*

## Tier 4 — Production & ecosystem
- ✅ **MCP client** — `aura/tools/mcp_client.py`: `register_mcp_tools(registry,
  transport)` bridges any external provider's tools into AURA's `ToolRegistry`
  (namespaced, schema-carried, traced, guard-governed) so the agent can act on real
  systems. Dependency-free bridge + a reference `StdioMCPTransport` (lazy `mcp` SDK,
  opt-in). *(Sprint #6 — done.)* Security: MCP servers are untrusted (CVE-2025-6514)
  — register only trusted ones; tools stay behind the tool guard.
- ✅ **Long-horizon reliability** — `aura/agents/checkpoint.py` `CheckpointStore`
  (SQLite) + `PlanExecutor(run_id=…, resume=True)` saves a checkpoint per step and
  **resumes after a crash** (skips completed steps); **re-anchor** compresses prior
  progress every `AURA_REANCHOR_INTERVAL` steps to fight context rot. No new dep.
  *(Sprint #7 — done.)*
- ✅ **Human-in-the-loop** — `aura/safety/approvals.py` `ApprovalStore` +
  `make_approval_guard` (write tools are **queued for human approval**, not run) +
  `/v1/approvals` list/decide endpoints. *(Sprint #8)*
- ✅ **Continuous evaluation (3 levels)** — component tests (pytest) + end-to-end
  golden meter (`--report`: profiler + retrieval) + **keyless trajectory eval**
  (`evaluate_trajectory`: tool-sequence quality, errors, loop & step-budget
  detection). DeepEval/RAGAS (LLM-judged) are documented opt-in adapters — heavy
  dep + key, kept out of the offline gate. *(Sprint #12)*
- ✅ **Observability** — OTLP export ✅ (`AURA_OTEL_ENDPOINT`) + aggregate
  `system_metrics()` via dashboard `/api/metrics` and server `/v1/metrics`
  (cache-hit rate, tool-error rate, token spend) *(Sprint #8)* + the **Agent Theater**
  (`/theater`): an animated, narrated replay of a run's orchestration. *(Sprint #9)*
- ✅ **Serving at scale** — background job queue (`/v1/ingest_async`, `/v1/jobs/{id}`)
  *(Sprint #8)*; **RBAC + multi-tenant** principals (`AURA_PRINCIPALS`, `require_role`,
  `/v1/whoami`); **true token-level streaming** (`/v1/chat_tokens` via `llm.stream`);
  **Postgres `TraceStore`** behind the interface (`AURA_TRACE_STORE=postgres`, lazy
  psycopg, graceful fallback to SQLite — not gate-verified without a server). *(Sprint #10)*

---

## Wave 1 — Adoption & packaging (Sprints #24–#28)
Not a capability wave: make a stranger's path from `git clone` to a running,
keyless demo short and verifiable. Per the value model (value = capability ×
credibility × discoverability × ease-of-adoption), this is the highest
realized-value-per-hour work available 23 capability sprints deep. Golden-set is
**N/A** for the whole wave (packaging, not accuracy).
- ✅ **`pyproject.toml` with extras (PEP 621)** — `aura/` is now installable as
  `pip install aura-agents`; every optional backend in `requirements.txt` maps to
  an extra (`aura-agents[kuzu,fastembed,mem0,otel,mcp]`, plus `[all]` and
  `[dev]`). Version is read dynamically from `aura.__version__` (single source of
  truth). The wheel ships only the `aura` package tree + the dashboard/theater
  static assets — verified no local `aura_bridge/` working copy, `aura.db`, or
  `node_modules` leaks in (`include = ["aura","aura.*"]`, not the `aura*` glob).
  `requirements.txt` stays as the documented `pip install -r` fallback. No runtime
  code touched; gate unchanged (119 tests, golden 5/5 · 2/2). *(Sprint #24 — done.
  Closes P1.)*
- ✅ **GitHub Actions CI (the three keyless gates)** — `.github/workflows/gate.yml`
  runs on every push/PR to `main`/`dev` (+ manual dispatch), zero secrets
  (`AURA_LLM=stub`): **gate job** on a Python 3.10/3.11/3.12 matrix installs
  `pip install -e ".[dev,kuzu,sqlite-vec]"` (exercising #24's packaging + the two
  keyless-testable optional backends, so CI runs the full 119-test suite — not the
  112-test subset a lean install silently collects) then runs all three gates;
  **package job** rebuilds the wheel, `twine check`s it, and asserts the #24
  hygiene invariants (only the `aura` tree + 19 static assets ship; no `.db`/
  `node_modules`/sibling-copy leaks); **theater job** runs `npm ci && vite build`.
  README's self-reported badges (one was stale: "tests-84") replaced by a live
  `shields.io/github/actions` status badge. Gate 3 already exits non-zero on any
  golden regression, so red = real. All three jobs rehearsed locally in a fresh
  venv before push. *(Sprint #25 — done. Closes P2.)*
- ✅ **Dockerfile + docker-compose (keyless-default)** — `docker compose up` →
  self-seeded dashboard on `:8000`, no key/GPU/network. `python:3.12-slim`
  CPU-only image (no CUDA, no build toolchain, core deps only; requirements.txt
  as the cached dep layer, then `pip install --no-deps .` exercising #24's
  packaging); non-root user; all runtime state under `/data` (volume-persisted —
  incl. `AURA_STORE=/data/knowledge_store`, without which first-boot seeding
  would crash on the root-owned cwd); first boot runs `seed_demo.py` if the db
  is absent (inline `sh -c` CMD, no entrypoint file → no CRLF footgun);
  HEALTHCHECK hits `/api/metrics`. `.dockerignore` keeps `.venv`/`node_modules`/
  zips/dbs out of the context. The plan's manual acceptance test is **automated
  as a CI job** (`docker demo boots keyless`): compose up → poll `/api/traces`
  for seeded traces → assert `/` + `/api/metrics` → teardown. README gains the
  Docker-first quickstart (+ stale "84 passed" fixed → 119). Boot sequence
  rehearsed locally end-to-end (seed → 7 traces → dashboard 200s) — Docker
  itself isn't installed on the dev box, so the live container run is verified
  by the CI job on ubuntu. *(Sprint #26 — done. Closes P3.)*
- ✅ **`aura` CLI** — `aura/cli.py`, installed as a console script
  (`[project.scripts]`): `aura demo | ingest | chat | serve | gate | report |
  version` — thin typer wrappers over entry points that already exist, no new
  logic. `typer>=0.26` (verified maintained, released 2026-06; vendors click →
  one light MIT dep) added to **core**, not an extra, because the CLI *is* the
  adoption surface (`pip install aura-agents && aura demo`). Seeding logic moved
  package-native (`aura/demo.py`; `seed_demo.py` stays as the shim Docker runs).
  `demo`/`report` default to the keyless stub; `demo` reads `AURA_DB` live for
  the same first-boot check as Docker. +7 CLI smoke tests (one per subcommand;
  the demo test spies `seed()` — a real in-process seed floods the shared test
  db and trips the context compressor in later tests, an extra stub turn that
  broke `test_supervisor_routes` until isolated). CI gate job gains a "CLI
  smoke" step running the installed entry point (`aura version|--help|report`).
  Gate: **119 → 126 tests**; golden 5/5 · 2/2 (N/A — no accuracy claim).
  *(Sprint #27 — done. Closes P4.)*
- ✅ **CHANGELOG + `v0.9.0` tag + repo-hygiene pass** — `CHANGELOG.md` derived
  from this sprint log (Sprints #1–#27 + TH track, grouped by tier, honest
  numbers: 126 tests · 5/5 · 2/2); hygiene audit confirmed **zero** build
  artifacts tracked (the gaps doc's P5 was zip-export noise, as suspected) and
  `*.tsbuildinfo` added to `.gitignore`; placeholder LICENSE replaced with the
  real MIT text (GitHub license detection needs it); `__version__` reconciled
  1.0.0 → **0.9.0** per this plan — `v1.0.0` stays reserved for the Wave-3
  self-learning release (Sprint #40) — and the bump propagates everywhere from
  the single source (wheel, `aura version`). Annotated tag `v0.9.0` + GitHub
  release with the changelog. *(Sprint #28 — done. Closes P5, P7.)*

**Wave 1 exit criteria: met.** `docker compose up` → seeded, keyless demo in
one command (CI-verified boot); the CI badge is real (three keyless gates on
every push, 3.10–3.12); `v0.9.0` is tagged with a changelog derived from this
log.

---

## Wave 2 — Make it measurably smarter (Sprints #29–#34)
Capability sprints, gated on the scoreboard existing first (Law 5): no sprint
below may claim an accuracy delta that #29's fixtures can't show.
- ✅ **Golden-QA expansion: routing category (G5)** — Gate 3 grows a third
  keyless category: `routing_golden()`/`check_routing()` in
  `aura/evaluation/golden_ingest.py` scores the supervisor's **deterministic
  routing layer** — semantic-discovery ranking (4 fixtures), exact-name
  acceptance of a valid LLM reply (1), rejection-and-recovery from garbage LLM
  replies: unknown name / empty / verbose sentence (3), and degraded-worker
  exclusion under cascade-guard semantics (1) — **9 fixtures, 9/9 at baseline**.
  Scoreboard: 2 → 3 keyless categories (5 profiler + 2 retrieval + 9 routing =
  16 fixtures), all wired into `--report`'s exit code — a routing MISS fails
  Gate 3 (test-proven, not assumed). Fixtures use vocabulary overlapping the
  right worker's capability description (the lexical embedding is a hashed
  bag-of-words — deterministic anywhere, no key). GOLDEN_TASKS also gains 2
  citation fixtures for the live-key answer check (feeds #32's
  cite-or-say-unknown). This is the baseline #30 (typed contracts) and #31
  (fan-out) must beat or hold. Gate: **126 → 130 tests**, golden 5/5 · 2/2 ·
  9/9. *(Sprint #29 — done. Closes G5.)*
- ✅ **Typed inter-agent contracts (T3)** — `aura/orchestration/contracts.py`
  generalizes the `OntologyProposal` pattern (proven in `aura/ingest/schemas.py`)
  to the DAG planner's step handoffs, behind `AURA_CONTRACTS=enabled`: each
  successful step's output is coerced into a schema-validated **`StepResult`**
  (`summary` ≥1 char, `evidence[]`, `confidence` 0–1, `unresolved[]`) before any
  dependent consumes it — dependents receive the validated summary, never the
  raw blob; the full contract travels on the entry for observability/Theater.
  Same discipline as the synthesizer: validate → **repair** via the utility
  model (bounded by `AURA_STRUCTURED_RETRIES`, format-only, never invents
  content) → **lenient wrap** (`confidence=0.5`, failure noted in `unresolved`)
  so the prose-only keyless stub still runs end-to-end. A contract whose
  summary is an ERROR marker **demotes the step to error** and feeds the
  existing replan machinery — validation reveals failures, never masks them.
  Flag off ⇒ byte-identical legacy behavior (test-proven). No new dependency
  (Pydantic already core). Gate: **130 → 138 tests**; golden routing category
  **held 9/9** (baseline → target: no regression; the fan-out sprint #31 builds
  on these contracts for its merge step). *(Sprint #30 — done. Closes T3.)*
- ✅ **Subagent context isolation + parallel fan-out** —
  `aura/agents/parallel_supervisor.py` `ParallelSupervisor`, opt-in via
  `AURA_SUPERVISOR=parallel` in `build_supervisor` (the classic single-route
  `Supervisor` stays the default, untouched). The top-`AURA_FANOUT_K` workers by
  deterministic semantic discovery run **concurrently** (real threads; the trace
  store was already lock-protected) with **per-worker context isolation** — each
  spoke sees only the bare task, never a sibling's output or a shared trail —
  and each thread runs in its own `contextvars.copy_context()` so worker spans
  nest in the ambient trace (test-proven) instead of fragmenting. Results merge
  **through #30's typed contracts** (validated summaries, never raw text), and
  divergent confident views (confidence ≥ `AURA_CONFLICT_MIN`, lexical cosine <
  `AURA_CONFLICT_SIM`) **queue a `supervisor:merge_conflict` review flag** in
  the approvals store — surfaced, not silently resolved, exactly as the sprint
  card required. One failed spoke doesn't sink the fan-out (cascade guard still
  marks it); all-fail degrades to the classic graceful error shape (flaw #3
  parity). Found & fixed in test: the merge synthesis needed `use_cache=False` —
  a cached merge could mask fresh worker results. Gate: **138 → 147 tests**
  (suite run 3× for order stability); golden routing **9/9 → 12/12** (3 new
  fan-out selection fixtures: top-k set correctness + degraded-member
  replacement). *(Sprint #31 — done. Closes G1's remainder.)*
- ✅ **Agentic retrieval loop + evidence-sufficiency / cite-or-say-unknown** —
  `AURA_RETRIEVAL=agentic` in `aura/knowledge/retrieval.py` (`agentic_context`):
  a bounded keep-retrieving-until-sufficient loop **around** the existing hybrid
  pass (round 1 *is* the hybrid pass; `hybrid` mode untouched).
  `evidence_sufficiency()` is the deterministic stop-signal — fraction of the
  query's content terms covered by gathered evidence, threshold
  `AURA_EVIDENCE_MIN` (0.6) — and `followup_query()` re-targets each next round
  at the **missing** terms instead of re-asking the same question harder, up to
  `AURA_RETRIEVAL_MAX_ROUNDS` (3). Evidence dedups across rounds. If the loop
  ends insufficient, the context block carries an explicit **cite-or-say-unknown
  instruction** (state coverage, name what's missing, do not guess) — the
  before-answering check the plan asked for; the live-key citation fixtures in
  GOLDEN_TASKS (#29) score the answer side. **Token-spend: the loop itself makes
  zero LLM calls** (pure recall + term math), so the cost lever is bounded
  recall rounds, traced per-run (`retrieval:agentic` span records rounds +
  coverage). Golden retrieval category **2/2 → 5/5** (sufficiency ×2 +
  gap-targeting fixtures). Gate: **147 → 156 tests** (suite 2×), routing held
  12/12. *(Sprint #32 — done. Closes part of G2.)*
- ✅ **Live SSE feed (D1)** — `aura/tracing/stream.py` `sse_span_events`: a
  typed event stream (`event: span` / `event: ping` heartbeat — the AG-UI-style
  "structured event, not free text" shape) over a new lock-protected
  `store.spans_since(rowid)` cursor; REPLACE re-inserts an updated span under a
  new rowid, so **start and completion both emit** (lifecycle events for free).
  Mounted twice from the one generator: `GET /v1/traces/stream` on the serving
  API (behind `require_key`, like `/v1/metrics`) and `GET /api/traces/stream`
  on the dashboard — declared **above** `/api/traces/{trace_id}` (FastAPI
  matches in declaration order; the parameterized route was swallowing
  `stream` as a trace id, caught by test). Dashboard header gains a **⚡ Live**
  toggle (EventSource; pulses per span, throttled refresh); **replay stays the
  default/offline path** — the feed is additive, poll-based, bounded via
  `max_seconds` for finite/test streams. The 3D theater_app "Live" tab is
  **deferred to the TH track** (needs an R3F rebuild at the keyboard; the feed
  it will consume now exists). Gate: **156 → 159 tests**; golden N/A
  (transport, no accuracy claim). *(Sprint #33 — done. D1 closed; TH live-tab
  open.)*
- ✅ **Report the wave: release notes with measured before→after deltas.** CHANGELOG [0.10.0]
  ledgers the wave (126→159 tests, golden 5/5·5/5·12/12, 2→3 categories, 7→22 fixtures); v0.10.0
  tagged. *(Sprint #34 — done. Closes the Wave 2 exit criterion.)*

**Wave 2 exit criteria: met.** Scoreboard exists and runs in CI (✓ #29); router has
isolation+contracts (✓ #30/#31); live SSE feed (✓ #33, 3D-theater consumer deferred to
TH track); before→after deltas in the release notes (✓ this sprint, v0.10.0).

- ✅ **Langfuse integration bridge sprint** —
  `aura/tracing/langfuse_bridge.py`, behind `AURA_LANGFUSE=enabled` (+ BYOK
  `LANGFUSE_PUBLIC_KEY`/`LANGFUSE_SECRET_KEY`/`LANGFUSE_HOST`). Law-2
  verified in-session (2026-07-06): Langfuse is **MIT** (core; `ee/`
  excluded), self-hostable, very actively maintained (multi-weekly releases;
  acquired by ClickHouse Jan 2026); the Python SDK was **rewritten for v4 in
  March 2026** — an API in flux, so the bridge deliberately does NOT depend
  on the SDK: it builds events for Langfuse's **documented HTTP ingestion
  API** behind a client protocol; the real transport is stdlib urllib
  (**zero new dependency**), tests inject a fake. Mapping: completed traces
  → `trace-create` (userId = tenant, tags carry tenant+status, sessionId
  from `meta.session`); llm spans → `generation-create` (usage tokens,
  model); every other kind → `span-create` (kind in metadata, errors as
  `level: ERROR`); verifier verdicts → `score-create`
  (`verifier.grounded` = vote confidence, wired); accepted prompt versions
  → prompt management (wired into `set_prompt`, the promote path). Hooks
  ride the SAME completed-item seam as the OTLP exporter — additive, never
  raises, first transport failure disables the bridge for the process (the
  OTLP precedent). A span whose trace this process will never export (e.g.
  a #56 remote shipment) gets a stub trace so it lands visibly; spans inside
  a live trace don't (exactly one `trace-create` per trace, test-proven).
  **Acceptance, all test-asserted keylessly:** one trace round-trips
  end-to-end (trace + nested generation + span, correct parentage); one
  evaluation score visible; **export-only** — the module has no read
  surface (asserted by test), flag-off = zero events, and the SQLite
  store + OTLP path remain the default. Gate: **398 → 406 tests**.
  *(Sprint #34b — done. The Wave-2 deferral ledger is now empty.)*

---

## Already shipped (foundation)
Tool-use loop · reflection · debate · supervisor routing · ontology synthesizer
(propose→reflect→validate) · profiler with self-ref FK / junction / enum detection ·
knowledge graph (NetworkX default, **Kuzu** opt-in) · long-term + short-term memory ·
ACE context playbook · hybrid retrieval (graph expansion) · guardrails (injection
screen, memory-poisoning screen, risk-scored tool guard, write-confirm) · circuit
breaker + retries · golden-set evaluator + keyless `--report` CLI · OTLP hook ·
A2A stub · dashboard (+ Basic-auth) · streaming FastAPI serving API · keyless stub
LLM + test gate.

## 2026-Ultra frontier track (T1–T8) + flaw fixes
From the Ultra kickoff plan. Implemented **domain-agnostically** (no vertical hardcoding).
- ✅ **T6 — FinOps per-agent cost tracking** — every LLM span records `cost_usd`
  (config-driven `AURA_PRICES`; 0 if unset); `system_metrics` + `/v1/cost_report`
  aggregate by model. *(Sprint #14)*
- ✅ **Flaw #1 — token counting** — `estimate_tokens` is now conservative (won't
  silently overflow the budget) + optional `AURA_TOKENIZER=tiktoken`. *(Sprint #14)*
- ✅ **T7 — cascade-failure guard + flaw #3 fallback** — `AgentRegistry` tracks
  consecutive failures and quarantines a worker after `AURA_CASCADE_THRESHOLD`
  (default 3); the `Supervisor` reroutes past failed/degraded workers and
  **fails gracefully when none succeed** (no more uncaught IndexError).
  Behind `AURA_CASCADE_GUARD=enabled`. *(Sprint #15)*
- ✅ **T5 — memory provenance/quarantine + flaw #4** — recalled memories carry
  `source`/`confidence` in meta; on recall, untrusted-source or below-threshold
  entries are quarantined (kept out of the prompt) and flagged to the approvals
  queue for human review. Behind `AURA_MEMORY_PROVENANCE=enabled`
  (`AURA_MEMORY_MIN_CONFIDENCE`, default 0.5). Wired into both recall paths
  (`KnowledgeBase.context_for` + hybrid GraphRAG). OWASP ASI06. *(Sprint #17)*
- ✅ **T1** speculative planning (`AURA_SPECULATE=enabled`, *Sprint #48b*) · ✅ **T2** typed-DAG planner (`AURA_PLANNER=dag`,
  `aura/orchestration/dag_planner.py`, *Sprint #23*) · ✅ **T3** typed inter-agent
  contracts (`AURA_CONTRACTS=enabled`, `aura/orchestration/contracts.py`,
  *Sprint #30*) · ✅ **T4** SimpleMem-pattern memory compression
  (`aura/memory/compressor.py`, *Sprint #39*) · ✅ **T8** flip-gate regression
  detector (`aura/evaluation/flip_gate.py`, *Sprint #37*).
- ✅ **Flaw #5 — hard dedupe** — learning loop now uses an exact meta match
  (`exists_with_meta`) instead of fuzzy recall, so a trace is learned once. *(Sprint #16)*
- ✅ **Stunning UI** — dashboard + Agent Theater redesigned: animated aurora
  backgrounds, glassmorphism, shimmer gradient titles, count-up metrics, glowing
  grow-in waterfall bars, cross-links. *(Sprint #16)*
- ✅ **Conversation Theater + content capture** — `/theater` gains a second tab:
  **Conversation** renders the agents' real dialogue as animated, streamed chat
  bubbles (two-agent debate, supervisor→worker, tool I/O) with a live inspector
  (system prompt, tool input/output, tokens, cost) on click; **Timeline** keeps
  the waterfall. Fed by opt-in, redaction-aware span content capture
  (`AURA_TRACE_CONTENT=off|summary|full`, secrets always redacted, PII opt-in)
  and real `tool` spans now emitted by the tool-use loop. *(Sprint #18)*
- Open flaws: ~~#2 flat planner~~ ✅ **closed by the typed-DAG planner (Sprint #23)**;
  ~~#6 advisory-only MCP trust~~ ✅ **closed by MCP trust enforcement (Sprint #38)**.
  **No open flaws remain.**

## AURA Theater — 3D agent experience
A futuristic front end: every agent becomes a rigged 3D character (Roblox-style),
debates happen around a round table, the judge rules in a courtroom, with full
cinematics — fed by real trace data. **15-sprint vision: see
[AURA_THEATER_ROADMAP.md](AURA_THEATER_ROADMAP.md).**
- ✅ **TH-1 — Cinematic 3D theater shipped** at **`/theater3d`**
  (`aura/dashboard/static/theater3d.html`): three.js + UnrealBloom + ACES tone
  mapping + GSAP camera cuts/typewriter. Auto-classifies each trace into a scene
  — **courtroom** (eval/judge/verify), **round table** (debate), **knowledge lab**
  (plan/ingest/synth), **command floor** (else) — and replays it with characters,
  captions, click-a-character trace cards, quality presets, auto/free camera, and
  keyboard controls. Pure client of `/api/traces`; backend stays CPU-only. Falls
  back to embedded real episodes offline and to the 2D `/theater` where WebGL is
  unavailable. Cross-linked from the dashboard + 2D theater. *(Sprint #19)*
- ✅ **TH-1b — Self-hosted / no-CDN.** three.js core + the 6 postprocessing/
  control addons (11 files, dependency-followed) and GSAP are vendored under
  `static/vendor/` and served via a mounted `/vendor` static route; the import
  map + script tag point local. `/theater3d` now runs on an air-gapped intranet
  with zero external requests (verified: no `unpkg`/`https://` refs remain).
  *(Sprint #20)*
- ✅ **TH-2 — React Three Fiber rebuild** (`theater_app/`, monorepo subfolder).
  The full requested stack: R3F + three.js, **@react-three/postprocessing**
  (bloom / depth-of-field / vignette / ACES), **custom GLSL** energy-glow on
  every agent core, **GSAP** auto-cinematography, **Framer Motion** HUD,
  **React Flow** workflow with a custom animated "energy" edge, and
  **Cytoscape** for the knowledge graph. Builds to
  `static/theater3d_app/`; backend serves it at `/theater3d` (StaticFiles), with
  the vanilla single-file demoted to `/theater3d-lite` and the 2D at `/theater`.
  No CDNs — fully bundled / intranet-safe. *(Sprint #21)*
- ✅ **TH-4 — character models.** Agents are now procedural **Roblox-style
  blocky humanoids** (head/torso/arms/legs/face, no external assets) instead of
  orbs: idle bob + arm sway, a **talking animation** (head nod, gesturing arm,
  animated mouth) and brighter emissive + energy-glow shell on the active
  speaker, each facing the stage centre. *(Sprint #22)*
- ⏳ **Open:** TH-5…TH-15 (per-agent TTS voices, lip-sync to real tokens,
  code-split the 1.85 MB bundle, recorded/shareable episodes, free-orbit camera
  toggle, live SSE feed).

## Wave 3 — Self-learning & v1.0.0 (Sprints #35–#42)
The wave that earns the 1.0 tag. But **self-learning only counts if the runtime is
trustworthy first**: before AURA may claim it can improve itself, it must close the
gap between the roadmap's production promises and the actual serving/runtime
behaviour — especially tenant isolation, auth parity, reliability parity across
sync/async/streaming, provenance hardening, and a reproducible dev/test runtime.
Wave 3 is therefore **hardening first, learning second**, all gated by the existing
scoreboard.

- ✅ **Multi-tenant isolation + serving contract hardening** — SHIPPED: knowledge-base
  state is now tenant-scoped by construction (`TenantStates` registry keyed by
  `principal.tenant`, replacing the old process-global `STATE` dict — no code path
  lets one tenant's write touch another's slot); every `/v1` handler now consumes
  `principal: Principal = Depends(require_role(<floor>))` per an explicit
  viewer/user/admin floor table; `ingest_async` closed the auth-parity hole and now
  requires the same `admin` floor as `ingest`; the `require_key` no-op in principals
  mode is closed (a missing/unknown key now 401s on viewer-floor endpoints instead of
  sneaking through); ingest for a non-default tenant persists to a tenant-scoped
  `store_dir`; and tenant is sanitized once at `Principal` resolution
  (`^[A-Za-z0-9_-]{1,64}$`, else 400) so no consumer needs to re-check it; background
  jobs carry an owner tenant and `/v1/jobs/{id}` 404s (not 403) for a cross-tenant
  request.
  **DEFERRED (explicitly out of scope this sprint, tracked for a later bounded
  sprint):** traces (`/v1/traces/stream`, `/v1/metrics`, `/v1/cost_report`) are
  role-gated but still read one global cross-tenant trace store; approvals
  (`/v1/approvals` GET and `{id}` decide) are role-gated but operate on a single
  global `approval_store`, not tenant-partitioned; long-term-memory namespaces are
  derived from the ontology's `domain_name`, not the tenant, so two tenants ingesting
  same-named domains still share a memory namespace (tenant-scoped store dirs prevent
  KB file overwrite but do not isolate the shared long-term-memory db); and the
  ingest inbox is still a single shared `AURA_INBOX` source folder across all tenants
  (only the ingest *output* is tenant-scoped this sprint, not the input).
  *(Sprint #35)*
- ✅ **Runtime reliability parity + reproducible environment** — the same safety and
  reliability guarantees now hold across **all** answer paths: single-shot, tool-use,
  SSE, token streaming, and background jobs. `aura/llm/client.py` `LLM.stream()`
  (real-backend branch) now checks the circuit breaker's `allow()` before opening a
  stream and retries a retryable failure with the same bounded backoff as `_create`
  — but **only before the first delta has been yielded**; a half-delivered stream is
  never silently replayed. `record_success()`/`record_failure()` mirror `_create`'s
  discipline. Guard semantics are now identical across every answer path: `/v1/chat`,
  `/v1/chat_tokens`, and `/a2a/task` all run `scan_input` before sanitizing and surface
  the same `{"warning": ...}` shape when a message is flagged (fail-open — the screen
  warns, it doesn't block). `aura/server/jobs.py` `status()`/`wait()` now return a
  typed failure: `error_type` (the exception class name) alongside `error` (the
  message), so a caller can tell a guard rejection from a transient API failure from a
  bug without parsing strings. Finally, the repo gets one boring, pinned dev runtime
  story: `.python-version` (3.11) plus a README "Supported runtime" section
  documenting the exact editable install CI already runs
  (`pip install -e ".[dev,kuzu,sqlite-vec]"`) — the existing end-user quickstart is
  unchanged; CI's 3.10–3.12 matrix remains the compatibility proof and Docker still
  runs 3.12-slim. *(Sprint #36)*
- ✅ **Flip-gate regression detector (T8)** — `aura/evaluation/flip_gate.py`
  tracks bounded per-fixture pass/fail history (`AURA_FLIP_HISTORY` JSON,
  gitignored; window `AURA_FLIP_WINDOW`=10). A fixture with ≥`AURA_FLIP_THRESHOLD`
  (3) green↔red transitions in the window is **flaky — a flaky pass is not a
  real pass** — and held red until `AURA_FLIP_RECOVERY` (3) consecutive passes;
  a recovery-length green tail also vetoes re-flagging from old flips still in
  the window. Wired into Gate 3's exit code as a fourth report block
  (`golden-set (flip-gate)`) over all 22 fixtures; `AURA_FLIP_GATE=off`
  disables; unreadable/corrupt history degrades to pass-through — the detector
  never breaks the gate it guards. Honesty note: a fresh CI checkout has no
  history, so the flip-gate bites locally and on deployments; caching the file
  in CI is a later step. Gate: **176 → 185 tests** (suite ×2), golden
  5/5·5/5·12/12 + flip-gate stable. *(Sprint #37 — done. Closes T8.)*
- ✅ **MCP trust enforcement (Flaw #6)** — `aura/tools/mcp_trust.py`: declarative
  JSON policy (`AURA_MCP_TRUST_FILE`) whitelisting servers + tools, with optional
  **sha256 pinning** of the stdio command binary. With a policy configured,
  enforcement is ON: an unlisted server is **blocked at registration**
  (`MCPTrustError`), unlisted tools are skipped (audibly) and **re-checked at
  call time** (a policy tightened after registration refuses the call before it
  reaches the server — test-proven), a tampered pinned binary is refused, and a
  corrupt/unreadable policy **fails closed** while enforcing (an allow-list must
  never fail open). Every invocation AND refusal is appended to the JSONL audit
  log (`AURA_MCP_AUDIT_FILE`, gitignored). No policy file = legacy advisory mode
  (existing users unbroken, audit still recorded); `AURA_MCP_TRUST=off` is the
  explicit escape hatch. Gate: **185 → 194 tests** (suite ×2), golden
  5/5·5/5·12/12 + flip-gate stable. *(Sprint #38 — done. Closes Flaw #6 — the
  last open security flaw.)*
- ✅ **SimpleMem compression (T4) + provenance normalization** —
  `aura/memory/compressor.py` implements the SimpleMem consolidation pattern
  (arXiv:2601.02553, aiming-lab/SimpleMem — verified this session)
  **dependency-free** on AURA's own embeddings: greedy cosine clustering
  (`AURA_MEM_COMPRESS_SIM`=0.92) → representative merge (longest text; merged
  meta records `merged_from`/`compressed_at` and keeps the cluster's max
  numeric confidence) → **recoverable tombstones** (`memories_tombstone` table;
  `restore_namespace()` undoes a pass exactly — test-proven back to byte-state).
  `maybe_compress` is gated by `AURA_MEM_COMPRESS=enabled`; scheduling is
  operator-side (cron/jobs), no background threads. **Gated, not blind:** the
  golden-set gains a fourth category — `memory compression` (4 fixtures: the
  duplicate cluster must merge 5→3 AND every recall check must still answer
  post-compression; ratio printed) — categories **3 → 4**, flip-gate now tracks
  **26** fixtures. Provenance normalization: `screen_provenance` no longer
  crashes on malformed confidence — missing passes (opt-in), unparseable
  (`"high"`, `[]`, `{}`) **quarantines fail-safe**, numeric strings parse, and
  non-dict meta degrades. Gate: **194 → 207 tests** (suite ×2).
  *(Sprint #39 — done. Closes T4.)*
- ✅ **LLM-powered query rewrite + cross-encoder re-ranker** — two opt-in BYOK
  stages in `aura/knowledge/retrieval.py`, both **fail-back-to-deterministic**:
  (1) `AURA_REWRITE=llm` — `rewrite_query_llm` asks the utility model for ≤3
  retrieval-optimised sub-queries and **unions them with the deterministic
  variants (the floor, never replaced)**; garbage/API failure returns the
  deterministic set alone (test-proven). (2) `AURA_RERANKER=cross-encoder` —
  `rerank_cross_encoder` scores (query, candidate) pairs jointly via a lazily
  loaded `sentence-transformers` CrossEncoder (`AURA_CROSS_ENCODER_MODEL`,
  default ms-marco-MiniLM-L-6-v2, CPU inference; new install extra
  `aura-agents[cross-encoder]`, mirrored in requirements.txt); load/score
  failure degrades to the embedding re-rank. Both wired into `_hybrid_lines`
  (recall union + rank dispatch); flags off = byte-identical legacy behavior.
  Golden-set: **N/A by design** — BYOK-only stages; the keyless categories held
  (5/5·5/5·12/12·4/4) and remain the gate. Gate: **207 → 215 tests** (suite ×2).
  *(Sprint #40 — done.)*
- 🟡 **DeepEval/RAGAS live-key evaluation adapters + optimizer proof** — the
  machinery is done and keyless-tested; the live-key demonstration is a
  one-command operator step. Shipped: `aura/evaluation/deepeval.py` +
  `aura/evaluation/ragas.py` (thin adapters — faithfulness / answer relevancy /
  context precision; lazy SDKs behind new extras `[deepeval]`/`[ragas]`, helpful
  install errors, SDK touchpoints isolated in `_evaluate_with_sdk` for test
  injection and version-drift containment) and
  `aura/evaluation/optimizer_proof.py` — the **before→after receipt harness**:
  per-category baseline → `optimize_prompt` (metric-gated, never regresses,
  test-proven) → after-score, then the FULL keyless golden gate + flip-gate
  re-runs and can **veto** the win; writes `OPTIMIZER_PROOF.md` with the ledger
  and an honest verdict (non-improving categories reported, `NOT PROVEN` when
  <2 improve or the gate regressed — all test-proven with fake judges).
  **Outstanding (operator, BYOK, deliberate token spend):** run
  `python -m aura.evaluation.optimizer_proof` with a configured judge and
  commit the generated `OPTIMIZER_PROOF.md` — that artifact is release
  criterion for #42. Gate: **215 → 223 tests** (suite ×2); golden N/A (keyless
  categories held 5/5·5/5·12/12·4/4). *(Sprint #41 — machinery done; live
  receipt pending.)*
- ✅ **v1.0.0 release — the trusted self-learning milestone** — CHANGELOG
  `[1.0.0]` ledgers the wave (tests **159 → 223**, golden categories **3 → 4**,
  fixtures **22 → 26**, open flaws **1 → 0**, T4+T8 closed), `__version__`
  bumped to `1.0.0`, annotated tag + GitHub release. Release criteria met:
  tenant isolation proven in tests (#35), async/streaming/runtime paths match
  sync reliability semantics (#36), memory compression preserves recall (#39,
  golden-gated), MCP trust enforced (#38). **One criterion shipped as an
  explicit deferral by operator decision:** the live-key optimizer
  demonstration — the harness is complete and keyless-proven (#41); the receipt
  is one deliberate BYOK command (`python -m aura.evaluation.optimizer_proof` →
  commit `OPTIMIZER_PROOF.md`). *(Sprint #42 — done; v1.0.0 tagged.)*

**Wave 3 exit criteria:** `v1.0.0` tagged ✓. The runtime contract is trustworthy
single-node and tenant-safe ✓, the flip-gate proves stability ✓, memory
compresses without recall loss ✓, MCP trust is enforced ✓, and the optimizer
loop's improvement proof is **machinery-done with the live-key receipt
deferred** (operator decision, stated in the release notes — the harness
enforces "measured, not claimed" whenever it runs).

**Deferred until after the Wave 3 hardening gate:** speculative planning (T1) and
DSPy/TextGrad proposer integration stay on the roadmap, but they no longer outrank
tenant isolation, runtime parity, or trust enforcement on the path to `v1.0.0`.

---

## Wave 4 — Multi-model & ecosystem (Sprints #43–#54)
AURA today is Anthropic-only. Wave 4 makes it a **model-agnostic, ecosystem-native**
framework — any LLM, any tool protocol, any plugin.

- ✅ **OpenAI backend** — `aura/llm/openai_client.py` behind `AURA_LLM=openai`:
  an **adapter speaking the same client protocol** the resilient `LLM` wrapper
  already talks to (`messages.create`/`messages.stream`, Anthropic-shaped
  responses — exactly how `StubClient` plugs in), so breaker, bounded retries,
  spans, cache and cost tracking apply **unchanged** — `client.py` gained one
  `elif` branch. Transparent translation both ways: tools
  (`input_schema` ↔ `function.parameters`), multi-turn tool loops (assistant
  `tool_use` ↔ `tool_calls`, user `tool_result` → `role:"tool"` turns), system
  cache-control blocks → system text, usage/stop_reason mapping, malformed
  tool-arguments degrade (`{"_raw": …}`) instead of crashing, and Claude model
  names substitute `AURA_OPENAI_MODEL` (default `gpt-4o-mini`) so the global
  instances work unmodified. Streaming adapter matches the Anthropic stream
  contract (`text_stream` + `get_final_message`, usage via
  `stream_options.include_usage`) — #36's stream retry/breaker discipline
  covers it for free. New extra `aura-agents[openai]`; BYOK via
  `OPENAI_API_KEY`; never in the keyless gate. Vision explicitly deferred to
  Anthropic backends (placeholder text, stated). Gate: **223 → 232 tests**
  (suite ×2, incl. full fake-SDK integration through the untouched wrapper).
  *(Sprint #43 — done. Wave 4 opens.)*
- ✅ **Gemini backend** — `aura/llm/gemini_client.py` behind `AURA_LLM=gemini`:
  the #43 adapter pattern with Google-shaped translation — tools →
  `function_declarations`, tool_use/tool_result ↔ `function_call`/
  `function_response` parts, assistant → `model` role, `usage_metadata` token
  mapping, `system_instruction` from cache-control blocks; Claude default model
  names substitute `AURA_GEMINI_MODEL` (default `gemini-2.5-flash`). Streaming
  matches the Anthropic stream contract, so #36's breaker/retry discipline
  applies for free. SDK touchpoints defensive + isolated (fake-SDK keyless
  tests; verify against your installed `google-genai` before live use). Extra
  `aura-agents[gemini]`; BYOK via `GOOGLE_API_KEY`. Gate: **232 → 239 tests**.
  *(Sprint #44 — done.)*
- ✅ **Local/Ollama backend** — `aura/llm/local_client.py` behind
  `AURA_LLM=local`: **fully air-gapped, zero-API-key operation.** Design
  economy: Ollama, llama.cpp and vLLM all expose OpenAI-compatible endpoints,
  so this subclasses the #43 adapter pointed at `AURA_LOCAL_BASE_URL` (default
  `http://localhost:11434/v1`) — every line of tool-use translation and
  streaming tested in #43 applies verbatim (test-proven: the tool-calling path
  runs through the local client unchanged). `AURA_LOCAL_MODEL` (default
  `llama3.2`) substitutes Claude default names; tool-use needs a
  function-calling-capable model. Extra `aura-agents[local]` (just the
  `openai` client package — no cloud, no key). Gate: **239 → 244 tests**.
  *(Sprint #45 — done.)*
- ✅ **Flip-gate history caching in CI** — closes the honesty note left in #37:
  a fresh CI checkout had no pass/fail history, so the flaky-fixture detector
  only bit locally. `gate.yml` now persists `.aura_flip_history.json` via
  `actions/cache` with a **rolling key** (each run saves under a unique
  branch+python+run key and restores the newest previous one), so green→red
  flip-flops accumulate across CI runs and a flaky fixture gets HELD RED in CI
  exactly as it does locally. No code change — the detector already read the
  env-configured path; golden N/A. *(Sprint #45b — done. Debt item #5 from the
  v1.0.0 deferred ledger.)*
- ✅ **Multi-model routing** — `aura/llm/router.py` + `AURA_MODEL_MAP` (JSON:
  role → `{backend, model}` or `{candidates: […]}`). `LLM` instances can now be
  **pinned to a backend** (`LLM(model, backend=…)`; None = follow `AURA_LLM`,
  exactly as before — the pin beats the env, test-proven), so the supervisor
  can run OpenAI while workers run Claude Haiku and the verifier runs a local
  model, each with its own breaker/cache/spans (instances cached per
  backend+model). Wired with behavior-preserving defaults into the supervisor
  (route + synthesis), the parallel-fan-out merge, the verifier, and `Agent`'s
  default worker LLM — **no map configured = byte-identical behavior**
  (test-proven). Cost-aware selection: among listed `candidates` the router
  picks the cheapest by the `AURA_PRICES` table (rate = in+out $/M); unpriced
  ties resolve to the operator's listed order. **Honesty note:** the card's
  "meets a quality threshold" needs per-model golden scores that don't exist
  yet — until they do, quality is vouched by the operator listing a candidate
  at all, and the router optimises price among approved options only (stated
  in the module docstring). Gate: **266 → 276 tests** (suite ×2).
  *(Sprint #46 — done. The five-backend capstone.)*
- ✅ **Plugin system** — `aura/plugins/loader.py` + `aura-plugin.json` manifest
  (JSON canonical, zero-dep; YAML accepted when PyYAML is present). A plugin
  dir under `AURA_PLUGINS_DIR` (default `plugins/`) declares `provides.tools`
  and `provides.workers` as `module:attr` targets; tools register namespaced
  `plugin.<name>.<tool>`, a worker target is a zero-arg Agent factory joined to
  the registry with its capabilities. **Fail-closed trust (same DNA as MCP #38):
  plugins execute code at import, so only names in `AURA_PLUGINS_ALLOW` load —
  everything else is discovered, reported, and refused.** `aura plugins` lists
  discovered plugins with trust status; `aura plugin-install <path|git-url>`
  copies/clones one in but **never allow-lists it** (enabling is a deliberate
  operator edit, printed as a reminder). A broken plugin is reported with its
  error, never a crashed loader; `build_supervisor` folds allow-listed plugin
  workers/tools in (no-op when the allow-list is empty — default behavior
  unchanged). **Bounded honestly:** evaluator/memory-backend plugin kinds,
  version pinning and dependency resolution are deferred to the marketplace
  sprint (#48) — this ships manifest + loader + trust + CLI for tools/workers.
  Gate: **276 → 284 tests**. *(Sprint #47 — done.)*
- ✅ **Plugin marketplace & registry** — `aura/plugins/registry.py`: a plugin
  **index file** (`AURA_PLUGIN_REGISTRY`; a local JSON catalog by default,
  URL-pointable for a shared read-only one). A *truly hosted* service would
  break the keyless/air-gap identity (Law 3), so the honest form is a
  self-hosted index that stays offline-testable. `search_registry(q)` substring-
  matches name/description; `publish_plugin(dir)` upserts an entry from the
  plugin's manifest + a deterministic **content sha256** (sorted rel-path +
  bytes) into a LOCAL index (pushing to a remote host is operator infra —
  refused with a clear error, not faked); `install_from_registry(name)` resolves
  name→source, installs via #47, then **verifies the installed content hash
  against the registry sha256 and fails closed on mismatch** (tamper-after-
  publish test-proven). **Integrity ≠ trust:** a verified plugin still needs
  `AURA_PLUGINS_ALLOW` to load — the #47 posture stands. CLI: `aura
  plugin-search`, `aura plugin-publish`, and `aura plugin-install` now also
  accepts a bare registry NAME (verified install). Missing/corrupt/unreachable
  index degrades to an empty catalog, never a crash. Keyless (local index, no
  network in the gate); no new dependency. Gate: **284 → 293 tests**.
  *(Sprint #48 — done.)*
- ✅ **A2A production protocol** — `aura/protocols/a2a.py`: the A2A stub becomes a
  real **Agent2Agent v1.0** implementation. Law-2 verified in-session
  (2026-07-05): A2A is now **Linux Foundation-hosted** (Apache-2.0, donated by
  Google April 2025), **v1.0 stable since April 2026** —
  [a2aproject/A2A](https://github.com/a2aproject/A2A),
  [a2a-protocol.org](https://a2a-protocol.org/latest/). What shipped, per the
  verified wire spec: a spec-shaped **AgentCard** at
  `GET /.well-known/agent-card.json` (protocolVersion 1.0, skills, capabilities
  that honestly say `streaming: false` — sync-only, no overpromising); a
  **JSON-RPC 2.0 endpoint** `POST /a2a` handling `message/send` (text parts →
  the tenant's copilot → a spec-shaped Task with `completed|failed` state +
  artifacts) and `tasks/get` (tenant-scoped in-memory task store — cross-tenant
  lookups are "not found"), with proper -32600/-32601/-32602 error codes;
  answerer exceptions become a `failed` task, never a 500. Outbound:
  `A2AClient` delegates `message/send` to a remote A2A agent and parses
  artifact text — **fail-closed behind `AURA_A2A_ALLOW`** (remote agents are
  untrusted by default; the #38 MCP posture applied to A2A). Inbound rides the
  same principal floor + injection screen as chat; the whole surface stays
  behind `AURA_A2A=enabled` (404 otherwise) and the pre-v1 `/a2a/task` stub
  endpoint remains for back-compat. Keyless (fake answerers/transports; stdlib
  urllib, no new dependency). Honest scope: task lifecycle here is the
  synchronous subset (`submitted → completed|failed`); streaming, push
  notifications, `input_required` negotiation, and remote-card discovery are
  **Sprint #50's card, not silently absorbed**. Gate: **293 → 308 tests**.
  *(Sprint #49 — done.)*
- ✅ **A2A discovery & negotiation** — `aura/protocols/a2a_discovery.py`. Law-2
  re-verified in-session: A2A v1.0 discovery is the **well-known URI**
  `/.well-known/agent-card.json` on the agent's origin (RFC 8615); skills carry
  `inputModes`/`outputModes` + tags; v1.0 "negotiation" is **card-driven
  modality matching**, not a wire handshake (the optional authenticated
  *extended card* is noted, not implemented) —
  [spec](https://a2a-protocol.org/latest/specification/),
  [agent-discovery topic](https://a2a-protocol.org/latest/topics/agent-discovery/).
  Shipped: `fetch_agent_card(url)` pulls + shape-validates a peer's card,
  **fail-closed on `AURA_A2A_ALLOW`** (discovery never widens trust — the
  allow-list decides *who* is a peer; cards only describe what trusted peers
  can do); `PeerDirectory.refresh()/find(q)` builds a capability index across
  all allow-listed peers (skill id/name/tag/description match; an unreachable
  or malformed peer is recorded as a problem, never a crash);
  `negotiate(card, input_mode, output_mode, skill_query)` decides serve-ability
  with per-skill modes overriding card defaults, returning a reasoned decision;
  `delegate_verified(url, text)` runs the full trust-aware loop — fetch card →
  negotiate (refuses before sending anything) → delegate via #49's `A2AClient`
  → **verify the result**: lifecycle state `completed`, non-empty artifact
  text, and the remote answer is screened with the same **injection scan as
  user input** (a peer's output is untrusted content entering our context;
  flags are surfaced to the caller, delivery isn't silently blocked). Keyless
  (fake card/RPC transports; no new dependency). Advertisement side was #49's
  card. Gate: **308 → 322 tests**. *(Sprint #50 — done.)*
- ✅ **MCP server mode** — `aura/protocols/mcp_server.py`: AURA **is** an MCP
  server; with the #38 client bridge, MCP is now bidirectional. Law-2 verified
  in-session (2026-07-06): the latest **stable** MCP revision is **2025-11-25**
  ([spec](https://modelcontextprotocol.io/specification/2025-11-25)); a
  2026-07-28 **release candidate** exists that goes stateless and drops
  `initialize` ([RC announcement](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/))
  — an RC is not a build target, so this implements the stable revision and
  can adopt the stateless core when it ships final. Shipped: a pure JSON-RPC
  2.0 dispatcher (`AuraMCPServer.handle`) — `initialize` (negotiates
  2025-11-25, declares tools capability), `ping`, `tools/list`, `tools/call`,
  notifications correctly ack-less, -32600/-32601/-32602 on protocol misuse;
  per spec, a tool's *execution* failure is an in-result `isError: true`,
  never a protocol error. `standard_tools(copilot)` exposes `knowledge_query`
  (grounded Q&A); **ingest/plan-execution deliberately stay behind the HTTP
  API's admin roles** — an MCP caller is viewer-grade, and the stdio transport
  can't gate mutation (honest narrowing of the card's "knowledge query,
  ingest, plan execution"). Trust posture inverted for server mode: the
  *caller* is untrusted — string arguments are screened with the same
  injection scan as user input, flags surfaced in `_meta`, not silently
  blocked. `serve_stdio()` (injectable streams, offline-tested) +
  `python -m aura.protocols.mcp_server` entrypoint — explicit opt-in, nothing
  enables it implicitly. Bidirectionality is test-proven: our own #38 client
  bridge consumes our own server through a loopback transport under a real
  trust file. No new dependency (the `mcp` SDK stays optional, client-side
  only). Gate: **322 → 334 tests**. *(Sprint #51 — done.)*
- ✅ **Webhook & event system** — `aura/events/webhooks.py`: configurable
  webhooks for lifecycle events (Slack/Teams/PagerDuty/custom dashboards
  without polling). Commodity tech — no frontier claim, **no new dependency**
  (stdlib urllib + hmac). Configuration *is* the switch: `AURA_WEBHOOKS`
  (inline JSON or a file path) lists `{url, events: [fnmatch globs], secret?}`;
  unset/corrupt config degrades to no hooks, never a crash. Delivery:
  canonical-JSON body `{event, timestamp, payload}`, and with a secret an
  **HMAC-SHA256 `X-Aura-Signature`** over the exact body bytes (verified
  receiver-side in tests, the GitHub-webhook pattern). **Emitting never raises
  and never breaks the host path** — failed deliveries come back in a report,
  not as exceptions. Payloads are deliberately **minimal** (ids/status/tenant;
  never answers, arguments, or documents — webhook bodies leave the box, so
  they carry references, not knowledge). Seams wired: background jobs
  (`job.submitted` / `job.completed` / `job.failed` with typed error names)
  and approvals (`approval.requested` / `approval.decided`); a decided
  approval only emits once. Honest scope: delivery is synchronous with a 5s
  timeout (a slow receiver briefly delays the emitting thread — an async
  dispatcher is a later sprint if it ever matters); step-level planner events
  deferred, stated. Gate: **334 → 343 tests**. *(Sprint #52 — done.)*
- ✅ **SDK & client libraries** — `aura/sdk/` (not `aura-sdk/`): a lightweight,
  **zero-dependency** (stdlib urllib) typed Python client for the serving API,
  shipped *inside* the `aura` package so `pip install aura` already includes
  it. Surface: `healthz/health/whoami`, `ingest` + `ingest_async`/`job`/
  `wait_job` (polling), `ontology`, `chat`/`chat_stream`/`chat_events` (SSE
  parsed into deltas; server-reported errors raise), `metrics`/`cost_report`,
  `approvals`/`decide_approval`, and `a2a_send` (the #49 JSON-RPC endpoint,
  artifacts parsed to text). All HTTP lives behind a tiny `Transport`
  interface (request + stream); tests inject an in-process transport wrapping
  FastAPI's `TestClient`, so the SDK is proven against the **real server
  app**, keylessly. The stream contract carries the status code so **an HTTP
  error raises `AuraAPIError` instead of reading as an empty stream**
  (regression-tested — found live when explicit JSON `null`s tripped a 422
  the SSE path swallowed; the client now only sends fields the caller set).
  Auth = the server's `X-Aura-Key` header; role errors surface with the
  server's own detail string. Honest deltas from the card: **publishing to
  PyPI/npm is release infrastructure, not code** (deferred, same judgment as
  #48's "hosted registry"); the **TypeScript client is deferred** — this
  repo's gate can't honestly test TS, and shipping untested code as "an SDK"
  would be theater; async variant deferred until someone needs it (the sync
  client streams fine). Gate: **343 → 353 tests**. *(Sprint #53 — done.)*
- ✅ **Wave 4 release notes + `v1.1.0` tag** — CHANGELOG entry for the
  ecosystem wave (Sprints #43–#53 + debt ledger #45b–#48b): **223 → 353
  tests**, 2 → 5 LLM backends, protocol surface grown from "MCP client" to
  A2A v1.0 (server + trust-aware client + discovery), bidirectional MCP,
  signed webhooks, and a zero-dep Python SDK. Version `1.0.0 → 1.1.0`
  (pyproject reads `aura.__version__`; the A2A card and MCP `serverInfo`
  advertise it live). Still open and stated: operator's optimizer receipt
  (BYOK), #34b Langfuse, TS SDK/publishing infra. *(Sprint #54 — done.
  Wave 4 closes.)*

**Wave 4 exit criteria:** `v1.1.0` tagged. AURA runs on ≥3 LLM backends (Anthropic,
OpenAI, local). Plugin system functional with ≥1 community plugin. A2A interop
demonstrated. MCP bidirectional.

---

## Deferred-work ledger (named at the v1.0.0 release; planned here, not hidden)
Letter-suffixed sprint numbers (repo precedent: #34b) so Waves 4–5 keep their
numbering. Ordered by leverage; none block Wave 4.

- ✅ **#45b — flip-gate CI history cache** *(done — see the Wave 4 entry above;
  debt item #5).*
- ✅ **#46b — tenant isolation, the remainder** *(debt item #4)* — the four
  surfaces #35 deferred are now tenant-scoped. **Traces:** tenant column on
  spans/traces (ALTER-TABLE migration for pre-#46b dbs) written from a new
  request-scoped **contextvar** (`aura/tracing/tenant.py`) that server handlers
  set from the Principal — including explicitly inside SSE generators and the
  `ingest_async` worker closure, which run on other threads; `/v1/traces/stream`
  always scopes to the caller's tenant, the dashboard stays the operator's
  unfiltered view. **Approvals:** tenant column (migrated); `request()` defaults
  to the contextvar so guard-created approvals (quarantine, merge conflicts)
  inherit the caller's tenant; `/v1/approvals` lists scoped; cross-tenant decide
  is **404, no existence leak**. **Memory namespaces:** `KnowledgeBase` gains a
  `namespace` override; `run_ingest(namespace_prefix=…)` yields
  `tenant:domain` so two tenants ingesting "retail" share nothing (memories AND
  graph — the graph name follows the namespace); default stays the bare
  ontology name. **Inbox:** `AURA_INBOX/tenants/<t>` is used when it exists,
  else the shared inbox (operator-managed staging; sharing a source tree is
  often intentional — memories/graph stay isolated regardless, stated in code).
  Gate: **244 → 252 tests** (suite ×2; +9 isolation tests incl. contextvar
  reset-no-leak, per-tenant SSE, cross-tenant 404). *(Sprint #46b — done.)*
- ✅ **#47b — candidate proposers (DSPy; TextGrad-pattern)** *(debt item #3)* —
  `aura/learning/proposers.py`: `run_improvement_cycle` gains an injectable
  `proposer(current, n) -> candidates` (resolved via `AURA_PROPOSER`; default
  `rewrite` = the original behavior, extracted). New: `textual_gradient`
  (critique→improve — the TextGrad *pattern*, dependency-free) and `dspy`
  (real adapter behind the new `[dspy]` extra; lazy import, helpful install
  error, per-call failures shrink the candidate list instead of crashing the
  loop). **Law-2 verification (2026-07-05):** DSPy v3.2.x active, MIT
  (stanfordnlp/dspy) → adapter shipped; **`textgrad`'s last PyPI release is
  0.1.8 (June 2024) — two years stale, fails the maintenance filter, so only
  its pattern ships, dependency-free** (stated in code). The gate is untouched:
  test-proven that a fancier proposer cannot loosen accept-only-if-improves;
  unknown proposer names fall back to `rewrite` (a typo must never kill the
  loop). Gate: **252 → 259 tests** (suite ×2). Found en route: the prompt store
  persists across pytest runs — loop tests now isolate it (`clear()`), same
  trap family as the #27 stub-pollution lesson. *(Sprint #47b — done.)*
- ✅ **#48b — T1 speculative planning** *(debt item #2)* — behind
  `AURA_SPECULATE=enabled` the DAG planner runs branch prediction: each pass
  launches (concurrently, `copy_context()` per step — the #31 pattern, capped
  at `AURA_SPECULATE_MAX`=2) the steps whose UNMET deps are all in that pass's
  ready set, with a context built from completed ancestors only ("some ancestor
  results are still in flight" stated in the prompt). The harvest commits a
  guess only when every dep landed OK; a mispredict is **discarded and counted**
  (`out["speculation"] = {speculated, wasted}`; speculative spans carry
  `meta.speculative=true` so their `cost_usd` is attributable — the card's
  cost-honesty requirement). The sequential loop skips in-flight speculative
  steps (the harvest owns them) — the first draft double-ran the speculated
  step when its dep committed mid-pass; caught by the suite. Flag off = the
  sequential planner untouched (test-proven, no stats key). Concurrency proven
  by barrier, not wall-clock (the #31 lesson). Gate: **259 → 266 tests**
  (suite ×2). *(Sprint #48b — done. Closes T1 — the last open frontier item.)*
- ✅ **TH-track: 3D theater Live tab** *(debt item #6)* — the R3F app now
  consumes the #33 typed SSE span feed: a **⚡ Live** HUD toggle streams the
  newest trace as a continuously-recompiled episode (span upsert by id gives
  start→complete lifecycle; follow-the-tail; replay clock disabled while
  live; capped-backoff reconnect with a surviving rowid cursor; feed loss
  degrades to replay). **Verified live in a real browser** (dashboard
  preview): toggling ⚡ Live, then writing a traced run to the store, made
  `⚡ live · <trace-id>` appear with correct beats and tail-follow — zero
  console errors; bundle rebuilt and committed. Details in
  [AURA_THEATER_ROADMAP.md](AURA_THEATER_ROADMAP.md) (TH-3's live half;
  the saved-trace replay-import half stays open). *(Done — 2026-07-06.)*

**Operator actions (not sprints — they need you, deliberately):**
1. *(debt item #1)* **The optimizer receipt** — one command, BYOK token spend:
   `python -m aura.evaluation.optimizer_proof` → commit the generated
   `OPTIMIZER_PROOF.md`. Upgrades #41 from 🟡 to ✅ and completes the last
   v1.0.0 release criterion.
2. *(debt item #7)* **Repo visibility** — flipping the repo public activates
   the "publicly verifiable" story (live CI badge, golden scoreboard, measured
   release notes). Strategic decision, zero engineering.

---

## Wave 5 — Enterprise & scale (Sprints #55–#68)
From single-process to production-grade distributed system. The wave that makes AURA
enterprise-ready.

- ✅ **Celery/Ray distributed backend** — `aura/distributed/worker.py`, behind
  `AURA_DISTRIBUTED=local|celery|ray`. Law-2 verified in-session (2026-07-06):
  Celery latest release 2026-03, BSD-3-Clause
  ([celery/celery](https://github.com/celery/celery)); Ray 2.56, 2026-06,
  Apache-2.0 ([ray-project/ray](https://github.com/ray-project/ray)) — both
  maintained, CPU-viable, free OSS; **optional extras `[celery]`/`[ray]`,
  never hard deps, never in the gate**. The MCP-bridge pattern applied to
  execution: a JSON-serializable **task envelope** (`task_payload`), a
  module-level remote entrypoint (`execute_task_payload` — importable by
  name, which is what Celery tasks and Ray remotes require; resolves the
  node's registry via `AURA_WORKER_PROVIDER="module:attr"` or
  `set_worker_provider`, restores the tenant contextvar, returns the same
  contract-shaped result as the in-process fan-out, never raises), and an
  `ExecutionBackend` seam (`local` = threads over the payload path;
  celery/ray adapters lazy-import with a helpful install error —
  naturally test-proven since the keyless venv lacks both). The
  `ParallelSupervisor` consults the seam: **env unset ⇒ the classic
  in-process path runs byte-identical**; set ⇒ payloads fan out through the
  backend and merge through the same typed contracts. The fake-remote test
  round-trips every payload AND result through a JSON wire — the exact
  property brokers need. Unknown backend names are a loud `ValueError`,
  never a silent fallback. Honest scope: a remote node answers from its own
  ingested store (shipping the KB is deployment, stated); remote spans land
  on the remote node — stitching them is #56's card. Gate: **353 → 364
  tests**. *(Sprint #55 — done.)*
- ✅ **Distributed trace aggregation** — `aura/distributed/trace_agg.py`,
  closing #55's honest note (remote spans used to land on the remote node —
  a distributed run was a supervisor span with a hole in it). **Capture:**
  `run_traced(name, fn)` runs the work inside a fresh trace on the worker
  node and attaches a JSON-serializable `{"trace", "spans"}` snapshot to the
  result (`execute_task_payload` now ships it under `remote_trace` — same
  envelope, one more key); capture failure never sinks the answer.
  **Merge:** `merge_remote_trace(remote, parent_span_id, node)` re-homes
  every shipped span onto the *current* trace — span ids preserved (uuid;
  re-merging a replayed result REPLACEs, never duplicates — test-proven),
  remote roots hang under the supervisor's fan-out span, nested structure
  survives intact, and every span is stamped `meta.remote=true` + node label
  so the dashboard/Theater can tell, while otherwise rendering the run **as
  if it were local** (the card's promise, asserted end-to-end through a JSON
  wire). Wired into `ParallelSupervisor`'s distributed branch only;
  in-process runs untouched. Telemetry is best-effort by design: a malformed
  shipment or missing local trace merges 0, never raises. Gate: **364 → 370
  tests**. *(Sprint #56 — done.)*
- ✅ **Persistent / durable workflows** — `aura/workflows/engine.py`, behind
  `AURA_WORKFLOWS=enabled` (the engine refuses to run otherwise). Zero new
  dependency: durable state is one SQLite file (`AURA_WORKFLOWS_DB`, default
  `.aura_workflows.db`, gitignored). The durability split, stated:
  **definitions are code** (steps are *named* functions in a registry;
  workflows are named sequences of `StepSpec`s — names, not closures, because
  a pickled closure is a lie after a deploy; boot code re-registers them) and
  **state is data** (every step's status/attempts/result persists per
  transition). `resume(wf_id)` after a crash **skips completed steps without
  re-running their side effects** (test-proven: a fresh engine on the same db
  completes the flow with the first step's side-effect counter still at 1,
  its result reloaded from disk); a failed step gets a fresh chance on
  resume. Per-step policy: `max_attempts` + `backoff_s` (injectable sleep —
  tests never wall-clock) and `timeout_s` (worker thread; on timeout the
  thread may still run — its result is discarded and the pool abandoned
  without waiting, so a hung step can't hang the engine — Python can't kill
  threads, honest note). Step results must round-trip JSON — a
  non-serializable return is a **typed, non-retryable** `WorkflowStateError`
  ("durable" state that can't round-trip isn't durable, and a bug is not
  retried). Tenant-tagged instances; `status()`/`list()` read models.
  Honest scope: **sequential v1** — branching, loops and human checkpoints
  are #58's DSL on top of this engine, not silently absorbed. Gate: **370 →
  379 tests**. *(Sprint #57 — done.)*
- ✅ **Workflow DSL** — `aura/workflows/dsl.py`: a declarative **JSON/dict
  (YAML when PyYAML is present — the plugin-manifest precedent)** description
  compiled onto the #57 durable engine via `compile_workflow(engine, spec)`.
  Nodes: `{"step": …}` (policy fields `max_attempts`/`backoff_s`/`timeout_s`
  pass straight through to the engine — test-proven), `{"if"/"then"/"else"}`
  branching and `{"repeat"/"until", "max_iterations"}` loops (each control
  block is **one durable unit** running its sub-steps inline — stated
  granularity, not hidden; runaway loops fail loudly at the iteration cap),
  and `{"checkpoint": name}` **human checkpoints that ride the existing
  approvals queue** — one review surface, not a second inbox. A checkpoint
  compiles to two durable steps: *request* (files the approval, durably
  records its id — resume reuses the **same** approval, test-proven) and
  *gate* (pending ⇒ the workflow pauses as a failed step saying "awaiting
  approval #N"; after the human decides, `resume(wf_id)` continues on
  approve and keeps refusing on reject — publish never runs, test-proven).
  Conditions are a deliberately tiny, **never-`eval()`** language
  (`path [op literal]`, dotted paths rooted at `results`/`input`, JSON
  literals, bare-path truthiness; missing paths are falsy, not crashes —
  a workflow file is operator data, not code). Bad DSL fails loudly at
  compile time. Honest deltas from the card: "Python DSL" is the engine's
  own `StepSpec` API (already shipped in #57); **scheduled triggers stay
  out** — cron belongs to the operator's scheduler, stated. Found by CI,
  not locally: the YAML test assumed PyYAML (incidentally present in the
  dev venv, absent in the gate) — split into a presence-gated test
  (`importorskip`) and an absence test (import-blocked), so **both sides of
  the optional dependency are asserted everywhere**. Gate: **379 → 398
  tests**. *(Sprint #58 — done.)*
- ✅ **Workflow UI** — the dashboard gains a **⛓ WORKFLOWS** panel over the
  #57/#58 durable state. **Monitor:** instance list with status pills +
  per-step drill-down (attempts, durable results, errors), auto-refreshing
  every 4s. **Manual intervention:** the pending-checkpoint queue
  (`workflow:*` approvals) with ✓ approve / ✗ reject — the #58 checkpoint
  mechanism *is* the intervention point, one review surface. **Builder,
  text-first:** a DSL editor with a real linter — new `lint_workflow()` in
  `dsl.py` validates structure/grammar/conditions **without requiring the
  step registry** (the runner process owns registration, so unknown steps
  are warnings, not errors — the honest split for an editor process).
  Dashboard API: `GET /api/workflows[/{id}]`, `GET/POST
  /api/workflows/approvals[...]/decide`, `POST /api/workflows/lint` — all
  behind the existing dashboard auth middleware; the READ model needs no
  step registry, and run/resume verbs deliberately stay with the runner
  (stated in code). **Verified live in a real browser:** a seeded workflow
  paused at its `ship_it` checkpoint appeared in the panel, drill-down
  showed the awaiting-approval gate, clicking ✓ approve cleared the queue,
  the runner's `resume()` completed the flow, and the panel showed
  `completed` — plus the lint editor rendering problems/warnings; zero
  console errors. Honest delta from the card: **drag-and-drop composition
  is deferred** — a validated text editor ships first because a node canvas
  without a runner-side registry would be theater. Gate: **468 → 473
  tests**. *(Sprint #59 — done. Wave 5 is now fully closed, no carry-overs
  in the sprint queue.)*
- ✅ **Full audit trail** — `aura/compliance/audit.py`, behind
  `AURA_AUDIT=enabled` (`AURA_AUDIT_FILE`, default `.aura_audit.jsonl`,
  gitignored). Append-only JSONL with a **sha256 hash chain** (each record
  embeds the previous record's hash; stdlib crypto, zero dependency): any
  edit, deletion, or insertion breaks `verify_chain()` at the **exact
  record** — all three tamper classes test-proven with distinct reasons.
  Honesty stated in the module: **tamper-evident ≠ tamper-proof** — an
  attacker with write access can rewrite the whole chain; `tail_hash()`
  exists precisely so operators can anchor the head externally (a #52
  webhook, another machine). **Failure posture is a stated choice, not a
  silent default:** lenient mode logs the failed write and lets the action
  proceed (availability); `AURA_AUDIT_STRICT=enabled` raises — if it can't
  be audited, it doesn't happen. Wired seams: **every tool invocation** in
  the ToolAgent loop with outcome `ok|blocked|error` (the refusals are the
  interesting part — a blocked `drop_table` lands in the trail,
  test-proven), **approval requests/decisions** (decisions carry
  `actor: human`), and **long-term memory recalls** (namespace + truncated
  query). `export_audit(jsonl|csv)` **refuses to export a broken chain**
  (an export of tampered history would launder it); chain survives process
  restarts (tail re-read, test-proven); `python -m aura.compliance.audit`
  verifies or exports from the CLI. Gate: **406 → 416 tests**.
  *(Sprint #60 — done.)*
- ✅ **Data residency & encryption** — `aura/compliance/residency.py`, both
  controls opt-in. **Residency:** `AURA_RESIDENCY` (inline JSON or file)
  declares `allowed_roots`; `verify_residency()` resolves every place AURA
  actually persists — knowledge store, trace db, workflows db, audit trail,
  inbox, plugins dir — and names each violation; a corrupt config is itself
  a violation (unverifiable ≠ compliant). Wired into the serving API's
  lifespan: violations print at boot; `AURA_RESIDENCY_STRICT=enabled`
  **refuses to serve** — a region promise you can't verify is marketing,
  not compliance. Honest scope: AURA's configured stores only; OS/swap/
  backups are the operator's domain. **Encryption at rest:**
  `AURA_ENCRYPTION_KEY` (Fernet; `python -m aura.compliance.residency
  --genkey`) encrypts AURA's **JSON file stores** via magic-prefixed
  `write_encrypted_json`/`read_json_flex` — pre-key plaintext keeps reading
  and migrates on next write (test-proven); a wrong key refuses to guess
  (typed error). Wired: the prompt store. Law-2 verified (2026-07-06):
  pyca/cryptography v49.0.0 (2026-06), Apache-2.0 OR BSD-3-Clause, very
  active — optional extra `[crypto]`, added to CI's gate install so the
  presence tests run there too (the #25/#58 lesson: **both sides of an
  optional dependency get tested** — key-without-library **fails closed
  with nothing written**, never silent plaintext, import-block-proven).
  Honest deferral, named: the SQLite stores (traces, memory, approvals)
  are NOT encrypted here — that's SQLCipher territory, a different and
  heavier integration. Gate: **416 → 428 tests**. *(Sprint #61 — done.)*
- ✅ **SOC2-supporting logging** — `aura/compliance/soc2.py`, behind
  `AURA_SOC2_LOGGING=enabled` (wired at serving-API boot). Naming honesty
  reshapes the card: **SOC2 is an organizational audit, not a code
  property** — no module "meets SOC2 Type II"; what code CAN provide are
  the logging *controls* a Type II audit asks about, and that is exactly
  the shipped surface. **Structured:** one JSON object per line (epoch +
  ISO ts, level, logger, message, tenant from the request contextvar,
  first-class `extra` fields, exceptions captured). **Rotation:** stdlib
  `RotatingFileHandler` (`AURA_LOG_MAX_BYTES` × `AURA_LOG_BACKUPS`) — logs
  bounded by construction, test-proven. **Retention:** `purge_expired()`
  enforces `AURA_LOG_RETENTION_DAYS` on rotated files (never the active
  log), tested by back-dating mtimes, never by sleeping — returns what it
  removed (retention that isn't enforced is a wish, not a policy).
  **Access control:** chmod 0600 — real on POSIX, advisory on Windows
  (ACLs are the real control there — stated). **Export:**
  `export_range(start, end)` filters every rotation into one auditor-ready
  JSONL with a record count. `configure_logging()` is idempotent (no
  handler stacking, test-proven); zero new dependency (stdlib `logging`).
  Gate: **428 → 437 tests**. *(Sprint #62 — done.)*
- ✅ **Admin surface (keys + usage)** — `aura/admin/` behind
  `AURA_ADMIN_KEYS=enabled`. **Key lifecycle** (`keys.py`, SQLite at
  `AURA_ADMIN_DB`, gitignored): `create_key` returns the secret **exactly
  once** — the DB stores only its sha256 (test asserts the secret appears
  nowhere at rest, including the raw db file); revoke tombstones (never
  deletes — the audit story needs the record); `last_used_at` touch on
  every resolve; creates/revokes land in the **#60 audit trail**. **Auth
  integration is additive and ordered:** `AURA_PRINCIPALS` env map wins
  (operator override, test-proven), then the key store, then legacy
  `AURA_API_KEY`, then open-dev — existing deployments byte-identical until
  the flag is set; revoked keys 401. **Role hierarchy** gains `org-admin`
  (viewer < user < org-admin < admin): org-admins list/mint/revoke keys and
  read usage **for their own tenant only** — cross-tenant revoke answers
  404 exactly like a missing key (no existence leak, endpoint-tested), and
  org-admins cannot mint admin keys. **Usage analytics** (`usage.py`): a
  read model over existing telemetry — llm spans aggregated per tenant and
  per model (requests, tokens, `cost_usd`, cache hits); no new writes
  (full-scan aggregate, right for single-node volumes — rollups when
  someone measures the need). Endpoints: `GET/POST /v1/admin/keys`,
  `POST /v1/admin/keys/{id}/revoke`, `GET /v1/admin/usage` (floor:
  org-admin). Honest deltas: the "dashboard" *UI* is a frontend session
  like #59 — this ships the API the UI will call; **org-level quotas are
  #64's card**, not absorbed. Gate: **437 → 446 tests**.
  *(Sprint #63 — done.)*
- ✅ **Quota & rate limiting** — `aura/admin/quotas.py`. BYOK means the
  operator's key pays for every token — quotas are how a shared instance
  stops one tenant spending everyone's budget (Law 6, mechanized). Config
  *is* the switch: `AURA_QUOTAS` (inline JSON or file) with per-tenant (and
  `"*"` default) `tokens_per_day`, `spend_usd_per_day`,
  `requests_per_minute`, and per-model `model_spend_usd_per_day`; unset ⇒
  **no-op, not even a db file created** (module-level `check`/`note` gate
  on config before the manager exists). **Hard limits block before the
  provider is contacted** — `QuotaExceeded` raises pre-call and the stub
  script stays unconsumed (test-proven). **Soft limits warn**: crossing
  `warn_at` (default 80%) emits a `quota.warning` #52 webhook — **once per
  tenant/limit/day**, not a firehose (durable dedup table). **Day budgets
  are durable** (SQLite beside the admin key store — bouncing the process
  must not reset a budget, test-proven); the per-minute window is
  process-local (stated: multi-process deployments rate-limit per process;
  the budgets that cost money stay global). Wired at the **LLM wrapper's
  choke points** (`_create` + `stream`), covering every backend and call
  path; usage recorded from the provider's own numbers. Honest delta:
  per-*user* limits are per-*tenant* v1 (principals map to tenants;
  per-key granularity when someone needs it). Found en route: re-exporting
  the `quotas()` accessor from `aura.admin` shadowed the
  `aura.admin.quotas` *module* — accessor stays module-level only.
  Gate: **446 → 455 tests**. *(Sprint #64 — done.)*
- ✅ **Multi-tenant isolation hardening** — `aura/admin/isolation.py`.
  Honesty reshaped the card before code did: "impossible by construction" is
  not achievable in one process with one master key — what IS achievable,
  and shipped, is **per-tenant key separation**: **HKDF-SHA256** over the
  #61 master (`info = "aura:tenant:<t>"`) derives a distinct Fernet key per
  tenant — one operator secret, unlimited tenants, no key database to leak;
  derivation is deterministic (restart-safe) and one-way. Tenant A's key
  **provably cannot open** tenant B's artifacts (typed `IsolationError`,
  never a guess, never data — test-proven), so a bug that hands code the
  wrong tenant's store gets ciphertext. Wired end-to-end: the per-tenant
  reload payload (`<store>/ontology.json`) is written/read through
  `write_tenant_json`/`read_tenant_json` — tenant identity flows from the
  #46b namespace prefix; `run_ingest(namespace_prefix="acme:")` persists
  under acme's key and `load_persisted(store, tenant="zeta")` is refused
  (end-to-end test). The #61 posture holds: no master key ⇒ plaintext,
  byte-identical; pre-key files keep reading and migrate on next write;
  key-without-library fails closed. `jsonld`/`report.md` stay plaintext
  (operator-facing artifacts, not the boot payload — stated). Honest scope:
  runtime isolation (memories/graph/traces in shared SQLite) remains #46b
  namespace/policy isolation — SQLCipher-and-beyond, deferred and named.
  Gate: **455 → 461 tests**. *(Sprint #65 — done.)*
- ✅ **Kubernetes Helm chart** — `deploy/helm/aura`: two deployments
  (dashboard :8000 — the Docker image's default CMD — and the serving API
  :8001 via command override), Services, optional Ingress (dashboard at `/`,
  API at `/v1`), a shared PVC, liveness/readiness probes on the *real*
  endpoints (`/api/metrics`, `/healthz` — asserted by test), and **keyless
  by default** (`env.AURA_LLM: stub` — the chart deploys a working demo
  with zero secrets). **Secrets are BYOK-shaped:** the chart *references*
  `existingSecret`, it never creates one — "kind: Secret" appearing nowhere
  in the chart is a test. Honest limits stated in values/NOTES rather than
  papered over: SQLite on one RWO volume is a **single-node story**, so
  **HPA ships but defaults off** — horizontally scaling the server is not
  meaningful until the trace store is Postgres; Prometheus/Grafana
  integration deferred (the OTLP exporter from #34 is the metrics path).
  Validation: a new **CI `helm` job** runs `helm lint` plus three
  `helm template` renders (defaults / everything-on / minimal) — the chart
  must render before it may claim to deploy; Python-side tests assert
  structure, keyless defaults, and no baked secrets. "Deploys to a real
  cluster" is operator-verified by nature — the wave exit note says so.
  Gate: **461 → 466 tests**. *(Sprint #66 — done.)*
- ✅ **Performance benchmarks** — `benchmarks/bench.py`. **Honesty defines
  the methodology:** the suite runs keyless (`AURA_LLM=stub`), so it measures
  **AURA's framework overhead** — tracing, contracts, retrieval, memory,
  orchestration plumbing — with **model latency excluded by design**. That is
  the number a framework can honestly publish: reproducible on any machine,
  no key, no network, no GPU; live-model latency belongs to the provider.
  Five suites (mean/p50/p95 ms): `ingest`, `chat_answer` (question varies per
  rep so the semantic cache can't turn it into a cache benchmark),
  `retrieval`, `memory_recall`, `span_write`. Published:
  [benchmarks/RESULTS.md](benchmarks/RESULTS.md) (dated, machine-labeled,
  with the reproduce command) + a README pointer. Tests assert **structure
  and positivity, never the clock** (the #31 lesson — no latency thresholds
  in CI). Honest deltas from the card: distributed-mode load tests need a
  real broker/cluster (operator infra — the #55 fake-wire proves the
  contract, not throughput); "cost" benchmarks are the #63 usage analytics'
  job, not a stopwatch's. Gate: **466 → 468 tests**. *(Sprint #67 — done.)*
- ✅ **Wave 5 release notes + `v1.2.0` tag** — CHANGELOG entry for the
  enterprise-&-scale wave (Sprints #55–#67 + the long-deferred #34b):
  **353 → 468 tests**, a 7th CI job (helm), and four new layers —
  distributed execution with stitched traces, durable declarative workflows
  with human checkpoints, compliance (audit chain / residency / at-rest
  crypto / SOC2-supporting logs), and admin (key lifecycle / org-admin /
  usage / quotas / tenant crypto). Version `1.1.0 → 1.2.0` (package + Helm
  `appVersion`). **Exit criteria honestly assessed in the notes**: durable
  restarts, audit/logging, and the admin API are test-proven ✅; "runs on
  ≥2 nodes" and "Helm deploys to a real cluster" are operator-verified by
  nature (the chart lints/renders in CI; the fake-wire test proves the
  contract, not a datacenter). Carried forward, named: #59 workflow UI,
  TH-track Live tab, BYOK optimizer receipt, TS SDK/publishing, MCP
  stateless core (RC), SQLCipher. *(Sprint #68 — done. Wave 5 closes.)*

**Wave 5 exit criteria:** `v1.2.0` tagged. AURA runs distributed on ≥2 nodes.
Durable workflows survive process restarts. Audit trail + SOC2 logging functional.
Admin dashboard operational. Helm chart deploys to a real cluster.

---

## Wave 6 — Intelligence frontier (Sprints #69–#88)
The research wave. AURA moves from a framework that *runs* agents to a framework
where agents *evolve*.

- ✅ **Tool discovery & synthesis** — `aura/tools/synthesizer.py`, behind
  `AURA_TOOL_SYNTHESIS=enabled` (Wave 6 opens — the first "agents evolve"
  sprint). Observes an agent's tool-call trail, finds the longest tool-name
  subsequence that repeats ≥ `AURA_SYNTH_MIN_REPEAT` (default 2,
  **non-overlapping** — deterministic, keyless, args ignored because that's
  what a parameterized tool abstracts), and **proposes a new composite tool**.
  The load-bearing rule: **a synthesized tool NEVER auto-activates** —
  `propose_from_trail` files the candidate in the approval queue
  (`tool:synthesize`, audited via #60); `activate_if_approved` is **fail-closed**
  (pending/rejected ⇒ not registered, test-proven), and only an approved
  proposal registers + runs the pipeline. Discovery ≠ trust — the #38/#47
  posture applied to code the framework wrote about itself. Honest scope
  stated in the module: v1 synthesizes a **composition of existing, trusted
  tools** (declarative spec, no new arbitrary code), so nothing here needs a
  sandbox yet; freeform Python-body synthesis rides #70's sandbox as a later
  increment. Already-approved patterns aren't re-proposed. Gate: **473 → 482
  tests**. *(Sprint #69 — done. Wave 6 opens.)*
- ✅ **Tool validation & sandboxing** — `aura/tools/sandbox.py`: two
  fail-closed gates so the *next* increment (freeform Python tool bodies) can
  be run without being trusted. **(1) Static AST screen** (`validate_source`)
  rejects before anything runs — imports outside an allow-list
  (`AURA_SANDBOX_IMPORTS`, default `math,json,re,statistics,datetime`), dunder
  attribute access (`__globals__`/`__class__`… — the classic escape vector),
  banned names (`eval`/`exec`/`open`/`__import__`/`compile`…), and a required
  `tool(**kwargs)` def; a syntax error is a rejection, not a crash. **(2)
  Isolated execution** (`run_sandboxed`) runs the body in a fresh
  `python -I -c` subprocess (not this interpreter — a crash/hang/`sys.exit`
  can't touch AURA) with a wall-clock timeout (`AURA_SANDBOX_TIMEOUT`, killed)
  and POSIX address-space + CPU rlimits (fork/alloc-bombs die); I/O is
  JSON-only, a non-JSON result is a failure. **Network isolation, honestly
  scoped:** the AST allow-list forbids every network-capable import
  (`socket`/`urllib`/`requests`/`http` all rejected — test-proven), the
  enforceable in-process guarantee; true namespace isolation is the
  operator's container, stated. Validation runs **before** execution
  (injected-runner test proves the body never runs on a failed gate); the
  real subprocess is integration-tested (clean tool → result, whitelisted
  import works). Nothing auto-registers — untrusted by default, always. Gate:
  **482 → 500 tests**. *(Sprint #70 — done.)*
- ✅ **Tool composition** — `aura/tools/composer.py`: turns a declarative
  pipeline spec into a **real, reusable, traced composite `Tool`** that chains
  existing registry tools — discoverable and guarded like any other tool, so
  an agent uses it without knowing it's a pipeline. Arg wiring is a tiny,
  **eval-free** template (the #58 discipline): `"$name"` resolves to an input
  kwarg or a prior step's captured `out` (missing ⇒ a clear
  `CompositionError`, never a silent None); literals pass through; a step with
  no `args` gets the shared kwargs (#69's default, generalized); the last
  step's result is the composite's return. **Fail-closed:** `build_pipeline`
  validates structure + that every referenced tool exists and registers
  **nothing** on a dangling/invalid spec (test-proven); cycles are impossible
  (steps are a straight sequence). Closes the #69 loop — `register_composition`
  is the composer hook `activate_if_approved(..., composer_register=…)` uses,
  so an approved synthesis proposal upgrades from the inline executor to a
  validated, traced composite (end-to-end test: propose → approve → activate
  via composer → registered). No code generation — every step is an
  already-trusted tool. Gate: **500 → 509 tests**. *(Sprint #71 — done. The
  tool-evolution trio #69–#71 is complete.)*
- ✅ **Cross-session pattern mining** — `aura/learning/patterns.py`: a pure,
  keyless read over the trace store that mines three structural families —
  `tool_sequences` (contiguous tool-name runs recurring across traces — the
  raw material #69 turns into composites), `plan_shapes` (collapsed span-kind
  signatures), and `failure_modes` (recurring span+error-family pairs) — above
  a min-support floor. **Privacy taken literally:** mining is
  **tenant-scoped** (`store.list_traces(tenant=…)`; cross-tenant sharing is
  #75's card, not here) and **anonymised by construction** — it reads only
  structural fields (names, kinds, counts), never inputs/outputs/values, with
  error strings folded to families (digits + quoted payloads stripped, so
  "no such id 42/99" → one family). A test asserts a rich secret-bearing
  trace leaks nothing into the report. `refresh_heuristics` emits the mined
  patterns as anonymised playbook lessons behind `AURA_PATTERNS=enabled`
  (no-op otherwise). Gate: **509 → 517 tests**. *(Sprint #72 — done.)*
- ✅ **Meta-learning: learning to learn** — `aura/learning/meta.py`: gives the
  memoryless improvement loop a memory. Records, per
  `(mutation_kind × task_kind)`, whether a proposal was accepted and the
  realized metric `delta`; `best_proposer_for(task_kind)` suggests the
  proposer with the best track record for a task shape. **Honesty rails:**
  the suggestion is **advisory** — it can pick which proposer to *try first*
  but NEVER loosens `optimize_prompt`'s accept-only-if-improves gate (a
  test drives a meta-suggested run whose candidate doesn't beat baseline and
  confirms it's still rejected); and it **abstains below `AURA_META_MIN_TRIALS`**
  (default 3 — the #37 "one pass isn't proof" discipline) and never suggests a
  zero-accept proposer. Wired into `run_improvement_cycle(task=…)` behind
  `AURA_META=enabled`; deterministic, keyless, no training — just counted
  evidence in a small JSON store. Gate: **517 → 529 tests**.
  *(Sprint #73 — done.)*
- ✅ **Curriculum learning** — `aura/learning/curriculum.py`: schedules *what
  to practise next* from real golden outcomes. `difficulty(task)` is a
  **Laplace-smoothed fail-rate** (a single lucky/unlucky run can't peg it to
  0/1; an unseen task is 0.5 — neither claimed easy nor hard); `schedule`
  orders tasks easy→hard (stable ties → deterministic); `focus(tasks, k)` is
  the adaptive part — the k hardest tasks *with enough attempts to justify the
  claim* (min-attempts floor; struggles only, difficulty > 0.5). Behind
  `AURA_CURRICULUM=enabled` for recording; ordering functions are pure.
  Honest scope: it schedules and prioritises, it doesn't itself run training
  (the operator's loop calls `record_result` after each golden run), and the
  ordering is gated by the same golden set it reads. Gate: **529 → 538
  tests**. *(Sprint #74 — done. The learning trio #72–#74 is complete.)*
- ✅ **Collective intelligence (multi-tenant)** — `aura/learning/collective.py`:
  federated pattern sharing across tenants, opt-in both directions
  (`AURA_COLLECTIVE=enabled` **and** the tenant listed in
  `AURA_COLLECTIVE_TENANTS` — no wildcard, fail-closed the #38 way).
  `contribute(tenant)` re-aggregates #72's *already-anonymised* mined patterns
  (names/kinds/counts, never inputs/outputs) into a shared JSON pool;
  contributors are stored as a **salted one-way sha256** (salt minted once per
  pool — enough to count distinct tenants and make re-contribution idempotent,
  not enough to name anyone), and a pattern stays **dark until ≥
  `AURA_COLLECTIVE_MIN_TENANTS` (k=2) distinct tenants corroborate it** (a
  pattern only one tenant ever produced *is* that tenant's behaviour — sharing
  it would leak). `adopt(tenant)` emits the corroborated patterns as
  `[collective]` playbook lessons; reciprocity-gated (won't-share ⇒
  won't-read). Test-proven: pool file never contains a tenant id or a span
  value; single-contributor patterns invisible; re-contribution replaces not
  doubles; corrupt pool degrades to empty. Gate: **538 → 550 tests**.
  *(Sprint #75 — done.)*
- ✅ **Autonomous goal decomposition** — `aura/orchestration/auto_planner.py`,
  behind `AURA_AUTO_PLANNER=enabled` (the runner refuses otherwise). An
  objective decomposes into **measurable sub-goals** — a sub-goal with no
  metric gets a *stated* "unmeasured" placeholder, never a silent blank
  (Law 1) — each pursued through the #23 DAG planner, evaluated for execution
  health, and **iterated with the failure fed back** (a revision, not the same
  ask re-sent) up to `AURA_AUTO_ITERATIONS` (default 2), then reported failed —
  bounded, never a loop. **Human checkpoints** (`AURA_AUTO_CHECKPOINTS=enabled`)
  file every sub-goal in the #8 approvals queue *before* execution and defer;
  `execute_if_approved` is **fail-closed** (pending/rejected/unknown/wrong-tool
  ⇒ nothing runs; approved ⇒ the real work runs — both test-proven). Evaluation
  honesty: the module checks the run completed cleanly and never claims a metric
  moved (that's the scoreboard's job, stated). Keyless (prose degrades to one
  sub-goal = the objective). Gate: **+13 tests**. *(Sprint #76 — done.)*
- ✅ **Goal tracking & progress reports** — `aura/orchestration/goals.py`:
  persistent, cross-run goal tracking on two SQLite tables (`goals` +
  append-only `goal_updates` — progress claims keep their provenance),
  tenant-scoped via the #46b contextvar. **Progress is recorded, never
  inferred** (`progress(fraction, note)` — latest wins, history kept);
  **blockers are first-class** (`block`/`unblock`); `status_report` is a
  deterministic keyless render (progress bar, open blockers, **staleness** flag
  after `AURA_GOALS_STALE_DAYS`=7). `track_objective()` is the #76 seam — an
  AutoPlanner run becomes a tracked goal with sub-goal outcomes as its first
  updates (deferred/failed sub-goals open blockers naming their approval id).
  Dashboard gains a **🎯 Goals** panel over a `GET /api/goals[/{id}]` read model.
  Gate: **+11 tests**. *(Sprint #77 — done.)*
- ✅ **Formal behaviour bounds** — `aura/safety/formal.py`: declarative
  constraints in `AURA_BOUNDS` (inline JSON or file). Unset ⇒ byte-identical
  behaviour; **set-but-unreadable OR a typo'd key ⇒ fail closed**
  (`BoundsConfigError` — a policy you can't read must never become "no policy").
  Plan-level bounds (`max_plan_steps`) are **enforced at planning time**:
  `enforce_plan` is wired into the DAG planner's `plan()` **and** `replan()` (a
  recovery plan obeys the same policy), rejecting a violating plan with
  `BoundsViolation` **before anything executes**. Runtime bounds ride a
  per-run `make_bounds_guard` on_tool_call hook: `forbidden_tools`,
  `max_tool_calls`, per-tool `tool_budget`, `forbidden_sequences` (adjacency —
  the "read-secrets then exfiltrate" shape), and `require_approval` (mandatory
  HITL via the #8 store). Blocked calls don't consume budget (test-proven).
  Honest scope: `max_cost_usd_per_run` is *declared* here but only #79 can
  verify it (cost is post-reply). Gate: **+13 tests**. *(Sprint #78 — done.
  T-frontier safety item.)*
- ✅ **Constraint verification** — `aura/safety/verifier_formal.py`: the
  post-execution receipt for #78. `verify_trace` reads a completed run's spans
  and checks every declared bound — including the ones only checkable
  post-hoc: **total cost** (summed `meta.cost_usd`, #14's FinOps figures) and
  **mandatory-approval bypass** (a `require_approval` tool that *executed* must
  have an approved record — execution without one means the guard was bypassed,
  exactly what this catches). A **missing trace is itself a violation** (an
  unverifiable run is not a verified run — the #61 posture).
  `verify_and_flag` escalates any violation behind `AURA_FORMAL_VERIFY=enabled`
  (one `bounds:violation` approvals entry per run + a #60 audit record);
  detection is unconditional, only escalation is flag-gated. Honest scope on
  "automatic rollback": AURA's tool side effects aren't transactional, so the
  card's rollback is truthfully **detection + human escalation**, stated — not
  a promise code can't keep. Gate: **+13 tests**. *(Sprint #79 — done.)*
- ✅ **Adversarial robustness testing** — `aura/evaluation/adversarial.py`: a
  **fixed, keyless red-team corpus** (no LLM generating attacks — that would be
  unrepeatable and need a key) across four classes, each asserting an
  **existing** defence fired: `prompt_injection` → `scan_input` flags it;
  `memory_poisoning` → `screen_memories` quarantines it; `schema_breaking` →
  `coerce_step_result` never crashes and degrades to a bounded contract (#30);
  `sandbox_escape` → `validate_source` rejects the body before it runs (#70).
  `run_suite` scores `resilience` per class + overall and **names every
  breach** so a regression is actionable. A test neuters a defence to prove the
  suite genuinely detects a breach (guard against a vacuous 100%). Optional CI
  gate `AURA_ADVERSARIAL_GATE=enabled` exits non-zero below
  `AURA_ADVERSARIAL_MIN` (default 1.0 — a single breach is a real regression);
  off by default. Gate: **+10 tests**. *(Sprint #80 — done.)*
- ✅ **Explainability layer** — `aura/orchestration/explain.py`: reconstructs a
  run's **decision chain** (why this plan / worker / tool / answer) *from the
  spans AURA already records* — it never re-asks a model to rationalise, because
  a post-hoc LLM story is plausible, not the actual reason (the repo's honesty
  rule). Each link is `{step, kind, decision, because}` where `because` is the
  recorded evidence (the manifest offered these workers; this step depends on
  s1; the verifier's vote said grounded). Missing evidence is stated honestly
  ("arguments not captured" / "enable AURA_TRACE_CONTENT"), never fabricated;
  a missing trace is a stated note, not a crash. `explain_text` renders a
  numbered narrative; surfaced at `GET /api/traces/{id}/explain` for the
  dashboard/Theater. Gate: **+8 tests**. *(Sprint #81 — done.)*
- ✅ **Causal reasoning** — `aura/orchestration/causal.py`, planner hook behind
  `AURA_CAUSAL=enabled`. Deterministic downstream-effect prediction over the
  knowledge graph's **recorded relations**: `causal_effects(graph, entity)`
  walks the backend-neutral `neighbors()` facade to `AURA_CAUSAL_DEPTH` (2)
  hops — bounded, cycle-safe (test-proven) — and every effect carries its full
  relation path as evidence (`(deploy) -[triggers]-> (restart) -[affects]->
  (sessions)`), ranked depth-then-lexical so output is stable. **Honesty is
  the design:** graph edges are recorded relations, not proven causation — the
  module predicts *plausible* effects and says so in the docstring and the
  rendered text; unknown vs. terminal entities are distinguished ("no known
  downstream effects recorded"), never fabricated; zero LLM calls. Wired into
  the DAG planner's plan prompt only (`plan_note(task)`: deterministic
  whole-word entity match, no NER; lazy import): flag off ⇒ `""` ⇒ the prompt
  is **byte-identical** to legacy (test-proven). Replan/scoped-context notes
  and a dashboard surface deferred, named. Gate: **617 → 631 tests**.
  *(Sprint #82 — done.)*
- ✅ **Counterfactual analysis** — `aura/orchestration/counterfactual.py`:
  "what if step X had succeeded/failed differently?" answered from **recorded
  spans** (accepts a trace id, a trace dict, a `DAGPlanExecutor.run()` output,
  or bare steps) — the explain.py precedent: pure read, zero LLM calls, no
  flag needed because nothing ambient changes. **The honesty split IS the
  output:** every claim is stamped `structural` (determined by the recorded
  `needs`-DAG + executor mechanics — which dependents' contexts change,
  whether the replan machinery would be triggered/avoided, graft-risk to
  steps pending at failure time) or `requires_reexecution` (all content: what
  any downstream step would actually have produced — a replay must never
  invent step outputs, and the rendered narrative says so); an operator's
  `hypothetical_result` is always labelled "a hypothesis, not a real result".
  Replan-grafted `r*_` steps reconstruct correctly from `plan:step` spans;
  missing trace/step ⇒ a stated note, never a crash; cycles degrade; claim
  order is deterministic (topological, then declared). Dashboard endpoint
  deferred, named (would ride `AURA_COUNTERFACTUAL=enabled`). Gate:
  **631 → 645 tests**. *(Sprint #83 — done.)*
- ✅ **Agent-to-agent teaching** — `aura/learning/teaching.py`, behind
  `AURA_TEACHING=enabled` (anything else ⇒ no-op: no store file, playbook
  untouched — test-proven byte-identical). **Keyless distillation, not
  LLM-authored examples:** demonstrations are drawn from *recorded evidence*
  — clean traces (plan shape + tool sequence, structural fields only),
  accepted optimizer prompts (prompt_store), golden passes — and emitted as
  `[teaching]`-tagged playbook lessons (the #72/#75 seam), each carrying
  teacher stats + `(evidence trace:…/prompt_store:…/golden:…)` provenance.
  **Measured, never asserted:** teacher eligibility = pass rate ≥
  `AURA_TEACHING_MIN_SCORE` (0.8) over ≥ `AURA_TEACHING_MIN_TRIALS` (3)
  recorded outcomes — abstains under-evidence (the #73 one-pass-isn't-proof
  discipline; a 2/2 lucky agent is excluded, test-proven); lessons are
  **advisory** — the learner's gated prompt store is provably untouched; with
  `tasks` given, curriculum `focus()` (read-only) targets the learner's
  hardest measured task kinds first. Privacy: a secret-bearing span value
  leaks into neither `distill()` output nor emitted lessons (test-proven);
  failed/error traces never become demonstrations. Deferred, named:
  per-learner playbook namespacing (single shared playbook — lessons tagged
  `for <learner>` instead) and automatic `record_outcome` wiring from the
  golden gate (operator-driven, the curriculum precedent). Gate:
  **645 → 659 tests**. *(Sprint #84 — done.)*
- ✅ **Continuous self-benchmarking** — `aura/evaluation/self_bench.py`.
  **Reuse, not reimplementation:** `snapshot()` re-runs the four keyless
  golden categories via golden_ingest's own checkers + a reduced pass of the
  #67 bench suites (same measurement code, reps capped by
  `AURA_SELF_BENCH_REPS`=3 — a cron tick must be cheap) into a bounded JSON
  trend store (`AURA_SELF_BENCH_DB`, default `.aura_self_bench.json`,
  gitignored; history capped at `AURA_SELF_BENCH_MAX_SNAPSHOTS`=500); cost
  read from usage metrics, omitted honestly on failure; test-count omitted
  honestly (counting tests means running the suite — CI's job, stated).
  `detect_degradation` compares the newest snapshot to a trailing-median
  baseline (`AURA_SELF_BENCH_WINDOW`=5): **any golden drop is degradation;
  latency only above `AURA_SELF_BENCH_LATENCY_TOL` (1.5×)** — wall-clock is
  machine-noisy (the #67 lesson: tests inject fake histories, never assert
  the clock). Alerts ride the #52 webhook seam (`selfbench.degraded`,
  minimal payload, never raises — broken-transport-tested). **Scheduling
  honesty (#39/#58 precedent): no background threads** —
  `python -m aura.evaluation.self_bench` (exit 1 on degraded) is the one
  command an operator crons; explicit invocation is consent (the audit-CLI
  pattern), ambient use gated by `AURA_SELF_BENCH=enabled`. Dashboard gains
  a **🩺 HEALTH** panel (`GET /api/health` behind the existing auth
  middleware; inline-SVG sparklines, zero new JS deps, 30s refresh — health
  changes slowly; empty store renders "no snapshots yet — cron …", never a
  crash). Honest caveat, named: with a 1-snapshot baseline at reps=1,
  latency noise can trip 1.5× (observed in smoke) — the defaults smooth it
  and golden scores are noise-free; per-suite tolerances are a possible
  follow-up. Gate: **659 → 679 tests**. *(Sprint #85 — done. The Wave-6
  code sprints are complete; #86–#88 remain.)*
- ✅ **Research paper** — [docs/AURA_PAPER.md](docs/AURA_PAPER.md): an
  arXiv-style technical report (~5,300 words; Abstract → Related Work →
  Architecture → **the gated-sprint methodology as the primary contribution**
  → self-improvement machinery → safety & trust → evaluation → lessons →
  limitations → 21 references). Law 1 all the way down: every quantitative
  claim traces to a repo number (679 tests, 26 fixtures/4 categories, 5
  backends, tag-by-tag gate growth 126→679; benchmark figures cross-checked
  against benchmarks/RESULTS.md); §7 separates **measured** from **not
  measured** — no live-model quality comparison exists yet (that is #87's
  card, stated), the optimizer receipt is still an operator action, distributed
  throughput is unmeasured. Law 2: every external citation either re-verified
  by web search in the authoring session (arXiv IDs for ReAct/Reflexion/CoT/
  Plan-and-Solve/AutoGen/RAG/GraphRAG/MemGPT/Mem0/DSPy/TextGrad; LangGraph/
  CrewAI repos) or carried with identifiers from the repo's own dated
  in-session verifications (SimpleMem, MCP spec, A2A v1.0, RFC 8615,
  CVE-2025-6514, OWASP ASI06) — the verification method is stated in the
  References preamble; one reserved slot is marked as such rather than
  back-filled. Honest format note up front: Markdown master; LaTeX conversion
  + actual arXiv/workshop submission are operator steps. Gate: unchanged
  (679 tests; golden N/A — a paper makes no accuracy claim). *(Sprint #86 —
  done.)*
- ✅ **Benchmark vs. alternatives** — `benchmarks/compare/`: honesty reshaped
  the card before code did — SWE-bench/HumanEval need live keys and hours of
  compute, so what ships reproducibly and keylessly is the #67 discipline
  applied comparatively: **orchestration overhead + sourced capability
  coverage, never answer quality** (stated everywhere; the live-key quality
  comparison is named operator future work with its protocol in
  METHODOLOGY.md). Three scenarios (tool chain / 3-agent pipeline / router
  dispatch) replay **identical scripted LLM turns** through each framework's
  own documented idiom — AURA (`ToolAgent`/`Supervisor` on the real stub),
  **LangGraph 1.2.9, CrewAI 1.15.2, AG2 0.14.0** (versions/licenses/the
  AutoGen→AG2 fork verified on PyPI 2026-07-11, links in the docs) — with a
  correctness gate per cell (a fast wrong run scores nothing), setup timed
  separately, and an exhaustion rule so a framework's extra internal LLM
  calls are honestly its own overhead. **All 12 cells ran green on this
  machine** (RESULTS_COMPARE.md, n=15, dated) in an isolated `.venv-compare`
  — the gate venv provably never grows the competitor deps (test-asserted).
  Fairness stated, not implied: AURA's cells run its production posture
  (persistent span tracing, guards, cache checks — the conservative
  direction), competitors run bare graphs; every adapter workaround is named
  in METHODOLOGY.md (CrewAI re-kickoff manager rebuild; AG2's re-register-
  after-tools, `message_retrieval_function` hasattr quirk, and custom
  speaker selection — which flatters AG2's router cell by one LLM call,
  stated). Feature matrix is **sourced, not scored** (doc links per cell,
  checked 2026-07-11). README gains the comparison section + the stale test
  count fixed (617 → 689). Adapters are published for maintainer review.
  Gate: **679 → 689 tests** (suite ×2). *(Sprint #87 — done.)*
- ✅ **Wave 6 release notes + `v2.0.0` tag** — CHANGELOG `[2.0.0]` ledgers
  the wave (tests **468 → 689**, golden 4 categories / 26 fixtures held green
  throughout; tool evolution #69–71, learning stack #72–75, autonomous goals
  #76–77, formal safety #78–80, explainability #81, causal/counterfactual
  #82–83, teaching #84, self-benchmarking #85, paper #86, peer comparison
  #87); `__version__` 1.2.0 → **2.0.0**; annotated tag. Carried forward,
  named: BYOK optimizer receipt, TS SDK/publishing, MCP stateless core (RC),
  SQLCipher, TH-track #89–#100. *(Sprint #88 — done. Wave 6 closes.)*

**Wave 6 exit criteria, assessed at the tag:** `v2.0.0` tagged ✓. Agents
autonomously synthesise tools (approval-gated ✓), learn across sessions
(patterns/meta/curriculum/collective/teaching ✓), decompose high-level goals
(✓, checkpointed), and explain their decisions (✓, from recorded spans).
Formal safety bounds enforced ✓ (plan-time + post-hoc verification).
Adversarial robustness tested ✓ (fixed keyless corpus). "Paper published":
written + citation-verified in-repo ✓; arXiv/workshop submission is a named
operator action ⚠ — the criterion is met in substance, stated honestly.

---

## Theater — ongoing enhancements (Sprints #89–#100)
The cinematic experience reaches its full vision: immersive, shareable, accessible.

- ✅ **TH-7 — Round-table debate stage** — `theater_app/src/three/stages/
  RoundTable.tsx` + the **stage registry seam** that makes this whole wave
  parallel-safe: `stages/index.ts` maps each SceneId to a themed set behind a
  typed `StageProps` contract (Stage.tsx keeps the shared floor + cast; a
  stage renders everything scene-specific and may read the store like
  Scene.tsx does), with a shared `reducedMotion` flag every stage must honor.
  The set: pedestal round table sized from the cast ring's actual radius,
  glowing rim, a hover-chair behind every seat trimmed in that agent's cast
  colour. Beat-driven staging, all derived from ep + store.beat + positions
  (deterministic): an **active-speaker spotlight** (SpotLight + additive cone
  and floor pool, tinted with the speaker's colour, gliding between seats —
  jumping instead when reducedMotion), a **rebuttal cue** (speaker change vs.
  the previous beat fires a quadratic energy arc + travelling orb between the
  two seats; disabled under reducedMotion), and a **live vote tally** board
  above the table counting argument beats per side (`Beat.side` left/right
  over beats[0..current] — an honest tally of real beats, not invented
  votes). Camera cuts stay TH-11's card. Verified live in-browser: a real
  keyless `debate()` run written to the store classified `round`, mounted,
  and stepped beats with zero console errors (full animation confirmation
  needs a visible pane — Chromium suppresses rAF in a hidden one, stated).
  Gate: 689 tests unchanged; `vite build` green. *(Sprint #89 — done.)*
- ✅ **TH-8 — Courtroom judge stage** — `stages/Courtroom.tsx`: judicial bench
  built around the judge's raised blocking (front panel kept below the torso
  band so the character stays readable), witness stand, gavel + sound block,
  scales of justice, ruling card. **Verdict and score are derived, never
  invented — the rules are stated in the file header:** verdict beat = the
  LAST beat by the lead (cast[0]) or of kind `eval` (fallback: last beat; no
  beats ⇒ no card, no strike); score = ok/total over all beats' spans
  (ok ⇔ no error and status ≠ "error"), zero spans ⇒ null ⇒ the scales stay
  balanced. Staging: when the verdict beat becomes current the gavel plays a
  gsap raise-and-strike with an emissive impact flash (blooms past the 0.62
  threshold) and the ruling card (verdict text, ~90 chars, drei Text) pops
  in and stays; the scales stay level until the verdict, then damp-tip
  `(score−0.5)×0.9` rad with pans counter-rotated to hang level (one
  allocation-free useFrame). reducedMotion: gavel rests, card just appears,
  scales snap. Verified live in-browser: a real keyless `verify()` run
  classified `courtroom`, mounted, and stepped beats with zero console
  errors. ~31 draw calls added. Gate: 689 unchanged; `vite build` green.
  *(Sprint #90 — done.)*
- ✅ **TH-9 — Knowledge-lab stage** — `stages/KnowledgeLab.tsx`: plan-and-
  execute as an assembly line, every element **derived from the cast's actual
  blocking** (deterministic `buildLine()`): a CatmullRom conveyor pulled 0.85
  toward the arc's focal point so it runs in front of the workers at knee
  height (crawling chevron flow + emissive guide strip), 2–3 **machines in
  the gaps between workers** (press straddling the belt oriented to the track
  tangent; side machines facing back at it) — on a `tool` beat machine
  `beat.i % count` lights past the bloom threshold and animates; **dispatch
  pulses** (while a worker speaks, a glowing packet travels the line from the
  planner's curve-parameter to the speaker's — plan dispatch made visible);
  and the **knowledge crystal** centre-back growing with real progress
  (scale + orbiting shard facets driven by `(beat+1)/beats.length`, extra
  glow pulse on `llm` synthesis beats), fed by pipes from both belt ends.
  Instanced flasks + containment rings for dressing; shared module-lifetime
  materials, allocation-free useFrame (reused temps). reducedMotion: static
  set — no crawl/packets/pulse, machines light without animating. Verified
  live in-browser: the live `ontology synthesis` trace classified `lab`,
  mounted, stepped beats, zero console errors. Gate: 689 unchanged;
  `vite build` green. *(Sprint #91 — done.)*
- ✅ **TH-10 — Copilot workstation stage** — `stages/Workstation.tsx` (the
  "floor" scene): a low desk + terminal in front of the row's centre
  character (desk top ≈0.78 so the copilot stays visible), the screen running
  a **procedural GLSL scrolling-activity shader** whose level follows real
  playback (full while `playing`, a decaying pulse after each beat change,
  frozen dim glow when idle); three desk gadgets — on a `tool` beat, device
  `beat % 3` flashes orange past the bloom threshold; a **memory bookshelf**
  (instanced books, deterministic hash-seeded sizes/colours) whose reserved
  book slides out and glows teal on `memory` beats; and an **answer beam**
  rising from the terminal toward the camera on the final beat. Mug, lamp,
  cables, rug for dressing; allocation-free useFrame. reducedMotion: frozen
  pattern, highlight-only cues, fixed-opacity beam. Verified live in-browser:
  the live `chat` trace classified `floor`, mounted, stepped beats, zero
  console errors across ALL FOUR stages this wave. The rebuilt
  `static/theater3d_app` bundle (all four themed stages) ships with this
  sprint. Gate: 689 unchanged; `vite build` green ×2. *(Sprint #92 — done.
  Phase C, the four themed stages, is complete.)*
- ✅ **TH-11 — Director / camera system** — `three/Director.tsx` rewritten as
  a real auto-cinematographer with a **deterministic shot grammar** (every
  shot derives from scene + blocking + the current beat; variation is beat-
  index parity, no randomness): **film cuts, not swoops** — a speaker change
  CUTS (instant reposition + 0.5s micro-settle) while the same speaker
  continuing GLIDES (1.1s ease); per-scene coverage — round = over-the-
  shoulder debate shots across the table with alternating lateral angles;
  courtroom = low counsel shots from the well, the bench shot when the judge
  speaks, and a **3.2s slow-motion push-in on the verdict beat** (same
  deterministic verdict rule the Courtroom stage uses); lab = a lateral
  tracking dolly following the speaker down the line, final beat pulling
  back to reveal the crystal; floor = frontal push-ins. **Manual camera:**
  a 🎥 HUD toggle (store `manualCam`) unmounts the Director and mounts drei
  OrbitControls (damped, clamped below the floor plane). **reducedMotion:**
  cuts only — instant repositions, no glides, no drift, no slow-mo. Honest
  delta from the card, named: **depth-of-field stays out** — TH-2's recorded
  lesson removed DoF as the dominant flicker + GPU cost; a bokeh pass
  belongs to TH-12's quality presets if anywhere. Verified live in-browser:
  camera toggle both ways, verdict-dolly path on the verify episode, cut/
  glide paths on the debate episode — zero console errors; `vite build`
  green; 689 Python tests unchanged. *(Sprint #93 — done.)*
- ✅ **TH-12 — Environment & FX** — the **quality preset is the backbone**:
  store `quality` (low/medium/high; ✨ HUD selector, persisted to
  localStorage — persistence verified across a reload; auto-detect starts
  coarse-pointer small screens and reduced-motion viewers on low, desktops
  on high) gates DPR ([1,1]/[1,1.5]/[1,2]), antialias, shadows, the post
  stack (low = none; medium = Bloom; high = Bloom + Vignette + a subtle
  BrightnessContrast grade — **DoF stays out on every preset**, the TH-2
  flicker/GPU lesson), the environment and all particles.
  `three/Environment.tsx`: procedural gradient sky dome (one draw call,
  GLSL, purple horizon matching the fog), drei Stars (1100 high / 450
  medium / none low; static under reducedMotion), and two fake-volumetric
  additive cones on high only (fake beats raymarching on a laptop).
  `three/FX.tsx`: three beat-driven particle systems, **each visualizing a
  real span property, never decoration** — token flow (arcing points from
  the speaker while an `llm` beat plays), cache sparkle (a 0.8s teal burst
  when the current beat's span has `cached: true`), error smoke (grey
  points rising while the current beat carries an error); buffers allocated
  once (160/80 flow, 48/24 smoke, 40/20 spark by preset), allocation-free
  useFrame; low preset or reducedMotion ⇒ the component renders nothing.
  **Mobile→2D fallback:** coarse-pointer small screens get a dismissible
  banner linking the lighter `/theater` (the 3D stage is the premium
  experience, never the only one). Verified live in-browser: all three
  presets cycled (renderer reconfig incl. post-stack/Stars/FX remounts),
  beats stepped, persistence round-tripped — zero console errors;
  `vite build` green; 689 Python tests unchanged. *(Sprint #94 — done.)*
- ✅ **TH-13 — HUD & inspector overlay** — honesty audit first: the
  click-a-character trace card already existed (#18's inspector: system
  prompt, tool I/O, tokens, cost — Stage.tsx routes a character click to the
  actor's latest beat), so this sprint shipped the real deltas. **Live
  meters:** the stat row now accumulates *through the current beat* (`32/48
  tokens so far`, `$cum/$total est cost`, current-beat latency, `beat n/N`)
  with a count-pop animation — the numbers grow with the performance instead
  of spoiling the episode. **Film-strip scrubber:** the dot track became
  avatar frames (kind-coloured border, error badge ⚠, hover tooltip with
  actor · kind · 90-char line preview; click = jump + open inspector).
  **Kind filter chips:** chips with per-kind beat counts (colours from
  KIND_COLOR); active chips dim non-matching frames and re-aim ◀▶ to hop
  between matching beats — **a navigation aid only: playback itself always
  performs every beat** (filtering the replay would misrepresent the run,
  stated in code). Inspector gains **per-actor run totals** (beats · tokens
  · cost for the selected character). Found en route by the browser pass:
  the first build crashed with React #310 — new `useMemo`s sat below the
  `!ep` early return (hooks-order violation invisible to `vite build`);
  fixed by hoisting hooks above the guard — exactly the failure class only
  live verification catches. Verified live: chips/filtered-jump/meters/
  film-strip all exercised on the multi-kind verify episode, zero errors on
  the fixed bundle. Gate: 689 unchanged; `vite build` green. *(Sprint #95 —
  done.)*
- ✅ **TH-14 — Replay, record & share** — **the share format is the trace,
  never the render** (`data/episodeFile.ts`, format `aura-episode/1`; a bare
  `/api/traces/{id}` JSON is also accepted): `compile()` is pure, so
  re-screening a saved file reproduces cast/beats/captions/staging
  deterministically — this also closes TH-3's long-open replay-spine import
  half. 💾 **Export** downloads the current run; 📼 **Import** (file picker)
  validates the shape (corrupt file ⇒ a friendly note, never a crash),
  compiles, becomes the current episode (re-import replaces by trace id,
  never duplicates), and persists to a localStorage **gallery** (capped at
  8, oldest dropped; oversized traces import session-only — stated in the
  UI note); persisted episodes rejoin the picker ahead of fetched runs on
  every load, prefixed 📼. ⏺ **Record** (`three/record.ts`): MediaRecorder
  over `canvas.captureStream(30)` (vp9→vp8→default), starts playback from
  the top, **auto-stops when the run ends** (700ms tail) or on ⏹, downloads
  the `.webm`. Honest deltas, named: **GIF/MP4 need an encoder dependency —
  WebM-only ships**; TTS audio cannot be routed out of speechSynthesis, so
  recordings are silent with captions in-frame (stated in the tooltip).
  Verified live: a seeded gallery trace survived reload, compiled into a
  playable 3-beat episode (frames/chips/scene classification all correct),
  export clicked, and a full record→auto-stop→download cycle completed —
  zero console errors. Gate: 689 unchanged; `vite build` green.
  *(Sprint #96 — done.)*
- ✅ **TH-15 — Theming, i18n, a11y & deploy** — honesty reshaped two claims
  before code: **beat captions are real trace content, so a keyless UI does
  not machine-translate the agents' words** (UI *chrome* is localized; trace
  text stays verbatim — stated in i18n.ts and the README), and static
  build + Docker already ship via the dashboard image (#26/TH-2), so the
  deploy delta is demo mode + docs. Shipped: **themes** — 🎨 aurora (default)
  / noir / daylight restyle the HUD chrome via CSS variables on
  `body[data-theme]` (the 3D stage keeps its cinematic palette — a lit
  control room around a dark theater), persisted; **i18n** — `ui/i18n.ts`
  dictionary for English / සිංහල / العربية / বাংলা covering every chrome
  string (buttons, stats, captions' empty state, tips), Arabic flips the HUD
  to RTL via `body.dir`, persisted; **a11y** — keyboard controls (Space
  play/pause, ←/→ beats, Home/End, V voice, C camera; never steals keys
  from form controls), `aria-label`s on icon-only controls, a
  `:focus-visible` ring, and the caption as an `aria-live="polite"` region
  so screen readers hear each beat (reduced-motion was already honored
  end-to-end by #89–#94); **demo mode** — `/theater3d?demo=1` skips the
  backend and screens the bundled episodes (a sales laptop needs a browser,
  nothing else); **docs** — theater_app/README gains the full TH-11…TH-15
  usage section. Verified live: demo mode ("bundled episodes" pill), Sinhala
  stat labels rendered, daylight theme on body, Arabic → `dir=rtl`, two
  synthetic ArrowRight presses advanced the beat 0→2/6, persistence
  round-tripped — zero console errors. Gate: 689 unchanged; `vite build`
  green. *(Sprint #97 — done. Follow-up by operator request: **Tamil/தமிழ்
  added** as a fifth chrome language, verified live — the requested Sinhala,
  Bengali and Arabic were already in.)*
- ✅ **Code-split the bundle** — measured before → after: one **2,024 KB
  monolith (615 KB gzip)** → an initial load of **~462 KB raw / 158 KB gzip**
  (entry 30.5 + vendor 245 + motion 185 + shared 1 + CSS 7.5) — **under the
  <500 KB target, 77% smaller**, with the heavy islands streaming lazily:
  three.js+R3F+drei+post (928 KB), Cytoscape (444 KB), React Flow (137 KB),
  Scene (44 KB). How: `React.lazy` islands (Scene / FlowPanel / GraphPanel;
  the HUD paints immediately from the entry with a "Setting the stage…"
  fallback) + function-form `manualChunks` vendor groups. **Found en route,
  worth the ledger:** two rollup chunking hazards silently dragged the 1.17 MB
  three chunk back into the eager graph — (1) Vite's 1 KB preload-helper got
  hoisted INTO the three chunk (the entry imported a megabyte to reach it;
  pinned to its own `shared` micro-chunk), and (2) `three-stdlib` slipped past
  a word-boundary regex into the eager vendor chunk and statically imported
  three (island regex widened to the whole ecosystem). Both were invisible in
  the size table and caught only by inspecting `index.html`'s modulepreloads
  and each chunk's import statements — chunk sizes lie about the eager graph;
  the preload list doesn't. Verified live: the network waterfall shows exactly
  the eager wave (index/vendor/shared/motion/CSS) then the lazy islands, app
  mounts, zero console errors. Gate: 689 unchanged; `vite build` green.
  *(Sprint #98 — done.)*
- ✅ **Environment realism overhaul (operator feedback)** — the operator's
  review: "no walls, no room, no furniture — make it realistic, keep it
  smooth in any browser." Honest framing first: within the theater's own
  constraints (no external assets, no CDNs, all-browser WebGL), realism =
  architecture + material response + lighting, not photo textures. Shipped:
  `stages/Room.tsx`, a parameterized procedural **room kit** — segmented
  instanced walls with baseboard + glowing crown cove, pillars, ceiling ring
  with recessed light cove (stars visible through the oculus), and a
  one-draw-call plank/tile floor shader with a baked edge shadow — plus a
  furniture kit (Table / Chair / BenchRow / Cabinet / Plant / WallScreen).
  Four room presets wired into the stages: **council** chamber (wall
  screens, plants) around the debate table; wood-panelled **courtroom**
  (wainscot rail, counsel tables, bar rail, two public gallery bench rows);
  industrial **lab hall** (wall pipe rings, side work tables, cabinets);
  ops **office** (wall screens, filing cabinet, plants, guest chair). The
  old sci-fi grid floor retired — every scene stands in a real room.
  **Smoothness levers, verified live: 50 FPS measured on integrated Intel
  Iris Xe at the high preset with the pane visible** — one shared material
  per surface family, instanced repeats, zero per-frame work in the shell,
  and the `low` preset mounts walls + floor only. All four rooms mounted in
  the browser with zero console errors; screenshot confirmed walls/
  furniture/lighting render. Gate: 689 unchanged; `vite build` green.
  *(Sprint #98b — done.)*
- ✅ **WebXR / VR mode** — `@react-three/xr@5.7` (Law-2 checked before
  install: peers react≥18 / three≥0.141 / fiber≥8 all satisfied; the
  package lands in the lazy three island). Shipped: the whole scene wrapped
  in the `<XR>` provider; a 🥽 HUD button (status-aware — **renders empty
  and hides when WebXR is unsupported: the standard WebGL view IS the
  fallback**, verified live in a non-XR browser); in an immersive session
  the **headset owns the camera** (Director + OrbitControls stand down),
  the world shifts so the viewer spawns standing on the stage floor ~4 m
  from centre — *walk around the agents*; `<Controllers>` render with rays,
  and every character is wrapped in `<Interactive onSelect>` so **pointing
  a controller at a character selects its latest beat** (same handler as a
  mouse click). Post-processing is skipped while presenting (the frame
  budget belongs to the headset compositor). Found en route, both caught by
  the #98 preload-list check and a stale-cache re-check: (1) importing the
  XR button statically in the HUD dragged the 1 MB three island eager —
  fixed by lazy-loading the button (it appears with the Scene island that
  needs the same chunk anyway); (2) the lib nullish-coalesces a null
  children-callback back to its own "HTTPS needed" label — return `""` so
  the unsupported button truly hides. Honest deferrals, named: the in-VR 3D
  trace card (DOM HUD doesn't exist in immersive; selection persists to the
  inspector on exit), spatialized voice (Web Speech can't be routed to 3D
  audio), and **real-headset verification is operator-side by nature** —
  what's machine-verified is the fallback path, session wiring, and a clean
  console. Gate: 689 unchanged; `vite build` green; initial load still
  ~462 KB. *(Sprint #99 — done.)*
- ✅ **AI cinematography director** — `three/tension.ts`: the director's
  brain is a **deterministic dramatic-tension model over the scene script**,
  built to AURA's own agent discipline — it *perceives* (compiled beats),
  *reasons with a stated evidence trail* (rebuttal / clash / failure /
  interrogation / verdict-near / consensus / opening / closing — every
  signal derives from real beat data, nothing invented), *acts* (the #93
  Director consumes the score: **≥0.62 pushes in 32% with snappier glides —
  zoom in on conflict; ≤0.28 pulls to the establishing wide — consensus;
  the verdict slow-mo stays**), and *explains itself*: a 🎬 HUD pill renders
  the current framing with its evidence ("tight — rebuttal · failure"),
  mirroring explain.py's rule that an unexplained decision is a
  rationalization. **One brain, two consumers** — found en route: setting
  the note from inside the R3F canvas via the store never surfaced while
  the tab was hidden (canvas-side effects don't process), so the HUD
  computes its explanation directly from the same pure module — better
  architecture forced by a real bug. Verified twice: **headlessly**
  (esbuild+node over a synthetic script: opening 0.31 → rebuttal 0.78 →
  failure 1.00 → verdict 0.90; consensus run 0.23 "wide") and **live**
  (debate beats narrated "clash · opening → rebuttal → steady"; the
  courtroom verdict beat read "tight — interrogation · verdict"), zero
  console errors, eager bundle untouched. Honest scope, named: this is the
  keyless director; an LLM-driven director reading the script and emitting
  shot hints is the BYOK increment. Gate: 689 unchanged; `vite build`
  green. *(Sprint #100 — done. **The 100-sprint roadmap is complete.**)*

**Theater exit criteria, assessed at #100:** all 4 themed stages operational
✓ (#89–#92, each in a real room since #98b); full auto-cinematography ✓ (#93
shot grammar + #100 tension-driven framing); episode recording & sharing ✓
(#96 — WebM + `.aura-episode.json`; GIF/MP4 stay named deferrals); VR mode ✓
(#99 — headset verification is operator-side by nature); AI-directed camera
✓ (#100 — keyless deterministic director; LLM director is the named BYOK
increment). The Theater is the premium way to observe and debug any AURA
orchestration — and every claim above traces to a verified sprint entry.

---

## Wave 7 — Close the self-evolution gap (Sprints #101–#103)
Planned 2026-07-12 (aura-sprint-planner, frontier claims web-verified that
session, not recalled). The 100-sprint roadmap left three self-evolution gaps
its own documents name: the learning loop's "beyond MIPRO" future-work note,
the Skills-Layer item (S3) that stayed a description, and #70's "true namespace
isolation is the operator's container, stated". **Exit criteria — receipts,
not claims:** (a) a GEPA-accepted prompt variant with a score delta, (b) a
human-approved runtime skill loaded into an agent's loadout, (c) a
WASM-isolated tool execution visible in the trace.

- ✅ **GEPA candidate proposer** — `aura/learning/proposers.py` grows a fourth
  `AURA_PROPOSER` value, `gepa` (Wave 7 opens): the standalone `gepa` package
  (verified 2026-07-12: MIT, actively maintained, gepa-ai/gepa) evolves the
  live prompt via **reflective mutation + Pareto-frontier selection**, seeded
  with the current prompt. Frontier grounding: "GEPA: Reflective Prompt
  Evolution Can Outperform Reinforcement Learning" (arXiv:2507.19457, ICLR 2026
  Oral) — ~+13% over MIPROv2, ~+20% over GRPO at up to 35× fewer rollouts;
  CPU-viable (reflection is LiteLLM calls, no GPU kernel). It is the
  actively-maintained successor of the textgrad lineage whose package #47b
  rejected as stale — the pattern AURA kept now gets its living implementation.
  **Better candidates, never a looser gate:** `optimize_prompt` /
  `run_improvement_cycle`'s accept-only-if-improves contract is byte-for-byte
  unchanged, and a test drives a gepa-shaped proposer end-to-end to pin it.
  Honesty, named: the adapter is deliberately **defensive against API drift**
  (entry point `optimize_anything`→`optimize` and result shape — best /
  candidates / Pareto list; str, dict, or nested candidates — resolved at call
  time; any gepa failure degrades to zero candidates, never a crashed loop);
  a real GEPA run is BYOK + `pip install "aura-agents[gepa]"` (lazy import,
  same contract as `[dspy]` — new extra in pyproject), keyless suite uses
  fakes. Gate: **689 → 695 tests**. *(Sprint #101 — done. Wave 7 opens.)*
- ✅ **Runtime skills layer (EvoSkill pattern)** — `aura/learning/skills.py`,
  behind `AURA_SKILL_DISCOVERY=enabled`: agents can now propose and hold
  **procedural skills** — SKILL.md-shaped instructions mined from failed
  trajectories and rendered into the system prompt via a `SkillLoadout`
  (`as_system_suffix()`) — closing the one Skills-Layer item (S3) that stayed
  a description across all 100 sprints (the learning loop was prompt-only).
  The load-bearing safety rule, same posture as #69 one level up: **a skill
  NEVER auto-activates.** `propose_skill` files the candidate in the existing
  approval queue under the new kind `skill:propose`, audited
  (`skill.proposed`, #60); `activate_if_approved` is **fail-closed**
  (pending/rejected ⇒ nothing loaded, test-proven). Retention keeps #5's
  discipline at the skill level: `retain_if_improves` measures an injected
  metric WITHOUT the skill then WITH it and keeps it only if with > without —
  never a regression, by construction (golden-set slices are the production
  metric; a plain callable keeps the gate keyless). Failure analysis reuses
  #12's keyless `evaluate_trajectory` (`AURA_SKILL_FAIL_THRESHOLD`, default
  0.7): errors / looping calls / step budget seed the one-call draft, and an
  unusable LLM draft degrades to a deterministic fallback, never a crash.
  Frontier grounding: EvoSkill (Alzubi et al., arXiv:2603.02766, Sentient +
  Virginia Tech, Apache-2.0, active — verified 2026-07-12; +7.3pp exact-match
  60.6%→67.9% on grounded reasoning by evolving the agent program over a
  frozen base model) — the *pattern* adapted, no new dependency; CPU-fine, a
  skill-authoring loop, not training. Named deferrals, stated not smuggled:
  skill `helpers` ship EMPTY in v1 (freeform helper scripts ride #70's
  sandbox later); SKILL.md folder-on-disk packaging (skill-creator
  coordination) is future work — v1 keeps skills in-store. Gate: **695 → 712
  tests** (neighbor suites rerun clean — no shared-db pollution).
  *(Sprint #102 — done.)*
- ✅ **WASM/Pyodide tool-sandbox tier** — `aura/tools/sandbox.py` grows a
  second, **opt-in** isolation tier: `AURA_SANDBOX_MODE=wasm` runs a freeform
  tool body under **Pyodide** (CPython compiled to WebAssembly) via a Node
  harness — capability-first isolation, no host syscalls reachable from the
  guest — closing the gap #70's own module comment named ("true namespace
  isolation is the operator's container, stated"). The #70 subprocess+rlimits
  tier stays the DEFAULT, byte-for-byte unchanged. **Fail-closed everywhere:**
  an unknown mode is refused outright (a typo can't silently downgrade
  isolation); a missing runtime refuses with `wasm sandbox unavailable: …` and
  explicitly does NOT fall back to the subprocess tier (a caller who asked for
  the strong tier must never silently get the weak one — test-proven with
  both real runners trapped); Gate 1's AST screen still runs FIRST for both
  tiers (invalid body ⇒ runner never called); same JSON-in/JSON-out, timeout,
  and never-raises contracts as #70. Frontier grounding is a **pattern, not
  one canonical repo** (flagged per the frontier-map honesty rule): NVIDIA's
  Pyodide-sandboxing pattern; The New Stack (Mar 2026) on WASM closing agents'
  isolation gap; a Feb 2026 four-primitive survey (containers / gVisor /
  Firecracker / WASM — "near-zero overhead, capability-first"). Named
  deferrals: the runtime is operator-installed (`npm install pyodide`, MIT,
  verified 2026-07-12 — nothing vendored, nothing installed this sprint);
  end-to-end wasm execution is verified via injected runners + a real-node
  availability probe in the keyless suite; memory/CPU limits inside the WASM
  tier ride Pyodide's own heap — stated. Gate: **712 → 723 tests** (#70's
  suite untouched and green). *(Sprint #103 — done. **Wave 7 is complete.**)*

**Wave 7 exit criteria, assessed at #103:** the three *mechanisms* shipped and
are test-proven keylessly — (a) GEPA candidates flow through the unchanged
accept-only-if-improves gate, (b) a human-approved skill loads into a
`SkillLoadout` fail-closed, (c) a wasm-tier execution carries the same traced
tool contract. The three *receipts* the wave asks a replayed session to show
are, honestly, operator actions: (a) needs BYOK + `pip install gepa`, (b) a
real human approval, (c) `npm install pyodide` — same posture as the v2.0.0
optimizer receipt. Follow-up kept in notes, not scheduled: `gepa.gskill`
vs #102's EvoSkill-pattern build (compose or replace — not answerable until
both exist in anger); an MCP-sprawl governance dashboard (packaging, not
capability); AP2/ERC-8004 stays a structural no under Law 3.

---

## Wave 8 — Prove the thesis (Sprints #104–#106)
Planned 2026-07-12 after a five-lab curator review (Anthropic / OpenAI / Meta /
Google / Nvidia lenses). The review's verdict: the engineering craft and the
honesty culture are strong, but the project's *headline claim* — agents that
improve themselves — had **never been demonstrated with a committed receipt**,
the eval stack had **zero contact with any external agent benchmark**, and the
LLM judge was an **uncalibrated single self-judge**. Wave 8 closes those three
(review findings C1/C2/C3). **Honest boundary held throughout:** real-model
runs stay BYOK; each sprint ships a harness *proven keylessly* against
deterministic/fake scorers plus a documented BYOK path to the real number — no
leaderboard claim we can't legitimately produce.

- ✅ **Optimizer proof — the loop, exercised end-to-end** —
  `aura/evaluation/optimizer_proof.py` grows a keyless, **model-free** proof
  path (`run_keyless_proof`, `python -m aura.evaluation.optimizer_proof
  --keyless`): a real, non-trivial, deterministic **structured-extraction**
  task (2 categories — contacts, invoices) drives the live prompt through the
  unchanged accept-only-if-improves optimizer and measures a genuine
  before→after gain, reusing the existing `run_proof` ledger + keyless
  flip-gate-stability machinery. This converts C1's "never exercised
  end-to-end" into a **committed receipt** (`OPTIMIZER_PROOF.md`: contacts
  0.5→1.0, invoices 0.5→1.0, verdict PASSED, keyless gate stable). Honesty,
  stated in code and docstrings: this is a keyless **mechanical proxy** proving
  the loop *machinery* improves a prompt with no key — NOT a substitute for the
  BYOK real-judge receipt (`_live_scorers`, unchanged, still a deliberate
  operator action). The scorer is monotone and inspectable (a richer prompt
  scores strictly higher; a flat category is honestly reported NOT PROVEN, and
  a test pins that). Gate: **723 → 729 tests**. *(Sprint #104 — done. Wave 8
  opens.)*
- ✅ **External agent-benchmark harness (τ-bench-style)** —
  `aura/evaluation/agent_bench.py`: AURA's eval stack had zero contact with any
  standard agent benchmark (C2 — a repo grep for tau-bench / SWE-bench / GAIA
  returned nothing); this ships the **task schema, a runner over the real
  `ToolAgent` think→act→observe loop, and three-part scoring** so AURA can be
  scored on the standard "use tools to finish a realistic multi-step task"
  format. Frontier grounding: **τ-bench** (Sierra, 2024) — retail/airline
  tool-agent tasks judged by exact-match on final backend state + required tool
  calls, with **pass^k** reliability; the shape is reproduced faithfully
  (`pass_hat_k` implements the standard unbiased `C(c,k)/C(n,k)` estimator).
  **Honest boundary, not a leaderboard number:** no network for the real
  dataset and no key for a real model, so the keyless path is proven against
  **5 hand-authored retail tasks** in the real schema, driven by a scripted
  stub that *replays* a known-good trajectory (it does not reason) —
  `--report` proves the plumbing (5/5, pass^1=pass^3=1.0), explicitly banners
  itself "NOT a τ-bench score", and joins the keyless gate. **BYOK to a real
  score:** `load_tasks(path=…)` accepts a converted real τ-bench export and
  `model_agent_factory` + `AURA_LLM=anthropic` runs the same
  `run_benchmark`/`score_task`/`pass_hat_k` — only that config may be called a
  τ-bench result. Scoring: state exact-match (subset default, exact opt-in) AND
  required tool calls satisfied (extras tolerated) ⇒ per-task pass. Gate:
  **729 → 743 tests**. *(Sprint #105 — done.)*
- ✅ **LLM-judge rigor (pairwise + position-bias swap + calibration)** —
  `aura/evaluation/evaluator.py` grows three functions beside the unchanged
  1-5 self-`judge()` (C3): `judge_pairwise` (MT-Bench/Chatbot-Arena-style
  preference — which answer is better), `judge_pairwise_debiased` (the
  load-bearing upgrade — judges BOTH orders and accepts a winner only if it
  survives the swap; a first-answer preference is flagged
  `position_bias_detected` and downgraded to a tie — the standard control for
  the position bias documented in **Zheng et al. 2023**, "Judging
  LLM-as-a-Judge with MT-Bench and Chatbot Arena"), and `calibrate_judge`
  (exact-agreement, MAE, scipy-free Pearson from (judge, human) label pairs,
  degrading gracefully on empty/constant input) so an operator can finally
  **validate the learning loop's `min_score=4.0` cut point against human
  labels**. Honest boundary: real judging is BYOK — the pairwise functions
  take the utility model and are keyless-tested by scripting the stub, exactly
  like `judge()`; the calibration harness is pure. **The existing `judge()`,
  the accept-only-if-improves optimizer gate, and every golden/gate number are
  untouched** (a regression-guard test pins `judge()`'s shape; the four
  existing evaluator-touching suites stay green). Gate: **743 → 758 tests**.
  *(Sprint #106 — done. **Wave 8 is complete.**)*

**Wave 8 exit criteria, assessed at #106:** all three review findings closed at
the *mechanism + keyless-proof* level, with the real-number step honestly BYOK —
(C1) the improvement loop now has a **committed keyless receipt**
(`OPTIMIZER_PROOF.md`) plus the unchanged BYOK real-judge path; (C2) an external
**τ-bench-style** harness runs AURA's real tool loop, self-tests 5/5 keylessly,
and has a BYOK path to a real τ-bench score; (C3) the judge gained a
**position-bias swap control** and a **calibration harness** without disturbing
any existing number. What stays BYOK, stated plainly: a real-model optimizer
receipt (`pip install "aura-agents[deepeval]"` + a key), a real τ-bench
leaderboard number (dataset export + `AURA_LLM=anthropic`), and a human-label
judge calibration set. The curator review's other findings — the single-node
SQLite/streaming ceiling (C6), the live tool-result injection gap (C4), the
cross-tenant `require_approval` receipt (C5), and the adoption/DX drift (C8/C9)
— are named here as the candidate Wave 9/10 backlog, not yet scheduled.

---

## Wave 9 — Harden the trust boundary (Sprints #107–#109)
Planned 2026-07-12 (aura-sprint-planner) from the CRITICAL/HIGH security cluster
of the five-lab curator review — chosen over the scale (C6) and DX (C8/C9)
clusters because each item is a live, self-contained, keyless-testable
trust-boundary hole, and each was re-verified at its exact cited line before
scheduling. **Exit criteria:** three receipts — (a) a tool/retrieval result
carrying an injected instruction is screened before it re-enters the model
context; (b) a `require_approval` bound is discharged ONLY by a tenant- and
invocation-matching approval; (c) a Windows freeform-tool run is memory-capped
or fail-closed, and the `str.format` class-graph gadget is rejected. No new
dependency — every fix hardens code AURA already owns.

- ✅ **Screen tool-result content before it re-enters the model** —
  `aura/safety/guardrails.py` gains `screen_tool_result`, called in
  `aura/agents/tool_agent.py` where a tool's `out_str` becomes a `tool_result`
  (C4). User input (`scan_input`) and A2A results (`delegate_verified`) were
  already screened; the primary agentic tool loop — MCP output, fetched
  pages/files, DB rows — was the last unscreened path into the prompt, the
  OWASP-LLM01 indirect-injection channel. Default `AURA_TOOL_RESULT_SCREEN=annotate`
  **wraps** flagged output in a clearly-fenced, non-executable "untrusted tool
  data" banner (the model still sees the content but is told to treat it as
  inert) — annotate, don't drop; `strict` redacts to a flag summary; `off`
  restores pre-#107 behavior byte-for-byte. The **tool trail keeps the raw
  output** (the true record); only the model-facing copy is annotated, and a
  `tool.result_screened` audit event fires when screening acts. The clean path
  is byte-identical to before (test-proven), so golden/gate numbers don't move.
  Gate: **758 → 771 tests**. *(Sprint #107 — done. Wave 9 opens.)*
- ✅ **Bind `require_approval` to tenant + invocation** — the safety "receipt"
  and the enforcement no longer disagree (C5). `ApprovalStore.approved_for`
  gains OPTIONAL `tenant` / `arguments` / `within_seconds` filters
  (**backward-compatible** — the no-kwargs call is unchanged), and
  `verifier_formal.verify_trace` now resolves the trace's tenant and each
  require_approval call's actual arguments and discharges the bound ONLY with a
  tenant- and invocation-matching, non-stale approval. Before #108, one approval
  authorised a tool forever, across every future run and every tenant, and a
  tenant-A approval satisfied the verifier for tenant-B; the per-call runtime
  guard was strict but the verifier that claims to catch guard-bypasses accepted
  stale/foreign approvals. Args are compared as parsed dicts (key-order/whitespace
  independent); a tool span with no captured input falls back to tenant binding
  (preserving the pre-#108 authorised-execution contract). Grounding:
  least-privilege binding of a human authorization to actor+action+time.
  **Found en route (fixed here):** the run exposed a latent non-determinism in
  the trace store — sibling spans share a coarse `started_at` (Windows ~15ms
  ticks), so `explain_trace` could pick a nested span's task as the run
  objective when an unstable tie flipped span order; `explain` now keys the
  objective off the **root** orchestration span (parent structure, not list
  position), and `save_span`'s deliberate REPLACE-bumps-rowid contract (the #33
  live-feed re-emit) is documented so future readers don't "fix" it. Gate:
  **771 → 780 tests**. *(Sprint #108 — done.)*
- ✅ **Close the Windows sandbox gap + the `str.format` AST bypass** —
  `aura/tools/sandbox.py`, two fail-closed hardenings, no new dependency (C7).
  **(1) AST screen (W2):** `validate_source` now rejects `.format`/`.format_map`
  attribute calls AND any string literal containing a dunder (`__\w+__`), closing
  the `"{0.__class__.__bases__}".format(())` class-graph-walk gadget that slipped
  past the node-level screen (the dunder hid inside a string, not an
  `ast.Attribute`). **(2) Windows memory cap (W3):** `resource` rlimits are
  POSIX-only and were silently swallowed on Windows, leaving only the 5s wall
  clock — a pure-Python alloc-bomb passed the gate and could exhaust host RAM.
  The subprocess tier now applies a best-effort per-process cap on Windows via a
  **Job Object** (pure `ctypes`, `ProcessMemoryLimit`), assigned before the
  child reads stdin (i.e. before the body runs); if NO cap can be established
  (`_mem_cap_available()` false), `run_sandboxed` **refuses** rather than run
  uncapped — the operator must set `AURA_SANDBOX_ALLOW_UNCAPPED=1` to override,
  never a silent uncapped run. The POSIX rlimit path is unchanged (it IS a cap);
  the wasm tier (#103) is unaffected (Pyodide heap); Gate 1's AST screen still
  runs first. Grounding: the documented CPython `str.format` escape primitive;
  capability-first isolation. Named honesty: the Job Object cap is best-effort
  and pragma-no-cover (the real Windows path can't be alloc-bombed safely in the
  keyless suite — the fail-closed *decision* is what's tested, via a monkeypatched
  probe). Gate: **780 → 791 tests** (#70/#103 suites untouched and green).
  *(Sprint #109 — done. **Wave 9 is complete.**)*

**Wave 9 exit criteria, assessed at #109:** all three CRITICAL/HIGH security
findings closed and test-proven keylessly — (C4) tool/retrieval output is
screened before it re-enters the model context, the last unscreened injection
channel; (C5) `require_approval` is discharged only by a tenant- and
invocation-matching approval, so enforcement and verification finally agree;
(C7) the `str.format` class-graph gadget is rejected and the Windows subprocess
tier is memory-capped or fail-closed. The remaining curator-review findings are
the named backlog, sequenced in `AURA_SPRINT_PLAN_2026-07.md` (the single forward
plan): **Wave 10 (act in the world, and persist)** — #110–#115: one canonical
test-count to end the C8 drift, then a persistent task queue/scheduler, screened
outbound notifications, opt-in approval-gated connectors, rule-based monitoring, and
a PyPI release + `examples/` gallery + docs (C8/C9 adoption). **Wave 11 (scale)** —
C6's single-node ceiling (SQLite WAL / Postgres backend, async-first serving, true
streaming through the tool loop). Also open, larger design: per-agent identity/authz
and egress controls once inside a run.

---

## Wave 10 — "Act in the world, and persist" (complete ✅ 2026-07-15)
*Full plan: `AURA_SPRINT_PLAN_2026-07.md`. Sprints #110–#115.*
- ✅ **One canonical, computed test count** — closes C8's measurement half. The
  count had drifted (617/689/758/785/791) and the README hardcoded `689 passed`
  while the suite really had 784. `aura/evaluation/test_inventory.py` now *computes*
  the count for the current host (isolated-tempdir `pytest --collect-only`, keyless)
  and documents why it's host-dependent (POSIX rlimit vs. Windows Job Object vs.
  WASM tier); a guard test blocks a stale hardcoded total from returning to the
  README quickstart. Gate: **784 → 787** (Windows). *(Sprint #110 — done. Wave 10 opens.)*
- ✅ **Persistent task queue + scheduler** — turns AURA into closed-loop automation
  on the #57 durable-engine + #7 checkpoint foundations. `aura/tasks/engine.py` +
  SQLite queue behind `AURA_TASKS=enabled`/`AURA_TASKS_DB`: cron scheduling (pure
  `aura/tasks/cron.py`, Vixie dom/dow rule), dependencies, retries, crash-resume via
  the CheckpointStore, and a **mandatory no-bypass guard chain** on every run
  (guardrail screen of params + #108 tenant/arg/**time**-bound approval — a stale or
  foreign approval no longer authorizes). `cancel()` is a real stop (a terminal task
  won't re-execute), a failed cron task recovers on its next beat, and each run is a
  `task` span for the Theater timeline. New CLI `aura task schedule|list|status|run`.
  Hardened after an adversarial review (cancel-bypass, forever-approval, cron-dep
  deadlock, determinism all fixed). Gate: **787 → 835** (Windows; +48: cron 28,
  engine 19, CLI 1). *(Sprint #111 — done.)*
- ✅ **Screened outbound notifications** — closes the outbound-action gap and
  completes the loop the scheduler opens. `aura/comms/notifications.py`:
  `notify(template, recipients, context)` sends templated email (SMTP) / Slack /
  Teams / webhook over #52's hmac transport. **Deliberately relaxes #52's "no
  content leaves the box"**, so the guards are load-bearing: (a) body AND subject
  are screened before egress, **default `strict`** (flagged content redacted, not
  just bannered) via its own `AURA_COMMS_SCREEN` switch; (b) a **content-egress
  approval class** `comms:content_egress`, distinct from #52's metadata webhook,
  gates **bulk sends** — tenant/argument/**time**-bound (#108), bound to the actual
  content inputs so an approval can't be replayed with different content; (c) a
  safe `{field}` renderer (not `str.format`, no #109 gadget); (d) emit-never-raises.
  A `builtin:notify` task wires it into #111 (schedule → guarded run → screened
  notify, test-proven end-to-end). Hardened after adversarial review (annotate-ships-
  raw, unscreened subject, unbound approval, never-raise env all fixed). Gate:
  **835 → 852** (Windows; +17). *(Sprint #112 — done.)*
- ✅ **File/document + calendar connectors** — AURA's first cloud/OAuth surface,
  trust-posture-first. `aura/connectors/`: a `Connector` interface (storage +
  calendar), a keyless `LocalFSConnector` (path-traversal fenced), and BYOK
  `OAuthCloudStorage`/`OAuthCalendar` (no cloud SDK dependency; real API bodies are
  pragma). **Untrusted by default is an enforced invariant**: every public op
  (`read`/`list`/`list_events`/`create_event`) passes `Connector._gate` —
  `AURA_CONNECTORS_ALLOW` allow-list (the #47/CVE-2025-6514 discipline) + fail-closed
  when the OAuth token is absent — so a directly-held instance can't route around
  it. Calendar writes are approval-gated (`connector:calendar_write`, tenant/arg/time
  bound) and the approval is **consumed** (one click = one write, no replay). A
  `builtin:connector_read` task makes ingestion schedulable via #111. `extract_text`
  reuses the ingest xlsx path. Keyless gate green with all connectors absent.
  Hardened after adversarial review (allow-list-as-invariant, write-path gate,
  one-time approval, arg-normalization, None-filename all fixed). Gate: **852 → 866**
  (Windows; +14). *(Sprint #113 — done.)*
- ✅ **Workflow templates + rule-based monitoring** — `aura/workflows/templates.py`:
  reusable parameterized DSL specs ({param} placeholders, safe non-str.format
  substitution) compiled onto the #57 durable engine via the existing
  `compile_workflow`; a missing param fails loudly at compile time.
  `aura/tasks/monitoring.py`: `TaskMonitor` raises **rule-based** anomaly flags over
  the #111 run history — run duration, recent failure rate, missed (overdue)
  schedule — plus a `status_panel` data contract for the dashboard/Theater. Rules-only
  by design; skill-based/learned anomaly detection is deferred and named. Deterministic
  & keyless (pure functions, caller clock). Hardened after review (failure_window=0
  disable, div-by-zero guard, deterministic last-run ordering by rowid). Gate:
  **866 → 876** (Windows; +10). *(Sprint #114 — done.)*
- ✅ **Wave 10 release + make it adoptable** — bumped to **v2.1.0**
  (`aura.__version__`), CHANGELOG `[2.1.0]` with the honest measured deltas (count
  now canonical per #110, golden-set unchanged), a PyPI **trusted-publishing (OIDC)**
  release workflow (`.github/workflows/release.yml` — no stored token; tagging is the
  operator's action), and an `examples/` gallery whose keyless end-to-end demo
  (`wave10_end_to_end.py`: schedule → guarded run → screened notify → monitor panel)
  runs offline and is smoke-tested. Gate: **876 → 881** (Windows; +5).
  *(Sprint #115 — done. **Wave 10 is complete.**)*

**Wave 10 exit criteria, assessed at #115:** a keyless, replayable
`schedule → sandboxed/guarded run → screened notify → task timeline` loop is
test-proven (`examples/wave10_end_to_end.py` + `tests/workflows/`), the test count is
one canonical computed value (C8 closed), and AURA is PyPI-publishable via OIDC — all
with **zero golden-set regression** (workflow tests live in their own suite, never the
accuracy scoreboard).

---

## Wave 11 — "Scale & emergence" (in progress)
*The scale wave (C6), demoted here from the old "Wave 10 = scale" per the
2026-07-14 reconciliation. Goal: lift the single-node concurrency ceiling without a
new hard dependency or breaking the keyless gate. Exit: N concurrent sessions with
no `SQLITE_BUSY`; a measured time-to-first-token. Postgres/async streaming and the
UNVERIFIED edge-swarm items stay later + labelled.*
- ✅ **SQLite WAL + busy-timeout concurrency foundation** — the cheapest, keyless,
  zero-new-dependency lift of the C6 ceiling. All **12** stores had opened a bare
  `sqlite3.connect` with the default rollback journal and no busy timeout, so two
  sessions writing at once hit `SQLITE_BUSY`. New `aura/util/sqlite.connect` (the one
  place AURA opens SQLite) sets `journal_mode=WAL` + `busy_timeout`
  (`AURA_SQLITE_BUSY_TIMEOUT_MS`, default 5000) + `synchronous=NORMAL`, skipping WAL
  only for `:memory:`; all 12 sites migrated to it (a 3-agent disjoint-file swarm). A
  concurrency test proves 6 parallel writers × 25 commits land with zero
  `SQLITE_BUSY`. Side benefit: the keyless suite got **~3× faster** (WAL+NORMAL). No
  Postgres — that stays a later opt-in behind a flag (a hard networked dep would break
  the keyless gate). Hardened after adversarial review (`mode=memory` path
  false-positive, negative-timeout clamp, WAL sidecar cleanup in conftest, test
  connection-close + WAL-tied assertion). Gate: **881 → 889** (Windows; +8).
  *(Sprint #116 — done. Wave 11 opens.)*
- ✅ **True token streaming through the tool loop** — closes the other half of C6's
  serving gap. Before: `/v1/chat` ran the tool loop but only *re-chunked* a finished
  string (fake streaming), and `/v1/chat_tokens` streamed real tokens but *skipped
  the tool loop*. New `KnowledgeCopilot.stream_agentic` + `/v1/chat_agentic` (SSE) run
  the full agentic loop (tools, guards, audit, trail) and then stream the final
  grounded synthesis token-by-token — real low-latency tokens AND tool use. Trust
  parity with `answer`: tool output is re-screened via `screen_tool_result` (C4/#107)
  before it grounds the synthesis (the trail is raw by design), the loop phase runs
  under a per-copilot lock so a concurrent request can't clobber the shared
  `last_trail`, and `AURA_VERIFY` is honored as a trailing caveat. Documented
  divergence: the `AURA_PLANNER` deliberative path is not applied to the streamed
  route. Hardened after adversarial review (raw-trail injection bypass, concurrency
  race, planner/verify honesty all fixed). Gate: **889 → 894** (Windows; +5).
  *(Sprint #117 — done.)*
- ✅ **Concurrency benchmark (the Wave 11 receipt)** — `benchmarks/concurrency.py`
  measures what #116 was for: **N concurrent sessions writing to one store with zero
  `SQLITE_BUSY`**, plus throughput, against a bare rollback-journal control (busy
  handler off) that still exhibits the failure mode — so the receipt proves a delta,
  not just a green number (measured: WAL landed 400/400 concurrent writes, 0 errors;
  control dropped writes with SQLITE_BUSY). Keyless, reproducible
  (`--write` → `benchmarks/CONCURRENCY.md`). Honest scope: storage concurrency only —
  time-to-first-token needs a live BYOK model to be a real number, so it's stated as
  future work, not a stub figure dressed up as a measurement. Gate: **894 → 896**
  (Windows; +2). *(Sprint #118 — done.)*
- ✅ **Per-agent capability enforcement (identity-scoped authz)** — closes the
  long-standing "no per-agent authz once inside a run" flaw from the security review.
  A `ToolAgent` scoped via `tool_names` only limited which tools the model *saw*; the
  runtime authz guard was applied only when the *caller* passed `on_tool_call`, and
  `Supervisor.handle` (plus `parallel_supervisor` and the distributed worker) delegates
  via `worker.run(...)` with **no guard** — so a delegated worker's tool calls ran
  unenforced. Now every `ToolAgent` **self-enforces its own capability scope** on every
  run (`_capability_guard`, composed *before* any caller guard), so a delegated worker
  can never call a tool outside its grant even un-guarded by the caller; blocks are
  audited with the agent identity, and a buggy caller guard fails **closed** (blocked)
  rather than crashing the run. Unscoped agents (`tool_names=None`, incl. the copilot's)
  stay unrestricted — clean path byte-unchanged. Adversarial review confirmed no
  tool-executes-without-enforcement path across all three delegation sites. **Egress
  control (which external endpoints an agent may reach) remains the open half of this
  item.** Gate: **896 → 902** (Windows; +6). *(Sprint #119 — done.)*
- ✅ **Central egress allow-list** — closes the **egress half** of the security item
  (#119 closed the authz half). `aura/safety/egress.py` `check_egress` enforces
  `AURA_EGRESS_ALLOW` — **opt-in** (unset ⇒ no-op, keyless suite unchanged),
  **deny-by-default once set**, exact host or `*.sub` wildcard (sub-domains only, not
  the apex or a look-alike), audited. Wired into the content-egress paths (#52
  webhooks `emit`, #112 notifications) *before* the network call and before any
  injected transport, so an injectable transport can't be an exfil bypass. Host
  parsing **fails closed** on checker/client-split tricks (`evil.com\@allowed.com`,
  userinfo confusion, malformed IPv6, junk authorities). Hardened after adversarial
  review (backslash-confusion, transport-bypass gap, malformed-IPv6 all fixed).
  Gate: **902 → 913** (Windows; +11). *(Sprint #120 — done.)*
- ✅ **Concurrency backpressure (graceful load shedding)** — bounds Wave 11's "N
  concurrent sessions" *gracefully*. #116 removed the SQLite write bottleneck, but an
  unbounded burst of heavy requests could still exhaust the threadpool/memory.
  `aura/server/limits.py` + a serving middleware cap in-flight requests at
  `AURA_MAX_CONCURRENCY`; over capacity returns **503 + Retry-After** instead of
  thrashing; liveness/readiness probes are exempt so they answer while shedding.
  **Opt-in** (unset/≤0 ⇒ unlimited, keyless suite + existing behavior unchanged),
  thread-safe compare-and-increment (no max+1 TOCTOU), per-request **token** so a
  slot is released only by the request that counted it (safe under live re-tuning).
  Honest limitation: for SSE endpoints the slot frees at admission, not stream-end.
  Hardened after adversarial review (live-reconfig slot-steal fixed). Gate:
  **913 → 919** (Windows; +6). *(Sprint #121 — done.)*
- ✅ **Exactly-once task execution (atomic claim)** — under concurrent ticks/workers
  (the multi-process world #116 enables), the #111 scheduler could **double-execute**
  a task: `run()` read status then set 'running' non-atomically, so two runners both
  passed the check. Now `run()` opens with an atomic compare-and-set claim
  (`UPDATE … SET status='running' WHERE id=? AND status IN ('scheduled','blocked')`)
  — only the winner (rowcount==1) executes; concurrent losers skip. Exactly-once
  across threads (engine lock) AND processes (SQLite atomic UPDATE + WAL). Crash
  recovery routes through the SAME primitive: `resume_interrupted` **requeues** a
  stale-`running` task to `scheduled` (opt-in `AURA_TASK_STALE_S` liveness gate so a
  live worker's task isn't grabbed) and re-runs it through the exclusive claim, so
  concurrent recovery can't double-run either. Proven with 4 racing threads (body
  runs once) + 3 racing recoverers. Hardened after adversarial review (the resume
  `running→running` non-exclusive claim, HIGH, fixed). Gate: **919 → 923** (Windows;
  +4). *(Sprint #122 — done.)*

---

## Wave 12 — "Zero Trust & Protocol Currency" (complete ✅ 2026-07-17)
- ✅ **Cryptographic per-agent identity + short-lived scoped credentials** — completes
  the #119/#120 authz+egress work to Anthropic's Zero-Trust **Foundation tier**.
  `aura/safety/identity.py`: a unique-per-instance `AgentIdentity` (stdlib HMAC root,
  keyless; Ed25519 a documented opt-in) issues **short-lived, capability-scoped, signed
  credentials**. Behind `AURA_AGENT_IDENTITY=enabled`, a scoped ToolAgent mints one per
  run (TTL `AURA_AGENT_CRED_TTL`, default 300s) and `_capability_guard` verifies it on
  every tool call — **deny-by-default** on missing/expired/mis-scoped/tampered. Flag off
  ⇒ behaviourally identical to #119 (identity minted, never consulted). Honest scope: in
  one process this is a self-signed TTL+scope wrapper; the signature's full value is
  across a trust boundary (delegation/A2A) — future. Hardened after adversarial review
  (JSON canonical claim fixes a delimiter-injection signature collision; non-finite TTL
  clamped so it can't become never-expiring). Gate: **923 → 938** (Windows; +15).
  *(Sprint #123 — done. Wave 12 opens.)*
- ✅ **Memory integrity hashing + source attribution** — the "Integrity & Recovery"
  Zero-Trust pillar on top of #17's provenance. `aura/memory/integrity.py`: a content
  digest (HMAC-SHA256 when `AURA_MEMORY_INTEGRITY_KEY` is set, else SHA-256) is stamped
  at `remember()` and re-verified at every `recall()`; a memory whose text no longer
  matches its digest is **tampered** → quarantined + audited (`memory:integrity_failed`)
  regardless of confidence (OWASP ASI06 memory-poisoning). Opt-in
  (`AURA_MEMORY_INTEGRITY=enabled`; default byte-unchanged) with a `_STRICT` mode that
  also quarantines unstamped entries (closes the strip-to-downgrade bypass). Recall
  backfills past a quarantined item so results stay at k. Hardened after adversarial
  review (fail-CLOSED on verify error, guarded audit so detection can't crash recall,
  JSON-injective digest, key-rotation hazard documented). Gate: **938 → 947** (Windows;
  +9). *(Sprint #124 — done.)*
- ✅ **OWASP ASI-2026 self-audit + blast-radius replay gate** — `SECURITY_ASI2026.md`
  maps AURA's controls line-by-line to the official **ASI01–ASI10** (web-verified;
  honest verdicts: 4 Covered, 6 Partial, 0 Gap — supply-chain provenance and
  continuous red-team named as the material gaps). `aura/evaluation/blast_radius.py`
  implements the standard's cascading-failure mitigation: it **replays a recorded
  trace** (read-only over the tracing store — inherently isolated) and gates its
  **action volume** — tool calls, distinct leaf agents, LLM calls — against a declared
  `BlastRadiusCap`; a breach is flagged `replay:blast_radius_exceeded` (audited) and
  `gate()` **fails closed** (raises) on a breach *or* an unresolvable trace. Hardened
  after adversarial review (fail-closed on can't-evaluate; leaf-agent-only count so
  planner sub-steps don't inflate it; registration spans excluded; span-kind mapping
  verified against what the tracer actually emits). Gate: **947 → 957** (Windows; +10).
  *(Sprint #125 — done.)*
- ✅ **MCP 2026-07-28 protocol currency** — assessed AURA's MCP surface against the
  web-verified 2026-07-28 revision (RC; Linux Foundation / AAIF governance). AURA's MCP
  is **stdio-only** (SDK `ClientSession` + stdio JSON-RPC server), so the OAuth-hardening
  SEPs target a transport AURA doesn't have: `aura/tools/mcp_auth.py` ships the
  **spec-conformant primitives** — `validate_iss` (RFC 9207 / **SEP-2468**, exact-match,
  fail-closed) and `dcr_application_type` (**SEP-837**) — tested and honestly **dormant**
  (ready for a future HTTP+OAuth transport). The one change that targets AURA's actual
  surface — the **stateless core** (**SEP-2575/2567**) — is verified conformant (server
  needs no `initialize` handshake, holds no protocol/session state). `MCP_CURRENCY.md`
  records the full assessment; the **Tasks extension (SEP-2663)** is evaluated as
  complementary to #111 (a wire handoff, not a scheduler) and **not adopted**. Hardened
  after adversarial review (doc framing made precise: verifies a pre-existing property +
  adds primitives, no new server code; tool-layer state scoped out). Gate: **957 → 964**
  (Windows; +7). *(Sprint #126 — done. **Wave 12 is complete.**)*

**Wave 12 exit criteria, assessed at #126:** identity/credential/memory-integrity posture
matches Anthropic's Zero-Trust **Foundation tier** (test-proven, #123/#124); a scored
**OWASP ASI01–ASI10** self-audit exists with the cascading-failure gap closed by a
fail-closed blast-radius gate (#125); and the MCP surface is **current** with the
2026-07-28 revision (stateless-core conformant; OAuth primitives ready; #126) — all
keyless, zero golden-set regression.

---

## Wave 18 — "Determinism & Throughput" (in progress)
Frontier re-verified live 2026-08-01: record-replay for agents is an active 2026
thread (arXiv:2505.17716 record & replay, arXiv:2606.08275 causal agent replay,
arXiv:2606.14805 zero-replay debugging), and the motivating measurement is that
agent accuracy varies up to ~15% between runs even with deterministic settings.
Async concurrency is separately verified to give 36–84% more throughput than sync
Python agent frameworks for I/O-bound calls.
- ✅ **#151 — deterministic execution replay.** `aura/llm/record.py` +
  `aura/llm/replay.py` + `REPLAY.md`. AURA could already replay a run *visually*;
  now it can **re-execute** one. `AURA_LLM_RECORD=run.jsonl` wraps whichever
  backend is active (stub included — that is what makes the feature keyless) in a
  passthrough `RecordingClient` that appends one full-fidelity JSONL line per model
  call; `AURA_LLM=replay` + `AURA_REPLAY_BUNDLE` then re-runs the *same agent code*
  with those answers injected, so routing/planning/tools/memory/verification all
  execute for real. `ReplayClient` satisfies the same duck-type `StubClient` already
  proves sufficient, so breaker/retries/quotas/spans/cost apply unchanged.
  **A miss is the signal:** asking for a call the recording lacks raises
  `ReplayMiss` rather than substituting an answer — which turns any recorded run
  into a keyless regression test. `LLM._bypass_cache` is mandatory, not an
  optimisation: a cache hit skips `messages.create`, so recording and replay would
  otherwise disagree on call count and every replay would report a false miss.
  **Three bugs found by replaying a real two-model debate, all regression-tested:**
  (1) per-client sequence counters produced duplicate `seq` (`[0,1,2,3,0]`) and one
  manifest per LLM instance, so the load-time sort scrambled the order — fixed with
  a process-wide sequence + write-once-per-bundle manifest; (2) strict order is not
  reproducible for a fan-out *at all*, because which caller goes first depends on
  state outside the recording (the recording run had itself grown the memory the
  replay then read) — multi-caller bundles are now marked `concurrent` and matched
  by fingerprint rather than position, without weakening the miss contract; (3) the
  concurrent flag was only written by `close()`, which nothing ever called, so
  bundles claimed an order they did not have. **Also closed two pre-existing latent
  concurrency bugs** of the same class as the py3.12 `InterfaceError` already fixed
  in `TaskEngine`: unlocked reads on the shared SQLite connection in
  `tracing/store.py` (`list_traces`/`get_trace`) and `llm/cache.py`
  (`SemanticCache.get`, which additionally held a live cursor open across embedding
  work — the lock is now released *before* the cosine pass so concurrent callers are
  not serialised). Golden-set **N/A by design** — determinism is not accuracy.
  Gate: 1275 → 1293 (py3.11), 1297 (py3.10), 1297 (py3.12).
- ⬜ **#152 — async concurrency for throughput.** Specced (opt-in async wrapper +
  bounded-concurrency gather over the existing sync core; no architectural rewrite).
  Not yet built. Note: its fan-out is exactly what would have surfaced the two
  SQLite read races #151 just fixed, so the groundwork is in place.

## Wave 17 — "Memory 2.0" (complete ✅ 2026-07-19, released v2.7.0)
Closes the three SimpleMem stages (arXiv:2601.02553, `aiming-lab/SimpleMem` —
re-verified live 2026-07-19) that Sprint #39's consolidation left open. All
deterministic/CPU-only/no-train, each **off by default** (opt-in flag; unset ⇒
byte-identical behaviour, test-proven). Two new keyless golden categories.
Honest triage note: DSP speculative planning (arXiv:2509.01920) was evaluated
and **declined** — its learned RL router violates the no-training law; only the
draft→verify *pattern* remains a candidate for a future wave.
- ✅ **#147 — measured conversation compression (stage 1).**
  `aura/context/compressor.py` gains `compression_fidelity` — a deterministic
  keyless meter (salient-term coverage: names/numbers/domain nouns surviving
  into the summary); every `compress()` records ratio + fidelity on its span.
  Opt-in fail-safe `AURA_CONTEXT_MIN_FIDELITY` keeps the originals rather than
  ship a summary that dropped facts. Golden category **context compression
  fidelity** (3 fixtures: faithful high, vague detected, ordered) wired into
  Gate 3's exit + flip-gate. Gate 1241→1250 (py3.11) / 1254 (py3.12).
- ✅ **#148 — Online Semantic Synthesis (stage 2).** A near-duplicate cluster's
  representative can be a **synthesised unified abstract** (BYOK utility model,
  injectable for tests) behind `AURA_MEM_SYNTHESIZE=enabled`, **fidelity-guarded**
  by #147's meter (`AURA_MEM_SYNTHESIZE_MIN_FIDELITY`, default 0.8): low
  fidelity / error / empty reply / flag-off all degrade to #39's longest-text
  representative — synthesis can never do worse than #39. Synthesised reps are
  re-embedded; `restore_namespace` still undoes a pass exactly; golden memory
  recall held 4/4. Gate →1256 / 1260.
- ✅ **#149 — Intent-Aware Retrieval Scope (stage 3).**
  `aura/memory/retrieval_scope.py`: deterministic classifier (narrow =
  id/wh-factoid lookup; broad = enumeration/aggregate/overview; else default)
  sizes recall `k`, bounded by `AURA_RETRIEVAL_SCOPE_MAX` (10). Consulted by
  `LongTermMemory.recall` only when `AURA_RETRIEVAL_SCOPE=intent` (intent +
  scope_k recorded on the span). Golden category **intent-aware retrieval
  scope** (6 fixtures). Gate →1272 / 1276.
- ✅ **#150 — receipt + v2.7.0 release.** `RESULTS_MEMORY.md` — live keyless
  numbers (fidelity 0.609 vs 0.0; narrow k=2 vs broad k=8; ratio 0.6) guarded
  by `tests/test_memory_receipt.py` so the doc can't drift; honest not-done
  section (meter proves the mechanism, real-model summary quality is BYOK; the
  intent classifier is a heuristic, not a learned model). `__version__` 2.7.0,
  server.json synced, CHANGELOG `[2.7.0]`, README badge/stat band. Operator
  must `git tag v2.7.0` to publish.

## Wave 15 — "Brain 2.0: Formal Ontology Grounding" (complete ✅ 2026-07-18)
- ✅ **OWL/RDF export layer** — the formal semantic layer the ontology never had. The existing
  `OntologyProposal` is a Pydantic **shape** check; `aura/knowledge/ontology_rdf.py` now serializes the
  runtime ontology (entity types + their properties + typed relations) to a real **OWL/RDF graph** in
  Turtle — `owl:Class` per entity, `owl:DatatypeProperty` per property, `owl:ObjectProperty` with
  `rdfs:domain`/`rdfs:range` per relation, under an `aura:` namespace. The emitter is **pure standard
  library** (deterministic Turtle needs no triple-store), so the keyless gate stays zero-dependency;
  **rdflib** (the new `[ontology]` extra, BSD, added to CI) is used only to *validate* the output
  round-trips as well-formed RDF (29 triples on the example, confirmed). **Honest (Law 1):** AURA's
  ontology is a *domain* schema, so it uses standard **W3C RDFS/OWL** vocabulary — AgentO/AIAO are the
  inspiration for modeling *agentic systems*, and aligning a domain schema to AgentO's `Agent`/`Task`
  classes would be mislabeling, so it isn't done. Read-only (ingest/retrieval byte-unchanged); a
  read-only `ontology_rdf` tool is registered behind `AURA_ONTOLOGY_RDF=enabled`. Dependency root for
  the SPARQL competency questions (#138) and SHACL validation (#139). Gate: **1212 → 1218 (py3.11) /
  1222 (py3.12)** (+10), golden-set unchanged. *AgentO (ESWC 2026) / AIAO v2.0 / rdflib (BSD) — verified
  2026-07-18.* *(Sprint #137 — done. Wave 15 opens.)*
- ✅ **Competency questions + SPARQL surface** — proving the ontology is *useful*, not just present.
  `aura/knowledge/competency.py` ships **7 domain-agnostic competency questions** (the semantic-web
  practice of interrogating an ontology with real questions) as parameterized SPARQL over #137's RDF —
  classes, typed relations + their domain/range, per-class attributes, one-hop reachability, relations
  into a class, and orphan classes. A `competency_report` runs the whole battery and scores it pass/fail
  (**7/7 on the example ontology**) — the "prove it, don't claim it" discipline applied to the ontology
  for the first time (a dedicated ontology receipt, distinct from the accuracy golden-set). Exposed as a
  read-only `ontology_query` tool. Class parameters are sanitized to safe IRI local names, so a
  malicious value can't inject SPARQL — it just matches nothing. Needs `rdflib` (the `[ontology]`
  extra); tests importorskip it. *AIAO v2.0's competency-questions-with-example-SPARQL methodology,
  verified 2026-07-18.* Gate: **1218 → 1228 (py3.11) / 1232 (py3.12)** (+10), golden-set unchanged.
  *(Sprint #138 — done.)*
- ✅ **SHACL shape validation** — the line between *structurally* valid (Pydantic shape) and
  *semantically* valid (a coherent ontology): the actual neurosymbolic constraint. `aura/knowledge/
  shacl.py` runs pySHACL over #137's RDF **alongside** (never instead of) the Pydantic check. Kept
  **domain-agnostic** per the general-purpose direction: the built-in shapes validate **ontology
  integrity** — every relation's `rdfs:domain`/`rdfs:range` must be a *declared* `owl:Class` (this
  catches a **dangling relation** whose range references a class the synthesizer never defined — a real
  failure mode), and every class must carry a label. **Domain** semantics stay in the operator's config:
  an `AURA_ONTOLOGY_SHAPES` Turtle file is validated alongside the built-ins (no domain-specific rules
  in core). The acceptance hook in `run_ingest` is **fail-open** behind `AURA_ONTOLOGY_SHACL=enabled`
  (a violation is flagged + audited `ontology:shacl_violation`, never blocks) — `strict` blocks; off by
  default ⇒ the ingest path is byte-unchanged (golden-set 5/5 confirmed). Read-only `ontology_validate`
  tool. Seeded good/bad proposals are the *ontology-validity* receipt (0 false-accepts: a valid ontology
  conforms, a dangling one is caught). Needs pySHACL (the `[ontology]` extra); tests importorskip it.
  *pySHACL (Apache-2.0, RDFLib) + arXiv:2604.00555's neurosymbolic-constraint pattern, verified
  2026-07-18.* Gate: **1228 → 1238 (py3.11) / 1242 (py3.12)** (+10), golden-set unchanged.
  *(Sprint #139 — done.)*
- ✅ **Brain 2.0 receipt (`RESULTS_ONTOLOGY.md`)** — closes the wave the way #125/#131/#140 close
  theirs. A scored, keyless-reproducible self-audit over the example ontology: **29 RDF triples**
  (rdflib round-trip ok), **competency 7/7**, **SHACL** valid-conforms + a seeded dangling relation
  caught (0 false-accepts). States the honest scope plainly — standard RDFS/OWL vocabulary (AgentO/AIAO
  are the *methodology* inspiration, not a schema the domain terms pretend to instantiate), and the
  named non-goals (no OWL DL reasoner → no JVM; no external triple-store; domain constraints are the
  operator's `AURA_ONTOLOGY_SHAPES`, not core). Linked from README; `test_ontology_receipt.py` keeps the
  doc's headline numbers matching a live run so it can't silently drift. Doc only; test count +3.
  *(Sprint #140 — done. **Wave 15 complete.**)*
- ✅ **Wave 15 release `v2.6.0`** — Brain 2.0 shipped *after* the v2.5.0 tag (#145 covered Wave 16
  only), so it gets its own release. `aura/__init__.py` → `2.6.0`; CHANGELOG `[2.6.0]` (#137–#140);
  `server.json` synced (the #143 CI check enforces it); README badge + stat band. Operator action: tag
  `v2.6.0` to publish. *(Sprint #146 — done.)*

---

## Wave 16 — "Agent Tooling & MCP Ecosystem" (complete ✅ 2026-07-18, released v2.5.0)
- ✅ **Code-interpreter tool (`code_exec`)** — computation as a first-class, general-purpose agent
  capability. `aura/tools/code_tool.py` wires AURA's existing sandbox (`aura/tools/sandbox.py`,
  #70/#103/#109) into the tool loop: an agent writes Python, it runs in a fresh isolated process under
  mem/CPU/wall-clock caps, and gets stdout + the last expression's value back — deterministic
  arithmetic, statistics, JSON/CSV parsing & transformation, sorting/aggregation, regex, date math,
  offloaded from the model. The wrapper inlines the snippet into the sandbox's `tool()` body (a custom
  `print` captures output; an AST last-expression rewrite captures the result), runs under a **curated
  pure-computation import allow-list** (a new `allowed` override on `run_sandboxed`; no os/sys/io/socket/
  subprocess) with `open`/`eval`/`exec`/`getattr` AST-banned. Output is screened (#107) before it
  re-enters the model, the #119 guard scopes which agents may call it, and every call is audited
  (`tool:code_exec`). Off by default (`AURA_CODE_TOOL=enabled`); no new dependency. **Security: an
  adversarial review found a CRITICAL pre-existing escape** in the shared sandbox — a bare
  `__builtins__` *name* (a dunder the AST screen only checked as an *attribute*) let
  `__builtins__['eval']`/`['open']` reach full host RCE. Closed with five layers: (1) the AST screen now
  rejects dunder `Name` ids, (2) the runner execs with a **restricted `__builtins__`** (drops
  eval/exec/open/getattr… keeps `__import__`/`__name__`/`__build_class__` so imports+classes still work),
  (3) `.mro`/`.mro_entries` banned as class-traversal gadgets, atop the existing (4) dunder-attribute and
  (5) import-allow-list gates — a genuine hardening of the whole #70/#103 sandbox, verified with a full
  escape battery. Gate: **1147 → 1176 (py3.11) / 1180 (py3.12)** (+29), golden-set unchanged; tested on
  both Python versions to keep CI green. *PAL (arXiv:2211.10435) / Program of Thoughts (arXiv:2211.12588)
  / ReAct (arXiv:2210.03629).* *(Sprint #141 — done. Wave 16 opens.)*
- ✅ **Web-research tools (`web_search` + `web_fetch`)** — external knowledge for agents, gated hard.
  `aura/tools/web_tools.py`: `web_search(query)` (ranked title/url/snippet) and `web_fetch(url)` (HTTP
  GET → readability text, **no JS**). **Two gates, both required:** off by default (`AURA_WEB_TOOLS`)
  AND a populated #120 egress allow-list — every request passes `check_egress` (deny-by-default), so a
  misconfig can't open the network. Fetched content + search results are #107-screened as untrusted
  **data** before reaching the model; every call audited. Provider-pluggable + honest: the free no-key
  **`ddgs`** (MIT, verified 2026-07-18) default is a *fragile scraper* (BYOK Brave/Tavily is the
  production path, wired by injecting a `searcher`); the network layer is injectable, so the keyless
  gate makes **no live calls** (#113 discipline). **Security review found + I closed** an
  **SSRF-via-redirect** (HIGH): urllib followed a 302 from an allowed host to an internal/metadata host
  without re-checking — now every redirect hop re-runs `check_egress` (verified with a real
  local-server redirect-to-localhost test); and the search `title`/`url`, not just the snippet, are
  screened. `host_of` fail-closed behaviour (userinfo/IDN-homograph/suffix-confusion) re-confirmed
  solid. Gate: **1176 → 1192 (py3.11) / 1196 (py3.12)** (+16), golden-set unchanged. *ReAct
  (arXiv:2210.03629) / Toolformer (arXiv:2302.04761); ddgs MIT.* *(Sprint #142 — done.)*
- ✅ **MCP-registry publish scaffold** — makes AURA's already-built MCP server **discoverable**.
  A schema-valid [`server.json`](server.json) (schema `2025-12-11`, name
  `io.github.rubansivanandam/aura` — the GitHub-OAuth namespace, chosen to sidestep the indexed
  `mezmo/aura` collision — a PyPI `aura-agents` package entry with `runtimeHint: uvx` + `transport:
  stdio`), a clean `aura-mcp` console entrypoint (`pyproject.toml`), a repeatable
  `scripts/publish_mcp_registry.sh` (preflight: valid JSON + version-sync, then `mcp-publisher login
  github && publish`), an honest [`MCP_REGISTRY.md`](MCP_REGISTRY.md), and a **CI check**
  (`test_mcp_registry.py`) that fails the gate if `server.json`'s version (top-level or any
  `packages[].version`) drifts from `aura.__version__` — a release bump can never silently desync the
  manifest. **Honest scope:** the two *publish* steps — publishing `aura-agents` to PyPI, and running
  `mcp-publisher` under your GitHub OAuth — are **operator actions**, not code, and are stated as such
  (findability precedes any "trending" goal; the repo isn't indexed yet). Gate: **1192 → 1198 (py3.11)
  / 1202 (py3.12)** (+6), golden-set unchanged. *Official MCP Registry (verified 2026-07-18).*
  *(Sprint #143 — done.)*
- ✅ **Vetted external-MCP-tool allow-list — host-scoping for remote servers.** AURA already consumed
  external MCP servers under an *enforced* trust policy (#38: namespace + per-tool allow-list, stdio
  binary **sha256** pin, full audit). #144 closes the remaining gap — **remote (HTTP) MCP servers** —
  by host-scoping them through the #120 egress allow-list: a trust entry with a `host` (or a
  `RemoteMCPTransport` that exposes one) must have that host on `AURA_EGRESS_ALLOW`, **fail-closed** (a
  remote server with the egress policy unset is refused — a remote tool source with no host control is
  exactly the CVE-2025-6514 supply-chain risk). New `RemoteMCPTransport` (Streamable-HTTP, lazy `mcp`
  SDK, not in the gate); a vetted reference config (`examples/mcp_trust.example.json`) with the two
  general entries — **`microsoft/playwright-mcp`** (stdio, sha256-pinned; Apache-2.0, verified) and
  **GitHub's official remote server** (`api.githubcopilot.com`, host-scoped, fine-grained PAT) — and an
  honest [`MCP_TOOLS.md`](MCP_TOOLS.md). A SQL/MSSQL MCP is **explicitly declined for now** (several
  community servers default to *write* access — the #125 incident class; read-only-against-a-replica
  only). Domain-agnostic: one mechanism, no hard-wired domain tools. Gate: **1198 → 1208 (py3.11) /
  1212 (py3.12)** (+10), golden-set unchanged. *playwright-mcp / GitHub MCP (verified 2026-07-18).*
  *(Sprint #144 — done.)*

---

## Wave 14 — "Multi-agent orchestration & autonomous research" (complete ✅ 2026-07-18)
- ✅ **Multi-agent topology selector (`SwarmExecutor`)** — one first-class, typed call to
  run a task under a chosen topology — **hierarchical / pipeline / debate / market** —
  behind `AURA_SWARM=enabled` (refuses to run when off, so no default path is touched).
  `aura/orchestration/swarm_executor.py`: `SwarmExecutor(registry, bounds, order)` unifies
  the previously-separate patterns (Supervisor routing #31, the debate panel, plus thin
  pipeline/market strategies) and — the load-bearing part — **enforces #129's swarm-skill
  BOUNDS at execution**: `build_swarm_from_skill` scopes each role agent to
  `bounds.allowed_tools` (fail-closed — no declared grant ⇒ **no** tools, never #119's
  allow-all), applies the allowed-tools guard at dispatch, caps participation by
  `max_agents`, sets each role's `max_steps`, and follows the skill's `workflow` order.
  That closes the loop #129 named explicitly (a portable skill's real containment is the
  executor honouring its tool grant). **Honest (Law 1):** "topology layer" is AURA's own
  abstraction, not an industry standard; "market" is a deterministic best-bid *selector*,
  not an auction; and `debate` is a fixed panel whose receipt states plainly that skill
  bounds do **not** govern it (no laundering an unenforced run as enforced). Hardened after
  adversarial review (fail-closed no-grant; workflow order honoured; duplicate role names
  rejected by #129's validator; hierarchical routes to the best match *within* the cap;
  topology-accurate `bounds_enforced` receipt; `max_steps` labelled per-agent). Gate:
  **1071 → 1090** (Windows; +19), golden-set unchanged.
  *2026 practice: LangGraph/CrewAI/AutoGen/ADK.* *(Sprint #132 — done. Wave 14 opens.)*
- ✅ **Multi-model benchmark suite (BYOK leaderboard)** — generalizes #105's τ-bench-style
  harness into an **adapter framework** over four published, open agent benchmarks:
  **τ²-bench** (Sierra, MIT, arXiv:2506.07982), **GAIA** (Meta/HF, CC-BY-4.0,
  arXiv:2311.12983), **SWE-bench** (Princeton, MIT, arXiv:2310.06770), **BFCL** (Berkeley
  Gorilla, Apache-2.0) — all web-verified this session. `aura/evaluation/benchmarks/`:
  `base.py` (`BenchmarkCase`/`ScoreResult`/`SuiteAdapter` contract + a fault-isolating
  `registry`), one adapter per suite mapping the suite's NATIVE task format + implementing
  its REAL scorer (τ²=state-subset+required-calls, GAIA=quasi-exact-match normalization,
  SWE-bench=FAIL_TO_PASS/PASS_TO_PASS resolution, BFCL=AST arg-value match incl. the
  relevance/abstention category), and `leaderboard.py` rendering `RESULTS_BENCHMARKS.md`.
  Built by a **4-agent swarm** (one adapter each, web-verifying its suite) with me as
  integrator. **Honest (Law 1) — enforced, not just claimed:** the *harness* is CPU-only +
  keyless (the gate is each adapter's deterministic oracle self-testing its own scorer); a
  real multi-model leaderboard is **BYOK** (supply the dataset + a per-suite
  `solver(case)->response`); **SWE-bench is BYOK-only** (needs a container to run repo
  tests) and is reported `skipped`, never a fabricated score. Hardened after adversarial
  review that found real **vacuous-pass** holes: an empty τ²/SWE-bench reference no longer
  "resolves" a do-nothing response; the leaderboard gate now fails on a 0-case suite or a
  silently-dropped adapter; GAIA no longer numerically-normalizes zero-padded ids; every
  native loader is fail-closed (returns `[]`, never raises). Gate: **1090 → 1119**
  (Windows; +29), golden-set unchanged. *(Sprint #133 — done.)*
- ✅ **Autonomous research agent (`ResearcherAgent`)** — the CPU-viable slice of the "AI
  scientist" pattern: a bounded **ingest → hypothesize → design → evaluate → iterate** loop
  (`aura/orchestration/researcher.py`) that proposes testable hypotheses, designs an
  experiment for each, runs it through a **real injected evaluator**, and refines under a
  hard `max_experiments` budget with early stop on a met target / convergence. Reuses
  existing machinery (retrieval for ingest, the eval harnesses as the scorer). **Two
  constitutional guarantees, enforced in code (Law 1 + Law 3):** (1) **no model training** —
  the guarantee is *structural* (the agent has no training capability; it only calls the
  injected evaluator), with `screen_experiment` a best-effort pre-filter that refuses obvious
  training / fine-tuning / gradient proposals (scanning *every* field, hyphen/spacing-evasion
  resistant) and audits the refusal; (2) **conclusions are backed by the evaluator's real
  metric, never fabricated** — with no evaluator the loop runs nothing and says so, and only
  a finite numeric metric can become the accepted `best` (NaN/string never wins). A
  side-effecting experiment is flagged for approval, never auto-run. Keyless + deterministic
  (LLM hypothesize/design degrade to deterministic fallbacks). Hardened after adversarial
  review (recursive-field training screen + evasion resistance; non-finite-metric guard).
  Gate: **1119 → 1134** (Windows; +15), golden-set unchanged. *Sakana AI-Scientist-v2
  (Nature 2026-03) — orchestration/eval loop only.* *(Sprint #134 — done.)*
- ✅ **Recursive self-improvement harness** — one eval-gated **propose → evaluate → retain**
  loop over AURA's *own* prompts/skills (`aura/learning/rsi.py`), wiring the pieces that
  already exist (proposals from #101 GEPA / #102 EvoSkill; the accept/reject decision is the
  #5/#41 strict-metric-gain rule) rather than a new optimizer. **Three guarantees enforced in
  code (Law 1 + Law 3):** (1) **never retain a regression** — the gate is a strict gain, so a
  tie or drop is rejected and the metric can only move up or stand still; (2) **no weight
  training** — candidates are prompt/skill TEXT edits and a training/fine-tune/gradient
  proposal is refused (reusing #134's evasion-resistant screen) + audited (GEPA-style program
  evolution, the Darwin-Gödel self-rewrite idea narrowed to eval-gated human-reviewed edits);
  (3) **never self-apply** — a passing improvement is `pending_approval` and only
  `commit_approved` (after a real approval) applies it via the operator's `apply_fn`. Honest
  loop semantics: within one run nothing is applied, so every candidate is compared to the
  same live baseline (a pending proposal can't suppress other valid ones); the recursion
  happens across runs after a human approves. A committed **markdown receipt** makes the trail
  auditable. Keyless (metric + proposer injected). Gate: **1134 → 1146** (Windows; +12),
  golden-set unchanged. *GEPA (ICLR 2026 Oral); DGM narrowed.* *(Sprint #135 — done.)*
- ✅ **Wave 14 release `v2.4.0`** — cut the release for the multi-agent orchestration +
  autonomous-research wave. `aura/__init__.py` → `2.4.0`; CHANGELOG `[2.4.0]` summarizing
  #132–#136; README release badge + wave table + intro prose updated to lead with the
  research-platform framing; `RESULTS_BENCHMARKS.md` (the leaderboard), the `SwarmExecutor`
  topologies, the `ResearcherAgent` loop, and the RSI harness all linked/described;
  `test_release.py` re-pinned to 2.4.0. Test count stays **computed**
  (`python -m aura.evaluation.test_inventory` → ~1146 on Windows/py3.11), golden-set
  unchanged. Operator action remaining: tag `v2.4.0` (the OIDC trusted-publish workflow fires
  on the tag). Gate: **1146** (Windows; a release sprint — version/docs only, no new tests),
  golden-set unchanged. *(Sprint #136 — done. **Wave 14 complete.**)*

---

## Wave 13 — "Frontier Autonomy, Proven" (complete ✅ 2026-07-18)
- ✅ **Calibrated confidence + faithfulness check** *(accuracy claim)* — closes the
  thrice-deferred "truth-seeking output" item. `aura/orchestration/calibration.py`:
  `parse_confidence` reads a verbalized **1–5** rating an answer states about itself
  (`ELICITATION_SUFFIX` asks the model for `Confidence: N/5` + an `Uncertain:` line);
  `calibrated_confidence` folds in the #3 self-consistency signal; `verify_faithful`
  makes a **write** faithful only if the stated confidence meets a threshold
  (`AURA_CALIBRATION_MIN`, default 4), **fail-closed** on unstated/low. Behind
  `AURA_CALIBRATION=enabled`, `KnowledgeCopilot.answer` elicits the rating and, when a
  tool named in `AURA_WRITE_TOOLS` was actually used this turn, gates the answer: a
  low/unstated-confidence write is flagged in the reply **and** audited
  (`calibration:unfaithful_action_blocked`) — never silently shipped. Flag off (or
  `AURA_WRITE_TOOLS` empty) ⇒ default surface byte-unchanged. Gated on a new
  **calibration/uncertainty golden slice** (5 fixtures: confident/unsure × read/write +
  unstated-write) wired into `golden_ingest --report`. Hardened after adversarial review
  (parser anchors to the **last** rating so a "confidence" mention in prose can't mask
  the real final score — the core fail-open; planner/streaming scope divergence
  documented honestly). Gate: **964 → 987** (Windows; +23; keyless, zero golden
  regression). *arXiv:2601.15703 / 2601.07767; Kuhn 2023.* *(Sprint #127 — done. Wave 13
  opens.)*
- ✅ **SLM-first cost-aware routing** *(cost claim)* — routes narrow/repetitive turns to
  a cheap BYOK model tier and reserves the frontier tier for deliberative/complex/write
  turns, with a confidence-based escalation. `aura/orchestration/router_slm.py`:
  `route()` classifies a turn (deliberative kind, **write/side-effect intent**,
  non-English input it can't assess cheaply, size, reasoning cues, tool fan-out) —
  every failure direction is toward the safe, more-expensive frontier tier; a size
  boundary can be **derived offline** from recorded `llm` spans via a deterministic 1-D
  2-means (`two_means_threshold`, no sklearn). `ToolAgent.run` gained a per-call `model`
  override (concurrency-safe, no shared mutation). Behind `AURA_SLM_ROUTING=enabled`,
  `KnowledgeCopilot.answer` runs the cheap tier and escalates a low-confidence/non-answer
  read-only turn once to frontier (`router:escalated` audited) — but **never re-runs a
  turn that already performed a write** (a low-confidence write is flagged by #127's gate
  instead, not silently re-executed on another model). `benchmarks/slm_routing.py` prices
  a fixed representative corpus control-vs-routed: **~11.6% BYOK cost saving** at a
  conservative 20% escalation rate (self-contained pricing so the receipt is well-posed
  under any `AURA_MODEL`; honest list-price/fixed-corpus caveat). Flag off ⇒ default
  path byte-unchanged. Hardened after adversarial review (write-intent + unknown-language
  fail-safe routing; no write double-execution on escalation; anchored escalation markers;
  robust benchmark pricing). Gate: **987 → 1009** (Windows; +22), golden-set unchanged.
  *NVIDIA arXiv:2506.02153.* *(Sprint #128 — done.)*

---

## Forward plan — Wave 13 (remaining, not yet built)
*Compressed from the web-verified `aura-sprint-planner` briefs (2026-07-17); the
standalone plan files are folded in here so this roadmap is the single source of
truth. Renumbered to start after the shipped #122. All keyless-gated, CPU-only,
BYOK; golden-set N/A unless a sprint is marked an accuracy claim.*

### Wave 12 — "Zero Trust & Protocol Currency" (4 sprints)
**Exit:** identity/credential/memory-integrity posture matches the **Foundation tier**
of Anthropic's *Zero Trust for AI Agents* (test-proven); a scored self-audit against
the official **OWASP Top 10 for Agentic Apps 2026** with the cascading-failure gap
closed; MCP surface current with the 2026-07-28 spec. **Not doing:** mTLS/hardware
identity (Advanced tier), forced MCP-Tasks adoption, continuous red-team CI.
- ✅ **#123 — Cryptographic per-agent identity + short-lived scoped credentials — DONE**
  (see the Wave 12 shipped entry above; gate 923→938).
- ✅ **#124 — Memory integrity hashing + source attribution — DONE** (see the Wave 12
  shipped entry above; gate 938→947).
- ✅ **#125 — OWASP ASI-2026 self-audit + blast-radius replay gate — DONE** (see the
  Wave 12 shipped entry above; gate 947→957).
- ✅ **#126 — MCP 2026-07-28 protocol currency — DONE** (see the Wave 12 shipped entry
  above; gate 957→964. Reshaped to AURA's stdio surface: stateless-core verified, OAuth
  primitives shipped dormant, Tasks evaluated vs #111 not adopted).

### Wave 13 — "Frontier Autonomy, Proven" (5 sprints)
**Exit:** a calibrated confidence output *faithfulness-checked* against actions; a
**measured** BYOK cost delta from SLM routing; a Swarm-Skills coordination skill gated
through approval + retain-if-improves; a causal failure-attribution replay. **Not doing:**
GPU causal world models, cross-tenant federated learning, continuous red-team CI (all
named, Law 3).
- ✅ **#127 — Calibrated confidence + faithfulness check — DONE** *(accuracy claim)* (see
  the Wave 13 shipped entry above; gate 964→987). Verbalized 1–5 confidence + #3
  self-consistency; `verify_faithful` fail-closes a write on unstated/low confidence;
  new calibration/uncertainty golden slice; flag `calibration:unfaithful_action_blocked`.
- ✅ **#128 — SLM-first cost-aware routing — DONE** *(cost claim)* (see the Wave 13
  shipped entry above; gate 987→1009, golden-set unchanged). Deterministic 1-D 2-means
  span-derived size boundary + write-intent/unknown-language fail-safe routing; per-call
  model override; confidence escalation that never re-runs a write; measured ~11.6% BYOK
  saving via `benchmarks/slm_routing.py`. Flag `router:escalated`.
- ✅ **Portable swarm skills + marketplace security audit** — extends #102's single-agent
  skill loadout with a **swarm skill**: a portable coordination pattern (roles / workflow
  / hard bounds) a future `SwarmExecutor` (#132) consumes. `aura/learning/swarm_skills.py`:
  `validate_swarm_skill` (roles/workflow/bounds structural check, `max_agents` capped);
  `audit_skill` — a **DeepTeam-style adversarial audit** that recursively scans **every**
  string in the candidate (all keys, values, nesting — no privileged "prose-only" subset)
  for prompt-injection/jailbreak (reusing #107's `scan_input`), excessive agency
  (destructive shell/SQL, pipe-to-shell RCE), safety-bypass, data-exfiltration (incl. raw
  IP + SSRF cloud-metadata) and credential access, plus a dangerous-capability denylist on
  the declared `allowed_tools`; `export_skill`/`import_skill` for portability. Same
  load-bearing rule as #102: **never auto-activates** — `propose_swarm_skill` **always**
  audits (the `external` flag can't skip it) and a HIGH-severity finding is rejected +
  audited (`swarm_skill.rejected_by_audit`) and **never reaches the human approval queue**;
  only an approved skill loads, and activation refuses to shadow a differing same-name
  entry. Retention reuses #5/#102's `retain_if_improves` (kept only if a skill-level slice
  improves). Hardened across **two** adversarial passes: the first defeated the keyword-only
  audit via four bypasses (unscanned fields, unaudited tool grant, truncation, opt-in flag);
  the second defeated the bounded re-scan via a **structural** bypass (a payload buried
  beyond the depth/count/length scan bounds) — closed by making the audit **fail-closed on
  any candidate it can't fully scan** (bounds now aligned between `validate` and `audit`),
  plus NFKC/zero-width normalization and rules for reverse shells / `dd` / encoded-payload
  execution / `git push` exfil. **Stated honestly:** a static scan is a defence-in-depth
  *pre-filter*, not the containment boundary — the real controls are mandatory human
  approval (nothing auto-activates) and the executor honouring `bounds.allowed_tools`
  (#132); the audit keeps the obvious attacks away from the human and fails closed on the
  un-scannable. Gate: **1009 → 1058** (Windows; +49), golden-set unchanged.
  *DeepTeam (Apache-2.0); arXiv:2605.10052 / 2603.21019.* *(Sprint #129 — done.)*
- ✅ **Causal failure attribution (Abduct–Act–Predict)** — localizes a failure to the step
  that most likely caused it (the localization #102's whole-trajectory analysis leaves
  open). `aura/evaluation/causal_attribution.py`: for each ERRORED step (abduct), replay the
  dependency DAG with it forced to success (act — reusing #83's `counterfactual`, zero LLM
  calls) and read the structural consequence (predict — was the failure-triggered replan
  averted, how much downstream depends on it). A bounded deterministic score ranks the
  candidates, with a **root-of-cascade** signal (an errored step that other errored steps
  descend from) that dominates the proximate-symptom signal, so a cascading root cause isn't
  out-ranked by its last downstream symptom. `attribute_failures` batch upgrades #102's
  failure seed with a per-step localization. **Honest (Law 1):** attribution is structural +
  heuristic, not proven causation — what each downstream step would actually produce needs
  re-execution, and is labelled so; a failure with no errored step is stated as
  un-localizable, not guessed. Read-only, no env flag (like `explain.py`/`counterfactual.py`).
  Hardened after adversarial review (added the root-of-cascade signal so the symptom no
  longer out-ranks the root; deduped duplicate step ids). Gate: **1058 → 1070** (Windows;
  +12), golden-set unchanged. *arXiv:2509.10401.* *(Sprint #130 — done.)*
- ✅ **Wave 11+13 release `v2.3.0`** — cut the release covering everything since v2.1.0
  (v2.2.0 was planned for Wave 11 but folded in here). `aura/__init__.py` → `2.3.0`;
  CHANGELOG `[2.3.0]` summarizing Waves 11/12/13; README release badge + wave table +
  intro prose updated; new posture/receipt docs [ZERO_TRUST.md](ZERO_TRUST.md) (Anthropic
  Zero-Trust **Foundation tier** map) and [RESULTS_RELIABILITY.md](RESULTS_RELIABILITY.md)
  (concurrency / exactly-once / calibration / SLM-saving / attribution receipts), linked
  alongside `SECURITY_ASI2026.md` + `MCP_CURRENCY.md`; `test_release.py` re-pinned + a new
  guard that the release docs exist and are linked. Test count stays **computed**
  (`python -m aura.evaluation.test_inventory` → ~1070 on Windows/py3.11), golden-set
  unchanged. Operator action remaining: tag `v2.3.0` (the OIDC trusted-publish workflow
  fires on the tag). Gate: **1070 → 1071** (Windows; +1). *(Sprint #131 — done. **Wave 13
  complete.**)*
- ✅ **#129 — Swarm Skills + marketplace security audit — DONE** (see the Wave 13 shipped
  entry above; gate 1009→1058, golden-set unchanged). Portable roles/workflow/bounds skill
  kind, `skill:propose_swarm`, always-on recursive DeepTeam-style audit (a defence-in-depth
  pre-filter that fails closed on the un-scannable) before the approval queue — hardened
  across two adversarial passes — retained only if a skill-level slice improves. Flags
  `swarm_skill.proposed` / `swarm_skill.rejected_by_audit`.
- ✅ **#130 — Causal failure attribution — DONE** (see the Wave 13 shipped entry above; gate
  1058→1070, golden-set unchanged). Abduct–Act–Predict over a failed trace (reuses #83's
  counterfactual replay), root-of-cascade-aware step ranking, `attribute_failures` batch
  upgrading #102's fail seed; structural/honest, no proven-causation claim. (Theater what-if
  replay mode remains a `theater_app` follow-up, not built here.)
- ✅ **#131 — Wave 11+13 release `v2.3.0` — DONE** (see the Wave 13 shipped entry above; gate
  1070→1071). Version bumped to 2.3.0; CHANGELOG `[2.3.0]`; `ZERO_TRUST.md` +
  `RESULTS_RELIABILITY.md` created and linked from the README alongside `SECURITY_ASI2026.md`
  + `MCP_CURRENCY.md`; test count re-derived via #110's machinery. **Wave 13 complete.**
  Operator action: tag `v2.3.0` to publish.

**Considered, not scheduled:** mTLS/HSM identity (Advanced tier, needs an enterprise
target); continuous red-team CI gate; cross-tenant federated swarm learning (a privacy
design, not one sprint); GPU causal world models (fails the CPU-only filter); Postgres
default TraceStore / full async serving (SQLite-WAL was the cheaper keyless lift).

**Sequencing:** #123→#124→#125→#126 (identity is the dependency root); #127 can start
alongside late Wave 12; #129 depends on #125's isolation, #130 on #125+#129; #131 last.

### Wave 14 — "Multi-agent orchestration & autonomous research" (5 sprints)
*Added 2026-07-17 after a skeptical triage of an operator wishlist + two external
analyses (a ChatGPT valuation and a generic "agentic-AI gap" deep-research report),
each item web-verified and diffed against the actual codebase. Most of the batch was
**already shipped** or **declined as out-of-constitution** (see below); these five are
what survived as genuinely new, real, CPU-viable, and high-leverage. Sequenced AFTER
Waves 12–13 because the security/reliability/cost work is the higher-value diligence
lever — but #133 (benchmarks) can pull forward, it depends on nothing.*
**Exit:** AURA reads as a *research platform*, not just a framework — selectable
multi-agent topologies, a published multi-model benchmark leaderboard, and a bounded
autonomous-research loop, all keyless-gated.
- ✅ **#132 — Explicit multi-agent topology selector (`SwarmExecutor`) — DONE** (see the
  Wave 14 shipped entry above; gate 1071→1090, golden-set unchanged). One typed API for
  hierarchical / pipeline / debate / market behind `AURA_SWARM=enabled`, enforcing #129's
  bounds at execution (fail-closed tool scope, agent/step caps, workflow order); honest
  receipt (debate is a fixed panel, market is a selector). (Theater round-table /
  dependency-graph views remain a `theater_app` follow-up, not built here.)
- ✅ **#133 — Large multi-model benchmark suite (BYOK leaderboard) — DONE** (see the Wave 14
  shipped entry above; gate 1090→1119, golden-set unchanged). Adapter framework over
  τ²-bench/GAIA/SWE-bench/BFCL (all web-verified), keyless harness self-test + BYOK real
  runs, `RESULTS_BENCHMARKS.md`; anti-vacuous-pass + fail-closed hardened. Built by a
  4-agent swarm. (JobBench confirmed not a runnable harness — not adopted.)
- ✅ **#134 — Autonomous research agent (`ResearcherAgent`) — DONE** (see the Wave 14
  shipped entry above; gate 1119→1134, golden-set unchanged). Bounded ingest→hypothesize→
  design→evaluate→iterate loop, real injected evaluator (no fabricated conclusions),
  structural no-training guarantee + evasion-resistant pre-filter, budget + convergence
  bounds, approval gate for side effects. The CPU-viable slice of the AI-scientist pattern.
- ✅ **#135 — Recursive self-improvement harness — DONE** (see the Wave 14 shipped entry
  above; gate 1134→1146, golden-set unchanged). Eval-gated propose→evaluate→retain over
  AURA's own prompts/skills, three enforced guarantees (no regression / no weight training /
  no self-apply-without-approval), committed receipt. GEPA-style program evolution, DGM
  narrowed to human-reviewed edits.
- ✅ **#136 — Wave 14 release `v2.4.0` — DONE** (see the Wave 14 shipped entry above; gate
  1146, golden unchanged). Version 2.4.0; CHANGELOG `[2.4.0]`; benchmark leaderboard + research-loop +
  topology gallery linked from README; count re-derived via #110. **Wave 14 complete.**
  Operator action: tag `v2.4.0` to publish.

**Reviewed & DECLINED (out of AURA's CPU-only / no-train / BYOK constitution — Law 3):**
GPU/hardware acceleration & distributed-GPU execution; RLHF / model fine-tuning /
distillation (AURA is BYOK-on-Claude and never trains weights); physical-AI / NVIDIA
Omniverse / robotics simulation; synthetic-data factories; cross-tenant "collective
superintelligence" & "agentic economies" (Gartner-peak hype, >40% of agentic projects
projected cancelled by 2027 — only a bounded *within-tenant* agent-memory slice was real,
and AURA already has the memory graph + provenance + causal layer). **Already shipped**
(so the deep-research report's 12 "enhancements" are ~10/12 done): multi-agent
orchestration, self-reflection/debate, structured DAG planning, tool/MCP integration,
GraphRAG, safety+governance (guardrails/sandbox/approvals/authz/egress/bounds),
interpretability (explanation chains + Theater), continuous evaluation (golden-set +
flip-gate + agent-bench + self-bench), cost-aware role routing.

**On "make it more valuable" (honest framing, not a valuation):** the external ratings
translate to three measurable levers already captured above — **credibility/diligence**
(Wave 12 Zero-Trust + OWASP posture), **evaluation proof** (#133 benchmark leaderboard),
and **differentiation** (#134 research agent). Fundraising, NVIDIA-Inception, and
partnership outreach are business actions, not code sprints — noted, not scheduled; and I
am not a valuation professional, so no dollar figure is asserted here.

---

## Forward plan — Waves 15 & 16 (proposed 2026-07-18, not yet built)
*Scoped from an external Wave-15/16 strategic proposal (`AURA_WAVE15_16_STRATEGIC_ROADMAP.md`),
re-triaged here against AURA's constitution. Every frontier source below was **re-verified by
live web search 2026-07-18** (Law 2), not carried over on trust: **AgentO** (ESWC 2026 resource
paper — confirmed on the ESWC-2026 accepted-papers list, SBA Research, U. Vienna, Springer LNCS
16550), **pySHACL** (RDFLib, Apache-2.0, last release Jan 2026), **rdflib** (BSD), the **official
MCP Registry** (`registry.modelcontextprotocol.io`, `mcp-publisher` CLI, `io.github.*` namespace
via GitHub OAuth), and **microsoft/playwright-mcp** (Apache-2.0, ~30–33k★). The proposal's core
finding held up on audit: ~85% of a separate "what agentic frameworks need" list is **already
shipped** here (MCP client+server+marketplace, A2A + discovery, Zero-Trust identity, OWASP ASI
audit, the #133 4-suite benchmark leaderboard, GraphRAG, causal attribution, calibrated
confidence, SLM routing, swarm topologies) — the one genuinely-open frontier gap is a **formal
semantic layer** (grep confirms zero `OWL`/`RDF`/`SPARQL`/`SHACL` in the repo; the existing
`OntologyProposal` is a Pydantic **shape** check, not a **semantic** one).*

### Wave 15 — "Brain 2.0: Formal Ontology Grounding" (#137–#140)
**Exit:** the ontology store exports as a real OWL/RDF graph; a fixed battery of competency
questions (SPARQL) answers questions about it; a SHACL shape set catches a seeded bad
`OntologyProposal` before it is accepted; a public receipt (`RESULTS_ONTOLOGY.md`) states
coverage honestly against AgentO's entity model. **Not doing** (Law 3): no external triple-store
(Jena/GraphDB) — rdflib's in-memory SPARQL is CPU-fine at AURA's scale; no OWL DL reasoner
(HermiT/Pellet) — they need a JVM, failing the CPU/free-OSS-first filter; pySHACL's bundled
OWL-RL gives the practical value at zero extra dependency weight. **Strong fit** — domain-agnostic,
CPU-only, free-OSS, closes a real named gap.
- ✅ **#137 — OWL/RDF export layer — DONE** (see the Wave 15 shipped entry above; gate 1212→1218/1222,
  golden-set unchanged). `aura/knowledge/ontology_rdf.py` — a **pure-stdlib** Turtle/OWL emitter
  (standard RDFS/OWL, `aura:` namespace) with optional rdflib round-trip validation; read-only tool
  behind `AURA_ONTOLOGY_RDF`; new `[ontology]` extra (rdflib+pyshacl) added to CI. Dependency root for
  #138/#139.
- ✅ **#138 — Competency questions + SPARQL surface — DONE** (see the Wave 15 shipped entry above;
  gate 1218→1228/1232, golden-set unchanged). 7 domain-agnostic competency questions as parameterized
  SPARQL over #137's RDF (class-param sanitized = no injection), a `competency_report` receipt (7/7 on
  the example), a read-only `ontology_query` tool. rdflib via the `[ontology]` extra (importorskip in
  tests). *AIAO v2.0 methodology.*
- ✅ **#139 — SHACL shape validation — DONE** (see the Wave 15 shipped entry above; gate
  1228→1238/1242, golden-set unchanged). `aura/knowledge/shacl.py` — **domain-agnostic** ontology-
  integrity shapes (no dangling relation refs, classes labeled) + an operator-extensible
  `AURA_ONTOLOGY_SHAPES` file; `validate_proposal` runs pySHACL **alongside** the Pydantic check;
  **fail-open** ingest hook behind `AURA_ONTOLOGY_SHACL` (strict blocks); flag `ontology:shacl_violation`.
  *pySHACL (Apache-2.0); arXiv:2604.00555.*
- ✅ **#140 — Brain 2.0 receipt (`RESULTS_ONTOLOGY.md`) — DONE** (see the Wave 15 shipped entry above).
  A scored self-audit — 29 RDF triples, competency 7/7, SHACL valid-conforms + dangling-caught — with
  the honest AgentO-coverage framing (domain schema uses standard RDFS/OWL, not AgentO's agent classes)
  and the named non-goals (no DL reasoner, no external triple-store). Linked from README; a test keeps
  its headline numbers matching a live run. **Wave 15 complete.**

### Wave 16 — "Agent Tooling & MCP Ecosystem" (#141–#145) — generalized, domain-agnostic
**Direction (operator, 2026-07-18):** AURA stays a **general-purpose** framework — **no domain
(vertical/domain-specific) logic in core.** Wave 16 gives every agent two high-leverage *general-purpose*
tools (the "code interpreter + web browsing" pair that transformed frontier assistants), then finishes
the MCP-ecosystem work and releases. Every tool below reuses AURA's **existing** safety controls
(sandbox caps, egress allow-list, tool-result screening, per-agent capability guard, audit) so it adds
capability without adding a new trust surface, and every one is **off by default** so the air-gapped
identity is preserved when unused. **Not doing** (Law 4): no second plugin marketplace (#129 exists);
no domain tools (declined below).
- ✅ **#141 — Code-interpreter tool (`code_exec`) — DONE** (see the Wave 16 shipped entry above; gate
  1147→1176/1180 on py3.11/3.12, golden-set unchanged). `code_exec` wires the existing sandbox into the
  tool loop for deterministic computation; off by default (`AURA_CODE_TOOL`), no new dependency. **En
  route it closed a CRITICAL pre-existing sandbox escape** (a `__builtins__['eval']` full-host-RCE hole
  in the #70/#103 sandbox's AST screen) with five defense layers — hardening the whole sandbox, not
  just this tool. *PAL/PoT/ReAct.*
- ✅ **#142 — Web-research tools (`web_search` + `web_fetch`) — DONE** (see the Wave 16 shipped entry
  above; gate 1176→1192/1196 on py3.11/3.12, golden-set unchanged). Egress-gated, off by default,
  fetched content #107-screened, free `ddgs` default + BYOK path, keyless (injected network).
  **Hardened after adversarial review**: an **SSRF-via-redirect** hole (urllib followed a 302 from an
  allowed host to an internal/metadata host without re-checking) is closed by re-running `check_egress`
  on **every** redirect hop; the search `title`/`url` (not just the snippet) are now screened.
- ✅ **#143 — Publish the MCP server to the official MCP Registry — DONE (scaffold; publish is an
  operator action)** (see the Wave 16 shipped entry above; gate 1192→1198/1202, golden-set unchanged).
  Schema-valid `server.json` (`2025-12-11`, `io.github.rubansivanandam/aura`, PyPI `aura-agents` +
  `uvx` + stdio), an `aura-mcp` console entrypoint, `scripts/publish_mcp_registry.sh` (preflight +
  publish), `MCP_REGISTRY.md`, and a CI check that fails if the manifest version drifts from
  `aura.__version__`. The two publish steps (PyPI publish + `mcp-publisher` GitHub-OAuth) remain
  operator actions, stated honestly.
- ✅ **#144 — Vetted external-MCP-tool allow-list — DONE** (see the Wave 16 shipped entry above; gate
  1198→1208/1212, golden-set unchanged). Extended #38's enforced trust with **host-scoping for remote
  MCP servers** via the #120 egress allow-list (fail-closed), added a `RemoteMCPTransport`, a vetted
  reference config (playwright stdio+sha256, GitHub remote host-scoped), and `MCP_TOOLS.md`. SQL/MSSQL
  MCP explicitly deferred (write-access risk).
- ✅ **#145 — Wave 16 release `v2.5.0` — DONE** (see the Wave 16 shipped entry above). Version 2.5.0;
  CHANGELOG `[2.5.0]` (#141–#145); `server.json` synced (the #143 CI check enforces it); README badge +
  stat band. Covers the shipped agent-tooling wave — Wave 15 (Brain 2.0 ontology, #137–#140) stays
  on-deck. Operator action: tag `v2.5.0` to publish. build directly.

**Declined for core — domain-specific (out of the generalized direction, per operator 2026-07-18):**
the source proposal's **cross-MCP domain reconciliation tool** and its **five domain-specific
skills** are real and useful, but they
are **vertical application logic**, not framework capability. They belong in a **downstream app built
*on* AURA** (a separate vertical repo), not in `aura/` core — keeping AURA reusable across domains.
The *mechanisms* they'd use already exist and stay general: #129 swarm skills + #102 `retain_if_improves`
for any domain's skills, #141's `code_exec` for any domain's computation, and #144's allow-list for any
domain's external tools.

**Sequencing (recommendation, not a rule):** #141 (code interpreter) is the single biggest
intelligence-per-sprint lever — do it first; #142 (web tools) next for external knowledge; #143 (publish
registry) is independent and nearly-free, slot it whenever; Wave 15 #137→#140 (the one real frontier
gap; grounds the ontology the #134 research agent reasons over) can run in parallel; #144 then the
release #145.

---

## Milestone summary

| Milestone | Sprint | Tag | What it means |
|-----------|--------|-----|---------------|
| Foundation | #0 | — | Core reactive agent shipped |
| Capability complete | #23 | — | All 4 tiers implemented |
| Adoption ready | #28 | `v0.9.0` | pip + Docker + CI + CLI |
| Measurably smarter | #34 | `v0.10.0` | Wave 2 deltas published |
| **Self-learning** | **#42** | **`v1.0.0`** | **Trusted single-node core + agents measurably improve themselves** |
| Ecosystem native | #54 | `v1.1.0` | Multi-model, plugins, A2A, MCP server |
| Enterprise ready | #68 | `v1.2.0` | Distributed, durable, compliant |
| Intelligence frontier | #88 | `v2.0.0` | Tool synthesis, meta-learning, formal safety |
| **Cinematic complete** | **#100** | — | **✅ 2026-07-12 — full Theater vision: rooms, VR, AI director, episodes** |
| Self-evolution closed | #103 | — | ✅ 2026-07-12 — Wave 7: GEPA proposer, runtime skills layer, WASM sandbox tier |
| Thesis proven | #106 | — | ✅ 2026-07-12 — Wave 8: committed optimizer receipt, τ-bench-style harness, judge rigor |
| Trust boundary hardened | #109 | — | ✅ 2026-07-12 — Wave 9: tool-result screening, tenant-bound approvals, Windows sandbox cap |
| **Acts in the world** | **#115** | **`v2.1.0`** | **✅ 2026-07-15 — Wave 10: persistent scheduler, screened notifications, approval-gated connectors, rule-based monitoring, PyPI release** |
| Scales safely | #122 | — | ✅ 2026-07-17 — Wave 11: WAL concurrency, streaming through the tool loop, per-agent authz + egress, backpressure, exactly-once tasks |
| **Zero Trust hardened** | **#126** | — | **✅ 2026-07-17 — Wave 12: crypto per-agent identity, memory-integrity hashing, OWASP ASI-2026 audit + blast-radius gate, MCP 2026-07-28 currency** |

---

## How to pick the next sprint
Highest leverage first: **close any roadmap→runtime truth gap, then Tier 1 → Tier 3
→ Tier 2 → Tier 4**. If a production promise (tenant isolation, auth semantics,
trust enforcement, reliability guarantees) is not yet true in code, that outranks
frontier capability work. One item per sprint; gate after; report the before→after
delta.

## The story behind the sprints
See [STORY.md](STORY.md) for the full narrative — origin story, technical blog post,
interview prep, and career positioning for AURA.
