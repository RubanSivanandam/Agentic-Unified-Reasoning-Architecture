# Changelog

All notable changes to AURA are documented here, derived from the sprint log in
[ROADMAP.md](ROADMAP.md) (sprints are the release notes — this file collects
them). Format loosely follows [Keep a Changelog](https://keepachangelog.com/);
versioning is semantic-ish: `v1.0.0` is reserved for the Wave-3 "self-learning"
release.

Every entry below shipped behind the same keyless gate (`AURA_LLM=stub`:
pytest · `example_ingest.py` · golden-set report), CPU-only, no GPU, BYOK.

## [2.7.0] — 2026-07-19

Wave 17 — **Memory 2.0** (Sprints #147–#150). Keyless end-to-end, CPU-only, no GPU,
BYOK. Closes the three SimpleMem stages ([arXiv:2601.02553](https://arxiv.org/abs/2601.02553))
that #39's consolidation left open, each **off by default** (opt-in flags; unset ⇒
byte-identical behaviour, test-proven). Test count is computed, not hand-typed
(~1274 on Windows/py3.11, ~1278 on py3.12 — gated on both). Golden-set grows two
keyless categories.

- **#147 — measured conversation compression (stage 1).** `compression_fidelity`
  gives a deterministic keyless signal (salient-term coverage) on the context
  compressor; every compression records ratio + fidelity on its span. Opt-in
  fail-safe `AURA_CONTEXT_MIN_FIDELITY` keeps originals rather than ship a lossy
  summary. Golden category **context compression fidelity** (3 fixtures).
- **#148 — online semantic synthesis (stage 2).** A near-duplicate cluster's
  representative can be a **synthesised unified abstract** (BYOK, injectable) when
  `AURA_MEM_SYNTHESIZE=enabled`, **fidelity-guarded** so it never does worse than
  #39's longest-text fallback. Recall preserved (golden memory held 4/4).
- **#149 — intent-aware retrieval scope (stage 3).** Recall scope sized to query
  intent (narrow lookup vs broad enumeration), bounded by
  `AURA_RETRIEVAL_SCOPE_MAX`, opt-in via `AURA_RETRIEVAL_SCOPE=intent`. Golden
  category **intent-aware retrieval scope** (6 fixtures).
- **#150 — receipt + release.** [RESULTS_MEMORY.md](RESULTS_MEMORY.md) with live,
  keyless-verified numbers and an honest not-done section; `v2.7.0`.

## [2.6.0] — 2026-07-18

Wave 15 — **Brain 2.0: formal ontology grounding** (Sprints #137–#140, #146). Keyless
end-to-end, CPU-only, no GPU, BYOK. Test count is **computed, not hand-typed** (~1241 on
Windows/py3.11, ~1245 on py3.12 — gated on both). Golden-set unchanged. The ontology moves
from a Pydantic **shape** check to a genuine **semantic** one — a queryable OWL/RDF graph
that SPARQL interrogates and SHACL validates — all **read-only and opt-in** (off by default
⇒ ingest/retrieval byte-unchanged).

- **#137 — OWL/RDF export.** `aura/knowledge/ontology_rdf.py` serializes the runtime ontology
  to Turtle (`owl:Class` / `owl:DatatypeProperty` / `owl:ObjectProperty`, `aura:` namespace).
  **Pure standard library** so the keyless gate stays zero-dependency; the new `[ontology]`
  extra (`rdflib` + `pySHACL`, both pure-Python, added to CI) powers validation.
- **#138 — Competency questions + SPARQL.** 7 domain-agnostic competency questions as
  parameterized SPARQL over the RDF (class params sanitized — no injection), a
  `competency_report` receipt (7/7), a read-only `ontology_query` tool.
- **#139 — SHACL validation.** Domain-agnostic **ontology-integrity** shapes (no dangling
  relation refs, classes labeled) via pySHACL **alongside** the Pydantic check; operator
  domain shapes via `AURA_ONTOLOGY_SHAPES`; **fail-open** ingest hook behind
  `AURA_ONTOLOGY_SHACL` (`strict` blocks); flag `ontology:shacl_violation`.
- **#140 — Receipt** [`RESULTS_ONTOLOGY.md`](RESULTS_ONTOLOGY.md) — 29 triples, competency 7/7,
  SHACL valid-conforms + dangling-caught, with the honest AgentO-coverage framing.
- **#146 — this `v2.6.0` release** (Brain 2.0 shipped after the v2.5.0 tag, so it gets its own).

**Honest (Law 1):** AURA's ontology is a *domain* schema, so it uses standard **W3C RDFS/OWL**
vocabulary — AgentO/AIAO are the *methodology* inspiration for modeling agentic systems, not a
schema the domain terms pretend to instantiate. No OWL DL reasoner (no JVM), no external
triple-store, no hardcoded domain rules (those are the operator's SHACL file). Verified 2026-07-18.

## [2.5.0] — 2026-07-18

Wave 16 — **agent tooling & MCP ecosystem** (Sprints #141–#145). Keyless end-to-end,
CPU-only, no GPU, BYOK. Test count is **computed, not hand-typed**
(`python -m aura.evaluation.test_inventory`; ~1208 on Windows/py3.11, ~1212 on py3.12 —
now gated on both). Golden-set unchanged. Every tool is **off by default** (the air-gapped
identity is preserved when unused) and reuses AURA's existing sandbox / egress / screening /
capability-guard / audit controls, so it adds capability without a new trust surface. Two
sprints surfaced (and closed) real security findings in adversarial review — the reason
network/execution tools get that gate.

- **#141 — Code-interpreter tool (`code_exec`).** Deterministic computation
  (arithmetic / stats / JSON-CSV / regex / date-math) offloaded to the existing sandbox,
  behind `AURA_CODE_TOOL`. En route it **closed a CRITICAL pre-existing sandbox escape** (a
  `__builtins__['eval']` full-host-RCE hole) with five defense layers — hardening the whole
  #70/#103 sandbox.
- **#142 — Web-research tools (`web_search` + `web_fetch`).** External knowledge, gated by
  `AURA_WEB_TOOLS` **and** the #120 egress allow-list (both required); fetched content
  screened as untrusted data. Fixed a **HIGH SSRF-via-redirect** (every redirect hop now
  re-checks egress). Free `ddgs` default, BYOK production path.
- **#143 — MCP-registry publish scaffold.** Schema-valid `server.json`
  (`io.github.rubansivanandam/aura`), an `aura-mcp` entrypoint, a publish script, a
  [`MCP_REGISTRY.md`](MCP_REGISTRY.md), and a CI check that the manifest version can't drift
  from `aura.__version__`. The two publish steps are honest operator actions.
- **#144 — Vetted external-MCP-tool allow-list.** **Host-scoping for remote MCP servers**
  via the egress allow-list (fail-closed), a `RemoteMCPTransport`, a vetted reference config
  (playwright stdio+sha256, GitHub remote), and [`MCP_TOOLS.md`](MCP_TOOLS.md).
- **#145 — this `v2.5.0` release.** Version bump, CHANGELOG, `server.json` synced, README.

## [2.4.0] — 2026-07-18

Wave 14 — **multi-agent orchestration & autonomous research** (Sprints #132–#136). Keyless
end-to-end, CPU-only, no GPU, BYOK. Test count is **computed, not hand-typed**
(`python -m aura.evaluation.test_inventory`; ~1146 on Windows/py3.11). Golden-set unchanged
(profiler 5/5 · retrieval 5/5 · routing 12/12 · memory 4/4 · calibration 5/5 · flip-gate
stable) — this wave adds orchestration, evaluation, and research capability, not accuracy
regressions. Every sprint shipped behind an interface + flag and was hardened after an
adversarial review (see [ROADMAP.md](ROADMAP.md)).

- **#132 — `SwarmExecutor`.** One typed API for **hierarchical / pipeline / debate / market**
  topologies behind `AURA_SWARM=enabled`, unifying the previously-separate patterns and —
  the load-bearing part — **enforcing #129's swarm-skill bounds at execution** (fail-closed
  tool scope, agent/step caps, workflow order), closing #129's containment loop.
- **#133 — Multi-model benchmark suite.** An adapter framework over **τ²-bench, GAIA,
  SWE-bench, BFCL** (all web-verified) with a keyless harness self-test + BYOK real runs and
  a rendered [RESULTS_BENCHMARKS.md](RESULTS_BENCHMARKS.md); honest by construction — no
  fabricated model scores, SWE-bench reported BYOK-only. Built by a 4-agent swarm.
- **#134 — `ResearcherAgent`.** A bounded ingest→hypothesize→design→**evaluate→iterate** loop
  with a *structural* no-training guarantee and conclusions backed by a real injected
  evaluator (never the model's say-so) — the CPU-viable slice of the "AI scientist" pattern.
- **#135 — Recursive self-improvement harness.** An eval-gated propose→evaluate→retain loop
  over AURA's own prompts/skills with three enforced guarantees: no regression retained, no
  weight training, no self-application without human approval; committed receipt.
- **#136 — this `v2.4.0` release.** Topology gallery + benchmark leaderboard + research-loop
  linked from the README; count re-derived via #110.

## [2.3.0] — 2026-07-18

Waves 11–13 — **scale, zero trust, and frontier autonomy** (Sprints #116–#131). Everything
since v2.1.0, in one release (v2.2.0 was planned for Wave 11 but folded in here). Keyless
end-to-end, CPU-only, no GPU, BYOK. Test count is **computed, not hand-typed**
(`python -m aura.evaluation.test_inventory`; host-dependent, ~1070 on Windows/py3.11).
Golden-set unchanged from 2.1.0 (profiler 5/5 · retrieval 5/5 · routing 12/12 · memory 4/4)
plus a new **calibration/uncertainty 5/5** slice — these waves add scale, security,
reliability, and cost receipts, not accuracy regressions. Every sprint shipped behind an
interface + flag and was hardened after an adversarial review (see [ROADMAP.md](ROADMAP.md)).

**Wave 11 — Scale & emergence (#116–#122).** SQLite **WAL + busy-timeout** as the one place
AURA opens SQLite (#116); **true token streaming through the tool loop** (`stream_agentic`,
#117); a **concurrency benchmark** receipt — 400/400 concurrent writes, 0 `SQLITE_BUSY`
(#118); **per-agent capability enforcement** on every run (#119); a **central egress
allow-list** (#120); **concurrency backpressure** with 503 + Retry-After (#121); and
**exactly-once task execution** via an atomic compare-and-set claim (#122).

**Wave 12 — Zero Trust & protocol currency (#123–#126).** **Cryptographic per-agent
identity** + short-lived scoped credentials (#123); **memory-integrity hashing** with
tamper→quarantine (#124); an **OWASP ASI-2026 self-audit** + a fail-closed **blast-radius
gate** (#125); and **MCP 2026-07-28 protocol currency** — stateless-core verified, OAuth
primitives shipped dormant (#126). Posture documented in
[ZERO_TRUST.md](ZERO_TRUST.md) (Foundation tier), [SECURITY_ASI2026.md](SECURITY_ASI2026.md),
and [MCP_CURRENCY.md](MCP_CURRENCY.md).

**Wave 13 — Frontier autonomy, proven (#127–#131).** **Calibrated 1–5 confidence** with a
fail-closed **faithfulness gate** on writes (#127); **SLM-first cost-aware routing** with a
measured ~11.6% BYOK saving and write-safe escalation (#128); **portable swarm skills** gated
by an always-on **DeepTeam-style security audit** before human approval (#129);
**Abduct–Act–Predict causal failure attribution** that localizes a failure to its root-cause
step (#130); and this **v2.3.0** release (#131). Receipts in
[RESULTS_RELIABILITY.md](RESULTS_RELIABILITY.md).

## [2.1.0] — 2026-07-15

Wave 10 — **act in the world, and persist** (Sprints #110–#115). AURA goes from
request-response to closed-loop automation, keyless end-to-end and behind the same
trust boundary Wave 9 hardened. Test count is now **computed, not hand-typed**
(#110 ended the 617/689/758/785/791 drift — the exact number is host-dependent;
run `python -m aura.evaluation.test_inventory`). Golden-set unchanged from 2.0.0
(5/5 profiler · 5/5 retrieval · 12/12 routing · 4/4 memory · flip-gate stable) —
this wave adds production capability, not accuracy claims; its tests live in
`tests/workflows/`.

- **#110 One canonical test count** — a computed single source of truth
  (`aura.evaluation.test_inventory`) + a guard against stale hardcoded totals in the
  docs. Closed the C8 measurement drift.
- **#111 Persistent task queue + scheduler** — `aura/tasks/engine.py`: cron
  scheduling, dependencies, retries, crash-resume via the #7 CheckpointStore, a
  mandatory no-bypass guard chain (guardrail + tenant/arg/time-bound approval) on
  every run, and `task` trace spans. New CLI `aura task schedule|list|status|run`.
- **#112 Screened outbound notifications** — `aura/comms/notifications.py`: email /
  Slack / Teams / webhook, body **and** subject screened before egress (default
  strict/redact), a dedicated `comms:content_egress` approval class for bulk sends,
  safe templating. Relaxes #52's "no content leaves the box" — deliberately, behind
  screening + approval.
- **#113 File/document + calendar connectors** — `aura/connectors/`: local + BYOK
  cloud/calendar, **untrusted-by-default** as an enforced invariant
  (`AURA_CONNECTORS_ALLOW` allow-list + fail-closed without OAuth), one-time
  approval-gated calendar writes. No hard cloud SDK dependency; keyless gate green
  with all connectors absent.
- **#114 Workflow templates + rule-based monitoring** — parameterized DSL templates
  on the #57 durable engine; `TaskMonitor` rule-based anomaly flags (duration,
  failure rate, missed schedule) + a dashboard status-panel data contract.
- **#115 Release** — this version, PyPI trusted-publishing (OIDC) release workflow,
  and an `examples/` gallery with the keyless end-to-end demo
  (schedule → guarded run → screened notify → monitor panel).

## [2.0.0] — 2026-07-11

Wave 6 — intelligence frontier (Sprints #69–#88). **689 tests · golden-set
5/5 profiler · 5/5 retrieval · 12/12 routing · 4/4 memory · flip-gate stable
(26 fixtures).** The wave where agents *evolve* — and where every evolution
claim is fenced by an instrument that existed first: tools are synthesized
but never auto-activate, teachers must be measured before they may teach,
improvement suggestions may never loosen the accept-only-if-improves gate,
and the framework now benchmarks itself continuously and against its peers.

| Metric | Wave start (v1.2.0) | Wave end (v2.0.0) |
|---|---|---|
| keyless tests | 468 | 689 |
| golden categories / fixtures | 4 / 26 | 4 / 26 (held green throughout) |
| self-evolution surfaces | — | tool synthesis→sandbox→composition; patterns→meta→curriculum→collective→teaching |
| safety proofs | trust fail-closed | + formal bounds (plan-time), post-hoc constraint verification, fixed adversarial corpus |
| published artifacts | benchmarks/RESULTS.md | + docs/AURA_PAPER.md, benchmarks/compare/ (vs LangGraph/CrewAI/AG2) |

### Sprints
- **#69–#71 tool evolution** — synthesis proposes composite tools from
  recurring call patterns (never auto-activates; approval-gated), the sandbox
  gates future freeform bodies (AST screen + subprocess isolation), the
  composer turns approved specs into real traced tools. 473→509.
- **#72–#75 learning stack** — cross-session pattern mining (structural
  fields only, tenant-scoped, leak-tested), meta-learning over proposer track
  records (advisory, abstains under-evidence), curriculum scheduling
  (Laplace-smoothed difficulty), k-anonymous collective intelligence.
  509→550.
- **#76–#77 autonomous goals** — bounded goal decomposition with fail-closed
  human checkpoints; persistent goal tracking with recorded-never-inferred
  progress. +24.
- **#78–#80 formal safety** — declarative behaviour bounds enforced at
  planning time; post-execution constraint verification (cost, approval
  bypass); fixed keyless adversarial corpus with a neutered-defence
  self-test. +36.
- **#81 explainability** — decision chains reconstructed from recorded spans,
  never post-hoc model rationalization. +8.
- **#82 causal reasoning** — deterministic downstream-effect prediction over
  the knowledge graph's recorded relations; evidence-carrying paths; planner
  prompt hook byte-identical when off. 617→631.
- **#83 counterfactual analysis** — what-if replay over recorded spans with
  every claim stamped `structural` vs `requires_reexecution`; a replay never
  invents step outputs. 631→645.
- **#84 agent-to-agent teaching** — keyless distillation from recorded
  evidence by measured teachers (≥0.8 over ≥3 trials, abstains under
  evidence); advisory lessons with provenance; secret-leak-tested. 645→659.
- **#85 continuous self-benchmarking** — cron-able golden+bench snapshots,
  trailing-median degradation detection (any golden drop; latency only past
  1.5×), webhook alerts, dashboard 🩺 HEALTH panel. 659→679.
- **#86 research paper** — docs/AURA_PAPER.md: the gated-sprint methodology
  as the contribution; every number repo-traceable; every citation verified;
  limitations stated (LaTeX/arXiv submission = operator step). 679 held.
- **#87 benchmark vs alternatives** — benchmarks/compare/: orchestration
  overhead + sourced capability coverage vs LangGraph 1.2.9 / CrewAI 1.15.2 /
  AG2 0.14.0, identical scripted turns, 12/12 cells green on this machine,
  isolated venv, fairness + residual bias stated; **no answer-quality claim**
  (live-key comparison = named operator future work). 679→689.
- **#88 this release.**

**Wave 6 exit criteria, honestly assessed:** tool synthesis, cross-session
learning, goal decomposition, explainability, formal bounds, adversarial
testing — all shipped and test-proven ✅. "Paper published": the paper is
written and citation-verified; arXiv/workshop *submission* is an operator
action, stated ⚠. Carried forward, named: the BYOK optimizer receipt (#41),
TS SDK/publishing infra, MCP stateless core (RC), SQLCipher, TH-track
sprints #89–#100.

## [1.2.0] — 2026-07-06

Wave 5 — enterprise & scale (Sprints #55–#67, plus the long-deferred #34b).
**468 tests · golden-set 5/5 profiler · 5/5 retrieval · 12/12 routing · 4/4
memory · flip-gate stable (26 fixtures).** From single-process framework to a
deployable, governable system: distributed execution with stitched traces,
durable declarative workflows with human checkpoints, a compliance layer
(audit/residency/encryption/logging), an admin layer (keys/quotas/tenant
crypto), a Helm chart, and honest benchmarks.

| Metric | Wave start (v1.1.0) | Wave end (v1.2.0) |
|---|---|---|
| keyless tests | 353 | 468 |
| CI jobs | 6 | 7 (helm lint+template added) |
| compliance surface | — | audit chain, residency, at-rest crypto, SOC2-supporting logs |
| admin surface | env-var keys only | key lifecycle, org-admin role, usage analytics, quotas |

### Sprints
- **#34b Langfuse bridge** — export-only adapter over the HTTP ingestion API
  (SDK-independent by design); traces/generations/scores/prompts; AURA stays
  the source of truth. 398→406.
- **#55 distributed execution** — JSON task envelopes + ExecutionBackend seam;
  Celery/Ray optional adapters; env unset = classic path byte-identical.
  353→364.
- **#56 distributed trace aggregation** — remote spans ship home with results
  and stitch into one trace (`meta.remote` labeled). 364→370.
- **#57 durable workflows** — SQLite state; resume skips completed side
  effects; retry/backoff/timeout policies; typed non-durable-state error.
  370→379.
- **#58 workflow DSL** — JSON/YAML → engine; if/else, repeat/until, human
  checkpoints riding the approvals queue; never-eval conditions. 379→398
  (incl. the PyYAML both-sides fix CI caught).
- **#60 audit trail** — sha256 hash chain, tamper class detected at the exact
  record; export refuses broken chains; tool/approval/memory seams. 406→416.
- **#61 residency & encryption** — allowed-roots verification (strict boot
  refusal) + Fernet at-rest for JSON stores, fail-closed. 416→428.
- **#62 SOC2-supporting logging** — structured JSON, bounded rotation,
  enforced retention, auditor export ("SOC2-certified code" doesn't exist —
  controls do). 428→437.
- **#63 admin surface** — hashed key lifecycle (secret shown once), org-admin
  role with no-existence-leak scoping, usage analytics. 437→446.
- **#64 quotas** — hard blocks before the provider is contacted; once-a-day
  soft warnings via webhooks; durable day budgets. 446→455.
- **#65 tenant crypto isolation** — HKDF per-tenant keys; wrong tenant gets
  ciphertext + typed refusal, end-to-end proven. 455→461.
- **#66 Helm chart** — keyless-by-default deploy; BYOK secrets referenced,
  never created; honest single-node limits; CI helm job. 461→466.
- **#67 benchmarks** — keyless framework-overhead suites, published RESULTS.md;
  no clock assertions in CI. 466→468.

**Wave-5 exit criteria, honestly assessed:** durable workflows survive
restarts ✅ (test-proven) · audit + SOC2-supporting logging functional ✅ ·
admin API operational ✅ (the *UI* remains a frontend session, like #59) ·
distributed contract proven over a JSON wire ✅ — running on ≥2 real nodes
and `helm install` on a real cluster are **operator-verified by nature**
(chart lints/renders in CI; the fake-wire test proves the contract, not a
datacenter).

**Carried forward, named:** #59 workflow UI (frontend session), TH-track 3D
Live tab, the BYOK optimizer receipt, TS SDK/publishing, MCP stateless core
(still an RC), SQLCipher for shared SQLite stores.

## [1.1.0] — 2026-07-06

Wave 4 — the ecosystem release (Sprints #43–#53, plus the v1.0.0 deferred-work
ledger #45b–#48b). **353 tests · golden-set 5/5 profiler · 5/5 retrieval ·
12/12 routing · 4/4 memory · flip-gate stable (26 fixtures).** AURA now speaks
to, and can be spoken to by, everything around it: five LLM backends, per-role
multi-model routing, a plugin system with a verifying registry, A2A v1.0
(server + trust-aware client + discovery), bidirectional MCP, signed webhooks,
and a zero-dependency Python SDK — all keyless-gated, CPU-only, BYOK.

| Metric | Wave start (v1.0.0) | Wave end (v1.1.0) |
|---|---|---|
| keyless tests | 223 | 353 |
| LLM backends | 2 (anthropic, stub) | 5 (+ openai, gemini, local/air-gapped) |
| protocol surfaces | MCP client | A2A v1.0 server+client+discovery, MCP server+client, webhooks, SDK |
| v1.0.0 debt items open | 7 | 0 sprint-shaped (operator receipt + Langfuse remain) |

### Debt ledger from v1.0.0 (closed first)
- **#45b flip-gate CI history** — `actions/cache` rolling key; flaky fixtures
  now held red in CI exactly as locally.
- **#46b tenant-isolation remainder** — tenant contextvar through every worker
  thread; tenant columns on traces/spans/approvals; namespaced memories/graphs;
  cross-tenant decide is a 404, no existence leak. 244→252 tests.
- **#47b candidate proposers** — DSPy-pattern + TextGrad-pattern proposers,
  dependency-free (TextGrad dep cut as stale — Law-2 PyPI check); golden gate
  still vetoes every candidate. (T6 closed.)
- **#48b speculative planning (T1)** — DAG planner speculates ahead under
  `AURA_SPECULATE`, harvest-or-discard with wasted-work accounting. (T1 closed
  — the frontier list T1–T8 is now fully shipped.)

### Wave 4 sprints
- **#43 OpenAI backend** — adapter speaks the wrapper's client protocol;
  breaker/retry/spans/cache/cost unchanged. 223→232 tests.
- **#44 Gemini backend** — same pattern, Google-shaped translation. 232→239.
- **#45 Local/Ollama backend** — air-gapped, zero-key; OpenAI-compatible
  endpoints reuse #43's adapter verbatim. 239→244.
- **#46 multi-model routing** — `AURA_MODEL_MAP` role → backend/model with
  cheapest-candidate selection by `AURA_PRICES`; no map = byte-identical
  behavior (test-proven). Quality-threshold routing honestly deferred until
  per-model golden scores exist. 266→276.
- **#47 plugin system** — `aura-plugin.json` manifest; tools/workers register
  namespaced; **fail-closed `AURA_PLUGINS_ALLOW`** (discovery ≠ trust). 276→284.
- **#48 plugin registry** — local index, deterministic content sha256,
  install verifies hash fail-closed; integrity ≠ trust. 284→293.
- **#49 A2A production protocol** — Agent2Agent v1.0 (Linux Foundation,
  verified in-session): AgentCard at `/.well-known/agent-card.json`, JSON-RPC
  `message/send`/`tasks/get`, tenant-scoped tasks; outbound `A2AClient`
  fail-closed behind `AURA_A2A_ALLOW`. 293→308.
- **#50 A2A discovery & negotiation** — peer card fetch (allow-listed),
  capability directory, card-driven mode negotiation, `delegate_verified`
  with an injection screen over remote answers. 308→322.
- **#51 MCP server mode** — AURA as an MCP server (stable rev 2025-11-25);
  bidirectional MCP test-proven (our client consumes our server); untrusted
  callers screened. 322→334.
- **#52 webhook & event system** — HMAC-signed lifecycle events (jobs,
  approvals); emitting never breaks the host path; payloads carry references,
  not knowledge. 334→343.
- **#53 Python SDK** — `aura.sdk.AuraClient`, zero-dep, SSE streaming, proven
  against the real app in-process; streaming HTTP errors raise instead of
  reading empty (found + regression-tested this sprint). 343→353.

**Deferred, stated plainly:** the live-key optimizer receipt (BYOK one-liner,
unchanged from v1.0.0); Langfuse adapter (#34b); TypeScript SDK + PyPI/npm
publishing (release infrastructure); A2A extended card / streaming / push
notifications (spec-optional); async webhook dispatcher; MCP 2026-07-28
stateless core (still a release candidate).

## [1.0.0] — 2026-07-05

Wave 3 — hardening first, learning second (Sprints #35–#42). **223 tests ·
golden-set 5/5 profiler · 5/5 retrieval · 12/12 routing · 4/4 memory ·
flip-gate stable (26 fixtures) · no open flaws.**

| Metric | Wave start | Wave end |
|---|---|---|
| keyless tests | 159 | 223 |
| golden categories | 3 | 4 (memory compression added) |
| golden fixtures | 22 | 26 |
| open security flaws | 1 (advisory-only MCP trust) | **0** |
| frontier items closed this wave | — | T4, T8 |

- **#35 multi-tenant isolation + serving auth parity** — tenant-scoped state by
  construction (`TenantStates`), `ingest_async` admin-parity fix, and the
  `require_key`-is-a-no-op hole closed (principals mode deliberately hardened);
  tenant-scoped store dirs, slug validation, tenant-owned jobs. 159→168 tests.
- **#36 runtime reliability parity** — streaming now uses the same circuit
  breaker + bounded retry as every other path (retries only before the first
  delta — a half-stream is never replayed); injection screen on `chat_tokens`
  and `a2a`; typed job failures (`error_type`); pinned 3.11 runtime story.
  168→176 tests.
- **#37 flip-gate regression detector (T8)** — per-fixture pass/fail history;
  a flaky pass is not a real pass; held red until N consecutive greens; wired
  into Gate 3's exit code. 176→185 tests.
- **#38 MCP trust enforcement (Flaw #6)** — declarative allow-list
  (`AURA_MCP_TRUST_FILE`) with sha256 command pinning; unlisted servers blocked
  at registration, tools re-checked at call time, fail-closed policy parsing,
  JSONL audit of every invocation and refusal. **No open flaws remain.**
  185→194 tests.
- **#39 SimpleMem-pattern memory compression (T4) + provenance fix** —
  dependency-free consolidation (cluster → representative merge → recoverable
  tombstones), gated by a new golden **memory** category (merge 5→3 AND recall
  preserved); malformed provenance now quarantines instead of crashing.
  194→207 tests.
- **#40 LLM query rewrite + cross-encoder re-ranker** — opt-in BYOK stages,
  fail-back-to-deterministic; keyless path unchanged. 207→215 tests.
- **#41 DeepEval/RAGAS adapters + optimizer-proof harness** — LLM-judged
  metrics behind `[deepeval]`/`[ragas]` extras; `optimizer_proof.run_proof`
  writes the before→after receipt and lets the keyless gate veto any win.
  215→223 tests.

**Deferred, stated plainly:** the **live-key optimizer receipt** — the release
criterion that the optimizer measurably improves ≥2 LLM-judged categories — is
machinery-complete and keyless-proven but the live run is a deliberate BYOK
token spend: run `python -m aura.evaluation.optimizer_proof` and commit the
generated `OPTIMIZER_PROOF.md`. Also open: T1 speculative planning, DSPy/
TextGrad proposers, tenant scoping of traces/approvals/memory namespaces/inbox,
flip-gate history caching in CI.

## [0.10.0] — 2026-07-05

Wave 2 — make it measurably smarter (Sprints #29–#34). **159 tests · golden-set
5/5 profiler · 5/5 retrieval · 12/12 routing.** No code-behavior change beyond
the version bump — this release reports the wave's measured deltas.

### Before → after ledger

| Metric | Wave start | Wave end |
|---|---|---|
| keyless tests | 126 | 159 |
| golden categories | 2 | 3 |
| golden fixtures | 7 (5 profiler + 2 retrieval) | 22 (5 profiler + 5 retrieval + 12 routing) |
| routing category | — (didn't exist) | 12/12 |
| retrieval category | 2/2 | 5/5 |

### Sprints

- **#29 — Golden-QA routing category (G5).** Scoreboard grows a third keyless
  category (`routing_golden()` / `check_routing()`); 9 fixtures, 9/9 at
  baseline; wired into `--report`'s exit code.
- **#30 — Typed inter-agent contracts (T3, `AURA_CONTRACTS=enabled`).**
  130→138 tests; routing held 9/9.
- **#31 — Parallel fan-out supervisor (G1, `AURA_SUPERVISOR=parallel`).**
  Routing 9/9→12/12; divergent merges flag for human review via approvals
  (`supervisor:merge_conflict`); 138→147 tests.
- **#32 — Agentic retrieval loop (G2 part, `AURA_RETRIEVAL=agentic`).**
  Retrieval 2/2→5/5; the loop makes zero LLM calls (bounded cost lever);
  cite-or-say-unknown note when evidence is insufficient; 147→156 tests.
- **#33 — Live SSE feed (D1) + Neural Core dashboard.** Typed `event: span`
  stream on both apps, ⚡ Live mode, 60fps canvas UI, CDN vis-network removed
  (air-gap restored), `/api/graph` union bug fixed (was blind to ingested
  domain graphs); 156→159 tests.

### Deferred

- 3D theater "Live" tab → TH track (needs the R3F rebuild; the feed it will
  consume now exists).
- G2 remainder (query-rewrite / cross-encoder re-ranker, live-key only).
- T1 (speculative planning), T4 (SimpleMem compression), T8 (flip-gate
  regression detector) — all still open, tracked in Wave 3.

## [0.9.0] — 2026-07-03

First tagged release. **126 tests · golden-set 5/5 profiler · 2/2 retrieval ·
CI public on every push.** Covers Sprints #1–#27 plus the Theater track.

### Packaging & adoption (Wave 1, Sprints #24–#27)
- **`pyproject.toml` (PEP 621)** — installable as `aura-agents`; optional
  backends as extras (`[kuzu,fastembed,mem0,sqlite-vec,otel,mcp]`, plus `[all]`,
  `[dev]`); version single-sourced from `aura.__version__`. *(#24)*
- **GitHub Actions CI** — the three keyless gates on every push/PR across
  Python 3.10/3.11/3.12; wheel-hygiene job (twine + content invariants);
  theater build job; live status badge replacing self-reported ones. *(#25)*
- **Docker** — `docker compose up` → self-seeded dashboard on `:8000`, keyless
  by default; CPU-only slim image, non-root, state under `/data`; the boot is
  CI-verified (`docker demo boots keyless`). *(#26)*
- **`aura` CLI** — `demo | ingest | chat | serve | gate | report | version`
  console script (typer); `pip install aura-agents && aura demo` is the whole
  quickstart. *(#27)*

### Reasoning & reliability (Tier 1)
- Plan-and-Execute planner behind `AURA_PLANNER=enabled`. *(#1)*
- Structured output: Pydantic schema + repair loop for ontology synthesis. *(#2)*
- Verifier / self-consistency agent (`AURA_VERIFY=enabled`, majority voting). *(#3)*
- **Typed-DAG hierarchical planner** (`AURA_PLANNER=dag`): dependency-scoped
  context, re-planning on step failure, bounded sub-goal recursion. *(#23)*

### Memory & retrieval (Tier 2)
- Hybrid GraphRAG: multi-query rewrite + embedding re-rank + guaranteed graph
  depth (`AURA_RETRIEVAL=hybrid`). *(#4)*
- Scalable vector index via `sqlite-vec` (`AURA_VECTOR=sqlite-vec`). *(#9)*
- Hierarchical memory adapter for Mem0 (`AURA_MEMORY=mem0`, tenant-scoped). *(#11)*
- Multi-turn session memory in the copilot. *(#13)*

### Self-improvement (Tier 3)
- Metric-gated evaluator→optimizer loop: prompt variants kept only if they beat
  the current score; accepted prompts persist. *(#5)*
- Hard dedupe in the learning loop (exact meta match). *(#16)*

### Production & ecosystem (Tier 4)
- MCP client bridge — external MCP tools into AURA's registry, guard-governed;
  servers untrusted by default (CVE-2025-6514 posture). *(#6)*
- Crash-resumable execution: per-step checkpoints + re-anchoring. *(#7)*
- Human-in-the-loop approvals for write tools + `/v1/approvals`. *(#8)*
- Background job queue, RBAC/multi-tenant principals, token-level streaming,
  optional Postgres trace store. *(#8, #10)*
- Three-level continuous evaluation incl. keyless trajectory eval. *(#12)*
- Observability: OTLP export, `system_metrics()`, cost report. *(#8, #14)*

### 2026-Ultra frontier track
- FinOps per-agent cost tracking (`cost_usd` on every LLM span). *(#14, T6)*
- Conservative token counting + optional tiktoken. *(#14, flaw #1)*
- Cascade-failure guard: worker quarantine + graceful supervisor fallback. *(#15, T7)*
- Memory provenance/quarantine on recall (OWASP ASI06). *(#17, T5)*
- Typed-DAG planner closes flaw #2 and delivers T2. *(#23)*

### Agent Theater (TH track)
- Cinematic 3D theater at `/theater3d`: scene auto-classification (courtroom /
  round table / knowledge lab / command floor). *(#19, TH-1)*
- Fully self-hosted: three.js/GSAP vendored, air-gap safe. *(#20, TH-1b)*
- React Three Fiber rebuild with postprocessing, GLSL glow, React Flow,
  Cytoscape. *(#21, TH-2)*
- Roblox-style blocky humanoid characters with talk animations. *(#22, TH-4)*
- Conversation Theater: agents' real dialogue as animated chat with a live
  inspector, fed by redaction-aware span content capture. *(#18)*
- Per-agent TTS narration with word-synced mouths. *(TH-6 + TH-5)*

### Foundation (pre-#1)
Tool-use loop · reflection · debate · supervisor routing · self-building
ontology synthesis · profiler (self-ref FK / junction / enum detection) ·
knowledge graph (NetworkX default, Kuzu opt-in) · guardrails (injection screen,
memory-poisoning screen, risk-scored tool guard) · circuit breaker · golden-set
evaluator · dashboard · streaming FastAPI serving API · keyless stub LLM.

[1.0.0]: https://github.com/RubanSivanandam/Agentic-AI-Study/releases/tag/v1.0.0
[0.10.0]: https://github.com/RubanSivanandam/Agentic-AI-Study/releases/tag/v0.10.0
[0.9.0]: https://github.com/RubanSivanandam/Agentic-AI-Study/releases/tag/v0.9.0
