---
title: Automatically Apply Terraform Changes from GitHub Actions — Council Record
status: Approved
owner: thiagojv@gmail.com
created: 2026-07-14
updated: 2026-07-14
source: /council
priority: High
labels: [terraform, github-actions, ci-cd, cd, apply, wif, oidc, github-app, gcs-state, concurrency, secrets, security, sre, devops, firebase, webhookr, supersedes-no-cd]
supersedes-decision: terraform/CLAUDE.md#applying-changes-via-make ("There is no CD pipeline that applies Terraform")
relates-finding: 2026-07-01-webhookr-architecture-audit/2026-07-01-audit-report-v1.md#P0.1
---

# Council Record — Automatically Apply Terraform Changes from GitHub Actions

> **Nature of this document.** Negotiated **decision record** produced by `/council` (multi-agent,
> **Critical** depth). It is **not** a Feature Specification — it stops at decisions. Every
> specialist position came from a real independent Task execution (7 in Round 1, 5 in Round 2
> negotiation, 3 challenge responses); the Chair authored none of them. The authoritative output
> is the **Verified Story + Decision Ledger** at the end; downstream stages
> (`/upstream` → `/gatekeeper` → `/downstream` → `artifact-reconciliation`) consume it by stable ID.

## Execution Trace

| Started | Ended | Duration | Tokens | Revision |
|---------|-------|----------|--------|:--------:|
| 2026-07-14T18:30Z | 2026-07-14T19:15Z | ~45 min | ≈898K (specialist executions, R1+R2+challenge; Historian Explore + Chair turns + Rev-2 validations not separately metered → partial) | 2 |

> **Revision 2 (2026-07-14) — Operator resolutions, no reopen of a decision.** The operator answered the open questions and the Chair validated Q-1/A-3 against the repo; no `D-#` was invalidated, so the Round-1..6 body below is unchanged and only the ledger registers move. Deltas:
> - **Q-1 RESOLVED (validated).** A GitHub Environment required-reviewer gate pauses a job **at its start (job boundary), not mid-job** — verified against the org's own `webhookr-toggles/.github/workflows/apply-toggles.yml` (separate `plan` job → `apply` job with `needs: plan` + `environment:`). So a single job **cannot** span plan→approval→apply. The **hard** applied==approved guarantee (D-4) is preserved anyway via a **two-job pattern with an age-sealed plan hand-off**: the plan job seals `tfplan` with the **SOPS age key** (the same key chosen in Q-2), uploads it as an *encrypted* artifact, and the approval-gated apply job decrypts it on the trusted runner and runs `apply tfplan` (Terraform still refuses a stale plan). This satisfies D-4 **and** D-8 (no plaintext plan/secret in an artifact) simultaneously. **A-4 resolved** (the "hold mid-job" assumption is false but no longer needed); **R-7 mitigated**.
> - **A-3 RESOLVED (validated).** `integrations/github ~> 6.0` supports GitHub App auth (`app_auth`), and `github_org_settings.tf`'s raw `curl` can consume an installation token from `actions/create-github-app-token` as `GITHUB_TOKEN`. A GitHub App can hold *Organization administration* + *Organization secrets* → covers today's `admin:org` usage. Ephemeral-App-token path is **viable**; confirm the exact App permission set at build. This keeps F-4's "concurrent" branch (not the blocking fallback).
> - **Q-2 RESOLVED → SOPS.** Third-party statics at rest live in the existing **SOPS+age** bundle, decrypted in-job with the age key held as the one Environment secret; doubles as the plan-sealing key above. (Chooses Architect's "GitHub holds ~zero static secrets" direction over raw Environment secrets.)
> - **Q-3 RESOLVED → accepted.** No second reviewer exists; single-maintainer **self-approval at the Environment gate is the accepted control** (audit/speed-bump, not four-eyes). Formalizes **R-8** as an accepted risk.
> - **Q-4 RESOLVED → firebase-dev fast-path** (no reviewer gate): lowest blast radius, isolated dev state, no admin:org; reviewer gate stays on firebase-prod + mono-root.
> - **NF-C corrected** (see below): the `GITOPS_DEPLOY_PAT` shadow-override **is** a recorded operational incident (project memory `google-auth.md`, 2026-07-01) — *not in the repo* because it was a `gh secret delete`, not code. No council control depends on it; the operator confirmed keeping the memory (it carries a real "don't recreate a repo-level `GITOPS_DEPLOY_PAT`" safety note).

---

## Council Metadata
- **Objective:** Deliver approved Terraform changes via GitHub Actions instead of an operator workstation, without weakening the current safety properties (locking, no-false-apply, least privilege, no-secret-leak) or the org's security posture.
- **Outcome:** APPROVED as a **phased, human-gated delivery**. End-state = on-merge apply behind a GitHub Environment required-reviewer gate. A hard, owned **blocking gate** governs the prod mono-root's first *apply*.
- **Depth:** Critical (production infra, org's most-privileged credentials, destructive potential, security trust boundary, state locking, no true rollback).
- **Status:** Closed — no OPEN blocking conflict; all conflicts RESOLVED or converted to owned gates.
- **Revision:** 1 · **Date:** 2026-07-14

---

## Council Plan

**Depth rationale:** an apply concentrates GitHub `admin:org` + Cloudflare DNS + Hetzner + Neon + Redis Cloud + Tailscale + Grafana-admin credentials in one run, mutates DNS/DB/servers/org-governance, has no transactional rollback, and crosses the CI trust boundary the security matrix deliberately keeps closed → Critical.

**Specialist selection**

| Specialist | Status | Why |
|---|---|---|
| Software Architect | Required | workflow topology, plan→apply contract, multi-root/state, failure modes |
| Security | Required | trust boundary, credential exposure, least privilege, PR/fork abuse, P0.1 |
| SRE/Platform | Required | apply safety, locking, drift, partial-apply recovery, rollback, disable |
| DevOps | Required | concrete GHA design, Environments, WIF, secrets delivery, precedent |
| QA/Test | Required | make each AC observable; test without touching prod; negative paths |
| Historian | Required | precedent + contradiction (the "no-CD" decision, audit P0s, toggles) |
| Devil's Advocate | Required | necessity, simpler alternative, blast radius, rollback reality |
| Product | Not required | internal delivery capability, no end-user product behavior |
| Backend / Database / Web / Mobile | Not required | no app code / schema / UI |
| Observability | Folded into SRE/DevOps | apply-result visibility is an AC, covered there |
| FinOps | Not required | GH Actions minutes negligible; no new cost resource |

---

## Evidence Package (verified facts)

- **F1.** Two Terraform roots: mono-root `terraform/` (prod-only) managing GitHub org governance (rulesets), Neon, Hetzner, Cloudflare DNS/SSL + Pages, Grafana dashboards+alerts, Tailscale, Redis Cloud, TLS, R2; and nested `terraform/firebase/` — a **separate** root with its own GCS backend using a per-**environment** prefix (`-backend-config=environments/{dev,prod}.backend.hcl` + `{dev,prod}.tfvars`).
- **F2.** Remote state = GCS bucket `webhookr-tfstate` (project `webhookr-prd`), prefix `terraform/state`, uniform bucket-level access, public-access-prevention, versioning ON, **native GCS locking**. Backend auth = Google **ADC** locally.
- **F3.** Existing CI `.github/workflows/terraform-ci.yml`: triggers on `pull_request`→main + `push`→main; `permissions: contents: read`; jobs = fmt / validate (`init -backend=false`) / tflint. **No plan, no apply, no state, no secrets.** TF pinned 1.9.0. Also security-scan.yml + secret-scan.yml; checkov/tflint/gitleaks/trivy configs present.
- **F4.** **No CD applies Terraform** (stated in `terraform/CLAUDE.md`). Applies are manual `make apply` (interactive), operator `.env` supplies all provider secrets; `make plan ARGS="-target=..."` documented to avoid sweeping drift; `make destroy` exists.
- **F5.** An apply holds the org's most-privileged credentials **at once** (admin:org, Cloudflare, Hetzner, Neon, Redis Cloud, Tailscale, Grafana admin, GCS).
- **F6.** `github_org_settings.tf` runs a `local-exec` (curl PATCH GitHub org API) **at apply** and a `data.external` (bash+curl+python3) **at plan** — i.e. plan itself executes shell with `admin:org`.
- **F7.** `github_secrets.tf` has Terraform **manage** the org Actions secret `GITOPS_DEPLOY_PAT` (bootstrap chicken-and-egg for any CI running TF).
- **F8.** Governance: mono-root sets `ruleset_enforcement="active"`; `main.tf:108` — *"Single-maintainer org: no human approval required"* (`required_approving_review_count = 0`); `allow_merge_commit=false` (**squash-only**). GitHub Environments exist only for `toggles` (toggles_prd/qa), not for TF apply.
- **F9.** Known standing drift (Grafana dashboard pending creds; toggles TF half-applied) — a blind full apply would fail or sweep it; the `-target` habit exists to prevent this.
- **F10.** Prior decisions in `webhookr-artifacts/engineering` (source of truth): infra-security-matrix ADR (deny-by-default egress, least privilege, CI-has-no-prod-access posture); audit 2026-07-01 P0.1 (org-wide PAT supply-chain) + P0.4 (SA key) — both **open**; Decision-Ledger SDLC; council-persistence.
- **New (Round-2 verified):**
  - **NF-A.** `firebase/environments/*.tfvars` are **git-ignored** (blanket `*.tfvars`); only `*.backend.hcl` are tracked. Contents (`project_id`, `project_name`, `authorized_domains`) are **non-secret** (real secrets arrive via `TF_VAR_*`). → firebase CI apply is blocked until these are committed/generated.
  - **NF-B.** Squash-only merge ⇒ the commit that lands on `main` is a **new SHA no plan ever ran against**; "applied commit == approved revision" can only mean **content/tree equality enforced at apply**, not SHA equality.
  - **NF-C (corrected in Rev 2).** The `GITOPS_DEPLOY_PAT` shadow-override **is** a real recorded operational incident (project memory `google-auth.md`, 2026-07-01: a repo-level secret on `webhookr-web` shadowed the TF-managed org secret → CD exit 128; fixed by deleting the repo override). The Historian couldn't find it **in the repo** because it was an operational `gh secret delete`, not code — expected, not "false". No council control depends on it. The real, repo-visible bootstrap hazards remain `backend.tf` (bucket bootstrapped out-of-band) and the `github_secrets.tf` self-reference.

**Third-party OIDC fact:** only the GCS backend can use GitHub OIDC / Workload Identity Federation. Cloudflare, Hetzner, Neon, Redis Cloud, Tailscale, Grafana have **no** GitHub OIDC → they require long-lived secrets somewhere.

---

## Round-1 Reports (independent — summarized; Chair edited none)

- **Architect.** No CI cloud identity exists (grep for oidc/workload_identity = none). `data.external` executes shell with admin:org **at plan time** → plan-in-PR is structurally unsafe for the mono-root. Blast radius = whole org; least-privilege only achievable per-root (splitting is out of scope). Bootstrap circularity (TF manages the secrets CI would use). Ambient drift makes blind full-apply unsafe; `-target` is load-bearing. Plan→apply handoff fights "no secrets in artifacts". Two roots × envs ⇒ matrix + per-prefix concurrency. Rollback undefined = forward-fix. Recommends apply-on-merge Environment-gated (not Atlantis), WIF, same-runner saved plan, ADR.
- **Security.** Moving crown-jewels from one laptop into a shared, networked CI runner is not automatically safer. PR-time faithful plan needs prod-read creds → collides with "no prod creds on PRs". `pull_request_target` forbidden. `plan` = arbitrary code execution with admin:org (F6). Privilege concentration = one compromised run compromises the platform. Saved `*.tfplan` job→job leaks secrets. Adopt WIF for GCS; GitHub App ephemeral token for admin:org; step-level Environment secrets; SHA-pinned actions; concurrency `cancel-in-progress:false`. Must not worsen open P0.1/P0.4.
- **SRE/Platform.** Drift makes apply-on-merge unsafe; `-target` has no automatic replacement. Re-plan-at-apply diverges from the reviewed plan (TOCTOU). Stale-lock recovery without a workstation is not solved → needs in-CI guarded force-unlock. Two-layer serialization (GH concurrency + GCS lock) both needed, keyed per prefix. Partial-apply recovery = rerun. Report on job conclusion. Rollback = forward-fix, say it plainly. Keep Makefile break-glass. Do NOT auto-apply-on-merge the mono-root until drift cleared + blast radius reduced.
- **DevOps.** Model on the `webhookr-toggles` precedent (push:main + `environment:` gate + concurrency + ephemeral tag:ci). New sibling apply workflow; keep terraform-ci.yml. WIF for GCS + two least-priv SAs; long-lived third-party secrets as Environment secrets; matrix over 3 states with path filters; disable var `TF_APPLY_ENABLED`. **Found the blocker:** firebase tfvars untracked. Recommends `dflook` for redacted PR plan / apply-of-approved-plan or toggles-style re-plan-on-merge.
- **QA/Test.** The "approval condition" doesn't exist on this repo yet (0 required approvals; no TF Environment). Plan needs live prod creds (F6) → PR-plan vs no-PR-creds cannot both hold. Approver==applier. Gave a concrete observable + negative test per AC. Test the pipeline WITHOUT prod via firebase-dev / throwaway resource. Deterministic no-secret-in-logs grep+mask gate. Prove PR-cred-denial by the platform, not by omission.
- **Historian.** **Headline contradiction:** the story reverses an explicit recorded decision — `terraform/CLAUDE.md` "There is no CD pipeline that applies Terraform" + "never apply blind" — so it needs a superseding ADR/council record, not a silent workflow. Audit P0.1 (org-wide PAT) + P0.4 (SA key) open; putting apply creds in CI widens exactly P0.1's blast radius; honor its remediation vector (ephemeral tokens + environment protection). `webhookr-toggles` = reusable pattern (ephemeral tag:ci, Environment gate, concurrency) but its flat `tag:ci` ACL is a caution; its default reviewer gate is empty (not a safe precedent for infra). No WIF/OIDC anywhere in the org today (greenfield). Firebase runbook = env-scoped-state + rollback precedent. The `GITOPS_DEPLOY_PAT` shadow incident **not found** — unverified.
- **Devil's Advocate.** Full auto-apply-to-prod isn't what the pain demands; plan-in-CI + one-click manual apply gets ~80% without moving secrets into GitHub. Moving all provider secrets into GitHub creates a NEW, larger surface and worsens open P0.1. Single-maintainer ⇒ the "approval gate" is largely theatre. `plan` is not side-effect-free and needs admin:org (F6). Auto-apply-on-merge is dangerous with standing drift. Squash-merge ⇒ applied SHA never plan-reviewed. No clean rollback for DNS/DB/server. Alternatives: plan-in-CI + manual apply; firebase-dev only first; ephemeral tokens; fix P0.1 first.

---

## Consensus Matrix (Phase 2)

**Unanimous (no conflict):** keep credential-free PR CI; never `pull_request_target`; WIF for GCS + long-lived secrets for third-party providers; per-prefix concurrency + `cancel-in-progress:false` layered on GCS lock; never publish `*.tfplan` plaintext; reconcile drift before enabling; rollback = forward-fix; disable switch + keep `make apply`; a superseding ADR + Security-Matrix row is required.

**Real conflicts (each quoted from two real outputs):** **F-1** apply-on-merge (DevOps/Architect) vs manual-dispatch-first (SRE/Devil's-Advocate). **F-2** plan-visibility vs "PRs get no prod creds" (Security: no PR plan vs DevOps: dflook option). **F-3** secret placement/runner (Security+DevOps Environment secrets vs Devil's-Advocate SOPS+self-hosted vs Historian ephemeral App token). **F-4** P0.1 sequencing (Devil's-Advocate: hard prerequisite vs Security: must-not-worsen dependency). Plus two blockers-not-conflicts: **NF-A** untracked firebase tfvars, **NF-B** squash-SHA.

---

## Revised Positions (Round 2 — real negotiation)

- **F-1 → RESOLVED.** All five converged: **end-state = on-merge apply behind a GitHub Environment required-reviewer gate**; **Phase-1 mono-root = `workflow_dispatch`** (time-boxed ACCEPTED-RISK) graduating to on-merge; **firebase-dev = on-merge auto pilot** (lowest blast radius). Devil's Advocate redline retained: **no *unattended* (no-reviewer) apply to the mono-root, ever.** SRE contributed a 6-point precondition checklist; mono-root goes **last**.
- **F-2 → RESOLVED.** **No credentialed plan on PRs** for admin:org roots (F6). Plan runs post-merge/dispatch in the trusted context; the sanitized summary is reviewed at the Environment gate; `dflook` redacted PR plan allowed only for firebase (no admin:org at plan). Security added a **plan-time-RCE lint** (advisory) + **hard review-gate** on `.terraform.lock.hcl` / `required_providers`.
- **F-3 → RESOLVED.** **WIF/OIDC for GCS** (no SA key) + **ephemeral GitHub App installation token for admin:org** (retires the static PAT — Historian's P0.1 vector, adopted by Architect/DevOps/Security) + **third-party statics as Environment-scoped secrets**, step-level, SHA-pinned actions, **GitHub-hosted ephemeral runners**. **Persistent self-hosted tailnet runner REJECTED** (SRE: SPOF + toolchain drift; DA withdrew it; a standing tailnet-resident target is a net regression). Ephemeral `tag:ci` runner only if a provider is tailnet-unreachable. (Residual **Q-2**: Architect prefers "GitHub holds ~zero static secrets — fetch third-party from a GCP-side store/SOPS via WIF"; DevOps/Security accept Environment statics as baseline — a non-blocking optimization.)
- **F-4 → RESOLVED (owned gate).** Split gate: the **admin:org-token piece is BLOCKING** — a prod apply must use an ephemeral App token, not a static broad PAT, and the pipeline's own creds must be bootstrapped **out-of-band** (breaks the `github_secrets.tf` self-reference; SRE's reliability prereq). Broader P0.1 remediation (`GITOPS_DEPLOY_PAT` → `visibility=selected` / full GitHub-App migration) + P0.4 (remove Firebase SA key) are **parallel**, not mechanical blockers. **firebase-dev + plan-only proceed now.**
- **NF-A → RESOLVED.** **Commit** the two non-secret firebase tfvars (negate the gitignore for those paths) + gitleaks check; **fail-closed preflight** that the var-file + all required vars resolve before any state touch. (Preferred over generate-in-job because it makes "applied==approved" true — config lives in the reviewed commit.)
- **NF-B → RESOLVED.** applied==approved = **content-equality on the post-squash SHA**, enforced at apply by same-runner `plan -out` → approve → `apply tfplan` (Terraform aborts on stale plan); no plan-artifact reuse.

---

## Challenge–Response (Phase 5)

| # | Challenge (target) | Response | Status |
|---|---|---|---|
| C1 | Does manual-dispatch-first quietly **narrow** the card's "AUTOMATICALLY apply"? (→ DevOps) | The card's own clauses ("controlled apply", "prod protected by GH environment", "after required approval conditions", "must not bypass reviews") mean *automatic = pipeline-executed-under-conditions*, not gate-free. **End-state = on-merge + Environment gate = the truer match** (mirrors the working `apply-toggles.yml` precedent). Phase-1 `workflow_dispatch` is the **same pipeline minus one auto-trigger on one root** for a bounded window — sequencing, not scope. The Environment `required_reviewers` **is** the card's "required approval condition" (the only human approval in a repo with `required_approving_review_count=0`) — additive, not a bypass. | **RESOLVED** (end-state) + **ACCEPTED-RISK** (time-boxed Phase-1, explicit graduation) |
| C2 | Are the **diff-scoping** + **plan-time-RCE** guards mechanically enforceable, or aspirational? (→ Architect) | **Both are ADVISORY, not hard.** No Terraform source-line→resource-address provenance exists → diff-scoping is false-positive-prone; operators would disable it. RCE-regex is trivially evadable (edit an *existing* local-exec, module indirection, an already-pinned provider). **Mono-root safety actually rests on: (hard) same-runner saved-plan apply of the exact `tfplan` + trust boundary (post-merge only) + no-destroy/replace-without-ack guard (exact via plan-actions JSON) + clean-baseline empty-plan gate + provider/lockfile pinning-with-review; (human) a person reading the plan at the Environment gate.** Do not over-claim automated safety — it's human-in-the-loop with mechanical guardrails. **Make-or-break flag:** the saved `tfplan` must live and die **within one job** across the approval, never persisted as an artifact; if the Environment approval forces a job boundary that drops it, the hard applied==approved guarantee downgrades to re-plan-and-hope. | **RESOLVED** (guards reclassified; real controls named) → raises **Q-1** |
| C3 | Does the **firebase-dev canary** validate the mono-root path, or only the scaffolding? (→ SRE) | **Only the scaffolding** (WIF→GCS auth, concurrency, saved-plan, force-unlock, reporting, disable, the tfvars control). It **cannot** prove the mono-root risk surface: admin:org App token, 6-provider secret injection, plan-time local-exec/data.external, drift-sweep, org-wide governance blast radius. **New mandatory control:** insert **Stage B — a mono-root `plan`-only stage with the FULL production cred set, run until it plans clean (`No changes`)** — the true risk canary, which also **machine-checks** the "drift=0" and "creds resolve" preconditions. Ladder: firebase-dev → firebase-prod → **mono-root plan-only to green** → mono-root gated apply. "canary proven on firebase-dev" **must NOT** count as a mono-root precondition (false confidence). | **RESOLVED** (adds D-13 Stage B) |

No challenge produced a new blocking conflict; all resolved with refinements.

---

## Council Decision (Phase 6)

**APPROVED — build the GitHub-Actions Terraform delivery pipeline, phased and human-gated.** Priority order applied: verified requirements → architectural constraints → security & data integrity → operational safety → simplicity → delivery speed. Delivery speed did **not** override the mono-root blocking gate.

**Accepted:** D-1 … D-14 (below). **The end-state fully delivers the card** (on-merge + Environment gate). **Rejected:** PR-time credentialed plan for the mono-root (X-1), persistent self-hosted runner (X-2), static admin:org PAT in CI (X-3), plaintext plan artifacts (X-4), auto-destroy / removing manual procedures (X-9). **Deferred:** mono-root auto-on-merge until preconditions met (X-5), mono-root split (X-6), hard diff-scoping guard (X-7), keyless third-party secrets (X-8).

**BLOCKING GATE (owned, recorded).** *The prod mono-root's first **apply** is authorized only when ALL hold simultaneously:* (1) the P0.1 admin:org lateral path is closed — ephemeral GitHub App token + pipeline creds bootstrapped out-of-band; (2) zero-drift baseline proven by **Stage B** (mono-root plan-only to `No changes`); (3) the Environment reviewer renders and approves the plan of the **exact post-squash commit** via a **same-job saved-plan** apply. **`plan`-only and firebase-dev may proceed now.** A superseding ADR + a Security-Matrix row must be published on implementation (`artifact-reconciliation`, not this stage).

---

## Council Record

### Verified Story

**Problem.** Applying Terraform for Webhookr requires an operator to run `make apply` from a workstation with all provider secrets in a local `.env`. This ties infra delivery to a machine, risks applied≠reviewed, weakens auditability, and lets repo config and live infra drift.

**Actor.** The Webhookr engineering team (single-maintainer org today).

**Business context.** Standardize infra delivery on the GitHub workflow; reduce manual ops, drift, and local-credential risk; shorten approval→deploy; keep an auditable trail — **without** weakening the security posture (least privilege, no-prod-creds-in-CI matrix stance) or the current apply safety (locking, `-target` drift discipline, no-blind-apply).

**Constraints (from card, upheld):** apply only from trusted GHA contexts; PRs get no prod creds; applied == approved revision; prod protected by GH environment + repo controls; least privilege; one apply per state; no state/secrets in artifacts/logs; preserve GCS remote-state + native locking; failed/cancelled never reported applied; must not bypass required reviews/branch protection; retryable without a workstation; documented rollback; disable-able; existing infra keeps running if disabled.

**Acceptance Criteria (testable)**
- **US-1** PR CI runs fmt + validate + tflint, credential-free (`contents: read`), unchanged.
- **US-2** A Terraform plan is generated for eligible changes in a trusted context, and its **sanitized** summary is visible for human review before apply.
- **US-3** Terraform apply executes **from GitHub Actions** (not a workstation), in a trusted post-merge/dispatch context, gated by a GitHub Environment required-reviewer.
- **US-4** The applied change equals the approved revision — **content-equality on the post-squash SHA**, enforced by a same-runner saved-plan apply (Terraform aborts on stale plan).
- **US-5** PR/untrusted workflows **cannot** access production credentials or state — provable by platform denial (no Environment secrets, no `id-token` on PR jobs), not by omission.
- **US-6** Concurrent applies against the same state are serialized (per-prefix GH concurrency `cancel-in-progress:false` + GCS native lock); cancelled/failed are **never** reported as applied.
- **US-7** Sensitive values never appear in logs/summaries/artifacts; state and plan files are never published.
- **US-8** A failed/partial apply is recoverable **without a workstation** (rerun + in-CI guarded force-unlock).
- **US-9** Operating + rollback (forward-fix) procedures are documented; automation is disable-able; the manual `make apply` break-glass keeps working unchanged.
- **US-10** The firebase root can CI-apply (dev auto, prod gated) with its per-env config committed/validated.

**Out of Scope:** redesigning TF architecture / splitting the mono-root; changing providers or the GCS state backend; replacing Terraform; auto-approving arbitrary changes; redesigning app-deploy workflows; auto-destroy in CI; removing emergency manual procedures.

### Decision Ledger

**Decision Log** — `D-# | Decision | Serves | Evidence | Alternatives rejected | Trade-off | Owner`
- **D-1** | New **sibling** apply workflow; keep `terraform-ci.yml` credential-free and unchanged | US-1,US-3 | F3 | rewriting the existing CI | two workflows to maintain | DevOps
- **D-2** | **Trigger:** end-state on-merge (`push:main`) + GitHub Environment (`production`/`development`) required-reviewer gate; **mono-root Phase-1 = `workflow_dispatch`** graduating to on-merge on 3 criteria; **firebase-dev on-merge from day one** | US-3 | C1, F8, toggles precedent | Atlantis apply-before-merge (X-1); unattended auto-apply (DA redline) | a bounded window where mono-root apply is manually triggered | DevOps/SRE
- **D-3** | **No credentialed plan on PRs** for admin:org roots; plan runs only post-merge/dispatch in trusted context; `dflook` redacted PR plan only for firebase; plan-time-RCE lint (advisory) + **hard review-gate on `.terraform.lock.hcl`/`required_providers`** | US-2,US-5 | F6, C2 | PR-time credentialed plan | reviewers see the plan at the gate, not on the PR | Security
- **D-4** | **applied == approved = content-equality on the post-squash SHA.** *(Rev-2, validated Q-1):* the Environment gate pauses at the **job boundary**, so the pattern is a **two-job hand-off** — `plan -out=tfplan` in the plan job → **seal `tfplan` with the SOPS age key** → upload as an *encrypted* artifact → approval-gated apply job **decrypts on the trusted runner** and runs `apply tfplan`; **no re-plan after approval** (Terraform refuses a stale plan). Plaintext plan/secret never leaves the runner (D-8 preserved). | US-4,US-7 | NF-B, C2, Q-1 | re-plan-at-apply; single-job-holds-approval (impossible); plaintext plan artifact (X-4) | needs the age-seal step so the cross-job artifact is encrypted | Architect
- **D-5** | **Credentials:** WIF/OIDC for GCS backend (no SA key); **ephemeral GitHub App installation token** for admin:org (retires static PAT — A-3 validated); third-party OIDC-less secrets held in **SOPS+age** (Rev-2 Q-2), decrypted in-job with the age key as the single Environment secret; SHA-pinned actions, GitHub-hosted **ephemeral** runners | US-3,US-5 | F5, F7, OIDC fact, Historian P0.1, Q-2 | static admin:org PAT (X-3); persistent self-hosted runner (X-2); raw Environment secrets per-provider (superseded by SOPS) | age key becomes the single crown-jewel to protect | Security/DevOps
- **D-6** | **Concurrency** group per state prefix (`terraform/state`, `firebase/dev`, `firebase/prod`), `cancel-in-progress:false`, layered on GCS native lock; never `-lock=false` | US-6 | F1,F2 | single global group; GH-lock only | independent states serialize independently | SRE
- **D-7** | **In-CI guarded force-unlock** (`workflow_dispatch`, LOCK_ID input, prints holder); rerun = first recovery; **pipeline's own creds bootstrapped out-of-band** (not TF-managed) to break the `github_secrets.tf` self-reference | US-8 | F2,F7,NF-C | automatic force-unlock; workstation recovery | manual, deliberate unlock step | SRE
- **D-8** | **Sensitive hygiene:** no plaintext `tfplan` artifact; forbid `TF_LOG`/`set -x`/`output -raw` of secrets; mask; state stays in GCS only | US-7 | F5, gitignore | — | — | Security
- **D-9** | **Outcome reporting** keyed on job conclusion (applied/failed/cancelled); GH Environment deployment record as durable ledger; cancelled warns "lock may be held" | US-6,US-9 | SRE R8 | `always()`-report | — | SRE
- **D-10** | **Honest rollback runbook** (forward-fix = git revert + reviewed re-apply; not guaranteed to restore destroyed resources; GCS state-version restore is a TF-record recovery) + **disable switch** (`TF_APPLY_ENABLED` var + not-triggering) + **Makefile break-glass preserved** + superseding ADR + Security-Matrix row required | US-9 | F4,F10, README | selling git-revert as "rollback" | — | SRE/Historian
- **D-11** | **Commit** the two non-secret firebase tfvars (negate `.gitignore` for those paths) + gitleaks check; **fail-closed preflight** that the var-file + all required vars resolve before any state touch | US-10 | NF-A | out-of-band tfvars on runner; generate-in-job | keeps env config in the reviewed commit | DevOps
- **D-12** | **Drift-sweep defense:** clean-baseline invariant (full plan of `main` empty) as the `-target` replacement + **no-destroy/replace-without-ack** guard (HARD, plan-actions JSON); **diff-scoping guard downgraded to ADVISORY** | US-4,US-9 | F9, C2 | hard diff-scoping (X-7) | baseline must be kept clean | Architect/SRE
- **D-13** | **Staged rollout ladder:** Stage A firebase-dev canary (scaffolding) → firebase-prod (prod-shaped) → **Stage B mono-root `plan`-only with FULL creds, run to green** (true risk canary; machine-checks drift=0 + creds resolve) → mono-root Environment-gated saved-plan apply | US-3 | C3 | firebase-dev as sufficient proof | extra pre-prod stage | SRE
- **D-14** | **Topology:** reusable-workflow / matrix per (root × env) with path filters; firebase re-`init -reconfigure` per env; a `firebase/**` change never triggers the mono-root and vice-versa | US-3,US-5 | F1 | one matrixed job with `matrix.*` concurrency | — | Architect/DevOps

**Assumption Register** — `A-# | Assumption | Source | Validation | Status | Impact if false`
- **A-1** | GitHub-hosted `ubuntu-latest` runners (have python3/curl/bash for F6 local-exec) | SRE | Stage B plan-only run | Open | apply fails mid-run on missing deps
- **A-2** | firebase tfvars (`project_id`/`project_name`/`authorized_domains`) are non-secret | DevOps/Security | gitleaks + manual review pre-commit | Open | a secret gets committed
- **A-3** | `integrations/github` provider supports GitHub App installation-token auth for the admin:org operations | Security/Architect | ✅ validated: `~> 6.0` supports `app_auth`; `curl` uses `create-github-app-token`; App can hold Org-administration + Org-secrets | **RESOLVED (Rev 2)** | (n/a — confirmed viable; verify exact App perms at build)
- **A-4** | A GH Environment approval can hold **mid-job** so one job spans plan→approve→apply keeping the `tfplan` in memory | Architect | ✅ validated **FALSE** against `apply-toggles.yml` (gate is job-boundary) — superseded by the two-job age-sealed hand-off in D-4 | **RESOLVED (Rev 2) — no longer needed** | (n/a — D-4 no longer depends on it)
- **A-5** | Third-party providers still lack GitHub OIDC federation | OIDC fact | recheck at build | Open | that provider's secret can move to keyless

**Conflict Register** — `F-# | Topic | Positions (holders) | Resolution | Status`
- **F-1** | apply trigger/scope | apply-on-merge (DevOps/Architect) vs dispatch-first/not-prod (SRE/Devil's-Advocate) | end-state on-merge + Environment gate; Phase-1 mono-root dispatch (time-boxed); firebase-dev auto pilot; no unattended mono-root apply | **RESOLVED** (D-2)
- **F-2** | plan-visibility vs no-PR-creds | no PR plan (Security) vs dflook PR plan (DevOps) | no credentialed PR plan for admin:org roots; plan post-merge, reviewed at gate | **RESOLVED** (D-3)
- **F-3** | secret placement / runner | Env secrets (Security/DevOps) vs SOPS+self-hosted (Devil's-Advocate) vs ephemeral App token (Historian) | WIF + ephemeral App token + Env-scoped statics; persistent self-hosted rejected | **RESOLVED** (D-5)
- **F-4** | P0.1 sequencing | hard prerequisite (Devil's-Advocate) vs must-not-worsen dependency (Security) | split gate: App token + out-of-band creds = blocking for mono-root; broader P0.1/P0.4 parallel; firebase-dev/plan-only proceed now | **RESOLVED (owned gate)**

**Open Question Register** — `Q-# | Question | Why it matters | Required evidence | Owner | Blocking`
- **Q-1** | Can a GH Environment approval hold mid-job so plan→approve→apply keeps the `tfplan` in memory? | Determines whether the **hard** applied==approved (D-4) holds | validated | DevOps/SRE | **RESOLVED (Rev 2):** No — gate is job-boundary (`apply-toggles.yml`); D-4 preserved via **two-job age-sealed plan hand-off**. No longer blocking.
- **Q-2** | Secret-at-rest home for third-party statics | Reduces GitHub-resident crown-jewels | operator decision | Security/DevOps | **RESOLVED (Rev 2) → SOPS+age** (decrypt in-job; age key = the one Environment secret; also seals the D-4 plan)
- **Q-3** | Is a second human reviewer possible, or is single-maintainer self-approval the accepted gate? | "approval" semantics under `required_approving_review_count=0` | operator decision | operator | **RESOLVED (Rev 2) → accepted** self-approval (formalizes R-8)
- **Q-4** | firebase-dev reviewer-gated or fast-path (`development` env)? | DX vs consistency | operator decision | DevOps | **RESOLVED (Rev 2) → fast-path** (ungated); reviewer gate stays on firebase-prod + mono-root

**Risk Register** — `R-# | Risk | Prob | Impact | Mitigation | Detection | Owner`
- **R-1** | Crown-jewel concentration in one run (F5) — compromised action/dependency exfiltrates all | Low | Critical | SHA-pinned minimal actions, step-level injection, ephemeral App token, WIF, no plaintext artifacts | secret-scan, audit log | Security
- **R-2** | Auto-apply sweeps standing drift (F9) | Med | High | clean-baseline gate + no-destroy guard + Stage B plan-to-green | `plan -detailed-exitcode` | SRE
- **R-3** | Cancelled apply strands GCS lock → workstation needed | Med | Med | `cancel-in-progress:false` + in-CI force-unlock | lock-info in next run | SRE
- **R-4** | No true rollback for DNS/DB/server | Med | High | prevention (plan review, no-destroy guard); documented forward-fix | post-apply monitoring | SRE
- **R-5** | Self-referential pipeline creds lock out next run | Low | High | out-of-band bootstrap (D-7) | run failure | DevOps
- **R-6** | Plan-time RCE via edit to existing local-exec / module indirection (lint evadable) | Low | Critical | trust boundary (post-merge only), lockfile-review gate | code review | Security
- **R-7** | applied==approved downgrades if the cross-job plan hand-off is dropped/plaintext | Low (Rev-2: mitigated) | Med | **two-job age-sealed plan hand-off** (D-4); no-destroy guard as backstop | CI check that apply consumes the sealed plan | Architect
- **R-8** | Single-maintainer self-approval = no true four-eyes | High | Med | Environment gate as audit/speed-bump; **accepted (Rev-2, Q-3)** | — | operator (accepted)

**Rejected & Deferred Register** — `X-# | Item | State | Reason | Revisit trigger`
- **X-1** | PR-time credentialed plan / Atlantis apply-before-merge for the mono-root | REJECTED | `data.external` runs admin:org at plan time (F6) | if `github_org_settings.tf` local-exec/data.external is refactored out of plan-time
- **X-2** | Persistent self-hosted tailnet runner | REJECTED | standing compromise surface + SPOF + toolchain drift | only ephemeral `tag:ci` runner if a provider becomes tailnet-only unreachable
- **X-3** | Static broad admin:org PAT in CI | REJECTED | worsens open P0.1 | never (use App token)
- **X-4** | Plan-artifact hand-off / plaintext `tfplan` artifact | REJECTED | embeds sensitive values (F5, gitignore) | never
- **X-5** | Mono-root auto-apply-on-merge | DEFERRED | needs 6-point checklist + Stage B green | all preconditions met
- **X-6** | Splitting the mono-root for true least-privilege | DEFERRED | out of scope (TF redesign) | audit-tracked follow-up
- **X-7** | Diff-scoping guard as a HARD control | DEFERRED | no source-line→address provenance (C2) | replaced by clean-baseline + no-destroy
- **X-8** | Moving third-party secrets to keyless/OIDC | DEFERRED | providers lack GitHub OIDC (A-5) | a provider adds OIDC
- **X-9** | Auto-destroy in CI; removing manual `make` procedures | REJECTED (per card) | destructive / break-glass must survive | never

---

## Stopping Criteria — met
Card passed admission; every selected specialist returned a Round-1 report; every named conflict is RESOLVED or an owned, recorded gate; no OPEN blocking challenge remains (Q-1 blocks only the *hard* form of D-4, not shipping firebase-dev/plan-only); each accepted decision has consensus or recorded owned dissent. No Feature Specification was authored — the council stopped at decisions.

## Recommended Next Command
`/upstream engineering/2026-07-14-terraform-auto-apply-council/2026-07-14-council-record-v2.md`
(Rev 2 closed Q-1/A-3/Q-2/Q-3/Q-4 — the spec can proceed. Carry forward the **A-3 caveat** (confirm exact GitHub App permission set at build) and **A-5** (recheck third-party OIDC). Only **A-1/A-2** remain to validate during Stage A/B. Operator prep before mono-root apply: out-of-band bootstrap root for WIF+App creds, reconcile the F9 drift to zero, commit the two firebase tfvars.)
