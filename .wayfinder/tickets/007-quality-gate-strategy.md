---
title: How do we prove the port did not degrade the factory?
label: wayfinder:grilling
status: open
assignee: 
blocked_by: [004]
---
## Question

How do we prove that the GitLab port behaves like upstream super-board on every lane: Ready → Building → QA → Review → merged, plus stop/resume, halts and the merge gate? Upstream is the behavioural reference; upstream tests on GitHub no longer apply.

Settle:
- how the ~20 gh-stub tests and the shared eval stub are ported to glab/GitLab-shaped fakes
- whether a shared fake replaces the per-test stubs
- how faithful the fakes must be to real gitlab.com responses (recorded fixtures?)
- the e2e smoke scenario on a sandbox gitlab.com group
- how evals show no regression against upstream results
