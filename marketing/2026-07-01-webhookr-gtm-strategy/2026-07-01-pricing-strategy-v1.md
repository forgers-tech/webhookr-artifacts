---
artifact: pricing-strategy
topic: webhookr-gtm-strategy
date: 2026-07-01
version: 1
source: manual
status: Draft
---

# Pricing Strategy v1 — Webhookr (INTERNAL — not for publication)

Decision (2026-07-01): pricing stays an internal recommendation until billing exists. No public
pricing page. Competitor numbers verified 2026-07-01 — see
`2026-07-01-competitive-analysis-v2.md` §2 for the full teardown and sources.

## 1. Model recommendation

**Self-serve subscription + soft usage limits.** Not pure usage-based (Hookdeck's model demands
metering infra and scares small teams); not flat-high (Convoy's $999 and Svix's $490 leave the
mid-market open). Anchor the paid tier in the **$10–$50/mo band developers already accept for
event infra** (Trigger.dev: Hobby $10, Pro $50).

**Value metrics (in order):**
1. **Events/month** — scales with customer value, industry-standard axis (Hookdeck, Svix).
2. **Retention days** — OUR differentiated axis. Durability is the positioning ("the inspector
   you keep"); sell retention, don't ration it as an afterthought.
3. Destinations / projects — generous, used as tier fences not meters.
4. Seats — free early (team features barely exist); introduce as a Team-tier fence later.

## 2. Proposed tiers

| | **Free** | **Pro — $20/mo** | **Team — $99/mo** |
|---|---|---|---|
| Events/mo | 10,000 | 100,000 (then $2/100k) | 500,000 (then $1.50/100k) |
| Retention | **7 days** | **30 days** | **90 days** |
| Projects | 2 | 10 | Unlimited |
| Destinations/endpoint | 3 | 10 | Unlimited |
| Seats | 1 | 3 | Unlimited |
| API tokens / Terraform | ✓ | ✓ | ✓ |
| Topology / replay | ✓ | ✓ | ✓ (bulk replay when shipped) |

Design logic:
- **Free beats Hookdeck Developer on retention** (7d vs 3d) at equal volume (10k) — the free tier
  *demonstrates* the durability positioning. It stays below Svix Free's 50k msgs, which we don't
  need to match: different job, and Svix Free exists to feed a $490 tier.
- **Pro at $20** undercuts Hookdeck Team ($39 with only 10k events included) while including 10×
  the events; sits inside the Trigger.dev-validated band. Overage $2/100k undercuts Hookdeck's
  entry overage ($3/100k).
- **Team at $99** owns the Svix-Free→$490 white space and the Convoy OSS→$999 gap for teams that
  want managed + longer retention.
- Everything IaC/topology/replay in every tier: differentiators are for *adoption*, not upsell.

`Assumption:` price points are hypotheses to validate in beta interviews; the structure (metrics
and fences) matters more than the exact numbers.

## 3. Beta policy
- All limits OFF or generous during public beta; banner: **"Free during beta."**
- Commitment to publish: beta users get **30 days notice** before billing starts + a founding-user
  discount (suggest 50% for 12 months on Pro/Team). Builds trust, creates the Wave-2 launch list.
- Instrument usage NOW (events/mo, retention reads, destinations per endpoint) to validate fences
  with real distributions before publishing prices.

## 4. Prerequisites before pricing goes public
1. Billing implementation (Stripe) + plan-limit enforcement in `webhookr-svc` (no billing code
   exists today — verified).
2. Usage metering + in-dashboard usage page.
3. Auto-retry shipped (a paid "reliability" story without it invites the comparison we avoid).
4. Pricing page brief (future `/design` session) — do not publish tiers before 1–3.

## 5. Open questions
- Annual discount at launch or later? (Suggest later; monthly-only reduces beta friction.)
- Does retention >90d become an Enterprise fence or a paid add-on? Depends on storage costs
  (Neon) — measure during beta.
- Throughput (events/sec) as a future fence — Hookdeck charges for it; ignore until someone hits
  a real limit.
