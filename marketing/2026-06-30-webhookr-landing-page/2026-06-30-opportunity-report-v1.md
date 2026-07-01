---
artifact: opportunity-report
topic: webhookr-landing-page
date: 2026-06-30
version: 1
source: /design
status: Draft
---

# Opportunity Report — what Webhookr could have that competitors under-serve

Scope: differentiation angles where Webhookr either already leads or could credibly lead, based on
what competitors do **not** emphasize. Each item notes whether it builds on something Webhookr
**already has** (verified in `webhookr-docs`) or would be **new**.

## 1. Visual topology as a first-class product (already has → deepen)
Webhookr already ships a **topology graph** (Source → Endpoint → Destination). None of Hookdeck,
Convoy, Svix, or Webhook.site lead with a live visual model of event flow.
- Deepen: per-edge health/latency, live event animation, click-through from a node to its events,
  "where did this event go?" tracing.
- Why it wins: turns an abstract pipeline into something a team can *see* — great for onboarding,
  incident triage, and demos.

## 2. IaC-native webhook platform (already has → lead with it)
Webhookr has a **first-class Terraform provider**. Convoy/Hookdeck expose APIs; a polished,
documented TF provider is a genuine wedge for platform teams.
- Deepen: drift detection, `terraform import` of existing setups, a published module, examples in CI.
- Why it wins: "commit your webhook infrastructure" is a message platform/DevOps buyers respond to
  and that inspection bins and most gateways don't own.

## 3. Bin-to-production continuity (new positioning of existing capability)
Webhook.site is loved for zero-friction inspection but is explicitly **not production** (7-day
deletion, 100-request cap, no path to prod — sourced). Webhookr captures the full record **and** is
durable + delivers to real destinations.
- Opportunity: an instant, no-config endpoint (like a bin) that you simply *keep* — same URL from
  first inspection to production. A migration-free path competitors force you to abandon.

## 4. Searchable event archive (new; leverages existing storage)
Every event is already stored with full payload + headers + source IP + timing. Competitors focus on
*delivery*, not on the stored corpus.
- Opportunity: full-text / structured search and filtering over historical events ("show every
  `order.created` from Stripe last week with a failed delivery"). A debugging superpower.

## 5. Replay ergonomics as a headline feature (already has → productize)
Replay exists; make it a *product surface*, not a button.
- Opportunity: bulk replay with filters, replay-to-localhost for dev, replay from a CLI, and
  "replay since <incident time>" — turning outage recovery into a one-liner.

## 6. The combination itself (positioning)
No single competitor combines **inspect (bin) + durable store + fan-out delivery + visual topology +
Terraform IaC** in one developer-first product. Svix is outbound; Webhook.site is ephemeral;
Hookdeck/Convoy are gateways without the visual/IaC emphasis. The bundle is the moat.

## Recommended near-term messaging bets (defensible today)
1. **"The inspector you keep in production."** (vs bins)
2. **"See your event flow — topology built in."** (unique surface)
3. **"Webhooks as code."** (Terraform wedge)

## Recommended product bets (to widen the moat) — for the team, not the LP
1. Automatic retries + backoff (closes the #1 competitive gap; unlocks "reliable delivery" messaging).
2. Replay-to-localhost + CLI (steals Webhook.site / Hookdeck's DX hook).
3. Searchable event archive (unique, high-value, leverages what's already stored).

## Sources
- Webhook.site limits — https://hookdeck.com/webhooks/platforms/webhook-site-alternatives
- Hookdeck — https://hookdeck.com/ · Convoy — https://www.getconvoy.io/ · Svix — https://www.svix.com/
- Webhookr docs — local `webhookr-docs/docs/` (intro, concepts/topology, terraform/*, product/replay)
