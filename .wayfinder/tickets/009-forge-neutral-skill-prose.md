---
title: How do skill instructions stop naming gh?
label: wayfinder:grilling
status: closed
assignee: yaroslav.y
blocked_by: [004]
---
## Question

About 27 skill and reference .md files tell agents to run literal `gh` commands. How should they refer to forge operations so that the agents run the right forge, and so that upstream prose edits still merge cleanly? Options:
- rewrite them to adapter verbs
- add a per-forge include
- keep the `gh` wording and intercept the calls with a shim or guard

## Resolution

Answered by [Where does the forge seam go?](004-adapter-seam.md). Prose is rewritten in place to name `glab` commands and GitLab-native fields, and `glab` reads go through `gitlab-read.py --kind`. Upstream merges no longer matter.
