---
name: h-dashboard-e2e-env
description: "Use when running h-dashboard E2E tests."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [h-dashboard, e2e, environment, testing]
    category: software-development
---

# h-dashboard E2E Environment Rules

## When to Use

Use this when running `scripts/e2e-test.sh` on h-dashboard, when `.env` is
missing or was left in the E2E state by an earlier run, or when an
`artisan serve` process needs to be cleaned up. Also read it before any task
that says "run the e2e suite" on this box.

## Concurrency

`scripts/e2e-test.sh` is NOT concurrency-safe. Never run two at once. It also
swaps `.env` to a backup file, so a previous run may have left `.env` in the E2E
state. After every run, verify:

```bash
grep DB_DATABASE .env   # must equal: h_dashboard
```

## Rebuilding a lost `.env`

`.env` is gitignored, so it cannot be restored from git. Rebuild it:

1. `cp .env.e2e .env`
2. Set `APP_URL=http://127.0.0.1:8000`
3. Set `DB_DATABASE=h_dashboard`
4. Drop every `secrets.` line, then pull the actual secret values out of
   `.env.e2e` — copy them, never invent them.
5. Verify with `php artisan about --only=environment`.

## `parse_ini_file` pitfall

`parse_ini_file()` fails on this `.env` (unquoted parentheses in a value). Do
not use it to read config. Scan with a regex instead, or go through
`config()` from inside Laravel.

## Orphaned servers

`php artisan serve` on port 8000 is shared with other work on this box. Kill
only your own instance:

```bash
kill $(pgrep -f 'artisan serve --port=800[1]')
```

Never `pkill -f 'artisan serve'` — that kills the shared port-8000 server too.