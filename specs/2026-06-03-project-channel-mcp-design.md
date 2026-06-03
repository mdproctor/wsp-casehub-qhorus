# Design Spec — `project_channel` MCP Tool

**Date:** 2026-06-03
**Issue:** qhorus#232
**Status:** Approved for implementation

---

## Problem

`ProjectionService.project()` is generic over `<S>` and cannot be exposed as an MCP tool directly — MCP tools must return concrete types. Two things are needed: a registry that lets the tool find the right projection by name, and a render step that converts the typed state `<S>` into a `String`.

---

## Design

### 1. `RenderableProjection<S>` SPI — `api/spi/`

A composite interface that extends `ChannelProjection<S>` with a name method and a render step:

```java
package io.casehub.qhorus.api.spi;

public interface RenderableProjection<S> extends ChannelProjection<S> {

    /**
     * The registration name used to select this projection via
     * {@code project_channel(projection_name = "...")}.
     *
     * <p>Must be unique across all {@link RenderableProjection} beans in the deployment.
     * Collisions are detected at startup by {@link ProjectionRegistry} and cause
     * immediate deployment failure.
     *
     * @return a stable, non-null, non-empty name
     */
    String projectionName();

    /**
     * Converts the materialised projection result to a String suitable for
     * return from an MCP tool. Called once per {@code project_channel} invocation,
     * after the fold completes.
     *
     * <p>Receives the full {@link ProjectionResult} so that the renderer can
     * distinguish two distinct cases that both produce the identity state:
     * <ul>
     *   <li>{@code result.isEmpty() == true} — the channel contained no messages at all.</li>
     *   <li>{@code result.isEmpty() == false} but {@code state == identity()} —
     *       the channel has messages, but none matched this projection's fold logic.</li>
     * </ul>
     * {@code state == identity()} alone cannot distinguish these cases. Use
     * {@code result.isEmpty()} for the definitive empty-channel signal.
     *
     * <p>Must be pure and non-blocking — called on the MCP dispatch thread.
     * Must not return {@code null}.
     *
     * @param result the full projection result from a completed fold
     * @return a human-readable or structured string representation
     */
    String render(ProjectionResult<S> result);
}
```

**Placement rationale:** `api/spi/` per `consumer-spi-placement` protocol — external consumers (Claudony, application repos) will implement `RenderableProjection<S>` and they depend only on the lightweight `api/` module. `RenderableProjection<S>` adds no CDI annotations or `jakarta.enterprise` dependencies — only the `ChannelProjection<S>` and `ProjectionResult<S>` types already in `api/spi/`.

**Scope:** Implement as `@ApplicationScoped`. The `ProjectionRegistry` acquires references at startup and cannot manage per-instance lifecycles. `@Dependent` scope is not supported.

**No CDI qualifier needed:** Name discovery uses `projectionName()` introspection — no annotation, no `AnnotationLiteral`. This matches the existing `ChannelBackend.backendId()` pattern in the gateway.

**Single responsibility:** `RenderableProjection<S>` is "a projection that knows its name and how to render itself." The fold (`identity`, `apply`), the name, and the render are cohesive concerns. Consumers cannot register a renderable projection without providing all three — correct by construction.

**Multiple formats — composition not parameterisation:** If the same fold logic needs two render formats (markdown and JSON), register two `RenderableProjection` beans — `"summary-markdown"` and `"summary-json"` — both delegating their `identity()`/`apply()` to a shared implementation. A format parameter on `render()` would impose a dispatch concern on every implementor; the CDI bean model handles it naturally.

`ProjectionService` requires no changes: it accepts `ChannelProjection<S>`, and `RenderableProjection<S>` is a `ChannelProjection<S>`.

---

### 2. `ProjectionRegistry` — `runtime/message/`

```java
package io.casehub.qhorus.runtime.message;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.spi.RenderableProjection;

/**
 * Registry for {@link RenderableProjection} beans, keyed by
 * {@link RenderableProjection#projectionName()}.
 *
 * <p>Collects all {@link RenderableProjection} beans at startup. Name collisions —
 * two beans with the same {@link RenderableProjection#projectionName()} — are
 * detected at {@link PostConstruct} time and cause immediate deployment failure.
 */
@ApplicationScoped
public class ProjectionRegistry {

    @Inject
    @Any
    Instance<RenderableProjection<?>> projectionBeans;

    private Map<String, RenderableProjection<?>> registry;

    @PostConstruct
    void build() {
        final var map = new HashMap<String, RenderableProjection<?>>();
        for (final RenderableProjection<?> proj : projectionBeans) {
            final String name = proj.projectionName();
            if (map.put(name, proj) != null) {
                throw new IllegalStateException(
                        "Duplicate RenderableProjection name '" + name + "'. " +
                        "Each RenderableProjection must have a unique projectionName().");
            }
        }
        registry = Collections.unmodifiableMap(map);
    }

    /**
     * Looks up a {@link RenderableProjection} by name.
     *
     * @throws IllegalArgumentException if no projection is registered with that name
     */
    @SuppressWarnings("unchecked")
    public <S> RenderableProjection<S> get(final String name) {
        final RenderableProjection<?> proj = registry.get(name);
        if (proj == null) {
            throw new IllegalArgumentException(
                    "No projection registered with name '" + name + "'. Available: " + registry.keySet());
        }
        return (RenderableProjection<S>) proj;
    }

    /** Available projection names — used by a future {@code list_projections} tool. */
    public Set<String> registeredNames() {
        return registry.keySet();
    }
}
```

**Startup collision detection:** `ProjectionRegistry.build()` throws `IllegalStateException` on name collision. This surfaces duplicate registration at deployment time, not at first tool call, with a clear error message naming the duplicate.

**No `@Dependent` support:** The registry acquires bean references during `@PostConstruct` iteration. `@ApplicationScoped` projections are CDI proxies — correct and efficient. `@Dependent` beans would be acquired and held without lifecycle management; do not use.

---

### 3. `resolveChannel` helper — `QhorusMcpToolsBase`

A shared channel resolution method used by both blocking and reactive tools. Resolves by name or UUID, and validates existence for the UUID path (a valid-format but non-existent UUID would otherwise silently project as if the channel were empty):

```java
// package-private — shared between QhorusMcpTools and ReactiveQhorusMcpTools
UUID resolveChannel(String channel) {
    UUID uuid;
    try {
        uuid = UUID.fromString(channel);
    } catch (IllegalArgumentException e) {
        // Not a UUID — resolve by name
        return channelService.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel))
                .id;
    }
    // Valid UUID — verify the channel exists.
    // ProjectionService returns identity() for unknown channelIds — indistinguishable
    // from an empty channel. Validate here to surface the error clearly.
    channelService.findById(uuid)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel));
    return uuid;
}
```

**`QhorusMcpToolsBase` field:** the existing `channelService` injection in `QhorusMcpToolsBase` is sufficient — no new injection needed for this method.

---

### 4. `projectAndRender` helper — `QhorusMcpToolsBase`

A shared type-capturing helper that bridges the wildcard `RenderableProjection<?>` from the registry to the typed `render(ProjectionResult<S>)` call:

```java
// package-private — shared; never @Tool and never public (ToolOverloadDiscoverabilityTest)
<S> String projectAndRender(UUID channelId, RenderableProjection<S> projection) {
    ProjectionResult<S> result = projectionService.project(channelId, projection);
    return projection.render(result);
}
```

`projectionService` is injected in `QhorusMcpToolsBase` alongside the other shared services.

---

### 5. `project_channel` MCP tool — `QhorusMcpTools`

```java
@Tool(name = "project_channel",
      description = "Project a channel's message history through a named projection and return " +
                    "the rendered result. The projection folds all messages via the named " +
                    "RenderableProjection and returns its render() output as a String. " +
                    "Reads proceed on paused channels — projection is a read-only operation. " +
                    "On LAST_WRITE channels the fold sees only the current snapshot (one message " +
                    "per sender, not full history) — projections that assume a complete history " +
                    "will produce incorrect results on LAST_WRITE channels.")
public String projectChannel(
        @ToolArg(name = "channel",
                 description = "Channel name or UUID") String channel,
        @ToolArg(name = "projection_name",
                 description = "Name of the registered RenderableProjection (e.g. 'channel-summary')") String projectionName) {

    UUID channelId = resolveChannel(channel);
    RenderableProjection<?> projection = projectionRegistry.get(projectionName);
    return projectAndRender(channelId, projection);
}
```

**`projectionRegistry` field:**

```java
@Inject
ProjectionRegistry projectionRegistry;
```

**`projectAndRender` and `resolveChannel`** are defined in `QhorusMcpToolsBase` (shared with `ReactiveQhorusMcpTools`). `projectionRegistry` is injected directly in each concrete tool class — not in the base class.

---

### 6. Reactive variant — `ReactiveQhorusMcpTools`

Mirrors the blocking tool. Uses `@Blocking` because `ProjectionService` (blocking) is used directly — consistent with the existing `get_instance` and `get_message` tools in `ReactiveQhorusMcpTools` which are also `@Blocking`:

```java
@Tool(name = "project_channel", ...)
@Blocking
public String projectChannel(String channel, String projectionName) {
    UUID channelId = resolveChannel(channel);
    RenderableProjection<?> projection = projectionRegistry.get(projectionName);
    return projectAndRender(channelId, projection);
}
```

**Why `@Blocking` rather than reactive:** `ProjectionService` is a blocking service. `ReactiveProjectionService` (the reactive mirror) is fully implemented but the reactive tool uses `@Blocking` for consistency with the existing `get_instance` / `get_message` pattern — all three are read operations that benefit from the worker-thread annotation without needing the reactive code path.

---

## What Is NOT in Scope

- **`list_projections` tool** — useful for LLM discoverability but not required for #232. Filed as qhorus#239. `ProjectionRegistry.registeredNames()` is already wired to support it when implemented.
- **Scoped projection via MCP** — `project_channel` always does a full scan. Incremental projection requires the caller to manage `ProjectionResult` cursors, which is not practical over MCP. Filed as follow-up.
- **Render output size limit** — `render()` is responsible for keeping its output to a practical size. No framework-level truncation. Known gap; to be revisited if large channels cause MCP transport failures.
- **`ProjectionBundle` validation** — `render(identity())` must return non-null, by contract. Not enforced at the framework level.

---

## Relationship to `@ChannelBound` (Future)

`ChannelProjection.java` documents a future `@ChannelBound` registry. That registry and `ProjectionRegistry` serve distinct purposes and will coexist:

| Registry | Selection | Use case |
|----------|-----------|----------|
| `ProjectionRegistry` | Explicit — tool caller names the projection | `project_channel("my-channel", "summary")` |
| `@ChannelBound` (future) | Automatic — channel name routes to projection | Dashboards, automated read-models |

`ChannelProjection.java`'s javadoc will be updated to document both registries. A `RenderableProjection<S>` can also be `@ChannelBound` — the two mechanisms are orthogonal qualifiers on the same bean.

---

## New Types

| Type | Module | Purpose |
|------|--------|---------|
| `RenderableProjection<S>` | `api/spi/` | Consumer SPI: fold + name + render |
| `ProjectionRegistry` | `runtime/message/` | Startup-time name→projection map; collision detection |

## Changed Files

| File | Change |
|------|--------|
| `QhorusMcpToolsBase` | new `resolveChannel()` + `projectAndRender()` helpers (package-private); move `channelService`, `channelStore`, `projectionService` injections here — no `ProjectionRegistry` injection in the base class |
| `QhorusMcpTools` | new `project_channel` tool; inject `ProjectionRegistry` directly (not via base class); call `registry.get(name)` then `projectAndRender()` |
| `ReactiveQhorusMcpTools` | new `project_channel @Blocking` tool; same `ProjectionRegistry` injection pattern as blocking tool |
| `ChannelProjection.java` | javadoc: document both `projectionName()` + `ProjectionRegistry` (explicit selection, current) and `@ChannelBound` (auto-routing, future) as planned registries — remove stale implication that `@ChannelBound` is the only path |

## Removed (vs. prior draft)

| Type | Reason |
|------|--------|
| `ProjectionBundle<S>` | Renamed `RenderableProjection<S>`; `render(S)` → `render(ProjectionResult<S>)` |
| `@ProjectionName` | Replaced by `projectionName()` introspection method — no CDI qualifier needed |
| `ProjectionNameLiteral` | Replaced by `ProjectionRegistry` map lookup — no `AnnotationLiteral` in `api/` |

## No Changes

| File | Reason |
|------|--------|
| `ChannelProjection<S>` | `RenderableProjection<S>` extends it — no modification needed |
| `ProjectionService` | already accepts `ChannelProjection<S>` |
| `ReactiveProjectionService` | already accepts `ChannelProjection<S>` |
| `ProjectionResult<S>` | unchanged |

---

## Testing

**Unit — `RenderableProjectionTest` (no CDI, no Quarkus):**
- `projectionName()` returns a non-null, non-empty string
- `render(result)` returns non-null for `result.isEmpty() == true` (empty channel case)
- `render(result)` returns non-null for `result.isEmpty() == false` with real state
- `apply` + `render` produce expected output for a known message sequence
- `identity()` returns a fresh instance on each call

**Unit — `ProjectionRegistryTest` (CDI-free, direct instantiation):**
- Two bundles with different names → `get("name-a")` and `get("name-b")` return correct beans
- Two bundles with the same name → `build()` throws `IllegalStateException` (collision detection)
- `get()` with unknown name → `IllegalArgumentException` with available-names list in message
- Empty registry (no projections registered) → `get()` throws `IllegalArgumentException`; `registeredNames()` returns empty set

**Integration — `ProjectChannelToolIT` (`@QuarkusTest` + `@TestTransaction`):**
- Add a `@ApplicationScoped` test bundle (no `@Alternative` needed — a new named bean with no production counterpart is discovered by Quarkus test CDI as a plain `@ApplicationScoped` bean in test sources)
- Write messages via `messageStore.put()` (not `MessageService.dispatch()` — keeps tests focused on projection behaviour)
- Call `tools.projectChannel("channel-name", "test-proj")` and assert rendered string
- Call with unknown projection name → assert `ToolCallException` (via `@WrapBusinessError`) with message containing the unknown name
- Call with valid-format but non-existent UUID → assert `ToolCallException` ("Channel not found")
- Call with LAST_WRITE channel → two messages from same sender → assert fold sees only one message
- Call on paused channel → assert projection succeeds (reads proceed on paused channels)
- `render(ProjectionResult)` where `isEmpty() == true` (empty channel) → assert non-null, non-empty result string (verifies the empty-channel contract)

---

## Platform Coherence

- **`consumer-spi-placement`** — `RenderableProjection<S>` in `api/spi/` ✓
- **`event-log-left-fold-projection`** — `RenderableProjection<S>` extends `ChannelProjection<S>`, preserving the left-fold contract ✓
- **`qhorus-entity-mapper-pure-transformer`** — `render()` is pure, non-blocking ✓
- **`ToolOverloadDiscoverabilityTest`** — `projectAndRender` is package-private in `QhorusMcpToolsBase`; `resolveChannel` is package-private ✓
- **`backendId()` pattern** — `projectionName()` follows the same introspection-over-annotation convention as `ChannelBackend.backendId()` ✓
- **Deferred issues** — qhorus#236 (slug enforcement), qhorus#237 (MCP UUID migration), qhorus#238 (dual-identity protocol), qhorus#239 (`list_projections`) ✓
