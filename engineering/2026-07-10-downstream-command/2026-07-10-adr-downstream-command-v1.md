---
artifact: adr-downstream-command
topic: downstream-command
date: 2026-07-10
version: 1
source: manual
status: Approved
---

# ADR-004: `/downstream` — the implementation stage of the feature pipeline

## Status

Approved (2026-07-10) — shipped in `forgers-tech/skills` v1.2.0 ([PR #18](https://github.com/forgers-tech/skills/pull/18)).

## Context

The `skills` plugin already gave us the first two stages of a feature's life as slash
commands:

- `/upstream` — turns an idea or User Story into an implementation-ready **Feature Specification**.
- `/gatekeeper` — reviews that specification and decides whether it is **ready for engineering**.

The stage that turns an approved specification into working, review-ready code was still ad hoc:
an operator would open a spec and prompt the agent freehand, so the implementation quality
(evidence discipline, test coverage, observability, documentation, rollout safety) depended on
how the request happened to be phrased that day. There was no single, repeatable entry point for
"now build this the way we build things here."

A draft `downstream.md` existed in the repo but did **not** work as a command: it had no
frontmatter, no `$ARGUMENTS` binding, and its body referenced a process that was defined *after*
the reference — so Claude Code could not invoke it as a real slash command.

## Decision

Add `/downstream` as the **implementation stage** of the pipeline, completing the loop:

```
/upstream  →  /gatekeeper  →  /downstream
 (spec)        (review)         (implement)
```

`/downstream` takes an approved task or specification (task, User Story, Feature Specification,
RFC, issue, plan, or repo/branch context) and drives it to a complete, production-ready,
review-ready implementation. It is deliberately **not** a single-persona command: it acts as the
implementation owner and dynamically applies whichever Staff+ perspectives the change actually
needs (backend, frontend, platform/DevOps, security, data, observability, testing).

## Why this exists (rationale, not implementation)

- **Close the pipeline.** A spec that is produced (`/upstream`) and approved (`/gatekeeper`) needs
  a matching, repeatable way to be *executed*. Without it, the quality gate we built upstream is
  lost the moment implementation starts.
- **Encode the org's engineering standards once.** The command bakes in the non-negotiables we
  otherwise re-explain every time: evidence-first reasoning (never invent APIs, schema, flags,
  infra), repository-first decisions (reuse existing patterns over "better" new ones), a
  no-hallucination policy, security-by-default, observability-by-default, real tests, updated
  documentation, and safe rollout/rollback. It is the executable counterpart to the
  `artifact-reconciliation` standard: `artifact-reconciliation` keeps *knowledge* in sync after a
  change; `/downstream` governs *how the change is made*.
- **Reduce operator variance.** One invocation yields a consistent implementation report
  (Implemented / Main changes / Specialist considerations / Validation / Tests / Security &
  operational notes / Remaining issues), so reviews start from a known shape.

## Consequences

- Every consuming repo that installs the plugin (project-scoped or global `~/.claude/`) gets
  `/downstream` alongside `/upstream`, `/gatekeeper`, `/design`.
- The command is intentionally opinionated about stopping: it halts on missing mandatory inputs
  rather than guessing, and only asks the user when a decision materially changes product
  behavior, contracts, billing, destructive data handling, or irreversible architecture.
- This is a reusable engineering capability, so the **implementation** (the command body, its
  frontmatter, versioning, packaging) lives in `forgers-tech/skills`; this ADR records only the
  **why**. The two cross-link: see `commands/downstream.md`, `README.md` and `CLAUDE.md` in the
  `skills` repo.

## References

- `forgers-tech/skills` — `commands/downstream.md`, PR #18, plugin v1.2.0.
- Related pipeline commands: `/upstream`, `/gatekeeper` (same repo).
- Related standard: `artifact-reconciliation` (same repo) — the knowledge-sync half of the loop.
