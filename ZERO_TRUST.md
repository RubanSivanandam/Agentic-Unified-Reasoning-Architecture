# AURA Zero-Trust Posture — Anthropic "Zero Trust for AI Agents"

**Framework:** AURA (self-hosted, CPU-only, BYOK multi-agent framework)
**Reference:** Anthropic, *Zero Trust for AI Agents* — the tiered model (Foundation →
Advanced → Leading) for treating every agent, tool call, memory, and message as
untrusted-by-default.
**Assessed:** 2026-07-17 (Wave 12, Sprints #119–#125). Keyless, re-checkable.

> Honesty note (Law 1): this is a self-assessment against a published model, not an
> external certification. Every "met" row points at a real file and a real test in this
> repo; every residual is named, not hidden. AURA claims the **Foundation tier** for
> identity, authorization, egress, and memory integrity — and explicitly does **not**
> claim the Advanced tier (mTLS / hardware-rooted identity / continuous red-team CI),
> which needs an enterprise deployment target.

---

## The zero-trust pillars, mapped to AURA controls

| Pillar | AURA control | Where | Default |
|--------|--------------|-------|---------|
| **Identity** — every agent is uniquely, cryptographically identified | Per-instance `AgentIdentity` (stdlib HMAC root; Ed25519 a documented opt-in) issuing **short-lived, capability-scoped, signed credentials** | `aura/safety/identity.py` (#123) | opt-in `AURA_AGENT_IDENTITY`; minted always, enforced when on |
| **Authorization** — least privilege, enforced on every action | `ToolAgent._capability_guard` self-enforces the agent's `tool_names` scope on **every** run, composed before any caller guard; deny-by-default | `aura/agents/tool_agent.py` (#119) | always on for scoped agents |
| **Network egress** — no unapproved outbound | Central `check_egress` allow-list (`AURA_EGRESS_ALLOW`), exact host or `*.sub`, enforced before every webhook/notification transport | `aura/safety/egress.py` (#120) | deny-by-default when set |
| **Credential scope + TTL** — no long-lived, broad grants | Signed credentials carry a frozen tool scope + a finite TTL (`AURA_AGENT_CRED_TTL`); verified on each tool call; non-finite TTL clamped | `aura/safety/identity.py` (#123) | 300s default |
| **Memory integrity** — recalled state is tamper-evident | Content digest (HMAC-SHA256 keyed, else SHA-256) stamped at `remember()`, re-verified at every `recall()`; a tampered memory is quarantined + audited | `aura/memory/integrity.py` (#124) | opt-in `AURA_MEMORY_INTEGRITY`; `_STRICT` closes the strip-to-downgrade bypass |
| **Untrusted tool output** — data, never instructions | `screen_tool_result` (the C4/#107 injection screen) annotates/strips tool output before it re-enters the model | `aura/safety/guardrails.py` (#107) | `AURA_TOOL_RESULT_SCREEN` |
| **Blast-radius containment** — a compromise can't cascade unbounded | `blast_radius.gate()` replays a recorded trace read-only and **fails closed** if its action volume exceeds a declared cap | `aura/evaluation/blast_radius.py` (#125) | opt-in; fail-closed on breach or unresolvable trace |
| **Auditability** — every security decision is recorded | Hash-chained audit log; identity mint, credential check, screen, quarantine, blast-radius breach all emit events | `aura/compliance/audit.py` (#60) | `AURA_AUDIT` |

## Tier claim

- **Foundation tier — met (test-proven).** Unique per-agent identity, least-privilege
  authorization enforced inside every run, scoped+short-lived credentials, deny-by-default
  egress, tamper-evident memory, and a fail-closed blast-radius gate all exist behind
  interfaces with keyless tests. Each was hardened after an adversarial review (see the
  per-sprint "Hardened after review" notes in [ROADMAP.md](ROADMAP.md)).
- **Advanced / Leading tiers — not claimed.** mTLS / hardware-rooted (HSM) identity,
  cross-trust-boundary delegation with asymmetric signatures, and a continuous red-team CI
  gate are named as the deliberate gaps — they require an enterprise target and are not a
  single keyless sprint. The Ed25519 identity path is stubbed as the documented upgrade.

## Honest residuals

- In one process, the signed credential is a self-signed TTL+scope wrapper; its full
  zero-trust value is realized only across a trust boundary (A2A / delegation) — future.
- Memory-integrity key rotation is an operator responsibility (documented hazard).
- Egress and identity are **opt-in** so the keyless default stays reproducible; a
  production deployment is expected to turn them on (the flags exist precisely so the
  posture is a switch, not a rewrite).

See also: [SECURITY_ASI2026.md](SECURITY_ASI2026.md) (OWASP Top 10 for Agentic Apps 2026),
[MCP_CURRENCY.md](MCP_CURRENCY.md) (MCP 2026-07-28 protocol currency),
[RESULTS_RELIABILITY.md](RESULTS_RELIABILITY.md) (reliability + cost receipts).
