# ORM Models — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/odoo/models.py`, `odoo/api.py`.

## Model base classes

| Class | Table | Persistence | Use for |
|---|---|---|---|
| `models.Model` | yes (`_auto=True`) | permanent | normal business objects |
| `models.TransientModel` | yes | rows auto-vacuumed after a few hours | wizards (`res.config.settings`, import/export dialogs) |
| `models.AbstractModel` | no (`_auto=False`) | none — provides methods/fields to inheritors only | mixins (`mail.thread`, `mail.activity.mixin`) |

Class attrs: `_name`, `_description`, `_inherit`, `_order`, `_rec_name`,
`_table`, `_log_access` (adds `create_date`/`write_date`/`create_uid`/
`write_uid`, default `True`).

## The three inheritance modes

| Mode | Declaration | Table | Effect |
|---|---|---|---|
| Classical (extension) | `_inherit = 'sale.order'` (no `_name`) | same table | adds/overrides fields and methods on the existing model, in place. This is what a first-party module normally does |
| Extension + rename | `_name = 'x'; _inherit = 'sale.order'` | new table `x` | copies all fields/behavior into a **new** model that also behaves like the parent (rare — used for e.g. `account.move` vs a specialized doc type) |
| Delegation | `_inherit = ['res.partner']` on a field: `partner_id = fields.Many2one('res.partner', delegate=True, required=True)` | own table + FK | composition — `self.partner_id.name` is reachable directly as `self.name` via automatic delegation, but the two are separate records |

Multiple classical inheritance: `_inherit = ['mail.thread', 'mail.activity.mixin']`
mixes in `AbstractModel` behavior without a table of its own.

## `api` decorators

| Decorator | Signature it expects | Meaning |
|---|---|---|
| `@api.model` | `def method(self):` — `self` may be empty | no need for a specific record; called on the model or an empty recordset |
| `@api.model_create_multi` | `def create(self, vals_list):` | **required** on any `create` override. Batch signature — `vals_list` is a list of dicts. `odoo/api.py:501`. Without it, RPC/batch callers must wrap as `[[], [vals]]` (`odoo-slop-audit` B2) |
| `@api.depends(*fields)` | on a compute method | declares read dependencies for cache invalidation, including dotted paths |
| `@api.depends_context(*keys)` | on a compute method | invalidates cache when a context key changes (e.g. `pricelist`, `company`) |
| `@api.onchange(*fields)` | UI-only | see `orm-fields.md` |
| `@api.constrains(*fields)` | raises `ValidationError` | see `orm-fields.md` |
| `@api.returns(model, downgrade=None, upgrade=None)` | on RPC-exposed methods | declares/converts the return type across `browse`/`id`/`ids` shapes for XML-RPC clients |
| `@api.autovacuum` | `@api.model` + cron-invoked | marks a method meant to be called by `_autovacuum` housekeeping only |

No decorator = a plain instance method, always called on a non-empty
recordset by convention (loop over `self` inside).

## CRUD batch idioms

```python
# create — always accepts (and, since model_create_multi, is called with) a list
records = self.env['res.partner'].create([
    {'name': 'A'}, {'name': 'B'},
])

# write — one call across a combined recordset, never one write() per record
(orders_a + orders_b).write({'state': 'done'})

# search + fetch fields in one round trip (18.0), avoids search() then read()
partners = self.env['res.partner'].search_fetch(
    [('customer_rank', '>', 0)], ['name', 'email'],
)  # models.py:1759

# count without loading records
n = self.env['res.partner'].search_count([('active', '=', True)])  # models.py:1718

# grouped aggregation — replaces manual dict-building loops
by_state = self.env['sale.order'].search([]).grouped('state')  # models.py:6548

# low-level group/aggregate query — the modern signature
result = self.env['sale.order']._read_group(
    domain=[('state', '=', 'sale')],
    groupby=['partner_id'],
    aggregates=['amount_total:sum'],
)  # models.py:1965
```

`read_group(domain, fields, groupby, ...)` (`models.py:2806`) is the legacy
public API (still present, dict-shaped rows). `_read_group(domain, groupby,
aggregates)` (`models.py:1965`) is the 18.0-preferred internal-style API used
throughout core code — returns tuples, no wrapping dict, and takes
`'field:aggregator'` strings for `aggregates`. Prefer `_read_group` in new
first-party code; it's what `sale`, `account`, etc. use internally.

Never call `search`/`browse`/`create`/`write`/`search_count` inside a `for`
loop — see `odoo-slop-audit` B1.

## Domains

```python
[('state', '=', 'draft'), ('partner_id', '!=', False)]        # implicit AND
['|', ('state', '=', 'draft'), ('state', '=', 'sent')]         # OR, prefix form
['&', ('a', '=', 1), '|', ('b', '=', 2), ('c', '=', 3)]        # nesting, prefix
[('partner_id.country_id.code', '=', 'AR')]                    # dotted relational traversal
[('line_ids', 'any', [('qty', '>', 0)])]                       # sub-domain on a relational field
```

Operators: `=`, `!=`, `>`, `>=`, `<`, `<=`, `in`, `not in`, `like`, `ilike`,
`not like`, `not ilike`, `=like`, `=ilike`, `child_of`, `parent_of`. `&`/`|`/`!`
are prefix (Polish notation) — the two operands of `|` follow it, not surround
it.

## `Command` (x2many writes)

```python
order.write({
    'order_line': [
        Command.create({'product_id': p.id, 'product_uom_qty': 1}),
        Command.update(line.id, {'product_uom_qty': 2}),
        Command.link(other_line.id),
        Command.unlink(line2.id),   # remove from this relation, keep the record
        Command.delete(line3.id),   # remove the relation AND delete the record
        Command.clear(),            # unlink everything currently there
        Command.set([id1, id2]),    # replace with exactly these ids (M2M-style)
    ],
})
```

`odoo.fields.Command` is an `IntEnum` (`fields.py:4282`) with classmethods
`create`/`update`/`delete`/`unlink`/`link`/`clear`/`set` that build the legacy
`(code, id, values)` tuples. Always import and use `Command.*` — never write
the raw tuples `(0, 0, {...})` in new code; they mean the same thing but are
unreadable and easy to get wrong. Over RPC, only the raw tuple form is valid
(no `Command` object survives serialization).

## Environment, sudo, context, company

```python
self.env                          # Environment: uid, context, cr, registry
self.env.user                     # res.users of the current uid
self.env.company                  # active company (single)
self.env.companies                # allowed companies (multi)
self.env['res.partner']           # empty recordset in another model
self.env.ref('base.group_user')   # lookup by XML id — never hardcode a numeric id

self.sudo()                       # bypass ir.rule + ir.model.access, keep self.env.uid for logging
self.sudo(False)                  # explicit un-sudo
self.with_context(lang='es_AR')   # new recordset, same records, different context
self.with_company(other_company)  # sets allowed_company_ids/company_id context keys
self.with_user(other_user)        # act as another user (permission-checked as them)
```

Every `.sudo()` needs a comment naming the rule it bypasses and why — see
`odoo-slop-audit` B6. `sudo()` does not disable `@api.constrains` or Python
validation, only `ir.model.access` and `ir.rule`.

## Exceptions

| Exception | When |
|---|---|
| `UserError` | expected, user-actionable failure — always wrap the message in `_()` |
| `ValidationError` | raised from `@api.constrains` or explicit business-rule checks |
| `AccessError` | raised by the framework on `ir.model.access`/`ir.rule` failure — don't raise it yourself for business logic |
| `MissingError` | a record referenced by id no longer exists (deleted between fetch and use) |
| `RedirectWarning` | `UserError` variant carrying an action to redirect to (e.g. "Configure a chart of accounts first" with a button) |

All live in `odoo.exceptions`. `except Exception: pass` around any of these
is a BLOCKER per `odoo-slop-audit` B8 — re-raise or surface a `UserError`.

## Raw SQL — the `SQL` wrapper (18.0)

```python
from odoo.tools import SQL

self.env.cr.execute(SQL(
    "UPDATE %s SET %s = %s WHERE id = %s",
    SQL.identifier(self._table), SQL.identifier('state'), 'done', record.id,
))
```

`odoo.tools.sql.SQL` (`tools/sql.py:48`) composes parameterized SQL safely —
nested `SQL` objects merge their params, a literal string `code` is guaranteed
injection-safe, `SQL.identifier(...)` escapes table/column names. Any SQL
built with f-strings or `%` directly on user input is a BLOCKER
(`odoo-slop-audit` B4). Raw SQL bypasses the ORM cache and record rules —
call `self.flush_model([...])` before reading what you just wrote through the
ORM, and `self.invalidate_model([...])` (or `self.env.invalidate_all()`)
after writing through SQL, before the ORM reads it again.

## 18.0 gotchas vs 17 / 19

| Gotcha | Detail |
|---|---|
| `models.Constraint` doesn't exist | that's a 19.0 declarative constraints API; 18.0 still uses `_sql_constraints` list + `@api.constrains` |
| `group_operator` deprecated | renamed `aggregator`; old kwarg still works but warns (`fields.py:482-484`) |
| `check_access` unified | 18.0 adds `check_access(operation)` / `has_access(operation)` (`models.py:4429-4451`) as the one call replacing the old two-step `check_access_rights()` + `check_access_rule()`. Both old methods still exist for compat but new code should call `check_access` |
| `company_dependent` fields are `jsonb` | not one column per company — a single jsonb column keyed by company id (`fields.py:776`) |
| `precompute` | 18.0 lets stored computed fields be filled before `INSERT` instead of via post-create `UPDATE` — opt-in, only valid on `store=True` computed fields |
| `_read_group(domain, groupby, aggregates)` | the modern internal signature (tuple-based); `read_group(domain, fields, groupby)` (dict-based) is the older public API, both exist in 18.0 |
| `search_fetch` | 18.0 addition — search + field read in one query, replaces `search()` then `.mapped()`/`.read()` for perf-sensitive batches |
| `post_init_hook(env)` | takes a single `env` argument in 18.0, not `(cr, registry)` — see `data-manifest.md` |
