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

## Artifact reconciliation

After completing any implementation, apply the `artifact-reconciliation` standard
(forgers-tech `skills` plugin): if the change created or modified long-term
knowledge — architecture/ADR, security matrix/threat model, runbook, API docs,
product behavior, or an engineering standard — update the corresponding artifact,
explaining the **why** rather than duplicating the implementation. Durable
rationale lives in `webhookr-artifacts/engineering`; operational docs live next
to the code.
