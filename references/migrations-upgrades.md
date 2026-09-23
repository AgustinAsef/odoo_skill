# Migrations and upgrades — Odoo 18

Covers when a migration script is needed, how the `migrations/` folder is
resolved and executed, and odoo.sh-specific consequences from the workspace
`CLAUDE.md`. Verified against `odoo/modules/migration.py` (V18 source, full
file read — line numbers below are exact).

## When a migration script is needed

A migration script is code that runs **once**, at upgrade time, to fix up
existing data that a plain manifest/view/field change cannot reach. Needed
when:

| Situation | Why |
|---|---|
| Renamed a field (kept the data) | ORM does not know old and new column are the same; write a `pre-` SQL rename or a `post-` copy |
| Changed a `Selection` field's stored keys | Existing rows still hold the old key strings; `post-` script remaps them |
| Renamed the **module** itself | Needs `pre-` SQL against `ir_module_module`, `ir_model_data`, etc. before the new name's tables exist |
| Renamed a **model** (`_name` changed) | Same category as a module rename — `ir_model`/`ir_model_data` bookkeeping |
| Changed a field's type in an incompatible way (e.g. `Char` → `Many2one`) | ORM's automatic column conversion cannot infer the mapping |
| Backfilling a new required/computed field for already-existing rows | A `post-` script after the module (and its new field) has loaded |
| Cleaning up data left by a previous, now-removed feature | `end-` script, runs after every module in the transaction has updated |

**Not** needed: a new optional field, a new model, a new view, a new default
that only affects future records — the module's own `data`/`demo` XML and
ORM auto-migration (Odoo adds/alters columns automatically on `-u`) already
cover those.

## Folder structure and resolution

```
<module>/migrations/
├── 18.0.2.0.0/              exact module version — only this Odoo major
│   ├── pre-rename_field.py
│   ├── post-backfill_data.py
│   └── end-cleanup.py
├── 2.0/                     "majorless" — matches any Odoo major, only module version part
│   └── post-fix_typo.py
└── 0.0.0/                   runs on every version change, any version
    └── end-invariants.py
```

Verified `odoo/modules/migration.py:58-88` (class docstring) and the
`VERSION_RE` regex at `migration.py:23-47`: a version directory is either a
full `major.minor[.patch...]` string or a bare module version (2+ digit
groups); a `tests` directory is explicitly excluded
(`migration.py:108-109`).

Scripts also resolve from a separate `odoo.upgrade` namespace package outside
the module (`_get_upgrade_path`, `migration.py:97-101`) — irrelevant to this
workspace's own modules, that path is for Odoo's own OCA/Enterprise upgrade
scripts collection.

## Script signature

```python
def migrate(cr, installed_version):
    ...
```

- File must start with `pre-`, `post-`, or `end-`; anything else (e.g. a
  `README.txt` or a helper module without that prefix) is ignored
  (`migration.py:189`, `exec_script` checks `.py` extension at
  `migration.py:234-235`).
- The function **must** be named `migrate` and take exactly two positional
  parameters named `cr`/`_cr` and `version`/`_version` — any other name or a
  keyword-only parameter raises `TypeError` at load time
  (`migration.py:221-224, 251-255`, `VALID_MIGRATE_PARAMS`).
- The second argument is the **currently installed** version (before this
  script runs), not the target version and not the version-folder name — the
  docstring's `migrate(cr, installed_version)` naming is correct, the
  official docs page's `migrate(cr, version)` phrasing is the same
  parameter, just named differently; don't confuse it with "the version
  being upgraded to".

## Execution order

1. **pre-** scripts run before the module's own files (views, data) load.
2. Module loads (fields updated, `data`/`demo` XML applied).
3. **post-** scripts run, in version order, after this module and its
   dependencies are loaded.
4. **end-** scripts run last, after **all** modules in the transaction have
   updated.

Within a stage, scripts run in ascending parsed-version order
(`_get_migration_versions`, `migration.py:163-177`), and files within the
same version run in filename lexical order (`_get_migration_files`,
`migration.py:179-192`) — `pre-10-*.py` before `pre-20-*.py`.

`0.0.0` is special-cased to always run: first in `pre`, last in `post`/`end`
(`migration.py:170-176`), and its own `compare()` branch fires whenever the
installed version is strictly less than the target version, regardless of
what the target version actually is (`migration.py:198-200`).

A version folder only actually executes when
`installed_version < script_version <= target_version` (`compare()`,
`migration.py:198-211`) — a script whose folder version is already covered
by the installed version, or that is ahead of the module's new manifest
version, never runs. A "majorless" folder (e.g. `2.0`, no `18.0.` prefix)
compares only the module-version part, so it does **not** re-run across an
Odoo major-version upgrade even if the module version number repeats
(`migration.py:201-209`, comment at `:206-207`).

**No migration ever runs on a fresh install** — `migrate_module` returns
immediately when `state == 'to install'` (`migration.py:153`). Migrations
only fire on `-u` (upgrade) of an already-installed module.

## `noupdate` data pitfalls

- `noupdate="1"` on a `<data>` block means Odoo loads the record once, on
  first install, and **never touches it again on later `-u`** — including
  never applying your own edits to that XML file on upgrade. A migration
  script (not a data-file edit) is the only way to change a `noupdate`
  record's value for already-installed databases.
- The inverse trap, and the one that actually costs money in this
  workspace: `noupdate="False"` (or the block's default) data **is**
  reloaded on every `-u` of that module, silently overwriting whatever a
  user edited through the UI since. This is exactly why the workspace
  `CLAUDE.md` forbids touching a manifest (which forces `-u`) of a module
  you didn't otherwise change — see `coding-guidelines.md` and
  `odoo-slop-audit` C4.
- A record moved from `noupdate="1"` to `noupdate="False"` (or vice versa)
  between versions needs a migration script if the goal is to reset or
  preserve specific rows — the manifest/XML change alone does not
  retroactively touch existing rows either way for the fields not in the
  diff.

## odoo.sh behaviour (workspace-specific)

- odoo.sh runs `-u` on every module whose **files changed** in the pushed
  commit — it does not parse or compare `version` strings. Bumping only the
  version number without a matching migration script does not skip the
  upgrade, and *not* bumping the version does not skip it either.
- Every client repo that includes a shared module as a submodule runs this
  independently; a migration script placed in the shared module's
  `migrations/` ships to all of them on the next submodule bump, each
  applying independently against that repo's own installed version.
- Because `-u` re-triggers `noupdate="False"` data load, never add a
  migration script "just in case" alongside an unrelated change to the same
  module version folder — each script is real, runs in every environment
  that upgrades through that version, and cannot be un-run.
- The lab (`Versiones/V18/`) validates against canonical HEAD; it does not
  reproduce a specific client's pinned commit history, so a migration script
  that depends on "the previous version's shape of the data" should be
  tested by actually installing the old version first in a local base, not
  assumed correct because it reads clean against HEAD.
