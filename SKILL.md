---
name: odoo-dev
description: "Trigger: Odoo module, addon, model, field, view, report, wizard, security, migration. Version-agnostic base; check out the branch for your Odoo version."
license: Apache-2.0
metadata:
  author: OLPA GROUP
  version: "1.0"
---

## Activation Contract

This is the version-agnostic base on `main`. It holds only principles that are true in every Odoo version. For real work, use the branch that matches the target Odoo version (for example `18.0`), which adds the version-specific rules and `references/`.

## Hard Rules

- Identify the target Odoo version first. APIs, view syntax and hook signatures change between majors; never assume one version's idiom holds in another.
- Verify every API against that version's source (`odoo/fields.py`, `odoo/models.py`, `odoo/addons/base/`). The docs drift; the source does not.
- Reuse core before writing: inherit (`_inherit`), extend views with `xpath`, call `super()`. A new model only when no core model owns the concept.
- Batch everything: no ORM call inside a loop over records; aggregate in the database, not in Python.
- Every new model gets access rights; `sudo()` only with a comment stating why, scoped to the smallest expression.
- Labels and user messages in English, messages wrapped in `_()`; other languages live in `i18n/*.po`.
- A renamed field, changed selection or moved data needs a migration script and a version bump.
- House conventions: manifest `author='OLPA GROUP'`, `website='https://olpagroup.com/'`, `version='<odoo>.X.Y.Z'`; never touch the manifest of a module you did not change.

## Decision Gates

| Situation | Action |
|---|---|
| Version branch for the target Odoo exists | Use it; this base is only the floor |
| No branch for the target version | Apply these rules, verify every API in that version's source, and flag the missing branch |
| Rule here conflicts with a version branch | The version branch wins |

## Execution Steps

1. Determine the Odoo version from the module manifest or the server source.
2. Load the matching version branch of this skill if available.
3. Read the core model or view you extend before writing.
4. Write the module following Odoo's layout: `models/`, `views/`, `security/`, `data/`, `wizard/`, `report/`, `static/`, `i18n/`, `tests/`, `migrations/`.
5. Add tests and watch a regression test fail before trusting it.

## Output Contract

Return: target Odoo version, files changed, core models extended (with source `path:line`), version bump and migration decision, tests run and their result.

## References

- `README.md` — branch model and how to add a version.
