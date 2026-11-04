# Sandbox verification on gitlab.com Free (2026-11-04)

The checks ran with glab 1.123.0, OAuth login as `yaroslav.y`, against group `yaroslav-sb-sandbox` (id 146588906) and its projects `app-a` (88478146) and `app-b` (88478148). The raw JSON samples are in `research/sandbox/`.

## Boards
- REST `POST groups/:id/boards` returns **403** on Free. GraphQL `createBoard(groupPath:)` **works for the first board** ("factory", `gid://gitlab/Board/11741004`). A second board fails with "You don't have the permission", which confirms one group board on Free.
- GraphQL `boardListCreate` with a GroupLabel works.
- **The board-list order equals the issue `relativePosition` ascending.** Verified after two reorders.
- REST `PUT projects/:id/issues/:iid/reorder` works. Its naming is inverted: `move_after_id=X` puts the issue *before* X, and `move_before_id=X` puts it *after* X.
- **`issueMoveList` fails** on a Free group board, both for reordering within a list and for moving between lists ("Item could not be moved"). Move cards by editing labels: `glab issue update --label lane:x --unlabel lane:y` works.

## glab JSON
- `glab issue view -F json` and `glab issue list -O json` have the same keys. **The list flag is `-O`, not `-F`.** `-F` on `issue list` means urls/ids/details.
- `glab mr view -F json` / `glab mr list -F json`: list lacks `changes_count`, `diff_refs`, `diverged_commits_count`, `head_pipeline`, `pipeline`, `merge_error`, `rebase_in_progress`, `user`, `first_*`, `latest_build_*` and `subscribed`. Use view for merge-gate data.
- The MR uses `draft` (plus the deprecated `work_in_progress`), `sha`, `detailed_merge_status`, `merge_when_pipeline_succeeds` and `diff_refs`.
- `glab mr update --ready` is **idempotent**: rc 0 both times.

## Access and limits
- `/-/raw/<ref>/<path>` on a private project without a token returns **302 to sign_in**.
- `per_page=500` is capped: `X-Per-Page: 100`.
- **The rate limit is ALREADY in effect:** `Ratelimit-Name: burst_traffic_per_user_free_plan`, `Ratelimit-Limit: 100`, with `Ratelimit-Remaining` and `Ratelimit-Reset` headers. Research said this limit was "proposed". The hourly limit was not observed.
- `glab api -F file=@x` uploads fail ("file is invalid"). Upload behaviour is untested and left to the screenshots ticket.

## Notes
- REST calls whose field names contain brackets (`actions[][...]`) need a JSON body with glab (`--input`).
