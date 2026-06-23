---
layout: post
title: "What RESPONSE got away with"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [normative-layer, attestation, ledger, trust, evidential-checking, spi]
---

Zone 2 showed the 1B model sending RESPONSE on every COMMAND — technically
completing the commitment while using query-fulfillment vocabulary. Zone 3
caught the type mismatch. But there was a gap one layer deeper.

When RESPONSE lands with a COMMAND's correlationId, `CommitmentService.fulfill()`
fires. The commitment closes as FULFILLED. `LedgerWriteService` runs its attestation
gate — and RESPONSE isn't in `ATTESTATION_TYPES`. No attestation written. The trust
ledger records a clean close with no signal. The agent used the wrong word and
the infrastructure said nothing about it.

## One line of addition, one guard

The fix:

```java
private static final Set<MessageType> ATTESTATION_TYPES = Set.of(
        MessageType.DONE, MessageType.FAILURE, MessageType.DECLINE, MessageType.RESPONSE);
```

The existing guard — only attest when the prior ledger entry is COMMAND or HANDOFF —
prevents false positives on QUERY. RESPONSE answering a QUERY is correct vocabulary.
RESPONSE answering a COMMAND is wrong vocabulary, and now it gets FLAGGED at 0.3 confidence.

The 0.3 reflects genuine ambiguity. The model may have misunderstood the vocabulary
requirement rather than lying deliberately. DECLINE sits at 0.4 because it's an
explicit refusal with reasoning behind it. The numbers are all configurable; these
are defensible defaults.

## Moving the checker where it belongs

`EvidentialChecker` has been living in the examples module since Zone 3 shipped.
That was fine for running benchmark tests, but casehub-devtown needs it for
pre-attestation evidential checks — and casehub-devtown can't take a dependency
on examples code.

Moving it to `runtime/audit/` as a `@DefaultBean` CDI bean makes it injectable
by any consumer. The method signature had to change: `check(AgentResponse, BenchmarkContext)`
became `check(String messageType, String content, BenchmarkContext)`, because
`AgentResponse` is an examples type and the runtime module can't reference it
without a circular dependency. The change is mechanical — six call sites in three
test files, all straightforward.

## Teaching the policy to ask questions

The attestation policy was a `@FunctionalInterface` with two parameters: the
message type and the resolved actor ID. That's enough for the default implementation,
but not enough for an evidential one. Devtown's trust scorer needs the commitment's
correlationId and channelId to query the ledger before deciding verdict.

`CommitmentContext` is a new record in the SPI: correlationId, channelId, channelName,
commitmentId. The interface gains a 3-arg abstract method; a 2-arg default delegates
with null for backward compat. The `@FunctionalInterface` annotation came off — you can't
have two abstract methods. Any implementation that dereferences context without a null
guard will fail on legacy callers. A project protocol captures this.

---

Separately: the workspace ARC42STORIES.MD had been in stub form since June. I dispatched
a separate Claude session to read the git history and blog entries and reconstruct the
delivery arc. It came back with 809 lines and 11 delivery chapters. That document is the
first thing any consumer reads when navigating the project cold — getting it done was overdue.

The normative benchmark found a gap in agent behavior. Closing this branch found a matching
gap in the infrastructure supposed to grade it. Both closed.
