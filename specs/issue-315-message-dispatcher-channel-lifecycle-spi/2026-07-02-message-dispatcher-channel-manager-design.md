# MessageDispatcher and ChannelManager SPIs — Design Spec

**Issue:** casehubio/qhorus#315
**Date:** 2026-07-02
**Status:** Draft

## Problem

casehubio/blocks#14 needs to dispatch messages and manage channel lifecycles from
`PlatformAgentInvoker`. Currently `MessageService.dispatch()` and
`ChannelService.create()/delete()` live in `casehub-qhorus` runtime — a Tier 3
module. Blocks cannot reference runtime types without violating the tier dependency
model.

The data types (`MessageDispatch`, `DispatchResult`, `ChannelCreateRequest`, `Channel`)
are already in `casehub-qhorus-api` (Tier 1). The missing piece is a service interface
that consumers can program against without pulling in JPA, ledger, CDI containers, etc.

## Design

### Taxonomy of api/ interfaces

The api/ module has three existing categories:

| Package | Role | Consumer relationship |
|---------|------|-----------------------|
| `api/store/` | Data access (CRUD) | Consumer reads/writes domain records |
| `api/spi/` | Extension points | Consumer *provides* custom behavior |
| `api/gateway/` | Integration contracts | Consumer *implements* to integrate |

This design adds a fourth: **service facades** — consumer *calls* to trigger
business-logic-enriched operations (enforcement, ledger, gateway fanout, commitment
lifecycle). These are colocated with their domain types, not in `api/spi/`, because
the consumer relationship is inverted: SPIs are provided by consumers, service facades
are consumed by them.

### New interfaces

#### MessageDispatcher (`api/message/`)

```java
package io.casehub.qhorus.api.message;

public interface MessageDispatcher {
    DispatchResult dispatch(MessageDispatch dispatch);
}
```

```java
package io.casehub.qhorus.api.message;

import io.smallrye.mutiny.Uni;

public interface ReactiveMessageDispatcher {
    Uni<DispatchResult> dispatch(MessageDispatch dispatch);
}
```

Single method. No query methods — consumers use `MessageStore` / `ReactiveMessageStore`
(already in `api/store/`) for reads.

#### ChannelManager (`api/channel/`)

```java
package io.casehub.qhorus.api.channel;

import java.util.List;
import java.util.Set;
import java.util.UUID;
import io.casehub.qhorus.api.message.MessageType;

public interface ChannelManager {
    // Lifecycle
    Channel create(ChannelCreateRequest request);
    FindOrCreateResult findOrCreate(ChannelCreateRequest request);
    long delete(UUID channelId, boolean force);
    Channel pause(UUID channelId);
    Channel resume(UUID channelId);

    // Configuration
    Channel setTypeConstraints(UUID channelId, Set<MessageType> allowedTypes, Set<MessageType> deniedTypes);
    Channel setRateLimits(UUID channelId, Integer perChannel, Integer perInstance);
    Channel setAllowedWriters(UUID channelId, List<String> allowedWriters);
    Channel setAdminInstances(UUID channelId, List<String> adminInstances);
}
```

```java
package io.casehub.qhorus.api.channel;

import java.util.List;
import java.util.Set;
import java.util.UUID;
import io.smallrye.mutiny.Uni;
import io.casehub.qhorus.api.message.MessageType;

public interface ReactiveChannelManager {
    Uni<Channel> create(ChannelCreateRequest request);
    Uni<FindOrCreateResult> findOrCreate(ChannelCreateRequest request);
    Uni<Long> delete(UUID channelId, boolean force);
    Uni<Channel> pause(UUID channelId);
    Uni<Channel> resume(UUID channelId);

    Uni<Channel> setTypeConstraints(UUID channelId, Set<MessageType> allowedTypes, Set<MessageType> deniedTypes);
    Uni<Channel> setRateLimits(UUID channelId, Integer perChannel, Integer perInstance);
    Uni<Channel> setAllowedWriters(UUID channelId, List<String> allowedWriters);
    Uni<Channel> setAdminInstances(UUID channelId, List<String> adminInstances);
}
```

### ChannelCreateRequest — CSV strings → `List<String>`

`ChannelCreateRequest` currently uses `String` (CSV) for `barrierContributors`,
`allowedWriters`, and `adminInstances`. `Channel` already uses `List<String>` for
these fields — the conversion happens inside `Channel.fromRequest()` via
`Channel.splitCsv()`. This is a type mismatch within the Tier 1 API.

**Change:** `ChannelCreateRequest.barrierContributors`, `.allowedWriters`, and
`.adminInstances` change from `String` to `List<String>`. The Builder methods update
accordingly. `Channel.fromRequest()` passes these fields through directly instead of
calling `splitCsv()`. `Channel.splitCsv()` moves entirely to the MCP tool boundary
(see MCP tool boundary adaptation below).

### Signature changes from current ChannelService

| Method | Current (ChannelService) | New (ChannelManager) | Reason |
|--------|--------------------------|----------------------|--------|
| `setAllowedWriters` | `(UUID, String)` — CSV | `(UUID, List<String>)` | Match domain record type |
| `setAdminInstances` | `(UUID, String)` — CSV | `(UUID, List<String>)` | Match domain record type |
| `findOrCreateWithBinding` | name includes impl detail | `findOrCreate` | Binding info is in ChannelCreateRequest |

### findOrCreate semantics

The current `findOrCreateWithBinding` requires a connector binding and uses
`(inboundConnectorId, externalKey)` as the lookup key. The renamed `findOrCreate`
generalises this to dual-mode lookup:

1. **With connector binding** (`request.hasConnectorBinding() == true`): look up by
   `(inboundConnectorId, externalKey)` via `ChannelBindingStore`. If found, return
   existing channel (`wasCreated=false`). If not found, create channel and binding
   (`wasCreated=true`). This preserves current `findOrCreateWithBinding` behavior.

2. **Without connector binding**: look up by channel name via `ChannelStore.findByName()`.
   If found, return existing channel (`wasCreated=false`). If not found, create channel
   (`wasCreated=true`).

This makes `findOrCreate` genuinely general — consumers without connector bindings
can use it for idempotent channel creation by name, while the connector backend
continues to use the binding-based path.

**Concurrency contract:** `findOrCreate` is self-healing on concurrent creation.
When two callers race to create the same channel (by name or by binding key), one
succeeds and the other catches the `PersistenceException` from the unique constraint
violation `(tenancy_id, name)`, retries with a lookup, and returns the existing
channel (`wasCreated=false`). This makes the method genuinely idempotent — callers
never see a constraint violation exception.

### Excluded from SPI

| Method | Why excluded |
|--------|-------------|
| `findById`, `findByName`, `listAll`, `findByNamePrefix`, `findByConnectorKey` | Query methods — use `ChannelStore` |
| `updateLastActivity` | Internal runtime bookkeeping |
| `updateConnectorBinding` | Internal runtime concern |
| `MessageService.findById`, `.pollAfter`, `.findByCorrelationId`, etc. | Query methods — use `MessageStore` |

### Type moves

`FindOrCreateResult` moves from `runtime/channel/` to `api/channel/`. Record unchanged.

### Dead code removal

`ChannelEntity.fromRequest(ChannelCreateRequest, String)` is removed. It has zero
production callers — only 3 test methods in `ChannelFromRequestTest`. The production
path goes through `Channel.fromRequest()` → `ChannelEntity.fromDomain()`, which is
the canonical domain-record-first construction. The method would also fail to compile
after the `ChannelCreateRequest` `String` → `List<String>` change (`barrierContributors`,
`allowedWriters`, `adminInstances` all assign directly to entity `String` fields via
`blankToNull()`). The 3 tests are updated to use the canonical path.

### Runtime implementation

- `ChannelService implements ChannelManager` — `List<String>` params are passed through
  to `Channel` domain record construction (which already uses `List<String>`); CSV
  conversion happens at the JPA entity boundary in `ChannelEntity.fromDomain()`.
  `setAllowedWriters` and `setAdminInstances` drop their `Channel.splitCsv()` calls
  and accept `List<String>` directly.
- `ReactiveChannelService implements ReactiveChannelManager` — same pass-through for
  existing methods. **`findOrCreate` is a new method** — `ReactiveChannelService` does
  not currently have it. The name-based lookup path uses `ReactiveChannelStore.findByName()`
  (already injected). The binding-based path injects blocking `ChannelBindingStore` and
  wraps calls using the established worker-pool pattern:
  `Uni.createFrom().item(() -> channelBindingStore.findByKey(...)).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`.
  This is the same pattern `ReactiveMessageService` uses for the blocking
  `ObligorTrustPolicy` SPI (line 215-216). No new `ReactiveChannelBindingStore` is needed.
  `BlockingTierPurityTest` is unaffected — it checks blocking→Uni direction only; reactive
  services are already allowed to inject blocking dependencies.
- `MessageService implements MessageDispatcher` — no signature change
- `ReactiveMessageService implements ReactiveMessageDispatcher` — no signature change

### MCP tool boundary adaptation

`QhorusMcpTools` and `ReactiveQhorusMcpTools` receive String from the LLM for
`set_allowed_writers`, `set_admin_instances`, `create_channel` (barrier_contributors,
allowed_writers, admin_instances). They split to `List<String>` at the MCP tool boundary
before calling `ChannelManager` or constructing `ChannelCreateRequest`. This is the
correct place for string parsing — the MCP tool is the system boundary.

**Splitting logic:** `Channel.splitCsv()` moves from `Channel` (Tier 1) to
`QhorusMcpToolsBase` (Tier 3) as a `protected static` helper — after this spec's
changes its only callers are MCP tools, and a CSV parsing utility on a Tier 1 domain
record with zero domain callers is an inverted dependency. Semantics are preserved:

| Input | Result | Meaning |
|-------|--------|---------|
| `null` | `null` | Open — no restriction |
| `""` (blank) | `null` | Same as null — open |
| `"alice"` | `["alice"]` | Single entry |
| `"alice,bob"` | `["alice", "bob"]` | Multiple entries, trimmed |
| `"alice,,bob"` | `["alice", "bob"]` | Empty segments filtered |

`Channel.fromRequest()` and `ChannelService.setAllowedWriters/setAdminInstances` no
longer call it — they receive `List<String>` directly.

### What does NOT change

- No Flyway migrations
- No new dependencies (mutiny already `provided` in api/)
- No changes to `persistence-memory/` — no in-memory service layer exists
- Store interfaces unchanged
- `api/spi/` extension points unchanged

## Testing

- Existing `ChannelServiceTest` and `MessageServiceTest` continue to pass — the services
  now implement interfaces but behavior is identical for existing methods
- **New test:** `ChannelService.findOrCreate()` name-based lookup path (no connector
  binding). The binding-based path is already tested by `ChannelServiceFindOrCreateTest`.
- **New test:** `ReactiveChannelService.findOrCreate()` — name-based lookup path
- **New test:** `ReactiveChannelService.findOrCreate()` — binding-based lookup path
  (worker-pool wrapped `ChannelBindingStore` calls). Must verify: (1) returns existing
  channel when binding exists, (2) creates channel and binding when not found, (3)
  self-healing concurrency contract works through the reactive pipeline
  (`PersistenceException` recovery via catch-retry)
- MCP tool tests updated for the `List<String>` boundary adaptation (both
  `set_allowed_writers`/`set_admin_instances` and `create_channel`)
- `ChannelCreateRequest` builder call sites updated from `String` to `List<String>` — affects
  `QhorusMcpTools.createChannel()`, `ReactiveQhorusMcpTools.createChannel()`, and test fixtures
- `FindOrCreateResult` import changes are mechanical — compile verifies correctness
- Full `mvn install` from project root to catch cross-module breakage (per CLAUDE.md)
