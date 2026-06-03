# Design Spec — `project_channel` MCP Tool

**Date:** 2026-06-03
**Issue:** qhorus#232
**Status:** Approved for implementation (v2 — post review)

---

## Problem

`ProjectionService.project()` is generic over `<S>` and cannot be exposed as an MCP tool directly — MCP tools must return concrete types. Two things are needed: a registry that lets the tool find the right projection by name, and a render step that converts the typed state `<S>` into a `String`.

---

## Design

### 1. `RenderableProjection<S>` SPI — `api/spi/`

A composite interface that extends `ChannelProjection<S>` with a name and a render step:

```java
package io.casehub.qhorus.api.spi;

public interface RenderableProjection<S> extends ChannelProjection<S> {

    /**
     * The name under which this projection is registered. Must be unique across
     * all {@code RenderableProjection} beans in the CDI context; a duplicate
     * detected at startup by {@code ProjectionRegistry} fails fast.
     *
     * <p>Use a stable, meaningful identifier — callers reference this by name
     * from MCP tool arguments.
     */
    String projectionName();

    /**
     * Converts the projection result to a String suitable for return from an
     * MCP tool.
     *
     * <p>Receives the full {@link ProjectionResult} so the renderer can
     * distinguish between an empty channel ({@code result.isEmpty() == true})
     * and a channel whose fold produced {@code identity()} because no messages
     * matched the projection's criteria. These are not equivalent; callers
     * must not treat them the same.
     *
     * <p>Must return a non-null, non-empty String even when
     * {@code result.isEmpty()} — a human-readable "channel is empty" message
     * is appropriate.
     *
     * <p>Must be pure and non-blocking — called on the MCP dispatch thread.
     * Must not throw — unchecked exceptions propagate from {@code project_channel}.
     *
     * @param result the completed fold result, including cursor and empty-channel signal
     * @return a human-readable or structured string representation
     */
    String render(ProjectionResult<S> result);
}
```

**Name over annotation:** `projectionName()` is introspectable without CDI machinery — it follows the existing `ChannelBackend.backendId()` pattern already in the codebase. No `@ProjectionName` qualifier annotation and no `AnnotationLiteral` subclass are needed, keeping `api/` free of CDI annotation dependencies.

**`render(ProjectionResult<S>)` not `render(S)`:** The full `ProjectionResult<S>` is passed rather than just `state` because `state == identity()` is ambiguous — it means either the channel is empty OR the fold produced no output for that projection's criteria (e.g., a COMMAND counter on a channel with only EVENTs). Only `result.isEmpty()` gives the definitive empty-channel signal.

**Multi-format:** A consumer needing markdown and JSON renders registers two `RenderableProjection` beans — `"summary-markdown"` and `"summary-json"` — both delegating their fold to a shared implementation. Adding a `format` parameter to `render()` preemptively is rejected: no current use case, harder to implement, worse naming at the MCP boundary.

**Placement:** `api/spi/` per `consumer-spi-placement` protocol — external consumers will implement `RenderableProjection<S>` and they depend only on the lightweight `api/` module.

**Recommended CDI scope:** `@ApplicationScoped`. Projections must be stateless (fold state is in `ProjectionResult`, not in the bean); `@Dependent` is legal but callers must note the registry holds the bean reference for the application lifetime.

**Update to `ChannelProjection<S>` javadoc:** The existing javadoc refers to a future `@ChannelBound` qualifier for automatic routing by channel name. That remains a separate future registry (auto-route: "this channel always uses this projection"). `RenderableProjection<S>` with `projectionName()` is the explicit-selection registry (tool-call: "project this channel using the summary projection"). The two coexist; update the javadoc to document both.

---

### 2. `ProjectionRegistry` — `runtime/message/`

An `@ApplicationScoped` bean that builds the name→projection map at construction and detects duplicates at startup:

```java
package io.casehub.qhorus.runtime.message;

@ApplicationScoped
public class ProjectionRegistry {

    private final Map<String, RenderableProjection<?>> registry;

    @Inject
    ProjectionRegistry(@Any Instance<RenderableProjection<?>> bundles) {
        Map<String, RenderableProjection<?>> map = new HashMap<>();
        bundles.forEach(b -> {
            String name = b.projectionName();
            if (map.put(name, b) != null) {
                throw new IllegalStateException(
                    "Duplicate RenderableProjection name '" + name + "' — " +
                    "each projection must have a unique projectionName()");
            }
        });
        this.registry = Collections.unmodifiableMap(map);
    }

    public RenderableProjection<?> get(String name) {
        RenderableProjection<?> p = registry.get(name);
        if (p == null) {
            throw new IllegalArgumentException(
                "No RenderableProjection registered with name '" + name + "'");
        }
        return p;
    }

    public Set<String> names() {
        return registry.keySet();
    }
}
```

**Startup collision detection:** `map.put()` returning non-null means two beans share a name — detected at CDI bootstrap, not at first tool call.

**Lifecycle:** The registry holds direct bean references. For `@ApplicationScoped` projections (the recommended scope) this is correct. For `@Dependent` projections, the registry acts as the owning scope — beans live as long as the registry, which is application lifetime. This is acceptable; `@Dependent` projections that require `@PreDestroy` cleanup should not be used with this registry.

---

### 3. `project_channel` MCP tool — `QhorusMcpTools` (tool method) + `QhorusMcpToolsBase` (shared logic)

**New injections in `QhorusMcpTools`:**

```java
@Inject
ProjectionRegistry projectionRegistry;

@Inject
ProjectionService projectionService;
```

**Tool method:**

```java
@Tool(name = "project_channel",
      description = "Project a channel's message history through a named " +
          "RenderableProjection and return the rendered result as a String. " +
          "On LAST_WRITE channels, only the current value per sender is visible — " +
          "the full message history has been overwritten in place. " +
          "Reads proceed on paused channels.")
public String projectChannel(
        @ToolArg(name = "channel",
                 description = "Channel name or UUID") String channel,
        @ToolArg(name = "projection_name",
                 description = "Name matching RenderableProjection.projectionName() " +
                     "(e.g. 'channel-summary')") String projectionName) {

    UUID channelId = resolveChannelId(channel);
    RenderableProjection<?> projection = projectionRegistry.get(projectionName);
    return projectAndRender(channelId, projection);
}
```

**`resolveChannelId` in `QhorusMcpToolsBase`:**

```java
// package-private — shared by QhorusMcpTools and ReactiveQhorusMcpTools
UUID resolveChannelId(String channel) {
    try {
        UUID id = UUID.fromString(channel);
        // Validate existence — UUID parse succeeds for non-existent channels;
        // ProjectionService would silently return render(identity()) without this check.
        channelStore.find(id)
            .orElseThrow(() -> new IllegalArgumentException(
                "Channel not found: " + channel));
        return id;
    } catch (IllegalArgumentException e) {
        if (e.getMessage().startsWith("Channel not found")) throw e;
        // Not a UUID — treat as name
        return channelService.findByName(channel)
            .orElseThrow(() -> new IllegalArgumentException(
                "Channel not found: " + channel))
            .id;
    }
}
```

Note: `channelService` and `channelStore` are not currently in `QhorusMcpToolsBase` — both need to be added as injections there (or `resolveChannelId` accesses them via the concrete subclass; see architecture note below).

**`projectAndRender` in `QhorusMcpToolsBase`:**

```java
// package-private — not @Tool; avoids ToolOverloadDiscoverabilityTest violation
// shared by QhorusMcpTools and ReactiveQhorusMcpTools
<S> String projectAndRender(UUID channelId, RenderableProjection<S> projection) {
    ProjectionResult<S> result = projectionService.project(channelId, projection);
    return projection.render(result);
}
```

**Architecture note on `QhorusMcpToolsBase` injections:**

`channelService`, `channelStore`, `projectionService`, and `projectionRegistry` are not currently in the base class — they live in `QhorusMcpTools`. Two options:

- Move them to `QhorusMcpToolsBase` (clean, avoids duplication with reactive tool)
- Keep them in the concrete classes and have `resolveChannelId` / `projectAndRender` accept them as parameters

Recommended: move `channelService`, `channelStore`, `projectionService`, and `projectionRegistry` to `QhorusMcpToolsBase`. Both concrete tools need them; the base class is the right home. This is an additional scope expansion captured in the Changed Files table.

---

### 4. Reactive variant — `ReactiveQhorusMcpTools`

```java
@Tool(name = "project_channel", description = "...")
@Blocking  // full channel scan is I/O-intensive; @Blocking follows the existing
            // get_instance / get_message convention in ReactiveQhorusMcpTools
public String projectChannel(String channel, String projectionName) {
    UUID channelId = resolveChannelId(channel);
    RenderableProjection<?> projection = projectionRegistry.get(projectionName);
    return projectAndRender(channelId, projection);  // calls blocking ProjectionService
}
```

`ReactiveProjectionService` has all four overloads implemented but the blocking `ProjectionService` is used here for consistency with the existing `@Blocking` pattern in the reactive tool class, avoiding a reactive pipeline for a synchronous read.

---

## Known Gaps

- **Output size:** `render()` can produce arbitrarily large output for channels with many messages. The spec does not bound it. Filed as qhorus#239. Options for a future fix: `max_messages` param on the tool, or a contract that `render()` is responsible for truncating its own output.

- **`list_projections` tool:** An LLM calling `project_channel` with an unknown name gets an error with no recovery path. `names()` on `ProjectionRegistry` makes this trivial to implement. Filed as qhorus#240.

---

## What Is NOT in Scope

- `list_projections` — filed as #240.
- Scoped/incremental projection via MCP — `project_channel` always does a full scan. Incremental projection requires the caller to manage `ProjectionResult` cursors across tool calls, which is not practical over stateless MCP.
- Multi-format `render(ProjectionResult<S>, String format)` — use two `RenderableProjection` beans with different names instead.
- `@ChannelBound` auto-routing registry — separate future design.

---

## New Types

| Type | Module | Purpose |
|------|--------|---------|
| `RenderableProjection<S>` | `api/spi/` | Consumer SPI: fold + name + render |
| `ProjectionRegistry` | `runtime/message/` | Startup-validated name→projection map |

## Changed Files

| File | Change |
|------|--------|
| `QhorusMcpToolsBase` | Add `@Inject` for `ChannelService`, `ChannelStore`, `ProjectionService`, `ProjectionRegistry`; add `resolveChannelId()` and `projectAndRender()` helpers (package-private) |
| `QhorusMcpTools` | New `project_channel` tool; move channel/projection injections to base class |
| `ReactiveQhorusMcpTools` | New `project_channel @Blocking` tool |
| `ChannelProjection.java` | Update javadoc: document both `@ChannelBound` (future auto-routing) and `RenderableProjection`/`ProjectionRegistry` (explicit selection); remove stale `@ProjectionName` reference |

## No Changes

| File | Reason |
|------|--------|
| `ChannelProjection<S>` (interface) | `RenderableProjection<S>` extends it — no modification to the interface |
| `ProjectionService` | accepts `ChannelProjection<S>`; `RenderableProjection<S>` satisfies this |
| `ProjectionResult<S>` | unchanged |

---

## Testing

**Unit — `RenderableProjectionTest` (no CDI, no Quarkus):**
- `render(emptyResult)` returns non-null String (enforces the contract that `identity()` has a render)
- `render(result)` produces expected output for a known message sequence
- `render(partialResult)` where no messages matched — distinguishes "fold returned identity because no match" from `isEmpty()`

**Unit — `ProjectionRegistryTest` (no CDI, instantiate directly):**
- Duplicate `projectionName()` throws `IllegalStateException` at construction
- `get("unknown")` throws `IllegalArgumentException`
- `names()` returns all registered names

**Integration — `ProjectChannelToolIT` (`@QuarkusTest`, `@TestTransaction`):**
- Register a `@ApplicationScoped` test bundle in test source set (no `@Alternative @Priority(1)` needed — no production bundle with the same name exists, so no ambiguity)
- Write messages via `messageStore.put()` (not `MessageService.dispatch()` — keeps test focused on projection correctness, not dispatch enforcement)
- `tools.projectChannel("channel-name", "test-proj")` → assert rendered string matches expected
- `tools.projectChannel(channelUUID.toString(), "test-proj")` → assert same result (UUID path)
- Unknown projection name → `ToolCallException` (via `@WrapBusinessError`)
- Non-existent UUID → `ToolCallException` (channel existence check)
- Non-existent channel name → `ToolCallException`
- LAST_WRITE channel: two messages from same sender → projection folds one, not two
- `render()` that throws → exception propagates (verify `@WrapBusinessError` wraps it)
- Empty channel → `render()` called with `result.isEmpty() == true`; verify non-null return

**`ToolOverloadDiscoverabilityTest`:** `projectChannel` in both `QhorusMcpTools` and `ReactiveQhorusMcpTools` must be `@Tool`-annotated; `projectAndRender` and `resolveChannelId` must be package-private in `QhorusMcpToolsBase`.

---

## Platform Coherence Review

- **`consumer-spi-placement`** — `RenderableProjection<S>` in `api/spi/` ✓; `ProjectionRegistry` in `runtime/` (not consumer-facing) ✓
- **`event-log-left-fold-projection`** — `RenderableProjection<S>` extends `ChannelProjection<S>` ✓
- **`qhorus-entity-mapper-pure-transformer`** — `render()` is pure, no side effects ✓
- **`qhorus-service-store-seam`** — `ProjectionService` reads via `MessageStore` seam ✓
- **`ToolOverloadDiscoverabilityTest`** — `projectAndRender` and `resolveChannelId` package-private ✓
- **`backendId()` pattern** — `projectionName()` follows existing `ChannelBackend.backendId()` convention ✓

## Deferred Issues Filed

| Issue | Description |
|-------|-------------|
| qhorus#236 | Slug enforcement on channel names |
| qhorus#237 | MCP tool migration from channel_name to UUID-or-slug |
| qhorus#238 | Protocol: channel dual identity (UUID + slug) |
| qhorus#239 | Bound output size for `project_channel` render |
| qhorus#240 | `list_projections` MCP tool |
