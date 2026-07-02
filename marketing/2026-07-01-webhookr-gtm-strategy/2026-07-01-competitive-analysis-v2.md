---
artifact: competitive-analysis
topic: webhookr-gtm-strategy
date: 2026-07-01
version: 2
source: manual
status: Draft
---

# Competitive Analysis v2 — Webhookr

Supersedes `2026-06-30-webhookr-landing-page/2026-06-30-competitive-analysis-v1.md` by adding
**Trigger.dev** and a **pricing teardown** per competitor. Web research performed 2026-07-01.
Webhookr claims verified against local `webhookr-docs` (see v1 honesty caveat, still valid:
**no auto-retry, no dedup, no rate limiting, no circuit breaking; replay is manual**).

## 1. Landscape (updated)

| Tool | Category | Direction | Strengths (sourced) | How Webhookr differs |
|---|---|---|---|---|
| **Hookdeck** | Managed event gateway | Inbound (Event Gateway) **+ now Outbound (Outpost)** | Durable queue, auto retries + backoff, dedup, localhost CLI, 99.999% uptime SLA (Growth+), metrics export, **new: "agent-ready" positioning w/ MCP server** | Simpler DX, **visual topology**, **Terraform IaC**; Webhookr lacks auto-retry/dedup/CLI |
| **Svix** | Webhooks-as-a-Service | Outbound (Dispatch) + Inbound (Ingest) + Stream + Play — **all four included in every tier** | Send at scale, HMAC signing, subscriber portal, SOC 2 (Pro+), HIPAA/on-prem (Ent), generous free tier (50k msgs/mo) | Different center of gravity: Svix's core is *sending to your users*; Ingest is an add-on surface. Webhookr is purpose-built for *consuming* third-party webhooks |
| **Convoy** | OSS webhook gateway | Inbound + Outbound | Open source (self-host free), auto retries, fan-out by type/payload, rate limiting, circuit breaking, static IPs, portal links | Managed & developer-first, no infra to run; topology + IaC. Convoy's OSS is free forever — Webhookr can't compete on "free self-host" |
| **Trigger.dev** *(new in v2)* | Background jobs / workflow platform | N/A (webhooks are *triggers*, not the product) | Open source (Apache 2.0), TypeScript-native, durable runs w/ retries, queues, schedules, observability; pivoted messaging to **"fully-managed AI agents and workflows"** | Different job: Trigger.dev is *build* (write task code that handles a webhook); Webhookr is *buy* (webhook capture/replay/fan-out as product). Competes for the same "how do I handle this Stripe webhook reliably?" search intent |
| Webhook.site / ngrok / bins | Inspection / tunnel | Inbound (dev) | Zero-friction inspect | Durable + production destinations + replay (v1 analysis unchanged) |

## 2. Pricing teardown (verified 2026-07-01)

### Hookdeck — usage-based on events/mo + retention + throughput
| Plan | Price | Included | Retention | Users |
|---|---|---|---|---|
| Developer | $0 | 10k events/mo, 5 events/s | 3 days | 1 |
| Team | $39+/mo | 10k events/mo included | 7 days | Unlimited |
| Growth | $499+/mo | 10k included + SLAs (99.999% uptime) | 30 days | Unlimited |
| Enterprise | Custom | Custom | Custom | Unlimited |

Overage: $3.00 → $0.35 per 100k events (volume-tiered); extra throughput $1.00–$3.00 per event/s.
Takeaway: **value metrics are events/mo, retention days, throughput**. Free tier is functional but
short retention (3 days) — a durability-positioned free tier can beat it.

### Svix — flat tiers on messages/mo; all products bundled
| Plan | Price | Included | Retention | Notes |
|---|---|---|---|---|
| Free | $0 | 50k msgs/mo, 200 msg/s | 30 days | Overage $0.0001/msg |
| Professional | from $490/mo | 50k msgs/mo | 90 days | SOC 2 Type II, static IPs, white-label |
| Enterprise | Custom | Custom | Custom | SSO, audit logs, VPC peering, on-prem, HIPAA |

Takeaway: **very generous free tier (50k/mo, 30-day retention)**, then a steep cliff to $490.
The $40–$400/mo mid-market band between Svix Free and Svix Pro is thinly served.

### Convoy — flat license (self-host) + cloud
| Plan | Price | Model |
|---|---|---|
| Community (OSS) | $0 forever | Self-host, incl. auto-retries, portal links |
| Premium | $999/mo | Self-host license: transformations, RBAC, circuit breaking, white-label, OTel export |
| Enterprise | Custom | SAML SSO, on-prem support, SOC 2 support |
| Cloud | from ~$99/mo, usage-based | `Assumption:` exact cloud tiers not published on pricing page; ~$99 figure from secondary sources |

Takeaway: flat "predictable" pricing, aimed at teams that would otherwise self-host. The jump
from free OSS to $999/mo Premium leaves a wide gap for a managed product priced in between.

### Trigger.dev — subscription + compute usage
| Plan | Price | Included |
|---|---|---|
| Free | $0 | $5 credits/mo, 20 concurrent runs, 1-day log retention |
| Hobby | $10/mo | $10 credits, 50 concurrent, 7-day logs |
| Pro | $50/mo | $50 credits, 200+ concurrent, 30-day logs, +$20/seat beyond 25 |
| Enterprise | Custom | SOC 2, SSO, RBAC, HIPAA BAA |

Compute: $0.0000169–$0.00068/sec by machine size + $0.000025/run. Apache 2.0 self-host.
Takeaway: developers accept **$10–$50/mo self-serve price points** for event-processing infra.
This is the reference band for a Webhookr Pro tier, not Svix's $490.

## 3. How Webhookr wins / loses / avoids — per competitor

### vs Hookdeck
- **Win on:** visual topology (Hookdeck has none as a first-class surface), Terraform provider
  (Hookdeck exposes API/undocumented TF support — `Assumption:` no first-class provider found),
  simpler mental model, retention-forward free tier.
- **Lose on:** auto-retries, dedup, CLI/localhost, throughput SLAs, maturity, observability export.
- **Avoid:** any reliability/delivery-guarantee comparison until auto-retry ships. Do not mention
  "Issues"-style grouping. Their new agent/MCP angle — don't chase it; stay focused.

### vs Svix
- **Win on:** category clarity — a purpose-built receiving product vs a sending platform that also
  ingests; simpler pricing for consumers of webhooks; mid-market price band ($0→$490 gap).
- **Lose on:** compliance (SOC 2/HIPAA), scale/SLA trust, free-tier volume (50k/mo), signing.
- **Avoid:** competing for "send webhooks to your users" buyers — actively disqualify them in copy
  (they are Svix's ICP, not ours). Never compare on enterprise trust.

### vs Convoy
- **Win on:** zero-ops managed experience, developer-first onboarding, topology, IaC provider.
- **Lose on:** open source / self-host (hard requirement for some buyers), retries, rate limiting,
  circuit breaking, static IPs.
- **Avoid:** self-host-required deals — disqualify honestly and early. Don't fight OSS on price.

### vs Trigger.dev
- **Win on:** buy-vs-build framing: with Webhookr the capture, storage, replay, fan-out and
  audit trail exist without writing/maintaining task code; provider-agnostic record of *what
  arrived* independent of your handler's correctness.
- **Lose on:** arbitrary processing logic (they run your code; we deliver to destinations),
  open source, AI-agent zeitgeist, retries on their runs.
- **Avoid:** positioning against them as a jobs platform — frame as *complementary* ("point the
  webhook at Webhookr, fan out one delivery to your Trigger.dev task") and compete only for the
  narrow "reliable webhook intake" search intent.

## 4. Positioning conclusions (v2)
1. v1 conclusions hold: inspector-you-keep-in-production (vs bins), topology + IaC (vs gateways),
   receiving-not-sending (vs Svix).
2. **New — pricing white space:** the self-serve $10–$50/mo band (Trigger.dev's) between
   Hookdeck Team ($39 w/ 10k events) and the Svix Free→$490 cliff is the natural home for a
   Webhookr Pro tier. See `2026-07-01-pricing-strategy-v1.md`.
3. **New — Trigger.dev is an intent competitor, not a product competitor.** Content should
   capture "handle Stripe/GitHub webhooks reliably" searches with a buy-not-build story, and can
   even show Webhookr → Trigger.dev as a pattern.
4. **New — Hookdeck moved up-market and sideways** (Outpost outbound, agent-ready). Their focus
   widening is an opening for a sharply-focused "receiving end" product story.

## Sources
- Hookdeck — https://hookdeck.com/ · https://hookdeck.com/pricing
- Svix — https://www.svix.com/ · https://www.svix.com/pricing/
- Convoy — https://www.getconvoy.io/ · https://www.getconvoy.io/pricing · https://www.getconvoy.io/cloud
- Trigger.dev — https://trigger.dev/ · https://trigger.dev/pricing · https://trigger.dev/changelog/http-endpoints · https://trigger.dev/docs/documentation/concepts/triggers/webhooks
- Webhookr capabilities — local `webhookr-docs/docs/` (intro, product/replay, concepts/topology, terraform/*)
