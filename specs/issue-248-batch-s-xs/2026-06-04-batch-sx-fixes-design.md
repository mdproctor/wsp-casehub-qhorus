# Batch S/XS Fix — Design Spec
**Branch:** `issue-248-batch-s-xs`
**Issues:** qhorus#248, #244, #240, #239, #238, parent#163
**Date:** 2026-06-04 (revised after code review)

---

## Implementation order

1. #248 — `FindOrCreateResult` (connector-backend module)
2. #238 — dual identity protocol doc (parent/docs/protocols/casehub/) — comes before #244 because the dual-identity parameter convention informs #244's tool design
3. #244 — `set_channel_type_constraints` (runtime: ChannelService + MCP tools)
4. #240 — `list_projections` (runtime: MCP tools)
5. #239 — `project_channel max_messages` (runtime: QhorusMcpToolsBase + MCP tools)
6. parent#163 — oversight doc update (parent repo)

Each issue committed separately. Commit tag: `Closes #N` for all items except any that remain open.

---

## #248 — FindOrCreateResult counter fix (S) · `Closes #248`

### Root cause

`ConnectorChannelBackend.tryAutoCreate()` calls `channelService.findOrCreateWithBinding(req)` and unconditionally increments `inbound_channels_auto_created_total` on every non-exceptional return. But `findOrCreateWithBinding()` has two success paths:

1. **Find-existing:** binding found on recheck under `REQUIRES_NEW` → returns the existing channel
2. **Create-new:** no binding found → creates channel and binding → returns new channel

Both return `Channel` with no way for the caller to distinguish them, so the counter fires for both. The symptom is `ConcurrentAutoChannelTest.concurrentFirstContact_oneBindingCreated` asserting `isEqualTo(autoCreatedBefore + 1.0)` but getting `autoCreatedBefore + 2.0` when run after `afterAutoCreate_fanOut_sendsToSenderPhone`.

### Why `channel.autoCreated` can't substitute

`Channel.autoCreated` is set to `true` only in the create-new path of `findOrCreateWithBinding`. On the find-existing path the returned entity's `autoCreated` reflects its original creation (always `true` — every auto-created channel is `autoCreated`). The flag answers "was this channel ever auto-created?" not "was it created by this call?" It cannot distinguish the two success paths.

### Design

Add `public record FindOrCreateResult(Channel channel, boolean wasCreated)` to the `io.casehub.qhorus.runtime.channel` package (must be `public` — `ConnectorChannelBackend` is in a sibling module that depends on `runtime`).

Change `ChannelService.findOrCreateWithBinding()`:
```java
public FindOrCreateResult findOrCreateWithBinding(ChannelCreateRequest req)
```

- Find-existing path → `new FindOrCreateResult(channel, false)`
- Create-new path → `new FindOrCreateResult(channel, true)`

In `ConnectorChannelBackend.tryAutoCreate()`: unwrap `result.channel()` for routing; only increment the counter when `result.wasCreated()`.

No change to `ReactiveChannelService` (no equivalent method exists there).

### Test

`ConcurrentAutoChannelTest.concurrentFirstContact_oneBindingCreated` already has the correct assertion:
```java
assertThat(backend.autoCreatedCount(CONNECTOR)).isEqualTo(autoCreatedBefore + 1.0);
```
No test update is needed. The test is currently failing because the code is wrong. The fix is to the code only.

---

## #238 — Channel dual identity protocol (S) · `Closes #238`

### Target

Write `docs/protocols/casehub/qhorus-channel-dual-identity.md` in the **`casehub/parent` repo** (`~/claude/casehub/parent/docs/protocols/casehub/`). This is the same location as all other casehub-scoped protocols (PP-20260604-c19f7c etc.). The garden at `~/.hortora/garden/` holds universal (non-casehub) protocols only.

### Purpose

Most content here is already documented in CLAUDE.md. The protocol makes it authoritative, machine-retrievable via the protocol index, and a citable reference for other tools (e.g., #244's parameter convention). The value is discoverability and normative weight, not novelty.

### Content

- **UUID** = machine identity. Assigned at creation, immutable, permanent.
- **Name** = semantic slug. Unique within a Qhorus instance. Human- and LLM-readable. Immutable: no `rename_channel` MCP tool exists or will be added without a Flyway migration strategy. Names cannot be changed once created.
- `resolveChannel(String channel)` in `QhorusMcpToolsBase` accepts either form. It tries UUID parse first. **Sharp edge:** a channel name that happens to be UUID-shaped (e.g., a channel named `"550e8400-e29b-41d4-a716-446655440000"`) will resolve as UUID, not as a name lookup. Channel names must not be UUID-shaped.
- All `ChannelDetail` responses include both `channelId` (UUID) and `channelName` (slug).
- UUID is the preferred reference for machine-to-machine and cross-repo use (stable across potential future renames). Name is preferred for human operators and LLM tool callers (readable, contextual).
- Slug format enforcement is pending qhorus#236 — currently only uniqueness is enforced at the DB level.
- New MCP tools added after this protocol must use `channel` (UUID-or-name) as the parameter name and delegate to `resolveChannel()`. Existing tools using `channel_name` (name-only) are migrated in qhorus#237.

---

## #244 — set_channel_type_constraints tool (S) · `Closes #244`

### Design

**New MCP tool:** `set_channel_type_constraints(channel, allowed_types?, denied_types?)`

- `channel` — UUID or name, per the dual-identity protocol (#238). Delegates to `resolveChannel()`.
- `allowed_types` and `denied_types` — both optional.
- Returns `ChannelDetail`.

**This is a full-replacement operation.** Both `allowedTypes` and `deniedTypes` on the channel entity are replaced atomically with whatever values are passed. Passing `null` for a field clears it (removes the constraint). There is no "leave unchanged" sentinel — if you pass `denied_types=null`, the existing `deniedTypes` is cleared. Callers must pass both fields if they want to preserve the other. The tool description must state this prominently.

This is intentional: `allowedTypes` and `deniedTypes` must be validated together (no overlap), so they must be set atomically. A partial-update API would require reading current state first and would make the validation boundary ambiguous.

**No admin access guard.** All other setter tools (`set_channel_rate_limits`, `set_channel_writers`, `set_channel_admins`) also lack admin guards. Consistency is more valuable here than per-tool ACL — if admin guards are needed, they should be added uniformly across all setters as a separate issue. Type constraints are a design-time decision, not a runtime security gate.

**Constraint is prospective only.** Adding `denied_types=QUERY` to a channel that already has QUERY messages does not affect those messages. The constraint applies to future `dispatch()` calls only. The tool description must state this.

**Service method:** `ChannelService.setTypeConstraints(UUID channelId, String allowedTypes, String deniedTypes)`:
```java
Set<MessageType> allowed = MessageType.parseTypes(allowedTypes);  // null/blank → empty set; throws IAE on bad names
Set<MessageType> denied  = MessageType.parseTypes(deniedTypes);
Set<MessageType> overlap = new HashSet<>(allowed); overlap.retainAll(denied);
if (!overlap.isEmpty()) throw new IllegalArgumentException(
    "allowed_types and denied_types must not overlap: " + overlap);
channel.allowedTypes = blankToNull(allowedTypes);
channel.deniedTypes  = blankToNull(deniedTypes);
```

UUID-based (not name-based) because the tool uses `resolveChannel()` to obtain the channel ID before calling the service — consistent with how future tools will work after #237.

**ReactiveChannelService:** add `Uni<Channel> setTypeConstraints(UUID channelId, String allowedTypes, String deniedTypes)` mirroring the blocking version.

Add `set_channel_type_constraints` to both `QhorusMcpTools` and `ReactiveQhorusMcpTools`.

**ToolOverloadDiscoverabilityTest:** `setTypeConstraints` is a new method name, no existing `@Tool` by that name — no overload risk. Confirm no `public` non-`@Tool` overloads are added.

### Tests

- Valid non-overlapping `allowed_types` and `denied_types` → both persisted, `ChannelDetail` reflects update
- Overlapping types → `IllegalArgumentException`
- Unknown type name → `IllegalArgumentException` (from `MessageType.parseTypes`)
- `null` for one field clears it; other field persists unchanged only if explicitly re-passed
- `null` for both clears both constraints
- Both `QhorusMcpTools.setChannelTypeConstraints` and `ReactiveQhorusMcpTools` paths exercised

---

## #240 — list_projections MCP tool (XS) · `Closes #240`

### Design

`ProjectionRegistry.registeredNames()` already exists — it's just not exposed via MCP.

**`projectionRegistry` is NOT injected in `QhorusMcpToolsBase`.** It is injected independently in `QhorusMcpTools` and `ReactiveQhorusMcpTools` as separate `@Inject` fields. No base class change needed.

Add to each concrete class:

```java
@Tool(name = "list_projections",
      description = "List all projection names registered with ProjectionRegistry. "
          + "Pass a name from this list as the projection_name argument to project_channel.")
public List<String> listProjections() {
    return projectionRegistry.registeredNames().stream().sorted().toList();
}
```

No description SPI method added (no confirmed consumers, YAGNI).

**ToolOverloadDiscoverabilityTest:** `listProjections` is a new name — no overload risk. Confirm no `public` non-`@Tool` overloads added.

### Tests

- Returns sorted list of registered projection names
- Returns empty list when no projections are registered (registry with no beans)

---

## #239 — project_channel max_messages (S) · `Closes #239`

### Semantic decision

Adding `max_messages` with `MessageQuery.builder().limit(maxMessages).build()` folds messages in ascending insertion order — this produces **the first N messages**, not the most recent N. The tool description previously said "200 most recent messages" — that was wrong. This spec owns "first N in insertion order" semantics. If most-recent-N semantics are ever needed, that requires a `tailLimit` query mode (descending DB scan, reversed before folding) and is a separate issue.

### Design

Add optional `max_messages` parameter to `project_channel`. When positive: use the scoped `ProjectionService.project(channelId, scope, projection)` overload with `MessageQuery.builder().limit(maxMessages).build()`. When null or non-positive: fold all messages (current behavior).

Update `projectAndRender()` in `QhorusMcpToolsBase` — add overload:
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

The existing zero-argument `projectAndRender(channelId, projection)` remains for call sites that don't pass a limit.

**ToolOverloadDiscoverabilityTest:** `projectAndRender()` is package-private — adding a second overload is safe and does not trigger the test.

Tool description: "Folds at most max_messages messages from the beginning of the channel history (ascending insertion order). Default 200. Pass null for the full history (may produce large output on busy channels)."

Remove the `-1` sentinel — `null` already means unlimited. Any non-positive value is treated as unlimited. No need for two representations.

**ReactiveQhorusMcpTools:** mirror the `max_messages` parameter. The `@Blocking` annotation already on `projectChannel` in `ReactiveQhorusMcpTools` must be retained (it calls the blocking `projectAndRender` from `QhorusMcpToolsBase`).

### Tests

- `max_messages=2` on a channel with 5 messages → result state reflects only messages 1–2
- `max_messages=null` → all messages folded
- `max_messages=-1` (or 0) → all messages folded (treated as unlimited)

---

## parent#163 — Oversight doc update (XS) · `Closes #163`

### Scope

Update `docs/repos/casehub-qhorus.md` in the `casehub/parent` repo. Change the normative channel layout table for `/oversight` from `allowedTypes=COMMAND,RESPONSE` to `deniedTypes=EVENT, allowedTypes=null`.

### PLATFORM.md verification

PLATFORM.md was read during brainstorming. Its Agent Communication Mesh section already reads: `oversight … deniedTypes = EVENT`. No change needed there.

### Rationale for the constraint change

`allowedTypes=COMMAND,RESPONSE` (the old allowlist) inadvertently blocked DONE, DECLINE, FAILURE, STATUS, HANDOFF, and QUERY from being sent on the oversight channel. In particular, it blocked DONE/DECLINE responses from the inbound normaliser — the human response path requires these types (see GE-20260519-28967d). The correct constraint is a denylist: oversight channels should admit all deliberative speech acts. Only EVENT (telemetry) is structurally excluded because it has no commitment effect, is invisible to `pollAfter` by default, and would pollute the governance record with operational noise.

---

## Cross-cutting notes

**ToolOverloadDiscoverabilityTest** applies to #244, #240, and #239. All new `@Tool` method names are fresh — no overload risk — but must be verified at implementation time. Never add `public` non-`@Tool` overloads sharing a name with a `@Tool` method.

**Testing discipline:** every item adds new behavior; each item's test section above covers the minimum. Follow `@TestTransaction` and named-datasource conventions documented in CLAUDE.md.
