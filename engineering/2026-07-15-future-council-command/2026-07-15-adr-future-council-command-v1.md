---
artifact: adr-future-council-command
topic: future-council-command
date: 2026-07-15
version: 1
source: manual
status: Approved
---

# ADR-008: `/future-council` — a foresight profile of `/council`

## Status

Approved (2026-07-15) — shipped in `forgers-tech/skills` v1.8.0 ([PR #27](https://github.com/forgers-tech/skills/pull/27)).

Builds on [ADR-006](../2026-07-14-council-multi-agent/2026-07-14-adr-council-multi-agent-v1.md) (multi-agent `/council`) and [ADR-007](../2026-07-14-council-persistence/2026-07-14-adr-council-persistence-v1.md) (mandatory Council-Record persistence): it reuses that orchestration engine and persistence discipline unchanged, and adds a **second, standalone entry point** with a different goal and output. It does **not** touch the [ADR-005](../2026-07-14-decision-ledger-sdlc/2026-07-14-adr-decision-ledger-sdlc-v1.md) delivery lifecycle.

## Context

`/council` (ADR-005/006) is the hard entry gate of the **feature-delivery lifecycle**: it takes one User Story card, admits it against a four-item bar, and refines a *single decision* into a Verified Story + Decision Ledger that `/upstream`/`/gatekeeper`/`/downstream` consume by stable ID. Its whole shape is tuned to "make one scoped decision correctly."

A different, recurring need does not fit that shape: **strategy foresight** — "which capabilities become inevitable for our category over a multi-year horizon, and which can we build into durable infrastructure?" This has no card, no testable acceptance criteria, no single decision; it wants a wide specialist roster, exploratory rounds (emergent ideas, first-principles redesign), a market-timing lens, and outputs aimed at *founders and future architects*, not at an implementer.

This was validated empirically before the command existed: on **2026-07-15** the full "Future Council — Webhookr 2030" specification was run *through* `/council` by pasting it as the objective. It worked — 14 specialists, 5 rounds, 33 independent agent executions, a persisted Council Record ([`2026-07-15-future-council-webhookr-2030/`](../2026-07-15-future-council-webhookr-2030/2026-07-15-council-record-v1.md)) — but only because a ~4,000-word bespoke prompt carried all the foresight structure by hand. That is not repeatable: the roster, the five-round protocol, the reasoning/time budget, and the four-artifact output would have to be re-authored, correctly, every time. The run proved the *pattern*; the pattern deserved to be a first-class primitive.

Two forces made "just paste the prompt again" the wrong answer: (1) the foresight structure is elaborate and easy to get subtly wrong (drop a round, forget the Economist's timing assessment, merge the four artifacts); (2) the long multi-round run hit real session-limit and transient-error failures mid-council — surviving them required a file-first, resume-don't-restart discipline that must be encoded, not rediscovered live.

## Decision

Add **`/future-council`** as a **foresight profile of `/council`** — a distinct command in the skills plugin, not a fork of the engine.

- **Inherit the engine, change the goal.** It reuses `/council`'s Prime Directive verbatim (orchestrate never simulate; never invent consensus; preserve dissent; block if independent agents can't be launched) and its persistence discipline (ADR-007). It overrides only three things: the *goal* (discover inevitable future capabilities vs. refine one card), the *roster + rounds* (below), and the *output* (four artifacts vs. one record).
- **Foresight admission, not the User-Story bar.** Discovery has no testable acceptance criteria, so the four-item card gate is replaced by a lighter bar: a real strategic question + a product/domain scope + a horizon (default 2026–2030). It still may never *invent* the question or scope to pass.
- **Fixed 14-specialist futures roster** over **five rounds** (Independent Thinking → Cross Review → Emergent Opportunities → First Principles → Strategic Prioritization), with an explicit reasoning/time budget. The roster adds futures/market seats `/council` lacks — Industry Futurist, AI Systems Architect, Product Strategist, VC Partner, Blue Ocean Strategist, permanent Red Team, permanent Technology Historian, and a Technology Economist who owns a per-finalist Technology Timing Assessment.
- **Four separate, audience-specific artifacts** (never merged, no shared paragraphs): `council-record` (source of truth + Decision Ledger + Final Synthesis), `executive-summary` (founder/CTO), `action-plan` (engineering leadership; a strategy→execution transform, not a roadmap), `knowledge-base` (durable principles/laws/mental-models for future councils). All four persist to the artifacts store under ADR-007's convention.
- **Encode the resilience.** File-first specialist outputs (full report to disk, short summary returned) plus resume-don't-restart are written into the command as operational rules, because the live run proved a long council will hit failures that only that discipline survives.

`/future-council` is **standalone** — it is not a stage of the forward-only delivery lifecycle. Its Council Record still ends in a Decision Ledger, so its output *can* feed `/upstream` when a specific "Build Now" decision is chosen for implementation, but the foresight run itself makes no implementation commitment.

## Why this exists (rationale, not implementation)

- **Two genuinely different jobs deserve two entry points.** Refining one card and discovering a category's future are different admission rules, rosters, rounds, and audiences. Overloading `/council` with a mode flag would blur the lifecycle's hard-gate identity; a sibling command keeps each one legible.
- **Encode the proven structure so it is repeatable and correct.** The 2026-07-15 run showed the pattern works but costs a large bespoke prompt each time; a command turns that into a primitive that can't silently drop a round or a required artifact.
- **Reuse, don't fork, the anti-simulation engine.** The value of `/council` is that conclusions come from real independent executions. `/future-council` layers on that guarantee rather than reimplementing it, so the two can't drift apart.
- **Four audiences, four artifacts.** A foresight run's readers — the founder deciding, the eng lead sequencing, the next council reusing the learning — need different documents. Forcing them into one record (as the lifecycle `/council` does) would serve none of them well; separating them is the durable decision.
- **Resilience is a design property, not luck.** A council spanning dozens of executions *will* meet a session limit or a transient error; making outputs file-first and executions resumable is what lets it finish instead of restarting.

## Alternatives rejected

- **Add a `--foresight` mode to `/council`.** Rejected: it would dilute the lifecycle gate's single responsibility and couple two admission models, two rosters, and two output contracts into one file. Revisit only if the two commands start duplicating large shared bodies of text that must stay in sync.
- **Keep pasting the bespoke prompt into `/council`.** Rejected: not repeatable, easy to get wrong, and it re-derives the resilience discipline live every time.
- **A fully independent orchestrator that re-implements the engine.** Rejected: it would fork the Prime Directive and the persistence rules, guaranteeing drift from `/council`; the profile approach keeps one engine.

## Consequences

- The plugin now has **two** council entry points; `CLAUDE.md`/`README.md` mark `/future-council` explicitly as standalone (not a lifecycle stage) so it isn't mistaken for a `/upstream` predecessor by default.
- A foresight run now ends with **four** file writes + an index update, and a run missing any of the four is an incomplete close — a deliberate, larger cost than a lifecycle council's single record, paid once per foresight exercise.
- Because it is a *profile*, changes to `/council`'s Prime Directive or persistence propagate conceptually to `/future-council`; the command references `council.md` for the full protocol rather than copying it, which is the intended coupling.
- The implementation (roster, five-round protocol, four-artifact templates, resilience rules, version bump 1.7.0→1.8.0) lives in `forgers-tech/skills`; this ADR records only the **why**. They cross-link: see `commands/future-council.md`, `commands/council.md`, `CLAUDE.md` in the `skills` repo.

## References

- `forgers-tech/skills` — add `/future-council` [PR #27](https://github.com/forgers-tech/skills/pull/27) (v1.8.0).
- Command implementation: `commands/future-council.md` (§0 Prime Directive inheritance, §4 roster, §5 five rounds, §6 four artifacts, §7 resilience, §8 completion guard).
- Validating run: [`2026-07-15-future-council-webhookr-2030/2026-07-15-council-record-v1.md`](../2026-07-15-future-council-webhookr-2030/2026-07-15-council-record-v1.md) — the Future Council executed via `/council` before the command existed.
- Related: [ADR-006](../2026-07-14-council-multi-agent/2026-07-14-adr-council-multi-agent-v1.md) — multi-agent `/council` engine reused here; [ADR-007](../2026-07-14-council-persistence/2026-07-14-adr-council-persistence-v1.md) — the persistence discipline the four artifacts follow; [ADR-005](../2026-07-14-decision-ledger-sdlc/2026-07-14-adr-decision-ledger-sdlc-v1.md) — the delivery lifecycle `/future-council` deliberately sits outside.
```
