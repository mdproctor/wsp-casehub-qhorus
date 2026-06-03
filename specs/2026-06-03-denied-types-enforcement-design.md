# Denied Types Enforcement — Design Spec
**Date:** 2026-06-03  
**Issue:** casehubio/qhorus#243  
**Context:** claudony#142 — oversight channel requires EVENT denial, not allowedTypes restriction

---

## Background

V16 migration and the supporting data layer shipped in #234:
- `channel.denied_types` column (V16 migration, oversight channel fix)
- `Channel.deniedTypes` entity field
- `ChannelDetail.deniedTypes` record component
- `MessageTypeViolationException.denied()` factory

Items 5–8 of the Claudony requirement remain: enforcement in the dispatch gate, validation at channel creation, service API, and MCP tool surface. This spec covers those four items.

The motivating pain (GE-20260519-28967d): oversight channels restricted with `allowedTypes=QUERY,COMMAND` block DONE/DECLINE from the human — wrong direction. `deniedTypes=EVENT` is the correct model: allow everything except telemetry.

---

## Design Decisions

### D1 — Overlap validation in `ChannelCreateRequest` compact constructor (not `StoredMessageTypePolicy`)

Claudony's spec suggested `StoredMessageTypePolicy.validateNoOverlap()` static, called at MCP layer. That is the wrong design for two reasons:

1. **Package cycle**: `runtime/message/` already imports `runtime/channel/Channel`. Adding the reverse dependency creates a cycle.
2. **Escape hatches**: A static on `StoredMessageTypePolicy` called only from MCP tools leaves `ChannelService.create()`, `AutoChannelPolicy`, and tests unprotected.

`ChannelCreateRequest` compact constructor is the correct enforcement point: it makes invalid state unrepresentable regardless of which code path creates the request.

### D2 — `MessageType.parseTypes(String csv)` added to the enum

Inline CSV-to-Set parsing is duplicated across `StoredMessageTypePolicy` and would need to be added to `ChannelCreateRequest`. Extracting to a `public static` on the `MessageType` enum centralises parsing, eliminates duplication, and avoids any new utility class. The `api/` module is the right home — it's already where all callers depend.

### D3 — Denial-first ordering in `StoredMessageTypePolicy.validate()`

"Denial wins when a type appears in both allowedTypes and deniedTypes." Implemented by checking denial before the allow-list. The current early return on `allowedTypes == null` is restructured: denial must run even on open channels (null allowedTypes).

---

## Changes

### 1. `api/src/main/java/.../api/message/MessageType.java`

Add static parser used by both validation sites:

```java
public static Set<MessageType> parseTypes(String csv) {
    if (csv == null || csv.isBlank()) return Set.of();
    return Arrays.stream(csv.split(","))
        .map(String::trim)
        .map(MessageType::valueOf)   // throws IllegalArgumentException on unknown name
        .collect(Collectors.toUnmodifiableSet());
}
```

### 2. `runtime/.../channel/ChannelCreateRequest.java`

Add `String deniedTypes` between `allowedTypes` and `inboundConnectorId`:

```java
public record ChannelCreateRequest(
    String name, String description, ChannelSemantic semantic, String barrierContributors,
    String allowedWriters, String adminInstances,
    Integer rateLimitPerChannel, Integer rateLimitPerInstance,
    String allowedTypes,
    String deniedTypes,           // ← new field
    // Connector binding (all four or all null)
    String inboundConnectorId, String externalKey,
    String outboundConnectorId, String outboundDestination
) { ... }
```

Compact constructor additions (after existing binding validation):

```java
// Validate type names and no overlap
Set<MessageType> allowed = MessageType.parseTypes(allowedTypes);
Set<MessageType> denied  = MessageType.parseTypes(deniedTypes);
// parseTypes already validates names via valueOf — invalid name throws here
Set<MessageType> overlap = new HashSet<>(allowed);
overlap.retainAll(denied);
if (!overlap.isEmpty()) {
    throw new IllegalArgumentException(
        "allowedTypes and deniedTypes must not intersect. Overlap: " + overlap);
}
```

`simple()` factory: update to pass `null` for `deniedTypes`.

### 3. `runtime/.../message/StoredMessageTypePolicy.java`

Restructure `validate()` — denial-first, then allow check:

```java
@Override
public void validate(Channel channel, MessageType type) {
    // Denial wins — checked before the allow-list
    if (channel.deniedTypes != null && !channel.deniedTypes.isBlank()) {
        Set<MessageType> denied = MessageType.parseTypes(channel.deniedTypes);
        if (denied.contains(type)) {
            throw MessageTypeViolationException.denied(channel.name, type, channel.deniedTypes);
        }
    }
    // Open channel (no allowedTypes) passes after denial check
    if (channel.allowedTypes == null || channel.allowedTypes.isBlank()) {
        return;
    }
    Set<MessageType> allowed = MessageType.parseTypes(channel.allowedTypes);
    if (!allowed.contains(type)) {
        throw new MessageTypeViolationException(channel.name, type, channel.allowedTypes);
    }
}
```

### 4. `runtime/.../channel/ChannelService.java`

Add 10-arg overload; existing 9-arg delegates with `null`:

```java
@Transactional
public Channel create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance,
        String allowedTypes, String deniedTypes) {
    return create(new ChannelCreateRequest(
        name, description, semantic, barrierContributors,
        allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance,
        allowedTypes, deniedTypes,
        null, null, null, null));
}
```

`populateChannel(ChannelCreateRequest req)` adds:
```java
channel.deniedTypes = blankToNull(req.deniedTypes());
```

### 5. `runtime/.../channel/ReactiveChannelService.java`

Add 10-arg overload; existing 9-arg delegates with `null`. Inline entity construction adds:
```java
channel.deniedTypes = (deniedTypes == null || deniedTypes.isBlank()) ? null : deniedTypes;
```

### 6. `runtime/.../mcp/QhorusMcpTools.java` — `create_channel`

Add `denied_types` ToolArg after `allowed_types`:

```java
@ToolArg(name = "denied_types",
    description = "Comma-separated MessageType names explicitly denied on this channel. "
        + "Denial wins over allowedTypes if a type appears in both. "
        + "Example: \"EVENT\" for an oversight channel that receives all message types except telemetry.",
    required = false) String deniedTypes,
```

Pass as 10th arg to `new ChannelCreateRequest(...)`. Update `@Tool` description to mention `denied_types`.

### 7. `runtime/.../mcp/ReactiveQhorusMcpTools.java` — `create_channel`

Identical addition. Already constructs `ChannelCreateRequest` — same 10th-arg addition.

---

## Test Plan

### Unit tests

**`MessageTypeTest`** (api module):
- `parseTypes(null)` returns empty set
- `parseTypes("EVENT")` returns `{EVENT}`
- `parseTypes("QUERY,COMMAND")` returns `{QUERY, COMMAND}`
- `parseTypes("INVALID")` throws `IllegalArgumentException`

**`ChannelCreateRequestTest`** (new):
- Valid allowedTypes + valid deniedTypes with no overlap → constructs successfully
- Overlapping allowedTypes + deniedTypes → `IllegalArgumentException`
- Invalid type name in allowedTypes → `IllegalArgumentException`
- Invalid type name in deniedTypes → `IllegalArgumentException`
- `simple()` factory → deniedTypes is null

**`StoredMessageTypePolicyTest`** (existing, extend):
- deniedTypes=EVENT, type=EVENT → `MessageTypeViolationException.denied`
- deniedTypes=EVENT, type=COMMAND → passes
- allowedTypes=QUERY, deniedTypes=null, type=COMMAND → `MessageTypeViolationException` (not-allowed)
- allowedTypes=null, deniedTypes=EVENT, type=EVENT → `MessageTypeViolationException.denied` (open channel still enforces denial)
- allowedTypes=QUERY, deniedTypes=QUERY, type=QUERY → should never reach validate (blocked at construction); paranoia test: denial wins

### Integration tests

**`ChannelServiceTest`** or `@QuarkusTest`:
- `create(... allowedTypes="QUERY,COMMAND" deniedTypes="EVENT")` succeeds; `dispatch(EVENT)` throws denied
- `create(... deniedTypes="EVENT")` succeeds; `dispatch(EVENT)` throws denied; `dispatch(COMMAND)` succeeds
- `create(... allowedTypes="EVENT" deniedTypes="EVENT")` → `IllegalArgumentException` from compact constructor

**`QhorusMcpToolsTest`** (integration, `@QuarkusTest`):
- `create_channel` with `denied_types="EVENT"` → channel has deniedTypes set
- `send_message` with EVENT on that channel → `MessageTypeViolationException`

---

## Out of Scope

- `AutoChannelPolicy` SPI callers: existing callers set `allowedTypes = null, deniedTypes = null` (the 10-arg overload defaults the old 9-arg to null). No migration needed; they will work correctly and can opt in to deniedTypes when ready.
- `update_channel` MCP tool: `deniedTypes` is not yet updateable after creation. Captured as qhorus#244.
- Reactive integration test for deniedTypes enforcement: reactive `@QuarkusTest` requires PostgreSQL DevServices. Deferred; add `@Disabled` with note.
