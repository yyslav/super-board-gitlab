---
title: How do we keep pulling upstream releases?
label: wayfinder:grilling
status: closed
assignee: 
blocked_by: [004]
---
## Question

What branch layout and merge routine keep the fork current with upstream super-board without re-porting each release, given the chosen seam? How do we catch new `gh` call sites that bypass the adapter?

## Resolution

Ruled out of scope by [Where does the forge seam go?](004-adapter-seam.md): the port is a hard fork with no upstream sync.
