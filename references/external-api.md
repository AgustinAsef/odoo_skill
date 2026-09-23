# External API (XML-RPC / JSON-RPC) — Odoo 18.0

Source of truth: `Versiones/V18/odoo-18.0/odoo/addons/base/controllers/rpc.py`,
`Versiones/V18/odoo-18.0/odoo/api.py` (`call_kw`, `model_create_multi`),
`Versiones/V18/odoo-18.0/odoo/addons/base/models/res_users.py` (`res.users.apikeys`).
Doc-only claims (not verified against 18.0 source, from `odoo.com/documentation/18.0`)
are marked **[doc]**.

## Endpoints

| Endpoint | `auth` | Notes |
|---|---|---|
| `/xmlrpc/2/<service>` | `none` | modern XML-RPC, int fault codes. `rpc.py:160` |
| `/xmlrpc/<service>` | `none` | legacy XML-RPC, string fault codes — kept for compat. `rpc.py:142` |
| `/jsonrpc` | `none` | JSON-RPC 2.0 envelope. `rpc.py:174` |

All three are `csrf=False, save_session=False` — auth happens per-call via
`uid`/`password` (or API key) in every request, not via a session cookie.
`service` is `'common'` (unauthenticated meta-calls) or `'object'` (business
object calls via `execute_kw`).

## Authenticating

```python
import xmlrpc.client

url, db = 'https://mycompany.odoo.com', 'mycompany'
username, password = 'user@example.com', 'API_KEY_OR_PASSWORD'   # see API keys below

common = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/common')
common.version()                                    # [doc] sanity check, no auth needed
uid = common.authenticate(db, username, password, {})

models = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/object')
```

`common.authenticate` returns a `uid` (int) or `False`. On Odoo Online/SaaS
users have no local password by default **[doc]** — either set one under
Settings > Users, or generate an API key (Preferences > Account Security >
New API Key) and use it as the `password` argument everywhere; it is
functionally a password for API calls only, cannot log into the web UI, and
cannot be recovered once lost — only revoked and regenerated **[doc]**.

Server-side, an API key authenticates through `res.users.apikeys._check_credentials
(scope='rpc', key=...)` (`res_users.py:2324`), tried after the normal password
check fails for non-interactive contexts (`res_users.py:2306-2337`). A model
can force API-key-only RPC access by overriding `_rpc_api_keys_only()`
(`res_users.py:2302-2304`, default `False`) — e.g. for 2FA-enabled users where
a bare password must never work over RPC.

## Calling methods — `execute_kw`

```python
partners = models.execute_kw(
    db, uid, password,
    'res.partner', 'search_read',
    [[('customer_rank', '>', 0)]],            # positional args: domain
    {'fields': ['name', 'email'], 'limit': 50, 'context': {'lang': 'en_US'}},
)
```

Signature: `execute_kw(db, uid, password, model, method, args, kwargs=None)`.
`args` is always a list (positional args to the ORM method), `kwargs` a dict
(keyword args, including `context`). This dispatches through `odoo.api.call_kw`
(`api.py:514`).

### `create` — the `model_create_multi` gotcha

```python
# create() decorated with @api.model_create_multi (the workspace standard,
# odoo-slop-audit B2) — args[0] can be a dict OR a list of dicts:
new_id = models.execute_kw(db, uid, password, 'res.partner', 'create',
                            [{'name': 'Acme'}])                       # -> int
new_ids = models.execute_kw(db, uid, password, 'res.partner', 'create',
                             [[{'name': 'A'}, {'name': 'B'}]])        # -> [int, int]
```

**If `create` is overridden without `@api.model_create_multi`** (no decorator
at all, or only `@api.model`), `call_kw` no longer recognizes it as a
"no ids" method. Verified mechanics, `odoo/api.py:514-538`:

```python
def call_kw(model, name, args, kwargs):
    ...
    api = getattr(method, '_api', None)
    if api:
        recs = model                       # @api.model / @api.model_create(_multi) path
    else:
        ids, args = args[0], args[1:]      # <-- undecorated create falls here
        recs = model.browse(ids)
    ...
    result = getattr(recs, name)(*args, **kwargs)
```

An undecorated `create` has no `_api` marker, so `call_kw` treats `args[0]`
as record ids to browse and the rest as positional args to the call. To reach
`create([vals])` you must send `args = [[], [{...}]]` — an empty id list
(browsed to an empty recordset) followed by the vals list as the next
positional arg. This is the exact bug behind the workspace gotcha
(`l10n_latam_check` breaking journal creation, memory
`odoo-l10n-latam-check-rompe-create-journal`): a third-party override dropped
`@api.model_create_multi`, and every RPC caller sending plain `[vals]`
silently mis-dispatched. **Never override `create` without
`@api.model_create_multi`** — see `orm-models.md` and `odoo-slop-audit` B2.

### `search`, `search_read`, `read`, `write`, `unlink`

```python
ids = models.execute_kw(db, uid, password, 'res.partner', 'search',
                         [[('is_company', '=', True)]], {'limit': 10, 'order': 'name'})

rows = models.execute_kw(db, uid, password, 'res.partner', 'search_read',
                          [[('id', 'in', ids)]],
                          {'fields': ['name', 'vat'], 'limit': 10})

models.execute_kw(db, uid, password, 'res.partner', 'write',
                   [ids, {'category_id': [(4, tag_id)]}])

models.execute_kw(db, uid, password, 'res.partner', 'unlink', [ids])
```

Always pass `fields` to `search_read`/`read` — omitting it reads every field
on the model, including computed ones, which is expensive over RPC for no
benefit. Always pass `limit` on `search`/`search_read` in a one-shot script —
an unbounded domain against a client's production table is the RPC-script
version of `odoo-slop-audit` B1.

### `Command` tuples over RPC — raw form only

```python
# a Python client CANNOT import odoo.fields.Command — send the raw tuple.
# Command.CREATE=0, UPDATE=1, DELETE=2, UNLINK=3, LINK=4, CLEAR=5, SET=6
# (odoo/fields.py:4305-4311)
models.execute_kw(db, uid, password, 'sale.order', 'write', [
    [order_id],
    {'order_line': [
        (0, 0, {'product_id': product_id, 'product_uom_qty': 1}),   # create
        (1, line_id, {'product_uom_qty': 2}),                        # update
        (4, other_line_id, 0),                                       # link
        (2, dead_line_id, 0),                                        # delete
    ]},
])
```

Server-side, `Command` is an `IntEnum` whose XML-RPC marshaller entry is
`dispatch[Command] = dispatch[int]` (`odoo/addons/base/controllers/rpc.py:108`)
— it marshals as a bare int, so on the way *out* (a `read()` result) a
one2many/many2many is just a list of ids, never `Command` tuples. On the way
*in* (a `write`/`create` payload) you always build the raw `(code, id, values)`
tuple yourself; there is no `Command` object to construct outside the Odoo
Python process.

## Batch create

Prefer one `create` call with a list of dicts over N calls with one dict each
— this is true over RPC even more than in-process, since every `execute_kw`
round-trip pays full HTTP + auth overhead:

```python
ids = models.execute_kw(db, uid, password, 'product.template', 'create', [[
    {'name': 'Product A', 'list_price': 10.0},
    {'name': 'Product B', 'list_price': 20.0},
]])
```

## Context over RPC

```python
models.execute_kw(db, uid, password, 'product.template', 'search_read',
                   [[]], {'fields': ['name'], 'context': {'lang': 'es_AR', 'active_test': False}})
```

`context` is a `kwargs` key like any other, not a separate `execute_kw`
parameter — merge it into the trailing dict. Common keys: `lang` (translated
field reads/writes), `active_test: False` (include archived records in
`search`), `tracking_disable: True` (suppress mail.thread log noise on bulk
loads), `allowed_company_ids` (multi-company scoping).

## Data marshalling gotchas

Verified from `OdooMarshaller` (`odoo/addons/base/controllers/rpc.py:71-110`):

| Python type | Wire form | Detail |
|---|---|---|
| `datetime` | ISO string, naive UTC | `dump_datetime` calls `Datetime.to_string(value)` (`rpc.py:84-87`) — Odoo stores/reads `Datetime` fields as naive UTC; the string carries no offset, so the client must know it's UTC |
| `date` | ISO string `YYYY-MM-DD` | `dump_date` calls `Date.to_string(value)` (`rpc.py:90-92`) |
| `bytes` | base64 string | `dump_bytes` decodes to text (`rpc.py:81-82`), historical Odoo convention — not raw XML-RPC `Binary` |
| `None` | **[doc]** — never appears; unset/`False` fields marshal as `False`, not `None`. A `search`/`read` result never contains a bare `null`/`None` for a scalar field |
| `Markup` | plain string | stringified before send (`rpc.py:110`) |
| Control chars (0-31 except tab/CR/LF) | stripped | `dump_unicode` strips them (`rpc.py:98-100`) — XML 1.0 disallows them; a field value containing one silently loses it on the wire |

Because unset is always `False`, never write `if value is None:` against a
value that came back from `execute_kw` — check `if value is False:` or, for a
relational field, `if not value:` (an empty `read` on a `Many2one` is `False`,
not `None`, not `0`).

## Exceptions over RPC

`odoo.exceptions.{UserError, AccessError, AccessDenied, RedirectWarning,
MissingError}` are translated into `xmlrpc.client.Fault` on the way out
(`xmlrpc_handle_exception_int`/`_string`, `rpc.py:34-68`) — catch
`xmlrpc.client.Fault` client-side and inspect `.faultCode`/`.faultString`
rather than assuming a raw traceback.

## One-shot script skeleton (workspace convention)

Per root `CLAUDE.md`, an RPC load script for one client's data lives in
`Implementaciones/<Client>/material/scripts/`, resolves paths two levels up,
and never writes its output beside itself:

```python
import os
import xmlrpc.client

MATERIAL_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
RECIBIDO_DIR = os.path.join(MATERIAL_DIR, "recibido")
GENERADO_DIR = os.path.join(MATERIAL_DIR, "generado")

URL = "https://<client>.odoo.com"
DB = "<db_name>"
USERNAME = "<user>"
API_KEY = os.environ["ODOO_API_KEY"]  # never hardcode a credential in the script

common = xmlrpc.client.ServerProxy(f"{URL}/xmlrpc/2/common")
uid = common.authenticate(DB, USERNAME, API_KEY, {})
models = xmlrpc.client.ServerProxy(f"{URL}/xmlrpc/2/object")

def execute(model, method, *args, **kwargs):
    return models.execute_kw(DB, uid, API_KEY, model, method, list(args), kwargs)

if __name__ == "__main__":
    rows = execute(
        "res.partner", "search_read",
        [("customer_rank", ">", 0)],
        fields=["name", "email"], limit=100,
    )
    print(f"{len(rows)} partners")
```

Credentials come from the environment or a secrets file outside the repo,
never hardcoded — see root `CLAUDE.md` "Credenciales cargadas: las usa Odoo,
no yo" and "Credenciales de produccion = solo lectura" conventions before
writing anything against a client's production database.
