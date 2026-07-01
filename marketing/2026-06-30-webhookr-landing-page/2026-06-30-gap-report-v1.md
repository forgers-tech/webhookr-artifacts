---
artifact: gap-report
topic: webhookr-landing-page
date: 2026-06-30
version: 1
source: /design
status: Draft
---

# Gap Report — what competitors have that Webhookr does not

Scope: capabilities that direct/adjacent competitors advertise and that are **not documented** in
`webhookr-docs` (as of 2026-06-30). "Not documented" = appears absent; each item is flagged
**[verify with team]** because absence in docs is not proof of absence in the product.

Competitors referenced: Hookdeck, Convoy, Svix, Webhook.site. Sources at the end.

## Reliability / delivery

| Capability | Who has it | Webhookr today | Impact |
|---|---|---|---|
| **Automatic retries with backoff** | Hookdeck, Convoy, Svix | ❌ replay is **manual** only (`product/replay.mdx`) | High — biggest reliability gap; today a failed delivery needs a human to replay |
| **Deduplication** (no duplicate side effects on retry) | Hookdeck | ❌ not documented | Medium — matters once retries exist |
| **Rate limiting / throttling to protect the destination** | Hookdeck, Convoy | ❌ not documented | Medium — downstream can be overwhelmed on bursts |
| **Circuit breaking** on failing destinations | Convoy | ❌ not documented | Medium |
| **Delivery signing / HMAC to destinations** | Svix, Convoy | ❌ not documented (X-Signature seen on *incoming* capture only) | Medium — destinations can't verify Webhookr as the sender |

## Routing / processing

| Capability | Who has it | Webhookr today | Impact |
|---|---|---|---|
| **Transformations** (mutate payload before delivery) | Hookdeck | ❌ | Medium |
| **Filtering / routing rules** (by type or payload) | Hookdeck, Convoy | ⚠️ fan-out is to *all* enabled destinations; no per-event routing rules documented | Medium |
| **Rolling / rotating secrets** | Convoy | ❌ | Low–Medium |
| **Static egress IPs** for firewall allowlists | Convoy | ❌ | Medium for enterprise destinations |

## Operations / trust

| Capability | Who has it | Webhookr today | Impact |
|---|---|---|---|
| **Observability export** (metrics/latency/alerting to your stack) | Hookdeck | ❌ dashboard metrics exist; no export/alerting documented | Medium |
| **Local-dev CLI / tunnel** (replay to localhost) | Hookdeck CLI, ngrok | ❌ | Medium — strong DX hook competitors use |
| **SSO / OAuth** | Svix (enterprise), Hookdeck | ❌ email + password only (`product/overview.mdx`) | Medium for teams |
| **Compliance certifications** (SOC 2, HIPAA, GDPR, …) | Svix | ❌ not stated | High for enterprise sales |
| **Self-host / open source** | Convoy, Svix (core) | ❌ managed only | Situational |

## Reading of the gap
Webhookr is strong on the **inbound record + inspect + fan-out + replay + IaC** story, but is behind
the heavyweight gateways (Hookdeck, Convoy) on **automatic delivery reliability** (retries, dedup,
rate limiting, circuit breaking) and behind Svix on **enterprise trust** (SSO, compliance). The
single highest-leverage gap is **automatic retries with backoff** — it is table stakes for
"reliable webhook delivery" and its absence is why the landing page must position on *visibility and
replay*, not on *reliability parity*.

## Sources
- Hookdeck — https://hookdeck.com/ · retries/replay/dedup: https://hookdeck.com/docs/retries
- Convoy — https://www.getconvoy.io/ · README: https://github.com/frain-dev/convoy/blob/main/README.md
- Svix — https://www.svix.com/ · Ingest: https://www.svix.com/ingest/
- Webhook.site limitations — https://hookdeck.com/webhooks/platforms/webhook-site-alternatives
- Webhookr docs — local `webhookr-docs/docs/` (intro, product/replay, product/overview, reference/*)
