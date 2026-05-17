# A2A Robust Integration — Design Spec
**Issue:** casehubio/qhorus#135  
**Date:** 2026-05-13  
**Depends on:** casehubio/ledger#75 (closed — `ActorTypeResolver` A2A rules shipped)

---

## What This Fixes

Four bugs in the current `A2AResource`:

| Bug | Problem | Fix |
|-----|---------|-----|
| 1 | `role:"agent"` classified as HUMAN in ledger | Fixed by ledger#75 — no work here |
| 2 | All inbound types hardcoded as QUERY | `A2AChannelBackend` resolves type from role |
| 3 | No outbound path — A2A caller must poll blindly | `getTask()` queries `CommitmentStore` for accurate durable state |
| 4 | `A2AResource` bypasses `ChannelGateway` | `A2AChannelBackend` registered as proper gateway backend |

Plus: the 6-step sender identity resolution chain for `role:"user"` (HUMAN vs AGENT vs SYSTEM).

---

## Foundational Change — `message.actorType`

This is a platform-wide correctness fix that qhorus#135 depends on. Actor type has always been re-derived from the sender string at ledger-write time, making it vulnerable to string convention drift. This change stores it explicitly at message-creation time.

### Flyway V8

```sql
ALTER TABLE message ADD COLUMN actor_type VARCHAR(10) NOT NULL DEFAULT 'HUMAN';
```

The DEFAULT is a safe value for the migration of existing dev/test rows only. All new writes set it explicitly.

### `Message.java`

```java
@Enumerated(EnumType.STRING)
@Column(name = "actor_type", nullable = false)
public ActorType actorType;
```

### `MessageService.send()` — single canonical signature

Three overloads collapsed into one. No overloads, no defaults:

```java
@Transactional
public Message send(UUID channelId, String sender, MessageType type, String content,
                    String correlationId, Long inReplyTo, String artefactRefs,
                    String target, ActorType actorType)
```

`actorType` is required. Every call site must declare what it knows about the actor.

### All call sites — what each passes

| Call site | `actorType` passed |
|-----------|-------------------|
| `QhorusMcpTools.sendMessage()` | `ActorTypeResolver.resolve(actorIdProvider.resolve(sender))` |
| `ReactiveQhorusMcpTools.sendMessage()` | Same |
| `ChannelGateway.receiveHumanMessage()` | `ActorType.HUMAN` |
| `ChannelGateway.receiveObserverSignal()` | `ActorType.HUMAN` |
| `QhorusChannelBackend.post()` (test-only path) | `message.actorType()` from `OutboundMessage` |
| `WatchdogEvaluationService` | `ActorType.SYSTEM`; sender changed `"watchdog"` → `"system:watchdog"` |
| `QhorusMcpTools` system EVENT sends (lines 458, 1278, 1303) | `ActorType.SYSTEM` |
| `A2AChannelBackend` (via `tools.sendMessage()`) | Implicit via structured sender string (see below) |
| All tests (~80 call sites) | `ActorTypeResolver.resolve(sender)` or explicit constant |

`QhorusMcpTools` injects `InstanceActorIdProvider` (new dependency). This is the only way to correctly classify Claudony session instanceIds (opaque strings like `"claudony-session-abc123"` that do not match `ActorTypeResolver` patterns but are enriched to persona strings by Claudony's `InstanceActorIdProvider`). The enrichment call (`actorIdProvider.resolve(sender)`) also happens in `LedgerWriteService` for `entry.actorId` — that is intentional and serves a different purpose (identity string vs type classification).

### `LedgerWriteService.record()` — simplified

```java
final String resolvedActorId = actorIdProvider.resolve(message.sender); // unchanged — for entry.actorId
entry.actorId   = resolvedActorId;
entry.actorType = message.actorType;  // explicit — no re-derivation
```

`ActorTypeResolver` import removed from `LedgerWriteService`. Same change in `ReactiveLedgerWriteService`.

### `fanOut()` OutboundMessage

`QhorusMcpTools.sendMessage()` uses `msg.actorType` (from the persisted entity) instead of re-calling `ActorTypeResolver.resolve(sender)`:

```java
channelGateway.fanOut(ch.id, new OutboundMessage(
    UUID.randomUUID(), sender, msgType, content, corrUuid, msg.actorType));
```

### `WatchdogEvaluationService` — latent bug corrected

The watchdog currently stores sender `"watchdog"` which classifies as HUMAN via the catch-all. This is wrong — the watchdog is a SYSTEM process. Fix: sender changed to `"system:watchdog"` (matches the `"system:*"` rule) and `ActorType.SYSTEM` passed explicitly. This is a correction exposed by the mandatory actorType change, not a regression.

---

## New Component: `A2AActorResolver`

**File:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AActorResolver.java`  
**Annotations:** `@ApplicationScoped`, `@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)`  
**Injects:** `InstanceService`

Kept separate from `A2AChannelBackend` so the 6 resolution cases can be unit-tested without any gateway or HTTP machinery.

### Resolution logic

```
resolve(String role, String actorTypeHeader, Map<String,String> metadata):

  // role:"agent" is unconditional — chain only fires for role:"user"
  if role.equals("agent") → return AGENT

  // Step 1: explicit header override
  if actorTypeHeader != null:
    try { return ActorType.valueOf(actorTypeHeader.toUpperCase()) }
    catch (IllegalArgumentException) { /* fall through — invalid value silently ignored */ }

  String agentId = metadata.getOrDefault("agentId", null)

  // Step 2: Instance registry lookup
  if agentId != null && instanceService.findByInstanceId(agentId).isPresent() → return AGENT

  // Step 3: Agent Card URL (A2A-native identity signal, survives relay)
  if metadata.get("agentCardUrl") is non-blank → return AGENT

  // Steps 4+5: delegate to ActorTypeResolver for persona format and system check
  if agentId != null:
    ActorType fromId = ActorTypeResolver.resolve(agentId)
    if fromId != HUMAN → return fromId   // AGENT (persona) or SYSTEM

  // Step 6: conservative default
  return HUMAN
```

**Key decisions:**
- `role:"agent"` is handled before the chain — unconditional, no header can override it to HUMAN
- Header parse failure is silent (fall-through) — an invalid header value is not a 400
- Steps 4+5 reuse `ActorTypeResolver.resolve()` to avoid duplicating the persona regex and "system:*" check
- `agentCardUrl` presence (not fetch) is sufficient — cheap string check, no HTTP call

---

## New Component: `A2AChannelBackend`

**File:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java`  
**Annotations:** `@ApplicationScoped`, `@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)`  
**Implements:** `ChannelBackend` (base interface — not `AgentChannelBackend` to avoid CDI ambiguity with `QhorusChannelBackend`)  
**Injects:** `QhorusMcpTools`, `ChannelGateway`, `A2AActorResolver`

### `ChannelBackend` contract

```java
public String backendId()  → "a2a"
public ActorType actorType() → ActorType.AGENT  // primary declared type; per-message routing is internal
public void open(ChannelRef, Map<String,String>)  → no-op
public void close(ChannelRef channel)             → registeredChannels.remove(channel.id())
```

### Registration

```java
private final Set<UUID> registeredChannels = ConcurrentHashMap.newKeySet();

public void ensureRegistered(UUID channelId, ChannelRef ref) {
    if (registeredChannels.add(channelId)) {   // atomic add-if-absent — thread-safe
        gateway.registerBackend(channelId, this, "agent");
        open(ref, Map.of());
    }
}
```

`registeredChannels.add()` uses `ConcurrentHashMap` semantics — only one thread wins the race and registers. Concurrent calls are safe. `close()` removes the channelId so a re-created channel can be registered again.

### Sender construction

The sender string encodes both identity and actor type so `ActorTypeResolver.resolve(sender)` — as called by `QhorusMcpTools.sendMessage()` via the InstanceActorIdProvider chain — produces the correct `ActorType` and the correct ledger entry:

```
buildSender(ActorType resolved, String agentId, String role):
  AGENT:
    if agentId != null && ActorTypeResolver.resolve(agentId) == AGENT → agentId  (identity preserved + classifies correctly)
    else → "agent"                                                                (generic AGENT marker)
  HUMAN:
    "human:" + (agentId != null ? agentId : role)                               (follows InboundNormaliser convention)
  SYSTEM:
    agentId != null ? agentId : "system"
```

This ensures `QhorusMcpTools.sendMessage(channelName, constructedSender, ...)` stores the correct `message.actorType` without A2AChannelBackend needing to bypass the tools layer.

### `receive()` — inbound routing

```java
public String receive(String channelName, A2AMessage msg, String actorTypeHeader) {
    Map<String, String> metadata = msg.metadata() != null ? msg.metadata() : Map.of();
    ActorType resolved = actorResolver.resolve(msg.role(), actorTypeHeader, metadata);
    String agentId = metadata.get("agentId");
    String sender = buildSender(resolved, agentId, msg.role());
    String type = "agent".equals(msg.role()) ? "response" : "query";
    String correlationId = (msg.taskId() != null && !msg.taskId().isBlank())
            ? msg.taskId() : UUID.randomUUID().toString();
    String text = extractText(msg);  // first text part

    tools.sendMessage(channelName, sender, type, text,
            correlationId, null, null, null, null);
    return correlationId;
}
```

**Why `tools.sendMessage()`:** gets the full pipeline — rate limiting (tracked #145), ledger write, fanOut, commitment tracking, artefact lifecycle (tracked #146). No duplication of pipeline logic.

**Message type mapping:**
- `role:"agent"` → `"response"` — the A2A agent is answering a delegated request
- `role:"user"` (any resolved type) → `"query"` — the initiating party is making a request
- Unknown roles → `"query"` (conservative; logs warning)

### `post()` — outbound hook

Called by `ChannelGateway.fanOut()` on a virtual thread for every message sent on registered channels. Currently a logging-only no-op; it is the correct hook for future SSE streaming (#147).

```java
@Override
public void post(ChannelRef channel, OutboundMessage message) {
    if (message.correlationId() != null) {
        LOG.debugf("A2A backend notified: channel=%s correlationId=%s type=%s",
                channel.name(), message.correlationId(), message.type());
    }
}
```

---

## `A2AResource` Refactor

### `A2AMessage` record — `metadata` field added

```java
public record A2AMessage(
        String role,
        List<A2APart> parts,
        String messageId,
        String taskId,
        String contextId,
        Map<String, String> metadata)  // nullable — A2A-native extension mechanism
```

Backward-compatible: absent JSON field deserialises to null, handled as `Map.of()` in the backend.

### `sendMessage()` — thin HTTP adapter

```java
@POST @Path("/message:send")
public Response sendMessage(SendMessageRequest request, @Context HttpHeaders headers) {
    // 1. validate (unchanged)
    // 2. look up channel
    Channel channel = channelService.findByName(msg.contextId()).orElse404();
    ChannelRef ref = new ChannelRef(channel.id, channel.name);
    // 3. register backend (idempotent)
    a2aBackend.ensureRegistered(channel.id, ref);
    // 4. extract actor-type header
    String actorTypeHeader = headers.getHeaderString("x-qhorus-actor-type");
    // 5. delegate
    String correlationId = a2aBackend.receive(channel.name, msg, actorTypeHeader);
    // 6. return
    return Response.ok(new SendMessageResponse(new Task(correlationId, msg.contextId(),
            new TaskStatus("submitted"), null))).build();
}
```

`A2AResource` no longer injects `QhorusMcpTools` directly.

### `getTask()` — CommitmentStore-based state

```java
@GET @Path("/tasks/{id}")
public Response getTask(@PathParam("id") String taskId) {
    // 1. Durable commitment-based state (preferred)
    Optional<Commitment> commitment = commitmentService.findByCorrelationId(taskId);
    if (commitment.isPresent()) {
        return Response.ok(new Task(taskId, channelNameFor(taskId),
                new TaskStatus(toA2AState(commitment.get().state)), null)).build();
    }
    // 2. Fallback: derive from message history (handles EVENT-only channels)
    List<Message> messages = messageService.findAllByCorrelationId(taskId);
    // ... existing logic unchanged
}

private String toA2AState(CommitmentState state) {
    return switch (state) {
        case FULFILLED, DELEGATED -> "completed";
        case FAILED, DECLINED, EXPIRED -> "failed";
        case ACKNOWLEDGED -> "working";
        case OPEN -> "submitted";
    };
}
```

**Why `CommitmentStore` is correct:** DONE/FAILURE/DECLINE messages transition commitment state at `MessageService.send()` time. The state is durable (DB-backed), survives restarts, and is semantically more accurate than `deriveState()` from the last message type. `CommitmentService.findByCorrelationId()` is a new public query method (3 lines delegating to the store).

`A2AResource` injected dependencies after refactor: `QhorusConfig`, `A2AChannelBackend`, `ChannelService`, `MessageService`, `CommitmentService`

### `ReactiveA2AResource`

Follows the same pattern. `ReactiveA2AChannelBackend` is the reactive mirror (same logic, `Uni<String>` return from `receive()`). In scope.

---

## `CommitmentService` — new query method

```java
public Optional<Commitment> findByCorrelationId(String correlationId) {
    return store.findByCorrelationId(correlationId);
}
```

A2AResource uses this via `CommitmentService` (service layer), not `CommitmentStore` directly.

---

## Platform Convention — `casehub-parent`

Update `docs/protocols/qhorus-actor-type-mapping.md` — remove the `(casehubio/ledger#75)` "pending" references from the A2A role mapping table (those are now implemented). Verify the A2A interop contract section is complete and accurate for the new 6-step chain.

---

## Testing Strategy

### `A2AActorResolverTest` — pure unit, no Quarkus, `InstanceService` mocked

| Test | Input | Expected |
|------|-------|---------|
| `role_agent_unconditional_isAgent` | role="agent", any header, any metadata | AGENT |
| `explicitHeader_agent_isAgent` | role="user", header="AGENT" | AGENT |
| `explicitHeader_human_isHuman` | role="user", header="HUMAN" | HUMAN |
| `invalidHeader_fallsThrough` | role="user", header="BANANA" | HUMAN (no exception) |
| `instanceRegistryLookup_isAgent` | role="user", metadata.agentId in registry | AGENT |
| `agentCardUrl_isAgent` | role="user", metadata.agentCardUrl non-blank | AGENT |
| `personaAgentId_isAgent` | role="user", metadata.agentId="claude:x@v1" | AGENT |
| `systemAgentId_isSystem` | role="user", metadata.agentId="system:sched" | SYSTEM |
| `noSignals_isHuman` | role="user", no header, empty metadata | HUMAN |
| `nullMetadata_isHuman` | role="user", metadata=null | HUMAN (no NPE) |

### `A2AChannelBackendTest` — pure unit, all deps mocked

- `ensureRegistered_calledTwiceSameChannel_registersOnce`
- `ensureRegistered_concurrentCalls_registersExactlyOnce`
- `receive_roleAgent_callsToolsWithResponseType`
- `receive_roleUserResolvedHuman_callsToolsWithQueryType_humanPrefixedSender`
- `receive_roleUserResolvedAgent_callsToolsWithQueryType_agentSender`
- `receive_noTaskId_generatesCorrelationId`
- `receive_providedTaskId_usedAsCorrelationId`
- `post_anyMessage_logsOnly_noException`
- `close_removesChannelFromRegisteredSet`

### `A2ASendMessageTest` — integration updates (`@QuarkusTest @TestProfile(A2AEnabledProfile.class)`)

Existing tests: verify they still pass (sender from role, correlationId, message created, 400 for missing fields).

New tests:
- `role_agent_senderIsAgent_typeIsResponse` — asserts message.sender="agent", messageType=RESPONSE
- `role_user_noSignals_senderIsHumanUser_typeIsQuery` — asserts sender="human:user", type=QUERY
- `role_user_headerAgent_senderIsAgent_typeIsQuery` — `x-qhorus-actor-type: AGENT` → sender="agent"
- `role_user_personaAgentId_senderIsPersona` — metadata.agentId="claude:x@v1" → sender="claude:x@v1"

### `A2AGetTaskTest` — integration updates

- `task_submitted_stateIsSubmitted` — QUERY with no response
- `task_committed_stateIsCompleted` — DONE sent → CommitmentState.FULFILLED → "completed"
- `task_failed_stateIsFailed` — FAILURE → "failed"
- `task_declined_stateIsFailed` — DECLINE → "failed"
- `task_notFound_returns404` — unchanged

### `MessageServiceTest` — update all ~80 call sites

All tests pass an explicit `ActorType`. Most use `ActorTypeResolver.resolve(sender)` for correct derivation; system actors pass `ActorType.SYSTEM`.

---

## Files Changed

| Module | File | Change |
|--------|------|--------|
| `runtime` | `V8__add_actor_type_to_message.sql` | New Flyway migration |
| `runtime` | `Message.java` | Add `actorType` field |
| `runtime` | `MessageService.java` | Merge 3 overloads → 1 canonical signature |
| `runtime` | `ReactiveMessageService.java` | Same |
| `runtime` | `QhorusMcpTools.java` | Inject `InstanceActorIdProvider`; pass actorType; use `msg.actorType` for fanOut |
| `runtime` | `ReactiveQhorusMcpTools.java` | Same |
| `runtime` | `ChannelGateway.java` | Pass `ActorType.HUMAN` to both `messageService.send()` calls |
| `runtime` | `QhorusChannelBackend.java` | Pass `message.actorType()` to `messageService.send()` |
| `runtime` | `WatchdogEvaluationService.java` | Sender `"watchdog"` → `"system:watchdog"`; pass `ActorType.SYSTEM` |
| `runtime` | `LedgerWriteService.java` | Use `message.actorType` directly; remove `ActorTypeResolver` call for type |
| `runtime` | `ReactiveLedgerWriteService.java` | Same |
| `runtime` | `CommitmentService.java` | Add `findByCorrelationId()` query method |
| `runtime` | `A2AActorResolver.java` | **New** |
| `runtime` | `A2AChannelBackend.java` | **New** |
| `runtime` | `A2AResource.java` | Refactor — thin adapter, CommitmentStore-based getTask() |
| `runtime` | `ReactiveA2AResource.java` | Same pattern |
| `runtime` (tests) | `A2AActorResolverTest.java` | **New** — 10 unit tests |
| `runtime` (tests) | `A2AChannelBackendTest.java` | **New** — 8 unit tests |
| `runtime` (tests) | `A2ASendMessageTest.java` | Update + 4 new tests |
| `runtime` (tests) | `A2AGetTaskTest.java` | Update + 4 new tests |
| `runtime` (tests) | ~80 test call sites of `messageService.send()` | Add `ActorType` parameter |
| `casehub-parent` | `docs/protocols/qhorus-actor-type-mapping.md` | Remove pending references; verify accuracy |

---

## Out of Scope — Tracked Issues

| Issue | What |
|-------|------|
| casehubio/qhorus#145 | Rate limiting for A2A inbound |
| casehubio/qhorus#146 | Artefact claim/release for A2A inbound |
| casehubio/qhorus#147 | SSE streaming (`TaskArtifactUpdateEvent`) |
| casehubio/qhorus#148 | LAST_WRITE channel semantics for A2A inbound |
