# GitHub coupling inventory: super-board v3.3.0

Source: this worktree at v3.3.0 (`VERSION`=3.3.0, HEAD `fcf3fa9`). `diagrams/` and `.wayfinder/` ignored. All claims below are from the code and `git log` of this repo. File:line refs are to v3.3.0.

Method: `grep` for `gh`, `graphql`, `github.com`, `GH_*`, `projectV2`, then reading each call site; churn via `git log -G<pattern>` / `git diff -G`. Pattern used for "gh call-site line": `gh (pr|issue|project|api|label|release|repo|auth)`, `graphql`, `gh_read`, `gh_write_once`, `github-read`, `--kind `, `projectV2`, `item-list|field-list|item-edit|item-add`, `rate_limit`, `GH_REPO`.

## 0. Shape of the coupling (summary)

1. Coupling is **three layers**, not one:
   - **Layer A, executable call sites** (scripts, hooks, workflow JS, CI): about 25 files. Real `gh` subprocesses.
   - **Layer B, agent-prose call sites** (SKILL.md and references/*.md): about 27 md files tell LLM agents to run `gh ...` literally. These are the largest *volume* of coupling: `gh issue edit` 19x, `gh pr view` 15x, `gh pr merge` 13x, `gh project item-list` 13x, `gh api graphql` 12x. They are code in all but name; a port must rewrite them.
   - **Layer C, data-model assumptions** baked into both: Projects v2 board with single-select `Status` field and 7 hardcoded columns, GraphQL `ProjectV2*` types, issue/PR numbers as card identity, `Closes #N` closing keywords, `{owner}/{repo}` placeholder, `github.com` remote parsing, GitHub rate-limit model (5000 GraphQL points, REST core bucket).
2. **Funnels exist.** Most read paths go through `scripts/super-board-github-read.py` (retry, halt file, `--kind` shape validation) and board reads/moves through `scripts/super-board-card.sh`. Writes go through `gh_write_once` in `super-board-run.sh` or are direct (`gh pr merge`, `gh api -X PATCH`, `gh project item-edit`). Funnels are the natural seam for a forge adapter; the md prose bypasses them.
3. Project board operations are the most GitHub-specific part (no `gh pr`/`issue` equivalent in GitLab). `glab` has no Projects v2 analogue (GitLab boards are label-driven), so the Status-column model is the hardest piece to port.

## 1. Per-file inventory

Legend for ops: R = read, W = write. "Shape" = fields parsed from gh JSON.

### 1.1 Scripts (`scripts/`)

| File | gh / graphql ops | Implicit GitHub assumptions |
|---|---|---|
| `super-board-github-read.py` | Wraps every R: `gh <args>` (l.330); `--kind` validators: issue `--json number,title,body`; comments `api repos/{owner}/{repo}/issues/N/comments --paginate --slurp`; head `pr view --json headRefName,headRefOid`; body; metadata `pr view --json files,labels,body,additions,deletions,changedFiles`; diff `pr diff`; state `issue view --json state`; merge-state `pr view --json state,headRefOid,mergeCommit`; prs-open / prs-merged `pr list --json ...`; issues-closed `issue list --json ...,stateReason`; issues = GraphQL `repository.issues(states:OPEN) {labels, comments(last:8)}`; also `--kind fields/project/quota` used by run.sh. Line 409 GraphQL issue-history query. | Retry classifier `permanent_error` regex-matches GitHub error strings (`Could not resolve to a`, `Resource not accessible`, `missing required scopes`, `GRAPHQL_VALIDATION_FAILED`, `HTTP 40[134]`); `read_only()` decides retry-safety from gh verbs (`pr|issue|project|repo` view/list/diff/checks/item-list/field-list) and GraphQL `query` vs `mutation` text; rate payload shape `.resources.graphql/.core` (l.247); `closingIssuesReferences` pagination (l.279); halt file `github-halt.json`; state/exit codes (64/79) tuned to gh. |
| `super-board-card.sh` | R GraphQL: `repositoryOwner{... on ProjectV2Owner{projectV2{id field(name:"Status"){... on ProjectV2SingleSelectField{id options{id name}}}}}}`; `repository.issueOrPullRequest{projectItems{nodes{id project{number owner} fieldValueByName(name:"Status")}}}`; paginated `projectV2.items(first:100){nodes{id fieldValueByName content{... on Issue/PullRequest/DraftIssue}}}`. W GraphQL mutation `updateProjectV2ItemFieldValue(singleSelectOptionId)`. | Hardcoded field name `"Status"`; PVT_/PVTSSF_/PVTI_ opaque node IDs cached in `.claude/super-board/cache/board-<owner>-<n>.json`; option IDs resolved by name (cache-dropped on mismatch, because `updateProjectV2Field` remints IDs); remote URL sed strips `https://github.com/` and `git@github.com:` (l.68); re-emits `gh project item-list` JSON shape so callers (`wave-plan.sh`, `run.sh`) stay unchanged; point cost model in header (101 pts/100 items etc.). |
| `super-board-run.sh` | R: `--kind fields -- project field-list`, `--kind project -- project view`, `--kind state -- issue view -q .state`, `--kind quota -- api rate_limit`, board items via `card.sh items`. W (all via `gh_write_once`): `project item-edit --id --project-id --field-id --single-select-option-id` (l.168), `issue edit --add-assignee/--remove-assignee $BOT_LOGIN` (l.300, 223, 403, 482, 552), `issue comment --body "[orchestrator] [report] ..."` (l.555). | `.status` string compared to column names (Done/Ready/Building...); assignee = claim lock (race lost when `issue edit` fails, l.296); bot robot-account login (`BOT_LOGIN`); `project item-edit` needs project node ID + field ID; quota via REST `rate_limit` then GraphQL correction (`sb_gh_quota_merge`); closed-issue state "CLOSED" from `issue view`. |
| `super-board-merge-gate.sh` | R: `pr view --json headRefName,headRefOid`, `--json state,mergeCommit,headRefOid`, `--json labels,files,additions,deletions,changedFiles,body`, `pr diff`. W: `gh pr merge --squash --delete-branch --subject --body-file --match-head-commit` (l.349). | Squash-only; `--match-head-commit` is the GitHub head-pin; merged state inferred from `state==MERGED` + `mergeCommit.oid` after a lost response; header comment refers to `mergeable`/`mergeStateStatus` (explicitly not trusted); `PR` = integer number; runs optional `cleanup-wt.py` after merge. |
| `super-board-merge-policy.py` | Consumes `gh pr view --json labels,files,additions,deletions,body` and `gh pr diff` output via files (no gh call). | Shape `labels[].name`, `files[].path`, diff only added lines. |
| `super-board-approval.py` | R: GraphQL `repository.pullRequest{headRefOid state baseRefName body closingIssuesReferences(first:100)}` + `repository.defaultBranchRef`; REST `repos/<repo>/issues/<n>/comments?per_page=100`. | Human sign-off bound to PR head SHA; closing issues only counted when PR targets default branch (GitHub rule, l.169) and `nameWithOwner` equals this repo; approval is detected from issue comment text; `GITHUB_READ` classifier of "approval"/"comments" inferred from args (l.145). |
| `super-board-pr-body.sh` | R: `--kind body -- api repos/{owner}/{repo}/pulls/$PR --jq {headRefOid, body}`. W: `gh api -X PATCH repos/{owner}/{repo}/pulls/$PR --input -` (l.163). | Chosen over `gh pr view/edit` for rate cost (REST 1 vs GraphQL); `GH_REPO` env fills `{owner}/{repo}`; exit 70 for gh failure; body markers edited in place; head-moved exit 6. |
| `super-board-status.py` | R: GraphQL `ITEMS_QUERY` (project items: number, title, labels, Status) via `gh api graphql`; `gh issue view N --json comments` per Blocked card. | Same ProjectV2 owner query; `Blocked` reason tags parsed out of issue comments; exit 67 on gh failure; `fieldValueByName`/`content.number` shapes; max 100x pages. |
| `super-board-setup.py` | R: `project field-list`, `project list --owner`, `project view`, `api repos/<repo>/labels?per_page=100 --paginate --slurp`, GraphQL `ITEMS_Q`/`ITEM_Q` (ProjectV2 items + `fieldValueByName("Status"){optionId}` + issue labels) and `OPTIONS_Q`. W: `label create`, `issue edit` (labels), `project item-edit`, GraphQL `updateProjectV2Field` (rewrites Status options, `UPDATE_M`). Requires `gh` binary listed as dependency (brew/apt/dnf/winget). | `COLUMNS = Backlog, Ready, Building, QA, Review, Blocked, Done` (l.61) and colour map (GRAY/BLUE/...); board-migrate rewrites whole option list then restores each card status (option rewrite reminting IDs is GitHub behaviour; see 3 fix commits); `gh auth` scopes `project,read:project,repo`; `ProjectV2SingleSelectFieldOptionInput` type; 500+ card snapshots via cursors. Largest single churn file (11 gh-touching commits). |
| `super-board-deps.sh` | R GraphQL via github-read `--kind issues` (open issues with labels, last 8 comments); `--from` accepts a saved `gh issue list --json` payload. | `blocked-by:` / `## Blocked by` lines with `- #N` refs parsed with jq regex `^[ \t]*-[ \t]*#[0-9]+` from bodies and comments (l.118-126); issue number = identity. |
| `super-board-preflight.sh` | R via github-read: `--kind prs-merged`, `prs-open`, `issues-closed`, `issue` per card (`pr list/issue list --json ...,stateReason`). | Duplicate/overlap detection by comparing PR `files` and bodies; `stateReason` is a GitHub issue field; limit 100. |
| `super-board-wave-plan.sh` | Reads board items JSON (from `card.sh items`), no direct gh. | Consumes `gh project item-list` shape (`labels` on item, or nested `content.labels`; l.137). |
| `super-board-gh-guard.sh` | R: `gh api rate_limit` (REST), `gh api graphql 'query{rateLimit{limit remaining used resetAt}}'` (l.40, measured because REST misreports GraphQL bucket). | Entire file encodes GitHub rate-limit model: GraphQL 5000 pts/h, REST core, thresholds 200/500, per-worker budget 150, sub-agent 50; `.resources.graphql.remaining/.reset`. Sourced by run, throttle, preflight, workers. |
| `super-board-throttle.sh` | R: `gh api rate_limit`; scans worker output for regex `rate.?limit(ed)? (exceeded|hit|reached)|api rate limit|secondary rate limit|graphql.{0,40}(quota|rate.?limit)|was submitted too quickly`. | GitHub rate-limit error strings. |
| `super-board-stop.sh` | R `gh pr list --state open --json number,headRefName --jq startswith("issue-N-")`. W `gh issue comment`, `gh pr comment`, `gh issue edit --remove-assignee/--remove-label`. | Branch name convention `issue-<N>-*`; label list; `[orchestrator] [report]` comment prefix. |
| `super-board-collect-state.py` | No gh call; imports github-read for halt (`gh.check_halt`). | Shares halt file. |
| `super-board-lane.sh`, `super-board-codex-lane.sh`, `super-board-codex-wave.sh` | No direct gh; worker prompts tell the agent to `source super-board-gh-guard.sh`, use github-read `--kind`, merge only via merge-gate with `--expect-head` (lane.sh l.179, 193; codex-wave.sh l.144 embeds `api repos/{owner}/{repo}/issues/N/comments`). | Prompt text embeds gh forms (Layer B in script clothing). |
| `super-qa-file-bug.sh` | W: `gh issue create`, `gh issue comment` (duplicate "Seen again"), `gh label create --force`, `gh project item-add --format json --jq .id`, `gh project item-edit`. R `gh issue list -l source:qa`. | Labels `source:qa`, `bug`; `project item-add` returns item node ID; `--owner`/`--url` of issue; best-effort label creation because `issue create` hard-fails on unknown label; exit 71 filed-not-placed. |
| `super-review-file-refactor.sh` | Same set as above: `issue comment`, `label create`, `issue create`, `project item-add`, `project item-edit`. | Same; `-R`/`REPO_FLAG` needed because worktree may have a different remote. |
| `super-board-skills-ready.py`, `-project-verifier.py`, `-verification-ready.py`, `-agents-md.py`, `-readme-sync.py`, `-calibration.py`, `-settings.py`, `-env-check.sh`, `-host.sh`, `-usage.sh`, `-card.sh` (see above) | No gh calls. `calibration.py:144` matches PR URL regex `https://[^/\s]+/[^/\s]+/[^/\s]+/pull/<N>` (host-agnostic but uses GitHub's `/pull/` path). | `/pull/` URL form; PR number in cards. |

### 1.2 Hooks (`hooks/`)

| File | ops | Assumptions |
|---|---|---|
| `cleanup-wt.py` | R `gh pr list --state open --limit 200 --json number,headRefName,baseRefName` and `--state merged --limit 300 --json headRefName,headRefOid` (l.184-187). Binary override `CLEANUP_WT_GH`. | Merged-PR tip OID == local branch tip is the "safe to delete" proof; 200/300 caps; degrades to "keep" when gh unavailable; env `GH_PROMPT_DISABLED/GH_NO_UPDATE_NOTIFIER/GH_PAGER`. |
| `guard-protected-push.py` | none (mentions that merges go through `gh pr merge`, so nothing needs to push a base). | Policy assumes the forge does the merge server-side. |
| `guard-key-literals.py` | none; regex `gh[pousr]_[A-Za-z0-9]{30,}` (GitHub token formats). | GitHub PAT prefixes. |
| other hooks (`guard-delete-outside`, `guard-secrets`, `guard-worktree-path`, `pre-commit-readme`, `dev/gate-skill-evals`) | none | none |

### 1.3 Workflows (`workflows/*.js`) and CI (`.github/`)

| File | ops | Assumptions |
|---|---|---|
| `workflows/super-board-wave.js` | No gh subprocess in JS; builds prompts for agents telling them to use `super-board-github-read.py --kind <kind> -- <gh args>` (l.45-48), `issue comments: --kind comments -- api repos/{owner}/{repo}/issues/<N>/comments --paginate --slurp (zero comments = [[]])`; comments about `gh project item-list` cost (~203 pts) and `field-list` (101). Preflight agents batch the "three gh list calls" (l.207). | `blocked-by:` rule; gh JSON shapes taught to the agent (Layer B). Second-highest churn among call-site files (5 gh-touching commits). |
| `workflows/ui-refine-loop.js` | none | none |
| `.github/workflows/release.yml` | `GH_TOKEN: ${{ github.token }}` -> `.github/scripts/publish-release.sh` | Tag push `v*` trigger; this is the pack's own release CI, not the product's forge use. |
| `.github/scripts/publish-release.sh` | `gh release create "$TAG" --verify-tag --target $SHA` | pack's own release process. |
| `.github/workflows/cross-platform.yml`, `pages.yml` | `actions/*` only | Pack's own CI on GitHub Actions/Pages. Not in the product's runtime coupling. |

### 1.4 Installer / bootstrap

| File | ops | Assumptions |
|---|---|---|
| `get.sh` | `gh` required tool check (l.97); `curl https://api.github.com/repos/$repo/releases/latest`, `codeload.github.com/.../tar.gz/$ref`, `git clone https://github.com/$repo.git`. | Distribution is from GitHub releases/tarballs; repo slug `EricTechPro/super-board`. |
| `install.sh` | Copies `super-board-github-read.py`, `gh-guard.sh`, `card.sh`, `collect-state.py` etc. to `.claude/bin` (l.229, SCRIPTS list). | Script names carry `gh`/`github` (these 10 install.sh churn commits are mostly list edits, not call-site changes). |
| `.claude-plugin/plugin.json`, `marketplace.json` | homepage/repository URLs, keyword `github-projects`. | Distribution metadata. |

### 1.5 Skills (`skills/*`)

Executable parts

| File | ops | Assumptions |
|---|---|---|
| `skills/super-collect/scripts/collect_prs.py` | R `gh api graphql` search (paginated on search cursor) over merged PRs (l.44-51). | GitHub search syntax `q`, GraphQL `search` connection shape; fixture `tests/fixtures/collect/prs-graphql.json`. |
| `skills/super-collect/scripts/super-collect-file.sh` | W: `gh project item-add`, `project view --format json --jq .id`, `project item-edit`, `label create --force`, `issue edit --add-label`, `issue comment` ("Seen again by super-collect"). | Fingerprint markers `<!-- super-collect-fingerprint: ... -->`, `<!-- super-collect-source: ... -->` stored in issue body (dedupe state lives in the forge); labels `source:collect`, `collect:<src>`; exit 70/71. |
| `skills/super-collect/scripts/collect_{common,custom,handoff,posthog,sentry}.py` | none to GitHub (Sentry/PostHog APIs only; "issues" there are Sentry issues). | `collect_handoff`/`custom` hand candidates to `super-collect-file.sh`. |
| `skills/super-build/scripts/super-build-dispatch.sh` | R `gh issue view #N` (error message at l.101). | Branch `loop/issue-N`, worktree `.claude/worktrees/issue-N`, commit `close #N`. |
| `skills/ui-refine-loop/scripts/pr-shots.sh` | none (builds `<img src="https://github.com/%s/raw/%s/%s">` at l.76). | Raw-URL scheme of GitHub for embedding screenshots in PR body. |
| `skills/visual/scripts/visual.py` | none; fetches `https://github.com/<login>.png?size=96` (avatar, l.436) and `--source-base https://github.com/OWNER/REPO/blob/main/` docs. | Cosmetic. |
| `skills/super-qa/scripts/*` | none | none |

Agent-prose call sites (Layer B). Count of literal `gh` commands in the md by file (non-exhaustive of ops):

| File | Count | Main ops told to agents |
|---|---|---|
| `skills/super-build/SKILL.md` | 23 | `project item-list`, `project list --owner`, `auth status`, `api repos/{owner}/{repo}/issues/<N> --jq .body`, `label create loop:in-progress/loop:halted/human-gated`, `issue edit --add-label/--remove-label`, `issue comment` (`[builder] [report]`), `issue view --json title,body,labels`, `pr create --draft --base --head loop/issue-N`, `api graphql resolveReviewThread` |
| `skills/super-board/references/run.md` | 19 | `auth refresh -s project,read:project,repo`, `api rate_limit`, `api graphql` (review threads), `pr view --json comments`, `api -X PATCH repos/<owner>/<repo>/issues/comments/<id>`, `pr merge --auto`, `pr ready`, `pr view --json state,mergeCommit`, `pr view --json headRefOid` |
| `skills/super-board/references/status.md` | 18 | `api graphql`, `project item-list --format json`, `issue list --assignee`, `issue view --json comments`; a deny-list of write verbs (`issue edit`, `project item-edit`, `pr edit`, `label edit`, `repo edit`, `issue create`, `pr create`, `project item-create`, `label create`, graphql mutations incl. `resolveReviewThread`) |
| `skills/super-board/references/second-account.md` | 18 | robot account, `gh auth login/switch`, scopes (`gist`, `read:org`, `repo`, `project`), `api user/users` |
| `skills/super-board/references/rate-limit-etiquette.md` | 18 | GraphQL points price list per command (item-list 101/100 cards, field-list 101, project view 2, view/list 1, comment/edit ~2), REST alternatives (`api repos/{owner}/{repo}/pulls/N`) |
| `skills/super-board/references/run-workflow.md` | 15 | merge/ready/comment forms |
| `skills/super-board/references/onboard.md` | 14 | `auth status/login/refresh`, `repo create`, `project create --owner`, `project field-list`, `label list`, `Bash(gh project:*)` permission allowlist, `api graphql '{viewer{login}}'` |
| `skills/super-review/SKILL.md` | 7 | `pr view --json comments`, PATCH issue comment, `project item-list`, `pr ready`, `pr view headRefOid/state,mergeCommit` |
| `skills/super-qa/SKILL.md` | 7 | `project list`, `repo view --json owner`, `issue list -l source:qa`, `issue create`, `pr view baseRefName`, `api graphql resolveReviewThread` |
| `super-qa/references/iteration-preamble.md`, `super-board/references/stop.md`, `docs/super-board/README.md`, `docs/hosts.md`, `super-collect/SKILL.md`, `lint.md`, `pr-security-review/SKILL.md`, `git-sync/SKILL.md`, `README.md`, `CLAUDE.md`, `RELEASING.md` | 1-5 each | mention-level |
| `skills/super-board/references/config-schema.json` | 4 `github.com`/`project` hits | schema: `project.owner`, `project.number`, `repo.remote` (config is GitHub-shaped: owner + Projects v2 number) |

Cross-cutting prose conventions (assumptions agents rely on, all GitHub-hosted state):
- Card identity = issue number; PR linked via `Closes #N` closing keyword (`block-template.md:144`, approval.py).
- Dependencies = `blocked-by:` line / `## Blocked by` section with `- #N` bullets in the issue body (`run.md` 10x, `block-template.md` 8x, `super-board-deps.sh` 8x).
- Status machine = Projects v2 single-select `Status` column names; `block-template.md` Blocked reason emoji tags (🔑 "gh scope denied", etc.).
- Comment prefixes `[orchestrator] [report]`, `[builder] [report]`, `[qa]` review threads, hidden HTML markers in issue bodies.
- Labels as protocol: `source:qa|collect|review`, `collect:<src>`, `loop:in-progress`, `loop:halted`, `human-gated`, type/complexity labels from `ticket-format.md`.
- PR draft -> `gh pr ready` -> `gh pr merge --squash`; review threads resolved with GraphQL `resolveReviewThread`.

## 2. How tests and evals mock `gh`

Mechanism everywhere: **PATH shadowing.** A fake executable named `gh` is written into a temp `bin/` that is prepended to `PATH` (or env var override `CLEANUP_WT_GH`; `patch.object(setup.subprocess, 'run', ...)` for Python in-process). No test touches the network. Nothing mocks the *forge model*; every fake answers in GitHub's exact JSON shape, so tests would pass or fail against a GitLab port only after the fake is rewritten per file.

Tests with their own inline fake `gh` (about 20 files, each a different hand-written stub, no shared helper):

| Test | Fake style |
|---|---|
| `tests/test-card.sh` | bash; classifies the call by GraphQL query text (`mutation`, `field(name:"Status")`, `issueOrPullRequest`, `items(first:100`), logs `kind args` to `$FAKE_LOG`, canned `PVT_1/PVTSSF_1/PVTI_232` payloads, `fail-mutation-once` flag to simulate stale option IDs. Asserts call counts (cold move = 3 calls, warm = 1). |
| `tests/test-merge-gate.sh` | bash stub keyed on `$1 $2` (`pr view` with `state,mergeCommit` / `labels`, `pr merge`, `pr diff`, `api graphql` closingIssuesReferences, `api --paginate` comments); env knobs `STUB_OID`, `STUB_OID_AFTER`, `STUB_STATE_AFTER`, `STUB_MERGE_RC`, `STUB_META`, `STUB_APPROVAL`... to simulate head moved, lost merge response. Also an `exit 1` gh (l.21) for gh-unavailable. |
| `tests/test_github_reads.py` | python stub `gh` replaying `READ_RESPONSES` JSON (`rc,out,err`) by call count; tests retry (3 attempts), halt file, shape validation, quota guard. |
| `tests/test_upgrade_recovery.py` (114 gh hits) | in-process stateful `GitHub` class (items, option generation, `fail`/`after` hooks, 501-item pagination) patched over `setup.subprocess.run`; models option-ID reminting and crash-mid-rewrite. The deepest GitHub model in the repo. |
| `tests/test-setup.sh` (47) | python `gh` stub for `project field-list/list/view`, GraphQL, label and issue edit. |
| `tests/test-collect-file.sh`, `test-file-bug.sh`, `test-file-refactor.sh`, `test-pr-body.sh`, `test-preflight.sh`, `test-run-gates.sh`, `test-throttle.sh`, `test-codex-lane.sh`, `test-codex-wave-contract.sh`, `test-codex-stop.sh`, `test-collect-baseline.sh`, `test-release.sh`, `test-get.sh` (no-op `exit 0`), `test-cleanup-wt.sh` (`echo "[]"`) | bash/python stubs that log argv and answer a handful of verbs. |
| `tests/test_approval.py`, `test_collect_baseline.py`, `test_worktree_paths.py`, `test_skill_dispatch.py`, `test_status_parse.py` | smaller stubs / fixture payloads. |
| `tests/fixtures/collect/prs-graphql.json`, `tests/fixtures/deps-issues.json`, `wave-*.json` | recorded GitHub JSON. |

Highest gh-density tests: `test_upgrade_recovery.py` 114, `test-setup.sh` 47, `test-merge-gate.sh` 36, `test-collect-file.sh` 36, `test-collect-baseline.sh` 33, `test-file-refactor.sh` 26, `test-file-bug.sh` 24, `test-run-gates.sh` 23.

Doc/lint tests that assert prose: `test_writing_format.py`, `test-hosts-doc.sh` (docs/hosts.md), `test_run_allowlist.py` (gh permission allowlist).

## 3. Evals (which touch the forge)

`evals/_stubs/bin/gh` is a single shared 100-line offline stub (the only shared fake in the repo). It reads fixtures from `<repo>/.git/gh-fixture/` (`pr.json`, `diff.patch`, `issue.json`, `issues.json`, `prs-merged.json`, `project-items.json`, `fields.json`, `graphql.json`), logs every call to `.gh-log` and `.git/gh-calls.log`, honours `--json`, `--jq`, `--state`, `--label`, `--search`, records writes (`.git/gh-created.md`, `.git/gh-comments.md`), `pr merge` is a logged no-op. Verbs it implements: `auth status`, `pr view|list|diff|checks|merge|comment|review|edit|ready`, `issue list|create|comment|edit|view`, `repo view`, `project list|view|item-list|field-list|item-add|item-edit`, `label create`, `api rate_limit`, `api graphql` (rateLimit special-cased, else `graphql.json`), `api *` (returns `{}`). Hardcoded: Status options Backlog/Ready/In progress/Done default; fake URLs `https://github.com/example/shop/issues/N`, project ids `PVT_eval`, `PVTSSF_status`.

All 7 evals touch the forge through this stub; graders inspect `.gh-log` / `gh-calls.log`:

| Eval | Forge touch |
|---|---|
| `collect-duplicate` | `issues.json` (#301 open, fingerprint), `project-items.json`; grader: "Seen again" comment on #301, no `issue create`. |
| `collect-posthog-gap` | `issues.json`, `project-items.json`, `prs-merged.json`; grader: planned-gap ticket, no signup gap. |
| `collect-prs-pattern` | `graphql.json` (collect_prs search), `merged.json`, `issues.json`, `project-items.json`. |
| `collect-sentry-verdicts` | `issues.json`, `project-items.json`, `prs-merged.json`; grader: #4411 filed, #4490 not, ticket format. |
| `test-gap` | `pr.json`, `issue.json`; grader `no-merge` (no `pr merge` call). |
| `ponytail-overengineering` | `pr.json`, `issue.json`, `formatters.json`; grader `no-merge`. |
| `review-remembers` | `pr.json`; graders `no-merge`, `loaded-prior`. |

Plus `evals/_stubs/collect-seed.sh` (shared scaffold writing fixtures), `evals/*/seed.sh`. `hooks/dev/gate-skill-evals.py` is a dev gate on evals (no gh). Cases are GitHub-vocabulary in prompts and graders (e.g. `no-merge.md` checks `gh pr merge`).

## 4. Churn analysis (git log)

Caveat on history: 119 commits, 2026-05-24 .. 2026-11-04. Pre-v3 history is coarse (roughly one commit per release: v1.2.0 .. v2.6.0 = 1-6 commits each); v3.0.0 onward is granular (96 commits after tag v3.0.0, tag `archive/pre-v3-rewrite-2026-10-02` exists). Local tags exist only through v3.1.2, so releases were mapped by the `VERSION` file content at each commit (VERSION skipped 3.2.0 locally; changes between v3.1.2 and v3.3.0 are labelled 3.1.2 / 3.3.0 by VERSION content; RELEASE-NOTES lists v3.2.0 on 2026-10-26 and v3.3.0 on 2026-10-27, so some commits under "3.1.2" below belong to the 3.2.0 release). Script: scratchpad `churn.sh`; command base `git log -G'<pattern>'`.

### 4.1 Headline numbers

| Metric | Value |
|---|---|
| Commits in repo | 119 |
| Commits whose diff adds/removes a gh call-site line in executable code (scripts, hooks, workflows, .github, skill scripts, install/get) | **27 of 119 (23%)**; 17 of them since v3.0.0 (17 of 96 = 18%) |
| Same, in tests/evals | 19 (13 since v3.0.0) |
| Same, in docs/skills prose | 16 (5 since v3.0.0). Under-counts prose: pattern is code-oriented. |
| Gh-pattern line delta in executable code since v3.0.0 | +166 / -85 lines |
| Commits touching GraphQL query text (`graphql|query(|mutation(|projectV2`) in code | 13, spanning 2026-05-24 .. 2026-10-29 |
| Commits touching any `--json <fields>` list in code | 15 |
| Scope of v3.0.0..HEAD code churn | 40 commits touched scripts/workflows/hooks/skill scripts: 36 files, +4649/-381 |
| Code files with at least one gh call/abstraction (approx) | scripts 20, hooks 1, workflows 1, skill scripts 4, CI/installer 4 |

### 4.2 Files that churn most on gh lines (commits that touch gh-pattern lines / total commits touching file)

| Commits (gh lines) | Total commits | File | Note |
|---|---|---|---|
| 11 | 17 | `scripts/super-board-setup.py` | board migrate/rank; ProjectV2 option rewrite fixes |
| 10 | 27 | `install.sh` | mostly script-list edits (false positive for call sites) |
| 5 | 18 | `workflows/super-board-wave.js` | agent prompts on gh forms/cost |
| 5 | 9 | `tests/test-setup.sh` | follows setup.py |
| 4 | 13 | `scripts/super-board-run.sh` | |
| 4 | 9 | `scripts/super-board-wave-plan.sh` | |
| 4 | 6 | `scripts/super-board-github-read.py` | the funnel; created v3.0.5/3.1.x |
| 4 | 4 | `scripts/super-board-gh-guard.sh` | every commit on it touches gh lines |
| 3 | 9 / 8 / 6 | `super-qa-file-bug.sh`, `super-collect-file.sh`, `super-review-file-refactor.sh`, `super-board-codex-lane.sh` | |
| 3 | 6 | `tests/test-collect-file.sh`, `tests/test_github_reads.py` | |
| 2 | 3 | `scripts/super-board-card.sh`, `super-board-approval.py` | young files, heavy churn ratio |
| 1 | 11 / 12 | `super-board-merge-gate.sh`, `tests/test-merge-gate.sh` | merge-gate is stable on gh lines |

`gh-guard.sh`, `github-read.py`, `card.sh`, `approval.py`, `pr-body.sh` are all v3-era and are the newest, most-changed gh surface: the "rate limit / read-shape / halt" machinery was built Oct 3-29 in response to production incidents (RELEASE-NOTES v3.0.5, v3.1.0 "~20x fewer GitHub calls", v3.1.2, #24 "pause runs when required reads fail", #33 and #89 "malformed agent reads exit 64 instead of halting").

### 4.3 Per-release (version at commit): commits / commits touching gh code lines / distinct code files touched

| VERSION | Commits | gh-code commits | files |
|---|---|---|---|
| 1.2.0 | 1 | 1 | 2 |
| 1.3.0 | 2 | 1 | 1 |
| 1.4.0 / 1.7.1 / 2.1.0 / 2.1.1 | 1 each | 0 | 0 |
| 1.6.0 / 2.0.0 / 2.3.0 / 2.4.0 | 1 each | 1 each | 1-2 |
| 2.5.0 | 5 | 1 | 2 |
| 2.6.0 | 6 | 3 | 5 |
| 3.0.0 / 3.0.1 / 3.0.2 / 3.0.3 | 3 / 1 / 1 / 8 | 0 | 0 |
| 3.0.4 | 3 | 1 | 1 |
| 3.0.5 | 6 | 3 | 15 |
| 3.1.0 | 1 | 1 | 11 (the "~20x fewer GitHub calls" rewrite: board reads moved to GraphQL, card.sh, robot account) |
| 3.1.1 | 24 | 4 | 6 |
| 3.1.2 (includes 3.2.0 work, see caveat) | 40 | 8 | 12 |
| 3.3.0 | 10 | 2 | 3 |

Of the 22 distinct VERSION states, **14 version states touched gh call sites in code, 8 did not** (the 8: 1.4.0, 1.7.1, 2.1.0, 2.1.1, 3.0.0-3.0.3, mostly one-commit or doc/test-only states). From v3.0.4 on, every version state (3.0.4, 3.0.5, 3.1.0, 3.1.1, 3.1.2, 3.3.0) altered gh lines.

Commits that touched gh call sites in executable code since v3.0.0 (17 by the broader pattern; the 15 below are those matching a narrower scripts/hooks/workflows/collect/build/qa path set, oldest first): `e8568e6` approval binds sign-off to head; `d5ef1c8` setup restores statuses after option rewrite; `b8dd7b4` upgrade state; `d6868fc` pause runs when required reads fail; `3426007` v3.1.0 20x fewer calls; `6332f63` gh guard sourceable from zsh; `35e2156` show read shapes; `9797741` verification before board merges; `ba7a984` calibration; `ef48346` collect repeated intake safe; `a848506` dispatch host-visible skills; `d4a5da4` project verifiers; `cc38679` generic lane launcher; `95ffe9e` state in submodule; `17a79c8` malformed reads exit 64.

### 4.4 What kind of change hits gh lines

1. **Cost and rate-limit tuning** (v3.0.5, v3.1.0, throttle, gh-guard): query shape and point costs. Upstream will keep tuning this; it is GitHub-specific and will conflict on any fork that swaps the transport.
2. **Reliability/halt semantics** (#24, #33, #88, #89): behaviour of the `github-read.py` funnel, `--kind` forms and exit codes.
3. **Board model changes** (setup.py `COLUMNS`, option rewrites, Blocked column v2.4.0, onboarding "three labels, no Skipped column" 9ec03ab).
4. **New collectors / filers** (collect, qa-file-bug, refactor): copy-pasted `project item-add` -> `item-edit` sequence in 3 scripts (`super-collect-file.sh`, `super-qa-file-bug.sh`, `super-review-file-refactor.sh`), so a board-API change needs three coordinated edits.

## 5. Upstream-sync pain assessment

- **Call-site count is moderate but diffused.** About 25 executable files and about 27 prose files; gh-line churn is concentrated: the highest-churn call-site files are `setup.py`, `run.sh`, `github-read.py`, `gh-guard.sh`, `card.sh`, `wave-plan.sh`, `wave.js`, plus the three filers.
- **Upstream changes gh lines in about 1 of 5 commits (27/119 all-time, 17/96 since v3.0.0) and in every version state since v3.0.4.** Expect a merge conflict in the gh-adapter files on nearly every upstream sync during the current v3.x phase. Pre-v3 style (1 commit/release) hid this; v3 granularity is the real signal.
- **Churn is clustered on the funnels** (`github-read.py`, `card.sh`, `gh-guard.sh`, `run.sh`). If the port is a fork that replaces these in place, every upstream fix to them conflicts. If instead the port keeps these files as-is and adds a forge-adapter layer behind a stable verb set (read-pr, read-issue, list-prs, move-card, merge, comment, rate-quota), conflicts shrink to the call sites that bypass the funnels: the three filers, `stop.sh`, `pr-body.sh`, `merge-gate.sh`, `cleanup-wt.py`, `approval.py`, `setup.py`, `status.py`.
- **Prose is the hidden tax.** Agent-facing md (run.md, super-build/SKILL.md, status.md, second-account.md, rate-limit-etiquette.md, onboard.md) literally contains `gh` command lines and price tables. These change with the code; the tests partly pin them (`test-hosts-doc.sh`, `test_run_allowlist.py`, `test_writing_format.py`). A port needs either template substitution of forge verbs or a parallel doc set.
- **Tests are not portable assets.** About 20 tests each own a hand-written GitHub-shaped fake; only evals share one stub. A forge adapter would let tests mock at the adapter boundary instead, but upstream tests will keep arriving in the gh-shaped form.
- **Hardest-to-port bits:** Projects v2 board (card.sh, setup.py, run.sh `item-edit`, status.py ITEMS_QUERY, 3 filers), `closingIssuesReferences` approval binding, `--match-head-commit` merge pin, GraphQL point budgeting (gh-guard, rate-limit-etiquette, throttle), `resolveReviewThread`, `gh auth` scope onboarding.
- **Easy-to-port bits:** `cleanup-wt.py` (just pr list shapes), `collect_prs.py` (single search query), `deps.sh` (issues + `#N` regex), `merge-policy.py` (file-based), `preflight.sh`, CI/release (pack's own, irrelevant to product).

## 6. Gaps / not verified

- Did not run the tests or evals; mocking descriptions are from reading the stubs.
- Per-release table uses `VERSION` content at each commit, which is approximate near v3.1.2/v3.2.0/v3.3.0 (no local tags past v3.1.2).
- md call-site counts are literal `gh <verb>` occurrences, not distinct operations; one fenced example can appear in several files.
- Docs-area `git log -G` count under-counts prose churn.
