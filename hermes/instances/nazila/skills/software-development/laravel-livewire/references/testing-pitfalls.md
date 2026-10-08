# Laravel Testing Pitfalls — Decision Table

## Resource Transformer Assertions

| Scenario | Wrong | Correct |
|---|---|---|
| Check `whenLoaded` not loaded | `expect($array['unit'])->toBeNull()` | `expect($array['unit'])->not->toBeInstanceOf(Unit::class)` or `array_key_exists` check |
| Validate ISO 8601 dates | `DateTimeImmutable::createFromFormat(ATOM, $str)` | `strtotime($str)` — flexible, handles `.000000Z` suffix |
| Assert factory nullable field | `'icon' => null` on NOT NULL column | Use column default: `'icon' => 'o-bell'` |
| Generate optional JSON data | `fake()->passthrough()` | `fake()->words(3)` — passthrough requires a value argument |

## PHPStan Patterns for Laravel/Livewire

| Scenario | Wrong | Correct |
|---|---|---|
| Access auth ID in blade component | `auth()->id()` | `Auth::id()` with `use Illuminate\Support\Facades\Auth;` |
| HasFactory on model | `use HasFactory;` alone | `/** @use HasFactory<\Database\Factories\XFactory> */` above `use HasFactory;` |
| JsonResource magic property | `$this->field` in `toArray()` | `@property-read` annotations + `$model = $this->resource;` with `@var Model $model` |
| After fixing PHPStan errors | Run `composer phpstan` once | Regenerate baseline: `vendor/bin/phpstan analyse --generate-baseline`, then verify |

## Factory Creation Checklist

1. Check the migration for NOT NULL columns — factory must not pass null.
2. Check if model has `HasFactory` trait — add it if missing.
3. UUID models: manual `Str::uuid()` in `boot()` + `HasFactory` is fine;
   do NOT add `HasUuids` trait (conflicts with boot logic).
4. After seeding explicit IDs in tests, resync Postgres sequence:
   `SELECT setval('table_id_seq', (SELECT MAX(id) FROM table))`.

## Scope & Fixture Traps

| Scenario | Wrong | Correct |
|---|---|---|
| Resolve an auth-scoped id list in `beforeEach` | create + attach a user, then read the scope | `actingAs($user)` first — a create/attach helper does **not** log in, and an `auth()`-based scope resolver returns `[]` |
| Trust a scope you just built | use it directly | `expect($scope)->not->toBeEmpty()` first — an empty scope makes every "excludes X" assertion pass for the wrong reason |
| "Unit with zero personnel" fixture | attach the user's backing person to the unit under test | the user factory creates its backing `Person` on the **first existing** unit — create the empty unit *after* the user, or assert on a unit created later |
| Query-count budget | guess a number | measure once, then set a bound with slack, and skip `BEGIN`/`COMMIT`/`ROLLBACK`/`SAVEPOINT` when counting |
| Tightened auth rule, old fixtures | assume old tests cover the new branch | fixtures with a null owner field keep passing on the deny path — add explicit allow-persona AND deny-persona tests for the new branch |

## Testing `#[Locked]` Properties (tampered-payload simulation)

`->set()` on a `#[Locked]` property throws `CannotUpdateLockedPropertyException`
in tests — the same rejection a real tampered request gets. That proves the
lock, not the server-side scope check behind it. To prove the check holds even
if the forged value reaches the component (defense in depth), tamper the
snapshot and re-sign it:

```php
use Livewire\Mechanisms\HandleComponents\Checksum;

function tamperLocked($component, string $prop, mixed $value)
{
    $snapshot = $component->snapshot;
    $snapshot['data'][$prop] = $value;
    unset($snapshot['checksum']);
    $snapshot['checksum'] = Checksum::generate($snapshot);
    $component->snapshot = $snapshot;

    return $component;
}
```

Rules for this recipe:

1. Tampering `data` without re-signing throws `CorruptComponentPayloadException`
   (checksum) — that failure tests the checksum, not your code. Always re-sign.
2. Assign via the `$component->snapshot = $snapshot` setter; mutating the live
   instance does not persist because each `->call()` replays the last snapshot.
3. Prefer the legitimate path where one exists (e.g. drive `editingId` through
   `editEvent` for the allow-persona) and reserve tampering for the attack path.
4. Adding `#[Locked]` breaks existing tests that `->set()` the property —
   migrate them to the tamper helper or the legitimate path, do not drop them.
5. Prove red-on-old: `git stash push -- <source files only>`, run the new
   tests (they must fail), `git stash pop`. A security test that never failed
   without the fix pins nothing.

## E2E Test Structure

```
tests/e2e/<feature>/<feature>.spec.ts
```

Imports from `../shared/fixtures`:
- `login(page, nCode?, password?)` — fills login form, waits for redirect
- `waitForLivewire(page)` — waits for `.wire-loading` to disappear
- `waitForToast(page, text?)` — waits for toast notification

Pattern:
1. `test.beforeEach` — login + navigate to page
2. Test: page loads (assert header, key elements visible)
3. Test: CRUD operations via Livewire modals
4. Use `page.locator('input[wire\\:model="field"]')` for Livewire inputs
5. Use `page.getByRole('button', { name: '...' })` for buttons
6. Always `waitForLivewire(page)` after actions that trigger updates
7. Prefer `expect.poll(() => locator.count(), { timeout })` over fixed sleeps for
   anything a `wire:click` changes — `waitForLivewire` only helps when the
   template actually renders `.wire-loading`.

### Assertions on seeded data

- **Never assume a filter narrows the result set.** A tree/search UI that
  *expands* its matches shows MORE nodes after filtering than the default view.
  Assert semantics instead of direction: the highlight class is present, a deep
  node that was previously hidden is now visible, clearing the term removes the
  highlights.
- **Query the seeded database for facts before choosing an assertion.** The
  suite migrates and seeds fresh every run, so tree depth, names and match
  counts are reproducible — a count-based guess is what fails:
  `psql -h 127.0.0.1 -U <user> -d <e2e_db> -tAc "SELECT count(*) FROM units WHERE name LIKE '%x%'"`
- **Match component-rendered attributes by prefix.** Wrappers rewrite what you
  pass them: a clearable input appends a trailing space to `placeholder`, so
  `input[placeholder="..."]` matches nothing while the page looks fine — use
  `input[placeholder^="..."]`.

### Runner lifecycle

- A `set -e` runner that swaps `.env` (or any config file) **skips its restore
  step when a test fails** — restore the backup, remove temp state and kill the
  dev server yourself after every run, pass or fail.
- Never `pkill -f` a pattern that appears in your own command line: the shell
  matches itself and SIGTERMs the job you are still running. Bracket one
  character — `pkill -f "port=800[1]"` matches the server but not the command
  that mentions it.