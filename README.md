# Mojro AI Workspace — rules, skills & agent context

Shared Cursor rules, skills, and reference docs that give AI coding agents
richer, token-efficient context about the Mojro backend. Currently a **pilot**.

This repo is the root of the `mojro` workspace folder. It tracks only the
AI-context files; the ~34 service repos (`api/*`, `api-common/*`,
`analytics/*`) are separate repos and are ignored here by design.

## What's in it

| Path | Purpose |
|---|---|
| `AGENTS.md` | Vendor-neutral entry point / router for any agent |
| `.cursor/rules/00-core.mdc` | The single always-on rule (kept ~400 tokens) |
| `.cursor/skills/platform/` | Cross-cutting skills (`build-apis`, `create-audit`, `mojro-masterdata`, `create-a-skill`) |
| `docs/ai/` | On-demand reference docs (platform utilities, glossary, architecture map) |

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

- **Add or change a skill:** read the `create-a-skill` skill first — it covers
  naming, placement, required frontmatter, and the size cap (~150 lines).
- **Keep it small.** Only one rule is always-on; everything else loads on
  demand. Put large reference material in `docs/ai/`, not in a rule.
- **Don't duplicate** what the code already says. Document conventions,
  invariants, and gotchas, not method lists.
- Track a new top-level path by adding an explicit `!` line in `.gitignore`
  (everything not re-included is ignored on purpose).
- Update `last_verified` in the frontmatter of anything you re-check.
- Pin a specific model in Cursor rather than `Auto` — Auto can switch models
  mid-conversation and break prompt caching, raising token cost.
