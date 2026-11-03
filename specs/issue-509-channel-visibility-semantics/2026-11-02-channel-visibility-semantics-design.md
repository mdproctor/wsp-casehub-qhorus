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

**Visibility declares organizational intent; `participationPolicy` and `allowedWriters` enforce
access. They are independent dimensions.** PUBLIC does not mean "anyone can write" — a PUBLIC
channel with ENFORCE policy is a gated broadcast: discoverable but membership-controlled.
PRIVATE does not mean "secret" — it means "membership is explicitly managed, not discoverable."
Visibility has zero impact on the message dispatch pipeline (`MessageService.dispatch()` does
not check visibility).

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

New nullable fields on `Channel` record:

```java
ChannelVisibility visibility   // nullable — null means legacy/unset, no visibility constraints
UUID boundGroupId              // nullable — non-null for group-bound channels
```

Position: `visibility` after `participationPolicy`, `boundGroupId` after `visibility` (last
fields in canonical constructor).

**ChannelEntity** gains:

```java
@Column(name = "visibility")
public String visibility;  // nullable, stored as enum name — same pattern as participationPolicy

@Column(name = "bound_group_id")
public UUID boundGroupId;  // nullable, indexed
```

**Flyway V61:**

```sql
ALTER TABLE channel ADD COLUMN visibility VARCHAR(20);
ALTER TABLE channel ADD COLUMN bound_group_id UUID;
CREATE INDEX idx_channel_visibility ON channel(visibility) WHERE visibility IS NOT NULL;
CREATE INDEX idx_channel_bound_group_id ON channel(bound_group_id) WHERE bound_group_id IS NOT NULL;
```

**Migration strategy:** Existing channels have `visibility = null` and `boundGroupId = null`.
No retroactive assignment. Visibility guards check `visibility != null` before enforcing.
Legacy channels behave exactly as they do today.

**Backward-compatible constructor:** New constructor delegates to canonical with
`visibility = null, boundGroupId = null` — existing call sites unaffected.

### 3. ChannelCreateRequest Extension

New fields on `ChannelCreateRequest`:

```java
ChannelVisibility visibility       // nullable
ParticipationPolicyMode participationPolicy  // nullable — enables caller override at creation
UUID boundGroupId                  // nullable — group binding, same pattern as spaceId
```

Builder gains `.visibility(ChannelVisibility v)`, `.participationPolicy(ParticipationPolicyMode p)`,
and `.boundGroupId(UUID v)` methods.

**Note:** `participationPolicy` was already on `Channel` (added in V58) and `ChannelEntity`, but
was never plumbed through the creation path. `Channel.fromRequest()` always passed `null`. This
spec adds it to `ChannelCreateRequest` so callers can set it at creation time. This is a
prerequisite for the visibility default mechanism to work correctly — without it, the claim
"caller-provided values always win" would be incoherent.

### 4. Default Semantics (R6)

When `ChannelCreateRequest` specifies a visibility but the caller hasn't explicitly set
semantic or participationPolicy, defaults are applied at creation time:

| Visibility | Default Semantic | Default ParticipationPolicy |
|---|---|---|
| PUBLIC     | BROADCAST        | OPEN                        |
| PRIVATE    | APPEND           | ENFORCE                     |
| DIRECT     | APPEND           | ENFORCE                     |

Applied in `Channel.fromRequest()`:

```java
public static Channel fromRequest(ChannelCreateRequest req, String tenancyId) {
    ChannelSemantic semantic = req.semantic();
    ParticipationPolicyMode policy = req.participationPolicy();

    if (req.visibility() != null) {
        if (semantic == null) {
            semantic = switch (req.visibility()) {
                case PUBLIC -> ChannelSemantic.BROADCAST;
                case PRIVATE, DIRECT -> ChannelSemantic.APPEND;
            };
        }
        if (policy == null) {
            policy = switch (req.visibility()) {
                case PUBLIC -> ParticipationPolicyMode.OPEN;
                case PRIVATE, DIRECT -> ParticipationPolicyMode.ENFORCE;
            };
        }
    } else {
        if (semantic == null) semantic = ChannelSemantic.APPEND;
    }
    // ... construct Channel with semantic, policy, req.visibility(), req.boundGroupId(), etc.
}
```

Caller-provided values always win — both `semantic` and `participationPolicy` can be set
explicitly on `ChannelCreateRequest` to override the visibility defaults.

This is purely a creation-time convenience. The visibility field doesn't enforce any invariant
on semantic or participationPolicy after creation.

**Meaningful visibility + semantic combinations:**

| Combination | Meaningful? | Notes |
|---|---|---|
| PUBLIC + BROADCAST | Yes (default) | Society-wide announcements, capability broadcasts |
| PUBLIC + APPEND | Yes | Open discussion forum |
| PRIVATE + APPEND | Yes (default) | Group internal comms |
| PRIVATE + BARRIER | Yes | Consensus-requiring group decisions |
| DIRECT + APPEND | Yes (default) | Agent-to-agent messaging |
| DIRECT + BROADCAST | Unusual | STATUS+EVENT only between 2 agents; use a regular channel |

No runtime validation of combinations — they are independent dimensions. The defaults steer
callers toward the sensible combinations.

### 5. API Decomposition

Visibility operations are distributed across Qhorus's existing API interfaces following the
established decomposition pattern:

**ChannelManager** (channel lifecycle + property mutation) gains:

```java
Channel setVisibility(UUID channelId, ChannelVisibility visibility);
```

This follows the established pattern: `setParticipationPolicy`, `setEnforcementMode`,
`setAllowedWriters`, etc. are all on ChannelManager.

**ChannelReader** (queries) gains:

```java
List<Channel> findByVisibility(ChannelVisibility visibility);
```

This follows the established pattern: `findByName`, `scan`, `listAll` are all on ChannelReader.

**VisibilityManager** — new interface for composite visibility-typed channel operations:

```java
public interface VisibilityManager {

    // DIRECT channels (R2)
    FindOrCreateResult getOrCreateDirect(String participantA, String participantB);
    List<Channel> findDirectChannelsFor(String participantId);

    // Group channels (R3)
    Channel createGroupChannel(UUID groupId, String groupName,
                               List<String> memberIds, String coordinatorId);
    Optional<Channel> findGroupChannel(UUID groupId);
}
```

This is a focused 4-method interface for composite operations (naming + creation + membership
setup as atomic workflows). It does not mix in simple property mutations, queries, or
membership CRUD — those belong on their established interfaces.

Implemented by `VisibilityService` in `runtime-core/channel/` (POJO). CDI wrapper
`CdiVisibilityService` in `runtime/cdi/` adds `@ApplicationScoped` + `@Inject`.

**Membership operations (R4):** The existing `MembershipManager.join()/leave()` path handles
all membership mutations, with visibility guards enforced in `ChannelMembershipService`
(§6). No separate addMember/removeMember methods — the enforcement boundary is in the
core membership service, preventing bypass through alternative API paths.

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
     Else:
       // Existing channel — verify DIRECT visibility
       if (channel.visibility() != DIRECT)
           throw IllegalStateException("Channel '" + name + "' exists but is not a DIRECT channel")
  7. Return result
```

Deduplication is inherent — `findOrCreate` returns the existing channel for the deterministic
name. Step 6 only sets up membership on first creation. The `Else` branch verifies the existing
channel has DIRECT visibility — this catches naming collisions with legacy channels that happen
to share the `dm/` prefix.

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

**Dependency change:** `ChannelMembershipService` gains a `ChannelStore` dependency (the store
SPI interface, not `ChannelService`) to load the channel for visibility checks. The dependency
graph remains acyclic: `ChannelService` depends on `ChannelMembershipStore` (store SPI), not
`ChannelMembershipService`; `ChannelMembershipService` depends on `ChannelStore` (store SPI),
not `ChannelService`. No circular injection risk. `ChannelService` → `ChannelGateway` →
`ChannelMembershipService` → `ChannelStore` is a DAG.

**Concurrency:** The read-then-check pattern for the DIRECT member count guard has a TOCTOU
race under concurrent joins. The happy path (`getOrCreateDirect`) is single-threaded, and
`ChannelGateway` auto-join failures are silently caught, but the guard must be safe against
arbitrary concurrent callers. JPA implementations must use `SELECT ... FOR UPDATE` on the
channel row (or equivalent pessimistic lock) before the count check to serialise concurrent
membership mutations on DIRECT channels. The InMemory implementation uses synchronized access.

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
AND visibility = ? AND tenancyId = ?`. The `idx_channel_visibility` partial index supports
this path. No N+1.

InMemory implementation: filter memberships → filter channels by visibility.

### 7. Group Channel Mechanics (R3)

#### Naming

`group/{groupName-slug}` — groupName slugified to a valid slug segment.

The caller is responsible for providing a slug-valid groupName (lowercase, hyphens). If
groupName is not a valid slug, `ChannelSlugValidator.validateSlugPath` rejects it at channel
creation time.

#### Binding

`boundGroupId` is a first-class nullable `UUID` field on `Channel` — same pattern as `spaceId`.
Lookup via `ChannelQuery.byBoundGroupId(groupId)` which maps to an indexed column.

This gives:
- **Indexed lookup** — `findGroupChannel` uses `WHERE bound_group_id = ?`, not JSON string search
- **Type safety** — `UUID`, not `String` buried in a `Map<String, String>`
- **Referential integrity** — visible in the schema, amenable to foreign key constraints
- **Clear collision semantics** — `channel.boundGroupId() == null` vs `!= requestedGroupId`
  is unambiguous, no confusion about "bound to different group" vs "not bound at all"

#### Collision Guard

If a channel with the computed name already exists but has a different `boundGroupId`,
`createGroupChannel` throws `IllegalStateException`:

- `boundGroupId == null` (channel exists but not group-bound): `"Channel 'group/research-team'
  exists but is not bound to any group — cannot bind to group (requested: ...)"`
- `boundGroupId != requestedGroupId` (bound to a different group): `"Channel
  'group/research-team' is bound to a different group (existing: ..., requested: ...)"`

Same `boundGroupId` → return existing channel (idempotent).

#### Creation

```
createGroupChannel(groupId, groupName, memberIds, coordinatorId):
  1. name = "group/" + groupName (must be slug-valid)
  2. request = ChannelCreateRequest.builder(name)
       .visibility(PRIVATE)
       .semantic(APPEND)
       .participationPolicy(ENFORCE)
       .boundGroupId(groupId)
       .build()
  3. result = channelService.findOrCreate(request)
  4. If result.wasCreated():
       membershipService.join(channel.id, coordinatorId, MODERATOR, tenancyId)
       for each memberId in memberIds (excluding coordinatorId):
         membershipService.join(channel.id, memberId, PARTICIPANT, tenancyId)
     Else:
       Verify channel.boundGroupId() matches groupId → throw if mismatch or null
  5. Return channel
```

#### Membership (R4)

For PRIVATE/GROUP channels, the caller (eidos SocietyLifecycleManager) manages who can
add/remove members through the existing `MembershipManager.join()/leave()` API.

Qhorus doesn't enforce "only MODERATOR can add members to PRIVATE channels" — that's the
society layer's responsibility. **Design decision (deviation from #509 R4):** R4 says
"Adding to a PRIVATE channel succeeds if caller has MODERATOR role." Qhorus is a communication
primitive layer, not an authorization layer. MODERATOR role enforcement belongs in the consuming
society layer (eidos) which has the organizational context to make access decisions.

### 8. Query Extensions (R5)

#### ChannelQuery (store layer) gains visibility and boundGroupId filters

```java
private final ChannelVisibility visibility;   // new field
private final UUID boundGroupId;               // new field

public static ChannelQuery byVisibility(ChannelVisibility v) { ... }
public static ChannelQuery byBoundGroupId(UUID groupId) { ... }
```

`matches()` adds:
```java
if (visibility != null && visibility != ch.visibility()) return false;
if (boundGroupId != null && !boundGroupId.equals(ch.boundGroupId())) return false;
```

#### findGroupChannel

```java
Optional<Channel> findGroupChannel(UUID groupId) {
    return channelStore.scan(ChannelQuery.byBoundGroupId(groupId))
                       .stream().findFirst();
}
```

#### findByVisibility (on ChannelReader)

```java
List<Channel> findByVisibility(ChannelVisibility visibility) {
    return channelStore.scan(ChannelQuery.byVisibility(visibility));
}
```

### 9. setVisibility Behavior

`ChannelManager.setVisibility(UUID channelId, ChannelVisibility visibility)` — implemented in
`ChannelService`, following the same pattern as `setParticipationPolicy`, `setEnforcementMode`,
etc.

**Transition rules:**

| From | To | Guard |
|---|---|---|
| Any | DIRECT | Must have exactly 2 members — throws `IllegalStateException` otherwise |
| DIRECT | null | Allowed — removes membership immutability (2-member invariant no longer enforced) |
| DIRECT | PUBLIC/PRIVATE | Allowed — membership immutability removed, new visibility rules apply |
| null/PUBLIC/PRIVATE | null/PUBLIC/PRIVATE | Always allowed |

No retroactive enforcement — changing a channel's visibility doesn't alter its existing
membership or settings. The guard in `join()/leave()` reads visibility at call time.

### 10. REST API

| Method | Path | Body | Description |
|---|---|---|---|
| POST | `/api/channels/direct` | `{"participantA": "...", "participantB": "..."}` | getOrCreateDirect |
| GET | `/api/channels/direct/{participantId}` | — | findDirectChannelsFor |
| POST | `/api/channels/group` | `{"groupId": "...", "groupName": "...", "memberIds": [...], "coordinatorId": "..."}` | createGroupChannel |
| GET | `/api/channels/group/{groupId}` | — | findGroupChannel |
| PUT | `/api/channels/{id}/visibility` | `{"visibility": "PUBLIC\|PRIVATE\|DIRECT"}` | setVisibility (via ChannelManager) |
| GET | `/api/channels?visibility=DIRECT` | — | Filter on existing list endpoint (via ChannelReader) |

Membership mutations use the existing `/api/channels/{id}/members` endpoints backed by
`MembershipManager`.

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

### 13. BroadcastMembershipManager Integration

Existing `broadcast/` channels created by `BroadcastMembershipManager` are architecturally
PUBLIC: discoverable, auto-join on capability registration, BROADCAST semantic. Today they
have `visibility = null` since the field doesn't exist yet.

**This spec includes updating BroadcastMembershipManager** to set `visibility = PUBLIC` on
the `ChannelCreateRequest` it builds. This is a one-line change:

```java
ChannelCreateRequest req = ChannelCreateRequest.builder(BROADCAST_PREFIX + capability)
        .semantic(ChannelSemantic.BROADCAST)
        .visibility(ChannelVisibility.PUBLIC)     // ← new
        .description("Broadcast channel for capability: " + capability)
        .allowedTypes(Set.of(MessageType.STATUS, MessageType.EVENT))
        .build();
```

New broadcast channels will have `visibility = PUBLIC`. Existing broadcast channels retain
`visibility = null` — they can be retroactively classified via `setVisibility(PUBLIC)` but
this is not part of this spec's migration.

`findByVisibility(PUBLIC)` returns only channels with explicit `visibility = PUBLIC`. It does
NOT return legacy channels with `visibility = null`. This is deliberate — null means "not
classified," not "implicitly PUBLIC."

### 14. File Changes Summary

**New files:**
- `api/channel/ChannelVisibility.java` — enum
- `api/channel/VisibilityManager.java` — interface (4 methods: DIRECT/GROUP lifecycle)
- `runtime-core/channel/VisibilityService.java` — POJO implementation
- `runtime/cdi/CdiVisibilityService.java` — CDI wrapper
- `runtime/api/VisibilityResource.java` — REST endpoints

**Modified files:**
- `api/channel/Channel.java` — add `visibility` + `boundGroupId` fields + backward-compat constructor
- `api/channel/ChannelCreateRequest.java` — add `visibility` + `participationPolicy` + `boundGroupId` fields + builder methods
- `api/channel/ChannelManager.java` — add `setVisibility()`
- `api/channel/ChannelReader.java` — add `findByVisibility()`
- `api/store/query/ChannelQuery.java` — add `visibility` + `boundGroupId` filters
- `api/store/ChannelMembershipStore.java` — add `findChannelsByMemberAndVisibility()`
- `runtime/channel/ChannelEntity.java` — add `visibility` + `bound_group_id` columns + fromDomain/toDomain
- `runtime-core/channel/ChannelMembershipService.java` — add DIRECT guards in join()/leave(), add ChannelStore dependency
- `runtime-core/channel/ChannelService.java` — implement `setVisibility()` + `findByVisibility()`
- `runtime-core/broadcast/BroadcastMembershipManager.java` — add `.visibility(PUBLIC)` to channel creation
- `runtime/channel/ChannelConfigBootstrap.java` — support `visibility` + `boundGroupId` in YAML channel config
- `persistence-memory/` — InMemory implementations for new store methods
- `testing/` — test helpers for visibility-aware channel creation

**Migration:**
- `V61__channel_visibility.sql` — `ALTER TABLE channel ADD COLUMN visibility VARCHAR(20)`,
  `ALTER TABLE channel ADD COLUMN bound_group_id UUID`, partial indexes on both

## Design Decisions

### D1: DIRECT naming requires slug-valid instance IDs
UUID-shaped IDs won't work. Instance IDs in the ecosystem are already human-readable slugs,
so this is acceptable.

### D2: Visibility is opt-in, not retroactive
`findByVisibility(PUBLIC)` won't return legacy channels. Consumers must handle null visibility
in mixed deployments. **Deviation from #509:** The issue says "Channels without a visibility
field default to PUBLIC behaviour." The spec treats null as "unclassified/legacy" — a different
semantic. null ≠ PUBLIC is architecturally cleaner because it distinguishes between "I haven't
been told what I am" and "I am PUBLIC." This avoids the false implication that all legacy
channels are PUBLIC when many may be private-in-practice.

### D3: No MODERATOR-only membership enforcement
Qhorus doesn't enforce "only MODERATOR can add members to PRIVATE channels" (deviation from
#509 R4). That's the society layer's responsibility. Qhorus is a communication primitive layer,
not an authorization layer. The MODERATOR role exists and is stored, but enforcement of
role-based access control belongs in the consuming application (eidos) which has the
organizational context.

### D4: Group name collision throws, not auto-disambiguates
The caller should use unique group names per tenancy.

### D5: Namespace prefixes are conventions, not enforced reservations
The `dm/` and `group/` prefixes are naming conventions used by `getOrCreateDirect` and
`createGroupChannel`. Any caller CAN still create channels with these prefixes via
`ChannelManager.create()`. The collision guards in the convenience methods handle conflicts
at lookup time (§6 step 6 Else branch, §7 collision guard). Prefix enforcement in
`ChannelManager.create()` would couple the core channel creation path to visibility knowledge
— that coupling isn't justified for a naming convention.

### D6: Visibility has no impact on the message dispatch pipeline
Visibility is an organizational label that drives membership guards and query convenience.
It does NOT control who can see messages or who can write. `MessageService.dispatch()` does
not check visibility — it checks `allowedWriters`, `participationPolicy`, and rate limits.
The name "visibility" follows industry convention (Slack, Discord, Teams) where it describes
channel discoverability and membership management intent, not message-level access control.

### D7: Visibility + semantic are independent dimensions
No runtime validation of visibility/semantic combinations. The defaults steer callers toward
sensible pairings (§4 table). Adding a validation matrix creates coupling between two
independent dimensions that will need maintenance as new semantics or visibility types are
added. Unusual combinations (e.g., DIRECT + BROADCAST) are permitted but documented as
atypical.

### D8: TenancyId via CurrentPrincipal
`VisibilityService` obtains `tenancyId` from `CurrentPrincipal.tenancyId()` — the same pattern
used by `ChannelMembershipService`. The 4-arg `join(channelId, memberId, role, tenancyId)`
method on `ChannelMembershipService` receives the tenancyId from the `VisibilityService` caller.

### D9: CHANNELS.md taxonomy update
CHANNELS.md defines 8 discriminator dimensions. Visibility is a 9th dimension:
**Visibility** (PUBLIC · PRIVATE · DIRECT · null/legacy). This should be added as part of
the implementation to keep the taxonomy current.

## Trade-offs

- **DIRECT naming requires slug-valid instance IDs** (D1) — UUID-shaped IDs won't work.
  Instance IDs in the ecosystem are already human-readable slugs, so this is acceptable.
- **Visibility is opt-in, not retroactive** (D2) — `findByVisibility(PUBLIC)` won't return
  legacy channels. Consumers must handle null visibility in mixed deployments.
- **No MODERATOR-only membership enforcement** (D3) — Qhorus doesn't enforce "only MODERATOR
  can add members to PRIVATE channels". That's the society layer's responsibility.
- **Group name collision throws, not auto-disambiguates** (D4) — deliberate; the caller
  should use unique group names per tenancy.

## References

- `api/channel/Channel.java` — 30-field record (32 with visibility + boundGroupId), enum patterns for participationPolicy/enforcementMode
- `api/channel/ChannelCreateRequest.java` — builder pattern, default semantics
- `api/channel/ChannelSlugValidator.java` — slug validation rules, `isValidSegment()`
- `api/channel/MembershipManager.java` — existing membership API (6 methods)
- `api/channel/ChannelManager.java` — channel lifecycle + property mutations (21 methods, gains setVisibility)
- `api/channel/ChannelReader.java` — channel queries (6 methods, gains findByVisibility)
- `api/store/query/ChannelQuery.java` — store query with metadata, visibility, boundGroupId filters
- `runtime-core/channel/ChannelMembershipService.java` — join/leave implementation, gains ChannelStore dependency for visibility guards
- `runtime-core/broadcast/BroadcastMembershipManager.java` — broadcast channel lifecycle, gains visibility = PUBLIC
- `runtime/channel/ChannelEntity.java` — JPA entity, nullable enum-as-string pattern
- `runtime/gateway/ChannelGateway.java:267-268` — auto-join try-catch (DIRECT safety)
- casehubio/qhorus#509 — issue requirements R1-R6
- eidos#216 — self-organising agent society (consumer)
