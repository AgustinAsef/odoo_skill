# Security — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/odoo/addons/base/models/ir_model.py`
(`ir.model.access`), `.../ir_rule.py`, `odoo/models.py` (`check_access`),
`odoo/fields.py` (`check_company`/`_check_company_auto`).

## Groups (`res.groups`)

```xml
<record id="group_foo_manager" model="res.groups">
    <field name="name">Foo Manager</field>
    <field name="category_id" ref="base.module_category_foo"/>
    <field name="implied_ids" eval="[(4, ref('group_foo_user'))]"/>
    <field name="comment">Full access to Foo.</field>
</record>
```

- `implied_ids`: granting this group auto-grants the listed groups too
  (manager implies user).
- `category_id`: groups the entry under an app in Settings > Users > access
  rights matrix.
- File goes in `security/foo_security.xml`, loaded **before**
  `ir.model.access.csv` in the manifest `data` list (access rows reference
  group XML ids).

## `ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_foo_user,foo.user,model_foo,group_foo_user,1,1,1,0
access_foo_manager,foo.manager,model_foo,group_foo_manager,1,1,1,1
```

- `model_id:id` is `model_<model_name_with_underscores>`, e.g. `sale.order` →
  `model_sale_order`.
- `group_id:id` empty = applies to **every** authenticated user (and
  portal/public if those groups are otherwise reachable) — never leave it
  empty on a model holding another party's data.
- Rows across groups are **additive**: a user in two groups gets the union of
  permissions. There's no way to subtract via `ir.model.access` — that's what
  record rules are for.
- File: `security/ir.model.access.csv`, referenced literally in the manifest
  `data` list (CSV files are always loaded, `noupdate` does not apply to
  them the same way — Odoo re-applies the file's current content on every
  upgrade).

## Record rules (`ir.rule`)

```xml
<record id="foo_rule_own_company" model="ir.rule">
    <field name="name">Foo: multi-company</field>
    <field name="model_id" ref="model_foo"/>
    <field name="domain_force">[('company_id', 'in', company_ids)]</field>
    <field name="groups" eval="[(4, ref('base.group_multi_company'))]"/>
</record>
```

`domain_force` evaluation context (from `ir_rule.py:_eval_context`):

| Name | Value |
|---|---|
| `user` | current `res.users` recordset |
| `company_id` | `self.env.company.id` — the single active company |
| `company_ids` | `self.env.companies.ids` — all allowed companies |
| `time` | the stdlib `time` module |

- **Global rule** (no `groups`, or `groups` left empty): domain is ANDed onto
  every access for every user, regardless of group. Used to enforce something
  no one may bypass (e.g. multi-company isolation).
- **Group-scoped rule** (`groups` set): domains from all matching rules for
  the user's groups are OR-ed together, then ANDed with the global rules.
  Adding a group-scoped rule *widens* what that group can see relative to
  other group rules, never narrows below the global rules.
- A model with **zero** rules and at least one `ir.model.access` row is fully
  visible to anyone with that access row — rules only restrict, they never
  grant.
- Copying a rule's domain onto a field that can be `False`/unset silently
  hides those records for everyone the rule applies to — this is a known
  workspace gotcha (see root `CLAUDE.md` memory `odoo-multicompania-ir-rule`).
  Verify a nullable field's rule domain explicitly handles the null case if
  "invisible record" is not the intended behavior.

## Multi-company (`check_company`)

```python
warehouse_id = fields.Many2one('stock.warehouse', check_company=True)
_check_company_auto = True
```

- `check_company=True` on a `Many2one`/`One2many`/`Many2many`: marks the
  field to be validated by `_check_company()` — the related record's company
  must be empty or match `self.company_id` (or be in `self.company_ids` for
  the M2M/O2M form). `fields.py:3108`, `3209-3210`, `4928-4929`.
- `_check_company_auto = True` on the model (`models.py:630-632`): calls
  `_check_company()` automatically on every `create`/`write` that touches a
  `check_company` field, instead of requiring an explicit call. Default is
  `False` — a model with `check_company=True` fields but no
  `_check_company_auto` only validates when something calls
  `self._check_company()` manually.
- This is a data-integrity check, not an access check — it prevents mixing
  records from company A into a document that belongs to company B; it does
  **not** by itself hide company B's records from a company-A user (that's
  the job of an `ir.rule` using `company_id in company_ids`).
- Never put `check_company=True` on the `company_id` field itself (doc-only,
  `howtos/company.html`) — it checks a relation's company against `self`'s
  own `company_id`/`company_ids`, which is meaningless applied to that field.
- The check is strict in the other direction too: if `self.company_id` is
  **empty**, every `check_company=True` relation on it must also point to a
  record with an empty `company_id` — an unset company does not mean "any
  company is fine". Verified `odoo/models.py:4328-4338`
  (`_check_company_domain`): with no `companies` given it returns
  `[('company_id', '=', False)]`, not an unrestricted domain. A record left
  without a company can silently fail `_check_company` against a normal,
  single-company related record — set `company_id` (or explicitly allow the
  relation to be company-less too) rather than leaving it blank "to be safe".

## `sudo()` usage rules

- Every `.sudo()` call needs a `# SAFETY:`-style comment naming which access
  rule/model it bypasses and why the bypass is intentional and safe
  (`odoo-slop-audit` B6).
- An unexplained `sudo()` reachable from a public or portal-facing route
  (controller, portal template, public form) is a BLOCKER — it is the single
  most common way a workspace module leaks cross-company or cross-customer
  data.
- `sudo()` does not bypass `@api.constrains` / Python-level `ValidationError`
  checks — only `ir.model.access` and `ir.rule`. Use it to read/write records
  the current user legitimately shouldn't need direct access to (e.g. a
  portal user's own linked `sale.order` looked up through a public
  controller), never as a blanket "make the permission error go away" fix.

## Field-level groups

```python
cost_price = fields.Float(groups="base.group_system")
```

```xml
<field name="cost_price" groups="base.group_system"/>
```

- On the field definition: the field is stripped entirely from `fields_get`
  and from any `read()`/`search()` result for users outside the listed
  groups (comma-separated = OR, any one group is enough) — this is a real
  security boundary, not just UI hiding.
- On a `<field>` in a view: only hides the widget for that view; the field is
  still readable via RPC/other views unless the field definition itself also
  carries `groups=`. Use the view-level `groups` for UX, the field-level
  `groups` when the data must actually be inaccessible.

## `check_access` (new unified call, 18.0)

```python
records.check_access('write')      # raises AccessError if not allowed
records.has_access('write')        # -> bool, no exception
allowed = records._filtered_access('write')  # subset the user may write
```

`models.py:4429-4487`. Replaces manually chaining
`check_access_rights()` + `check_access_rule()` (both still exist for
backward compat, `models.py:4490` onward, but new code should use
`check_access`/`has_access`). Internally still goes through
`self.env['ir.model.access'].check(...)` (`ir_model.py:2151`) and
`self.env['ir.rule']._compute_domain(...)` (`ir_rule.py:142`).
