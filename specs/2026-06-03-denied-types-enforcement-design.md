# Denied Types Enforcement — Design Spec
**Date:** 2026-06-03 (rev 3 — post-review 2)
**Issue:** casehubio/qhorus#243
**Context:** claudony#142 — oversight channel requires EVENT denial, not allowedTypes restriction

---

## Background

V16 migration and the supporting data layer shipped in #234:
- `channel.denied_types` column (V16 migration, oversight channel fix)
- `Channel.deniedTypes` entity field
- `ChannelDetail.deniedTypes` record component (QhorusEntityMapper.toChannelDetail() already passes ch.deniedTypes)
- `MessageTypeViolationException.denied()` factory

Items 5–8 of the Claudony requirement remain: enforcement in the dispatch gate, validation at channel creation, service API, and MCP tool surface. This spec covers those four items.

The motivating pain (GE-20260519-28967d): oversight channels restricted with `allowedTypes=QUERY,COMMAND` block DONE/DECLINE from the human — wrong direction. `deniedTypes=EVENT` is the correct model: allow everything except telemetry.

---

## Design Decisions

### D1 — Overlap validation in `ChannelCreateRequest` compact constructor

The correct enforcement point for channel type constraint invariants is the `ChannelCreateRequest` compact constructor. Any construction of an invalid request throws immediately — covers all callers uniformly with no escape hatch.

**Why not `StoredMessageTypePolicy.validateNoOverlap()` static (Claudony's spec):**
1. Package cycle: `runtime/message/` already imports `runtime/channel/Channel`. The reverse dependency would create a cycle.
2. Coverage gap: a static called only from MCP tools leaves `ChannelService.create()`, `ConnectorChannelBackend.tryAutoCreate()`, and tests unprotected.

**Critical: this guarantee requires that ALL creation paths go through ChannelCreateRequest.** Review found two violations in the original spec — both fixed below:
- `ReactiveChannelService` was specced to do inline entity construction, never touching ChannelCreateRequest.
- `ReactiveQhorusMcpTools` constructs a ChannelCreateRequest for routing then destructures it back to named params when calling ReactiveChannelService — bypassing the validated request entirely.

Both are fixed by giving ReactiveChannelService a `create(ChannelCreateRequest)` primary method and a `populateChannel(ChannelCreateRequest)` private helper (mirroring ChannelService), and having ReactiveQhorusMcpTools pass the already-constructed req directly.

### D2 — `MessageType.parseTypes(String csv)` added to the enum

Inline CSV-to-Set parsing is duplicated wherever type strings are validated. Extracting to `public static Set<MessageType> parseTypes(String csv)` on the `MessageType` enum centralises parsing, validates names at the point of conversion (via `valueOf`), and avoids any new utility class. The `api/` module is the right home — already depended on by all callers.

### D3 — Denial-first ordering in `StoredMessageTypePolicy.validate()`

"Denial wins when a type appears in both allowedTypes and deniedTypes." Implemented by running the denial check before the allow-list. The current early return on `allowedTypes == null` is restructured: denial must run even on open channels.

### D4 — `AutoChannelSpec.deniedTypes` added

`AutoChannelSpec` currently has `allowedTypes` but not `deniedTypes` — asymmetric and incomplete. `ConnectorChannelBackend.tryAutoCreate()` constructs a `ChannelCreateRequest` from an `AutoChannelSpec`; without `deniedTypes` on the spec, connector-initiated auto-creation can never specify denial. Adding it now is one field and one construction-site update; leaving it is a future breaking change to the `AutoChannelPolicy` SPI and its implementors. Since both `ChannelCreateRequest` and `AutoChannelSpec` are being touched anyway, this is the right time.

### D5 — ReactiveChannelService structural parity with ChannelService

`ChannelService` uses a private `populateChannel(ChannelCreateRequest)` helper shared by all creation paths. `ReactiveChannelService` duplicates entity construction inline in every overload — any future field addition to `Channel` must be manually tracked in two places with no compiler enforcement. The `deniedTypes` change exposes this gap and is the natural forcing function to fix it. Adding `populateChannel(ChannelCreateRequest)` to `ReactiveChannelService` and routing all overloads through it makes this the last time the duplication requires manual coordination.

---

## Construction Sites — All Locations

Adding `deniedTypes` (field position 10) to `ChannelCreateRequest` is a compile-breaking change at all existing construction sites. After Change 2, compile immediately to surface them all. Known sites:

| File | Type | Fix |
|------|------|-----|
| `runtime/.../mcp/QhorusMcpTools.java:254` | production | Change 6 (add `deniedTypes` arg) |
| `runtime/.../mcp/ReactiveQhorusMcpTools.java:217` | production | Change 7 (add `deniedTypes` arg; pass req directly to service) |
| `runtime/.../channel/ChannelCreateRequest.java:43` | factory | Change 2 (show below) |
| `connector-backend/.../ConnectorChannelBackend.java:145` | production | Change 8 (use `spec.deniedTypes()`) |
| `runtime/test/.../channel/ChannelServiceFindOrCreateTest.java:33` | test | add `null` at position 10 |
| `runtime/test/.../channel/ChannelBindingUpdateTest.java:31` | test | add `null` at position 10 |
| `connector-backend/test/.../ConnectorChannelBackendIntegrationTest.java:60` | test | add `null` at position 10 |
| `connector-backend/test/.../ConnectorChannelBackendIntegrationTest.java:139` | test | add `null` at position 10 |

---

## Changes

### 1. `api/src/main/java/.../api/message/MessageType.java`

Add to the enum body:

```java
/**
 * Parses a comma-separated list of MessageType names.
 * Returns an empty set for null or blank input.
 * Throws {@link IllegalArgumentException} if any name is not a valid MessageType.
 */
public static Set<MessageType> parseTypes(String csv) {
    if (csv == null || csv.isBlank()) return Set.of();
    return Arrays.stream(csv.split(","))
            .map(String::trim)
            .map(MessageType::valueOf)
            .collect(Collectors.toUnmodifiableSet());
}
```

Add imports to the enum file: `java.util.Arrays`, `java.util.Set`, `java.util.stream.Collectors`.

### 2. `runtime/.../channel/ChannelCreateRequest.java`

Add `String deniedTypes` between `allowedTypes` and `inboundConnectorId`:

```java
public record ChannelCreateRequest(
        String name,
        String description,
        ChannelSemantic semantic,
        String barrierContributors,
        String allowedWriters,
        String adminInstances,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        String allowedTypes,
        String deniedTypes,                                          // ← new
        // Connector binding — all four non-null together, or all null
        String inboundConnectorId,
        String externalKey,
        String outboundConnectorId,
        String outboundDestination
) { ... }
```

Compact constructor — add after the existing binding validation block:

```java
// Validate type names are valid and allowedTypes ∩ deniedTypes = ∅
Set<MessageType> allowed = MessageType.parseTypes(allowedTypes);   // throws on invalid name
Set<MessageType> denied  = MessageType.parseTypes(deniedTypes);    // throws on invalid name
if (!allowed.isEmpty() && !denied.isEmpty()) {
    Set<MessageType> overlap = new HashSet<>(allowed);
    overlap.retainAll(denied);
    if (!overlap.isEmpty()) {
        throw new IllegalArgumentException(
                "allowedTypes and deniedTypes must not intersect. Overlap: " + overlap);
    }
}
```

Updated `simple()` factory (explicit, all 14 args):

```java
public static ChannelCreateRequest simple(String name, ChannelSemantic semantic) {
    return new ChannelCreateRequest(name, null, semantic, null,
            null, null, null, null,
            null,   // allowedTypes
            null,   // deniedTypes
            null, null, null, null);
}
```

### 3. `runtime/.../message/StoredMessageTypePolicy.java`

Replace `validate()` entirely:

```java
@Override
public void validate(Channel channel, MessageType type) {
    // Denial-first: denial wins over allowedTypes
    if (channel.deniedTypes != null && !channel.deniedTypes.isBlank()) {
        if (MessageType.parseTypes(channel.deniedTypes).contains(type)) {
            throw MessageTypeViolationException.denied(channel.name, type, channel.deniedTypes);
        }
    }
    // Open channel (no allowedTypes restriction) passes after denial check
    if (channel.allowedTypes == null || channel.allowedTypes.isBlank()) {
        return;
    }
    if (!MessageType.parseTypes(channel.allowedTypes).contains(type)) {
        throw new MessageTypeViolationException(channel.name, type, channel.allowedTypes);
    }
}
```

Remove the inline `Arrays.stream(...)` parsing that was previously duplicated here.

### 4. `runtime/.../channel/ChannelService.java`

Add 10-arg overload; existing 9-arg delegates to it with `null`:

```java
@Transactional
public Channel create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance,
        String allowedTypes) {
    return create(name, description, semantic, barrierContributors,
            allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance,
            allowedTypes, null);  // deniedTypes = null
}

@Transactional
public Channel create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance,
        String allowedTypes, String deniedTypes) {
    return create(new ChannelCreateRequest(
            name, description, semantic, barrierContributors,
            allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance,
            allowedTypes, deniedTypes,
            null, null, null, null));   // no connector binding
}
```

`populateChannel(ChannelCreateRequest req)` — add one line:

```java
channel.deniedTypes = blankToNull(req.deniedTypes());
```

Shorter overloads (4/5/6-arg) are unchanged — they already cascade to 8-arg → 9-arg → new 10-arg → `create(ChannelCreateRequest)`. No edits needed to their bodies.

### 5. `runtime/.../channel/ReactiveChannelService.java`

Add `create(ChannelCreateRequest)` primary method and `populateChannel()` helper (structural parity with ChannelService). Add 10-arg named overload; 9-arg delegates to it with null:

```java
/** Primary creation path — all named-param overloads funnel here via the 10-arg overload. */
public Uni<Channel> create(ChannelCreateRequest req) {
    // populateChannel() is pure (no IO) — entity construction happens outside the transaction.
    // A JPA entity is a transient POJO until persist() is called inside the session;
    // it becomes managed when channelStore.put(channel) runs. Do NOT move this inside the lambda.
    Channel channel = populateChannel(req);
    return Panache.withTransaction("qhorus", () -> channelStore.put(channel));
}

public Uni<Channel> create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance,
        String allowedTypes) {
    return create(name, description, semantic, barrierContributors,
            allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance,
            allowedTypes, null);
}

public Uni<Channel> create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance,
        String allowedTypes, String deniedTypes) {
    // ChannelCreateRequest construction runs validation before entering the transaction
    return create(new ChannelCreateRequest(
            name, description, semantic, barrierContributors,
            allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance,
            allowedTypes, deniedTypes,
            null, null, null, null));
}

private static Channel populateChannel(ChannelCreateRequest req) {
    Channel channel = new Channel();
    channel.name = req.name();
    channel.description = req.description();
    channel.semantic = req.semantic();
    channel.barrierContributors = req.barrierContributors();
    channel.allowedWriters = blankToNull(req.allowedWriters());
    channel.adminInstances = blankToNull(req.adminInstances());
    channel.rateLimitPerChannel = req.rateLimitPerChannel();
    channel.rateLimitPerInstance = req.rateLimitPerInstance();
    channel.allowedTypes = blankToNull(req.allowedTypes());
    channel.deniedTypes = blankToNull(req.deniedTypes());
    return channel;
}
```

Existing shorter overloads (4-arg, 5-arg, 6-arg) already cascade to the 8-arg → now cascade to the 9-arg, which cascades to the 10-arg, which constructs ChannelCreateRequest. No change needed to their bodies. The previously-inline entity construction block inside `Panache.withTransaction(...)` is replaced entirely by `create(ChannelCreateRequest)`.

Add `blankToNull` private static (mirror ChannelService):
```java
private static String blankToNull(String s) {
    return (s == null || s.isBlank()) ? null : s;
}
```

### 6. `runtime/.../mcp/QhorusMcpTools.java` — `create_channel`

Add `denied_types` ToolArg between `allowed_types` and `inbound_connector_id`:

```java
@ToolArg(name = "denied_types",
    description = "Comma-separated MessageType names explicitly denied on this channel. "
        + "Denial wins if a type appears in both allowedTypes and deniedTypes. "
        + "Example: \"EVENT\" for an oversight channel open to all agent messages "
        + "but not telemetry.",
    required = false) String deniedTypes,
```

Pass as 10th positional arg to `new ChannelCreateRequest(...)`:

```java
Channel ch = channelService.create(new ChannelCreateRequest(
        name, description, sem, barrierContributors, allowedWriters, adminInstances,
        rateLimitPerChannel, rateLimitPerInstance, allowedTypes, deniedTypes,
        inboundConnectorId, externalKey, outboundConnectorId, outboundDestination));
```

Update `@Tool` description — replace existing text with:

```java
@Tool(name = "create_channel", description = "Create a named channel with declared semantic. "
        + "Semantic defaults to APPEND if not specified. "
        + "Use allowed_types to restrict which MessageType values may be sent to this channel "
        + "(enforced at both MCP and service layers). "
        + "Use denied_types to explicitly block specific types regardless of allowed_types — "
        + "denial wins if a type appears in both. "
        + "Example: denied_types=\"EVENT\" for an oversight channel open to all agent messages "
        + "but not telemetry. "
        + "Optionally attach a connector binding by supplying all four connector fields together.")
```

### 7. `runtime/.../mcp/ReactiveQhorusMcpTools.java` — `create_channel`

Same `denied_types` ToolArg addition and same `@Tool` description update (identical text). After adding `deniedTypes` to the ChannelCreateRequest construction (position 10), pass `req` directly to the reactive service — no destructuring:

```java
ChannelCreateRequest req = new ChannelCreateRequest(name, description, sem, barrierContributors,
        allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance,
        allowedTypes, deniedTypes,
        inboundConnectorId, externalKey, outboundConnectorId, outboundDestination);
if (req.hasConnectorBinding()) {
    return Uni.createFrom().item(() -> blockingChannelService.create(req))
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
            .invoke(ch -> channelGateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name)))
            .flatMap(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())));
}
return channelService.create(req)   // ← pass req directly, no destructuring
        .invoke(ch -> channelGateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name)))
        .flatMap(ch -> messageStore.countByChannel(ch.id)
                .map(count -> toChannelDetail(ch, count.longValue())));
```

The previous destructuring (`channelService.create(req.name(), req.description(), ..., req.allowedTypes())`) is replaced by `channelService.create(req)`. This requires Change 5's `create(ChannelCreateRequest)` method on ReactiveChannelService.

### 8. `connector-backend/.../ConnectorChannelBackend.java` and `connector-backend/.../AutoChannelSpec.java`

**`AutoChannelSpec`** — add `deniedTypes` after `allowedTypes`:

```java
public record AutoChannelSpec(
        String channelName,
        String description,
        ChannelSemantic semantic,
        String allowedTypes,
        String deniedTypes,           // ← new
        String outboundConnectorId,
        String outboundDestination
) {}
```

**`ConnectorChannelBackend.tryAutoCreate()`** — use `spec.deniedTypes()`:

```java
ChannelCreateRequest req = new ChannelCreateRequest(
        spec.channelName(),
        spec.description(),
        spec.semantic(),
        null, null, null, null, null,
        spec.allowedTypes(),
        spec.deniedTypes(),           // ← new
        msg.connectorId(),
        lookupKey,
        spec.outboundConnectorId(),
        spec.outboundDestination());
```

Adding `deniedTypes` at position 5 breaks all existing `AutoChannelSpec` construction sites at compile time. Known sites (separate from the `ChannelCreateRequest` table):

| File | Type | Fix |
|------|------|-----|
| `connector-backend/.../ConfiguredAutoChannelPolicy.java:71` | production | insert `null` at position 5 (between existing `null`/allowedTypes and `outboundConnectorId`) |
| `connector-backend/test/.../ConnectorAutoChannelBackendTest.java:76` | test | same — insert `null` at position 5 |

`ConfiguredAutoChannelPolicy.java:71` is production code — an implementer who only touches `ConnectorChannelBackend.java` and the two test files will hit a compile failure there unexpectedly. Fix: `new AutoChannelSpec(channelName, description, semantic, null, null, outboundConnectorId, outboundDestination)`. Correct behaviour is unchanged — channels remain open on both dimensions.

---

## Test Plan

### Unit tests

**`MessageTypeTest`** (api module — new or existing):
- `parseTypes(null)` → empty set
- `parseTypes("")` → empty set
- `parseTypes("EVENT")` → `{EVENT}`
- `parseTypes("QUERY,COMMAND")` → `{QUERY, COMMAND}`
- `parseTypes(" EVENT , COMMAND ")` → `{EVENT, COMMAND}` (trims whitespace)
- `parseTypes("INVALID")` → `IllegalArgumentException`
- `parseTypes("EVENT,INVALID")` → `IllegalArgumentException`

**`ChannelCreateRequestTest`** (new unit test, no Quarkus):
- `allowedTypes="QUERY,COMMAND"`, `deniedTypes="EVENT"` — no overlap → constructs
- `allowedTypes="QUERY,COMMAND"`, `deniedTypes="QUERY"` — overlap → `IllegalArgumentException`
- `allowedTypes="INVALID"` — bad name → `IllegalArgumentException`
- `deniedTypes="INVALID"` — bad name → `IllegalArgumentException`
- `allowedTypes=null`, `deniedTypes="EVENT"` — valid → constructs
- `allowedTypes="QUERY"`, `deniedTypes=null` — valid → constructs
- `simple()` factory → `deniedTypes` is null

**`StoredMessageTypePolicyTest`** (existing, extend):
- `deniedTypes=EVENT`, type=EVENT → `MessageTypeViolationException` with denied message
- `deniedTypes=EVENT`, type=COMMAND → passes
- `allowedTypes=QUERY`, `deniedTypes=null`, type=COMMAND → `MessageTypeViolationException` (not-allowed)
- `allowedTypes=null`, `deniedTypes=EVENT`, type=EVENT → denied (open channel still enforces denial)
- `allowedTypes=null`, `deniedTypes=null`, type=EVENT → passes (fully open)
- denial-wins ordering: channel with `allowedTypes=EVENT,QUERY` and `deniedTypes=EVENT` cannot be constructed (compact constructor blocks it); this invariant means the "denial wins" tie-break in validate() is unreachable in practice but is the correct semantic ordering

**`ReactiveChannelServiceTest`** (unit-level, no Quarkus/Panache):
- Direct unit test is difficult due to Panache dependency. Use a `@QuarkusTest` with H2 in blocking mode (not PostgreSQL DevServices).
- `create(name, ..., allowedTypes="QUERY", deniedTypes="QUERY")` → `IllegalArgumentException` before transaction starts (validation in ChannelCreateRequest constructor)
- `create(name, ..., allowedTypes="QUERY,COMMAND", deniedTypes="EVENT")` → succeeds, channel persisted with correct deniedTypes
- Purpose: verify that the reactive path runs ChannelCreateRequest validation — this was the D1 gap.

### Integration tests (`@QuarkusTest`)

**`ChannelServiceTest`** or dedicated integration test:
- `create(... allowedTypes="QUERY,COMMAND", deniedTypes="EVENT")` → channel persisted; `dispatch(EVENT)` throws `MessageTypeViolationException.denied`
- `create(... deniedTypes="EVENT", allowedTypes=null)` → open channel; `dispatch(EVENT)` denied; `dispatch(COMMAND)` succeeds
- `create(... allowedTypes="QUERY", deniedTypes="QUERY")` → `IllegalArgumentException` from compact constructor

**`QhorusMcpToolsTest`** (integration, `@QuarkusTest`):
- `create_channel` with `denied_types="EVENT"` → `ChannelDetail.deniedTypes == "EVENT"`
- `send_message` with EVENT on that channel → `ToolCallException` wrapping `MessageTypeViolationException.denied`
- `create_channel` with `denied_types="INVALID_TYPE"` → `ToolCallException` wrapping `IllegalArgumentException`

---

## Out of Scope

- `update_channel` MCP — `deniedTypes` not yet updatable after creation. Tracked as qhorus#244.
- Reactive integration test for deniedTypes enforcement against PostgreSQL DevServices: deferred. Add `@Disabled("Requires PostgreSQL DevServices — see #243")` with note.
- `NormativeChannelLayout` changes on the Claudony side — owned by Claudony team (claudony#142).
