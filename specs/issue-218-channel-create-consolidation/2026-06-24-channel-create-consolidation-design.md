# ChannelService.create() Consolidation — Design Spec

**Issue:** casehubio/qhorus#218
**Date:** 2026-06-24
**Revised:** 2026-06-24 (spec review round 2)

---

## Problem

`ChannelService` has 5 `create()` methods: 4 convenience overloads stepping from 4 to 8 positional parameters, plus the canonical `create(ChannelCreateRequest)`. The overloads exist because `ChannelCreateRequest` — a record with 14 fields — is painful to construct positionally. The same overload chain is duplicated in `ReactiveChannelService` (5 methods) and `QhorusMcpTools` (4 package-private convenience overloads). This is an escalating anti-pattern: every new field added to channel creation requires extending the chain in three classes.

A secondary duplication exists in `populateChannel(ChannelCreateRequest)` and `blankToNull(String)`, which are identical across `ChannelService` and `ReactiveChannelService`.

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
        public Builder inboundConnectorId(String id) { this.inboundConnectorId = id; return this; }
        public Builder externalKey(String key) { this.externalKey = key; return this; }
        public Builder outboundConnectorId(String id) { this.outboundConnectorId = id; return this; }
        public Builder outboundDestination(String dest) { this.outboundDestination = dest; return this; }

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

Each connector binding field gets its own named setter — no grouped `connectorBinding(String, String, String, String)` method. Four positionally-identical strings would reintroduce the exact ambiguity the builder eliminates. The compact constructor's all-or-nothing validation fires in `build()` regardless.

Validation (slug format, connector binding completeness, type overlap) stays in the record's compact constructor — `build()` delegates to it. No duplication.

**Constructor visibility:** The canonical record constructor remains public. JLS §8.10.4.2 prohibits a public record's canonical constructor from having more restricted access — package-private is not an option without abandoning records entirely, which would sacrifice value semantics, equals/hashCode, immutability, and pattern matching for no architectural gain. The builder is the conventional API. The public constructor serves as a natural enforcement mechanism: when field 15 is added, constructor callers break (compile error forces explicit handling), while builder callers are unaffected (new field defaults to null). The breakage activates precisely when it matters — at field evolution time — making it better than constructor hiding, which would prevent all direct construction including valid same-package uses.

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

The 14-param `@Tool` method stays — it is the MCP interface. Its internal `new ChannelCreateRequest(...)` construction is migrated to use the builder for consistency.

### 5. Delete ChannelCreateRequest.simple()

Replaced by `builder("name").build()` which is equivalent and more capable.

### 6. Extract Channel.fromRequest()

Both `ChannelService` and `ReactiveChannelService` have identical `populateChannel(ChannelCreateRequest)` and `blankToNull(String)` private methods. Extract to a static factory on the entity:

```java
public class Channel extends PanacheEntityBase {

    public static Channel fromRequest(ChannelCreateRequest req, String tenancyId) {
        Channel ch = new Channel();
        ch.name = req.name();
        ch.description = req.description();
        ch.semantic = req.semantic();
        ch.barrierContributors = req.barrierContributors();
        ch.allowedWriters = blankToNull(req.allowedWriters());
        ch.adminInstances = blankToNull(req.adminInstances());
        ch.rateLimitPerChannel = req.rateLimitPerChannel();
        ch.rateLimitPerInstance = req.rateLimitPerInstance();
        ch.allowedTypes = MessageType.serializeTypes(req.allowedTypes());
        ch.deniedTypes = MessageType.serializeTypes(req.deniedTypes());
        ch.tenancyId = tenancyId;
        return ch;
    }

    private static String blankToNull(String s) {
        return (s == null || s.isBlank()) ? null : s;
    }
}
```

Both services become one-liners:
```java
// ChannelService.create()
Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());
channelStore.put(channel);

// ReactiveChannelService.create()
Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());
return Panache.withTransaction("qhorus", () -> channelStore.put(channel));
```

`findOrCreateWithBinding()` also calls `populateChannel(req)` internally — replaced by `Channel.fromRequest()` as part of this extraction. Its method signature is unchanged.

`CurrentPrincipal` is not passed to the entity — tenancyId is resolved in the service and passed as a plain string. The entity has no CDI dependency.

## Call site migration

| Category | Count | Migration |
|----------|-------|-----------|
| `channelService.create(4-param)` | ~76 | `channelService.create(ChannelCreateRequest.builder("name").description("desc").build())` |
| `channelService.create(8-param)` | 2 | Builder — ChannelServiceTest:23 and ReactiveChannelServiceTest:27 contract test helpers |
| `channelService.create(ChannelCreateRequest)` with `new` | ~26 (blocking) | Replace `new ChannelCreateRequest(...)` with builder |
| `reactiveChannelService.create(ChannelCreateRequest)` with `new` | 1 | ReactiveChannelServiceDeniedTypesTest:32 — builder |
| `tools.createChannel(4-param pkg-private)` | 9 | Switch to `channelService.create(builder)` — these tests (CommitmentToolTest, CommitmentLifecycleTest) use channel creation as setup, not as the thing under test |
| `tools.createChannel(14-param @Tool)` | ~444 | No change — MCP interface |
| `ChannelCreateRequest.simple()` | 5 | `builder("name").build()` |
| `QhorusMcpTools.createChannel` @Tool body | 1 | Internal `new ChannelCreateRequest(...)` → builder |
| `ReactiveQhorusMcpTools.createChannel` @Tool body | 1 | Internal `new ChannelCreateRequest(...)` → builder |
| `ConnectorChannelBackend.tryAutoCreate()` | 1 | Production code: `new ChannelCreateRequest(14 args)` → builder (passes to `findOrCreateWithBinding`, not `create`) |

The 5-param and 6-param ChannelService overloads have 0 external callers (internal delegation chain only).

## Cross-repo impact

**Claudony:** 1 production call site constructs `new ChannelCreateRequest(...)` at `ClaudonyReactiveCaseChannelProvider.java:176`. The method signature (`create(ChannelCreateRequest)`) doesn't change. The canonical record constructor still exists (records require it). This site should be migrated to the builder for consistency — a separate claudony commit, not gated on this work. The test file (`ClaudonyReactiveCaseChannelProviderTest`) uses Mockito `argThat` matchers that don't construct instances — no migration needed.

**connector-backend (in-repo):** `ConnectorChannelBackend.tryAutoCreate()` constructs `new ChannelCreateRequest(14 positional args)` and passes to `findOrCreateWithBinding()`. Migrated to builder as part of this work.

**Connectors (casehub-connectors), Drafthouse:** No `ChannelService.create()` or `ChannelCreateRequest` construction calls.

## Not in scope

- 14-param `@Tool` method signature (MCP framework constraint — the @Tool bodies are migrated to use the builder internally, but the parameter list is fixed by the MCP protocol)
- `findOrCreateWithBinding(ChannelCreateRequest)` method signature (unchanged — its internal `populateChannel()` call is replaced by `Channel.fromRequest()` as part of Section 6)
- Reactive `@Tool` method signature (stays as-is — internal body migrated to builder)
