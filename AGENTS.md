  # Mojro Backend — Agent Entry Point

Map for Cursor agents working in this workspace. This file is a **router**,
not content — it points to where the real guidance lives; it doesn't
duplicate it.

## What this is

Java 8, Vert.x (HTTP routes + event-bus workers) + Spring (context/DI),
Postgres (transactional data) + Elasticsearch (enterprise master data).
`C:\codebase\mojro` is a workspace containing ~34 independently git-tracked
service repos under `api/*`, `api-common/*`, `analytics/*`. This root itself
is a separate, small git repo that tracks only the AI-context system below —
none of the service code.

## Where things live

| Need | Look here |
|---|---|
| Platform-wide invariants, always relevant | `.cursor/rules/00-core.mdc` |
| Creating or restructuring a skill | skill `create-a-skill` |
| Building a new API endpoint / event-bus worker | skill `build-apis` |
| Adding audit logging for an entity | skill `create-audit` |
| Elasticsearch enterprise master-data (catalogs/config) | skill `mojro-masterdata` |
| Shared utility classes worth reusing before writing a new helper | `docs/ai/platform-utilities.md` |
| Service map, module layout, request-flow diagram | `docs/ai/ARCHITECTURE.md` (read on demand, not auto-loaded — `@`-reference it) |
| Domain vocabulary (enterprise, hierarchy, master data, outbox, …) | `docs/ai/glossary.md` |

## How this is organized (so you know where to add things)

- **Rules and service-specific docs stay inside each service's own repo**
  (e.g. `api/dock-api/.cursor/rules/`, `api/dock-api/docs/`) — they travel
  with the code they describe.
- **Skills currently live only here, at the workspace root**, under
  `.cursor/skills/platform/`. Service-specific skills and per-service
  architecture views are built when required, not pre-created — see
  `create-a-skill` for placement policy.
- Large reference material is a plain doc under `docs/ai/`, not a rule —
  pull it in deliberately (e.g. in Plan Mode) rather than expecting it to
  auto-load.
