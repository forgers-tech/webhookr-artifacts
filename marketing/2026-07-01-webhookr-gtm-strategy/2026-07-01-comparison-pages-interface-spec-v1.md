---
artifact: comparison-pages-interface-spec
topic: webhookr-gtm-strategy
date: 2026-07-01
version: 1
source: manual
status: Draft
---

# Comparison Pages Interface Spec v1 — one template, five pages

For future implementation in `webhookr-web` (Next.js App Router). One route
`/compare/[competitor]` driven by a content map; copy per page in
`2026-07-01-comparison-pages-brief-v1.md`. Follow the visual language of the existing home LP
(`src/app/(public)/home/page.tsx`): Tailwind v4, Inter, dark-mode aware, existing button/card
primitives.

## Layout (top → bottom)
1. **Hero** — breadcrumb ("Compare / {X}"), H1, verdict paragraph (max 3 lines), beta ribbon
   reused from home.
2. **Twin decision cards** — side-by-side "Choose {X} if…" / "Choose Webhookr if…" bullet cards;
   equal visual weight (honesty is the design statement). Stack on mobile.
3. **Feature table** — sticky first column; cells ✓ / ✗ / "partial" with superscript footnote
   markers; footnotes below table with source links (competitor docs/pricing URLs). Max ~10 rows.
4. **Deep-dive sections** — 2–3 alternating text/visual blocks; visuals = real product
   screenshots (topology graph, delivery-status table), never invented UI.
5. **FAQ** — accordion, 3–4 items, FAQPage structured data.
6. **CTA band** — primary "Get started" → `/signup`; secondary "Read the docs"; on pages where a
   gap feature is decisive (Hookdeck/Convoy: auto-retry), inline GA-waitlist email field labeled
   "Get notified: auto-retries, teams, SOC 2".

## Components
- Reuse: header/footer, buttons, card, ribbon from home LP.
- New: `ComparisonTable` (accessible `<table>`, `scope` attrs, footnote refs), `DecisionCards`,
  `FaqAccordion` (disclosure pattern, keyboard-operable).

## Content model (per page)
`{ slug, competitorName, title, metaDescription, verdict, chooseThem[], chooseUs[],
tableRows[{feature, them, us, footnote?}], sections[{heading, body, image?}], faq[{q,a}],
waitlistVariant?: boolean }` — plain TS map in the repo; no CMS.

## SEO/meta
Per-page title/description from brief §SEO; canonical `/compare/{slug}`; OG image = topology
screenshot with competitor name overlay (`Assumption:` OG template to be produced with brand
assets in `brand/`); comparison pages linked from home "How Webhookr compares" strip and from
docs sidebar "Compare" section.

## Accessibility
Table readable by screen readers (real `<th scope>`), ✓/✗ cells carry `aria-label`
("supported"/"not supported"), accordion uses `button` + `aria-expanded`, color never the only
signal in table cells.
