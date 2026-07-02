---
artifact: marketing-strategy
topic: webhookr-gtm-strategy
date: 2026-07-01
version: 1
source: manual
status: Draft
---

# Marketing Strategy v1 — Webhookr

Inputs: `2026-07-01-competitive-analysis-v2.md`, `2026-07-01-icp-personas-v1.md`, v1 artifact set
(2026-06-30). Constraint carried from gap-report-v1: **never claim delivery-reliability
superiority** (no auto-retry/dedup today; replay is manual).

## 1. Positioning statement

> For engineering teams that depend on webhooks they **receive** from third-party providers,
> **Webhookr** is a developer-first webhook platform that keeps a durable, replayable record of
> every event and fans it out to every service that needs it — unlike inspection bins that forget,
> gateways you have to operate, and sending platforms built for the other direction.

**Category framing:** "webhook infrastructure for the receiving end" (keep from v1 — it cleanly
excludes Svix's territory and names the job).

## 2. Messaging house

**Roof (key message):** Stop losing and guessing at webhooks. See every request, replay any
event, deliver to every service.

**Pillar 1 — The inspector you keep in production** (vs bins)
Full record of every request — payload, headers, source IP, timing — durable, not deleted in
7 days. The same endpoint from first test to production.
*Proof:* event record schema in docs; no request cap; destinations + per-delivery status.

**Pillar 2 — See your event flow** (unique surface)
A live topology graph of Source → Endpoint → Destination. The mental model, visualized — for
onboarding, incident triage, and demos.
*Proof:* `concepts/topology` docs + dashboard page. No competitor leads with a visual model
(competitive-analysis-v2 §1).

**Pillar 3 — Webhooks as code** (platform wedge)
Projects, endpoints, destinations and tokens in Terraform, reviewed like everything else.
*Proof:* published Go Terraform provider + docs.

**Supporting messages:** fan-out with per-delivery tracking (`PENDING|SUCCESS|FAILED`); replay an
event or a single delivery on demand; encrypted at rest (AES-256-GCM).

**What we do NOT say (until shipped):** "guaranteed delivery", "automatic retries", "never miss
an event" (we say "never *lose* an event" — storage claim, verified), "enterprise-ready",
"full-text search" (filtering by endpoint/timestamp only today).

## 3. Category conversations — how to answer "how are you different from…"
- **Webhook.site / bins:** "They show you a request, then forget it. We keep it — and deliver it."
- **Hookdeck / Convoy (gateways):** "Gateways optimize the pipe. We optimize what you can *see*
  and *do about it* — full record, replay, a topology you can look at, Terraform you can commit.
  If you need managed auto-retries today, Hookdeck is the right choice — we're honest about that."
- **Svix:** "Svix helps you send webhooks to your customers. We help you consume the ones you
  receive. Different job."
- **Trigger.dev / build-it-yourself:** "You can write a handler with retries on a jobs platform —
  you'll still want the record of what actually arrived, independent of your code being right.
  Many teams point the provider at Webhookr and fan out to their job runner."

## 4. Channel strategy (lean budget, solo founder)
Priority order:
1. **SEO / comparison & alternative pages** — highest intent, compounding; see
   `2026-07-01-comparison-pages-brief-v1.md` and `2026-07-01-content-roadmap-v1.md`.
2. **Docs-as-marketing** — docs.webhookr.tech already strong; add integration guides
   (Stripe/GitHub/Shopify) that rank and convert.
3. **Terraform ecosystem** — provider listed in the registry, example modules, a "webhooks as
   code" post; uncontested watering hole (P2).
4. **Community launches** — HN Show HN, dev.to, X threads, Product Hunt (timing per launch plan).
5. **No paid acquisition** during beta; revisit at GA.

## 5. Objection handling (top 5)
| Objection | Response (honest) |
|---|---|
| "Extra hop = extra failure point" | Provider→Webhookr ingestion is a store-then-deliver design (BullMQ queue); your destination being down no longer means data loss — the event is stored and replayable. No SLA yet (beta) — say so. |
| "You don't auto-retry" | Correct, today replay is manual (event or single delivery). Auto-retry is on the roadmap (GA gate). If contractual retries are required now, we'll say Hookdeck fits better. |
| "Can I trust you with payloads?" | AES-256-GCM envelope encryption at rest, API tokens with hashing/expiry/revocation. No SOC 2 yet — on the GA path. |
| "Why not Webhook.site free?" | 7-day deletion and request caps; no production destinations, no replay to real services, no IaC. |
| "Why not self-host Convoy?" | Valid choice if self-host is a requirement. Webhookr is for teams that don't want to run the gateway. |

## 6. Conversion strategy
- **Primary CTA everywhere:** create a free account → first endpoint → first captured event
  (time-to-first-event is THE activation metric; docs "Fast Track" curl flow already exists).
- **Secondary CTA:** read the docs.
- **Beta framing:** "Free during beta" banner; collect email + company on signup; in-product
  waitlist prompt for GA features (retries, teams, SOC 2) to build the Wave-2 launch list.
- Trust elements are still TODO (no logos/testimonials yet) — use *specific product truth*
  (screenshots of topology, delivery statuses, TF snippets) instead of invented social proof.

## 7. Metrics
Beta wave: signups, activation rate (first event received < 10 min), weekly active projects,
qualitative feedback threads. GA wave: conversion to paid, MRR, retention cohorts.
