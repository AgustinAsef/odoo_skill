# HTTP Controllers — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/odoo/http.py`, and real controllers
under `Versiones/V18/odoo-18.0/addons/{payment,payment_stripe,website_sale,web}/`.

## `@route()` / `@http.route()` decorator

```python
from odoo import http
from odoo.http import request

class MyController(http.Controller):
    @http.route('/my/path', type='http', auth='user', methods=['GET'], csrf=True)
    def my_handler(self, **kw):
        return request.render('my_module.my_template', {})
```

Must re-decorate every overridden method (`odoo/http.py:687-734` docstring
warning); omitted params inherit from the parent's `@route`.

| Param | Type | Meaning |
|---|---|---|
| `route` | `str` or iterable of `str` | werkzeug URL pattern(s). Multiple methods can share one route (different `methods`) |
| `type` | `'http'` \| `'json'` | where params live and how the response is serialized — see below |
| `auth` | `'user'` \| `'bearer'` \| `'public'` \| `'none'` | see table below. `odoo/http.py:705-721` |
| `methods` | iterable of `str` | HTTP verbs; omit = all verbs allowed |
| `cors` | `str` | value for `Access-Control-Allow-Origin` |
| `csrf` | `bool` | default `True` for `type='http'`, default `False` for `type='json'` (`http.py:725-727`) |
| `readonly` | `bool` or `Callable[[registry, request], bool]` | route a read-only replica cursor instead of primary read/write (`http.py:728-730`) |
| `handle_params_access_error` | `Callable[[Exception], Response]` | custom handler when a URL-param-derived record lookup raises `AccessError`/`MissingError` |
| `save_session` | `bool` | default `True`; RPC entrypoints set `False` (`odoo/addons/base/controllers/rpc.py:142`) |
| `website` | `bool` | website-module convention flag, not a core `http.py` param — pulled in by the `website` addon's routing layer |

### `auth` modes — verbatim from `http.py:705-721`

- **`'user'`** — must be authenticated; request runs with that user's rights.
- **`'bearer'`** — reads an `Authorization: Bearer <token>` header; runs with
  the token owner's rights. Falls back to session (`'user'`-style) auth if the
  header is absent. This exists in 18.0 — it is not a 17→18 addition claimed
  by third-party docs, it is documented in the route decorator itself.
- **`'public'`** — authenticated or not; unauthenticated requests run as the
  shared Public user (`base.public_user`). This is the mode for any
  customer-facing route that must work logged-out (checkout, webhooks,
  portal pages with a token).
- **`'none'`** — always active, even with no database selected (multi-db
  selector, `/web/login`, RPC endpoints). No `request.env` user context.

Pick `'public'` for anything a browser or an external service hits without a
session, `'user'` for backend/portal actions that require a real login,
`'none'` only for framework-level, DB-less endpoints.

### `type='http'` vs `type='json'`

- `'http'`: params come from the query string / form body; the return value is
  wrapped through `Response.load(result)` (`http.py:755-756`) — a plain string
  becomes an HTML response, a `Response` object passes through.
- `'json'`: params come from the JSON-RPC-style request body (`{"params": {...}}`
  in the case of the `/jsonrpc` gateway, or a plain JSON body for `type='json'`
  ajax routes); the return value is JSON-serialized directly, no `Response`
  wrapping.
- CSRF default flips between the two (see table) because JSON routes are
  normally called from `fetch`/`rpc.js` with the Odoo session cookie plus the
  ajax layer's own protections, not from a plain HTML form.

## Request object (`odoo.http.request`)

```python
request.env                        # Environment bound to the current db/user/context
request.httprequest                # the underlying werkzeug Request
request.session                    # odoo.http.Session
request.params                     # merged/deserialized request parameters
```

Key methods (`http.py`, line refs from the `Request` class):

| Method | Line | Purpose |
|---|---|---|
| `request.render(template, qcontext=None, lazy=True, **kw)` | `http.py:1958` | lazy QWeb render — deferred until end of dispatch so a later step can still swap the template/qcontext |
| `request.make_response(data, headers=None, cookies=None, status=200)` | `http.py:1893` | non-HTML response, or HTML with custom headers/cookies. **Required** whenever the handler doesn't just return markup |
| `request.make_json_response(data, headers=None, cookies=None, status=200)` | `http.py:1916` | `json.dumps` + `Content-Type: application/json` |
| `request.redirect(location, code=303, local=True)` | `http.py:1942` | local-path-safe redirect; `local=True` strips scheme/netloc to prevent open-redirect |
| `request.not_found(description=None)` | `http.py:1935` | shortcut for a werkzeug `NotFound` (404) |
| `request.update_env(user=None, context=None, su=None)` | `http.py:1702` | swap user/context/sudo on the *current* request's env, in place |
| `request.csrf_token(time_limit=None)` | `http.py:1776` | generate/read the session CSRF token, e.g. to embed in a hand-built form |

## Returning files / downloads

```python
from odoo.http import content_disposition, request

@http.route('/my/report/<int:doc_id>', type='http', auth='user')
def download(self, doc_id):
    record = request.env['my.model'].browse(doc_id)
    pdf, _ = request.env['ir.actions.report'].sudo()._render_qweb_pdf(
        'my_module.report_action', [doc_id],
    )
    response = request.make_response(
        pdf,
        headers=[('Content-Type', 'application/pdf'),
                 ('Content-Disposition', content_disposition(f'{record.name}.pdf'))],
    )
    return response
```

Verified pattern: `addons/web/controllers/report.py:138` uses
`response.headers.add('Content-Disposition', content_disposition(filename))`
after building the response; `addons/web/controllers/vcard.py:36,45` builds
the header directly in `make_response(..., headers=[...])`.
`content_disposition` (imported from `odoo.http`) escapes the filename —
never build the header string by hand.

## CSRF

- Enabled by default on `type='http'` routes; the framework validates the
  token automatically on state-changing requests (POST/PUT/DELETE) when
  `csrf` is not explicitly disabled (`http.py:725-727`).
- Set `csrf=False` for any route a non-Odoo client posts to without ever
  holding an Odoo session cookie — external webhooks, RPC gateways,
  server-to-server callbacks. A CSRF token is session-bound; an outside
  caller cannot obtain one, so leaving CSRF on for such a route makes it
  permanently unreachable, not "more secure" — it just breaks.
- **`csrf=False` on a webhook is only safe paired with a payload signature
  check.** Verified pattern, `addons/payment_stripe/controllers/main.py`:
  ```python
  @http.route(_webhook_url, type='http', methods=['POST'], auth='public', csrf=False)
  def stripe_webhook(self):
      ...
      self._verify_notification_signature(tx_sudo)   # main.py:94
  ```
  `_verify_notification_signature` (`main.py:196-239`) reads the
  `Stripe-Signature` header, recomputes an HMAC over the raw body with the
  webhook secret, and compares with `hmac.compare_digest` — never `==`, which
  is not constant-time. Treat `csrf=False` + no signature/token check on a
  state-changing public route as a BLOCKER.
- Odoo's own RPC gateways (`/xmlrpc/2/<service>`, `/jsonrpc`) are `auth='none',
  csrf=False, save_session=False` (`odoo/addons/base/controllers/rpc.py:142,
  160, 174`) — they authenticate per-call via `execute_kw(db, uid, password, ...)`
  instead of a session, so there is nothing for CSRF to protect.

## `sudo()` in a public/portal route

`auth='public'` routes run as the Public user, who by design has almost no
`ir.model.access`/`ir.rule` grants. Reading or writing anything beyond the
Public user's own rights requires `.sudo()`, scoped as tightly as possible —
verified throughout `addons/website_sale/controllers/main.py` (e.g. line 483
`request.env['product.document'].browse(document_id).sudo().exists()`, line
776 `request.env['sale.order'].sudo().search([('access_token', '=',
access_token)], limit=1)`).

The pattern that keeps this safe: never `sudo()` a bare id from the URL —
always gate it behind a secret the caller must also supply (an
`access_token`, a signed id, a webhook signature). `sudo()` bypasses access
control, not authentication; the route still has to prove the caller is
allowed to see *this* record. See `odoo-slop-audit` B6 and workspace
`CLAUDE.md`/`security.md` — every `sudo()` needs a `# SAFETY:` comment naming
what it bypasses and why that's fine here.

## Inheriting a core controller

```python
from odoo.addons.web.controllers.report import ReportController

class MyReportController(ReportController):
    @http.route()                       # re-declare, no args needed to keep the same route
    def report_download(self, data, context=None):
        # do_setup() before/after, or short-circuit
        return super().report_download(data, context=context)
```

- Subclass the controller class, override the method, **always re-apply
  `@route()`** even with no arguments — the framework only rebuilds routing
  metadata for methods that carry the decorator in the child class too
  (`http.py:687-696` docstring).
- Omitted `@route()` kwargs are inherited/merged from the parent's routing;
  passed kwargs override just that key. This is how a module narrows `auth`
  from `'public'` to `'user'`, or flips `csrf`, without repeating the whole
  route signature.
- Odoo's own MRO-merging in `_generate_routing_rules` (`http.py:765-819`)
  walks the controller inheritance tree per installed module, so multiple
  addons can each extend the same route without knowing about each other —
  no explicit registration step needed, just `_inherit`-style class
  inheritance from the target controller class.

## Portal / website routes — basics

- Portal routes typically pair `auth='user'` with a `sudo()`'d lookup guarded
  by an `access_token` so a portal user can see one specific record without a
  full `ir.rule` grant on the model (the `payment_portal` base for
  `WebsiteSale.shop_payment`-style flows).
- Website (frontend, logged-out-capable) routes use `auth='public'`, often
  with `website=True` in the routing kwargs so the `website` addon's routing
  dispatcher applies (multi-website resolution, page cache, SEO furniture).
  This is a `website`-addon-level convention layered on top of core
  `http.route`, not a core `http.py` parameter.

## Error handling

- Let `UserError`/`AccessError`/`MissingError` propagate on `type='json'`
  routes — the JSON-RPC dispatcher serializes them into the standard error
  envelope the JS layer already knows how to show.
- On `type='http'` routes returning a rendered page, catch expected failures
  explicitly and turn them into a redirect or an error page — an uncaught
  exception becomes a raw 500, not a useful message, for a browser user.
- `handle_params_access_error` (routing kwarg) is the documented hook for
  converting an `AccessError`/`MissingError` raised while resolving a URL
  converter (e.g. `<model("my.model"):record>`) into a custom `Response`
  instead of the default error page.
- Never swallow the exception silently (`except Exception: pass`) to "make
  the route not crash" — that's `odoo-slop-audit` B8, and on a controller it
  also hides the caller getting a `200` for a request that actually failed.
