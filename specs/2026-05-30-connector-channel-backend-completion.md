# Connector Channel Backend — Completion Design
**Issue:** casehubio/qhorus#219  
**Date:** 2026-05-30  
**Branch:** issue-219-connector-channel-backend

---

## Context

The bulk of #219 was pre-implemented as untracked files: `ConnectorChannelBackend`, `ChannelConnectorBinding`, `ChannelBindingStore` (interface + JPA + InMemory), `ChannelCreateRequest`, `ChannelService` additions, V14 migration, all unit tests, and an integration test skeleton. `QhorusEntityMapper` and `QhorusDashboardService` were also pre-written and already reference `ChannelDetail.ConnectorBinding` (which exists). `ChannelDetail` is already updated with the 14-arg constructor and nested `ConnectorBinding` record. The mapper compiles against the current api module.

The pre-written mapper uses **Option A** (mapper injects `ChannelBindingStore`, queries per-channel). The approved design is **Option B** (caller supplies the binding, mapper stays a pure transformer). This spec describes the refactor and all remaining gaps.

---

## Gap 1 — Refactor `QhorusEntityMapper` to Option B

**Current state (Option A):** `QhorusEntityMapper` injects `ChannelBindingStore` and queries it per-channel inside `toChannelDetail(Channel, long)`.

**Target (Option B):** Remove `ChannelBindingStore` injection from `QhorusEntityMapper`. New signature:

```java
public ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                     Optional<ChannelConnectorBinding> binding)
```

The mapper maps `binding` to `ChannelDetail.ConnectorBinding` if present, null otherwise. No store dependency. No query side-effect.

The public testing constructor changes from `QhorusEntityMapper(ObjectMapper, ChannelBindingStore)` to `QhorusEntityMapper(ObjectMapper)`. **`QhorusDashboardServiceTest`** constructs the mapper at line 60 — update to the new signature.

---

## Gap 2 — `ChannelBindingStore` in `QhorusMcpToolsBase`

Neither `QhorusMcpTools` nor `ReactiveQhorusMcpTools` currently injects `ChannelBindingStore`. Under Option B, the lookup responsibility moves to callers. The cleanest location is `QhorusMcpToolsBase`, which already injects `QhorusEntityMapper` and is the base for both concrete tool classes.

Add to `QhorusMcpToolsBase`:

```java
@Inject ChannelBindingStore bindingStore;

// Single-item path — base class resolves the lookup
protected ChannelDetail toChannelDetail(Channel ch, long messageCount) {
    return entityMapper.toChannelDetail(ch, messageCount,
            bindingStore.findByChannelId(ch.id));
}

// Batch path — caller pre-fetches, base class resolves by key
protected ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                        Map<UUID, ChannelConnectorBinding> allBindings) {
    return entityMapper.toChannelDetail(ch, messageCount,
            Optional.ofNullable(allBindings.get(ch.id)));
}
```

`QhorusMcpTools` and `ReactiveQhorusMcpTools` need **zero new injections**. All single-channel call sites (`create_channel`, `get_channel`, `pause_channel`, etc.) already call `toChannelDetail(ch, count)` — the single-item overload handles the lookup. The only call site that must explicitly use the batch path is **`list_channels`**:

```java
// list_channels in QhorusMcpTools
Map<UUID, ChannelConnectorBinding> allBindings = bindingStore.findAll();
return channels.stream()
    .map(ch -> toChannelDetail(ch, countByChannel.getOrDefault(ch.id, 0L), allBindings))
    .toList();
```

`find_channel` iterates filtered matches (typically small sets) and uses the single-item path — one `findByChannelId` per match. This is intentionally not optimised; acceptable for typical search result sizes.

---

## Gap 3 — `ChannelBindingStore.findAll()`

Add to the interface:

```java
Map<UUID, ChannelConnectorBinding> findAll();
```

Implementations:
- `JpaChannelBindingStore`: `ChannelConnectorBinding.<ChannelConnectorBinding>listAll().stream().collect(toMap(b -> b.channelId, b -> b))`
- `InMemoryChannelBindingStore`: `Map.copyOf(byChannelId)`

Add four contract test cases to `ChannelBindingStoreContractTest`:
1. `findAll_emptyStore_returnsEmptyMap`
2. `findAll_afterPut_containsBinding`
3. `findAll_afterDelete_excludesDeletedBinding`
4. `findAll_returnsSnapshotNotLiveView` — call `findAll()`, mutate store, verify returned map is unchanged

---

## Gap 4 — `QhorusDashboardService` update

**Current state:** has a private `toChannelDetail(Channel, int)` that duplicates mapper logic with Option A binding lookup.

**Target:** remove the private method entirely. All channel detail mapping goes through `entityMapper.toChannelDetail()` via `QhorusMcpToolsBase` — except `QhorusDashboardService` does not extend the base. It directly injects `QhorusEntityMapper` (already present) and needs a `ChannelBindingStore` injection (new).

In `listChannels()`, load bindings before the reactive channel chain. Since `bindingStore.findAll()` is blocking and `listChannels()` returns `Uni<T>` without a `@Blocking` annotation, the blocking call must be explicitly dispatched to the worker pool:

```java
public Uni<List<ChannelDetail>> listChannels() {
    return Uni.createFrom().item(bindingStore::findAll)
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
            .flatMap(bindings -> channelService.listAll().flatMap(channels -> {
                if (channels.isEmpty()) return Uni.createFrom().item(List.of());
                List<Uni<ChannelDetail>> unis = channels.stream()
                    .map(ch -> messageStore.countByChannel(ch.id)
                        .map(count -> entityMapper.toChannelDetail(ch, count,
                                Optional.ofNullable(bindings.get(ch.id)))))
                    .toList();
                return Uni.join().all(unis).andFailFast();
            }));
}
```

`runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` is required — without it, `bindingStore::findAll` (blocking JPA) runs on the Vert.x event loop thread and blocks it.

---

## Gap 5 — Fix integration test

**`ConnectorChannelBackendIntegrationTest` changes:**

1. **Remove `@Inject InboundConnectorService inboundConnectorService`** — no longer used. Calling through `InboundConnectorService.receive()` → `@ObservesAsync` is unreliable in `@QuarkusTest` (GE-20260513-b15933). Call `backend.onInboundMessage(msg)` directly through the injected CDI proxy instead.

2. **`inboundMessage_routesToMessageService`:** Call `backend.onInboundMessage(msg)` directly. The chain is fully synchronous: InMemoryChannelBindingStore lookup → `gateway.receiveHumanMessage()` → mocked `messageService.dispatch()`. Replace `verify(messageService, timeout(2000)).dispatch(...)` with plain `verify(messageService).dispatch(...)`.

3. **`unknownSender_noChannelBound_discardCounterIncremented`:** Call `backend.onInboundMessage(msg)` directly. Remove `verify(messageService, after(1000).never()).dispatch(any())` — synchronous call means dispatch is never called before the line returns. The counter increment is also synchronous; remove the polling loop and assert directly.

4. **`fanOut_sendsViaConnectorService`:** Remove the first routing block entirely — `@BeforeEach` calls `gateway.initChannel()` which fires the synchronous `@Observes ChannelInitialisedEvent`, populating the cache before the test body runs. The fanOut call itself is sufficient:

    ```java
    @Test
    void fanOut_sendsViaConnectorService() {
        OutboundMessage outbound = new OutboundMessage(UUID.randomUUID(), "agent",
                MessageType.RESPONSE, "We can help", null, null, ActorType.AGENT);
        gateway.fanOut(channelId, "sms-alice", outbound);

        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService, timeout(1000).atLeastOnce()).send(eq("twilio-sms"), captor.capture());
        assertThat(captor.getValue().destination()).isEqualTo("+15551110000");
        assertThat(captor.getValue().body()).isEqualTo("We can help");
    }
    ```

    `timeout(1000)` is still required on `connectorService.send()` — `gateway.fanOut()` dispatches `backend.post()` on a virtual thread (`Thread.ofVirtual().start()`).

5. **Coverage gap acknowledged:** No test exercises the CDI async path `InboundConnectorService.receive()` → `Event.fireAsync()` → `@ObservesAsync ConnectorChannelBackend.onInboundMessage`. Tracked as **casehubio/qhorus#221**.

---

## Gap 6 — `NativeImageResourcePatternsBuildItem` in `QhorusProcessor`

Per PP-20260528-flyway-ext-reg. `QhorusProcessor` currently only has a `FeatureBuildItem` step. Add:

```java
@BuildStep
NativeImageResourcePatternsBuildItem registerMigrationResources() {
    return NativeImageResourcePatternsBuildItem.builder()
            .includeGlob("db/qhorus/migration/*.sql")
            .includeGlob("db/ledger/migration/*.sql")
            .build();
}
```

Two globs are required: `db/qhorus/migration/*.sql` for Qhorus domain migrations, and `db/ledger/migration/*.sql` for ledger base migrations. `LedgerProcessor` (casehub-ledger deployment) does **not** register its own SQL files for native image — if `QhorusProcessor` doesn't register them, ledger migrations are absent in native builds.

---

## Gap 7 — PLATFORM.md Cross-Repo Dependency Map

Per PP-20260523-605b90. Add to `casehub-parent/docs/PLATFORM.md` Cross-Repo Dependency Map:

| `casehub-connectors-core` | `casehub-qhorus` | `connector-backend` | optional — `InboundMessage` CDI events → `ConnectorChannelBackend` → Qhorus channel routing; activates by classpath presence |

Extend the build-order comment for `casehub-qhorus` to note `connector-backend` alongside the existing `connectors` optional module.

---

## Gap 8 — Protocol update: bridge module placement exception

Update `cross-foundation-bridge-module-placement.md` (PP-20260528-6b1d80). Add after the existing rule paragraph:

> **Exception — runtime coupling:** If the bridge requires the consuming repo's runtime beans (not just its api module), placing it in the event-source repo creates a circular dependency. In this case the bridge lives in the consumer's repo. `casehub-qhorus/connector-backend` is the canonical example: it depends on `ChannelGateway`, `ChannelService`, and `ChannelBindingStore` from qhorus runtime; moving it to `casehub-connectors` would require qhorus runtime as a dep of connectors, which already depends on connectors.

---

## Gap 9 — `create_channel` MCP tool scope note

`create_channel` does not expose connector binding fields (4 optional params: `inboundConnectorId`, `externalKey`, `outboundConnectorId`, `outboundDestination`). MCP clients cannot create connector-bound channels. This is intentional — that scope is **deferred to casehubio/qhorus#217** ("MCP tools for creating and updating channels with connector bindings"). No action in this issue.

---

## Testing Summary

| Test class | Type | Status / Change |
|---|---|---|
| `ConnectorKeyStrategyTest` | Pure unit | Complete ✅ |
| `OutboundTitleTest` | Pure unit | Complete ✅ |
| `ConnectorChannelBackendTest` | Unit (Mockito) | Complete ✅ |
| `InMemoryChannelBindingStoreTest` | Contract runner | Complete ✅ |
| `ChannelBindingStoreContractTest` | Abstract base | Add 4 `findAll` test cases |
| `ConnectorChannelBackendIntegrationTest` | `@QuarkusTest` | Remove `InboundConnectorService`; call backend directly; remove sleeps and wrong timeouts; remove redundant routing block in fanOut test |
| CDI async wiring | — | Untested — tracked as #221 |

---

## Out of Scope (separate issues)

- `#217` — MCP tools for connector-bound channel create/update
- `#218` — consolidate `ChannelService.create()` overloads; old overloads remain
- `#215` — fire `ChannelInitialisedEvent` on binding update; disabled test is regression harness
- `#214`, `#216` — v2 connector features
- `#221` — CDI async wiring test coverage for `@ObservesAsync InboundMessage`
