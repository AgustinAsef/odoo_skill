# Manifest & Data Files — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/odoo/modules/module.py:42-79`
(`_DEFAULT_MANIFEST`), `odoo/modules/loading.py:244-246` (hook calling).

## `__manifest__.py` keys

```python
{
    'name': 'Foo',                       # required, human-readable
    'version': '18.0.1.0.0',             # workspace: 5 segments, 18.0 prefix
    'category': 'Sales',                 # "/" for subcategory, e.g. 'Sales/CRM'
    'summary': 'One-line pitch',
    'description': '''Longer text (RST/plain).''',
    'author': 'OLPA GROUP',              # workspace: exact casing
    'website': 'https://olpagroup.com/', # workspace: trailing slash
    'license': 'LGPL-3',                 # required by odoo.sh lint, no default enforced in code
    'depends': ['base', 'sale'],
    'external_dependencies': {'python': ['xlsxwriter'], 'bin': ['wkhtmltopdf']},
    'data': [                            # loaded on install AND update, in listed order
        'security/foo_security.xml',
        'security/ir.model.access.csv',
        'data/foo_data.xml',
        'views/foo_views.xml',
        'wizard/foo_wizard_views.xml',
        'report/foo_report.xml',
    ],
    'demo': [],                          # loaded only when demo data is enabled
    'assets': {'web.assets_backend': ['foo/static/src/js/foo.js']},
    'installable': True,
    'auto_install': False,               # True, or a list of module names (18.0: "install when ALL of these are also installed")
    'application': False,                # True = shows as a top-level App, not a technical module
    'pre_init_hook': 'pre_init',
    'post_init_hook': 'post_init',
    'uninstall_hook': 'uninstall',
}
```

Full default-key list as the loader actually reads it
(`module.py:42-79`): `application`, `assets`, `author`, `auto_install`,
`category`, `data`, `demo`, `depends`, `description`,
`external_dependencies`, `installable`, `post_init_hook`, `post_load`,
`pre_init_hook`, `sequence`, `summary`, `uninstall_hook`, `version`,
`website`. (`icon` and `license` are handled specially — `icon` auto-resolves
to `static/description/icon.png`; `license` and `name` have no default and
must be set explicitly, though only `name` is truly required to load.)
`init_xml`/`update_xml`/`demo_xml`/`test` are legacy pre-8.0 keys, still
accepted but dead — use `data`/`demo` instead.

`auto_install` as a **list** (18.0/17.0 behavior): the module installs
automatically only once the listed modules are *also* about to be installed
in the same transaction — a bridge module auto-installing on top of already-
installed dependencies does **not** fire just because those dependencies
exist; at least one of them has to be newly "to install" in that same run
(see `odoo-auto-install-no-alcanza` — a workspace-verified gotcha).

## Workspace manifest conventions (from root `CLAUDE.md`)

- `version`: `'18.0.X.Y.Z'` — five segments, `18.0` prefix.
- `author`: exactly `'OLPA GROUP'`.
- `website`: exactly `'https://olpagroup.com/'` (trailing slash).
- **Never edit these fields on a module you are not otherwise changing** —
  odoo.sh reloads/upgrades a module because its files changed, not because
  the version string changed. A drive-by manifest fix costs a full upgrade
  across six client repos for nothing.
- Any module that creates, posts, pays, or reads invoices declares
  `'accountant'` in `depends` — not bare `'account'`.

## Data file syntax (`data.html`)

```xml
<odoo>
    <record id="foo_record_1" model="foo.model">
        <field name="name">Example</field>
        <field name="partner_id" ref="base.res_partner_1"/>
        <field name="line_ids" eval="[(0, 0, {'qty': 1})]"/>
        <field name="company_id" search="[('name', '=', 'My Company')]"/>
        <field name="active" eval="True"/>
    </record>

    <delete model="foo.model" search="[('active', '=', False)]"/>

    <function model="foo.model" name="some_method">
        <value eval="[ref('foo_record_1')]"/>
    </function>
</odoo>
```

- `<record id="..." model="...">`: `id` becomes `<module>.<id>` as the XML id
  once loaded, unless already dotted (referencing another module's record).
- `<field name="x" ref="other_module.xml_id"/>`: resolve by XML id.
- `<field name="x" search="[domain]"/>`: resolve by domain lookup instead of
  a fixed id — useful when the target isn't guaranteed to have a stable XML
  id (e.g. the demo company).
- `<field name="x" eval="python_expr"/>`: `eval` context includes `ref()`,
  `time`, `datetime`. Common for `Many2many`/`One2many` command tuples and
  booleans.
- `type="xml"/"html"` on a field: inline markup, `%(module.xml_id)d` /
  `%(module.xml_id)s` substitution for cross-references inside the markup.
- `<delete>`: remove records matching `id` or `search=` at load time.
- `<function>`: call an arbitrary model method during loading — use sparingly,
  prefer declarative `<record>` data.

## `noupdate`

```xml
<odoo noupdate="1">
    ...
</odoo>
<!-- or per-record -->
<record id="foo_1" model="foo.model" noupdate_records_only>
```

- `noupdate="1"` on the `<odoo>` (or `<data>`) wrapper: records inside load
  **only on first install**. On every subsequent `-u` (module update), this
  block is skipped entirely — the database is assumed to own these records
  now (they may have been edited by the user).
- Used for: user-editable seed data (default sequences, demo-adjacent
  reference data), anything that must not snap back to the module's shipped
  value on every upgrade.
- **Not** used for: views, security, anything that must always match the
  code — those files ship without `noupdate` (or `noupdate="0"`, the
  default) so every upgrade re-applies them.
- Editing a `noupdate="1"` record's XML and pushing it changes nothing in
  already-installed databases — this is the exact mechanism the root
  `CLAUDE.md` C4 rule warns about: touching *any* file in the module
  triggers `-u`, which reloads every **non**-`noupdate` data file and can
  overwrite hand-edited config, even though the specific line you meant to
  change was harmless.

## XML ids

- Format inside a module: bare `foo_record_1` → becomes `<module_name>.foo_record_1`.
- Cross-module reference: always fully qualified, `sale.action_orders`.
- Look up at runtime: `self.env.ref('module.xml_id')` — never a hardcoded
  numeric id (`odoo-slop-audit` B3).

## `ir.sequence`

```xml
<record id="seq_foo" model="ir.sequence">
    <field name="name">Foo Sequence</field>
    <field name="code">foo.code</field>
    <field name="prefix">FOO/%(year)s/</field>
    <field name="padding">5</field>
    <field name="company_id" eval="False"/>
</record>
```

```python
self.env['ir.sequence'].next_by_code('foo.code')
```

`company_id = False` makes the sequence global; set it to scope per-company
(each company then needs its own sequence row, or `use_date_range=True` for
per-period numbering).

## `ir.cron`

```xml
<record id="ir_cron_foo_cleanup" model="ir.cron">
    <field name="name">Foo: nightly cleanup</field>
    <field name="model_id" ref="model_foo_model"/>
    <field name="state">code</field>
    <field name="code">model._cron_cleanup()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">days</field>
    <field name="numbercall">-1</field>
    <field name="active">True</field>
</record>
```

`interval_type` is one of `minutes`/`hours`/`days`/`weeks`/`months`
(`addons/base/models/ir_cron.py:75`). `numbercall=-1` means run forever.
`nextcall` (`ir_cron.py:80`) defaults to `fields.Datetime.now` at creation —
set it explicitly if the first run must not happen immediately after
install. Crons ship `noupdate="1"` by convention so a user-adjusted schedule
survives upgrades — but then a real code fix to the cron's `code`/interval
needs a data migration or manual DB update to actually reach existing
installs.

## Module hooks

```python
# __init__.py
from . import models

def post_init(env):
    env['foo.model'].search([])._recompute_something()

def pre_init(env):
    ...

def uninstall(env):
    ...
```

- **18.0 signature is `hook(env)`** — a single `odoo.api.Environment`, not the
  old `(cr, registry)` pair. Verified at `odoo/modules/loading.py:246`:
  `getattr(py_module, post_init)(env)`.
- `pre_init_hook` runs before any data file of the module loads (useful to
  fail fast on a missing external dependency check that isn't expressible
  via `external_dependencies`).
- `post_init_hook` runs after all data files load, only on **install**, not
  on update.
- `uninstall_hook` runs before the module's tables/data are dropped — use to
  clean up non-ORM state (cron-scheduled OS resources, etc.), not needed for
  anything the ORM's own uninstall already handles.
- Per the official guidance: use hooks only when the setup/cleanup is
  otherwise impossible through data files/`@api.model` methods — they run
  outside the normal transactional data-loading flow and are easy to get
  subtly wrong (e.g. forgetting a record already exists on a re-run).

## Module directory layout

```
foo/
├── __init__.py              # imports models, wizard, controllers (not report/static)
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── foo_model.py
├── views/
│   └── foo_views.xml
├── security/
│   ├── foo_security.xml     # groups — loaded first
│   └── ir.model.access.csv
├── data/
│   └── foo_data.xml         # sequences, crons, seed records
├── wizard/                  # TransientModel + their views
├── report/                  # ir.actions.report + qweb templates
├── controllers/
│   └── main.py
├── static/
│   ├── description/icon.png
│   └── src/{js,scss,xml}/
├── i18n/
│   └── es_AR.po             # Spanish translations — never in the manifest strings
└── tests/
    ├── __init__.py
    └── test_foo.py
```

`i18n/` is never imported from `__init__.py` (translations load via the
`.po` mechanism, not Python import) and `tests/` is only imported by the
loader when running with `--test-enable`/`--test-tags`, never in production
`__init__.py`.
