---
artifact: communication-brief
topic: webhookr-landing-page
date: 2026-06-30
version: 1
source: /design
status: Draft
---

# Communication Brief — Webhookr landing page

Ground rule: every claim is verified against `webhookr-docs`. Competitive framing from
`2026-06-30-competitive-analysis-v1.md`; do not over-claim (see `2026-06-30-gap-report-v1.md`).

## 1. Artifact
- **Type:** landing page (`webhookr-web` `/home`, previously template boilerplate).
- **Objective:** in ~5s make a developer grasp what Webhookr does and start a free account.
- **Primary action:** sign up (`/signup`). Secondary: read the docs (`https://docs.webhookr.tech`).

## 2. Audience
- **ICP:** backend / platform / integrations engineers who **consume** third-party webhooks
  (Stripe, GitHub, Shopify, Typeform) or route events between services.
- **Personas:** integrations engineer (paged on silent failures), platform engineer (durability +
  fan-out + IaC), developer testing locally.

## 3. Positioning & message
- **Positioning:** For developers who depend on inbound webhooks, Webhookr is a developer-first
  platform that captures, inspects, replays, and fans out every webhook — so events are never a
  black box and never lost.
- **Key message:** Stop losing and guessing at webhooks. See every request, replay any event,
  deliver to every service.
- **Differentiation (defensible today):** visibility (full record) + on-demand replay + fan-out with
  per-delivery tracking + **visual topology** + **Terraform/IaC** + developer-first simplicity.

## 4. Customer (verified pain, docs/intro)
No visibility into what was sent · events lost during outages · painful local dev · one source, many
consumers (fan-out) · no infrastructure as code.

## 5. Verified capabilities (source: webhookr-docs)
Endpoints w/ unique slug · full event record (payload, headers, source IP, timing) · destinations
with fan-out · per-delivery status `PENDING|SUCCESS|FAILED` · **manual** replay (event or single
delivery) · topology graph · projects · API tokens (`whk_…`) · Terraform provider · dashboard;
auth = email + password.

## 6. Refined copy (attract & retain — v1)
- **Eyebrow:** Webhook infrastructure for the receiving end
- **H1:** Every webhook you receive — captured, searchable, and replayable.
- **Subhead:** Not a disposable bin, and not a queue you have to run. Point any provider at a
  Webhookr endpoint and keep a durable, replayable record of every event — in development and in
  production.
- **Why-Webhookr block:** Inspection tools show you a request, then forget it in 7 days. Queues and
  gateways make you run infrastructure. Webhookr keeps the full, durable record and lets you replay
  to real destinations — with a topology you can see and Terraform you can commit.
- **Features:**
  - Capture & inspect — See exactly what arrived — full payload, headers, source IP, timing. Not a
    bin that deletes it in 7 days.
  - Never lose an event — Every event is stored durably, so an outage means a replay — not lost data.
  - Replay on demand — Re-send any stored event, or a single delivery, after a fix or an outage.
  - Fan-out delivery — One event, many destinations, each delivery tracked independently.
  - Topology you can see — A live graph of Source → Endpoint → Destination — the mental model,
    visualized.
  - Webhooks as code — Commit your setup: manage projects, endpoints, and destinations with the
    Terraform provider.
- **CTAs:** Primary "Get started" → `/signup`; Secondary "Read the docs" → `https://docs.webhookr.tech`.

## 7. SEO
- Title: Webhookr — Capture, inspect & replay your webhooks
- Description: Webhookr is a developer-first platform to receive, inspect, replay, and fan out
  webhooks. A full record of every event, durable storage, on-demand replay, and a Terraform provider.

## 8. Assumptions & open questions
- `Assumption:` target app is `webhookr-web`; LP replaces `/home` template content. (Applied.)
- `TODO:` real trust elements (logos, metrics, testimonials, compliance) — none exist yet; not invented.
- `TODO:` product decision — add automatic retries to unlock "reliable delivery" messaging (gap report).
