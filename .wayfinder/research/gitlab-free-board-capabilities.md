# What can a gitlab.com Free board express?

Ticket: `.wayfinder/tickets/001-gitlab-free-board-capabilities.md`. Fetched 2026-11-04 from docs.gitlab.com (tier badges as rendered). Unverified items are marked.

## Capability table

| Item | Tier | Source / note |
|---|---|---|
| Project issue boards, multiple per project | Free | https://docs.gitlab.com/user/project/issue_board/ (tier table: project boards "Multiple") |
| Group issue board, exactly 1 | Free | same page: "Users on GitLab Free can use a single group issue board." |
| Multiple group boards | Premium+ | same page; API: create/delete group board badged Premium, Ultimate (https://docs.gitlab.com/api/group_boards/) |
| Group board spans all projects in group and subgroups | Free | same page: "display issues from all projects that fall under the group and its descendant subgroups" |
| Project board showing other projects' issues | Not stated in docs (assume no) | same page only claims this for group boards |
| Label lists (column = label) | Free | same page, not badged |
| Open (leftmost) / Closed (rightmost) lists; drag to Closed closes issue | Free | same page |
| Drag card between label lists: removes source label, adds target label | Free | same page: "removes the label from the list it came from and adds the label from the list it goes to" |
| Board card order persisted and shared | Free | same page |
| Assignee / Milestone / Iteration / Status lists | Premium+ | same page; GraphQL `boardListCreate` has `assigneeId`, `milestoneId`, `iterationId`, `statusId` (https://docs.gitlab.com/api/graphql/reference/mutations/) |
| Configurable board scope (milestone/labels/assignee/weight/iteration filter) | Premium+ | same page; REST update params "Premium and Ultimate only" (https://docs.gitlab.com/api/boards/) |
| WIP limits, blocked indicators, epic swimlanes, sum of weights | Premium+ | same page |
| Scoped labels (`key::value`, one per key) | Premium+ | https://docs.gitlab.com/user/project/labels/ ("Tier: Premium, Ultimate") |
| Plain labels, group labels (inherited by subgroup projects), project labels | Free | same page |
| Label priority (star) | Free (no badge) | same page |
| Custom fields (single-select, multi-select, number, text) | Premium+ | https://docs.gitlab.com/user/work_items/custom_fields/ (defined on top-level group) |
| Work item Status (custom statuses, Triage/To do/In progress/Done/Canceled) | Premium+ | https://docs.gitlab.com/user/work_items/status/ (GA 18.4). Quote: "more granular tracking than the traditional binary open/closed state system used in GitLab Free" |
| Assignees: one per issue | Free | https://docs.gitlab.com/user/project/issues/multiple_assignees_for_issues/ |
| Multiple assignees | Premium+ | same page |
| Issue weight | Premium+ | https://docs.gitlab.com/user/project/issues/issue_weight/ ; REST `weight` param Premium (https://docs.gitlab.com/api/issues/) |
| Health status | Ultimate | https://docs.gitlab.com/user/project/issues/managing_issues/ |
| Linked issues: "relates to" | Free | https://docs.gitlab.com/user/project/issues/related_issues/ |
| Linked issues: "blocks" / "is blocked by" | Premium+ | same page |
| Milestones (project and group) | Free | https://docs.gitlab.com/user/project/milestones/ (burndown tier not confirmed) |
| Iterations | Premium+ | managing_issues page |
| Epics (parent of issues) | Premium+ | managing_issues page; REST `epic_id`/`epic_iid` Premium |
| Issue types (change issue type) | Free (no badge) | managing_issues page |

## API

| Item | Tier / fact | Source |
|---|---|---|
| REST boards: CRUD boards, CRUD board lists (`/projects/:id/boards[/:board_id/lists]`) | Free (scoping params Premium) | https://docs.gitlab.com/api/boards/ |
| REST group boards: list/get Free; create/delete Premium; list-create params assignee/milestone/iteration/status Premium | mixed | https://docs.gitlab.com/api/group_boards/ |
| REST: no "move card" endpoint on boards page | fact | boards API page. Move = edit issue labels |
| REST edit issue: `add_labels`, `remove_labels`, `labels`, `state_event`, `assignee_ids` | no tier badge | https://docs.gitlab.com/api/issues/ |
| REST list issues across projects: `GET /groups/:id/issues` (scope=all default); label filters `labels` (AND), `not[labels]`, `labels=None/Any` | Free | same page |
| GraphQL `issueMoveList` (boardId!, iid!, fromListId, toListId, moveBeforeId/moveAfterId, positionInList) | exists; Free assumed for label lists (not verified on Free account) | https://docs.gitlab.com/api/graphql/reference/mutations/ |
| GraphQL limits: complexity 250 authed / 200 unauth; default page 100 nodes; 30s timeout; 10,000 char query; `read_api` for queries, `api` for mutations | fact | https://docs.gitlab.com/api/graphql/ |
| Rate limit, current: 2,000 req/min per user authenticated API; unauth 500/min per IP | fact | https://docs.gitlab.com/user/gitlab_com/rate_limits/ |
| Rate limit, PROPOSED per-plan (page says not yet in effect; lower applies once effective): Free 100/min burst + 5,000/hr; Premium 1,250/min + 15,000/hr | fact, watch | same page |
| 429 + `Retry-After`, `RateLimit-ResetTime`; `RateLimit-Limit/Remaining` headers (not on Projects/Groups/Users APIs) | fact | same page |
| Webhooks: Free 500 calls/min per top-level namespace | fact | https://docs.gitlab.com/user/gitlab_com/ |

Not verified: about.gitlab.com/pricing/feature-comparison is JS-rendered; fetch returned no tier rows, so tiers above come from docs.gitlab.com badges only.

## Implications for modelling lanes

1. Lane = label list. No single-select field, no Status field, no scoped labels on Free. The only column mechanism is label lists plus the built-in Open and Closed lists.
2. Mutual exclusivity must be enforced by our code. Board drag (and `issueMoveList` with fromListId) swaps source/target label, but manual labelling, the REST `labels` set, or other tools can leave two lane labels. Rule: every lane change is one `PUT .../issues/:iid` with `remove_labels=<all other lane labels>&add_labels=<new>` (or `labels=` full set). Add a reconcile/lint pass that reads label sets and repairs duplicates. Use a distinct prefix like `lane-` or a naming convention (unscoped `lane:in-review` is just a name, no semantics).
3. Done = Closed list (closing the issue), not a label. Backlog = Open list (issues with no lane label), so keep lane labels off the backlog.
4. Multi-repo: one group board only (Free). Put all repos under one group (subgroups OK), create lane labels as GROUP labels so all projects share them, and use the single group board. Project boards cannot be assumed to span repos. If boards must span different top-level groups, Free cannot do it; fall back to our own aggregation via `GET /groups/:id/issues`, or one board per group (but each group gets only one group board).
5. Single assignee only: model agent/human ownership with one assignee; extra roles (reviewer, worker) go in labels or issue text.
6. No weight, iteration, epic, custom field, or health: store priority, size, or any metadata as plain labels or in a structured issue-description block; do not rely on filters over them.
7. Dependencies: only "relates to" is Free. Model blocked-by as a `blocked` label plus "Blocked by #N" text we parse, or a relates link with convention. Native blocked indicators are Premium.
8. Milestones are Free and can serve as epoch/release grouping.
9. Rate limits: budget for the proposed Free ceiling (100 req/min, 5,000/hr per user) not the current 2,000/min. Prefer one GraphQL query per board (stay under complexity 250, page 100) or `GET /groups/:id/issues` with `per_page=100` over per-issue calls, honour `Retry-After`, and use webhooks (500/min) rather than polling.
10. Open question for a later ticket: whether `issueMoveList` and group-board GraphQL work with label lists on a real Free account (docs give no Free/Premium split for that mutation).
