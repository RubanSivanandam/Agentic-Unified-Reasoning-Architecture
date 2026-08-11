# AURA Brain 2.0 — Formal Ontology Grounding, receipt (Wave 15)

The same "prove it, don't claim it" discipline as [SECURITY_ASI2026.md](SECURITY_ASI2026.md)
and [RESULTS_RELIABILITY.md](RESULTS_RELIABILITY.md), applied to the ontology. Every number
below is produced by a **keyless, reproducible** run over the example ontology
(`aura.knowledge.ontology.example_ontology`); regenerate with the snippet at the end.

## What Brain 2.0 adds, and the gap it closes

Before Wave 15 the ontology the ingest pipeline builds had **no formal semantic layer** —
`OntologyProposal` is a Pydantic **shape** check (valid JSON of the right form), not a
**semantic** one. Wave 15 makes it a real, queryable, validatable OWL/RDF graph:

| Sprint | Capability | Receipt (example ontology) |
|--------|------------|----------------------------|
| **#137** | OWL/RDF export (`ontology_rdf.py`) — `owl:Class`/`owl:DatatypeProperty`/`owl:ObjectProperty` in Turtle, `aura:` namespace | **29 RDF triples**, round-trips through rdflib (`ok: True`) |
| **#138** | SPARQL competency questions (`competency.py`) — 7 domain-agnostic questions | **7 / 7 pass** (`competency_report`) |
| **#139** | SHACL integrity validation (`shacl.py`) via pySHACL | valid ontology **conforms**; a seeded **dangling relation is caught** (0 false-accepts) |

## Honest scope (Law 1)

- **Standard W3C vocabulary, not AgentO's agent classes.** AURA's ontology is a *domain*
  schema (Customer/Order/…), so the export uses **RDFS/OWL** — the correct choice for an
  arbitrary domain. **AgentO** (ESWC 2026) and **AIAO v2.0** model *agentic systems*;
  aligning a domain schema to AgentO's `Agent`/`Task`/`Workflow` classes would be
  **mislabeling**, so it is deliberately **not** done. AgentO/AIAO are cited as the
  vocabulary methodology (competency questions, formal grounding), not as a schema AURA's
  domain terms pretend to instantiate.
- **No OWL DL reasoner.** pySHACL's bundled OWL-RL profile gives the practical integrity
  checks at zero extra dependency; a full DL reasoner (HermiT/Pellet) needs a JVM, which
  fails the CPU-only / free-OSS-first filter — **not shipped, by design**.
- **No external triple-store.** rdflib's in-memory SPARQL is CPU-fine at AURA's scale; no
  Jena/GraphDB.
- **The SHACL shapes are domain-agnostic integrity checks** (no dangling relation refs,
  every class labeled). *Domain* constraints are the **operator's** responsibility via
  `AURA_ONTOLOGY_SHAPES` — AURA core stays general-purpose, no hardcoded domain rules.

## Dependency + gating posture

- The OWL/RDF **export is pure standard library** (deterministic Turtle), so the keyless
  gate stays zero-dependency. `rdflib` + `pySHACL` (the `[ontology]` extra, both BSD/
  Apache-2.0, pure-Python) power validation + SPARQL; tests `importorskip` them and CI
  installs the extra, so the round-trips and shape checks run on both py3.11 and py3.12.
- Everything is **read-only and opt-in**: the `ontology_rdf` / `ontology_query` /
  `ontology_validate` tools register only behind `AURA_ONTOLOGY_RDF=enabled`; the SHACL
  acceptance hook is **fail-open** behind `AURA_ONTOLOGY_SHACL` (`strict` to block). Off by
  default ⇒ ingest/retrieval are byte-unchanged (golden-set 5/5 confirmed across the wave).

## Reproduce

```python
from aura.knowledge.ontology import example_ontology
from aura.knowledge.ontology_rdf import to_turtle, validate_turtle
from aura.knowledge import competency, shacl

o = example_ontology()
print(validate_turtle(to_turtle(o)))                 # {'ok': True, 'triples': 29, ...}
print(competency.competency_report(o))               # passed 7 / 7
print(shacl.validate_ontology(o)["conforms"])        # True
```

See also: [ROADMAP.md](ROADMAP.md) (Wave 15 shipped entries), and the frontier sources —
AgentO (ESWC 2026), AIAO v2.0, rdflib (BSD), pySHACL (Apache-2.0) — all re-verified by live
web search 2026-07-18.
