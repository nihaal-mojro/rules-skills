---
name: build-apis
description: >-
  Generic Mojro Vert.x + Spring conventions for a new HTTP endpoint,
  event-bus worker, or service method in any backend microservice.
  Use when adding a route, a Worker.Operations case, a service method, or
  when a service-specific skill doesn't exist yet for the service you're in.
  Trigger phrases: "new endpoint", "new API route", "new event-bus worker",
  "add a service method", "wire up a new operation".
disable-model-invocation: false
owner: platform
last_verified: 2026-09-22
---

# Building a Mojro API (generic pattern)

This is the platform-wide version of a pattern first written down for
dock-api (`api/dock-api/.cursor/skills/mojro-dock-api/` — check there first
if you're in dock-api; it's the concrete instance of everything below).

## Architecture (every endpoint, every service)

```
HTTP  <Service>API (Vert.x)  → event bus → <Service>Worker → I<Service>*Service → DAO / IMasterDataService
      ApplicationConstants        Operations enum              @Transactional provider
      BaseAPI.invokeAsyncOperation
```

| Layer | Typical class | Responsibility |
|-------|---------------|-----------------|
| Routes | `<Service>API` | Params, validation, `DeliveryOptions`, async invoke |
| Worker | `<Service>Worker` | Deserialize body, read headers, call service, JSON reply |
| Service | `I<Service>*Service` + `*ServiceImpl`/`provider/*Service` | Business logic, `@Transactional` |
| DTO | `mojro-common/.../dto/<service>/` | Request/response JSON |
| Entity/DAO | `domain/entity`, `repository` | Postgres persistence — see [dao-patterns.md](dao-patterns.md) |

## Hard rules

1. **No business logic in API or Worker.** Both layers extract/validate/dispatch only.
2. Context (`enterpriseId`, `role`, hierarchy, `userAuthId`) comes from the
   JWT via `ParentServiceContextLoader`/`LocalContext` — never a
   client-supplied query/body param. See `.cursor/docs/platform-utilities.md`.
3. Controller→worker call goes through `BaseAPI.invokeAsyncOperation` (+
   `DeliveryOptions` header propagation); the worker reads `msg.headers()`.
   Don't hand-roll an event-bus send that bypasses this.
   (Note: `invokeAsyncOperation` sets a no-op header with a comment claiming
   Vert.x "sometimes drops the final header" — that's dead/misleading code,
   not a convention; don't copy that part.)
4. Validate with `ValidationUtil`; funnel failures through
   `ExceptionUtil.handleContextfailures`/`handleMessagefailures` +
   `ErrorMapping.<SERVICE>_*` constants (add new ones for new validations,
   don't reuse another service's codes).
5. Event-bus addresses are registered once in `QueueNames` — never invent an
   ad-hoc address string.
6. Catalog/config data the endpoint reads or writes is master data → see
   `mojro-masterdata`, not a new Postgres table.
7. **Search `.cursor/docs/platform-utilities.md` before writing a new
   cross-cutting helper** — check whether `CommonUtil`, `DateUtil`,
   `JacksonUtil`, `EventUtil`, etc. already do what you need.

## New endpoint checklist

```
- [ ] URL constant in ApplicationConstants
- [ ] Route + handler in <Service>API.registerRoutes()
- [ ] <Service>Worker.Operations enum value + switch case
- [ ] Service interface method + @Transactional provider implementation
- [ ] DTO in mojro-common/.../dto/<service>/ if new shape
- [ ] ErrorMapping.<SERVICE>_* for new validation failures
- [ ] No business logic in API or Worker
- [ ] Checked platform-utilities.md before adding a new helper
```

## Handler skeleton

```java
private void myOperation(RoutingContext context) {
    try {
        String enterpriseRefId = context.request().getParam(Constants.ENTERPRISE_REF_ID);
        ValidationUtil.notEmptyValidation(ErrorMapping.ENTERPRISE_MANDATORY_ID, enterpriseRefId);

        DeliveryOptions eventOptions = new DeliveryOptions();
        eventOptions.addHeader(Constants.WORKER_MESSAGE_OPERATION_HEADER_KEY,
                <Service>Worker.Operations.MY_OP.name());
        eventOptions.addHeader(Constants.ENTERPRISE_REF_ID, enterpriseRefId);
        // ...additional required params → headers

        invokeAsyncOperation(eventOptions, context, QueueNames.EVENT_BUS_<SERVICE>);
    } catch (Exception e) {
        ExceptionUtil.handleContextfailures(e, context);
    }
}
```

## Worker skeleton

- `handleOperations`: set `LocalContext` from `REQUEST_ID`, `switch` on
  `Operations`, `LocalContext.clear()` in `finally`.
- POST/PUT: parse `(String) msg.body()`; blank body →
  `COMMON_REQUEST_PAYLOAD_MISSING`.
- Read scope/filter params from **message headers** (set by the API layer),
  not from re-parsing the request.
- Failures: `ExceptionUtil.handleMessagefailures(e, msg)`.
- Success: `msg.reply(objectMapper.writeValueAsString(dto))`.

## Anti-patterns

- Business logic in the API or Worker layer.
- Hand-rolled event-bus send that bypasses `BaseAPI.invokeAsyncOperation`.
- A new Postgres table for what is actually catalog/config data.
- Client-supplied `enterpriseRefId`/`role` trusted from a query/body param.
- Reusing another service's `ErrorMapping.*` prefix instead of adding your own.

## Related

- [dao-patterns.md](dao-patterns.md) — entity base classes, audit listener wiring, pagination
- `mojro-masterdata` — if the endpoint touches catalog/config data
- `create-audit` — if the endpoint creates/updates an entity that needs an audit trail
- `api/dock-api/.cursor/skills/mojro-dock-api/` (in the dock-api repo) — a fully worked, service-specific instance of this pattern
- `.cursor/docs/platform-utilities.md` — shared helpers to check before writing new ones
- Future children (not yet authored): `async-methods`, `kafka-events`, `eb-service-calls` — add as `.cursor/skills/platform/<name>/` with `disable-model-invocation: true` when written, and link them here
