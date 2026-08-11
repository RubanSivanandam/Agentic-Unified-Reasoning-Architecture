# AURA Benchmark Leaderboard

Adapter-based results across published, open agent benchmarks — **τ²-bench, GAIA, SWE-bench, BFCL** — through one harness (Sprint #133, extending #105).

> **Honest scope (Law 1).** The *harness* is CPU-only and keyless; the numbers below are the **keyless self-test** — each case answered by the adapter's deterministic oracle, which proves the harness + scorer end-to-end with no API key and no dataset download. A **real multi-model leaderboard is BYOK**: supply the actual dataset and a per-suite `solver(case)->response` that calls your model, and the same scorers apply. SWE-bench additionally needs a container to run repo tests, so it is BYOK-only here. Nothing below is a fabricated model score.

Reproduce: `AURA_LLM=stub python -m aura.evaluation.benchmarks.leaderboard --report`

Run label: **keyless-oracle**

| Suite | Metric | Mode | Cases | Passed | Pass-rate | License |
|-------|--------|------|-------|--------|-----------|---------|
| `bfcl` | AST match (function name + arg names/values, type-aware, order-independent) | keyless | 3 | 3 | 100% | Apache-2.0 |
| `gaia` | quasi-exact-match (normalized numeric/string exact match; GAIA arXiv:2311.12983) | keyless | 2 | 2 | 100% | CC-BY-4.0 |
| `swebench` | resolved rate: patch applies, then ALL FAIL_TO_PASS pass and ALL PASS_TO_PASS still pass (tests run by the operator's container harness) | skipped | — | — | BYOK-only (no keyless oracle — needs a real dataset/runner) | MIT |
| `tau2` | state-based reward: expected DB end-state is a subset of the agent's final state AND all required tool calls were made (proxy for τ²-bench DB+ACTION); reliability reported as pass^k | keyless | 2 | 2 | 100% | MIT |

## What each adapter is

- **`bfcl`** — AST match (function name + arg names/values, type-aware, order-independent). [https://gorilla.cs.berkeley.edu/leaderboard.html](https://gorilla.cs.berkeley.edu/leaderboard.html) · license: Apache-2.0 · keyless self-test: yes.
- **`gaia`** — quasi-exact-match (normalized numeric/string exact match; GAIA arXiv:2311.12983). [https://huggingface.co/datasets/gaia-benchmark/GAIA](https://huggingface.co/datasets/gaia-benchmark/GAIA) · license: CC-BY-4.0 · keyless self-test: yes.
- **`swebench`** — resolved rate: patch applies, then ALL FAIL_TO_PASS pass and ALL PASS_TO_PASS still pass (tests run by the operator's container harness). [https://www.swebench.com](https://www.swebench.com) · license: MIT · keyless self-test: no (BYOK-only).
- **`tau2`** — state-based reward: expected DB end-state is a subset of the agent's final state AND all required tool calls were made (proxy for τ²-bench DB+ACTION); reliability reported as pass^k. [https://github.com/sierra-research/tau2-bench](https://github.com/sierra-research/tau2-bench) · license: MIT · keyless self-test: yes.

