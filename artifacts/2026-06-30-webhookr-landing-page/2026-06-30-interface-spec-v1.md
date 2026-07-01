---
artifact: interface-spec
topic: webhookr-landing-page
date: 2026-06-30
version: 1
source: /design
status: Draft
---

# Interface Specification — Webhookr landing page

Target: `webhookr-web/src/app/(public)/home/page.tsx` (Next.js 16 / React 19 / Tailwind 4).
Source brief: `2026-06-30-communication-brief-v1.md`. **Implemented** in v1 of the page.

## Information architecture
Single scroll: hero → problem → how it works → features → use cases → quickstart → FAQ → final CTA.
Reuses the existing public `SiteHeader` / `SiteFooter`.

## Structure & wireframe
1. Hero — Badge + H1 + subhead + [Get started][Read the docs] + flow chips (Provider → Endpoint →
   Event → Destinations → Delivery).
2. Problem — short muted band.
3. How it works — 5 numbered steps (1→2→5 columns).
4. Features — Card grid, 6 items (icon + title + one-liner).
5. Use cases — checked list, 5 items.
6. Quickstart — real `curl` from docs (`api.webhookr.tech/v1/ingest/<slug>` → `202 Accepted`).
7. FAQ — native `<details>` (server component, no client JS).
8. Final CTA — brand band → `/signup`.

## Design system (reuse only)
`@/shared/components/ui/{button,card,badge}` · `layout/{site-header,site-footer}` · `lucide-react`.
Tokens: `brand-*`, `neutral-*`, radius; Inter; dark mode via `dark:` classes. No new dependencies.

## Responsive / a11y / performance
Mobile-first; CTAs and flow chips stack on mobile. Skip link, `aria-labelledby` per section, icons
`aria-hidden`, WCAG AA contrast. Server component; no hero image (CSS/SVG flow); `export const metadata`.

## Implementation notes (as built)
Replaced template copy verbatim from the brief; kept the existing hero/skip-link patterns; FAQ uses
native `<details>` for zero client JS. Passed `typecheck`, `eslint`, `prettier`.

## Open questions
- `TODO:` optional comparison section (see competitive-analysis) — not yet added to the page.
- `TODO:` add a smoke test per `webhookr-web` CLAUDE.md convention.
