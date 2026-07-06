---
artifact: threat-model-infra
topic: infra-security-matrix
date: 2026-07-05
version: 1
source: manual
status: Approved
---

# Threat model — Webhookr infrastructure

Companion to ADR-003 (infra security matrix). Scope: the network and
infrastructure trust boundaries of the platform — not application-logic bugs.
STRIDE-lite: for each threat we name the surface, the attacker, and the
control(s) that mitigate it (mapped to the operational matrix in
`gitops/docs/security-matrix.md`).

## Assets

- Webhook event data (payloads, headers) — sealed AES-256-GCM in transit through
  the queue; stored encrypted at rest.
- Customer credentials & secrets (API tokens, signing secrets, DB/Redis creds).
- The Postgres database (source of truth) and the Redis Cloud queue.
- The single Hetzner node / cluster control plane.
- Availability of the ingest edge and delivery pipeline.

## Actors / entry surfaces

1. **Anonymous internet → ingest** (`wbhkr.tech`, no auth by design).
2. **Authenticated internet → svc API** (`api.webhookr.tech`, PAT or Firebase
   JWT) and **→ web/bff** (`app.webhookr.tech`, Firebase session).
3. **Admin** (operator) over Tailscale (SSH, kubectl, ArgoCD, Unleash).
4. **Compromised workload** (a pod running attacker-controlled code after an app
   RCE or malicious dependency) — the insider-in-the-cluster case.
5. **Managed SaaS** and the CI/CD supply chain.

## Threats & mitigations

| # | Threat (STRIDE) | Surface / actor | Mitigation |
|---|---|---|---|
| T1 | **SSRF via webhook delivery** (Info-disclosure / Elevation): svc fetches attacker-chosen URLs to reach cloud metadata (169.254.169.254), internal services, or non-standard ports | Authenticated user registering a destination | `IsPublicHttpsUrl` blocks non-HTTPS, private/loopback/link-local hosts, IPv4-mapped IPv6, and now **non-443 ports**; svc egress `ipBlock` excepts RFC1918+CGNAT+link-local so even a bypass cannot reach inward; delivery is 443-only |
| T2 | **Data exfiltration from a compromised pod** (Info-disclosure): stolen DB/queue data shipped to an attacker endpoint | Actor 4 | default-deny egress; each workload may only reach its declared dependencies on declared ports; arbitrary outbound (e.g. attacker C2 on 443 to a random host) is *allowed only where the workload legitimately needs 443* — svc/web/bff — and blocked entirely for ingest/unleash/db/cronjobs beyond their needs |
| T3 | **Lateral movement between workloads** (Elevation): pivot from a low-value pod to the database or queue | Actor 4 | default-deny ingress *and* egress; e.g. ingest cannot reach unleash-db or unleash; unleash-db has zero egress; only svc/backup/keepalive may reach Neon:5432; the `100.64.0.0/10` except blocks reaching tailnet hosts |
| T4 | **Reconnaissance via ops endpoints** (Info-disclosure): scraping `/metrics`, `/health`, `/docs` (Swagger schema) from the internet | Anonymous internet | Blocked in depth: Cloudflare WAF rule + Traefik internal-only middleware; ingest Swagger additionally gated by `NODE_ENV`; `docs.webhookr.tech` excluded from the block |
| T5 | **Origin bypass of Cloudflare** (Spoofing/Tampering): hitting the node IP directly to skip WAF/rate-limit/TLS | Internet | Hetzner firewall accepts 80/443 only from Cloudflare edge CIDRs (dynamic); Full-strict TLS with origin CA cert |
| T6 | **Admin plane exposure** (Elevation): SSH / K8s API / ArgoCD reachable from the internet | Internet | 22 and 6443 not opened publicly; API server TLS SAN is the tailnet IP; ArgoCD/Unleash admin only via Tailscale Ingress; auth upstream at Tailscale |
| T7 | **Slug enumeration / abuse at ingest** (DoS/Info): guessing endpoint slugs, flooding the anonymous edge | Anonymous internet | High-entropy `whr_` slugs; svc-side resolver returns uniform 404 (no existence oracle); Cloudflare + app throttling; queue payloads sealed |
| T8 | **Redis Cloud / Postgres direct attack** (Tampering): connecting to the managed datastores from anywhere | Internet | Redis Cloud `source_ips` = Hetzner /32 (rejects all other origins); Postgres — *accepted risk*, mitigated by TLS + strong creds + backups (see ADR) |
| T9 | **Secret disclosure at rest** (Info-disclosure): reading credentials from the repo or cluster | Repo/cluster reader | SOPS/age encryption for all secrets in git; mounted only into the workloads that need them; K3s secrets-encryption at rest |
| T10 | **Supply-chain / CI compromise** (Tampering/Elevation): CI reaching production data or pushing malicious images | CI actor | CI has no production DB/Redis access (integration tests use local containers); GHCR + fine-grained gitops PAT only; branch protection (PR-only). Exception noted: rotate-encryption-key workflow holds DATABASE_URL |
| T11 | **Telemetry pipeline abuse** (Tampering/DoS): injecting/among spoofing OTLP into the collector | In-cluster | observability default-deny ingress; only webhookr + kube-system may send OTLP (4317/4318); collector egress to Grafana Cloud is token-authenticated |

## Residual risks (accepted / deferred)

- **Neon has no network allowlist** (T8, Postgres half) — accepted until GA; see
  ADR compensating controls.
- **Egress from observability/tailscale/argocd is unrestricted** — Phase 2; these
  are single-purpose namespaces with no untrusted workloads. A compromise of the
  otel-collector could exfiltrate telemetry over its Grafana Cloud egress.
- **41641/ICMP world-open** (T5/T6 adjacent) — inherent to the Tailscale data
  plane; authentication is upstream.
- **Workloads that legitimately need external 443** (svc/web/bff) could, if
  compromised, use that 443 allowance for C2 to an arbitrary host — port-scoping
  cannot prevent this without an egress proxy / FQDN policy (out of scope; a
  future hardening if a Cilium-class CNI is adopted).

## Validation

Controls are exercised by `gitops/docs/runbooks/security-validation.md`
(positive: authorized paths succeed; negative: unlabeled pod denied DNS+egress,
cross-workload denied, ops endpoints 403, datastores reject foreign IPs). Re-run
on any change to firewall, NetworkPolicies, IngressRoutes, or Cloudflare rules.
