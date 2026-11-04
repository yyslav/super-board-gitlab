---
title: How do UI screenshots get into MRs?
label: wayfinder:grilling
status: open
assignee: 
blocked_by: [008]
---
## Question

`pr-shots` commits before/after screenshots and links them by `github.com/.../raw/<sha>/`. Raw URLs in a private GitLab project need a token. How should screenshots reach the MR description?
- project uploads (markdown `/uploads/...`)
- committed files with a `/-/raw/` link
- something else

What does that mean for MR size and cleanup?
