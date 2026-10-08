# Seeder Data Integrity

Applies when a project seeds from data files under `database/seeders/data/*.php`
or when a seed run reports rows it could not insert.

## Diagnosing "N rows skipped (not found)"

A seeder that resolves its rows by a shared key (commonly `n_code`) skips
silently on a bad key instead of throwing. The message is a lookup miss in the
seeder's own data, so **diff the key sets of the two data files before touching
the seeder or the database.**

1. Read the seeder and find its lookup line (e.g.
   `DB::table('hardwares')->where('n_code', $nCode)->first()`) and the file it
   loads from `__DIR__.'/data/...'`.
2. Parse both data files and diff the key sets with PHP rather than grepping
   hundreds of rows:

   ```
   php -r '$a = require "database/seeders/data/child.php"; echo json_encode(array_keys($a));'
   php -r '$b = require "database/seeders/data/parent.php"; echo json_encode(array_map(fn($r)=>(string)$r["n_code"], $b));'
   ```

3. Cross-check the human label. Data files usually carry a comment above each
   key (`// ── AB-17SH-P3-1 ──`). If the comment label does not match the parent
   row for that key, one of the two files is wrong; the comment is the weaker
   evidence, so the parent data file wins the key.
4. Fix by correcting the key in the child data file — unless the parent genuinely
   lacks the row, in which case the parent is incomplete.

**Compare codes as strings.** `n_code` is `varchar(10)`; values like
`0885727024` keep a leading zero that integer comparison silently drops, which
manufactures false "found" and "missing" results.

## Reading column names cheaply

Hardware-style tables rarely have a `name` column; descriptive fields are
`pc_name`, `cpu`, `ram`, `hdd`. Guessing a column costs a round trip on
`ERROR: column "name" does not exist` — read the migration or `\d <table>` once
instead.

## `migrate:fresh --seed` safety

`migrate:fresh` drops every table and is not reversible. Before running:

1. Read the target from `.env` (`DB_CONNECTION`, `DB_DATABASE`) — the file may
   have been swapped by an E2E run.
2. Snapshot live row counts and quote them in the confirmation prompt.
3. Run with `--force` so the safety prompt does not block a non-interactive run.
4. Compare row counts after the seed. Identical counts mean the seeders rebuilt
   the same reference data; lower counts mean previously manual data is gone.

Read `DatabaseSeeder::run()` for the authoritative call order. Seeders that
insert explicit IDs need their sequences reset afterwards (PostgreSQL:
`setval` per `*_id_seq`) or later manual inserts hit duplicate-key errors.

## Counting rows in PostgreSQL

```bash
PGPASSWORD=$(grep -E '^DB_PASSWORD=' .env | cut -d= -f2-) \
  psql -h 127.0.0.1 -U <user> -d <db> -c \
  "select relname, n_live_tup from pg_stat_user_tables where n_live_tup > 0 order by n_live_tup desc"
```

`n_live_tup` is an estimate — good for spotting an order-of-magnitude drop after
a fresh seed, not for exact assertions. Use `count(*)` when the exact number
matters.
