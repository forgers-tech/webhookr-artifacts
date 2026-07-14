---
title: Processor Execution Timeline — Council Benchmark Record
status: Reviewed
owner: thiagojv@gmail.com
created: 2026-07-11
updated: 2026-07-11
source: /council (benchmark mode)
priority: Reference
labels: [event-processing, council, benchmark, developer-experience, webhookr-svc, webhookr-bff, webhookr-web]
benchmarks: 2026-07-10-processor-execution-timeline/2026-07-10-feature-spec-processor-execution-timeline-v3.md
---

# Council Record — Processor Execution Timeline (Independent Benchmark)

> **Nature of this document.** This is **not** a new feature specification and it does **not**
> supersede the implemented spec (`…-v3`, Implemented + merged 2026-07-11). It is an *independent
> reconstruction* of the same decision, produced by the `/council` command in **benchmark mode**:
> the council reasoned from the user story + repository evidence, with the prior spec's
> feature-specific decisions quarantined until convergence, and only then compared its output
> against the registered artifact. The single open divergence (force-send scoping) is a
> **documentation/scope note, not a code change** — the shipped implementation already matches v3.

## Metadata
- **Objective:** Refine the Processor Execution Timeline user story into implementation-ready shared knowledge, reconstructed independently.
- **Outcome requested:** Council record + benchmark comparison vs. the existing artifact. Existing artifact **not** modified.
- **Depth:** Critical (delivery gate = safety boundary; persistence semantics vs. at-least-once queue retries; discovered blocked-event dead-end = safety-bypass question).
- **Status:** APPROVED WITH CONDITIONS.
- **Revision:** 1.
- **Date:** 2026-07-11.

---

## 1. Council Composition

**Selected:** Chair, Historian, Devil's Advocate, Scribe, Product Manager, Backend Staff Engineer,
Software Architect, Database Engineer, Web Staff Engineer, SRE, Security Engineer, QA Engineer,
Observability Engineer.

**Not selected (with reason):** Mobile (event-detail out of scope; Expo drift fast-follow),
FinOps (no cost delta — columns, not new rows/services), Data Engineer (no analytics pipeline),
Platform Engineer (no infra topology change; flag-gated code path), Performance Engineer (added
cost = one indexed read + two columns; delegated to Backend/DB).

---

## 2. Evidence Pack

### Verified Facts (FACT)
- **F1 — Engine already halts + marks skipped.** `event-processor.engine.ts` sets `haltedBy` on the first non-`passed` result or a thrown error and records every subsequent processor as `skipped` with `reasonCode = PIPELINE_HALTED`. Executed runs capture `startedAt`/`endedAt` (wall clock) + `durationMs` (`performance.now()`).
- **F2 — Status domain exceeds the AC's four.** `ProcessorRunStatus = PASSED, STOPPED, FAILED, SKIPPED, DISABLED`; `EventProcessingStatus = PENDING, PROCESSED, STOPPED, FAILED`. AC lists four run statuses.
- **F3 — Delivery gate exists, queue-driven.** `ingest.service.ts` enqueues delivery only when `eventProcessing.run()` returns `passed`, else `processing_blocked` and returns. Processing itself is behind `PROCESSORS_ENABLED_FLAG` (default off).
- **F4 — Ingest→processing→delivery on BullMQ (at-least-once).** Jobs can retry; delivery enqueued with `jobId=eventId` (idempotent enqueue).
- **F5 — "Payload context" already immutable on `Event`.** `Event.payload`/`headers` written once at ingest; engine mutates only in-memory `currentContext`.
- **F6 — Metrics conventions.** `event_processing_processor_results_total{processor,category,status}` already carries `STOPPED`/`FAILED`; ms-scale histogram buckets mandated by CLAUDE.md.
- **F7 — Blocked events are a dead-end today.** `DeliveryService.assertReplayable()` throws `ConflictException` for `STOPPED`/`FAILED` → no delivery path.
- **F8 — Ownership model.** Event-detail authorized by project/endpoint ownership (`@CurrentUser`); no role system.

### Relevant Existing Decisions (Historian — cross-cutting only)
- **P1** — Least-privilege network posture ([[webhookr-security-matrix]]): delivery 443-only; no new egress.
- **P2** — Queue payload encryption ([[webhookr-queue-encryption]]): sealed in transit; metadata-only logs.
- **P3** — Artifacts-as-source-of-truth (org CLAUDE.md): durable rationale lands in `webhookr-artifacts/engineering`.
- **P4** — CLAUDE.md standards: no repository abstraction over Prisma; no new abstraction until a 2nd use case; ms-suffixed metrics with explicit buckets.
- *(Feature-specific v3 decisions quarantined until convergence, per benchmark mode.)*

### Unknowns → Open Questions
- U1 — Semantics of "**required** processors" (tiered vs. "every enabled").
- U2 — "preserve original payload context" = per-run snapshot or link to `Event.payload`?
- U3 — Intended fate of **blocked** events (AC silent).
- U4 — Should `DISABLED` (config-off) appear in a four-status "timeline"?

---

## 3. Shared Understanding

**Problem:** Developers can't see per-processor outcomes; a non-delivered event is opaque. Persist an
ordered, immutable per-event timeline (name, status, timing, reason code, message), surface it in
event-detail, and make the delivery gate + its metrics legible.

**Proposed scope — In:** timeline persistence (`startedAt`/`endedAt`; keep `durationMs`),
immutability (write-once), event-detail exposure (svc→bff→web), gate confirmation + metrics,
idempotency hardening *iff* immutability changes retry semantics. **Out:** mobile UI, reprocess on
replay, optional/non-blocking tier, per-step snapshots, retention changes. **Contested:** force-send +
non-overridable (discovered adjacent to the AC, not in it).

---

## 4. Proposals Considered

- **P-A "Minimal timeline" (selected core):** add timestamps; write-once persist; nested
  `processorRuns` + `processingStatus` on event-detail; render in web; reuse labeled metric + Grafana
  panel. **No force-send.** Satisfies every AC; smallest change. Leaves blocked-event dead-end (a
  *product* gap, not an *AC* gap).
- **P-B "Timeline + force-send + non-overridable":** P-A plus `DeliveryTrigger` enum, force-delivery
  endpoint/mutation bypassing the gate, code-level `overridable`, `delivery_forced_total`, Redis
  in-flight guard. Closes the dead-end; materially larger scope + a new safety-bypass surface; none of
  it named in the ACs.
- **P-C "Event-sourced timeline aggregate" (rejected early):** versioned timeline for reprocess on
  replay. No current requirement; violates "no abstraction until a 2nd use case" (P4).

---

## 5. Challenges (cross-examination)

- **C-001 (Devil's Advocate → Product) — "required".** No tier exists (F2/F3); engine halts on any
  non-`passed`. **Resolved as ASSUMPTION A1** (not silent fact): "required" = "every enabled
  processor", to confirm with the author because it *narrows* the literal wording.
- **C-002 (SRE → Product) — blocked-event dead-end (F7).** Real gap, not an AC. → routed to **F-001**.
- **C-003 (Backend, self-critique) — immutability breaks queue idempotency.** Insert-once changes
  retry behavior; a crash *after persist commit but before delivery enqueue* would silently lose the
  webhook unless the short-circuit **returns the persisted outcome**. Concurrent-worker `P2002` caught
  as a no-op (no retry storm). → **D-004.** Makes idempotency hardening genuinely *coupled + in-scope*.
- **C-004 (Security → Product, conditional) — force-send bypasses a safety boundary.** If built, must
  be paired with a hard `overridable=false` block (409 when the halting processor is non-overridable),
  audited (`trigger=FORCED`, `delivery_forced_total`, log). Binding only if F-001 = bundle.
- **C-005 (Database → Architect) — "preserve payload context".** F5: payload already immutable on
  `Event`; per-run snapshots add privacy risk (P2) + unbounded storage. **Preserve = original payload
  unaltered + runs FK to event; no per-step diffs.** → **D-006.**
- **C-006 (QA → Backend) — `DISABLED` vs the four statuses.** Keep as informational, non-execution
  state, rendered distinctly (hiding it makes an "execution timeline" misleading). → **D-007** (A2).
- **C-007 (Observability → Product) — metrics.** F6: existing labeled counter already carries
  `STOPPED`/`FAILED`; a derived Grafana query satisfies the AC; dedicated counters optional. → **D-008.**

---

## 6. Conflicts and Resolutions

### Conflict F-001 — Force-send: bundle into this feature, or defer?
- **Position A (Product/Security/SRE):** Bundle — the gate *creates* the dead-end (F7), so own the
  escape hatch now; "why wasn't it delivered — and let me override" is one story.
- **Position B (Devil's Advocate/Architect):** Defer — force-send + trigger provenance +
  non-overridable + Redis lock is a *second* feature (a gate-bypass write with its own threat model);
  none of it is in the user story; principle "smallest coherent solution" favors timeline first.
- **Selected resolution:** **Defer to a scoped fast-follow, with recorded dissent.** Ship
  timeline + immutability + coupled idempotency now; force-send is a separate (already-specified)
  follow-up that **must land before the processors flag is enabled in prod** (otherwise the gate goes
  live with no recovery path). Option "never build it" rejected — the dead-end is real.
- **Trade-offs accepted:** two coordinated releases; a window where the gate exists without a UI
  escape hatch (mitigated: processing flag stays off until force-send ships).
- **Dissent (recorded):** Product/Security note bundling is defensible and arguably preferable
  operationally — and the **prior artifact chose to bundle** (see §10).
- **Validation required:** confirm force-send's story membership with the author (OQ-2).

### Conflict F-002 — "required" narrowing (from C-001)
Interpret as "every enabled processor", recorded as **Assumption A1 (UNVALIDATED)** — not promoted to
a silent requirement. If a real optional tier was intended, reopen.

---

## 7. Consolidated Decision

1. **Schema:** add nullable `startedAt`/`endedAt` (`timestamptz`) to `EventProcessorRun`; keep
   `durationMs` monotonic; `@@unique([eventId, sequence])` is the immutability backstop.
2. **Immutability:** `persist()` write-once (no delete+recreate); second write's `P2002` caught as a
   no-op.
3. **Idempotency (coupled, in-scope):** `run()` short-circuits on terminal `processingStatus` and
   **returns the persisted outcome** so a crash-window retry re-triggers the idempotent
   (`jobId=eventId`) delivery enqueue. Only `PENDING` events execute.
4. **Delivery gate:** behavior unchanged — enqueue only on `passed`; confirm + document.
5. **Exposure:** nested ordered `processorRuns` on event-detail (svc REST → bff GraphQL
   `processorRuns: [ProcessorRun!]!` → web panel), matching the `deliveries` precedent; expose
   `processingStatus`; render `DISABLED` as informational.
6. **Payload context:** preserved via existing immutable `Event.payload`; no per-run snapshots.
7. **Observability:** reuse `…_processor_results_total{status}` + a Grafana stop/failure panel;
   metadata-only logs.
8. **Deferred fast-follow (separately specified):** force-send + `DeliveryTrigger` + non-overridable +
   `delivery_forced_total` + Redis in-flight guard — must ship before the processors flag is enabled
   in prod.

---

## 8. Specialist Verdicts

| Role | Verdict | Key condition |
|---|---|---|
| Product | APPROVED WITH CONDITIONS | Confirm A1 ("required") + F-001 (force-send scope) with author |
| Backend | APPROVED | Explicit crash-window test |
| Architect | APPROVED | Event-sourcing deferred (reprocess one-way door noted) |
| Database | APPROVED WITH CONDITIONS | Read path `ORDER BY sequence`; additive/reversible migration |
| Web | APPROVED WITH CONDITIONS | Distinct visual for `SKIPPED` vs `DISABLED` vs `FAILED` |
| SRE | APPROVED WITH CONDITIONS | Processors flag stays off in prod until force-send ships (R5) |
| Security | APPROVED WITH CONDITIONS | `message` = reason text, never payload/secret echo; carries non-overridable into follow-up |
| QA | APPROVED | Add immutability + crash-window + idempotency tests |
| Observability | APPROVED | Grafana stop/failure panel |
| Historian | APPROVED | No contradiction with P1–P4 |
| Devil's Advocate | APPROVED WITH CONDITIONS | Force-send stays a separate reviewed feature |

No CHANGES REQUIRED remains → council closed.

---

## 9. Registers

### Decision Log
| ID | Decision | Evidence | Alt. rejected | Trade-off | Owner |
|---|---|---|---|---|---|
| D-001 | Nullable `startedAt`/`endedAt`; keep monotonic `durationMs` | F1 | Derive from `createdAt`+duration | Two columns | DB |
| D-002 | Write-once persist; `@@unique` backstop; `P2002` no-op | F4, C-003 | Keep delete+recreate | Loses reprocess-in-place | Backend |
| D-003 | Nested `processorRuns` (no separate endpoint) | `deliveries` precedent | Separate `/timeline` | Couples to event-detail | Architect |
| D-004 | Short-circuit returns persisted outcome (crash-window safe) | F3/F4, C-003 | Skip-and-return-nothing (lost delivery) | Subtler control flow | Backend |
| D-005 | Force-send/non-overridable/trigger **deferred** to fast-follow | F7, F-001 | Bundle now | Two releases; temp dead-end | Product |
| D-006 | Preserve payload via existing `Event.payload`; no snapshots | F5, C-005 | Per-step snapshots | No step-level input history | Architect |
| D-007 | Keep `DISABLED` as informational non-execution state | F2, C-006 | Hide it | Timeline shows non-executed rows | Backend |
| D-008 | Reuse `…_processor_results_total{status}` + Grafana panel | F6, C-007 | New dedicated counters | Derived query vs. first-class | Observability |

### Assumption Register
| ID | Assumption | Validation | Status | Impact if false |
|---|---|---|---|---|
| A1 | "required" = every enabled processor (no tier) | Confirm with author | **UNVALIDATED** | Reopen: model optional/non-blocking tier |
| A2 | Surfacing `DISABLED` is desired UX | Design review | UNVALIDATED | Minor: hide rows |
| A3 | Force-send is a sibling story, not this one | Confirm with author | **UNVALIDATED** | Reopen scope; rebundle |
| A0 | 4 AC statuses map to PASSED/STOPPED/FAILED/SKIPPED | Code | VALIDATED | — |
| A4 | Force-send inherits replay authz (no new role) | Follow-up DoR | ACCEPTED (defers w/ D-005) | Privilege gap |

### Risk Register
| ID | Risk | P | I | Mitigation | Detection | Owner |
|---|---|---|---|---|---|---|
| R1 | Immutability change → retry storm / lost delivery | M | H | D-004 + `P2002` no-op | Crash-window test; `timeline_immutable_skip` log | Backend |
| R2 | Backfill: old rows lack timestamps | H | L | Nullable + UI "—" | Visual | Web |
| R3 | Contract drift svc→bff→web | M | M | Additive fields, dep order | CI/e2e | Architect |
| R4 | `message` echoes payload/secret (PII) | L | H | Reason-text-only + 500-char clamp | Review | Security |
| R5 | Gate live in prod with no escape hatch (D-005 window) | M | H | Keep processors flag off until force-send ships | Flag-state audit | SRE |
| R6 | (deferred) Force-send bypasses safety-critical processor | L | H | Non-overridable guard in follow-up | `delivery_forced_total` spike alert | Security |

### Open Questions
| ID | Question | Blocking | Owner |
|---|---|---|---|
| OQ-1 | "required" = tier or "every enabled"? (A1) | No (assumption recorded) | Product |
| OQ-2 | Does force-send belong to this story? (A3/F-001) | No | Product |
| OQ-3 | Is surfacing `DISABLED` desired? (A2) | No | Web |

### Deferred Scope
Force-send feature; mobile timeline; reprocess-on-replay (event-sourced); optional/non-blocking tier;
per-run payload snapshots; dedicated `EventProcessorRun` retention.

---

## 10. Benchmark Comparison — Council vs. Registered Artifact (v3)

Compared only after verdicts. **The implementation shipped and matches v3** — confirmed in-repo:
`DeliveryTrigger` enum, `force-delivery.controller.ts`, BFF `forceDeliver` mutation,
`delivery_forced_total`, write-once persist, event-level idempotency, web timeline panel.

### Convergences
- Data model (nullable `startedAt`/`endedAt`, monotonic `durationMs`) — = v3 ADR#1 / BR-008.
- Write-once immutability with `@@unique` backstop — = v3 BR-005 / ADR#2.
- Nested read-model over a separate endpoint — = v3 ADR#3 (same `deliveries` precedent).
- Payload = link to immutable `Event.payload`, no snapshots — = v3 BR-007.
- Metrics reuse the labeled counter; dedicated counters optional — = v3 metrics / D-008.
- "required" = every enabled processor — same *conclusion* as v3 OQ-1.
- **Crash-window idempotency** (return persisted outcome; `P2002` no-op) — **independently
  rediscovered** the subtle lost-delivery window (v3 BR-012 / IR-1). Strongest signal the decision is
  robust and reconstructible.
- Event-sourcing deferred — = v3 Alternative A.

### Divergences
- **Force-send scoping (material).** Council **DEFERS** force-send to a scoped fast-follow (D-005,
  with recorded Product/Security dissent); v3 **BUNDLED** it (BR-009/010/013). A *process* divergence
  about deliverable boundaries, not a technical disagreement — the council agrees force-send is needed
  and agrees on *how* to build it safely.
- **"required" as an open assumption (A1)** vs. v3's "Decided (OQ-1)." Council is more conservative
  about a wording that literally implies a tier.

### Omissions (present in v3, council under-specified — routed into the deferred feature)
- Redis in-flight guard mechanics (`SET … NX PX ~300s`) — v3 BR-012 "User idempotency".
- Reason-code catalog + status legend + force-send UX copy — v3 Appendix (left to spec-writer).
- `lastReplayedAt` must not change on forced delivery — v3 BR-009 (deferred with force-send).

### Improvements (council contributes over v3)
- Explicit **release-ordering safety gate** (R5/D-005): "gate must not go live without an escape
  hatch" is a first-class, checkable follow-up condition (implicit in v3's bundle).
- **Assumption discipline** on "required" (A1) + force-send scope (A3): validation-needed rather than
  silently decided.
- Cleaner isolation of the **safety-bypass threat model** into its own reviewable unit.

### Regressions (council weaker than v3)
- **Reduced immediate completeness:** deferring force-send leaves the blocked-event dead-end open
  until a second release — the exact gap v3 closed in one shot. If the author intended force-send
  in-story (OQ-2), the split is a regression and triggers a reopen; v3's bundle avoided that
  round-trip.
- **Less prescriptive downstream detail:** v3 shipped a reason-code catalog, UX copy, DoR/DoD; the
  council stopped at negotiated knowledge.

### Net assessment
On the **AC-scoped feature** the council and v3 **fully converge**, including the non-obvious
crash-window idempotency insight. The single real divergence is **scope philosophy** — council
optimizes for the smallest coherent AC-scoped deliverable and defers force-send with recorded
dissent; v3 optimized for closing the operational dead-end in one feature and bundled it. Neither is
wrong; the deciding input is the story author's intent on **OQ-2**, which the council flags as the one
thing worth confirming rather than assuming.

---

## 11. References
- Registered spec: `engineering/2026-07-10-processor-execution-timeline/2026-07-10-feature-spec-processor-execution-timeline-v3.md` (Implemented + merged 2026-07-11).
- Implementation notes v2 (as-built): same folder.
- Code: `webhookr-svc/src/event-processing/` (engine, service, repository, interfaces), `webhookr-svc/src/ingest/ingest.service.ts`, `webhookr-svc/prisma/schema.prisma`; `webhookr-bff/src/events`, `webhookr-bff/src/deliveries`; `webhookr-web/src/features/events`.
- Cross-cutting: [[webhookr-security-matrix]], [[webhookr-queue-encryption]].
