---
name: git-workflow
description: "Git pitfalls: nested repos, staging, identity."
version: 1.0.0
author: Sydney
license: MIT
metadata:
  hermes:
    tags: [git, workflow, pitfalls, staging, nested-repos, commit]
    category: software-development
---

# Git Workflow

Pitfalls and procedures for everyday git operations that fall outside standard
`gh` CLI workflows (for gh-specific flows see the `github` skill).

## Standing Rules

- Always check `git status` and `git remote -v` before staging or pushing.
- Configure per-repo identity before first commit if global config is absent.
- Never assume a branch name — read it from `git branch --show-current`.

## Pitfalls

### Nested `.git` directories when copying content

When `cp -r` (or similar) a directory that contains its own `.git` into another repo,
`git add` detects the nested `.git` and stages only a submodule reference (mode 160000),
not the actual files. Clones of the outer repo will not contain the copied content.

**Fix — remove nested `.git` before staging:**

```bash
cp -r /source/dir target/inside/repo
rm -rf target/inside/repo/.git
git add target/inside/repo/
```

If already staged as submodule:

```bash
git rm -r --cached target/inside/repo/
rm -rf target/inside/repo/.git
git add target/inside/repo/
git commit -m "Add dir contents (fix nested repo)"
```

### Unintended file changes in feature commits

When committing feature work, run `git diff --stat` before staging to catch
unintended modifications to config/meta files (e.g. `AGENTS.md`, `.hermes.md`)
that were modified by tooling or agents during the session but are not part of
the feature.

**Prevention:** `git add -p` or selective `git add <paths>` instead of
`git add -A`. Review `git diff --staged --stat` before committing.

**Fix — restore from upstream before amend:**

```bash
git show upstream/beta:AGENTS.md > AGENTS.md
git add AGENTS.md
git commit --amend --no-edit
```

### Reading files from a branch the checkout does not have

When a task requires judging the current state of code (issue audits, fix
verification, comparing branches), the working tree often LAGS the branch that
carries the work. Grepping the checkout then yields a confident verdict on
stale code. Do not `git checkout` the other branch just to read it — projects
may forbid switching branches, and it disturbs session state.

**Fix — fetch the branch as an explicit remote-tracking ref and read through it:**

```bash
git fetch <remote> <branch>:refs/remotes/<remote>/<branch>   # e.g. beta
git rev-list --left-right --count HEAD...<remote>/<branch>   # behind X, ahead Y
git show <ref>:path/to/file            # one file
git grep -n 'pattern' <ref> -- path/   # search the branch's tree
git diff --stat HEAD <ref> -- path     # what changed vs checkout
```

Run these as small batches with explicit timeouts — one oversized shell call
dying takes every command's output down with it.

### Missing git identity on fresh clones

New clones may lack both global and per-repo `user.name`/`user.email`.
Commits fail with `empty ident name`.

**Fix — set per-repo config before first commit:**

```bash
git config user.email "user@users.noreply.github.com"
git config user.name "username"
```

Detect from `gh auth status` or set manually.

### Local branch tracking a differently-named upstream

A local branch can track a different remote branch name (e.g. local `nazila` tracks `origin/beta`), so `git status` reporting 'up to date' says nothing about whether the same-named branch exists on the remote. A bare `git push` follows the upstream mapping and can target the wrong branch.

**Fix — verify the remote branch exists, then push an explicit refspec:**

```bash
git branch -vv                                        # see what local tracks
git config --get branch.<name>.merge                  # tracked ref, e.g. refs/heads/beta
git branch -r                                         # remote branches the fetch knows
git ls-remote --heads <remote>                        # authoritative remote state
git push <remote> HEAD:refs/heads/<local-branch>      # create/update same-named remote branch only
git ls-remote --heads <remote> | grep <local-branch>  # verify it landed
```

Never push to the tracked branch name (`beta`, `main`) unless that is the intent — the refspec above leaves it untouched.

### Tracked dependency lockfiles block the sync merge

`composer install` / `npm install` on the box rewrite the *tracked* lockfiles with
resolved-version bumps the repo never committed. The sync merge then refuses to
start at all:

```
error: Your local changes to the following files would be overwritten by merge:
        package-lock.json
Aborting
```

Prove the drift is install churn before discarding it — compare the working copy
against the ref you are about to merge:

```bash
git diff --stat composer.lock package-lock.json
git show <ref>:composer.lock | grep -A2 '"name": "laravel/framework"'
grep -A2 '"name": "laravel/framework"' composer.lock
```

If the working copy is merely *newer* than every committed version, it is churn,
not work:

```bash
git checkout -- composer.lock package-lock.json   # or: git stash push -- <those files>
git merge --ff-only <ref>
```

Discard rather than commit: the committed lockfiles are the CI contract, and a
box-local resolve pins versions nobody else can reproduce. Re-run
`git status --porcelain` after the post-merge installs — they dirty the lockfiles
again, and a dirty tree reads as unfinished work.

### Fast-forwarding a fork's stale mirror branch

When session rules require syncing the canonical `beta` into the fork's own
`beta`, fetch it to an explicit ref and push **that ref** to the mirror branch:

```bash
git fetch <canonical-url> beta:refs/remotes/canonical/beta
git rev-list --left-right --count HEAD...canonical/beta   # 0 ahead = pure-behind
git merge --ff-only canonical/beta
git push <origin> canonical/beta:refs/heads/beta          # mirror update
```

Git refuses a non-fast-forward push without `--force`, so this can only *advance*
the mirror — it never loses fork-side commits. That is why it is legitimate where
pushing your own working branch onto `beta` is not: it is a mirror update, not
development. Confirm the push output shows a fast-forward (`old..new`) and never
add `--force`. Name the fetched ref the way the project's own docs name it, and
delete any ad-hoc ref you invented (`git update-ref -d refs/remotes/<tmp>/<branch>`) —
leftover refs make later fetch output unreadable.

## Verification

- `git status` shows no unexpected submodule entries.
- `git diff --cached --stat` shows actual file additions, not just mode changes.
- Commit and push succeed without warnings about embedded repos.
