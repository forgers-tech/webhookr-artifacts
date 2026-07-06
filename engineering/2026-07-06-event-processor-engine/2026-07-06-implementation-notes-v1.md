---
artifact: implementation-notes-event-processor-engine
topic: event-processor-engine
date: 2026-07-06
version: 1
source: manual
status: Reviewed
---

# Event Processor Engine — Implementation Notes

Records how the [feature spec v2](2026-07-06-feature-spec-event-processor-engine-v2.md)
was implemented, and the one place the as-built code deviates from it. The spec remains the
design record; this note captures the delta so the store stays honest without re-versioning the
whole spec for a single field.

## Shipped

- Repo: `webhookr-svc`, module `src/event-processing/`.
- PR: [forgers-tech/webhookr-svc#127](https://github.com/forgers-tech/webhookr-svc/pull/127).
- Durable code-side records: `docs/adr/0003-event-processor-engine.md` (as-built decisions) and
  `docs/event-processing.md` (subsystem guide). These are the operational source of truth next to
  the code and cross-link back here.
- Migration `20260706000000_event_processor_engine` (additive); applied on deploy by the svc
  `initContainer` (`npx prisma migrate deploy`). Global flag `webhookr.processors.enabled` ships
  **off** in prod, so runtime behavior is unchanged until a real processor is registered.

## Deviations from spec v2

1. **`EventProcessorResult.processorName` removed.** Spec §5 shows the result contract carrying
   `processorName`, but the engine always records the run under `processor.name` (from the
   registry) and never reads a name echoed back by the processor. The field was therefore
   redundant and a silent-mismatch risk, so it was dropped from `EventProcessorResult`. The
   engine's own output type, `EngineProcessorRun`, still carries `processorName`. Net contract:

   ```ts
   interface EventProcessor {
     readonly name: string;
     readonly category: ProcessorCategory;
     execute(context: EventProcessorContext): Promise<EventProcessorResult>;
   }

   interface EventProcessorResult {
     status: 'passed' | 'stopped' | 'failed';
     reasonCode?: string;
     message?: string;
     context?: EventProcessorContext;
   }
   ```

   Everything else in spec v2 (four-layer resolution, PLATFORM/PRODUCT defaults, status split,
   delivery gate BR-010, replay gate BR-015, idempotent persist BR-013/BR-017) shipped as
   specified.

## Convention codified during review

The duration histogram must declare millisecond-scale `buckets` (prom-client defaults are
second-scale). This lesson was generalized into a repo rule in `webhookr-svc/CLAUDE.md`
(Observability Rules): name metrics with an explicit unit suffix and match histogram buckets to
that unit.
