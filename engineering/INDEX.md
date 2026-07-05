# Engineering Artifact Index

Newest first. One line per artifact. Follows the same storage convention as `marketing/` (see `../CLAUDE.md`).

## 2026-07-05 — ingest-slug-validation (manual)
- [adr-ingest-slug-validation v1](2026-07-05-ingest-slug-validation/2026-07-05-adr-ingest-slug-validation-v1.md) — validate slug at the edge via authenticated internal resolver in svc + in-process cache + opossum breaker + stale-while-error; `whr_` + 16 base62 slugs; 5 options compared

## 2026-07-01 — webhookr-architecture-audit (manual)
- [audit-report v1](2026-07-01-webhookr-architecture-audit/2026-07-01-audit-report-v1.md) — full prioritized audit (P0–P3): supply-chain PAT, Redis SPOF, unprovisioned alerts, secret hygiene; refuted false positives; remediation plan
