---
title: Adopt Infisical as the Webhookr Secret Management Platform — Council Record
status: Approved
owner: thiagojv@gmail.com
created: 2026-07-14
updated: 2026-07-14
source: /council
priority: High
labels: [secrets, infisical, sops, gitops, argocd, security, sre, devops, migration, webhookr-svc, webhookr-ingest, webhookr-bff, webhookr-web, gitops, terraform]
supersedes-finding: 2026-07-01-webhookr-architecture-audit/2026-07-01-audit-report-v1.md#P0.4
---

# Council Record — Adopt Infisical as the Webhookr Secret Management Platform

> **Nature of this document.** This is the negotiated **decision record** produced by `/council`
> (multi-agent, Critical depth). It is **not** a Feature Specification — it stops at decisions.
> Every specialist position below came from a real independent Task execution (8 in Round 1,
> 5 in Round 2, 2 challenge responses); the Chair authored none of them. The authoritative
> output is the **Verified Story + Decision Ledger** at the end; downstream stages
> (`/upstream` → `/gatekeeper` → `/downstream` → `artifact-reconciliation`) consume it by stable ID.

## Metadata
- **Objective:** Refine the card "Adopt Infisical as the Webhookr Secret Management Platform" into a negotiated, implementation-ready decision record.
- **Outcome:** Adopt **Infisical Cloud**, scoped and phased, via the Kubernetes Operator; **SOPS+age retained** as a permanent bootstrap/DR anchor.
- **Depth:** **Critical** (secrets root-of-trust, infra architecture, production migration on a single-node cluster).
- **Status:** **CLOSED / Approved** — all four conflicts RESOLVED; no OPEN blocking challenge; owned dissent recorded.
- **Revision:** 1
- **Date:** 2026-07-14

## Execution Trace
| Started | Ended | Duration | Tokens | Revision |
|---|---|---|---|:--:|
| 2026-07-14T15:10:00Z | 2026-07-14T15:52:00Z | ~42 min wall-clock | ≈880k (summed specialist executions; Historian/Explore + Chair turns not separately reported by runtime) | 1 |

---

## 1. Council Plan

- **Admission:** PASS — problem (fragmented secrets with verified deficiencies) ✓; product (Webhookr, all repos/runtimes) ✓; actor (engineering team / developers) ✓; testable ACs ✓.
- **Depth rationale:** touches the secret root-of-trust, ArgoCD/GitOps architecture, and a live production single-node cluster → Critical.

**Specialist selection**

| Specialist | Required/Optional/Not | Why |
|---|---|---|
| Historian | Required | precedent for SOPS choice, prior secret decisions, egress posture |
| Devil's Advocate | Required | necessity vs incumbent SOPS; reversibility; scope creep |
| Security | Required | secret root-of-trust shift, least-privilege, egress, abuse |
| SRE/Platform | Required | K3s integration, availability, rollback, self-host vs Cloud |
| DevOps | Required | CI/CD + local-dev secret flow, incremental cutover |
| Software Architect | Required | source-of-truth boundary, references-vs-values, GitOps, failure domains |
| Backend | Required | validate the "no app code change" constraint |
| QA/Test | Required | per-env validation, silent-corruption, rollback drill |
| Product | Not required | no user-visible behaviour |
| Database | Not required | no schema change |
| Observability | Folded into SRE/Security | operator alerting |
| FinOps | Not launched | flagged as Q-2 (Cloud cost under lean budget) — surfaced, not resolved |

---

## 2. Evidence Package (verified facts)

- **Current mechanisms:** local gitignored `.env` (+ committed `.env.example`), a handful of GitHub Actions secrets, and **SOPS+age** `*.enc.yaml` in the `gitops` repo, decrypted by **ksops** in `argocd-repo-server` at manifest-generation time → native K8s Secret consumed via `envFrom.secretRef`. **No plaintext in Git today.**
- **Root of trust:** ONE cluster age recipient (`gitops/.sops.yaml`, `age1ggal…`), private key in cluster secret `argocd/sops-age` + offline operator backup. Every app encrypts to the same key — **no per-app / per-env isolation**.
- **Runtime topology:** single K3s node, single `webhookr` namespace. **No QA namespace, no dev cluster.** Real environments = local-dev + prod. All webhookr pods run the `default` ServiceAccount; **no RoleBinding grants secrets read** (secrets reach pods only via kubelet `envFrom`).
- **CI:** deploy path (`github-workflows/build-deploy.yml`) needs **zero app secrets** (only `GITOPS_DEPLOY_PAT` + native `GITHUB_TOKEN`); the sole app-secret consumer is `webhookr-svc/.github/workflows/rotate-encryption-key.yml` (manual `workflow_dispatch`, env `production`, `npm run crypto:rewrap`).
- **Egress:** deny-by-default, port-scoped (`gitops/webhookr/network-policies/egress-policy.yaml`); managed SaaS already allowlisted on 443 (Neon, Redis Cloud, Grafana Cloud, R2). API server `6443` is **tailnet-only, not public**.
- **Prior decision:** audit finding **P0.4 recommended "extend SOPS+age," not replace** (`2026-07-01-audit-report-v1.md:45`). No ADR ever recorded the SOPS choice. Infisical is greenfield (no references anywhere).
- **Sensitive invariants:** `QUEUE_ENCRYPTION_KEYS` must be byte-identical svc↔ingest (today manual duplication across two SOPS files); `EVENT_ENCRYPTION_KEYS` versioned; a wrong-but-valid key **boots clean and silently corrupts data**; `/health` is Redis-only.
- **Hidden seam:** Terraform hand-copies generated app secrets (`INTERNAL_API_TOKEN` — triple-copied into svc + ingest, `redis_url`, Neon `database_url`) into SOPS with no drift detection (`terraform/internal_api_token.tf`, `redis_cloud.tf`, `outputs.tf`).

---

## 3. Round-1 Reports (independent — faithful summaries; key quotes verbatim)

- **Historian** *(HIGH)* — SOPS entered via milestone commits (M3–M6.3), **no ADR**. The one directional record (**P0.4**) says *"move operational secrets to a secret manager (SOPS+age is already used in gitops — **extend it** to the TF flow)."* Must preserve deny-by-default egress + Git-as-source-of-truth; must not repeat the Redis/Unleash single-node SPOF; bootstrap chicken-and-egg means SOPS can't fully vanish. Governance: adoption **requires an ADR + Security Matrix update + explicit supersession of P0.4**.
- **Devil's Advocate** *(HIGH)* — *"Keep SOPS+age. Close the two real gaps at a fraction of the cost."* Most of the six card motivations are already solved by SOPS; the genuine gaps are local-`.env` off-machine backup + the single age-key SPOF. QA env = scope creep. Reversibility only holds if SOPS is never deleted.
- **Security** *(MED-HIGH)* — Infisical relocates the root of trust to a machine identity (more powerful than a decrypt-only age key). **Operator + CRD** (not sidecar/CSI); **per-workload identities**; resolve ArgoCD ownership; migration operator-driven, plaintext never in CI; don't delete SOPS until validated. Least-privilege + audit are *"genuinely **unmet** by the status quo."*
- **SRE/Platform** *(MED-HIGH)* — *"SOPS+KSOPS is already near-optimal for this scale."* If adopted → **Cloud** (self-host = non-rebuildable SPOF per the unleash-db single-replica/local-path/no-backup precedent) + Operator + parallel-run + SOPS as bootstrap anchor. Resolve selfHeal ownership via a distinct Secret name. Roll out web-first.
- **DevOps** *(HIGH on scoping)* — Minimal scope: deploy needs zero app secrets; only local dev + the one rotate workflow. Two Infisical envs (dev, prod), **no qa**. Prefer **GitHub OIDC** for CI bootstrap. Keep `.env.example` as schema.
- **Architect** *(MED-HIGH)* — Reference contract = `InfisicalSecret` CRD in Git (value in Infisical); `envFrom.secretRef` is the stable seam (deployment unchanged); `kustomization.yaml` is the reversible seam. **Collapse the TF→SOPS manual double-write.** Source of truth is **relocated, not restored** → drift becomes invisible to `argocd diff`.
- **Backend** *(HIGH)* — **No-code-change constraint HOLDS**: 100% of secret reads are `process.env` via a config layer with boot-time fail-fast; no file/mount reads; values delivered byte-identical. Caveats are ops-only (traefik TLS is a typed Secret, not app env; BFF has soft `?? ''` fallbacks worth hardening).
- **QA/Test** *(HIGH on risk model)* — Core risk is **silent corruption** (wrong-but-valid key). `/health` is Redis-only → insufficient gate. Requires rendered-Secret **parity diff** + **functional round-trip** (ingest→queue→svc→deliver, zero `decrypt_failed`) + **removal gate** + **executed rollback drill**; recommends a throwaway staging namespace to rehearse the svc/ingest pair.

---

## 4. Consensus Matrix (Round-1 aggregation)

| Topic | Agreement | Real conflict? |
|---|---|---|
| No app code change | `process.env` config layer; `envFrom.secretRef` stable seam | No — consensus |
| Necessity: adopt vs extend-SOPS | SOPS is offline/zero-runtime-dependency | **YES — F-1 (blocking)** |
| Deployment model Cloud vs self-host | self-host on one node = SPOF anti-pattern | **YES — F-2** |
| Scope of migration | local dev + CI in scope | **YES — F-3 (foundational)** |
| Cluster bootstrap / machine-identity auth | a residual bootstrap secret is irreducible | **YES — F-4** |
| Integration pattern | Operator + CRD → native Secret (not sidecar/CSI) | No — consensus |
| ArgoCD selfHeal/prune ownership | distinct name / Prune=false / ignoreDifferences | No — consensus |
| Reversible seam | keep `*.enc.yaml`; revert = one-line `kustomization.yaml` | No — consensus |
| Validation / silent-corruption | parity diff + functional round-trip + removal gate | No — consensus |
| Cross-service invariant | shared Infisical path for `QUEUE_ENCRYPTION_KEYS` | No — consensus |
| QA Infisical environment | no standing `qa` env; ephemeral rehearsal OK | No — consensus |
| Governance | ADR + Security Matrix + supersede P0.4 | No — consensus |

---

## 5. Revised Positions (Round 2 — conflict resolution)

- **Devil's Advocate → REVISE:** *"I withdraw my Round-1 claim that a second age recipient + per-app keys can satisfy these ACs — on **audit** it categorically cannot, and on **least-privilege** it re-centralizes at the repo-server."* Holds: minimum footprint (crown-jewel secrets only), SOPS permanent, two availability guardrails (outage + never-delete-SOPS).
- **Security → DEFEND F-1 / REVISE F-2→Cloud / WITHDRAW F-4 absolutism:** *"Centralized read audit is physically impossible under SOPS… the security bright line."* Cloud egress is *"one more SaaS on the exact rail we already run Neon/Redis-Cloud on."* Tailnet-only API server forecloses K8s-native auth → a static Universal-Auth secret via SOPS is forced and acceptable. A second age recipient is an **availability/escrow** fix, **not** least-privilege — must not be scored against that AC.
- **SRE → DEFEND Cloud + DEFEND SOPS-anchor:** self-host reintroduces *"a non-rebuildable critical SPOF the unleash-db precedent proves this node won't back or replicate."* Attaches 5 reliability requirements. Verified externally: Infisical K8s-native auth validates the SA-JWT by calling the cluster's TokenReview API, which the tailnet-only `6443` refuses → Cloud forces Universal Auth. *"SOPS cannot be fully decommissioned… 'single platform' is unachievable."*
- **DevOps → REVISE:** *"I accept runtime migration to the Infisical Operator"* — bound by 3 constraints: (C1) SOPS/ksops retained as break-glass, **not deleted**; (C2) byte-identical parallel-run verification before cutover; (C3) the operator identity documented in the DR runbook. Residual dissent: the migration does **not** reduce imperative bootstrap secrets; deploy pipeline stays out (zero app secrets).
- **Architect → REVISE (phasing) / DEFEND (contract + 2 unique constraints):** minimal-scope and reference-contract are *"the SAME plan at two phases."* Architectural verdict: **SOPS-ciphertext-in-Git does not literally satisfy the AC** ("references… not values" + "K8s workloads consume from Infisical") → Phase-2 runtime cutover is required to close the card, or the AC must be amended. Non-negotiables: collapse the TF→SOPS double-write; source of truth is relocated (drift invisible to `argocd diff`).

**Conflict outcomes:** F-1 RESOLVED (adopt, scoped); F-2 RESOLVED (Cloud); F-3 RESOLVED (phased, runtime in scope, SOPS break-glass); F-4 RESOLVED (static UA via SOPS, forced by tailnet-only API).

---

## 6. Challenge–Response (Round 5)

| Challenge | Target | Response | Status |
|---|---|---|---|
| **Does the operator leave the last-rendered Secret intact on Infisical outage/auth-fail?** (SRE flagged as potential blocker) | SRE/reliability | **CONFIRMED-by-source**: in `Infisical/kubernetes-operator` `reconciler.go`, auth failure and API-fetch failure both `return` early **before** any Secret write; the managed Secret is untouched and pods read it from the API server independent of the operator. Caveat: guarantee holds under `creationPolicy: Orphan` (**default**); `Owner` cascades a delete on **CRD-delete** (a distinct event). Doc-unverified → **must run a throwaway-namespace outage test before Phase 2** (render → block egress → restart pod → observe Secret unchanged + pod Running). | **RESOLVED** (accepted-risk: pin version, use Orphan, test first) → A-1, Q-3 |
| **Does adopting Infisical actually enforce least-privilege on this single-namespace/default-SA topology?** (Devil's residual dissent) | Security/Architect | **PARTIALLY MET.** Repo confirmed: no webhookr Deployment sets `serviceAccountName`; no RoleBinding grants `secrets get/list`; secrets reach pods only via kubelet `envFrom`. Infisical closes the **read boundary** (per-workload identity via `kubernetesAuth.serviceAccountRef` — confirmed in CRD docs) but **not** the **K8s Secret-object boundary**. Full AC requires net-new work: per-app ServiceAccounts, `resourceNames`-scoped RBAC, `automountServiceAccountToken:false`, no namespace-wide secrets read. **Feasible without a namespace redesign.** | **RESOLVED** → D-5, R-4 |

---

## 7. Council Decision

Resolved by the priority order (verified requirements → architectural constraints → security/integrity → operational safety → simplicity → maintainability → speed):

**ACCEPTED.** Adopt **Infisical Cloud** as the authoritative store for in-scope application secrets, integrated via the **Kubernetes Operator (InfisicalSecret CRD → native Secret, `creationPolicy: Orphan`, pinned version)**, phased (local-dev → TF-collapse + rotate + operator pilot → gated per-app runtime cutover), with **per-workload identities + per-app ServiceAccount/RBAC**, **SOPS+age retained permanently** as bootstrap/DR anchor and break-glass, and a **validation gate + executed rollback drill** before any old copy is removed. An **ADR + Security Matrix update** are required and **supersede audit P0.4**.

**REJECTED.** Self-hosted Infisical; extend-SOPS-instead (for the audit/least-privilege ACs); sidecar/CSI; full SOPS retirement; standing QA env; deploy-pipeline secret migration.

**KEY OWNED DISSENTS (accepted risks, not blockers).** Vendor blast-radius of Cloud (Security/SRE — R-1); DR cold-bootstrap on a single node (DevOps — R-3); "least-privilege is delivered by the RBAC work, not by adoption" (Devil's — folded into D-5 as mandatory).

---

## 8. Council Record

### 8.1 Verified Story

**Problem.** Webhookr's secrets are fragmented across local `.env`, GitHub Actions secrets, and SOPS+age→ksops K8s Secrets. This is *not* a plaintext-in-Git problem (already solved), but the incumbent has three verified deficiencies it **categorically cannot fix**: (1) **no centralized read-audit** (SOPS decrypts client-side — nothing to log); (2) **no per-identity, revocable least-privilege** (one age key decrypts everything; `argocd-repo-server` is an all-keys super-reader); (3) **manual, drift-prone distribution** (TF→SOPS hand-copy; local `.env` filled by hand).

**Actor.** The engineering team — developers (onboarding, local dev) and the operator (rotation, DR).

**Business context.** Single-founder lean SaaS; single K3s node / single `webhookr` namespace; deny-by-default egress; GitOps via ArgoCD (auto-sync, selfHeal, prune). API server tailnet-only.

**Constraints.** Git never holds plaintext (already true); GitOps preserved; reversible; incremental; production stays up.

**Acceptance Criteria (testable):**
- **US-1** — Infisical Cloud is the authoritative store for in-scope app secrets; their **values are edited in Infisical, not Git**.
- **US-2** — Each service boots in local dev with **no manually-maintained real `.env`** (via `infisical run`); `.env.example` retained as schema.
- **US-3** — The one CI job consuming app secrets (`rotate-encryption-key.yml`) reads them from Infisical via a machine identity (GitHub OIDC preferred), not GitHub Actions secrets.
- **US-4** — K8s workloads receive in-scope secrets via the **Infisical Operator (CRD → native Secret)** through the **unchanged `envFrom.secretRef`** seam (no app code change).
- **US-5** — Least-privilege enforced at **both** boundaries: one scoped Infisical identity per workload **and** per-app ServiceAccount + `resourceNames`-scoped RBAC + `automountServiceAccountToken:false` + no namespace-wide secrets read.
- **US-6** — Environment isolation via Infisical environments (dev/prod); **no standing `qa` env**.
- **US-7** — Per-app cutover validated by (a) **byte-identical parity** between Infisical- and SOPS-rendered Secret and (b) an **end-to-end functional round-trip** (ingest→queue→svc→deliver, zero `decrypt_failed`) before any old copy is removed.
- **US-8** — SOPS `*.enc.yaml` + ksops generator **retained** through migration (reversible seam) and **permanently** for bootstrap/DR-anchor secrets; removed per-secret only after the gate passes.
- **US-9** — Rollback is **documented and executed as a drill** (low-blast-radius app first), timed against an RTO.
- **US-10** — An **ADR** is written, the **Security Matrix** updated, and **audit P0.4 explicitly superseded**.

**Out of scope.** Terraform *provider* credentials; platform/observability ksops secrets (tailscale, traefik-tls, otel, ghcr-pull, backup, unleash); deploy-pipeline secret migration; creating a QA cluster/namespace; replacing GitOps/ksops wholesale; app business-logic changes; non-sensitive config.

### 8.2 Decision Ledger

**Decision Log**

| ID | Decision | Serves | Evidence | Alternatives rejected | Trade-off | Owner |
|---|---|---|---|---|---|---|
| D-1 | Adopt Infisical (not extend-SOPS) | US-1,US-5,US-10 | Security verdict: audit categorically unmeetable by SOPS; Devil's revised concession | Extend-SOPS (X-2) | New platform + dependency | Eng team |
| D-2 | **Infisical Cloud**, not self-hosted | US-1 | SRE: self-host = non-rebuildable SPOF (unleash-db precedent); Security revised (egress already 443-SaaS class) | Self-host (X-1) | Vendor blast-radius (R-1) | SRE |
| D-3 | Integrate via **Operator + InfisicalSecret CRD → native Secret**, `creationPolicy: Orphan`, pinned version | US-4 | Backend: `envFrom` unchanged; Challenge-1 CONFIRMED-by-source | Sidecar/CSI (X-6) | Operator is Tier-0 | SRE |
| D-4 | ArgoCD ownership: distinct Secret name during parallel-run; operator-managed Secret excluded from prune (`Prune=false`/`IgnoreExtraneous`) | US-4,US-7 | selfHeal+prune in `gitops/apps/*`; 3 specialists converged | — | Managed Secret leaves GitOps audit | SRE |
| D-5 | Least-privilege at **both** boundaries (per-workload identity **+** per-app SA/RBAC/`automount:false`) | US-5 | Challenge-2: PARTIALLY MET without RBAC work | Adopt-only | Net-new `gitops/webhookr` RBAC files | Security |
| D-6 | Bootstrap: static **Universal-Auth** client secret delivered via **SOPS/ksops** | US-4 | 6443 tailnet-only forecloses TokenReview; Security withdrew "no static token" | K8s-native SA-JWT (needs inbound API exposure) | One residual static secret (R-5) | Security/SRE |
| D-7 | **SOPS retained permanently** as bootstrap/DR anchor + break-glass | US-8 | SRE/Security: bootstrap irreducible; DevOps break-glass | Full SOPS retirement (X-7) | "Single platform" is aspirational | Operator |
| D-8 | **Phased**: P0 local-dev → P1 TF-collapse + rotate + operator pilot on 1 low-risk app → P2 gated per-app runtime cutover (svc/ingest last, shared path) | US-2,US-3,US-7,US-8 | DevOps/Architect reconciled to same phased plan | Big-bang | Longer timeline | Tech lead |
| D-9 | Scope line: in = app runtime + local dev + rotate CI + TF-generated app secrets; out = TF provider creds, platform ksops secrets, deploy pipeline, QA env | Out-of-scope | DevOps (deploy=0 secrets); Architect scope line | — | Some fragmentation persists (intended) | Architect |
| D-10 | **Collapse TF→SOPS manual double-write**: TF/operator writes generated app secrets once into Infisical | US-1 | `terraform/internal_api_token.tf` triple-copy | Status quo hand-copy | — | Architect |
| D-11 | `QUEUE_ENCRYPTION_KEYS`/`INTERNAL_API_TOKEN`/`REDIS_URL` on a **shared Infisical path** (svc+ingest) | US-7 | byte-identity invariant, `queue-encryption-secret.md` | Duplicate per-app | — | Backend |
| D-12 | **Validation gate + executed rollback drill** before removing any old copy; functional round-trip (not `/health`) | US-7,US-9 | QA silent-corruption model | Health-only gate | Slower cutover | QA |
| D-13 | **Drift detection outside `argocd diff`** (operator reconcile-status/staleness alerting) | US-1 | Architect Constraint B | — | New alert | SRE |

**Assumption Register**

| ID | Assumption | Source | Validation | Status | Impact if false |
|---|---|---|---|---|---|
| A-1 | Operator retains last-known-good Secret on outage/auth-fail | Challenge-1 (`reconciler.go`) | Throwaway-ns outage test before P2 | CONFIRMED-by-source, doc-unverified | Outage = prod-down on restart (blocker) |
| A-2 | Cloud can't do K8s-native TokenReview vs tailnet-only API → static UA required | Security/SRE inference on verified 6443-not-public | Confirm at setup | PLAUSIBLE | Could avoid static secret (better) |
| A-3 | `creationPolicy` defaults to Orphan (CRD-delete ≠ Secret-delete) | Infisical CRD docs | — | FACT | Incident CRD-delete wipes Secret |
| A-4 | Infisical Cloud cost fits lean budget | FinOps not convened | FinOps check | UNVALIDATED | Reconsider self-host (X-1) |

**Conflict Register**

| ID | Topic | Positions (agents) | Resolution | Status |
|---|---|---|---|---|
| F-1 | Adopt vs extend-SOPS | Devil's/SRE "keep SOPS" vs Security "audit unmeetable" | Devil's revised; audit is the bright line → adopt, scoped | RESOLVED |
| F-2 | Cloud vs self-host | SRE/Architect Cloud vs Security egress→self-host | Security revised to Cloud; self-host = SPOF | RESOLVED (Cloud) |
| F-3 | Scope | DevOps minimal vs Architect/others runtime | Both revised → phased, runtime in scope, SOPS break-glass | RESOLVED |
| F-4 | Bootstrap auth | Security SA-JWT vs SRE SOPS-static | Tailnet-only API forecloses SA-JWT → static UA via SOPS | RESOLVED |

**Open Question Register**

| ID | Question | Why it matters | Required evidence | Owner | Blocking |
|---|---|---|---|---|---|
| Q-1 | Is centralized read-audit + revocable least-privilege a genuine requirement (vs just env isolation)? | It is the load-bearing justification for D-1 | Eng-team confirmation (card ACs already support it) | Eng team | No (AC text supports) |
| Q-2 | Infisical Cloud pricing under lean budget | Go/no-go affordability | FinOps analysis | Eng team | Blocks go, not shape |
| Q-3 | Empirically validate A-1 (outage → Secret retained) | Gates the graceful-degradation guarantee | Throwaway-ns test | SRE | **Blocks Phase 2 only** |
| Q-4 | Close card via Phase 2, or amend AC to "no plaintext in Git"? | Literal AC satisfaction | Council resolved: Phase 2 in scope | Council | Resolved |

**Risk Register**

| ID | Risk | Prob | Impact | Mitigation | Detection | Owner |
|---|---|---|---|---|---|---|
| R-1 | Cloud vendor holds all secrets (blast-radius) | Low | High | Accepted (Neon/Redis-class); scoped identities; reopen F-2 if zero-external-dep required | Infisical audit log | Security |
| R-2 | Silent corruption (wrong-but-valid key) | Med | High | Parity diff + functional round-trip + gate (D-12) | `svc.ingest.decrypt_failed`=0 over soak | QA |
| R-3 | DR cold-bootstrap on single node depends on Infisical | Low | High | SOPS offline break-glass (D-7); document in DR runbook | DR drill | SRE |
| R-4 | Least-privilege illusion (adopt without RBAC) | Med | Med | D-5 mandatory (per-app SA/RBAC) | Negative cross-app read test | Security |
| R-5 | Static UA client-secret compromise | Low | High | Single scoped read-only identity, SOPS-sealed, short TTL, rotation, IP-pin | Infisical access log | Security |
| R-6 | Drift invisible to `argocd diff` | Med | Med | Operator reconcile-status/staleness alert (D-13) | Alert | SRE |

**Rejected & Deferred Register**

| ID | Item | State | Reason | Revisit trigger |
|---|---|---|---|---|
| X-1 | Self-hosted Infisical | REJECTED | Non-rebuildable SPOF + bootstrap circularity on single node | HA cluster, or zero-external-dep requirement |
| X-2 | Extend SOPS (2nd recipient + per-app keys) instead | REJECTED (for audit/LP ACs) | Categorically no read-audit; re-centralizes at repo-server | If audit AC dropped (2nd recipient still worth doing as escrow) |
| X-3 | Migrate platform/observability ksops secrets | DEFERRED | Bootstrap-plane, no per-reader authz need | After app-plane validated |
| X-4 | Migrate deploy pipeline / TF provider creds | OUT/DEFERRED | Deploy needs 0 app secrets; provider creds are control-plane | Separate decision |
| X-5 | Standing QA Infisical env | REJECTED | No QA cluster consumes it (drift/false confidence) | QA cluster stood up |
| X-6 | Sidecar/CSI injection | REJECTED | Worse availability; sprays egress/token surface | — |
| X-7 | Fully retire SOPS ("single platform") | REJECTED | Bootstrap anchor irreducible | — |

---

## 9. Recommended Next Command

```
/upstream <the Infisical card> + this Council Record
```

The upstream stage should produce the Feature Specification honouring every `D-#`, and **must** carry forward two gates before Phase-2 runtime cutover: **Q-3** (empirically validate A-1 on a throwaway namespace) and **Q-2** (FinOps confirms Cloud cost). The **D-5** RBAC work (per-app ServiceAccounts + scoped Roles) is net-new and must be specified, not assumed — without it the least-privilege AC (US-5) is only partially met. On implementation, `artifact-reconciliation` must publish the **ADR** (US-10) and the **Security Matrix** row, superseding audit P0.4.
