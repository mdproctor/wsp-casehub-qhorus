# Channel Null Lists, findOrCreate Race, and Reactive Create Parity

**Issues:** #319, #317, #318
**Date:** 2026-07-04
**Branch:** `issue-319-channel-null-lists-and-fixes`

---

## Summary

Three related fixes to Channel creation and the Channel domain record:

1. **#319** — `Channel.builder().build()` leaves list fields (`allowedWriters`, `barrierContributors`, `adminInstances`) null. Downstream consumers NPE when iterating or calling `.size()` without null checks.
2. **#317** — `ChannelService.findOrCreate()` name-based race recovery catches `PersistenceException` and retries a query inside the same `@Transactional(REQUIRES_NEW)` transaction. H2 permits this; PostgreSQL aborts the entire transaction on constraint violation, rejecting subsequent queries.
3. **#318** — `ReactiveChannelService.create()` does not call `channelGateway.initChannel()` or create connector bindings. Consumers switching from blocking to reactive `create()` silently lose backend registration and binding creation.

---

## Design

### 1. Null list normalization (#319)

**Records affected:** `Channel` and `ChannelCreateRequest` (both in `api/`).

**Change:** In both records' compact constructors, the three list fields normalize null to `List.of()`:

```java
// Before:
barrierContributors = barrierContributors != null ? List.copyOf(barrierContributors) : null;
// After:
barrierContributors = barrierContributors != null ? List.copyOf(barrierContributors) : List.of();
```

Applied to: `barrierContributors`, `allowedWriters`, `adminInstances`.

**Set fields unchanged:** `allowedTypes` and `deniedTypes` preserve null. Null means "no constraint" (open), which is semantically distinct from an empty set ("nothing permitted"). The service layer (`setTypeConstraints`) explicitly normalizes empty sets to null at the write boundary, confirming this distinction is intentional.

**Rationale:** For all three list fields, null and empty have identical runtime semantics across the entire platform:
- `AllowedWritersPolicy`: `if (allowedWriters == null || allowedWriters.isEmpty()) return true;`
- `QhorusMcpToolsBase`: `if (ch.adminInstances() == null || ch.adminInstances().isEmpty())`
- `WatchdogEvaluationService`: `ch.barrierContributors() != null ? ch.barrierContributors() : List.of()`
- `ReactiveMessageService`: `ch != null && ch.allowedWriters() != null && !ch.allowedWriters().isEmpty()` (pre-check before `AllowedWritersPolicy`)

No code path distinguishes null from empty for these fields.

**Persistence alignment:** `ChannelEntity.joinCsv()` updated to return null for empty lists:

```java
private static String joinCsv(List<String> list) {
    return list == null || list.isEmpty() ? null : String.join(",", list);
}
```

Prevents `""` appearing in the DB column where null was stored before. `splitCsv(null)` returns null, which Channel's constructor normalizes to `List.of()` on read.

**QhorusEntityMapper alignment:** `QhorusEntityMapper.joinCsv()` (line 121) has the same pattern — after null→`List.of()` normalization, `String.join(",", List.of())` returns `""` instead of null, changing the MCP API contract for `ChannelDetail` fields. `ChannelDetail`'s Javadoc documents null semantics explicitly: "or null if management is open to any caller." Apply the same empty-list→null fix:

```java
private static String joinCsv(List<String> list) {
    return list == null || list.isEmpty() ? null : String.join(",", list);
}
```

**Comment update:** `ChannelCreateRequest`'s compact constructor has an inline comment (lines 53–56) that reads "Null is preserved (not normalized to Set.of()) — null means 'open' and is a meaningful contract distinct from 'empty allowed set (nothing permitted)'." This comment spans all five defensive-copy lines (three list fields + two set fields). After this fix, it's only correct for the set fields. Update the comment to clarify: set fields preserve null (open vs empty are semantically distinct), list fields normalize null→`List.of()` (null and empty are equivalent).

**Test impact:** Tests asserting `assertNull(ch.allowedWriters())` change to `assertThat(ch.allowedWriters()).isEmpty()`. Mechanical migration.

### 2. findOrCreate PostgreSQL race recovery (#317)

**Root cause:** `findOrCreate()` is `@Transactional(REQUIRES_NEW)`. It calls `create()` via self-invocation (CDI proxy bypass), so `create()` runs in the same REQUIRES_NEW transaction. On unique constraint violation, PostgreSQL marks the entire transaction as aborted. The retry query in the catch block fails with "current transaction is aborted, commands ignored until end of transaction block."

Garden entry: GE-20260703-30313f.

**Fix:** Extract channel creation into a package-private `ChannelCreateHelper` bean. The helper's `@Transactional(REQUIRES_NEW)` goes through the CDI proxy, giving it an isolated transaction that can fail without corrupting the outer transaction.

**New bean:** `ChannelCreateHelper` in `io.casehub.qhorus.runtime.channel`:

```java
@ApplicationScoped
class ChannelCreateHelper {
    @Inject ChannelStore channelStore;
    @Inject ChannelBindingStore channelBindingStore;
    @Inject ChannelGateway channelGateway;
    @Inject CurrentPrincipal currentPrincipal;

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    Channel createInNewTransaction(ChannelCreateRequest req, boolean autoCreated) {
        Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());
        if (autoCreated) {
            channel = channel.toBuilder().autoCreated(true).build();
        }
        channel = channelStore.put(channel);
        if (req.hasConnectorBinding()) {
            channelBindingStore.findByKey(req.inboundConnectorId(), req.externalKey())
                    .ifPresent(existing -> {
                        throw new IllegalStateException(
                                "Connector binding already exists for connector '"
                                + req.inboundConnectorId() + "' key '" + req.externalKey() + "'");
                    });
            channelBindingStore.put(new ChannelConnectorBinding(
                    channel.id(), req.inboundConnectorId(), req.externalKey(),
                    req.outboundConnectorId(), req.outboundDestination()));
        }
        channelGateway.initChannel(channel.id(), new ChannelRef(channel.id(), channel.name()));
        return channel;
    }
}
```

**ChannelService changes:**

- `create()` delegates to `channelCreateHelper.createInNewTransaction(req, false)`. Its `@Transactional` annotation remains (no-op shell around the helper's REQUIRES_NEW for standalone calls).
- `findOrCreateByName()` calls the helper via CDI. On `PersistenceException`, the helper's REQUIRES_NEW rolls back independently; `findOrCreate`'s outer REQUIRES_NEW stays clean for the retry query.
- `findOrCreateWithBinding()` delegates to `channelCreateHelper.createInNewTransaction(req, true)` — the `autoCreated` flag is passed through, preserving the existing behavior where connector-triggered channels are marked as auto-created.

**Transaction isolation change:** Standalone callers of `create()` previously participated in the caller's transaction (REQUIRED). With the helper's REQUIRES_NEW, channel creation is always isolated — if the caller's outer transaction rolls back, the channel persists. This is intentional: `findOrCreate` already used REQUIRES_NEW, and channel creation should be atomic and independent. The `initChannel()` CDI event fires during the helper's transaction, so rolling back the channel after observers have already registered backends would leave ghost state. All current production call paths to `create()` route through `findOrCreateByName()` (already REQUIRES_NEW) or the MCP tool layer (no outer transaction), so the effective behavior for current callers is unchanged.

**Transaction flow on race:**

```
findOrCreate (REQUIRES_NEW → tx1)
  └─ findOrCreateByName
       ├─ channelStore.findByName() in tx1 → empty
       ├─ helper.createInNewTransaction() via CDI → REQUIRES_NEW → tx2
       │     └─ channelStore.put() → PersistenceException → tx2 rolls back
       └─ catch → channelStore.findByName() in tx1 (clean) → winner found
```

**Reactive path:** Hibernate Reactive has no REQUIRES_NEW. Recovery uses `onFailure(PersistenceException.class).recoverWithUni()`. The failed `withTransaction` completes (rolls back), and the recovery starts a fresh session:

```java
private Uni<FindOrCreateResult> findOrCreateByName(ChannelCreateRequest req) {
    return Panache.withTransaction("qhorus", () ->
            channelStore.findByName(req.name())
                    .flatMap(existing -> {
                        if (existing.isPresent()) {
                            return Uni.createFrom().item(new FindOrCreateResult(existing.get(), false));
                        }
                        return create(req).map(ch -> new FindOrCreateResult(ch, true));
                    }))
            .onFailure(PersistenceException.class).recoverWithUni(ex ->
                    Panache.withSession("qhorus", () ->
                            channelStore.findByName(req.name())
                                    .map(opt -> opt.map(ch -> new FindOrCreateResult(ch, false))
                                            .orElseThrow(() -> (RuntimeException) ex))));
}
```

### 3. ReactiveChannelService.create() parity (#318)

**Missing from reactive `create()`:** connector binding creation and `channelGateway.initChannel()`.

**Fix:** Add both inside the `withTransaction`, offloaded to the worker pool. `channelBindingStore` is a blocking JPA store, and `channelGateway.initChannel()` fires synchronous CDI events whose observers (`ConnectorChannelBackend.onChannelInitialised`, `SlackChannelBackend.onChannelInitialised`) perform blocking I/O (database lookups, cache warming). Both must be offloaded via `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`, matching the existing pattern in `findOrCreateWithBinding`. `ReactiveChannelService` also needs `@Inject ChannelGateway channelGateway` added (currently not injected).

```java
@Override
public Uni<Channel> create(ChannelCreateRequest req) {
    Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());
    return Panache.withTransaction("qhorus", () ->
            channelStore.put(channel)
                    .chain(ch -> Uni.createFrom().item(() -> {
                        if (req.hasConnectorBinding()) {
                            channelBindingStore.findByKey(
                                    req.inboundConnectorId(), req.externalKey())
                                    .ifPresent(existing -> {
                                        throw new IllegalStateException(
                                                "Connector binding already exists for connector '"
                                                + req.inboundConnectorId() + "' key '"
                                                + req.externalKey() + "'");
                                    });
                            channelBindingStore.put(new ChannelConnectorBinding(
                                    ch.id(), req.inboundConnectorId(), req.externalKey(),
                                    req.outboundConnectorId(), req.outboundDestination()));
                        }
                        channelGateway.initChannel(ch.id(),
                                new ChannelRef(ch.id(), ch.name()));
                        return ch;
                    }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())));
}
```

**Also fix: reactive `findOrCreateWithBinding` create path** — add `initChannel()` call inside the existing worker pool block, after binding creation:

```java
return Panache.withTransaction("qhorus", () ->
        channelStore.put(autoCreated)
                .chain(saved -> Uni.createFrom().item(() -> {
                    channelBindingStore.put(new ChannelConnectorBinding(
                            saved.id(), req.inboundConnectorId(), req.externalKey(),
                            req.outboundConnectorId(), req.outboundDestination()));
                    channelGateway.initChannel(saved.id(),
                            new ChannelRef(saved.id(), saved.name()));
                    return new FindOrCreateResult(saved, true);
                }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())));
```

This is already offloaded to the worker pool, so the blocking `initChannel()` call (with its CDI event observers) is safe here.

**Protocol update:** PP-20260609-fe1300 (`channel-create-requires-init-channel`) documented this as a caller responsibility. The blocking `ChannelService.create()` already calls `initChannel()` internally (pre-dates this spec). This fix brings the reactive `ReactiveChannelService.create()` into alignment. After this fix, both `create()` implementations call initChannel internally. The protocol should be updated to mark both paths as internally handled, noting that the blocking path was already fixed.

---

## Scope

**Files modified (production):**
- `api/src/main/java/.../channel/Channel.java` — compact constructor null normalization
- `api/src/main/java/.../channel/ChannelCreateRequest.java` — compact constructor null normalization
- `runtime/src/main/java/.../channel/ChannelEntity.java` — `joinCsv` empty list handling
- `runtime/src/main/java/.../channel/ChannelCreateHelper.java` — new bean
- `runtime/src/main/java/.../channel/ChannelService.java` — delegate to helper
- `runtime/src/main/java/.../channel/ReactiveChannelService.java` — create parity + race recovery + `@Inject ChannelGateway channelGateway`
- `runtime/src/main/java/.../QhorusEntityMapper.java` — `joinCsv` empty list handling

**Files modified (test):**
- `api/src/test/java/.../channel/ChannelTest.java` — null → isEmpty assertions
- `runtime/src/test/java/.../channel/ChannelFromRequestTest.java` — null → isEmpty assertions
- `runtime/src/test/java/.../conversion/EntityConversionTest.java` — null → isEmpty assertions
- `runtime/src/test/java/.../channel/ChannelServiceTest.java` — null → isEmpty assertions
- `runtime/src/test/java/.../channel/ChannelServiceFindOrCreateTest.java` — race recovery test
- `runtime/src/test/java/.../mcp/ChannelWritePermissionsTest.java` — null → isEmpty assertions
- `runtime/src/test/java/.../mcp/ChannelAdminRoleTest.java` — null → isEmpty assertions

**Protocol update:**
- `casehub/garden/docs/protocols/casehub/channel-create-requires-init-channel.md` — note that create() now calls initChannel internally

**No Flyway migration needed.** DB representation unchanged — null columns stay null. `joinCsv(List.of())` returns null (same as before for unset lists).

---

## What this does NOT change

- Set field (`allowedTypes`/`deniedTypes`) null semantics — preserved
- `ChannelService.findOrCreate()` public API contract — still self-contained, still REQUIRES_NEW
- `ChannelService.create()` public API signature — still `@Transactional`, still returns Channel (note: transaction isolation semantics changed from REQUIRED to effective REQUIRES_NEW — see §2 "Transaction isolation change")
- Reactive beans' build-time gating — unchanged
- Message dispatch enforcement gate — unchanged
