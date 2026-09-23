# Mixins and Useful Classes — Odoo 18.0

Source of truth: `addons/mail/models/mail_thread.py`, `mail_activity_mixin.py`,
`mail_thread_blacklist.py`, `mail_thread_cc.py`, `mail_alias_mixin.py`;
`addons/portal/models/portal_mixin.py`; `addons/utm/models/utm_mixin.py`;
`addons/website/models/mixins.py`; `addons/rating/models/rating_mixin.py`;
`odoo/addons/base/models/{image_mixin,avatar_mixin,res_partner}.py`;
`addons/account/models/sequence_mixin.py`. All are `models.AbstractModel` — no
table of their own, pulled in via multiple classical inheritance
(`_inherit = [...]`, see `orm-models.md`).

## Mixin catalog

| Mixin | Module | Gives | Key fields | Use for |
|---|---|---|---|---|
| `mail.thread` | `mail` | chatter: messages, followers, tracking | `message_ids`, `message_follower_ids`, `message_is_follower` | any document users discuss or whose field changes must be logged |
| `mail.thread.blacklist` | `mail` | opt-out tracking on an email field | `email_normalized`, `is_blacklisted`, `message_bounce` | marketing/contact models with a primary email (`_primary_email`) |
| `mail.thread.cc` | `mail` | preserves CC recipients from inbound mail | `email_cc` | models fed by `mail.alias`/incoming mail that must remember CC |
| `mail.activity.mixin` | `mail` | to-do/activity scheduling, shown in Chatter | `activity_ids`, `activity_state`, `activity_type_id` | any document with follow-up tasks (calls, meetings, "to do") |
| `mail.alias.mixin` | `mail` | one dedicated inbound email alias per record | `alias_id` (delegated, `_inherits`), `alias_name` | model where each record gets its own `catchall@` address (helpdesk ticket, project) |
| `mail.alias.mixin.optional` | `mail` | same, but the alias is optional (no `_inherits`) | `alias_id` (plain M2O) | same use case when not every record needs an alias |
| `portal.mixin` | `portal` | tokenized public/portal URL for a record | `access_url`, `access_token` | any document a portal user must open without logging in as internal |
| `rating.mixin` | `rating` | customer satisfaction rating stats | `rating_avg`, `rating_count`, `rating_last_value` | documents that trigger a rating request (helpdesk, project task) — requires `mail.thread` |
| `utm.mixin` | `utm` | captures campaign/source/medium from URL/cookies | `campaign_id`, `source_id`, `medium_id` | leads/orders created from a tracked marketing link |
| `image.mixin` | `base` | one image field + 4 auto-resized, stored variants | `image_1920` … `image_128` | any model needing a picture without hand-rolling `related`+`store` resizes |
| `avatar.mixin` | `base` (`_inherit = image.mixin`) | falls back to a generated initials SVG when no image is set | `avatar_1920` … `avatar_128` | users/partners/anything shown as an avatar in Discuss/Chatter |
| `format.address.mixin` | `base` | renders a postal address per the user's country format | — (methods only) | any model rendering `res.partner`-shaped addresses in its own view |
| `sequence.mixin` | `account` | parses/generates the next value of an editable sequence field (`INV/2024/00042`-style) | `sequence_prefix`, `sequence_number` (computed, stored) | documents needing a user-editable, gap-aware sequential number — despite living in `account`, it is generic |
| `website.seo.metadata` | `website` | per-record SEO meta tags | `website_meta_title/description/keywords` | any model with a public website page |
| `website.published.mixin` | `website` | publish/unpublish toggle with an access check | `website_published`, `is_published`, `can_publish` | any model with a public website page |
| `website.multi.mixin` | `website` | restrict a record to one website in a multi-website setup | `website_id` | combine with `website.published.mixin` (most modules use `website.published.multi.mixin` instead, which already inherits both) |
| `website.searchable.mixin` | `website` | plugs a model into the website global search | — (methods to override: `_search_get_detail`) | models that should show up in the website's search bar |

## `mail.thread` — how to inherit

```python
class HelpdeskTicket(models.Model):
    _name = 'helpdesk.ticket'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _description = 'Helpdesk Ticket'

    stage_id = fields.Many2one('helpdesk.stage', tracking=True)     # logs old -> new on every write
    priority = fields.Selection([...], tracking=15)                 # int = ordering among several tracked changes in one log entry
```

`tracking=True`/`tracking=<int>` is not a real ORM field kwarg — `mail.thread`
whitelists it via `_valid_field_parameter` (`mail_thread.py:480-482`), so
`tracking=` on a field of a model that does **not** inherit `mail.thread` is
silently ignored, not an error. `crm.lead.priority` uses `tracking=15`
(`addons/crm/models/crm_lead.py:128`) to control ordering when several tracked
fields change in the same `write()`.

Chatter in the form view — one tag, no manual `message_ids`/`activity_ids`
widgets needed:

```xml
<form>
    <sheet>...</sheet>
    <chatter/>
</form>
```

`reload_on_post`/`reload_on_attachment`/`reload_on_follower` control when the
whole form reloads vs. the chatter re-rendering in place, e.g.
`<chatter reload_on_post="True"/>` (`addons/crm/views/crm_lead_views.xml:333`).

### Posting from code

```python
self.message_post(
    body=_("Order %(name)s was cancelled by %(user)s.", name=self.name, user=self.env.user.name),
    subtype_xmlid='mail.mt_note',   # internal note, no customer notification
)
```

- `message_post(self, *, body='', ...)` (`mail_thread.py:2145`) — keyword-only,
  `self.ensure_one()` (`:2185`), and raises `ValueError` if called on the
  `mail.thread` abstract model itself or on an empty recordset — use
  `message_notify()` to reach a user with no document.
- `body` **must** be `_()`-wrapped per C2/B8 in `odoo-slop-audit`; whatever you
  pass gets HTML-escaped (`escape(body)`, `:2269`) unless it is already a
  `Markup` object.
- `body_is_html=True` exists only for RPC clients that cannot construct
  `Markup` themselves — internal-user calls that set it log a warning
  (`:2256-2258`) telling you to pass `Markup(...)` instead.
- `message_post_with_view()`/`message_post_with_source()` render a QWeb
  template into the body instead of building HTML by hand.

### Context keys that silence the mixin (imports, migrations, tests)

| Key | Silences |
|---|---|
| `mail_create_nosubscribe` | do not auto-subscribe `uid` on create |
| `mail_create_nolog` | skip the "Document created" log message |
| `mail_notrack` | skip field-change tracking (create and write) |
| `tracking_disable` | all of the above at once — subscription, tracking, auto-post |
| `mail_activity_automation_skip` | (`mail.activity.mixin`) skip all automated activity scheduling |

```python
self.with_context(tracking_disable=True).create(vals_list)   # bulk import, no chatter noise
```

Verified: the `create()`/`write()` overrides check these keys first thing
(`mail_thread.py:289`, `:336`, `:350`, `:353`; `mail_activity_mixin.py:35-37,
322, 358, 409, 429, 450, 466`).

### `_track_subtype` and `_mail_post_access`

- Override `_track_subtype(self, initial_values)` (`:596`, default returns
  `False`) to route a tracked change to a specific `mail.message.subtype`
  (e.g. notify followers only when `stage_id` changes, not on every field).
- `_track_get_fields()` (`:586`) is `ormcache`d and derived automatically from
  every field carrying `tracking=`/`track_visibility=` — you don't populate it
  yourself.
- `_mail_post_access = 'write'` (`:103`) is the access level required to call
  `message_post` on the record; override to `'read'` on a class attribute if
  read-only users (e.g. portal) must be able to comment.

## `mail.activity.mixin`

```python
self.activity_schedule(
    'mail.mail_activity_data_todo',
    date_deadline=fields.Date.context_today(self) + relativedelta(days=3),
    summary=_("Follow up with customer"),
    user_id=self.user_id.id,
)
self.activity_feedback(['mail.mail_activity_data_todo'], feedback=_("Called, no answer."))
```

- `activity_schedule(act_type_xmlid=..., date_deadline=..., summary=..., **act_values)`
  (`mail_activity_mixin.py:345`) — pass the activity type as an XML id, not a
  hardcoded id (B3 in `odoo-slop-audit`). `date_deadline` must be a `date`, not
  `datetime` — passing a `datetime` only logs a warning (`:363-364`), it does
  not raise.
- `activity_feedback(act_type_xmlids, user_id=None, feedback=None)` (`:447`)
  marks matching activities done.
- Archiving a record (`active = False`) deletes its pending activities —
  `write()` (`:237-243`) and `toggle_active()` (`:288-299`) both do this; do
  not schedule an activity expecting it to survive an archive.
- `activity_ids`, `activity_state`, etc. all carry `groups="base.group_user"`
  (`:50-94`) — a portal/public user never sees them, by design.

## `portal.mixin`

```python
class HelpdeskTicket(models.Model):
    _inherit = ['portal.mixin']

    def _compute_access_url(self):
        super()._compute_access_url()
        for ticket in self:
            ticket.access_url = f'/my/tickets/{ticket.id}'
```

- The base `_compute_access_url()` just sets `'#'` (`portal_mixin.py:25-27`) —
  **you must override it**, the mixin gives you the field and the token
  machinery, not the route.
- `get_portal_url(suffix=None, report_type=None, download=None, ...)`
  (`:115-134`) builds the full URL including `access_token`.
- `_portal_ensure_token()` (`:29-34`) lazily generates and persists a UUID
  token via `sudo().write(...)` the first time it's needed — don't generate
  your own token field for this.

## `website.*` mixins

- `website.published.mixin` guards publish state with an access check:
  `create`/`write` raise `AccessError` if a record ends up `is_published=True`
  while `can_publish` is `False` (`website/models/mixins.py:205-217`).
  `can_publish` defaults to "current user has write access" — override
  `_compute_can_publish` for anything stricter.
- Most first-party website content should inherit
  `website.published.multi.mixin` (`:250-303`), not `website.published.mixin`
  + `website.multi.mixin` separately — it already combines both and adds the
  per-website `website_published` compute (`:261-269`).
- `website.seo.metadata` fields are `translate=True` and grouped with
  `prefetch="website_meta"` (`:25-29`) — reading one meta field prefetches the
  other two in the same query.

## `image.mixin` / `avatar.mixin`

```python
class ProductBrand(models.Model):
    _inherit = ['image.mixin']          # -> image_1920, image_1024/512/256/128 for free
```

- `image_1920` is the field you write to; the four smaller variants are
  `related=` + `store=True` on it (`image_mixin.py:16-19`) — Odoo resizes and
  stores them as attachments automatically, don't hand-write your own
  `image_128 = fields.Image(...)`.
- `avatar.mixin` inherits `image.mixin` and adds `avatar_*` computed fields
  that fall back to a colored SVG with the record's initials
  (`_avatar_generate_svg`, `avatar_mixin.py:64-73`) when no image is set —
  set `_avatar_name_field` (default `'name''`) if the display name lives in a
  different field.

## Cost warning — `mail.thread` on high-volume models

`mail.thread`'s `create()`/`write()` overrides always run — auto-subscribe,
`_message_auto_subscribe`, and (unless `mail_notrack`/`tracking_disable`) a
tracking diff on every tracked field (`mail_thread.py:282-378`).
`message_follower_ids`/`message_ids` are `auto_join=True` One2many fields
(`:109-120`), so every chatter-bearing record also carries a row in
`mail.followers` per follower and a growing `mail.message` history. On a
model with heavy bulk `create`/`write` traffic (import staging tables, IoT
telemetry, POS order lines) this multiplies writes several-fold per business
operation. Prefer:
- `self.with_context(tracking_disable=True)` around bulk imports/migrations —
  see the context-key table above.
- Not inheriting `mail.thread` at all on line-level/detail models; put it on
  the header model only (e.g. `sale.order`, not `sale.order.line`).

## `sequence.mixin` (generic, lives in `account`)

```python
class MyDoc(models.Model):
    _inherit = ['sequence.mixin']
    _sequence_field = 'name'        # default
    _sequence_date_field = 'date'   # default
```

Parses the previous record's `_sequence_field` value with regex to compute
prefix/year/month/number, then increments — the mechanism behind editable,
gap-aware sequences like `INV/2024/00042` (`account/models/sequence_mixin.py:17-45`).
Despite the module path, nothing in it is AR/accounting-specific; it is
usable from any first-party module needing the same editable-sequence UX.

## Access rights for followers/activities

`mail.followers`, `mail.activity`, and `mail.message` already ship
`ir.model.access.csv` rows scoped to `base.group_user`
(`addons/mail/security/ir.model.access.csv:5,8,37`) — inheriting
`mail.thread`/`mail.activity.mixin` does **not** require you to add your own
access rows for those three models, only for your own model as usual.

## `mail.alias.mixin` vs `mail.alias.mixin.optional`

`mail.alias.mixin` uses `_inherits = {'mail.alias': 'alias_id'}`
(delegation — see `orm-models.md`) with `alias_id` **required**
(`mail_alias_mixin.py:14-19`): every record gets its own alias row, created
lazily via `_init_column_alias_id` for pre-existing rows (`:31-54`). Use
`mail.alias.mixin.optional` instead when only some records need an inbound
address (plain `Many2one`, not `_inherits`) — cheaper, no forced join.
