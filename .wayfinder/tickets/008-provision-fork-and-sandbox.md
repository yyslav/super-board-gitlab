---
title: Provision the GitHub fork and a gitlab.com sandbox
label: wayfinder:task
status: open
assignee: yaroslav.y
blocked_by: []
---
## Question

Provision the access later tickets need to verify against real APIs:
- fork EricTechPro/super-board on GitHub and point the local clone's `origin` at it, keeping `upstream`
- create a sandbox gitlab.com group with two projects (needed to test a multi-repo board)
- `glab auth login` with the required scopes

Then verify these facts live on Free and record the results:
- issue GraphQL `relativePosition` matches the manual board-list order
- whether `issueMoveList` is available
- the real `glab ... -F json` shapes for issue and MR view/list
- `/-/raw/` access on a private project
- `glab mr ready` idempotency
- `per_page` max
- the rate-limit headers returned

Record the URLs, where credentials live, and the verified facts.
