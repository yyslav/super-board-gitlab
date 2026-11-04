---
title: Provision the GitHub fork and a gitlab.com sandbox
label: wayfinder:task
status: closed
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

## Resolution

Done on 2026-11-04.
- **GitHub fork:** https://github.com/yyslav/super-board-gitlab. The local clone has `origin` set to the fork and `upstream` set to EricTechPro. The `wayfinder` and `research/*` branches are pushed.
- **GitLab sandbox:** private group https://gitlab.com/groups/yaroslav-sb-sandbox. It was created in the UI, because the API returns 403 for top-level groups on gitlab.com. It holds private projects `app-a` and `app-b` (default branch `main`) and the group board "factory" with lists `lane:ready` and `lane:building`. Test issues app-a #1–3 and draft MR !1 are left in place.
- **Credentials:** `glab` OAuth in the OS keyring (`yaroslav.y`). `gh` uses the GitHub account `yyslav`.
- **Verified facts:** see [sandbox-verification](../research/sandbox-verification.md). Key results:
  - board order = `relativePosition`
  - `issueMoveList` does not work, so cards move by label edit
  - one group board, created via GraphQL
  - the 100 req/min Free limit is already in effect
  - `per_page` max is 100
  - private raw links redirect to sign-in
  - `mr ready` is idempotent
