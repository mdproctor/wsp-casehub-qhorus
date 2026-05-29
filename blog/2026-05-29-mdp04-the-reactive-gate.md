---
layout: post
title: "The reactive gate finally has teeth"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [reactive, quarkus, dispatch, enforcement, postgresql]
---

`ReactiveMessageService.dispatch()` has existed for a while with a sign above it
saying "full enforcement coming in #193." That day arrived today.

The issue wasn't laziness. Reactive integration tests in Quarkus need a native async
driver — H2's Hibernate Reactive support is synchronous underneath, which means it
doesn't properly integrate with the Vert.x context management that
`Panache.withTransaction()` depends on. Once Podman was configured with 4 GB and
postgres:17-alpine was available, the blocker was gone and the real work could begin.

## What the spec got wrong the first time

I wrote the design spec, then handed it to a reviewer. Two critical findings came
back before a line of code was written.

The first was the commitment consistency gap. I'd proposed creating the commitment
(`OPEN` state) in a separate transaction after the message insert committed. The
reviewer pointed out the failure mode: message persists with a non-null
`commitmentId` UUID, then the commitment creation fails. No FK constraint fires.
`CommitmentService.findByCorrelationId()` returns empty; the obligation is silently
lost. The fix was simple in the reactive path — there's no `REQUIRES_NEW` constraint
in Hibernate Reactive anyway. Fold the commitment save into Phase 2's
`withTransaction("qhorus", ...)` alongside the message insert. Atomic by default.

The second was the LAST_WRITE early-exit problem. My spec said "return early" from
the reactive chain when LAST_WRITE detects the same sender and updates in place. In
a Mutiny `.flatMap().map()` pipeline, "return early" isn't a statement — it's a
design question. I chose a sealed interface:

```java
sealed interface TransactResult permits OverwriteResult, FullResult {}
record OverwriteResult(DispatchResult result) implements TransactResult {}
record FullResult(DispatchContext ctx) implements TransactResult {}
```

The `withTransaction` block returns `Uni<TransactResult>`. Pattern-matching on the
result drives the rest of the chain. `OverwriteResult` short-circuits; `FullResult`
continues through commitment state transitions and side effects. Boolean flags or
`Optional<DispatchContext>` would have worked but this is cleaner to read.

The spec also missed the ACL pre-fetch optimisation. The original proposal called
`reactiveInstanceService.findCapabilityTagsForInstance(sender)` unconditionally on
every dispatch. For open channels — the common case — this is a DB round-trip for
nothing. The actual guard: skip the fetch entirely if `allowedWriters` is null or
blank or the message is EVENT. Same semantics as the blocking path's lazy supplier.

## What the implementation found

Claude started on the first task (store seam additions) and worked upward. The
`ReactiveJpaChannelStore.updateLastActivity()` implementation was the first surprise.
I'd spec'd it as a direct `repo.update("lastActivityAt = ?1 WHERE id = ?2", ...)` —
targeted, no entity load. Claude discovered that `PanacheRepositoryBase.update()`
in a named-PU reactive context throws "Mutiny.SessionFactory bean not found" when
only the named datasource has `reactive=true`. The default SessionFactory it reaches
for doesn't exist. The workaround: `findById()` then mutate the field and rely on
Hibernate's dirty checking. Javadoc updated to say what actually happens rather than
what I intended.

The nested transaction semantics were worth documenting. `ReactiveLedgerWriteService.record()`
wraps its body in `Panache.withTransaction("qhorus", ...)`. When called from inside
`ReactiveMessageService.dispatch()`'s outer `withTransaction`, the inner call joins
the enclosing transaction — it doesn't start a new one. The blocking service uses JTA
`REQUIRES_NEW` so the ledger entry survives outer failures. The reactive path is
actually stricter: message insert and ledger entry commit together or not at all.
I documented this as an intentional divergence from the blocking semantics, not a bug.

The PostgreSQL FK constraint caught the contract tests. The existing
`MessageServiceContractTest` created messages with random `UUID.randomUUID()` as
channel IDs. With H2 `drop-and-create`, no FK constraint fires and the tests pass.
With Flyway-managed PostgreSQL, the `fk_message_channel` constraint rejects every
one of them. Claude fixed the contract test setup to use real persisted channels
via `QuarkusTransaction.requiringNew().run(() -> channelService.create(...))`.
This became the standing pattern for reactive test entity setup: JTA (`requiringNew`)
from test threads, not `Panache.withTransaction()` which needs a Vert.x context.
It's now a protocol.

The final reviewer flag was minor: two `@Inject` fields for the same
`ReactiveChannelStore` bean — `channelStore` for the channel load and
`reactiveChannelStore` for `updateLastActivity`. Both resolve to the same bean
instance; it's not a bug, but it's confusing. Consolidated before the commit.

## What it looks like now

The reactive `dispatch()` enforces the same eight concerns as the blocking gate, in
the same order: paused check, writer ACL (guarded tag fetch, sync check), rate limit,
trust gate (one `ManagedExecutor` hop for the JPA query), type policy, LAST_WRITE
semantics, commitment open (inline in the write transaction), and ledger write
returning a populated `LedgerWriteOutcome`. State transitions (acknowledge, fulfill,
decline, fail, delegate) go through `ReactiveCommitmentService` after the transaction
commits. Observer dispatch moved post-commit — it was inside `withTransaction` before,
which meant observers could query state that hadn't committed yet.

All 11 tests pass. Full build at 278 tests green. Two casehub-ledger issues filed for
the remaining deferred items: reactive `LedgerAttestation` persistence and a
`Uni<Boolean>` variant for `TrustGateService.meetsThreshold()`.

The `@Disabled` annotations are gone.
