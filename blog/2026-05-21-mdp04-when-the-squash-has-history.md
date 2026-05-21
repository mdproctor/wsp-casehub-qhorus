---
layout: post
title: "When the squash history has history"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
---

The previous session had done the real work — three refactors (#175, #176, #177)
that moved `ChannelDetail`, `InstanceInfo`, and `MessageResult` out of
`QhorusMcpToolsBase` into the api module and extracted shared mappers into a
proper CDI bean. Interesting design work. Today's session started with tidying
up after it.

The thing I wanted to know: how much noise was left in the commit history after
the two squash runs in early May?

We ran the analysis across all 325 commits. The answer: 40 candidates —
`docs(claude)` updates, design journal entries, epic closure markers,
editorial passes on normative docs. About 12% of the history. Not bad for a
project that had already been through two squash rounds.

The question was whether to execute it.

The first attempt answered that. We wrote a rebase todo file with the 40
squash operations and ran `git rebase -i 7b3cafe`. Git rejected it immediately:

```
error: invalid line 303: pick 8244e1dda...
hint: replay the merge, use 'merge -C' on the commit.
```

Seven of the 325 commits are real two-parent merge commits — the epic branches
(`epic-153-cdi-message-event`, `epic-154-inbound-correlationid`,
`epic-142-flyway-versioning`, `epic-a2a-lifecycle-cleanup`) landed via
`git merge`, not squash. Without `--rebase-merges`, git won't touch them.

With `--rebase-merges` it generates a different todo format — `label`,
`reset`, `merge -C <sha> <label>` — and replays the merge topology. We modified
that todo to add the squash operations and tried again.

It failed on `feat: merge epic-153-cdi-message-event`:

```
Auto-merging CLAUDE.md
CONFLICT (content): Merge conflict in CLAUDE.md
```

This one took a moment to understand. The squash operations had modified
CLAUDE.md — absorbing a dozen `docs(claude)` updates into their neighbouring
feature commits. By the time the merge commit replayed, CLAUDE.md was in a
different state than the original merge had expected. The commit that merged
the epic-153 branch was computing a three-way merge against a base that no
longer existed.

The fix was obvious once it was clear: squash only the commits above the merge
topology. The most recent merge commit (`8244e1d feat: merge
epic-a2a-lifecycle-cleanup`) provides a clean boundary. Everything above it is
linear — no merge commits, no conflicts.

```bash
git rebase -i 8244e1d
```

Five squash operations on 21 commits. Two epic closure markers absorbed into
`fix(dashboard)`. Two CLAUDE.md updates absorbed into `fix(store)`. One
`docs(claude)` absorbed into `fix(#177)`. 326 → 321 commits. Zero conflicts.

The 15 commits inside the merge topology that were squashable — the design
journals, the CLAUDE.md updates inside epic branches — stay. They're real
history that would require either conflict resolution or careful file-by-file
surgery to remove. Not worth it.

We also merged PR #178 and confirmed the push workflow: `git push origin main`
to the fork, promote to casehubio/qhorus manually when ready. Two things I
hadn't made explicit before.
