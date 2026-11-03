# Wayfinder tracker (local markdown)

- Map: `map.md` (label `wayfinder:map`).
- Tickets: `tickets/NNN-slug.md`, one file per ticket; the file number is its id.
- Frontmatter fields:
  - `title`
  - `label`: `wayfinder:research|prototype|grilling|task`
  - `status`: `open|closed`
  - `assignee`: empty means unclaimed; claim by setting it before any work
  - `blocked_by`: list of ticket ids
- Unblocked = every `blocked_by` ticket is closed. Frontier = open + unblocked + unassigned.
- Resolution: append a `## Resolution` section, set `status: closed`, add a line to the map's "Decisions so far".
- Research findings live on `research/<name>` branches (worktrees); the ticket links the file.
