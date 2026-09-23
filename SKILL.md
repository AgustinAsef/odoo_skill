---
name: odoo-18-dev
description: "Trigger: Odoo 18 module, addon, model, field, view, report, QWeb, OWL, wizard, security, migration. Write clean Odoo 18 code the way core and the workspace expect."
license: Apache-2.0
metadata:
  author: OLPA GROUP
  version: "1.0"
---

## Activation Contract

Load before writing or changing any Odoo 18 module code: Python models, XML views/data/reports, security, JS/OWL, tests or migrations. Pair with `odoo-slop-audit` (the don't-do list); this skill is the do-list.

## Hard Rules

- Verify every API against `Versiones/V18/odoo-18.0/` (and `enterprise-18.0/`) before using it. Docs drift; the source does not. Never port 17/19 idioms blindly.
- 18.0 idioms, non-negotiable: `<list>` not `<tree>`; `invisible="expr"` not `attrs`/`states` (rejected at validation); `@api.model_create_multi` on every `create` override; `_compute_display_name` not `name_get`; `_sql_constraints` + `@api.constrains` (no `models.Constraint`, that is 19); `aggregator=` not `group_operator=`; hooks receive `env`; `check_access()`/`has_access()`.
- Reuse core before writing: inherit (`_inherit`), extend with `xpath`, call `super()`. A new model is justified only when no core model owns the concept.
- Batch everything: no ORM call inside a loop over records; `create(vals_list)`, `write` on recordsets, `_read_group` for aggregates.
- `sudo()` only with a comment stating why, scoped to the smallest expression. Multi-company models set `_check_company_auto = True` + `check_company=True`.
- Workspace rules win over Odoo/ADHOC defaults: manifest `author='OLPA GROUP'`, `website='https://olpagroup.com/'`, `version='18.0.X.Y.Z'`; English labels and `_()` messages, Spanish only in `i18n/es_AR.po` (no accents); invoicing modules depend on `accountant`; never touch a manifest of a module you did not change; conventional commits, no AI files committed.
- Resolve where the module lives with root `CLAUDE.md` before creating files; ask before placing a new reusable module.

## Decision Gates

| Need | Use | Reference |
|---|---|---|
| Add data to an existing model | `_inherit` same `_name` | `orm-models.md` |
| New concept owned by us | new `models.Model` | `orm-models.md` |
| One-off user dialog | `TransientModel` in `wizard/` | `coding-guidelines.md` |
| Derived value | computed field; `store=True` only if searched/grouped | `orm-fields.md` |
| UI-only reaction while editing | computed with `readonly=False` before `@api.onchange` | `orm-fields.md` |
| Data invariant | `_sql_constraints` if SQL-expressible, else `@api.constrains` | `orm-fields.md` |
| PDF document | `ir.actions.report` + QWeb (`web.external_layout`) | `qweb-pdf-reports.md` |
| Excel output | OCA `report_xlsx` or controller + `xlsxwriter` (no core engine) | `qweb-pdf-reports.md` |
| Analytical report | pivot/graph view or `account.report` before any custom report | `views-actions-menus.md` |
| Custom widget/client behavior | OWL component + registry, `patch()` to extend | `frontend-owl.md` |
| Field renamed, selection changed, data moved | migration script + version bump | `migrations-upgrades.md` |

## Execution Steps

1. Locate the owning directory (root `CLAUDE.md`), read the module's manifest and the core model you extend.
2. Read the relevant `references/*.md` for the layer you touch; open the cited core source when unsure.
3. Lay out files per `coding-guidelines.md` (singular `wizard/`, `report/`; one file per model; `ir.model.access.csv` for every new model).
4. Write English labels, `_()` messages, then the `es_AR.po` entries.
5. Add tests (`TransactionCase`, `@tagged('post_install', '-at_install')`); watch a regression test fail before trusting it.
6. Install/upgrade in the lab base with `--test-tags` and run `odoo-slop-audit` on the diff.

## Output Contract

Return: files changed, core models/methods extended (with source `path:line`), version bump and whether a migration was needed, tests run and their result.

## References

- `references/orm-fields.md` — field types, params, compute/related/constraints
- `references/orm-models.md` — models, inheritance, decorators, CRUD, domains, env, SQL
- `references/security.md` — groups, ACL, record rules, multi-company, sudo
- `references/data-manifest.md` — manifest keys, data files, noupdate, crons, hooks
- `references/views-actions-menus.md` — view skeletons, xpath, widgets, actions, menus
- `references/qweb-pdf-reports.md` — how Odoo builds a report, custom report models
- `references/frontend-owl.md` — assets, OWL, patch, registries, services
- `references/testing.md` — test classes, tags, Form, running tests
- `references/coding-guidelines.md` — layout, file and XML id naming, method prefixes
- `references/migrations-upgrades.md` — versions, pre/post/end scripts
- `references/performance.md` — N+1, prefetch, `_read_group`, indexes
- `references/i18n.md` — `_()`, `_lt`, `.po`, es_AR fallback
- `references/repo-organization.md` — ADHOC model vs ours, branches, tooling
