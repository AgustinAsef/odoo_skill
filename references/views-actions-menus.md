# Views, Actions, Menus — Odoo 18.0

LLM-first reference for the presentation layer: view XML, window/report/server/client
actions, menus. Verify against local source before trusting anything marked
"doc-only" below — the local V18 tree is the ground truth for this workspace.

## 18.0 breaking changes (confirmed against local source)

| Change | Confirmed at |
|---|---|
| `<tree>` renamed to `<list>`. `view_mode` uses the string `"list"`, not `"tree"`. There is no `tree_view.rng` in base/rng — only `list_view.rng`. | `Versiones/V18/odoo-18.0/odoo/addons/base/rng/list_view.rng` (file exists; no tree_view.rng) |
| `attrs` and `states` attributes are **rejected outright** since 17.0 — view validation raises `ValidationError` if either is present anywhere in the arch. | `odoo/addons/base/models/ir_ui_view.py:385-387` |
| `invisible`, `readonly`, `required` are plain Python expressions evaluated against the record/view context — no dict wrapper. | doc-only (developer docs, view_architectures.html), pattern confirmed live in `addons/sale/views/sale_order_views.xml` |
| `column_invisible` is the list-view-only visibility attribute (hides a column, not the record). Widely used for technical fields inside list/one2many. | `addons/sale/views/sale_order_views.xml:24,57` and `sale_order_line_views.xml:19` |
| `<chatter/>` self-closing tag replaces the old `<div class="oe_chatter">` + message/activity widget boilerplate in form views. | `addons/mail/views/res_partner_views.xml:22` |

```xml
<!-- invisible/readonly as Python expressions, no attrs= -->
<field name="team_id"
       column_invisible="context.get('default_move_type') not in ('out_invoice','out_refund','out_receipt')"/>
<field name="partner_shipping_id" invisible="not partner_id" readonly="state != 'draft'"/>
```

## View XML id naming

Convention (doc-only, view_records.html): `<module>.<technical_model_with_dots_to_underscores>_view_<type>`
e.g. `sale.view_order_form`, `crm.view_lead_list`. In practice the workspace's own
modules follow `view_<model>_<type>` inside the module's own namespace (the module
prefix is implicit via the XML id's module). Keep `form`/`list`/`search`/`kanban`
etc. as the trailing segment — never abbreviate it.

Action ids: `action_<model_or_purpose>`. Menu ids: `menu_<parent>_<purpose>`.

## `ir.ui.view` record shape

```xml
<record id="view_my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <form>
            ...
        </form>
    </field>
</record>
```

Fields: `name` (free label, convention `<model>.<type>`), `model` (required),
`arch` (`type="xml"`), `priority` (int, lower wins as default), `inherit_id`
(m2o to parent view — triggers inheritance mode), `mode` (`primary` default,
or `extension` when the view is a pure delta merged into its parent at load
time — extension views never appear standalone in an action), `groups_id`
(restrict visibility), `key` (used by website-editable views, rarely needed
in backend modules).

## Inheritance via xpath

```xml
<record id="view_my_model_form_inherit_x" model="ir.ui.view">
    <field name="name">my.model.form.inherit.x</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="my_module.view_my_model_form"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='partner_id']" position="after">
            <field name="x_extra_field"/>
        </xpath>
    </field>
</record>
```

`position` values: `before`, `after`, `inside` (append as last child),
`replace` (drop the matched node, or use `<xpath position="replace">` with
the special child `$0` to keep-and-rewrap the original node), `attributes`
(child `<attribute name="...">value</attribute>` elements only — no other
children allowed under `attributes` position).

Shorthand form (no `<xpath>` needed when matching a single top-level tag by
name/attrs):

```xml
<field name="partner_id" position="after">
    <field name="x_extra_field"/>
</field>
```

Every element in the arch can be a match target this way, including
`<list position="attributes">` (seen live at `sale_order_views.xml:186`) to
patch attributes of the root list tag itself.

## Form view

```xml
<form>
    <header>
        <button name="action_confirm" type="object" string="Confirm"
                class="btn-primary" invisible="state != 'draft'"/>
        <field name="state" widget="statusbar"/>
    </header>
    <sheet>
        <div class="oe_title">
            <field name="name" placeholder="Name..."/>
        </div>
        <group>
            <group>
                <field name="partner_id"/>
            </group>
            <group>
                <field name="date"/>
            </group>
        </group>
        <notebook>
            <page string="Lines" name="lines">
                <field name="line_ids">
                    <list editable="bottom">
                        <field name="product_id"/>
                        <field name="qty"/>
                    </list>
                </field>
            </page>
        </notebook>
    </sheet>
    <chatter/>
</form>
```

- `<header>`: workflow buttons + statusbar, sits above `<sheet>`, not padded.
- `<sheet>`: the paper-like responsive container; everything business goes inside.
- `<group>`: two-column layout grid; nested `<group>` splits into sub-columns.
- `<notebook>`/`<page>`: tabs. `<page>` needs `string` and usually `name`.
- `<chatter/>`: self-closing, replaces the old manual mail-thread widget wiring.
  Place as a direct child of `<form>`, after `</sheet>`.
- `<field name="line_ids">` embedding a `<list>` defines the one2many's inline
  editable grid; `editable="bottom"|"top"` makes rows inline-editable.
- `<button type="object">` calls a model method by name; `type="action"` calls
  a stored `ir.actions.*` record by `name` (xmlid via `%(module.xmlid)d` in
  older syntax, or plain `name` matching an action's technical name — prefer
  `type="object"` for anything Python-side).

## List view (`<list>`, formerly `<tree>`)

```xml
<list class="o_my_list" decoration-muted="state == 'cancel'"
      decoration-success="state == 'done'" editable="bottom" default_order="date desc">
    <field name="name" decoration-bf="1"/>
    <field name="partner_id" optional="show"/>
    <field name="amount_total" sum="Total" widget="monetary"/>
    <field name="currency_id" column_invisible="1"/>
    <button name="action_done" type="object" string="Done" icon="fa-check"/>
</list>
```

- `decoration-<bootstrap-class>="<python expr>"` on the root `<list>`: row-level
  styling (`decoration-info`, `-success`, `-warning`, `-danger`, `-muted`,
  `-bf` bold, `-it` italic). Confirmed live at `sale_order_views.xml:116-169`.
  Field-level `decoration-bf="1"` bolds a single cell.
- `column_invisible="1"`: hides the column entirely (still loaded — needed by
  other widgets/decorations on the same row). Different from `invisible`,
  which on a `<list>` column also just hides it — but `column_invisible` is the
  documented, ORM-aware way used throughout core.
- `optional="show"|"hide"`: user-togglable column via the gear menu.
- `sum=`/`avg=`: footer aggregate label, only meaningful on numeric fields.
- `editable="bottom"|"top"`: makes the whole list inline-editable (used for
  one2many lines; a standalone list view for a real model can be editable too).

## Search view

```xml
<search>
    <field name="name" string="Reference"/>
    <field name="partner_id"/>
    <filter string="My Orders" name="my_orders" domain="[('user_id','=',uid)]"/>
    <filter string="Draft" name="draft" domain="[('state','=','draft')]"/>
    <separator/>
    <filter string="Late Activities" name="activities_overdue"
            domain="[('activity_ids.date_deadline','&lt;', context_today().strftime('%Y-%m-%d'))]"/>
    <group expand="0" string="Group By">
        <filter string="Salesperson" name="salesperson" domain="[]"
                context="{'group_by': 'user_id'}"/>
        <filter string="Order Date" name="order_month" domain="[]"
                context="{'group_by': 'date_order'}"/>
    </group>
</search>
```

Confirmed pattern at `addons/sale/views/sale_order_views.xml:846-857`.
`<field>` in a search view adds it to the free-text search box (optionally with
`filter_domain` for custom matching). `<filter>` is a togglable predefined
domain; grouping filters set `context="{'group_by': '<field>'}"` inside a
`<group>` block labeled "Group By" by convention (any string works, but stick
to that label). `<searchpanel>` adds the sidebar facet tree (used on list/kanban).

## Kanban view (minimal)

```xml
<kanban class="o_kanban_small_column">
    <field name="state"/>
    <templates>
        <t t-name="card">
            <field name="name"/>
            <field name="partner_id"/>
        </t>
    </templates>
</kanban>
```

Note: Odoo 17+ kanban templates use `t-name="card"` as the single supported
record template name (the old `kanban-box` name from 16 and earlier is
deprecated but still widely present in core for backward compat — doc-only,
verify against the specific view before copying an old pattern).

## Common widgets by field type

| Field type | Widget(s) |
|---|---|
| `many2one` | default; `widget="many2one_avatar"`, `many2one_barcode` |
| `many2many` | `many2many_tags`, `many2many_tags_avatar`, `many2many_checkboxes` |
| `one2many` | inline `<list>`/`<kanban>` child arch (no `widget=` needed) |
| `selection` | default dropdown; `widget="radio"`, `widget="statusbar"` (on `state`-like fields with `statusbar_visible=`) |
| `boolean` | default checkbox; `widget="boolean_toggle"` |
| `monetary` | `widget="monetary"` (needs a `currency_id` field in the view, can be `column_invisible`) |
| `date`/`datetime` | default; `widget="daterange"` for a pair of fields |
| `text`/`html` | default `text`; `widget="html"` for rich text |
| `binary` | default download link; `widget="image"` for images, `widget="pdf_viewer"` |
| `char` | default; `widget="url"`, `widget="email"`, `widget="phone"`, `widget="badge"` |
| `integer`/`float` | default; `widget="percentage"`, `widget="progressbar"` |

## Actions

### Window action (`ir.actions.act_window`)

```xml
<record id="action_my_model" model="ir.actions.act_window">
    <field name="name">My Models</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">list,form</field>
    <field name="domain">[('active', '=', True)]</field>
    <field name="context">{'default_state': 'draft'}</field>
</record>
```

`view_mode` is a comma-separated string using the new names: `list`, `form`,
`kanban`, `calendar`, `pivot`, `graph`, `activity`, `gantt`, `map`. `target`:
`current` (default, replaces breadcrumb), `new` (dialog), `fullscreen`,
`inline` (embedded, no chrome). `views` field (computed from `view_mode` when
omitted) lets you pin a specific `view_id` per type as `[(id, 'list'), (id, 'form')]`.

### Report action — see `reports-qweb.md`.

### Server action (`ir.actions.server`)

```xml
<record id="action_server_my_task" model="ir.actions.server">
    <field name="name">Run My Task</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">records.action_do_something()</field>
</record>
```

`state` selects the execution mode: `code` (Python snippet, `records`/`env`/
`model` in scope), `object_create`, `object_write`, `multi` (chain other
server actions), `webhook`, etc. Prefer a real model method over inline
`code` for anything non-trivial — the action becomes a one-line dispatcher.

### Client action (`ir.actions.client`)

```xml
<record id="action_my_client" model="ir.actions.client">
    <field name="name">My Dashboard</field>
    <field name="tag">my_module.dashboard</field>
</record>
```

`tag` must match a key registered in the JS `actions` registry
(`registry.category("actions").add("my_module.dashboard", MyDashboardComponent)`
— see `frontend-owl.md`).

### Buttons that call actions vs methods

```xml
<button name="action_my_server_action_xmlid" type="action" string="Run"/>
<button name="action_confirm" type="object" string="Confirm"/>
```

`type="object"`: calls `self.action_confirm()` on the current record(s) —
by far the most common case for a workflow button.
`type="action"`: `name` is the **technical name / xmlid** of a stored
`ir.actions.*` record, not a method — used to open a related window action.

## Menus (`ir.ui.menu`)

```xml
<menuitem id="menu_my_root" name="My App" sequence="10" web_icon="my_module,static/description/icon.png"/>
<menuitem id="menu_my_model" name="My Models" parent="menu_my_root"
          action="action_my_model" sequence="10"/>
```

`<menuitem>` is XML sugar over a `<record model="ir.ui.menu">`. Top-level
(app) menu needs `web_icon="<module>,<relative_path_to_icon>"`. `action`
points to any action xmlid (window/client/server). `groups` restricts
visibility, same as views.
