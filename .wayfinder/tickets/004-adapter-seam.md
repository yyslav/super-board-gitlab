---
title: Where does the forge seam go?
label: wayfinder:grilling
status: open
assignee: yaroslav.y
blocked_by: [002, 003]
---
## Question

What shape does the forge adapter take, and where does it sit, so that the GitHub path stays byte-for-byte equivalent and the upstream diff stays minimal? Candidates:
- a `forge` CLI that skills, scripts, and hooks call
- a python module
- a `gh`-compatible shim that translates calls to GitLab
How is the forge selected: from the git remote, or from config?
