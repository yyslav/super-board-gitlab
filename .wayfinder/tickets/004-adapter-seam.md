---
title: Where does the forge seam go?
label: wayfinder:grilling
status: closed
assignee: yaroslav.y
blocked_by: [002, 003]
---
## Question

What shape does the forge adapter take, and where does it sit, so that the GitHub path stays byte-for-byte equivalent and the upstream diff stays minimal? Candidates:
- a `forge` CLI that skills, scripts, and hooks call
- a python module
- a `gh`-compatible shim that translates calls to GitLab
How is the forge selected: from the git remote, or from config?

## Resolution

The user decided this in a grilling session on 2026-11-04. This decision reverses the charting decisions to support both forges and to keep syncing upstream.

- **Scope:** hard fork, GitLab only, no upstream sync. For GitHub, the user runs the original super-board.
- **Transport:** `glab` everywhere (scripts, hooks, agent prose), mirroring how upstream uses `gh`. There is no shim and no custom forge CLI.
- **Read funnel:** `gitlab-read.py` replaces `github-read.py` and keeps its contract:
  - 3 attempts per logical read
  - a halt file and exit 79
  - `--kind` shape validation, with exit 64 for a malformed agent call
- **Data shapes:** code and prose switch to GitLab-native fields (`iid`, `web_url`, `description`). Nothing is normalised back to GitHub shapes.
- **Writes:** keep the `gh_write_once` semantics through `glab`: no retry, and halt with exit 79 on failure.
- **Guards:**
  - The permission allowlist is rewritten for `glab`.
  - `gh-guard.sh` becomes `glab-guard.sh` and works from GitLab rate-limit headers and `Retry-After`, budgeting for 100 req/min.
  - No new blocking hook.
- **Structure:** rewrite in place, keeping every file's role. Rename only files with `github`/`gh` in the name. Refactors such as merging the three filers come after the port.
- **Per-file `glab` traps** are recorded in the mapping research and belong to the spec, not to a separate decision:
  - `mr merge` must pass `--auto-merge=false`
  - `--sha` pins the head
  - issues need an explicit close
  - quick actions must be sanitised
