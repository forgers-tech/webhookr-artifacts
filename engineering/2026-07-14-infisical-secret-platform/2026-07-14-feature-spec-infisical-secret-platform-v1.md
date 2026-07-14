---
title: Adopt Infisical as the Webhookr Secret Management Platform — Feature Specification
status: Draft
owner: thiagojv@gmail.com
council_record: ../2026-07-14-infisical-secret-platform-council/2026-07-14-council-record-v1.md
created: 2026-07-14
updated: 2026-07-14
estimate: GG
priority: High
labels: [secrets, infisical, sops, gitops, argocd, security, sre, devops, migration, webhookr-svc, webhookr-ingest, webhookr-bff, webhookr-web, gitops, terraform]
---

# Feature Specification

This document describes everything required to implement the adoption of Infisical as Webhookr's secret management platform, from business motivation to production rollout.

> **Traceability.** This spec authors the decisions in its Council Record (`../2026-07-14-infisical-secret-platform-council/2026-07-14-council-record-v1.md`); it does not make new ones. Every rule, contract and architectural section cites its governing decision as `⟶ D-#`. Content that cannot cite a `D-#` or `US-#` is out of bounds — reopen Council instead of inventing it.

## Execution Trace

| Started | Ended | Duration | Tokens | Revision |
|---------|-------|----------|--------|:--------:|
| 2026-07-14T16:05:00Z | 2026-07-14T16:22:00Z | ~17 min | n/a (runtime usage not exposed to the authoring stage) | 1 |

## 1. Context

### Background
Webhookr's secrets live across three planes: local gitignored `.env` (with committed `.env.example` templates), a small set of GitHub Actions secrets, and **SOPS+age** `*.enc.yaml` files in the `gitops` repo, decrypted by **ksops** inside `argocd-repo-server` at manifest-generation time into native K8s Secrets consumed via `envFrom.secretRef`. There is **no plaintext in Git today**. The runtime is a single K3s node, single `webhookr` namespace; the API server (`6443`) is tailnet-only; egress is deny-by-default and port-scoped.

### Problem Statement
The incumbent has three deficiencies it **categorically cannot fix** (Verified Story): (1) **no centralized read-audit** — SOPS decrypts client-side, so there is nothing to log; (2) **no per-identity, revocable least-privilege** — one age key decrypts everything and `argocd-repo-server` is an all-keys super-reader; (3) **manual, drift-prone distribution** — Terraform hand-copies generated app secrets into SOPS (`INTERNAL_API_TOKEN` triple-copied) with no drift detection, and local `.env` is filled by hand.

### Objective
Infisical Cloud is the authoritative store for the in-scope application secrets; local dev, the one secret-consuming CI job, and Kubernetes workloads consume from Infisical; Git holds references (and retained SOPS ciphertext for bootstrap/DR only); least-privilege and environment isolation are enforced; migration is reversible, incremental, and leaves production running. `⟶ D-1, D-9`

### Business Value
Reduce operational risk (drift, single-copy loss), improve onboarding, enable centralized read-audit and revocable per-workload access, and simplify rotation — while preserving GitOps and removing secret values from developer workstations. `⟶ D-1`

### Success Metrics
- 100% of in-scope app secrets served from Infisical after cutover, with SOPS retained only for bootstrap/DR-anchor secrets. `⟶ D-1, D-7`
- Zero manually-maintained real `.env` files for local dev (each service boots via `infisical run`). `⟶ US-2`
- Per-workload access enforced: a negative cross-app read test fails for every workload. `⟶ US-5, D-5`
- Zero `svc.ingest.decrypt_failed` across each cutover soak window. `⟶ US-7, R-2`
- An executed (not just documented) rollback drill with a recorded RTO. `⟶ US-9, D-12`

## 2. Scope

### In Scope
- **App runtime secrets** for `webhookr-svc`, `webhookr-ingest`, `webhookr-bff`, `webhookr-web` — delivered via the Infisical Kubernetes Operator. `⟶ D-3, D-9`
- **Local development** — replace hand-filled `.env` with `infisical run`; keep `.env.example` as schema. `⟶ US-2, D-8`
- **The one secret-consuming CI job** — `webhookr-svc/.github/workflows/rotate-encryption-key.yml`. `⟶ US-3, D-9`
- **Terraform-generated app secrets** — `INTERNAL_API_TOKEN`, `redis_url`, Neon `database_url` written once to Infisical (collapse the TF→SOPS manual copy). `⟶ D-10`
- **Least-privilege enablement** — per-workload Infisical machine identities + per-app ServiceAccounts + scoped RBAC. `⟶ D-5`
- **ADR + Security Matrix update**, superseding audit finding P0.4. `⟶ US-10`

### Out of Scope
- Terraform **provider** credentials (control-plane bootstrap). `⟶ D-9, X-4`
- Platform/observability ksops secrets (tailscale-operator, traefik-tls, otel-collector, ghcr-pull, backup, unleash) — deferred to a later phase. `⟶ D-9, X-3`
- The deploy pipeline / `build-deploy.yml` (needs zero app secrets). `⟶ D-9, X-4`
- Creating a QA cluster/namespace or a standing QA Infisical env. `⟶ D-9, X-5`
- Self-hosting Infisical; sidecar/CSI injection; full SOPS retirement. `⟶ X-1, X-6, X-7`
- App business-logic changes (the `envFrom.secretRef` contract is unchanged). `⟶ D-3`

### Assumptions
- **A-1** — the operator retains the last-known-good managed Secret on Infisical outage/auth-fail (CONFIRMED-by-source in `Infisical/kubernetes-operator` `reconciler.go`; doc-unverified). Must be empirically validated before Phase 2 (see Open Questions). *Impact if false: an outage becomes production-down on pod restart.*
- **A-2** — Infisical Cloud cannot validate K8s-native TokenReview against the tailnet-only API server, so a static Universal-Auth identity is required. `⟶ D-6`
- **A-3** — `creationPolicy` defaults to `Orphan` (CRD-delete does not cascade-delete the managed Secret). `⟶ D-3`
- **A-4** — Infisical Cloud cost fits the lean budget (UNVALIDATED — FinOps not convened). See Open Questions.
- `Assumption:` the Infisical Kubernetes Operator supports per-`InfisicalSecret` authentication scoping (`kubernetesAuth.serviceAccountRef` / `universalAuth.credentialsRef`), enabling one identity per workload (confirmed against Infisical CRD docs during Council Challenge-2; pin the exact operator version at implementation).

### Risks
- **R-1** — Cloud vendor holds all secrets (blast-radius). Mitigation: accepted (Neon/Redis-class); scoped identities; reopen F-2 if a zero-external-dependency requirement appears.
- **R-2** — Silent corruption: a wrong-but-valid encryption key boots clean and corrupts data. Mitigation: parity diff + functional round-trip gate. `⟶ D-12`
- **R-3** — DR cold-bootstrap on a single node now depends on Infisical + its identity. Mitigation: SOPS retained as offline break-glass; document in the DR runbook. `⟶ D-7`
- **R-4** — Least-privilege illusion (adopt without the RBAC work). Mitigation: D-5 is mandatory, not optional.
- **R-5** — Static UA client-secret compromise. Mitigation: single scoped read-only identity, SOPS-sealed, short TTL, rotation, IP-pin to node egress where supported.
- **R-6** — Drift invisible to `argocd diff` (value leaves Git). Mitigation: operator reconcile-status/staleness alerting. `⟶ D-13`

### Open Questions
- **TODO: (Q-2)** FinOps must confirm Infisical Cloud pricing under the lean budget before go-live. *Blocks go, not authoring.* (Owner: eng team.)
- **TODO: (Q-3)** Empirically validate A-1 on a throwaway namespace (render a secret → block egress to Infisical → restart/reschedule the pod → assert the managed Secret is unchanged and the pod reaches Running) **before any Phase-2 runtime cutover.** *Blocks Phase 2.* (Owner: SRE.)
- **TODO: (Q-1)** Confirm that centralized read-audit + revocable least-privilege are genuine requirements (the load-bearing justification for D-1). Council read the card ACs as supporting this; a change would reopen Council. *Non-blocking given AC text.*

## 3. Functional Design

> This feature is an infrastructure/secret-delivery change with no end-user UI. "Flows" below describe the **operator/developer** flows for migrating and consuming secrets.

### Main Flow
Per-app cutover (the unit of migration), executed inside the single prod environment (there is no QA cluster): `⟶ D-8`
1. Author the app's Infisical secrets under the app's project path (and shared secrets under the shared path). `⟶ D-11`
2. Commit an `InfisicalSecret` CRD for the app that renders a managed Secret under a **distinct name** (e.g. `webhookr-svc-secrets-infisical`), `creationPolicy: Orphan`, authenticated by the app's per-workload identity. `⟶ D-3, D-4, D-5`
3. Validate: (a) byte-identical **parity diff** between the Infisical-rendered Secret and the ksops-rendered Secret; (b) **functional round-trip** (ingest→queue→svc→deliver) with zero `decrypt_failed`. `⟶ D-12`
4. Flip the app Deployment's `envFrom.secretRef.name` to the Infisical-managed Secret. `⟶ D-3`
5. Soak; confirm health + zero decrypt failures over the window. `⟶ D-12`
6. Only then remove the app's ksops generator from `kustomization.yaml` (keep the `*.enc.yaml` in Git until the gate is fully recorded). `⟶ D-7, D-12`

### Alternative Flows
#### AF-01 — Local development consumption `⟶ US-2, D-8`
Developer runs `infisical login` (personal, browser OAuth) → `infisical init` (links repo to the Infisical project + `dev` env) → `infisical run --env=dev --path=/webhookr-<svc> -- <npm run start:dev>`. No real `.env` is hand-filled; `.env.example` remains the committed schema. Offline escape hatch: `infisical export --env=dev --format=dotenv > .env`.

#### AF-02 — Rotate-encryption-key CI job `⟶ US-3, D-9`
`rotate-encryption-key.yml` fetches `DATABASE_URL`, `EVENT_ENCRYPTION_KEYS`, `EVENT_ENCRYPTION_ACTIVE_KEY` from Infisical (via a machine identity; GitHub OIDC preferred, else a single `INFISICAL_MACHINE_TOKEN` GH secret) instead of `${{ secrets.* }}`, then runs `npm run crypto:rewrap`. GH secrets removed only after a green dry-run.

#### AF-03 — Terraform-generated app secrets `⟶ D-10`
Terraform (or the operator) writes `INTERNAL_API_TOKEN`, `redis_url`, Neon `database_url` **once** into Infisical; the manual "copy the sensitive TF output into both services' SOPS files" step is retired.

### Error Flows
#### EF-01 — Infisical unreachable during a value change `⟶ A-1, R-3`
Running pods keep the last-rendered Secret (operator does not blank it). Rotations/new keys do not propagate until Infisical returns; the operator requeues with backoff. No fallback that re-exposes plaintext is permitted (no re-enabling SOPS render as an automatic failover, no dropping a plaintext `.env`).
#### EF-02 — Parity diff mismatch at cutover `⟶ D-12`
Cutover halts; the app stays on the ksops Secret; the Infisical secret set is corrected and re-diffed. No `envFrom` flip until the diff is empty.
#### EF-03 — Functional round-trip shows `decrypt_failed` `⟶ R-2, D-11`
Indicates a wrong-but-valid key (most likely `QUEUE_ENCRYPTION_KEYS` divergence). Halt; verify the shared-path value is byte-identical for svc and ingest; never remove the old copy.

### Business Rules
#### BR-001 ⟶ D-1, D-9
Only in-scope application secrets migrate to Infisical. Bootstrap, control-plane, and platform/observability secrets remain on SOPS/ksops.
#### BR-002 ⟶ D-3
Each in-scope secret group is delivered by an `InfisicalSecret` CRD that renders a **native** K8s Secret with `creationPolicy: Orphan`, consumed via the existing `envFrom.secretRef` (no app code change). Sidecar/CSI injection is prohibited. `⟶ X-6`
#### BR-003 ⟶ D-4
During parallel-run the operator-managed Secret uses a name **distinct** from the ksops-rendered Secret, and is excluded from ArgoCD prune (`Prune=false` / `argocd.argoproj.io/compare-options: IgnoreExtraneous`). The Deployment `envFrom` name is flipped to the managed Secret only after validation. The operator must never manage a Secret whose name is also produced by ksops in the same app.
#### BR-004 ⟶ D-5
Least-privilege is enforced at **both** boundaries: (a) one Infisical machine identity per workload, scoped read-only to that workload's secret path; (b) a per-app ServiceAccount, `resourceNames`-scoped Role/RoleBinding (never a namespace-wide `secrets get/list`), and `automountServiceAccountToken: false` where the pod does not call the K8s API.
#### BR-005 ⟶ D-6
The operator authenticates to Infisical Cloud with a static Universal-Auth client secret, sealed as a `*.enc.yaml` and rendered by the existing ksops path. It is scoped read-only to the operator's project/env, short-TTL, rotated, and IP-pinned to the node egress where supported.
#### BR-006 ⟶ D-7
SOPS `*.enc.yaml` + the ksops generator are retained through migration as the reversible seam, and **permanently** for bootstrap/DR-anchor secrets. A migrated secret's old copy is removed only after its validation gate is recorded (BR-008); bootstrap/DR-anchor secrets are never removed.
#### BR-007 ⟶ D-11
Shared cross-service secrets — `QUEUE_ENCRYPTION_KEYS`, `INTERNAL_API_TOKEN`, `REDIS_URL` — live at a single shared Infisical path referenced by both `webhookr-svc` and `webhookr-ingest`, making byte-identity structural rather than manual.
#### BR-008 ⟶ D-12
No old secret copy (`.enc.yaml` entry, GH Actions secret, or `.env` value) is deleted until (a) the parity diff is empty and (b) the functional round-trip passes over a soak window, both recorded (date + operator), and (c) for unrecoverable values (`EVENT_ENCRYPTION_KEYS`) an offline escrow is verified restorable.
#### BR-009 ⟶ D-10
Terraform-generated app secrets are written once to Infisical; the manual TF→SOPS copy is retired. Terraform provider credentials are unchanged. `⟶ D-9`
#### BR-010 ⟶ D-2, D-13
Infisical **Cloud** is the store. The operator's namespace gets a self-contained egress policy allowing DNS + external `443` only (SSRF-guarded ipBlock excepting RFC1918/CGNAT/link-local), added with `default-deny-egress` last. Because the value leaves Git, drift is detected via operator reconcile-status/staleness alerting, not `argocd diff`.

### Validation Rules
- Parity diff compares **decoded** values (key set + bytes, sorted), watching trailing newlines/quoting (SOPS emits `stringData`; the operator emits base64 `data`). `⟶ D-12`
- `QUEUE_ENCRYPTION_KEYS` and `INTERNAL_API_TOKEN` decoded values must be byte-identical across svc and ingest (`cmp`/`sha256sum`, not eyeball). `⟶ D-11`
- Boot-time app validation (existing, unchanged) rejects missing/malformed/wrong-length keys; it does **not** catch wrong-but-valid keys — hence the mandatory round-trip. `⟶ R-2`

### Permissions
- **Per-workload Infisical machine identity** — read-only, single project/env, single secret path. `⟶ D-5`
- **Operator identity** — read-only, operator project/env only; static UA, SOPS-sealed. `⟶ D-6`
- **Human developers** — personal Infisical login (per-user audit + instant revoke on offboarding); never a shared machine identity on a laptop. `⟶ AF-01`
- **Migration bulk-load** — performed by the operator on a trusted workstation (`sops -d` → Infisical), never from CI. `⟶ D-6`
- **Per-app K8s ServiceAccount** — no namespace-wide secrets read; `resourceNames`-scoped only. `⟶ D-5`

## 4. Non-Functional Requirements

### Performance
No hot-path impact: secrets are materialized into a native K8s Secret and read once at container start via `envFrom`; the operator reconcile is out-of-band. `⟶ D-3`

### Scalability
Single-cluster, single-namespace; one `InfisicalSecret` + identity per workload. No horizontal scaling concern; the operator handles all app CRDs. `⟶ D-3`

### Availability
Running/rescheduled pods survive an Infisical outage because they read the etcd/datastore-persisted managed Secret (A-1). Only value **changes** (rotation/new keys) block during an outage. Operator hardening: PodDisruptionBudget `minAvailable: 1`, resource requests/limits, liveness/readiness, conservative `resyncInterval`, `selfHeal` restores the operator Deployment. `⟶ A-1, R-3, D-3`

### Security
- New root of trust = the operator's static UA client secret; scoped, SOPS-sealed, rotated, IP-pinned. `⟶ D-6, R-5`
- Per-workload least-privilege at both boundaries. `⟶ D-5, R-4`
- Migration plaintext never touches CI. `⟶ D-6`
- No outage fallback may re-expose plaintext. `⟶ EF-01`
- Requires an ADR + a Security Matrix row (new external 443 egress for the operator namespace; new machine identities), superseding audit P0.4. `⟶ US-10, D-2`

### Observability
#### Logging
Operator reconcile logs; log-based alert on Infisical-operator reconcile/auth errors (mirror the existing Loki-alert pattern in `gitops/docs/runbooks/grafana-alerts.md`). `⟶ D-13`
#### Metrics
Managed-Secret staleness / last-successful-sync age; Infisical API 5xx/timeout rate; identity token-expiry approach; operator crashloop (extend the existing Pod-Restart-Loop rule to the operator namespace). `⟶ D-13, R-6`
#### Tracing
N/A — no request-path change; the operator is a controller, not on the request path.

## 5. Technical Design

### Impacted Repositories
- `gitops` — `InfisicalSecret` CRDs, per-app ServiceAccounts + scoped RBAC, operator ArgoCD Application, operator egress NetworkPolicy, retained `*.enc.yaml`/ksops generators, the operator UA `*.enc.yaml`. `⟶ D-3, D-4, D-5, D-6, D-7, D-10, BR-010`
- `webhookr-svc` — `rotate-encryption-key.yml` secret source; README local-dev flow. `⟶ US-3, US-2`
- `webhookr-ingest`, `webhookr-bff`, `webhookr-web` — README local-dev flow; per-app ServiceAccount on the Deployment. `⟶ US-2, D-5`
- `terraform` — write generated app secrets to Infisical instead of hand-copy into SOPS; new Infisical provider/machine-identity resources. `⟶ D-10`
- `webhookr-artifacts` — ADR + Security Matrix update. `⟶ US-10`
- **TODO:** confirm whether a dedicated `infisical` config lives in `gitops` platform vs a new folder (authoring detail; no decision).

### Architecture
Infisical Cloud is the authoritative store. The **Infisical Kubernetes Operator** (pinned version, `creationPolicy: Orphan`) runs in its own namespace, authenticates with a static Universal-Auth identity (SOPS-sealed), and reconciles one `InfisicalSecret` CRD per workload into a native K8s Secret consumed by the app via the unchanged `envFrom.secretRef`. Git holds the CRD **reference**; Infisical holds the **value**. SOPS/ksops remains for bootstrap (the operator's own credential) and DR break-glass, and for all out-of-scope platform secrets. The reversible seam is each app's `kustomization.yaml` (swap `generators:` ksops ⇄ `resources:` CRD); the consumption seam (`envFrom.secretRef`) is stable, so Deployments are otherwise untouched. `⟶ D-2, D-3, D-6, D-7`

### ADR ⟶ D-1, D-2, D-3, D-6, D-7
> Restates the governing Council decisions; introduces none. Full rationale: Council Record §5–§8 and the pending ADR in `webhookr-artifacts`.
#### Decision
Adopt Infisical **Cloud** as the authoritative store for in-scope app secrets, integrated via the **Kubernetes Operator (CRD → native Secret, Orphan, pinned)**, phased, with **per-workload identities + per-app SA/RBAC**, a **static UA operator identity sealed in SOPS**, and **SOPS retained permanently** as bootstrap/DR anchor. `⟶ D-1, D-2, D-3, D-5, D-6, D-7`
#### Alternatives Considered
Self-hosted Infisical (rejected — non-rebuildable SPOF + bootstrap circularity on one node, `X-1`); extend-SOPS with a second age recipient + per-app keys (rejected for the audit/least-privilege ACs — no read-audit, re-centralizes at repo-server, `X-2`); sidecar/CSI injection (rejected — worse availability, sprays token/egress surface, `X-6`); full SOPS retirement (rejected — bootstrap anchor irreducible, `X-7`); K8s-native SA-JWT operator auth (foreclosed by the tailnet-only API server).
#### Tradeoffs
Gains centralized read-audit, revocable per-workload least-privilege, structural cross-service consistency, and collapse of the TF→SOPS double-write. Costs a new external vendor dependency (R-1), a residual static bootstrap secret (R-5), drift invisible to `argocd diff` (R-6), and Infisical on the DR cold-bootstrap path (R-3).
#### Consequences
Every council-decided constraint (Orphan, distinct-name parallel-run, both-boundary least-privilege, retained SOPS, validation gate) is mandatory. An ADR + Security Matrix row are required and supersede audit P0.4. `⟶ US-10`

### Data Model
N/A — no application database schema change. The only "data" changes are Kubernetes objects (Secrets, ServiceAccounts, Roles/RoleBindings, `InfisicalSecret` CRs, NetworkPolicies) and Infisical projects/environments/paths/identities.

### API Contracts
N/A — no application API added or changed. The relevant "contracts" are the `InfisicalSecret` CRD reference and the `envFrom.secretRef` consumption seam, described in Architecture.

#### Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| N/A | N/A | No HTTP API surface in this feature |

#### Request
```json
{}
```
#### Response
```json
{}
```

### Event Contracts
#### Published Events
N/A.
#### Consumed Events
N/A. (The validation round-trip exercises the existing `ingest.events` → `delivery.events` BullMQ flow but adds no new event contract.) `⟶ D-12`

### Message Contracts
```json
{}
```
N/A — no new message contract. The existing sealed queue envelope (`{v,keyVersion,alg,iv,tag,ct}`) is unchanged; `QUEUE_ENCRYPTION_KEYS` continuity is guaranteed structurally via the shared path. `⟶ D-11`

### External Dependencies
- **Infisical Cloud** — new managed SaaS dependency over `443` from the operator namespace (Neon/Redis-Cloud/Grafana-Cloud class). `⟶ D-2, BR-010`
- **Infisical Kubernetes Operator** — new in-cluster controller (pinned version). `⟶ D-3`
- **Infisical CLI** — local dev + optional CI. `⟶ US-2, US-3`
- **`infisical` Terraform provider** — to write TF-generated app secrets. `⟶ D-10`

### Feature Flags
N/A — migration is gated by per-app cutover (envFrom name flip) and validation, not by an application feature flag. The "flag" is operational: the reversible `kustomization.yaml` seam. `⟶ D-7, D-8`

### Migrations
Per-secret/per-app cutover, not a schema migration. Order: local dev → TF-collapse + rotate workflow + operator pilot on one low-risk app → per-app runtime cutover, **svc/ingest last** (shared-path crypto pair, cut over together). Bulk-load is operator-driven from a trusted workstation; SOPS files retained until each gate is recorded. `⟶ D-8, D-6, BR-008`

### Rollback Strategy
Per app: `git revert` the cutover PR — because the ksops generator + `*.enc.yaml` remain in Git and the age key stays in-cluster, ArgoCD `selfHeal` re-renders the SOPS Secret and reverts the `envFrom` name; delete the orphaned `*-infisical` Secret manually. Rollback requires no re-keying and no data recovery. The drill is **executed** (low-blast-radius app first, timed vs RTO), not just documented. `⟶ D-7, D-12, US-9`

## 6. Implementation Strategy

### Recommended Implementation Order
1. **Phase 0 — Local dev** (no runtime blast radius): stand up the Infisical project + `dev`/`prod` environments; per-repo `infisical run` onboarding; keep `.env.example`. `⟶ US-2, US-6, D-8`
2. **Phase 1 — Non-runtime**: collapse the TF→SOPS double-write (`⟶ D-10`); migrate the rotate workflow (`⟶ US-3`); deploy the operator + its SOPS-sealed identity + egress policy (`⟶ D-6, BR-010`); pilot the operator on **one low-risk app** (e.g. `web`) end-to-end including an outage test (Q-3). `⟶ D-8`
3. **Phase 2 — Runtime cutover** (gated on Q-2 + Q-3): per-app, `web → bff → (backup/unleash out of scope) → ingest+svc last`, with per-app SA/RBAC (`⟶ D-5`), parallel-run + validation gate (`⟶ D-12`), then remove ksops generator. `⟶ D-8`
4. **Reconcile**: publish the ADR + Security Matrix row, supersede P0.4. `⟶ US-10`

### Dependencies
- Phase 2 depends on **Q-3** (A-1 validated) and **Q-2** (FinOps sign-off). `⟶ TODO Q-2, Q-3`
- svc/ingest cutover is a single coordinated step (shared `QUEUE_ENCRYPTION_KEYS`). `⟶ D-11`
- Operator identity (D-6) must exist before any `InfisicalSecret` reconciles. `⟶ D-6`

### Breaking Changes
None to application behavior or contracts (`envFrom.secretRef` unchanged). `⟶ D-3`

### Backward Compatibility
Full during migration: both the ksops Secret and the Infisical-managed Secret exist in parallel until the `envFrom` flip; SOPS retained. `⟶ D-4, D-7`

### Rollout Strategy
Per-app blast-radius control inside the single prod env; parallel-run + validation gate + soak before removing the old copy; svc/ingest last. `⟶ D-8, D-12`

## 7. Task Split

### Infrastructure
**Repositories:** `gitops`, `terraform`
**Tasks:**
- [ ] Provision Infisical Cloud project + `dev`/`prod` environments + secret paths (per-app + shared). `⟶ D-8, D-11`
- [ ] Create one read-only machine identity per workload + the operator identity. `⟶ D-5, D-6`
- [ ] Seal the operator UA client secret as `*.enc.yaml` (ksops). `⟶ D-6`
- [ ] Deploy the Infisical Operator (pinned) as an ArgoCD Application + PDB/limits/probes. `⟶ D-3`
- [ ] Add the operator-namespace egress NetworkPolicy (DNS + 443, default-deny last). `⟶ BR-010`
- [ ] Per-app `InfisicalSecret` CRD (Orphan, distinct managed-Secret name), excluded from prune. `⟶ D-3, D-4`
- [ ] Per-app ServiceAccount + `resourceNames`-scoped RBAC + `automountServiceAccountToken:false`. `⟶ D-5`
- [ ] Retain `*.enc.yaml` + ksops generators until each gate recorded; remove generator per-app post-validation. `⟶ D-7, BR-008`
- [ ] Terraform: write `INTERNAL_API_TOKEN`/`redis_url`/`database_url` to Infisical; retire the SOPS hand-copy. `⟶ D-10`
- [ ] Operator reconcile/staleness alerts in `grafana-alerts.md`. `⟶ D-13`

### Backend
**Repositories:** `webhookr-svc`, `webhookr-ingest`, `webhookr-bff`, `webhookr-web`
**Tasks:**
- [ ] None to app code (verified no-code-change). Confirm each Deployment gets its per-app ServiceAccount. `⟶ D-3, D-5`
- [ ] `Assumption:` optionally harden BFF `?? ''` fallbacks to fail-fast (non-blocking, out of this constraint).

### API
**Repositories:** N/A
**Tasks:**
- [ ] N/A — no API surface. `⟶ D-3`

### Frontend
**Repositories:** N/A
**Tasks:**
- [ ] N/A — no UI. Local-dev flow for `webhookr-web` is covered under Documentation. `⟶ US-2`

### Documentation
**Repositories:** `webhookr-svc`, `webhookr-ingest`, `webhookr-bff`, `webhookr-web`, `gitops`, `webhookr-artifacts`
**Tasks:**
- [ ] Replace `cp .env.example .env` with the `infisical run` flow in each README. `⟶ US-2`
- [ ] DR runbook: document the operator identity as a recovery dependency + the SOPS break-glass path. `⟶ R-3, D-7`
- [ ] Rollback runbook + recorded drill. `⟶ US-9, D-12`
- [ ] ADR + Security Matrix row; supersede P0.4. `⟶ US-10`

## 8. Testing Strategy

> Must cover every `US-#`.

### Unit Tests
- Existing crypto/config boot-validation specs remain green under Infisical-sourced env (no code change). `⟶ US-4`

### Integration Tests
- Rendered-Secret **parity diff** (decoded, sorted) Infisical vs ksops, per app. `⟶ US-7, D-12`
- Cross-service byte-identity of `QUEUE_ENCRYPTION_KEYS` / `INTERNAL_API_TOKEN` (`cmp`/hash). `⟶ US-7, D-11`
- Rotate workflow green sourcing from Infisical. `⟶ US-3`

### End-to-End Tests
- **Functional round-trip**: POST a synthetic webhook → sealed envelope on `ingest.events` → `svc.ingest.persisted` increments, **zero `decrypt_failed`** → event decrypts on read-back → delivery byte-intact. Validates QUEUE + EVENT keys, DB, Redis, pepper, internal token at once. `⟶ US-7, R-2`
- **Negative least-privilege**: a workload's identity cannot read another app's path; a `dev` identity cannot read `prod`. `⟶ US-5, US-6, D-5`
- **Outage (Q-3)**: block egress to Infisical, restart/reschedule a pod, assert the managed Secret is unchanged and the pod reaches Running. `⟶ A-1`
- **Rollback drill**: revert one app to ksops, confirm recovery via health + round-trip, timed vs RTO. `⟶ US-9, D-12`

### Manual Validation
- Local dev: boot each service via `infisical run` with **no real `.env`** present; suite green. `⟶ US-2`
- Removal gate checklist recorded (date + operator) before deleting any old copy; escrow verified for `EVENT_ENCRYPTION_KEYS`. `⟶ US-8, BR-008`

## 9. Definition of Ready
- [ ] Council Record present and consumed (this spec). `⟶ D-1..D-13`
- [ ] Infisical Cloud account + project reachable; operator version pinned. `⟶ D-2, D-3`
- [ ] **Q-2** FinOps sign-off recorded (before go-live). `⟶ TODO Q-2`
- [ ] **Q-3** outage behavior validated on a throwaway namespace (before Phase 2). `⟶ TODO Q-3`
- [ ] Per-workload identities + per-app SA/RBAC design reviewed by Security. `⟶ D-5`

## 10. Definition of Done
- [ ] All in-scope app secrets served from Infisical; SOPS retained only for bootstrap/DR + out-of-scope secrets. `⟶ US-1, D-7`
- [ ] Local dev boots with no manual `.env`; `.env.example` retained. `⟶ US-2`
- [ ] Rotate workflow reads from Infisical; old GH secrets removed post-validation. `⟶ US-3`
- [ ] Every app consumes via the operator through unchanged `envFrom`. `⟶ US-4`
- [ ] Per-workload least-privilege enforced at both boundaries; negative tests pass. `⟶ US-5`
- [ ] `dev`/`prod` Infisical environments; no standing `qa`. `⟶ US-6`
- [ ] Each cutover validated by parity diff + functional round-trip (zero `decrypt_failed`) before any removal. `⟶ US-7`
- [ ] SOPS retained through migration; removed per-secret only after the recorded gate; never for bootstrap/DR. `⟶ US-8`
- [ ] Rollback documented **and** drilled with a recorded RTO. `⟶ US-9`
- [ ] ADR published + Security Matrix updated; P0.4 superseded. `⟶ US-10`
- [ ] Operator reconcile/staleness alerts live. `⟶ D-13`

## 11. Estimate

| Size | Selected |
|------|:--------:|
| P    | ☐        |
| M    | ☐        |
| G    | ☐        |
| GG   | ☑        |

Rationale: touches 6 repos, a new external dependency + in-cluster operator, per-workload RBAC net-new, phased multi-app runtime cutover with validation/rollback drills, plus TF and CI changes. `⟶ D-8, D-5`

## 12. References
- Council Record: `../2026-07-14-infisical-secret-platform-council/2026-07-14-council-record-v1.md`
- Superseded finding: `../2026-07-01-webhookr-architecture-audit/2026-07-01-audit-report-v1.md` §P0.4
- Security posture: `../2026-07-05-infra-security-matrix/2026-07-05-adr-infra-security-matrix-v1.md`
- Cross-service key runbook: `gitops/docs/runbooks/queue-encryption-secret.md`
- Alerting runbook: `gitops/docs/runbooks/grafana-alerts.md`

## 13. Future Improvements
- Migrate platform/observability ksops secrets to Infisical once the app plane is validated. `⟶ X-3`
- Reconsider a second age recipient as escrow for the retained SOPS bootstrap secrets. `⟶ X-2`
- Extract the duplicated queue cipher into `lib-node-crypto` (orthogonal, pre-existing).

## 14. Appendix
In-scope secret inventory (from Council evidence): `webhookr-svc` — `DATABASE_URL, REDIS_URL, EVENT_ENCRYPTION_KEYS, QUEUE_ENCRYPTION_KEYS, TOKEN_HASH_PEPPER, INTERNAL_API_TOKEN, UNLEASH_API_TOKEN`; `webhookr-ingest` — `QUEUE_ENCRYPTION_KEYS, INTERNAL_API_TOKEN, REDIS_URL`; `webhookr-bff` — its subset (`REDIS_URL`, JWKS-related, `UNLEASH_API_TOKEN`); `webhookr-web` — its subset. Shared path: `QUEUE_ENCRYPTION_KEYS, INTERNAL_API_TOKEN, REDIS_URL`. `⟶ D-11`
