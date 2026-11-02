# Channel Visibility Semantics — Design Spec

**Issue:** casehubio/qhorus#509
**Date:** 2026-11-02
**Branch:** issue-509-channel-visibility-semantics

## Overview

Adds a typed visibility dimension to Qhorus channels — PUBLIC, PRIVATE, DIRECT — enabling
the self-organising agent society (eidos#216) to use channels as agent mail inboxes, group
communication, and broadcast. Visibility is an organizational/access layer; the full normative
stack (speech acts, commitments, ledger, trust, attestation, watchdog) applies unchanged
regardless of visibility.

## Requirements (from #509)

- R1: `ChannelVisibility` enum — PUBLIC, PRIVATE, DIRECT
- R2: DIRECT channel convenience — `getOrCreateDirect(A, B)` with deterministic name
- R3: Group-bound channels — `createGroupChannel(groupId, groupName, memberIds, coordinatorId)`
- R4: Membership synchronisation — addMember, removeMember, getMembers on PRIVATE channels
- R5: Query by visibility and binding — findByVisibility, findDirectChannelsFor, findGroupChannel
- R6: Default semantics by visibility — creation-time defaults, caller can override

## Design

### 1. ChannelVisibility Enum

New enum in `api/channel/`:

```java
public enum ChannelVisibility {
    PUBLIC,
    PRIVATE,
    DIRECT
}
```

### 2. Channel Record Extension

New nullable field on `Channel` record:

```java
ChannelVisibility visibility   // nullable — null means legacy/unset, no visibility constraints
```

Position: after `participationPolicy` (last field in canonical constructor).

**ChannelEntity** gains:

```java
@Column(name = "visibility")
public String visibility;  // nullable, stored as enum name — same pattern as participationPolicy
```

**Flyway V59:** `ALTER TABLE channel ADD COLUMN visibility VARCHAR(20)`

**Migration strategy:** Existing channels have `visibility = null`. No retroactive assignment.
Visibility guards check `visibility != null` before enforcing. Legacy channels behave exactly
as they do today.

**Backward-compatible constructor:** New constructor delegates to canonical with
`visibility = null` — existing call sites unaffected.

### 3. ChannelCreateRequest Extension

New field on `ChannelCreateRequest`:

```java
ChannelVisibility visibility   // nullable
```

Builder gains `.visibility(ChannelVisibility v)` method.

### 4. Default Semantics (R6)

When `ChannelCreateRequest` specifies a visibility but the caller hasn't explicitly set
semantic or participationPolicy, defaults are applied at creation time:

| Visibility | Default Semantic | Default ParticipationPolicy |
|---|---|---|
| PUBLIC     | BROADCAST        | OPEN                        |
| PRIVATE    | APPEND           | ENFORCE                     |
| DIRECT     | APPEND           | ENFORCE                     |

Applied in `Channel.fromRequest()` — if visibility is non-null and the respective field is
null/default, substitute the visibility default. Caller-provided values always win.

This is purely a creation-time convenience. The visibility field doesn't enforce any invariant
on semantic or participationPolicy after creation.

### 5. VisibilityManager Interface

New interface in `api/channel/`:

```java
public interface VisibilityManager {

    // DIRECT channels (R2)
    FindOrCreateResult getOrCreateDirect(String participantA, String participantB);
    List<Channel> findDirectChannelsFor(String participantId);

    // Group channels (R3)
    Channel createGroupChannel(UUID groupId, String groupName,
                               List<String> memberIds, String coordinatorId);
    Optional<Channel> findGroupChannel(UUID groupId);

    // Membership sync (R4) — visibility-aware
    ChannelMembership addMember(UUID channelId, String memberId, MemberRole role);
    void removeMember(UUID channelId, String memberId);

    // Query (R5)
    List<Channel> findByVisibility(ChannelVisibility visibility);

    // Mutation
    Channel setVisibility(UUID channelId, ChannelVisibility visibility);
}
```

Implemented by `VisibilityService` in `runtime-core/channel/` (POJO). CDI wrapper
`CdiVisibilityService` in `runtime/cdi/` adds `@ApplicationScoped` + `@Inject`.

### 6. DIRECT Channel Mechanics (R2)

#### Naming

Deterministic sorted slug path: `dm/{sorted-a}/{sorted-b}`

- Sort participant IDs alphabetically (String.compareTo)
- Channel name: `dm/` + sorted-first + `/` + sorted-second
- Example: `getOrCreateDirect("researcher", "coordinator")` → `dm/coordinator/researcher`

#### Slug Validation

Both participant IDs must be valid slug segments (`ChannelSlugValidator.isValidSegment()`).
`getOrCreateDirect` validates both IDs before constructing the channel name. Throws
`IllegalArgumentException` with message:
`"Participant ID '...' is not a valid slug segment — must match [a-z][a-z0-9]*(-[a-z0-9]+)*"`

Instance IDs in the Qhorus ecosystem are human-readable slugs (e.g. `researcher-agent`,
`coordinator`). UUID-shaped instance IDs are already rejected by `isValidSegment()`.

#### Creation

```
getOrCreateDirect(A, B):
  1. Validate A, B are valid slug segments
  2. Sort A, B → (first, second)
  3. name = "dm/" + first + "/" + second
  4. request = ChannelCreateRequest.builder(name)
       .visibility(DIRECT)
       .semantic(APPEND)
       .participationPolicy(ENFORCE)
       .build()
  5. result = channelService.findOrCreate(request)
  6. If result.wasCreated():
       membershipService.join(channel.id, first, PARTICIPANT, tenancyId)
       membershipService.join(channel.id, second, PARTICIPANT, tenancyId)
  7. Return result
```

Deduplication is inherent — `findOrCreate` returns the existing channel for the deterministic
name. Step 6 only runs on first creation.

#### Membership Immutability

Guard in `ChannelMembershipService.join()`:

```
if (channel.visibility() == DIRECT && memberCount >= 2)
    throw IllegalStateException("DIRECT channels have immutable membership — exactly 2 members")
```

The guard checks **member count**, not just visibility — this allows the initial two joins in
`getOrCreateDirect` (count goes 0→1, 1→2) while blocking any subsequent join attempt.

Guard in `ChannelMembershipService.leave()`:

```
if (channel.visibility() == DIRECT)
    throw IllegalStateException("Cannot leave a DIRECT channel")
```

**ChannelGateway interaction:** `receiveHumanMessage()` auto-joins participants on first
interaction. The auto-join is wrapped in try-catch (existing code, line ~267-268) — the
DIRECT guard rejection is silently absorbed. No design change needed. The two participants
were already joined at creation time.

#### Inbox Query

`findDirectChannelsFor(participantId)` must be efficient — this is the agent inbox poll
hot path.

**Approach:** Add `ChannelVisibility visibility` filter to `ChannelQuery` (store layer). Add
a dedicated store method:

```java
// On ChannelMembershipStore or a new query interface
List<Channel> findChannelsByMemberAndVisibility(String memberId, ChannelVisibility visibility, String tenancyId);
```

JPA implementation: single JOIN query — `channel_membership JOIN channel WHERE memberId = ?
AND visibility = ? AND tenancyId = ?`. No N+1.

InMemory implementation: filter memberships → filter channels by visibility.

### 7. Group Channel Mechanics (R3)

#### Naming

`group/{groupName-slug}` — groupName slugified to a valid slug segment.

The caller is responsible for providing a slug-valid groupName (lowercase, hyphens). If
groupName is not a valid slug, `ChannelSlugValidator.validateSlugPath` rejects it at channel
creation time.

#### Binding

`metadata["boundGroupId"] = groupId.toString()` — UUID stored as string in the channel's
metadata map. Lookup via `ChannelQuery.byMetadata("boundGroupId", groupId.toString())`.

#### Collision Guard

If a channel with the computed name already exists but has a different `boundGroupId` in
its metadata, `createGroupChannel` throws `IllegalStateException`:
`"Channel 'group/research-team' already exists but is bound to a different group (existing: ..., requested: ...)"`.

Same `boundGroupId` → return existing channel (idempotent).

#### Creation

```
createGroupChannel(groupId, groupName, memberIds, coordinatorId):
  1. name = "group/" + groupName (must be slug-valid)
  2. request = ChannelCreateRequest.builder(name)
       .visibility(PRIVATE)
       .semantic(APPEND)
       .participationPolicy(ENFORCE)
       .metadata(Map.of("boundGroupId", groupId.toString()))
       .build()
  3. result = channelService.findOrCreate(request)
  4. If result.wasCreated():
       membershipService.join(channel.id, coordinatorId, MODERATOR, tenancyId)
       for each memberId in memberIds (excluding coordinatorId):
         membershipService.join(channel.id, memberId, PARTICIPANT, tenancyId)
     Else:
       Verify metadata["boundGroupId"] matches groupId → throw if mismatch
  5. Return channel
```

#### Membership Sync (R4)

`addMember(channelId, memberId, role)`:
- If visibility == DIRECT → throw (immutable membership)
- Otherwise → delegate to `membershipService.join(channelId, memberId, role, tenancyId)`

`removeMember(channelId, memberId)`:
- If visibility == DIRECT → throw (immutable membership)
- Otherwise → delegate to `membershipService.leave(channelId, memberId)`

For PRIVATE channels, the caller (eidos SocietyLifecycleManager) manages who can add/remove.
Qhorus doesn't enforce "only MODERATOR can add" — that's the society layer's responsibility.

### 8. Query Extensions (R5)

#### ChannelQuery (store layer) gains visibility filter

```java
private final ChannelVisibility visibility;  // new field

public static ChannelQuery byVisibility(ChannelVisibility v) { ... }
```

`matches()` adds: `if (visibility != null && visibility != ch.visibility()) return false;`

#### findGroupChannel

```java
Optional<Channel> findGroupChannel(UUID groupId) {
    return channelStore.scan(ChannelQuery.byMetadata("boundGroupId", groupId.toString()))
                       .stream().findFirst();
}
```

#### findByVisibility

```java
List<Channel> findByVisibility(ChannelVisibility visibility) {
    return channelStore.scan(ChannelQuery.byVisibility(visibility));
}
```

### 9. ChannelManager Extension

One new method:

```java
Channel setVisibility(UUID channelId, ChannelVisibility visibility);
```

On `ChannelService` — updates the entity field. No retroactive enforcement — changing a
channel's visibility doesn't alter its existing membership or settings. The guard in
`join()/leave()` reads visibility at call time.

### 10. REST API

| Method | Path | Body | Description |
|---|---|---|---|
| POST | `/api/channels/direct` | `{"participantA": "...", "participantB": "..."}` | getOrCreateDirect |
| GET | `/api/channels/direct/{participantId}` | — | findDirectChannelsFor |
| POST | `/api/channels/group` | `{"groupId": "...", "groupName": "...", "memberIds": [...], "coordinatorId": "..."}` | createGroupChannel |
| GET | `/api/channels/group/{groupId}` | — | findGroupChannel |
| PUT | `/api/channels/{id}/visibility` | `{"visibility": "PUBLIC\|PRIVATE\|DIRECT"}` | setVisibility |
| GET | `/api/channels?visibility=DIRECT` | — | Filter on existing list endpoint |
| POST | `/api/channels/{id}/members` | `{"memberId": "...", "role": "PARTICIPANT"}` | addMember |
| DELETE | `/api/channels/{id}/members/{memberId}` | — | removeMember |

### 11. Normative Stack Interaction

Visibility is purely organizational. The full normative stack applies unchanged:

- **Speech acts:** All 10 message types work identically on DIRECT/PRIVATE/PUBLIC channels
- **Commitments:** Full obligation lifecycle on every channel regardless of visibility
- **Ledger:** Every dispatch creates a `MessageLedgerEntry` — DM messages get full tamper evidence
- **Trust:** `ObligorTrustPolicy`, `RoutingBridge`, peer attestation — operate on message/channel level
- **Enforcement:** Channel protocols, type policies, `EnforcementMode` — configurable per channel
- **Watchdog:** Condition evaluation runs across all channels — DM loops still caught

`MessageService.dispatch()` does not check visibility. It checks `allowedWriters`,
`participationPolicy`, `rateLimits`, etc. — orthogonal to visibility.

### 12. Integration Contract with Eidos

The inbox view (combined DIRECT + PRIVATE channels with unread counts) is eidos's
responsibility. Qhorus provides the primitives:

- `findDirectChannelsFor(participantId)` — efficient single-query inbox for DMs
- `findByVisibility(PRIVATE)` + membership filter — group channels
- `UnreadCountProvider.getUnreadCounts(memberId, tenancyId)` — unread counts across all channels

Eidos composes these into the agent's unified inbox. This keeps Qhorus as a communication
primitive layer without opinionated inbox UX.

### 13. File Changes Summary

**New files:**
- `api/channel/ChannelVisibility.java` — enum
- `api/channel/VisibilityManager.java` — interface
- `runtime-core/channel/VisibilityService.java` — POJO implementation
- `runtime/cdi/CdiVisibilityService.java` — CDI wrapper
- `runtime/api/VisibilityResource.java` — REST endpoints

**Modified files:**
- `api/channel/Channel.java` — add `visibility` field + backward-compat constructor
- `api/channel/ChannelCreateRequest.java` — add `visibility` field + builder method
- `api/store/query/ChannelQuery.java` — add `visibility` filter
- `api/store/ChannelMembershipStore.java` — add `findChannelsByMemberAndVisibility()`
- `runtime/channel/ChannelEntity.java` — add `visibility` column + fromDomain/toDomain
- `runtime-core/channel/ChannelMembershipService.java` — add DIRECT guards in join()/leave()
- `runtime/channel/ChannelConfigBootstrap.java` — support `visibility` in YAML channel config
- `persistence-memory/` — InMemory implementations for new store methods
- `testing/` — test helpers for visibility-aware channel creation

**Migration:**
- `V59__channel_visibility.sql` — `ALTER TABLE channel ADD COLUMN visibility VARCHAR(20)`

## Trade-offs

- **DIRECT naming requires slug-valid instance IDs** (D1) — UUID-shaped IDs won't work.
  Instance IDs in the ecosystem are already human-readable slugs, so this is acceptable.
- **Visibility is opt-in, not retroactive** (D7) — `findByVisibility(PUBLIC)` won't return
  legacy channels. Consumers must handle null visibility in mixed deployments.
- **No MODERATOR-only membership enforcement** — Qhorus doesn't enforce "only MODERATOR can
  add members to PRIVATE channels". That's the society layer's responsibility.
- **Group name collision throws, not auto-disambiguates** (Issue 4) — deliberate; the caller
  should use unique group names per tenancy.

## References

- `api/channel/Channel.java` — 31-field record, enum patterns for participationPolicy/enforcementMode
- `api/channel/ChannelCreateRequest.java` — builder pattern, default semantics
- `api/channel/ChannelSlugValidator.java` — slug validation rules, `isValidSegment()`
- `api/channel/MembershipManager.java` — existing membership API (6 methods)
- `api/store/query/ChannelQuery.java` — store query with metadata filter
- `runtime-core/channel/ChannelMembershipService.java` — join/leave implementation
- `runtime/channel/ChannelEntity.java` — JPA entity, nullable enum-as-string pattern
- `runtime/gateway/ChannelGateway.java:267-268` — auto-join try-catch (DIRECT safety)
- casehubio/qhorus#509 — issue requirements R1-R6
- eidos#216 — self-organising agent society (consumer)
