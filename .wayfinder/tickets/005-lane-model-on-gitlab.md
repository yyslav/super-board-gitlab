---
title: How are lanes and card fields modelled on GitLab Free?
label: wayfinder:grilling
status: closed
assignee: yaroslav.y
blocked_by: [001, 003]
---
## Question

How do super-board's lanes, Projects v2 fields, and the assignee mutex map onto GitLab Free primitives (labels, board lists, assignees, issue links) with the same semantics as upstream, including multi-repo boards? Who enforces that a card sits in exactly one lane?

## Resolution

The user decided this in a grilling session on 2026-11-04.

- **Lanes:** plain labels `lane:ready`, `lane:building`, `lane:qa`, `lane:review`, `lane:blocked`, each shown as a board list.
  - Backlog = an open issue with no `lane:*` label.
  - Done = a closed issue. If the MR merged into a non-default branch, the issue is closed explicitly.
  - Projects v2 `Status` and its 7 columns map onto these. No other fields are needed, since upstream uses only `Status`.
- **Board scope:** config `board: group:<path>` uses the single group board with group labels, giving multi-repo boards as upstream has them. `board: project:<path>` uses a project board with project labels, e.g. for a user namespace. `setup.py` creates the labels and lists at the matching level.
- **One lane per card:**
  - `card.sh move` is one `glab issue update`: `--label <new> --unlabel <every other lane:*>`.
  - Only `lane:*` labels are ever touched. Type labels (`qa`/`bug`/`feature`) and user labels are left alone.
  - A card with more than one `lane:*` label is skipped by wave-plan, which posts a warning comment. Nothing is auto-fixed.
- **Ready order:** manual board-list order (GraphQL `relativePosition`), falling back to `created_at`. Still to verify on the sandbox.
- **Draft issues:** dropped, since GitLab has none. Backlog holds plain issues.
- **Claim mutex:** the assignee is the user's own account (`BOT_LOGIN` = the user).
  - A factory claim = that assignee **plus** a `loop:in-*` label.
  - Release (stop, reconcile) removes the assignee only when `loop:in-*` is present.
  - A card with an assignee and no `loop:in-*` counts as taken by a human: the factory skips it and changes nothing.
- **Rate limit:** the factory shares the 100 req/min per-user bucket with the user's own glab/API use. `glab-guard` must budget for that.
- **Unchanged from upstream:** `blocked-by:` comment lines and the type labels.
