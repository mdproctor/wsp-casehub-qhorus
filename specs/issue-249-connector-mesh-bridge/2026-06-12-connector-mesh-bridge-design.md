# ConnectorMeshBridge — Qhorus Implementation Design

**Issue:** casehubio/qhorus#249  
**Date:** 2026-06-12  
**Branch:** issue-249-connector-mesh-bridge

---

## Problem

`casehub-connectors-mcp` MCP tools (send_slack, send_teams, etc.) call `ConnectorMeshBridge.notifyDelivered()` after each successful delivery. The default `NoOpConnectorMeshBridge` discards these notifications. Without the bridge, the mesh has no visibility into what agents communicated externally. `notifyDelivered` is only called on the success path — failed deliveries never reach the bridge.

## Key Design Decisions

### Message type: STATUS, not EVENT

The issue spec said EVENT. That is wrong for two reasons:

1. **EVENT is content-free** — `MessageDispatch.Builder.build()` rejects EVENT+content (PP-20260608-054090). A content-free notification carries no useful information about what was delivered.

2. **The informatory role — not the type — defines an observation.** A message is informatory when it carries information without opening an expectation of reply in that channel. STATUS serves this role when posted as a standalone observation with no correlating COMMAND and no expected reply. EVENT is designed for content-free signals. STATUS is the correct type for content-bearing informatory messages.

This is a principled choice, not a workaround. After `casehub-ledger#126` lands, EVENT may gain application-tier content support — but STATUS remains semantically valid regardless.

**ACL and rate-limit implication:** `MessageService.dispatch()` bypasses the writer ACL check and rate-limit check for EVENT but not STATUS (see `MessageService.java` lines 149–165). Our STATUS dispatch is subject to both. See the "Channel constraints" section.

### Observe channel allows STATUS

The normative layout example (`allowed_types="EVENT"`) overstated the restriction. PLATFORM.md's canonical definition includes EVENT and STATUS as the primary observe channel types. Both are informatory. The example is corrected as part of this issue.

### Destination is excluded from content — security

The `destination` parameter carries either a credential (Slack/Teams webhook URL) or PII (phone number, email address). Writing it to the immutable Merkle-chained ledger is a security and privacy violation — even a hash leaves timing signals. **Destination is excluded entirely from content.** The `connectorId` is sufficient for categorisation and ledger queries.

Content format: `"Delivered via %s: %s".formatted(connectorId, content != null ? content : "")`

### Sender encodes connector type for ledger queryability

Sender: `"system:connector:" + connectorId` (e.g. `"system:connector:slack"`, `"system:connector:teams"`). Colon is valid in sender strings (established by `system:watchdog`). This makes `list_ledger_entries(sender=system:connector:slack)` useful without mixing all connector deliveries into one unqueryable sender.

### Config-driven target channel, no auto-creation

`casehub.qhorus.connector-backend.delivery-channel` (blank = no-op). Consumer explicitly opts in by naming a channel. The bridge looks up the channel — if not found, warns and no-ops. No auto-creation: channel creation has side effects (DB write, gateway registration) that belong to the consumer, not the bridge.

### Channel ID cache keyed by tenancyId

`findByName` is a DB query. The channel UUID is stable for the lifetime of the process. Cache in `ConcurrentHashMap<String, UUID>` keyed by tenancyId — Tenant A and Tenant B each look up the same configured channel name but get their own tenant-scoped UUID. On cache hit, no DB query. On miss (first call per tenant), one indexed query; result cached permanently.

`ConcurrentHashMap.computeIfAbsent` does not store null. If `findByName` returns empty, nothing is cached and every subsequent call re-queries until the channel exists. This gives correct behavior for both cases: a missing channel retries on every call; a found channel is stable.

Stale-cache behavior: if the channel is deleted and recreated, the cached UUID becomes invalid. Dispatch fails with a warning. Cache is not invalidated automatically — requires process restart. Acceptable; channel deletion is an intentional operator action.

### Synchronous context capture, async dispatch

`currentPrincipal.tenancyId()` is called synchronously on the HTTP request thread. `ChannelService.findByName()` (for cache misses) is also called synchronously — one indexed DB read, not network I/O. Both values are captured before the thread boundary. The `MessageService.dispatch()` call is handed off to a `ManagedExecutor` with the captured `tenancyId` and `channelId` as explicit values — no CDI proxies cross the thread boundary.

`QhorusInboundCurrentPrincipal.tenancyId()` already catches `ContextNotActiveException` internally and returns `DEFAULT_TENANT_ID`. The bridge does not need a defensive catch around `currentPrincipal.tenancyId()`.

`tenancyId` is set explicitly on `MessageDispatch` — `MessageService` uses it directly without consulting `CurrentPrincipal` (established pattern: `WatchdogEvaluationService`).

### SPI "must never throw" — full method wrap

The SPI contract: "Must never throw — exceptions propagate to the MCP tool caller." The entire `notifyDelivered` body is wrapped in try-catch. A DB exception during cache-miss lookup must not surface as a delivery failure to the MCP tool caller.

---

## Channel constraints

### ACL-restricted channels

If the delivery channel has `allowedWriters` set, the bridge dispatches with `ActorType.SYSTEM`. `MessageService` builds the synthetic role tag as `"role:system"` (lowercase: `dispatch.actorType().name().toLowerCase()`). To permit the bridge:
- Add `"role:system"` to `allowedWriters` — covers all connector types in one entry
- Or add the exact sender ID (e.g. `"system:connector:slack"`) — per-connector granularity

An open channel (null or blank `allowedWriters`) requires no configuration — all writers are permitted.

### Rate-limited channels

If the delivery channel has `rateLimitPerChannel` or `rateLimitPerInstance` set, STATUS dispatches are subject to rate limiting (the bypass applies only to EVENT). High-frequency connector deliveries may be throttled. Use an unconstrained APPEND channel as the delivery channel if throttling is a concern.

### Do not use a connector-backed channel as the delivery channel

If the delivery channel has `ConnectorChannelBackend` registered, every STATUS notification dispatched by the bridge is delivered externally via the connector's outbound destination. For example, if the outbound destination is a Slack webhook, the message "Delivered via slack: Hello" appears in Slack alongside the original delivery. `ConnectorChannelBackend.post()` calls `connectorService.send()` directly — the bridge is not re-triggered (no loop). The cascade is exactly one extra external message per notification, not unbounded. But it is semantic noise for external recipients and almost certainly unintended. Use a dedicated APPEND channel (e.g. `"connector-audit"`) with no backend registration as the delivery channel.

---

## Implementation

### New class: `ConnectorQhorusMeshBridge`

**Location:** `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorQhorusMeshBridge.java`

```java
@ApplicationScoped
public class ConnectorQhorusMeshBridge implements ConnectorMeshBridge {

    // Package-private: allows @ConfigProperty injection at runtime and direct
    // field assignment in CDI-free unit tests (@InjectMocks does not process
    // @ConfigProperty, so the test sets bridge.deliveryChannelName directly).
    @ConfigProperty(name = "casehub.qhorus.connector-backend.delivery-channel",
                    defaultValue = "")
    String deliveryChannelName;

    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject CurrentPrincipal currentPrincipal;
    @Inject ManagedExecutor executor;

    // Keyed by tenancyId — each tenant has its own channel UUID for the same name
    private final ConcurrentHashMap<String, UUID> channelIdCache = new ConcurrentHashMap<>();

    @Override
    public void notifyDelivered(String connectorId, String destination, String content) {
        try {
            if (deliveryChannelName.isBlank()) return;

            String tenancyId = currentPrincipal.tenancyId(); // never throws — absorbed internally
            UUID channelId = channelIdCache.computeIfAbsent(tenancyId, tid ->
                    channelService.findByName(deliveryChannelName)
                            .map(ch -> ch.id)
                            .orElse(null));

            if (channelId == null) {
                LOG.warnf("delivery-channel '%s' not found for tenancy '%s' — no-op",
                        deliveryChannelName, tenancyId);
                return;
            }

            String text = "Delivered via %s: %s"
                    .formatted(connectorId, content != null ? content : "");
            String sender = "system:connector:" + connectorId;

            executor.execute(() -> {
                try {
                    messageService.dispatch(MessageDispatch.builder()
                            .channelId(channelId)
                            .sender(sender)
                            .type(MessageType.STATUS)
                            .content(text)
                            .actorType(ActorType.SYSTEM)
                            .tenancyId(tenancyId)
                            .build());
                } catch (Exception e) {
                    LOG.warnf(e, "ConnectorMeshBridge dispatch failed for channel '%s'",
                            deliveryChannelName);
                }
            });
        } catch (Exception e) {
            LOG.warnf(e, "ConnectorMeshBridge setup failed — delivery still succeeded");
        }
    }

    /** Package-private test helper — clears the channel ID cache between test methods. */
    void clearCache() {
        channelIdCache.clear();
    }
}
```

Displaces `NoOpConnectorMeshBridge` (`@DefaultBean`) by classpath presence — no `@Alternative` or `@Priority` needed.

### `connector-backend/pom.xml` addition

```xml
<!-- ManagedExecutor compile dependency — runtime supplied by casehub-qhorus
     extension augmentation (quarkus-smallrye-context-propagation-deployment
     is declared in casehub-qhorus-deployment/pom.xml). -->
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-smallrye-context-propagation</artifactId>
  <scope>provided</scope>
</dependency>
```

---

## Documentation changes

### `casehubio/connectors` — `ConnectorMeshBridge.java` full javadoc rewrite

The existing javadoc is wrong on three counts: message type (EVENT), target (case observe channel), and SPI contract framing (case session). Full replacement:

```java
/**
 * SPI — notifies the active mesh implementation that a connector delivery has been
 * dispatched via an MCP tool call.
 *
 * <p>The default implementation ({@link NoOpConnectorMeshBridge}) does nothing.
 * When {@code qhorus/connector-backend} is on the classpath, its implementation
 * activates by classpath presence and posts a {@code STATUS} message to the channel
 * configured via {@code casehub.qhorus.connector-backend.delivery-channel}.
 * If no channel is configured, the implementation is a no-op (casehubio/qhorus#249).
 *
 * <h2>Contract for implementations</h2>
 * <ul>
 * <li>Must return quickly — no blocking network I/O on the calling thread.</li>
 * <li>Must tolerate missing or misconfigured delivery channel without throwing.</li>
 * <li>Must never throw — exceptions propagate to the MCP tool caller.</li>
 * </ul>
 */
```

Update `NoOpConnectorMeshBridge` comment accordingly. File as a `casehubio/connectors` issue before or alongside this implementation.

### `casehubio/parent` — PLATFORM.md

**Channel taxonomy table — observe row:**

> `observe` | Informatory messages — carry information without opening an expectation of reply. Primary types: EVENT (content-free signal), STATUS (content-bearing). | EVENT, STATUS

**ConnectorMeshBridge capability entry:** change "posts an EVENT to the active observe channel for the current case session" → "posts a STATUS to the configured delivery channel."

### `casehubio/qhorus` — `docs/agent-mesh-framework.md`

Normative layout example:
```
# Before
create_channel("case-abc/observe", "Telemetry", "APPEND", allowed_types="EVENT")

# After
create_channel("case-abc/observe", "Telemetry", "APPEND", allowed_types="EVENT,STATUS")
```

Add note: *STATUS is permitted on observe channels for content-bearing informatory messages — state reports that carry no expectation of reply. EVENT remains the type for content-free signals.*

### `casehubio/qhorus` — PP-20260608-054090

Reframe from "EVENT must not carry content" to:

> The informatory role — not the message type — defines whether a message belongs on an observe channel. A message is informatory when it carries information without opening an expectation of reply in that channel. EVENT is designed exclusively for this role (content-free signal, no deontic character). STATUS serves an informatory role when posted as a standalone observation with no correlating COMMAND and no expected reply — use it for content-bearing informatory messages. The type declares intent; the role is determined by use and context.

---

## Testing

### `ConnectorQhorusMeshBridgeTest` (CDI-free unit test)

**Setup:** `@InjectMocks ConnectorQhorusMeshBridge bridge` creates the instance with Mockito-injected mocks. `deliveryChannelName` is package-private and NOT populated by `@InjectMocks` (CDI annotations are ignored). Set it directly in `@BeforeEach`: `bridge.deliveryChannelName = "connector-audit"`. Tests that verify the blank-channel no-op path set it to `""` explicitly.

**Executor mock:** `doAnswer(inv -> { inv.getArgument(0, Runnable.class).run(); return null; }).when(executor).execute(any())` — runs async task synchronously. No latch or sleep needed.

Covers:
- Blank `deliveryChannelName` → immediate return, no interactions
- Channel not found (first call, cache miss) → `findByName` called, WARN logged, no dispatch
- Channel found, second call same tenancy → cache hit, `findByName` NOT called again, dispatch proceeds
- Channel found, different tenancyId → cache miss for new tenant, `findByName` called, dispatch proceeds with correct tenancyId
- Happy path → `messageService.dispatch()` called with STATUS, sender `"system:connector:slack"`, ActorType.SYSTEM, tenancyId explicitly set, content `"Delivered via slack: <body>"`
- `content = null` → dispatch called with empty string, no NPE
- `destination` NOT present in dispatched content
- Exception in `findByName` (first call) → swallowed by outer catch, WARN logged, no dispatch
- Exception in async dispatch → swallowed by inner catch, WARN logged

### `ConnectorMeshBridgeIntegrationTest` (`@QuarkusTest`)

Pattern consistent with `ConnectorChannelBackendIntegrationTest`: `@InjectMock MessageService`, `@InjectMock CurrentPrincipal`, `@InjectMock ManagedExecutor`.

**Setup:** `@Inject ConnectorQhorusMeshBridge bridge` — Quarkus injects `deliveryChannelName` via `@ConfigProperty` from test `application.properties`. Set `casehub.qhorus.connector-backend.delivery-channel=connector-audit` in test config. Call `bridge.clearCache()` in `@BeforeEach` to prevent channelId cache bleed between test methods — the `@ApplicationScoped` bean is shared across the entire `@QuarkusTest` class lifecycle.

**Executor mock** (same as unit test): `doAnswer(inv -> { inv.getArgument(0, Runnable.class).run(); return null; }).when(executor).execute(any())`. Eliminates the async race — task runs synchronously on the test thread before `verify()` is called.

Covers:
- Channel pre-created, `delivery-channel` configured, `currentPrincipal.tenancyId()` returns DEFAULT_TENANT_ID → `messageService.dispatch()` called with STATUS, correct sender, correct tenancyId
- Channel not pre-created → `messageService.dispatch()` never called, no exception
- `delivery-channel` blank → `messageService.dispatch()` never called

The multi-tenant cache scenario (different tenancyIds → different cache entries → separate `findByName` calls) is covered in the unit test where `CurrentPrincipal` and `ChannelService` are directly mockable with sequential stubbing.

---

## Configuration

| Property | Default | Description |
|----------|---------|-------------|
| `casehub.qhorus.connector-backend.delivery-channel` | `` (blank) | Channel name to post STATUS delivery notifications. Blank disables the bridge. In multi-tenant deployments, each tenant must independently create a channel with this name — the lookup is tenant-scoped. Tenant A and Tenant B each look up the channel in their own namespace. |

---

## Out of scope

- Destination logging (credential/PII risk — excluded permanently from this feature)
- Auto-channel creation
- Per-case channel routing (requires `CaseChannelResolver` SPI, future issue)
- `notifyFailed()` SPI extension (call sites only invoke bridge on success)
- `allowedTypes` advisory enforcement (tracked separately as qhorus#271)
- Reactive (`@IfBuildProperty`) variant — phase 2 if needed
