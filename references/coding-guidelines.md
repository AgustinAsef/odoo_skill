# Coding guidelines — Odoo 18 module structure and naming

Merges Odoo's official coding guidelines with this workspace's `CLAUDE.md`
conventions. Where they conflict, the workspace wins — noted inline.

## Module directory tree

```
<module>/
├── __init__.py              imports models/, wizard/, report/, controllers/
├── __manifest__.py
├── controllers/              HTTP routes, web endpoints
├── models/                   business logic, one file per main model
├── views/                    backend views + portal/website templates
├── security/                 ir.model.access.csv, groups, record rules
├── data/                     data and demo XML, split by model
├── report/                   printable reports (QWeb + report.py actions)
├── wizard/                   transient models (multi-step user actions)
├── static/
│   ├── description/          icon.png, index.html
│   └── src/                  css/, js/, img/, xml/ (owl templates), lib/
├── i18n/                     es_AR.po (this workspace: no other language files)
├── tests/                    TransactionCase / HttpCase, test_*.py
└── migrations/               <version>/pre-*.py, post-*.py, end-*.py
```

Only `models/`, `views/`, `security/`, `data/` are near-universal; `wizard/`,
`report/`, `static/`, `i18n/`, `tests/`, `migrations/` are added when the module
actually needs them — do not scaffold empty directories.

## File naming

| Directory | Pattern | Example |
|---|---|---|
| `models/` | one file per **main** model; each inherited/extended Odoo model gets its own file | `models/plant_order.py`, `models/res_partner.py` |
| `views/` | `<model>_views.xml` (backend), `<model>_templates.xml` (portal/website) | `sale_order_views.xml` |
| `views/` (menus) | optional `<module>_menus.xml` when menus are not tied 1:1 to an action | `stock_lot_menus.xml` |
| `security/` | `ir.model.access.csv` (always this name) + `<module>_groups.xml` + `<model>_security.xml` per record-rule set | `sale_order_security.xml` |
| `data/` | `<model>_data.xml` and `<model>_demo.xml`, split by purpose | `res_currency_data.xml` |
| `report/` | `<model>_report.py` (actions) + `<model>_report_templates.xml` (QWeb) | |
| `wizard/` | `<model>_wizard.py` matching the transient model | |
| `tests/` | `test_<feature>.py`, imported from `tests/__init__.py` | `test_invoice_flow.py` |
| `migrations/` | `<version>/{pre,post,end}-<description>.py` | `migrations/18.0.2.0.0/post-backfill_lang.py` |

## XML id naming

| Kind | Pattern | Example |
|---|---|---|
| Security group | `group_<description>` | `group_stock_manager` |
| Action | `action_<model_name>` or `action_<description>` | `action_sale_order` |
| View | `<model_name>_<view_type>` | `plant_order_form`, `plant_order_list` |
| Menu | `menu_<description>` | `menu_plant_nursery_root` |

Every id is referenced as `<module>.<xml_id>` from outside its own module; never
hardcode the numeric `id` — see `odoo-slop-audit` B3.

## Python import order

1. Standard library (`import os`, `from datetime import date`)
2. Third-party libraries (`import requests`)
3. Odoo (`from odoo import api, fields, models`, `from odoo.exceptions import UserError`)
4. Local module imports (`from . import models`, relative imports within the module)

Alphabetize within each group. Do not mix Odoo core imports with
`odoo.addons.<other_module>` imports — the latter is a cross-module dependency
and belongs after `from odoo import ...`, on its own line, only when
`depends` already lists that module.

## Method naming prefixes

| Prefix | Meaning |
|---|---|
| `_compute_<field>` | computed-field method, matches `@api.depends` |
| `_search_<field>` | `search=` callable for a non-stored computed field |
| `_onchange_<field>` | onchange handler (UI-only, avoid for new code — prefer compute) |
| `action_<verb>` | button-triggered method, callable from a view, returns `None` or an action dict |
| `_default_<field>` | default-value callable |
| `_get_<noun>` | private getter, no side effects |
| `_check_<constraint>` | `@api.constrains` method |
| `_prepare_<noun>_vals` | builds a `vals` dict for a `create`/`write`, does not call the ORM itself |

A leading underscore means "not part of the public/RPC-callable API" — every
private helper, compute, and constraint method starts with `_`.

## Variable naming

| Field type | Suffix | Example |
|---|---|---|
| `Many2one` | `_id` | `partner_id`, `product_id` |
| `One2many` / `Many2many` | `_ids` | `invoice_line_ids`, `tag_ids` |
| plain value | no suffix, name describes the value | `amount_total`, `state` |

Name a variable after what it *is*, not its Python type
(`vals_dict` → `vals`; see `odoo-slop-audit` A2).

## Manifest conventions — merged

Odoo core defaults (`odoo/modules/module.py:42-79`) vs. this workspace's rule
(root `CLAUDE.md`, "Manifest conventions"):

| Key | Odoo default if omitted | Workspace rule (new modules only) |
|---|---|---|
| `name` | **mandatory**, no default | descriptive, English |
| `license` | mandatory; silently defaults to `LGPL-3` with a warning if missing (`module.py:322-324`) | always set explicitly |
| `version` | `'1.0'` | `'18.0.X.Y.Z'` — five segments, `18.0` prefix |
| `author` | `'Odoo S.A.'` | `'OLPA GROUP'` — exact casing |
| `website` | `''` | `'https://olpagroup.com/'` — trailing slash |
| `installable` | `True` | leave `True` unless deliberately disabling |
| `depends` | `[]` | must include `accountant` (not bare `account`) for any module touching invoices — see workspace `CLAUDE.md` "Invoicing means accountant" |

**Hard rule (workspace, overrides everything above): never edit the manifest
of a module you did not otherwise change.** odoo.sh upgrades a module because
its *files* changed, not because the version bumped — see `odoo-slop-audit` C4.
The five-segment version and author/website conventions apply only to new
modules or modules already being touched for another reason, never as a sweep.

## Commit messages

Two conflicting conventions exist for this workspace — **the workspace one
applies**, not Odoo's own:

- **Odoo upstream convention** (`git_guidelines.html`, only relevant if
  contributing to Odoo/OCA core, not to this workspace's own modules):
  `[TAG] module: short description`, tags `[FIX] [ADD] [IMP] [REF] [REM] [REV]
  [MOV] [REL] [MERGE] [I18N] [PERF] [CLN] [LINT]`, body explains *why*, one
  module per commit, references (`Fixes #123`, `opw-123`) in a trailing block.
- **This workspace**: root `CLAUDE.md` and `.claude/CLAUDE.md` require
  **conventional commits** (`feat:`, `fix:`, `refactor:`, etc.) and forbid any
  AI attribution line. Never commit or push unless explicitly asked.

Use the Odoo `[TAG]` style only when the change targets a third-party
(OCA/Odoo core) repository under `Terceros/`, since that code is read-only
here and any patch would be submitted upstream under its own convention.
Everything under `Compartido/` or a client repo uses conventional commits.

## Symbols and style

- PEP 8 is the baseline; f-strings for string building, but never for SQL or
  for a string passed to `_()` (breaks translation extraction — see `i18n.md`).
- Double quotes for strings, consistently within a file.
- Docstrings on public/RPC-callable methods; private `_`-prefixed helpers may
  skip them if the name is self-explanatory.
- `@api.model_create_multi` on every `create` override — no `vals: dict`
  parameter, always `vals_list: list[dict]` (see `odoo-slop-audit` B2,
  verified `odoo/api.py:501`).
