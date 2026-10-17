# Issue-finding audit waves — publish a verdict comment per issue

For the deliverable "(B)" of the parent skill: a queue of issues whose bodies are
audit findings (often with an embedded ready-to-execute plan), where your job is
to verify each finding against the pinned branch and publish a structured verdict
comment. Nothing else about the issue is touched.

## Pin a branch anchor once, and cite it in every comment

Fetch the audit base as an explicit ref and never move off it:

```bash
git fetch https://github.com/O/R.git <branch>:refs/remotes/canonical/<branch>
git rev-parse --short=8 canonical/<branch>
```

Put `<branch>@<sha>` on the second line of every comment. When several agents work
the same queue concurrently, the sha is the only thing that makes a comment
reproducible — a reviewer cannot diff your claims against a moving branch.

## Selection order, and the anti-collision gate

Re-derive the open-issue inventory each round instead of trusting an earlier
list, because other agents close/comment issues while you work:

```bash
for p in 1 2 3 4; do
  gh api "repos/O/R/issues?state=open&per_page=100&page=$p" \
    --jq '.[] | select(.pull_request == null) | select(.number >= <lo> and .number <= <hi>) | "\(.number) \(.comments)"'
done | sort -n > /tmp/inv.txt
awk '$2==0{print $1}' /tmp/inv.txt > /tmp/zero.txt
shuf -n 1 /tmp/zero.txt          # randomise when the user asks to reduce collisions
```

Check `.comments == 0` **twice**: once immediately before the deep read, and once
again immediately before posting.

```bash
n=$(gh api repos/O/R/issues/<N> --jq '.comments')
if [ "$n" = "0" ]; then gh issue comment <N> --repo O/R --body-file /tmp/c<N>.md; else echo SKIP; fi
```

The second check catches the common failure: you spend twenty minutes verifying
an issue and a concurrent agent posted in between. Do not paste your analysis
over theirs — skip and take the next one. **Never post a partially-written
comment**: write the body to a file first, read it back, then post.

## Progress accounting

Report progress as a count of *posted comment URLs*, not as prose tallies you
accumulate in chat. Informal counts drift when issues get skipped by the gate;
the URL list is the ground truth. If you must summarise, derive it from the
posted links.

## Comment shape

Language and technical-term conventions follow the user's standing preferences
(e.g. prose in the issue's language, code identifiers verbatim). Structure, with
fixed slot names so a reviewer can scan many comments:

```
<one-line verdict>            # fixed / partially fixed / not a defect / needs a decision / unverifiable
<base line>                   # branch@sha — no drift | drift in <files>
§ verification                # the finding re-checked against code, with file:line
§ what changes if executed    # behaviour, files, blast radius
§ tests that may break        # real test paths + why; or "none identified"
§ new tests needed
§ decision                    # "no decision needed — clear path" OR question + lettered options + your recommendation
§ suggested execution plan
```

Rules that make these comments worth reading:

- Sections 2..6 only when the finding holds. For "already fixed / duplicate",
  section 1 is the whole comment.
- **Code wins over the issue text.** Where they conflict, say so with evidence in
  the verification section; do not silently pick one.
- Cite `file:line` for every claim, and name existing test files by path rather
  than describing them generically.
- When you could not verify something, write "not verified" plus why. Do not
  guess to fill the slot.
- Recommend dropping a plan step rather than hedging on it when evidence says the
  step reverses an intentional, test-pinned decision.

## What earns the "needs a decision" verdict

Reserve it for choices a maintainer owns — policy, ownership, scope trade-offs,
keep-or-replace. If the code makes the answer obvious, say "clear path" and move
on; over-reporting indecision makes the queue unusable.