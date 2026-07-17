---
layout: post
title: "The Obligation Model Was Already There"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [obligation-model, correlation, telemetry, speech-acts]
---

## The Obligation Model Was Already There

I went into this session expecting to build three separate features: correlation validation (#353), QUERY tracking (#362), and context window telemetry (#363). The first two turned out to be the same problem.

Issue #362 said QUERYs have "no commitment, no tracking, no deadline." I traced the dispatch gate to check — and found this at line 305 of `MessageService.dispatch()`:

```java
case QUERY, COMMAND -> commitmentService.open(
    storedCommitmentId, dispatch.correlationId(),
    dispatch.channelId(), dispatch.type(),
    dispatch.sender(), dispatch.target(), saved.deadline());
```

QUERYs already create Commitment records. They already support deadlines, acknowledgement, expiration, and the full 7-state lifecycle. The issue's premise was wrong.

The real gap was subtler: the commitment model is **type-blind at resolution time**. Both RESPONSE and DONE call `commitmentService.fulfill()`. An agent can send RESPONSE on a COMMAND's correlationId and the commitment silently transitions to FULFILLED — semantically wrong (RESPONSE is query-vocabulary, not command-vocabulary), but structurally accepted. Protocol PP-20260623-fd69f3 had already documented this exact problem.

Once that root was visible, all three issues collapsed into a coherent design:

**Type-aware resolution matching.** A `CorrelationIntegrityChecker` now sits in the dispatch gate alongside the existing policy beans. Four advisory checks: inReplyTo validation (does the referenced message exist? same channel?), resolution type matching (RESPONSE on a COMMAND? DONE on a QUERY?), obligor identity (is the sender actually the assigned obligor?), and cross-channel detection. All advisory — WARN log plus `DispatchResult.advisories()` — because the normative layer should surface violations without blocking legitimate edge cases.

**Default QUERY deadlines.** Since QUERYs already create commitments, all they needed was a sensible default deadline. `casehub.qhorus.commitment.default-query-deadline` applies to the Commitment's `expiresAt` only — `Message.deadline` stays null, preserving the distinction between "sender set 5 minutes" and "infrastructure applied 10 minutes." The existing `expireOverdue()` scheduler and the notification bridge (#365) handle everything from there.

**Context pressure telemetry.** This one was genuinely independent — a new `contextWindowPct` column on `MessageLedgerEntry` (V2002), parsed from EVENT telemetry JSON, and a `CONTEXT_PRESSURE` watchdog condition that fires when any agent's reported context usage exceeds the threshold. Adding the new `ContextPressureContext` to the sealed `AlertContext` interface broke an exhaustive switch in `ConnectorAlertBridge` — a ripple the compiler catches, but one that crosses module boundaries in a way you wouldn't expect until it happens.

The pattern that made #362 interesting: sometimes the infrastructure you need already exists and the issue is a misunderstanding of what's there. Tracing the code before designing the solution saved building something that already worked.
