# Website, Themes, Portal Frontend — Odoo 18.0

Public website (`website`), Website Builder theming, and the customer/vendor
portal (`portal`). Companion to `frontend-owl.md` (OWL/JS mechanics) and
`views-actions-menus.md` (view XML) — read those first; this file does not
repeat the OWL component skeleton or the `<list>`/`attrs` breaking changes.

## Decision: theme module vs regular module with website pages

| Need | Build |
|---|---|
| A handful of pages/snippets for one client's own site | A **regular module** with `website.page`/`website.menu` data records and `web.assets_frontend` assets. No `category: Theme`. |
| A reusable, switchable "look" picked from Website > Configuration > Themes, with its own SCSS palette/presets | A **theme module**: `'category': 'Theme'` (or `'Theme/<Vertical>'`), ships `primary_variables.scss`; data (pages, menus, views, snippets, assets, attachments) goes through `theme.*` shadow models, not the live ones. |

A theme module is heavier: every page/menu/view/asset/attachment it ships is
declared against a **shadow model** (`theme.ir.ui.view`, `theme.website.page`,
`theme.website.menu`, `theme.ir.asset`, `theme.ir.attachment`) instead of the
real one. On install, `theme.utils._post_copy` copies each shadow record into
the real model (`ir.ui.view`, `website.page`, `website.menu`, `ir.asset`,
`ir.attachment`) **per website**, stamping `theme_template_id` on the copy —
`Versiones/V18/odoo-18.0/addons/website/models/theme_models.py:14-212,356-405`.
That's what lets one theme install on several websites in the same database,
each getting its own editable copy; a regular module's `website.page`/
`ir.ui.view` records are shared as-is (no per-website copy). Don't reach for
a theme module to ship 3 pages for one client — the indirection has no payoff
when there is only ever one website.

## Theme module skeleton and manifest keys

```python
# __manifest__.py
{
    'name': 'Theme Example',
    'category': 'Theme/Creative',   # 'Theme' alone also valid; subcategory is cosmetic Apps-list grouping
    'version': '18.0.1.0.0',
    'depends': ['website'],
    'images': ['static/description/cover.png'],   # shown in the theme picker
    'assets': {
        'web._assets_primary_variables': ['theme_example/static/src/scss/primary_variables.scss'],
        'web.assets_frontend': ['theme_example/static/src/scss/theme.scss', 'theme_example/static/src/js/*.js'],
    },
}
```

Layout: `data/{pages,menu,shapes,gradients}.xml` (shadow-model + colorpicker
records), `views/layout.xml` (`theme.ir.ui.view` inheriting `website.layout`),
`views/snippets/{snippets.xml,s_my_block.xml}` (registration + markup),
`static/src/scss/primary_variables.scss` + `theme.scss`, `static/src/js/`.

`web._assets_primary_variables` confirmed at
`Versiones/V18/odoo-18.0/addons/website/__manifest__.py:238-239` (loads
`website/static/src/scss/primary_variables.scss` — a theme overrides/extends
this, it does not replace core's). Gradient/color-picker registration
(`web_editor.colorpicker`) is a plain `ir.ui.view` inheritance, not a
`theme.*` model — confirmed live pattern, see Gradients below.

## Theme shadow-model data (pages, menus, views, snippets as data)

Regular module — declare the real models directly:

```xml
<record id="page_about_us" model="website.page">
    <field name="name">About us</field>
    <field name="url">/about-us</field>
    <field name="website_id" eval="1"/>
    <field name="is_published" eval="True"/>
    <field name="type">qweb</field>
    <field name="key">my_module.page_about_us</field>
    <field name="arch" type="xml">
        <t t-name="my_module.page_about_us">
            <t t-call="website.layout"><div id="wrap" class="oe_structure"/></t>
        </t>
    </field>
</record>

<record id="menu_about_us" model="website.menu">
    <field name="name">About us</field>
    <field name="url">/about-us</field>
    <field name="parent_id" search="[('url','=','/default-main-menu'),('website_id','=',1)]"/>
    <field name="website_id">1</field>
    <field name="sequence" type="int">10</field>
</record>
```

Theme module — same fields, `theme.` prefix on the model, no `website_id`
(the copy-per-website step assigns it): `model="theme.website.page"` /
`model="theme.website.menu"`.

`ThemeMenu`/`ThemePage` fields and their `_convert_to_base_model` mapping:
`theme_models.py:136-215`. `website.page`/`website.menu` each carry a
`theme_template_id = fields.Many2one('theme.website.page'|'theme.website.menu', copy=False)`
on the real model (`theme_models.py:396-405`) so the copy traces back.

Wrap non-theme, single-website data in `<odoo noupdate="1">` (or
`<data noupdate="1">`) so a module update never clobbers a page the user
edited through the Website Builder — that editor writes directly to these
records, and a plain data-file reload on `-u` would overwrite those edits.

## Layout, header, footer inheritance

Same `ir.ui.view` xpath/shorthand mechanics as `views-actions-menus.md`,
targeting `website.layout` (regular module) or `theme.ir.ui.view` with
`inherit_id` set to the `website.layout` xmlid, as a `Reference` field
(`theme_models.py:71`, selection `[('ir.ui.view', ...), ('theme.ir.ui.view', ...)]`
— a theme view can inherit either a core view or another theme view).

```xml
<template id="layout" inherit_id="website.layout" name="Custom Layout">
    <xpath expr="//header" position="replace">
        <header>...</header>
    </xpath>
    <xpath expr="//footer" position="inside">
        <div class="copyright">
            <span t-esc="datetime.now().year"/>
        </div>
    </xpath>
</template>
```

`website.layout` structure: `<header/>`, `<main>` wrapping `<div id="wrap">`,
`<footer/>` (doc-only, `layout.html`). Selectors seen in core: `//header`,
`//*[@id="wrap"]`, `//*[hasclass('breadcrumb')]`. Header/footer presets
(`website.template_header_*`/`_footer_*`) are enumerated as class
attributes `_header_templates`/`_footer_templates` on `theme.utils`
(`theme_models.py:217-246`) — the builder's picker reads that list, so a
theme adds a preset via `theme.utils` inheritance, not by editing
`website.layout` directly. Front-end custom fields: a normal field
(`fields.Char`/`Html`), read with `t-field` (not `t-esc`/`t-out`) to stay
editable in-place through the builder.

## Custom snippets / building blocks

Registration: a `<t t-snippet="module.xmlid" string="Label" group="group_name">`
inside a template that inherits `website.snippets` — confirmed at
`Versiones/V18/odoo-18.0/addons/website/views/snippets/snippets.xml:422`
(`<template id="external_snippets" inherit_id="website.snippets" priority="8">`)
with dozens of `t-snippet` entries (e.g. lines 45-136, groups `intro`,
`columns`, `content`, ...).

```xml
<template id="snippets" inherit_id="website.snippets" name="Theme Snippets">
    <xpath expr="//div[@id='snippet_structure']" position="inside">
        <t t-snippet="theme_example.s_my_block" string="My Block" group="content">
            <keywords>section, block</keywords>
        </t>
    </xpath>
</template>

<!-- views/snippets/s_my_block.xml -->
<template id="s_my_block" name="My Block">
    <section class="s_my_block" data-name="My Block" data-snippet="s_my_block">
        <div class="container">...</div>
    </section>
</template>
```

Behavior/editor options go in
`static/src/snippets/s_my_block/{000.js,000.scss,000.xml,options.js}` —
doc-only (`building_blocks.html`), mirrors the real `s_*/000.js` layout in
`website/static/src/snippets/s_website_form/000.js`. Never nest a
`<section>` snippet inside another — the builder shows duplicate option
panels; use an inner `div`, not `section`, for inner-content blocks.

## SCSS variables and palettes

Theme option variables live in `static/src/scss/primary_variables.scss`,
loaded through `web._assets_primary_variables`
(`website/__manifest__.py:238-239`) — compiled *before* the main frontend
bundle, so `$primary`/`$secondary`/`$success`/... and Bootstrap overrides
(`$font-size-base`, `$h1-font-size`...`$h6-font-size`) are available
everywhere else. `presets` (named color/font combos switchable from the
design panel) are SCSS maps in the same file — doc-only, verify the mixin
against `website/static/src/scss/options/user_values.scss` (referenced
live at `theme_models.py:261`, `web_editor.assets.make_scss_customization`)
before copying one verbatim.

Custom background shapes: SVG under `static/shapes/<category>/<file>.svg`
using only the 5 default palette colors (`'1'`..`'5'`), registered via a
`theme.ir.attachment` data record plus `change-shape-colors-mapping()`/
`add-extra-shape-colors-mapping()` in `primary_variables.scss` — doc-only
(`shapes.html`).

Custom gradients for the color picker: extend `web_editor.colorpicker` (a
plain `ir.ui.view`, not a `theme.*` model) and append to the `gradients`
t-set via xpath — doc-only (`gradients.html`):

```xml
<template id="gradients" inherit_id="web_editor.colorpicker">
    <xpath expr="//div[@data-name='predefined_gradients']/t[@t-set='gradients']" position="after">
        <t t-set="gradients" t-value="gradients + ['linear-gradient(135deg, rgb(203,94,238) 0%, rgb(75,225,236) 100%)']"/>
    </xpath>
</template>
```

Scope any bespoke selector under a module-specific class, same rule as
`frontend-owl.md`'s SCSS section — a theme's generic `.hero` or `.card`
bleeds into every other installed module's markup.

## Animations, media, forms (doc-only — verify classes against a live build before shipping)

- Appearance/scroll: `o_animate`, `o_anim_fade_in` (+ direction modifiers
  `o_anim_from_bottom`/`_left`/`_right`), `o_animate_on_scroll`,
  `data-scroll-zone-start`/`-end`, CSS var `--wanim-intensity`.
- Hover (images only): `o_animate_on_hover` + `data-hover-effect*` —
  Website Builder must process it once to generate the effect's SVG before
  it renders on a page loaded without the editor.
- Images: `ir.attachment` with `res_model='ir.ui.view'`, referenced as
  `/web/image/<module>.<xmlid>`; logo goes on `website.default_website` as
  base64. Keep under ~200KB/1500px — past 1920px the builder recompresses
  aggressively. Icons: Font Awesome v4 (`fa fa-*`, sized `fa-2x`...`fa-5x`).
- Forms: `<form action="/website/form/" data-model_name="<model>" data-success-mode="redirect|message" data-success-page="...">`
  posts through the generic `/website/form/` controller keyed by
  `data-model_name` (`mail.mail`, `crm.lead`, `hr.applicant`, `res.partner`,
  `helpdesk.ticket`, `project.task`, ...) — verify the target model is
  actually form-submittable (`website_form_ok`-enabled by the module that
  owns it) before wiring a custom form to it.

## publicWidget — interactive JS on the website/portal in 18.0

18.0 uses `publicWidget.registry.<Name> = publicWidget.Widget.extend({...})`,
not a newer "Interactions" system — confirmed live at
`Versiones/V18/odoo-18.0/addons/website/static/src/js/show_password.js:12`
and `.../snippets/s_website_form/000.js:27,63`. The 18.0
`frontend_owl_components.html` docs make no mention of an "Interactions"
system — treat it as **not part of 18.0**; don't port a newer version's
Interactions patterns into an 18.0 module.

```javascript
import publicWidget from "@web/legacy/js/public/public_widget";

publicWidget.registry.MyBlock = publicWidget.Widget.extend({
    selector: ".s_my_block",
    events: { "click .btn": "_onClick" },
    start() {
        return this._super(...arguments);
    },
    _onClick(ev) { /* ... */ },
});
```

`selector` is what auto-attaches the widget to every matching element on
page load/edit-mode toggle — no manual mount call needed, unlike an OWL
component. Load it in `web.assets_frontend`.

## OWL components on the website/portal

Use `public_components`, not `actions`/`fields` — confirmed at
`Versiones/V18/odoo-18.0/addons/web/static/src/public/public_component_service.js:20`
(`ComponentManager.mountComponents` iterates
`registry.category("public_components").getEntries()` and mounts one OWL
`App` per `<owl-component name="...">` found in the DOM). Real registration:
`auth_password_policy_signup/static/src/public/components/password_meter/password_meter.js:35`.

```javascript
import { registry } from "@web/core/registry";
import { Component } from "@odoo/owl";

export class PriceEstimator extends Component {
    static template = "my_module.PriceEstimator";
    static props = { productId: { type: Number } };
}

registry.category("public_components").add("my_module.price_estimator", PriceEstimator);
```

```xml
<owl-component name="my_module.price_estimator" props='{"productId": 42}'/>
```

`props` is a JSON string attribute, parsed once at mount
(`public_component_service.js:25`). No Python-side data loader, unlike a
backend action — everything is fetched client-side via `useService("orm")`
after mount. All HTML around `<owl-component>` is server-rendered QWeb; only
that subtree is OWL, so a component that resizes after mount causes layout
shift — reserve fixed space or place it below the fold. Prefer `publicWidget`
for anything that only attaches behavior to server-rendered markup (a
toggle, a scroll listener); reach for an OWL mount only when the block needs
real client-side state (a live estimator, a stepper) — that subtree's SEO
indexing isn't guaranteed, so never put crawlable content (price,
description) inside it.

## Website controllers, `website=True`, sitemap

`website=True` on `@http.route` pulls in the website request context
(current `website`, active languages, `website.layout` availability, the
editor toolbar when logged in as an editor) — every route that renders a
public/portal page, or is called from one via RPC, needs it. Confirmed at
`Versiones/V18/odoo-18.0/addons/website/controllers/main.py`, e.g.:

```python
@http.route('/', auth="public", website=True, sitemap=True)
```

`sitemap` (confirmed across `controllers/main.py:86-966`) accepts: `True` —
included in `/sitemap.xml` at default priority; `False` — excluded
(technical/JSON endpoints, `/robots.txt` itself, `/website/save_xml`); or a
callable (`sitemap=sitemap_website_info`, line 339) for conditional
inclusion, checked at sitemap-generation time, not per request.

`auth="public"` is what actually allows anonymous access; `website=True`
alone does not relax auth. A portal route (logged-in customer, not
anonymous) uses `auth="user"`/`"portal"` with `website=True` still set, for
the layout/language context — see `portal.portal_layout` at
`Versiones/V18/odoo-18.0/addons/portal/views/portal_templates.xml:162`, the
template every `my.*` portal page calls via `<t t-call="portal.portal_layout">`
(line 226 for the portal home dashboard).

Publish/unpublish on a model shown on the frontend: inherit
`website.published.mixin` (`WebsitePublishedMixin`,
`Versiones/V18/odoo-18.0/addons/website/models/mixins.py:178`) instead of
hand-rolling an `is_published` boolean — it supplies `website_published`
(the field the "Publish" toggle binds to) and an ACL-driven `can_publish`
compute, so a user without the underlying write right can't publish just
because the frontend button is visible.

## Translations of website content

Two separate mechanisms — don't conflate them:

1. **Frontend content** (page text, snippet copy): edited in-place through
   the Website Builder's "Translate" mode. Odoo materializes a **separate
   `ir.ui.view`/`website.page` record per language** — editing the
   *source*-language version after translations exist breaks the link to
   those copies, which don't auto-update. Never "fix a typo" on an English
   page with an existing Spanish translation without checking that too.
2. **Backend/technical strings** (`string=`, `help=`, Python messages): the
   normal `.po` pipeline — `i18n/<module>.pot` plus `i18n/es_AR.po` per this
   workspace's convention (unaccented, never declared in the manifest —
   `C1` in `odoo-slop-audit/SKILL.md`). QWeb text nodes are translatable
   automatically; `t-attf-` (not `t-att-`) is required for an *attribute*
   value to be translatable (doc-only, `translations.html`).

A client's Spanish website page is data the client wrote through the
builder, not a first-party module's `string=` attribute — it is the one
legitimate place non-English source content lives; the module's own
XML/Python still ships English throughout.

## Going-live checklist

- **Import path differs by hosting.** SaaS: zip, dev mode, install
  `base_import_module`, Apps > Import Module, tick **Force init** only on
  the *first* import — a re-import with it still ticked can wipe data. A
  zip import never loads Python model code the way an addons-path install
  does (matches this workspace's `odoo-importar-modulo-vs-addons-path`
  finding) — fine for pure-XML/theme content only. odoo.sh: push to the
  repo, Apps > Update Apps List, install — no zip, no Force-init trap.
  50 MB cap on the SaaS import zip.
- Before publishing: SEO metadata per page (`additional_title`,
  `meta_description`), old-URL redirects mapped, domain/DNS correct.
- `noupdate="1"` on page/menu data files — otherwise the next module
  upgrade resets Website-Builder-edited content back to the shipped XML.
