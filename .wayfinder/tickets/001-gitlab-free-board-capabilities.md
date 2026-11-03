---
title: What can a gitlab.com Free board express?
label: wayfinder:research
status: closed
assignee: yaroslav.y
blocked_by: []
---
## Question

What does gitlab.com Free give us to rebuild super-board's GitHub Projects v2 board?
- project vs group issue boards and how many per project/group
- multi-project (group) boards, since upstream boards can span repos
- label-list columns; scoped labels are Premium, so how to keep labels mutually exclusive
- custom fields: is there any equivalent of Projects v2 single-select fields?
- assignees (single on Free?), linked issues / blocking, weights, milestones, iterations
- the REST and GraphQL APIs to read and move cards; gitlab.com rate limits
Mark each item Free / Premium-only, citing primary docs.

## Resolution

Findings: `.wayfinder/research/gitlab-free-board-capabilities.md` on branch `research/gitlab-free-board-capabilities` (commit 8c32d70).

**What Free gives us:**
- multiple project boards, label lists, one assignee, milestones, "relates to" links, REST and GraphQL APIs
- exactly one group board, which spans every project in the group and its subgroups

**Premium-only:** scoped labels, custom fields, Status, weight, iterations, epics, multiple assignees, blocks links, WIP limits.

**What this means for the port:**
- Lane = label list, with Closed as Done. Keeping labels mutually exclusive is our job: one edit with `add_labels` + `remove_labels`, plus a reconcile pass.
- Multi-repo board = put the repos in one group, use group labels and the single group board. A board across different top-level groups needs our own aggregation.
- Moving a card = an issue label edit. Whether GraphQL `issueMoveList` works on Free still needs checking.
- Rate limits: the current limit is 2,000 req/min. Budget for the proposed Free limit of 100 req/min and 5,000 req/hr.
- Blocking, priority and size become labels or a description block.
