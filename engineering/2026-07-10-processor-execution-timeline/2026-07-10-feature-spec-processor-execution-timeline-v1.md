---
title: Processor Execution Timeline per Event (+ Force Send)
status: Draft
owner: thiagojv@gmail.com
created: 2026-07-10
updated: 2026-07-10
estimate: M
priority: High
labels: [event-processing, observability, developer-experience, force-send, webhookr-svc, webhookr-bff, webhookr-web]
---

# Feature Specification

This document describes everything required to implement a feature, from business motivation to production rollout.

## 1. Context

### Background
Webhookr already runs a deterministic, ordered **event-processing engine** (`webhookr-svc/src/event-processing`, shipped 2026-07-06). For every ingested event, the engine resolves the effective pipeline (platform + product processors, per project/endpoint enablement), executes each processor in sequence, halts on the first non-`passed` result, and marks the remaining processors as `skipped`. Delivery is only enqueued when the overall outcome is `passed` (`webhookr-svc/src/ingest/ingest.service.ts:123`).

The engine **already persists** a per-event run trail in the `EventProcessorRun` table (`processorName`, `category`, `sequence`, `status`, `reasonCode`, `message`, `durationMs`, `createdAt`) and **already emits** metrics (`event_processing_outcomes_total{outcome}`, `event_processing_processor_results_total{processor,category,status}`, `event_processing_duration_ms`).

What is **missing** today:
1. The persisted trail is **not exposed** anywhere — the event-detail REST endpoint (`webhookr-svc/src/event/event.controller.ts`), the BFF GraphQL `Event` type (`webhookr-bff/src/events/models/event.model.ts`), and the web/mobile event-detail screens do not surface it. Developers cannot see *why* an event was delivered, stopped, failed, or skipped.
2. There is **no start/end timestamp** per processor run — only `durationMs` and a row `createdAt`. Acceptance criteria require explicit `startTime` and `endTime`.
3. The timeline is **not immutable**: `EventProcessingRepository.persist()` does `deleteMany({ eventId })` + `createMany(...)` inside a transaction (`webhookr-svc/src/event-processing/event-processing.repository.ts`), so any re-run of the engine for the same event **overwrites** the previous timeline.

### Problem Statement
When an event is not delivered (or is delivered unexpectedly), developers have no self-serve way to understand the pipeline decision. They cannot see which processor stopped or failed the pipeline, why (reason code / message), in what order processors ran, or how long each took. This forces log spelunking or engineering escalations and undermines trust in the delivery guarantee.

### Objective
After this feature ships, every event exposes an **immutable, ordered processor execution timeline** — processor name, status, start time, end time, duration, reason code, and message — visible in the event-detail UI (web first, mobile follow-up) and via the API, with metrics exposing processor stop/failure counts. Delivery remains gated on all required processors passing, and a user can **force send** a blocked event via an audited, flagged override.

### Business Value
- **Developer self-service & trust:** turns "why wasn't this delivered?" from a support ticket into a 5-second UI read, directly supporting the reliability/transparency positioning (see [[webhookr-gtm-strategy]] — note the no-reliability-claims rule; this feature is *transparency*, not an uptime claim).
- **Unblocks stuck events:** force send gives users a way out of a pipeline block instead of a permanent dead-end (today a blocked event can never be delivered), reducing lost-webhook incidents.
- **Reduced support load:** fewer escalations about "silent" non-deliveries and blocked events.
- **Foundation for future processors:** an inspectable, immutable audit trail is a prerequisite for shipping more product-tier processors (transforms, filters, enrichers) with confidence.

### Success Metrics
- **Adoption:** ≥ 60% of events opened in event-detail view have the timeline panel rendered (front-end analytics event) within 30 days of GA.
- **Support deflection:** measurable drop in "why wasn't my event delivered" support/Discord threads (baseline vs. 30 days post-GA). `Assumption:` baseline is tracked qualitatively today; treat as directional.
- **Operational visibility:** `event_processing_processor_results_total{status="STOPPED"|"FAILED"}` and a derived stop/failure rate are graphed on the existing Grafana Cloud event-processing dashboard, alertable.
- **Correctness:** 0 observed timeline overwrites in production after immutability guarantee ships (assert via absence of duplicate-sequence write attempts / update attempts in logs).
- **Force-send usage & safety:** `delivery_forced_total` is tracked; force send resolves stuck events (qualitative) while staying rare — a sustained high forced rate flags a misconfigured processor to investigate, not normal operation.

## 2. Scope

### In Scope
- Add `startedAt` and `endedAt` (UTC timestamps) to each processor run; engine captures wall-clock start/end.
- Enforce **timeline immutability**: write-once, no delete+recreate; reprocessing/replay must not mutate an existing timeline.
- Expose the timeline (ordered by `sequence`) in the `webhookr-svc` event-detail REST response.
- Surface the timeline through the `webhookr-bff` GraphQL `Event` type as an ordered list.
- Render the timeline in the `webhookr-web` event-detail screen, in execution order, with per-processor status, timing, reason code, and message.
- Confirm & document the delivery gate: delivery is attempted only when all required processors pass.
- Ensure metrics expose processor stop/failure counts (extend/label existing counters as needed; add a Grafana panel).
- Backfill/compat handling for events that already have runs without `startedAt`/`endedAt`.
- **Force send (manual delivery override):** a user-initiated action to deliver an event that was blocked by the pipeline (`STOPPED`/`FAILED`), bypassing the delivery gate; the resulting `Delivery` is flagged as forced and the action is audited. The timeline stays immutable (still shows the block). This closes a real dead-end: today `DeliveryService.assertReplayable()` throws `ConflictException` for blocked events, so there is no path to deliver them.

### Out of Scope
- **Mobile event-detail UI** (`webhookr-mobile`) — tracked as a fast-follow; Expo drift ([[claudemd-artifacts-source-of-truth]], mobile #19) makes it a separate PR/effort. Contract is designed mobile-ready.
- Introducing new processor *types* or a "required vs optional" processor configuration model beyond what exists (see Open Questions / BR-006).
- Editing, re-running, or annotating a timeline from the UI.
- **Reprocessing the pipeline** on replay or force send — resolved (OQ-2): replay/force send re-attempt *delivery* only; they never re-run processors or regenerate the timeline.
- A first-class **"optional / non-blocking" processor** configuration — resolved (OQ-1): all enabled processors remain required. The only escape hatch for a blocked event is the manual force send. (Kept on the roadmap for when an enrichment/side-effect processor exists whose failure genuinely should not block.)
- Storing full per-processor input/output payload snapshots (privacy + storage cost); we preserve the *original event payload context* (already on `Event.payload`) and link runs to it, not per-step diffs.
- Long-term retention/archival policy changes for `EventProcessorRun` (retention resolved to follow the parent `Event`; a dedicated policy is deferred).

### Assumptions
- `Assumption:` The four acceptance-criteria statuses map to the existing `ProcessorRunStatus` enum: `passed→PASSED`, `stopped→STOPPED`, `failed→FAILED`, `skipped→SKIPPED`. The existing extra value `DISABLED` (processor configured off) is retained and rendered as an informational state, not one of the four "execution" statuses.
- **Decided (OQ-1):** "All required processors pass" == every *enabled* processor passes. The pipeline halts on the first non-`passed` result, so automatic delivery only occurs on outcome `passed`. No optional/non-blocking processor concept is introduced. The manual **force send** is the only override.
- `Assumption:` "Timeline preserves the original event payload context" is satisfied by the existing immutable `Event.payload`/`Event.headers` columns plus the FK from `EventProcessorRun.eventId`; processors mutating in-memory context do not alter stored event data (verified: `engine.execute` mutates only in-memory `currentContext`).
- **Decided (OQ-2):** Replay and force send re-attempt **delivery only** and never re-run the processing engine; therefore the timeline is written exactly once at ingest and stays immutable. (Verified in code: `DeliveryService.replayEvent`/`replayDelivery` call `deliverEvent`, not `EventProcessingService.run`.)
- **Decided (OQ-4):** Timeline retention follows the parent `Event` lifecycle (`onDelete: Cascade`); a dedicated retention policy is deferred.
- `Assumption:` Force send requires the same authorization as replay/event-read (project owner/member); no new role.
- `Assumption:` Web is the primary surface for GA; BFF and svc are same-team owned, enabling a coordinated contract change.

### Risks
- **R1 — Immutability vs. reprocessing:** current `persist()` deletes+recreates. If any code path re-invokes `EventProcessingService.run()` for an existing event (future replay-with-reprocess), it silently overwrites history. Mitigation: make persist write-once and guard against second write (BR-005, ADR).
- **R2 — Backfill gap:** pre-existing `EventProcessorRun` rows have no `startedAt`/`endedAt`. Nullable columns + UI "—" fallback avoids a data migration; risk is minor UI inconsistency for historical events.
- **R3 — Contract drift across 3 repos** (svc → bff → web) must ship in a backward-compatible order (additive fields) to avoid breaking existing event-detail reads.
- **R4 — PII exposure (scoped to `message`):** the raw payload panel and event headers are intentionally shown to the owner; delivery request headers are already redacted (`redactHeaders` in `webhookr-svc/src/delivery/delivery.service.ts` covers `authorization`, `cookie`, `x-api-key`, etc.). The *only* residual risk is the free-text processor `message` (from `result.message`/`error.message`) echoing raw payload or secret values. Mitigation: BR-010 (message = reason description, not payload echo) + existing 500-char clamp. Align with [[webhookr-security-matrix]].
- **R5 — Timezone/clock:** `startedAt`/`endedAt` from `performance.now()` deltas vs. wall clock. Must persist real UTC `Date`s, not monotonic clock values.
- **R6 — Force send bypasses the safety gate:** a forced delivery sends an event a processor deliberately stopped (e.g. a filter or a signature check). Mitigation: explicit confirm step in UI, `trigger=FORCED` audit on `Delivery`, forced-delivery metric, and clear surfacing that the timeline still shows the block. `Assumption:` no processor is *security-critical to bypass-prohibit*; if one is (e.g. signature verification), a future per-processor "non-overridable" flag may be needed (Future Improvements).

### Open Questions
_All blocking questions resolved 2026-07-10 (see Assumptions). Remaining items are non-blocking follow-ups:_
- **OQ-5 (non-blocking):** Should force send be prohibited for specific security-critical processors (e.g. signature verification), i.e. a "non-overridable block"? `TODO:` product/security call; default = force send overrides any block. Tracked in Future Improvements, does not block this feature.

## 3. Functional Design

### Main Flow
1. Event is ingested and persisted (`Event` row created with original `payload`/`headers`).
2. If the `processors-enabled` flag is on, `EventProcessingService.run(context)` executes.
3. Engine resolves the effective, ordered pipeline (enabled processors only; disabled → `DISABLED` record).
4. For each enabled processor, in `sequence` order, the engine records `startedAt` (UTC), runs `execute()`, records `endedAt` (UTC) and `durationMs = endedAt - startedAt`, and captures `status`, `reasonCode`, `message`.
5. On the first non-`passed` result (`stopped`/`failed`, or a thrown error → `failed` with reason `PROCESSOR_ERROR`), the engine stops executing further processors and records every subsequent processor as `skipped` with reason `PIPELINE_HALTED` (no `startedAt`/`endedAt`).
6. The full ordered run set + overall `processingStatus` is persisted **once**, immutably, in a single transaction.
7. Metrics are recorded (outcome, per-processor result incl. stop/failure, duration).
8. If outcome is `passed`, delivery is enqueued; otherwise delivery is skipped and a `processing_blocked` log is emitted.
9. Later, a user opens the event detail; svc returns the event **with** its ordered `processorRuns`; BFF maps it to GraphQL; web renders the timeline in execution order.
10. If the event was blocked and the user chooses **Force send**, the delivery is dispatched with `trigger = FORCED` (bypassing the gate), audited, and counted; the timeline is unchanged.

### Alternative Flows
#### AF-01 — Processors flag disabled
If `processors-enabled` is off for the environment, no engine run occurs; event has an **empty timeline** and proceeds to delivery (current behavior preserved). UI renders an empty-state ("No processors ran for this event").

#### AF-02 — Pipeline halted (stopped)
A processor returns `stopped`. Overall outcome = `STOPPED`; delivery is **not** attempted; timeline shows the stopping processor as `STOPPED` (with its reason/message) and all following as `SKIPPED`. Event `processingStatus = STOPPED`.

#### AF-03 — Processor failed (error or explicit failure)
A processor returns `failed` or throws. Overall outcome = `FAILED`; delivery not attempted; failing processor row carries `reasonCode` (`PROCESSOR_ERROR` for throws) and clamped `message`; following processors `SKIPPED`. Event `processingStatus = FAILED`.

#### AF-04 — Empty effective pipeline
No enabled processors. Outcome = `passed`, timeline may contain only `DISABLED` rows (or be empty); delivery proceeds. UI renders the disabled rows / empty-state.

#### AF-05 — Mixed enabled/disabled
Disabled processors appear in the timeline as `DISABLED` (with `disabledReason`), interleaved by `sequence`; they do not affect outcome.

#### AF-06 — Force send of a blocked event
User opens a `STOPPED`/`FAILED` event and triggers **Force send** (after an explicit confirm). The system dispatches delivery to the endpoint's destination(s) via the existing delivery path, bypassing the gate, and records each resulting `Delivery` with `trigger = FORCED`. The event's `processingStatus` and its timeline are **unchanged** (the block remains visible for audit). A forced-delivery metric is incremented and an audit log is written. UI then shows the forced delivery(ies) with a "Forced" badge.

#### AF-07 — Force send of a passed event
Allowed but redundant with normal replay; if invoked, it behaves like replay but records `trigger = FORCED`. `Assumption:` UI only surfaces the Force-send action for blocked events; passed events use the existing Replay action (`trigger = REPLAY`).

### Error Flows
#### EF-01 — Persist failure after engine run
If the persist transaction fails, the event is left in a consistent prior state (transaction rollback); the ingest job fails and is retried by BullMQ per existing retry policy. No partial timeline is written (atomic transaction). Must not double-write on retry (BR-005 / idempotency).

#### EF-02 — Timeline requested for event with no runs
Event-detail returns `processorRuns: []`. Not an error; UI empty-state.

#### EF-03 — Attempt to reprocess an event that already has a timeline
Write-once guard: the persist path detects an existing timeline for the event and **does not overwrite** it. Behavior: no-op + warning log (`svc.event_processing.timeline_immutable_skip`). (See BR-005; final semantics gated by OQ-2.)

#### EF-04 — Event not found / not owned by caller
Existing 404 behavior of the event-detail endpoint is unchanged; timeline inherits the same authorization as the parent event (BR-007).

#### EF-05 — Force send with no enabled destination
If the event's endpoint has no enabled destination, force send returns a 409/validation error ("no destination to deliver to") and writes no `Delivery`. Mirrors the existing delivery path behavior.

#### EF-06 — Concurrent/duplicate force send
Force send is user-initiated and may be pressed twice. It is **not** idempotent by default (each press creates a delivery attempt), matching the existing replay behavior. `Assumption:` UI disables the button while a request is in flight; a server-side guard is optional (see Open Questions is closed — treat as low risk).

### Business Rules
#### BR-001 — One timeline per event, ordered by sequence
Each `Event` has exactly one processor execution timeline, an ordered list of runs keyed by monotonically increasing `sequence` starting at the pipeline's defined order. `@@unique([eventId, sequence])` already enforces uniqueness.

#### BR-002 — Run record shape
Each run stores: `processorName`, `category`, `sequence`, `status`, `reasonCode?`, `message?` (≤ 500 chars), `startedAt?`, `endedAt?`, `durationMs?`. `startedAt`/`endedAt`/`durationMs` are present for **executed** processors (`PASSED`/`STOPPED`/`FAILED`) and null for `SKIPPED`/`DISABLED`.

#### BR-003 — Status domain
Execution statuses: `PASSED`, `STOPPED`, `FAILED`, `SKIPPED`. `DISABLED` is an additional non-execution state for configured-off processors (retained; not part of the four AC statuses).

#### BR-004 — Halt propagation
When a processor yields a non-`passed` execution status (or throws), no subsequent processor in the pipeline executes; each subsequent processor is recorded as `SKIPPED` with `reasonCode = PIPELINE_HALTED`. (Already implemented in the engine; timeline must reflect it.)

#### BR-005 — Immutability (write-once)
Once a timeline is persisted for an event, it is immutable: no updates, no delete+recreate. The persist operation must write the full run set exactly once. Any later processing attempt for the same event must not mutate the existing timeline (see EF-03). Replaces the current `deleteMany`+`createMany` pattern with an insert-once path guarded by the `@@unique([eventId, sequence])` constraint. Force send and replay never write to the timeline (BR-009).

#### BR-006 — Delivery gate
Automatic delivery is enqueued **only** when overall outcome is `passed`, i.e. all *required* processors passed. Per OQ-1, "required" == every *enabled* processor; any `STOPPED`/`FAILED` outcome blocks automatic delivery. The **only** way to deliver a blocked event is an explicit force send (BR-009).

#### BR-007 — Payload context preservation
The original event payload/headers are preserved unaltered on the `Event` record; the timeline references the event and never rewrites its payload. In-memory context transformations by processors are not persisted as event data in this feature.

#### BR-008 — Time semantics
`startedAt`/`endedAt` are real UTC timestamps captured around each `execute()`. `durationMs` is derived from a monotonic clock (`performance.now()`, existing behavior) to avoid wall-clock skew; the two need not be arithmetically identical and `durationMs` remains the source of truth for duration.

#### BR-009 — Force send (manual override)
An authorized user may force delivery of an event regardless of its `processingStatus`. Force send: (a) bypasses the delivery gate (BR-006); (b) dispatches via the existing delivery path to the endpoint's enabled destination(s); (c) records each resulting `Delivery` with `trigger = FORCED`; (d) does **not** alter the event's `processingStatus` or its timeline (BR-005); (e) is audited (who/when) and counted. It is the sole sanctioned escape hatch for a blocked event and replaces today's hard dead-end (`assertReplayable` throws for `STOPPED`/`FAILED`). Automatic deliveries record `AUTOMATIC`; replays record `REPLAY` (BR-011).

#### BR-011 — Delivery trigger provenance
Every `Delivery` carries a `trigger` (`AUTOMATIC` | `REPLAY` | `FORCED`) recording how it was initiated. The gate path sets `AUTOMATIC`, replay sets `REPLAY`, force send sets `FORCED`. This is provenance only — it does not change delivery behavior.

#### BR-010 — Processor message content
A processor `message` is a human-readable *reason* for the run outcome. It must not echo raw event payload or secret values. The 500-char clamp remains; message content is a review/guideline concern for processor authors. The raw payload is surfaced only in the dedicated (owner-authorized) payload panel, and delivery request headers are redacted by the existing `redactHeaders` allowlist.

### Validation Rules
- `sequence` ≥ 0, unique per event.
- `status` ∈ `{PASSED, STOPPED, FAILED, SKIPPED, DISABLED}`.
- `message` length ≤ 500 (clamped at write; already enforced).
- `endedAt` ≥ `startedAt` when both present.
- `durationMs` ≥ 0 when present.
- API: event-detail query is authorized against the caller's ownership of the project/endpoint/event (unchanged).

### Permissions
- Reading the timeline requires the same authorization as reading the parent event: an authenticated user (JWT) who owns / is a member of the project that owns the endpoint. No new roles introduced. `Assumption:` webhookr currently uses single-tenant-per-user project ownership (per existing `@CurrentUser` guard on `EventController`); timeline inherits it verbatim.
- No write API is exposed for timelines (system-generated only).
- **Force send** requires the same authorization as replay (project owner/member); it is the only write action introduced and it writes a `Delivery`, never the timeline.

## 4. Non-Functional Requirements

### Performance
- Timeline write adds two timestamp columns; negligible write overhead. Persist remains a single transaction.
- Event-detail read adds one indexed query (`EventProcessorRun` by `eventId`, existing `@@index([eventId])`) ordered by `sequence`. Target: event-detail P95 latency increase < 15 ms.
- Typical pipeline is small (single-digit processors); no pagination needed for the run list.

### Scalability
- Row growth is bounded by (events × processors); same order as today (feature adds columns, not rows). Retention follows parent event (`onDelete: Cascade`).

### Availability
- No new runtime service. Read path is a DB join already colocated in `webhookr-svc`. No new external dependency.

### Security
- No new secrets. Messages already clamped; enforce BR-010 (processor messages carry no raw payload/secret) — the one residual leak vector (R4).
- Timeline read authorization inherits event authorization; no broadening of data exposure.
- **Force send** is a privileged, gate-bypassing action: same authz as replay, explicit confirm, audited (`svc.delivery.forced` + `trigger = FORCED`), and metered (`delivery_forced_total`). It can send an event a security processor deliberately stopped — see R6/OQ-5 for the potential "non-overridable" processor follow-up.
- Aligns with least-privilege network posture ([[webhookr-security-matrix]]); no new egress (delivery uses the existing 443-only delivery path).

### Observability
#### Logging
- Existing: `svc.event_processing.<outcome> eventId=... haltedBy=...`.
- Add: `svc.event_processing.timeline_immutable_skip eventId=...` when a write is skipped due to an existing timeline (EF-03).
- Add (audit): `svc.delivery.forced eventId=... userId=... processingStatus=...` when a force send is executed (BR-009 audit trail — who/when).
- Ensure no payload/secret content in logs (structured, metadata-only — consistent with ingest logging conventions).

#### Metrics
- Existing (reused): `event_processing_processor_results_total{processor,category,status}` already carries `status` label including `STOPPED` and `FAILED` — **satisfies "expose processor stop/failure counts"**.
- Add Grafana Cloud panel(s): processor stop rate and failure rate by processor; overall `stopped`/`failed` outcome counts from `event_processing_outcomes_total{outcome}`.
- **New counter for force send:** `delivery_forced_total{endpoint?}` (or a `forced` label on the existing delivery counter) so overrides of the safety gate are observable and alertable (a spike in forced deliveries is a signal worth watching, R6).
- `Assumption:` No new counter is strictly required for stop/failure; a derived dashboard query over the existing labeled counter is sufficient. If a first-class convenience counter is desired, add `event_processing_processor_stops_total` / `_failures_total` (optional, low cost).

#### Tracing
- Existing spans (`event-processing.run`, `.resolve`, `.execute`, `.persist`) retained. Optionally annotate the `.execute` span with per-processor `startedAt`/`status` as span events (nice-to-have, not required).

## 5. Technical Design

### Impacted Repositories
- `webhookr-svc` — Prisma model + migration (`EventProcessorRun` timestamps, `Delivery.trigger` enum), engine timing capture, immutable persist, event-detail response includes `processorRuns`, **force-send endpoint + service**. **(primary)**
- `webhookr-bff` — GraphQL `Event` type gains `processorRuns: [ProcessorRun!]!`, `Delivery` gains `trigger`, **`forceDeliverEvent` mutation**; events service maps svc response. **(primary)**
- `webhookr-web` — event-detail screen renders the timeline + **Force-send action** (confirm dialog) and "Forced" delivery badge. **(primary)**
- `webhookr-mobile` — event-detail timeline. **(out of scope for GA; fast-follow)**
- `webhookr-docs` — document the timeline in the event/observability docs. **(docs)**
- `webhookr-artifacts` — this spec + implementation notes + (optional) ADR entry. **(docs)**

### Architecture
No new services. This is an **additive read-model exposure + persistence-semantics hardening** within the existing modular-monolith `webhookr-svc`, propagated through the BFF-for-frontend and web:

```
Ingest (BullMQ) ──▶ EventProcessingService.run()
                      ├─ PipelineResolver (enabled/disabled + sequence)
                      ├─ EventProcessorEngine.execute()  ── captures startedAt/endedAt per run
                      └─ EventProcessingRepository.persistOnce()  ── write-once, immutable
                                                  │
                          ┌───────────────────────┘
                          ▼
                   Postgres: Event 1──* EventProcessorRun (+ startedAt/endedAt)
                          ▲
   Read: GET /v1/.../events/:eventId  (EventService.findOne → includes ordered processorRuns)
                          ▲
   webhookr-bff:  Query event { ... processorRuns { ... } }  (WebhookrClientService REST → GraphQL map)
                          ▲
   webhookr-web:  Event detail → Timeline panel (ordered by sequence)
```

### ADR
#### Decision
1. **Extend the existing `EventProcessorRun` table** with nullable `startedAt`/`endedAt` (`timestamptz`) rather than introducing a separate `Timeline` aggregate. The table already *is* the timeline; a single ordered set per event with `@@unique([eventId, sequence])` is the natural model.
2. **Make persistence write-once (insert-only).** Replace `deleteMany + createMany` with an idempotent single insert of the full run set; rely on the unique constraint to reject a second write, and treat a pre-existing timeline as immutable (no overwrite).
3. **Expose the timeline as a nested read model** on the existing event-detail contract (svc REST → BFF GraphQL nested field), not as a separate endpoint/query — keeps one round-trip and matches how `deliveries` are already associated.
4. **Model force send via a `trigger` enum on `Delivery` (`AUTOMATIC`/`REPLAY`/`FORCED`), not a timeline mutation.** The override is recorded on the delivery it produces; the timeline stays immutable and the block stays auditable. Reuse the existing `DeliveryService` dispatch path; force send differs from replay only by (a) skipping the `assertReplayable` block guard and (b) recording `trigger = FORCED`.

#### Alternatives Considered
- **A. Separate `EventTimeline` / event-sourcing aggregate with versioned runs.** Supports reprocess-on-replay (append a new timeline version). Rejected for now: heavier model, no current requirement to reprocess (OQ-2); revisit if reprocess-on-replay is greenlit.
- **B. Keep `deleteMany+createMany` and enforce immutability only in application code.** Rejected: the delete path is exactly what threatens immutability; removing it is safer and simpler.
- **C. Separate `/events/:id/timeline` endpoint & GraphQL query.** Rejected: extra round-trip, no independent lifecycle; nested field is cheaper and matches `deliveries` precedent.
- **D. Derive start/end from `createdAt` + `durationMs`.** Rejected: `createdAt` is the row-write time (post-run, batched in one transaction), not the per-processor execution time — would be inaccurate.

#### Tradeoffs
- Nested read model keeps the API simple but couples timeline retrieval to event-detail (acceptable; that is the only consumer).
- Write-once removes the ability to re-run processors for an event without a schema/semantics change; acceptable given replay/force send = redelivery only (decided, OQ-2).
- Recording provenance on `Delivery` keeps immutability intact but means "was this event overridden?" is answered by inspecting its deliveries, not the event/timeline row. Acceptable; the UI joins them anyway.
- A `trigger` enum (chosen over a minimal `forced` boolean) adds a Prisma enum + migration but yields a single provenance field covering automatic/replay/forced — cleaner than a boolean and immediately useful for observability (replay vs automatic). Cost: the replay path must now pass its trigger too (small, one param).
- Two nullable timestamp columns add minor storage; avoids a data backfill.

#### Consequences
- `EventProcessingRepository.persist()` semantics change (no delete). Any future "reprocess" feature must consciously design timeline versioning (documented here as the extension path).
- `DeliveryService.assertReplayable` gains a force path that intentionally bypasses the block guard; that guard's hard dead-end for blocked events is removed in favor of an explicit, audited override.
- Historical rows have null `startedAt`/`endedAt`; consumers must tolerate nulls.
- One coordinated additive contract change across svc/bff/web.

### Data Model
`EventProcessorRun` (existing) — **add** two columns:

```prisma
model EventProcessorRun {
  id            String             @id @default(uuid())
  eventId       String
  processorName String
  category      String
  sequence      Int
  status        ProcessorRunStatus
  reasonCode    String?
  message       String?
  durationMs    Int?
  startedAt     DateTime?          // NEW: UTC wall-clock start of execute()
  endedAt       DateTime?          // NEW: UTC wall-clock end of execute()
  createdAt     DateTime           @default(now())
  event         Event              @relation(fields: [eventId], references: [id], onDelete: Cascade)

  @@unique([eventId, sequence])
  @@index([eventId])
}
```
`Delivery` (existing) — **add** the trigger discriminator:

```prisma
enum DeliveryTrigger {
  AUTOMATIC   // enqueued by the gate after outcome == passed
  REPLAY      // user re-sent an already-deliverable event/delivery
  FORCED      // user overrode a pipeline block (BR-009)
}

model Delivery {
  // ...existing fields...
  trigger DeliveryTrigger @default(AUTOMATIC)   // NEW: how this delivery was initiated
}
```
- No change to `EventProcessingStatus` / `ProcessorRunStatus` enums.
- No change to `Event` (payload context already preserved).
- Migration is additive (nullable timestamp columns on `EventProcessorRun` + `trigger` enum with default `AUTOMATIC` on `Delivery`) — no backfill required; existing deliveries default to `AUTOMATIC`.
- `Note:` chosen over a minimal `forced Boolean` (approved 2026-07-10): the enum also makes **replay** deliveries distinguishable from automatic ones (a free observability win), and models the three initiation modes on one field. The replay path (`replayEvent`/`replayDelivery`) sets `REPLAY`; force send sets `FORCED`; the gate path sets `AUTOMATIC`. Threaded via a `trigger` parameter on the shared `deliverToDestination`.

### API Contracts
#### Endpoints
One additive field on event-detail; one new force-send endpoint.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/projects/:projectId/endpoints/:endpointId/events/:eventId` | Event detail — now includes ordered `processorRuns` |
| POST | `/v1/projects/:projectId/endpoints/:endpointId/events/:eventId/force-deliver` | Force-send: deliver a (typically blocked) event, bypassing the gate; produces `forced` deliveries (BR-009). Returns 202/200; 409 if no enabled destination (EF-05). |

#### Request
```json
{}
```
_(GET, no body; path params + bearer JWT as today.)_

#### Response
```json
{
  "id": "evt_...",
  "projectId": "prj_...",
  "endpointId": "ep_...",
  "endpointSlug": "whr_...",
  "method": "POST",
  "receivedAt": "2026-07-10T12:00:00.000Z",
  "sourceIp": "203.0.113.10",
  "headers": { "content-type": "application/json" },
  "payload": { "order_id": 123 },
  "metadata": { "userAgent": "...", "contentType": "application/json" },
  "processingStatus": "STOPPED",
  "processorRuns": [
    {
      "processorName": "signature-verify",
      "category": "PLATFORM",
      "sequence": 0,
      "status": "PASSED",
      "reasonCode": null,
      "message": null,
      "startedAt": "2026-07-10T12:00:00.010Z",
      "endedAt": "2026-07-10T12:00:00.012Z",
      "durationMs": 2
    },
    {
      "processorName": "payload-filter",
      "category": "PRODUCT",
      "sequence": 1,
      "status": "STOPPED",
      "reasonCode": "FILTER_NO_MATCH",
      "message": "event type 'ping' not in allowlist",
      "startedAt": "2026-07-10T12:00:00.012Z",
      "endedAt": "2026-07-10T12:00:00.013Z",
      "durationMs": 1
    },
    {
      "processorName": "delivery-guard",
      "category": "PLATFORM",
      "sequence": 2,
      "status": "SKIPPED",
      "reasonCode": "PIPELINE_HALTED",
      "message": null,
      "startedAt": null,
      "endedAt": null,
      "durationMs": null
    }
  ]
}
```
Backward compatibility: `processorRuns` is an **additive** field; existing consumers ignore it. Existing `EventDetail` interface (`webhookr-svc/src/event/interfaces/event-detail.interface.ts`) is extended.

BFF GraphQL (additive):
```graphql
type ProcessorRun {
  processorName: String!
  category: String!
  sequence: Int!
  status: ProcessorRunStatus!   # PASSED | STOPPED | FAILED | SKIPPED | DISABLED
  reasonCode: String
  message: String
  startedAt: DateTime
  endedAt: DateTime
  durationMs: Int
}

type Event {
  # ...existing fields...
  processingStatus: EventProcessingStatus!
  processorRuns: [ProcessorRun!]!   # ordered by sequence asc
}

enum DeliveryTrigger { AUTOMATIC REPLAY FORCED }

type Delivery {
  # ...existing fields...
  trigger: DeliveryTrigger!         # NEW: AUTOMATIC | REPLAY | FORCED
}

type Mutation {
  # deliver a (typically blocked) event, bypassing the gate (BR-009)
  forceDeliverEvent(projectId: ID!, endpointId: ID!, eventId: ID!): ForceDeliverResult!
}
```

### Event Contracts
#### Published Events
N/A — no new domain/integration events published. Existing `DeliveryEventV1` on the delivery queue is unchanged.

#### Consumed Events
N/A — consumes the existing `IngestEventV1` flow indirectly (no contract change).

### Message Contracts
```json
{}
```
N/A — no queue message contract changes. Timeline is derived and persisted within `webhookr-svc`; not transported over a queue.

### External Dependencies
N/A — no new external dependencies. Uses existing Postgres (via Prisma), Prometheus/Grafana Cloud, and BullMQ flow.

### Feature Flags
- Reuses the existing `processors-enabled` flag (`PROCESSORS_ENABLED_FLAG`) that already gates engine execution.
- **New flag (front-end read exposure):** `event-timeline-ui-enabled` — gates rendering of the timeline panel in web (and later mobile) so the UI can ship dark and be enabled after the contract is verified end-to-end. Managed declaratively via [[webhookr-toggles]] (YAML + GH Actions). `Assumption:` a boolean toggle in the existing Unleash single-project setup is sufficient.
- **New flag (force send):** `event-force-send-enabled` — gates the force-send action (UI + endpoint) so this gate-bypassing capability can be enabled independently and killed instantly if abused. Also via [[webhookr-toggles]].

### Migrations
1. Prisma migration adding nullable `startedAt`/`endedAt` to `EventProcessorRun` (additive, no backfill; safe/online — nullable columns, no default backfill scan).
2. Prisma migration adding the `DeliveryTrigger` enum and `trigger DeliveryTrigger @default(AUTOMATIC)` to `Delivery` (additive; existing rows default to `AUTOMATIC` without a scan). `Note:` historical deliveries pre-date the replay/forced distinction, so `AUTOMATIC` is a safe (if slightly lossy) default for them.
3. No data migration for historical rows (nulls / default `AUTOMATIC` tolerated by contract & UI).

### Rollback Strategy
- **Schema:** additive nullable columns + `DeliveryTrigger` enum defaulting to `AUTOMATIC` — safe to leave in place on rollback; no down-migration needed. If reverting the persist change, the old `deleteMany+createMany` code can be restored without schema impact.
- **UI:** toggle `event-timeline-ui-enabled` off to hide the panel instantly (no deploy). Force-send action gated by its own flag `event-force-send-enabled` so it can be disabled independently without hiding the timeline.
- **Contract:** `processorRuns`, `Delivery.trigger`, and `forceDeliverEvent` are additive; reverting svc/bff to not emit/expose them does not break older/newer web (web tolerates absence).
- **Order of rollback:** disable force-send flag → disable timeline UI flag → revert web → revert bff → revert svc (reverse of rollout).

## 6. Implementation Strategy

### Recommended Implementation Order
1. **svc – schema + engine timing (write path):** add `startedAt`/`endedAt`, capture in engine, persist write-once (immutability). Ship behind existing `processors-enabled`; timeline now recorded with timestamps and immutable.
2. **svc – read path:** include ordered `processorRuns` in event-detail response + interface + swagger + tests.
3. **svc – force send:** `DeliveryTrigger` enum + `Delivery.trigger` column + migration; `forceDeliverEvent` service (reuse `DeliveryService` dispatch, skip block guard, record `trigger=FORCED`, audit log + metric) + REST endpoint; thread `trigger` through `deliverToDestination` (replay→`REPLAY`, gate→`AUTOMATIC`); tests.
4. **bff – GraphQL exposure:** `ProcessorRun` type + `Event.processorRuns`; `DeliveryTrigger` + `Delivery.trigger`; `forceDeliverEvent` mutation; map from svc.
5. **web – UI:** timeline panel behind `event-timeline-ui-enabled`; Force-send action + confirm dialog + "Forced" badge behind `event-force-send-enabled`.
6. **observability + docs:** Grafana panels (stop/failure + forced); docs update; artifact reconciliation.
7. **(fast-follow) mobile – UI (timeline + force send).**

### Dependencies
- Steps 2/3 (svc read + force send) → 4 (bff) → 5 (web) are sequential (contract flows svc→bff→web).
- Step 1 is independent and lands first (prerequisite for meaningful timestamps).
- Steps 2 and 3 are independent of each other and can be parallelized within svc.
- Grafana panels (6) depend only on emitted metrics (parallelizable once step 3 lands the forced metric).

### Breaking Changes
None. All API/GraphQL changes are additive (`processorRuns`, `Delivery.trigger`, `forceDeliverEvent`). The only *behavioral* changes are (a) persist becoming write-once (the intended immutability guarantee) and (b) `assertReplayable`'s hard block for blocked events being superseded by the explicit force-send path — both intentional.

### Backward Compatibility
- Historical `EventProcessorRun` rows: `startedAt`/`endedAt` null → UI shows "—".
- Older web against newer bff: ignores new field. Newer web against older bff (during rollout): UI flag stays off until bff ships; empty-state otherwise.

### Rollout Strategy
1. Ship svc write-path + force-send (steps 1–3) to dev; verify timelines recorded with timestamps + immutability (double-process → no overwrite), and force send of a blocked event produces a `forced` delivery without touching the timeline.
2. Ship bff (step 4) to dev; verify nested `processorRuns`, `Delivery.trigger`, and `forceDeliverEvent`.
3. Ship web (step 5) with both flags **off**; enable `event-timeline-ui-enabled` first (read-only, low risk), validate, then enable `event-force-send-enabled`; staged dev→prod ([[webhookr-toggles]] cutover discipline).
4. Enable in prod after end-to-end validation; monitor Grafana stop/failure + forced-delivery panels and audit logs.
5. Fast-follow mobile.

## 7. Task Split

### Infrastructure
**Repositories:** `webhookr-toggles`, `terraform` (if flag/alert wiring needs it)
**Tasks:**
- [ ] Add `event-timeline-ui-enabled` and `event-force-send-enabled` toggles (YAML) and roll dev→prod.
- [ ] Add Grafana Cloud dashboard panel: processor stop rate & failure rate by processor (from `event_processing_processor_results_total{status}`), plus stopped/failed outcome counts. (Optional) alert on failure-rate spike.
- [ ] Add Grafana panel + alert for `delivery_forced_total` (spike in forced deliveries).

### Backend
**Repositories:** `webhookr-svc`
**Tasks:**
- [ ] Prisma: add nullable `startedAt`/`endedAt` to `EventProcessorRun`; add `DeliveryTrigger` enum + `trigger @default(AUTOMATIC)` to `Delivery`; generate migration(s).
- [ ] Engine: capture `startedAt`/`endedAt` (UTC) around each `execute()`; thread through `EngineProcessorRun` + `ProcessorRunRecord`.
- [ ] Repository: replace `deleteMany+createMany` with write-once insert; add immutability guard + `timeline_immutable_skip` log (EF-03/BR-005).
- [ ] Confirm/annotate delivery gate (BR-006) with a test asserting non-`passed` outcomes never enqueue delivery.
- [ ] Force send: `forceDeliverEvent` service reusing `DeliveryService` dispatch, skipping the block guard, recording `trigger=FORCED`, emitting `svc.delivery.forced` audit log + `delivery_forced_total`; handle no-destination (EF-05). Thread the `trigger` param through `deliverToDestination` and set `REPLAY` on the existing replay paths.
- [ ] Regression test: replay/force send do not reprocess the pipeline nor mutate the timeline.
- [ ] Unit + repository tests for timing, halt→skip propagation, immutability, and `trigger` provenance (AUTOMATIC/REPLAY/FORCED).

### API
**Repositories:** `webhookr-svc`, `webhookr-bff`
**Tasks:**
- [ ] svc: extend `EventDetail` interface + `EventService.findOne` to include ordered `processorRuns`; update swagger response schema.
- [ ] svc: add `POST /v1/projects/:projectId/endpoints/:endpointId/events/:eventId/force-deliver` controller + swagger; authz via `@CurrentUser`.
- [ ] svc: event-detail e2e test asserts ordered runs + timestamps + skipped propagation; force-send e2e asserts blocked event delivers with `trigger=FORCED` and unchanged timeline/status.
- [ ] bff: add `ProcessorRun` type + `Event.processorRuns`; `DeliveryTrigger` enum + `Delivery.trigger`; `forceDeliverEvent` mutation; map svc REST → GraphQL in `EventsService`/deliveries service; unit tests (incl. null timestamps).

### Frontend
**Repositories:** `webhookr-web` (GA), `webhookr-mobile` (fast-follow)
**Tasks:**
- [ ] web: Timeline panel on event-detail, ordered by sequence, status badges (PASSED/STOPPED/FAILED/SKIPPED/DISABLED — DISABLED de-emphasized), start/end/duration, reason code + message; empty-state; behind `event-timeline-ui-enabled`.
- [ ] web: Force-send action on blocked events — confirm dialog explaining the pipeline block is being overridden; "Forced" badge on resulting deliveries; behind `event-force-send-enabled`.
- [ ] web: analytics event on timeline render (success metric).
- [ ] web: component/interaction tests + a11y (status conveyed beyond color — R4/UX).
- [ ] mobile (fast-follow): equivalent timeline view + force send.

### Documentation
**Repositories:** `webhookr-docs`, `webhookr-artifacts`
**Tasks:**
- [ ] docs: "Event processor timeline" page (statuses, reason codes, delivery gate).
- [ ] artifacts: implementation-notes-v1 + run `artifact-reconciliation`; link to [[claudemd-artifacts-source-of-truth]] before/after workflow.

## 8. Testing Strategy

### Unit Tests
- Engine: `startedAt`/`endedAt` captured for executed processors; null for skipped; `endedAt ≥ startedAt`; halt→all-following `SKIPPED` with `PIPELINE_HALTED`; throw→`FAILED`/`PROCESSOR_ERROR`; message clamp ≤ 500.
- Repository: write-once inserts full set; second persist attempt for same event does not overwrite (immutability) and logs skip.
- BFF mapping: svc payload (incl. null timestamps, `DISABLED` rows) → GraphQL `ProcessorRun[]` ordered by sequence.

- Force send: service produces `trigger=FORCED` delivery, does not mutate timeline/`processingStatus`, emits audit log + metric; no-destination → validation error. Replay paths record `trigger=REPLAY`; gate path records `AUTOMATIC`.

### Integration Tests
- svc event-detail (`processor-config.e2e`/event e2e): ingest → process (stopped scenario) → GET event returns ordered `processorRuns` with correct statuses, reason codes, timestamps; delivery not enqueued.
- Delivery gate: `passed` → delivery enqueued; `stopped`/`failed` → not enqueued.
- Force-send e2e: blocked event → `force-deliver` → delivery attempted with `trigger=FORCED`; timeline and `processingStatus` unchanged; second event-detail read shows same immutable timeline.
- Immutability: reprocess/replay path does not mutate timeline.

### End-to-End Tests
- web: open an event that was stopped → timeline panel shows PASSED → STOPPED → SKIPPED in order with timings and reason; flag-off hides panel.
- web: on the same blocked event, Force send (with confirm) → delivery appears with "Forced" badge; timeline still shows the block.

### Manual Validation
- Trigger a webhook that a processor stops; confirm in web UI the timeline explains the stop, delivery shows not-attempted, and Grafana stop counter increments.
- Trigger a processor error; confirm FAILED row with `PROCESSOR_ERROR` and following SKIPPED.
- Force send the blocked event; confirm it delivers, the delivery shows Forced, the timeline is unchanged, and `delivery_forced_total` increments.
- Open a historical event (pre-migration) → timestamps render "—" without error.
- Confirm processor messages contain no raw payload/secret material (BR-010).

## 9. Definition of Ready
- [x] OQ-1 resolved: all enabled processors are required; no optional-processor concept.
- [x] OQ-2 resolved: replay/force send do **not** reprocess; timeline stays immutable.
- [x] OQ-3/OQ-4 resolved: `DISABLED` shown de-emphasized; retention = parent `Event` lifecycle.
- [ ] OQ-5 (non-blocking) noted: no "non-overridable" processor for GA; force send overrides any block.
- [ ] Reason-code catalog for current processors documented (values surfaced in UI).
- [ ] `event-timeline-ui-enabled` and `event-force-send-enabled` toggles created in dev.
- [ ] UX agreed for timeline panel (status badges, a11y) **and** force-send confirm dialog + "Forced" badge.
- [ ] BR-010 confirmed with processor authors: `message` cannot leak payload/secrets.

## 10. Definition of Done
- [ ] `startedAt`/`endedAt` persisted for executed processors; migrations applied dev+prod.
- [ ] Timeline is write-once immutable; replay/force send proven non-mutating by test.
- [ ] Automatic delivery attempted only on `passed` (test-enforced).
- [ ] Force send delivers a blocked event, sets `Delivery.trigger=FORCED`, audits + meters, and leaves the timeline unchanged (test-enforced); replay/automatic record `REPLAY`/`AUTOMATIC`.
- [ ] Event-detail REST + BFF GraphQL expose ordered `processorRuns`; `Delivery.trigger` + `forceDeliverEvent` exposed (all additive, backward-compatible).
- [ ] web renders timeline in execution order and offers force send on blocked events, both behind flags; enabled in prod.
- [ ] Grafana panels show processor stop/failure counts and forced-delivery count.
- [ ] Unit/integration/e2e tests green; coverage for halt/skip, immutability, delivery gate, force send.
- [ ] Docs updated; artifacts reconciled ([[claudemd-artifacts-source-of-truth]]).
- [ ] No payload/secret leakage in messages/logs verified (BR-010).

## 11. Estimate

| Size | Selected |
|------|:--------:|
| P    | ☐        |
| M    | ☑        |
| G    | ☐        |
| GG   | ☐        |

**Rationale:** Core write-path (model, engine timing, persistence, delivery gate, metrics) is largely already implemented; the work is a focused additive column + immutability hardening + a 3-repo additive read-model exposure + one web panel, **plus the force-send override** (one column, one endpoint/mutation, reusing the existing delivery path, one web action). Still **M** — force send is small and reuses `DeliveryService` — but it sits at the top of the M band. Mobile and a first-class "optional processor" model remain excluded (would push to G).

## 12. References
- `webhookr-svc/src/event-processing/event-processor.engine.ts` — engine, halt/skip, message clamp.
- `webhookr-svc/src/event-processing/event-processing.service.ts` — orchestration, status mapping, metrics.
- `webhookr-svc/src/event-processing/event-processing.repository.ts` — persist (current delete+recreate; to become write-once).
- `webhookr-svc/src/ingest/ingest.service.ts:105-134` — delivery gate on outcome.
- `webhookr-svc/prisma/schema.prisma` — `Event`, `EventProcessorRun`, `ProcessorRunStatus`, `EventProcessingStatus`.
- `webhookr-svc/src/event/event.controller.ts` + `interfaces/event-detail.interface.ts` — event-detail contract.
- `webhookr-bff/src/events/models/event.model.ts` + `services/events.service.ts` — BFF Event + REST proxy.
- ADRs: `webhookr-artifacts/engineering/2026-07-06-event-processor-engine/` (0003), `.../2026-07-06-processor-config-management/` (0004).
- Related memory: [[webhookr-security-matrix]], [[webhookr-toggles]], [[webhookr-gtm-strategy]], [[claudemd-artifacts-source-of-truth]].

## 13. Future Improvements
- Versioned/append timelines to support reprocess-on-replay (ADR Alternative A) if the reprocess decision (OQ-2) is ever revisited.
- Per-processor input/output snapshots (privacy-gated) for deep debugging.
- Timeline export / share link for support workflows.
- First-class "optional (non-blocking) processor" configuration (OQ-1) with a per-processor `required` flag driving the delivery gate — the automatic counterpart to today's manual force send.
- **Non-overridable / security-critical processors (OQ-5):** allow marking a processor (e.g. signature verification) whose block force send cannot bypass.
- **Revisit `Delivery.requestHeaders` redaction vs. debuggability:** today the stored audit copy is redacted-then-encrypted (destination auth headers show `[REDACTED]` even to the owner). Delivery is unaffected (the outbound `fetch` uses raw headers); this is a deliberate defense-in-depth choice. If delivery debuggability matters, consider storing a non-reversible hint (hash/prefix) instead of a full mask. Out of scope here — separate task.
- Structured reason-code registry surfaced in UI with human-readable descriptions.
- Trace correlation: deep-link from a timeline run to its Tempo span.

## 14. Appendix

### Current vs. Target (gap map)
| Acceptance Criterion | Current State | Work Needed |
|---|---|---|
| Stores execution timeline | ✅ `EventProcessorRun` persisted | — |
| name/status/start/end/duration/reason/message | ⚠️ has all **except start/end** | Add `startedAt`/`endedAt` |
| Statuses passed/stopped/failed/skipped | ✅ enum (plus `DISABLED`) | Render mapping |
| Stop ⇒ following skipped | ✅ engine `PIPELINE_HALTED` | Surface in UI |
| Event detail shows timeline in order | ❌ not exposed | svc+bff+web read model |
| Delivery only when required pass | ✅ gate on `outcome==='passed'` | Confirm + test (all enabled = required) |
| Preserves original payload context | ✅ `Event.payload` immutable | — |
| Timeline immutable after execution | ❌ `deleteMany+createMany` overwrites | Write-once persist |
| Metrics expose stop/failure counts | ✅ `...processor_results_total{status}` | Grafana panel (+ optional counters) |
| **Force send blocked events (added)** | ❌ `assertReplayable` hard-blocks | `Delivery.trigger` (enum) + `forceDeliverEvent` + web action |

### Status legend (UI)
`PASSED` (green) · `STOPPED` (amber, intentional halt) · `FAILED` (red, error) · `SKIPPED` (grey, not run due to halt) · `DISABLED` (muted, configured off). Deliveries produced by force send carry a `Forced` badge.
