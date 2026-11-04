---
title: Port super-board to GitLab without degrading the factory
label: wayfinder:map
---
## Destination

A hand-off-ready spec for a GitLab-only hard fork of super-board on gitlab.com Free. It covers:
- a per-file rewrite plan (gh → glab, GitLab-native fields)
- the lane/board model on GitLab
- the test and quality-gate strategy that proves behaviour matches upstream

No implementation happens inside this map.

## Notes

- Domain: a hard fork of super-board (MIT), from upstream EricTechPro/super-board v3.3.0. The fork is hosted on GitHub and never offered upstream as a PR. It is **GitLab-only** and does not sync with upstream. For GitHub, the user runs the original.
- Target: gitlab.com **Free**. No scoped labels, no native blocking, and probably a single assignee.
- Board semantics match upstream, including boards that span repos, built on GitLab features.
- Merge gate: local `verify_commands` are mandatory; a GitLab pipeline counts if the project has one.
- Scope: every skill, board-run and user-typed. Backend: `workflow` (Claude in-session) only.
- Quality: the port behaves like upstream on every lane. Ported tests and evals pass against GitLab fakes, and an e2e smoke run on a sandbox gitlab.com group passes.
- Every session: load the `grilling` and `domain-modeling` skills, and use `.wayfinder/GLOSSARY.md`.
- Tracker conventions: `.wayfinder/README.md`.

## Decisions so far

- [What can a gitlab.com Free board express?](tickets/001-gitlab-free-board-capabilities.md): lanes = mutually exclusive labels; multi-repo = one group board; no custom fields/blocking on Free; budget 100 req/min
- [How does each gh operation map to glab/GitLab API?](tickets/002-gh-to-glab-operation-mapping.md): no --json selector → adapter maps fields; merge with --auto-merge=false + verify; approvals advisory on Free; api scope
- [Where is super-board coupled to GitHub?](tickets/003-github-coupling-inventory.md): 3 layers (≈25 exec files, ≈27 prose files, data-model); Projects v2 hardest; funnels exist but bypassed; churn high in funnels → stable verb adapter
- [Where does the forge seam go?](tickets/004-adapter-seam.md): hard fork, GitLab-only, no upstream sync; glab everywhere; gitlab-read.py keeps funnel contract; GitLab-native fields; rewrite in place
- [How are lanes and card fields modelled on GitLab Free?](tickets/005-lane-model-on-gitlab.md): plain lane:* labels (Backlog=no label, Done=closed); group or project board; only lane:* labels touched; claim = own account + loop:in-*
- [How do we prove the port did not degrade the factory?](tickets/007-quality-gate-strategy.md): upstream baseline first; tests 1:1 + dropped-behaviours ledger; one stateful glab fake from sandbox recordings; e2e smoke on 2-project group; evals ≥ baseline over 3 runs
- [How do skill instructions stop naming gh?](tickets/009-forge-neutral-skill-prose.md): prose rewritten in place to glab + gitlab-read.py --kind

## Not yet specified

- `pr-security-review` on MR diffs: its base/merge-base semantics (`diff_refs`).
- `super-collect` bug sources and `/to-tickets` filing GitLab issues: per-file detail, possibly folded into the rewrite spec.
- Stop/resume comments and claim release under the shared 100 req/min bucket.
- Assembling the final hand-off: the per-file rewrite plan that collects every decision into one spec.

## Out of scope

- [How do we keep pulling upstream releases?](tickets/006-upstream-sync-strategy.md): there is no upstream sync because the port is a hard fork.
- GitHub support in the fork: the user runs the original for GitHub.
- Post-port refactors, such as merging the three filers.

- Offering the port upstream as a PR: the user's choice.
- Moving distribution (`get.sh`, plugin marketplace) to GitLab: it stays on GitHub.
- Porting the `claude-p` and `--codex` backends to GitLab: the user runs `workflow`.
- Premium-only GitLab features and self-hosted GitLab.
- Implementing the port: this map ends at the spec.
