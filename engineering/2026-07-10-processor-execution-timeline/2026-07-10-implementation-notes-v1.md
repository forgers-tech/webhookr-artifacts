---
artifact: implementation-notes-processor-execution-timeline
topic: processor-execution-timeline
date: 2026-07-10
version: 1
source: manual
status: Reviewed
---

# Processor Execution Timeline (+ Force Send) — Implementation Notes

Records how the [feature spec v2](2026-07-10-feature-spec-processor-execution-timeline-v2.md)
was implemented across the four repos, and the few places the as-built code makes a decision the
spec left to implementation. The spec remains the design record; this note captures the *why* of
those choices and the accepted follow-ups so the store stays honest.

## Implemented (PRs open, awaiting review/merge)

- **webhookr-svc** [#130](https://github.com/forgers-tech/webhookr-svc/pull/130) — schema (two
  migrations), engine timing, write-once persist, idempotency short-circuit, event-detail read
  model, force-send endpoint + service, `Delivery.trigger`, Redis in-flight lock, metrics, docs.
- **webhookr-bff** [#77](https://github.com/forgers-tech/webhookr-bff/pull/77) — GraphQL
  `processingStatus` + `processorRuns`, `Delivery.trigger`, `forceDeliverEvent` mutation.
- **webhookr-web** [#88](https://github.com/forgers-tech/webhookr-web/pull/88) — timeline panel,
  force-send action, "Forced" delivery badge, both behind flags.
- **webhookr-docs** [#12](https://github.com/forgers-tech/webhookr-docs/pull/12) — user-facing
  Processor Timeline concept page.
- **webhookr-toggles** — the three flags (`web/event-timeline`, `svc/force-send`,
  `web/force-send`) were already declared during upstream; no change here.

## Decisions made at build time (why)

1. **`overridable` is not exposed on the GraphQL contract.** BR-013 is enforced server-side; the
   web offers force send on any blocked event and surfaces the svc **409** verbatim if the
   halting processor is non-overridable. This matches the spec (the ProcessorRun GraphQL type has
   no `overridable` field, and making it user-visible is listed under Future Improvements) and
   avoids leaking a security axis the UI doesn't need yet. Safe because the initial non-overridable
   set is empty — no event can currently be refused, so the 409 path is exercised only by tests.

2. **BFF maps svc `403 → ForbiddenException`.** The shared `WebhookrClientService` previously fell
   through to a 500 for 403. The force-send kill-switch (`webhookr.svc.force-send` off → 403) is
   the first caller that returns 403, so the mapping was added to surface it correctly. A minimal,
   contract-completeness change, not new behavior.

3. **The Redis in-flight lock owns its own ioredis client** (`DeliveryLockService`), mirroring the
   existing `RedisHealthIndicator` pattern rather than introducing a shared client DI token — the
   repo's convention and its "avoid new DI tokens without clear benefit" rule. Lock is a
   `SET NX PX ~60s` with a token-checked Lua release; the PX is a crash cap, release is in
   `finally`. Wraps the shared force/replay entry only — the automatic path stays deduped by the
   BullMQ `jobId=eventId`.

4. **`trigger` was threaded through the delivery *list* projection too** (not only the detail
   type the spec's GraphQL block showed), across svc/bff/web, so the "Forced" badge can render in
   the deliveries table without an extra round-trip. Additive.

5. **The web adoption metric is a `webhookr:analytics` CustomEvent** (`event_timeline_viewed`)
   dispatched on panel mount. No analytics sink exists in the web app and the server logger is
   pino (not browser-safe), so a listener-ready DOM event is the least-invasive hook that
   satisfies the success-metric task without inventing an analytics framework.

6. **`force-deliver` is a dedicated controller** at `.../events/:eventId/force-deliver` (sibling
   of `/deliveries`), mirroring the replay endpoint's `204`/empty-body contract.

No deviation changes the approved design; each is an implementation-level choice within it.

## Verification

- svc: build + lint clean; **693 unit** and **130 e2e** green (idempotency short-circuit incl.
  terminal→re-enqueue, write-once immutability + P2002 no-op, delivery gate, force send guard
  paths, non-overridable 409, Redis duplicate guard).
- bff: build + lint clean; **346 tests** green (timeline mapping, `trigger`, `forceDeliverEvent`,
  403→Forbidden).
- web: typecheck + lint clean; **236 tests** green (timeline render/empty/analytics, force-send
  dialog/confirm/error/cancel, flag gating).
- docs: Docusaurus build passes, no broken links.

## Accepted follow-ups (not in these PRs)

- **Mobile** timeline + force send — fast-follow (Expo drift, out of GA scope per spec).
- **Grafana panels** for processor stop/failure rate and `delivery_forced_total` — observability
  task tracked separately; the metrics they read are emitted by svc #130.
- Making `overridable` user-visible in the processor catalog (spec Future Improvements).
- Rollout is flag-gated and staged (`qa`→`prd`), endpoint-first for force send; all three flags
  ship **off**.

## Related

- Design record: [feature-spec v2](2026-07-10-feature-spec-processor-execution-timeline-v2.md).
- Code-side operational doc: `webhookr-svc/docs/event-processing.md` (updated in #130).
- Memory: [[webhookr-toggles]], [[webhookr-security-matrix]], [[claudemd-artifacts-source-of-truth]].
