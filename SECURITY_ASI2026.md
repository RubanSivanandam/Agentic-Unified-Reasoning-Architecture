# AURA Security Self-Audit — OWASP Top 10 for Agentic Applications 2026 (ASI01–ASI10)

**Framework audited:** AURA (self-hosted multi-agent framework)
**Standard:** OWASP Top 10 for Agentic Applications 2026 — OWASP GenAI Security Project (Agentic Security Initiative), announced 2025-12-09.
**Audit date:** 2026-07-17
**Verdict legend:** **Covered** = a dedicated control materially addresses the risk · **Partial** = a control exists but leaves a named residual gap · **Gap** = no dedicated control.

> Honesty note: this is a self-audit, not an external certification. Every "Covered" below points at a real file in this repo; every "Partial"/"Gap" names the specific residual so we do not overclaim. Where the official ASI title wording could not be pinned to a single canonical string, the row records the variance (see "ASI list verification" at the end).

---

## Verified ASI01–ASI10 titles

The exact wording of a few titles differs across published summaries (the official PDF is gated behind a download and did not render its list to the fetcher). Titles below are the best-attested form; known variants are noted. Items marked **(title variance)** were mapped to the closest verified category rather than a single fabricated string.

| ID | Title (best-attested) | Known variant wording |
|----|-----------------------|-----------------------|
| ASI01 | Agent Goal Hijack | (consistent across sources) |
| ASI02 | Tool Misuse and Exploitation | "Tool Misuse & Exploitation" |
| ASI03 | Identity and Privilege Abuse | "Agent Identity & Privilege Abuse" *(title variance)* |
| ASI04 | Agentic Supply Chain Vulnerabilities | "Agentic Supply Chain Compromise" *(title variance)* |
| ASI05 | Unexpected Code Execution | (consistent across sources) |
| ASI06 | Memory and Context Poisoning | "Context Management and Retrieval Manipulation" *(title variance)* |
| ASI07 | Insecure Inter-Agent Communication | (consistent across sources) |
| ASI08 | Cascading Failures | "Cascading Agent Failures" *(title variance)* |
| ASI09 | Human-Agent Trust Exploitation | (consistent across sources) |
| ASI10 | Rogue Agents | (consistent across sources) |

---

## Control mapping (line-by-line)

| ASI | Risk | AURA controls (files / issue refs) | Verdict | Notes / residual |
|-----|------|------------------------------------|---------|------------------|
| **ASI01** | Agent Goal Hijack — attacker rewrites objectives/decision logic via content the agent reads | Prompt-injection screen `aura/safety/guardrails.py::scan_input`; typed inter-agent contracts `aura/orchestration/contracts.py` (#30) constrain what a delegated goal may be; formal behaviour bounds `aura/safety/formal.py` + `verifier_formal.py` (#78) assert the agent stays inside declared limits; adversarial suite `aura/evaluation/adversarial.py` (#80) exercises hijack payloads | **Covered** | Input scanning is heuristic/pattern-based, so novel obfuscated injections can still slip the screen; formal bounds + contracts are the backstop that limits damage even on a missed injection. |
| **ASI02** | Tool Misuse and Exploitation — legitimate tools driven in unsafe/unintended ways | Risk-scored tool guard `guardrails.make_tool_guard`; per-agent capability authz `tool_agent._capability_guard` (#119); tool sandbox `aura/tools/sandbox.py` (AST screen + isolated subprocess + rlimit + Windows Job Object cap; WASM tier); outbound tool-output screen `guardrails.screen_tool_result` (#107) | **Covered** | Sandbox posture is strongest on Linux (rlimit); Windows relies on Job Object caps — parity is asymmetric but both tiers are enforced. |
| **ASI03** | Identity and Privilege Abuse — agent acts with more/other privilege than intended | Cryptographic per-agent identity + short-lived scoped credentials `aura/safety/identity.py` / `AURA_AGENT_IDENTITY` (#123); per-agent capability authz (#119); HITL approvals (tenant/arg/time-bound, one-time consumable) `aura/safety/approvals.py` | **Partial** | Identity signatures are **symmetric / in-process** today; there is no cross-boundary asymmetric (Ed25519) attestation yet, so a compromised in-process component could mint identities. Authz + one-time approvals limit blast radius. Upgrade tracked. |
| **ASI04** | Agentic Supply Chain Vulnerabilities — compromised deps, frameworks, tools, or model artifacts | Egress allow-list `aura/safety/egress.py` (#120) limits where tools/deps can reach; tool sandbox constrains untrusted tool code; hash-chained audit trail `aura/compliance/audit.py` (#60) records provenance of actions | **Partial** | No dedicated dependency/model-artifact integrity pipeline (SBOM, pinned-hash verification of third-party tool/model packages, signed releases). Egress + sandbox reduce exploitation impact but do not verify supply-chain provenance at install time. **Honest gap in the supply-chain-provenance layer.** |
| **ASI05** | Unexpected Code Execution — NL paths that reach RCE | Tool sandbox `aura/tools/sandbox.py` (AST static screen → isolated subprocess → rlimit / Windows Job Object → WASM tier); risk-scored tool guard; capability authz (#119) | **Covered** | AST screen is a denylist-style static gate; the isolated-subprocess + resource-cap + WASM tiers are the containment guarantee if the screen is bypassed. |
| **ASI06** | Memory and Context Poisoning — false/malicious data implanted into persistent memory/context | Memory-screening at ingest `guardrails.screen_memories`; memory provenance / quarantine `aura/memory/integrity.py` (#17); memory integrity hashing `aura/memory/integrity.py` (#124) | **Covered** | Provenance quarantine + integrity hashing detect tampering and untrusted-origin memories; screening is still heuristic for semantically-clean poison, so quarantine-by-provenance is the primary guarantee. |
| **ASI07** | Insecure Inter-Agent Communication — auth/integrity failures in multi-agent messaging | Typed inter-agent contracts `aura/orchestration/contracts.py` (#30); verified A2A delegation `aura/protocols/a2a.py` + `aura/protocols/a2a_discovery.py` (#49/#50); per-agent identity `aura/safety/identity.py` (#123) | **Partial** | Message typing + verified delegation + identity are in place, but identity verification is symmetric/in-process (same residual as ASI03). No wire-level asymmetric mutual auth across a trust boundary yet. |
| **ASI08** | Cascading Failures — one agent's failure propagates through tools/memory/other agents | Blast-radius replay gate `aura/evaluation/blast_radius.py` (#125) bounds action VOLUME (tools/agents/LLM calls) of a recorded trace against a declared cap — the standard's own cascading-failure mitigation; backpressure (#121); egress allow-list `aura/safety/egress.py` (#120) | **Partial** | Blast-radius gate is **being built now** (#125): it gates on *replayed/recorded* traces against a declared cap, not yet a live runtime circuit-breaker on in-flight cascades. Backpressure + egress cap concurrent/outbound amplification in production; the replay gate is the pre-merge guardrail. |
| **ASI09** | Human-Agent Trust Exploitation — humans over-trust agent output/recommendations | HITL approvals (tenant/arg/time-bound, one-time consumable) `aura/safety/approvals.py`; hash-chained audit trail `aura/compliance/audit.py` (#60) gives humans a verifiable record before/after acting; outbound tool-output screen `guardrails.screen_tool_result` (#107) | **Partial** | Approvals force a human decision on high-risk actions and audit makes claims checkable, but there is no dedicated control for calibrating/undermining *misplaced* human trust (e.g. confidence signalling, provenance surfacing in the UI). Process control, not a UX-trust control. |
| **ASI10** | Rogue Agents — agent exhibits misalignment, concealment, or self-directed behaviour | Formal behaviour bounds `aura/safety/formal.py` + `verifier_formal.py` (#78); adversarial testing `aura/evaluation/adversarial.py` (#80); capability authz (#119); blast-radius gate (#125); hash-chained audit `aura/compliance/audit.py` (#60) | **Partial** | Formal bounds + authz + blast-radius contain what a rogue agent *can do*, and audit detects after the fact. There is **no continuous external red-team CI gate** and no live behavioural-drift/anomaly detector running in production — detection of emergent misalignment is offline (adversarial suite) rather than continuous. **Honest gap.** |

---

## Summary of verdicts

| Verdict | ASI rows |
|---------|----------|
| **Covered** | ASI01, ASI02, ASI05, ASI06 |
| **Partial** | ASI03, ASI04, ASI07, ASI08, ASI09, ASI10 |
| **Gap** (no dedicated control) | none stand fully uncovered; the two most material *honest gaps* are the supply-chain-provenance layer inside **ASI04** and the continuous-red-team / live-drift detection inside **ASI10**. |

### Named honest gaps (do not overclaim)
1. **Identity is symmetric / in-process** (affects ASI03, ASI07) — no cross-boundary Ed25519/asymmetric attestation yet; a compromised in-process component could forge agent identity. Mitigated, not eliminated, by capability authz + one-time approvals.
2. **No supply-chain provenance pipeline** (ASI04) — no SBOM, pinned-hash verification, or signed-release check for third-party tools/models at install time. Egress + sandbox limit exploitation, not provenance.
3. **Blast-radius gate is replay/pre-merge, not a live runtime circuit-breaker** (ASI08) — #125 is under construction and bounds recorded traces; in-flight cascade interruption in production still leans on backpressure + egress.
4. **No continuous external red-team CI gate and no live behavioural-drift detector** (ASI10) — rogue/misalignment detection is offline via the adversarial suite, not continuous.

---

## Reproducibility note

Regenerate/verify this mapping from a clean checkout: `cd /d/aura/aura_project && git branch` (expect `dev`), confirm each cited control file exists (`aura/safety/guardrails.py`, `identity.py`, `egress.py`, `approvals.py`, `formal.py`; `aura/tools/sandbox.py`; `aura/evaluation/adversarial.py`, `blast_radius.py`; `aura/memory/integrity.py`; `aura/orchestration/contracts.py`; `aura/protocols/a2a.py`, `a2a_discovery.py`; `aura/compliance/audit.py`), then re-run the ASI-list web verification below.

## Sources for the ASI01–ASI10 list

- OWASP GenAI Security Project — OWASP Top 10 for Agentic Applications for 2026 (official landing page; list gated in downloadable PDF): https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
- OWASP GenAI Security Project — Agentic Security Initiative: https://genai.owasp.org/initiatives/agentic-security-initiative/
- DeepTeam — OWASP Top 10 for Agents 2026 (per-item titles): https://www.trydeepteam.com/docs/frameworks-owasp-top-10-for-agentic-applications
- NeuralTrust — OWASP Agentic AI Top 10: Every Risk Explained (per-item titles + descriptions): https://neuraltrust.ai/blog/owasp-agentic-ai-top-10
- Palo Alto Networks — OWASP Top 10 for Agentic Applications 2026 Is Here (partial title confirmation): https://www.paloaltonetworks.com/blog/cloud-security/owasp-agentic-ai-security/

*Title-variance caveat:* the official PDF did not render its list to the fetcher; titles were cross-checked across the three third-party sources above. Where wording diverged (ASI03, ASI04, ASI06, ASI08) the row records both forms and maps to the closest verified category rather than asserting a single canonical string.
