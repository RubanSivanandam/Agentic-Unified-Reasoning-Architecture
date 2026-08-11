# AURA Results — Reliability & Cost Receipts

Every claim here is produced by a **keyless, reproducible** script or test in this repo
(`AURA_LLM=stub`, CPU-only, no GPU, BYOK). Numbers are receipts, not marketing: each
points at the command that regenerates it, and each states its honest scope.

---

## Concurrency (Wave 11)

- **SQLite WAL + busy-timeout** (`aura/util/sqlite.py`, #116) is the single place AURA opens
  SQLite. Receipt: `benchmarks/concurrency.py` (#118) lands **400/400 concurrent session
  writes with 0 `SQLITE_BUSY`**, against a bare rollback-journal control that drops writes.
  Reproduce: `AURA_LLM=stub python benchmarks/concurrency.py`.
- **Scope:** this measures *storage* concurrency (meaningful keyless). Time-to-first-token
  needs a live BYOK model and is left as operator/BYOK work — not faked with a stub figure.

## Exactly-once execution (Wave 11)

- **Atomic compare-and-set task claim** (`aura/tasks/engine.py`, #122): a task runs at most
  once even under concurrent workers — an atomic `UPDATE … WHERE status IN
  ('scheduled','blocked')` means only the single row-count winner executes; stale `running`
  tasks are requeued through the same exclusive claim (`AURA_TASK_STALE_S`). Proven by the
  task-engine tests.

## Calibrated confidence + faithfulness (Wave 13)

- **Fail-closed faithfulness gate** (`aura/orchestration/calibration.py`, #127): a write is
  faithful only if the answer states a confidence at/above threshold
  (`AURA_CALIBRATION_MIN`, default 4); an unstated/low-confidence write is flagged + audited
  (`calibration:unfaithful_action_blocked`), never silently shipped. Gated by a
  **calibration/uncertainty golden slice** (5 fixtures) in
  `python -m aura.evaluation.golden_ingest --report` → *golden-set (calibration/uncertainty):
  5/5 correct*.

## SLM-first cost routing (Wave 13)

- **Measured BYOK saving** (`aura/orchestration/router_slm.py`, #128): routing narrow
  turns to a cheap tier and reserving the frontier tier for deliberative/complex/write
  turns yields **~11.6% BYOK cost reduction** on a representative corpus at a conservative
  20% cheap-escalation rate, with a write-safe escalation that never re-runs a turn that
  already wrote. Reproduce: `AURA_LLM=stub python benchmarks/slm_routing.py`.
- **Scope:** an estimate from a fixed corpus + list prices (self-contained pricing so the
  receipt is well-posed under any `AURA_MODEL`); a live BYOK run differs — the receipt
  proves the sign and shape of the saving, not a specific dollar figure.

## Causal failure attribution (Wave 13)

- **Root-cause localization** (`aura/evaluation/causal_attribution.py`, #130): an
  Abduct–Act–Predict counterfactual replay (zero LLM calls) ranks a failed run's steps by a
  structural causal score, with a root-of-cascade signal so a cascading root cause isn't
  out-ranked by its downstream symptom. Honest: structural + heuristic, not proven causation.

---

## The gate every number passed

`AURA_LLM=stub`: `pytest` · `example_ingest.py` · `python -m aura.evaluation.golden_ingest
--report` — run twice, CPU-only. The exact test count is **computed, not hand-typed**
(`python -m aura.evaluation.test_inventory`; host-dependent — some tests are platform-gated).
Golden-set has stayed unchanged across Waves 11–13 (profiler 5/5 · retrieval 5/5 · routing
12/12 · memory 4/4 · calibration 5/5 · flip-gate stable): these waves add capability,
security, reliability, and cost receipts — not accuracy regressions.

See also: [ZERO_TRUST.md](ZERO_TRUST.md), [SECURITY_ASI2026.md](SECURITY_ASI2026.md),
[MCP_CURRENCY.md](MCP_CURRENCY.md).
