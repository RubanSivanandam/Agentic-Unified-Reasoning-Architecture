# RESULTS — Memory 2.0 (Wave 17)

*A measured receipt for AURA's memory layer. Every number here is re-derived from
a live keyless run by `tests/test_memory_receipt.py`, so this document cannot
silently drift from the code.*

Wave 17 closes the three SimpleMem stages ([arXiv:2601.02553](https://arxiv.org/abs/2601.02553),
`aiming-lab/SimpleMem`) that Sprint #39's consolidation left open — implemented
**dependency-free on AURA's own embeddings**, CPU-only, no GPU, BYOK, and each
one **off by default** (the flags below are opt-in; unset ⇒ behaviour is
byte-identical to before, test-proven).

## What shipped

| Sprint | SimpleMem stage | What it adds | Flag |
|--------|-----------------|--------------|------|
| **#147** | 1 — measured compression | A deterministic **fidelity meter** on the conversation compressor + an opt-in fail-safe that keeps originals rather than ship a lossy summary | `AURA_CONTEXT_MIN_FIDELITY` |
| **#148** | 2 — online synthesis | Consolidation representative can be a **synthesised unified abstract**, not just the longest member — **fidelity-guarded**, falls back to #39 | `AURA_MEM_SYNTHESIZE` |
| **#149** | 3 — intent-aware retrieval | Recall **scope sized to query intent** (narrow lookup vs broad enumeration), bounded | `AURA_RETRIEVAL_SCOPE=intent` |

## Measured numbers (live, keyless)

**Stage 1 — compression fidelity meter (#147).** On the golden fixture, a
faithful summary scores **0.609** salient-term coverage while a vague one scores
**0.0** — the meter cleanly separates a fact-preserving summary from a
fact-dropping one (golden `3 / 3`). The fail-safe rejects any summary below the
configured floor and keeps the originals.

**Stage 2 — online semantic synthesis (#148).** With synthesis enabled, a
3-member duplicate cluster consolidates to **1** synthesised representative
(`synthesized = 1`) that still answers recall. Because the synthesis is
**fidelity-guarded** (must preserve ≥ `AURA_MEM_SYNTHESIZE_MIN_FIDELITY`, default
0.8, of the cluster's salient terms), a lossy synthesis is rejected and the
merge degrades to #39's longest-text representative — **synthesis can never do
worse than #39**, and the golden memory recall slice holds `4 / 4` (compression
ratio **0.6**).

**Stage 3 — intent-aware retrieval scope (#149).** The deterministic classifier
sizes scope to intent: a pointed id/factoid lookup gets **k = 2**, an
enumeration/overview gets **k = 8** at the same base — a strictly wider net only
when the query asks for one (golden `6 / 6`, bounded by
`AURA_RETRIEVAL_SCOPE_MAX`).

## Honest scope (Law 1) — what is **not** done

- The compression fidelity meter proves the **mechanism** — a fact-dropping
  summary *is* detected. Whether a *real* model writes good summaries is a **BYOK
  claim** measured with a key, **not asserted** by the keyless gate.
- Online synthesis quality likewise depends on the BYOK summariser; the gate only
  proves the fidelity guard and the never-worse-than-#39 fallback. The keyless
  path (no synthesiser) is unchanged from #39.
- The retrieval-intent classifier is a **heuristic over query shape**, not a
  learned/semantic intent model — deliberately, to stay CPU-only and no-train. It
  will misclassify adversarially-phrased queries; it is a scope *default*, not a
  guarantee, and callers may always pass an explicit `k`.
- Salient-term coverage is a **proxy** for recall, not recall itself; it rewards
  keeping names/numbers/nouns, which is what later turns key on, but a summary
  could in principle keep the terms and still garble the relation.

## Reproduce

```bash
AURA_LLM=stub python -m aura.evaluation.golden_ingest --report   # context 3/3 · scope 6/6 · memory 4/4
AURA_LLM=stub python -m pytest tests/test_context_fidelity.py tests/test_memory_synthesis.py tests/test_retrieval_scope.py -q
```
