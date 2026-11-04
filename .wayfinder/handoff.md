# Handoff: GitLab port of super-board (wayfinder)

Delete this file after reading it (user rule).

## Where things are
- Project: `~/Documents/Work/Projects/super-board-gitlab`, on branch `wayfinder`, which is pushed. It is added to Orca.
  - `origin` = https://github.com/yyslav/super-board-gitlab (the fork; GitHub account `yyslav`)
  - `upstream` = EricTechPro/super-board v3.3.0
- Map: `.wayfinder/map.md`. Tracker rules: `.wayfinder/README.md` (local markdown; `blocked_by` in frontmatter; claim = `assignee: yaroslav.y`). Glossary: `.wayfinder/GLOSSARY.md`.
- Research: `research/*` branches, also checked out as worktrees `../sbg-research-*`. Sandbox facts are in `.wayfinder/research/sandbox-verification.md` on `wayfinder`.
- Sandbox: gitlab.com group `yaroslav-sb-sandbox` (private), with projects `app-a` and `app-b` and group board "factory". `glab` is authed as `yaroslav.y` (OAuth, keyring).

## How to continue
Run `/wayfinder .wayfinder/map.md`. It picks the first frontier ticket, claims it, and works it as the type its label names (grilling → load the `grilling` and `domain-modeling` skills). Resolve one ticket per session. Talk to the user in Ukrainian, write files in English, keep reports terse.

## Frontier (open, unblocked, unclaimed)
1. What does onboard do on GitLab? (`tickets/010`, grilling)
2. How do UI screenshots get into MRs? (`tickets/011`, grilling; its blocker, the sandbox task, is now closed). Note: `glab api -F file=@` uploads failed, so uploads are untested.
3. How are agent-written notes kept from triggering quick actions? (`tickets/012`, grilling)

Blocked: In what order is the port built? (`tickets/013`, waits on 010).

## Key decisions so far (details in the tickets)
- The port is a hard fork: GitLab-only, no upstream sync. `glab` is used everywhere. `gitlab-read.py` keeps the read-funnel contract. Fields are GitLab-native. Files are rewritten in place.
- Lanes are plain `lane:*` labels: Backlog = no lane label, Done = closed. The board is `group:` or `project:`. The factory touches only `lane:*` labels. A claim = the user's own account as assignee + `loop:in-*`.
- Quality:
  - upstream baseline first
  - tests translated 1:1, with a dropped-behaviours ledger
  - one stateful glab fake built from sandbox recordings
  - e2e smoke on the 2-project group
  - evals at least as good as baseline over 3 runs
  - CI on GitHub Actions
- **Surprise from the sandbox:** the Free limit of 100 req/min per user is already in effect (`burst_traffic_per_user_free_plan`). `issueMoveList` does not work on a Free group board, so cards move by label edit. `per_page` max is 100. Only one group board can be created, via GraphQL.

## Remaining fog (map: Not yet specified)
- `pr-security-review` base semantics on MR diffs
- super-collect / `to-tickets` filing
- stop/resume under the shared rate limit
- assembling the final per-file rewrite spec, which is the destination
