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

## Not yet specified

- Config and forge detection: how a project declares or auto-detects GitLab, and what changes in the config schema.
- Onboard flow: rework its "GitHub" step into a forge step (auth scopes, board creation on GitLab).
- `ui-refine-loop` / `pr-shots`: hosting screenshots on GitLab, including raw URLs in private projects and whether images render in MR descriptions.
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
