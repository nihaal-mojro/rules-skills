---
name: create-a-skill
description: >-
  Use when creating, restructuring, or reviewing a Cursor skill for the
  Mojro codebase — naming, placement (root vs. service repo), skill vs.
  rule vs. plain doc, required frontmatter, and the SKILL.md skeleton.
  Trigger phrases: "new skill", "add a skill", "where should this skill
  live", "skill vs rule", "create a skill for X".
disable-model-invocation: false
owner: platform
last_verified: 2026-09-22
---

# Creating a Mojro skill

## Naming

- Platform / cross-service skill: `mojro-<topic>` (e.g. `mojro-masterdata`)
  or a bare descriptive name if it's clearly platform-scoped in context
  (e.g. `build-apis`, `create-audit`).
- Skill nested under a parent, reachable from multiple parents: plain
  `<topic>` (e.g. `async-methods`, `kafka-events`).

## Placement — current policy (interim, not permanent)

**As of now, all skills live at the workspace root**, in
`.cursor/skills/platform/<name>/`. There is no `services/` folder — create
service-specific skills or per-service architecture views only when a real
need arises, not in advance. Do **not** create `.cursor/skills/` inside any
individual service repo yet.

If service-specific skills accumulate at root and become hard to maintain
(3+ never referenced outside one service, or the standing description-index
cost shows up in a usage-CSV review), revisit and move them into that
service's repo.

## Skill vs. glob rule vs. plain doc

| If… | Use |
|---|---|
| A deterministic file-path pattern exists (e.g. `**/app/api/*API.java`) | A glob-scoped `.mdc` rule — zero idle cost, no agent judgment call |
| Content is substantial, but only ever relevant from one parent skill | A plain `.md` file under that parent's folder (no frontmatter, not independently invocable) |
| Content is substantial and reachable from multiple parents, or is a standalone topic | A full child `SKILL.md`, `disable-model-invocation: true` if it should only load via a parent's link, `false` if it should also self-trigger |
| It's a fact that's always true and rarely changes | A line in the root `00-core.mdc` (budget: ~400 tokens total, don't grow this file) |

## Required frontmatter

```yaml
---
name: <skill-name>
description: >-
  What it's for, stated so the agent can match a real task to it. Front-load
  WHEN to use it — trigger phrases — not just what it is. This description
  is the only thing visible to the agent every turn before it decides to
  invoke the skill; get it wrong and the skill never fires.
disable-model-invocation: false   # true = only loads via explicit link/mention
owner: <team/handle>
last_verified: <YYYY-MM-DD>
---
```

## SKILL.md skeleton

Every content skill in this repo follows this section order:

1. Architecture / layer table
2. Hard rules (numbered, "do / never" phrasing)
3. Checklist (markdown task list, copy-paste block)
4. Code pattern (fenced code block)
5. Anti-patterns (bullet list)
6. **Related** — links to child docs/skills and sibling skills worth knowing about

Cap `SKILL.md` at roughly 150 lines — `mojro-masterdata` (~127 lines) is the
reference example. Push exhaustive detail (code anchors, payload sketches,
full class lists) to a linked `reference.md` instead of growing the main
file. A purely process/meta skill (like this one) is a deliberate exception
to progressive disclosure — no `reference.md` needed if there's nothing to
defer.

## Cascading children

A parent skill can link to children two ways:

- **Plain `.md` child** — when the sub-topic only matters from this one
  parent. No frontmatter, never independently invoked, just a file the agent
  opens because the parent told it to.
- **Full child `SKILL.md`, `disable-model-invocation: true`** — when the
  sub-topic is substantial *and* genuinely reachable from more than one
  parent. One copy on disk, linked from every relevant parent's "Related"
  section — never duplicate the content.

Cap nesting at 2 hops (parent → child). If reaching the actual instructions
takes more hops than that, flatten something.

## After writing: trigger-eval

Test 3–4 realistic prompts in a fresh chat and confirm:
- It auto-invokes on prompts that should trigger it, without being named.
- It stays quiet on adjacent-but-different tasks (no false positives).

If it doesn't fire reliably, the `description` is almost always the fix —
sharpen the trigger phrases before restructuring the content.

## Related

- `.cursor/docs/platform-utilities.md` — check before a new skill needs to
  document a "which util class do I use" table of its own; link there
  instead of duplicating.
- `.cursor/rules/00-core.mdc` — the always-on core; add a skill's *name*
  here once it exists, never its content.
