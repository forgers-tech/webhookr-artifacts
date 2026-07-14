---
title: Adopt Infisical as the Webhookr Secret Management Platform — Council Record (Revision 2, DEFERRED)
status: Deferred
owner: thiagojv@gmail.com
council_record_v1: ./2026-07-14-council-record-v1.md
created: 2026-07-14
updated: 2026-07-14
source: /council (reopen §15 — evidence-invalidated assumption)
priority: Low (parked)
labels: [secrets, infisical, sops, deferred, deprioritized, cost, audit, least-privilege, webhookr]
---

# Council Record — Revision 2 (DEFERRED)

> **This is a revision delta, not a rewrite.** It preserves [v1](./2026-07-14-council-record-v1.md) intact and records only what changed: the resolution of the open questions and the resulting decision to **defer the entire initiative**. Read v1 for the full deliberation, ledger, and rationale.

## Execution Trace

| Started | Ended | Duration | Tokens | Revision |
|---------|-------|----------|--------|:--------:|
| 2026-07-14T15:10:00Z | 2026-07-14T17:05:00Z | ~1h55m wall-clock | n/a (reopen delta; runtime usage not separately exposed) | 2 |

> `Started` is immutable from v1. Revision incremented to 2 on this reopen.

## 1. Why the Council reopened (§15)

New evidence invalidated a **material assumption (A-4: "Infisical Cloud cost fits the lean budget")**. After the operator confirmed the three open questions, a **direct verification of the Infisical pricing/site (2026-07-14)** showed the **free tier cannot meet the requirements the operator had just confirmed as real**:

- **No audit logs on Free.** Centralized read-audit is available only from the **Pro** tier (90-day retention). Sources: [infisical.com/pricing](https://infisical.com/pricing), [Secrets Manager Pricing 2026](https://infisical.com/blog/secrets-manager-pricing).
- **Identity cap of 5 total (machine + human) on Free.** The design needs **5 machine identities** (svc, ingest, bff, web + operator) per D-5 — which exhausts the cap alone, leaving none for the human account owner/developers.
- Pro is **~$18 / identity / month → ≈ $90–110 / month** for this design.

This is decisive because **the two capabilities that justified choosing Infisical over the incumbent SOPS — centralized read-audit (F-1's deciding line) and revocable per-workload least-privilege (D-5) — are exactly the ones gated behind the paid tier.** On the free tier, Infisical offers no decisive advantage over SOPS for the operator's actual goal.

## 2. Open questions — resolved

| ID | Resolution | Note |
|----|-----------|------|
| **Q-1** | **Resolved — YES.** Centralized read-audit + revocable least-privilege *are* genuine requirements (ratifies the intent behind D-1). | But they require Pro (see Q-2). |
| **Q-2** | **Resolved — Free tier INSUFFICIENT.** Meeting Q-1 + D-5 requires **Infisical Pro (~$90–110/mo)**. **A-4 is FALSE** for the free tier. The operator **declines a new financial commitment** at this time. | This is the deciding input. |
| **Q-3** | **Resolved — A-1 accepted** (operator retains last-known-good Secret on outage, CONFIRMED-by-source). Moot given the deferral; the empirical outage test is not needed unless the initiative resumes. | — |

## 3. Revised decision

**X-8 (new) — DEFER / DEPRIORITIZE the entire Infisical adoption.** `REJECTED-FOR-NOW`
The initiative does **not** proceed to `/downstream`. The Feature Specification (`../2026-07-14-infisical-secret-platform/…-v1.md`, Gatekeeper 95/100) and this Council decision are **parked**, not deleted.

**Rationale (the "why" to resume from):**
- The operator's real objective is to **centralize environment secrets in one place**. **SOPS+age already satisfies that** — secrets are centralized, encrypted, versioned in the `gitops` repo, and recoverable; there is no plaintext in Git. This was never the gap.
- The genuine gaps Infisical would have closed — **centralized read-audit** and **revocable per-workload least-privilege** — are **paid-tier features** (Pro). They are real requirements (Q-1) but **not urgent enough to justify a new ~$90–110/mo commitment** for a single-founder, lean-budget SaaS today.
- Self-hosting remains rejected (`X-1`) on the single-node **SPOF + bootstrap-circularity** grounds — those are unchanged by the cost finding; cost was never the reason self-host was rejected, so the cost problem does not revive it.

**Decision status changes (from v1):**
- **D-1 (adopt Infisical) → DEFERRED.** Not pursued now.
- **D-2..D-13 → PARKED** with D-1 (design remains valid *if* resumed; nothing to implement now).
- **Supersession of audit finding P0.4 → WITHDRAWN.** Since Infisical is not being adopted, P0.4's recorded direction — *"extend SOPS+age"* — **stands as the status quo** again. SOPS+age remains Webhookr's secret mechanism, unchanged.

## 4. Status quo after deferral (nothing to implement)

- **SOPS+age via ksops** remains the secret mechanism for runtime (K8s) secrets, unchanged.
- Local dev continues on hand-filled `.env` (+ committed `.env.example`).
- The rotate-encryption-key CI job continues on GitHub Actions secrets.
- The Terraform→SOPS manual copy (`INTERNAL_API_TOKEN` etc.) **remains** — the D-10 cleanup is parked with the rest. (It is an independent, low-cost hygiene fix that *could* be done under SOPS alone if desired; noted as a standalone opportunity, not part of this parked initiative.)

## 5. Resume triggers (revisit this when any becomes true)

- **Budget** — a ~$90–110/mo (Pro) line item becomes acceptable, or Infisical changes its free-tier limits (audit + ≥6 identities) — then this initiative resumes **directly at `/downstream`** (the spec is Gatekeeper-approved; only Q-2 blocked it).
- **Compliance** — a SOC2-style / customer requirement makes centralized read-audit non-optional (elevates Q-1 from "real but not urgent" to "mandatory").
- **Team growth** — beyond a single operator, so per-user access control + audit of who-read-what becomes operationally necessary.
- **A cheaper equivalent** — a secret platform (or an Infisical tier) that provides audit + enough identities within budget appears.
- **Self-host revisited** — only if the cluster's single-node **SPOF/HA** story changes (reopen `F-2`/`X-1`); the cost finding alone does **not** justify reopening self-host.

## 6. What was NOT reverted

- The `/council`-persistence tooling change (ADR-007, skills v1.7.0) is **unrelated** and stays.
- v1 of this Council Record, the Feature Spec, and the Gatekeeper Review remain as **historical, resumable artifacts** — this revision parks them, it does not delete them.

## References
- [Council Record v1](./2026-07-14-council-record-v1.md) — full deliberation + ledger.
- [Feature Specification v1](../2026-07-14-infisical-secret-platform/2026-07-14-feature-spec-infisical-secret-platform-v1.md) — parked (Gatekeeper 95/100).
- [Gatekeeper Review v1](../2026-07-14-infisical-secret-platform/2026-07-14-gatekeeper-review-v1.md) — parked.
- Pricing evidence: [infisical.com/pricing](https://infisical.com/pricing), [Secrets Manager Pricing 2026](https://infisical.com/blog/secrets-manager-pricing).
- Status-quo direction restored: [`2026-07-01` audit §P0.4](../2026-07-01-webhookr-architecture-audit/2026-07-01-audit-report-v1.md) ("extend SOPS+age").
