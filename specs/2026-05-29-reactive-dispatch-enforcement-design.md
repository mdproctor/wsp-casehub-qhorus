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

The blocking service also has a `PP-20260529-eb19c3` store-seam violation: the LAST_WRITE branch
calls `Message.<Message> find(...)` (Panache static) instead of going through `MessageStore`. This
breaks `InMemoryMessageStore` substitution in tests and is fixed alongside the reactive work.

---

## Scope

### New class
- `ReactiveCommitmentService` — reactive mirror of `CommitmentService` for state-transition
  operations only (acknowledge / fulfill / decline / fail / delegate / expireOverdue).
  Gated `@IfBuildProperty(casehub.qhorus.reactive.enabled=true)`. Each method opens its own
  `Panache.withTransaction("qhorus", ...)`.
  Note: the `open()` case (COMMAND/QUERY) is handled inline in Phase 2, not via this service —
  see dispatch design below.

### Store seam additions

**`MessageStore` + `ReactiveMessageStore`:**
Add `findLastMessage(UUID channelId)` → `Optional<Message>` / `Uni<Optional<Message>>`.
Returns the most recent message in the channel regardless of sender.

**`ReactiveChannelStore`:**
Add `updateLastActivity(UUID channelId)` → `Uni<Void>`. Issues a targeted `UPDATE` query
(`lastActivityAt = now()`). Does not load or re-attach the channel entity — avoids session
management concerns when the channel was loaded pre-transaction.

**Implementations (all four stores + `InMemoryReactiveChannelStore`):**
- `JpaMessageStore.findLastMessage()` — `Message.<Message>find("channelId=?1 ORDER BY id DESC", id).page(0,1).firstResultOptional()`
- `ReactiveJpaMessageStore.findLastMessage()` — reactive Panache equivalent
- `InMemoryMessageStore.findLastMessage()` — scan in-memory list for max-id entry
- `InMemoryReactiveMessageStore.findLastMessage()` — wrap blocking impl via `Uni.createFrom().item(...)`
- `ReactiveJpaChannelStore.updateLastActivity()` — `repo.update("lastActivityAt=?1 WHERE id=?2", Instant.now(), channelId).replaceWithVoid()`
- `InMemoryReactiveChannelStore.updateLastActivity()` — find-and-mutate in-memory map

**Fix `MessageService.dispatch()` LAST_WRITE block:**
Replace `Message.<Message> find(...)` with `messageStore.findLastMessage(ch.id)`.

### `ReactiveLedgerWriteService` — signature change

`record(Channel, Message)` → `record(MessageDispatch dispatch, Long messageId, UUID commitmentId, Instant occurredAt)`.

Aligns with blocking `LedgerWriteService.record(dispatch, messageId, commitmentId, occurredAt)`.
Enables full three-priority `subjectId` resolution:
1. `dispatch.subjectId()` if explicitly set
2. Earliest entry with non-null `subjectId` in correlationId thread
3. Fallback to `dispatch.channelId()`

Return type changes from `Uni<Void>` to `Uni<LedgerWriteOutcome>`. Map result:
`new LedgerWriteOutcome(entry.id, resolvedSubjectId, entry.causedByEntryId)`.

Attestation remains deferred (casehub-ledger does not yet expose reactive `LedgerAttestation`
persistence). Replace `logSkippedAttestation` with `LOG.infof` (deferred by design, not an error).
File casehub-ledger issue.

---

## `ReactiveMessageService.dispatch()` — Full Enforcement Sequence

### LAST_WRITE early-exit: discriminated union

The `withTransaction` block returns `Uni<TransactResult>` where:

```java
sealed interface TransactResult permits OverwriteResult, FullResult {}
record OverwriteResult(DispatchResult result) implements TransactResult {}
record FullResult(DispatchContext ctx) implements TransactResult {}
```

Pattern-matching on `TransactResult` drives the chain after Phase 2. `OverwriteResult` carries
a complete `DispatchResult` — Phase 3 and Phase 4 are skipped entirely.

### DispatchContext — primitive carrier across boundaries

```java
record DispatchContext(
    long messageId, UUID commitmentId, Instant occurredAt,
    LedgerWriteOutcome ledgerOutcome, String channelName, int replyCount)
```

Primitives are extracted from the persisted `Message` entity before the transaction closes.
No JPA entities cross the transaction boundary.

---

### Pre-transaction phase — channel load + sync/reactive checks

```
channelStore.find(channelId)   // reactive load, outside tx
  .flatMap(chOpt -> {
    // 1. Paused check — sync
    // 2. ACL — conditional reactive fetch, then sync check
    // 3. Rate limit check — sync, in-memory
    // 4. Trust gate — worker thread hop via ManagedExecutor
    // 5. Type policy — sync, in-memory
  })
```

**Paused check** — sync, same as blocking path.

**ACL (guarded fetch):**
```java
Uni<List<String>> tagsUni =
    (ch != null && ch.allowedWriters != null && !ch.allowedWriters.isBlank()
     && dispatch.type() != MessageType.EVENT)
    ? reactiveInstanceService.findCapabilityTagsForInstance(dispatch.sender())
    : Uni.createFrom().item(List.of());
```
`flatMap` on `tagsUni`, then call `allowedWritersPolicy.isAllowedWriter(sender, ch.allowedWriters,
() -> preResolvedTags)` synchronously. The supplier is `() -> preResolvedTags` — no I/O. Open
channels and EVENT messages skip the DB fetch entirely.

**Rate limit check** — `rateLimiter.check(...)` — in-memory, safe on event loop.

**Trust gate:**
```java
@Inject ManagedExecutor executor;  // CDI/MP context-preserving, not Infrastructure.getDefaultWorkerPool()

Uni<Void> trustCheck = (ch != null && dispatch.type() == MessageType.COMMAND
    && dispatch.target() != null && !dispatch.target().contains(":")
    && config.commitment().minObligorTrust() > 0.0)
    ? Uni.createFrom().item(() -> {
          if (!trustGateService.meetsThreshold(dispatch.target(), config.commitment().minObligorTrust()))
              throw new IllegalStateException("...");
          return null;
      }).runSubscriptionOn(executor).replaceWithVoid()
    : Uni.createFrom().voidItem();
```
One `ManagedExecutor` hop for the JPA query. Control returns to the event loop before
`withTransaction` opens.

**Type policy** — `messageTypePolicy.validate(ch, dispatch.type())` — sync, in-memory.

---

### Phase 2 — Single `Panache.withTransaction("qhorus", ...)`

Returns `Uni<TransactResult>`.

```
1. LAST_WRITE semantics
   reactiveMessageStore.findLastMessage(channelId)
   — Same sender: update in-place, updateLastActivity, recordSend, return OverwriteResult
   — Different sender: throw IllegalStateException
   — No entry: continue

2. Normal insert
   reactiveMessageStore.put(message)
   Extract messageId, commitmentId (UUID), occurredAt as primitives

3. Commitment open — inline, same session/tx
   IF COMMAND or QUERY with non-null correlationId:
     reactiveCommitmentStore.save(new Commitment(...OPEN...))
   Atomic with message insert — eliminates the consistency window where message.commitmentId
   is set but no matching commitment row exists.
   (REQUIRES_NEW in the blocking path was a JTA workaround; Panache has no such constraint.)

4. Reply count
   IF inReplyTo != null:
     reactiveMessageStore.find(inReplyTo).invoke(opt -> opt.ifPresent(p -> p.replyCount++))

5. Channel activity
   reactiveChannelStore.updateLastActivity(ch.id)
   — Targeted UPDATE query; the channel entity stays outside the session (loaded pre-tx).

6. Ledger write
   reactiveLedgerWriteService.record(dispatch, messageId, commitmentId, occurredAt)
   → Uni<LedgerWriteOutcome>

7. Return FullResult(new DispatchContext(messageId, commitmentId, occurredAt,
                                        ledgerOutcome, ch.name, replyCount))
```

---

### Phase 3 — State transitions (ReactiveCommitmentService)

Only invoked from `FullResult` path. Skipped for LAST_WRITE overwrite and EVENT.

```java
reactiveCommitmentService.updateState(dispatch, ctx.commitmentId())
```

`updateState(MessageDispatch dispatch, UUID commitmentId)` — switches on `dispatch.type()`:
- STATUS → `acknowledge(correlationId)`
- RESPONSE, DONE → `fulfill(correlationId)`
- DECLINE → `decline(correlationId)`
- FAILURE → `fail(correlationId)`
- HANDOFF → `delegate(correlationId, dispatch.target())`
- COMMAND, QUERY, EVENT → no-op (open already done in Phase 2)

Each transition opens its own `withTransaction("qhorus", ...)`.

---

### Phase 4 — Post-transaction side effects

```java
.invoke(ctx -> {
    // Observer dispatch — post-commit (correctness fix: blocking path dispatches
    // inside @Transactional, risking observers seeing pre-commit state)
    MessageObserverDispatcher.dispatch(ctx.channelName(), dispatch.channelId(), message, observers.handles());

    // Rate limit recording (EVENT bypasses, same as blocking path)
    if (ch != null && dispatch.type() != MessageType.EVENT)
        rateLimiter.recordSend(ch.id, dispatch.sender(), ch.rateLimitPerChannel, ch.rateLimitPerInstance);

    // fanOut (ch null guard: channel may not exist in gateway registry)
    if (ch != null) {
        try {
            channelGateway.fanOut(ch.id, ch.name, new OutboundMessage(...));
        } catch (Exception e) { /* non-fatal, logged by ChannelGateway per-backend */ }
    }
})
.map(ctx -> new DispatchResult(
    ctx.messageId(), dispatch.channelId(), dispatch.sender(), dispatch.type(),
    dispatch.correlationId(), dispatch.inReplyTo(),
    ArtefactRefParser.parse(dispatch.artefactRefs()), dispatch.target(),
    ctx.ledgerOutcome().entryId(), ctx.ledgerOutcome().subjectId(),
    ctx.ledgerOutcome().causedByEntryId(), ctx.replyCount()))
```

---

### New injections in `ReactiveMessageService`

```java
@Inject ReactiveInstanceService reactiveInstanceService;
@Inject AllowedWritersPolicy allowedWritersPolicy;
@Inject RateLimiter rateLimiter;
@Inject TrustGateService trustGateService;
@Inject MessageTypePolicy messageTypePolicy;
@Inject ReactiveLedgerWriteService reactiveLedgerWriteService;
@Inject ReactiveCommitmentService reactiveCommitmentService;
@Inject ReactiveChannelStore reactiveChannelStore;
@Inject ChannelGateway channelGateway;
@Inject QhorusConfig config;
@Inject ManagedExecutor executor;
```

`AllowedWritersPolicy`, `RateLimiter`, `MessageTypePolicy`, `ChannelGateway`, `QhorusConfig`,
and `ManagedExecutor` carry no `@IfBuildProperty` gate — safe to inject in the reactive service.

---

## `ReactiveCommitmentService` — Design

All state-transition methods mirror `CommitmentService` one-for-one, returning `Uni<?>`.

**`delegate()` — two-save flatMap chain (explicit, not delegated to blocking helper):**

```java
public Uni<Optional<Commitment>> delegate(String correlationId, String delegatedTo) {
    if (correlationId == null || correlationId.isBlank())
        return Uni.createFrom().item(Optional.empty());
    return Panache.withTransaction("qhorus", () ->
        store.findByCorrelationId(correlationId)
            .flatMap(opt -> {
                if (opt.isEmpty() || opt.get().state.isTerminal())
                    return Uni.createFrom().item(Optional.empty());
                Commitment c = opt.get();
                UUID parentId = c.id;
                c.state = CommitmentState.DELEGATED;
                c.delegatedTo = delegatedTo;
                c.resolvedAt = Instant.now();
                return store.save(c).flatMap(saved -> {
                    Commitment child = new Commitment();
                    child.correlationId = correlationId;
                    child.channelId = c.channelId;
                    child.messageType = c.messageType;
                    child.requester = c.requester;
                    child.obligor = delegatedTo;
                    child.expiresAt = c.expiresAt;
                    child.state = CommitmentState.OPEN;
                    child.parentCommitmentId = parentId;
                    return store.save(child).map(ignored -> Optional.of(saved));
                });
            })
    );
}
```

All other transitions (`acknowledge`, `fulfill`, `decline`, `fail`) are straightforward single-save
`transition(correlationId, targetState, consumer)` — exactly as in the blocking service.

`expireOverdue()` uses a `flatMap` to iterate `findExpiredBefore(now())` → save each → count.

---

## Testing

### PostgreSQL DevServices profile

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
# Default datasource stub — satisfies casehub-ledger @Default EntityManager
%reactive-pg.quarkus.datasource.db-kind=h2
%reactive-pg.quarkus.datasource.jdbc.url=jdbc:h2:mem:reactive-ledger-stub;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
%reactive-pg.quarkus.hibernate-orm.database.generation=drop-and-create
# Required: ledger service uses named PU for its own queries
%reactive-pg.casehub.ledger.datasource=qhorus
```

`ReactiveTestProfile.getConfigOverrides()` keeps `casehub.qhorus.reactive.enabled=true`.

### Contract test expansion

`MessageServiceContractTest` (abstract base) gets abstract setup helpers that each runner
provides via its own `@BeforeEach` / helper methods — not datasource-aware, just channel-config
factories. The two concrete runners (blocking H2 via `MessageServiceTest`, reactive PG via
`ReactiveMessageServiceTest`) each satisfy the abstract helper contract through their own
`@QuarkusTest` context.

New test methods added to the base:

| Test | What it asserts |
|------|-----------------|
| `paused_channel_rejects_send` | `IllegalStateException` when channel.paused=true |
| `acl_rejects_unauthorised_sender` | Non-ACL sender → rejected |
| `acl_permits_authorised_sender` | ACL-listed sender → accepted |
| `rate_limit_rejects_burst` | Dispatch until limit fires |
| `type_policy_rejects_disallowed_type` | `MessageTypeViolationException` from dispatch |
| `last_write_same_sender_updates_in_place` | Second dispatch returns same message id |
| `last_write_different_sender_throws` | `IllegalStateException` |
| `commitment_opened_for_command` | CommitmentStore has OPEN entry after COMMAND |
| `ledger_entry_id_populated_in_result` | `DispatchResult.ledgerEntryId()` non-null |
| `observers_see_committed_state` | Observer CDI event fires after tx commits; observer can read message from store |

The last test (`observers_see_committed_state`) verifies the correctness fix: the existing
`ReactiveMessageService` dispatches observers inside `withTransaction` (pre-commit). Moving
dispatch to Phase 4 (post-commit) ensures observers see committed state. The test wires a
`@TestObserver` that reads back from the store inside the handler and asserts the message exists.

`ReactiveMessageServiceTest` removes `@Disabled`.

---

## Out of scope / issues filed

- **casehub-ledger**: Reactive `LedgerAttestation` persistence — file as casehub-ledger issue.
- **casehub-ledger**: `Uni<Boolean> meetsThreshold()` reactive variant on `TrustGateService` — file as casehub-ledger issue; `ManagedExecutor` bridge is the interim solution.
