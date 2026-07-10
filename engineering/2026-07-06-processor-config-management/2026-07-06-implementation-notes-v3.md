---
artifact: implementation-notes-processor-config-management
topic: processor-config-management
date: 2026-07-06
version: 3
source: manual
status: Reviewed
---

# Processor Config Management — Implementation Notes

Records how the [feature spec v2](2026-07-06-feature-spec-processor-config-management-v2.md)
was implemented, and where the as-built code deviates from it. Supersedes
[v2](2026-07-06-implementation-notes-v2.md): the only change is **deviation #4** (the web
navigation surface), refined again below after the project config moved to its own page
(2026-07-10). Everything else in v2 still holds — see it for the full deviation list (standard Nest
exceptions, 3-reason rejection metric, project On/Off toggle, auth-guard naming, Playwright
deferred).

## Changelog v3 (2026-07-10)

The v2 project surface was a `Processors` tab on the **global Settings** screen — a scope mismatch
(project-scoped config living under account/workspace settings), chosen only because projects had
no detail screen. Projects now have their **own page**, so the config is contextual per scope.
Shipped in `webhookr-web` [#87](https://github.com/forgers-tech/webhookr-web/pull/87)
(merge `0a6b67d`).

## Deviation #4 — Web surface (final)

Both scopes configure processors on **their own screen**, via an `Overview` / `Processors` tab pair
(mirrors the `event-detail` tab pattern):

- **Endpoint** → `/dashboard/endpoints/[endpointId]` — `Overview` (endpoint `DashboardStats`) /
  `Processors` (`EndpointProcessorSettings`, On/Off/Inherit tri-state). Unchanged since v2.
- **Project** → **new** `/dashboard/projects/[projectId]` page (`ProjectDetailPage`) — `Overview`
  (project-scoped `DashboardStats`) / `Processors` (`ProjectProcessorSettings`, On/Off). The project
  **name in the projects list now links** to this page.

Superseded intermediate states (kept for history):
- **v1:** standalone unlinked route `/dashboard/projects/[projectId]/processors` (discoverability
  gap) — removed.
- **v2:** `Processors` tab on the global **Settings** screen scoped to the selected project — removed
  in #87; the `ProjectProcessorSettingsTab` component was deleted and Settings returned to its single
  `API tokens` tab.

Rationale for the final shape: the app had no per-project detail screen, so v1/v2 improvised. Giving
projects a page (like endpoints already have) puts each scope's config where the user manages that
scope, and matches the endpoint pattern exactly.

## Deployment

All layers in production via GitOps/ArgoCD: svc `4e365c0`, bff `d4f59d4`, web `0a6b67d` (latest;
progression `08fc76e` → `5798188` tabs → `0a6b67d` project page). The global flag is off and no
PRODUCT processor is registered yet, so the catalog is empty and the UI shows the empty state —
ingest / delivery behavior is unchanged until the first PRODUCT processor ships.
