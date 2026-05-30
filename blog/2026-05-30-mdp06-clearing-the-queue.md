---
layout: post
title: "Clearing the Queue: Eleven Issues and One Narayana Surprise"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [jta, cdi, spi, testing, maven]
---

The sprint list going into this session was eleven issues — #213 through a batch
of S and XS items that had been accumulating while the connector backend work ran.
I brought Claude in to work through them sequentially on a single branch.

## Extracting ObligorTrustPolicy (#213)

The main item was an M-scale SPI extraction. `MessageService.dispatch()` had an
inline `if` block calling `TrustGateService.meetsThreshold()` directly — no
extension point, no way for Claudony to plug in capability-scoped trust evaluation.
The fix is a `@FunctionalInterface ObligorTrustPolicy` in `api/spi/`, with a
default impl that wraps the old gate logic.

The interesting design choice was the context record. The issue described a simple
`(String obligorId, double minTrust)` signature, but that leaks config into the
SPI and leaves Claudony without the channel context it needs for per-capability
routing. We went with `ObligorTrustContext(String obligorId, UUID channelId, String channelName)` —
both channel fields, because Claudony's trust routing maps from channel names to
capability dimensions. I nearly wrote `long channelId` before the compiler flagged
it. `Channel.id` is `UUID`, which required reading the entity before writing the record.

Claude's code review afterwards caught a dead `QhorusConfig` injection in
`ReactiveMessageService` — the threshold check had moved into the default impl,
but the field stayed behind. Easy miss that would have been an unused import at
best, a misleading `config.commitment().minObligorTrust()` call that no longer
existed at worst.

## Three Issues Already Done

When we got to #150, #148, and #146 — the A2A cleanup items — all three were
already resolved. They'd been waiting behind #135 and when that landed, the fixes
came with it. Closed all three without a line of code.

The issue queue had inflated. Worth a periodic pass to check which "blocked by"
items resolved when their blocker merged.

## The CDI Observer That Wouldn't

For #215 (fire `ChannelInitialisedEvent` on binding update), I wanted to verify
the event fires via a CDI observer in the integration test. Tried an
`@ApplicationScoped` static inner class inside the `@QuarkusTest` class — observer
method never called. Moved to a separate top-level class in the test package —
same result.

The event was firing; the production code was correct. But the CDI observer wasn't
receiving it in the `@TestTransaction` context. We couldn't find a clean explanation.
The working approach: drop CDI observation and mock `Event<ChannelInitialisedEvent>`
directly in a pure Mockito unit test, then verify the call with the correct arguments.
Simpler, and it actually tests what matters.

The failure mode is now in the garden (GE-20260530-c13942).

## JTA ROLLBACK_ONLY and Narayana's "State 1"

The most interesting problem came from #166 — deferring `MessageObserver` calls
to post-commit via `TransactionSynchronizationRegistry`. The implementation was
straightforward. Then nine unrelated tests started failing with:

```
IllegalStateException: Syncs are not allowed to be registered when the
transaction is in state 1
```

"State 1" is not a JTA spec constant. The spec says `STATUS_ACTIVE = 0`,
`STATUS_MARKED_ROLLBACK = 1`. Narayana's internal state machine uses different
numbering. When a `@Transactional` method throws inside a `@TestTransaction`
outer transaction — as happens in pause/resume tests where one `sendMessage`
is expected to fail — Narayana marks the outer TX as ROLLBACK_ONLY. Any subsequent
`registerInterposedSynchronization()` call then throws, because the TX isn't active.

The fix:

```java
if (tsr == null || tsr.getTransactionStatus() != Status.STATUS_ACTIVE) {
    dispatchSynchronously(event, observers);
    return;
}
tsr.registerInterposedSynchronization(synchronization);
```

There's a neat testing consequence: `mock(TransactionSynchronizationRegistry.class)`
returns `0` for `int` methods by default — which is `STATUS_ACTIVE` — so the
deferred path activates in tests without explicit stubbing.

This became two garden entries and a protocol (PP-20260530-332d70).

## Fifty Test Files and a Python Parser

When `create_channel` gained four connector binding params, every test calling the
old 9-arg version needed four trailing nulls. Fifty files, mix of single-line and
multi-line calls. Regex wasn't sufficient — calls that spread across lines with
non-null trailing args don't match a simple suffix pattern.

We wrote a Python script that walks balanced parentheses character by character,
counts top-level args, and injects the nulls before the closing paren. Handled
all fifty files in one pass, including the multi-line cases. The technique is in
the garden now (GE-20260530-319607) — it comes up any time a public Java method
gains new parameters with dozens of existing call sites.

## The SPI Placement Discovery

During brainstorming for #213, I checked whether `CommitmentAttestationPolicy`
and `InstanceActorIdProvider` were correctly placed — CLAUDE.md described them as
being in `runtime/ledger/`, which would violate the `consumer-spi-placement`
protocol we'd just drafted. Checked the actual source: they're already in `api/spi/`.
The CLAUDE.md description was about the *default implementations*, not the interfaces.

Filed and closed issue #223 within the same session. The protocol still stands —
`api/spi/` is the right place, and now there's a written rule for it.
