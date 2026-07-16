---
artifact: dependency-major-upgrade-blockers
topic: dependency-major-upgrade-blockers
date: 2026-07-16
version: 1
source: manual
status: Active
---

# Dependency major-upgrade blockers & revisit register (2026-07 sweep)

## Status

**Active register.** These major upgrades were evaluated during the 2026-07-15/16
Dependabot sweep and **deliberately not merged** because each is blocked by an
upstream incompatibility or is a full migration in disguise. This is the *why*
and — more importantly — the **revisit gate**: before anyone (human or agent)
retries "just follow the majors" for one of these, they must first re-verify the
blocker below is actually resolved. Attacking the migration before the gate is
green wastes effort and, for the Expo/RN cases, can break `main`.

This register does not duplicate the PR comments; each blocked PR carries its own
detailed explanation. It records the durable constraint and the condition that
retires it.

## Why this exists

Patch/minor Dependabot PRs sweep cleanly. **Majors do not** — a major bump is only
mergeable when the rest of the toolchain's peer graph already supports it. Several
of our majors are gated by a *single* transitive dependency that has not yet
shipped support. A bump PR looks red for an unrelated reason (`npm ci` ERESOLVE,
a plugin crash, a provider schema error), so the real blocker must be recorded or
it will be rediscovered from scratch every sweep.

## Blocked majors — revisit register

Revisit trigger = the condition that must hold **before** retrying. Verify it
(usually one `npm view … peerDependencies` or a provider changelog check) — do not
assume time has fixed it.

| Upgrade | Repos / PR | Blocker (as of 2026-07-16) | Revisit trigger |
|---|---|---|---|
| `graphql` 16 → 17 | template-bff #44, webhookr-bff #65 | Every `@apollo/server` (incl. latest 5.5.1) peers `graphql@^16.11.0`; no Apollo release supports graphql 17 | An `@apollo/server` release peers `graphql ^17`. Then bump `graphql` + `@apollo/server` **together** (combined PR). |
| `eslint` 9 → 10 | webhookr-web #70 | `eslint-plugin-react@7.37.5` (latest, pulled by `eslint-config-next`) still calls the removed `context.getFilename()` and peers `eslint … \|\| ^9.7` | `eslint-plugin-react` ships an eslint-10-compatible release **and** `eslint-config-next` depends on it. devDep — low priority. |
| `typescript` 6 → 7 | webhookr-toggles #5 | `typescript-eslint@8.64.0` (latest) peers `typescript >=4.8.4 <6.1.0`; no version supports TS 7 (the native "Corsa" port) | `typescript-eslint` raises its `typescript` upper bound to include 7. Then combined bump. |
| `@react-native-async-storage/async-storage` 2 → 3 | webhookr-mobile #14 | Expo SDK 56 expects `2.2.0`; v3 removed the bundled jest mock (`.../jest/async-storage-mock`) → Unit Tests fail. Expo-SDK-locked. | Bundled with an **Expo SDK upgrade** that expects async-storage 3. Never bump independently. |
| `cloudflare/cloudflare` provider 4 → 5 | terraform #103 | Two blockers: (1) `modules/cloudflare/versions.tf` still pins `~> 4.0` → contradictory constraint; (2) provider v5 is a ground-up rewrite (`cloudflare_record`→`cloudflare_dns_record`, `cloudflare_zone_settings_override` removed, schema changes across ruleset/pages/r2/origin_ca) — needs resource rewrite + state migration + **likely DNS-record replacement (downtime risk)** | Treat as a **dedicated migration**, not a bump: align the module constraint, rewrite resources to the v5 schema, add `moved`/`state mv` blocks, and review the real plan (use the on-demand PR plan check, below) resource-by-resource before apply. |

## Verified safe and merged (for contrast)

- **`grafana/grafana` provider 3 → 4** (terraform #89) — real `terraform plan` = `No changes`. Merged.
- **`@types/node` 25 → 26** (webhookr-mobile #13) — all real checks green (typecheck passes). Merged.

## Expo/RN-managed packages — do not sweep

In `webhookr-mobile`, these follow the Expo SDK and must move **only** with an Expo
SDK upgrade — bumping any of them independently desyncs the peer graph and breaks
`main` (it did, twice, on 2026-07-15: `@react-native/jest-preset` 0.85→0.86 and
`react-dom` 19.2.3→19.2.7, both reverted in webhookr-mobile #20):

`@react-native/*`, `react`, `react-dom`, `react-native`,
`@react-native-async-storage/async-storage`, `expo`, `expo-*`.

**Action recommended:** add them to Dependabot `ignore` in `.github/dependabot.yml`
and to `expo.install.exclude` in `package.json`. For `0.x` deps, remember
`0.85 → 0.86` is a **breaking** change, not a minor.

## Tooling: validate provider majors before merge

The on-demand **Terraform Plan (PR)** check (`terraform/.github/workflows/terraform-plan-pr.yml`,
gated to Dependabot + maintainers via the `tf:plan` label) runs a read-only
`terraform plan` on the PR head using the trusted `main` machinery, and posts the
plan (or the setup/init error) as a sticky PR comment. This is the pre-merge signal
that `terraform validate -backend=false` cannot give (provider-schema breaks,
resource replacements). Use it as the gate for #103 and any future provider major.

## How to re-evaluate before retrying a major

1. Re-run the **revisit trigger** check for the specific upgrade (peer range /
   provider changelog). If not satisfied, stop — the blocker still holds.
2. If satisfied, bump the blocking dependency **and** the target together in one PR.
3. For provider majors, get a green **Terraform Plan (PR)** with no destructive
   changes before merge.
4. For Expo/RN packages, only as part of an Expo SDK upgrade — never in a sweep.
