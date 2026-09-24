# DAO layer patterns

Linked from [SKILL.md](SKILL.md) — only relevant when you're at the DAO/entity
layer of an endpoint you're building. Not an independently-triggered skill.

## Entity base classes

- `Entity` (`api-common/mojro-base/src/main/java/com/mojro/base/common/entity/Entity.java`)
  — `@MappedSuperclass` every JPA entity extends. Gives you `createdTime`,
  `updatedTime`, `createdBy`, `updatedBy` for free, and is wired to
  `AuditEntityListener` (stamps those fields automatically — don't set them
  by hand).
- `VersionEntity extends Entity` (`.../common/entity/VersionEntity.java`) —
  adds an optimistic-lock `version` column, auto-incremented by the same
  listener. Use this instead of plain `Entity` whenever concurrent updates
  need conflict detection (e.g. approve/reject race conditions).

## Audit listener wiring

`AuditEntityListener` (`api-common/mojro-base/.../repository/audit/AuditEntityListener.java`)
is attached via `@EntityListeners` on `Entity`. It handles timestamps/version/
request-id stamping only — it does **not** write full audit-trail records
(that call is currently commented out). If the entity needs a real audit
trail (who changed what, historical diff), that's the `create-audit` skill's
outbox pattern, not this listener.

## Generated QueryDSL `Q*` classes

Services using QueryDSL for typed queries generate `Q<Entity>` classes at
build time (see `api/audit-api/src/main/generated/.../QBaseAudit.java`,
`QAudit.java`, `QMasterAudit.java` as the confirmed example). Prefer these
over string-based JPQL/native queries for anything beyond a trivial lookup —
they're compile-time-checked against the entity's actual fields.

## Pagination

Use `PaginatedResponseInfo<T>`
(`api-common/mojro-common/.../dto/indent/response/PaginatedResponseInfo.java`)
for any paged list endpoint — `page`, `size`, `totalItems`, `totalPages`,
`hasNext`, `items`. Note its package location is `dto/indent/response/`
despite being general-purpose; don't be misled into thinking it's
indents-api-specific.
