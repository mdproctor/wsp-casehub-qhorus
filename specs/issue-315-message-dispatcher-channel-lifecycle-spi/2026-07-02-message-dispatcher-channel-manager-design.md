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

### Signature changes from current ChannelService

| Method | Current (ChannelService) | New (ChannelManager) | Reason |
|--------|--------------------------|----------------------|--------|
| `setAllowedWriters` | `(UUID, String)` — CSV | `(UUID, List<String>)` | Match domain record type |
| `setAdminInstances` | `(UUID, String)` — CSV | `(UUID, List<String>)` | Match domain record type |
| `findOrCreateWithBinding` | name includes impl detail | `findOrCreate` | Binding info is in ChannelCreateRequest |

### Excluded from SPI

| Method | Why excluded |
|--------|-------------|
| `findById`, `findByName`, `listAll`, `findByNamePrefix`, `findByConnectorKey` | Query methods — use `ChannelStore` |
| `updateLastActivity` | Internal runtime bookkeeping |
| `updateConnectorBinding` | Internal runtime concern |
| `MessageService.findById`, `.pollAfter`, `.findByCorrelationId`, etc. | Query methods — use `MessageStore` |

### Type moves

`FindOrCreateResult` moves from `runtime/channel/` to `api/channel/`. Record unchanged.

### Runtime implementation

- `ChannelService implements ChannelManager` — `List<String>` params are passed through
  to `Channel` domain record construction (which already uses `List<String>`); CSV
  conversion happens at the JPA entity boundary in `ChannelEntity.fromDomain()`
- `ReactiveChannelService implements ReactiveChannelManager` — same pass-through
- `MessageService implements MessageDispatcher` — no signature change
- `ReactiveMessageService implements ReactiveMessageDispatcher` — no signature change

### MCP tool boundary adaptation

`QhorusMcpTools` and `ReactiveQhorusMcpTools` receive String from the LLM for
`set_allowed_writers` and `set_admin_instances`. They split to `List<String>` at the
MCP tool boundary before calling `ChannelManager`. This is the correct place for
string parsing — the MCP tool is the system boundary.

### What does NOT change

- No Flyway migrations
- No new test infrastructure
- No new dependencies (mutiny already `provided` in api/)
- No changes to `persistence-memory/` — no in-memory service layer exists
- Store interfaces unchanged
- `api/spi/` extension points unchanged

## Testing

- Existing `ChannelServiceTest` and `MessageServiceTest` continue to pass — the services
  now implement interfaces but behavior is identical
- MCP tool tests updated for the `List<String>` boundary adaptation
- `FindOrCreateResult` import changes are mechanical — compile verifies correctness
- Full `mvn install` from project root to catch cross-module breakage (per CLAUDE.md)
