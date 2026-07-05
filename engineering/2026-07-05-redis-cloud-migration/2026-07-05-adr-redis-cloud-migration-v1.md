---
artifact: adr-redis-cloud-migration
topic: redis-cloud-migration
date: 2026-07-05
version: 1
source: manual
status: Approved
---

# ADR-002: Migrate Redis from in-cluster K3S to Redis Cloud

## Status

Approved (2026-07-05)

## Context

Redis backs the BullMQ queues (`ingest.events`, `delivery.events`) and the
per-IP rate limiter of ingest, svc, and bff. It currently runs as a
single-replica StatefulSet in the K3S cluster on Hetzner (redis:7.4-alpine,
2 Gi local-path PVC, 384 MB maxmemory, noeviction, password auth, no TLS,
no persistence tuning, no backups).

The 2026-07-01 architecture audit flagged this as a P0: **Redis is a single
point of failure with no backup**. The slug-validation ADR (ADR-001) makes the
problem sharper: `202 Accepted` now explicitly means "accepted for
processing", so losing queued events on a Redis crash is a broken promise,
not just an internal risk.

The product is not live; a big-bang cutover with loss of any in-flight queue
jobs is acceptable.

## Options considered

### Option A — Harden the in-cluster StatefulSet

Enable AOF, schedule RDB snapshots to object storage, add memory/disk alerts.

- **Pros:** no new vendor, no WAN latency, no new spend.
- **Cons:** still one replica on one node on one Hetzner server; node loss
  still loses the queue between snapshots; HA (Sentinel/replica) in a
  single-node K3S is theater; backup/restore tooling becomes our operational
  burden — poor fit for a lean-budget MVP team.

### Option B — Managed Redis (Redis Cloud) ✅

Move to a Redis Cloud database: HA single-zone (99.9% uptime SLA), AOF
`appendfsync everysec`, `noeviction` (required by BullMQ), TLS.

- **Pros:** removes the SPOF and the backup burden in one move; durability
  (AOF 1 s) restores the `202` contract; TLS in transit (queue payloads are
  additionally sealed with AES-256-GCM at the application layer); provider
  handles failover, patching, monitoring of the datastore itself.
- **Cons:** enqueue latency now includes WAN RTT from Hetzner (nbg1) to the
  Redis Cloud region — must be validated against ingest p99; new vendor and
  monthly cost; egress traffic leaves the cluster (mitigated by TLS +
  app-layer encryption); Essentials tier exposes no Prometheus endpoint, so
  datastore-level metrics rely on the vendor console (queue-level health is
  still observable via app metrics).

## Decision

**Option B.** The database was provisioned manually (HA single zone, 99.9%,
AOF everysec, noeviction, TLS); Terraform imports it (`RedisLabs/rediscloud`
provider, `import {}` blocks) so it is state-managed from day one — config
drift, plan changes, and the connection string all flow through code review.

Implementation rules:

1. **Single `REDIS_URL` env var, `rediss://` scheme, everywhere.** All three
   consumers (ingest, svc, bff) replace `REDIS_HOST`/`REDIS_PORT`/
   `REDIS_PASSWORD` with one URL parsed by a small per-repo
   `redis.config.ts`. TLS is implied by the scheme; SNI `servername` is set
   explicitly (Redis Cloud endpoints are SNI-routed; ioredis does not set it
   automatically). Local dev keeps `redis://` against docker-compose.
2. **Code before config:** the three apps ship `REDIS_URL` support first
   (inert until the env var exists), then GitOps flips the secrets and
   deployments, then the in-cluster Redis is deleted in a separate PR
   (ArgoCD syncs Applications in arbitrary order; flip and prune must not
   race). StatefulSet PVCs are removed manually post-prune.
3. **Big-bang cutover:** in-flight jobs in the old queue are abandoned —
   accepted, the product is not live.
4. The Terraform output composes the sensitive
   `rediss://default:<password>@<host>:<port>` URL; the operator copies it
   into the SOPS-encrypted GitOps secrets (Terraform cannot write SOPS
   files).

## Consequences

**Positive**

- Closes the audit's Redis P0: HA + AOF-per-second durability + vendor
  backups replace a single unbacked replica.
- `202 Accepted` is now backed by a durable queue.
- Redis config becomes reviewable infrastructure code instead of a
  hand-managed StatefulSet.
- The cluster sheds a stateful workload (2 Gi PVC, memory pressure, probe
  tuning).

**Negative / accepted risks**

- Enqueue latency includes WAN RTT; region proximity to Hetzner nbg1 must be
  verified and ingest p99 watched after cutover.
- Vendor dependency and recurring cost (accepted within the lean-budget
  constraint as the price of durability).
- Datastore metrics live in the vendor console (Essentials tier); alerting is
  built on application-side metrics (enqueue failures, health checks)
  instead.
- Network policies are ingress-only today, so no policy change is needed for
  egress — but this also means egress is unrestricted; tightening egress is a
  separate future concern.
