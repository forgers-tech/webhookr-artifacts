---
feature: Automatically Apply Terraform Changes from GitHub Actions
reviewer: Gatekeeper
review_date: 2026-07-14
status: 🟡 Approved with Recommendations
score: 93
---

# Gatekeeper Review

> Quality gate for deciding whether a Feature Specification is ready for implementation.

## Execution Trace

| Started | Ended | Duration | Tokens | Revision |
|---------|-------|----------|--------|:--------:|
| 2026-07-14T19:45Z | 2026-07-14T19:45Z | ~0 (single review pass) | n/a (runtime usage not exposed) | 1 |

## 1. Verdict

### Status
- [ ] ✅ Approved
- [x] 🟡 Approved with Recommendations
- [ ] ❌ Changes Required

### Readiness Score

| Area | Max | Score |
|------|----:|------:|
| Product | 25 | 24 |
| Business Rules | 25 | 23 |
| Software Architecture | 25 | 23 |
| Tech Leadership | 25 | 23 |
| **Total** | **100** | **93** |

### Decision Rationale
The spec is faithful to Council Record v2: **every `D-1..D-14` is expanded, no requirement is invented, no rejected `X-#` is revived, and the Rev-2 resolutions (Q-1..Q-4) come from the ledger — the spec surfaces the still-open `A-1/A-3/A-5` as explicit `TODO:` rather than silently answering them.** The traceability audit is clean on all five checks. Nothing blocks implementation. The deductions are authoring-clarity gaps, not decision defects: (a) the spec does not state whether the **ungated firebase-dev** target uses the two-job age-sealed hand-off or a simpler single-job flow — which matters because that means the load-bearing **D-4 sealed-plan mechanism is first exercised on firebase-prod, not on the firebase-dev canary**; (b) the sealing tool for a **binary** `tfplan` is described as both "age-seal" and "`sops -d`" without pinning which; (c) the mono-root plan-time `local-exec` dependency (**A-1**) is only validated at the late plan-only stage. All are Important Risks / Minors, appropriately handled with recommendations.

## 2. Blocking Issues

No blocking issues identified.

## 3. Important Risks

| ID | Area | Risk | Mitigation |
|----|------|------|------------|
| IR-1 | Functional / Delivery | The **D-4 age-sealed saved-plan hand-off** is the make-or-break mechanism for `applied == approved`, but the spec's canary (firebase-dev) is **ungated/fast-path** — so a two-job gated hand-off is first exercised only at **firebase-prod**. If firebase-dev runs a simpler single-job flow, the sealing/gating path reaches production (firebase-prod) untested. | Make the spec explicit: either firebase-dev also runs the two-job sealed flow (so the canary proves D-4), or add a Stage requirement that the **sealed-plan + gate + decrypt** path is exercised and asserted on firebase-prod **before** the mono-root apply. Add an e2e test for the decrypt step specifically. |
| IR-2 | Technical | The plan bound to the sealed artifact is a **binary** `tfplan`; the spec uses "age-seal" and "`sops -d`" interchangeably. `sops` targets structured text, not an opaque binary — the wrong choice silently breaks the hand-off. | Pin the mechanism: encrypt the binary `tfplan` with **`age -r <recipient>`** (the age key already chosen in Q-2), decrypt with `age -d` on the trusted runner. Reserve `sops` for the structured third-party secret bundle. Authoring clarification (Upstream), not a decision change. |
| IR-3 | Delivery / Ops | BR-015 condition (1) + Steps 1/5 depend on the **out-of-band bootstrap** (WIF pool/SAs + GitHub App→SOPS) that is **operator prep not yet done**; the GitHub App key currently lives only in the operator `.env` (bootstrap copy). Implementation of Steps 5–7 is gated on this landing. | DoR already lists it; track the bootstrap as an explicit predecessor task with an owner. Ensure the CI copy of the App key + age key is in SOPS+age (not `.env`) and **not** managed by the TF root the pipeline applies (X-3). |
| IR-4 | Technical (A-1) | The mono-root plan executes `github_org_settings.tf`'s `data.external` (`bash+curl+python3`) at plan time; runner-tool presence (**A-1**) is only checked at the late plan-only stage. A missing tool fails the first mono-root plan. | Add an early preflight assertion for `python3`/`curl` on the runner (cheap, fail-fast) rather than discovering it at Stage B. |

## 4. Traceability Audit

| Check | Findings |
|-------|----------|
| Lost decisions (`D-#` in Ledger, absent from spec) | **None** — D-1..D-14 all expanded (D-1 §2/§7, D-2 BR-002/§6, D-3 BR-003, D-4 BR-004/§5 ADR, D-5 BR-005/§5, D-6 BR-006, D-7 BR-007/EF-02, D-8 BR-008/§4, D-9 BR-009, D-10 BR-010/§5 rollback, D-11 BR-011/EF-01, D-12 BR-012/EF-04-05, D-13 BR-013/§6, D-14 BR-014). |
| Invented requirements (spec claim with no `D-#`/`US-#`) | **None** — BR-015 is authored from the ledger's owned blocking gate (cites D-2/D-13); all BR-# and NFRs carry `⟶ D-#`. N/A sections (Data Model, API, Events, Message, Backend, Frontend) are justified, not invented. |
| Decision drift (spec contradicts a cited `D-#`) | **None** — the trigger model (dispatch-first mono-root, firebase-dev auto, on-merge end-state), the no-PR-plan rule, the WIF+App-token+SOPS credential model, and the advisory-vs-hard guard classification all match D-2/D-3/D-5/D-12 and the Rev-2 deltas. |
| Silently resolved open questions (`Q-#`) | **None** — Q-1..Q-4 were resolved in **Council Record Rev 2** (operator input + validation), not by the spec; the spec cites those resolutions. The still-open **A-1/A-3/A-5** are surfaced as explicit `TODO:` in §2 and the DoR. |
| Revived rejects (`X-#` reintroduced) | **None** — X-1..X-9 appear only in §5 "Alternatives Considered" labelled as rejected/deferred, consistent with the ledger. |

Routing: no decision-level defect. The two authoring clarifications (IR-1 job model, IR-2 sealing tool) are **Upstream**-routable refinements if the team wants them folded in before build; neither blocks.

## 5. Review Notes

### Product (24/25)

| Item | Status |
|------|:------:|
| Problem is clearly defined | ☑ |
| Target user or actor is identified | ☑ |
| Objective is clearly defined | ☑ |
| Business value is explicit | ☑ |
| Success criteria or metrics are defined | ☑ |
| Scope is clear | ☑ |
| Out of Scope is clear | ☑ |
| Assumptions are documented | ☑ |
| Product risks are documented | ☑ |
| Open questions are identified | ☑ |

**Missing Information:** No missing product information identified.
**Notes:** Success metrics are defined but two (workstation-vs-CI apply count; secret-in-log count) lack a named sink/dashboard — fine for an infra-delivery feature; the audit trail is the GH deployment record. −1 for the soft metric plumbing.

### Business Rules (23/25)

| Item | Status |
|------|:------:|
| Main flow is defined | ☑ |
| Alternative flows are defined | ☑ |
| Error flows are defined | ☑ |
| Business rules are explicit | ☑ |
| Validation rules are defined | ☑ |
| Permissions are defined | ☑ |
| Preconditions are clear | ☑ |
| Postconditions are clear | ☑ |
| Edge cases are covered | ☑ |
| State transitions are defined when applicable | ☑ |
| Default behaviors are defined | ☑ |
| Idempotency requirements are defined when applicable | ☑ |
| Ordering requirements are defined when applicable | ☑ |
| Multi-tenant constraints are considered when applicable | ☑ (N/A — single org, noted) |

**Missing Information:** Whether the **ungated firebase-dev** flow is two-job-sealed or single-job (IR-1) — the one functional ambiguity.
**Notes:** 15 BR-# each `⟶ D-#`; error flows (EF-01..06) are thorough (stale lock, partial apply, stale plan, drift, no-destroy). Idempotency (convergent re-apply, EF-03) and ordering (per-prefix concurrency) covered. −2 for IR-1.

### Software Architecture (23/25)

| Item | Status |
|------|:------:|
| Architecture is documented | ☑ |
| Architectural decision is explicit | ☑ |
| Alternatives and trade-offs are documented | ☑ |
| Impacted repositories/services are identified | ☑ |
| Service responsibilities are clear | ☑ |
| Data ownership is clear | ☑ (Terraform state in GCS, unchanged) |
| Data model changes are described when applicable | ☑ (N/A — justified) |
| API contracts are defined when applicable | ☑ (N/A — justified) |
| Event contracts are defined when applicable | ☑ (N/A — justified) |
| Message contracts are defined when applicable | ☑ (N/A — justified) |
| Integration boundaries are clear | ☑ |
| Consistency model is defined when applicable | ☑ (state lock + saved-plan) |
| Idempotency strategy is defined when applicable | ☑ |
| Retry strategy is defined when applicable | ☑ |
| Failure scenarios are considered | ☑ |
| Security considerations are documented | ☑ |
| Observability requirements are documented | ☑ |
| Performance impact is considered | ☑ |
| Scalability impact is considered | ☑ |
| Migration strategy is defined when applicable | ☑ |
| Rollback strategy is defined | ☑ |
| Backward compatibility is considered | ☑ |

**Missing Information:** The concrete sealing primitive for the binary `tfplan` (IR-2).
**Notes:** ADR restates the decided `D-#` without inventing; alternatives map to X-1..X-4/X-7; rollback is honestly "forward-fix". −2 for the age/sops ambiguity on a binary artifact.

### Tech Leadership (23/25)

| Item | Status |
|------|:------:|
| Implementation strategy is defined | ☑ |
| Recommended implementation order is clear | ☑ |
| Task decomposition is clear | ☑ |
| Tasks are small enough for reviewable PRs | ☑ |
| Cross-service dependencies are identified | ☑ |
| Team or service ownership is clear | ☑ (single maintainer) |
| Delivery risks are documented | ☑ |
| Technical dependencies are identified | ☑ |
| Feature flags are considered when applicable | ☑ (`TF_APPLY_ENABLED`) |
| Migration sequence is defined when applicable | ☑ |
| Backward compatibility is considered | ☑ |
| Rollout strategy is defined | ☑ |
| Rollback strategy is defined | ☑ |
| Testing strategy is clear | ☑ |
| Manual validation steps are defined when applicable | ☑ |
| Observability needed for release validation is defined | ☑ |
| Definition of Ready is complete | ☑ |
| Definition of Done is complete | ☑ |
| Estimate is provided | ☑ (GG) |

**Missing Information:** An explicit predecessor task for the out-of-band bootstrap as a hard gate on Steps 5–7 (IR-3) — implied by DoR but not a task row.
**Notes:** Testing maps a concrete observable + negative test to each `US-#`; the US-5 platform-denial test and the no-secret-in-logs grep gate are strong. −2 for IR-3 sequencing visibility + IR-1's missing decrypt-path test.

## 6. Open Questions

| ID | Question | Impact | Owner |
|----|----------|--------|-------|
| A-1 | Does the GitHub-hosted runner image ship python3+curl for the plan-time `local-exec`? | First mono-root plan fails if absent | operator/DevOps |
| A-3 | Exact GitHub App permission set for the mono-root org operations | App-token path completeness | operator/Security |
| A-5 | Has any third-party provider added GitHub OIDC since 2026-07? | Could move a secret off long-lived storage | operator |

*(All correctly surfaced in the spec as `TODO:`; none is blocking.)*

## 7. Recommended Next Actions
1. Fold IR-1 + IR-2 into the spec as a small authoring revision (firebase-dev job model + `age -r` sealing for the binary plan; ensure firebase-prod exercises the decrypt path before mono-root). Optional Upstream touch-up; not required to start.
2. Track the **out-of-band bootstrap** (WIF + GitHub App→SOPS + Environments) as the explicit predecessor task gating Steps 5–7 (IR-3); move the App key from `.env` into SOPS+age.
3. Add the early `python3`/`curl` preflight (A-1) and confirm the App permission set (A-3) during Step 1/2.

## 8. Final Recommendation

`Approved with recommendations. Implementation can start, but the listed risks should be tracked.`

Phase 0/1 (bootstrap root, reusable `_tf-apply.yml`, firebase-dev canary) can begin now. The IR-1/IR-2 clarifications are cheap and best folded in before the first **gated** (firebase-prod) apply; the mono-root apply remains governed by the BR-015 blocking gate regardless.
