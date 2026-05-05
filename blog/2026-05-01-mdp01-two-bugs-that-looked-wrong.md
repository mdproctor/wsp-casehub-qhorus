---
layout: post
title: "Two Bugs That Looked Wrong"
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [testing, quarkus, debugging]
---

Two tests were failing. Both had misleading symptoms.

## The Intermittent One

`WatchdogEnabledTest.e2eBarrierWatchdogFullLifecycle` was blowing up with `expected: <1> but was: <2>`. The test calls `watchdogService.evaluateAll()` once and checks the alert channel. Two alerts instead of one.

My first instinct: the test is calling `evaluateAll()` somewhere implicitly, or there's a data leak from another test. We looked at the test class carefully — six condition evaluation tests with no `@TestTransaction`, all committing watchdogs permanently into the session's H2 database.

The real cause was subtler. The `WatchdogScheduler` runs on a separate thread with its own transaction. It can see any committed watchdog. The debounce window for threshold-0 watchdogs is one second. If the scheduler fires between the test's own `evaluateAll()` call and the `checkMessages()` assertion — or if a prior run left a watchdog uncommitted because the test failed before the delete step — you get two alerts.

The fix is elegant once you see it: `@TestTransaction` makes test data invisible to the scheduler's thread. The scheduler runs in its own transaction and can't see uncommitted rows. `evaluateAll()` uses `@Transactional(REQUIRED)` so it joins the test's transaction and sees the watchdog correctly. Everything rolls back at the end, no state bleeds between tests.

We added `@TestTransaction` to all six condition evaluation tests and a UUID suffix to the e2e channel names. The UUID suffix is belt-and-suspenders — if `@TestTransaction` is accidentally removed later, channels from a previous failed run can't accumulate.

## The Always-Broken One

`LedgerCaptureExampleTest` was failing in CI with `NoSuchElement: No value present` at line 67. The test creates a channel, sends messages through it, then queries the channel entity to get its UUID:

```java
Channel ch = Channel.<Channel>find("name", "ledger-ex-all-types")
    .firstResultOptional()
    .orElseThrow();  // always throws
```

The type-system example uses InMemory stores selected via `quarkus.arc.selected-alternatives`. When `tools.createChannel()` runs, the channel goes into `InMemoryChannelStore` — a `ConcurrentHashMap`. Nothing touches the H2 entity table. `Channel.find()` is a Panache static method; it bypasses CDI dispatch entirely and queries the JPA entity table, which is empty.

Claude caught this when we looked at the application.properties. The fix is to use `tools.listChannels()` instead — it routes through whichever `ChannelStore` is active.

The test was broken from day one and made it through code review unnoticed because CI wasn't reliably reaching the examples module until recent workflow changes.

## Channel Architecture

Between the fixes, I spent time thinking through what Qhorus #131 (generalised channel abstraction) actually means for the project boundary with Claudony. The key conclusion: Qhorus should be participant-blind — it doesn't know or care whether a message comes from an agent, a Slack webhook, or a human typing in Claudony. Claudony isn't a layer above Qhorus for human participation; it's a participant in Qhorus channels that proxies humans.

That lands the gateway clearly in Qhorus, the `ClaudonyChannelBackend` clearly in Claudony, and the external backends (Slack first, WhatsApp later) clearly in casehub-connectors. Issues logged, design recorded on #131.
