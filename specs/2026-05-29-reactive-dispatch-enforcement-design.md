# Reactive Dispatch Enforcement Parity — Design Spec

**Issue:** casehubio/qhorus#193
**Branch:** issue-193-reactive-dispatch-enforcement
**Date:** 2026-05-29

---

## Problem

`ReactiveMessageService.dispatch()` is missing seven enforcement concerns that the blocking
`MessageService.dispatch()` applies. The reactive path persists messages without ACL, rate
limiting, trust gating, type policy, LAST_WRITE semantics, ledger writes, or fanOut. It also
calls `CommitmentService` (blocking JTA) from inside a Hibernate Reactive event loop — a latent
threading bug that surfaces as soon as the tests are un-@Disabled.

The blocking service also has a store-seam violation: the LAST_WRITE branch calls
`Message.<Message> find(...)` directly (Panache static) instead of going through `MessageStore`.
This breaks `InMemoryMessageStore` substitution in tests and is fixed here alongside the reactive
work.

---

## Scope

### New class
- `ReactiveCommitmentService` — full reactive mirror of `CommitmentService`, gated
  `@IfBuildProperty(casehub.qhorus.reactive.enabled=true)`, uses `ReactiveCommitmentStore` +
  `Panache.withTransaction("qhorus", ...)`. Each state-transition method opens its own transaction
  (equivalent semantics to `REQUIRES_NEW` in the blocking service).

### Store seam fix
Add `findLastMessage(UUID channelId): Optional<Message>` to `MessageStore` and
`findLastMessage(UUID channelId): Uni<Optional<Message>>` to `ReactiveMessageStore`. Implement in:
- `JpaMessageStore` — `Message.<Message> find("channelId = ?1 ORDER BY id DESC", channelId).page(0,1).firstResultOptional()`
- `ReactiveJpaMessageStore` — reactive Panache equivalent
- `InMemoryMessageStore` — stream over in-memory store, max-id entry
- `InMemoryReactiveMessageStore` — wrap blocking impl via `Uni.createFrom().item(...)`

Fix `MessageService.dispatch()` LAST_WRITE block to call `messageStore.findLastMessage(ch.id)`
instead of the direct Panache static call.

### `ReactiveLedgerWriteService` — return type change
`record(Channel, Message)` changes from `Uni<Void>` to `Uni<LedgerWriteOutcome>`. After saving
the ledger entry, map to `new LedgerWriteOutcome(entry.id, ch.id, entry.causedByEntryId)`.
`causedByEntryId` is already resolved for DONE/FAILURE/DECLINE/HANDOFF via
`findLatestByCorrelationId`. Attestation remains deferred: replace the no-op
`logSkippedAttestation` with a `LOG.warn` and file casehub-ledger issue for reactive
`LedgerAttestation` persistence.

### `ReactiveMessageService.dispatch()` — full enforcement sequence

The method is restructured into explicit phases, preserving the exact same enforcement ordering as
`MessageService.dispatch()`:

**Phase 1 — Pre-transaction reactive checks**

Inside a `channelStore.find(channelId).flatMap(...)` chain (no transaction open yet):

1. Paused check — sync on loaded channel (already present)
2. ACL: `reactiveInstanceService.findCapabilityTagsForInstance(sender)` → `Uni<List<String>>`.
   In the flatMap callback, call `allowedWritersPolicy.isAllowedWriter(sender, ch.allowedWriters,
   () -> preResolvedTags)` synchronously. The supplier is `() -> preResolvedTags` — no I/O; safe
   on the event loop.
3. Rate limit check — `rateLimiter.check(...)` — in-memory, safe on event loop.
4. Trust gate — `Uni.createFrom().item(() -> trustGateService.meetsThreshold(target, threshold))
   .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`. One hop to a worker pool thread for
   the JPA query; control returns to the event loop before the transaction opens. Trust gate is
   skipped (pass) when `minObligorTrust == 0.0` or type != COMMAND — same guard as blocking path.
5. Type policy — sync on loaded channel, safe on event loop.

**Phase 2 — Reactive transaction**

`Panache.withTransaction("qhorus", () -> ...)`:

6. LAST_WRITE: `reactiveMessageStore.findLastMessage(channelId)`. If present and same sender →
   in-place update, return early (no ledger, no fanOut, no commitment — mirrors blocking path).
   If present and different sender → throw `IllegalStateException`. If absent → continue.
7. Normal insert — `reactiveMessageStore.put(message)`.
8. Reply count — `reactiveMessageStore.find(inReplyTo).invoke(opt -> opt.ifPresent(p -> p.replyCount++))`.
9. Channel activity update.
10. Ledger write — `reactiveLedgerWriteService.record(ch, message)` → `Uni<LedgerWriteOutcome>`.
    The ledger write runs inside the same transaction so it commits atomically with the message.

**Phase 2 output — primitive carrier**

The `withTransaction` flatMap returns a local record (not `DispatchResult`) carrying extracted
primitives before the transaction closes — matching the blocking service's "Extract primitives
BEFORE REQUIRES_NEW boundary" comment:

```
record DispatchContext(long messageId, UUID commitmentId, Instant occurredAt,
                       LedgerWriteOutcome ledgerOutcome, String channelName,
                       int replyCount)
```

Entities are not passed across the transaction boundary.

**Phase 3 — Post-transaction commitment**

`.flatMap(ctx -> reactiveCommitmentService.updateState(dispatch, ctx.commitmentId()).replaceWith(ctx))`:

- `ReactiveCommitmentService` opens its own `withTransaction("qhorus", ...)` for each call.
- This is semantically equivalent to `REQUIRES_NEW` in the blocking path: the outer message
  transaction commits first, then the commitment state is updated in a separate transaction.
- EVENT messages skip commitment as in the blocking path.

**Phase 4 — Sync side effects and DispatchResult construction** (any thread, already on event loop)

`.map(ctx -> { sideEffects(ctx); return buildDispatchResult(ctx); })`:

- Observer fan-out — `MessageObserverDispatcher.dispatch(...)` (already present).
- Rate limit recording — `rateLimiter.recordSend(...)` — in-memory.
- `channelGateway.fanOut(...)` — already spawns a virtual thread per backend, returns immediately;
  safe to call directly from the event loop.
- Returns `DispatchResult` built from `ctx` — `ledgerEntryId`, `subjectId`, `causedByEntryId`
  from `ctx.ledgerOutcome()`, now all non-null.

### New injections in `ReactiveMessageService`

```
@Inject ReactiveInstanceService reactiveInstanceService;
@Inject AllowedWritersPolicy allowedWritersPolicy;
@Inject RateLimiter rateLimiter;
@Inject TrustGateService trustGateService;
@Inject MessageTypePolicy messageTypePolicy;
@Inject ReactiveLedgerWriteService reactiveLedgerWriteService;
@Inject ReactiveCommitmentService reactiveCommitmentService;
@Inject ChannelGateway channelGateway;
@Inject QhorusConfig config;
```

`AllowedWritersPolicy`, `RateLimiter`, `MessageTypePolicy`, `ChannelGateway`, and `QhorusConfig`
are not gated by `@IfBuildProperty` — injecting them in the reactive service is safe.

### `ReactiveCommitmentService` — method signatures

Mirrors `CommitmentService` one-for-one, all returning `Uni<?>`:

```java
Uni<Commitment> open(UUID commitmentId, String correlationId, UUID channelId,
    MessageType type, String requester, String obligor, Instant expiresAt)
Uni<Optional<Commitment>> acknowledge(String correlationId)
Uni<Optional<Commitment>> fulfill(String correlationId)
Uni<Optional<Commitment>> decline(String correlationId)
Uni<Optional<Commitment>> fail(String correlationId)
Uni<Optional<Commitment>> delegate(String correlationId, String delegatedTo)
Uni<Integer> expireOverdue()
Uni<Optional<Commitment>> findByCorrelationId(String correlationId)
```

`updateState(MessageDispatch, Long messageId, UUID commitmentId)` is a package-private
dispatcher method (mirrors the switch block in `MessageService.dispatch()`) — not `@Tool`-exposed.

Each mutating method uses `Panache.withTransaction("qhorus", () -> ...)`. `findByCorrelationId`
uses the store directly (no transaction needed for reads in Hibernate Reactive outside a session —
wrap in `Panache.withSession("qhorus", ...)`).

---

## Testing

### PostgreSQL DevServices test profile

`ReactiveTestProfile.getConfigProfile()` returns `"reactive-pg"`.

`%reactive-pg` block in `application.properties`:

```properties
%reactive-pg.quarkus.datasource.qhorus.db-kind=postgresql
%reactive-pg.quarkus.datasource.qhorus.devservices.enabled=true
%reactive-pg.quarkus.datasource.qhorus.devservices.image-name=postgres:17-alpine
%reactive-pg.quarkus.datasource.qhorus.reactive=true
%reactive-pg.quarkus.datasource.qhorus.jdbc=true
%reactive-pg.quarkus.flyway.qhorus.migrate-at-start=true
%reactive-pg.quarkus.hibernate-orm.qhorus.database.generation=none
# Default datasource stub for casehub-ledger beans (@Default EntityManager)
%reactive-pg.quarkus.datasource.db-kind=h2
%reactive-pg.quarkus.datasource.jdbc.url=jdbc:h2:mem:ledger-stub-reactive;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
%reactive-pg.quarkus.hibernate-orm.database.generation=drop-and-create
# casehub.ledger.datasource must point to qhorus so ledger beans use the named PU
%reactive-pg.casehub.ledger.datasource=qhorus
```

`ReactiveTestProfile.getConfigOverrides()` keeps `casehub.qhorus.reactive.enabled=true`.

### Test coverage additions

`MessageServiceContractTest` adds abstract setup/teardown helpers and new test methods:

- `paused_channel_rejects_send` — channel.paused=true → `IllegalStateException`
- `acl_channel_rejects_unauthorised_sender` — `allowedWriters` set, wrong sender
- `acl_channel_permits_authorised_sender` — sender in `allowedWriters`
- `rate_limit_channel_rejects_burst` — send until rate limit fires
- `type_policy_rejects_disallowed_type` — channel with `allowedTypes` set
- `last_write_channel_update_in_place` — same sender overwrites; different sender throws
- `commitment_opened_for_command` — `CommitmentStore` has OPEN entry after COMMAND
- `ledger_entry_populated_in_dispatch_result` — `DispatchResult.ledgerEntryId` is non-null

The abstract class adds new abstract setup helpers for creating channels with specific
configurations (paused, ACL, rate limit, type policy, LAST_WRITE).

`ReactiveMessageServiceTest` removes `@Disabled`.

---

## Out of scope / issues filed

- **casehub-ledger**: Reactive `LedgerAttestation` persistence — `saveAttestation()` in
  `ReactiveMessageLedgerEntryRepository` currently throws `UnsupportedOperationException`.
  File as casehub-ledger issue.
- **casehub-ledger**: Reactive `TrustGateService.meetsThreshold()` — `Uni<Boolean>` variant.
  File as casehub-ledger issue. Until then, worker-thread bridge used in
  `ReactiveMessageService`.
