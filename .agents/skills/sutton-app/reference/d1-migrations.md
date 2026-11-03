# Database migrations

Digit applies `backend/migrations/*.sql` during a platform deployment — not when the app
handles requests, not during local dev, and not as a separate manual step. A draft deploy
applies migrations to the draft D1. Promotion applies the build's pending migrations to the
published D1; a draft deploy never migrates published data.

## Workflow

1. Add SQL files to `src/backend/migrations/` (flat directory; no subfolders)
2. Name each file with a zero-padded number and short description:
   `0001_init.sql`, `0002_add_column.sql`, `0003_add_index.sql`
3. Declare a `database` binding in `manifest.json`
4. Run `npm run pack` and deploy a draft or publish to the published app

**Completing the platform deployment is what runs migrations.** Uploading a zip without
completing the deployment does not apply any SQL.

## How Digit applies them

- Files run in **filename order** (lexicographic — zero-padding keeps order correct)
- Digit records which filenames have already been applied separately in each D1 and **skips**
  them on later deployments to that channel
- The first draft deploy and the first published deployment each run every migration file for
  their own database
- Later deployments run **only new files** that were not applied in that database

## Rules

- **Never edit or delete a migration file that has already been applied** on a channel. Digit
  will not re-run it in that D1. To change the schema or fix a bad migration, **add a new
  forward file** with the next number (`0003_…`, `0004_…`). Do not write down or undo SQL
  files. If a migration was edited while developing only in draft, reset/replay the draft
  through the Sutton UI rather than relying on the filename ledger to notice the edit.
- **Always use the naming convention** — zero-padded numeric prefix plus a short snake_case
  description: `0001_init.sql`, not `1_init.sql` or `init.sql`.
- Keep each file **small and focused** — one logical change per file when possible.
- Migrations are **additive and forward-only**. Rollback runs older code against the newer
  schema. See [Additive-only migrations](#additive-only-migrations).
- If a published-app migration cannot capture a D1 Time Travel bookmark, promotion **fails
  closed** and does not migrate — see [publish.md](publish.md). Digit does not auto-restore
  from that bookmark if a later step fails.
- If a migration fails partway through, fix the SQL and **publish again**. Digit retries
  files that have not yet been recorded as successfully applied. Once a file is recorded as
  applied, leave it alone and add a new forward file instead.

## Additive-only migrations

App D1 migrations are forward-only. There are no down or undo files. Rolling back an app
version runs older code against the newer schema, so every applied migration has to keep
working for any version you might roll back to.

**Allowed:** new tables, new columns that are nullable or have a default, new non-unique
indexes, and backfills that only fill new columns. Write `INSERT` statements with an
explicit column list so later added columns don't break them.

**Not allowed** while any version you might roll back to still uses the old shape:
`DROP TABLE`, `ALTER TABLE ... DROP COLUMN`, `RENAME` of tables or columns, new `UNIQUE`
indexes, and backfills that rewrite values older code reads. Do not rely on publish to catch
these. Check every new migration yourself before you publish.

**Fix a bad migration** by writing a new forward migration. Never edit or delete an applied
migration file.

**Clean up later (contract step):** drop unused columns or tables in a later migration, after
the current code has stopped reading and writing them. Ask the user first: once the drop is
applied, rolling back to any version that still uses them will break.

**SQLite:** there is no `ALTER COLUMN`. Type or constraint changes need a table rebuild,
which drops and renames tables, so it is **not allowed** either. Add a new column instead.

Good — add a nullable column the previous version can ignore:

```sql
-- 0002_add_notes.sql
ALTER TABLE items ADD COLUMN notes TEXT;
```

Bad — drop a column the previous version still reads. Rollback would run that older code
against a schema that no longer has `title`:

```sql
-- 0003_drop_title.sql
ALTER TABLE items DROP COLUMN title;
```

## What does not run migrations

- Local dev in this starter repo (`npm run dev`)
- Backend code at runtime (no auto-migrate on requests)
- Config-only updates (env vars / secrets sync without a full deployment)
