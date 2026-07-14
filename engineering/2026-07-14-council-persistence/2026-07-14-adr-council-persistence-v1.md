---
artifact: adr-council-persistence
topic: council-persistence
date: 2026-07-14
version: 1
source: manual
status: Approved
---

# ADR-007: `/council` persists the Council Record to the artifacts store

## Status

Approved (2026-07-14) — shipped in `forgers-tech/skills` v1.7.0 ([PR #26](https://github.com/forgers-tech/skills/pull/26)).

Extends [ADR-005](../2026-07-14-decision-ledger-sdlc/2026-07-14-adr-decision-ledger-sdlc-v1.md) and [ADR-006](../2026-07-14-council-multi-agent/2026-07-14-adr-council-multi-agent-v1.md): the Decision-Ledger contract and the multi-agent orchestration are unchanged; this ADR changes only what happens to the record **after** the council decides.

## Context

Under ADR-005/006, `/council` is the hard entry gate of the feature lifecycle and owns every engineering decision, emitting the **Verified Story + Decision Ledger** that `/upstream`, `/gatekeeper`, `/downstream`, and `artifact-reconciliation` all consume **by stable ID**.

But the command stopped at *rendering* the Council Record into the chat response. Persisting it was left implicit — a human had to notice and copy it into `webhookr-artifacts/` by hand. That is a real gap in a forward-only, ledger-referencing lifecycle:

- Every later stage is documented as taking `<Council Record path>` as input. If the record only ever exists in a transcript, the next command has nothing on disk to point at, and the `D-#`/`US-#` references it depends on are not durably addressable.
- The whole value of the ledger — that lost, invented, or drifted decisions are *mechanically* detectable across stages — assumes the ledger is a persisted artifact, not a scrollback. A transcript is lost on a new session; the decisions of record must outlive it.
- The org's own standard (this repository is the source of truth; `artifact-reconciliation` lands durable knowledge here) already says decisions belong on disk. `/council` was the one lifecycle stage not honoring it at the moment it produced the most decision-dense artifact of all.

This surfaced in practice: a Critical council (adopt Infisical as the secret platform) closed with a full ledger that existed only in chat until it was persisted manually — the trigger for closing the gap in the command itself.

## Decision

Make persistence a **mandatory Phase 7 (Scribe) step**, not an afterthought. After composing the record, the Scribe **must** write it to the project's long-term engineering-artifacts store and report the path; a record that exists only in the chat transcript is a **failed close** (Completion Guard).

- **Discover, don't hardcode.** The Scribe locates the store rather than assuming a repo name: a sibling `*-artifacts/engineering/`, an `engineering/`/`docs/adr/` tree, or whatever the project's `CLAUDE.md` / `artifact-reconciliation` standard designates — preferring where existing ADRs/specs already live. If none exists, it falls back to `docs/councils/` in the current repo and says so. This keeps the command portable across every consuming repo, consistent with ADR-006's runtime-agnostic, inline-contract design.
- **Follow the store's own convention.** Dated folder `<date>-<slug>-council/`, versioned file `<date>-council-record-v1.md`, required frontmatter, and an `INDEX.md` line if the store has one — mirrored from a neighbouring artifact, not invented.
- **Persist the whole record.** Execution Trace + Plan + Evidence + Round-1 summaries + Consensus Matrix + Revised Positions + Challenge–Response + Council Decision + Verified Story + Decision Ledger. Every register is referenced forward, so nothing is dropped for brevity.
- **Faithful, not editorial.** The Scribe copies the negotiated record to disk; it does not add, soften, or silently resolve anything in the writing — the same anti-invention rule ADR-006 places on the specialists.
- **Reopen writes a new version.** A council reopen (§15) writes `-v<N+1>` beside the prior file; council history is never overwritten — matching this repository's immutable-versioning rule.
- **Surfaced in the output.** A new `## Persisted Artifacts` section reports the written path(s) so the next stage can be pointed straight at the file (`/upstream <persisted path>`).

Persisting the Council Record is explicitly **not** the same as publishing the ADR / Security-Matrix / runbook updates a decision may require — those remain `artifact-reconciliation`'s job after implementation. The record only *records that they are required*, as a `D-#`/`US-#`.

## Why this exists (rationale, not implementation)

- **A decision that isn't written down didn't happen.** The lifecycle treats the Council Record as an addressable, ID-bearing input to every downstream stage; persistence is what makes that literally true instead of aspirational.
- **Close the loop at the source.** The org standard already lands durable knowledge in this repo after implementation; making `/council` self-persist means the most decision-dense artifact is captured at the moment it is produced, not reconstructed later from memory.
- **Portable by construction.** Discovery-not-hardcoding keeps one command correct across every repo — the same reason ADR-006 kept specialist contracts inline and runtime-agnostic.
- **History is preserved, not rewritten.** Versioned reopen files mean the evolution of a decision stays auditable, consistent with the repository's never-overwrite rule.

## Consequences

- Every council run now ends with a file write + index update in the artifacts store; a run that fails to persist is not a successful close. This is a deliberate, small cost paid once per council.
- Consuming repos without a recognizable artifacts store get a `docs/councils/` fallback and an explicit note, rather than a silent skip — so the behavior is predictable everywhere.
- The implementation (Phase 7 wording, Artifact Flow, Execution Graph, Final Response Format, Completion Guard, version bump) lives in `forgers-tech/skills`; this ADR records only the **why**. They cross-link: see `commands/council.md`, `CLAUDE.md` in the `skills` repo.

## References

- `forgers-tech/skills` — mandatory Council-Record persistence [PR #26](https://github.com/forgers-tech/skills/pull/26) (v1.7.0).
- Command implementation: `commands/council.md` (Phase 7, §9, §16, §17).
- First persisted record: [`2026-07-14-infisical-secret-platform-council/`](../2026-07-14-infisical-secret-platform-council/2026-07-14-council-record-v1.md).
- Related: [ADR-006](../2026-07-14-council-multi-agent/2026-07-14-adr-council-multi-agent-v1.md) — multi-agent `/council`; [ADR-005](../2026-07-14-decision-ledger-sdlc/2026-07-14-adr-decision-ledger-sdlc-v1.md) — the Decision-Ledger operating system this serves.
