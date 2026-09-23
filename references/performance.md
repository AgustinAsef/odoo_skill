# Performance — Odoo 18 ORM

Rules for avoiding the two costs that actually show up in this workspace:
N+1 queries against odoo.sh's shared Postgres, and recompute storms on
`noupdate="False"` data reloads. Verified against V18 source
(`odoo/models.py`, `odoo/fields.py`) — see `odoo-slop-audit` Part B for the
audit-time version of these same rules.

## Rule 1 — no ORM call inside a loop

`search`, `browse`, `read`, `write`, `create`, `search_count` inside a
`for` is an N+1: one round-trip to Postgres per iteration, instead of one for
the whole batch.

```python
# N+1 — one query per order
for order in orders:
    lines = self.env['sale.order.line'].search([('order_id', '=', order.id)])

# one query, then in-memory partition
lines = self.env['sale.order.line'].search([('order_id', 'in', orders.ids)])
by_order = lines.grouped('order_id')   # models.py:6548 — dict, same prefetch set
```

`write()` is the same cost, and the fix is usually cheaper than for `search`:
build the recordset first, `write()` once.

```python
# N writes
for line in lines:
    line.write({'state': 'done'})

# one write, one recompute pass
lines.write({'state': 'done'})
```

`create()` in a loop is worse than N+1 — each single-record `create` also
runs the model's full compute/constrain/onchange machinery separately.
Always batch into one `vals_list` and call `create(vals_list)` once; this is
also why every `create` override needs `@api.model_create_multi`
(`odoo/api.py:501`) — a non-multi override forces callers back into the
one-at-a-time pattern.

## Rule 2 — batch reads: `search_fetch`, `_read_group`, `grouped`

| Need | Use | Not |
|---|---|---|
| ids + specific fields, one query | `search_fetch(domain, field_names)` (`models.py:1759`) | `search()` then loop-`read()` |
| aggregates grouped by a key | `_read_group(domain, groupby, aggregates)` (`models.py:1965`) | manual loop + `sum()`/`len()` |
| partition an already-loaded recordset by a key, no aggregation | `records.grouped('field')` or `grouped(callable)` (`models.py:6548`) | `records.filtered(lambda r: r.x == v)` per value |
| dict-shaped rows for a JS/report consumer | `search_read(domain, fields)` (`models.py:6123`) | `search()` + manual `read()` |

`_read_group` signature in 18 (`models.py:1965`):

```python
def _read_group(self, domain, groupby=(), aggregates=(), having=(), offset=0, limit=None, order=None):
```

`groupby` and `aggregates` are lists of strings, not the old `fields`/`groupby`
dict-return API (`read_group`, `models.py:2806`, still exists for
backward/RPC compatibility but `_read_group` is the internal, typed, faster
entry point — prefer it in new Python code). `aggregates` entries are
`'field:agg'` (e.g. `'amount_total:sum'`); the possible `agg` values are any
PostgreSQL aggregate plus `count_distinct` and `recordset`
(`models.py:1979-1981`).

`grouped()` (`models.py:6548-6573`) does **not** hit the database — it
partitions a recordset already in memory/prefetch. Use it instead of a
`filtered()` per distinct value; a `filtered()` in a loop over N possible
values is itself an O(N × len(records)) scan of the same recordset N times.

## Rule 3 — `mapped`/`filtered` cost model

Both are O(n) single-pass over the recordset **already in memory** — cheap
compared to a query, but not free:

- `mapped('partner_id.name')` on a dotted path triggers the prefetch of
  `partner_id` for the whole recordset in one query (not one per record) —
  this is the correct way to touch a related field across many records, not
  a loop of `record.partner_id.name`.
- `mapped(callable)` and `filtered(callable)` run Python on every record;
  chaining several (`recs.filtered(f).mapped(g).filtered(h)`) is three
  passes where one comprehension or one `_read_group` call would be one —
  see `odoo-slop-audit` A3/A4 for the general anti-pattern.
- Never call `mapped`/`filtered` **inside** a loop over another recordset —
  that reintroduces the N+1 shape even though no SQL keyword appears in the
  line.

## Rule 4 — prefetch

Reading one field on one record of a recordset prefetches that field (and
Odoo's heuristically-chosen sibling fields) for the **entire prefetch set**
in one query — this is why `for rec in records: rec.some_field` is fine
performance-wise (one query total) while `for rec in records:
self.env['other.model'].search(...)` is not (a fresh query per record,
because `search` is not a field read and has no prefetch set to share).

`grouped()`'s returned recordsets deliberately keep the same
`_prefetch_ids` as the source (`models.py:6572`,
`browse = functools.partial(type(self), self.env,
prefetch_ids=self._prefetch_ids)`) — reading a field on one group after
`grouped()` does not re-trigger a full prefetch per group.

## Rule 5 — indexes

`fields.py:160-168` documents the `index` kwarg values:

| Value | Use for |
|---|---|
| `"btree"` / `True` | standard index — most `Many2one` fields that are searched/joined on |
| `"btree_not_null"` | most values are NULL, or NULL is never searched for — smaller index, `fields.py:472` makes this the **default** for `Many2one` when `index` is not explicitly set |
| `"trigram"` | GIN index for `ilike`/full-text search on a `Char`/`Text` field |
| `None` / `False` (default for non-relational fields) | no index |

`index` only affects **stored** fields (`fields.py:161`); setting it on a
non-stored computed field does nothing. Do not add `index=True` to every
`Many2one` reflexively — a searched-on `Many2one` already defaults to
`btree_not_null`; only override when the field is heavily searched *and*
mostly non-NULL (then plain `"btree"`), or when it's a `Char` searched with
`ilike` (`"trigram"`).

## Rule 6 — raw SQL, flush, invalidate

`self.env.cr.execute(...)` bypasses the ORM cache, record rules, and
computed fields entirely. Only justified for a measured performance problem
(bulk update touching millions of rows where ORM overhead dominates) — and
then:

- Call `self.env.flush_all()` (or `flush_model`/the field-specific flush)
  **before** the raw SQL if any pending ORM writes touch the same rows, or
  the SQL will read stale data.
- Call `self.env.invalidate_all()` (or the model-specific invalidate)
  **after** the raw SQL, or the in-memory ORM cache will keep serving the
  pre-SQL values for the rest of the transaction.
- Build the query with the `odoo.tools.sql.SQL` composable helper
  (`odoo/tools/sql.py:48`, `SQL(code, *args, to_flush=...)`), never an
  f-string or `%`-formatted string with a value that ever originated from
  user input — see `odoo-slop-audit` B4, a BLOCKER-level finding.

## Rule 7 — profiling

- `Profiler` context manager (`odoo/tools/profiler.py`) records SQL queries
  and/or periodic stack traces; results are viewable as a speedscope
  flamegraph or exported to JSON.
- In tests, `self.profile()` on a `TransactionCase` gives the same
  instrumentation around a test body.
- **Odoo Online (odoo.sh included) databases cannot be profiled** — profile
  against the local Docker lab (`Versiones/V18/`), not against a staging or
  production odoo.sh build.
