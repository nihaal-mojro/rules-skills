---
owner: platform
last_verified: 2026-09-22
---

# Platform Utilities — Tier 1 (curated, not exhaustive)

Hand-curated list of shared classes in `api-common/mojro-common` and
`api-common/mojro-base` worth checking **before** writing a new
cross-cutting helper. This is deliberately small: it exists to prevent
re-implementing something that already exists platform-wide, not to
catalog every method in the shared libraries. It will not be kept in sync
automatically — add an entry only when a class earns genuine cross-service,
Tier-1 status. Service-local reuse is a search-before-you-build habit
(see the `build-apis` skill), not something to document here.

## Validation / errors

| Class | Path | Use for |
|---|---|---|
| `ValidationUtil` | `mojro-base/src/main/java/com/mojro/base/util/ValidationUtil.java` | Request-payload validation (`notEmptyValidation`, UUID/date-range checks) before hitting the DB/event-bus |
| `ExceptionUtil` | `mojro-common/src/main/java/com/mojro/common/util/ExceptionUtil.java` | The single place to funnel exceptions into HTTP/event-bus replies (`handleContextfailures`, `handleMessagefailures`) |
| `MojroException` / `MojroRuntimeException` / `MojroValidationException` / `MojroServiceException` / `MojroCriticalException` | `mojro-common/src/main/java/com/mojro/common/exception/` | The exception hierarchy every service throws/catches — never a raw `RuntimeException` |
| `Error` / `ErrorType` / `ErrorCodes` / `ErrorMapping` | `mojro-common/src/main/java/com/mojro/common/error/` | Wire-format error objects and the error-code registry; each service adds its own `ErrorMapping.<SERVICE>_*` constants |

## Master data

Fully covered by the `mojro-masterdata` skill — no separate entry needed
here.

## DTOs / entities

| Class | Path | Use for |
|---|---|---|
| `PaginatedResponseInfo<T>` | `mojro-common/src/main/java/com/mojro/common/dto/indent/response/PaginatedResponseInfo.java` | Generic paginated-list wrapper. **Note the odd package** (`dto/indent/response/`) — it's general-purpose despite the name, used well beyond indents. |
| `Entity` | `mojro-base/src/main/java/com/mojro/base/common/entity/Entity.java` | `@MappedSuperclass` base for every JPA entity — `createdTime`/`updatedTime`/`createdBy`/`updatedBy`, wired to `AuditEntityListener`. |
| `VersionEntity` | `mojro-base/src/main/java/com/mojro/base/common/entity/VersionEntity.java` | Extends `Entity`, adds an optimistic-lock `version` column — use when concurrent updates need conflict detection. |
| `JacksonUtil` | `mojro-common/src/main/java/com/mojro/common/util/JacksonUtil.java` | Quick (de)serialization (`fromString`, `toString`, `clone`) without wiring an `ObjectMapper` bean. |
| `CommonUtil` | `mojro-common/src/main/java/com/mojro/common/util/CommonUtil.java` | Grab-bag: `getObjectMapper()`, `generateUUID()`, `generateReferenceNumber(...)`, base64 encode/decode for ref IDs, JWT generation, page-count math. |

## Context / auth

| Class | Path | Use for |
|---|---|---|
| `ParentServiceContextLoader` | `mojro-base/src/main/java/com/mojro/base/services/config/ParentServiceContextLoader.java` | Base Spring config every service extends — decodes the JWT, sets `enterpriseId`/`role`/`userAuthId`, does URI-role authorization. Don't reinvent this. |
| `RequestContext` | `mojro-base/src/main/java/com/mojro/base/context/RequestContext.java` | POJO: `requestId`, `userName`, `enterpriseId`, `hierarchyId`, `userAuthId`, `locale`, `role`. |
| `LocalContext` | `mojro-base/src/main/java/com/mojro/base/context/LocalContext.java` | `ThreadLocal<RequestContext>` + MDC wiring — reach for this outside a `RoutingContext` (service/DAO layer). |
| `ContextUtil` | `mojro-base/src/main/java/com/mojro/base/common/util/ContextUtil.java` | Bridges request params → cached `RequestContext`, resolves a user's hierarchy IDs. |
| `ApplicationContextProvider` | `mojro-base/src/main/java/com/mojro/base/context/ApplicationContextProvider.java` | Static Spring bean accessor for non-Spring-managed code (e.g. JPA listeners). |

## Event-bus

| Class | Path | Use for |
|---|---|---|
| `QueueNames` | `mojro-common/src/main/java/com/mojro/common/constant/QueueNames.java` | The canonical registry of event-bus addresses (~100 constants, one per service). Never invent an ad-hoc address string. |
| `EventUtil` | `mojro-base/src/main/java/com/mojro/base/common/util/EventUtil.java` | `createDeliveryOptions`, `processEvent` (sync send+await), `sendEvent` (fire-and-forget). |
| `BaseAPI.invokeAsyncOperation` (+ siblings) | `mojro-base/src/main/java/com/mojro/base/app/api/BaseAPI.java` | The standard controller→worker call, with header propagation (`REQUEST_ID`, `USER_AUTH_ID`). **Caveat:** this method also sets a no-op header with a comment claiming Vert.x "sometimes drops the final header" — that isn't real Vert.x behavior; treat it as dead/misleading code, not a pattern to copy, and flag to whoever owns `BaseAPI` rather than propagate it. |

## Audit — base entity hooks only

The real cross-service audit mechanism (the outbox pattern) is documented in
the `create-audit` skill, not here. These two classes are the low-level JPA
hooks it and `Entity` rely on:

| Class | Path | Note |
|---|---|---|
| `IAuditable` | `mojro-base/src/main/java/com/mojro/base/repository/audit/IAuditable.java` | Marker interface (`setRequestId`) for entities carrying a request-id for correlation. |
| `AuditEntityListener` | `mojro-base/src/main/java/com/mojro/base/repository/audit/AuditEntityListener.java` | JPA lifecycle listener on `Entity` — stamps timestamps/version/requestId. The actual audit-log-write call inside it is currently **commented out/disabled** — don't assume it writes audit records. |

## Date / time

| Class | Path | Use for |
|---|---|---|
| `DateUtil` | `mojro-common/src/main/java/com/mojro/common/util/DateUtil.java` | The one date/time utility every service uses (~150 static methods: parse/format, arithmetic, range checks, month boundaries, IST/default timezone constants). |
| `TimezoneUtil` | `mojro-common/src/main/java/com/mojro/common/util/TimezoneUtil.java` | Resolves a timezone ID from country/abbreviation (address/geo flows). |

## Misc (lower priority)

| Class | Path | Note |
|---|---|---|
| `VertxUtil` | `mojro-common/src/main/java/com/mojro/common/util/VertxUtil.java` | Creates the clustered `Vertx`/Hazelcast instance — used by `ParentServiceContextLoader`'s `vertx()` bean. |
| `UriRegistry` | `mojro-base/src/main/java/com/mojro/base/util/UriRegistry.java` | Role-based URI authorization registry consumed by `ParentServiceContextLoader.authorize(...)`. |
| `RedisConnectionPoolUtil` | `mojro-common/src/main/java/com/mojro/common/util/RedisConnectionPoolUtil.java` | Shared Redis pool config/connection factory. |
