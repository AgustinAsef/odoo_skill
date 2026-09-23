# Frontend / OWL — Odoo 18.0

OWL 2 component framework, native ES modules, no build step for a normal
module. This covers assets bundling, component skeletons, patching, the
registry system, services, field widgets, SCSS, and tours.

## 18.0 breaking changes / confirmations

| Claim | Status |
|---|---|
| No `/** @odoo-module */` header comment needed for files under `<module>/static/src/**` or `<module>/static/tests/**` — those directories are auto-transpiled as native ES modules. The comment is only required for JS files placed **outside** those conventional directories. | Confirmed: zero `@odoo-module` occurrences across core `web/static/src` (e.g. `web/static/src/core/registry.js`, `web/static/src/core/notifications/notification.js`) |
| `static template` / `static props` are plain class fields (Owl 2 style), not decorators. | Confirmed verbatim at `addons/web/static/src/core/notifications/notification.js:3-5` |
| `patch(target, extension)` takes the object/prototype directly — patch a Component via `MyComponent.prototype`, not the class itself. Returns an `unpatch` function. | doc-only (patching_code.html); shape consistent with core usage |
| `registry.category("<name>").add(key, value, { sequence })` is the universal registration call across every extension point (fields, services, systray, actions, ...). | Confirmed live: `webclient.js:30`, `user_menu.js:43`, `settings_form_view/fields/upgrade_boolean_field.js:41`, systray/fields registrations across `web/static/src/webclient/**` |
| Bundle naming convention: `<first-defining-module>.assets_<purpose>` (e.g. `web.assets_backend`, `web.assets_frontend`). Bundles compose via `('include', 'other.bundle')`. | Confirmed at `addons/web/__manifest__.py:27-41,45-148` |

## Assets in the manifest

```python
"assets": {
    "web.assets_backend": [
        "my_module/static/src/**/*.js",
        "my_module/static/src/**/*.xml",
        "my_module/static/src/**/*.scss",
    ],
    "web.assets_frontend": [
        "my_module/static/src/frontend/**/*",
    ],
},
```

Standard bundles to target:

| Bundle | Loads for |
|---|---|
| `web.assets_backend` | logged-in backend (views, actions, systray, dialogs) |
| `web.assets_frontend` | public website (ecommerce, portal, blog) |
| `web.assets_backend_lazy` | backend, loaded lazily after first paint |
| `web.assets_unit_tests` | JS unit test files (`static/tests/**`) |
| `web.assets_tests` | tour/integration test JS |
| `point_of_sale._assets_pos` | POS frontend (own app, not `assets_backend`) |

Directives, applied as tuples instead of a plain path string:

```python
"web.assets_backend": [
    ("include", "web._assets_helpers"),                      # pull in another bundle's files
    ("prepend", "my_module/static/src/scss/overrides.scss"),  # force load order early
    ("before", "web/static/src/core/registry.js", "my_module/static/src/patch_first.js"),
    ("after", "web/static/src/core/registry.js", "my_module/static/src/patch_after.js"),
    ("remove", "web/static/src/legacy/some_legacy_file.js"),
],
```

`'path/**/*'` globs recursively; JS/SCSS get concatenated+minified, XML
templates are read and registered into the OWL template registry at runtime
— no separate `t-call-assets` step needed for component templates declared
this way (that mechanism is for including whole *other bundles* inside a
regular page template, not for loading a component's own XML).

## Component skeleton

```
my_module/static/src/my_widget/
    my_widget.js
    my_widget.xml
    my_widget.scss
```

```javascript
import { Component, useState } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";

export class MyWidget extends Component {
    static template = "my_module.MyWidget";
    static props = {
        recordId: { type: Number },
        onDone: { type: Function, optional: true },
    };

    setup() {
        this.state = useState({ count: 0 });
        this.orm = useService("orm");
    }

    async onClickIncrement() {
        this.state.count++;
        await this.orm.write("my.model", [this.props.recordId], { count: this.state.count });
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="my_module.MyWidget">
        <div class="o_my_widget" t-on-click="onClickIncrement">
            <t t-out="state.count"/>
        </div>
    </t>
</templates>
```

Naming rule: template `t-name` = `<module>.<ComponentName>` (PascalCase),
matched exactly by `static template`. Never reuse another module's template
name — collisions overwrite silently at registration.

`setup()` replaces the constructor for anything that needs to run
initialization logic — plain class constructors on an Owl `Component` cannot
be overridden/patched by `patch()` because JS handles `constructor` specially;
this is also why `patch()` targets methods, not constructors.

## Patching existing code

```javascript
import { patch } from "@web/core/utils/patch";
import { MyWidget } from "@my_module/my_widget/my_widget";

patch(MyWidget.prototype, {
    setup() {
        super.setup();
        this.extra = true;
    },
});
```

- Patch a **class prototype** for instance methods (`MyWidget.prototype`).
- Patch the **class itself** for static members.
- Patch a **service definition object** (before it's registered, or the
  registry entry) the same way — `patch(myService, { start() { ... } })`.
- `super.method(...)` only works with real method syntax on the extension
  object — not an arrow function, not `function()`.
- Keep the returned `unpatch()` reference only if the patch is meant to be
  temporary (e.g. inside a test) — a permanent module-load-time patch can
  discard it.

## Registries — extension points

```javascript
import { registry } from "@web/core/registry";
```

| Category | Registers | Example |
|---|---|---|
| `fields` | field widgets | `registry.category("fields").add("my_widget", MyFieldWidget)` |
| `services` | app-wide services | `registry.category("services").add("myService", myService)` |
| `actions` | client actions (`ir.actions.client` tag) | `registry.category("actions").add("my_module.dashboard", Dashboard)` |
| `systray` | navbar systray icons | `registry.category("systray").add("my_module.icon", { Component: MyIcon }, { sequence: 20 })` |
| `main_components` | top-level always-mounted components | `registry.category("main_components").add("MyOverlay", { Component: MyOverlay })` |
| `user_menuitems` | entries in the user avatar dropdown | doc-only |
| `view_widgets` | non-field view-level widgets | doc-only |

`.add(key, value, { sequence, force })` — `sequence` controls display order
(lower first), `force: true` silently overwrites an existing key instead of
raising (only use when deliberately replacing a core entry).

## Field widget skeleton

```javascript
import { registry } from "@web/core/registry";
import { CharField, charField } from "@web/views/fields/char/char_field";

export class MyCharField extends CharField {
    static template = "my_module.MyCharField";
}

export const myCharField = {
    ...charField,
    component: MyCharField,
};

registry.category("fields").add("my_char", myCharField);
```

Use in a view with `<field name="x" widget="my_char"/>`. Prefer extending an
existing field's component + spec object (as above) over writing one from
scratch — inherits formatting, validation, and props handling for free.

## Services

```javascript
import { registry } from "@web/core/registry";

const myService = {
    dependencies: ["orm", "notification"],
    start(env, { orm, notification }) {
        return {
            async ping() {
                await orm.call("my.model", "ping", []);
                notification.add("pong", { type: "success" });
            },
        };
    },
};

registry.category("services").add("myService", myService);
```

Consume from a component:

```javascript
import { useService } from "@web/core/utils/hooks";

setup() {
    this.myService = useService("myService");
}
```

`start(env, deps)` return value becomes the service's public API — return an
object of methods, not the raw internal state. `dependencies` names other
services to inject (order-resolved automatically); the second `start` param
is those services keyed by name.

## SCSS

```python
"assets": {
    "web.assets_backend": [
        "my_module/static/src/scss/my_widget.scss",
    ],
},
```

```scss
.o_my_widget {
    display: flex;
    gap: $o-horizontal-padding; // reuse core SCSS variables when available
}
```

Scope every top-level selector under a module-specific class
(`o_my_widget`, not `.card` or another generic Bootstrap-looking name) to
avoid bleeding styles into unrelated views. Prefer existing Bootstrap/Odoo
SCSS variables/mixins over hardcoded colors — hardcoded hex breaks dark mode.

## Tours (brief)

```javascript
import { registry } from "@web/core/registry";

registry.category("web_tour.tours").add("my_module.my_tour", {
    url: "/odoo/my-app",
    steps: () => [
        { trigger: ".o_my_widget", content: "Click the widget", run: "click" },
    ],
});
```

Registered in the `web_tour.tours` category, loaded from
`web.assets_tests`. Run manually from the browser console
(`odoo.startTour("my_module.my_tour")`) or via the workspace Docker lab test
runner — see `docker-lab-tours-nunca-corrian` memory: a missing
`websocket-client` + too-small `/dev/shm` makes tours skip silently and
report green, use the `v18-odoo-postest` image with `--shm-size=2g`.

## Client action wiring (component-backed)

```xml
<record id="action_my_dashboard" model="ir.actions.client">
    <field name="name">My Dashboard</field>
    <field name="tag">my_module.dashboard</field>
</record>
```

```javascript
registry.category("actions").add("my_module.dashboard", MyDashboard);
```

`tag` on the `ir.actions.client` record must exactly match the registry key.
No `_get_report_values`-style Python override exists for client actions —
all data loading happens client-side via `useService("orm")` inside `setup()`.
