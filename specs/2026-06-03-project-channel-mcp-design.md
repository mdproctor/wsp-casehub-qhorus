# Design Spec — `project_channel` MCP Tool

**Date:** 2026-06-03
**Issue:** qhorus#232
**Status:** Approved for implementation

---

## Problem

`ProjectionService.project()` is generic over `<S>` and cannot be exposed as an MCP tool directly — MCP tools must return concrete types. Two things are needed: a registry that lets the tool find the right projection by name, and a render step that converts the typed state `<S>` into a `String`.

---

## Design

### 1. `ProjectionBundle<S>` SPI — `api/spi/`

A composite interface that extends `ChannelProjection<S>` with a render step:

```java
package io.casehub.qhorus.api.spi;

public interface ProjectionBundle<S> extends ChannelProjection<S> {

    /**
     * Converts the materialised projection state to a String suitable for
     * return from an MCP tool. Called once per {@code project_channel} invocation,
     * after the fold completes.
     *
     * <p>Must handle the empty-channel case: when the channel has no messages,
     * {@code state} equals {@code identity()} — implementations must not assume
     * {@code state} contains real data.
     *
     * <p>Must be pure and non-blocking — called on the MCP dispatch thread.
     *
     * @param state the materialised state from a completed fold
     * @return a human-readable or structured string representation
     */
    String render(S state);
}
```

**Placement rationale:** `api/spi/` per `consumer-spi-placement` protocol — external consumers (Claudony, application repos) will implement `ProjectionBundle<S>` and they depend only on the lightweight `api/` module.

**Single responsibility:** `ProjectionBundle<S>` is "a projection that knows how to render itself." The fold (`identity`, `apply`) and the render are the bundle's cohesive concern. Consumers cannot register a projection that is renderable without providing the render — correct by construction.

`ProjectionService` requires no changes: it accepts `ChannelProjection<S>`, and `ProjectionBundle<S>` is a `ChannelProjection<S>`.

---

### 2. `@ProjectionName` CDI qualifier — `api/spi/`

```java
package io.casehub.qhorus.api.spi;

@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
public @interface ProjectionName {
    String value();
}
```

Consumers annotate their beans:

```java
@ProjectionName("channel-summary")
@ApplicationScoped
public class ChannelSummaryBundle implements ProjectionBundle<SummaryState> { ... }
```

No registration call — CDI discovers the bean automatically at startup. Projections activate by being on the classpath.

---

### 3. `ProjectionNameLiteral` — `api/spi/`

An `AnnotationLiteral` subclass so the tool can select projections dynamically:

```java
package io.casehub.qhorus.api.spi;

import jakarta.enterprise.util.AnnotationLiteral;

public final class ProjectionNameLiteral extends AnnotationLiteral<ProjectionName>
        implements ProjectionName {

    private final String value;

    public ProjectionNameLiteral(String value) { this.value = value; }

    @Override public String value() { return value; }
}
```

Placed in `api/spi/` alongside `@ProjectionName` — consumers need it if they write tests that select projections programmatically.

---

### 4. `project_channel` MCP tool — `QhorusMcpTools`

```java
@Tool(name = "project_channel",
      description = "Project a channel's message history through a named projection and return the rendered result. ...")
public String projectChannel(
        @ToolArg(name = "channel",
                 description = "Channel name or UUID") String channel,
        @ToolArg(name = "projection_name",
                 description = "Name of the registered ProjectionBundle (e.g. 'channel-summary')") String projectionName) {

    UUID channelId = resolveChannel(channel);

    Instance<ProjectionBundle<?>> found = projectionBundles.select(
            new ProjectionNameLiteral(projectionName));
    if (found.isUnsatisfied()) {
        throw new IllegalArgumentException(
                "No projection registered with name '" + projectionName + "'");
    }

    return projectAndRender(channelId, found.get());
}

// package-private — not @Tool, avoids ToolOverloadDiscoverabilityTest violation
<S> String projectAndRender(UUID channelId, ProjectionBundle<S> bundle) {
    ProjectionResult<S> result = projectionService.project(channelId, bundle);
    return bundle.render(result.state());
}
```

**Channel resolution** (`resolveChannel`) — reuses the existing pattern:

```java
private UUID resolveChannel(String channel) {
    try {
        return UUID.fromString(channel);
    } catch (IllegalArgumentException e) {
        return channelService.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel))
                .id;
    }
}
```

**`projectionBundles` field:**

```java
@Inject
@Any
Instance<ProjectionBundle<?>> projectionBundles;
```

**`projectionService` field:** already present in the tool class (injected alongside `channelService`).

**`projectAndRender` is package-private** — captures `<S>` to preserve type safety between the wildcard `ProjectionBundle<?>` and the typed `render(S)` call. Per `ToolOverloadDiscoverabilityTest` convention, never `public`.

---

### 5. Reactive variant — `ReactiveQhorusMcpTools`

Mirrors the blocking tool. `ReactiveProjectionService.project()` is called instead:

```java
@Tool(name = "project_channel", ...)
@Blocking  // ProjectionService read is blocking; reactive tool needs worker-thread annotation
public String projectChannel(String channel, String projectionName) { ... }
```

`ReactiveProjectionService` is not yet implemented for all overloads — the reactive tool delegates to the blocking `ProjectionService` via `@Blocking` rather than introducing a reactive-only path. This is consistent with the existing `get_instance` and `get_message` tools in `ReactiveQhorusMcpTools` which are already annotated `@Blocking`.

---

## What Is NOT in Scope

- `list_projections` tool — useful but not required for #232; projections are named by convention, callers know their names.
- Scoped/incremental projection via MCP — `project_channel` always does a full scan. Incremental projection requires the caller to manage `ProjectionResult` cursors, which is not practical over MCP.
- `ProjectionBundle` validation (e.g., assert `render(identity())` returns non-null) — convention only.

---

## New Types

| Type | Module | Purpose |
|------|--------|---------|
| `ProjectionBundle<S>` | `api/spi/` | Consumer SPI: fold + render |
| `@ProjectionName` | `api/spi/` | CDI qualifier for registry lookup |
| `ProjectionNameLiteral` | `api/spi/` | `AnnotationLiteral` for `Instance.select()` |

## Changed Files

| File | Change |
|------|--------|
| `QhorusMcpTools` | new `project_channel` tool + `projectAndRender` helper + `projectionBundles` injection |
| `ReactiveQhorusMcpTools` | same, with `@Blocking` |

## No Changes

| File | Reason |
|------|--------|
| `ChannelProjection<S>` | `ProjectionBundle<S>` extends it — no modification needed |
| `ProjectionService` | already accepts `ChannelProjection<S>` |
| `ProjectionResult<S>` | unchanged |

---

## Testing

**Unit — `ProjectionBundleTest` (no CDI, no Quarkus):**
- `render(identity())` returns non-null for a sample bundle
- `apply` + `render` produce expected output for a known message sequence

**Integration — `ProjectChannelToolTest` (@QuarkusTest + @TestTransaction):**
- Register a `@ProjectionName("test-proj") @ApplicationScoped` bundle as a CDI `@Alternative @Priority(1)` (test-only bean)
- Write messages via `messageStore.put()` (not `MessageService.dispatch()`)
- Call `tools.projectChannel("channel-name", "test-proj")`
- Assert rendered string matches expected output
- Assert `project_channel` with unknown projection name throws `ToolCallException` (via `@WrapBusinessError`)

---

## Platform Coherence

- **`consumer-spi-placement`** — `ProjectionBundle<S>`, `@ProjectionName`, `ProjectionNameLiteral` all in `api/spi/` ✓
- **`event-log-left-fold-projection`** — `ProjectionBundle<S>` extends `ChannelProjection<S>`, preserving the left-fold contract ✓
- **`qhorus-entity-mapper-pure-transformer`** — `render()` is pure, no side effects ✓
- **`ToolOverloadDiscoverabilityTest`** — `projectAndRender` is package-private ✓
- **Deferred issues** — qhorus#236 (slug enforcement), qhorus#237 (MCP UUID migration), qhorus#238 (dual-identity protocol) ✓

---

## Open Questions (deferred)

- Should `project_channel` return the channel name/id alongside the rendered string (e.g. a structured result record)? Decision: return plain `String` for now — simpler for LLM consumption.
- Should `render(identity())` have a sentinel contract ("return empty-channel message")? Decision: by convention only — not enforced at the SPI level.
