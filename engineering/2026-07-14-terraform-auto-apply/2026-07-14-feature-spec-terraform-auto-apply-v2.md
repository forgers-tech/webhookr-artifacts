---
title: Automatically Apply Terraform Changes from GitHub Actions — Feature Specification
status: Draft
owner: thiagojv@gmail.com
council_record: engineering/2026-07-14-terraform-auto-apply-council/2026-07-14-council-record-v2.md
created: 2026-07-14
updated: 2026-07-14
estimate: GG
priority: High
labels: [terraform, github-actions, ci-cd, cd, apply, wif, oidc, github-app, sops, gcs-state, concurrency, security, sre, devops, firebase]
---

# Feature Specification

This document describes everything required to implement a feature, from business motivation to production rollout.

> **Traceability.** This spec authors the decisions in its Council Record (`engineering/2026-07-14-terraform-auto-apply-council/2026-07-14-council-record-v2.md`); it does not make new ones. Every rule, contract and architectural section cites its governing `⟶ D-#` (or `US-#`). Any content that cannot cite one is out of bounds — reopen Council instead of inventing it.

## Execution Trace

| Started | Ended | Duration | Tokens | Revision |
|---------|-------|----------|--------|:--------:|
| 2026-07-14T19:30Z | 2026-07-14T19:52Z | ~22 min | n/a (runtime usage not exposed) | 2 |

> **Revision 2 (2026-07-14) — Gatekeeper authoring refinements, no decision change.** Folds the Gatekeeper v1 review's Important Risks into the spec; every change stays within an existing `D-#` (no reopen). Deltas: **IR-1** — clarify the gated (two-job age-sealed) vs ungated (single-job) apply flows and require the sealed decrypt/gate path to be exercised on firebase-prod before the mono-root; **IR-2** — pin the binary-`tfplan` sealing to `age -r`/`age -d` (reserve `sops` for the structured secret bundle); **IR-3** — the out-of-band bootstrap is an explicit predecessor task gating Steps 5–7, and the App key moves from `.env` to SOPS+age; **IR-4** — early `python3`/`curl` preflight for A-1.

## 1. Context

### Background
Webhookr infrastructure is Terraform-managed across two roots: a prod-only mono-root `terraform/` (GitHub org governance/rulesets, Neon, Hetzner, Cloudflare DNS/SSL + Pages, Grafana, Tailscale, Redis Cloud, TLS, R2) and `terraform/firebase/` (a separate root with dev+prod environments via per-env backend prefix). Remote state is a versioned GCS bucket `webhookr-tfstate` with native locking. Today there is **no CD**: applies are manual `make apply` from an operator workstation whose gitignored `.env` supplies every provider secret. `⟶ US-1..US-10`, Council Evidence F1–F10.

### Problem Statement
Applying Terraform requires an operator workstation with all provider secrets local. This ties infra delivery to one machine, risks the applied change differing from the reviewed one, weakens auditability, and lets repo config and live infra drift. `⟶` Verified Story (Problem).

### Objective
Approved Terraform changes are delivered **from GitHub Actions** — planned and applied in a trusted, auditable, human-gated pipeline — without weakening the current safety properties (state locking, no-blind-apply, least privilege, no secret leakage) or the recorded security posture. `⟶` Verified Story (Objective), D-1..D-14.

### Business Value
Removes the workstation dependency for routine applies; makes applied == reviewed; reduces drift and local-credential risk; standardizes infra delivery on the GitHub workflow; shortens approval→deploy; produces an auditable apply trail. `⟶` Verified Story (Business context).

### Success Metrics
- Routine applies executed from GitHub Actions, not a workstation (count of workstation applies → ~0 for covered roots).
- 100% of applies traceable to a workflow run + commit + authorized actor (GH deployment record). `⟶ D-9`
- Applied revision == approved revision (content-equality on the squash SHA), enforced by the saved-plan mechanism. `⟶ D-4`
- Zero prod credentials reachable from PR-triggered workflows (verified by the denial test). `⟶ US-5, D-3`
- Zero sensitive values in logs/summaries/artifacts (verified by the log-grep gate). `⟶ US-7, D-8`

## 2. Scope

### In Scope
- A new GitHub Actions **apply** workflow (sibling to the unchanged CI), covering the mono-root and both firebase environments. `⟶ D-1, D-14`
- Trigger model + GitHub Environments + reviewer gate. `⟶ D-2`
- Plan visibility model (post-merge/dispatch; no credentialed PR plan for admin:org roots). `⟶ D-3`
- Saved-plan (age-sealed) apply guaranteeing applied == approved. `⟶ D-4`
- Credential model: WIF (GCS), ephemeral GitHub App token (admin:org), SOPS+age (third-party). `⟶ D-5`
- Concurrency/serialization + in-CI force-unlock; out-of-band pipeline-cred bootstrap. `⟶ D-6, D-7`
- Sensitive-value hygiene; outcome reporting. `⟶ D-8, D-9`
- Disable switch, honest rollback runbook, preserved `make apply` break-glass, superseding-ADR + Security-Matrix-row requirement. `⟶ D-10`
- Firebase per-env config committed + fail-closed preflight. `⟶ D-11`
- Drift-sweep defense (clean-baseline gate + no-destroy guard; advisory diff-scoping). `⟶ D-12`
- Staged rollout ladder incl. the mono-root plan-only-to-green risk canary. `⟶ D-13`

### Out of Scope
Redesigning the TF architecture / splitting the mono-root (`X-6`); changing providers or the GCS backend; replacing Terraform; auto-approving arbitrary changes; redesigning app-deploy workflows; auto-destroy in CI (`X-9`); removing emergency manual procedures. `⟶` Verified Story (Out of Scope).

### Assumptions
- **A-1** GitHub-hosted `ubuntu-latest` runners have python3/curl/bash for the `github_org_settings.tf` local-exec. *Validate in Stage B.* `⟶ A-1`
- **A-2** The firebase tfvars are non-secret. *Confirmed and committed (terraform#105).* `⟶ A-2`
- **A-3** `integrations/github ~> 6.0` supports GitHub App auth and the org operations map to App permissions. *Validated; exact permission set to be confirmed at build.* `⟶ A-3`
- **A-5** Third-party providers still lack GitHub OIDC federation. *Recheck at build.* `⟶ A-5`

### Risks
Carried from the Ledger: **R-1** crown-jewel concentration; **R-2** drift-sweep; **R-3** stale-lock strands prod state; **R-4** no true rollback for DNS/DB/server; **R-5** self-referential pipeline creds; **R-6** plan-time RCE via existing constructs; **R-7** applied==approved downgrade if the plan hand-off is mishandled (mitigated by D-4 age-seal); **R-8** single-maintainer self-approval (accepted). See §6/§8 for mitigations. `⟶ R-1..R-8`

### Open Questions
- **TODO (A-1):** confirm the GitHub-hosted runner image provides python3+curl for the plan-time local-exec, or add an explicit install/preflight step.
- **TODO (A-3 caveat):** confirm the exact GitHub App permission set required by the mono-root (Org administration, Org secrets, Repo administration/rulesets, Actions secrets/variables, Environments, Contents) — start minimal, widen from apply errors.
- **TODO (A-5):** recheck whether any of Cloudflare/Hetzner/Neon/Redis Cloud/Tailscale/Grafana added GitHub OIDC since 2026-07; if so, move that provider off a long-lived secret.
- *(All Council `Q-1..Q-4` were resolved in Council Record Rev 2 — no open decision questions remain.)*

## 3. Functional Design

> The "flows" here are CI/CD control-flows (plan → gate → apply), not end-user flows.

### Main Flow
1. Developer opens a PR touching `**.tf`/`**.hcl` → **PR CI** runs fmt + validate + tflint, credential-free (`contents: read`), unchanged. `⟶ D-1, US-1`
2. Reviewer reviews the **code diff** on the PR. For admin:org roots no plan and no prod creds are exposed on the PR. `⟶ D-3, US-2, US-5`
3. PR is **squash-merged** to `main` (repo is squash-only) → a new merge SHA. `⟶ D-2, NF-B`
4. **firebase-dev (ungated / single-job):** on-merge auto, fast-path (no reviewer gate). Because there is no approval boundary, the reusable workflow runs `plan -out=tfplan` **and** `apply tfplan` in **one job on one runner** — no cross-job hand-off, so no plan sealing is needed and `applied == approved` holds trivially (the plan just produced is the plan applied). `⟶ D-2 (Q-4), D-4`
5. **mono-root & firebase-prod (gated / two-job age-sealed):** the reusable workflow runs in the trusted context (Phase-1 mono-root via `workflow_dispatch`; end-state on-merge). Because the Environment reviewer gate is a **job boundary** (it pauses a job at its start, not mid-job — Council Rev-2 Q-1), the plan must cross jobs: a **plan job** runs `terraform plan -out=tfplan` on the target SHA, **encrypts the binary `tfplan` with `age -r <recipient>`** (the age key from Q-2), uploads the *encrypted* artifact, and posts a **sanitized** plan summary. The **apply job** targets a GitHub **Environment** with a **required reviewer**; on approval it `age -d` decrypts the sealed plan on the trusted runner and runs `apply tfplan`. `⟶ D-2, D-3, D-4, D-5`
6. Outcome is reported by job conclusion (applied/failed/cancelled) and recorded as a GH deployment. `⟶ D-9, US-6`

### Alternative Flows
#### AF-01 — Manual dispatch (retry / Phase-1 mono-root)
`workflow_dispatch` triggers the same plan→gate→apply for a chosen target; used as the Phase-1 mono-root trigger and as the workstation-free retry path. `⟶ D-2, D-7, US-8`
#### AF-02 — firebase-dev auto-apply, ungated
On merge touching `firebase/**`, firebase-dev plans and applies without a reviewer gate (lowest blast radius). `⟶ D-2 (Q-4)`
#### AF-03 — Redacted PR plan for firebase roots
A `dflook`-style redacted plan comment MAY run on PRs for firebase roots only (they invoke no admin:org at plan time). Never for the mono-root. `⟶ D-3`
#### AF-04 — Disable switch engaged
With `TF_APPLY_ENABLED=false`, every apply job no-ops (guard checked in-job, not only on trigger); `make apply` from a workstation continues to work unchanged. `⟶ D-10, US-9`

### Error Flows
#### EF-01 — Missing/expired credential or var-file
A fail-closed **preflight** verifies all required provider creds and the firebase `-var-file` resolve *before any state touch*; on failure the job errors without acquiring a lock or mutating state. `⟶ D-11, US-5 (fail-safe)`
#### EF-02 — Cancelled run leaves a stale GCS lock
A guarded `workflow_dispatch` **force-unlock** workflow (LOCK_ID input, prints the lock holder before unlocking) clears it without a workstation. `⟶ D-7, US-8`
#### EF-03 — Failed/partial apply
Rerun the apply (Terraform is convergent) as the first recovery; if a stale lock is reported, force-unlock (EF-02) then rerun. `⟶ D-7, US-8`
#### EF-04 — Plan contains destroy/replace without acknowledgement
The **no-destroy/replace-without-ack** guard (exact, from plan-actions JSON) fails the apply unless the PR carries an explicit ack. `⟶ D-12`
#### EF-05 — Plan not empty beyond the reviewed change (drift)
The **clean-baseline** gate fails when a full plan of `main` is non-empty; auto-apply is refused until drift is reconciled. `⟶ D-12, R-2`
#### EF-06 — Stale saved plan (state moved between plan and apply)
Terraform refuses to apply a stale saved plan; the run fails safely and is retried (re-plan → re-gate). `⟶ D-4`

### Business Rules
#### BR-001 ⟶ D-1
The existing PR CI (fmt/validate/tflint, `permissions: contents: read`, `terraform init -backend=false`) remains credential-free and unchanged; apply is a **separate** workflow.
#### BR-002 ⟶ D-2
Apply runs **only** in a trusted context: post-merge `push:main` (end-state) or `workflow_dispatch`. Never on `pull_request`/`pull_request_target`. The end-state gate is a GitHub Environment **required reviewer**; mono-root is `workflow_dispatch` in Phase-1; firebase-dev is on-merge auto.
#### BR-003 ⟶ D-3
No credentialed plan runs on PRs for admin:org roots (the mono-root's `data.external` executes admin:org shell at plan time). Plan for those roots runs only post-merge/dispatch. Redacted PR plan is allowed only for firebase roots.
#### BR-004 ⟶ D-4
Applied == approved is **content-equality on the post-squash SHA**. For **gated** targets (mono-root, firebase-prod) it is enforced by `plan -out=tfplan` → **`age -r` seal of the binary plan** → approval-gated `age -d` + `apply tfplan` (two jobs, because the gate is a job boundary). For the **ungated** firebase-dev target it is enforced by a **single-job** `plan -out=tfplan` immediately followed by `apply tfplan` (no seal, no cross-job artifact). In both cases there is **no re-plan after the plan is fixed**, and Terraform aborts on a stale plan. A plaintext `tfplan` is never persisted as an artifact; the sealed plan is decrypted only on the trusted runner. (`sops` is used only for the structured third-party secret bundle, not for the binary plan.)
#### BR-005 ⟶ D-5
Credentials: **WIF/OIDC** for the GCS backend (no SA key); an **ephemeral GitHub App installation token** for admin:org (no static PAT); third-party OIDC-less provider secrets live in **SOPS+age** and are decrypted in-job with the age key held as the single Environment secret. Actions are SHA-pinned; runners are GitHub-hosted and ephemeral. `id-token: write` only on the apply job.
#### BR-006 ⟶ D-6
At most one apply per state prefix: a GitHub `concurrency` group per prefix (`terraform/state`, `firebase/dev`, `firebase/prod`) with `cancel-in-progress: false`, layered on GCS native locking. Never `-lock=false`.
#### BR-007 ⟶ D-7
The pipeline's own credentials (WIF identity, GitHub App key) are bootstrapped **out-of-band** — created in a separate root/state that the apply pipeline never applies — to break the `github_secrets.tf` self-reference and avoid self-lockout.
#### BR-008 ⟶ D-8
No plaintext plan or state in artifacts/logs. `TF_LOG`, `set -x`, and `terraform output -raw` of secrets are forbidden; provider tokens are masked; state stays in GCS only.
#### BR-009 ⟶ D-9
The terminal outcome is derived from the apply job's conclusion — `applied` / `failed` / `cancelled` are distinct; a cancelled run is never reported as applied and warns "lock may be held". A GH Environment deployment record is the durable audit ledger.
#### BR-010 ⟶ D-10
A disable switch (`TF_APPLY_ENABLED` variable, checked in-job) can pause auto-apply without editing workflow code; `make apply` remains the break-glass path unchanged; rollback is documented honestly as forward-fix.
#### BR-011 ⟶ D-11
The firebase per-env tfvars are tracked in git (non-secret; committed in terraform#105). A fail-closed preflight asserts the `-var-file` and all required vars resolve before any state touch.
#### BR-012 ⟶ D-12
Drift-sweep defense: the **clean-baseline** invariant (a full plan of `main` is empty) plus a **no-destroy/replace-without-ack** guard are HARD controls; the diff-scoping guard is ADVISORY only.
#### BR-013 ⟶ D-13
Rollout follows the ladder: firebase-dev (scaffolding canary) → firebase-prod (prod-shaped) → **mono-root plan-only run to green** (true risk canary; machine-checks drift=0 + cred resolution) → mono-root Environment-gated saved-plan apply. "canary proven on firebase-dev" does **not** by itself authorize the mono-root apply.
#### BR-014 ⟶ D-14
The workflow is a reusable-workflow/matrix per `(root × env)` with path filters; a `firebase/**` change never triggers the mono-root and vice-versa; firebase re-`init -reconfigure` per env.
#### BR-015 ⟶ D-2, D-13 (Blocking Gate)
The mono-root's first *apply* is authorized only when ALL hold: (1) the P0.1 admin:org lateral path is closed (ephemeral App token + out-of-band pipeline creds); (2) a zero-drift baseline is proven by the mono-root plan-only-to-green stage; (3) the Environment reviewer approves the saved plan of the exact post-squash commit. `plan`-only and firebase-dev may proceed before this. *(mono-root drift baseline was reconciled to `No changes` on 2026-07-14.)*

### Validation Rules
- Preflight: all required provider creds + firebase `-var-file` present/non-empty before state access (fail-closed). `⟶ D-11`
- Plan-actions check: fail on `delete`/`replace` without ack. `⟶ D-12`
- Clean-baseline check: fail auto-apply if the `main` plan is non-empty. `⟶ D-12`
- Supply-chain lint: actions pinned to full commit SHA; `persist-credentials: false` on checkout. `⟶ D-5`
- Plan-time-RCE lint (ADVISORY) + HARD required-review gate on changes to `.terraform.lock.hcl` / `required_providers`. `⟶ D-3`

### Permissions
- **PR jobs:** `permissions: contents: read`; no `id-token`; no Environment secrets. Denial of prod creds is by platform scoping, provable. `⟶ D-1, US-5`
- **Apply jobs:** `permissions: { contents: read, id-token: write }`; bound to a GitHub Environment (`production` for mono-root + firebase-prod; `development` for firebase-dev). Third-party secrets decrypted from SOPS+age in the apply step only. `⟶ D-5`
- **Approval:** single-maintainer self-approval at the Environment gate is the accepted control (no four-eyes). `⟶ R-8 (Q-3 accepted)`

## 4. Non-Functional Requirements

### Performance
Not latency-critical. Note: a full `terraform plan` refreshes many providers (GitHub org across ~13 repos, Cloudflare, Hetzner, Neon, Redis, Grafana) and is minutes-scale; back-to-back plans may hit GitHub API rate limits. Author plans to run once per pipeline, not repeatedly. `⟶ Evidence F5`
### Scalability
N/A — single org, low apply frequency. Concurrency is about *serialization*, not throughput. `⟶ D-6`
### Availability
The pipeline must be disable-able and must never leave state locked without an in-CI recovery path; `make apply` is the always-available fallback. `⟶ D-7, D-10, US-8, US-9`
### Security
Least privilege: WIF keyless GCS; ephemeral App token (retires static PAT, advances audit P0.1); SOPS+age for third-party secrets; SHA-pinned actions; no plaintext plan/state; ephemeral runners; no `pull_request_target`; no persistent self-hosted runner (`X-2`). A superseding ADR + a Security-Matrix row are required on implementation (via `artifact-reconciliation`). `⟶ D-5, D-8, D-10, R-1, R-6`
### Observability
#### Logging
Apply logs masked; no secret material; no `TF_LOG`/`set -x`. `⟶ D-8`
#### Metrics
Outcome per run (applied/failed/cancelled); optional count of workstation vs CI applies (success metric). `⟶ D-9`
#### Tracing
GH Environment deployment record links run → commit → actor as the durable trace. `⟶ D-9`

## 5. Technical Design

### Impacted Repositories
- **`terraform`** — new apply workflow(s) + reusable workflow, force-unlock workflow, a new out-of-band `bootstrap/` root (WIF + SA/IAM), preflight scripts, guard scripts, docs (`README.md`, `CLAUDE.md`), the committed firebase tfvars (done). `⟶ D-1..D-14`
- **`webhookr-artifacts`** — superseding ADR + Security-Matrix row (authored later by `artifact-reconciliation`, not in this spec). `⟶ D-10`
- **GitHub org / GCP (webhookr-prd)** — GitHub App (manual create) + WIF pool/provider/SAs (bootstrap root); GitHub Environments `production` / `development`. `⟶ D-2, D-5, D-7`

### Architecture
Two workflows in `terraform/.github/workflows/`: the **unchanged** `terraform-ci.yml` (PR gate), and a new **`terraform-apply.yml`** that dispatches a **reusable `_tf-apply.yml`** once per `(root × env)` target via path filters. Each **gated** target (mono-root, firebase-prod): `google-github-actions/auth` (WIF) → `setup-terraform@v4` (pin 1.9.0) → `init` (mono-root default prefix; firebase `-backend-config=environments/<env>.backend.hcl -reconfigure`) → preflight (incl. `python3`/`curl` presence, A-1) → `plan -out=tfplan` → **`age -r` seal of the binary tfplan** → upload encrypted artifact + sanitized summary → (Environment gate = job boundary) → `age -d` decrypt → `apply tfplan`. The **ungated** firebase-dev target collapses plan+apply into one job (no seal, no artifact). A separate `terraform-force-unlock.yml` (guarded `workflow_dispatch`). A separate **`terraform/bootstrap/`** root (own state prefix) holds the WIF pool/provider/SAs and is applied only by hand. Credentials: WIF for GCS; `actions/create-github-app-token` → installation token used both as the provider `app_auth` and as `GITHUB_TOKEN` for the `github_org_settings.tf` curl; third-party secrets via `sops -d` using the age key (single Environment secret). `⟶ D-1..D-8, D-14`

### ADR ⟶ D-2, D-3, D-4, D-5, D-13
#### Decision
Deliver Terraform via a **phased, human-gated GitHub Actions pipeline**: end-state = on-merge apply behind a GitHub Environment required-reviewer gate; plan and apply run only in trusted post-merge/dispatch contexts; applied == approved is enforced by an age-sealed saved-plan hand-off on the squash SHA; credentials are WIF (GCS) + ephemeral GitHub App token (admin:org) + SOPS+age (third-party) on ephemeral GitHub-hosted runners; rollout climbs a ladder ending in a mono-root plan-only-to-green risk canary before the first mono-root apply. This **supersedes** the recorded `terraform/CLAUDE.md` "There is no CD pipeline that applies Terraform" decision.
#### Alternatives Considered
Atlantis-style credentialed PR plan (`X-1`, rejected — admin:org at plan time); static admin:org PAT in CI (`X-3`, rejected — worsens P0.1); persistent self-hosted tailnet runner (`X-2`, rejected — SPOF + standing surface); plaintext plan-artifact hand-off (`X-4`, rejected — leaks secrets); hard diff-scoping guard (`X-7`, downgraded to advisory); a single job holding the approval mid-job (impossible — the gate is a job boundary, per Rev-2 Q-1).
#### Tradeoffs
Human-in-the-loop at the gate (not lights-out) for the mono-root; a bounded Phase-1 window where the mono-root is manually dispatched; third-party crown-jewels remain long-lived (in SOPS+age) because those providers lack OIDC; the age key becomes the single most valuable secret to protect.
#### Consequences
The mono-root apply rests on human review + hard mechanical controls (saved-plan apply, no-destroy guard, clean-baseline gate, provider pinning), not on the advisory guards. A superseding ADR + Security-Matrix row must be published. `make apply` stays as break-glass.

### Data Model
N/A — no database. The only "state" is Terraform state in GCS, unchanged (same bucket, prefixes, native locking). `⟶ D-6`

### API Contracts
N/A — no application API. The interfaces are GitHub Actions workflow triggers/inputs and the reusable-workflow contract.

#### Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| N/A | N/A | No HTTP API; triggers are `push:main`, `pull_request`, `workflow_dispatch`. |

#### Request
```json
{ "N/A": "workflow_dispatch inputs only — e.g. { target: root|firebase-dev|firebase-prod, dry_run: bool }" }
```
#### Response
```json
{ "N/A": "no API response; outcome is the job conclusion + GH deployment record" }
```

### Event Contracts
#### Published Events
N/A (no domain events). GH deployment/status events are emitted by the platform. `⟶ D-9`
#### Consumed Events
N/A.

### Message Contracts
```json
{ "N/A": "no message bus involved" }
```

### External Dependencies
GitHub Actions + Environments; GCP Workload Identity Federation (`google-github-actions/auth`); a GitHub App; `actions/create-github-app-token`; SOPS + age; `hashicorp/setup-terraform@v4` (TF 1.9.0); the existing providers (unchanged). `⟶ D-5`

### Feature Flags
`TF_APPLY_ENABLED` — repo/Environment **variable** (not a product flag) gating every apply job in-body; the disable switch. `⟶ D-10`

### Migrations
No schema migration. Infra/config "migrations": (a) commit firebase tfvars — **done** (terraform#105); (b) create the out-of-band `bootstrap/` root + GitHub App + WIF — operator prep; (c) reconcile mono-root drift to zero — **done** (2026-07-14, `No changes`). `⟶ D-7, D-11, D-13`

### Rollback Strategy
No true infra rollback. Rollback = **forward-fix**: revert the PR → reviewed re-apply; not guaranteed to restore destroyed resources (DNS/Neon/Hetzner). GCS state-version restore recovers a corrupted *state file* only (a TF-record recovery, not an infra un-delete). Prevention (plan review + no-destroy guard + clean-baseline) is the real control. Disable switch + `make apply` are the operational fallbacks. `⟶ D-10, R-4`

## 6. Implementation Strategy

### Recommended Implementation Order
1. **Bootstrap (out-of-band, prereq).** Create `terraform/bootstrap/` (WIF pool/provider + `tf-apply-root` [bucket-only] + `tf-apply-firebase` [bucket + firebase/project admin] SAs + IAM), apply by hand; create the GitHub App (manual) + install; store `app_id`/`installation_id`/private key + the age key in SOPS+age / Environment secrets. `⟶ D-5, D-7, BR-015(1)`
2. **Reusable `_tf-apply.yml` + `terraform-apply.yml` + firebase-dev on-merge (Stage A canary).** WIF, SOPS decrypt, preflight, concurrency, age-sealed plan, outcome reporting, disable switch; plus `terraform-force-unlock.yml`. `⟶ D-1, D-5, D-6, D-7, D-8, D-9, D-11, D-13`
3. **Validate Stage A** (scaffolding): auth, concurrency, saved-plan, force-unlock, reporting, disable, tfvars preflight. `⟶ D-13`
4. **firebase-prod** gated apply (`production` Environment). `⟶ D-2, D-13`
5. **Mono-root plan-only-to-green (Stage B risk canary):** full cred set, `plan` only, run until `No changes`; proves App token + 6-provider secret resolution + drift=0. `⟶ D-13, BR-015(2)`
6. **Mono-root Phase-1 apply** via `workflow_dispatch`, Environment-gated saved-plan apply, with no-destroy + clean-baseline guards. `⟶ D-2, D-4, D-12, BR-015(3)`
7. **Graduate mono-root to on-merge** (`push:main`) once drift=0 + re-plan-on-SHA + App-token are all green. `⟶ D-2, X-5`
8. **Docs + reconciliation:** runbook (force-unlock, rollback, disable, retry); update `terraform/CLAUDE.md` + `README.md`; superseding ADR + Security-Matrix row via `artifact-reconciliation`. `⟶ D-10`

### Dependencies
Step 2+ depends on Step 1 (bootstrap creds). Steps 5–7 depend on the closed P0.1 lateral path (App token) and zero drift (BR-015). firebase-dev (Step 2) has no admin:org dependency and can proceed independently of P0.1.

### Breaking Changes
None to running infra. Behavioral change: for covered roots, applies move from workstation to CI; `make apply` remains available. `⟶ D-10`

### Backward Compatibility
`make apply` and the GCS state/locking model are unchanged; the automation must not alter the backend or make state-write creds CI-exclusive, so the manual path keeps working. `⟶ D-10, US-9`

### Rollout Strategy
The D-13 ladder (Steps 3→7), each stage gated on the prior stage's evidence; the mono-root apply is blocked by BR-015 until its three conditions hold. `⟶ D-13, BR-015`

## 7. Task Split

### Infrastructure
**Repositories:** `terraform` (+ GCP `webhookr-prd`, GitHub org)
**Tasks:**
- [ ] Create `terraform/bootstrap/` root: WIF pool/provider + attribute condition (`repository == forgers-tech/terraform && ref == refs/heads/main`), two SAs, IAM bindings; own backend prefix; apply out-of-band. `⟶ D-5, D-7`
- [ ] Create + install the GitHub App; **move `app_id`/`installation_id`/private key from the operator `.env` (current bootstrap copy) into SOPS+age** + the age key as the single Environment secret — out-of-band, never TF-managed (X-3). **This bootstrap task is a hard predecessor of Steps 5–7 (IR-3).** **TODO(A-3):** finalize App permission set. `⟶ D-5, D-7`
- [ ] Create GitHub Environments `production` (reviewer = maintainer) and `development` (no reviewer). `⟶ D-2 (Q-3, Q-4)`

### Backend
**Repositories:** N/A — no application backend code changes. `⟶` (no D-# touches app code)
**Tasks:**
- [ ] N/A

### API
**Repositories:** N/A
**Tasks:**
- [ ] N/A

### Frontend
**Repositories:** N/A
**Tasks:**
- [ ] N/A

### CI/CD (Automation)
**Repositories:** `terraform`
**Tasks:**
- [ ] `_tf-apply.yml` reusable workflow: WIF auth, App-token mint, SOPS-decrypt of the secret bundle, preflight (creds + var-file + **`python3`/`curl` presence, A-1**), `plan -out=tfplan`; **gated targets** → `age -r` seal → encrypted artifact → sanitized summary → Environment gate → `age -d` → `apply tfplan`; **ungated firebase-dev** → single-job `apply tfplan`; outcome reporting; `concurrency` per prefix, `cancel-in-progress:false`; `TF_APPLY_ENABLED` guard. `⟶ D-4, D-5, D-6, D-8, D-9, D-10, D-11`
- [ ] `terraform-apply.yml`: triggers (`push:main` paths + `workflow_dispatch` target input), path-filter matrix over `(root × env)`, firebase-dev on-merge, mono-root/firebase-prod dispatch-gated (Phase-1). `⟶ D-2, D-14`
- [ ] `terraform-force-unlock.yml`: guarded `workflow_dispatch`, LOCK_ID input, prints holder. `⟶ D-7`
- [ ] Guards: no-destroy/replace-without-ack (plan-actions JSON), clean-baseline empty-plan gate, SHA-pin + `persist-credentials:false`, plan-time-RCE lint + lockfile-review gate. `⟶ D-3, D-12`
- [ ] Keep `terraform-ci.yml` unchanged; confirm it stays credential-free. `⟶ D-1`

### Documentation
**Repositories:** `terraform`, `webhookr-artifacts`
**Tasks:**
- [ ] Operations runbook: force-unlock, rollback (forward-fix), disable switch, retry, stale-lock. `⟶ D-10`
- [ ] Update `terraform/CLAUDE.md` (supersede the "no CD" statement) + `README.md`. `⟶ D-10`
- [ ] (post-impl) Superseding ADR + Security-Matrix row via `artifact-reconciliation`. `⟶ D-10`

## 8. Testing Strategy

Every acceptance criterion has a concrete observable + negative test. Primary test surface is **firebase-dev** (safe target) then the mono-root **plan-only** stage; no test applies to the mono-root before BR-015.

### Unit Tests
- `actionlint` on all workflow YAML. `⟶ D-1`
- Guard scripts (no-destroy, clean-baseline, preflight) unit-tested against sample `plan -json` fixtures (destroy present/absent; empty/non-empty; missing var). `⟶ D-11, D-12`

### Integration Tests
- **US-1:** push a mis-formatted `.tf` + a type error → Format/Validate go red and block merge. `⟶ D-1`
- **US-5 (decisive):** a PR job attempting `printenv | grep -i token` / a provider call fails to authenticate — denial by platform (no Environment secrets, no `id-token` on PR) not by omission. `⟶ D-3, D-5`
- **US-6/US-8:** start two applies on one prefix → one queues (concurrency) or errors on state lock; cancel an apply mid-run → stale lock reported → force-unlock clears it → rerun succeeds. Run on firebase-dev. `⟶ D-6, D-7`
- **US-7:** post-run log+artifact `grep -F` for the actual secret values → zero matches; assert `::add-mask::`; assert no plaintext `*.tfplan` artifact (only the age-sealed one). `⟶ D-8`
- **US-5 (fail-safe)/EF-01:** bogus backend prefix / empty provider token / missing var-file → job fails at preflight/init, acquires no lock, mutates nothing. `⟶ D-11`

### End-to-End Tests
- **US-3/US-4:** firebase-dev on-merge → plan sealed → (dev ungated) apply the sealed plan bound to the merge SHA; assert logged SHA == merge SHA; mutate state between plan and apply → `apply tfplan` errors "stale plan". `⟶ D-2, D-4`
- **US-3 (gated) + D-4 decrypt path (IR-1, mandatory before mono-root):** firebase-prod dispatch → plan job seals `tfplan` with `age -r` → apply job waits at the `production` gate; approve → `age -d` decrypt → `apply tfplan` succeeds; assert an unapproved run performs zero apply and acquires no lock; assert the decrypted plan's SHA == the planned SHA. This is the **first exercise of the two-job age-sealed hand-off** — it must pass on firebase-prod before the mono-root apply is attempted. `⟶ D-2, D-4`
- **US-9 (disable):** set `TF_APPLY_ENABLED=false` → merge a `*.tf` PR → no apply runs; a manual dispatch also no-ops (in-body guard); `make apply` locally still works. `⟶ D-10`
- **US-2:** the sanitized plan summary is visible on the run for the reviewer; a `sensitive = true` attribute shows `(sensitive value)`, raw secret absent. `⟶ D-3, D-8`
- **Stage B:** mono-root `plan`-only with full creds runs to `No changes` (proves cred resolution + drift=0). `⟶ D-13, BR-015`

### Manual Validation
- Operator dry-run of the rollback runbook on firebase-dev state (restore a prior GCS generation → known-good). `⟶ D-10`
- Confirm the mono-root plan-only stage exercises the `github_org_settings.tf` local-exec on the runner (validates **A-1**). `⟶ A-1`

## 9. Definition of Ready
- [ ] Council Record v2 consumed; every `D-#` mapped to a section (this spec). `⟶ all D-#`
- [ ] Bootstrap prerequisites identified (WIF, GitHub App, Environments) with owner. `⟶ D-5, D-7`
- [ ] firebase tfvars tracked (terraform#105 merged). `⟶ D-11`
- [ ] Mono-root drift baseline = `No changes` (done 2026-07-14) or re-verified. `⟶ D-12, BR-015`
- [ ] TODO(A-1/A-3/A-5) validation owners assigned.

## 10. Definition of Done
- [ ] firebase-dev applies on-merge from CI; scaffolding tests (auth/concurrency/saved-plan/force-unlock/reporting/disable/preflight) green. `⟶ D-13`
- [ ] firebase-prod applies via gated dispatch. `⟶ D-2`
- [ ] Mono-root plan-only stage runs to `No changes` with the full cred set + App token. `⟶ D-13, BR-015(2)`
- [ ] Mono-root apply (Phase-1 dispatch) gated, saved-plan, guards active — only after BR-015 (1)(2)(3) all green. `⟶ BR-015`
- [ ] No secret in logs/artifacts (grep gate green); no plaintext plan artifact. `⟶ D-8`
- [ ] PR workflows proven credential-denied (US-5 test). `⟶ D-3`
- [ ] Runbook + `CLAUDE.md`/`README.md` updated; superseding ADR + Security-Matrix row filed via reconciliation. `⟶ D-10`
- [ ] `make apply` break-glass verified still working with automation disabled. `⟶ D-10, US-9`

## 11. Estimate

| Size | Selected |
|------|:--------:|
| P    | ☐        |
| M    | ☐        |
| G    | ☐        |
| GG   | ☑        |

Rationale: security-critical, multi-stage rollout, a new bootstrap root + GitHub App + WIF, three apply targets, sealed-plan hand-off, guards, force-unlock, and docs — spanning several gated PRs.

## 12. References
- Council Record v2 — `engineering/2026-07-14-terraform-auto-apply-council/2026-07-14-council-record-v2.md` (D-1..D-14, A/F/Q/R/X registers, BR-015 blocking gate).
- Same-org precedent — `webhookr-toggles/.github/workflows/apply-toggles.yml` (on-merge + Environment gate + concurrency + ephemeral tag:ci).
- Audit P0.1/P0.4 — `engineering/2026-07-01-webhookr-architecture-audit/2026-07-01-audit-report-v1.md`.
- Superseded decision — `terraform/CLAUDE.md` ("There is no CD pipeline that applies Terraform").
- `terraform#105` — commit the non-secret firebase tfvars (D-11).
- [gatekeeper-review v1](2026-07-14-terraform-auto-apply/2026-07-14-gatekeeper-review-v1.md) — 🟡 93/100; IR-1..IR-4 folded into this v2.

## 13. Future Improvements
- Split the mono-root into lower-blast-radius states for true per-provider least privilege (`X-6`, audit-tracked).
- Move any third-party provider to keyless OIDC if it adds GitHub federation (`X-8`, A-5).
- Graduate the mono-root to lights-out on-merge once operational confidence is high (`X-5`).
- Migrate the Hetzner module `datacenter` → `location` (separate maintenance task, spawned 2026-07-14).

## 14. Appendix
- **Blocking Gate (BR-015):** mono-root apply requires (1) P0.1 lateral path closed (App token + out-of-band creds), (2) zero-drift baseline via plan-only-to-green, (3) reviewer-approved saved plan of the exact squash commit.
- **Rev-2 resolutions carried in:** Q-1 (job-boundary gate → two-job age-sealed hand-off), A-3 (GitHub App auth viable), Q-2 (SOPS+age), Q-3 (self-approval accepted), Q-4 (firebase-dev fast-path).
- **Credential env keys** (operator local `.env`, bootstrap copy): `TF_APPLY_GH_APP_ID`, `TF_APPLY_GH_APP_INSTALLATION_ID`, `TF_APPLY_GH_APP_PRIVATE_KEY` (+ unused `_CLIENT_ID`/`_CLIENT_SECRET`); CI copies live in SOPS+age, never TF-managed. `⟶ D-5, D-7`
