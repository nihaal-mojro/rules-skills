# Mojro Cursor Rules & Skills

Shared Cursor rules, skills, and reference docs for Mojro backend developers
using **Cursor**. They give the Cursor agent Mojro-specific context (how we
build APIs, audit entities, use master data) without bloating every prompt.
Currently a **pilot**.

This repo **is** the `.cursor` folder of your `mojro` workspace. It is
separate from the ~34 service repos (`api/*`, `api-common/*`, `analytics/*`).

## How Cursor uses it

| What | When it loads |
|---|---|
| `rules/00-core.mdc` | Always — a short set of platform invariants |
| `skills/platform/*` | On demand — the agent picks a skill when your task matches its description, or you can name it in chat |
| `docs/*` | Only when you `@`-reference the file (e.g. `@.cursor/docs/ARCHITECTURE.md`, handy in Plan Mode) |

Because only the core rule is always on, keep it tiny and put everything else
in skills or docs.

## Setup (one time)

From your `mojro` workspace root (e.g. `C:\codebase\mojro`), with no existing
`.cursor` folder there (move it aside first if you have one):

```
git clone https://github.com/nihaal-mojro/rules-skills.git .cursor
```

Then open Cursor **at the workspace root** so the rules and skills load. To
update later: `cd .cursor && git pull`.

## Contributing

Work like any other Mojro repo, from inside the `.cursor` folder:

```
cd .cursor
git checkout -b my-improvement
# edit rules, skills or docs
git commit -am "Describe the change"
git push -u origin my-improvement
# open a pull request on GitHub
```

Cursor's Source Control panel may not list this nested repo, so use the
terminal (or open `.cursor` as its own folder). Changes to `master` should go
through a reviewed PR — these files shape every developer's agent.

## Improving it

- **Add or change a skill:** ask Cursor to use the `create-a-skill` skill — it
  covers naming, placement, required frontmatter, and the size cap (~150 lines).
- **Keep it small.** Only one rule is always on; everything else loads on
  demand. Put large reference material in `docs/`, not in a rule.
- **Don't duplicate** what the code already says. Document conventions,
  invariants, and gotchas, not method lists.
- Update `last_verified` in the frontmatter of anything you re-check.
