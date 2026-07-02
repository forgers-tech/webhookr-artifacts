---
artifact: launch-plan
topic: webhookr-gtm-strategy
date: 2026-07-01
version: 1
source: manual
status: Draft
---

# Launch Plan v1 — two waves

Decision (2026-07-01): **Wave 1 = public beta soft launch now** (learning + list-building);
**Wave 2 = GA launch gated on readiness criteria**. Messaging per
`2026-07-01-marketing-strategy-v1.md`; honesty constraints from gap-report-v1 apply to every
launch asset (no reliability claims).

## Wave 1 — Public beta soft launch (target: within ~2–4 weeks)

**Goal:** signal and feedback, not scale. Success = qualified feedback threads + activated users,
not signup vanity numbers.

### Pre-launch checklist
- [ ] Home LP updated per `2026-07-01-communication-brief-v2.md` (H1 fix, beta ribbon, routing
      block, GA-waitlist capture) — *requires an implementation session*
- [ ] Signup funnel captures email + optional company; UTM tracking live
- [ ] Activation path verified end-to-end: signup → endpoint → first event < 10 min (docs Fast
      Track); fix any friction found
- [ ] `/compare/webhook-site` live (best deflection for "isn't this just a bin?")
- [ ] Status/feedback channel chosen (GitHub Discussions or a simple feedback email) — beta users
      must have somewhere to talk to us
- [ ] Error budget honesty: no SLA promised anywhere; "beta" visible in-product

### Channel sequence (staggered, not same-day)
1. **Show HN** — title suggestion: "Show HN: Webhookr – capture, inspect and replay the webhooks
   you receive". First comment: honest capability list *including* the gaps (no auto-retry yet,
   manual replay) — HN rewards this. Founder replies all day.
2. **dev.to post** (launch story crosspost) + **X thread** with topology demo GIF, same week.
3. **Targeted subreddits** (r/devops, r/webdev, r/SaaS) only where self-promo rules allow; frame
   as "I built this, tear it apart".
4. **Product Hunt: NOT in Wave 1.** PH is a one-shot asset — spend it at GA when retries/CLI/
   billing make a bigger story. (Explicit decision; revisit only if beta traction is strong.)

### Wave 1 metrics (review at +30 days)
Signups · activation rate (first event <10 min of signup) · week-2 retention of projects ·
# of substantive feedback threads · GA-waitlist emails collected.

## Wave 2 — GA launch (date = when gates pass, not a calendar date)

### Readiness gates (all required)
1. **Automatic retries + exponential backoff** shipped (unlocks "reliable delivery" messaging —
   the single biggest positioning constraint today)
2. **CLI / replay-to-localhost** shipped (DX parity hook vs Hookdeck/Webhook.site)
3. **Billing live** (Stripe + plan limits per `2026-07-01-pricing-strategy-v1.md`) + public
   pricing page (needs its own /design session)
4. **Searchable event archive** shipped (headline-able differentiator; lets the H1 finally say
   "searchable")
5. Trust page: security overview (encryption at rest, token model), uptime history from beta
   `Assumption:` SOC 2 explicitly NOT a GA gate (long lead time; start the process, market it later)

### Wave 2 assets & sequence
- New home LP revision (v3 brief) with "reliable delivery" messaging finally allowed
- **Product Hunt launch** (the saved shot) + second Show HN framed around what changed since beta
  ("we shipped retries, search and a CLI — here's what 6 months of beta taught us")
- Email the GA waitlist + beta users with founding-user discount (pricing-strategy §3)
- Comparison pages updated: flip the auto-retry rows, add pricing rows vs Hookdeck/Svix
- Launch-week content: retries deep-dive post, CLI demo video/GIF, migration guide "from
  Webhook.site/ngrok to production"

### Wave 2 metrics
Free→paid conversion · MRR · PH ranking as awareness proxy · retention cohorts vs Wave 1.

## Risks & mitigations
- **Beta launch draws enterprise-ish leads we must refuse** → routing block + disqualifier copy
  (ICP §disqualifiers) does this on-page; keep a polite "not yet" template reply.
- **HN asks "how are you not Hookdeck?"** → answer is rehearsed in marketing-strategy §3; honesty
  about gaps is the strategy, not a liability.
- **Gates slip and Wave 2 drifts** → quarterly re-review; if >6 months, consider a mid-wave
  "retries shipped" mini-launch to keep the waitlist warm.
