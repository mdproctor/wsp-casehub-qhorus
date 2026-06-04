# Batch S/XS Fix — Design Spec
**Branch:** `issue-248-batch-s-xs`
**Issues:** qhorus#248, #244, #240, #239, #238, parent#163
**Date:** 2026-06-04

---

## #248 — FindOrCreateResult counter fix (S)

### Problem

`ConnectorChannelBackend.tryAutoCreate()` calls `channelService.findOrCreateWithBinding(req)` and increments `inbound_channels_auto_created_total` on every non-exceptional return. But `findOrCreateWithBinding()` has two success paths:

1. **Find-existing:** binding found on recheck under REQUIRES_NEW → returns the existing channel
2. **Create-new:** no binding found → creates channel and binding → returns new channel

Counter fires for both. Symptom: `concurrentFirstContact_oneBindingCreated_bothMessagesDelivered` fails with `expected: 2.0 but was: 3.0` when run after `afterAutoCreate`.

### Design

Add `public record FindOrCreateResult(Channel channel, boolean wasCreated)` in `runtime/channel/` package (public — `ConnectorChannelBackend` is in a sibling module).

Change `ChannelService.findOrCreateWithBinding()` signature:
```java
public FindOrCreateResult findOrCreateWithBinding(ChannelCreateRequest req)
```

- Find-existing path → `new FindOrCreateResult(channel, false)`  
- Create-new path → `new FindOrCreateResult(channel, true)`

In `ConnectorChannelBackend.tryAutoCreate()`: only increment counter when `result.wasCreated()`.

No change to `ReactiveChannelService` (doesn't have `findOrCreateWithBinding`).

**Test:** update `ConnectorAutoChannelBackendTest.concurrentFirstContact_oneBindingCreated_bothMessagesDelivered` to assert the counter delta is 1 (not 2), confirming the concurrent loser doesn't increment.

---

## #244 — set_channel_type_constraints tool (S)

### Problem

`create_channel` accepts `allowed_types` and `denied_types`. No MCP tool exists to update these on an existing channel. The pattern for updatable channel fields is individual setter tools (`set_channel_writers`, `set_channel_admins`, `set_channel_rate_limits`). Since `allowed_types` and `denied_types` must be validated together (no overlap), they share a single setter tool.

### Design

**New MCP tool:** `set_channel_type_constraints(channel_name, allowed_types?, denied_types?)`

- Both params optional. `null` clears the constraint (removes it).
- Returns `ChannelDetail`.

**Validation in `ChannelService.setTypeConstraints(String name, String allowedTypes, String deniedTypes)`:**

```java
Set<MessageType> allowed = MessageType.parseTypes(allowedTypes);   // throws IAE on bad names
Set<MessageType> denied  = MessageType.parseTypes(deniedTypes);
Set<MessageType> overlap = new HashSet<>(allowed); overlap.retainAll(denied);
if (!overlap.isEmpty()) throw new IllegalArgumentException(
    "allowed_types and denied_types must not overlap: " + overlap);
channel.allowedTypes = blankToNull(allowedTypes);
channel.deniedTypes  = blankToNull(deniedTypes);
```

`MessageType.parseTypes()` already handles null/blank → returns empty set.

Mirror in `ReactiveChannelService.setTypeConstraints(...)` → `Uni<Channel>`.

Add to both `QhorusMcpTools` and `ReactiveQhorusMcpTools`.

**Protocol compliance:** PP-20260604-c19f7c (D1 gate) applies to creation. For updates, validation lives in the service method — acceptable because the update path has no compact constructor. PP-20260604-a7ad99 governs what constraints are architecturally meaningful.

---

## #240 — list_projections MCP tool (XS)

### Problem

An LLM calling `project_channel` with an unknown `projection_name` gets an error with no recovery path. `ProjectionRegistry.registeredNames()` already exists but is not exposed.

### Design

**New MCP tool:** `list_projections()` → `List<String>` (sorted)

```java
@Tool(name = "list_projections",
      description = "List all projection names registered with ProjectionRegistry. "
          + "Use with project_channel's projection_name argument.")
public List<String> listProjections() {
    return projectionRegistry.registeredNames().stream().sorted().toList();
}
```

`projectionRegistry` is already injected in `QhorusMcpToolsBase`. Add field access in both `QhorusMcpTools` and `ReactiveQhorusMcpTools`.

No description SPI method added (YAGNI — no confirmed consumers need it).

---

## #239 — project_channel max_messages (S)

### Problem

`project_channel` folds the entire channel history and returns `render()` output. On large channels this can produce output that exceeds MCP transport limits.

### Design

Add optional `max_messages` param (default 200, -1 = unlimited) to `project_channel`.

When `maxMessages > 0`: use the scoped `ProjectionService.project(channelId, scope, projection)` overload with `MessageQuery.builder().limit(maxMessages).build()`. When null or ≤ 0: fold all (current behavior).

Update `projectAndRender()` in `QhorusMcpToolsBase` to accept `Integer maxMessages`:

```java
<S> String projectAndRender(UUID channelId, RenderableProjection<S> projection, Integer maxMessages) {
    ProjectionResult<S> result;
    if (maxMessages != null && maxMessages > 0) {
        result = projectionService.project(channelId,
                MessageQuery.builder().limit(maxMessages).build(), projection);
    } else {
        result = projectionService.project(channelId, projection);
    }
    return projection.render(result);
}
```

Tool description documents: "Defaults to the 200 most recent messages. Pass -1 to fold the full history (may produce large output on busy channels)."

Mirror in `ReactiveQhorusMcpTools`.

**`ToolOverloadDiscoverabilityTest` compliance:** `projectAndRender()` is package-private — adding a new overload is fine. Do not make it `public`.

---

## #238 — Channel dual identity protocol (S)

Write `docs/protocols/casehub/qhorus-channel-dual-identity.md`:

- **UUID** = machine identity. Assigned at creation, immutable, stable across renames (if renames were possible — they are not).
- **Name** = semantic slug. Unique within a Qhorus instance. Human- and LLM-readable. Immutable: channel names cannot be changed once created (no `rename_channel` MCP tool exists or will be added without a migration strategy).
- `resolveChannel(String)` in `QhorusMcpToolsBase` accepts either UUID or name for all tools that take a `channel` parameter.
- All `ChannelDetail` responses include both `channelId` (UUID) and `channelName` (slug).
- UUID is the preferred reference for machine-to-machine and cross-repo use. Name is preferred for human operators and LLM tool callers.
- Slug format enforcement pending qhorus#236 — currently only uniqueness is enforced.

---

## parent#163 — Oversight doc update (XS)

In `casehubio/parent` repo, update `docs/repos/casehub-qhorus.md` normative channel layout table:

Change `/oversight` row from `allowedTypes=COMMAND,RESPONSE` to `deniedTypes=EVENT, allowedTypes=null`.

PLATFORM.md already correct (`deniedTypes = EVENT`). Only the per-repo deep-dive needs updating.

---

## Implementation Order

1. #248 — `FindOrCreateResult` (connector-backend module)
2. #244 — `set_channel_type_constraints` (runtime: ChannelService, MCP tools)
3. #240 — `list_projections` (runtime: MCP tools only)
4. #239 — `project_channel max_messages` (runtime: QhorusMcpToolsBase, MCP tools)
5. #238 — dual identity protocol doc (workspace → parent protocols dir)
6. parent#163 — oversight doc update (parent repo)

Each issue committed separately with `Closes #N` or `Refs #N`.
