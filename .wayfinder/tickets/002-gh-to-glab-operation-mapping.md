---
title: How does each gh operation map to glab/GitLab API?
label: wayfinder:research
status: closed
assignee: yaroslav.y
blocked_by: []
---
## Question

For every `gh` operation super-board uses, what is the GitLab Free equivalent (`glab` command, REST, or GraphQL), and where are the semantic gaps? Operations:
- `issue view/edit/comment/list/create`
- `pr create/view/merge/ready/edit/diff/comment/list`
- auto-merge vs merge-when-pipeline-succeeds
- check/run status vs pipelines
- `api rate_limit`
- `repo view`
- raw file URLs (public and private projects)
- closing keywords and `#123` refs vs `!123`
- draft MRs
- approvals on Free
- auth scopes
Output a mapping table with gap notes.

## Resolution

Findings: `.wayfinder/research/gh-to-glab-operation-mapping.md` on branch `research/gh-to-glab-operation-mapping` (commit ae4770d). The table has about 30 rows; each row is marked doc, inferred or unverified.

**Key gaps:**
- **JSON:** `glab` has no `--json` field selector, so the adapter must map fields, e.g. `number`←`iid`, `url`←`web_url`, `headRefOid`←`sha`, `body`←`description`.
- **Merging:** `glab mr merge` uses auto-merge by default. Pass `--auto-merge=false`, then verify the MR state. `--sha` replaces match-head-commit and returns 409 when the head has moved.
- **Approvals:** on Free they are advisory only, so super-board's merge gate stays the only gate.
- **Closing:** an issue auto-closes only on merge into the default branch. Otherwise the adapter must close it explicitly. There is no close reason.
- **Rate limits:** there is no `rate_limit` endpoint. Use the RateLimit headers and the 429 `Retry-After`.
- **Token:** use the `api` scope. There is no `project` scope.
- **Traps:**
  - Quick actions such as `/merge` in a note body get executed.
  - `glab mr create -H` means the head repo, not the head branch.
  - Raw images in private projects need a token, so use uploads instead.

**Unverified:** the `/-/raw/` behaviour, GraphQL draft and discussion mutations, `per_page` max, and whether `mr ready` is idempotent. Check these on the sandbox project.
