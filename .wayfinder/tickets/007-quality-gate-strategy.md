---
title: How do we prove the port did not degrade the factory?
label: wayfinder:grilling
status: open
assignee: 
blocked_by: [004]
---
## Question

How do we make these agreed quality criteria concrete and runnable?
1. Upstream tests stay green on the GitHub path.
2. The same suite, parameterized with a `glab`/GitLab mock, passes.
3. An e2e smoke run on a sandbox gitlab.com project passes.
4. Evals do not regress.

Settle the parameterization approach, mock fidelity, smoke scenario, and how this runs on every upstream sync.
