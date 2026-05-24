---
layout: post
title: "The platform docs catch up"
date: 2026-05-24
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
---

The code moved faster than the docs. That's not unusual, but it starts to matter when the docs are the coordination surface between repos.

Two things needed fixing in the parent platform docs. First: the `MessageService.dispatch(MessageDispatch)` enforcement gate — the single path through which all channel writes flow — wasn't documented at the platform level at all. The qhorus CLAUDE.md had it, but `PLATFORM.md` and the deep dive still described messaging as if `send()` was the entry point. We updated both with the full enforcement order and the `MessageDispatch` builder fields.

The second was more interesting. The normative 3-channel layout defines an `/observe` channel with `allowedTypes` of `EVENT, QUERY, INFORM`. Except `INFORM` doesn't exist. The 9-type speech-act taxonomy — QUERY, COMMAND, RESPONSE, STATUS, DECLINE, HANDOFF, DONE, FAILURE, EVENT — has no INFORM. It's been STATUS all along.

That kind of drift is easy to miss. Nobody was sending INFORM messages and getting rejected; the `/observe` channel in the normative layout tests was working fine. The error lived only in the docs. But if someone read the platform docs and tried to create a channel with `allowedTypes` including INFORM, they'd get a silent surprise at runtime.

We fixed it in both places and committed to the parent repo.

There was also a CI failure — a 401 Unauthorized from GitHub Packages while resolving `casehub-platform-api`. Transient auth, not a code problem. The previous four runs succeeded; re-running it cleared it.

The dispatch gate and the INFORM→STATUS fix are small individually. Together they're a reminder that platform docs aren't just reference material — they're the assumption layer for every session that opens a new feature. If the docs say INFORM and the code enforces STATUS, every implementation built on that assumption starts wrong.
