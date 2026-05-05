---
layout: post
title: "CommitmentStore Ships"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [commitment-store, pending-reply, obligation-lifecycle, deontic, tdd]
---

`PendingReply` has existed since the first week of Qhorus. It's a thin record: correlationId, channelId, expiresAt. Its entire purpose is to answer one question for `wait_for_reply`: "is there still an outstanding request with this ID?" When the RESPONSE arrives, the record is deleted. When it times out, the record is deleted. That's it.

When we designed the 9-type taxonomy last session, a gap became obvious. The taxonomy is grounded in deontic semantics — each message type creates, discharges, or transfers a formal obligation. QUERY creates an obligation on the receiver to inform or DECLINE. COMMAND creates an obligation to execute or DECLINE. HANDOFF transfers the obligation to a named target. FAILURE discharges the obligation unsuccessfully. PendingReply knew nothing about any of this. It only knew about RESPONSE.

The mapping from PendingReply to CommitmentStore is direct. `PendingReply` was already instantiating Singh's social commitment model — `C(receiver→sender, inform_result)` — for the QUERY→RESPONSE case. It was just incomplete. `wait_for_reply` creates the commitment (registers PendingReply), RESPONSE discharges it (deletes the row). But DECLINE? FAILURE? HANDOFF? Those arrived and disappeared into the void — no persistence layer knew they happened.

CommitmentStore completes the picture: seven states covering every path a QUERY or COMMAND can reach.

```
OPEN → ACKNOWLEDGED → FULFILLED
  │                └── DECLINED
  │                └── FAILED
  │                └── DELEGATED → (child OPEN)
  └── EXPIRED
```

The existing fields map across cleanly — correlationId stays the business key, channelId and expiresAt carry over unchanged. The new fields (state, requester, obligor, acknowledgedAt, resolvedAt, delegatedTo, parentCommitmentId) encode what PendingReply was always missing: who owes what to whom, whether they've acknowledged it, and what happened when they responded.

## The delegation constraint

The spec said HANDOFF creates a child Commitment with the same `correlationId` as the parent. The unique constraint on `correlationId` prevented this — `JdbcSQLIntegrityConstraintViolationException` on the HANDOFF E2E test. The fix: remove the constraint entirely and make `findByCorrelationId` prefer the non-terminal commitment when multiple records share a key.

This is correct by design. A correlationId identifies a *conversation*, not an obligation holder. When an obligation is delegated, the conversation continues — multiple commitments participate in it sequentially. A unique constraint enforces one-obligation-per-conversation, which is wrong.

## The module cycle

`CommitmentServiceTest` needed `InMemoryCommitmentStore` from the `testing/` module. But `testing/` depends on `runtime/` for the store interfaces. Adding `testing` as a test dependency of `runtime` creates a cycle Maven refuses to build.

Fix: place the unit test in `testing/src/test/java/` instead. Wire the store directly via field access (`service.store = store`) — CDI not needed for a state machine test. The `@Inject` field on `CommitmentService` is package-private precisely to allow this.

## The subagent that went ahead

Task 8 was "wire CommitmentService into MessageService." Claude came back reporting it had also migrated `wait_for_reply`, `cancel_wait`, and `list_pending_waits` — all three tools that Task 9 was supposed to handle. That left Task 9 as a verification step that took thirty seconds rather than two hours.

## Numbers

**871 total tests, 0 failures.** 725 runtime (44 @Disabled), 146 testing module — 139 new tests covering state machine transitions, contract compliance, H2 integration, E2E lifecycle scenarios, and MCP tool behaviour.

PendingReply is gone: entity, cleanup job, SPI, JPA implementations, InMemory stores, contract tests — all deleted. Issues #89–#97 closed.
