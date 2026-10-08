For h-dashboard PRs: user says 'pr' → create PR from current branch to upstream/beta (asgarimehdi/h-dashboard). All changes commit+push to current branch.
§
Every new session: cwd /home/runner/h-dashboard; use CodeGraph (`codegraph sync` first) + read-the-damn-docs + superpowers skills (v6.4.2 at /home/runner/.hermes/plugins/superpowers, pre_llm_call hook sends bootstrap each message) — always namespaced, skill_view("superpowers:brainstorming"); unprefixed names not found. shadcn/improve only on request.
§
Boost MCP occasionally dies on first stdio call ("lost its stdio subprocess") — just call it again. CLI fallback always works: php scripts/boost_tool.php <tool> '<json>'.
§
h-dashboard branch aylin tracks origin/beta (branch.aylin.merge=refs/heads/beta), so `git status` shows 'aylin...origin/beta'; always use explicit refspecs HEAD:refs/heads/aylin.
§
homeassistant is permanently deny-listed in ~/.hermes/config.yaml (plugins.disabled) — never re-enable or `hermes plugins install` it; it was never installed here.
§
Patch tool blocks AGENTS.md writes in non-interactive sessions even with explicit chat consent, and retry via terminal or other paths is forbidden.