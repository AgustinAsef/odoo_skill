# Payment Providers — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/addons/payment/` (base module).
Templates studied: `payment_demo` (direct/token flow, no real API),
`payment_mercado_pago` (redirect flow, real HTTP API), `payment_paypal`
(redirect flow, signature-verified webhook).

## Module skeleton

```
payment_<x>/
    __init__.py                 # imports + post_init_hook/uninstall_hook
    __manifest__.py
    const.py                     # status/currency/payment-method mappings, no I/O
    controllers/main.py          # return route + webhook route
    models/payment_provider.py, payment_transaction.py, (payment_token.py)
    data/payment_provider_data.xml, (payment_method_data.xml)
    views/payment_<x>_templates.xml, payment_provider_views.xml
    tests/  common.py, test_payment_provider.py, test_payment_transaction.py,
            test_processing_flows.py
```

`payment_token.py` is only needed if the provider tokenizes itself — most
don't ship one. Verified file lists: `addons/payment_demo/`,
`addons/payment_mercado_pago/`.

## Manifest

```python
{
    'name': "Payment Provider: <X>",
    'category': 'Accounting/Payment Providers',
    'depends': ['payment'],
    'data': [
        'views/payment_x_templates.xml',
        'views/payment_provider_views.xml',
        'data/payment_provider_data.xml',  # LAST — references the views above by ref()
    ],
    'post_init_hook': 'post_init_hook',
    'uninstall_hook': 'uninstall_hook',
    'license': 'LGPL-3',
}
```

Verified: `addons/payment_mercado_pago/__manifest__.py:1-20`. `data:` order
matters: `payment_provider_data.xml` sets `inline_form_view_id` by `ref()`,
so the template file must load first.

**Workspace note**: a module reconciling `payment.transaction` to invoices
needs `accountant` in `depends`, not bare `account` (root `CLAUDE.md`,
"Invoicing means `accountant`").

## `post_init_hook` / `uninstall_hook`

```python
from odoo.addons.payment import setup_provider, reset_payment_provider

def post_init_hook(env):
    setup_provider(env, 'mercado_pago')

def uninstall_hook(env):
    reset_payment_provider(env, 'mercado_pago')
```

Verified: `addons/payment_mercado_pago/__init__.py:1-14`. Both are thin
wrappers in the base module (`addons/payment/__init__.py:9-14`).
`setup_provider` calls `_setup_provider(code)` (override point, default
no-op, `payment_provider.py:670-680`). `reset_payment_provider` calls
`_remove_provider(code, **kwargs)` (`:682-713`): searches by
`_get_removal_domain` (default `[('code', '=', code)]`) and writes
`_get_removal_values()` — resets `code`/`state`/publish, clears the four
view fields. Override `_get_removal_values` (`super()` first) for own fields.

## Provider record: placeholder + data file

The base module ships one **placeholder** `payment.provider` per known code,
`noupdate="1"`, with only `name`/`image_128`/`module_id`/`payment_method_ids`
set (`addons/payment/data/payment_provider_data.xml:209-214`, `:237-254`). A
new provider module extends that placeholder by the same xml id (prefixed
`payment.`), it does not create its own record:

```xml
<odoo noupdate="1">
    <record id="payment.payment_provider_demo" model="payment.provider">
        <field name="code">demo</field>
        <field name="state">test</field>
        <field name="inline_form_view_id" ref="inline_form"/>
        <field name="allow_tokenization">True</field>
        <field name="payment_method_ids" eval="[Command.set([ref('payment_demo.payment_method_demo')])]"/>
    </record>
</odoo>
```

Verified: `addons/payment_demo/data/payment_provider_data.xml:1-20`. No
placeholder for a brand-new provider (rare, ~20 listed): add one in your own
module's data, without the `payment.` prefix.

## `payment.provider` fields

| Field | Kind | Where |
|---|---|---|
| `code` | `Selection`, extend via `selection_add=[('x', "X")]`, `ondelete={'x': 'set default'}` | `payment_provider.py:27-33` |
| `state` | `disabled`/`enabled`/`test` | `:34-39` |
| `available_country_ids`, `available_currency_ids` | M2M, empty = "all" | `:99-120` |
| `maximum_amount` | `Monetary`, currency = `main_currency_id` | `:121-126` |
| `redirect_form_view_id`, `inline_form_view_id`, `token_inline_form_view_id`, `express_checkout_form_view_id` | `ir.ui.view` (`type='qweb'`), `ondelete='restrict'` | `:71-96` |
| `module_id` | link to `ir.module.module` for the "Install" button | `:181-183` |

A credential field uses the custom `required_if_provider` parameter instead
of `required=True` — required only once the provider is `enabled`/`test`:

```python
mercado_pago_access_token = fields.Char(
    required_if_provider='mercado_pago', groups='base.group_system',
)
```

Verified: `payment_mercado_pago/models/payment_provider.py:24-28`.
`_valid_field_parameter` whitelists the parameter (`payment_provider.py:21-22`);
`_check_required_if_provider` enforces it in `create`/`write` (`:331-356`).
`groups='base.group_system'` keeps secrets out of `read()` — see `references/security.md`.

### Overridable methods

| Method | Default | Override to |
|---|---|---|
| `_compute_feature_support_fields()` | all four `support_*` → `None`/`'none'` | `@api.depends('code')`, `super()` first, then `.filtered(lambda p: p.code == 'x')` |
| `_get_supported_currencies()` | all currencies (`ensure_one`) | filter by ISO code list |
| `_get_default_payment_method_codes()` | `set()` (`ensure_one`) | codes to auto-activate on enable |
| `_should_build_inline_form(is_validation=False)` | `True` | `False` for pure-redirect providers |
| `_get_validation_amount()` / `_get_validation_currency()` | `0.0` / currency intersection | only for tokenization-by-validation-charge |
| `_setup_provider(provider_code)` | no-op | install-time setup (rare) |
| `_get_removal_values()` | resets `code`/`state`/publish/views | extend with `super()` + own fields |

Every provider module calls `super()` first in `_compute_feature_support_fields`,
then narrows with `.filtered` — it runs over the *whole* `payment.provider`
recordset (`payment_demo/models/payment_provider.py:16-24`).

## `payment.transaction` — state machine

`draft` → `pending` → `authorized` → `done`, with `pending`/`authorized` also
able to fall to `cancel`/`error`:

| Method | Allowed source states | Also does |
|---|---|---|
| `_set_pending(state_message=None, extra_allowed_states=())` | `draft` | logs "received" message |
| `_set_authorized(...)` | `draft`, `pending` | — |
| `_set_done(...)` | `draft`, `pending`, `authorized`, `error` | `_update_source_transaction_state()` |
| `_set_canceled(...)` | `draft`, `pending`, `authorized` | `_update_source_transaction_state()` |
| `_set_error(state_message, ...)` | `draft`, `pending`, `authorized` | — |

Verified: `payment_transaction.py:678-763`. `state == 'authorized'` requires
`provider_id.support_manual_capture` truthy, enforced by
`@api.constrains('state')` (`:140-150`). A transition from a disallowed
source state is **not** an error: `_update_state` logs a warning and skips
the write silently (`:765-823`) — check `tx.state` after the call for a
double-notification race, don't rely on an exception.

### Hooks per flow

**Redirect** (customer leaves Odoo):

```python
def _get_specific_rendering_values(self, processing_values):
    res = super()._get_specific_rendering_values(processing_values)
    if self.provider_code != 'x':
        return res
    return {'api_url': ..., 'url_params': ...}  # built from a provider API call
```

Called from `_get_processing_values()` (`:402-458`) only when `operation in
('online_redirect', 'validation')` and `redirect_form_view_id` is set.

**Direct/token** (`operation in ('online_token', 'offline')`, `:509-524`,
pattern in `payment_demo/models/payment_transaction.py:62-78`):

```python
def _send_payment_request(self):
    super()._send_payment_request()  # checks provider not disabled, logs "sent"
    if self.provider_code != 'x':
        return
    self._handle_notification_data('x', notification_data)  # after the API call
```

**Every flow — close the loop through this pair**, never call
`_process_notification_data` directly:

```python
def _get_tx_from_notification_data(self, provider_code, notification_data):
    tx = super()._get_tx_from_notification_data(provider_code, notification_data)
    if provider_code != 'x' or len(tx) == 1:
        return tx
    reference = notification_data.get('external_reference')
    if not reference:
        raise ValidationError("X: " + _("Received data with missing reference."))
    tx = self.search([('reference', '=', reference), ('provider_code', '=', 'x')])
    if not tx:
        raise ValidationError("X: " + _("No transaction found matching reference %s.", reference))
    return tx

def _process_notification_data(self, notification_data):
    super()._process_notification_data(notification_data)
    if self.provider_code != 'x':
        return
    self.provider_reference = notification_data['payment_id']
    # map provider status -> _set_pending / _set_done / _set_canceled / _set_error
```

`_handle_notification_data` (`:637-647`) is the only entry point — same
skeleton in `payment_demo` (`:126-189`) and `payment_mercado_pago` (`:104-194`).

**Refund/capture/void** (optional, gated by `support_refund`/`support_manual_capture`):
`_send_refund_request`/`_send_capture_request`/`_send_void_request`
(`:526-586`) already call `_create_child_transaction` for you — an override
just adds the API call and returns what `super()` returned. Pattern:
`payment_demo/models/payment_transaction.py:80-124`.

## Post-processing: cron + poll route

`payment/data/payment_cron.xml:4-13` defines `cron_post_process_payment_tx`,
inactive by default; `_toggle_post_processing_cron()`
(`payment_provider.py:358-373`) flips it active only while some provider is
`enabled`/`test`. It calls `_cron_post_process()` (`payment_transaction.py:849-875`),
which retries any tx with `is_post_processed = False` and `last_state_change
>= now() - 4 days`, committing per-tx and rolling back on error (logged,
never re-raised — legitimate exception to slop rule B8, the cron must keep
going for the rest of the batch).

Normally the cron never has to act: `controllers/post_processing.py` exposes
`/payment/status` and `/payment/status/poll` (`type='json'`, polled by the
frontend after redirect), calling `tx._post_process()` directly (`:39-71`).
Override `_post_process()` (not `_cron_post_process`) for your own side
effect — base only sets `is_post_processed = True` (`:877-887`).

## Webhook controller

```python
@http.route(f'{_webhook_url}/<path:reference>', type='http', auth='public',
            methods=['POST'], csrf=False)
def x_webhook(self, reference, **_kwargs):
    data = request.get_json_data()
    try:
        request.env['payment.transaction'].sudo()._handle_notification_data(
            'x', {'external_reference': reference, ...}
        )
    except ValidationError:  # Acknowledge anyway — avoid a provider retry-storm.
        _logger.exception("Unable to handle the notification data; skipping to acknowledge")
    return ''
```

Verified against `payment_mercado_pago/controllers/main.py:36-64` and
`payment_paypal/controllers/main.py:51-84`. `auth='public'` + `csrf=False`:
the provider's server has no Odoo session, so `sudo()` on
`_handle_notification_data` is the matching `# SAFETY:` case. **Always
return 200** — catching `ValidationError` and swallowing it is deliberate
(not B8 slop: it's the provider's own retry contract, not our bug). Filter
by event type first (`mercado_pago`: `data.get('action') in
('payment.created', 'payment.updated')`; `paypal`: `data.get('event_type')
in const.HANDLED_WEBHOOK_EVENTS`).

### Origin verification — two patterns

1. **Re-fetch from the API** (Mercado Pago): the webhook carries only a
   payment id; `_process_notification_data` does a server-to-server
   `GET /v1/payments/{id}` and trusts *that* response, never the webhook
   body (`payment_mercado_pago/models/payment_transaction.py:148-151`) —
   the body can't spoof state because it never drives the transition.
2. **Verify a signature header** (PayPal): `_verify_notification_origin`
   builds a payload from `PAYPAL-TRANSMISSION-*` headers plus the stored
   `paypal_webhook_id`, POSTs it to PayPal's own verification endpoint, and
   raises `Forbidden` on anything but `SUCCESS`
   (`payment_paypal/controllers/main.py:118-141`).

Use whichever the API supports; don't invent a third pattern (self-computed
HMAC) unless the provider's docs specify it exactly.

## `payment.method` and `payment.token`

`payment.method`: `code` (provider-agnostic, e.g. `'card'`), `provider_ids`
(M2M back), `brand_ids`/`primary_payment_method_id` (a brand like "VISA"
points at "Card"), `supported_country_ids`/`supported_currency_ids` narrow
availability like the provider's own `available_*` fields
(`payment_method.py:16-95`). Map a provider code to the generic one with
`env['payment.method']._get_from_code(code, mapping=const.PAYMENT_METHODS_MAPPING)`
(`:306-320`, `mapping` = `{generic: 'specific1,specific2'}`). Add a new
`payment.method` record only if the base module's set
(`addons/payment/data/payment_method_data.xml`) has no match.

`payment.token`: created via a provider override of
`_get_specific_create_values(provider_code, values)` (`:63-76`) or directly
in `create()` from `_process_notification_data`. `provider_ref` is the
provider's own token id, distinct from `payment_transaction.provider_reference`
(`:48-50`). Unarchiving (`active: True`) is blocked if the method is
inactive or the provider `disabled` (`:78-100`).

## `payment.utils` helpers

| Helper | Use |
|---|---|
| `generate_access_token(*values)` / `check_access_token(...)` | HMAC token for a customer-facing route — not for provider webhook auth |
| `singularize_reference_prefix(prefix='tx', separator='-')` | timestamp-suffix a placeholder prefix so `_compute_reference` isn't scanning a huge shared sequence |
| `to_major_currency_units` / `to_minor_currency_units` | convert `amount` to/from the integer minor-unit format most APIs expect |
| `check_rights_on_recordset(recordset)` | `recordset.check_access('write')` before an RPC-reachable action that then runs `sudo()` |
| `generate_idempotency_key(tx, scope=None)` | sha1 of `database.uuid + reference + scope` |

Verified: `addons/payment/utils.py:15-243`.

## Testing

`payment.tests.common.PaymentCommon` (`addons/payment/tests/common.py:17`,
extends `BaseCommon`) sets up currencies/countries/users/partners and a
`dummy_provider` (`code='none'`) with a minimal qweb redirect form.
`payment.tests.http_common.PaymentHttpCommon` (`http_common.py:15`) adds
`HttpCase` for controller/tour tests. A provider module's `tests/common.py`
subclasses one of these and adds its provider record + fake API responses —
patch the `requests` call at the transport boundary (`odoo-slop-audit` A5),
never the module's own `_process_notification_data`. See
`payment_mercado_pago/tests/test_processing_flows.py`: build a fake webhook
payload, call the controller, assert on `tx.state`. For `TransactionCase`/
`Form`/`tagged()`/`odoo-bin --test-tags` mechanics, see `references/testing.md`.
