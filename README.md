# odoo_skill

A Claude Code skill for writing clean, idiomatic Odoo modules: models, fields,
views, QWeb reports, OWL, security, tests and migrations — verified against the
Odoo source code, not just the documentation.

## Branch model

One branch per Odoo major version. `main` is the version-agnostic base.

| Branch | Content |
|---|---|
| `main` | Principles that hold in every Odoo version. No version-specific APIs |
| `18.0` | Full skill for Odoo 18: `SKILL.md` + `references/` |

A version branch always starts from `main` and only adds version-specific
content on top. This keeps the direction of flow one-way:

```
main ──●──────────●──────────  version-agnostic fixes land here
        \          \ merge
18.0     ●──●──●────●────────  Odoo 18 content
```

Rules:

- A change that holds for every version goes to `main`, then `main` is merged
  into each version branch. Never cherry-pick version content back into `main`.
- A change that depends on the Odoo version goes only to that version branch.
- Every claim in a version branch must be verified against that version's source
  (`odoo/fields.py`, `odoo/models.py`, `odoo/addons/base/...`). Cite it as
  `path:line`.

### Adding a new version

```bash
git switch main
git switch -c 19.0
# write SKILL.md + references/ for 19.0, verified against the 19.0 source
git push -u origin 19.0
```

Do not create a branch before there is verified content for it. An empty version
branch is a skill that claims knowledge it does not have.

## Install

Clone the branch that matches your Odoo version into your skills directory. The
folder name must match the skill `name` in `SKILL.md`:

```bash
# project scope
git clone -b 18.0 git@github.com:AgustinAsef/odoo_skill.git .claude/skills/odoo-18-dev
# or user scope
git clone -b 18.0 git@github.com:AgustinAsef/odoo_skill.git ~/.claude/skills/odoo-18-dev
```

Update with `git pull` inside that folder.

## Layout (version branch)

```
SKILL.md          entry point: hard rules, decision table, steps
references/       one file per topic, loaded on demand
  orm-fields.md  orm-models.md  security.md  data-manifest.md
  views-actions-menus.md  qweb-pdf-reports.md  frontend-owl.md
  testing.md  coding-guidelines.md  migrations-upgrades.md
  performance.md  i18n.md  repo-organization.md
```

`SKILL.md` stays short because it is loaded on every activation; detail lives in
`references/` and is read only when the task touches that layer.

## House conventions

The skill encodes OLPA GROUP conventions on top of Odoo's own guidelines:
manifest `author: 'OLPA GROUP'`, `website: 'https://olpagroup.com/'`,
`version: '<odoo>.X.Y.Z'`; English labels and translatable messages, Spanish only
in `i18n/es_AR.po`. Adjust `SKILL.md` if you use it elsewhere.

## Commits

Conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`). Scope version
branch changes with the version: `fix(18.0): ...`.
