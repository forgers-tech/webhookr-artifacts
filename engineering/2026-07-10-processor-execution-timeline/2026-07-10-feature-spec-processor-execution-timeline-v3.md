---
title: Processor Execution Timeline per Event (+ Force Send)
status: Implemented
owner: thiagojv@gmail.com
created: 2026-07-10
updated: 2026-07-11
estimate: M
priority: High
labels: [event-processing, observability, developer-experience, force-send, webhookr-svc, webhookr-bff, webhookr-web]
---

# Feature Specification

> **As-built record (v3, supersedes v2).** Implemented and merged 2026-07-11 across webhookr-svc (#130), webhookr-bff (#77), webhookr-web (#88), webhookr-docs (#12), webhookr-artifacts (#18) and terraform (#101/#102); see [implementation-notes v1](2026-07-10-implementation-notes-v1.md). Two design details were refined during Copilot review and are corrected here versus v2: (1) the Redis in-flight lock TTL is **~300s, not ~60s** — the `PX` is only a crash-safety cap (the lock is released in `finally`), but it must comfortably exceed a realistic parallel fan-out of destination timeouts (default 30s each, configurable) so it cannot expire mid-delivery and let a concurrent request overlap; (2) **force send leaves the event entirely unchanged** — it does not bump `lastReplayedAt` (BR-009), since a forced delivery is not a replay; the override is audited via the `FORCED` delivery, the `svc.delivery.forced` log and the `delivery_forced_total` metric. Everything else is unchanged from the Gatekeeper-approved v2 (**98/100 ✅**).
>
> **Review record (v2, supersedes v1).** Gatekeeper: **98/100 — ✅ Approved** (2026-07-10; trajectory 91→95→96→98, no blockers/majors/open questions). Changes since v1 (Draft): idempotency short-circuit returns the persisted outcome (IR-1); `haltedBy` derived from the timeline for the non-overridable check (IR-2); Redis in-flight lock for force/replay dedup (IR-3); `processingStatus` exposed on the read contract (IR-2); non-overridable processor axis added, initial set **none** (OQ-5/BR-013); `trigger` enum over a `forced` boolean; "required processor" abstraction removed; reason-code catalog + force-send UX copy added; three toggles declared in `webhookr-toggles`; dedicated `webhookr.web.force-send` button gate (R-2). Ready for implementation.

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
After this feature ships, every event exposes an **immutable, ordered processor execution timeline** — processor name, status, start time, end time, duration, reason code, and message — visible in the event-detail UI (web first, mobile follow-up) and via the API, with metrics exposing processor stop/failure counts. Delivery remains gated on the pipeline passing (every enabled processor passes → outcome `passed`), and a user can **force send** a blocked event via an audited, flagged override.

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
- Expose the timeline (ordered by `sequence`) **and `processingStatus`** in the `webhookr-svc` event-detail REST response (the current contract exposes neither).
- **First-class idempotency & duplicate protection (BR-012):** an event-level guard so ingest-job retries never re-execute processors or overwrite the timeline, plus a server-side in-flight guard so duplicate force/replay submissions cannot spawn duplicate deliveries.
- **Non-overridable processors (BR-013):** a code-level `overridable` property so security-critical processors (e.g. signature verification) cannot be force-bypassed; force send is rejected (409) when such a processor blocked the event.
- Surface the timeline through the `webhookr-bff` GraphQL `Event` type as an ordered list.
- Render the timeline in the `webhookr-web` event-detail screen, in execution order, with per-processor status, timing, reason code, and message.
- Confirm & document the delivery gate: delivery is attempted only when every enabled processor passes (outcome `passed`).
- Ensure metrics expose processor stop/failure counts (extend/label existing counters as needed; add a Grafana panel).
- Backfill/compat handling for events that already have runs without `startedAt`/`endedAt`.
- **Force send (manual delivery override):** a user-initiated action to deliver an event that was blocked by the pipeline (`STOPPED`/`FAILED`), bypassing the delivery gate; the resulting `Delivery` is flagged as forced and the action is audited. The timeline stays immutable (still shows the block). This closes a real dead-end: today `DeliveryService.assertReplayable()` throws `ConflictException` for blocked events, so there is no path to deliver them.

### Out of Scope
- **Mobile event-detail UI** (`webhookr-mobile`) — tracked as a fast-follow; Expo drift ([[claudemd-artifacts-source-of-truth]], mobile #19) makes it a separate PR/effort. Contract is designed mobile-ready.
- Introducing new processor *types* or an optional/non-blocking processor tier — every enabled processor blocks on a non-`passed` result; there is no required/optional distinction to model (see BR-006).
- Editing, re-running, or annotating a timeline from the UI.
- **Reprocessing the pipeline** on replay or force send — resolved (OQ-2): replay/force send re-attempt *delivery* only; they never re-run processors or regenerate the timeline.
- An **optional / non-blocking processor tier** — out of scope (OQ-1): every enabled processor blocks on a non-`passed` result; the only escape hatch is the manual force send. (Roadmap: revisit only if an enrichment/side-effect processor appears whose failure genuinely should not block.)
- Storing full per-processor input/output payload snapshots (privacy + storage cost); we preserve the *original event payload context* (already on `Event.payload`) and link runs to it, not per-step diffs.
- Long-term retention/archival policy changes for `EventProcessorRun` (retention resolved to follow the parent `Event`; a dedicated policy is deferred).

### Assumptions
- `Assumption:` The four acceptance-criteria statuses map to the existing `ProcessorRunStatus` enum: `passed→PASSED`, `stopped→STOPPED`, `failed→FAILED`, `skipped→SKIPPED`. The existing extra value `DISABLED` (processor configured off) is retained and rendered as an informational state, not one of the four "execution" statuses.
- **Decided (OQ-1):** Automatic delivery occurs only on outcome `passed` — every enabled processor passed (the pipeline halts on the first non-`passed`). The acceptance criterion's "required processors" maps directly to "every enabled processor"; no separate required/optional tier is modeled. The manual **force send** is the only override.
- `Assumption:` "Timeline preserves the original event payload context" is satisfied by the existing immutable `Event.payload`/`Event.headers` columns plus the FK from `EventProcessorRun.eventId`; processors mutating in-memory context do not alter stored event data (verified: `engine.execute` mutates only in-memory `currentContext`).
- **Decided (OQ-2):** Replay and force send re-attempt **delivery only** and never re-run the processing engine; therefore the timeline is written exactly once at ingest and stays immutable. (Verified in code: `DeliveryService.replayEvent`/`replayDelivery` call `deliverEvent`, not `EventProcessingService.run`.)
- **Decided (OQ-4):** Timeline retention follows the parent `Event` lifecycle (`onDelete: Cascade`); a dedicated retention policy is deferred.
- **Decided (OQ-5):** Force send is **not** universally allowed — a processor can be marked non-overridable (`overridable=false`, code-level) and its block cannot be bypassed (BR-013). This replaces the earlier "future flag" stance; `overridable` is the meaningful safety axis (there is no required/optional axis).
- **Decided (IR-1):** The idempotency key is `eventId` + `processingStatus` — no separate `messageId` (redundant: one webhook = one `Event`). Guard is first-class (BR-012).
- `Assumption:` Force send requires the same authorization as replay/event-read (project owner/member); no new role.
- `Assumption:` Web is the primary surface for GA; BFF and svc are same-team owned, enabling a coordinated contract change.

### Risks
- **R1 — Immutability vs. reprocessing:** current `persist()` deletes+recreates. If any code path re-invokes `EventProcessingService.run()` for an existing event (future replay-with-reprocess), it silently overwrites history. Mitigation: make persist write-once and guard against second write (BR-005, ADR).
- **R2 — Backfill gap:** pre-existing `EventProcessorRun` rows have no `startedAt`/`endedAt`. Nullable columns + UI "—" fallback avoids a data migration; risk is minor UI inconsistency for historical events.
- **R3 — Contract drift across 3 repos** (svc → bff → web) must ship in a backward-compatible order (additive fields) to avoid breaking existing event-detail reads.
- **R4 — PII exposure (scoped to `message`):** the raw payload panel and event headers are intentionally shown to the owner; delivery request headers are already redacted (`redactHeaders` in `webhookr-svc/src/delivery/delivery.service.ts` covers `authorization`, `cookie`, `x-api-key`, etc.). The *only* residual risk is the free-text processor `message` (from `result.message`/`error.message`) echoing raw payload or secret values. Mitigation: BR-011 (message = reason description, not payload echo) + existing 500-char clamp. Align with [[webhookr-security-matrix]].
- **R5 — Timezone/clock:** `startedAt`/`endedAt` from `performance.now()` deltas vs. wall clock. Must persist real UTC `Date`s, not monotonic clock values.
- **R6 — Force send bypasses the safety gate:** a forced delivery sends an event a processor deliberately stopped (e.g. a filter). Mitigations: (a) security-critical processors are marked non-overridable and cannot be bypassed at all (BR-013 / EF-07); (b) for overridable blocks — explicit UI confirm, `trigger=FORCED` audit on `Delivery`, forced-delivery metric with alert, and the timeline still showing the block. Residual `Assumption:` the non-overridable set is correctly chosen in DoR (mis-marking a security processor as overridable is the real risk).
- **R7 — Idempotency regression:** switching persist from delete+recreate to write-once changes retry semantics. Failure modes: (a) a mishandled `P2002` → retry storm; (b) a `run()` short-circuit that returns nothing → **lost delivery** on the crash-between-persist-and-enqueue window (IR-1). Mitigation: BR-012 guard returns the persisted outcome; `P2002` caught as no-op; explicit crash-window + double-process tests.

### Open Questions
_All blocking questions resolved 2026-07-10 (see Assumptions). No open input remains:_
- **OQ-5 → resolved & closed:** force send is prohibited for non-overridable processors (BR-013). Initial non-overridable set = **none** (decided) — all current processors overridable; the mechanism ships ready for the first security-critical processor.

## 3. Functional Design

### Main Flow
1. Event is ingested and persisted (`Event` row created with original `payload`/`headers`).
2. If the `processors-enabled` flag is on, `EventProcessingService.run(context)` executes — **but first short-circuits if the event's `processingStatus` is already terminal, returning the persisted outcome without re-executing processors** (BR-012 idempotency guard; handles ingest-job retries and preserves the downstream delivery enqueue).
3. Engine resolves the effective, ordered pipeline (enabled processors only; disabled → `DISABLED` record).
4. For each enabled processor, in `sequence` order, the engine records `startedAt` (UTC), runs `execute()`, records `endedAt` (UTC) and `durationMs = endedAt - startedAt`, and captures `status`, `reasonCode`, `message`.
5. On the first non-`passed` result (`stopped`/`failed`, or a thrown error → `failed` with reason `PROCESSOR_ERROR`), the engine stops executing further processors and records every subsequent processor as `skipped` with reason `PIPELINE_HALTED` (no `startedAt`/`endedAt`).
6. The full ordered run set + overall `processingStatus` is persisted **once**, immutably, in a single transaction.
7. Metrics are recorded (outcome, per-processor result incl. stop/failure, duration).
8. If outcome is `passed`, delivery is enqueued; otherwise delivery is skipped and a `processing_blocked` log is emitted.
9. Later, a user opens the event detail; svc returns the event **with** its ordered `processorRuns`; BFF maps it to GraphQL; web renders the timeline in execution order.
10. If the event was blocked and the user chooses **Force send**: the service checks the halting processor is overridable (else 409, EF-07) and passes the duplicate-protection guard (BR-012), then dispatches delivery with `trigger = FORCED` (bypassing the gate), audited and counted; the timeline is unchanged.

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
If the persist transaction fails, the event is left in a consistent prior state (transaction rollback); the ingest job fails and is retried by BullMQ per existing retry policy. No partial timeline is written (atomic transaction). On retry, the BR-012 guard applies: if the event is already terminal, `run()` short-circuits (no re-execution, no double-write) **but returns the persisted outcome** so a delivery that had not yet been enqueued still gets enqueued (idempotently via `jobId=eventId`) — no lost delivery. The `@@unique([eventId, sequence])` `P2002` is caught as a no-op if a race occurs.

#### EF-02 — Timeline requested for event with no runs
Event-detail returns `processorRuns: []`. Not an error; UI empty-state.

#### EF-03 — Attempt to reprocess an event that already has a timeline
Two layers (BR-012): (1) `run()` short-circuits **before** re-executing processors when `processingStatus` is already terminal; (2) if a concurrent worker slips past the check, the write-once persist relies on `@@unique([eventId, sequence])` — the `P2002` is caught and emitted as a no-op warning log (`svc.event_processing.timeline_immutable_skip`), never propagated to the job. The timeline is never overwritten.

#### EF-04 — Event not found / not owned by caller
Existing 404 behavior of the event-detail endpoint is unchanged; timeline inherits the same authorization as the parent event (BR-007).

#### EF-05 — Force send with no enabled destination
If the event's endpoint has no enabled destination, force send returns a 409/validation error ("no destination to deliver to") and writes no `Delivery`. Mirrors the existing delivery path behavior.

#### EF-06 — Concurrent/duplicate force send or replay
A user may double-submit force send (or replay). Because these dispatch deliveries directly (bypassing the `jobId=eventId` queue dedup), the **server-side Redis in-flight lock (BR-012)** — `SET lock:delivery:<key> … NX PX ~300s`, key = `eventId` (event-level) or `eventId:deliveryId` (single-delivery replay), released in `finally` — makes the first request win and the concurrent second return `409 Conflict`, preventing duplicate deliveries. The disabled UI button is a convenience layer, not the guarantee.

#### EF-07 — Force send blocked by a non-overridable processor
If the halting processor (the timeline's `STOPPED`/`FAILED` run, resolved in the registry per BR-013) is non-overridable, force send is rejected with `409 Conflict` ("event blocked by a non-overridable processor and cannot be force-sent"); no `Delivery` is written. The timeline still shows the block; the UI surfaces the reason and does not offer (or disables) the force-send action for such events.

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
Automatic delivery is enqueued **only** when the overall outcome is `passed` — i.e. every enabled processor passed (any `STOPPED`/`FAILED` blocks it). There is no required/optional tier: every enabled processor blocks. The **only** way to deliver a blocked event is an explicit force send (BR-009).

#### BR-007 — Payload context preservation
The original event payload/headers are preserved unaltered on the `Event` record; the timeline references the event and never rewrites its payload. In-memory context transformations by processors are not persisted as event data in this feature.

#### BR-008 — Time semantics
`startedAt`/`endedAt` are real UTC timestamps captured around each `execute()`. `durationMs` is derived from a monotonic clock (`performance.now()`, existing behavior) to avoid wall-clock skew; the two need not be arithmetically identical and `durationMs` remains the source of truth for duration.

#### BR-009 — Force send (manual override)
An authorized user may force delivery of an event that was blocked (`STOPPED`/`FAILED`) — **unless** the halting processor is non-overridable (BR-013). Force send: (a) bypasses the delivery gate (BR-006); (b) dispatches via the existing delivery path to the endpoint's enabled destination(s); (c) records each resulting `Delivery` with `trigger = FORCED`; (d) does **not** alter the event at all — not its `processingStatus`, its timeline (BR-005), **nor `lastReplayedAt`** (a forced delivery is not a replay); (e) is audited (who/when) via the `FORCED` delivery, the `svc.delivery.forced` log line and the `delivery_forced_total` metric; (f) is subject to the duplicate-protection guard (BR-012). It is the sole sanctioned escape hatch for a blocked event and replaces today's hard dead-end (`assertReplayable` throws for `STOPPED`/`FAILED`). Automatic deliveries record `AUTOMATIC`; replays record `REPLAY` (BR-010).

#### BR-010 — Delivery trigger provenance
Every `Delivery` carries a `trigger` (`AUTOMATIC` | `REPLAY` | `FORCED`) recording how it was initiated. The gate path sets `AUTOMATIC`, replay sets `REPLAY`, force send sets `FORCED`. This is provenance only — it does not change delivery behavior.

#### BR-011 — Processor message content
A processor `message` is a human-readable *reason* for the run outcome. It must not echo raw event payload or secret values. The 500-char clamp remains; message content is a review/guideline concern for processor authors. The raw payload is surfaced only in the dedicated (owner-authorized) payload panel, and delivery request headers are redacted by the existing `redactHeaders` allowlist.

#### BR-012 — Idempotency & duplicate protection (first-class)
The system guarantees an event is processed at most once and protects against accidental duplicate user actions. The idempotency key is the **`eventId` + its `processingStatus` state**, not a separate message id (one inbound webhook = one `Event` row; a dedicated `messageId` would be redundant).
- **Machine (ingest) idempotency.** When the event's `processingStatus` is already terminal (`PROCESSED`/`STOPPED`/`FAILED`), `EventProcessingService.run()` **does not re-execute processors** and does not touch the timeline — but it **re-derives and returns the persisted outcome** (`processingStatus → passed/stopped/failed`) so the caller's downstream logic is unchanged. This is critical: `ingest.service` enqueues delivery only when `run()` returns `passed`; if a job crashes **after the persist commit but before the delivery enqueue**, the retry must still return `passed` so the (idempotent, `jobId=eventId`) delivery enqueue runs again — otherwise the webhook is silently lost. Only `PENDING` events are actually processed. This subsumes any "which processors already ran" concern (Q-IR1): because the run set is persisted **atomically** (BR-005 / engine ADR 0003 BR-017 two-write boundary), a terminal status means *all* ran and a `PENDING` status means *none* were persisted — there is no partial state to resume. The `@@unique([eventId, sequence])` constraint is the DB backstop for the concurrent-worker race: a second writer's `P2002` is caught and logged as a no-op (`timeline_immutable_skip`), never surfaced to the BullMQ job (so no retry storm; EF-01/EF-03).
- **User (force/replay) idempotency.** Force send and replay dispatch deliveries **directly** (not via the `jobId=eventId` delivery queue), so they lack the queue's dedup. A first-class **server-side in-flight guard backed by a Redis lock** (`SET lock:delivery:<key> <token> NX PX ~300s`, released in a `finally`; the `PX` is only a crash-safety cap, sized (~300s) well above a realistic parallel fan-out of destination timeouts so it cannot expire mid-delivery and let a concurrent request overlap) rejects a concurrent second force/replay with `409 Conflict`, so a double-click or repeated request cannot spawn duplicate deliveries — the disabled UI button is a convenience, not the guarantee (EF-06, IR-4). Redis is chosen over a DB advisory lock (which would be held across the outbound HTTP call) and over routing through the delivery queue (which would break the synchronous `204` contract). The guard covers **both** force send and replay at their shared `deliverEvent`/`deliverToDestination` entry point; lock key granularity is `eventId` for event-level ops (`forceDeliverEvent`, `replayEvent`) and `eventId:deliveryId` for `replayDelivery`.
- **Note (side-effecting processors):** per-processor resume would only be needed if incremental persistence + side-effecting processors (future "enrichers") are introduced; today's processors are pure, so event-level idempotency is sufficient. Tracked in Future Improvements.

#### BR-013 — Non-overridable processors
A processor may declare itself **non-overridable** via a code-level `overridable = false` property on the `EventProcessor` definition (default `true`; not user-configurable). **Resolving the halting processor:** `haltedBy` is not a stored column — force send derives it from the persisted timeline as the single run whose status is `STOPPED` or `FAILED`, then looks up that processor's `overridable` in the `EventProcessorRegistry`. If it is non-overridable, **force send is rejected** (409, EF-07) — the safety block cannot be bypassed. This is the enforcement counterpart to the delivery gate and exists specifically for security-critical processors (e.g. a future signature-verification processor) that must never be force-bypassed. **Decided:** the initial non-overridable set is **empty** — every current processor is `overridable=true` (the only registered processor today is `noop`). The mechanism (property + EF-07 enforcement + tests) ships ready so a future security-critical processor is marked `overridable=false` with a one-line change, no further design.

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
- No new secrets. Messages already clamped; enforce BR-011 (processor messages carry no raw payload/secret) — the one residual leak vector (R4).
- Timeline read authorization inherits event authorization; no broadening of data exposure.
- **Force send** is a privileged, gate-bypassing action: same authz as replay, explicit confirm, audited (`svc.delivery.forced` + `trigger = FORCED`), metered (`delivery_forced_total`), and **hard-blocked for non-overridable processors** (BR-013/EF-07) so security-critical checks (e.g. signature verification) can never be bypassed. Duplicate submissions are guarded server-side (BR-012), not just by the UI.
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
- `webhookr-toggles` — declares `web/event-timeline`, `svc/force-send`, `web/force-send` (done). **(config)**
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
4. **Model force send via a `trigger` enum on `Delivery` (`AUTOMATIC`/`REPLAY`/`FORCED`), not a timeline mutation.** The override is recorded on the delivery it produces; the timeline stays immutable and the block stays auditable. Reuse the existing `DeliveryService` dispatch path; force send differs from replay only by (a) skipping the `assertReplayable` block guard, (b) recording `trigger = FORCED`, and (c) enforcing the non-overridable check (BR-013).
5. **Idempotency is event-level, keyed on `eventId` + `processingStatus`** (not a new `messageId`). `run()` short-circuits on terminal status; the `@@unique([eventId, sequence])` constraint is the concurrency backstop. Atomic persistence makes per-processor resume unnecessary (BR-012).
6. **Non-overridable enforcement is a code-level processor property** (`overridable`, default `true`) rather than user-facing config — keeps the "no optional-processor config model" scope decision (OQ-1) intact while still allowing a hard, un-bypassable safety block (BR-013).

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
- `EventProcessingService.run()` gains a terminal-state short-circuit that **must return the persisted outcome** (not skip) to keep the downstream delivery enqueue idempotent across ingest-job retries (IR-1); reconciles with engine ADR 0003 BR-017.
- A Redis in-flight lock is introduced around the shared delivery-dispatch entry (force + replay) using the existing Redis instance — no new infrastructure, but a new use of Redis for locking (IR-3).
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
- No change to `Event` (payload context already preserved); `Event.processingStatus` already exists — this feature only newly **exposes** it through the read contract (IR-2), no schema change.
- Migration is additive (nullable timestamp columns on `EventProcessorRun` + `trigger` enum with default `AUTOMATIC` on `Delivery`) — no backfill required; existing deliveries default to `AUTOMATIC`.
- `Note:` `trigger` enum chosen over a minimal `forced Boolean` (approved 2026-07-10): also makes **replay** deliveries distinguishable from automatic ones (a free observability win). Replay paths set `REPLAY`; force send sets `FORCED`; gate sets `AUTOMATIC`. Threaded via a `trigger` parameter on the shared `deliverToDestination`.

**Non-schema code change — `EventProcessor` interface** (`webhookr-svc/src/event-processing/interfaces/event-processor.interface.ts`): add an optional `readonly overridable?: boolean` (default `true`) alongside `name`/`category` (BR-013). It is a code property of the processor definition, **not** a DB column and **not** user-configurable. The engine already records `haltedBy`; force send resolves the halting processor's `overridable` from the registry to enforce EF-07.

### API Contracts
#### Endpoints
One additive field on event-detail; one new force-send endpoint.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/projects/:projectId/endpoints/:endpointId/events/:eventId` | Event detail — now includes ordered `processorRuns` **and `processingStatus`** |
| POST | `/v1/projects/:projectId/endpoints/:endpointId/events/:eventId/force-deliver` | Force-send: deliver a (typically blocked) event, bypassing the gate; produces `FORCED` deliveries (BR-009). **Contract mirrors the existing `replay` endpoint exactly — `@HttpCode(204)`, empty body.** 409 if no enabled destination (EF-05) or if a non-overridable processor blocked the event (EF-07). |

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
_(Processor names/reason codes above are illustrative — only `noop` and the engine codes `PIPELINE_HALTED`/`PROCESSOR_ERROR` exist today; see Appendix §14.)_

Backward compatibility: `processorRuns` and `processingStatus` are **additive** fields; existing consumers ignore them. The existing `EventDetail` interface (`webhookr-svc/src/event/interfaces/event-detail.interface.ts`) — which today has **no** `processingStatus` — is extended with both `processingStatus` and `processorRuns`. `processingStatus` is required so the UI can detect a blocked event and offer force send.

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

enum EventProcessingStatus { PENDING PROCESSED STOPPED FAILED }

type Event {
  # ...existing fields...
  processingStatus: EventProcessingStatus!   # NEW: exposed for UI block-detection
  processorRuns: [ProcessorRun!]!            # ordered by sequence asc
}

enum DeliveryTrigger { AUTOMATIC REPLAY FORCED }

type Delivery {
  # ...existing fields...
  trigger: DeliveryTrigger!         # NEW: AUTOMATIC | REPLAY | FORCED
}

type Mutation {
  # deliver a (typically blocked) event, bypassing the gate (BR-009).
  # Mirrors the existing replayEvent mutation exactly (returns Boolean).
  forceDeliverEvent(projectId: ID!, endpointId: ID!, eventId: ID!): Boolean!
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
No **new** external dependencies. Uses existing Postgres (via Prisma), Prometheus/Grafana Cloud, and the BullMQ flow. The force/replay in-flight guard (BR-012/IR-3) uses the **existing Redis** (Redis Cloud, same `REDIS_URL` as BullMQ) for a short-lived lock — a new *use* of Redis, not new infrastructure.

### Feature Flags
Declared in [[webhookr-toggles]] (`toggles.yaml`); the Unleash name is `webhookr.<service>.<name>`. Logical envs are `prd` and `qa` (there is no `dev` env in this repo — non-prod validation happens in `qa`). All three new flags ship **off** in every env.
- Reuses the existing engine flag **`webhookr.svc.processor-engine`** (declared toggle `svc/processor-engine`, referenced in code as `PROCESSORS_ENABLED_FLAG`) that already gates engine execution.
- **New — `webhookr.web.event-timeline`** (toggle `web/event-timeline`): gates rendering of the timeline panel in web (later mobile) so the UI can ship dark and be turned on after the contract is verified end-to-end. The svc always returns `processorRuns` (additive); this flag is purely presentational.
- **New — `webhookr.svc.force-send`** (toggle `svc/force-send`): the **authoritative kill-switch** — when off, the `force-deliver` endpoint is disabled (404/`ForbiddenException`), so the capability is killed instantly regardless of UI state.
- **New — `webhookr.web.force-send`** (toggle `web/force-send`): the **UI gate** — shows/hides the force-send action in web independently of the endpoint. Two independent layers: turning off the web flag removes the button (clean UX, no click-then-error window, resolves gatekeeper R-2); turning off the svc flag is the hard security kill-switch. The web flag reads its own toggle — no BFF-surfaced capability needed.

### Migrations
1. Prisma migration adding nullable `startedAt`/`endedAt` to `EventProcessorRun` (additive, no backfill; safe/online — nullable columns, no default backfill scan).
2. Prisma migration adding the `DeliveryTrigger` enum and `trigger DeliveryTrigger @default(AUTOMATIC)` to `Delivery` (additive; existing rows default to `AUTOMATIC` without a scan). `Note:` historical deliveries pre-date the replay/forced distinction, so `AUTOMATIC` is a safe (if slightly lossy) default for them.
3. No data migration for historical rows (nulls / default `AUTOMATIC` tolerated by contract & UI).

### Rollback Strategy
- **Schema:** additive nullable columns + `DeliveryTrigger` enum defaulting to `AUTOMATIC` — safe to leave in place on rollback; no down-migration needed. If reverting the persist change, the old `deleteMany+createMany` code can be restored without schema impact.
- **UI:** toggle `webhookr.web.event-timeline` off to hide the panel instantly (no deploy). Force send has two independent levers: `webhookr.web.force-send` off removes the button (UX), `webhookr.svc.force-send` off hard-disables the endpoint (security kill-switch) — both independent of the timeline flag.
- **Contract:** `processorRuns`, `Delivery.trigger`, and `forceDeliverEvent` are additive; reverting svc/bff to not emit/expose them does not break older/newer web (web tolerates absence).
- **Order of rollback:** disable force-send flag → disable timeline UI flag → revert web → revert bff → revert svc (reverse of rollout).

## 6. Implementation Strategy

### Recommended Implementation Order
1. **svc – schema + engine timing (write path):** add `startedAt`/`endedAt`, capture in engine, persist write-once (immutability). Ship behind existing `processors-enabled`; timeline now recorded with timestamps and immutable.
2. **svc – read path:** include ordered `processorRuns` in event-detail response + interface + swagger + tests.
3. **svc – force send:** `DeliveryTrigger` enum + `Delivery.trigger` column + migration; `forceDeliverEvent` service (reuse `DeliveryService` dispatch, skip block guard, record `trigger=FORCED`, audit log + metric) + REST endpoint; thread `trigger` through `deliverToDestination` (replay→`REPLAY`, gate→`AUTOMATIC`); tests.
4. **bff – GraphQL exposure:** `ProcessorRun` type + `Event.processorRuns`; `DeliveryTrigger` + `Delivery.trigger`; `forceDeliverEvent` mutation; map from svc.
5. **web – UI:** timeline panel behind `webhookr.web.event-timeline`; Force-send action + confirm dialog + "Forced" badge behind `webhookr.web.force-send` (endpoint independently enforced by `webhookr.svc.force-send`).
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
3. Ship web (step 5) with all flags **off**; enable `webhookr.web.event-timeline` first (read-only, low risk) and validate; then enable force send **endpoint-first** — `webhookr.svc.force-send` on, validate the endpoint, then `webhookr.web.force-send` on to reveal the button (avoids a button-without-endpoint window); staged `qa`→`prd` ([[webhookr-toggles]] cutover discipline).
4. Enable in prod after end-to-end validation; monitor Grafana stop/failure + forced-delivery panels and audit logs.
5. Fast-follow mobile.

## 7. Task Split

### Infrastructure
**Repositories:** `webhookr-toggles`, `terraform` (if flag/alert wiring needs it)
**Tasks:**
- [x] Declare `web/event-timeline`, `svc/force-send`, `web/force-send` toggles in `toggles.yaml` (done — all `prd:false`/`qa:false`); enable per the rollout once the code lands.
- [ ] Add Grafana Cloud dashboard panel: processor stop rate & failure rate by processor (from `event_processing_processor_results_total{status}`), plus stopped/failed outcome counts. (Optional) alert on failure-rate spike.
- [ ] Add Grafana panel + alert for `delivery_forced_total` (spike in forced deliveries).

### Backend
**Repositories:** `webhookr-svc`
**Tasks:**
- [ ] Prisma: add nullable `startedAt`/`endedAt` to `EventProcessorRun`; add `DeliveryTrigger` enum + `trigger @default(AUTOMATIC)` to `Delivery`; generate migration(s).
- [ ] Engine: capture `startedAt`/`endedAt` (UTC) around each `execute()`; thread through `EngineProcessorRun` + `ProcessorRunRecord`.
- [ ] Repository: replace `deleteMany+createMany` with write-once insert; catch `@@unique` `P2002` as a logged no-op (`timeline_immutable_skip`) (EF-03/BR-005).
- [ ] **Idempotency guard (BR-012):** `run()` short-circuits when `processingStatus` is terminal — **re-deriving and returning the persisted outcome** (`processingStatus → passed/stopped/failed`) so the caller still enqueues delivery on retry (IR-1); no processor re-execution. Reconcile with engine ADR 0003 BR-017 crash-recovery. No new `messageId` (key = `eventId`+`processingStatus`).
- [ ] Confirm/annotate delivery gate (BR-006) with a test asserting non-`passed` outcomes never enqueue delivery.
- [ ] Add `overridable?: boolean` (default `true`) to `EventProcessor`; **no current processor is marked `overridable=false`** (initial set empty) — a test fixture processor covers the EF-07 rejection path.
- [ ] Force send: `forceDeliverEvent` service reusing `DeliveryService` dispatch, skipping the block guard, **deriving `haltedBy` from the timeline's `STOPPED`/`FAILED` run and rejecting when that processor is non-overridable (409, EF-07)** (IR-2), recording `trigger=FORCED`, emitting `svc.delivery.forced` audit log + `delivery_forced_total`; handle no-destination (EF-05). Thread the `trigger` param through `deliverToDestination` and set `REPLAY` on the existing replay paths.
- [ ] **Redis in-flight lock (BR-012/IR-3)** wrapping the shared `deliverEvent`/`deliverToDestination` entry for **force + replay** (`SET … NX PX ~300s`, key `eventId` / `eventId:deliveryId`, release in `finally`, 409 on contention).
- [ ] Regression test: replay/force send do not reprocess the pipeline nor mutate the timeline.
- [ ] Unit + repository tests for timing, halt→skip propagation, immutability, `trigger` provenance, idempotency short-circuit **(incl. terminal event still returns outcome → delivery re-enqueued)**, non-overridable rejection, and the Redis duplicate-guard.

### API
**Repositories:** `webhookr-svc`, `webhookr-bff`
**Tasks:**
- [ ] svc: extend `EventDetail` interface + `EventService.findOne` to include ordered `processorRuns` **and `processingStatus`** (currently absent); update swagger response schema.
- [ ] svc: add `POST /v1/projects/:projectId/endpoints/:endpointId/events/:eventId/force-deliver` controller + swagger; `@HttpCode(204)` mirroring `replay`; authz via `@CurrentUser`.
- [ ] svc: event-detail e2e test asserts ordered runs + timestamps + skipped propagation + `processingStatus`; force-send e2e asserts blocked (overridable) event delivers with `trigger=FORCED` and unchanged timeline/status, and a non-overridable block returns 409.
- [ ] bff: add `ProcessorRun` type + `Event.processorRuns` + `Event.processingStatus` (`EventProcessingStatus` enum); `DeliveryTrigger` enum + `Delivery.trigger`; `forceDeliverEvent` mutation (returns `Boolean`, mirroring `replayEvent`); map svc REST → GraphQL; unit tests (incl. null timestamps).

### Frontend
**Repositories:** `webhookr-web` (GA), `webhookr-mobile` (fast-follow)
**Tasks:**
- [ ] web: Timeline panel on event-detail, ordered by sequence, status badges (PASSED/STOPPED/FAILED/SKIPPED/DISABLED — DISABLED de-emphasized), start/end/duration, reason code + message; empty-state; behind `webhookr.web.event-timeline`.
- [ ] web: Force-send action on **overridable** blocked events — confirm dialog (copy in Appendix §14); "Forced" badge on resulting deliveries; action hidden/disabled (with reason) for non-overridable blocks (EF-07); **button gated by `webhookr.web.force-send`** (the svc endpoint independently enforces `webhookr.svc.force-send`). Button disabled while a request is in flight.
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
- Force send: service produces `trigger=FORCED` delivery, does not mutate timeline/`processingStatus`, emits audit log + metric; no-destination → validation error; **non-overridable block → 409, no delivery** (EF-07). Replay paths record `trigger=REPLAY`; gate path records `AUTOMATIC`.
- Idempotency (BR-012): `run()` on a terminal event does not re-invoke processors **but returns the persisted outcome**; concurrent double-persist → one succeeds, the other logs `timeline_immutable_skip` (no thrown error); duplicate force/replay → second gets 409 from the Redis lock.

### Integration Tests
- svc event-detail (`processor-config.e2e`/event e2e): ingest → process (stopped scenario) → GET event returns ordered `processorRuns` with correct statuses, reason codes, timestamps; delivery not enqueued.
- Delivery gate: `passed` → delivery enqueued; `stopped`/`failed` → not enqueued.
- Force-send e2e: blocked event → `force-deliver` → delivery attempted with `trigger=FORCED`; timeline and `processingStatus` unchanged; second event-detail read shows same immutable timeline.
- **Crash-window (IR-1):** simulate a retried ingest job for an event whose timeline is already persisted but whose delivery was never enqueued → assert delivery is enqueued exactly once (no lost webhook, no duplicate).
- Immutability: reprocess/replay path does not mutate timeline.

### End-to-End Tests
- web: open an event that was stopped → timeline panel shows PASSED → STOPPED → SKIPPED in order with timings and reason; flag-off hides panel.
- web: on the same blocked event, Force send (with confirm) → delivery appears with "Forced" badge; timeline still shows the block.

### Manual Validation
- Trigger a webhook that a processor stops; confirm in web UI the timeline explains the stop, delivery shows not-attempted, and Grafana stop counter increments.
- Trigger a processor error; confirm FAILED row with `PROCESSOR_ERROR` and following SKIPPED.
- Force send the blocked event; confirm it delivers, the delivery shows Forced, the timeline is unchanged, and `delivery_forced_total` increments.
- Open a historical event (pre-migration) → timestamps render "—" without error.
- Confirm processor messages contain no raw payload/secret material (BR-011).
- Force send a **non-overridable**-blocked event → confirm 409 and no delivery; force send an **overridable**-blocked event → confirm delivery with Forced badge.
- Retry an ingest job for an already-processed event → confirm no duplicate timeline and processors are not re-executed.

## 9. Definition of Ready
- [x] OQ-1 resolved: delivery gates on `outcome==passed` (every enabled processor passes); no required/optional tier.
- [x] OQ-2 resolved: replay/force send do **not** reprocess; timeline stays immutable.
- [x] OQ-3/OQ-4 resolved: `DISABLED` shown de-emphasized; retention = parent `Event` lifecycle.
- [x] OQ-5 resolved: non-overridable processors block force send (BR-013); initial set = **none** (all overridable).
- [x] Reason-code catalog documented — see Appendix §14 (initial, non-exhaustive; 2 real engine codes + convention + illustrative future codes).
- [x] UX defined in-spec: status legend + force-send dialog copy in Appendix §14 (no separate design cycle needed for GA; a11y note: status conveyed beyond color).
- [x] Toggles declared in [[webhookr-toggles]] `toggles.yaml` (`web/event-timeline`, `svc/force-send`, `web/force-send`; all off) — applied to `qa`/`prd` by the Apply workflow on merge.
- [x] BR-011: `message` cannot leak payload/secrets — trivially satisfied today (`noop` emits no message); recorded as a code-review gate for the first PRODUCT processor.

## 10. Definition of Done
- [ ] `startedAt`/`endedAt` persisted for executed processors; migrations applied dev+prod.
- [ ] Timeline is write-once immutable; replay/force send proven non-mutating by test.
- [ ] Automatic delivery attempted only on `passed` (test-enforced).
- [ ] Force send delivers an **overridable** blocked event, sets `Delivery.trigger=FORCED`, audits + meters, leaves the timeline unchanged (test-enforced); **non-overridable block returns 409** (EF-07); replay/automatic record `REPLAY`/`AUTOMATIC`.
- [ ] Idempotency verified (BR-012): terminal event → `run()` returns persisted outcome without re-executing (delivery still enqueued once, crash-window test); concurrent persist → no overwrite/no thrown error; duplicate force/replay → 409 via Redis lock.
- [ ] Event-detail REST + BFF GraphQL expose ordered `processorRuns` **and `processingStatus`**; `Delivery.trigger` + `forceDeliverEvent` exposed (all additive, backward-compatible).
- [ ] web renders timeline in execution order and offers force send on overridable blocked events (hidden/disabled for non-overridable), both behind flags; enabled in prod.
- [ ] Grafana panels show processor stop/failure counts and forced-delivery count.
- [ ] Unit/integration/e2e tests green; coverage for halt/skip, immutability, delivery gate, force send, idempotency, non-overridable.
- [ ] Docs updated; artifacts reconciled ([[claudemd-artifacts-source-of-truth]]).
- [ ] No payload/secret leakage in messages/logs verified (BR-011).

## 11. Estimate

| Size | Selected |
|------|:--------:|
| P    | ☐        |
| M    | ☑        |
| G    | ☐        |
| GG   | ☐        |

**Rationale:** Core write-path (model, engine timing, persistence, delivery gate, metrics) is largely already implemented; the work is a focused additive column + immutability hardening + a 3-repo additive read-model exposure + one web panel, **plus the force-send override** and its safety rails (idempotency guard BR-012, non-overridable BR-013, in-flight dedup) — all reusing existing paths (`DeliveryService`, `haltedBy`, `processingStatus`). **M**, firmly at the top of the band; re-check against **M/G** after the svc write-path PR lands (gatekeeper R-1) if the idempotency/crash-recovery or Redis-lock work proves cross-cutting. Mobile and a first-class "optional processor" config model remain excluded (would push to G).

## 12. References
- `webhookr-svc/src/event-processing/event-processor.engine.ts` — engine, halt/skip, message clamp.
- `webhookr-svc/src/event-processing/event-processing.service.ts` — orchestration, status mapping, metrics.
- `webhookr-svc/src/event-processing/event-processing.repository.ts` — persist (current delete+recreate; to become write-once).
- `webhookr-svc/src/ingest/ingest.service.ts:105-134` — delivery gate on outcome.
- `webhookr-svc/prisma/schema.prisma` — `Event`, `EventProcessorRun`, `ProcessorRunStatus`, `EventProcessingStatus`.
- `webhookr-svc/src/event/event.controller.ts` + `interfaces/event-detail.interface.ts` — event-detail contract.
- `webhookr-bff/src/events/models/event.model.ts` + `services/events.service.ts` — BFF Event + REST proxy.
- `webhookr-toggles/toggles.yaml` — `web/event-timeline`, `svc/force-send`, `web/force-send`.
- ADRs: `webhookr-artifacts/engineering/2026-07-06-event-processor-engine/` (0003), `.../2026-07-06-processor-config-management/` (0004).
- Related memory: [[webhookr-security-matrix]], [[webhookr-toggles]], [[webhookr-gtm-strategy]], [[claudemd-artifacts-source-of-truth]].
- Gatekeeper review: 98/100 ✅ Approved (2026-07-10).

## 13. Future Improvements
- Versioned/append timelines to support reprocess-on-replay (ADR Alternative A) if the reprocess decision (OQ-2) is ever revisited.
- Per-processor input/output snapshots (privacy-gated) for deep debugging.
- Timeline export / share link for support workflows.
- Optional / non-blocking processor tier (OQ-1) — a processor whose failure does not block delivery — as the automatic counterpart to today's manual force send. Only if a real enrichment/side-effect processor needs it.
- Per-processor idempotency / resume for future **side-effecting** processors (enrichers) if incremental persistence is ever introduced (today's event-level guard suffices — BR-012 note).
- Make the `overridable` property user-visible in the processor catalog (read-only) so users understand why a block can't be force-sent.
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
| Event detail shows timeline in order | ❌ not exposed (`processingStatus` also absent) | svc+bff+web read model + expose `processingStatus` |
| Delivery only when processors pass | ✅ gate on `outcome==='passed'` | Confirm + test (every enabled processor passes) |
| Preserves original payload context | ✅ `Event.payload` immutable | — |
| Timeline immutable after execution | ❌ `deleteMany+createMany` overwrites | Write-once persist + idempotency guard (BR-012) |
| Metrics expose stop/failure counts | ✅ `...processor_results_total{status}` | Grafana panel (+ optional counters) |
| **Force send blocked events (added)** | ❌ `assertReplayable` hard-blocks | `Delivery.trigger` (enum) + `forceDeliverEvent` + web action |
| **Non-overridable safety block (added)** | ❌ no concept | `overridable` on `EventProcessor` + 409 on force (BR-013/EF-07) |

### Status legend (UI)
`PASSED` (green) · `STOPPED` (amber, intentional halt) · `FAILED` (red, error) · `SKIPPED` (grey, not run due to halt) · `DISABLED` (muted, configured off). Deliveries produced by force send carry a `Forced` badge. **a11y:** status must be conveyed by icon/label as well as color (not color alone).

### Force-send UX copy (GA)
- **Action label:** "Force send" (shown only on `STOPPED`/`FAILED` events whose halting processor is overridable; hidden/disabled with tooltip "This event was blocked by a non-overridable check and cannot be force-sent" for non-overridable — EF-07).
- **Confirm dialog title:** "Force send this event?"
- **Confirm dialog body:** "This event was blocked by the processing pipeline (`{haltedBy}` → `{reasonCode}`). Force sending bypasses that block and delivers to all enabled destinations. The timeline keeps the original block for audit. This can't be undone."
- **Confirm button:** "Force send" · **Cancel:** "Cancel".
- After success: toast "Force sent" + the new delivery appears with a **Forced** badge.

### Reason-code catalog (initial, non-exhaustive)
Reason codes are stable `SCREAMING_SNAKE_CASE` identifiers on each run, surfaced in the API/UI. They are part of the read contract: **add** new codes rather than repurposing existing ones (a rename is a breaking change for consumers).

**Engine-level — exist today** (grounded in `webhookr-svc/src/event-processing/event-processor.engine.ts`):

| Code | On status | Meaning |
|------|-----------|---------|
| `PIPELINE_HALTED` | `SKIPPED` | The processor did not run because an earlier processor halted the pipeline. |
| `PROCESSOR_ERROR` | `FAILED` | The processor threw an unhandled error; `message` carries the clamped (≤500) error text. |

`PASSED` runs carry no reason code (`null`). `DISABLED` runs carry the resolver's `disabledReason` (or `null`).

**Processor-level — convention + illustrative** (`Assumption:` no PRODUCT processor exists yet — `noop` is the only registered processor and emits no reason code; the following are **naming conventions and anticipated examples, not implemented codes**):
- Convention: each processor owns the reason codes for its `STOPPED`/`FAILED` results, namespaced `<DOMAIN>_<REASON>`.

| Future processor | Code | On status | Meaning |
|------------------|------|-----------|---------|
| filter | `FILTER_NO_MATCH` | `STOPPED` | Event did not match the configured filter; intentionally not delivered. |
| signature | `SIGNATURE_MISSING` | `FAILED` | Required signature header absent. |
| signature | `SIGNATURE_INVALID` | `FAILED` | Signature verification failed. |
| schema | `SCHEMA_INVALID` | `FAILED` | Payload failed schema validation. |

_(The response example in §5 uses illustrative processor names — `signature-verify`, `payload-filter`, `delivery-guard` — and `FILTER_NO_MATCH`; only `noop` and the two engine codes exist today.)_
