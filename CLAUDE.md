# CLAUDE.md — webhookr-artifacts

This repository is the **artifact store** for Webhookr marketing and product-communication work
(landing pages, briefs, competitive analysis, launch copy, etc.), most of it produced by the
`/design` workflow.

Its purpose is **history**: every artifact is versioned and dated so we can see how the messaging
and positioning evolved over time. Artifacts are never overwritten in place.

## General rules

- **Never invent or assume any information.** If there is any doubt, stop and ask. Every product
  claim in an artifact must be grounded in a verifiable source (codebase, docs, product UI) or a
  cited external source. Label assumptions explicitly as `Assumption:`.
- **Everything in English** inside the artifacts (copy, specs, reports).

## Storage convention (required)

```
marketing/
  <YYYY-MM-DD>-<topic-slug>/            # one folder per generation session, date-prefixed
    <YYYY-MM-DD>-<artifact-name>-v<N>.md
```

Rules:

1. **Folder** — a new folder per session, named `<YYYY-MM-DD>-<topic-slug>`
   (e.g. `2026-06-30-webhookr-landing-page`). Use the date the work was produced.
2. **File** — every artifact file is prefixed with the same date and suffixed with a version:
   `<YYYY-MM-DD>-<artifact-name>-v<N>.md` (e.g. `2026-06-30-communication-brief-v1.md`).
3. **Versioning** — a revision of an existing artifact is a **new file** with an incremented
   `v<N>`. Never edit or delete a previous version; history must be preserved.
4. **Front-matter** — every artifact starts with:

   ```yaml
   ---
   artifact: <name>
   topic: <topic-slug>
   date: <YYYY-MM-DD>
   version: <N>
   source: /design | manual | other
   status: Draft | Reviewed | Approved
   ---
   ```

5. **Index** — add one line per new artifact to `marketing/INDEX.md` (newest first) so the history
   is scannable without walking the tree.

## Artifact types (typical `<artifact-name>` values)

`communication-brief`, `interface-spec`, `competitive-analysis`, `gap-report`,
`opportunity-report`, `chain-retrospective`, `design-command-improvement-report`,
`launch-copy`, `email`, `announcement`.

## Artifacts workflow (source of truth)

This repository **is** the org's long-term source of truth. Across every repo, code is an
implementation of these artifacts, not the primary record — so the workflow both consumes
knowledge from here before coding and contributes knowledge back here after.

### Before (consume)

Before producing or revising an artifact — and before any implementation in a consuming repo —
read the existing artifacts for the topic first (walk the relevant `INDEX.md`). Understand the
current architecture and product decisions, and verify whether a governing artifact — RFC, ADR,
User Story, PMM, Security Matrix / Threat Model, or Operational guide / Runbook — already exists.
Never contradict a documented decision silently: a revision is a new `v<N>` (never overwrite),
and anything unverified is labelled `Assumption:`.

### After (reconcile)

After any implementation, the `artifact-reconciliation` standard (forgers-tech `skills` plugin)
decides whether durable knowledge changed and lands the update here: architecture / ADR / RFC,
security matrix / threat model, runbook or operational procedure, infrastructure or deployment
docs, API documentation, product behavior / User Story / PMM, or a reusable engineering
standard — always explaining **why** the decision exists rather than duplicating the
implementation. The *why* lives here under `engineering/` (or `marketing/`); the operational
what/how lives next to the code and cross-links back.
