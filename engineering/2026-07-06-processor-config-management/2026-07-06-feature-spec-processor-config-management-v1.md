---
artifact: feature-spec-processor-config-management
topic: processor-config-management
date: 2026-07-06
version: 1
source: /upstream
status: Draft
title: Processor Config Management (write surface)
owner: thiagojv
created: 2026-07-06
updated: 2026-07-06
estimate: G
priority: Medium
labels: [webhookr-svc, webhookr-bff, webhookr-web, backend, api, frontend, processors, follow-up]
---

# Feature Specification

This document describes everything required to implement a feature, from business motivation to production rollout.

> **Follow-up story.** The [Event Processor Engine spec v2](../2026-07-06-event-processor-engine/2026-07-06-feature-spec-event-processor-engine-v2.md)
> is the **source of truth** for the enablement model. That story shipped the `ProjectProcessorConfig` /
> `EndpointProcessorConfig` tables and the four-layer **resolver read-path** only; the **write surface** was
> explicitly deferred (engine spec §2 Out of Scope, §13). This story delivers that write surface — the ability
> for a customer to enable/disable a **PRODUCT** processor per project and per endpoint — without changing any
> engine, resolver, or delivery behavior.

## 1. Context

### Background
`webhookr-svc` runs an ordered pre-delivery processor pipeline. Whether a given processor runs for a given event
is decided by the **four-layer enablement model** (engine spec BR-003):

```
global platform flag            (webhookr.processors.enabled)
AND per-processor platform flag (webhookr.processors.<name>.enabled)
AND project config              (ProjectProcessorConfig,  PRODUCT only)
AND endpoint config             (EndpointProcessorConfig, PRODUCT only)   — endpoint overrides project
```

The two platform layers are operator-controlled (Unleash flags). The two customer layers live in
`ProjectProcessorConfig` / `EndpointProcessorConfig` and are read every event by
`ProcessorConfigRepository.findEnablement` → `PipelineResolver.resolve`
([`webhookr-svc/src/event-processing/`](../../../webhookr-svc/src/event-processing/)). Today those two tables
have **no writer**: no API, no BFF field, no UI. They start empty, so the resolver always falls back to the
PRODUCT default (`disabled`, BR-005). There is deliberately nothing to configure yet because **no PRODUCT
processor exists** — only the PLATFORM `noop` processor is registered.

### Problem Statement
When the first PRODUCT processor ships (validation, filtering, transformation, deduplication, routing,
enrichment), a customer must be able to turn it on/off for a project and override it per endpoint. There is
currently no way to write `ProjectProcessorConfig` / `EndpointProcessorConfig`, and no way for a UI to discover
which processors are configurable. Without this surface the four-layer model is inert past the platform flags,
and every PRODUCT processor would be permanently stuck at its default.

### Objective
After this feature a customer can, through the web app (and the REST/GraphQL surfaces behind it):
- see the list of **configurable (PRODUCT, generally-available)** processors and their current effective state
  for a project and for an endpoint;
- enable or disable a processor at the **project** level;
- enable, disable, or **inherit** (fall back to project) a processor at the **endpoint** level;
each write persists a `ProjectProcessorConfig` / `EndpointProcessorConfig` row that the existing resolver already
consumes on the next ingested event — with **zero change** to the engine, resolver, or delivery path.

### Business Value
- Activates the customer-facing half of the four-layer model the engine story built read-only.
- Delivers the "Product control" business value promised by the engine spec (customers decide which processors
  run; new processors stay inert until opted in — BR-005), a differentiator for a developer-first webhook platform.
- Hard prerequisite for shipping any real PRODUCT processor with customer opt-in.

### Success Metrics
- 100% of writes land a correctly-scoped, ownership-checked row; the **next** event for that endpoint resolves
  using the new value (asserted by integration test: write config → ingest → assert `EventProcessorRun` reflects
  the new decision).
- 0 successful writes for **PLATFORM** processors or **unknown** processor names (all rejected — BR-C2/BR-C3).
- Endpoint override precedence holds in 100% of resolver reads (endpoint > project > default), unchanged from the
  engine story's matrix tests.
- No regression to ingest hot-path latency: the write surface adds no per-event work; the resolver read is
  unchanged (engine spec NFR Performance still holds).

## 2. Scope

### In Scope
- **`webhookr-svc`**: a `processor-config` capability exposing
  - a **catalog** read: list registered **configurable** processors (PRODUCT + generally-available), with
    category and platform availability;
  - **project-scoped** config read + upsert + reset (`ProjectProcessorConfig.enabled`);
  - **endpoint-scoped** config read + upsert + reset (`EndpointProcessorConfig.enabled`);
  - `JwtAuthGuard` + project/endpoint ownership on every route (mirrors `EventService.findOne` /
    `EndpointService.findOne`);
  - validation: processor must be **registered** and **PRODUCT** category; reject unknown/PLATFORM names;
  - the `config Json` column stays **reserved** — writes touch `enabled` only.
- **`webhookr-bff`**: GraphQL queries + mutations proxying the above (catalog, project config, endpoint config),
  scoped and forwarded through `WebhookrClientService` (JWT passthrough, ownership enforced in svc).
- **`webhookr-web`**: a "Processors" settings surface on the project and endpoint screens — a list of
  configurable processors with a toggle at project level and a tri-state (On / Off / Inherit) at endpoint level,
  showing the effective state and its source.
- Observability for writes (structured logs + a counter), and documentation/ADR + artifact reconciliation.

### Out of Scope
- Any **real** PRODUCT processor implementation (validation/filtering/…); this surface configures processors, it
  doesn't create them. With none registered today the catalog is **empty** until the first ships (see Risks R-C1).
- Interpreting the per-processor **`config Json`** column (engine OQ5) — writes set `enabled` only; the reserved
  column is untouched.
- Changing the **platform** layers (global flag or per-processor `webhookr.processors.<name>.enabled`) — those
  remain operator-controlled in Unleash. This surface writes only the two **customer** layers.
- Any change to `EventProcessorEngine`, `PipelineResolver`, `ProcessorConfigRepository.findEnablement`, ingest,
  delivery, or the replay gate (BR-015). This story is **additive** and read-compatible with them.
- The processor-execution **timeline** read API/UI (separate follow-up).
- A retroactive "re-process already-ingested events with the new config" action (engine spec BR-016: `STOPPED`/
  `FAILED` are terminal; replay bypasses the engine). Config changes are **forward-only** (BR-C7).
- A dedicated persisted **audit-log table** (no such infra exists in the repo today). Writes are audited via
  structured logs + metric only; a durable audit trail is a labeled Future Improvement.

### Assumptions
- **Assumption:** The resolver has **no config cache** today (engine spec NFR Performance says a short-TTL cache
  is introduced *only if* the per-event read becomes hot; it has not). Therefore writes need **no cache
  invalidation** in v1. If that cache is ever added, its invalidation-on-write hook is specified here as a
  forward requirement (BR-C8) but implemented with the cache, not now.
- **Assumption:** Ownership for endpoint config is exactly the engine/`EventService` model: project belongs to the
  authenticated user, endpoint belongs to the project. No workspace/role/RBAC layer exists in `webhookr-svc`
  today (single-owner project model), so "owner" = the user who owns the project.
- **Assumption:** The per-processor platform flag `webhookr.processors.<name>.enabled` governs a PRODUCT
  processor's **availability / effective state**, **not** its catalog visibility. The catalog lists **every**
  registered PRODUCT processor; a flag that is off surfaces as `platformAvailable = false` /
  `source = platform_disabled` (see BR-C4).
- **Assumption:** Endpoint config is meaningfully tri-state (On / Off / **Inherit**) because the resolver treats a
  missing endpoint row as "fall back to project" (`endpointValue ?? projectValue ?? false`). "Inherit" is
  represented by the **absence** of a row, so `reset` = delete row.
- **Assumption:** Concurrency is last-write-wins on the unique `(scopeId, processorName)` row; a boolean toggle
  needs no optimistic-lock/version field. `updatedAt` records recency.

### Risks
- **R-C1 — Empty catalog at launch.** No PRODUCT processor is registered, so the catalog is legitimately empty and
  the UI shows an empty state. Mitigation: ship with an explicit, tested empty state; validate end-to-end with a
  **stub** PRODUCT processor in integration tests (as the engine story already does).
- **R-C2 — Configuring a PLATFORM/unknown/platform-off processor.** A write for a PLATFORM, unknown, or
  platform-disabled processor would create a dormant, misleading row. Mitigation: validate against the registry +
  category + platform flag on every write; reject (BR-C2/C3, EF-C6).
- **R-C3 — Write/read model drift.** The write surface encodes the same precedence the resolver applies; if the
  read model shown to the user diverges from `PipelineResolver`, users see a wrong "effective" state. Mitigation:
  derive the displayed effective state from the **same** resolver inputs (registry + flags + config), reusing
  `PipelineResolver` rather than re-implementing precedence in the config service.
- **R-C4 — Surprising "no effect" toggles.** A customer tries to enable a processor whose per-processor
  **platform** flag is off → effective would stay `PLATFORM_DISABLED`. Mitigation: the read model surfaces
  `platformAvailable = false` / `source = platform_disabled` / `effectiveEnabled = false` so the UI renders the row
  as read-only "temporarily unavailable by the platform," **and** the `PUT` write is rejected (EF-C6) so no
  no-effect toggle is ever persisted. `DELETE`/reset stays allowed (removing a row is always safe).
- **R-C5 — Ownership bypass.** A missing ownership check would let a user write another tenant's config.
  Mitigation: every route runs the same `projectService.findOne` / `endpointService.findOne` gate as
  `EventService`, covered by unit + e2e tests including a cross-tenant 404 case.

### Open Questions
- **OQ-C1 — Resolved.** Endpoint scope is **tri-state** (On / Off / Inherit); Inherit = no row = `reset`. Project
  scope is **bi-state persisted** (On / Off) with "no row" meaning default-disabled; project `reset` deletes the
  row (returns to default). Matches the resolver's `?? ` fallback exactly.
- **OQ-C2 — Resolved.** REST verb for a set is **PUT** (idempotent upsert of `enabled`) on
  `.../processor-configs/:processorName`; **DELETE** resets (idempotent). Chosen over POST/PATCH because the
  operation is an idempotent set of a single boolean keyed by `(scope, processorName)`.
- **OQ-C3 — Resolved (revised).** Catalog **lists every** registered PRODUCT processor regardless of the
  per-processor platform flag. The flag governs **availability / effective state**, not visibility: a
  platform-off processor still appears, with `platformAvailable = false` / `source = platform_disabled`, rendered
  read-only, and its `PUT` write rejected (EF-C6). See ADR “Catalog visibility.”
- **OQ-C4 — Deferred (does not block).** A persisted audit-log table for config changes. v1 uses logs + metric;
  a durable table is a Future Improvement, aligned with whenever platform-wide audit is introduced.
- **OQ-C5 — Deferred (does not block).** Bulk write (set many processors in one call). v1 is one processor per
  request; a bulk endpoint can be added later without breaking the per-processor contract.

### Decision log
- Write surface writes only the **two customer layers**; platform flags stay in Unleash.
- Endpoint = tri-state (Inherit via row absence); project = on/off (default via row absence).
- Catalog lists **all** registered PRODUCT processors; PLATFORM/unknown writes rejected. The per-processor
  platform flag controls **availability/effective state** (surfaced as `platform_disabled`), not visibility;
  writes to a platform-off processor are rejected (EF-C6) to avoid no-effect toggles.
- Reuse `PipelineResolver` (event-free) to compute displayed effective state (single source of precedence).
- No new tables, no migration (tables already shipped by the engine story); additive code only.
- No cache invalidation in v1 (no cache exists); invalidation specified as a forward hook (BR-C8).
- Config changes are forward-only; no retroactive re-processing.
- **Scope confirmed (owner, 2026-07-06):** ship **project + endpoint** in v1 (full four-layer surface).
- **Catalog auth confirmed:** `GET /v1/processors` is auth-only, no project scope (global metadata, no tenant data).
- **No UI-gating flag confirmed:** system not yet in use and PRODUCT processors ship alongside this work; empty-state
  covers the transient gap. Audit stays logs + metric in v1 (no persisted audit table).

## 3. Functional Design

Actors: an **authenticated customer** (project owner) via `webhookr-web`. The engine/worker is **not** an actor
here — it only *reads* what this surface writes, on the next event.

### Main Flow
1. Customer opens **Project → Settings → Processors** (or **Endpoint → Processors**).
2. Web calls the BFF; BFF calls svc `GET .../processor-configs` → svc verifies ownership, reads the registry +
   platform flags + stored config, and returns the **configurable-processor list** with, per processor:
   `processorName`, `category` (always `PRODUCT`), `platformAvailable`, the stored value for this scope
   (`true | false | null`), and the **effective** state + `source` for this scope.
3. UI renders one row per processor: project screen = On/Off toggle; endpoint screen = On/Off/Inherit control,
   each showing the effective state and, when it differs from the toggle, why (e.g. “Inheriting project: On”,
   “Disabled by platform”).
4. Customer changes a control:
   - **Set On/Off** → BFF mutation → svc `PUT .../processor-configs/:processorName { enabled }` → svc validates
     (registered ∧ PRODUCT ∧ platform-available), ownership, then **upserts** the unique `(scopeId, processorName)`
     row.
   - **Endpoint → Inherit** (or **project → Reset to default**) → BFF mutation → svc
     `DELETE .../processor-configs/:processorName` → svc deletes the row if present (idempotent).
5. svc returns the updated per-processor read model; the UI reflects the new effective state. A structured log +
   counter record the change.
6. The **next event ingested for that endpoint** is resolved by the unchanged `PipelineResolver` using the new
   row (BR-C7). In-flight/already-persisted events are unaffected.

### Alternative Flows
#### AF-C1 — Endpoint override diverges from project
Project row = On, endpoint row = Off → effective (endpoint) = Off, `source = endpoint`. Project screen still shows
On. Resolver applies endpoint over project (`endpointValue ?? projectValue`). Setting the endpoint control back to
**Inherit** deletes the endpoint row → effective follows project again.

#### AF-C2 — Reset an already-default value
`DELETE` for a processor with no stored row (project already at default, or endpoint already inheriting) →
**204 No Content / idempotent success**; no row change.

#### AF-C3 — Processor whose platform flag is off
The processor **still appears** in the catalog and the scope list. The read model returns
`platformAvailable = false`, `source = platform_disabled`, `effectiveEnabled = false` regardless of any stored
value. The UI renders the row **read-only** with a “temporarily unavailable by the platform” note; the toggle is
disabled and a `PUT` for it is rejected (EF-C6). `DELETE`/reset remains allowed. When the platform later enables
the flag, the same processor’s effective state resumes following its stored config (or default).

#### AF-C4 — Empty catalog (launch state)
No configurable processor registered → list is `[]` → UI shows an empty state (“No configurable processors yet”).
All read endpoints succeed with an empty array; write endpoints only ever 404 (unknown processor).

### Error Flows
#### EF-C1 — Unknown processor name
`PUT`/`DELETE .../processor-configs/:unknownName` → **404 Not Found** (`ProcessorNotFound`); no row written.

#### EF-C2 — PLATFORM (non-configurable) processor
Write targets a registered **PLATFORM** processor (e.g. `noop`) → **409 Conflict** (`ProcessorNotConfigurable`);
no row written. (Distinguished from 404 so a client can tell “doesn’t exist” from “exists but isn’t yours to
configure”.)

#### EF-C6 — Set a platform-disabled processor
`PUT .../processor-configs/:name` where the target is a registered PRODUCT processor whose per-processor platform
flag is **off** (`platformAvailable = false`) → **409 Conflict** (`ProcessorUnavailable`); no row written. This
prevents persisting a toggle that would have no effect. `DELETE`/reset is **not** rejected in this state (removing
a row is always safe). Distinct from EF-C2 (`ProcessorNotConfigurable`, which is for PLATFORM-category processors).

#### EF-C3 — Ownership failure
Project not owned by the user, or endpoint not under the project → **404 Not Found** (same opaque-404 pattern as
`ProjectService.findOne` / `EndpointService.findOne`; never 403-with-existence-leak). No row read or written.

#### EF-C4 — Invalid body
`PUT` with missing/non-boolean `enabled` (or extra fields) → **400 Bad Request** via the global `ValidationPipe`
(whitelist + `forbidNonWhitelisted`).

#### EF-C5 — Persistence/infra fault
Prisma error on upsert/delete → **500**, standard error filter; no partial state (single-row op). Client may retry
(operations are idempotent).

### Business Rules
#### BR-C1 — Writes cover only the two customer layers
This surface writes `ProjectProcessorConfig.enabled` and `EndpointProcessorConfig.enabled` only. It never sets the
global or per-processor **platform** flags, and never writes `config Json`. The four-layer resolution (engine
BR-003) is unchanged; this feature only populates two of its inputs.

#### BR-C2 — Only PRODUCT processors are configurable
A write is accepted only if the target is a **registered** processor with `category = PRODUCT`. PLATFORM
processors are rejected (EF-C2). Mirrors engine BR-002 (PLATFORM = non-configurable).

#### BR-C3 — Unknown processors are rejected
A write is accepted only for a processor **present in the registry** (`EventProcessorRegistry.ordered()`). Unknown
names are rejected (EF-C1). This prevents dormant rows for processors that don’t exist.

#### BR-C4 — Catalog = all registered PRODUCT processors
The catalog (`GET /v1/processors`) lists **every** registered `PRODUCT` processor, independent of its
per-processor platform flag. PLATFORM processors are excluded (not customer-configurable — BR-C2). The
per-processor platform flag does **not** filter the catalog; it drives `platformAvailable` / effective state:
a flag that is off yields `platformAvailable = false`, `effectiveEnabled = false`, `source = platform_disabled`,
and a rejected `PUT` (EF-C6), but the processor remains **visible** so the UI can explain the state. The catalog
is the authoritative “what exists to configure” set for the UI; per-row `platformAvailable` tells it what is
currently actionable.

#### BR-C5 — Endpoint tri-state; project bi-state; row-absence = fallback
- **Project** config for a processor is `enabled ∈ {true, false}` when a row exists, else **default** (disabled).
  `reset` (DELETE) removes the row → default.
- **Endpoint** config is `enabled ∈ {true, false}` when a row exists, else **inherit project**. `reset` (DELETE)
  = Inherit. This exactly matches the resolver’s `endpointValue ?? projectValue ?? productDefault(false)`.

#### BR-C6 — Effective state is resolver-derived
`platformAvailable` + `effectiveEnabled` in the read model are computed from the **same** inputs and precedence
the ingest resolver uses. `PipelineResolver.resolve(processors, enablement)` is **event-free** (it takes the
processor list + the project/endpoint enablement maps and reads the platform flags internally, returning
`enabled` + `disabledReason`), so the config service reuses it directly rather than re-implementing precedence
(R-C3): `platformAvailable = disabledReason !== 'PLATFORM_DISABLED'`, `effectiveEnabled = resolved.enabled`. The
`source ∈ { endpoint, project, default, platform_disabled }` and `stored` are derived from the config maps the
service already loaded (`platform_disabled` when the flag is off; else `endpoint` if an endpoint row exists, else
`project` if a project row exists, else `default`). **Implementation note:** if reusing `resolve` proves awkward
(e.g. the caller wants `source` without re-deriving it), extract a **pure enablement-resolution helper** shared by
both `PipelineResolver` and this service — **without** changing the resolver's or the ingest path's behavior.

#### BR-C7 — Forward-only application
A config write affects only events ingested **after** it commits. Already-persisted events keep their
`EventProcessorRun` rows and `processingStatus`; there is no retroactive re-processing (engine BR-016). Replay
still bypasses the engine (BR-011) and is gated (BR-015), so replay is unaffected by config changes.

#### BR-C8 — Cache invalidation is a forward requirement
No resolver config cache exists in v1, so writes need no invalidation. **If** the short-TTL, `endpointId`-keyed
resolver cache described in the engine spec (NFR Performance) is later introduced, every successful write/reset in
this surface MUST invalidate the affected `endpointId` (endpoint scope) or all endpoints of the `projectId`
(project scope). Captured so the future cache lands with its invalidation hook.

#### BR-C9 — Idempotent writes
`PUT` is an idempotent upsert on the unique `(scopeId, processorName)` key; repeating it yields the same row.
`DELETE` is idempotent (succeeds whether or not a row existed). No duplicate rows are possible (DB unique
constraint from the engine migration).

### Validation Rules
- `processorName`: path param; must match a registry entry (BR-C3) **and** be `PRODUCT` (BR-C2). Format guard:
  `^[a-z0-9][a-z0-9-]{0,62}$` (reject before DB lookup).
- **PUT only:** the target's per-processor platform flag must be **on** (`platformAvailable = true`); otherwise
  reject with 409 `ProcessorUnavailable` (EF-C6/BR-C4). `DELETE`/reset is exempt from this check.
- `enabled`: required `boolean` in the `PUT` body; no other fields accepted (`forbidNonWhitelisted`).
- `config`: **not** accepted in any body (reserved column; ignored/rejected if sent).
- Scope ids (`projectId`, `endpointId`): validated by ownership lookup, not format-trusted.
- Uniqueness: enforced by `@@unique([projectId, processorName])` / `@@unique([endpointId, processorName])` (already
  in schema); the service upserts on that key.

### Permissions
`JwtAuthGuard` on every route (global default; no `@Public()`). Authorization mirrors `EventService.findOne`:
- **Project scope:** `projectService.findOne(projectId, userId)` must succeed (project owned by the caller).
- **Endpoint scope:** `endpointService.findOne(endpointId, projectId, userId)` must succeed (endpoint under a
  project owned by the caller).
Failure → opaque **404** (EF-C3), never leaking existence. No new roles/permissions are introduced (single-owner
model, consistent with the rest of `webhookr-svc`). **Decision:** the catalog read (`GET /v1/processors`) requires
`JwtAuthGuard` but **no project scope** — it lists global processor metadata only (name/category/availability), no
tenant data — so it is not a nested project route.

## 4. Non-Functional Requirements

### Performance
- Reads: catalog = in-memory registry scan + one flag check per PRODUCT processor (bounded, tiny). Scope config
  read = the existing `ProcessorConfigRepository.findEnablement` shape (one indexed `findMany` per scope on
  `projectId` / `endpointId`; indexes already exist). No N+1.
- Writes: a single indexed upsert/delete on a unique key. O(1).
- **No impact on the ingest hot path** — this surface runs in the request/API path, never in the worker; the
  per-event resolver read is byte-for-byte unchanged.

### Scalability
Row growth is bounded by (#projects + #endpoints) × (#configurable processors) — small. No fan-out, no queues, no
new backing services.

### Availability
Additive HTTP surface; if it’s down, ingest/delivery are unaffected (resolver reads whatever rows exist). No new
external dependency. Degrades independently of the data plane.

### Security
- Every route authenticated + ownership-checked (Permissions); cross-tenant writes impossible (R-C5).
- Bodies validated by `class-validator` DTOs under the global `ValidationPipe` (whitelist + forbidNonWhitelisted).
- No sensitive data involved: config rows carry only `enabled` + names (engine BR-014 already forbids payload/
  secrets in the processing domain). Nothing new to redact.
- Rate limiting: covered by the global `ThrottlerGuard`.
- BFF forwards the caller JWT to svc (existing `WebhookrClientService` pattern); svc is the single authorization
  authority — the BFF performs no independent authz (consistent with existing endpoints/events modules).

### Observability
#### Logging
Structured Pino at the svc service boundary:
`svc.processor_config.read | set | reset`, with `userId` (auto-injected), `projectId`, `endpointId?`,
`processorName`, `scope` (`project|endpoint`), `enabled?`, and for writes the `previous` → `next` value. No payload
data. Rejections log `svc.processor_config.rejected` with `reason` (`unknown | not_configurable | not_owned`).

#### Metrics
Prometheus (`@willsoto/nestjs-prometheus`):
- `processor_config_writes_total{scope,processor,action}` (`action ∈ set_enabled | set_disabled | reset`).
- `processor_config_rejections_total{scope,reason}`.
Name/units per repo convention (counters; no histogram needed — no meaningful latency dimension beyond the generic
HTTP metrics interceptor already in place).

#### Tracing
Wrap the service methods in `runWithSpan('processor-config.<read|set|reset>')`, consistent with `EventService` /
`EndpointService`. Nested under the incoming HTTP span.

## 5. Technical Design

### Impacted Repositories
- `webhookr-svc` — new `src/processor-config/` capability (service + controllers + DTOs + a small read repository),
  reusing the existing `EventProcessorRegistry` and `PipelineResolver` from `src/event-processing/`; `docs/` + ADR.
  **No Prisma migration** (tables already exist).
- `webhookr-bff` — new `src/processor-configs/` GraphQL module (models, inputs, resolvers, service) over
  `WebhookrClientService`.
- `webhookr-web` — new `src/features/processor-configs/` (data layer + settings UI) wired into the project and
  endpoint dashboard screens.
- `webhookr-artifacts` — this spec + ADR/reconciliation.
- `gitops` / Unleash — **no change** (no new flag; the existing per-processor platform flags govern
  availability/effective state, read-only here).

### Architecture
Reuse-first, mirroring existing modules. No engine/resolver changes; the resolver is **imported read-only** to
derive effective state (BR-C6).

**`webhookr-svc/src/processor-config/`**
```
processor-config.module.ts          # imports EventProcessingModule (registry+resolver), ProjectModule, EndpointModule
processor-config.service.ts         # ownership → validate (registry+category+platform-available) → upsert/delete → resolver-derived read model
processor-config.repository.ts      # thin Prisma upsert/delete/read on Project/EndpointProcessorConfig (writes)
project-processor-config.controller.ts   # /v1/projects/:projectId/processor-configs
endpoint-processor-config.controller.ts  # /v1/projects/:projectId/endpoints/:endpointId/processor-configs
processor-catalog.controller.ts          # /v1/processors  (configurable catalog)
dto/
  set-processor-config.dto.ts       # { enabled: boolean }
interfaces/
  processor-config-view.interface.ts # { processorName, category, platformAvailable, stored, effectiveEnabled, source }
  processor-catalog-item.interface.ts
*.spec.ts co-located
```
- `EventProcessingModule` must **export** `EventProcessorRegistry` + `PipelineResolver` (currently it exports only
  `EventProcessingService`) so this module can reuse them. Small, additive export change — no behavior change.
- Ownership is delegated to the existing `ProjectService` / `EndpointService` (already the pattern in
  `EventService`), not re-implemented.

**`webhookr-bff/src/processor-configs/`** (GraphQL, mirrors `endpoints` module)
```
processor-configs.module.ts
services/processor-configs.service.ts        # WebhookrClientService GET/PUT/DELETE passthrough
resolvers/processor-configs.query.resolver.ts   # processorCatalog, projectProcessorConfigs, endpointProcessorConfigs
resolvers/processor-configs.mutation.resolver.ts# setProject/Endpoint..., resetProject/Endpoint...
models/processor-config.model.ts, processor-catalog-item.model.ts
dto/set-processor-config.input.ts
```

**`webhookr-web/src/features/processor-configs/`** (mirrors `features/endpoints`)
```
data/processor-config.queries.ts / .mutations.ts / .repository.ts (hooks)
types/processor-config.types.ts
components/processor-settings-list.tsx       # rows + toggle (project) / tri-state (endpoint)
components/processor-settings-section.tsx    # wraps list, empty state, loading/error
```
Wired into `dashboard/projects/[projectId]/…` (project settings) and
`dashboard/endpoints/[endpointId]/…` (endpoint settings).

**Read model** returned by svc (per processor, per scope):
```ts
interface ProcessorConfigView {
  processorName: string;
  category: 'PRODUCT';
  platformAvailable: boolean;            // per-processor platform flag
  stored: boolean | null;                // this scope's row value; null = no row (default/inherit)
  effectiveEnabled: boolean;             // resolver-derived for this scope
  source: 'endpoint' | 'project' | 'default' | 'platform_disabled';
}
```

### ADR
#### Decision
1. **New `processor-config` capability, engine untouched.** A separate module owns the write surface and the
   read model; it reuses `EventProcessorRegistry` + `PipelineResolver` and delegates ownership to
   `ProjectService`/`EndpointService`. The engine/resolver/ingest/delivery code is not modified (only
   `EventProcessingModule` exports two providers).
2. **Idempotent PUT set + DELETE reset**, keyed by the existing unique `(scopeId, processorName)`.
3. **Endpoint tri-state via row-absence = Inherit; project default via row-absence.** Mirrors the resolver’s
   `??` fallback so write and read semantics can’t drift.
4. **Catalog = all registered PRODUCT processors; platform flag governs availability, not visibility.** Every
   PRODUCT processor is shown; a platform-off one appears read-only as `platform_disabled` and its `PUT` is
   rejected (EF-C6). PLATFORM processors are excluded (not configurable).
5. **Effective state derived from `PipelineResolver`**, not re-implemented in the config service.
6. **Logs + metric for audit; no audit table in v1.**

#### Alternatives Considered
- **Extend the endpoint/project modules instead of a new module.** Rejected: config touches the event-processing
  registry/resolver and spans project+endpoint scopes; a dedicated capability keeps bounded contexts clean
  (svc CLAUDE.md: “prefer extending existing modules” is outweighed here by two-context coupling).
- **Single boolean per scope with no “inherit”.** Rejected for endpoints: it can’t express “fall back to project,”
  which the resolver already does — you’d lose the override model. Row-absence = inherit fits the data model.
- **PATCH with a partial config object / bulk set.** Rejected for v1: a single idempotent boolean set (PUT) is
  simpler and matches REST semantics; bulk is a non-breaking future add (OQ-C5).
- **Hide platform-off PRODUCT processors from the catalog.** Rejected (was the earlier draft position): hiding
  makes a processor blink in/out as an operator flips its platform flag, and gives the customer no way to see that
  a processor exists but is temporarily unavailable. Chosen instead (BR-C4): **show all** PRODUCT processors, mark
  platform-off ones `platform_disabled` and read-only, and **reject their `PUT`** (EF-C6) so no no-effect toggle
  is persisted — visibility without a misleading actionable control.
- **Re-implement precedence in the config service for the read model.** Rejected (R-C3): two copies of the
  four-layer logic will drift; reuse the resolver.
- **Persisted audit table now.** Rejected for v1: no audit infra exists; logs+metric suffice; a table lands with
  platform-wide audit (OQ-C4).
- **New feature flag gating the surface (svc or web).** Rejected as unnecessary: the catalog is empty until a GA
  PRODUCT processor exists, so the surface is inert by construction; the system is not yet in use and the first
  PRODUCT processors ship alongside this work, so there is no live window a kill-switch would protect. The
  empty-state UI covers the transient gap.

#### Tradeoffs
- A new module + exporting two providers vs. keeping event-processing fully closed — chosen for clean contexts and
  zero engine edits.
- Tri-state endpoint UX is slightly more complex than a plain toggle — chosen because it’s the honest
  representation of the resolver’s override model.
- Logs-only audit is weaker than a queryable table — accepted for v1; upgrade path noted.

#### Consequences
- The four-layer model becomes fully operable end-to-end; PRODUCT processors can ship with real customer opt-in.
- `EventProcessingModule` gains two exports (additive, safe).
- The web app gains reusable processor-settings components for both project and endpoint scopes.
- The catalog is empty until the first GA PRODUCT processor — the feature is safely shippable ahead of any real
  processor.

### Data Model
**No schema change.** The engine story already shipped and migrated `ProjectProcessorConfig`,
`EndpointProcessorConfig` (both with `enabled Boolean`, reserved `config Json?`, `@@unique([scopeId,
processorName])`, `@@index`, `onDelete: Cascade`). This story only **writes** existing columns.

- Reads for a scope: `findMany({ where: { projectId | endpointId }, select: { processorName, enabled } })`
  (reuses the `ProcessorConfigRepository.findEnablement` shape).
- Writes: `upsert({ where: { projectId_processorName | endpointId_processorName }, create/update: { enabled } })`.
- Reset: `deleteMany({ where: { scopeId, processorName } })` (idempotent; no throw if absent).
- `config Json?` is never read or written by this story (reserved).

### API Contracts
All under `JwtAuthGuard`; ownership per Permissions. New REST surface in `webhookr-svc`:

#### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/processors` | List **all** registered PRODUCT processors: `[{ processorName, category, platformAvailable }]` (platform-off ones included, `platformAvailable: false`) |
| GET | `/v1/projects/:projectId/processor-configs` | Project-scope view: `ProcessorConfigView[]` |
| PUT | `/v1/projects/:projectId/processor-configs/:processorName` | Upsert project `enabled`; returns the updated `ProcessorConfigView` |
| DELETE | `/v1/projects/:projectId/processor-configs/:processorName` | Reset project to default (delete row); `204` |
| GET | `/v1/projects/:projectId/endpoints/:endpointId/processor-configs` | Endpoint-scope view: `ProcessorConfigView[]` |
| PUT | `/v1/projects/:projectId/endpoints/:endpointId/processor-configs/:processorName` | Upsert endpoint `enabled`; returns updated `ProcessorConfigView` |
| DELETE | `/v1/projects/:projectId/endpoints/:endpointId/processor-configs/:processorName` | Reset endpoint to Inherit (delete row); `204` |

#### Request
```json
// PUT .../processor-configs/:processorName
{ "enabled": true }
```

#### Response
```json
// GET .../processor-configs  (endpoint scope example)
[
  {
    "processorName": "validation",
    "category": "PRODUCT",
    "platformAvailable": true,
    "stored": false,
    "effectiveEnabled": false,
    "source": "endpoint"
  },
  {
    "processorName": "filtering",
    "category": "PRODUCT",
    "platformAvailable": true,
    "stored": null,
    "effectiveEnabled": true,
    "source": "project"
  },
  {
    "processorName": "routing",
    "category": "PRODUCT",
    "platformAvailable": false,
    "stored": true,
    "effectiveEnabled": false,
    "source": "platform_disabled"
  }
]
```
```json
// 404 unknown processor
{ "statusCode": 404, "error": "ProcessorNotFound", "message": "Processor \"foo\" is not registered" }
```
```json
// 409 not configurable (PLATFORM category)
{ "statusCode": 409, "error": "ProcessorNotConfigurable", "message": "Processor \"noop\" is not customer-configurable" }
```
```json
// 409 platform-disabled (PRODUCT, platform flag off) — PUT only
{ "statusCode": 409, "error": "ProcessorUnavailable", "message": "Processor \"validation\" is temporarily unavailable by the platform" }
```

**BFF (GraphQL)** — mirrors the REST surface:
- Queries: `processorCatalog: [ProcessorCatalogItem!]!`,
  `projectProcessorConfigs(projectId: ID!): [ProcessorConfig!]!`,
  `endpointProcessorConfigs(projectId: ID!, endpointId: ID!): [ProcessorConfig!]!`.
- Mutations: `setProjectProcessorConfig(projectId, processorName, input: {enabled}) : ProcessorConfig`,
  `resetProjectProcessorConfig(projectId, processorName): Boolean`,
  `setEndpointProcessorConfig(projectId, endpointId, processorName, input): ProcessorConfig`,
  `resetEndpointProcessorConfig(projectId, endpointId, processorName): Boolean`.
`ProcessorConfig` GraphQL type = the `ProcessorConfigView` shape.

### Event Contracts
#### Published Events
None. This surface writes DB rows only; it does not enqueue or emit any queue/event.
#### Consumed Events
None.

### Message Contracts
N/A — no queue/message involvement (the resolver reads the rows synchronously during the existing ingest flow,
which is unchanged).

### External Dependencies
None new. Existing Prisma/Postgres, OpenFeature/Unleash (read-only flag checks), OTel, Prometheus, Pino, and the
BFF `WebhookrClientService`.

### Feature Flags
**None.** No new flag is introduced by this story. The svc surface is inert until a GA PRODUCT processor is
registered (catalog empty), and the existing per-processor `webhookr.processors.<name>.enabled` flags (owned by
the engine story) already govern GA. **Decision:** no UI-gating flag — the system is not yet in use and the first
PRODUCT processors are being implemented alongside this work, so the empty-state UI (AF-C4) briefly covers the gap
and is then naturally populated; a kill-switch flag would add config with no real window to protect.

### Migrations
**None.** Tables + enums + indexes were created by the engine story’s migration
`20260706000000_event_processor_engine`. This story is code-only.

### Rollback Strategy
- **Web:** revert the web PRs — the settings section disappears; stored rows remain harmless (resolver reads them
  as today).
- **BFF/svc:** revert the additive PRs. The `EventProcessingModule` export change and the new module are additive;
  removing them restores the exact prior behavior. No data migration to undo.
- **Data:** any rows written are valid resolver inputs; leaving them in place is safe. If a full purge is desired,
  `DELETE FROM "ProjectProcessorConfig"; DELETE FROM "EndpointProcessorConfig";` returns every processor to its
  default — but this is not required for rollback.

## 6. Implementation Strategy

### Recommended Implementation Order
1. **svc — export reuse:** export `EventProcessorRegistry` + `PipelineResolver` from `EventProcessingModule`.
2. **svc — read repository + service core:** catalog build, scope reads, resolver-derived `ProcessorConfigView`;
   unit-tested.
3. **svc — write path:** upsert/delete + validation (registry ∧ PRODUCT ∧ platform-available for `PUT`) +
   ownership; unit-tested.
4. **svc — controllers + DTO + Swagger + observability** (catalog, project, endpoint); e2e happy + error paths.
5. **bff — module:** service (client passthrough), models, inputs, query + mutation resolvers; unit + integration.
6. **web — data layer:** queries/mutations/repository hooks (`processor-config.*`); tests.
7. **web — UI:** `processor-settings-list` (toggle + tri-state), section with empty/loading/error, wired into
   project + endpoint screens; component + e2e tests; optional UI flag gate.
8. **docs + ADR + artifact reconciliation** (svc `docs/`, ADR `docs/adr/`, engineering INDEX).

### Dependencies
- #2–#4 depend on #1; #5 depends on #4 (REST contract stable); #6 depends on #5 (GraphQL contract stable);
  #7 depends on #6. Cross-repo contracts are the sequencing gates.

### Breaking Changes
None. Purely additive across all three repos. Existing endpoints/queries/mutations are untouched.

### Backward Compatibility
Fully backward compatible: no schema change, no engine/resolver change, no altered existing contract. With an empty
catalog the behavior is identical to today. Stored rows are the same shape the resolver already reads.

### Rollout Strategy
1. Ship svc → bff → web in dependency order, each behind normal PR review.
2. In non-prod, register a **stub** PRODUCT processor (as the engine e2e already does) with its platform flag on to
   exercise the full catalog → toggle → resolve loop.
3. The surface goes live with an **empty catalog** (no GA PRODUCT processor yet) — safe, and the system is not yet
   in production use. As the first real PRODUCT processors ship (their own stories, in parallel with this work),
   flipping each `webhookr.processors.<name>.enabled` flag on makes it appear in the catalog and become
   configurable — no UI flag to coordinate.

## 7. Task Split

### Infrastructure
**Repositories:** N/A
**Tasks:**
- [ ] None. No new feature flag or infra change — the existing per-processor `webhookr.processors.<name>.enabled`
      flags (engine story) already gate catalog visibility; no UI flag is introduced (see Feature Flags).

### Backend
**Repositories:** `webhookr-svc`
**Tasks:**
- [ ] Export `EventProcessorRegistry` + `PipelineResolver` from `EventProcessingModule`.
- [ ] `ProcessorConfigModule` importing event-processing (registry+resolver), project, endpoint modules.
- [ ] `ProcessorConfigService`: ownership delegation; validate registered ∧ PRODUCT (BR-C2/C3); reject `PUT` when
      `platformAvailable = false` (EF-C6); upsert/delete; catalog = **all** PRODUCT; resolver-derived
      `ProcessorConfigView` (reuse event-free `PipelineResolver.resolve`; extract a shared enablement helper only
      if needed, BR-C6); `runWithSpan` wrappers.
- [ ] `ProcessorConfigRepository`: scope `findMany` read; `upsert`; idempotent `deleteMany`.
- [ ] `SetProcessorConfigDto { enabled: boolean }` (whitelist; reject `config`/extras).
- [ ] Controllers: catalog (`/v1/processors`), project + endpoint config (GET/PUT/DELETE), Swagger annotations,
      error responses (404 unknown, 409 not-configurable, 409 platform-unavailable, 400 invalid body, 404 ownership).
- [ ] Observability: structured logs (`svc.processor_config.*`), counters
      (`processor_config_writes_total`, `processor_config_rejections_total`).
- [ ] Unit specs (service, repository, controllers) to the repo bar; e2e for happy + each error flow + cross-tenant
      404 + write→ingest→resolve reflects new value (with a stub PRODUCT processor).

### API
**Repositories:** `webhookr-bff`
**Tasks:**
- [ ] `ProcessorConfigsModule` (imports `WebhookrClientModule`).
- [ ] `ProcessorConfigsService`: GET/PUT/DELETE passthrough to svc (JWT forwarded).
- [ ] Models `ProcessorConfig`, `ProcessorCatalogItem`; input `SetProcessorConfigInput`.
- [ ] Query resolver (catalog, project configs, endpoint configs) + mutation resolver (set/reset × project/
      endpoint).
- [ ] Unit + integration specs mirroring the `endpoints` module tests.

### Frontend
**Repositories:** `webhookr-web`
**Tasks:**
- [ ] `features/processor-configs/data`: `queries.ts`, `mutations.ts`, `repository.ts` (typed hooks, cache
      refetch/normalized-update per the endpoints pattern), `types/processor-config.types.ts`.
- [ ] `components/processor-settings-list.tsx` (project toggle; endpoint On/Off/Inherit), showing effective state +
      source; `platformAvailable = false` rows render **read-only** with a “temporarily unavailable by the
      platform” note (control disabled, no `PUT`).
- [ ] After a `reset`/`DELETE`, the mutation hook **refetches the scope list** (no optimistic/stale display), per
      the endpoints `refetchQueries` + `awaitRefetchQueries` pattern.
- [ ] `components/processor-settings-section.tsx` (empty/loading/error states).
- [ ] Wire into project settings and endpoint settings screens.
- [ ] Component tests + Playwright e2e (toggle project, override endpoint, inherit reset, empty state).

### Documentation
**Repositories:** `webhookr-svc`, `webhookr-artifacts`
**Tasks:**
- [ ] `webhookr-svc/docs/`: processor-config surface (endpoints, tri-state semantics, ownership, validation);
      cross-link to `docs/event-processing.md`.
- [ ] ADR under `webhookr-svc/docs/adr/` (the 6 decisions above), cross-linking engine ADR 0003.
- [ ] Reconcile into `webhookr-artifacts/engineering/` (this spec + INDEX line) per the source-of-truth workflow;
      link back from the engine spec’s Future Improvements as “delivered by processor-config-management”.

## 8. Testing Strategy

### Unit Tests
Co-located `*.spec.ts`, boundary mocks (Prisma, flags, registry, resolver, `ProjectService`/`EndpointService`),
no `any`:
- **Service — validation:** unknown → 404; PLATFORM (`noop`) → 409 `ProcessorNotConfigurable`; platform-off
  PRODUCT → **present** in catalog (`platformAvailable: false`) but `PUT` → 409 `ProcessorUnavailable` while
  `DELETE` still succeeds; platform-on PRODUCT → accepted.
- **Service — read model:** effective/source matrix reusing resolver — endpoint row overrides project;
  project-only; default (no rows); `platform_disabled` when the per-processor flag is off (with **and** without a
  stored row) — `effectiveEnabled` stays false and `source = platform_disabled` regardless of `stored`.
- **Service — write:** upsert creates then updates same row; `set` toggles both directions; `reset` deletes and is
  idempotent (no throw when absent).
- **Service — ownership:** project not owned → 404; endpoint not under project → 404 (no read/write).
- **Repository:** upsert on unique key; `deleteMany` idempotency; scope `findMany` selection.
- **Controllers:** route/param wiring, DTO validation (missing/non-boolean `enabled`, extra `config` rejected),
  status codes.

### Integration Tests
- **svc e2e** (real Postgres): register a **stub** PRODUCT processor with its platform flag **on** → catalog lists
  it; PUT project enabled → row exists; GET reflects effective; PUT endpoint disabled → endpoint override wins;
  DELETE endpoint → inherits project; ingest an event for that endpoint → assert the resulting `EventProcessorRun`
  decision matches the config (BR-C7 forward application). Flip the stub's platform flag **off** → it still appears
  in the catalog with `platformAvailable: false` / `source: platform_disabled`, its `PUT` → 409
  `ProcessorUnavailable`, but `DELETE` still succeeds. PLATFORM/unknown writes rejected (409/404); cross-tenant 404.
- **bff integration:** each query/mutation proxies to a mocked svc with correct paths + JWT forwarding + error
  mapping.

### End-to-End Tests
- **web (Playwright):** project screen toggles a processor on → persists; endpoint screen sets Off (override) then
  Inherit (reset); empty-state renders when catalog is empty; “Disabled by platform” row is read-only.

### Manual Validation
Non-prod with a stub GA PRODUCT processor: toggle at project, override at endpoint, reset to inherit; send a
webhook and confirm the `EventProcessorRun` reflects the latest config; confirm logs/metrics
(`processor_config_writes_total`, rejections) in Grafana; confirm cross-tenant access 404s.

## 9. Definition of Ready
- [x] Source-of-truth engine spec v2 read; write surface scope confirmed (engine §2/§13).
- [x] Enablement model reused unchanged (BR-003); this story writes only the two customer layers.
- [x] Tri-state (endpoint) vs bi-state (project) semantics decided and mapped to row-absence (OQ-C1).
- [x] REST verbs decided: PUT set / DELETE reset, idempotent (OQ-C2).
- [x] Catalog visibility decided (revised): **all** registered PRODUCT processors listed; platform flag drives
      `platformAvailable`/effective state (not visibility); platform-off `PUT` rejected (OQ-C3, EF-C6, BR-C4).
- [x] Ownership model confirmed = `EventService.findOne` pattern; no new roles.
- [x] No migration required (tables already shipped); confirmed additive-only.
- [x] Audit (OQ-C4) and bulk (OQ-C5) explicitly deferred — do not block.
- [x] Owner assigned: thiagojv.

## 10. Definition of Done
- [ ] svc catalog + project + endpoint config REST endpoints exist under `JwtAuthGuard` with ownership checks
      mirroring `EventService.findOne`; unit + e2e to the repo bar.
- [ ] Catalog lists **all** registered PRODUCT processors; PLATFORM excluded. Writes accepted **only** for
      registered PRODUCT processors whose platform flag is on; PLATFORM → 409 `ProcessorNotConfigurable`,
      platform-off PRODUCT `PUT` → 409 `ProcessorUnavailable` (DELETE still allowed), unknown → 404; `config Json`
      never written.
- [ ] Endpoint tri-state (On/Off/Inherit=reset) and project bi-state (On/Off/reset=default) behave exactly as the
      resolver’s `??` fallback; effective state is resolver-derived (no duplicated precedence logic).
- [ ] A write is reflected by the resolver on the **next** ingested event (integration-proven); already-persisted
      events and replay are unaffected.
- [ ] BFF queries/mutations proxy the surface with JWT forwarding and error mapping; tests mirror `endpoints`.
- [ ] Web project + endpoint “Processors” settings render the catalog, toggle/tri-state, effective state + source,
      empty/loading/error states; component + Playwright tests pass.
- [ ] Observability: logs + `processor_config_writes_total` / `processor_config_rejections_total` emitted.
- [ ] `EventProcessingModule` exports registry+resolver (additive); no engine/resolver/ingest/delivery behavior
      changed.
- [ ] Docs + ADR written (svc), engineering artifact reconciled + INDEX updated; engine spec Future Improvements
      cross-linked as delivered.
- [ ] All PRs pass ESLint, TS build, unit + e2e; each PR single-concern.

## 11. Estimate

| Size | Selected |
|------|:--------:|
| P    | ☐        |
| M    | ☐        |
| G    | ☑        |
| GG   | ☐        |

Rationale: three repos (svc + bff + web), a catalog + two scoped CRUD surfaces, resolver-reuse for a
resolver-consistent read model, tri-state endpoint UX, and full test matrices — but **no schema/migration**, no
hot-path work, and every layer mirrors an existing module. ~8–10 single-concern PRs. **De-scope lever to M:** ship
**project-scope only** (defer endpoint override + tri-state to a fast follow-up); the resolver already supports the
endpoint layer, so it can land later without rework.

## 12. References
- Engine spec (source of truth): [`feature-spec-event-processor-engine-v2.md`](../2026-07-06-event-processor-engine/2026-07-06-feature-spec-event-processor-engine-v2.md) — BR-002 (categories), BR-003 (four-layer resolution), BR-005 (defaults), BR-011/BR-015 (replay), NFR Performance (cache trigger).
- Engine implementation notes: [`implementation-notes-v1.md`](../2026-07-06-event-processor-engine/2026-07-06-implementation-notes-v1.md).
- `webhookr-svc/src/event-processing/pipeline-resolver.service.ts` — four-layer resolution to reuse (BR-C6).
- `webhookr-svc/src/event-processing/processor-config.repository.ts` — existing read shape (`findEnablement`).
- `webhookr-svc/src/event-processing/event-processor.registry.ts` — registered processors + categories (validation source).
- `webhookr-svc/src/event/event.service.ts` — ownership pattern to mirror (`findOne` → project/endpoint checks).
- `webhookr-svc/src/endpoint/endpoint.controller.ts` — nested-resource controller pattern (`projects/:projectId/…`).
- `webhookr-bff/src/endpoints/*` — GraphQL module pattern (resolver/service/model/input) to mirror.
- `webhookr-web/src/features/endpoints/*` — web feature pattern (data hooks + settings UI) to mirror.
- `webhookr-svc/prisma/schema.prisma` — `ProjectProcessorConfig` / `EndpointProcessorConfig` (already migrated).

## 13. Future Improvements
- **Per-processor `config Json` editor** (engine OQ5) once processors carry settings beyond on/off.
- **Bulk write** (set many processors in one request) — OQ-C5.
- **Persisted audit-log table** for config changes (who/when/old→new) when platform-wide audit is introduced — OQ-C4.
- **Resolver config cache + write invalidation** (engine NFR Performance) — implement BR-C8’s invalidation hook
  when/if the cache is added.
- **Org/role-based permissions** if/when `webhookr-svc` gains a multi-user workspace model (today: single-owner).
- **Config change → optional targeted re-process** action (distinct from replay) if `STOPPED`/`FAILED` events ever
  need to re-enter the pipeline under new config (engine Future Improvements).
- **Import/export or Terraform provider coverage** for processor config (align with `terraform-provider-webhookr`).

## 14. Appendix
Write-surface data flow (customer plane) vs. the unchanged data plane:
```
webhookr-web (Processor settings: project toggle / endpoint tri-state)
  → webhookr-bff (GraphQL: processorCatalog / *ProcessorConfigs / set*/reset*)
    → webhookr-svc REST (JwtAuthGuard + ownership)
        GET  /v1/processors                                   → catalog (ALL PRODUCT; flag → platformAvailable) [registry + flags]
        GET  /v1/projects/:p/processor-configs                → ProcessorConfigView[]     [resolver-derived]
        PUT  /v1/projects/:p/processor-configs/:name          → upsert ProjectProcessorConfig.enabled
        DEL  /v1/projects/:p/processor-configs/:name          → reset (delete row) → default
        GET  /v1/projects/:p/endpoints/:e/processor-configs   → ProcessorConfigView[]     [resolver-derived]
        PUT  /v1/projects/:p/endpoints/:e/.../:name           → upsert EndpointProcessorConfig.enabled
        DEL  /v1/projects/:p/endpoints/:e/.../:name           → reset (delete row) → Inherit
      writes rows in ProjectProcessorConfig / EndpointProcessorConfig  (existing tables, no migration)

Data plane (UNCHANGED by this story):
  INGEST_EVENTS_QUEUE → IngestProcessor → IngestService.process
    → EventProcessingService.run → PipelineResolver.resolve(registry, flags, config rows)   ← reads the rows above
    → engine executes → delivery gated on `passed`
  Config write applies to the NEXT event only (forward-only, BR-C7); replay bypass + gate unchanged (BR-011/BR-015).

Enablement precedence (unchanged, engine BR-003):
  global flag ∧ per-processor flag ∧ project(row?) ∧ endpoint(row?)   — endpoint overrides project overrides default
  This story writes only the project(row?) and endpoint(row?) inputs; platform flags stay operator-owned.
```
