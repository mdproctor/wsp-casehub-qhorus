---
layout: post
title: "The ledger was lying about what each message was about"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [ledger, audit, speech-act, builder-api]
---

Every `MessageLedgerEntry` in qhorus recorded the same field as its `subjectId`: the channel it was sent on. "entity-resolution". "pattern-analysis". Useful for routing. Useless for compliance. A regulator querying `?subjectId=TXN-001` got nothing. The case chain in casehub-engine used `caseId` as the subject. The qhorus message chain used the channel. They were permanently disconnected at every module boundary.

That's what this feature was about — making the ledger honest about what each message actually concerned.

The fix sounds mechanical: add `subjectId` and `causedByEntryId` to the dispatch API, propagate them into the ledger. The interesting part is the propagation semantics, which aren't what they look like. Both fields auto-populate when callers don't supply them — but they follow different sources.

`subjectId` follows the correlation root: the earliest entry in the same `correlationId` thread that has a non-null subject. When an agent sends COMMAND → STATUS → DONE about TXN-001, the STATUS and DONE inherit TXN-001 automatically. You can't inherit via `inReplyTo` for this — a RESPONSE replies to the QUERY it answers, not to the original COMMAND, so a naive inReplyTo chain breaks for any dialogue with more than one hop.

`causedByEntryId` follows `inReplyTo` exactly: the ledger entry of the message being replied to. DONE → COMMAND, RESPONSE → QUERY, HANDOFF → COMMAND. The immediate predecessor, not the correlation root. These are different concepts and the original design treated them as the same thing.

We used the AML team to validate the design before writing a line of code — three rounds of back-and-forth over what the propagation rules should be and what the API should look like. One of the more useful reviews was a simple question: *does `dispatch()` return `DispatchResult` for all 9 message types, including DONE?* AML's chain requires `doneResult.ledgerEntryId()` to close the causal graph into `COMPLIANCE_REVIEW_OPENED`. The answer has to be yes. It is.

Claude running as a code reviewer caught two things I'd missed. The `DISABLED` sentinel (`LedgerWriteOutcome(null, null, null)`) was ambiguous — if a ledger write fails silently, a catch block returning the sentinel would make "disabled" and "failed" indistinguishable. Clarifying that write failures propagate as exceptions (no catch, ever) meant the sentinel is unambiguous. The same reviewer caught that the attestation path was linking against any prior entry type, not just COMMAND or HANDOFF — DONE replying to a STATUS would have written a trust attestation against a STATUS, which is semantically wrong. A one-line type guard fixed it.

The builder enforcement was supposed to be a quality gate. It turned out to be an audit tool. Once `MessageDispatch.Builder.build()` started throwing `IllegalArgumentException` for HANDOFF without `inReplyTo`, DONE without `correlationId`, RESPONSE without `inReplyTo` — it found violations everywhere. Tests that had been running cleanly for months had been sending protocol-malformed messages. Not wrong by any execution check, because the old nine-parameter `send()` accepted nulls for everything. Wrong by the speech-act semantics the system is supposed to enforce.

Sixty-two test failures in the first full run. None of them were bugs in the feature. All of them were latent protocol violations now surfaced at the only point where they can usefully fail: construction time, before anything is sent.
