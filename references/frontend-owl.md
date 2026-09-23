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

## Environment, services, registries — the short version

- `env` (`this.env`) is shared state: `env.services` (fetch via `useService`, don't
  index it directly), `env.bus` (app-wide event bus), `env.debug`.
- **Services** are singletons: `{ dependencies: [...], start(env, deps) { return api } }`
  on `registry.category("services")` — see *Services* below.
- **Registries** are the extension points every subsystem (fields, views, actions,
  systray, error handlers...) reads from at boot; nothing is wired by import order.

## Hooks

Two families. Owl's own, from `@odoo/owl` (`useState`, `useRef`, `useEffect`,
`onWillStart`, `onMounted`, `onWillUnmount`, `onPatched`, `onError`...) — standard Owl 2,
not Odoo-specific. And Odoo's, all exported from `core/utils/hooks.js` (verified against
that file directly, not just the doc — the doc's hook list was incomplete):

| Hook | Signature | Does |
|---|---|---|
| `useService(name)` | `useService(serviceName)` | Returns the service's public API. **Throws** `Error("Service X is not available")` if not deployed — `core/utils/hooks.js:136` |
| `useBus(bus, event, cb)` | `useBus(bus, eventName, callback)` | Adds/removes a bus listener across mount/unmount automatically |
| `useAutofocus({refName, selectAll, mobile})` | returns a `Ref` | Focuses `t-ref="autofocus"` once it appears; skipped on touch devices unless `mobile: true` |
| `useChildRef()` / `useForwardRefToParent(refName)` | pair | Forward a DOM ref up one component level — child calls `useForwardRefToParent`, parent holds `useChildRef()` and passes it down as a prop |
| `useOwnedDialogs()` | returns `addDialog(...)` | Wraps `useService("dialog")`; auto-closes dialogs it opened when the component unmounts |
| `useSpellCheck({refName})` | — | Toggles `spellcheck` on focus/blur only, so idle fields don't show the squiggly underline |
| `useRefListener(ref, ...listenerArgs)` | — | Attaches/detaches a raw DOM listener on a ref's element for its lifetime |

`useService` and `useBus` cover most components; the rest are situational. All are thin
wrappers over `useEffect`/`onWillUnmount` — safe pattern to copy for a project hook.

## Error handling

Errors and rejected promises reach a central `error` service, which walks
`registry.category("error_handlers")` in `sequence` order until one returns truthy
(verified `core/errors/error_service.js:67`, `core/errors/error_handlers.js:21-208`):

```javascript
import { registry } from "@web/core/registry";

function myHandler(env, error, originalError) {
    if (originalError instanceof MyDomainError) {
        env.services.notification.add(originalError.message, { type: "danger" });
        return true; // handled — no error dialog shown
    }
}
registry.category("error_handlers").add("myHandler", myHandler, { sequence: 50 });
```

Core handlers in registration order: `rpcErrorHandler` (97) → `lostConnectionHandler` (98)
→ `requestEntityTooLargeHandler` (99) → `defaultHandler` (100, generic dialog). Lower
`sequence` runs first — register a domain handler below 97 to intercept before RPC does.
A server `UserError`/`ValidationError` arrives client-side as an `RPCError`
(`core/network/rpc.js:18`); `rpcErrorHandler` maps it to `RPCErrorDialog` through the
separate `error_dialogs` registry (`core/errors/error_dialogs.js:228`). Inside a
component, Owl's own `onError(error)` is an error boundary that can catch a render
error before it ever reaches the error service.

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

Read the current value with `this.props.record.data[this.props.name]`; write it with
`this.props.record.update({ [this.props.name]: value })`. `standardFieldProps` carries
only `id`, `name`, `readonly`, `record` (`views/fields/standard_field_props.js`) — there
is **no** `props.update()`. A field spec object also typically carries `displayName`,
`supportedTypes`, `supportedOptions` (drives the option panel in the view editor), and
`extractProps({attrs, options})` mapping arch attributes to props — verified shape at
`views/fields/char/char_field.js:106-130`.

## Custom view type

Four pieces, verified against `views/list/list_view.js:7-33`:

```javascript
import { registry } from "@web/core/registry";
import { XMLParser } from "@web/core/utils/xml";
import { RelationalModel } from "@web/model/relational_model/relational_model";

class MyArchParser extends XMLParser {
    parse(xmlDoc) { return { /* whatever Controller/Renderer need out of the arch */ }; }
}
class MyController extends Component {
    static template = "my_module.MyController";
    static props = ["*"];
    setup() {
        this.model = useState(new this.props.Model(this.env, this.props.archInfo, this.props.state));
    }
}
class MyRenderer extends Component {
    static template = "my_module.MyRenderer";
    static props = ["*"];
}

export const myView = {
    type: "my_view",
    Controller: MyController,
    ArchParser: MyArchParser,
    Renderer: MyRenderer,
    Model: RelationalModel, // reuse unless the data genuinely isn't records
};
registry.category("views").add("my_view", myView);
```

Cheaper path: extend an existing view instead of building one — subclass its
`Controller` (and `Renderer` if needed), register the subclass under a new key, and
point the arch at it with `js_class="my_kanban"`. Reuse `RelationalModel`
(`model/relational_model/relational_model.js`) — a from-scratch `Model` means
reimplementing paging, grouping, and dirty-tracking yourself.

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

### SCSS variable inheritance

Bootstrap/Odoo SCSS variables use `!default`, so the **first** assignment wins and
every later `!default` for the same name is ignored. Backend load order, first to
last: `web.dark_mode_variables` → `web._assets_primary_variables` →
`web._assets_secondary_variables` → `web._assets_bootstrap` → `web.assets_backend`
(verified `web/__manifest__.py:378-404`).

To override a Bootstrap or core variable, never edit core SCSS — add your own file to
the `web._assets_primary_variables` (or `_secondary_variables`) bundle key in **your**
module's manifest. Verified live pattern, `account/__manifest__.py:96-99`:

```python
"assets": {
    "web._assets_primary_variables": [
        "my_module/static/src/scss/variables.scss",
    ],
},
```

```scss
// my_module/static/src/scss/variables.scss
$o-my-brand-color: #3c3c3c !default;
$border-radius: 8px !default; // overrides Bootstrap's default cleanly
```

`web/static/src/**/*.variables.scss` is also auto-included into
`_assets_primary_variables` (`web/__manifest__.py:380`), but only for files physically
under the `web` module — a third-party module must declare the bundle key itself, as
above.

### SCSS tips

- Prefer semantic HTML tags (`<h5>`, not a styled `<span>`) — a heading style change
  then propagates automatically everywhere.
- Check for an existing Bootstrap/Odoo utility class (`d-flex`, `px-3`,
  `position-relative`...) before writing a new declaration.
- Static classes in `class="..."`; state-dependent classes in `t-att-class="{...}"` —
  don't mix both into one dynamic string.
- A class that "looks right" visually can still be semantically wrong (e.g. a button
  class on a title) and can carry unwanted JS behavior — verify its real purpose.

## Standalone OWL application

For an app that is *not* the backend/website/POS — its own page, own bundle, own
route. Verified against `howtos/standalone_owl_application.html`:

```javascript
// app.js
import { whenReady } from "@odoo/owl";
import { mountComponent } from "@web/env";
import { Root } from "./root";

whenReady(() => mountComponent(Root, document.body));
```

`mountComponent` (`@web/env`) builds the same Owl `env` (services, translations) the
main web client builds, so `useService` works inside the standalone app too. Give it
its own manifest bundle (`"my_module.assets_standalone_app"`, including
`web._assets_helpers`, `web._assets_bootstrap`, `web._assets_core`, then your
`static/src/standalone_app/**/*`), and serve it from an HTTP controller rendering a
QWeb page that sets the `odoo` global (`csrf_token`, `debug`, `__session_info__`) and
pulls the bundle with `t-call-assets`. Never add the standalone bundle's files to
`web.assets_backend`/`_frontend` too — double-loading the same components under two
different envs breaks service singletons.

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
