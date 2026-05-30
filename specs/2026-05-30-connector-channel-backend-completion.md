# Connector Channel Backend — Completion Design
**Issue:** casehubio/qhorus#219  
**Date:** 2026-05-30  
**Branch:** issue-219-connector-channel-backend

---

## Context

The bulk of #219 was pre-implemented as untracked files: `ConnectorChannelBackend`, `ChannelConnectorBinding`, `ChannelBindingStore` (interface + JPA + InMemory), `ChannelCreateRequest`, `ChannelService` additions, V14 migration, all unit tests, and an integration test skeleton. `QhorusEntityMapper` and `QhorusDashboardService` were also partially pre-written with `ChannelDetail.ConnectorBinding` references that do not compile yet because `ChannelDetail` itself is missing the field.

The pre-written mapper and dashboard both use **Option A** (mapper injects `ChannelBindingStore` and queries per-channel). The design decision (approved in brainstorming) is **Option B** (caller passes the binding, mapper stays a pure transformer). This spec describes the refactor needed and all remaining gaps.

---

## Gap 1 — `ChannelDetail.ConnectorBinding` (api module)

Add a nested record and a nullable field to `ChannelDetail`:

```java
public record ChannelDetail(
    UUID channelId,
    String name,
    String description,
    String semantic,
    String barrierContributors,
    long messageCount,
    String lastActivityAt,
    boolean paused,
    String allowedWriters,
    String adminInstances,
    Integer rateLimitPerChannel,
    Integer rateLimitPerInstance,
    String allowedTypes,
    ConnectorBinding connectorBinding   // null if no binding
) {
    public record ConnectorBinding(
        String inboundConnectorId,
        String externalKey,
        String outboundConnectorId,
        String outboundDestination
    ) {}
}
```

This is a binary break on the canonical constructor. All direct constructions of `ChannelDetail` must be updated. Currently: `QhorusEntityMapper` and `QhorusDashboardService` (private mapping method). Both are already written with the 14-arg form referencing `ConnectorBinding` — they just don't compile yet.

---

## Gap 2 — Refactor `QhorusEntityMapper` to Option B

**Current state (pre-written, Option A):** `QhorusEntityMapper` injects `ChannelBindingStore` and queries it per-channel inside `toChannelDetail(Channel, long)`.

**Target (Option B):** Remove `ChannelBindingStore` injection from `QhorusEntityMapper`. Change signature to:

```java
public ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                     Optional<ChannelConnectorBinding> binding)
```

The mapper converts `binding` to `ChannelDetail.ConnectorBinding` if present, null otherwise. No store dependency. No query. Input in / output out.

**`QhorusMcpToolsBase.toChannelDetail()` wrapper** — the base class wraps the mapper call. It also needs the binding parameter:

```java
protected ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                        Optional<ChannelConnectorBinding> binding) {
    return entityMapper.toChannelDetail(ch, messageCount, binding);
}
```

---

## Gap 3 — `ChannelBindingStore.findAll()` for batch loading

Add to the `ChannelBindingStore` interface:

```java
Map<UUID, ChannelConnectorBinding> findAll();
```

Implementations:
- `JpaChannelBindingStore`: `ChannelConnectorBinding.<ChannelConnectorBinding>listAll().stream().collect(toMap(b -> b.channelId, b -> b))`
- `InMemoryChannelBindingStore`: `Map.copyOf(byChannelId)`

Add a `findAll` contract test to `ChannelBindingStoreContractTest`.

---

## Gap 4 — Update all `toChannelDetail` call sites

### `QhorusMcpTools` (blocking)

Seven call sites. Pattern:

- **Single-channel calls** (create, get, pause, resume, update, delete): `bindingStore.findByChannelId(ch.id)` then pass result
- **`list_channels`**: call `bindingStore.findAll()` once before the stream, look up by `ch.id` per iteration — eliminates N+1

`QhorusMcpTools` already injects `ChannelBindingStore` (it uses the seam for channel operations). No new injection needed.

### `ReactiveQhorusMcpTools` (reactive)

Same pattern. The binding lookup for single-channel calls is blocking; wrap in `Uni.createFrom().item(...)` if needed, or call on the worker thread (MCP tool methods annotated `@Blocking` are safe to block). For `list_channels`, load all bindings with `Uni.createFrom().item(() -> bindingStore.findAll())` before the reactive chain, then pass by channelId.

### `QhorusDashboardService` (reactive)

**Current state:** has its own private `toChannelDetail(Channel, int)` that duplicates mapper logic — this duplication goes away. All mapping delegates to `entityMapper.toChannelDetail()`.

In `listChannels()`: load bindings before the reactive chain:

```java
public Uni<List<ChannelDetail>> listChannels() {
    return channelService.listAll().flatMap(channels -> {
        if (channels.isEmpty()) return Uni.createFrom().item(List.of());
        Map<UUID, ChannelConnectorBinding> bindings =
            Uni.createFrom().item(bindingStore::findAll).await().indefinitely();
        // ... map channels using entityMapper.toChannelDetail(ch, count, Optional.ofNullable(bindings.get(ch.id)))
    });
}
```

Wait — `await().indefinitely()` on a Vert.x event loop thread blocks the loop. The correct pattern for `QhorusDashboardService`:

```java
return Uni.createFrom().item(bindingStore::findAll)   // blocking Uni — runs on worker pool
    .flatMap(bindings -> channelService.listAll()
        .flatMap(channels -> { ... }));
```

Or keep it simple: since the dashboard is reactive and `bindingStore.findAll()` is blocking, wrap it:

```java
Uni<Map<UUID, ChannelConnectorBinding>> bindingUni =
    Uni.createFrom().item(bindingStore::findAll);
return bindingUni.flatMap(bindings -> channelService.listAll()
    .flatMap(channels -> { ... use bindings ... }));
```

Quarkus runs `Uni.createFrom().item(supplier)` on the caller's thread for the subscribe — if that's a Vert.x event loop, it blocks. The safe way is `Uni.createFrom().item(bindingStore::findAll).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`. However, `QhorusDashboardService` is `@IfBuildProperty(name = "quarkus.datasource.qhorus.reactive", stringValue = "true")` — it is only active in the reactive stack where MCP tool methods are `@Blocking`. In that context, the initial Uni subscription happens on a worker thread. Treat the whole dashboard method as worker-initiated and use the simple form.

The private `toChannelDetail()` method in `QhorusDashboardService` is removed entirely. All channel detail mapping goes through `entityMapper.toChannelDetail(ch, count, Optional.ofNullable(bindings.get(ch.id)))`.

---

## Gap 5 — Fix integration test `@ObservesAsync` reliability

GE-20260513-b15933: `@ObservesAsync` events are silently not delivered in `@QuarkusTest`. The integration test fires through `inboundConnectorService.receive(msg)` and waits with `verify(timeout(2000))` and `Thread.sleep(300)` — flaky.

**Fix:** Remove `@Inject InboundConnectorService`. Call `backend.onInboundMessage(msg)` directly through the injected CDI proxy. This is synchronous and deterministic. Remove all `Thread.sleep` and `timeout()` Mockito calls. The `fanOut_sendsViaConnectorService` test drives `gateway.fanOut()` directly — no sleep needed there either.

---

## Gap 6 — `NativeImageResourcePatternsBuildItem` in `QhorusProcessor`

Per PP-20260528-flyway-ext-reg. Add to `QhorusProcessor`:

```java
@BuildStep
NativeImageResourcePatternsBuildItem registerMigrationResources() {
    return NativeImageResourcePatternsBuildItem.builder()
            .includeGlob("db/qhorus/migration/*.sql")
            .build();
}
```

Covers all Qhorus domain migrations in one glob.

---

## Gap 7 — PLATFORM.md Cross-Repo Dependency Map

Per PP-20260523-605b90. Add row to `casehub-parent/docs/PLATFORM.md`:

| `casehub-connectors-core` | `casehub-qhorus` | `connector-backend` | optional — `InboundMessage` CDI events → `ConnectorChannelBackend` → Qhorus channel routing; activates by classpath presence |

Extend the build-order comment for `casehub-qhorus` to mention `connector-backend` alongside the existing `connectors` optional module.

---

## Gap 8 — Protocol update: bridge module placement exception

`cross-foundation-bridge-module-placement.md` (PP-20260528-6b1d80) states bridges live in the event-source repo. This would create a circular dependency here: `connector-backend` needs qhorus runtime beans, so it cannot live in `casehub-connectors`. Update the protocol with: **when the bridge requires the consuming repo's runtime (not just its api module), it lives in the consumer's repo.**

---

## Testing Summary

| Test class | Type | Status / Change |
|---|---|---|
| `ConnectorKeyStrategyTest` | Pure unit | Complete ✅ |
| `OutboundTitleTest` | Pure unit | Complete ✅ |
| `ConnectorChannelBackendTest` | Unit (Mockito) | Complete ✅ |
| `InMemoryChannelBindingStoreTest` | Contract runner | Complete ✅ |
| `ChannelBindingStoreContractTest` | Abstract base | Add `findAll` test case |
| `ConnectorChannelBackendIntegrationTest` | `@QuarkusTest` | Fix: drop `inboundConnectorService`, call `backend.onInboundMessage()` directly, remove sleeps |

---

## Out of Scope (separate issues)

- `#218` — consolidate `ChannelService.create()` overloads: old overloads remain; `create(ChannelCreateRequest)` added ✅
- `#215` — fire `ChannelInitialisedEvent` on binding update: disabled test `cacheRefreshesAfterBindingUpdate` is the regression harness
- `#214`, `#216`, `#217` — v2 connector features: tracked separately
