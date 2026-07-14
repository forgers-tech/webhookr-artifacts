---
artifact: adr-decision-ledger-sdlc
topic: decision-ledger-sdlc
date: 2026-07-14
version: 1
source: manual
status: Approved
---

# ADR-005: The feature lifecycle becomes a Decision-Ledger operating system

## Status

Approved (2026-07-14) — shipped in `forgers-tech/skills` v1.4.0 ([PR #20](https://github.com/forgers-tech/skills/pull/20)); execution-time traceability added in v1.5.0 ([PR #22](https://github.com/forgers-tech/skills/pull/22)).

Builds on [ADR-004](../2026-07-10-downstream-command/2026-07-10-adr-downstream-command-v1.md), which added `/downstream` and first described the pipeline as `/upstream → /gatekeeper → /downstream`.

## Context

The feature pipeline had grown to five slash commands (`/council`, `/upstream`, `/gatekeeper`, `/downstream`, plus the parallel `/design`) plus the `artifact-reconciliation` standard. Benchmarking `/council` against `/upstream` exposed a structural fault: **the commands were independent prompts, not a system.** Multiple stages re-performed the same reasoning — product discovery, architecture design, business-rule creation, risk analysis.

The root cause was a single broken seam: **`/upstream` never consumed the Council Record.** Its input was a raw idea/User Story, and its Stage 1 re-ran product discovery while its Stage 3 re-derived the architecture and ADR — all of which `/council` had already decided. Downstream consequences followed: duplicated reasoning, inconsistent decisions, wasted tokens, decision drift, and artifacts diverging over time. `/gatekeeper` read only the Feature Specification, so it structurally *could not* detect a lost or invented decision. There was no shared decision-identity scheme linking a spec section back to the decision that authorized it, and no canonical place for the immutability rule ("a downstream stage may not change an upstream decision") — the one instance of it lived orphaned inside `council.md`.

The `/council` benchmark that surfaced this is recorded at [`2026-07-11-processor-execution-timeline-council`](../2026-07-11-processor-execution-timeline-council/2026-07-11-council-record-v1.md).

## Decision

Redesign the pipeline from a set of prompts into a **forward-only operating system built on a single authoritative Decision Ledger**, with one owner per decision and a stable identity that flows forward and is never redefined.

```
[User Story card] → /council → /upstream → /gatekeeper → /downstream → artifact-reconciliation
     (input)         (decide)    (author)     (audit)      (implement)      (verify + sync)
```

- **The User Story is an input, not a command.** It is the card text pasted into `/council`. There is deliberately **no `/user-story` command**.
- **`/council` is the hard entry gate.** It admits the card against a fixed 4-item bar (problem, affected domain, actor, at least one testable acceptance criterion) — a deterministic `COUNCIL BLOCKED` when the card is too thin — normalizes it into a **Verified Story** with `US-#` acceptance criteria, and produces the **Decision Ledger**: `D-#` decisions (each serving a `US-#`), plus `A-#` assumptions, `Q-#` open questions, `R-#` risks, `X-#` rejected/deferred. Council decides the *shape* of things; it no longer authors a Feature Specification.
- **`/upstream` authors, it does not discover.** It requires the Council Record (blocks without it), expands each `D-#` into implementation-ready detail, and cites `⟶ D-#` on every section.
- **`/gatekeeper` audits fidelity.** It now also consumes the Council Record and runs a Traceability Audit — lost decisions, invented requirements, decision drift — routing defects to Council (decision-level) or Upstream (authoring).
- **`/downstream` implements with both.** The Feature Specification is the *what*; the Ledger is the *why*. It cites `D-#`/`BR-#` in PRs.
- **`artifact-reconciliation`** runs a decision-survival audit (card → `D-#` → `BR-#` → code) before syncing durable knowledge.
- **Decision immutability, enforced everywhere.** Any stage that needs to change a decision **stops and reopens `/council`** rather than self-resolving. The guard now lives in each stage's own file instead of only in `council.md`.

Separately, every command's output document (Council Record, Feature Specification, Gatekeeper Review, implementation report, Communication Brief, Interface Specification) now carries an **Execution Trace** table — `Started / Ended / Duration / Revision` — where `Started` is stamped once and is immutable, and `Ended`/`Duration`/`Revision` are recomputed on each revision.

## Why this exists (rationale, not implementation)

- **Make decisions once, then only expand them.** The lifecycle's value is lost if each stage re-derives what an earlier stage already settled. A single Decision Ledger, owned by `/council`, is the source of *why*; every later stage consumes it and adds detail, never re-opening the reasoning.
- **Turn drift into a mechanical check.** A shared, forward-only ID namespace (`D-#`, `US-#`, `BR-#`) is the keystone: "lost decision," "invented requirement," and "decision drift" become things a stage can *compute* by diffing artifacts, not qualities that depend on reviewer diligence. This is only possible once `/gatekeeper` and `artifact-reconciliation` actually receive the Ledger.
- **Determinism at the front door.** A fixed admission checklist means the same thin card blocks the same way every time, pushing quality to the source (the card) instead of letting malformed input propagate downstream unchecked.
- **One authority per decision.** Decision immutability plus an explicit reopen path removes the ambiguity of "which stage owns this?" — the reason artifacts used to diverge. Changing a decision is always a reopen of its owner, never a silent local edit.
- **Traceable execution.** The Execution Trace records elapsed time per artifact across revisions, giving the lifecycle a lightweight, immutable-at-the-start audit of how long each stage's output took to reach its current revision.

## Consequences

- The pipeline is now five commands, not six: the User Story is the card fed to `/council`, which absorbed the entry-gate responsibility. `/design` remains a separate, parallel communication lifecycle, unchanged.
- Every consuming repo that installs the plugin gets the new contracts. Because the Ledger IDs flow forward, a Feature Specification section, a business rule, a PR, and a reconciled artifact can all be traced back to the `D-#` that authorized them.
- `/upstream`, `/gatekeeper`, and `/downstream` now **hard-block** without the Council Record when running the standard lifecycle path — a deliberate cost: the pipeline refuses to run half-informed.
- This is a reusable engineering capability, so the **implementation** (command bodies, templates, guards, versioning, packaging) lives in `forgers-tech/skills`; this ADR records only the **why**. The two cross-link: see `commands/{council,upstream,gatekeeper,downstream,design}.md`, `skills/artifact-reconciliation/SKILL.md`, and `CLAUDE.md` in the `skills` repo.

## References

- `forgers-tech/skills` — lifecycle redesign [PR #20](https://github.com/forgers-tech/skills/pull/20) (v1.4.0); execution-time traceability [PR #22](https://github.com/forgers-tech/skills/pull/22) (v1.5.0); version bump [PR #23](https://github.com/forgers-tech/skills/pull/23).
- Command implementations: `commands/council.md`, `commands/upstream.md`, `commands/gatekeeper.md`, `commands/downstream.md`, `commands/design.md`.
- Related: [ADR-004](../2026-07-10-downstream-command/2026-07-10-adr-downstream-command-v1.md) (`/downstream`); the `artifact-reconciliation` standard (same repo) — now the decision-survival + knowledge-sync tail of the loop.
- Benchmark that surfaced the duplication: [`2026-07-11-council-record-v1`](../2026-07-11-processor-execution-timeline-council/2026-07-11-council-record-v1.md).
