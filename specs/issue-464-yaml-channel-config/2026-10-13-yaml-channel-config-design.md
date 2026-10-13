# YAML-Declared Channel Configuration

**Issue:** casehubio/qhorus#464
**Date:** 2026-10-13
**Status:** Design

---

## Problem

Channel definitions are currently Java-based — consuming apps call `ChannelCreateRequest.builder("name").semantic(...).build()` and `ChannelService.findOrCreate()`. For YAML-only apps (casehubio/engine#1017), there is no way to declare channels without writing Java.

## Solution

Add `casehub.qhorus.channels.<name>.*` to `QhorusConfig` via SmallRye `@ConfigMapping` with `Map<String, ChannelConfig>`. A new `ChannelConfigBootstrap` bean processes declarations at startup: creates missing channels via `findOrCreate()`, detects drift on existing channels, and logs warnings for divergence.

## YAML Format

```yaml
casehub:
  qhorus:
    channels:
      reviews:
        semantic: APPEND
        description: "Code review discussions"
        allowed-writers:
          - review-agent
          - lead-agent
        allowed-types:
          - QUERY
          - COMMAND
          - RESPONSE
          - DONE
        protocols:
          - REQUEST_RESPONSE
          - TASK_COMPLETION
        enforcement-mode: ADVISORY
        track-delivery: true
        metadata:
          team: platform
      status-feed:
        semantic: BROADCAST
        description: "Agent status broadcasts"
        allowed-types:
          - STATUS
          - EVENT
      support-intake:
        semantic: COLLECT
        description: "Incoming support requests"
        space: "customer-ops"
        reviewer-instances:
          - senior-reviewer
        connector:
          inbound-connector-id: slack
          external-key: C0SUPPORT
          outbound-connector-id: slack
          outbound-destination: "#support-intake"
```

The map key is the channel name (slug-validated by `ChannelCreateRequest`). All fields except `semantic` are optional. `semantic` defaults to `APPEND`.

## Config Interface

```java
// Added to QhorusConfig
Map<String, ChannelConfig> channels();

interface ChannelConfig {
    @WithDefault("APPEND")
    ChannelSemantic semantic();

    Optional<String> description();

    // Access control
    Optional<List<String>> allowedWriters();
    Optional<List<String>> adminInstances();
    Optional<List<String>> barrierContributors();
    Optional<List<String>> reviewerInstances();

    // Type constraints — List in config, converted to Set at bootstrap
    Optional<List<MessageType>> allowedTypes();
    Optional<List<MessageType>> deniedTypes();

    // Governance
    Optional<List<String>> protocols();
    Optional<List<String>> protocolParticipants();
    Optional<EnforcementMode> enforcementMode();
    Optional<List<String>> enforcementExclusions();

    // Rate limiting
    OptionalInt rateLimitPerChannel();
    OptionalInt rateLimitPerInstance();

    // Routing
    Optional<Double> routingTrustThreshold();

    // Delivery
    Optional<Boolean> trackDelivery();

    // Organization
    Optional<String> space();     // resolved by name via SpaceService
    Optional<Map<String, String>> metadata();

    // Connector binding — all-or-nothing group
    Optional<ConnectorConfig> connector();

    interface ConnectorConfig {
        String inboundConnectorId();
        String externalKey();
        String outboundConnectorId();
        String outboundDestination();
    }
}
```

**Design notes:**
- `allowedTypes`/`deniedTypes` use `List<MessageType>` (SmallRye `Set<T>` support is inconsistent) — converted to `Set` at the bootstrap boundary.
- `space` is a name string, resolved to UUID via `SpaceService.findByName()` at bootstrap. If the space doesn't exist, WARN and skip the space assignment (channel still created).
- `connector` is an optional sub-group. When present, all four fields are required (validated by `ChannelCreateRequest`'s compact constructor).
- `metadata` is `Map<String, String>` — straightforward in SmallRye Config YAML.
- `Map<String, ChannelConfig>` with zero entries returns an empty map — bootstrap does nothing.

## Bootstrap Lifecycle

### ChannelConfigBootstrap

```java
@ApplicationScoped
public class ChannelConfigBootstrap {

    private final QhorusConfig config;
    private final ChannelService channelService;
    private final SpaceService spaceService;

    // constructor injection

    void onStartup(@Observes StartupEvent event) {
        var channels = config.channels();
        if (channels.isEmpty()) return;

        for (var entry : channels.entrySet()) {
            try {
                processChannel(entry.getKey(), entry.getValue());
            } catch (Exception e) {
                LOG.warnf("Failed to bootstrap channel '%s': %s",
                          entry.getKey(), e.getMessage());
            }
        }
    }
}
```

Per-channel try-catch follows PP-20260522-c3a8c1 (startup exception isolation).

### Channel Processing Flow

For each declared channel:

1. **Build `ChannelCreateRequest`** from config — map all present fields via the builder API.
2. **Resolve space** — if `space` is set, call `SpaceService.findByName()`. Ambiguous name (multiple matches) → WARN and skip space. Not found → WARN and skip space. Channel is still created without space assignment.
3. **Call `channelService.findOrCreate(request)`** — idempotent. Returns `FindOrCreateResult(channel, wasCreated)`.
4. **If `wasCreated`** → `LOG.infof("Channel '%s' created from config", name)`.
5. **If not created (already existed)** → run drift detection.

### Drift Detection

Compare the declared config against the existing channel's state. For each field where the declared value differs from the actual value, produce a `ChannelDrift` record:

```java
record ChannelDrift(String channelName, String field,
                    String declared, String actual) {}
```

**Fields compared:** semantic, description, allowedWriters, adminInstances, barrierContributors, reviewerInstances, allowedTypes, deniedTypes, protocols, protocolParticipants, enforcementMode, enforcementExclusions, rateLimitPerChannel, rateLimitPerInstance, routingTrustThreshold, trackDelivery, space (by name).

**Fields NOT compared:** connector binding (wiring may change independently), metadata (additive in nature — declared metadata may be a subset of runtime metadata).

**Output:** Each drift logged as WARN:
```
Channel 'reviews': drift detected — semantic declared as COLLECT, actual is APPEND
Channel 'reviews': drift detected — allowedWriters declared as [review-agent, lead-agent], actual is [review-agent]
```

At the end of bootstrap, if any drift was detected:
```
Channel config bootstrap: 5 channels processed, 3 created, 2 existing (1 with drift — 2 fields diverged)
```

**Drift is never auto-corrected.** Operators reconcile manually via the REST/GraphQL API.

## Spring Counterpart

`QhorusConfigProperties` gains:
- `Map<String, ChannelConfigProps> channels` field with getter/setter
- `ChannelConfigProps` inner class implementing `QhorusConfig.ChannelConfig`
- `ConnectorConfigProps` inner class implementing `QhorusConfig.ChannelConfig.ConnectorConfig`

## Testing

### Unit tests (CDI-free)

`ChannelConfigBootstrapTest` — construct with mock `ChannelService` and `SpaceService`:
- Empty config map → no calls
- Single channel created → `findOrCreate()` called with correct request
- Existing channel without drift → no warnings
- Existing channel with drift → drift records produced with correct field/value pairs
- Space resolution: found → spaceId set; not found → WARN, no spaceId; ambiguous → WARN, no spaceId
- Connector binding: all-or-nothing validation (via `ChannelCreateRequest` compact constructor)
- Per-channel exception isolation → remaining channels still processed

### Integration test (@QuarkusTest)

`ChannelConfigBootstrapIntegrationTest` with `@TestProfile` that declares two channels in config overrides. Verify:
- Both channels exist after startup
- `channelService.findByName()` returns them with correct semantic and properties

## What This Does NOT Cover

- **Channel deletion from YAML** — removing a channel from YAML does not delete it. Channels are durable state.
- **Runtime-to-YAML sync** — no mechanism to export current channel state back to YAML.
- **Channel templates** — no inheritance or "extends" between channel definitions.
- **Cross-app channel references** — channel names are global within a qhorus instance; no namespace scoping per app.

## References

- `ChannelCreateRequest.java` — 24-field record with builder, compact constructor validation
- `ChannelService.findOrCreate()` — idempotent create-or-find with binding support
- `QhorusConfig.java` — existing `@ConfigMapping(prefix = "casehub.qhorus")`
- PP-20260522-c3a8c1 — startup exception isolation protocol
- PP-20260522-e5e527 — ChannelInitialisedEvent idempotency protocol
- casehubio/engine#1017 — parent epic: zero-authored-Java deployment
