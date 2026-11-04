---
title: Where is super-board coupled to GitHub?
label: wayfinder:research
status: open
assignee: 
blocked_by: []
---
## Question

Inventory every GitHub coupling in super-board v3.3.0:
- call sites of `gh` and graphql, grouped by operation and by skill/script/hook/workflow/test
- implicit assumptions: Projects v2 field/item IDs, PR numbers, `github.com` URLs, `gh` JSON shapes parsed by `jq`/python
- how tests mock `gh`
- which evals touch the forge

Also check git history: how often have these call sites changed across releases? That decides how painful upstream syncs will be.
Findings: .wayfinder/research/github-coupling-inventory.md (branch research/github-coupling-inventory)
