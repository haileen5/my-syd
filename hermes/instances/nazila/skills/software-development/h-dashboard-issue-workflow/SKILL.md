---
name: h-dashboard-issue-workflow
description: "Use when working h-dashboard issues (refactor/fix) here."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [h-dashboard, laravel, livewire, refactor, phpstan, pull-request]
    related_skills: [laravel-livewire, github, git-workflow, codebase-inspection]
---

# h-dashboard issue workflow (this server)

## When to Use

Use for any implementation task on the `h-dashboard` repo (an issue from the `improve` audit wave, a refactor, a bug fix): syncing, editing single-file Livewire components, running the PHPStan/test gates, and opening the PR.

Repo: `/home/runner/h-dashboard` (fork `haileen5`, working branch read from `git branch --show-current`, base = `beta` of `asgarimehdi/h-dashboard`). Read `AGENTS.md` + `docs/agents/hermes-session-rules.md` first; they are authoritative.

## 1. Before touching code

- `gh pr list --repo asgarimehdi/h-dashboard --state open` — the audit wave is worked in parallel by several agents/fork owners; confirm no PR already exists for the issue.
- Read the **issue comments**, not just the body: `gh api repos/asgarimehdi/h-dashboard/issues/<N>/comments --jq '.[] | "--- \(.user.login) \(.created_at) ---\n\(.body)"'`. For `improve`-wave issues the body is discovery history; a later kimya30 `` گیت فاز ۳ `` comment holds the delivery gate (branch from beta, `Closes #N`, which of section 4's tests are required) and supersedes body line numbers.
- The GitHub MCP has **no** `list_issue_comments`; `get_issue` returns only the count. Use `gh api`.
- Look for cross-referenced issues before widening scope: an audit body phrase like "connect to X" may point at a *separate* open issue that owns the decision (e.g. #1055 step 3 referenced #1029 for `acceptTicket`). Check that issue's plan and `gh pr list --search` for its history — a past merged PR may have deliberately decided the opposite and pinned it with a test.

## 2. Blade anonymous-class components (the big one)

Most components are single-file: a PHP anon class at the top of `resources/views/livewire/<feature>/⚡<name>.blade.php`, terminated by `};` then `?>`. PHPStan-baseline entries for these files are keyed to the **anon class identity line** in the message text, e.g. `Method Livewire\Component@anonymous/resources/views/livewire/tickets/⚡inbox.blade.php:21::forward() ...`.

- Never add or remove a line **above** `new class` (no new top-of-file import). Use a fully-qualified trait reference inside the class body: `use \App\Traits\TicketsQueryScopes;`.
- Missing-type errors keep the `Livewire\Component@anonymous/<file>:21::method()` label **even after the method moves into a trait** — the existing baseline rows keep matching. But some errors (property access, e.g. `nullsafe.neverNull`) get **re-attributed to the trait file path**. So after an extraction: `composer phpstan` first, then regenerate with `composer phpstan-baseline` and `diff` the baseline — expect path moves and removals, never a *new* error identifier. Report the net in the PR body.
- Refactoring a `\DB::` call into a namespaced trait: import `Illuminate\Support\Facades\DB` and use `DB::` — that retires an old `class.notFound` baseline row (`Call to static method transaction() on an unknown class DB`).
- Extraction target: a **trait**, not a nested component (Livewire docs: traits are for "extracting code from a single component into a dedicated file"). Public trait properties work (`App\Traits\HardwareIndexHelpers` is the in-repo precedent); keep `#[Url]` state in the component next to its `updated*` handlers and move only non-URL flow state.
- Verify the extraction is verbatim: `git show HEAD:<file>` → slice the moved method range → normalized `difflib` diff against the trait body; expect only Pint normalisations (spacing, single quotes, trailing commas).
- Compare line counts before/after (`wc -l`) — reviewers require the number in the PR.

## 3. Formatting, tests, verification

- **`vendor/bin/pint` is blocked** by the Hermes lifecycle guard (phar > 1 MiB). Use **`composer pint`** (Pint `--dirty`; the pre-commit hook does too). `.pint.json` sets `not-name: *blade.php`, so blade components are never formatted — that is why unused imports survive there and must not be "cleaned".
- **Never run two test suites concurrently**: shared Postgres test DB ⇒ `SQLSTATE[40P01]: Deadlock detected` flakes. If a run fails with deadlocks, re-run before suspecting your change (`composer test <file>` bakes in `config:clear`/`route:clear`).
- Gates: `composer phpstan` (blocking in CI), `composer verify` (preflight + pint --test + phpstan + full suite; ~10 min, run it in the background with notify), `composer test <files>` for the targeted set.
- Prove a new pin test has teeth: temporarily reintroduce the bug (e.g. put `#[Url]` back on a property), watch it go red, revert, watch it green — and state both in the PR. Add a positive control next to a negative assertion so the negative is not a dead no-op.

## 4. Commit, push, PR

- One commit per plan step, both green; `git push origin HEAD:refs/heads/<current branch>`; the pre-push hook runs full PHPStan.
- **GitHub MCP `create_pull_request` currently fails** with `MCPError: Authentication Failed: Requires authentication` while `get_issue`/`search_issues` work. Fall back to `gh pr create --repo asgarimehdi/h-dashboard --base beta --head <fork-owner>:<branch> --body-file <file>` (write the body with `write_file`, never inline in the shell — the text is Persian and multi-line).
- PR body in Persian, sections: `## چرا؟` then خلاصه / اندازهٔ فایل (قبل/بعد) / فایلهای تغییریافته / تستها و نتیجه / PHPStan baseline / چکلیست, ending with `Closes #<N>`. After opening, add a short `پایان اجرا` comment on the issue and check CI with `gh pr checks <N>` (blocking: Tests & Coverage, Code Style (Pint), PHPStan; E2E/mutation/PHP8.6 are informational).
