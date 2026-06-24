# ChannelService.create() Consolidation — Design Spec

**Issue:** casehubio/qhorus#218
**Date:** 2026-06-24

---

## Problem

`ChannelService` has 5 overloads of `create()` stepping from 4 to 14 positional parameters. The overloads exist because `ChannelCreateRequest` — a record with 14 fields — is painful to construct. The same pattern is duplicated in `ReactiveChannelService` and `QhorusMcpTools`. This is an escalating anti-pattern: every new field added to channel creation requires extending the overload chain in three classes.

## Design

### 1. Builder on ChannelCreateRequest

Add a `Builder` static inner class to `ChannelCreateRequest`.

**Required field:** `name` (passed to factory method).
**Defaulted field:** `semantic` defaults to `ChannelSemantic.APPEND`.
**All other fields:** default to `null`.

```java
public record ChannelCreateRequest(...) {

    public static Builder builder(String name) {
        return new Builder(name);
    }

    public static final class Builder {
        private final String name;
        private String description;
        private ChannelSemantic semantic = ChannelSemantic.APPEND;
        private String barrierContributors;
        private String allowedWriters;
        private String adminInstances;
        private Integer rateLimitPerChannel;
        private Integer rateLimitPerInstance;
        private Set<MessageType> allowedTypes;
        private Set<MessageType> deniedTypes;
        private String inboundConnectorId;
        private String externalKey;
        private String outboundConnectorId;
        private String outboundDestination;

        private Builder(String name) { this.name = name; }

        public Builder description(String d) { this.description = d; return this; }
        public Builder semantic(ChannelSemantic s) { this.semantic = s; return this; }
        public Builder barrierContributors(String b) { this.barrierContributors = b; return this; }
        public Builder allowedWriters(String w) { this.allowedWriters = w; return this; }
        public Builder adminInstances(String a) { this.adminInstances = a; return this; }
        public Builder rateLimitPerChannel(Integer r) { this.rateLimitPerChannel = r; return this; }
        public Builder rateLimitPerInstance(Integer r) { this.rateLimitPerInstance = r; return this; }
        public Builder allowedTypes(Set<MessageType> t) { this.allowedTypes = t; return this; }
        public Builder deniedTypes(Set<MessageType> t) { this.deniedTypes = t; return this; }
        public Builder connectorBinding(String inboundId, String externalKey,
                String outboundId, String outboundDest) {
            this.inboundConnectorId = inboundId;
            this.externalKey = externalKey;
            this.outboundConnectorId = outboundId;
            this.outboundDestination = outboundDest;
            return this;
        }

        public ChannelCreateRequest build() {
            return new ChannelCreateRequest(name, description, semantic,
                    barrierContributors, allowedWriters, adminInstances,
                    rateLimitPerChannel, rateLimitPerInstance,
                    allowedTypes, deniedTypes,
                    inboundConnectorId, externalKey,
                    outboundConnectorId, outboundDestination);
        }
    }
}
```

Validation (slug format, connector binding completeness, type overlap) stays in the record's compact constructor — `build()` delegates to it. No duplication.

### 2. Delete ChannelService convenience overloads

Delete from `ChannelService`:
- `create(String, String, ChannelSemantic, String)` — 4-param
- `create(String, String, ChannelSemantic, String, String)` — 5-param
- `create(String, String, ChannelSemantic, String, String, String)` — 6-param
- `create(String, String, ChannelSemantic, String, String, String, Integer, Integer)` — 8-param

`create(ChannelCreateRequest)` is the sole entry point.

### 3. Delete ReactiveChannelService convenience overloads

Same 4 overloads deleted. `create(ChannelCreateRequest)` returning `Uni<Channel>` is the sole entry point.

### 4. Delete QhorusMcpTools convenience overloads

Delete from `QhorusMcpTools`:
- `createChannel(String, String, String, String)` — 4-param pkg-private
- `createChannel(String, String, String, String, String)` — 5-param pkg-private
- `createChannel(String, String, String, String, String, String)` — 6-param pkg-private
- `createChannel(String, String, String, String, String, String, Integer, Integer)` — 8-param pkg-private

The 14-param `@Tool` method stays — it is the MCP interface.

### 5. Delete ChannelCreateRequest.simple()

Replaced by `builder("name").build()` which is equivalent and more capable.

## Call site migration

| Category | Count | Migration |
|----------|-------|-----------|
| `channelService.create(4-param)` | ~76 | `channelService.create(ChannelCreateRequest.builder("name").description("desc").build())` |
| `channelService.create(6-param)` | 1 | Builder with explicit fields |
| `channelService.create(ChannelCreateRequest)` with `new` | ~26 | Replace `new ChannelCreateRequest(...)` with builder |
| `tools.createChannel(4-param)` | 9 | Replace with 14-param @Tool call |
| `tools.createChannel(14-param)` | ~444 | No change — @Tool interface |
| `ChannelCreateRequest.simple()` | 5 | `builder("name").build()` |

## Cross-repo impact

**Claudony:** 3 call sites use `channelService.create(ChannelCreateRequest)` with `new ChannelCreateRequest(...)`. The method signature doesn't change. The canonical record constructor still exists (records require it). These sites should be migrated to the builder for consistency — a separate claudony commit, not gated on this work.

**Connectors, Drafthouse:** No direct `ChannelService.create()` calls.

## Not in scope

- 14-param `@Tool` method signature (MCP framework constraint)
- `findOrCreateWithBinding(ChannelCreateRequest)` (already canonical)
- Reactive `@Tool` method (stays as-is)
