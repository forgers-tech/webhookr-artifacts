---
artifact: communication-brief
topic: webhookr-gtm-strategy
date: 2026-07-01
version: 2
source: manual
status: Draft
---

# Communication Brief v2 — Webhookr home landing page

Revises `2026-06-30-webhookr-landing-page/2026-06-30-communication-brief-v1.md` with the GTM
strategy (`2026-07-01-marketing-strategy-v1.md`) and formal ICP
(`2026-07-01-icp-personas-v1.md`). **Spec only — no code changes in this session.**

## Changes from v1 (rationale)
1. **H1 fix:** v1 said "captured, **searchable**, and replayable" — full-text search is a known
   gap (gap-report-v1); only endpoint/timestamp filtering exists. v2 uses "inspectable".
2. **Audience-routing block added** (from ICP disqualifiers): say early who this is NOT for —
   builds trust with a technical audience and pre-empts the Svix/Convoy confusion.
3. **Beta banner + GA waitlist** capture added (feeds Wave-2 launch list, launch-plan-v1).
4. **"How it compares" strip added** linking to future `/compare/*` pages (SEO + objection
   handling on the spot).
5. Everything else in v1 (positioning, features block, CTAs, SEO) carries over — it was verified
   and remains accurate.

## Refined copy (v2 deltas only; v1 copy stands where not mentioned)
- **H1:** Every webhook you receive — captured, inspectable, and replayable.
- **Beta ribbon (new):** Free during beta · You'll get 30 days notice before any billing starts.
- **Audience routing block (new, after features):**
  - "Sending webhooks *to your* customers? You want an outbound platform like Svix — Webhookr is
    for the receiving end."
  - "Need self-hosting? Convoy's open-source gateway is a great fit — Webhookr is managed only."
  - "Need automatic retries with SLAs today? We're honest: replay is manual while auto-retry is
    in development. Join the waitlist to hear when it ships."
- **Comparison strip (new):** "How Webhookr compares → vs Webhook.site · vs Hookdeck · vs Svix ·
  vs Convoy · vs Trigger.dev" (links to `/compare/*`, see comparison-pages brief).
- **GA waitlist micro-CTA (new, footer of routing block):** email capture, label "Get notified:
  auto-retries, teams, SOC 2."

## Unchanged from v1 (still verified)
Eyebrow · subhead · why-Webhookr block · six feature cards (capture/inspect, never lose an event,
replay, fan-out, topology, webhooks-as-code) · CTAs (Get started → `/signup`; Read the docs) ·
SEO title/description.

## Open TODOs (carried)
- Real trust elements still absent — do not invent; use product screenshots (topology, delivery
  table) as evidence.
- Revisit H1 to "searchable" only when the searchable event archive ships.
