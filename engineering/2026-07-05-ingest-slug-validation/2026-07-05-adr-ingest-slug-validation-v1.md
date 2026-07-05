---
artifact: adr-ingest-slug-validation
topic: ingest-slug-validation
date: 2026-07-05
version: 1
source: manual
status: Approved
---

# ADR-001: Endpoint slug validation in webhookr-ingest

## Status

Approved (2026-07-05)

## Context

`webhookr-ingest` is the minimal public edge of the product: it receives external
webhooks on `POST /v1/ingest/:endpointSlug`, enqueues them on BullMQ/Redis, and
returns `202 Accepted`. Today the slug is only validated syntactically (loose
regex); existence and active-status checks happen downstream in `webhookr-svc`'s
worker, which **silently drops** events for unknown or inactive slugs after the
`202` was already returned.

Problems with the current behavior:

- `202` is returned for events that will never be processed, breaking the
  semantic contract of "accepted for processing".
- Garbage traffic (random or scanned slugs) is encrypted, enqueued, and consumes
  Redis/BullMQ resources before being discarded.
- The source of truth for endpoints (Postgres via Prisma) lives in
  `webhookr-svc`; ingest was deliberately built without database access.

Constraints: keep ingest a "dumb edge" (no Prisma, no business rules), keep
`webhookr-svc` as the single source of truth, avoid premature cache/sync
complexity, MVP budget and timeline.

## Options considered

### Option 1 — Ingest calls an internal resolver endpoint on webhookr-svc

Ingest calls `GET /internal/endpoints/resolve/:slug` (authenticated with a
service token) before enqueueing. `200 {endpointId}` → enqueue + `202`;
`404` → `404`.

- **Pros:** simplest for MVP; no Prisma duplication; no duplicated state in
  Redis; no sync job; no stale cache; svc remains the single source of truth.
- **Cons:** svc enters the ingest hot path (availability coupling: svc down ⇒
  ingest cannot validate); potential bottleneck at high volume; invalid slugs
  can pressure svc without rate limiting; requires short timeouts, a circuit
  breaker, and metrics.

### Option 2 — Redis registry maintained by svc (push/delete/sync)

svc writes `slug:{slug}` keys to Redis on endpoint create/activate and deletes
them on deactivate/remove; a reconciliation job repairs drift. Ingest only reads
Redis.

- **Pros:** no svc call in the hot path; extremely fast validation; ingest's
  runtime dependency set is actually *smaller* (Redis is already required for
  enqueueing).
- **Cons:** duplicated state between Postgres and Redis; write-path consistency
  (Postgres commit + Redis write can diverge); needs sync/rebuild tooling and
  drift metrics; deactivate/delete paths must be test-covered exhaustively;
  premature complexity for MVP.

### Option 3 — Redis cache-aside with fallback to svc (TTL ~5 min)

Ingest checks Redis first; on miss it calls the svc resolver and caches the
OK/NOK verdict with a short TTL.

- **Pros:** reduces svc calls once warm; NOK caching absorbs repeated invalid
  slugs; no full sync machinery.
- **Cons:** still calls svc on miss; staleness window; must model OK/NOK, TTLs,
  failure behavior, and thundering-herd locking; random-slug attacks still
  generate misses without IP rate limiting; shared-state complexity for a
  benefit an in-process cache already provides at MVP scale.

### Option 4 — Duplicate Prisma/model in ingest

Ingest queries Postgres directly.

- **Pros:** no HTTP hop; Postgres remains the source of truth.
- **Cons:** couples ingest to svc's data model; duplicates Prisma schema/client;
  widens database attack surface; dirty public traffic pressures Postgres;
  destroys the "dumb edge" property that motivated the service split. Rejected.

### Option 5 — Fold ingest back into webhookr-svc

svc exposes the public ingest route and validates locally.

- **Pros:** no inter-service call; no registry/cache/sync problem; fewer
  deployables.
- **Cons:** mixes anonymous, potentially malicious, high-volume webhook traffic
  with the authenticated core API (which also serves the Terraform provider);
  couples scaling and deploy/rollback of edge and core; abuse on ingest can
  degrade the API/BFF. Exposing an authenticated API is not the same risk class
  as exposing an anonymous ingest edge. Rejected.

## Decision

**Option 1, hardened.** Ingest resolves the slug through an authenticated
internal endpoint on svc, with the following design choices:

1. **Strict syntactic pre-check first.** Slugs are generated (never
   user-chosen) as `whr_` + 16 base62 characters (~95 bits of entropy;
   OWASP-comparable to a session token, since a guessed slug grants event
   injection). Ingest rejects anything not matching `^whr_[A-Za-z0-9]{16}$`
   with `404` before any lookup. High entropy also neutralizes slug
   enumeration — `404` responses reveal nothing enumerable.
2. **Resolver contract:** `GET /internal/endpoints/resolve/:slug` returns
   `200 {endpointId}` only when the endpoint exists **and** is active,
   otherwise `404` with an empty body (existence vs. inactive is not leaked).
   The returned `endpointId` is stamped into the queue job envelope so the
   worker does not re-resolve by slug for routing (it still re-validates as
   defense in depth) and slug renames cannot misroute in-flight events.
3. **Auth:** static shared secret (`INTERNAL_API_TOKEN`) generated by
   Terraform (`random_password`), delivered to both services via SOPS-encrypted
   GitOps secrets, compared in constant time. Firebase/Identity Platform has no
   client-credentials flow, and the call never leaves the cluster
   (`webhookr-svc.webhookr:3000`), so a Google service-account OIDC flow was
   judged unnecessary complexity for MVP. Network policy allows ingest→svc
   **only** on this path (it is otherwise denied by design).
4. **In-process micro-cache in ingest** (Map-based, no shared state): OK
   verdicts fresh for 30 s with a 10-minute stale window, NOK verdicts fresh
   for 10 s with no stale use. Webhook traffic concentrates on few slugs, so
   this removes most hot-path calls without any of Option 2/3's consistency
   machinery — and likely removes the need for Option 3 entirely.
5. **Circuit breaker: opossum** (timeout 300 ms on the HTTP call, zero
   retries, opens after ~5 consecutive failures, half-open after 10 s).
   Chosen over hand-rolling because it is the de-facto Node.js breaker,
   dependency-free, event-emitting (drives a Prometheus state gauge), and
   battle-tested; a bespoke breaker would be undertested risk in the hot path.
   A `404` from the resolver is a *value* (NOK), never an error — invalid-slug
   floods must not trip the breaker.
6. **Stale-while-error:** when the breaker is open or the call fails, slugs
   with a stale OK entry (≤10 min) are still accepted — the worker re-validates
   on processing, so the worst case is one discarded job. Slugs with no cached
   verdict get `503` + `Retry-After` (fail closed; webhook providers retry on
   5xx). NOK is never served stale.
7. **Rate limiting layers:** Cloudflare WAF/rate limit at the edge, NestJS
   Throttler per IP in ingest (both pre-existing). Per-slug rate limiting was
   rejected as a primary control: attackers vary slugs freely, and it can
   throttle legitimate bursts to a single endpoint.
8. **Metrics:** resolve latency histogram, resolve errors, cache
   hit(fresh/stale)/miss, breaker state gauge, rejected-slug counters by
   reason, resolver-unavailable counter — provisioned into Grafana dashboards
   and alert rules via Terraform.

## Consequences

**Positive**

- `202` regains its meaning: it is returned only after the slug is validated
  and the event is enqueued.
- Garbage never reaches Redis/BullMQ; svc is shielded by regex pre-check,
  micro-cache, IP throttling, and the breaker.
- Ingest stays free of Prisma and business rules; svc stays the single source
  of truth; no distributed state to reconcile.

**Negative / accepted risks**

- Ingest availability is coupled to svc for *cold* slugs (bounded by the stale
  window for warm ones). This must be monitored before any external
  reliability claims are made.
- The ingest→svc network isolation rule gains a deliberate, documented
  exception.
- The event contract (`ingest-event.contract.ts`) is duplicated across the two
  repos and now carries `endpointId`; both copies must stay byte-identical
  until a shared package is extracted.
- The shared secret requires manual rotation (Terraform taint + SOPS edit).

**Evolution path:** if metrics show the resolver becoming a real bottleneck or
ingest availability becomes contractual, migrate to Option 2 (Redis registry,
push/delete/sync) — the resolver contract and job envelope already carry
`endpointId`, so the switch is confined to ingest's resolver module.
