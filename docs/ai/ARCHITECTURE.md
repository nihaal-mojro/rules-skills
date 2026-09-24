---
owner: platform
last_verified: 2026-09-22
---

# Mojro Backend — Architecture Map

Not a Cursor rule — no frontmatter trigger, not glob-scoped. `@`-reference
this manually (e.g. in Plan Mode) when a task spans services. Kept short on
purpose; per-service detail belongs in that service's own `docs/`.

## Request flow (standard shape, every service)

```
Client
  │  HTTP (Vert.x route)
  ▼
<Service>API            — param extraction, validation, DeliveryOptions
  │  event bus (Constants.WORKER_MESSAGE_OPERATION_HEADER_KEY + address from QueueNames)
  ▼
<Service>Worker          — deserialize, read headers, dispatch on Operations enum
  │
  ▼
I<Service>*Service        — business logic, @Transactional
  │
  ▼
DAO (Postgres) / IMasterDataService (Elasticsearch)
```

Context (`enterpriseId`, `role`, `userAuthId`, hierarchy) is injected by
`ParentServiceContextLoader` from the JWT before the route handler runs —
never a client-supplied param. See `docs/ai/platform-utilities.md`.

## `api-common` — shared library layer

| Module | Purpose |
|---|---|
| `mojro-base` | Spring context wiring (`ParentServiceContextLoader`), base JPA entities (`Entity`/`VersionEntity`), event-bus/context/audit-outbox utilities, `BaseAPI` |
| `mojro-common` | DTOs, constants (`QueueNames`, `Constants`), exceptions, `ErrorMapping`, pure utils (`DateUtil`, `JacksonUtil`, `CommonUtil`) |
| `mojro-dm` | Parent/dependency-management POM for all modules — no application code |
| `mojro-resources` | Locale-specific resource bundles (property files only, no Java) |
| `mojro-cache-common` | Shared cache domain objects (`CacheDevice`, `CacheTrip`, …) |
| `mojro-bot-common` | Document categorization / ML model utilities (chatbot support) |
| `mojro-proto-schema` | Protobuf schemas + generated classes (e.g. address matching, pickup-date recommendation) |
| `mojro-hazelcast-server` | Standalone Hazelcast cluster server |
| `mojro-local-dm` | Local dependency-management variant — no application source |

## `api/*` — services

One row per service confirmed present under `api/`. Purpose inferred from
naming/package where not independently verified — treat as a starting point,
correct entries as you learn more (this doc is meant to be edited).

| Service | Purpose (best known) |
|---|---|
| `dock-api` | Warehouse dock/bay booking — availability grid, slot booking, approve/reject. |
| `audit-api` | Read-only audit-trail query API; ingests via Kafka outbox events from producer services. See `create-audit` skill. |
| `enterprise-api` | Enterprise/hierarchy management, addresses. Uses the audit outbox pattern (`AddressService`). |
| `carrier-api` | Carrier/vendor profile and related master data. Uses the audit outbox pattern. |
| `supplier-api` | Supplier-side trip/activity workflows. Uses the audit outbox pattern extensively (`ActivityService`, `TripActionService`, `TripEditService`, `TripStageService`). |
| `shipper-api` | Shipper-side order/trip workflows. Uses the audit outbox pattern. |
| `indents-api` | Indent creation/lifecycle (trip requests). Uses the audit outbox pattern. |
| `common-api` | Cross-cutting platform APIs, including enterprise master-data CRUD (`MasterDataAPI`). See `mojro-masterdata` skill. |
| `auth-api` | Authentication/JWT issuance. |
| `admin-api` | Admin/back-office operations. |
| `aggregator-api` | Cross-service data aggregation for UI/reporting. |
| `pricing-api` | Rate cards / pricing calculation. |
| `tracker-api` | Vehicle/shipment tracking. |
| `api-doc` | API documentation hosting (module, not a runtime doc source for agents — superseded here by `docs/ai/`). |
| `data-api` | General data access/reporting service. |
| `geo-resolver-api` | Geocoding / address-to-location resolution. |
| `planner-api` | Trip planning (see also `analytics/planner-common`, `analytics/trip-assigner`). |
| `scheduler-api` | Cron/scheduled jobs (e.g. dock-api SLA auto-reject publisher). |
| `map-api` | Map/routing data. |
| `notifications-api` | Notification dispatch. |
| `optimization-api` | Route/load optimization (see also `analytics/optimization-engine`). |
| `simulation-api` | Simulation runs for planning/optimization. |
| `events-api` | Event ingestion/processing. |
| `web-notification-api` | Web push notifications. **Note:** present as a directory but not currently listed as a module in the root `pom.xml` — verify before assuming it builds as part of the aggregator. |

## `analytics/*`

| Module | Purpose |
|---|---|
| `planner-common` | Shared planning domain logic used by `planner-api` and related services |
| `optimization-engine` | Route/load optimization core, backs `optimization-api` |
| `trip-assigner` | Trip assignment logic |

## Known cross-service mechanism: audit outbox

Producer services (`carrier-api`, `enterprise-api`, `indents-api`,
`shipper-api`, `supplier-api` confirmed today) write an `AuditOutboxEvent`
row and a periodic dispatcher (`AbstractAuditOutboxDispatcher` subclass)
publishes it to Kafka; `audit-api` consumes and stores it. Full detail in
the `create-audit` skill. `dock-api` does **not** use this yet (local
`DockBookingTimeline` instead) — known gap, not a pattern to copy.
