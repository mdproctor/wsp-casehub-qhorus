---
layout: post
title: "When Advisory Makes Orphans"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [qhorus, type-enforcement, commitments, advisory]
---

The plan coming in was to do three issues in sequence — #266, #271, #261. The first one dissolved before we started: #266 tracked a MessageReceivedEvent consumer migration that turned out to already be complete. All the consuming repos had updated their constructor calls. I closed the issue with a note and moved on.

#271 was the real work. The issue asked for a straightforward change: make `allowedTypes` and `deniedTypes` advisory instead of hard-enforced. The first spec draft said make everything advisory — symmetric, simple, all violations just log a warning and the message goes through. The argument seemed to hold: hard enforcement erases mistakes from the audit trail; advisory enforcement makes them visible.

The first review round pointed out I'd missed something load-bearing. COMMAND and QUERY are the two types that call `commitmentService.open()`. When an LLM sends COMMAND to the wrong channel and receives an advisory, it corrects itself. But the correction comes after the first COMMAND has already opened a Commitment on the wrong channel. Now there are two OPEN Commitments for the same correlationId — one on the wrong channel, which will stall permanently because no agent is listening for it there. `list_stalled_obligations` picks it up and surfaces a governance failure that isn't one.

Full advisory for COMMAND and QUERY breaks the normative model in a way that only becomes visible when you trace what happens when the advisory actually works.

We revised to hybrid: COMMAND/QUERY violations stay hard-enforced; all other types become advisory. STATUS, EVENT, DONE, FAILURE, DECLINE, RESPONSE, HANDOFF don't call `commitmentService.open()`, so advisory dispatch for those types produces an accurate audit entry without orphan risk. The spec went through several more review rounds before implementation — each caught something real. The AtomicReference justification for the Mutiny chain was wrong (we'd claimed single-thread execution, but the chain uses `runSubscriptionOn(workerPool)` so the justification needed fixing). A dead `@Inject MessageTypePolicy` field got left in the MCP tools after we removed the validate() call; code review found it.

One I noticed late: the first spec draft had `map.put("advisories", result.advisories())` code in the MCP tool section. It would never compile. `QhorusMcpTools.sendMessage()` returns `DispatchResult` directly — the MCP framework serialises it via Jackson. `@JsonInclude(NON_EMPTY)` on the field handles the rest. No map construction needed.

HANDOFF was an interesting edge case. We checked `CommitmentService.delegate()` directly: the child Commitment is created with `c.channelId` where `c` is the parent found by correlationId lookup — the channel the original COMMAND was on, not the channel the HANDOFF message arrived at. A HANDOFF on the wrong channel doesn't orphan the child. It's safe under advisory.

There's a `wait_for_reply` limitation the design accepts: if an LLM sends DONE to the wrong channel, the Commitment resolves correctly but `wait_for_reply` on the correct channel times out. The DONE-sender gets the advisory; if it retries on the right channel, `fulfill()` is idempotent — already-FULFILLED commitments are a silent no-op — and `wait_for_reply` finds the DONE and returns. The design is self-correcting through the advisory signal, not just a documented failure mode.

The ADR is `docs/adr/0016-hybrid-channel-type-enforcement.md`. The garden protocol was updated. #261 didn't make it onto this branch — that's next.
