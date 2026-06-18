---
layout: post
title: "The fix that missed main"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [git, ci, housekeeping, branches]
---

CI on casehub was red. The error was a compilation failure in `QhorusLedgerEntryRepository` and `ReactiveLedgerEntryJpaRepository` — both importing `ActorIdentityProvider` from `io.casehub.ledger.runtime.privacy`, a package that no longer exists after the ledger moved the class to `api.spi`.

I'd seen this before. We'd already updated the imports in an earlier session. The local files confirmed it — both showed the correct `api.spi` import. So why was CI failing?

`git branch --all --contains c15807e` answered it. The fix lived exclusively on `issue-261-slack-channel-backend`. Neither `origin/main` nor `upstream/main` had ever seen it. Local file state was correct because that branch was checked out — CI builds main, and main had the old code.

The fix was already written. It just needed to be somewhere it could be built.

Two issues had been filed for this (#285, #286 — duplicates, slightly different titles). With the code confirmed present, both were closed, the branch fast-forwarded to main, and the fix pushed to both remotes.

---

That pointed at a broader question: what else was open that shouldn't be? I ran a full audit — every open issue cross-checked against the codebase, every branch checked for unmerged work, all 68 blog entries verified against the publication index.

Nothing was lost. The open issues were genuinely open. All 68 blogs published. Specs and ADRs promoted. But eleven branches — from issue-236 through issue-282, each with closed issues and code confirmed in main — had never received the `chore: branch closed` stamp. The squash workflow had brought the code to main and left the branch tips looking live. Each got an empty closing commit.

The session started as a session resume and ended as an archaeology dig. The real find wasn't the missing stamp — it was that `git branch --all --contains <sha>` is the authoritative tool when local file state and CI disagree. Local files tell you what's on disk. That command tells you where the commit actually landed.

The Slack channel backend (#261) is next. The spec is in the workspace. A fresh session can go straight in.
