---
layout: post
title: "Branch hygiene and a recovered entry"
date: 2026-05-24
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [git, workflow]
---

Spent a session on housekeeping — the kind that doesn't show up in the feature log but matters for the next person who opens the repo and wonders what's live and what's done.

The project had 22 non-main branches. All closed, all with squash-rebase content on main — but none marked. From the outside, `git log main..<branch>` showing 43 commits ahead looks identical to a branch with real unmerged work. We went through all of them, verified the key artifacts were on main, and stamped each with an empty closure commit: `chore: branch closed`. Now `git log -1 --format="%s" <branch>` is the reliable signal.

While auditing the workspace blog branches for entries that never reached main, I hit a subtle trap. `git diff --name-only main..$b -- blog/` looked like the right query but returned 18 "missing" entries on the oldest epic branch — all of which were on main already. The diff direction was backwards: diff A..B shows what needs to change to go from A to B, so main's newer files appeared as if they were absent from the old branch. The right query is `git log main..$b -- blog/` — commits on the branch, not a file diff.

That audit did surface one genuine miss. A blog entry written on `epic-142-flyway-versioning` on May 17 — the A2A `DELEGATED` state bug and the Flyway V2 classpath collision — was written after `work-end` had already run and promoted the branch's artifacts to workspace main. The promotion step can't see what doesn't exist yet. The entry sat on the closed epic branch for a week.

We recovered it, published it, and noted the pattern: if you write a blog after closing an epic, immediately cherry-pick it to workspace main.

The `chore: branch closed` convention went into `~/.claude/working-style.md` so future sessions know to apply it. The initial draft said "squash-merged to main" — which isn't the workflow here. We rebase-merge, sometimes with a semantic squash on the branch beforehand. Corrected before it calcified.
