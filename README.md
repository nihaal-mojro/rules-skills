# Mojro Cursor Rules & Skills

Shared Cursor rules, skills, and reference docs for Mojro backend developers
using **Cursor**. They give the Cursor agent Mojro-specific context (how we
build APIs, audit entities, use master data) without bloating every prompt.
Currently a **pilot**.

This repo is the root of the `mojro` workspace folder. It tracks only the
Cursor context files; the ~34 service repos (`api/*`, `api-common/*`,
`analytics/*`) are separate repos and are ignored here by design.

## How Cursor uses it

| What | When it loads |
|---|---|
| `.cursor/rules/00-core.mdc` | Always — a short set of platform invariants |
| `.cursor/skills/platform/*` | On demand — the agent picks a skill when your task matches its description, or you can name it in chat |
| `docs/ai/*` | Only when you `@`-reference the file (e.g. `@docs/ai/ARCHITECTURE.md`, handy in Plan Mode) |

Because only the core rule is always on, keep it tiny and put everything else
in skills or docs.

## What's in it

| Path | Purpose |
|---|---|
| `.cursor/rules/00-core.mdc` | The single always-on rule (~400 tokens) |
| `.cursor/skills/platform/` | Cross-cutting skills: `build-apis`, `create-audit`, `mojro-masterdata`, `create-a-skill` |
| `docs/ai/` | Reference docs: platform utilities, glossary, architecture map |
| `AGENTS.md` | Short router listing where things live (Cursor also reads it) |

## Setup (one time, in your workspace root)

`git clone` refuses to run in a non-empty folder, so from your existing
`mojro` workspace root (e.g. `C:\codebase\mojro`):

```
git init
git remote add origin https://github.com/nihaal-mojro/rules-skills.git
git fetch
git checkout -t origin/master
```

The deny-by-default `.gitignore` leaves your service repos untouched. Then
open Cursor **at the workspace root** so the rules and skills load.

To update later: `git pull` from the workspace root.

## Improving it

- **Add or change a skill:** ask Cursor to use the `create-a-skill` skill — it
  covers naming, placement, required frontmatter, and the size cap (~150 lines).
- **Keep it small.** Only one rule is always on; everything else loads on
  demand. Put large reference material in `docs/ai/`, not in a rule.
- **Don't duplicate** what the code already says. Document conventions,
  invariants, and gotchas, not method lists.
- Track a new top-level path by adding an explicit `!` line in `.gitignore`
  (everything not re-included is ignored on purpose).
- Update `last_verified` in the frontmatter of anything you re-check.