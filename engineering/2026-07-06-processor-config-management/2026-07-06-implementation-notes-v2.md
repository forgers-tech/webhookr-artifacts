---
artifact: implementation-notes-processor-config-management
topic: processor-config-management
date: 2026-07-06
version: 2
source: manual
status: Reviewed
---

# Processor Config Management — Implementation Notes

Records how the [feature spec v2](2026-07-06-feature-spec-processor-config-management-v2.md)
was implemented, and the places the as-built code deviates from it. Supersedes
[v1](2026-07-06-implementation-notes-v1.md): the only change is **deviation #4** (the web
navigation surface), updated below after a UI-discoverability follow-up (2026-07-10). Everything
else in v1 still holds.

## Changelog v2 (2026-07-10)

The v1 project surface was a **standalone, unlinked route** — reachable only by URL. It was
replaced with **configuration tabs**, so both scopes are discoverable in place. Shipped in
`webhookr-web` [#86](https://github.com/forgers-tech/webhookr-web/pull/86) (merge `5798188`).

## Shipped

Three layers, each an additive change mirroring an existing module. No Prisma migration (the
`ProjectProcessorConfig` / `EndpointProcessorConfig` tables already shipped with the engine story,
migration `20260706000000_event_processor_engine`).

- **`webhookr-svc`** — new `src/processor-config/` module: `ProcessorCatalogController`
  (`GET /v1/processors`), `ProjectProcessorConfigController` + `EndpointProcessorConfigController`
  (GET / PUT / DELETE), `ProcessorConfigService`, `ProcessorConfigRepository`,
  `SetProcessorConfigDto`, view/catalog interfaces, two Prometheus counters. `EventProcessingModule`
  now additively exports `EventProcessorRegistry` + `PipelineResolver`. Code-side records:
  `docs/adr/0004-processor-config-management.md` and `docs/processor-config.md`. 32 unit + 14 e2e
  tests; full svc suite green. ([#128](https://github.com/forgers-tech/webhookr-svc/pull/128))
- **`webhookr-bff`** — new `src/processor-configs/` GraphQL module (catalog + project/endpoint
  queries and set/reset mutations) proxying svc via `WebhookrClientService`; the `ProcessorConfig`
  type carries the composite `id`. Added a `put<T>()` method to `WebhookrClientService`.
  ([#76](https://github.com/forgers-tech/webhookr-bff/pull/76))
- **`webhookr-web`** — new `src/features/processor-configs/` (data hooks + `ProcessorSettingsList`
  + `ProcessorSettingsSection`). ([#83](https://github.com/forgers-tech/webhookr-web/pull/83);
  tabs in [#86](https://github.com/forgers-tech/webhookr-web/pull/86)).

## Deviations from spec v2

1. **Errors use standard Nest exceptions, not custom `error` codes.** Spec §5 shows response
   bodies with `error: "ProcessorNotFound" | "ProcessorNotConfigurable" | "ProcessorUnavailable"`.
   The repo has **no** global exception filter, and the delivery replay gate (the nearest
   precedent) throws plain `NotFoundException` / `ConflictException` with descriptive messages.
   Adding a custom error-code class + filter would be over-engineering for one module, so the
   service follows the existing precedent. **HTTP status codes match the spec exactly** (unknown →
   404, PLATFORM → 409, platform-off `PUT` → 409, invalid body → 400, ownership → 404, reset →
   204); only the JSON `error` label differs (standard `Not Found` / `Conflict`). Clients
   distinguish the two 409s by message.

2. **Rejection metric emits three reasons, not four.** `processor_config_rejections_total{reason}`
   is incremented for `unknown | not_configurable | unavailable` (the validations the service
   raises directly). `not_owned` is **not** emitted as a rejection label — ownership failures
   surface as the standard opaque 404 from `ProjectService` / `EndpointService.findOne` and are
   already visible via the HTTP metrics interceptor and logs. The `not_owned` value was dropped
   from the `ProcessorConfigRejectionReason` type (Copilot review) so it reflects only the reasons
   actually emitted.

3. **Project scope is a two-value toggle in the UI (no explicit "reset to default" control).**
   Spec §3 describes project = On/Off toggle; the `resetProjectConfig` API/mutation exists and is
   tested, but the project UI exposes only the On/Off `Switch` (endpoint keeps the full
   On/Off/Inherit control, where Inherit = reset). Matches the spec's described UI; the project
   reset endpoint is available for future use / API clients.

4. **Web surface is tabbed, not a standalone route (updated v2).** The v1 project surface was a
   dedicated route `/dashboard/projects/[projectId]/processors` with **no navigation link** — a
   discoverability gap. It was replaced (2026-07-10, #86) with **configuration tabs**:
   - **Endpoint** detail page gained `Overview` / `Processors` tabs (mirrors the `event-detail`
     tab pattern); the Processors tab hosts the On/Off/Inherit settings.
   - **Project** processors are a `Processors` tab on the **Settings** screen, scoped to the
     currently selected project (`ProjectContext`), with a loading state before the "select a
     project" prompt (Copilot review). There is no per-project detail screen to host it, so
     Settings — the app's existing tabbed configuration surface — is the home.
   - The orphaned standalone route + `project-processors-page` component were removed.

5. **Auth guard naming.** The spec says "`JwtAuthGuard` + ownership"; the as-built controllers add
   no per-route guard and rely on the **global `CompositeAuthGuard`** (JWT-based, `APP_GUARD`),
   exactly like the other resource controllers (e.g. `EndpointController`). Ownership is enforced in
   the service. Clarified in `docs/processor-config.md` after Copilot review; no behavior change.

Everything else shipped as specified: catalog lists all PRODUCT processors; read model reuses the
event-free `PipelineResolver.resolve` and applies the global `webhookr.processors.enabled` gate on
top (IR-1 — `engineEnabled` / `source = engine_disabled`); writes allowed while the engine is
globally off but rejected for a per-processor platform-off processor (BR-C10 / EF-C3); `config
Json` never written; tri-state endpoint via row-absence; BFF composite `id` for Apollo
normalization with refetch-on-reset (IR-3); ownership mirrors `EventService.findOne`.

## Not done / follow-up

- **Playwright e2e (web).** The spec lists a Playwright flow. It needs a running Next + BFF (+ auth)
  stack that isn't available in this environment, so it was not added; the Vitest component +
  repository tests cover the web logic (toggle project, endpoint override, inherit reset, empty
  state, platform-/engine-disabled read-only rows, tab navigation, loading state). Tracked as
  follow-up.
- **`.env.example`.** No change — this story introduces no new flag (it reads the engine's existing
  global + per-processor flags).

## Deployment

All layers merged to `main` and deployed to production via GitOps/ArgoCD: svc `4e365c0`, bff
`d4f59d4`, web `08fc76e` (initial) then `5798188` (tabs). The global flag is off and no PRODUCT
processor is registered yet, so the catalog is empty and the UI shows the empty state — ingest /
delivery behavior is unchanged until the first PRODUCT processor ships.
