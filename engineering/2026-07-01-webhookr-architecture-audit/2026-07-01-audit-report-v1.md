---
artifact: audit-report
topic: webhookr-architecture-audit
date: 2026-07-01
version: 1
source: manual
status: Reviewed
---

# Webhookr — Prioritized Audit Report
### Architectural risks · Performance bottlenecks · Security issues · Technical debt

*Date: 2026-07-01 · Scope: webhookr-{svc,ingest,bff,web,mobile}, terraform, gitops, github-workflows, terraform-provider-webhookr*

## Context

Read-only audit of the entire webhookr ecosystem, executed by 3 exploration agents (backend, infra/CD, clients) with manual verification of the highest-severity findings. Two findings the agents reported as "critical" were **refuted by direct verification**: (1) the `.env` files are **not tracked in git** in any repo (gitignored, no history in `git log -- .env`); (2) the gitops `kustomization.yaml` files **pin `newTag` to commit SHAs**, not `main`. The report below is already calibrated for this.

**As-is architecture:** ingest (public, 2 replicas) → BullMQ/Redis → svc (persists AES-256-GCM-encrypted event to Neon Postgres, dispatches deliveries) → HTTPS destinations. GraphQL BFF in front of svc; Next.js 16 web with Auth.js→Firebase bridge; everything on a single Hetzner K3s node (nbg1), Cloudflare at the edge, ArgoCD + gitops, Grafana Cloud.

**Overall assessment:** solid foundation — global auth enforced, tenant isolation verified, complete SSRF protection (validation + delivery-time DNS check), envelope encryption with key rotation, versioned message contracts, zero TODO/FIXME, pinned dependencies. The real risks are concentrated in **infra resilience (SPOFs)**, the **deploy chain (org-wide PAT + auto-merge)**, and **operational gaps** (alerts not provisioned, no data retention).

---

## P0 — Act now (incident or compromise risk)

### P0.1 · Supply chain: org-wide `GITOPS_DEPLOY_PAT` + auto-merged deploy PRs — SECURITY
- `terraform/github_secrets.tf:1-14` distributes the PAT (Contents+PR write on gitops) as an **org secret to all private repos**; `github-workflows/.github/workflows/build-deploy.yml:104` auto-merges (`gh pr merge --squash`) into gitops.
- Chain: compromised dependency in **any** repo → CI uses the PAT → push+merge into gitops → ArgoCD deploys arbitrary code to prod, with no human review.
- **Fix:** restrict the secret to the 4 active service repos (`visibility=selected`); replace the PAT with a GitHub App issuing ephemeral tokens; require approval (environment protection) for prod gitops merges. Additionally, escape `IMAGE`/`SHA` in the `yq` step (build-deploy.yml:86-87).

### P0.2 · Webhook loss: single-replica Redis on single-node local storage — ARCHITECTURE/RELIABILITY
- `gitops/webhookr/redis/statefulset.yaml:10` (`replicas: 1`, local-path PVC) on a **single K3s node** (`terraform/modules/hetzner/main.tf:27-50`). The BullMQ queues (`ingest.events`, `delivery.events`) live only there.
- Node failure = loss of all queued events (unbounded RPO) + the whole platform offline. Postgres is backed up to R2; **Redis is not**.
- Webhooks are fire-and-forget from the sender's side: a lost event is unrecoverable — this hits the core of the product.
- **Fix (in cost order):** enable AOF `everysec` + ship the dump to R2; 2+ replicas for bff/svc/web + PDBs (`deployment.yaml:9` in each); make an explicit decision (ADR) to accept or not the single node — if accepted, mitigate with P0.3.

### P0.3 · Invisible outage: alerts not provisioned and no external uptime check — OBSERVABILITY
- Alerts are only **documented** in `gitops/docs/runbooks/grafana-alerts.md:133-154`, not provisioned via Terraform (no `grafana_rule_group`, no automated contact points). No synthetic/uptime check on the public ingest endpoint.
- Ingest down = webhooks silently lost for hours with nobody knowing. Combines dangerously with P0.2.
- **Fix:** provision rule groups + a contact point (email/Slack) via TF in `terraform/grafana_dashboards.tf`; add a Grafana Cloud synthetic check on the ingest and web health endpoints.

### P0.4 · Secret hygiene: production tokens in local plaintext + Firebase key in TF state — SECURITY
- **Correction of the agents' finding: none of this is committed to git.** However: the local `terraform/.env` holds a real classic PAT, a fine-grained PAT, and Cloudflare/Hetzner/Tailscale/Neon tokens in plaintext; `terraform/modules/firebase-auth/outputs.tf:16-20` materializes `admin_sdk_private_key` in the GCS state (recoverable if the bucket leaks) — with no evidence this key is used (svc/bff only validate JWTs via JWKS).
- **Fix:** remove the SA key from the firebase-auth module if unused; move operational secrets to a secret manager (SOPS+age is already used in gitops — extend it to the TF flow); rotate as a precaution the tokens that have already circulated through terminals/transcripts.

---

## P1 — High priority (next cycle)

### P1.1 · Message contract duplicated between ingest and svc — ARCHITECTURE/DEBT
`IngestEventV1` is copied in `webhookr-ingest/src/contract/ingest-event.contract.ts` and `webhookr-svc/src/contract/ingest-event.contract.ts` ("keep both copies in sync"). Divergence = silent event loss. **Fix:** a `@webhookr/contracts` package published to the GHCR npm registry, or a CI contract test that diffs the two files (cheap, immediate).

### P1.2 · Ingest rate limit is per IP, no per-endpoint/tenant quota — SECURITY/PERFORMANCE
100 req/min per IP (`webhookr-ingest/src/config/throttle.config.ts`). No per-endpoint quota: a leaked slug allows unlimited distributed flooding (every event is enqueued, encrypted, and persisted); corporate NAT can block legitimate traffic. Also a prerequisite for plan-based pricing (Free/Pro/Team). **Fix:** Redis counter per endpointSlug with per-plan limits; metric for dropped events.

### P1.3 · No HMAC signature on deliveries — SECURITY/PRODUCT
Destinations have no way to authenticate that the POST came from webhookr (`x-webhookr-event-id` is spoofable). Industry standard (Stripe/GitHub/Svix). **Fix:** per-destination secret + `x-webhookr-signature` header (HMAC-SHA256 with timestamp).

### P1.4 · BFF→svc without timeout/retry/circuit breaker — ARCHITECTURE
`webhookr-bff/src/webhookr-client/webhookr-client.service.ts:12,44-58`: raw fetch to `WEBHOOKR_SERVICE_URL`. A slow svc = BFF requests stuck, cascading failure. **Fix:** AbortSignal timeout (~10s), retry with backoff for idempotent GETs.

### P1.5 · No event retention policy — COMPLIANCE/COST
Encrypted events accumulate forever in Neon (no TTL in `webhookr-svc/prisma/schema.prisma`). Growing cost + GDPR exposure (third-party payloads retained indefinitely). Also a pricing lever (retention per plan). **Fix:** age-based purge job (e.g., 7d Free / 30d Pro / 90d Team) + deletion endpoint.

### P1.6 · Deployments without an explicit strategy and manual rollback — RELIABILITY
No `strategy:` field in the deployments; rollback is a manual `argocd app rollback`. **Fix:** `RollingUpdate maxSurge:1 maxUnavailable:0` on critical services; rollback runbook in gitops.

---

## P2 — Medium priority

| # | Finding | Evidence | Category |
|---|---------|----------|----------|
| P2.1 | Deliveries per event unpaginated (response grows unbounded with replays) | `webhookr-svc/src/delivery/delivery.service.ts:60-70` | Performance |
| P2.2 | Unlimited replays create Delivery records with no constraint/daily cap | `delivery.service.ts:90-125`; no unique (eventId, destinationId, attempt) | Performance/abuse |
| P2.3 | Orphan event if delivery enqueue fails after `createEvent` (no outbox/transaction) | `webhookr-svc/src/ingest/ingest.service.ts:72-91` | Architecture |
| P2.4 | No operational DLQ: failed jobs sit in BullMQ (KEEP_FAILED_JOBS=1000) with no metric/alert; drops for unknown endpoints only log a warn | `queue.module.ts:8,14`; `ingest.service.ts:58` | Observability |
| P2.5 | Sensitive-header lists diverge between ingest (9) and delivery (6) — a new signature header could leak on one path | `ingest.service.ts:16-29` vs `delivery.service.ts:19-26` | Security |
| P2.6 | No egress NetworkPolicies — a compromised pod can exfiltrate to any IP | `gitops/webhookr/network-policies/network-policy.yaml` | Security |
| P2.7 | Neon backup: 14-day retention, no CronJob failure alert, no restore test | `gitops/webhookr/backup/cronjob.yaml:85` | Reliability |
| P2.8 | Web: refresh-token error does not redirect to login (blank screen/stale data) | `webhookr-web/src/shared/auth/auth.ts:28-61` + `use-current-user.ts` | UX/Auth |
| P2.9 | Web: test coverage 19% (44/185 files), below backend standard (54–72%); auth flows under-covered | `webhookr-web/src/shared/auth/*` | Debt |
| P2.10 | Worker concurrency (INGEST/DELIVERY_CONCURRENCY) can exceed the default Prisma pool (~10 connections) | `webhookr-svc` has no explicit pool config | Performance |

## P3 — Low priority / record

- Duplicated configs (cors/throttle/swagger/logger) copy-pasted across the 3 services — config drift; extract to a shared package together with P1.1.
- BFF without DataLoader — N+1 HTTP calls on nested GraphQL queries (low impact at current volume).
- Grafana dashboards manually imported into state, JSON not versioned (`terraform/grafana_dashboards.tf:1-7`).
- TF providers loosely pinned (`hcloud ~> 1.0`, `neon ~> 0.13` pre-1.0).
- Payload rendering: React escapes by default (no direct XSS), but control characters could break layout; no `dangerouslySetInnerHTML` anywhere (good).
- Hardcoded `window.location.href` instead of `useRouter` (`use-auth-actions.ts:47,64`).
- Mobile: token hydration race on startup (`webhookr-mobile/src/core/auth/auth-context.tsx:50-80`).
- No ADRs (BullMQ choice, encryption strategy, retry policy).
- Agent findings **refuted and discarded**: ".env committed to git" (false — gitignored, no history), "images tagged `main`" (false — SHA-pinned), "CSRF on GraphQL mutations" (mitigated — auth is a Bearer header, not a cookie; browsers do not send the header cross-origin).

---

## Suggested remediation plan

1. **Week 1 (P0):** restrict the org secret + environment protection on gitops; Redis AOF+backup; provision alerts/uptime checks via TF; remove the SA key from the firebase-auth module.
2. **Weeks 2–3 (P1):** contract test in CI; per-endpoint quota; delivery HMAC; timeout on the BFF client; retention job.
3. **Backlog (P2/P3):** table items as capacity allows, prioritizing P2.1–P2.4 before the GA launch (Product Hunt).

## Verification

- P0.1: run `gh api /orgs/{org}/actions/secrets/GITOPS_DEPLOY_PAT/repositories` and confirm the restricted list; attempt a gitops merge without approval and confirm it is blocked.
- P0.2/P0.3: `kubectl delete pod` on redis/svc and confirm recovery + alert firing; simulate ingest down and confirm the synthetic check notifies.
- P1.2/P1.3: e2e tests in webhookr-svc (quota returns 429; destination validates the HMAC).
