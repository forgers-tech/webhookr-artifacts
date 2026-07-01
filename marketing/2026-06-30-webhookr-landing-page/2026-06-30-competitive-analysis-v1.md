---
artifact: competitive-analysis
topic: webhookr-landing-page
date: 2026-06-30
version: 1
source: /design
status: Draft
---

# Competitive Analysis — Webhookr

Verified via web research on 2026-06-30. Webhookr claims verified against local `webhookr-docs`.

## Landscape

| Tool | Category | Direction | Strengths (sourced) | How Webhookr differs |
|---|---|---|---|---|
| **Webhook.site** | Inspection bin | Inbound (view only) | Zero-friction, no account, instant inspect of payload/headers/timing | Durable + production: no 100-req cap, no 7-day deletion, real destinations + replay |
| **Hookdeck** | Managed event gateway | Inbound (+queue/deliver) | Durable queue, **auto retries + backoff**, **dedup**, "Issues" grouping, metrics/observability export, localhost CLI | Simpler DX, **visual topology**, **Terraform IaC**; *(Webhookr lacks auto-retry/dedup)* |
| **Convoy** | OSS webhook gateway | Inbound + Outbound | Fan-out by type/payload, retries (constant & exp+jitter), batch retry, rate limit, static IPs, circuit breaking, signing, embeddable dashboard, self-host | Managed & developer-first, no infra to run; topology + IaC |
| **Svix** | Webhooks-as-a-Service | **Outbound** (Dispatch) + Ingest | Send at scale, HMAC signing, subscriber portal, multi-tenancy, SOC 2 / HIPAA / GDPR | Different job: Webhookr *consumes* third-party webhooks; Svix helps you *send* to your users |
| ngrok / RequestBin / Pipedream | Tunnel / bin / workflow | Inbound (dev) | Local tunneling / quick bins / automation | Persistent, team, production delivery + IaC — not just dev/test *(category-level research)* |

## Positioning conclusion
- **vs bins (Webhook.site):** durability + production destinations + replay — "the inspector you keep in production."
- **vs gateways (Hookdeck/Convoy):** developer-first simplicity + **visual topology** + **Terraform IaC**, without running infra. Do **not** claim reliability parity (see gap report).
- **vs Svix:** clarify category — Webhookr is for *receiving/consuming*, not *sending*.

## Honesty caveat (verified in webhookr-docs)
Webhookr does **not** document automatic retries/backoff, deduplication, rate limiting, or circuit
breaking. Replay is **manual** (`product/replay.mdx`). Differentiate on **visibility + replay +
fan-out + topology + IaC + simplicity**, never on delivery-reliability superiority. See
`2026-06-30-gap-report-v1.md`.

## Sources
- Hookdeck — https://hookdeck.com/ · https://hookdeck.com/docs/use-cases/receive-webhooks · https://hookdeck.com/docs/retries
- Svix — https://www.svix.com/ · https://www.svix.com/ingest/
- Convoy — https://www.getconvoy.io/ · https://github.com/frain-dev/convoy/blob/main/README.md
- Webhook.site — https://webhook.site/ · https://hookdeck.com/webhooks/platforms/webhook-site-alternatives
