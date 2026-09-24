---
name: mojro-masterdata
description: >-
  Create, update, validate, and consume Mojro enterprise master data (ES catalogs).
  Use when adding MasterDataType entries, MasterData POJOs, MasterDataProcessor
  validators, enterprisemasterdata APIs, or reading catalogs via IMasterDataService.
  Applies across the Mojro codebase; not dock-api specific.
disable-model-invocation: false
owner: platform
last_verified: 2026-09-22
---

# Mojro Master Data

Generic platform rules for enterprise/hierarchy **catalog configuration**.  
Transactional domain entities (bookings, trips, orders) are **not** master data.

## Purpose

| Master data IS | Master data is NOT |
|----------------|--------------------|
| Configurable catalogs / settings per enterprise (± hierarchy) | Occupancy, bookings, trips, invoices |
| Stored in **Elasticsearch** + cache | Postgres/Mongo domain tables |
| Versioned upsert via common-api | Microservice-local DDL for catalogs |
| Read via `IMasterDataService` | Direct ES/DAO access from feature services |

**Storage:** ES index `es.enterprise.masterdata.index`  
**Document id:** `{MASTER_DATA_TYPE}` or `{TYPE}_{enterpriseId}` or `{TYPE}_{enterpriseId}_{hierarchyId}`  
**History:** prior version → history index before overwrite  
**Envelope:** `MasterDataList` = `masterdataType` + `metadata` + `masterdata[]`

## Hard rules (do not violate)

1. **Never** create a Postgres/Mongo table for a new catalog type. Persist only through `MasterDataService` → ES.
2. **Never** invent a parallel CRUD API in a feature service for catalog save. Use **common-api** enterprisemasterdata.
3. **Always** register the type in `MasterDataType` enum (string key is the contract).
4. **Always** put the typed POJO in `api-common/mojro-common/.../masterdata/`, extending `MasterData`.
5. **Create = upsert.** There is no separate update API; POST replaces the document for that type+enterprise+hierarchy.
6. **Split validation:**
   - **Save-time** → generic checks + optional `MasterDataProcessor`
   - **Use-time** → consuming microservice (slot engine, pricing, etc.)
7. **Do not** put runtime domain rules (capacity, slot continuity, booking SLA) in the POJO or processor unless they gate *configuration correctness*.
8. Class name suffix `Master` is **optional**. Package + `extends MasterData` + `MasterDataType` are mandatory.
9. Default types are **hierarchical**. Use `MasterDataType(name, false)` only for enterprise-only types (e.g. `ENTERPRISE_PROFILE`).
10. Consumers load with `getEnterpriseMasterDataList` / `getMasterData` — respect hierarchy inheritance; do not assume exact-level-only unless using exact-match APIs.

## Create / update workflow

Copy and complete:

```
MasterData checklist:
- [ ] POJO extends MasterData (+ @JsonIgnoreProperties(ignoreUnknown = true))
- [ ] MasterDataType enum entry (hierarchical flag correct)
- [ ] @FieldName on UI-facing fields when peers use them
- [ ] Each masterdata[] item has unique non-blank `name` (platform enforces)
- [ ] MasterDataProcessor ONLY if save-time domain rules needed
- [ ] Processor registered via Spring (@Component extending MasterDataProcessor)
- [ ] No feature-service write path; reads via IMasterDataService
- [ ] Use-time validation lives in the consuming service
- [ ] No PG/Mongo schema for the catalog
```

### Artifacts and locations

| Artifact | Location |
|----------|----------|
| POJO | `api-common/mojro-common/.../masterdata/<Type>.java` |
| Type key | `MasterDataType` enum |
| Save-time validator | `api-common/mojro-base/.../processor/<Type>Processor.java` |
| HTTP create/delete/read | `common-api` `MasterDataAPI` → `MasterDataWorker` |
| Persist/cache | `MasterDataService.createMasterData` / `deleteEnterpriseMasterData` |
| Read (any service) | `IMasterDataService` |

### HTTP (v2)

- `POST /common/v2/enterprisemasterdata` — create/upsert (`masterdataType`, `enterpriseRefId`, optional hierarchy)
- `GET /common/v2/enterprisemasterdata` — read
- `DELETE /common/v2/enterprisemasterdata` — delete (archives to history)
- Body: `MasterDataList`; path/query type must match `masterdataType`

### Save-time validation (platform)

`MasterDataService.validateAndUpdateMasterData`:

1. Body `masterdataType` matches request type → else `COMMON_MASTERDATA_TYPE_MISMATCH`
2. Every item has unique non-blank `name` → else `COMMON_MASTERDATA_NAME_INVALID`
3. If `MasterDataProcessorFactory.getMasterDataProcessor(type) != null` → `processor.process(list)`
4. Special id assignment only for `VEHICLE_CATEGORY_TYPE` / `EXECUTION_INSTRUCTIONS` (do not copy unless same need)

Existing processors today: VehicleCategoryType, ExecutionCost, ExecutionInstructions, CronJobs, ActivityUpdateStatusReason. Most types have **none**.

### Update semantics

1. Load existing for same type+enterprise+hierarchy  
2. If exact match → write current doc to **history** index with versioned id  
3. Write new payload to live index  
4. Refresh cache (`CACHE_KEY_ENTERPRISE_MASTERDATA_QUALIFIER` + documentId)  
5. May propagate cache to sub-hierarchies  

Treat POST as **full replace** of `masterdata[]` for that document key (not patch-merge unless a processor implements otherwise).

### Read patterns

```java
// List (hierarchy-aware)
masterDataService.getEnterpriseMasterDataList(
    MasterDataType.YOUR_TYPE.getMasterDataType(), YourClass.class, enterpriseId, hierarchyId);

// Single by id
masterDataService.getMasterData(
    MasterDataType.YOUR_TYPE.getMasterDataType(), id, YourClass.class, enterpriseId, hierarchyId);
```

Fail closed in the consumer if required catalog missing/inactive/misconfigured.

## Anti-patterns

- Postgres entity "because we need to query it" for static config → use ES master data  
- Dock/feature API that POSTs bay/schedule JSON into local DB  
- Duplicating enterprisemasterdata routes in another microservice  
- Putting booking/slot business rules in `MasterDataProcessor`  
- Assuming all class names end with `Master`  
- Skipping `MasterDataType` and relying on raw strings only in one service  

## Related

- [reference.md](reference.md) — code anchors and payload sketch
- `.cursor/docs/platform-utilities.md` — general platform utilities (this skill covers master data specifically, not the wider utility surface)
- `build-apis` skill — for the API/worker/service layers a catalog-consuming endpoint sits in
