---
artifact: adr-terraform-cd-pipeline
topic: terraform-cd-pipeline
date: 2026-07-15
version: 1
source: manual
status: Approved
supersedes-decision: terraform/CLAUDE.md#applying-changes-via-make
---

# ADR-008: Terraform is applied from GitHub Actions (phased, human-gated CD)

## Status

Approved + **shipped & proven end-to-end** (2026-07-15). Merged: [#105](https://github.com/forgers-tech/terraform/pull/105) tfvars, [#107](https://github.com/forgers-tech/terraform/pull/107) bootstrap, [#109](https://github.com/forgers-tech/terraform/pull/109) actionlint, [#110](https://github.com/forgers-tech/terraform/pull/110) workflows, [#111](https://github.com/forgers-tech/terraform/pull/111) SOPS bundle, [#112](https://github.com/forgers-tech/terraform/pull/112)/[#113](https://github.com/forgers-tech/terraform/pull/113)/[#114](https://github.com/forgers-tech/terraform/pull/114) canary fixes. All three targets ran green (see **As-built**). `X-# / BR-015` in the [council record](../2026-07-14-terraform-auto-apply-council/2026-07-14-council-record-v2.md).

Realizes the Decision Ledger of the **[terraform-auto-apply council v2](../2026-07-14-terraform-auto-apply-council/2026-07-14-council-record-v2.md)** (D-1..D-14, BR-015) and its **[feature spec v2](../2026-07-14-terraform-auto-apply/2026-07-14-feature-spec-terraform-auto-apply-v2.md)**. This ADR records the *why*; those artifacts hold the decisions and the implementation contract. **Supersedes** the prior operating decision in `terraform/CLAUDE.md` — "There is no CD pipeline that applies Terraform" — and extends [ADR-003](../2026-07-05-infra-security-matrix/2026-07-05-adr-infra-security-matrix-v1.md) with a new, explicitly-accepted CI trust-boundary crossing.

## Context

Terraform for all forgers-tech infrastructure was applied **manually** from an operator workstation (`make apply`) whose gitignored `.env` held every provider secret. That posture was deliberate (ADR-003's threat model records "CI has no production access"), but it ties infra delivery to one machine, risks the applied change diverging from the reviewed one, weakens auditability, and lets repo config and live infra drift. A Critical-depth `/council` (7 independent specialists, 5 negotiation rounds, 3 challenge pairs) evaluated automating apply and concluded the pain (workstation dependency, consistent creds, auditability) is real but that **naive auto-apply-to-prod is unsafe** for this estate — a single apply concentrates the org's most-privileged credentials (GitHub `admin:org`, Cloudflare DNS, Hetzner, Neon, Redis Cloud, Tailscale, Grafana admin) and has no clean rollback.

## Decision

Apply Terraform from **GitHub Actions**, as a **phased, human-gated** pipeline — not lights-out auto-apply:

1. **Trigger (D-2).** End-state = apply on merge to `main` behind a GitHub **Environment required-reviewer** gate (the reviewer approval *is* the card's "required approval condition"; the repo has `required_approving_review_count = 0`). Phase-1 the mono-root runs via `workflow_dispatch` and graduates to on-merge only after the BR-015 gate holds. **firebase-dev** applies on-merge (ungated) as the canary. Apply **never** runs on `pull_request` (the credential-free CI is unchanged).
2. **applied == approved (D-4).** Enforced as content-equality on the post-squash commit via a same-target saved plan. Because the Environment gate is a **job boundary**, gated targets use a two-job flow with an **age-sealed** `tfplan` handed between jobs (never a plaintext artifact); the ungated firebase-dev is single-job.
3. **Credentials (D-5).** **Workload Identity Federation** (keyless) for the GCS state backend — no static SA key; an **ephemeral GitHub App installation token** for `admin:org` — retiring the static org PAT (advances audit **P0.1**); the OIDC-less third-party providers stay long-lived but live in an **age-encrypted SOPS bundle** decrypted in-job, with the age key the only Environment secret. GitHub-hosted **ephemeral** runners; **no persistent self-hosted runner** (rejected — a standing tailnet-resident target).
4. **Safety (D-6/D-7/D-12).** Per-state-prefix concurrency (`cancel-in-progress: false`) layered on GCS native locking; an in-CI guarded force-unlock for stale locks (no workstation); the pipeline's own identity bootstrapped **out-of-band** (a separate `bootstrap/` root) so a bad apply can't lock the pipeline out of its own credentials; HARD guards — no-destroy/replace-without-ack and a clean-baseline empty-plan gate — replace the operator's `-target` discipline.
5. **Blocking gate (BR-015).** The mono-root's first apply is authorized only when ALL hold: (1) the P0.1 `admin:org` lateral path is closed (App token + out-of-band creds); (2) a zero-drift baseline is proven by a mono-root plan-only-to-green stage; (3) the reviewer approves the saved plan of the exact squash commit.
6. **Reversibility (D-10).** A disable switch (`TF_APPLY_ENABLED`) and the manual `make apply` break-glass are preserved. "Rollback" is documented honestly as **forward-fix** — not guaranteed to restore destroyed DNS/DB/server resources.

## Security Matrix / Threat Model impact (extends ADR-003)

This introduces a new, deliberately-accepted CI trust-boundary crossing — the analogue of ADR-003's `rotate-encryption-key` runner holding `DATABASE_URL`:

> **T-11 — CI Terraform apply.** A GitHub-hosted runner, in a trusted post-merge/dispatch context, holds the org's most-privileged credential set to mutate production infrastructure. **Controls:** never on `pull_request` / no `pull_request_target`; WIF (no static GCP key) + ephemeral GitHub App token (no static `admin:org` PAT); provider secrets in age-encrypted SOPS, decrypted in-job; SHA-pinned actions + `persist-credentials: false`; least-privilege service accounts (`tf-apply-root` = state-bucket-only); reviewer-gated Environment for prod; no plaintext plan/state in logs or artifacts; per-prefix concurrency + native lock. **Residual/accepted:** crown-jewel concentration in one run (mitigated, not eliminated — the OIDC-less providers cannot be keyless); single-maintainer self-approval (no four-eyes); no true infra rollback. **Owner:** thiagojv.

The corresponding **row in the operational security matrix (gitops)** must be added on rollout, and this crossing supersedes ADR-003's "CI has no production access" line for the Terraform-apply path specifically.

## Alternatives considered

- **Keep manual apply, plan-only in CI** — rejected as the end-state: solves ~80% but not the workstation dependency for the resources that matter; retained as break-glass.
- **Atlantis-style credentialed PR plan** — rejected: `github_org_settings.tf` runs `admin:org` shell at *plan* time, so a PR-reachable plan would expose the org's top credential.
- **Static `admin:org` PAT in CI** — rejected: worsens open P0.1.
- **Persistent self-hosted tailnet runner** — rejected: standing compromise surface + SPOF, no net crown-jewel reduction.
- **Plaintext plan-artifact hand-off** — rejected: leaks state/secrets; replaced by the age-sealed plan.

## As-built (2026-07-15) — deltas from the design that operators must know

Two constraints surfaced during rollout forced documented adaptations of D-2/D-5:

1. **No Environment reviewer gate (GitHub plan limitation).** `required_reviewers` on an
   Environment of a **private** repo needs GitHub **Team+** (a 422 "billing plan" on Free).
   BR-015 condition 3 ("reviewer approves the saved plan") is therefore **adapted to option (a)**:
   the `production` Environment carries only a `main`-only branch policy (no reviewer); the human
   control for gated roots is the **manual `workflow_dispatch`**, and the age-sealed saved plan
   still guarantees `applied == approved`. Backstops: the no-destroy guard, the clean-baseline
   gate, and the disable switch. Consistent with **R-8** (single-maintainer self-approval was
   already weak). Alternatives left open: upgrade to Team (b), or a two-dispatch plan→apply flow (c).

2. **`SOPS_AGE_KEY` is a repository secret, not an Environment secret.** GitHub does **not**
   propagate Environment secrets into a **reusable** workflow's `secrets` context, so the age key
   is passed explicitly from the caller and therefore lives as a repo secret — which is reachable
   by `pull_request` workflows in this repo (the apply workflow itself never runs on PRs).
   Accepted for now under the single-maintainer posture (R-8). **Hardening follow-up:** inline the
   apply jobs (drop the reusable) so the key can return to an Environment secret. Tracked.

**Operational facts:** the GitHub App is installed on the **org** with **All repositories** access
(the mono-root governs every org repo — a `terraform`-only install 404s the rest) and the org/repo
permission set verified at build. The mono-root **plan-only-to-green** ran clean (`No changes`),
proving BR-015 condition 2 and that the Fase-1 firebase SA roles + the App token suffice. The
canary caught and fixed three real bugs before any prod mutation (caller `id-token`; env-secret→
reusable + fail-closed decrypt; base64 App-PEM mangled on a second `$GITHUB_ENV` hop).

## Consequences

- Routine applies no longer require a workstation; every apply traces to a workflow run + commit + authorized actor; applied == reviewed on the squash SHA.
- The mono-root remains **human-in-the-loop** (reviewer gate + hard mechanical guards), not lights-out — appropriate for its blast radius.
- A new standing dependency on WIF + a GitHub App + a SOPS bundle; the age key becomes the single most valuable secret to protect.
- P0.1 is advanced (static org PAT retired for the apply path); P0.4 (Firebase SA key) is a parallel dependency, not resolved here.
- The operational security-matrix row (gitops) and this ADR must be kept in sync as the pipeline graduates from firebase-dev → firebase-prod → mono-root.
