---
artifact: implementation-notes-processor-execution-timeline
topic: processor-execution-timeline
date: 2026-07-10
version: 2
source: manual
status: Implemented
---

# Processor Execution Timeline (+ Force Send) — Implementation Notes

As-built record. Supersedes [v1](2026-07-10-implementation-notes-v1.md), which was
a snapshot at PR-open time (it said "PRs open" and "Grafana not applied" — both now
untrue). The design record is [feature-spec v3](2026-07-10-feature-spec-processor-execution-timeline-v3.md);
this note captures what shipped, the decisions made during review, and verification.

## Shipped & deployed (2026-07-11)

Merged and deployed **dark** (all flags off) across:

- **webhookr-svc** [#130](https://github.com/forgers-tech/webhookr-svc/pull/130) · **bff** [#77](https://github.com/forgers-tech/webhookr-bff/pull/77) · **web** [#88](https://github.com/forgers-tech/webhookr-web/pull/88) · **docs** [#12](https://github.com/forgers-tech/webhookr-docs/pull/12)
- **artifacts** [#18](https://github.com/forgers-tech/webhookr-artifacts/pull/18) (impl-notes v1), [#19](https://github.com/forgers-tech/webhookr-artifacts/pull/19) (spec v3)
- **terraform** [#101](https://github.com/forgers-tech/terraform/pull/101) (Grafana dashboard), [#102](https://github.com/forgers-tech/terraform/pull/102) (documented the `make` apply procedure)

All four app **Deploy** workflows completed green. Deploy path: the svc Deployment's
`migrate` initContainer runs `prisma migrate deploy` (the two additive migrations)
before the app starts; the reusable build-deploy then bumps the image tag in `gitops`
via an auto-merged PR and ArgoCD rolls it out.

The Grafana **`webhookr-event-processing`** dashboard was applied live via
`make apply ARGS="-target=grafana_dashboard.webhookr_event_processing …"` (plan: 1 to
add, 0 change, 0 destroy).

## Decisions that survived / changed in review

The build-time decisions in v1 still hold (notably: `overridable` kept **off** the
GraphQL contract — enforced server-side with a 409, initial non-overridable set empty).
Copilot review refined two design details, now folded into spec v3:

1. **Delivery lock TTL ~60s → ~300s.** The `PX` is only a crash-safety cap (the lock is
   released in `finally`), but it must exceed a realistic *parallel* fan-out of
   destination timeouts (default 30s each, configurable) so it cannot expire mid-delivery
   and let a concurrent force/replay overlap. Full lock renewal was judged out of scope.
2. **Force send leaves the event untouched.** It no longer bumps `lastReplayedAt` — a
   forced delivery is not a replay. The override is auditable purely via the `FORCED`
   delivery, the `svc.delivery.forced` log and the `delivery_forced_total` metric.

Two review points were **answered and kept** (would contradict the approved design):
force send of an already-`PROCESSED` event stays allowed (spec AF-07; the UI only
surfaces it for blocked events); and `startedAt`/`endedAt` stay `DateTime?`/`TIMESTAMP(3)`
for consistency with every other timestamp column (Prisma normalizes to UTC).

Security posture is unchanged at the infra layer — force send adds no component, port
or egress (it reuses the existing 443-only delivery path), so the infra
[[webhookr-security-matrix]] needed no entry; its app-level authz/audit/kill-switch
rationale lives in the spec (§4 Security).

## Verification

svc 693 unit + 130 e2e; bff 347; web 236; docs Docusaurus build; terraform
`fmt`/`validate` + targeted plan. All app deploys green; dashboard live in Grafana Cloud.

## Remaining (operational, by design)

- **Rollout:** the three toggles ([[webhookr-toggles]]) ship **off**. Enable staged —
  `webhookr.web.event-timeline` first (read-only), then force send endpoint-first
  (`webhookr.svc.force-send` before `webhookr.web.force-send`), `qa`→`prd`.
- **Fast-follow:** mobile timeline + force send (Expo drift).
- **Deferred:** Grafana alerts on stop/failure & forced-delivery spikes (thresholds
  need an ops call); making `overridable` visible in the processor catalog.

## Related

- Design record: [feature-spec v3](2026-07-10-feature-spec-processor-execution-timeline-v3.md).
- Code-side operational doc: `webhookr-svc/docs/event-processing.md`.
- Memory: [[webhookr-toggles]], [[webhookr-security-matrix]], [[claudemd-artifacts-source-of-truth]].
