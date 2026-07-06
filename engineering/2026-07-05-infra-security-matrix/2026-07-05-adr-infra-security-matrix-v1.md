---
artifact: adr-infra-security-matrix
topic: infra-security-matrix
date: 2026-07-05
version: 1
source: manual
status: Approved
---

# ADR-003: Infrastructure Security Matrix & network least-privilege

## Status

Approved (2026-07-05)

## Context

The platform runs on a single Hetzner K3s node with a set of managed SaaS
backing services (Neon Postgres, Redis Cloud, Grafana Cloud, Cloudflare, R2,
GitHub, Google/Firebase). The inbound posture was already strong — SSH and the
Kubernetes API are reachable only over Tailscale, ports 80/443 are firewalled to
Cloudflare edge IPs, admin UIs (ArgoCD, Unleash) are tailnet-only, and the
`webhookr` namespace had default-deny **ingress** NetworkPolicies.

Three gaps motivated this work:

1. **No authoritative Security Matrix.** "Who may talk to whom" lived only in
   scattered manifests and operator memory; there was no single source of truth
   with an owner and justification per communication path.
2. **No egress controls.** Any compromised pod could exfiltrate to any
   destination and move laterally — the largest unmitigated risk.
3. **The ingest Swagger `/docs` was publicly reachable** (no `NODE_ENV` gate,
   and neither the Traefik middleware nor the Cloudflare rule covered `/docs`).

We also found, while designing egress, that `webhookr-svc` delivers webhooks to
**arbitrary customer URLs** (`fetch(destination.url)`), and the `IsPublicHttpsUrl`
validator constrained scheme and host but **not port** — a delivery-egress
decision had to reconcile with the product contract.

## Decision

Adopt an Infrastructure Security Matrix as the operational source of truth
(lives in `gitops/docs/security-matrix.md`, versioned next to the enforcement so
it cannot drift), and harden the network to least privilege:

1. **Deny by default at the network layer, then allowlist.** The `webhookr`
   namespace gets default-deny **egress** in addition to the existing
   default-deny ingress. Each workload receives a self-contained egress policy.
2. **Egress is allowlisted by *port*, not IP.** Neon, Redis Cloud, Grafana
   Cloud, Google/Firebase and R2 are managed services with rotating IPs, so
   pinning IPs would be brittle. The external allow rules use an `ipBlock` of
   `0.0.0.0/0` that **excepts** RFC1918 + CGNAT/tailnet (`100.64.0.0/10`) +
   link-local — so an "external 443" allowance cannot be turned back inward to
   reach in-cluster or tailnet services (an SSRF / lateral-movement guard). DNS
   still resolves the public hostnames via kube-dns.
3. **Webhook delivery is HTTPS/443-only, enforced in two places.** The svc
   egress policy allows only 443 externally, and the `IsPublicHttpsUrl`
   validator now rejects non-443 destination ports — so a destination that would
   silently fail delivery cannot be registered. This makes the network control
   and the product contract agree.
4. **`/docs` is blocked in depth:** app-level `NODE_ENV` gate (ingest, mirroring
   svc), Traefik internal-only middleware, and the Cloudflare block ruleset —
   the last excludes `docs.webhookr.tech` (the Pages docs site shares the zone).
5. **The Security Matrix documents every component** (visibility, allowed
   clients, ports, auth, enforcement mechanism, owner, justification) plus the
   accepted risks below, and links a validation runbook (positive + negative
   probes) and an IP-rotation recovery runbook.

Enforcement was rolled out workload-by-workload (each policy self-contained,
because kube-router isolates a pod to the union of its egress policies the moment
any Egress policy selects it — a blanket policy shipped early would sever DNS),
with `default-deny-egress` applied only after every workload policy was verified.

## Accepted risks (with justification and revisit triggers)

- **Neon Postgres has no network IP allowlist.** The criterion "only
  webhookr-svc may connect to Postgres" is **not achievable at the network layer
  on the current plan**: IP Allow is a paid Neon feature (Scale) and the pinned
  `kislerdm/neon` v0.13 provider exposes no allowlist resource. **Accepted** with
  compensating controls: TLS mandatory (`sslmode=require`), strong SOPS-only
  credentials, 6h PITR, daily logical backups to R2, and runtime access limited
  to the in-cluster svc/migrate/backup/keepalive workloads via egress policy.
  Exception: the `rotate-encryption-key` GitHub workflow receives `DATABASE_URL`
  and runs on GitHub-hosted runners. **Revisit at GA / first paying customer** →
  Neon Scale + IP Allow (Hetzner egress IP only).
- **41641/udp and ICMP are open to the world.** Required for the Tailscale
  WireGuard data plane (auth happens upstream at Tailscale) and diagnostics. No
  service is exposed beyond the tunnel.
- **The BFF is public same-origin, not "Internal."** The original target matrix
  labelled it Internal; in reality the browser calls `app.webhookr.tech/graphql`
  directly (Traefik path route). It is protected by Cloudflare, a forwarded
  Firebase JWT, CORS, throttling, disabled introspection, and query
  depth/complexity limits — corrected in the matrix.
- **`flags.webhookr.tech` is public** but path-restricted to `/api/frontend`
  (frontend flag evaluation only); admin/client Unleash APIs stay in-cluster.

## Alternatives considered

- **Upgrade Neon to Scale now for IP Allow.** Rejected for cost at pre-launch;
  deferred to the GA revisit trigger. Compensating controls hold in the interim.
- **Defer egress policies to a later phase (ingress-only now).** Rejected —
  egress is the single most valuable anti-exfiltration/lateral-movement control;
  the incremental, self-contained rollout made it safe to do now.
- **Allowlist SaaS egress by resolved IP / CIDR.** Rejected — managed endpoints
  rotate IPs; port-scoping with a private-range `except` is both robust and
  sufficient given TLS everywhere.
- **Restrict egress in observability/tailscale/argocd now.** Deferred to Phase 2
  (documented): an argocd egress mistake would wedge GitOps itself; tailscale
  proxies need coordination-plane egress (incl. DERP); the collector needs
  Grafana Cloud, kubelet, and API-server reachability that is fiddly to
  enumerate safely. Observability *ingress* was restricted now.

## Consequences

**Positive**

- A compromised application pod can now reach only its declared dependencies on
  their declared ports; DNS-rebinding SSRF back into the cluster/tailnet is
  blocked; the blast radius of any single workload is bounded.
- Every communication path has an owner and a written justification; drift is
  caught because the operational matrix sits next to the YAML.
- The public attack surface shrank: no Swagger, no `/health` /`/metrics` from the
  internet; delivery cannot target non-443 ports.

**Negative / operational cost**

- Each new workload or external dependency now requires an egress-policy edit
  (and a matrix update) — friction that is the point of least privilege.
- Custom-port HTTPS webhook destinations are rejected (a product constraint,
  acceptable pre-launch; documented).
- The Neon network gap remains until GA; mitigated, not closed.
- kube-router's "first egress policy isolates the pod" semantics is a sharp edge
  for future edits — every workload egress policy must remain self-contained
  (embed its own DNS rule).

The concrete rules, the component/egress tables, the exceptions, and the
validation + recovery runbooks are maintained in
`gitops/docs/security-matrix.md`. The threat model that this posture answers to
is the companion artifact in this folder.
