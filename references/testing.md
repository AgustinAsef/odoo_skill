# Testing — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/odoo/tests/common.py`,
`odoo/tests/form.py`, `odoo/tests/__init__.py`, `odoo/tools/config.py`.

## Test case classes

| Class | Transaction model | Use for |
|---|---|---|
| `TransactionCase` (`common.py:931`) | one shared transaction from `setUpClass`, each test method in its own **savepoint** (rolled back after the method, class-level data persists across methods) | the default — almost everything |
| `SingleTransactionCase` (`common.py:1165`) | the whole class shares one transaction, methods run in declared order, nothing rolled back between methods | tests that build up state step by step across methods (rare — order-dependent tests are fragile, prefer `TransactionCase` + `setUpClass`) |
| `HttpCase` (`common.py:2169`, extends `TransactionCase`) | same as `TransactionCase`, plus a running HTTP server and a browser/tour test harness | controllers, JS-driven `web_tour` tests, portal routes |

Do not mock the ORM or patch your own module's functions to test it
(`odoo-slop-audit` A5) — `TransactionCase` gives a real registry; patch only
genuine external I/O (HTTP calls, AFIP, a carrier API), and patch the
transport layer, not your own wrapper around it.

## `setUpClass` idiom

```python
from odoo.tests import TransactionCase, tagged

@tagged('post_install', '-at_install')
class TestFoo(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Test Partner'})
        cls.product = cls.env.ref('product.product_product_4')

    def test_something(self):
        record = self.env['foo.model'].create({'partner_id': self.partner.id})
        self.assertEqual(record.state, 'draft')
```

- Always call `super().setUpClass()` first.
- Put every fixture that's read-only across test methods in `setUpClass`
  (class attribute, `cls.xxx`) — it runs once per class, not once per test.
  Per-test mutable state still goes in `setUp`/inline in the test method.
- Never hardcode a numeric or database-specific id in a fixture —
  `self.env.ref('module.xml_id')`, not `self.browse(37)`
  (`odoo-slop-audit` B3). This applies doubly to test fixtures: a company or
  journal id baked into a test hides real bugs the moment the suite runs
  against a different database.
- `self.assertRaises(SomeError)` **reverts the savepoint** it wraps on
  failure — any state your test code wrote before raising is discarded, not
  just the raise itself. If the test needs to assert on state left behind by
  a failed operation, use `try/except` instead of `assertRaises`.

## `tagged()` and standard tags

```python
@tagged('post_install', '-at_install')
class TestFoo(TransactionCase):
    ...
```

`common.py:2627-2649`. Every `BaseCase` subclass defaults to
`{'standard', 'at_install'}`. A tag prefixed `-` removes it. Odoo warns if a
class ends up with **both** or **neither** of `at_install`/`post_install` —
pick exactly one:

| Tag | Runs |
|---|---|
| `at_install` (default) | right after the module's own data loads, before other modules are installed — no cross-module dependency guaranteed yet |
| `post_install` | after **all** modules in the current `-i`/`-u` finish installing — use whenever the test needs another module's data/views (e.g. `accountant`, `sale`) |
| `standard` (default) | included in the default `--test-tags` run |
| a custom tag | filter subset, e.g. `@tagged('foo_module')` then `--test-tags foo_module` |

## `Form`

```python
from odoo.tests import Form

with Form(self.env['sale.order']) as f:
    f.partner_id = self.partner
    with f.order_line.new() as line:
        line.product_id = self.product
        line.product_uom_qty = 3
    order = f.save()
```

`odoo/tests/form.py:27`, exported via `odoo/tests/__init__.py:10`
(`from .form import Form, O2MProxy, M2MProxy`). `Form` simulates the web
client's onchange/readonly/required resolution — it is the only way to
exercise `@api.onchange` logic in a test, since a plain `create()`/`write()`
never triggers onchanges.

**Gotcha**: `Form` does not fully emulate what the real web client does with
`readonly` fields — the client omits readonly fields from the payload it
sends, but `Form` does not reproduce that omission faithfully in every case.
A bug that only manifests because the client never sends a readonly field
needs an `HttpCase` tour, not a `Form` test (workspace-verified: see root
memory `odoo-test-form-no-cubre-readonly`).

## Running tests with `odoo-bin`

```bash
odoo-bin -c /etc/odoo/odoo.conf -d mydb -i foo_module --test-tags foo_module --stop-after-init
odoo-bin -c /etc/odoo/odoo.conf -d mydb -u foo_module --test-tags /foo_module --stop-after-init
odoo-bin -c /etc/odoo/odoo.conf -d mydb -i foo_module --test-tags :TestFoo.test_something --stop-after-init
```

| Flag | Meaning |
|---|---|
| `-i module` | install (fresh) and run its tests as part of install |
| `-u module` | update an already-installed module and run its tests as part of the update |
| `-d db` | target database |
| `--test-tags` | filter which tests run — `module_name`, `/module_name` (path form), `:ClassName.method_name`, `+tag`/`-tag`. `tools/config.py:176-189` |
| `--test-enable` | legacy flag — if `--test-tags` isn't already set, it defaults `test_tags` to `+standard` (`config.py:687-689`). **In 18.0, passing `--test-tags` alone already enables tests** (`config.py:582`: `test_enable = bool(test_tags)`) — `--test-enable` is rarely needed if you're already filtering with `--test-tags` |
| `--stop-after-init` | exit after the install/update/test run instead of starting the HTTP server — always use this for a CI-style test run |
| `--log-level=test` | verbose test output |

Practical default for "run this module's tests and nothing else, then exit":

```bash
odoo-bin -c /etc/odoo/odoo.conf -d mydb -u foo_module --test-tags foo_module --stop-after-init --log-level=test
```

## Docker lab gotcha (workspace-specific)

Tours (`HttpCase` + `web_tour`) silently no-op and report green if
`websocket-client` isn't installed in the container, or if `/dev/shm` is too
small for the headless browser. Use the `v18-odoo-postest` image/profile with
`--shm-size=2g`, and check the test log for the tour actually starting, not
just for a passing exit code (root memory
`docker-lab-tours-nunca-corrian`). A regression test is not proven until you
have seen it fail once — count `skipped`, not just `failed`, in the test
summary (root memory `feedback-verificar-tests-con-mutacion`).
