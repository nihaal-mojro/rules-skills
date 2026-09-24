---
name: create-audit
description: >-
  Add audit logging for an entity in any Mojro backend service, using the
  platform's existing transactional-outbox audit pattern (not a new
  mechanism). Use when asked for "audit logging", "audit trail", "outbox
  event", "new auditable entity", or when adding a change-history
  requirement to an entity.
disable-model-invocation: false
owner: platform
last_verified: 2026-09-22
---

# Adding audit logging (the outbox pattern)

**Don't invent a new mechanism.** A working transactional-outbox pattern
already exists and is used by 5 services. Extend it; don't design around it.

## Architecture

```
<Service> (writer)                          audit-api (reader, read-only)
  entity change
    → AuditOutboxEvent row (own table)
    → AbstractAuditOutboxDispatcher (periodic verticle)
         → Kafka (topic defined by dispatcher subclass)
                                                → AuditDiffKafkaConsumer / AuditSnapshotKafkaConsumer
                                                     → AuditHandlerRegistry
                                                          → I<Entity>AuditHandler
                                                               → Postgres audit_event_log / master_audit_event_log
```

`audit-api` exposes **read-only** REST — `GET /audit/v2/audits/summary`,
`GET /audit/v2/audits` (`AuditAPI` → `AuditWorker` → `AuditService`,
`api/audit-api/src/main/java/com/mojro/audit/app/`). There is no write
endpoint. Writes only ever arrive via Kafka.

## Writer side — what you add in your service

1. **Outbox row**: create `<Service>AuditOutboxEvent` extending
   `AuditOutboxEvent` (`api-common/mojro-base/src/main/java/com/mojro/base/common/entity/AuditOutboxEvent.java`,
   `@MappedSuperclass` — eventId, entityType/Id, aggregateType/Id, intent,
   changeId, hierarchy ids, actor, payload jsonb) in your own domain/entity
   package, with its own table.
2. **Diff payload**: use `AuditOutboxDiffBuilder`
   (`api-common/mojro-base/.../common/util/AuditOutboxDiffBuilder.java`)
   with `@AuditOutboxDiffField`/`@AuditOutboxImpField`
   (`api-common/mojro-common/src/main/java/com/mojro/common/{AuditOutboxDiffField,AuditOutboxImpField}.java`)
   annotations on the entity fields you want diffed, to build the JSON
   payload from old vs. new state.
3. **Dispatcher**: create `<Service>AuditOutboxDispatcher extends
   AbstractAuditOutboxDispatcher`
   (`api-common/mojro-base/.../timer/AbstractAuditOutboxDispatcher.java`) —
   a periodic Vert.x verticle that batches unpublished outbox rows and
   publishes to Kafka; you only implement `topic()`.
4. **Event factory** (optional but the established pattern): a
   `<Service>AuditOutboxEventFactory` to build event lists from your
   domain changes before they're persisted as outbox rows.

### Worked examples (all confirmed present, use as templates)

| Service | Outbox entity | Dispatcher | Factory / usage site |
|---|---|---|---|
| carrier-api | `CarrierAuditOutboxEvent` | `CarrierAuditOutboxDispatcher` | `CarrierAuditOutboxEventProcessor`, `CarrierRateCardAuditOutboxEventProcessor`, `CarrierPenaltyAuditOutboxEventProcessor` |
| enterprise-api | `EnterpriseAuditOutboxEvent` | `EnterpriseAuditOutboxDispatcher` | `EnterpriseAuditOutboxEventFactory`, used from `AddressService` |
| indents-api | `IndentAuditOutboxEvent` | `IndentAuditOutboxDispatcher` | `IndentTenderAuditOutboxEventProcessor` |
| shipper-api | `ShipperAuditOutboxEvent` | `ShipperAuditOutboxDispatcher` | `ShipperAuditOutboxEventFactory` |
| supplier-api | `SupplierAuditOutboxEvent` | `SupplierAuditOutboxDispatcher` | used from `ActivityService`, `TripActionService`, `TripEditService`, `TripStageService` |

All under `api/<service>/src/main/java/com/mojro/<pkg>/{domain/entity,timer,service/provider}/`.

## Reader side — registering a new entity type in audit-api

Only needed if you're introducing a genuinely new `entityType`/
`aggregateType`, not for every writer-side addition:

1. Pick an unused `entityType`/`aggregateType` int.
2. Add an `I<Entity>AuditHandler` implementation in
   `api/audit-api/src/main/java/com/mojro/audit/service/provider/`
   (confirmed existing handlers to pattern-match:
   `TripAuditHandler`, `TripOrderAuditHandler`, `TripAggregateAuditHandler`,
   `CarrierProfileMasterAuditHandler`, `RateCardMasterAuditHandler`,
   `PenaltyRuleSetMasterAuditHandler`, `TenderMasterAuditHandler`,
   `VehicleMasterAuditHandler`, `AddressMasterAuditHandler`,
   `SAPOrderAuditHandler`, `ActivityAuditHandler`, plus `snapshot/`
   variants for full-state snapshots).
3. Register it in `AuditHandlerRegistry`
   (`.../service/provider/AuditHandlerRegistry.java`).
4. Add the payload class to `AuditPayloadRegistry`
   (`.../service/provider/AuditPayloadRegistry.java`).

## Known stragglers — do not copy these

- **`dock-api`'s `DockBookingTimeline`**
  (`api/dock-api/src/main/java/com/mojro/dock/domain/entity/DockBookingTimeline.java`,
  with its own local `DockBookingTimelineDao`/`DockBookingTimelineService`)
  is a one-off local JPA entity/table, **not** wired to the outbox pattern.
  It's tech debt, not a template — if you're extending dock-api's audit
  trail, migrate toward the outbox pattern rather than adding more to this.
- **Dead mechanism**: `AuditLog`/`AuditLogService`/`AuditLogDao`
  (`api-common/mojro-base/src/main/java/com/mojro/base/{common/entity/AuditLog,services/provider/AuditLogService,repository/provider/AuditLogDao}.java`)
  plus the disabled write call inside `AuditEntityListener` — effectively
  dead code. Do not use.
- **Legacy/parallel mechanism**: `AuditPublisher`/`AuditEventUtil.publishAuditData`
  (`api-common/mojro-base/src/main/java/com/mojro/base/{publish/AuditPublisher,util/AuditEventUtil}.java`,
  topic `audit-data-event`) is a separate path not obviously wired to
  `audit-api`'s consumers. Do not use.

**If you find an entity not using the outbox pattern, that's tech debt —
flag it, don't copy it.**

## Related

- `build-apis` — the writer-side entity/service you're adding audit
  logging to is built per that skill's layering.
- `.cursor/docs/ARCHITECTURE.md` — `audit-api` row, and the "known cross-service
  mechanism: audit outbox" section.
- `.cursor/skills/platform/build-apis/dao-patterns.md` — `Entity`/
  `VersionEntity` base classes and `AuditEntityListener`'s actual (limited)
  scope.
