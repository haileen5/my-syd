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

**Where the peers live (verify before trusting):**
Both peer workflows live in `haileen5/my-syd`, display names `aylin` and
`nazila`. The repos this skill used to cite (`my-ayl`, `my-naz`) no longer
exist, so a hardcoded name 404s — re-verify with the discovery step below
whenever a repo/workflow lookup fails, and fix this section in place when the
peers move again.
```bash
gh workflow list --repo haileen5/my-syd
gh repo list haileen5 --limit 100 --json name,visibility   # discovery
```
Discovery rule: when `gh workflow list --repo <guessed>` 404s, do NOT guess
another name — list the user's repos and grep for the peer workflow names.
The workflows have already moved once; the repo is not a stable constant.

**Wake a peer:**
1. Confirm the display name — it is NOT always the filename. The first column
   of `gh workflow list` is the display name; use THAT with `gh workflow run`.
2. Run it:
   ```bash
   gh workflow run aylin  --repo haileen5/my-syd
   gh workflow run nazila --repo haileen5/my-syd
   ```
   A 404 (`workflow <name> not found`) means you used the filename or a
   wrong name — list again and use the display name exactly.
3. Confirm the run started, then probe the peer until it answers. A started
   run stays `in_progress` for the peer's whole life — never wait for it to
   reach `completed`, and never read "in progress" as "not up yet".
   ```bash
   gh run list --repo haileen5/my-syd --limit 3 \
     --json name,status,conclusion --jq '.[]|"\(.name)\t\(.status)"'
   ```

**Message a peer:**
```bash
hermes peer list          # configured peer names + URLs + key state
hermes peer dm nazila "..."
hermes peer dm aylin "..."
```
Peers answer in Persian — probe and report in Persian to match.
- Peer DMs for heavy tasks (issue reading, coding, commit+push) can run 5–10 min.
First send a short liveness probe (e.g. "سلام، آماده‌ای؟") with `timeout 30`.
If it replies, the channel is up — then send the full payload with `timeout 550`.
If the full DM yields to background, track via process poll/wait. If the first
DM times out, retry once before concluding the peer is down.
- A timeout on a heavy DM often means the task COMPLETED anyway — probe with
a lightweight message before resending; the peer usually reports the finished
result. Resending the full payload duplicates the work.

**Watch PRs opened from a peer branch:**
```bash
gh pr view <n> --repo asgarimehdi/h-dashboard --json comments,reviews
```
Poll for `reviews[].state == "CHANGES_REQUESTED"`; route each requested fix
back to the peer via DM (short payload, it already knows the context), have
it commit+push, then re-poll. Reviewers may leave comments minutes-to-hours
later — do not assume silence means done.

**Stop a peer:** cancel its Actions run (the peer runs as a workflow):
```bash
gh run list --repo haileen5/my-syd --limit 5 --json databaseId,name,status
gh run cancel <run-id> --repo haileen5/my-syd
```
`hermes peer stop` only stops an async `hermes peer run`, NOT the gateway
process itself — use `gh run cancel` to actually turn the peer off.
`gh run cancel` may return HTTP 502 on first attempt — retry once; also
allow ~30–60s (and re-poll) for the run's status to flip to `cancelled`.

## Pitfalls

- **Cold-start latency dominates a wake.** After `gh workflow run`, the peer
  needs roughly 3–6 min (image build + gateway boot + Tailscale DNS) before a
  DM lands. Probe repeatedly instead of declaring failure:
  1. `getent hosts aylin` / `getent hosts nazila` — empty means Tailscale DNS
     has not published the host yet; wait 60–120s and re-check.
  2. DNS resolves but `hermes peer dm` says `Connection refused` — host is up,
     port 8642 not yet bound; wait another 90–150s and retry.
  3. Only after DNS resolves AND the port has refused for several minutes
     across multiple rounds is the peer genuinely down; check the run's job
     steps (`gh run view <id> --json jobs`) for where it stalled.
  One refusal is a normal boot state, not an outage — treat 3+ refusals across
  5+ minutes as the failure signal.
- `hermes peer dm` can time out (180s) on first contact even when the peer
  is alive — retry once before concluding it's down; a direct test from
  the user confirms reachability.
- Peer memory fills up; periodically ask each peer via DM to dedupe its
  memory and migrate reusable entries into skills.
- Never hardcode a peer's branch or fork URL — read `git remote -v` /
  `git config --get-regexp '^branch\.'` on that peer when needed.
- Peers run in a non-interactive gateway: they CANNOT write to protected
  files (AGENTS.md) — the file-mutation verifier blocks the patch and the
  peer reports it as a WARNING. Never claim the write landed. To apply the
  text yourself: from the home repo, `git stash` any dirty files,
  `git checkout <peer-branch>` (create a local tracking branch), patch,
  commit, `git push origin HEAD:refs/heads/<peer-branch>`, then
  `git checkout <original>` and `git stash pop`. Ask the peer to print the
  exact block so you paste it verbatim — do not reconstruct from memory.
- When a peer reports success, always check the trailing
  `File-mutation verifier` warning — it lists every edit that failed
  despite the wording above.
