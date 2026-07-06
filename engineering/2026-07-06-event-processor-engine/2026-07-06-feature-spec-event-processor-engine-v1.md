---
artifact: feature-spec-event-processor-engine
topic: event-processor-engine
date: 2026-07-06
version: 1
source: /upstream
status: Draft
title: Event Processor Engine (pre-delivery pipeline)
owner: TODO: assign (default: thiagojv)
created: 2026-07-06
updated: 2026-07-06
estimate: GG
priority: High
labels: [webhookr-svc, backend, event-driven, pipeline, prerequisite]
---

# Feature Specification

This document describes everything required to implement a feature, from business motivation to production rollout.

## 1. Context

### Background
In `webhookr-svc`, an ingested webhook follows a fixed two-stage worker flow:
`IngestProcessor → IngestService.process()` persists the normalized `Event`, then immediately
enqueues a `delivery.events` job; `DeliveryProcessor → DeliveryService.deliverEvent()` fans the
event out to every destination. There is **no interception point** between persistence and
delivery — every persisted event is unconditionally delivered.

The roadmap requires business logic before delivery (validation, filtering, transformation,
deduplication, routing, enrichment), operational control to disable a faulty processor in prod,
customer-facing control over which processors run, and a per-event **processor execution
timeline** (the immediate follow-up story). None are possible today.

### Problem Statement
There is no predictable, extensible, observable, **configurable** stage between ingest and
delivery. Adding pre-delivery behavior today means editing `IngestService` inline, with no
contract, no deterministic ordering, no per-step result, no platform/customer enablement, and
nothing to render a timeline from.

### Objective
After this feature, `webhookr-svc` runs an ordered pipeline of processors over the normalized
event context after ingest persistence and before delivery. The **engine only orchestrates
execution**; a separate **resolver** computes the *effective pipeline* (which processors run for
this event) from processor categories, platform feature flags, and project/endpoint
configuration. Delivery is enqueued **only** when the pipeline passes. Every processor run
(including disabled/skipped) is persisted as structured data for the future timeline. Adding a
processor requires no change to the engine core.

### Business Value
- Unlocks the full pre-delivery processor roadmap behind one stable contract.
- **Operational safety:** disable a faulty processor in prod via a flag without disabling the
  whole engine.
- **Product control:** customers enable/disable processors per project/endpoint; new processors
  don't silently change existing behavior.
- Hard prerequisite for the processor-execution-timeline story (a core "inspect webhooks" value).

### Success Metrics
- 100% of events (with the engine enabled) produce a persisted `Event.processingStatus` and a
  complete ordered set of `EventProcessorRun` rows (executed + disabled).
- 0 events reach `delivery.events` with a non-`PROCESSED` outcome (asserted in tests + a prod
  metric that must read 0 until a real processor is registered).
- Adding the first real processor touches **0 lines** of `EventProcessorEngine`.
- A processor can be disabled globally in < 1 min via a flag (no deploy).
- Engine + resolver add < 5 ms p95 to ingest with the shipped no-op processor (Assumption;
  validate under load — resolver reads config, so watch DB cost — see BR-013).
- New-module unit coverage ≥ repo bar (near-100% per CLAUDE.md).

## 2. Scope

### In Scope
- Engine + contracts in the `webhookr-svc` service layer: `EventProcessor` (with `category`,
  **no `required`**), `EventProcessorContext`, `EventProcessorResult`, `EventProcessorEngineResult`.
- **Processor categories**: `PLATFORM` (internal, non-configurable) and `PRODUCT` (customer-
  configurable).
- **`PipelineResolver`**: computes the ordered *effective pipeline* + the disabled set from the
  four enablement layers (§3 BR-003).
- **`EventProcessingService`** facade: resolve → execute → persist → return outcome; called by
  `IngestService`.
- Platform feature flags: global `webhookr.processors.enabled` + per-processor
  `webhookr.processors.<name>.enabled`.
- Data model: separated `Event.processingStatus` and `EventProcessorRun` (with `DISABLED`), plus
  `ProjectProcessorConfig` / `EndpointProcessorConfig` (schema + resolver read-path).
- A default **no-op PLATFORM processor** so the pipeline is real but inert.
- Delivery gating, observability (spans/metrics/logs), unit + integration tests.

### Out of Scope
- Any real business processor (validation, filtering, transformation, dedup, routing,
  enrichment) — separate stories.
- The **configuration management surface** — CRUD API/BFF/UI to *write* `ProjectProcessorConfig` /
  `EndpointProcessorConfig`. This story ships the schema + resolver **read-path** only; writing
  config lands with the first product processor that needs it (nothing configurable exists yet).
  Tracked as a dedicated follow-up story (see §13).
- The timeline **read API/UI** (`webhookr-bff`, `webhookr-web`) — next story.
- Changes to `webhookr-ingest`, the ingest HTTP contract, or `DeliveryService` delivery logic.
- Replay-path re-processing (BR-011 keeps replay bypassing the engine).

### Assumptions
- **Assumption:** The engine runs **synchronously inside the existing ingest worker**, not a new
  queue/worker (CLAUDE.md forbids new event-driven infra unless required; processing is in-memory;
  the ingest worker already holds decrypted payload/headers).
- **Assumption:** All processors in the effective pipeline are **stop-on-fail** in v1 (no
  `required`, no fail-open — see BR-006 / OQ1).
- **Assumption:** With no product processors yet, `ProjectProcessorConfig` / `EndpointProcessorConfig`
  have no rows; the resolver falls back to category defaults. The config **write** surface is
  deferred (see Out of Scope).
- **Assumption:** Persisted `reasonCode`/`message` are metadata only, never payload/header content.
- **Assumption:** Processor runs are **replaced per event on each worker attempt**, keyed by
  `(eventId, sequence)`, for idempotency.

### Risks
- **R1 — Delivery regression:** a bug blocks valid events. Mitigation: global flag default-off,
  no-op processor, explicit gating tests, prod `blocked_deliveries` alert.
- **R2 — Latency / DB cost in the ingest hot path:** resolver reads project/endpoint config per
  event. Mitigation: single indexed query (or short-TTL cache); with zero config rows the read is
  trivial; watch histogram (BR-013).
- **R3 — Naming collision:** "processor" overloads BullMQ workers. Mitigation: domain concept is
  `EventProcessor*` under `src/event-processing/`; workers are never called "processors" in new code.
- **R4 — Enablement-logic bugs** (four layers) cause a processor to run/not-run unexpectedly.
  Mitigation: resolver is a pure, exhaustively unit-tested function; DISABLED runs make every
  decision auditable in the timeline.
- **R5 — Scope creep from config tables** that nothing writes yet. Mitigation: read-path only;
  write surface explicitly deferred; tables are additive.
- **R6 — Idempotency on worker retry:** duplicate runs / double enqueue. Mitigation: replace-on-
  attempt writes; delivery `jobId = eventId` dedups enqueue.

### Open Questions
- **OQ1 — Deferred:** Future fail-open — the eventual shape is a per-processor
  `failureMode: BLOCK | CONTINUE`, introduced only when a real need arises. Does not block this story.
- **OQ2 — Resolved:** Replay **bypasses** the engine (BR-011). Rationale: replay re-sends the same
  already-processed, immutable payload; it is not a new ingest, so there is nothing to re-process.
- **OQ3 — Deferred:** Category of ambiguous processors (`validation`, `enrichment`): PLATFORM
  (always-on infra) vs PRODUCT (opt-in) — decided with the story that builds each processor. Does
  not block this story (only the no-op / PLATFORM exists here).
- **OQ4 — Deferred:** Retention/pruning policy for `EventProcessorRun` — defined with the timeline
  story. Does not block this story.
- **OQ5 — Resolved:** Ship the per-processor `config Json?` column now; v1 reads only `enabled`.

### Decision log (resolved during upstream review)
- `required` removed from the contract; fail-open is a separate future concern (`failureMode`).
- Two processor categories: PLATFORM (non-configurable) vs PRODUCT (customer-configurable).
- Four-layer effective-pipeline resolution; endpoint overrides project overrides platform default.
- Global + per-processor platform feature flags.
- `ProjectProcessorConfig` / `EndpointProcessorConfig` — schema + read-path in; write surface deferred.
- Event vs processor-run status separated; `DISABLED` added to run status.
- Global flag off ⇒ engine fully bypassed; on + only no-op ⇒ engine runs and persists a run.
- Engine is orchestration-only; discovery/enablement/config live outside it.

## 3. Functional Design

### Main Flow
1. `IngestProcessor` dequeues an `INGEST_EVENTS_QUEUE` job → `IngestService.process`.
2. `IngestService` opens the sealed payload/headers, resolves the endpoint (now also selecting
   `projectId`), and persists the `Event` with initial `processingStatus = PENDING`.
3. **Global gate:** if `webhookr.processors.enabled` is **false**, the engine is **completely
   bypassed** — no resolution, no runs; enqueue `delivery.events` as today; `processingStatus`
   stays `PENDING` (transitional; the flag is removed after rollout).
4. Otherwise `IngestService` calls `EventProcessingService.run(context)`:
   a. **Registry** enumerates all registered processors in canonical order (each with `category`).
   b. **PipelineResolver** computes, per processor, whether it is enabled (BR-003), producing an
      ordered **effective pipeline** (enabled) and a **disabled set** (with reasons).
   c. **EventProcessorEngine.execute(context, effectivePipeline)** runs enabled processors in
      order, halting on the first `stopped`/`failed`, marking downstream enabled processors
      `SKIPPED` (BR-007/BR-008).
   d. The facade merges results into the full ordered run list: executed results
      (`PASSED/STOPPED/FAILED/SKIPPED`) + disabled processors (`DISABLED`), each with its
      `sequence` in canonical order.
   e. The facade persists `Event.processingStatus` + all `EventProcessorRun` rows in **one
      transaction**, replace-on-attempt.
5. If outcome is `passed` → enqueue `delivery.events` (unchanged). Otherwise do **not** enqueue;
   the worker completes successfully (business-terminal, not an infra error).

### Alternative Flows
#### AF-01 — Processor disabled by platform flag
`webhookr.processors.<name>.enabled = false` → that processor is excluded from the effective
pipeline and persisted as `DISABLED` (reason `PLATFORM_DISABLED`). Others proceed. If all pass,
delivery happens.

#### AF-02 — Processor disabled by customer config
A `PRODUCT` processor is disabled at project or endpoint level → excluded, persisted `DISABLED`
(reason `CONFIG_DISABLED`). Endpoint config overrides project config.

#### AF-03 — Intentional stop (future filter)
A processor returns `stopped` → outcome `stopped`; `processingStatus = STOPPED`; downstream
enabled processors `SKIPPED`; delivery **not** enqueued; worker completes successfully (no retry).

#### AF-04 — Global flag on, only the no-op registered
Effective pipeline = `[NoOp]` (PLATFORM, default-on) → `PASSED` → `processingStatus = PROCESSED`
→ delivery enqueued; one run row persisted. (Engine executes normally and persists runs.)

#### AF-05 — Empty effective pipeline
Every registered processor disabled → engine runs nothing → outcome `passed` (vacuous) →
`PROCESSED` → delivery enqueued; all processors persisted as `DISABLED` (BR-010: disabling
processors never blocks delivery).

### Error Flows
#### EF-01 — Processor failure (returned `failed` or thrown)
Engine records that processor `FAILED` (metadata `reasonCode`/`message`, no payload), marks
downstream `SKIPPED`, outcome `failed`; `processingStatus = FAILED`; delivery **not** enqueued.
The **ingest worker still completes successfully** (business-terminal — no BullMQ retry storm;
contrast the `UnrecoverableError` used for un-decryptable payloads).

#### EF-02 — Persistence/infra failure (status/runs or config read fails)
Transient infra fault → rethrow so the ingest worker retries (BullMQ). Replace-on-attempt makes
retry safe. Delivery not enqueued this attempt.

#### EF-03 — Event not found / endpoint inactive
Unchanged: `IngestService` returns early before persistence; the engine never runs.

### Business Rules
#### BR-001 — Engine is orchestration-only
`EventProcessorEngine` executes a **given** ordered list and nothing else. It must not know which
processors exist, which are enabled, rollout, or customer config. Discovery = Registry;
platform availability = Feature-flag layer; project/endpoint availability = Config layer;
execution = Engine.

#### BR-002 — Two processor categories
Every processor declares `category`: `PLATFORM` (internal: normalization, metadata enrichment,
security validation, compatibility — **not** customer-configurable) or `PRODUCT` (validation,
filtering, transformation, deduplication, routing, enrichment — customer-configurable).

#### BR-003 — Effective-pipeline resolution
A processor executes **iff all** hold:
```
global platform enabled            (webhookr.processors.enabled)
AND per-processor platform enabled (webhookr.processors.<name>.enabled)
AND project enabled                (ProjectProcessorConfig, PRODUCT only)
AND endpoint enabled               (EndpointProcessorConfig, PRODUCT only)
```
Precedence for the customer layer: **Endpoint overrides Project overrides platform default.**
`PLATFORM` processors ignore project/endpoint config (always enabled when platform flags allow).

#### BR-004 — Platform flags are hard kill-switches
Global `webhookr.processors.enabled = false` ⇒ engine fully bypassed (BR-009).
Per-processor `webhookr.processors.<name>.enabled = false` ⇒ that processor is `DISABLED`
regardless of customer config.

#### BR-005 — Default enablement
- `PLATFORM` processors: **enabled by default** (per-processor flag default = true; no customer override).
- `PRODUCT` processors that change business behavior (filter, transform, routing, dedup, …):
  **disabled by default** until explicitly enabled by the customer (per-processor flag default =
  false; project/endpoint default = disabled). Upgrades never silently change behavior.

#### BR-006 — `required` removed
`EventProcessor` has **no** `required` field. Every processor in the effective pipeline is
stop-on-fail. Fail-open, if ever needed, is a distinct future concept (`failureMode: BLOCK | CONTINUE`).

#### BR-007 — Deterministic order
Registered processors have a fixed canonical order (ordered provider array). Resolution and
persistence preserve that order via `sequence`.

#### BR-008 — Halt + skip semantics
The first `stopped`/`failed` halts execution; every **enabled** processor after it is `SKIPPED`
(reason `HALTED_BY:<name>`). Disabled processors are `DISABLED`, independent of halts.

#### BR-009 — Global bypass
Global flag off ⇒ engine not invoked; no runs persisted; delivery proceeds; `processingStatus`
stays `PENDING`. Global flag on + only no-op registered ⇒ engine runs, persists a `PASSED` run,
`processingStatus = PROCESSED`, delivery proceeds.

#### BR-010 — Delivery gate
`delivery.events` is enqueued **iff** outcome is `passed`. `stopped`/`failed` never enqueue.
Disabling processors (DISABLED, empty pipeline) does **not** block delivery.

#### BR-011 — Replay bypasses the engine
Replay (`DeliveryService.replayEvent/replayDelivery`) re-sends an event that was **already
processed and `PROCESSED`** — it is a re-delivery of the same persisted, immutable payload, **not
a new ingest**. Since the payload is unchanged and the pipeline already ran and passed at ingest
time, there is nothing to re-process; replay delivers directly. (Re-running the pipeline on replay
would only be needed if replay could mutate the payload or if processor config could retroactively
block already-accepted events — neither is true in v1.)

#### BR-012 — Status separation
`Event.processingStatus ∈ {PENDING, PROCESSED, STOPPED, FAILED}` (overall outcome).
`EventProcessorRun.status ∈ {PASSED, STOPPED, FAILED, SKIPPED, DISABLED}` (per-processor). These
are distinct enums; the event never reuses processor status. Mapping: `passed→PROCESSED`,
`stopped→STOPPED`, `failed→FAILED`; initial `PENDING`.

#### BR-013 — Idempotent, cheap resolution
Config resolution is one indexed read per event (project + endpoint config for the endpoint).
Re-processing the same `eventId` replaces prior runs (keyed `(eventId, sequence)`) and re-sets
`processingStatus`; never duplicates rows or double-enqueues delivery.

#### BR-014 — No sensitive data in runs
`reasonCode` matches `^[A-Z0-9_:.\-]{1,64}$`; `message` ≤ 500 chars; neither embeds payload,
headers, or secrets. Run rows are unencrypted (must be timeline-queryable) — so they must carry
metadata only.

### Validation Rules
- `EventProcessorResult.status` returned by a processor ∈ `{passed, stopped, failed}` only
  (`skipped`/`disabled` are engine/resolver-assigned).
- `processorName` non-empty, unique within the registry (guard at startup — duplicates throw).
- `category` ∈ `{PLATFORM, PRODUCT}`.
- Config rows unique per `(projectId|endpointId, processorName)`.
- `reasonCode`/`message` per BR-014.

### Permissions
Engine runs in a background worker — no end-user context. The future config **write** surface
(out of scope) will enforce `JwtAuthGuard` + project/endpoint ownership (as `EventService.findOne`
does today). No permission change in this story.

## 4. Non-Functional Requirements

### Performance
In-memory execution; added I/O = one config read (indexed) + one batched write (status +
`createMany` runs) per event, in a single transaction. Target < 5 ms p95 added latency with the
no-op processor. No N+1: runs via `createMany`; config via one query per event (optionally a
short-TTL in-memory cache keyed by endpointId).

### Scalability
Runs within existing ingest-worker concurrency (`INGEST_CONCURRENCY`). No new queue/worker.
Run-row growth bounded per event (# registered processors); retention = OQ4.

### Availability
No new external dependency. Global flag off ⇒ path identical to today. A processor failure or a
resolution fault degrades a single event, never the worker.

### Security
- Engine operates on already-decrypted, in-memory payload/headers (same trust boundary as
  `IngestService` today). Sensitive inbound headers already redacted upstream.
- Run rows carry metadata only (BR-014) → stored unencrypted for query-ability; reviewed against
  the repo redaction list. Never log payload/headers from the engine.

### Observability
#### Logging
Structured Pino at the facade/engine boundaries using `stage=` convention:
`svc.event_processing.resolved|started|passed|stopped|failed|disabled`, with `eventId`,
`endpointId`, `projectId`, `processorName`, `category`, `decision`, `outcome`. No payload fields.

#### Metrics
Prometheus (`@willsoto/nestjs-prometheus`):
- `event_processing_outcomes_total{outcome}` (`passed|stopped|failed`).
- `event_processing_processor_results_total{processor,category,status}` (incl. `disabled`).
- `event_processing_duration_ms` (pipeline) + `event_processing_resolve_duration_ms` (resolver).
- **Alert:** `blocked_deliveries` (`stopped+failed`) must be 0 while only the no-op is registered.

#### Tracing
`runWithSpan('event-processing.run')` (facade) → `event-processing.resolve` (resolver) →
`event-processing.execute` (engine) → `event-processing.processor {processor}` per step, nested
under the existing `ingest.process` span.

## 5. Technical Design

### Impacted Repositories
- `webhookr-svc` — new `src/event-processing/` module; edits to `IngestService` (+ endpoint query
  now selects `projectId`); Prisma schema + migration; `.env.example`; `docs/` + ADR.
- `webhookr-artifacts` — engineering design record / ADR (Artifacts source-of-truth workflow).
- `gitops` / Unleash — register `webhookr.processors.enabled` and per-processor flags.
- `webhookr-ingest` — **no change** (contract boundary respected).

### Architecture
```
src/event-processing/
  event-processing.module.ts
  event-processing.service.ts        # FACADE: resolve → execute → persist → outcome (IngestService calls this)
  event-processor.engine.ts          # PURE orchestration of the given effective pipeline
  pipeline-resolver.service.ts       # 4-layer enablement → effective pipeline + disabled set (BR-003)
  event-processor.registry.ts        # ordered EventProcessor[] provider, with categories
  processor-config.repository.ts     # reads ProjectProcessorConfig / EndpointProcessorConfig
  event-processing.repository.ts     # persists Event.processingStatus + EventProcessorRun[]
  interfaces/
    event-processor.interface.ts
    event-processor-context.interface.ts
    event-processor-result.interface.ts
    processor-category.enum.ts
    resolved-pipeline.interface.ts
  processors/
    noop.processor.ts                # default PLATFORM processor, always `passed`
  *.spec.ts co-located per CLAUDE.md
```

Contracts:
```ts
enum ProcessorCategory { PLATFORM = 'PLATFORM', PRODUCT = 'PRODUCT' }

type ProcessorOutcome = 'passed' | 'stopped' | 'failed';           // what a processor may return
type ProcessorRunStatus = 'PASSED' | 'STOPPED' | 'FAILED' | 'SKIPPED' | 'DISABLED'; // persisted

interface EventProcessor {
  readonly name: string;
  readonly category: ProcessorCategory;      // no `required`
  execute(context: EventProcessorContext): Promise<EventProcessorResult>;
}

interface EventProcessorContext {
  readonly eventId: string;
  readonly endpointId: string;
  readonly projectId: string;
  readonly endpointSlug: string;
  readonly method?: string;
  readonly headers: Record<string, string | string[] | undefined>; // redacted, in-memory
  readonly payload: unknown;                                        // decrypted, in-memory
  readonly metadata: {
    readonly payloadSize: number; readonly headerCount: number;
    readonly sourceIp?: string; readonly userAgent?: string;
    readonly contentType?: string; readonly receivedAt: Date;
  };
  readonly attributes: Record<string, unknown>; // mutable bag for future enrich/transform
}

interface EventProcessorResult {
  processorName: string;
  status: ProcessorOutcome;
  reasonCode?: string;
  message?: string;
  context?: EventProcessorContext;
}

interface ResolvedProcessor {
  processor: EventProcessor;
  sequence: number;                 // canonical order
  enabled: boolean;
  disabledReason?: 'PLATFORM_DISABLED' | 'CONFIG_DISABLED' | 'DEFAULT_DISABLED';
}

interface EventProcessorEngineResult {
  outcome: ProcessorOutcome;        // 'passed' unless a run stopped/failed
  results: EventProcessorResult[];  // executed processors, in order
  haltedBy?: string;
  context: EventProcessorContext;
}
```

Resolver (pure; the heart of Points 3–6):
```ts
resolve(registry, flags, projectCfg, endpointCfg): ResolvedProcessor[] {
  // for each processor in canonical order:
  //   if !flags(`webhookr.processors.${name}.enabled`, defaultFor(category)) -> DISABLED/PLATFORM_DISABLED
  //   else if category === PLATFORM -> enabled
  //   else /* PRODUCT */:
  //     enabled = endpointCfg[name] ?? projectCfg[name] ?? productDefault(name) // false by default (BR-005)
  //     enabled ? enabled : DISABLED/(config present ? CONFIG_DISABLED : DEFAULT_DISABLED)
}
```
(The global `webhookr.processors.enabled` gate is applied by the **facade** before calling the
resolver — BR-009.)

Integration point in `IngestService.process()`:
```
persist Event (processingStatus = PENDING)
if !flags('webhookr.processors.enabled'): enqueue delivery.events; return   // BR-009 bypass
result = eventProcessing.run(context)   // resolve → execute → persist runs + processingStatus
if result.outcome !== 'passed': return  // BR-010 gate
enqueue delivery.events                 // unchanged
```

### ADR
#### Decision
1. **Engine orchestration-only**; a separate **PipelineResolver** owns enablement; a
   **facade service** wires resolve→execute→persist (BR-001).
2. **Two categories** (`PLATFORM`/`PRODUCT`) with different default-enablement and configurability.
3. **Four-layer enablement** (global flag → per-processor flag → project → endpoint), endpoint
   overrides project.
4. **Separate status enums** for event vs processor run, adding `DISABLED`.
5. **Run engine synchronously in the ingest worker** (no new queue).
6. **Persist runs in a dedicated queryable table** (not a JSON blob) for the timeline.
7. **Remove `required`**; defer fail-open to a future `failureMode`.

#### Alternatives Considered
- **Enablement inside the engine.** Rejected: couples the engine to flags/config, breaks
  "closed for modification," makes processors non-plug-and-play.
- **Single status enum reused by event + run.** Rejected: conflates overall vs per-step; a future
  timeline is clearer with distinct vocabularies + `DISABLED`.
- **Keep `required` for fail-open.** Rejected: "required" (is it in the pipeline?) and "failure
  mode" (what happens when it fails?) are different concerns; conflating them limits both.
- **Config as JSON on Endpoint/Project.** Rejected: per-processor rows give clean uniqueness,
  indexing, and a natural place for future per-processor `config`.
- **New BullMQ processing queue.** Rejected: unneeded infra/latency; violates "no event-driven
  infra unless required."
- **JSON blob for runs.** Rejected: the next story is a paginated per-processor timeline.

#### Tradeoffs
- More moving parts (resolver, facade, config tables, two flag layers) vs. a genuinely small,
  stable engine and full operational + product control — chosen.
- A per-event config read vs. fine-grained control — chosen; mitigated (indexed / cacheable).
- Config tables with no writer yet vs. a stable resolution model from day one — chosen (read-path
  only; write surface deferred).

#### Consequences
- "Persisted ⇒ delivered" no longer holds; replay explicitly bypasses (BR-011).
- Faulty processors are disabled in prod via a flag without downtime.
- New processors are inert-by-default for customers (PRODUCT) → predictable upgrades.
- A queryable, fully-audited run model (incl. DISABLED reasons) exists for the timeline with no
  further migration.

### Data Model
Prisma (`webhookr-svc/prisma/schema.prisma`), all additive:
```prisma
enum EventProcessingStatus { PENDING PROCESSED STOPPED FAILED }
enum ProcessorRunStatus    { PASSED STOPPED FAILED SKIPPED DISABLED }

model Event {
  // ...existing...
  processingStatus EventProcessingStatus @default(PENDING)
  processorRuns    EventProcessorRun[]
  @@index([endpointId, processingStatus])
}

model EventProcessorRun {
  id            String             @id @default(uuid())
  eventId       String
  processorName String
  category      String             // PLATFORM | PRODUCT (denormalized for timeline)
  sequence      Int                // canonical order
  status        ProcessorRunStatus
  reasonCode    String?            // e.g. HALTED_BY:validation, PLATFORM_DISABLED, CONFIG_DISABLED
  message       String?            // metadata only; <= 500 chars; no payload/secrets
  durationMs    Int?
  createdAt     DateTime           @default(now())
  event         Event              @relation(fields: [eventId], references: [id], onDelete: Cascade)
  @@unique([eventId, sequence])    // idempotent replace-on-attempt
  @@index([eventId])
}

model ProjectProcessorConfig {
  id            String   @id @default(uuid())
  projectId     String
  processorName String
  enabled       Boolean
  config        Json?    // reserved (OQ5); v1 reads only `enabled`
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  project       Project  @relation(fields: [projectId], references: [id], onDelete: Cascade)
  @@unique([projectId, processorName])
  @@index([projectId])
}

model EndpointProcessorConfig {
  id            String   @id @default(uuid())
  endpointId    String
  processorName String
  enabled       Boolean
  config        Json?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  endpoint      Endpoint @relation(fields: [endpointId], references: [id], onDelete: Cascade)
  @@unique([endpointId, processorName])
  @@index([endpointId])
}
```
Add back-relation fields on `Project`/`Endpoint` (`processorConfigs`), and `processorRuns` on
`Event`. Backfill: existing events default `PENDING`; no historical runs; config tables start empty.

### API Contracts
N/A in this story — no HTTP surface added or changed. (Config write API + timeline read API are
later stories.)

| Method | Endpoint | Description |
|--------|----------|-------------|
| N/A | N/A | No API surface in this story |

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
`delivery.events` / `DeliveryEventV1` **unchanged**; still enqueued, now only on a passing
pipeline. `jobId = eventId` continues to dedup.
#### Consumed Events
Unchanged: `IngestProcessor` consumes `INGEST_EVENTS_QUEUE` / `IngestEventV1`.

### Message Contracts
```json
// delivery.events job payload (DeliveryEventV1) — UNCHANGED
{ "schemaVersion": 1, "eventId": "<uuid>" }
```

### External Dependencies
None new. Existing Prisma/Postgres, OpenFeature/Unleash, OTel, Prometheus, Pino.

### Feature Flags
- **Global (kill switch):** `webhookr.processors.enabled` — boolean, default **off** in prod at
  launch, on in non-prod. Off ⇒ engine fully bypassed (BR-009).
- **Per processor:** `webhookr.processors.<processor-name>.enabled` — boolean; default follows
  category (PLATFORM → true, PRODUCT → false, BR-005). Examples:
  `webhookr.processors.validation.enabled`, `...filtering.enabled`, `...transformation.enabled`,
  `...deduplication.enabled`, `...routing.enabled`.
- Resolved through `FeatureFlagsService.isEnabled(flag, categoryDefault)` (OpenFeature/Unleash).
  Document defaults in `.env.example` for local dev.

### Migrations
One additive Prisma migration: `EventProcessingStatus` + `ProcessorRunStatus` enums,
`Event.processingStatus` (default `PENDING`), `EventProcessorRun`, `ProjectProcessorConfig`,
`EndpointProcessorConfig`, indexes. No drops, no backfill. Apply **before** enabling the global flag.

### Rollback Strategy
- **Runtime (primary):** `webhookr.processors.enabled = false` → engine bypassed, delivery never
  gated. Instant, no deploy. Per-processor flags disable a single faulty processor without
  touching the rest.
- **Code:** revert the `IngestService` integration PR; module + schema stay dormant.
- **Schema:** additive migration is safe to leave in place; a down-migration exists if fully
  abandoning.

## 6. Implementation Strategy

### Recommended Implementation Order
1. **Schema + migration** (statuses, `EventProcessorRun`, config tables). Merge first.
2. **Contracts + engine + no-op + registry** (`src/event-processing/`), unit-tested pure.
3. **PipelineResolver + config repository** (four-layer enablement, category defaults), unit-tested.
4. **Persistence repository** (`processingStatus` + runs, `createMany`, replace-on-attempt).
5. **EventProcessingService facade** + **IngestService integration** behind the global flag, with
   delivery gating + tests (endpoint query now selects `projectId`).
6. **Observability** (spans/metrics/logs) — fold into #3/#5 or a thin follow-up.
7. **Docs + ADR + artifact reconciliation.**

### Dependencies
- #2–#6 depend on #1; #3 depends on #2; #5 depends on #2/#3/#4; flags exist in Unleash before #5
  is enabled anywhere non-dev.

### Breaking Changes
None. Additive schema; unchanged delivery contract; global flag default off.

### Backward Compatibility
Global flag off ⇒ behaviorally identical to today; existing events default `PENDING`; no consumer
of `delivery.events` changes.

### Rollout Strategy
1. Ship #1–#7 with global flag **off** in prod, **on** in non-prod.
2. Enable in staging; verify all outcomes `passed`, `blocked_deliveries = 0`, latency, runs
   (including a stub PRODUCT processor to see `DISABLED` by default → then enable via config).
3. Enable global flag in prod. Real processors land later, each gated by its own per-processor
   flag (default off for PRODUCT) and customer config.

## 7. Task Split

### Infrastructure
**Repositories:** `gitops` / Unleash, `webhookr-svc`
**Tasks:**
- [ ] Register `webhookr.processors.enabled` (off prod / on non-prod).
- [ ] Register per-processor flag convention `webhookr.processors.<name>.enabled` (+ the no-op if flagged).
- [ ] Document flags + category defaults in `.env.example`.
- [ ] Ensure the migration is applied before global flag enable.

### Backend
**Repositories:** `webhookr-svc`
**Tasks:**
- [ ] Prisma: statuses, `EventProcessorRun`, `ProjectProcessorConfig`, `EndpointProcessorConfig`,
      indexes; generate migration.
- [ ] Contracts: `EventProcessor` (category, **no `required`**), context, result, engine result,
      `ProcessorCategory`, `ResolvedProcessor`.
- [ ] `EventProcessorEngine.execute` (order, halt, skip, throw→failed) — pure.
- [ ] `EventProcessorRegistry` (ordered, with categories) + startup uniqueness guard.
- [ ] `NoopProcessor` (PLATFORM, always `passed`).
- [ ] `ProcessorConfigRepository` (read project + endpoint config for an endpoint).
- [ ] `PipelineResolver` (four-layer enablement, category defaults, disabled reasons).
- [ ] `EventProcessingRepository` (status + `createMany` runs, one tx, replace-on-attempt).
- [ ] `EventProcessingService` facade (global-flag gate → resolve → execute → merge → persist → outcome).
- [ ] Integrate into `IngestService.process`; select `projectId` on the endpoint query; gate
      `deliveryQueue.add`.
- [ ] Spans, Prometheus counters/histograms, structured logs.

### API
**Repositories:** N/A — no API surface this story (config write + timeline read are later stories).

### Frontend
**Repositories:** N/A — no UI this story.

### Documentation
**Repositories:** `webhookr-svc`, `webhookr-artifacts`
**Tasks:**
- [ ] `webhookr-svc/docs/` subsystem doc: engine, resolver, categories, enablement layers,
      status model, extension steps.
- [ ] ADR (7 decisions above) under `webhookr-svc/docs/adr`.
- [ ] Reconcile into `webhookr-artifacts/engineering` per the source-of-truth workflow.

## 8. Testing Strategy

### Unit Tests
Co-located `*.spec.ts`, boundary mocks (Prisma, flags, OTel), no `any`. Cover all AC + new logic:
- **Engine:** all pass → `passed`; one `stopped` → downstream `SKIPPED`, delivery blocked; one
  `failed` (returned + thrown) → downstream `SKIPPED`, delivery blocked, worker does not rethrow;
  deterministic order; `haltedBy`.
- **Resolver (matrix):** global off → bypass; per-processor platform flag off → `DISABLED`
  (`PLATFORM_DISABLED`); PLATFORM ignores customer config; PRODUCT default disabled →
  `DEFAULT_DISABLED`; project enable; endpoint overrides project (both directions);
  `CONFIG_DISABLED`.
- **Facade:** merges executed + disabled into ordered runs; maps outcome→`processingStatus`;
  empty effective pipeline → `passed`/delivery allowed (BR-010); idempotent replace-on-attempt
  (BR-013); persistence failure rethrows (EF-02).
- **IngestService:** global off → engine skipped, delivery enqueued (parity); on + passing →
  enqueued; on + stopped/failed → not enqueued; endpoint query includes `projectId`.

### Integration Tests
`test/` e2e with real Postgres + Redis: drive an ingest job through with the no-op registry →
assert `processingStatus = PROCESSED`, a `delivery.events` job enqueued, one `PASSED` run row.
Add a stub stopping PRODUCT processor enabled via an `EndpointProcessorConfig` row → assert
delivery **not** enqueued, `STOPPED`, correct run rows. Add a disabled stub → assert `DISABLED`
row + delivery proceeds.

### End-to-End Tests
Extend the ingest→delivery e2e: a stubbed stopping processor produces **no** delivery row; a
disabled processor does not affect delivery.

### Manual Validation
Staging with global flag on: send a webhook, confirm `PROCESSED`, ordered runs, delivery happens,
spans/metrics in Grafana/Tempo, `blocked_deliveries = 0`. Register a stub PRODUCT processor:
confirm it's `DISABLED` by default, enable via config row, confirm it runs; flip its platform
flag off, confirm `PLATFORM_DISABLED`.

## 9. Definition of Ready
- [x] OQ1 (future `failureMode`) deferred explicitly — does not block.
- [x] OQ2 (replay re-processing) confirmed — replay = bypass (BR-011).
- [x] OQ3 (per-processor default classification) deferred to the processor stories — does not block.
- [x] OQ4 (run retention) deferred to the timeline story — does not block.
- [x] OQ5 (per-processor `config Json`) decided — column created, v1 reads only `enabled`.
- [x] Scope confirmed: config **schema + read-path** in; config **write surface** deferred (§13).
- [x] Flag names + Unleash taxonomy agreed (`webhookr.processors.enabled` + `webhookr.processors.<name>.enabled`).
- [ ] Owner assigned (admin — default: thiagojv).
- [ ] Spec passes `/gatekeeper`.

## 10. Definition of Done
- [ ] Engine, registry, resolver, facade, no-op processor exist under `src/event-processing/`,
      unit-tested to the repo bar; engine core has **no** enablement/config knowledge (BR-001).
- [ ] `IngestService.process` calls the facade before delivery, behind the global flag; endpoint
      query selects `projectId`.
- [ ] Delivery enqueued **only** on `passed`; disabling/empty pipeline never blocks delivery.
- [ ] `Event.processingStatus` + full ordered `EventProcessorRun[]` (incl. `DISABLED`) persisted
      per attempt, idempotently; migration merged/applied.
- [ ] Adding a processor requires **no** `EventProcessorEngine` change (demonstrated by no-op +
      a stub registering with zero engine edits).
- [ ] Four-layer enablement + category defaults implemented and matrix-tested; a faulty processor
      can be disabled in prod via `webhookr.processors.<name>.enabled` with no deploy.
- [ ] Spans/metrics/logs emitted; `blocked_deliveries` alert wired.
- [ ] Global + per-processor flags registered; defaults per category.
- [ ] Docs + ADR written; `webhookr-artifacts` reconciled.
- [ ] All PRs pass ESLint, TS build, unit + e2e; each PR single-concern.

## 11. Estimate

| Size | Selected |
|------|:--------:|
| P    | ☐        |
| M    | ☐        |
| G    | ☐        |
| GG   | ☑        |

Rationale: one service, but the surface is broad — schema/migration with three new tables +
two enums, a pure engine, a four-layer resolver with a category default matrix, a config
read repository, a facade, hot-path integration with delivery gating, two flag layers, and
exhaustive matrix tests. ~6–7 reviewable PRs. **De-scope lever:** deferring
`ProjectProcessorConfig`/`EndpointProcessorConfig` (resolver uses flags + category defaults only)
brings this back to **G**; the config tables can land with the first PRODUCT processor.

## 12. References
- `webhookr-svc/src/ingest/ingest.service.ts` — integration point (persist → resolve → enqueue).
- `webhookr-svc/src/ingest/ingest.processor.ts` — worker; `UnrecoverableError` precedent (EF-01).
- `webhookr-svc/src/delivery/delivery.service.ts` — delivery + replay (BR-011).
- `webhookr-svc/src/contract/delivery-event.contract.ts` — unchanged handoff contract.
- `webhookr-svc/src/feature-flags/feature-flags.service.ts` — flag mechanism.
- `webhookr-svc/prisma/schema.prisma` — `Project`, `Endpoint`, `Event`, `Delivery`.
- `webhookr-svc/CLAUDE.md` — architecture/testing/over-engineering constraints; Artifacts workflow.
- Follow-up: Processor Execution Timeline (reads `EventProcessorRun`); Processor Config Management.

## 13. Future Improvements
- **Processor Config Management (write surface)** — API/BFF/UI to write
  `ProjectProcessorConfig`/`EndpointProcessorConfig` (this story ships schema + read-path only).
  Tracked as a dedicated follow-up story.
- Real processors: normalization/security-validation (PLATFORM); validation/filtering/transformation/
  dedup/routing/enrichment (PRODUCT).
- Per-processor `config Json` interpretation (OQ5) once processors need settings.
- Per-processor `failureMode: BLOCK | CONTINUE` fail-open (OQ1).
- Short-TTL cache for config resolution if the per-event read becomes hot.
- Per-processor timeout/circuit-breaking for processors that do external I/O.
- Retention/pruning for `EventProcessorRun` (OQ4).
- Revisit async processing worker if processors become long-running.

## 14. Appendix
Resolved-pipeline flow (with categories, flags, config, and the delivery gate):
```
INGEST_EVENTS_QUEUE → IngestProcessor → IngestService.process
  → open sealed payload / resolve endpoint (+projectId)
  → persist Event (processingStatus = PENDING)
  → if !webhookr.processors.enabled → enqueue delivery.events; return           (BR-009 bypass)
  → EventProcessingService.run(context):
       Registry.ordered()            → all processors (canonical order, category)
       PipelineResolver.resolve()    → effective pipeline (enabled) + disabled set
            per processor: global flag ∧ per-processor flag ∧ project ∧ endpoint  (BR-003)
            endpoint overrides project overrides platform default
       EventProcessorEngine.execute(effectivePipeline)
            run in order; halt on stopped/failed; downstream enabled → SKIPPED
       persist processingStatus + EventProcessorRun[] (executed + DISABLED), 1 tx (replace-on-attempt)
  → if outcome == passed → enqueue delivery.events                              (BR-010 gate)
→ DeliveryProcessor → DeliveryService.deliverEvent (unchanged)

Layers          Engine sees only the effective pipeline:
  Registry      → which processors exist
  Feature flags → platform availability (global + per-processor)
  Config        → project/endpoint availability (PRODUCT only)
  Engine        → executes the resulting ordered pipeline, nothing else
```
Note: domain "EventProcessor" is deliberately distinct from BullMQ "@Processor" workers
(`IngestProcessor`, `DeliveryProcessor`). New code lives under `src/event-processing/` and never
reuses "processor" for a worker.
