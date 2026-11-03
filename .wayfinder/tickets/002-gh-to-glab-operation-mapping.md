---
title: How does each gh operation map to glab/GitLab API?
label: wayfinder:research
status: open
assignee: 
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
