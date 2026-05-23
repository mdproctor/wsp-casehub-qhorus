# Dispatch Enforcement Design

**Date:** 2026-05-23
**Issue:** casehubio/qhorus#188
**Branch:** `issue-185-review-findings`

---

## Problem

`MessageService.dispatch()` is too thin. After #184 replaced the positional `send()` API with the `MessageDispatch` builder, `A2AChannelBackend.receive()` was updated to call `dispatch()` directly. This exposed a structural flaw: enforcement of channel write policy — paused check, writer ACL, rate limiting, LAST_WRITE semantics, and fanOut — lives in `QhorusMcpTools.sendMessage()`, not in the service. Every caller that doesn't go through the MCP tool layer bypasses it silently.

The root cause is not A2A specifically. It is that enforcement belongs in `MessageService.dispatch()`, where it applies to every caller automatically and cannot be bypassed. Placing it in a caller (`QhorusMcpTools`) means every new caller (A2A, future webhooks, scheduled messages) must remember to apply it — and the system has already proven that it won't.

---

## Design

### 1. `MessageService.dispatch()` becomes the enforcement gate

Every channel write policy check moves into `dispatch()`. The enforcement sequence:

1. Fetch `Channel` entity (already done for `MessageTypePolicy`)
2. Paused check — `IllegalStateException` if `ch.paused`
3. ACL check via `AllowedWritersPolicy` — skipped for `EVENT` (telemetry always flows)
4. Rate limit check via `RateLimiter` — skipped for `EVENT`
5. LAST_WRITE detection — update-in-place path if `ch.semantic == LAST_WRITE` (moved from `QhorusMcpTools`)
6. Normal insert path (existing)
7. Rate limit recording after successful persistence
8. `ChannelGateway.fanOut()` after persistence

New injected dependencies on `MessageService`:
- `RateLimiter` (existing `@ApplicationScoped` bean in `runtime.channel`)
- `ChannelGateway` (existing `@ApplicationScoped` bean in `runtime.gateway`)
- `AllowedWritersPolicy` (new — see §2)
- `InstanceService` (existing — for capability tag lookup in ACL)

### 2. `AllowedWritersPolicy` — extracted ACL check

`isAllowedWriter()` is extracted from `QhorusMcpToolsBase` (where it is a `protected static`) into a new `@ApplicationScoped AllowedWritersPolicy` bean in `runtime.channel`. The logic is identical; the injection changes from static utility to CDI bean.

`MessageService.dispatch()` calls it with a **unified capability supplier** that works for all sender types:

```java
() -> {
    List<String> tags = new ArrayList<>(
        instanceService.findCapabilityTagsForInstance(dispatch.sender()));
    tags.add("role:" + dispatch.actorType().name().toLowerCase());
    return tags;
}
```

This resolves correctly for every sender type:

| Sender type | Returned tags | ACL behaviour |
|-------------|---------------|---------------|
| Registered Qhorus instance (agent, `capability:analysis`) | `["capability:analysis", "role:agent"]` | Exact ID, capability, and role entries all work |
| Unregistered A2A agent (`ActorType.AGENT`) | `["role:agent"]` | Role entries work; capability entries block (correct — external agents have no attested capabilities) |
| Human sender (`ActorType.HUMAN`) | `["role:human"]` | Role entries work |

`QhorusMcpToolsBase` injects `AllowedWritersPolicy` and delegates to it. After enforcement moves to `dispatch()`, the `QhorusMcpToolsBase.isAllowedWriter()` delegation site in `sendMessage()` is removed (the ACL check fires in `dispatch()` before `sendMessage()` does anything further).

### 3. CDI cycle elimination — `QhorusChannelBackend` decoupling

Moving `ChannelGateway` injection into `MessageService` would introduce a CDI cycle:

```
MessageService → ChannelGateway → QhorusChannelBackend → MessageService
```

`ChannelGateway.fanOut()` already skips `QhorusChannelBackend` at runtime (`if (entry.backend() == agentBackend) continue`), so there is no functional circularity — only a CDI wiring cycle. The fix is structural:

- `QhorusChannelBackend.post()` becomes an explicit no-op. The Javadoc states it was "called only from the test-only `ChannelGateway.post()` path — do NOT use from production fan-out." Since fanOut already skips it, the no-op is semantically correct.
- `MessageService` dependency removed from `QhorusChannelBackend`.
- `ChannelGateway.post()` (the package-private test-only method) is deleted. Tests that previously used it to inject messages call `messageService.dispatch()` directly.
- `qhorus#190` ("QhorusChannelBackend.post() can't handle DONE/RESPONSE/HANDOFF") is closed as resolved — the method is a no-op; the gap no longer exists.

After this change the injection graph is acyclic:

```
MessageService → ChannelGateway
MessageService → AllowedWritersPolicy
MessageService → InstanceService
MessageService → RateLimiter
ChannelGateway → QhorusChannelBackend   (no outbound deps)
```

### 4. `QhorusMcpTools.sendMessage()` simplification

The following are removed from `sendMessage()` (they now execute inside `dispatch()`):
- Channel paused check
- ACL check
- Rate limit check and recording
- LAST_WRITE overwrite path
- `channelGateway.fanOut()` call

What remains in `sendMessage()` is genuinely MCP-specific:
- Content and target format validation (`msgType.requiresContent()`, target prefix check)
- Artefact ref validation (batch query for unknown refs)
- Artefact auto-claim on send
- Artefact auto-release on commitment resolution (RESPONSE/DONE/DECLINE/FAILURE)
- Deadline parsing and assignment (`msg.deadline = Instant.now().plus(...)`)
- `subjectId` / `causedByEntryId` UUID parsing

`ReactiveQhorusMcpTools.sendMessage()` mirrors the same simplification.

### 5. `A2AChannelBackend.receive()` — no code changes

`A2AChannelBackend.receive()` already calls `messageService.dispatch()`. After this change, that call carries full enforcement. The Javadoc note ("rate limiting, allowed_writers ACL, and artefact lifecycle are currently bypassed — tracked in #188") is removed. No enforcement code is added to `A2AChannelBackend` itself.

Artefact lifecycle bypass is intentional and correct for A2A — the A2A protocol has no artefact ref passing. This is not a gap; it is the correct behaviour for an external protocol adapter.

---

## LAST_WRITE path in `dispatch()`

The LAST_WRITE update-in-place path currently in `QhorusMcpTools` moves into `MessageService.dispatch()` as a semantic branch:

```
if ch.semantic == LAST_WRITE:
    existing = messageStore.findLatestByChannel(channelId)
    if existing present:
        if existing.sender == dispatch.sender():
            → update existing message in place
            → record rate limit
            → return DispatchResult (ledger suppressed, parentReplyCount=0)
        else:
            → throw IllegalStateException (only current writer may update)
    // else: no existing message, fall through to normal insert
normal insert path
```

The `DispatchResult` for the LAST_WRITE overwrite case continues to carry `null` for ledger fields (`ledgerEntryId`, `subjectId`, `causedByEntryId`) and `parentReplyCount = 0`. This is the documented behaviour (no ledger write on overwrite). Issue #191 tracks the `parentReplyCount` correctness concern independently.

Rate limit recording happens after overwrite, same as after normal insert.

fanOut is NOT called on LAST_WRITE overwrite — the message is an in-place update, not a new event. This is a pre-existing decision; tracked in #189/#5.

---

## Out of scope

- **`ReactiveMessageService.send()` parity** — has the same enforcement gap; deferred because the service is `@Disabled` in all tests and not production-used. Tracked in #193.
- **LAST_WRITE fanOut** — pre-existing gap; tracked in #189/#5.
- **LAST_WRITE parentReplyCount** — tracked in #191.

---

## Testing

| Layer | What to test |
|-------|--------------|
| `AllowedWritersPolicy` unit | null ACL (open), exact match, capability match, role match, synthetic role for unregistered A2A sender, EVENT bypass |
| `MessageService.dispatch()` integration | paused rejection, ACL rejection, rate limit rejection, rate limit recording, fanOut fires (captured backend), LAST_WRITE overwrite, LAST_WRITE wrong-sender rejection |
| `QhorusMcpTools.sendMessage()` | artefact claim/release still fires; enforcement failures (paused, ACL, rate limit) come from dispatch with correct exception types |
| `A2AChannelBackend` | one integration test per enforcement concern (paused, ACL, rate limit) exercised through `A2AChannelBackend.receive()` end-to-end — confirms the wiring, not the logic |
| `QhorusChannelBackend` | `post()` is no-op; existing tests updated to remove `ChannelGateway.post()` usage |

---

## Platform coherence

**Boundary rules confirmed:**
- Enforcement moves into the service, not deeper into callers — consistent with "application service owns use case" principle.
- LAST_WRITE (a channel semantic) moves from `QhorusMcpTools` (MCP layer) into `MessageService` (service layer) — correct tier placement.
- `AllowedWritersPolicy` lands in `runtime.channel` alongside `RateLimiter` — consistent module grouping.
- No domain logic crosses into MCP or API layers.

**Protocol compliance:**
- PP-20260522-056cc2 (`no-jpa-entities-across-requires-new`): LAST_WRITE path skips `LedgerWriteService.record()` (REQUIRES_NEW); no entity crosses the boundary.
- PP-20260522-3dca14 (`message-dispatch-builder-validation`): builder validation unchanged; `build()` remains the single enforcement point for speech-act protocol invariants.
