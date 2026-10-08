---
name: laravel-livewire
description: "Laravel 13 + Livewire 4 conventions, pitfalls, testing."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [laravel, livewire, php, pest, testing]
    category: software-development
---

# Laravel 13 + Livewire 4 Development

## When to Use

Use this skill when writing PHP/Laravel code, Livewire components, Pest tests,
API Resource transformers, or E2E tests for a Laravel 13 + Livewire 4 project.

Standing conventions and pitfalls for Laravel/Livewire projects. Applies to
PHP code changes, test writing, and API resource transformers.

## Always-On Rules

1. **Run Pint before commit.** `vendor/bin/pint --dirty` is enforced in CI.
2. **Clear config+route cache before running tests.** Stale `routes-v7.php`
   causes Livewire endpoint-hash mismatch — tests silently return 404 on
   `->set()`/`->call()`.
3. **Use `assertDatabaseHas` with all relevant fields.** Don't just assert on
   `title` — include `user_id`, `unit_id`, or any field the code was supposed
   to set. Partial assertions miss bugs.
4. **Test Resource transformers directly.** Don't rely solely on controller
   tests to cover API response shape — instantiate the Resource class and
   call `toArray(new Request())` to assert exact field contracts.
5. **E2E tests go in `tests/e2e/<feature>/`** with `.spec.ts` extension.
   Import from `../shared/fixtures` for `login`, `waitForLivewire`, etc.
6. **Regenerate PHPStan baseline after fixing errors.** When you fix errors
   that exist in `phpstan-baseline.neon`, the old entries become unmatched
   and PHPStan reports new errors. Run `vendor/bin/phpstan analyse
   --generate-baseline` after every fix round, then verify with
   `composer phpstan`.

## Pitfalls

- **`whenLoaded` returns `MissingValue`, not `null`.** When calling
  `toArray()` directly on a JsonResource (not through `toResponse()`),
  `whenLoaded('relation')` returns a `MissingValue` object. Tests must
  check `instanceof` or use `array_key_exists` — `assertNull()` fails.
- **Factory defaults ≠ nullable columns.** A migration with
  `->default('o-bell')` on a NOT NULL column means the factory must NOT
  pass `null` for that field. Use the default value, not `null`.
- **Faker `passthrough()` requires an argument.** Use
  `fake()->optional(0.6)->words(3)` or another generator for optional
  JSON fields — `passthrough()` needs a value parameter.
- **Carbon `toISOString()` ≠ ATOM format.** `toISOString()` returns
  `.000000Z` suffix; ATOM uses timezone offset. Use `strtotime()` for
  flexible ISO 8601 validation in tests.
- **UUID primary key models + HasFactory.** Models with manual UUID
  generation in `boot()` (via `Str::uuid()`) work with `HasFactory`.
  Don't add `HasUuids` trait — it would conflict with the manual boot
  logic.
- **PHPStan `auth()->id()` vs `Auth::id()` in Livewire blade components.**
  PHPStan types `auth()` as `Illuminate\Contracts\Auth\Factory` which
  lacks `id()`. In anonymous-class Livewire blade components (single-file
  components with `return new class extends Component`), add
  `use Illuminate\Support\Facades\Auth;` and call `Auth::id()` instead.
  `auth()->user()` works fine (returns User|null), but `auth()->id()`
  does not.
- **PHPStan generic type for HasFactory.** PHPStan level 6+ requires the
  generic type annotation on `use HasFactory`. Write
  `/** @use HasFactory<\Database\Factories\YourFactory> */` immediately
  above the `use HasFactory;` statement. Without it, PHPStan reports
  `missingType.generics`.
- **PHPStan `@property-read` on JsonResource.** When a Resource class
  accesses `$this->some_field` (magic proxied from the underlying model),
  PHPStan reports `property.notFound`. Fix: add `@property-read` PHPDoc
  annotations for every accessed field, then access via
  `$model = $this->resource;` with a `@var Model $model` cast. Match
  the pattern used by other Resources in the project (see
  `NotificationResource.php` for the reference implementation).
- **PHPStan without larastan degrades an Eloquent chain to `Query\Builder`
  after `whereIn()`.** `Eloquent\Builder` declares `@mixin Query\Builder`, so
  the first call PHPStan resolves through the mixin types the receiver as
  `Query\Builder` and any later Eloquent-only call fails with
  `Call to an undefined method Illuminate\Database\Query\Builder::with()/withCount()`.
  **Fix (verified — clears the error with zero baseline entries): end every
  chain on a call that `Eloquent\Builder` defines itself.** Put eager loads
  first, then move the scope filters into a trailing closure —
  `->where(function ($q) use ($ids) { $q->whereIn(...); })` — or filter by key
  with `whereKey($ids)` instead of `whereIn('id', $ids)`. `where()` and
  `whereKey()` are declared on the Eloquent builder with `@return $this`, so the
  body returns an Eloquent builder and `return.type` disappears; callers already
  honour the declared `@return Builder<Model>`. Do NOT reach for an inline
  `@var`/`assert()` to override the inferred type — PHPStan rejects that
  explicitly. Only a chain that must end on a mixin-only call
  (`orderBy()`/`limit()` before `get()`) still reports; that residual goes into
  the regenerated baseline (Always-On rule 6). The root fix is larastan, which
  this project does not run.
- **`updateOrCreate` overwrites ownership on edit.** When using
  `Model::updateOrCreate(['id' => $editingId], [...])` and one field
  (e.g. `user_id`, `created_by`) should only be set on create — not on
  update — do NOT include it in the attributes array. The update path
  would overwrite the original value. Instead, omit the field from
  `updateOrCreate`, then conditionally set it after:
  ```php
  $model = Model::updateOrCreate(['id' => $editingId], [...]);
  if (! $editingId) {
      $model->update(['user_id' => Auth::id()]);
  }
  ```
- **Verify the environment before `migrate:fresh --seed`.** It drops every table, so confirm the target is a local/dev database first — check `APP_ENV`, `DB_CONNECTION`/`DB_HOST`/`DB_DATABASE` in `.env` plus `php artisan db:show` (connection, database, table count) before running. Never run it against a shared or production database.
- **Trace skipped seeder rows to the data file, not the log line.** Idempotent seeders skip rows whose referenced record was never seeded and report only a count, so re-running the seed never names the missing key. Load the data file's keys and probe the target table per key (e.g. `where(...)->exists()`) to identify the exact missing record.
- **Route gates do not cover Livewire writes on union-gated pages.** A page behind a read-union gate (`role_or_permission:read_perm|write_perm`) that also mutates must call `$this->authorize('<write-permission>')` at the top of every mutating method — the route admits readers the mutation must refuse. Org/unit scoping (`accessibleUnitIds()`) constrains *which* unit, never *whether* the caller may write; it is not a permission check.
- **`Model::find($id)` in a Livewire method without a parent-membership check is a cross-entity IDOR.** Livewire methods are directly invocable with arbitrary IDs, so verify the record belongs to the loaded parent (e.g. `$comment->ticket_id === $this->ticket->id`) and passes scope before any ownership/role check.
- **A registered policy that nothing calls is dead access control.** Before trusting a policy mapping, grep for `authorize(` callers of it; and read the policy method body before wiring to it — `authorize('create')` against a `create()` that returns unconditional `true` gates nothing.
- **Pin both permission personas for gated mutations.** The authorized persona succeeds; the read-only persona is refused with no row changed. A happy-path-only test passes while the gate is missing entirely.
- **Model `@property` types must match column nullability.** `@property int $x` on a nullable column makes PHPStan report `identical.alwaysFalse` on `$x === null` — fix the docblock to `int|null` (confirm against the migration), never loosen the comparison to silence it.
- **Baseline entries for single-file blade components embed the anonymous-class line.** Adding even one `use` line shifts the class line (`:12` → `:13`) and unmatches every entry for that file — regenerate the baseline and confirm the diff shows only line shifts and expected count changes, never new suppressions.

## Extracting a Reusable Nested Component

When one single-file Livewire component holds two responsibilities, split it
into a child component and a thin parent page:

1. Move the mechanics **verbatim** into `resources/views/livewire/<ns>/<name>.blade.php`.
   The child owns state and methods; the parent keeps access checks, detail
   panels and page chrome (header, theme selector).
2. Child → parent: `$this->dispatch('event', id: $id)` in the child,
   `#[On('event')] public function handler(int $id)` on the parent. Dispatched
   **named** arguments must match the listener's parameter names.
3. Per-node render hooks: pass a Blade view name plus a data array down as
   props and `@include($view, ['unit' => $unit, 'data' => $data])` in the node
   partial. Never let the child query page-specific data per node — that is the
   N+1 trap the extraction exists to remove.
4. Retarget mechanics tests to the child; keep auth/panel/render tests on the
   parent.

Test contract after the split:
- Parent `assertSee()` **does** include child-rendered HTML (children render
  inline), so page-level render assertions keep working.
- Parent `->get('prop')` / `->call('method')` do **not** reach the child — assert
  child state with `Livewire::test('child', $props)` and drive the parent through
  the event: `Livewire::test('parent')->dispatch('event', id: ...)`.
- Cover both halves of the wiring: `->assertDispatched(...)` on the child proves
  it fires, `->dispatch(...)` on the parent proves the listener runs. Asserting
  only the parent's handler by direct `->call()` passes even when the event is
  never dispatched.
- Batch level-wise loads (`whereIn('parent_id', $ids)`) rather than one query
  per node, and pin the result with a measured query-count assertion so the N+1
  cannot come back.

## Merge Conflicts in Auto-Generated Files

When a PR has merge conflicts with `upstream/beta` in auto-generated files
(like `phpstan-baseline.neon`), do NOT manually merge the conflict markers.
These files are machine-generated — manual merge produces invalid output.

**Procedure:**
1. `git fetch upstream beta && git merge upstream/beta`
2. For the conflicted auto-generated file: `git checkout --theirs <file>`
   (take upstream's version as starting point)
3. `git add <file>`
4. Regenerate from scratch: `vendor/bin/phpstan analyse --no-progress --generate-baseline`
5. Verify: `composer phpstan`
6. `git add <file> && git commit --no-edit`
7. `git push origin <branch>`

**Never** edit phpstan-baseline.neon by hand to resolve conflicts.
The regenerate step produces the correct baseline for the current code state.

## Map / Heavy-Visualization Performance

On h-dashboard the `/map` page was server-thought slow, but measurement showed
the bottleneck was **main-thread rendering, not the server**: a 620ms longtask
during map pan dropped to 52ms once rendering was fixed.

When a Livewire page feels sluggish, record a longtask/perf trace before
touching the query. If a single long task dominates, the win is in render work,
not more indexes or fewer `SELECT`s. What fixed it: `circleMarker` instead of
DOM markers, lazy popup rendering, an icon cache, `Map`-keyed memo with shallow
depth, canvas-rendered lines, and deleting a dead `loadStats` Livewire load.

## MaryUI `x-select` in Blade Components

MaryUI's `<x-select>` defaults to `optionValue="id"` / `optionLabel="name"`.
When the options array is keyed `value`/`label`, every `<option>` renders empty
and the control looks blank — pass `option-value="value" option-label="label"`
explicitly.

In a single-file Livewire blade component, a bare `$myOptions` in the Blade view
is an undefined variable. Pass it from a component method:
`:options="$this->myOptions()"`.

## Testing Patterns

See `references/testing-pitfalls.md` for the full decision table on
assertion patterns, factory creation, scope/fixture traps, and E2E test
structure.