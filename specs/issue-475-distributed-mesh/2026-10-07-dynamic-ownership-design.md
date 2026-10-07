# Dynamic Ownership Heuristics — Design Specification

**Issue:** casehubio/qhorus#475
**Date:** 2026-10-07
**Status:** Draft
**Depends on:** [2026-10-06-distributed-mesh-consolidated.md] (§2 Level 4, §4 Write Model, D5, D11, D18, D20)

## 1. What This Is

A write-frequency-based ownership heuristic for the qhorus relay cluster.
Instead of the static hash ring assigning channels to nodes deterministically,
relays earn ownership of the channels they write to most frequently. This
minimises proxy hops — the relay closest to the writing agents handles
writes locally without cross-node dispatch.

| Strategy | How ownership is determined | When to use |
|----------|---------------------------|-------------|
| `hash-ring` | SHA-256 hash of channel UUID → consistent ring | Predictable, even distribution regardless of workload |
| `dynamic` | Write-frequency tracking with hash ring fallback | Write-locality optimisation when agents cluster on specific relays |
| `none` | No routing — all writes execute locally with DB locks | Level 3 (relays without channel ownership) |

Same binary, same config prefix (`casehub.qhorus.relay.routing`). The
`dynamic` strategy layers on top of the hash ring — it never replaces it.
Channels with no write history or no dominant writer always fall back to
their hash ring assignment.

## 2. Architecture

### Component overview

```
WriteRoutingDecorator
  │
  ├── recordWrite(channelId) → WriteFrequencyTracker
  │
  └── owner(channelId) → ClusterManager
                            └── DynamicOwnershipResolver
                                  ├── ownership map (dynamic claims) → NodeInfo
                                  └── fallback: ConsistentHashRing → NodeInfo
```

Four new components, all in the `cluster` module:

| Component | Responsibility |
|-----------|---------------|
| `WriteFrequencyTracker` | Per-channel sliding window write counter. Tracks originating writes on this node. |
| `DynamicOwnershipResolver` | Layered ownership resolution: dynamic claims first, hash ring fallback. |
| `OwnershipEvaluator` | Periodic evaluation: claims channels when local writes exceed 2x the current owner's. |
| `OwnershipClaim` | Record carried in `HeartbeatResponse` — channel ID + write count. |

### WriteFrequencyTracker

`@ApplicationScoped` CDI-free POJO with `Clock` injection. Maintains
a `ConcurrentHashMap<UUID, BucketWindow>` keyed by channel ID.

Each `BucketWindow` is a fixed-size circular array of `AtomicLong`
counters (default: 10 buckets × 30 seconds = 5-minute window):

```
Bucket layout (10 × 30s = 5 min window):

  [0] [1] [2] [3] [4] [5] [6] [7] [8] [9]
                              ^
                           current

recordWrite(): increment bucket[current]
getCount():    sum all buckets
rotate():      advance current, clear next bucket
```

- `recordWrite(UUID channelId)` — O(1), lock-free (`AtomicLong.incrementAndGet`)
- `getCount(UUID channelId)` — O(buckets), sums non-expired buckets
- `rotate()` — called by `OwnershipEvaluator` on each tick; only advances the bucket pointer when the current bucket's duration has elapsed (30s default). Multiple evaluator ticks (10s) may pass without rotation.
- `getActiveChannels()` — returns channel IDs with non-zero counts (for evaluation scan)

Memory footprint: 10 longs × 8 bytes = 80 bytes per channel. At 10,000
channels (extreme), that's 800 KB — negligible.

Channels with zero counts across all buckets are eligible for eviction.
The evaluator prunes them on each tick.

### DynamicOwnershipResolver

Wraps a `ConsistentHashRing` and a `ConcurrentHashMap<UUID, OwnershipClaim>`
(the ownership map, populated from heartbeats).

```java
NodeInfo owner(UUID channelId) {
    OwnershipClaim claim = ownershipMap.get(channelId);
    if (claim != null) {
        return clusterManager.nodeInfo(claim.nodeId());
    }
    return hashRing.owner(channelId);  // fallback
}
```

The resolver does not evaluate — it only looks up. The evaluator decides
when to add/remove entries in the ownership map.

### OwnershipEvaluator

`@ApplicationScoped` with `@Scheduled` tick (default 10 seconds).

On each tick:
1. Call `tracker.rotate()` to advance sliding window buckets
2. Scan `tracker.getActiveChannels()`
3. For each channel:
   a. Get local write count from tracker
   b. Get current owner from resolver
   c. If current owner is this node — check relinquishment (count == 0 → remove claim)
   d. If current owner is remote — get owner's write count from heartbeat claims
   e. If `localCount > hysteresisRatio * ownerCount` AND `localCount >= minClaimWrites` → claim
4. Update local ownership map with claims/relinquishments
5. `ClusterManager` exposes claims for heartbeat serialisation

### Heartbeat integration

`HeartbeatResponse` gains a new field:

```java
record HeartbeatResponse(
    String nodeId,
    Instant timestamp,
    String ringHash,
    String status,
    Map<UUID, OwnershipClaim> ownershipClaims  // NEW
) {}

record OwnershipClaim(String nodeId, long writeCount) {}
```

On receiving a heartbeat response, `HeartbeatService` calls
`clusterManager.updateRemoteOwnership(peerId, claims)` to merge the
peer's claims into the ownership map. When a peer transitions to DEAD,
all its claims are removed (channels revert to hash ring).

On restart, the ownership map starts empty (all channels → hash ring).
After one heartbeat round (~3s), all surviving peers' claims are
reconstructed. The local evaluator begins earning ownership from live
write patterns independently.

## 3. Ownership lifecycle

### Bootstrap

```
Startup
  │
  ├── ownershipMap = empty
  ├── all channels → hash ring (deterministic)
  │
  ├── first heartbeat round (~3s)
  │     └── peers' claims populate ownershipMap
  │
  └── evaluator starts ticking (10s)
        └── local writes accumulate → claims earned
```

### Claim flow

```
Agent writes to channel X via relay A
  │
  ├── WriteRoutingDecorator.dispatch()
  │     ├── tracker.recordWrite(channelX)
  │     └── owner = resolver.owner(channelX) → hash ring says node B
  │           └── proxy to node B (or fallback to local)
  │
  ├── ... writes accumulate over 5-minute window ...
  │
  └── OwnershipEvaluator tick
        ├── localCount = tracker.getCount(channelX) → 100
        ├── currentOwner = node B (hash ring)
        ├── ownerCount = heartbeat claims from B → 5 (B's own agents write rarely)
        ├── 100 > 2 × 5 AND 100 >= minClaimWrites(5) → CLAIM
        └── ownershipMap.put(channelX, OwnershipClaim("A", 100))

Next heartbeat:
  └── A's HeartbeatResponse includes channelX claim
        └── all peers update: channelX owned by A

Subsequent writes:
  └── resolver.owner(channelX) → A (dynamic claim)
        └── local dispatch — no proxy hop
```

### Relinquishment flow

```
Agent disconnects from relay A (or stops writing to channel X)
  │
  ├── ... sliding window expires (5 min) ...
  │
  └── OwnershipEvaluator tick
        ├── localCount = tracker.getCount(channelX) → 0
        ├── currentOwner = this node (A)
        └── count == 0 → RELINQUISH
              └── ownershipMap.remove(channelX)

Next heartbeat:
  └── A's claims no longer include channelX
        └── peers remove channelX from A's claim set

Subsequent writes:
  └── resolver.owner(channelX) → hash ring fallback
```

### Transfer flow (owner change)

```
Relay B starts getting heavy writes to channel X (currently owned by A)
  │
  ├── OwnershipEvaluator tick on B
  │     ├── localCount(B) = 80
  │     ├── ownerCount(A) = 10 (from A's heartbeat claim)
  │     ├── 80 > 2 × 10 → CLAIM
  │     └── ownershipMap updated: channelX → B
  │
  └── Next heartbeat round
        ├── B advertises channelX claim
        ├── A sees B's claim — A does NOT contest (A's count is 10 < B's 80)
        └── all peers route channelX to B
```

**Conflict resolution:** Two nodes cannot both validly claim the same
channel simultaneously. Node A claims with count 100, node B claims with
count 30. B sees A's claim (count 100) in the next heartbeat — 30 < 2×100,
so B relinquishes. The 2x threshold plus heartbeat propagation ensures
convergence within one heartbeat round. During the brief overlap, DB locks
(D11, D20) guarantee correctness.

**Edge case — sporadic writers:** A channel receiving one write every
6 minutes oscillates between dynamic ownership (the write triggers a
claim) and hash ring fallback (the 5-minute window expires before the
next write). This is harmless: the `minClaimWrites` threshold (default 5)
prevents a single write from triggering a claim. A channel must
sustain at least 5 writes within the window to earn dynamic ownership.
Below that rate, it stays on the hash ring permanently.

## 4. ClusterManager changes

Current `owner()` delegates directly to `ConsistentHashRing`:

```java
// Before
public NodeInfo owner(UUID channelId) {
    return ring.get().owner(channelId);  // always hash ring
}
```

After:

```java
// After
public NodeInfo owner(UUID channelId) {
    if (resolver != null) {
        return resolver.owner(channelId);  // dynamic + hash ring fallback
    }
    return ring.get().owner(channelId);   // hash-ring-only mode
}
```

The `resolver` is null when `routing=hash-ring` (static mode). When
`routing=dynamic`, the `RelayProducer` creates and injects the resolver.

New methods on `ClusterManager`:

| Method | Purpose |
|--------|---------|
| `claimChannel(UUID, long writeCount)` | Called by evaluator — adds to local claims |
| `relinquishChannel(UUID)` | Called by evaluator — removes local claim |
| `getLocalClaims()` | Called by heartbeat — returns `Map<UUID, OwnershipClaim>` for serialisation |
| `updateRemoteOwnership(String peerId, Map<UUID, OwnershipClaim>)` | Called by heartbeat — merges peer's claims into ownership map |
| `clearPeerOwnership(String peerId)` | Called on peer DEAD transition — removes all claims from that peer |

## 5. WriteRoutingDecorator changes

Single addition — record the write after dispatch:

```java
public MessageResult dispatch(MessageDispatch dispatch) {
    // ... existing routing logic ...
    MessageResult result = delegate.dispatch(dispatch);
    if (tracker != null) {
        tracker.recordWrite(dispatch.channelId());
    }
    return result;
}
```

The tracker is null when `routing != dynamic`. Recording happens after
successful dispatch — failed writes are not counted.

## 6. Configuration

All under `casehub.qhorus.relay.ownership`:

| Property | Default | Description |
|----------|---------|-------------|
| `window-seconds` | 300 | Sliding window duration (5 minutes) |
| `bucket-count` | 10 | Number of buckets in the window |
| `evaluation-interval-seconds` | 10 | How often the evaluator ticks |
| `hysteresis-ratio` | 2.0 | Challenger must exceed this multiple of owner's count |
| `min-claim-writes` | 5 | Minimum writes in window to claim ownership |

These properties are only read when `casehub.qhorus.relay.routing=dynamic`.
In `hash-ring` or `none` mode, the tracker and evaluator are not created.

## 7. Testing strategy

All tests are CDI-free unit tests with Mockito — matching the existing
cluster module pattern (no `@QuarkusTest`).

| Test class | What it covers |
|------------|---------------|
| `BucketWindowTest` | Bucket rotation, write counting, window expiry, zero-count detection |
| `WriteFrequencyTrackerTest` | Multi-channel tracking, concurrent writes, active channel listing, eviction |
| `DynamicOwnershipResolverTest` | Dynamic claim lookup, hash ring fallback, claim removal → revert to hash ring |
| `OwnershipEvaluatorTest` | Claim logic (2x threshold, min writes), relinquishment (zero count), transfer (challenger beats owner), no-claim when below threshold |
| `HeartbeatOwnershipTest` | Ownership claim serialisation in HeartbeatResponse, peer claim merge, peer death → claim removal |

Clock injection for deterministic time control (same pattern as
`ClusterManagerTest`). Statistical tests not needed — ownership is
deterministic given write counts, unlike hash ring distribution.

## 8. What this does NOT include

- **WriteProxyClient wiring** — the HTTP proxy remains a stub. Ownership
  logic is testable without actual cross-node dispatch.
- **Load balancing** — no channel shedding based on node load. Ownership
  is purely write-locality.
- **Persistence** — ownership state is in-memory only, reconstructed
  from heartbeats on restart.
- **Multi-tenancy** — write counts are per-channel, not per-tenant.
  Ownership is a relay-level concern, orthogonal to tenancy.

## References

- `cluster/src/main/java/io/casehub/qhorus/cluster/ConsistentHashRing.java` — existing hash ring implementation
- `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterManager.java` — ownership delegation point
- `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java` — write interception point
- `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatService.java` — heartbeat protocol
- `cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java` — existing config with `routing` property
- Consolidated spec §2 (Level 4: Channel ownership)
- Consolidated spec §4 (Write Model)
- Consolidated spec §11 (Phase 6 roadmap)
- Consolidated spec risk register (dynamic ownership oscillation)
- Decisions D5, D11, D18, D20, D32-D38
