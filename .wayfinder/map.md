---
title: Port super-board to GitLab without degrading the factory
label: wayfinder:map
---
## Destination

A hand-off-ready spec for porting super-board to gitlab.com Free, alongside the unchanged GitHub path. It covers:
- the forge adapter interface
- a GitHub→GitLab mapping for each operation
- the test and quality-gate strategy
- the upstream-sync routine

No implementation happens inside this map.

## Notes

- Domain: the super-board fork (MIT), upstream EricTechPro/super-board v3.3.0. The fork lives on GitHub and is never offered upstream as a PR. Keep the diff vs upstream minimal and keep pulling upstream releases.
- Both forges must work. The GitHub path stays behaviourally identical, and the upstream test suite is the regression net.
- Target: gitlab.com **Free**. No scoped labels, no native blocking, and probably a single assignee.
- Board semantics match upstream, including boards that span repos, built on GitLab features.
- Merge gate: local `verify_commands` are mandatory; a GitLab pipeline counts if the project has one.
- Scope: every skill, board-run and user-typed. Backend: `workflow` (Claude in-session) only.
- Quality criteria:
  1. Upstream tests green on GitHub.
  2. The same suite green against a GitLab mock.
  3. e2e smoke on a sandbox gitlab.com project.
  4. Evals do not regress.
- Every session: load the `grilling` and `domain-modeling` skills, and use `.wayfinder/GLOSSARY.md`.
- Tracker conventions: `.wayfinder/README.md`.

## Decisions so far

- [What can a gitlab.com Free board express?](tickets/001-gitlab-free-board-capabilities.md): lanes = mutually exclusive labels; multi-repo = one group board; no custom fields/blocking on Free; budget 100 req/min
- [How does each gh operation map to glab/GitLab API?](tickets/002-gh-to-glab-operation-mapping.md): no --json selector → adapter maps fields; merge with --auto-merge=false + verify; approvals advisory on Free; api scope
- [Where is super-board coupled to GitHub?](tickets/003-github-coupling-inventory.md): 3 layers (≈25 exec files, ≈27 prose files, data-model); Projects v2 hardest; funnels exist but bypassed; churn high in funnels → stable verb adapter

## Not yet specified

- Config and forge detection: how a project declares or auto-detects GitLab, and what changes in the config schema.
- Onboard flow: rework its "GitHub" step into a forge step (auth scopes, board creation on GitLab).
- `ui-refine-loop` / `pr-shots`: hosting screenshots on GitLab. Raw URLs in private projects need a token, so project uploads look likely (see the mapping ticket).
- Comment and note bodies: sanitise GitLab quick actions (`/merge`, `/close`) in agent-written text.
- `pr-security-review` on MR diffs: the base/merge-base semantics.
- `super-collect` bug sources and `/to-tickets` creating GitLab issues.
- Stop/resume comments and the assignee mutex under GitLab rate limits.
- The rollout order of skills, and whether there is a first vertical slice (one card, Ready → merged MR).

## Out of scope

- Offering the port upstream as a PR: the user's choice.
- Moving distribution (`get.sh`, plugin marketplace) to GitLab: it stays on GitHub.
- Porting the `claude-p` and `--codex` backends to GitLab: the user runs `workflow`.
- Premium-only GitLab features and self-hosted GitLab.
- Implementing the port: this map ends at the spec.
