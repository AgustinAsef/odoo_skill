# QWeb PDF Reports — Odoo 18.0

The only report engine Odoo core ships: QWeb HTML templates rendered to PDF via
`wkhtmltopdf`. This is "the way" Odoo wants a printable document built — do not
reach for a Python PDF library unless the report is not really document-shaped.

## Minimal working report: the three pieces

1. An `ir.actions.report` record — the button/menu entry, points at a template.
2. A QWeb `<template>` — the actual markup, wrapped in the standard layout.
3. (Optional, for anything beyond simple field display) a `report.<module>.<name>`
   Python model overriding `_get_report_values`.

### 1. The action record

Real, verbatim example (`addons/account/views/account_report.xml:5-17`,
`:31-41`):

```xml
<record id="account_invoices" model="ir.actions.report">
    <field name="name">Invoice PDF</field>
    <field name="model">account.move</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">account.report_invoice_with_payments</field>
    <field name="report_file">account.report_invoice_with_payments</field>
    <field name="print_report_name">(object._get_report_base_filename())</field>
    <field name="attachment"/>
    <field name="binding_model_id" ref="model_account_move"/>
    <field name="binding_type">report</field>
</record>
```

Field notes, confirmed against `odoo/addons/base/models/ir_actions_report.py`:

| Field | Meaning |
|---|---|
| `model` | technical name of the model the report is generated for |
| `report_type` | `qweb-pdf` (the normal case), `qweb-html`, `qweb-text` |
| `report_name` | `<module>.<template_xmlid>` — must match the `<template id=...>` xmlid, module-qualified. `report_model_name = 'report.%s' % report.report_name` at `ir_actions_report.py:1106` is exactly how the custom Python model name is derived, see below |
| `report_file` | conventionally the same value as `report_name`; used for default download filename fallback |
| `print_report_name` | a Python expression string, evaluated with `object` bound to the record — sets the downloaded filename, e.g. `(object._get_report_base_filename())` or `'Invoice - %s' % (object.name)` |
| `binding_model_id` | set (`ref="model_<model>"`) to make the report appear in that model's Print menu; `eval="False"` explicitly removes it from the Print menu while keeping the action callable |
| `binding_type` | `report` (default) vs `action` |
| `paperformat_id` | m2o to `report.paperformat`; falls back to `self.env.company.paperformat_id` when unset (`ir_actions_report.py:274`) |
| `groups_id` | restrict who can print it, same semantics as view `groups_id` |
| `attachment` / `attachment_use` | Python expression for a filename to save the generated PDF as an `ir.attachment`; `attachment_use=True` re-serves the cached attachment instead of re-rendering on repeat prints |

### 2. The template — standard layout

Real, verbatim skeleton (`addons/account/views/report_invoice.xml`, template
id `report_invoice_document`, called by the top-level `account.report_invoice`
template which wraps it in `web.html_container` + `t-foreach docs`):

```xml
<template id="report_invoice">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="account.report_invoice_document" t-lang="o.partner_id.lang"/>
        </t>
    </t>
</template>

<template id="report_invoice_document">
    <t t-call="web.external_layout">
        <t t-set="o" t-value="o.with_context(lang=lang)"/>
        <div class="page">
            <h2><span t-field="o.name"/></h2>
            <div t-field="o.partner_id"
                 t-options='{"widget": "contact", "fields": ["address", "name"], "no_marker": True}'/>
        </div>
    </t>
</template>
```

Layout building blocks:

- `web.html_container` — outermost wrapper, sets up the HTML document + head
  (must wrap every top-level report template).
- `web.external_layout` — company header/footer, letterhead-style branding,
  reads `self.env.company` / `res_company.layout_background` etc. Use for
  anything sent to a customer.
- `web.internal_layout` — bare header/footer, no branding, for internal-only
  documents.
- `<div class="page">` — one logical PDF page's content block; `wkhtmltopdf`
  paginates automatically, this class only sets print CSS margins/box.

Rendering context available inside a template (doc-only, reports.html):
`docs` (the recordset being printed), `doc_ids`, `doc_model`, `user`,
`res_company`, `web_base_url`, `context_timestamp`. When a custom
`_get_report_values` override is used (below), whatever it returns replaces/
extends this dict — `docs` must be re-supplied there if the override is used.

### Translatable reports — the `t-lang` pattern

Confirmed live pattern at `report_invoice.xml`: the outer template loops
`docs` and calls the per-record template with `t-lang="o.partner_id.lang"` on
the `t-call`. This re-renders that sub-template with the partner's language
active, so "Invoice Date" etc. translate per-recipient even when the batch
mixes languages (e.g. printing 20 invoices for customers in different
countries in one PDF). Do not try to set `lang` via `with_context` alone
on the outer loop — `t-lang` on `t-call` is what actually swaps the
translation catalog for that sub-render.

```xml
<t t-call="my_module.my_document_body" t-lang="o.partner_id.lang"/>
```

### 3. Custom report model — `report.<module>.<name>`

Needed only when the template needs computed/aggregated data beyond what
plain field access can do in QWeb (e.g. grouping, running totals, output that
isn't 1:1 with `docids`).

```python
from odoo import models


class ReportMyDocument(models.AbstractModel):
    _name = "report.my_module.report_my_document"
    _description = "My Document Report"

    def _get_report_values(self, docids, data=None):
        docs = self.env["my.model"].browse(docids)
        return {
            "doc_ids": docids,
            "doc_model": "my.model",
            "docs": docs,
            "extra_totals": self._compute_totals(docs),
        }

    def _compute_totals(self, docs):
        return sum(docs.mapped("amount_total"))
```

The model `_name` must be exactly `report.<report_name>` where `report_name`
is the action's `report_name` field value — i.e. `report.<module>.<template_xmlid>`.
Confirmed mechanically at `ir_actions_report.py:1106`:
`report_model_name = 'report.%s' % report.report_name`, then
`data.update(report_model._get_report_values(docids, data=data))` at
`ir_actions_report.py:1117`. Get the `_name` wrong and Odoo silently falls
back to the default context builder — no error, just missing keys in the
template.

## Print button

```xml
<button name="%(my_module.action_report_my_document)d" type="action"
        string="Print" icon="fa-print"/>
```

Or, with `binding_model_id` set on the action, it appears automatically in the
model's Print dropdown menu — no button needed in the form view at all.
Prefer the binding over a manual button unless the print needs to be gated by
a state condition the Print menu can't express.

## Barcodes

```xml
<img t-att-src="'/report/barcode/QR/%s' % o.name"/>
<img t-att-src="'/report/barcode/?barcode_type=Code128&amp;value=%s&amp;width=300&amp;height=100' % o.barcode"/>
```

The `/report/barcode/<type>/<value>` controller renders common symbologies
(`QR`, `Code128`, `EAN13`, `UPCA`, ...) server-side as a PNG; query-string form
adds width/height/humanreadable options.

## `report.paperformat`

`ir.actions.report.paperformat_id` (m2o) drives the `wkhtmltopdf` command line
directly — page size/format, margins, orientation, dpi, header/footer spacing
(`ir_actions_report.py:274,289-354`). Create one per module only when the
default company paperformat is wrong for that document (e.g. a shipping label
that needs a non-A4 format) — otherwise leave it unset and let it inherit
`self.env.company.paperformat_id`.

## XLSX: no native QWeb engine

Odoo core's report engine is HTML-to-PDF only. There is no `qweb-xlsx` report
type in core. Producing a real `.xlsx` output means either:

- the OCA `report_xlsx` module (external dependency, not in core/Enterprise) — adds an `xlsx` report type and an `xlsxwriter`-based template class, or
- a plain server action / controller that builds the file with `xlsxwriter` directly and returns it as a download — no `ir.actions.report` involved at all.

Do not invent a `qweb-xlsx` report_type — it does not exist in this version.

## Analytical (SQL view) reports

Not a PDF document — a read-only, pivot/graph-able model backed by a SQL
`VIEW` instead of a real table. This is how core builds every "Analysis"
screen (`sale.report`, `purchase.report`, `stock.report`, ...). Reach for
this instead of a QWeb template when the deliverable is an aggregated data
grid the user pivots/filters/exports, not a fixed printable document.

### Model declaration

```python
class SaleReport(models.Model):
    _name = "sale.report"
    _description = "Sales Analysis Report"
    _auto = False              # no physical table — the DB object is a VIEW
    _rec_name = 'date'
    _order = 'date desc'

    name = fields.Char(string="Order Reference", readonly=True)
    partner_id = fields.Many2one('res.partner', readonly=True)
    price_subtotal = fields.Monetary(readonly=True)
    price_unit = fields.Float(readonly=True, aggregator='avg')
```

Verified against `addons/sale/report/sale_report.py:8-11,20-24,83-84` (real,
current 18.0 core code) — `_auto = False`, every field `readonly=True`
(there is nothing to write back to), and a non-additive numeric field gets
an explicit `aggregator=` (`'avg'` here) so pivot/graph don't silently sum it.

### Populating the view: `_table_query` (the pattern core actually uses)

```python
@property
def _table_query(self):
    return self._query()

def _query(self, with_=False, fields=None, groupby=None, from_clause=None):
    return f"""
        {"WITH " + with_ + " " if with_ else ""}
        SELECT {self._select_sale()}
        FROM {self._from_sale()}
        WHERE {self._where_sale()}
        GROUP BY {self._group_by_sale()}
    """
```

Verified `addons/sale/report/sale_report.py:98,175,183,197,201,243-245` — the
query is split into overridable `_select_sale()`/`_from_sale()`/
`_where_sale()`/`_group_by_sale()` helper methods precisely so another module
can extend one clause via `super()` without rebuilding the whole query. This
`_table_query` property is the pattern used throughout core in 18.0.

### `init()` + `tools.drop_view_if_exists` — the older/alternative pattern

```python
from odoo import tools

def init(self):
    tools.drop_view_if_exists(self.env.cr, self._table)
    self.env.cr.execute(f"""
        CREATE OR REPLACE VIEW {self._table} AS (
            SELECT {self._select()} FROM {self._from()}
        )
    """)
```

Still valid and still present in older/simpler core reports, but `_table_query`
is preferred in new code: it recomputes the view lazily on each ORM access
instead of requiring `init()` to re-run (which only fires on module
install/update), so it reflects overrides from modules installed *after* the
defining one without needing its own migration. Doc-only claim
(`howtos/create_reports.html`) for the preference; the mechanical difference
(`init()` runs once at `-i`/`-u`, `_table_query` evaluates per query) is
verified from `_auto`/`init` handling in `odoo/models.py`.

### Security and views

`_auto = False` does not exempt the model from `ir.model.access` — it still
needs a `security/ir.model.access.csv` row, normally **read-only**
(`perm_read=1, perm_write=0, perm_create=0, perm_unlink=0` — write/create/
unlink against a view fail at the SQL level anyway). Verified
`addons/sale/security/ir.model.access.csv:56`:
`access_sale_report_salesman,sale.report,model_sale_report,sales_team.group_sale_salesman,1,0,0,0`.
Pair it with an `ir.rule` if the underlying data is multi-company or
per-salesperson scoped — the view itself enforces no row-level security.

Expose it through `<pivot>`/`<graph>`/`<list>` views, `view_mode` in that
order for the default action tab (`view_mode="pivot,graph,list"` is the
common core convention, e.g. `addons/sale/report/sale_report_views.xml:128`)
— `<form>` is normally omitted or added last, since there is nothing to edit.

## Spreadsheet / `account.report` alternative

For accounting-style tabular output (aged balance, tax report, general
ledger-like grids), Odoo Enterprise's `account.report` engine (dynamic
columns, drilldown, export to PDF/XLSX built in) is usually the right tool
instead of hand-rolling a QWeb template — see the workspace's own
`Documentacion/Documentacion-Contabilidad-Odoo18/` and
`odoo-estado-cuenta-cliente-columnas` memory for how an existing module
(`account.report`, not `account.followup.report`) was extended. Reach for a
plain QWeb PDF template only when the output is a real printable document
(invoice, delivery slip, label) rather than a data grid.

## Practical notes when building a new one

- `report_name` on the action and `id` on the `<template>` must match
  exactly, module-qualified (`account.report_invoice`, not just
  `report_invoice`) — a mismatch fails silently with a generic "template not
  found" only at print time, not at module install.
- `_get_report_values` must return `docs` (or whatever key the template
  reads) — it fully replaces the default context, it does not merge into it.
- `print_report_name` is evaluated as a Python expression string against a
  namespace with `object` bound to the record — no f-strings, use `%` or
  `.format()`/string concatenation inside the expression text.
- A document reachable from a public/portal controller (e.g.
  `/report/pdf/...`) needs the same sudo/access-rule scrutiny as any other
  public endpoint — see Part B6 of the slop-audit rubric.
