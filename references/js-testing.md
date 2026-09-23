# JS Testing — Odoo 18.0

**Hoot is the JS unit test framework in 18.0.** QUnit still exists in the source tree
but only for legacy suites explicitly excluded from the new bundle — treat any new
test as a Hoot test. Verified against `addons/web/static/tests/`,
`addons/web/static/lib/hoot/`, `addons/web/tests/test_js.py`, and `web/__manifest__.py`.

## Hoot vs QUnit — what's actually true in 18.0

- `web.assets_unit_tests` (`web/__manifest__.py:468-472`) = `web/static/tests/**/*`
  **minus** `('remove', 'web/static/tests/tours/**/*')` and
  `('remove', 'web/static/tests/legacy/**/*')` — the manifest comment reads
  `# to remove when all legacy tests are ported`. Legacy is on its way out, not a
  parallel-forever option.
- `web/static/lib/qunit/` and `web/static/tests/legacy/qunit.js` still ship, and a
  separate bundle (`web.tests_assets`) still loads QUnit CSS/JS for the legacy suite —
  but `web/tests/test_js.py:20-22` labels its own QUnit error checker `# ! DEPRECATED`
  in a first-party comment. Don't port a test to QUnit; port it away from QUnit.
- `mail/static/tests/legacy/qunit_suite_tests` is the only other QUnit holdout found
  in core — another sign it's a sunset path, not a pattern to imitate.

## Where tests live and how they're bundled

```
my_module/static/tests/
    my_widget.test.js          # unit test, Hoot picks up "*.test.js" automatically
    some_setup.hoot.js          # global setup for the whole test run, not a suite
```

```python
"assets": {
    "web.assets_unit_tests": [
        "my_module/static/tests/**/*",
    ],
},
```

`web.assets_unit_tests_setup` (`web/__manifest__.py:440-462`) is what actually loads
Owl, the module loader, and `('include', 'web.assets_backend')` +
`('include', 'web.assets_backend_lazy')` before any test file runs — a unit test has
the whole backend environment available, not a stripped-down sandbox.

## Writing a test

Real example, verified verbatim at
`addons/web/static/tests/core/orm_service.test.js:1-34`:

```javascript
import { describe, expect, test } from "@odoo/hoot";
import { getService, makeMockEnv, onRpc } from "@web/../tests/web_test_helpers";

describe.current.tags("headless");

test("add user context to a simple read request", async () => {
    onRpc(async (params) => {
        expect.step(params.route);
        expect(params).toMatchObject({
            args: [[3], ["id", "descr"]],
            kwargs: { context: { uid: 7 /* ... */ } },
            method: "read",
            model: "res.partner",
        });
        return false; // don't call the real read logic
    });

    const { services } = await makeMockEnv();
    await services.orm.read("res.partner", [3], ["id", "descr"]);

    expect.verifySteps(["/web/dataset/call_kw/res.partner/read"]);
});
```

- `describe`, `test`, `expect` come from `@odoo/hoot` (the test framework, unit-test
  only). `@odoo/hoot-dom` is the DOM query/interaction layer, and it's the one usable
  inside tours too (`click`, `queryAll`, `queryText`, `waitFor`...).
- `describe.current.tags("headless")` opts a suite into the headless runner (no visible
  browser chrome needed) — check an existing test in the same directory before adding a
  tag, several exist (`desktop`, `mobile`, `headless`) and they gate what `preset=` the
  Python runner uses.
- `expect.step(...)` / `expect.verifySteps([...])` is the idiomatic way to assert a
  sequence of RPC calls happened, not a list of `assert callCount == N`.

## Web test helpers (`@web/../tests/web_test_helpers`)

All verified against `addons/web/static/tests/web_test_helpers.js` and
`_framework/component_test_helpers.js` / `_framework/mock_server/mock_server.js`:

| Helper | Signature | Does |
|---|---|---|
| `makeMockEnv(partialEnv?, options?)` | async | Boots the mock env + services with no component mounted — for testing a service directly |
| `mountWithCleanup(ComponentClass, options?)` | async | Mounts a component with a mock env, auto-unmounts after the test. `options.props`, `options.target` |
| `defineModels(ModelClasses, options?)` | — | Registers mock ORM model classes on the current `MockServer` |
| `defineWebModels()` / `webModels` | — | Pulls in the stock `res.partner`/`res.users`/etc. mock models so you don't redefine them |
| `onRpc(routeOrMethodOrModel, method?, callback?)` | — | Intercepts an RPC; 4 overloads — bare callback (every call), route string, method name, or `[model, method]` |
| `getService(name)` | — | Fetches a service from the current mock env outside a component |
| `patchWithCleanup(obj, patch)` | — | Same contract as `@web/core/utils/patch`'s `patch()`, but auto-reverts after the test |
| `contains(selector)` / `click(target)` / `edit(target, value)` | — | DOM assertion + interaction helpers scoped to the test fixture |
| `serverState` | object | Shared mutable test server state (`serverState.debug`, current user, companies...) |

Mounting a real backend surface (a view, the `WebClient`) pulls in the full env — most
tests only need `makeMockEnv()` plus `onRpc`, reach for `mountWithCleanup` when the
assertion is actually about rendering.

## Defining mock models

Verified pattern, `addons/web/static/tests/core/debug/debug_manager.test.js:495-527`:

```javascript
import { fields, models, defineModels } from "@web/../tests/web_test_helpers";

class Custom extends models.Model {
    _name = "custom";
    name = fields.Char();
    raw = fields.Binary();
    _records = [{ id: 1, name: "custom1", raw: "<raw>" }];
}

defineModels([Custom]);
```

`fields.Char()`, `fields.Many2one()`, `fields.Properties()` etc. mirror the server-side
field API closely enough that `_records` can be written like real ORM data. This
replaces the server for the test — no real database round-trip, no fixture cleanup
needed between tests (each test gets its own `MockServer` instance).

## Running tests

- **Browser, interactively**: `/web/tests` (desktop) or Debug menu → *Run Unit Tests*.
  The legacy QUnit suite is served separately, at `/web/tests/legacy?mod=<module>`.
- **From Python** (what CI actually runs): `WebSuite.test_unit_desktop` in
  `addons/web/tests/test_js.py:189-191` calls
  `self.browser_js('/web/tests?headless&loglevel=2&preset=desktop...', success_signal="[HOOT] Test suite succeeded")`
  — i.e. Hoot runs *inside a real headless browser* driven by `HttpCase.browser_js`,
  not in Node. A `--test-tags` filter like `@web/core/autocomplete` gets hashed
  (`_generate_hash`, `test_js.py:110-126`) into `&id=<hash>` query params before hitting
  that URL — the human-readable tag and the URL param are not the same string.
- No filter and no visible browser at all is the default (`headless`); dropping
  `headless` or passing `debug=True`-style `browser_js` kwargs opens one for local
  debugging, but that's a manual/local-only path, not what CI does.

## Tours — end-to-end, not unit

A tour (`registry.category("web_tour.tours")`, see `references/frontend-owl.md`) is
browser automation, not a Hoot test. Its Python side is
`HttpCase.start_tour(url_path, tour_name, **kwargs)`
(`odoo/tests/common.py:2479-2497`), a thin wrapper over `browser_js` that runs
`odoo.startTour(tour_name, options)` and waits for the `"tour succeeded"` signal:

```python
from odoo.tests import HttpCase, tagged

@tagged("-at_install", "post_install")
class TestMyTour(HttpCase):
    def test_my_tour(self):
        self.start_tour("/odoo/my-app", "my_module.my_tour", login="admin")
```

**A tour that can't start a browser skips silently and the test suite still reports
green.** Confirmed in this workspace's Docker lab: a missing `websocket-client` Python
package plus a too-small `/dev/shm` (the default 64MB) makes Chrome headless fail to
launch, `browser_js` swallows it as a skip rather than a failure, and nothing in the
console output flags it as a problem. When validating that a tour actually ran:

- Use the `v18-odoo-postest` image with `--shm-size=2g`, not the plain dev image.
- Read the test *run* summary for a skipped count, not just failures — `0 failed` is
  not the same claim as `N passed`.
- If in doubt, deliberately break a `trigger` selector and confirm the tour then
  reports a real failure — a tour that can never fail was never actually running.
