# Mojro Master Data — Reference

Companion to [SKILL.md](SKILL.md). Read only when implementing or reviewing a master-data change.

## Code anchors

| Concern | Microservice / module | Type | Notes |
|---------|----------------------|------|--------|
| HTTP routes | `common-api` | `MasterDataAPI` | v2 enterprisemasterdata / masterdata |
| Worker ops | `common-api` | `MasterDataWorker.Operations` | GET, CREATE, DELETE, IMPORT, EXPORT, … |
| Service contract | `mojro-base` | `IMasterDataService` | create, delete, get list/by id, settings |
| Upsert + validate | `mojro-base` | `MasterDataService.createMasterData` | ES save + cache |
| Generic validate | `mojro-base` | `MasterDataService.validateAndUpdateMasterData` | type + unique name + processor |
| Processor SPI | `mojro-base` | `MasterDataProcessor` | `getMasterDataType()` + `process(MasterDataList)` |
| Processor registry | `mojro-base` | `MasterDataProcessorFactory` | Spring-collected map by type string |
| ES DAO | `mojro-base` | `MasterDataDao.saveMasterData` | index + documentId |
| Envelope | `mojro-common` | `MasterDataList` | masterdataType, metadata, masterdata[] |
| Base fields | `mojro-common` | `MasterData` | id, name, code, description, … |
| Type registry | `mojro-common` | `MasterDataType` | enum; `isHierarchical` |
| UI labels | `mojro-common` | `@FieldName` | on POJO fields |

Config properties (typical): `es.enterprise.masterdata.index`, `.history.index`, `.search.index`.

## Payload sketch (upsert)

```json
{
  "masterdataType": "BAY_MASTER",
  "metadata": {
    "hierarchyRefId": "<dc-hierarchy-uuid>"
  },
  "masterdata": [
    {
      "id": 1,
      "name": "BAY-01",
      "code": "B01",
      "zone": "AMBIENT",
      "acceptedVehicleTypeIds": [4],
      "operationType": 1,
      "reservedCustomerId": null,
      "reservedCustomerName": null,
      "isActive": true,
      "sequence": 1
    }
  ]
}
```

Rules:

- Every item **must** have unique `name` (case-insensitive uniqueness via lowercasing in validator).
- `masterdataType` in body **must** equal request `masterdataType` param.
- Hierarchical types: scope with enterprise + hierarchy (ref/code resolved in service).
- Non-hierarchical (`isHierarchical=false`): hierarchy stripped to null on save.

## Validation layers

```
POST enterprisemasterdata
  → MasterDataAPI (param presence, enterprise when not default)
  → MasterDataWorker CREATE
  → MasterDataService.createMasterData
       → validateAndUpdateMasterData (type, names, processor)
       → optional setMasterDataIds (VC / execution instructions only)
       → handleBaselineMetricsMasterData (BASELINE_METRICS only)
       → MasterDataDao.saveMasterData (ES)
       → cache update
```

**Use-time** (example pattern — any consumer):

```
FeatureService
  → IMasterDataService.getEnterpriseMasterDataList(...)
  → if empty/inactive → MojroValidationException (feature ErrorMapping)
  → apply domain rules (windows, eligibility, …)
```

## Naming guidance

| Prefer | When |
|--------|------|
| `*Master` | Discrete catalog rows (bays, schedules) |
| `*Configuration` / `*Config` | Nested settings blobs |
| `*Type` / `*Info` | Enumerations / settings peers already named that way |

Mandatory: `extends MasterData`, package `com.mojro.common.masterdata`, `MasterDataType` entry.

## Dock note (out of scope for this skill)

`BayMaster` / `ScheduleMaster` are ES catalogs. `DockBooking` is Postgres occupancy — transactional, not master data. Dock booking/slot rules belong in dock-api's own rules/docs, not here.
