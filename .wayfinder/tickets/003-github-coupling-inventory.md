---
title: Where is super-board coupled to GitHub?
label: wayfinder:research
status: closed
assignee: yaroslav.y
blocked_by: []
---
## Question

Inventory every GitHub coupling in super-board v3.3.0:
- call sites of `gh` and graphql, grouped by operation and by skill/script/hook/workflow/test
- implicit assumptions: Projects v2 field/item IDs, PR numbers, `github.com` URLs, `gh` JSON shapes parsed by `jq`/python
- how tests mock `gh`
- which evals touch the forge

Also check git history: how often have these call sites changed across releases? That decides how painful upstream syncs will be.

## Resolution

Findings: `.wayfinder/research/github-coupling-inventory.md` on branch `research/github-coupling-inventory` (commit 8833f28).

**Three layers of coupling:**
1. About 25 executable files call `gh`.
2. About 27 skill and reference .md files tell agents to run literal `gh` commands.
3. Shared data-model assumptions.

**Funnels:**
- Two funnels already exist: `super-board-github-read.py` for reads and `super-board-card.sh` for board moves.
- They are bypassed by the prose, the three filers, `stop.sh`, `pr-body.sh`, `merge-gate.sh`, `approval.py`, `setup.py` and `cleanup-wt.py`.

**Hardest part: Projects v2.** It relies on:
- the `Status` field
- the 7 columns in `setup.py`
- `ProjectV2*` GraphQL
- cached `PVT_*` node IDs

**Implicit protocol:**
- `Closes #N` and `blocked-by: #N` lines
- `closingIssuesReferences` for binding approvals
- `--match-head-commit`
- labels, comment prefixes and HTML markers used as protocol
- GraphQL and REST rate-limit regexes

**Tests and evals:**
- About 20 tests mock `gh` through PATH shadowing, each with its own stub and no shared helper.
- All 7 evals share one fixture stub, `evals/_stubs/bin/gh`.

**Churn:**
- 27 of 119 commits touched executable `gh` lines.
- Every version since v3.0.4 has changed them, concentrated in the funnel files (`setup.py` 11 commits, `wave.js` 5).
- Conclusion: put an adapter behind a stable verb set. An in-place rewrite would be painful to sync.
