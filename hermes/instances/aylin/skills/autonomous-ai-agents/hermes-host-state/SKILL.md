---
name: hermes-host-state
description: "Use when Hermes settings vanish or don't persist."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, config, env, peers, provisioning, logs, diagnostics]
    category: autonomous-ai-agents
    related_skills: [read-the-damn-docs, hermes-agent]
---

# Diagnosing Hermes host state: what persisted, what a restart erased

## When to Use

- A setting or credential the user stored is gone at the start of a new session, or a
  CLI reports something unregistered / unset.
- A config edit "keeps reverting" across restarts, or a container/VM is rebuilt and
  state the agent added is absent.
- The user asks why something disappeared, or corrects your assumption about where
  they put it.

Do not answer with a guess about which subsystem did it. Locate the authoritative
file, then prove what happened to it.

## The two-file rule

- `~/.hermes/.env` holds **credentials only** (tokens, `HERMES_*_KEY`). `config.yaml`
  holds **every setting**, including registries that point at a secret by env-var name.
  Saving an entry to `.env` can never register a setting.
- Canonical example: `hermes peer` splits across both files — the name→url registry is
  `bot_peers` in `config.yaml`; only `HERMES_PEER_<NAME>_KEY` is written to `.env`.
  `hermes peer list` reads `bot_peers` and nothing else, so a peer added only in
  `.env` reports as unregistered even though the key is present.
- Generalize instead of guessing: open the subcommand's source under
  `hermes_cli/subcommands/` and read it. `_load_peers`-style helpers show the
  `cfg.get(...)` key that names the config path; `save_env_value(...)` shows the
  secret's env name. One read answers "which file, which key" for any subcommand.

## Proof order — run these before explaining

1. **mtimes first.** `stat -c '%n %y' ~/.hermes/config.yaml ~/.hermes/.env`. If
   `config.yaml`'s mtime is boot time rather than your edit time, something restored
   the file wholesale and your change was never going to survive.
2. **Find the provisioning path.** `grep -rl 'config.yaml' /home/runner/work/_temp/*.sh`
   (or wherever the boot scripts live), then read the restore block. A loop that
   `cp`s `config.yaml` (and friends) from a Git checkout into `~/.hermes/` means every
   CLI write to `config.yaml` is discarded on the next boot.
3. **Confirm with a diff.** `diff <repo copy> ~/.hermes/config.yaml`. A diff that
   contains only boot-time edits (schema `_config_version` bump, installed plugin
   paths) proves the live file is the restored copy, not your edited one.
4. **Then name the fix.** Commit the section into the copy the restore pulls from,
   and/or re-apply it in the boot script as a second guarantee. Re-running the CLI
   alone is a fix that survives exactly one boot — say that plainly.

## Attribution: prove who did the thing

- **Never assume an inbound event came from this box.** Grep the access log for the
  path and User-Agent, then compare the remote IP with local interfaces
  (`ip -o addr show`, `hostname -I`). An IP that is not listed is another machine or
  container; the local request path may have no log line at all.
- **Before blaming a recent action, check it actually touched the file.** If
  `config.yaml`'s mtime predates the action, the action is exonerated — retract the
  association in one line rather than leaving it standing.
- When you cannot find any record of the step in question (no session, no log, no
  script), say that directly instead of inferring that it ran.

## This box

- Boot scripts live in `/home/runner/work/_temp/*.sh` and rebuild `~/.hermes/` from a
  Git checkout (`.../my-ayl/my-ayl/hermes/`), writing `.env` from GitHub Secrets.
  Anything needed on every boot must live in that repo, not only in `~/.hermes`.
- Re-find the current script rather than trusting a remembered filename:
  `grep -rln 'HERMES_PEER\|cp .*config.yaml' /home/runner/work/_temp/*.sh`.

## Reporting

- Lead with the authoritative file + key, then the evidence (mtime, diff, log line),
  then the fix. Match the user's language.
- If an earlier statement in the session was wrong, correct it in the first line
  before giving the corrected account.