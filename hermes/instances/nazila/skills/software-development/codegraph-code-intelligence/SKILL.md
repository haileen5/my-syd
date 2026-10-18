---
name: codegraph-code-intelligence
description: "Use CodeGraph for code-structure questions and index health."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [codegraph, code-intelligence, mcp, index, structural-queries]
    category: software-development
---

# CodeGraph Code Intelligence

CodeGraph is a local (SQLite, no API keys) code knowledge graph. It answers
structural questions — callers, blast radius, flow — far better than grep, and it
returns verbatim on-disk source with line numbers.

## When to Use

- Any "how does X resolve Y / who calls Z / what breaks if I touch this" question,
  before reaching for grep, glob or a full file read.
- Before editing a method you have not read: `explore` returns its callers and the
  tests that cover it.
- When a CLI `codegraph sync` reports the index database is locked.

Prefer the MCP tool (`codegraph_explore`) when it is registered in the session;
the CLI is the fallback and the only path when the daemon is not running.

## Procedure

1. Check the install and the index:

```bash
command -v codegraph     # global npm bin; if absent: npm install -g @colbymchenry/codegraph && codegraph init
codegraph status .       # files, nodes, edges, db size — is this repo indexed at all
```

2. Ask a whole question with `explore`, a single symbol with `query`:

```bash
codegraph explore "how does AccessService accessibleUnitIds resolve unit hierarchy"
codegraph query "AccessService" --limit 5
```

3. Treat the returned source as already read: `explore` prints the verbatim
   current on-disk source (line-numbered) for the files it covers, and names the
   files it elided. For those, issue another `explore` naming them — do not spend
   separate reads on files already shown.

## Index freshness and the daemon lock

When `codegraph serve --mcp` is running, that daemon owns the index and a file
watcher keeps the graph current. A CLI sync then fails with:

```
✗ Failed to sync: Sync could not acquire the file lock; retry when the index is available.
  CodeGraph database is locked by another process (PID …).
```

That lock is **live, not stale** — the daemon is already doing the syncing.

- Verify instead of guessing, from the daemon's own files:

```bash
cat .codegraph/daemon.log    # 'File watcher active' + 'Auto-synced N file(s) in Nms'
cat .codegraph/daemon.pid    # pid, version, socket path
ps -o pid,etime,cmd -p <pid> # confirm the pid really is the serve --mcp process
```

  An auto-sync line newer than your last edit means the index is current: query it
  now and report the index as fresh.
- Do NOT run `codegraph unlock`, and do not kill the daemon to free the lock — you
  would destroy an up-to-date index to re-run a command the daemon already ran.
- The CLI sync path belongs to the case where no daemon is running (no
  `daemon.sock`, pid gone or not a codegraph process). A large tree change — a big
  merge, a branch switch — appears as one auto-sync burst in the log.
- The MCP tools only register at session start; when `codegraph_explore` is not in
  the tool catalog, use the CLI for the rest of the session.

## Environment note

`.codegraph/` lives inside the repo and is git-ignored; it never belongs in a
commit. If the bin is missing after a server rebuild, reinstall it
(`npm install -g @colbymchenry/codegraph`) and run `codegraph init` in the project
root before falling back to grep.

## Verification

- `codegraph status .` reports an index whose file/node counts match the repo's size.
- `daemon.log` shows an auto-sync newer than the last edit, or no daemon is running
  and the CLI sync completed.
- `git status` shows nothing for `.codegraph/`.
