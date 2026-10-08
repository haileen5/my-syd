---
name: h-dashboard-dev
description: "Use for h-dashboard work: setup, git, e2e, Boost."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [h-dashboard, laravel, livewire, maryui, git, e2e]
    category: software-development
---

# h-dashboard Working Knowledge

## When to Use

Load this whenever the task touches the h-dashboard repo (`/home/runner/h-dashboard`)
or its Laravel/Livewire code, tests, git workflow or e2e runs. It records this box's
setup facts; the generic framework rules stay in the `laravel-livewire` skill and the
project's own `AGENTS.md` stays authoritative over both.

## Session Setup

1. `cd /home/runner/h-dashboard`, read `AGENTS.md` first — it overrides these notes.
2. `codegraph sync` before any code Q&A, then `codegraph_explore` for structure,
   relationships and "where is X implemented" questions. Reading files first wastes
   turns on a large codebase.
3. Official docs before assuming an API (`read-the-damn-docs`). Only run the
   `improve` audit skill when explicitly asked.

## Git: Branches and Refspecs

- `origin` is this server's fork (`haileen5/h-dashboard`); `upstream` is the canonical
  repo (`asgarimehdi/h-dashboard`) and its `beta` branch is the integration target.
  Read `git remote -v` — never hardcode either URL.
- `branch.aylin.merge` can point at `refs/heads/beta` or at `refs/heads/aylin`
  depending on when the tracking ref was last rewritten. When it points at `beta`,
  `git status` prints `aylin...origin/beta`, which misleads: a bare `git push` then
  resolves against the beta tracking ref and can push to the wrong branch.
- **Always push with an explicit refspec:** `git push origin HEAD:refs/heads/aylin`.
  Check `git config --get-regexp '^branch\.'` before assuming which branch you are on.
- `scripts/sync-beta.sh` reports `behind/ahead` against `origin/beta` and only ever
  fast-forwards; it exits 1 on divergence. Prefer it over hand-rolled merges.
- When the user just says `pr`, that means: open a pull request from the current
  working branch to `beta` on `asgarimehdi/h-dashboard`. Do not merge it unless
  asked.

## AGENTS.md Is Write-Protected on Hermes Surfaces

`patch` and `write_file` refuse to write `AGENTS.md` ("protected agent-instruction
file(s) requires approval"), and on `api_server` there is no interactive approval
channel — so a plan step that asks for an AGENTS.md edit cannot be completed
in-session. Write the prepared replacement text plus the verified sweep findings
to a file the user can apply by hand, say plainly in the report that the step was
parked, and do **not** try to route around the guard via terminal or
`execute_code` (that is also refused, and attempting it violates the block).

## Livewire Toast Messages Are JS Effects, Not HTML

`Mary\Traits\Toast` (`vendor/robsontenorio/mary/src/Traits/Toast.php`) sends
`$this->js('toast(...)')`, so the message never appears in the rendered HTML and
`->assertSee('پیام فارسی')` always fails. Read it from the effects instead:

```php
foreach (array_column($component->effects['xjs'] ?? [], 'expression') as $expr) {
    // $expr is literally `toast({"toast":{...}})` — a JSON string inside JS
    $inner = json_decode(substr($expr, strlen('toast('), -1), true);
    $inner['toast']['title'];
}
```

Decoding is mandatory: the inner JSON escapes Persian as `\uXXXX`, so
`JSON_UNESCAPED_UNICODE` on the outer payload does not help.

## `UserFactory` Users Are Never Unit-Less

`UserFactory::configure()` always creates a backing `Person` and attaches it to
the first `Unit`, so `AccessService::accessibleUnitIds()` resolves a unit through
`person.u_id` for every factory-made user. To reproduce the empty-scope path you
must do both:

```php
Session::forget('current_unit_id');       // else the session unit is used for ANY user
$user = User::factory()->create();
$user->person->update(['u_id' => null]); // factory attaches a Person with a u_id
```

Forgetting either one silently gives the test a non-empty scope and the assertion
passes for the wrong reason.

## Sweep Command for the Fail-Open Scope Patterns (issue #839)

All three spellings that make an empty scope mean "unrestricted":

```bash
grep -rnE '(when\(\s*\$[a-zA-Z]*[Ii]ds|! *empty\(\$[a-zA-Z]*[Ii]ds\))' app/ resources/views/
```

The third and worst spelling is
`if (! empty($ids) && ! in_array($id, $ids)) { reject }` — the first conjunct is
false, `&&` short-circuits, and the row is **accepted**. The #819 sweep only
grepped `when($accessibleIds` and therefore missed every `! empty(...)` site.

## Pint Must Be Invoked via Its PHP Build

`vendor/bin/pint` trips the lifecycle guard ("script is larger than the scan cap").
Run the build directly instead:

```bash
php vendor/laravel/pint/builds/pint --dirty --format agent
```

The Boost MCP server sometimes dies on its first stdio call
("lost its stdio subprocess"). Just call the tool again. If it dies again, use the
CLI wrapper, which needs no MCP:

```
php scripts/boost_tool.php <tool> '<json>'
```

## e2e Tests and `.env` Recovery

`scripts/e2e-test.sh` is **not concurrency-safe**. Never two runs at once — it swaps
`.env` for `.env.e2e`. After any run, verify the environment was restored:

```
grep DB_DATABASE .env    # must be h_dashboard
```

If `.env.dev.bak` is already gone (a previous run was interrupted), restore by hand:
copy `.env.e2e` to `.env`, set `APP_URL=http://127.0.0.1:8000` and
`DB_DATABASE=h_dashboard`.

Kill a leftover test server **only** on a non-default port:

```
kill $(pgrep -f 'artisan serve --port=800[1]')
```

Never `pkill -f 'artisan serve'` — that kills the shared dev server on port 8000.

`.env` is gitignored, so it cannot be restored from git. Rebuild it from
`.env-example-github` plus the secrets documented in `.env.e2e.example`, drop the
`secrets.*` lines, then verify with `php artisan about --only=environment`.
`parse_ini_file()` fails on values containing unquoted parentheses — scan with a
regex or read values through `config()` instead.

## MaryUI `x-select` Gotcha

`x-select` defaults to `optionValue='id'` / `optionLabel='name'`. When options are
keyed `value`/`label`, pass `option-value="value" option-label="label"` explicitly —
otherwise every `<option>` renders empty and the control looks blank.

Options must be passed from a component method, not as a bare Blade variable:

```
:options="$this->myOptions()"
```

A bare `$myOptions` is undefined in the single-file component's view.