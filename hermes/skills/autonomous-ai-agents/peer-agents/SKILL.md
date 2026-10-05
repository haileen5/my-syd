---
name: peer-agents
description: "Wake, message, or stop peer Hermes gateways (Aylin, Nazila)."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, peers, multi-agent, aylin, nazila, gh-actions]
---

# Peer Hermes Gateways (Aylin / Nazila)

## When to Use

The user says "wake up Aylin/Nazila", "give X a task", "ask X", or "shut
down X" — anything involving the Aylin/Nazila peer gateways.

Standing user convention (already in USER profile): "wake up X" = run X's
GitHub Actions workflow; "give X a task" = `hermes peer dm X "..."`.

## Procedure

**Wake a peer:**
1. Find the workflow's display name — it is NOT always the filename.
   ```bash
   gh workflow list --repo haileen5/my-naz   # or my-ayl
   ```
   The first column is the display name; use THAT with `gh workflow run`.
2. Run it:
   ```bash
   gh workflow run <display-name> --repo haileen5/my-naz
   ```
   A 404 (`workflow <name> not found`) means you used the filename or a
   wrong name — list again and use the display name exactly.

**Message a peer:**
```bash
hermes peer dm nazila "..."
hermes peer dm aylin "..."
```

**Stop a peer:** cancel its Actions run (the peer runs as a workflow):
```bash
gh run list --repo haileen5/my-naz --limit 5     # find run id + status
gh run cancel <run-id> --repo haileen5/my-naz
```
`hermes peer stop` only stops an async `hermes peer run`, NOT the gateway
process itself — use `gh run cancel` to actually turn the peer off.

## Pitfalls

- `hermes peer dm` can time out (180s) on first contact even when the peer
  is alive — retry once before concluding it's down; a direct test from
  the user confirms reachability.
- Peer memory fills up; periodically ask each peer via DM to dedupe its
  memory and migrate reusable entries into skills.
- Never hardcode a peer's branch or fork URL — read `git remote -v` /
  `git config --get-regexp '^branch\.'` on that peer when needed.
