# Engineering Artifact Index

Newest first. One line per artifact. Follows the same storage convention as `marketing/` (see `../CLAUDE.md`).

## 2026-07-06 — event-processor-engine (/upstream)
- [implementation-notes v1](2026-07-06-event-processor-engine/2026-07-06-implementation-notes-v1.md) — as-built record: shipped in webhookr-svc#127; one deviation from spec v2 (`processorName` dropped from `EventProcessorResult`, redundant with the registry-known `processor.name`); code-side records are ADR 0003 + docs/event-processing.md; histogram-bucket convention codified in svc CLAUDE.md
- [feature-spec-event-processor-engine v2](2026-07-06-event-processor-engine/2026-07-06-feature-spec-event-processor-engine-v2.md) — **current.** v1 + Gatekeeper findings: replay gate (BR-015: reject replay of `STOPPED`/`FAILED`, keep `PENDING`/`PROCESSED` replayable), explicit event state machine + terminality, two-write transaction boundary + crash recovery (BR-017), config-read perf budget/cache trigger
- [feature-spec-event-processor-engine v1](2026-07-06-event-processor-engine/2026-07-06-feature-spec-event-processor-engine-v1.md) — superseded by v2. Upstream feature spec for the pre-delivery processor pipeline in `webhookr-svc`: orchestration-only engine + `PipelineResolver` (4-layer enablement: global flag ∧ per-processor flag ∧ project ∧ endpoint, endpoint>project>platform default), PLATFORM vs PRODUCT categories, `Event.processingStatus` split from `EventProcessorRun` status (+DISABLED), delivery gated on `passed`, config tables schema+read-path (write surface deferred); reviewed 95/100 Approved-with-Recommendations

## 2026-07-05 — infra-security-matrix (manual)
- [adr-infra-security-matrix v1](2026-07-05-infra-security-matrix/2026-07-05-adr-infra-security-matrix-v1.md) — network least-privilege: deny-by-default egress (port-scoped, private-range except = SSRF guard), delivery 443-only, /docs blocked in depth; accepted risks (Neon no IP allowlist → GA revisit, 41641/ICMP, BFF public); operational matrix in gitops
- [threat-model-infra v1](2026-07-05-infra-security-matrix/2026-07-05-threat-model-infra-v1.md) — STRIDE-lite over the infra trust boundaries (SSRF, exfiltration, lateral movement, recon, origin bypass, admin exposure, supply chain) with the mitigating control per threat; residual/accepted risks

## 2026-07-05 — redis-cloud-migration (manual)
- [adr-redis-cloud-migration v1](2026-07-05-redis-cloud-migration/2026-07-05-adr-redis-cloud-migration-v1.md) — replace in-cluster K3S Redis (audit P0: SPOF, no backup) with Redis Cloud HA + AOF 1s + TLS; single `rediss://` `REDIS_URL` across ingest/svc/bff; Terraform-imported; big-bang cutover

## 2026-07-05 — ingest-slug-validation (manual)
- [adr-ingest-slug-validation v1](2026-07-05-ingest-slug-validation/2026-07-05-adr-ingest-slug-validation-v1.md) — validate slug at the edge via authenticated internal resolver in svc + in-process cache + opossum breaker + stale-while-error; `whr_` + 16 base62 slugs; 5 options compared

## 2026-07-01 — webhookr-architecture-audit (manual)
- [audit-report v1](2026-07-01-webhookr-architecture-audit/2026-07-01-audit-report-v1.md) — full prioritized audit (P0–P3): supply-chain PAT, Redis SPOF, unprovisioned alerts, secret hygiene; refuted false positives; remediation plan
