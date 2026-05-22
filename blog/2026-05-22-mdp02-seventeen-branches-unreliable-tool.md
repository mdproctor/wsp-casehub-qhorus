---
layout: post
title: "Seventeen branches and an unreliable tool"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
---

The branch was closed, the blog written, everything published. That should
have been the end of it.

Then I looked at the branch list.

Seventeen stale branches across project and workspace — epics, issues, and
backup branches from the May 21 squash session. The previous handover had
flagged some as deletable. Most had EPIC-CLOSED markers but showed `merged=0`
against main. I brought Claude in to verify them before we touched anything.

`git branch --merged main` checks SHA ancestry. After a squash-rebase, the
branch's original commits are no longer ancestors of main — even though every
change they introduced is present. The tool simply cannot see them. We had to
verify each branch by searching main's log by subject:

```bash
git log main --oneline --grep="fix(dashboard): remove @Alternative"
# d3b82cd fix(dashboard): remove @Alternative, fix gating property...
```

Match found. Different SHA, same content. Every "unmerged" branch came back
the same way — confirmed on main, safe to delete. Seventeen branches gone: 13
local project, 3 remote project, 13 local workspace, 6 remote workspace.

Then the squash. Thirteen commits between main and upstream — the #179/#180
migration work, the #161 prefix query, the #181 gateway event, protocols, ADR,
spec promotions. We collapsed them to 8: two fix commits merged into one, a
CLAUDE.md sync absorbed into its adjacent feature, a one-line protocol path
update relocated to its semantic home in the protocol commit.

The verification step produced one alarming result. Group 5 showed 13 lines
of divergence against its original. Claude flagged it immediately and checked
the cause: we'd moved the protocol path update from position 5 to position 10
in the rebase todo. Every commit in between now lacked that change in their
tree. The final tree check cleared it:

```bash
git diff HEAD <original-HEAD> -- ':!.claude/'
# (no output)
```

Intermediate divergence when reordering commits in a rebase is expected and
harmless. The tree at HEAD is what matters, and it was clean.

Eight commits instead of thirteen. Both origin and upstream in sync.
