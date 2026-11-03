# Glossary

**Forge**: the code-hosting platform that holds the repo, cards, and change requests: GitHub or GitLab.
**Board**: the kanban that super-board drains; its columns are lanes.
**Lane**: one stage a card moves through (Ready, Building, QA, Review, ...); a card sits in exactly one lane.
**Card**: one unit of work on the board, backed by a forge issue.
**Change request**: the forge's reviewable merge proposal: a PR on GitHub, an MR on GitLab.
**Merge gate**: the check that must pass before super-board merges a change request.
