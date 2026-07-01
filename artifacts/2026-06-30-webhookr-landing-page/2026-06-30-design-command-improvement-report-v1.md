---
artifact: design-command-improvement-report
topic: webhookr-landing-page
date: 2026-06-30
version: 1
source: /design
status: Applied
---

# Report — improvements to the `/design` command & skills

Derived from running `/design landing-page` on Webhookr. Items marked **[applied]** were committed
to the `forgers-tech/skills` plugin in this session.

1. **PMM: mandatory competitive research with sources. [applied]**
   `product-marketing-manager` must output a competitor scan (with citations), a **verified
   capabilities** table, and an explicit **"not-yet-supported" gaps** list. Prevents the `TODO` and
   the over-claim seen this run.
2. **Artifact persistence. [applied]**
   `/design` now instructs persisting each stage output to a dated artifact store
   (`<YYYY-MM-DD>-<topic>/<YYYY-MM-DD>-<artifact>-vN.md`) when one exists, else inline.
3. **Design: anchor in the target repo's design system. [applied]**
   `design-engineer` must read the target repo's tokens + brand guidelines + component inventory
   before specifying.
4. **Frontend: close the verification loop. [applied]**
   Stage 3 must run typecheck/lint, produce a visual check (run + screenshot), and add the smoke
   test the target repo requires.
5. **Human approval gate between Stage 2 and Stage 3. [applied]**
   For code-producing artifacts, pause for review before writing code.
6. **Optional comparison-section trigger. [deferred]**
   A flag/segment to add a competitor comparison with the same sourcing discipline.
7. **Anti-overclaim as a named principle. [applied]**
   Beyond "never invent capabilities," require verifying **absences** before claiming a
   differentiator ("we don't just X, we Y" only if Y is verified).

## Not done yet
- Item 6 (comparison-section trigger) — deferred; low effort, add on request.
