# GitHub Issue to Pull Request

Turn a GitHub issue into a tested, verified PR. This skill owns the end-to-end discipline — premise validation, duplicate sweeps, class-level fixes, and honest CI reporting; the sibling GitHub and development skills own their own mechanics.

## When to Use

- "Fix issue #123 and open a PR."
- "Implement this GitHub feature request."
- "Take this bug from issue to green CI."

Don't use for: reviewing an existing PR, or answering a code question with no requested change.

## Procedure

### 1. Read the live issue — body AND full thread

Use `terminal` to run `gh issue view <N> --comments`. The body is a snapshot from filing time; the newest comments carry the live state: partial fixes already merged, new root-cause analyses, maintainer decisions, or questions directed at you that change the task. Also read repository instructions (`AGENTS.md`, contribution docs) with `read_file`. Done when the currently requested behavior, non-goals, and any unanswered thread questions are known.

### 2. Sweep for existing and duplicate work

Before writing anything, run `gh pr list --search "#<N>" --state all` plus at least two keyword/synonym variants of the symptom (`gh pr list --search "<subsystem> <symptom>" --state open`). Popular issues attract multiple independent fixes; building a duplicate wastes the work and the credit. Also check whether a recent commit already fixed it: `git log --oneline -20 -- <relevant files>`. Done when you know every open PR and recent commit touching this issue, or that none exist.

### 3. Validate the premise against current code — and against design intent

Reproduce the bug or demonstrate the missing behavior on the current default branch with a failing test or fixture, using `search_files` and `read_file` to trace the reported path. Then check the second question: is the "bug" actually deliberate design? Run `git log -p -S "<symbol>"` on the code the issue wants changed and read the original commit's intent — a missing link or restriction is often the feature. Challenge stale or flawed issue prose instead of implementing it blindly. Done when the root cause or feature gap is demonstrated in current code AND the change doesn't fight an intentional design.

### 4. Define acceptance and risk

List acceptance criteria, interfaces, migrations/state changes, compatibility, security/privacy, rollout, and rollback. Map every criterion to a test or explicit verification. Done when review has a finite contract.

### 5. Implement the smallest complete change — and fix the class

Work on an isolated branch or worktree, loading `systematic-debugging` or `test-driven-development` when the bug class calls for them.

If the base branch has no configured remote under `origin` (another repo's branch), fetch it by URL in one step — `git fetch https://github.com/<org>/<repo>.git <branch>` and branch off `FETCH_HEAD`; never remap `origin` and never guess the URL. Before branching, sweep the working tree: stash unrelated local drift ( lockfile churn from stray installs, scratch files) — `git checkout -b` carries the working tree, so an unswept diff rides into the scoped PR. If the scoped task forbids lockfile churn or specific files, drop them before branching. Add regression tests first, then implement. When the fix is in hand, `search_files` for the same bug shape at sibling call sites and fix the whole class in this PR — an incomplete fix that leaves known siblings broken is worse than none. Every changed line must trace to the issue; no drive-by cleanup. Done when targeted tests pass, the original failure no longer reproduces, and sibling sites are fixed or explicitly ruled out.

### 6. Prove the regression test bites (sabotage run)

Temporarily restore the old behavior of the exact function under test, run the new test, and confirm it FAILS; then restore the fix and confirm it passes. A regression test that passes with and without the fix proves nothing. Done when the test demonstrably fails on pre-fix code.

### 7. Run repository quality gates, then open the PR immediately

Run the formatter, lint, typecheck, and the repo's canonical test entrypoint on affected areas; use `requesting-code-review` on the diff. Then push and open the PR right away — the PR is what dispatches CI, and CI latency is the long pole; do not sit on finished work. Load `github-pr-workflow` for PR mechanics: conventional branch/commit, body linking the issue with problem, approach, tests, risk, and exclusions. Read the PR back and verify head SHA, base, title, and files. Done when the PR exists with the intended diff and CI is running.

### 8. Shepherd CI honestly and close the loop

Inspect live checks and failure logs via `gh pr checks` / `gh run view --log-failed`. Distinguish failures introduced by your diff from pre-existing baseline or infrastructure failures — reproduce on the default branch when unsure, and rerun once only for genuine infra flakes. Never say "green," "merged," or "released" without live evidence of that exact state. When the PR lands, comment on the issue with the PR link and a one-line explanation so the reporter gets a traceable resolution. Done when CI state, remaining blockers, and the issue thread all reflect reality.

## Variant: documentation-only issues (stale counts, dead references, false gotchas)

Same loop, different premise check (step 3): re-verify every claim against the branch you will PR from before touching the text. Does the referenced file exist, is the symbol still live, does the stated count reproduce. A claim that no longer reproduces is reported in the PR body, not silently rewritten or "fixed" on faith. Two hard gates on the diff:

- Every refreshed number is re-derived with the command the document itself names for counting (or the count and its "verified" stamp get dropped); the derivation is recorded in the PR body. A fresh number not produced by that command is the same defect as the stale one.
- Encoding and marker gates: if the issue declares a byte-level constraint (forbidden codepoints like U+200C, no conflict markers), prove before == after with a byte-level count of those bytes in the PR body, captured BEFORE the first edit.

The diff is docs-only: any change to a code file or a package lockfile is out of scope, even when it looks required to run the suite the doc describes.

## Pitfalls

- Coding before reading issue comments, sweeping for duplicate PRs, or reading current code.
- "Fixing" behavior that the original commit shows is intentional design.
- Fixing a symptom at one call site while sibling sites keep the same bug.
- Shipping a regression test that also passes without the fix.
- Opening a PR with unrun tests or unrelated formatting churn.
- Claiming the issue is delivered because a PR exists.
- Swept-before-branching skipped: unrelated working-tree drift rides into the scoped PR diff.
- Branching from a foreign repo's base by remapping `origin` or guessing the URL instead of one-shot `git fetch <url> <branch>` + `FETCH_HEAD`.
- A docs-only fix ships a refreshed number not re-derived with the doc's own command, or an edit that breaches a stated encoding/marker gate without a before-count in the PR body.
- A new formatted status report mentions PR opened, review requested, or CI green with no fresh `gh` read proving each state at that moment; when the session is cut mid-task, report the verified partial state and stop.

## Verification

- [ ] Full issue thread read; newest comment state reflected in the plan.
- [ ] Duplicate-PR sweep run with issue number + 2 keyword variants.
- [ ] Premise reproduced on current code; design intent checked via git history.
- [ ] Regression test proven to fail without the fix.
- [ ] Sibling call sites fixed or explicitly ruled out.
- [ ] Every changed line traces to the issue.
- [ ] CI state reported from live evidence only; issue commented with the PR link.
