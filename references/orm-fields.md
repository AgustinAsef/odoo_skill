# ORM Fields — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/odoo/fields.py`. Line numbers below are
verified against that file unless marked "docs only".

## Field types

| Type | Key params (beyond common) | Use for |
|---|---|---|
| `Char` | `size`, `trim` (default True) | short text |
| `Text` | — | long free text |
| `Html` | `sanitize=True`, `sanitize_tags`, `strip_style`, `strip_classes` | rich text, sanitized by default |
| `Boolean` | — | flags |
| `Integer` | `aggregator='sum'` (default) | counts, sequences |
| `Float` | `digits` (str referencing `decimal.precision` or tuple `(total, decimals)`), `aggregator='sum'` | measures, quantities |
| `Monetary` | `currency_field='currency_id'` (required companion field), `aggregator='sum'` | money amounts — always pair with a `Many2one('res.currency')` |
| `Date` | — | calendar date, stored as `date` |
| `Datetime` | — | UTC timestamp, stored as `timestamp` |
| `Binary` | `attachment=True` (default — stored as `ir.attachment`, not inline) | files |
| `Image` | `max_width`, `max_height`, `verify_resolution` | images, auto-resizes |
| `Selection` | `selection` (list of tuples or callable/method name string), `selection_add`, `ondelete` (per-value policy when extending), `group_expand` | closed enum |
| `Reference` | `selection` of model names | polymorphic FK, stores `"model,id"` as text — avoid, prefer `Many2one` + `active_model` pattern |
| `Many2one` | `comodel_name`, `ondelete` (`'set null'`/`'restrict'`/`'cascade'`), `check_company`, `domain`, `context`, `auto_join`, `delegate` | FK |
| `One2many` | `comodel_name`, `inverse_name` (the `Many2one` field on the comodel) | reverse FK, never stored as a column |
| `Many2many` | `comodel_name`, `relation`, `column1`, `column2`, `domain`, `check_company` | M2M through junction table (auto-named if omitted) |
| `Json` | — | opaque JSON blob |
| `Properties` | `definition`, `definition_record`, `definition_record_field` | dynamic per-record schema (studio-style), 18.0 addition |
| `PropertiesDefinition` | — | companion field holding the schema for `Properties` |

`Many2one`, `One2many`, `Many2many` all accept `check_company` (bool) — see
`security.md`. `fields.py:3108` (`Many2one`), `fields.py:3209-3210`,
`fields.py:4928-4929` (`One2many`/`Many2many` variants).

## Common `Field` base attributes

| Attr | Default | Meaning |
|---|---|---|
| `string` | field name, title-cased | UI label. **English**, per workspace convention |
| `help` | `''` | tooltip. **English** |
| `required` | `False` | NOT NULL at ORM level |
| `readonly` | `False` | not editable from UI (not a security control) |
| `index` | `False` | DB index; `True` gives btree, also accepts `'trigram'` for Char full-text |
| `default` | `None` | static value or a callable `lambda self: ...` / method name |
| `groups` | `None` | comma-sep external ids — field hidden and stripped from `fields_get`/values for users outside these groups |
| `copy` | `True` for regular, `False` for `One2many` | whether `copy()` duplicates the value |
| `tracking` | — | **not a core `Field` attribute.** Recognized only by `mail.thread` via `_valid_field_parameter` override (`addons/mail/models/mail_thread.py:482`). Declaring it on a model that doesn't inherit `mail.thread` is silently ignored |
| `company_dependent` | `False` | value is a `res.company`-scoped "property field", stored in a single `jsonb` column keyed by company (`fields.py:776`, `995-1008`) rather than a per-company row. Only allowed on a fixed set of column types (`COMPANY_DEPENDENT_FIELDS`); cannot be `required` or `translate` (`fields.py:463-469`) |
| `translate` | `False` | `Char`/`Text`/`Html` only |
| `aggregator` | `None` (`'sum'` on numeric types) | operator used by `_read_group`. **Replaces `group_operator`, deprecated since 18.0** — using `group_operator` raises a `DeprecationWarning` and is silently mapped (`fields.py:482-484`) |
| `group_expand` | `None` | method name (or `True` on `Selection`) to control/complete groups shown by `read_group` when grouping on this field (`fields.py:2900-2902`) |

## Computed / related / stored fields

```python
total = fields.Float(compute='_compute_total', store=True, digits='Product Price')

@api.depends('line_ids.price', 'line_ids.qty')
def _compute_total(self):
    for rec in self:
        rec.total = sum(l.price * l.qty for l in rec.line_ids)
```

- `store=False` (default for computed): value is never persisted, recomputed on
  every read, cannot be searched unless `search=` is given.
- `store=True`: persisted, kept in sync by the compute on every dependency
  change, searchable/sortable natively.
- `@api.depends` must list **every** field read inside the compute, including
  dotted paths through relations (`line_ids.price`). Missing a dependency is
  the single most common stale-cache bug (see `odoo-slop-audit` B7).
- `compute_sudo`: run the compute with `sudo()` semantics for access — use when
  the compute legitimately needs to read records the current user cannot.
- `precompute=True`: only valid on a **stored** computed field. Lets Odoo
  compute the value *before* the `INSERT` (in the same batch as other
  precomputed fields) instead of via a follow-up `UPDATE` after `create()`.
  Warns and is dropped if the field isn't `store`d, isn't computed, or (with
  some nuance) depends on a non-precomputed field (`fields.py:456-462`,
  `807-831`). Big perf win on `create(vals_list)` with many records.
- `inverse='_inverse_method'`: makes a non-stored (or stored-but-editable)
  computed field writable — the inverse method receives the new value and
  must write the underlying fields itself.
- `search='_search_method'`: makes a non-stored computed field usable in a
  domain — the method receives `(operator, value)` and must return an
  equivalent domain on stored fields.

```python
partner_name = fields.Char(related='partner_id.name', store=False)
```

- `related`: shortcut compute that follows a dotted path. `store=True` on a
  related field creates a real stored copy kept in sync automatically — no
  manual `@api.depends` needed, Odoo infers it from the path.
- `related` fields are `readonly=True` by default unless overridden.

## Onchange vs compute

| | `@api.onchange` | `compute=` |
|---|---|---|
| Runs | only in the UI form, on field change, before save | on every read (non-stored) or on every dependency write (stored) |
| Persists | never by itself — user must save | yes if `store=True` |
| RPC / server-side writes | **not triggered** — onchange is a UI-only simulation | always triggered |
| Use when | you want to suggest/warn without forcing a value | the value must always be correct regardless of entry point |

Prefer `compute` + `store=True` for anything that must hold under RPC, import,
or server actions — `onchange` alone is invisible to `write()` calls that
don't go through the form view.

## Constraints

```python
_sql_constraints = [
    ('qty_positive', 'CHECK(qty > 0)', 'Quantity must be positive.'),
]

@api.constrains('qty', 'price')
def _check_qty_price(self):
    for rec in self:
        if rec.qty and rec.price < 0:
            raise ValidationError(_("Price cannot be negative when quantity is set."))
```

- `_sql_constraints` is still the 18.0 mechanism (`models.py:609`) — declared
  as a class list, name gets the table prefix on creation
  (`models.py:3508-3516`). **`models.Constraint` as a class-based declarative
  API does not exist in 18.0** — that is an Odoo 19 addition. Do not use it.
- `@api.constrains(*fields)`: Python-level check run on `create`/`write` of
  the listed fields, raises `ValidationError`. Runs per-record inside `self`;
  loop over `self` explicitly.
- SQL constraints catch violations at the DB level (fast, but the error
  message is generic unless mapped); Python constraints give a precise
  message but only run through the ORM, never on raw SQL writes.

## Defaults

```python
state = fields.Selection([...], default='draft')
date = fields.Date(default=fields.Date.context_today)
user_id = fields.Many2one('res.users', default=lambda self: self.env.user)
```

- A callable default receives `self` (an empty recordset in the right model,
  with the right context) — never call `fields.Datetime.now()`/`.today()` as
  a plain Python value; pass the method reference so it re-evaluates per
  record. `fields.Datetime.now` at `fields.py:2409`, `fields.Date.today` at
  `fields.py:2303`, `fields.Date.context_today` at `fields.py:2311`
  (timezone-aware — prefer this over `.today()` for anything user-facing, per
  `odoo-slop-audit` B5).

## `display_name` / `name_search`

18.0 has no `name_get()` override hook. Instead:

- Override `_compute_display_name` (a real computed field, `models.py:1793`,
  depends on `_rec_name` automatically) to customize how a record's label is
  built.
- `_rec_name` (`models.py:611`): the field used for the default label
  (`'name'` if present).
- `_rec_names_search` (`models.py:612`): list of extra fields consulted by
  `name_search`/searching on `display_name`, beyond `_rec_name`.
- Override `name_search(self, name='', args=None, operator='ilike', limit=100)`
  (`models.py:1872`) directly only when the search logic can't be expressed
  as `_rec_names_search` + a `search=` on a computed display field.
