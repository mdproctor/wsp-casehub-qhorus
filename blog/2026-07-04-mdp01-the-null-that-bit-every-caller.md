---
layout: post
title: "The Null That Bit Every Caller"
date: 2026-07-04
type: phase-update
entry_type: note
subtype: diary
projects: [Qhorus]
tags: [channel, null-safety, transaction, reactive]
---

`Channel.builder("name").build()` returns null for `allowedWriters()`. Every consumer that iterates the list without a null check gets an NPE. The fix sounds trivial — default to `List.of()` — but the design question underneath is more interesting: when does null carry meaning?

For Channel's three list fields (`allowedWriters`, `barrierContributors`, `adminInstances`), null and empty are semantically identical everywhere in the platform. `AllowedWritersPolicy` checks `null || isEmpty()`. `QhorusMcpToolsBase` checks `null || isEmpty()`. `WatchdogEvaluationService` does `!= null ? list : List.of()`. No code path, across qhorus, engine, ops, or claudony, distinguishes "not set" from "empty list" for any of these fields.

For the set fields (`allowedTypes`, `deniedTypes`), null means something different: "no constraint" — the channel is open to all message types. An empty set would mean "nothing is permitted," which is the opposite. The service layer explicitly normalizes empty sets to null at the write boundary, confirming the distinction is intentional.

So the fix is asymmetric. List fields normalize null to `List.of()` in the compact constructor. Set fields preserve null. The same change applies to `ChannelCreateRequest` — it's a validated request type whose compact constructor is already the single enforcement gate for slug format and type-set overlap, so list normalization belongs there too.

The persistence layer needed alignment. `ChannelEntity.joinCsv(List.of())` was returning `""` instead of null — an empty string in a DB column where null was the established contract. Both `ChannelEntity.joinCsv()` and `QhorusEntityMapper.joinCsv()` now return null for empty lists. The round-trip is clean: `List.of()` → null in DB → `splitCsv(null)` → null → compact constructor → `List.of()`.

## The Transaction That Worked on H2

`ChannelService.findOrCreate()` has a race recovery path: two threads try to create the same channel name, one wins, the other catches `PersistenceException` and retries a lookup. On H2, this works. On PostgreSQL, the failed INSERT marks the entire transaction as aborted — every subsequent query in the same transaction is rejected with "current transaction is aborted, commands ignored until end of transaction block."

The root cause is CDI self-invocation. `findOrCreate()` is `@Transactional(REQUIRES_NEW)`. It calls `create()` on the same bean — but self-invocation bypasses the CDI proxy, so `create()` joins the same transaction instead of getting its own. When the constraint violation fires, the outer transaction is poisoned.

The fix extracts creation into `ChannelCreateHelper` — a package-private `@ApplicationScoped` bean whose `createInNewTransaction()` method is `@Transactional(REQUIRES_NEW)`. Because the call goes through CDI now, the helper gets its own transaction. If it fails, its transaction rolls back independently; the outer `findOrCreate` transaction stays clean for the retry query.

This also centralises channel creation. `ChannelService.create()`, `findOrCreateByName()`, and `findOrCreateWithBinding()` all delegate to the helper — single source of truth for store put, connector binding, and `channelGateway.initChannel()`. The `autoCreated` flag passes through as a boolean parameter instead of being applied differently in each code path.

## The Reactive Path Nobody Tested

`ReactiveChannelService.create()` was a stripped-down version — just `channelStore.put()`. No connector binding creation, no `channelGateway.initChannel()` call. A consumer switching from blocking to reactive `create()` would silently lose backend registration.

The fix brings it to full parity: connector binding with duplicate-key guard, `initChannel()` for backend registration, and `PersistenceException` race recovery via `onFailure().recoverWithUni()`. The blocking operations are offloaded to the worker pool via `runSubscriptionOn()` since `channelBindingStore` is a JPA store and `initChannel()` fires synchronous CDI events.

An adversarial design review caught several things I'd missed. The reactive `create()` originally used `.invoke()` for the blocking work — which would have run on the Vert.x event loop. The review also caught that `ReactiveChannelService` didn't inject `ChannelGateway` at all, that the `autoCreated` flag needed to be a parameter on the helper, and that `QhorusEntityMapper.joinCsv()` had the same empty-list problem as `ChannelEntity.joinCsv()`.

The transaction isolation change has a test-level consequence worth knowing about. `ChannelCreateHelper`'s `REQUIRES_NEW` commits independently of `@TestTransaction` rollback. Tests that relied on rollback for cleanup — using fixed channel names like `"auth-refactor"` — now collide when test classes share the same Quarkus context. Every test that creates a channel through the service path needs unique names.
