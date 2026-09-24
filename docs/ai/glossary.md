---
owner: platform
last_verified: 2026-09-22
---

# Mojro Domain Glossary

Definitions only — how-to lives in skills, not here.

- **Enterprise** — a Mojro customer/tenant. `enterpriseId` (numeric) /
  `enterpriseRefId` (UUID) both identify one; UUID form is what clients send,
  numeric form is resolved server-side and used internally.
- **Hierarchy** — a tenant's org tree (region → zone → ... → DC, up to L5 in
  most services). `hierarchyId` scopes a request to a node in that tree.
  Some APIs expand a mid-tree `hierarchyId` to all active leaf nodes beneath
  it (e.g. dock-api's booking list).
- **Master data** — configurable catalog/settings data (bays, schedules,
  vehicle categories, rate cards) stored in Elasticsearch via
  `IMasterDataService`, versioned per enterprise (± hierarchy). Not the same
  as transactional/domain data (bookings, trips, orders), which lives in
  Postgres. See the `mojro-masterdata` skill.
- **Event-bus "operation"** — a named action a service's `Worker` class
  dispatches on (its own enum, e.g. `DockWorker.Operations`), delivered over
  the Vert.x event bus to an address registered in `QueueNames`.
- **Outbox (audit)** — the pattern where a service writes an
  `AuditOutboxEvent` row locally, then a periodic dispatcher publishes it to
  Kafka for `audit-api` to consume — decouples the write from the audit
  pipeline. See the `create-audit` skill.
- **Aggregate vs. entity (audit terms)** — `aggregateType`/`aggregateId` on
  an audit event identify the parent/root object (e.g. a trip); `entityType`/
  `entityId` identify the specific thing that changed (which may be the
  aggregate itself or a child of it).
- **`RequestContext`** — the per-request POJO (`requestId`, `enterpriseId`,
  `hierarchyId`, `userAuthId`, `role`, `locale`) populated from the JWT and
  held in `LocalContext` (a `ThreadLocal`) for the duration of a request.
