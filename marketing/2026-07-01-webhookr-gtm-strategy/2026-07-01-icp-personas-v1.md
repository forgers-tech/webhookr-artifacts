---
artifact: icp-personas
topic: webhookr-gtm-strategy
date: 2026-07-01
version: 1
source: manual
status: Draft
---

# ICP & Personas v1 — Webhookr

Formalizes the audience sketch in `2026-06-30-communication-brief-v1.md` §2. Capabilities
referenced are verified in `webhookr-docs`; market context from
`2026-07-01-competitive-analysis-v2.md`.

## 1. Ideal Customer Profile

**One-liner:** engineering teams whose product depends on webhooks they *receive* from third
parties, who have been burned (or fear being burned) by silent failures and lost events, and who
manage infrastructure as code.

| Dimension | Profile |
|---|---|
| Company | SaaS, e-commerce, fintech, marketplaces — products built on 2+ external providers (Stripe, GitHub, Shopify, Typeform, payment/logistics APIs) |
| Size | ~5–200 engineers. Below 5: solo devs on the free tier (funnel, not revenue). Above ~200: enterprise requirements we can't meet yet (SOC 2, SSO, SLAs) |
| Roles involved | Backend / integrations engineer (user + champion), platform/DevOps engineer (user + influencer), eng lead (buyer) |
| Tech stack signals | Terraform in use (strong signal — we ship a provider), TypeScript/Node or similar API-first stack, GitHub, cloud-native |
| Volume band | ~10k–1M inbound events/mo — above bins, below "we built a Kafka pipeline for this" |
| Buying trigger events | (a) incident: provider outage or silent webhook failure caused data loss / a paged on-call; (b) audit or debugging session that needed "what exactly did Stripe send us at 3:07?"; (c) platform team standardizing integrations as code; (d) N-th consumer added to one event source (fan-out pain) |
| Willingness to pay | Self-serve card, $10–$50/mo band (see competitive-analysis-v2 §2, Trigger.dev reference) |

### Disqualifiers (route away, honestly)
- **Needs to SEND webhooks to their own customers at scale** → Svix's ICP, not ours.
- **Hard self-host / data-residency requirement** → Convoy OSS. (Webhookr is managed SaaS only.)
- **Requires SOC 2 / HIPAA / SSO today** → not yet; capture for the GA-wave waitlist.
- **Requires automatic retries as a contractual guarantee** → gap today; be explicit (replay is
  manual). Capture for GA wave.

## 2. Personas

### P1 — The Integrations Engineer ("paged at 3am")
- **Context:** owns Stripe/Shopify/GitHub integrations at a SaaS company; webhooks feed billing,
  fulfillment, or sync logic.
- **Pains (verified against docs/intro pain list):** no visibility into what was actually sent;
  events lost during their own outages; every incident starts with guessing.
- **Desired outcome:** the full record of every inbound request (payload, headers, source IP,
  timing) and one-click replay after a fix.
- **Watering holes:** HN, r/webdev / r/ExperiencedDevs, dev.to, Stripe/Shopify dev communities.
- **Objections:** "another hop adds latency/risk" · "can I trust you with payload data?"
  (answer: AES-256-GCM envelope encryption at rest — verified in `webhookr-svc`) · "what if
  *you* go down?" (honest answer: beta, no SLA yet).
- **Buys when:** the week after an incident. Content should meet them mid-incident (SEO:
  "replay stripe webhook", "webhook failed silently").

### P2 — The Platform Engineer ("if it's not in Terraform it doesn't exist")
- **Context:** platform/DevOps team serving many product squads; standardizes how services
  receive events.
- **Pains:** webhook endpoints configured by hand in provider dashboards; no audit trail; every
  squad reinvents intake.
- **Desired outcome:** projects, endpoints, destinations and tokens declared in Terraform,
  reviewed in PRs (verified: first-class Terraform provider). Topology graph as the shared map.
- **Watering holes:** r/devops, r/Terraform, HashiCorp community, platform-engineering Slacks.
- **Objections:** "is the provider maintained?" · "can I import existing setups?" (roadmap —
  do not claim) · "multi-user / RBAC?" (gap — email+password + OAuth GitHub/Google only).
- **Buys when:** standardization initiatives, quarterly platform planning.

### P3 — The Solo Dev / Early Founder ("just show me the payload")
- **Context:** building an MVP against Stripe/GitHub; uses Webhook.site or ngrok today.
- **Pains:** bins delete after days and cap requests; nothing survives to production.
- **Desired outcome:** an endpoint in seconds that they *keep* — same URL from first test to prod.
- **Watering holes:** X, indie-hacker communities, YouTube tutorials, Product Hunt.
- **Objections:** "is the free tier actually usable?" · "will you charge me suddenly?"
- **Role in strategy:** funnel volume + word of mouth, not revenue. Converts to P1 as they grow.

## 3. Anti-persona
The enterprise procurement-led buyer (needs compliance paperwork, vendor review, SLAs, SSO).
Politely defer until the GA wave (`2026-07-01-launch-plan-v1.md` readiness gates).
