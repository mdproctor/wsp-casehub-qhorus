# Dynamic Ownership Heuristics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #475 — epic: distributed qhorus mesh — standalone service with clustering
**Issue group:** #475

**Goal:** Implement write-frequency-based dynamic channel ownership so relays earn ownership of channels they write to most frequently, minimising proxy hops.

**Architecture:** Four new CDI-free POJOs in the `cluster` module (`BucketWindow`, `WriteFrequencyTracker`, `DynamicOwnershipResolver`, `OwnershipEvaluator`) plus modifications to `ClusterManager`, `HeartbeatResponse`, `HeartbeatService`, `WriteRoutingDecorator`, and `RelayProducer`. All ownership state is in-memory, reconstructed via heartbeats on restart. Hash ring remains the fallback for channels with no write history.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Mockito, `java.util.concurrent` (AtomicLong, ConcurrentHashMap)

## Global Constraints

- All new classes are CDI-free POJOs — no `@ApplicationScoped`, no `@Inject`
- Constructor-injected `Clock` for deterministic time in tests
- Tests use JUnit 5 + AssertJ + Mockito (no `@QuarkusTest`)
- Package: `io.casehub.qhorus.cluster`
- Source: `cluster/src/main/java/io/casehub/qhorus/cluster/`
- Tests: `cluster/src/test/java/io/casehub/qhorus/cluster/`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
- Config prefix for ownership: `casehub.qhorus.relay.ownership`

---

## Batch 1: Sliding window and write tracking

### Task 1: BucketWindow — circular bucket array for write counting

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/BucketWindow.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/BucketWindowTest.java`

**Interfaces:**
- Consumes: nothing (leaf component)
- Produces:
  - `BucketWindow(int bucketCount, Duration bucketDuration, Clock clock)`
  - `void recordWrite()` — increments current bucket
  - `long getCount()` — sums all buckets
  - `boolean tryRotate()` — advances pointer if bucket duration elapsed, returns true if rotated
  - `boolean isEmpty()` — true when all buckets are zero

- [ ] **Step 1: Write BucketWindowTest**

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;

import static org.assertj.core.api.Assertions.assertThat;

class BucketWindowTest {

    private static final Instant NOW = Instant.parse("2026-10-06T12:00:00Z");

    private Clock fixedClock(Instant instant) {
        return Clock.fixed(instant, ZoneId.of("UTC"));
    }

    @Test
    void recordWriteIncrementsCurrent() {
        var window = new BucketWindow(10, Duration.ofSeconds(30), fixedClock(NOW));
        window.recordWrite();
        window.recordWrite();
        assertThat(window.getCount()).isEqualTo(2);
    }

    @Test
    void getCountSumsAllBuckets() {
        var clock = fixedClock(NOW);
        var window = new BucketWindow(3, Duration.ofSeconds(10), clock);
        window.recordWrite(); // bucket 0: 1

        window = new BucketWindow(3, Duration.ofSeconds(10), fixedClock(NOW));
        window.recordWrite();
        window.recordWrite();
        // all in bucket 0
        assertThat(window.getCount()).isEqualTo(2);
    }

    @Test
    void tryRotateAdvancesWhenDurationElapsed() {
        var window = new BucketWindow(3, Duration.ofSeconds(10), fixedClock(NOW));
        window.recordWrite(); // bucket 0: 1

        // advance clock past bucket duration
        window.setClock(fixedClock(NOW.plusSeconds(11)));
        assertThat(window.tryRotate()).isTrue();

        window.recordWrite(); // bucket 1: 1
        assertThat(window.getCount()).isEqualTo(2); // bucket 0 + bucket 1
    }

    @Test
    void tryRotateDoesNotAdvanceBeforeDuration() {
        var window = new BucketWindow(3, Duration.ofSeconds(10), fixedClock(NOW));
        window.recordWrite();
        window.setClock(fixedClock(NOW.plusSeconds(5)));
        assertThat(window.tryRotate()).isFalse();
        assertThat(window.getCount()).isEqualTo(1);
    }

    @Test
    void rotationClearsOldestBucket() {
        var window = new BucketWindow(3, Duration.ofSeconds(10), fixedClock(NOW));
        window.recordWrite(); // bucket 0: 1

        // rotate through all 3 buckets to wrap around
        window.setClock(fixedClock(NOW.plusSeconds(11)));
        window.tryRotate(); // now on bucket 1, clears bucket 1
        window.recordWrite(); // bucket 1: 1

        window.setClock(fixedClock(NOW.plusSeconds(22)));
        window.tryRotate(); // now on bucket 2, clears bucket 2

        window.setClock(fixedClock(NOW.plusSeconds(33)));
        window.tryRotate(); // now on bucket 0, clears bucket 0 (old data gone)

        assertThat(window.getCount()).isEqualTo(1); // only bucket 1 survives
    }

    @Test
    void isEmptyWhenAllBucketsZero() {
        var window = new BucketWindow(3, Duration.ofSeconds(10), fixedClock(NOW));
        assertThat(window.isEmpty()).isTrue();

        window.recordWrite();
        assertThat(window.isEmpty()).isFalse();
    }

    @Test
    void multipleRotationsOnLargeTimeJump() {
        var window = new BucketWindow(3, Duration.ofSeconds(10), fixedClock(NOW));
        window.recordWrite(); // bucket 0: 1

        // jump 40s — should rotate through all buckets and clear everything
        window.setClock(fixedClock(NOW.plusSeconds(40)));
        window.tryRotate();
        assertThat(window.isEmpty()).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=BucketWindowTest`
Expected: FAIL — `BucketWindow` class does not exist

- [ ] **Step 3: Implement BucketWindow**

```java
package io.casehub.qhorus.cluster;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicLong;

public class BucketWindow {

    private final AtomicLong[] buckets;
    private final long bucketDurationMillis;
    private Clock clock;
    private int currentIndex;
    private Instant currentBucketStart;

    public BucketWindow(int bucketCount, Duration bucketDuration, Clock clock) {
        this.buckets = new AtomicLong[bucketCount];
        for (int i = 0; i < bucketCount; i++) {
            this.buckets[i] = new AtomicLong(0);
        }
        this.bucketDurationMillis = bucketDuration.toMillis();
        this.clock = clock;
        this.currentIndex = 0;
        this.currentBucketStart = clock.instant();
    }

    public void recordWrite() {
        buckets[currentIndex].incrementAndGet();
    }

    public long getCount() {
        long sum = 0;
        for (AtomicLong bucket : buckets) {
            sum += bucket.get();
        }
        return sum;
    }

    public boolean tryRotate() {
        Instant now = clock.instant();
        long elapsedMillis = now.toEpochMilli() - currentBucketStart.toEpochMilli();
        if (elapsedMillis < bucketDurationMillis) {
            return false;
        }
        int steps = (int) (elapsedMillis / bucketDurationMillis);
        if (steps >= buckets.length) {
            for (AtomicLong bucket : buckets) {
                bucket.set(0);
            }
            currentIndex = 0;
        } else {
            for (int i = 0; i < steps; i++) {
                currentIndex = (currentIndex + 1) % buckets.length;
                buckets[currentIndex].set(0);
            }
        }
        currentBucketStart = now;
        return true;
    }

    public boolean isEmpty() {
        for (AtomicLong bucket : buckets) {
            if (bucket.get() != 0) {
                return false;
            }
        }
        return true;
    }

    void setClock(Clock clock) {
        this.clock = clock;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=BucketWindowTest`
Expected: PASS — all 7 tests green

- [ ] **Step 5: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/BucketWindow.java cluster/src/test/java/io/casehub/qhorus/cluster/BucketWindowTest.java
git commit -m "feat(#475): add BucketWindow — circular bucket array for sliding window write counting

Refs #475

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: WriteFrequencyTracker — per-channel write counter

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteFrequencyTracker.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/WriteFrequencyTrackerTest.java`

**Interfaces:**
- Consumes: `BucketWindow(int, Duration, Clock)` from Task 1
- Produces:
  - `WriteFrequencyTracker(int bucketCount, Duration bucketDuration, Clock clock)`
  - `void recordWrite(UUID channelId)` — creates BucketWindow on first write, increments
  - `long getCount(UUID channelId)` — returns 0 if no data
  - `Set<UUID> getActiveChannels()` — channels with non-zero counts
  - `void rotateAll()` — calls tryRotate on all windows, prunes empty ones
  - `int channelCount()` — number of tracked channels (for testing)

- [ ] **Step 1: Write WriteFrequencyTrackerTest**

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class WriteFrequencyTrackerTest {

    private static final Instant NOW = Instant.parse("2026-10-06T12:00:00Z");
    private static final Duration BUCKET_DURATION = Duration.ofSeconds(30);
    private static final int BUCKET_COUNT = 10;

    private Clock clock;
    private WriteFrequencyTracker tracker;

    @BeforeEach
    void setUp() {
        clock = Clock.fixed(NOW, ZoneId.of("UTC"));
        tracker = new WriteFrequencyTracker(BUCKET_COUNT, BUCKET_DURATION, clock);
    }

    @Test
    void recordWriteCreatesWindowOnFirstWrite() {
        UUID ch = UUID.randomUUID();
        tracker.recordWrite(ch);
        assertThat(tracker.getCount(ch)).isEqualTo(1);
        assertThat(tracker.channelCount()).isEqualTo(1);
    }

    @Test
    void getCountReturnsZeroForUnknownChannel() {
        assertThat(tracker.getCount(UUID.randomUUID())).isEqualTo(0);
    }

    @Test
    void tracksMultipleChannelsIndependently() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        tracker.recordWrite(ch1);
        tracker.recordWrite(ch1);
        tracker.recordWrite(ch2);
        assertThat(tracker.getCount(ch1)).isEqualTo(2);
        assertThat(tracker.getCount(ch2)).isEqualTo(1);
    }

    @Test
    void getActiveChannelsReturnsOnlyNonZero() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        tracker.recordWrite(ch1);
        tracker.recordWrite(ch2);
        assertThat(tracker.getActiveChannels()).containsExactlyInAnyOrder(ch1, ch2);
    }

    @Test
    void rotateAllPrunesEmptyChannels() {
        UUID ch = UUID.randomUUID();
        tracker.recordWrite(ch);

        // advance clock past full window (10 × 30s = 300s)
        tracker.setClock(Clock.fixed(NOW.plusSeconds(310), ZoneId.of("UTC")));
        tracker.rotateAll();

        assertThat(tracker.getCount(ch)).isEqualTo(0);
        assertThat(tracker.channelCount()).isEqualTo(0);
        assertThat(tracker.getActiveChannels()).isEmpty();
    }

    @Test
    void rotateAllKeepsActiveChannels() {
        UUID active = UUID.randomUUID();
        UUID stale = UUID.randomUUID();
        tracker.recordWrite(active);
        tracker.recordWrite(stale);

        // advance clock past full window
        tracker.setClock(Clock.fixed(NOW.plusSeconds(310), ZoneId.of("UTC")));
        tracker.rotateAll();

        // write to active after rotation
        tracker.recordWrite(active);

        assertThat(tracker.channelCount()).isEqualTo(1);
        assertThat(tracker.getActiveChannels()).containsExactly(active);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=WriteFrequencyTrackerTest`
Expected: FAIL — `WriteFrequencyTracker` class does not exist

- [ ] **Step 3: Implement WriteFrequencyTracker**

```java
package io.casehub.qhorus.cluster;

import java.time.Clock;
import java.time.Duration;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class WriteFrequencyTracker {

    private final int bucketCount;
    private final Duration bucketDuration;
    private Clock clock;
    private final ConcurrentHashMap<UUID, BucketWindow> windows = new ConcurrentHashMap<>();

    public WriteFrequencyTracker(int bucketCount, Duration bucketDuration, Clock clock) {
        this.bucketCount = bucketCount;
        this.bucketDuration = bucketDuration;
        this.clock = clock;
    }

    public void recordWrite(UUID channelId) {
        windows.computeIfAbsent(channelId, id -> new BucketWindow(bucketCount, bucketDuration, clock))
                .recordWrite();
    }

    public long getCount(UUID channelId) {
        BucketWindow window = windows.get(channelId);
        return window == null ? 0 : window.getCount();
    }

    public Set<UUID> getActiveChannels() {
        return Set.copyOf(windows.keySet());
    }

    public void rotateAll() {
        var iterator = windows.entrySet().iterator();
        while (iterator.hasNext()) {
            var entry = iterator.next();
            entry.getValue().tryRotate();
            if (entry.getValue().isEmpty()) {
                iterator.remove();
            }
        }
    }

    public int channelCount() {
        return windows.size();
    }

    void setClock(Clock clock) {
        this.clock = clock;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=WriteFrequencyTrackerTest`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/WriteFrequencyTracker.java cluster/src/test/java/io/casehub/qhorus/cluster/WriteFrequencyTrackerTest.java
git commit -m "feat(#475): add WriteFrequencyTracker — per-channel sliding window write counter

Refs #475

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Ownership resolution and evaluation

### Task 3: OwnershipClaim record and DynamicOwnershipResolver

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/OwnershipClaim.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/DynamicOwnershipResolver.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/DynamicOwnershipResolverTest.java`

**Interfaces:**
- Consumes: `ConsistentHashRing.owner(UUID) → String`, `NodeInfo(String, String)`, existing `ClusterManager` for `nodeInfo()` lookup
- Produces:
  - `record OwnershipClaim(String nodeId, long writeCount)` — serialised in heartbeat
  - `DynamicOwnershipResolver(ConsistentHashRing hashRing, Map<String, String> nodeAddresses)`
  - `NodeInfo owner(UUID channelId)` — dynamic claim first, hash ring fallback
  - `void updateClaim(UUID channelId, OwnershipClaim claim)` — add/replace a claim
  - `void removeClaim(UUID channelId)` — remove a claim (revert to hash ring)
  - `void clearClaimsForNode(String nodeId)` — remove all claims from a dead peer
  - `OwnershipClaim getClaim(UUID channelId)` — returns null if no dynamic claim
  - `Map<UUID, OwnershipClaim> getAllClaims()` — for heartbeat serialisation

- [ ] **Step 1: Write OwnershipClaim record**

```java
package io.casehub.qhorus.cluster;

public record OwnershipClaim(String nodeId, long writeCount) {
    public OwnershipClaim {
        if (nodeId == null || nodeId.isBlank()) {
            throw new IllegalArgumentException("nodeId must not be blank");
        }
        if (writeCount < 0) {
            throw new IllegalArgumentException("writeCount must not be negative");
        }
    }
}
```

- [ ] **Step 2: Write DynamicOwnershipResolverTest**

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class DynamicOwnershipResolverTest {

    private ConsistentHashRing ring;
    private Map<String, String> nodeAddresses;
    private DynamicOwnershipResolver resolver;

    @BeforeEach
    void setUp() {
        ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        nodeAddresses = new LinkedHashMap<>();
        nodeAddresses.put("node-1", "node-1:8080");
        nodeAddresses.put("node-2", "node-2:8080");
        nodeAddresses.put("node-3", "node-3:8080");
        resolver = new DynamicOwnershipResolver(ring, nodeAddresses);
    }

    @Test
    void fallsBackToHashRingWithNoClaims() {
        UUID ch = UUID.randomUUID();
        String expected = ring.owner(ch);
        assertThat(resolver.owner(ch).nodeId()).isEqualTo(expected);
    }

    @Test
    void dynamicClaimOverridesHashRing() {
        UUID ch = UUID.randomUUID();
        String hashOwner = ring.owner(ch);
        String claimant = hashOwner.equals("node-1") ? "node-2" : "node-1";

        resolver.updateClaim(ch, new OwnershipClaim(claimant, 50));
        assertThat(resolver.owner(ch).nodeId()).isEqualTo(claimant);
    }

    @Test
    void removeClaimRevertsToHashRing() {
        UUID ch = UUID.randomUUID();
        String hashOwner = ring.owner(ch);

        resolver.updateClaim(ch, new OwnershipClaim("node-2", 50));
        resolver.removeClaim(ch);
        assertThat(resolver.owner(ch).nodeId()).isEqualTo(hashOwner);
    }

    @Test
    void clearClaimsForNodeRemovesAllClaimsFromPeer() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        UUID ch3 = UUID.randomUUID();

        resolver.updateClaim(ch1, new OwnershipClaim("node-2", 10));
        resolver.updateClaim(ch2, new OwnershipClaim("node-2", 20));
        resolver.updateClaim(ch3, new OwnershipClaim("node-3", 30));

        resolver.clearClaimsForNode("node-2");

        assertThat(resolver.getClaim(ch1)).isNull();
        assertThat(resolver.getClaim(ch2)).isNull();
        assertThat(resolver.getClaim(ch3)).isNotNull();
    }

    @Test
    void getAllClaimsReturnsSnapshot() {
        UUID ch = UUID.randomUUID();
        resolver.updateClaim(ch, new OwnershipClaim("node-1", 42));
        Map<UUID, OwnershipClaim> claims = resolver.getAllClaims();
        assertThat(claims).hasSize(1);
        assertThat(claims.get(ch).writeCount()).isEqualTo(42);
    }

    @Test
    void ownerReturnsNodeInfoWithAddress() {
        UUID ch = UUID.randomUUID();
        resolver.updateClaim(ch, new OwnershipClaim("node-2", 10));
        NodeInfo info = resolver.owner(ch);
        assertThat(info.nodeId()).isEqualTo("node-2");
        assertThat(info.address()).isEqualTo("node-2:8080");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=DynamicOwnershipResolverTest`
Expected: FAIL — `DynamicOwnershipResolver` class does not exist

- [ ] **Step 4: Implement DynamicOwnershipResolver**

```java
package io.casehub.qhorus.cluster;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class DynamicOwnershipResolver {

    private final ConsistentHashRing hashRing;
    private final Map<String, String> nodeAddresses;
    private final ConcurrentHashMap<UUID, OwnershipClaim> claims = new ConcurrentHashMap<>();

    public DynamicOwnershipResolver(ConsistentHashRing hashRing, Map<String, String> nodeAddresses) {
        this.hashRing = hashRing;
        this.nodeAddresses = nodeAddresses;
    }

    public NodeInfo owner(UUID channelId) {
        OwnershipClaim claim = claims.get(channelId);
        if (claim != null) {
            return nodeInfo(claim.nodeId());
        }
        return nodeInfo(hashRing.owner(channelId));
    }

    public void updateClaim(UUID channelId, OwnershipClaim claim) {
        claims.put(channelId, claim);
    }

    public void removeClaim(UUID channelId) {
        claims.remove(channelId);
    }

    public void clearClaimsForNode(String nodeId) {
        claims.entrySet().removeIf(e -> e.getValue().nodeId().equals(nodeId));
    }

    public OwnershipClaim getClaim(UUID channelId) {
        return claims.get(channelId);
    }

    public Map<UUID, OwnershipClaim> getAllClaims() {
        return Map.copyOf(claims);
    }

    private NodeInfo nodeInfo(String nodeId) {
        String address = nodeAddresses.get(nodeId);
        if (address == null) {
            address = nodeId + ":8080";
        }
        return new NodeInfo(nodeId, address);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=DynamicOwnershipResolverTest`
Expected: PASS — all 6 tests green

- [ ] **Step 6: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/OwnershipClaim.java cluster/src/main/java/io/casehub/qhorus/cluster/DynamicOwnershipResolver.java cluster/src/test/java/io/casehub/qhorus/cluster/DynamicOwnershipResolverTest.java
git commit -m "feat(#475): add OwnershipClaim and DynamicOwnershipResolver — layered ownership with hash ring fallback

Refs #475

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: OwnershipEvaluator — periodic claim/relinquish logic

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/OwnershipEvaluator.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/OwnershipEvaluatorTest.java`

**Interfaces:**
- Consumes:
  - `WriteFrequencyTracker.getCount(UUID) → long`, `.getActiveChannels() → Set<UUID>`, `.rotateAll()`
  - `DynamicOwnershipResolver.owner(UUID) → NodeInfo`, `.updateClaim(UUID, OwnershipClaim)`, `.removeClaim(UUID)`, `.getClaim(UUID) → OwnershipClaim`
- Produces:
  - `OwnershipEvaluator(String localNodeId, WriteFrequencyTracker tracker, DynamicOwnershipResolver resolver, double hysteresisRatio, int minClaimWrites)`
  - `void evaluate()` — runs one evaluation cycle: rotate, scan, claim/relinquish
  - `Map<UUID, OwnershipClaim> getLocalClaims()` — local node's claims for heartbeat

- [ ] **Step 1: Write OwnershipEvaluatorTest**

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class OwnershipEvaluatorTest {

    private static final Instant NOW = Instant.parse("2026-10-06T12:00:00Z");
    private static final String LOCAL_NODE = "node-1";
    private static final double HYSTERESIS = 2.0;
    private static final int MIN_CLAIMS = 5;

    private WriteFrequencyTracker tracker;
    private DynamicOwnershipResolver resolver;
    private OwnershipEvaluator evaluator;

    @BeforeEach
    void setUp() {
        Clock clock = Clock.fixed(NOW, ZoneId.of("UTC"));
        tracker = new WriteFrequencyTracker(10, Duration.ofSeconds(30), clock);
        ConsistentHashRing ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        Map<String, String> addresses = new LinkedHashMap<>();
        addresses.put("node-1", "node-1:8080");
        addresses.put("node-2", "node-2:8080");
        addresses.put("node-3", "node-3:8080");
        resolver = new DynamicOwnershipResolver(ring, addresses);
        evaluator = new OwnershipEvaluator(LOCAL_NODE, tracker, resolver, HYSTERESIS, MIN_CLAIMS);
    }

    @Test
    void claimsChannelWhenWriteExceedsTwiceHashRingOwnerWithNoWrites() {
        UUID ch = UUID.randomUUID();
        // hash ring assigns to some node — with no claims, owner has 0 writes
        for (int i = 0; i < 10; i++) {
            tracker.recordWrite(ch);
        }
        evaluator.evaluate();
        // if hash ring owner is not local, we should claim
        // if hash ring owner IS local, we already own it — no claim needed
        NodeInfo hashOwner = resolver.owner(ch);
        if (!hashOwner.nodeId().equals(LOCAL_NODE)) {
            // we claimed it
            assertThat(evaluator.getLocalClaims()).containsKey(ch);
            assertThat(evaluator.getLocalClaims().get(ch).writeCount()).isEqualTo(10);
        }
    }

    @Test
    void doesNotClaimWhenBelowMinClaimWrites() {
        UUID ch = UUID.randomUUID();
        for (int i = 0; i < 3; i++) { // below MIN_CLAIMS=5
            tracker.recordWrite(ch);
        }
        evaluator.evaluate();
        assertThat(evaluator.getLocalClaims()).isEmpty();
    }

    @Test
    void doesNotClaimWhenOwnerWriteCountTooHigh() {
        UUID ch = UUID.randomUUID();
        // remote node owns with 100 writes
        resolver.updateClaim(ch, new OwnershipClaim("node-2", 100));
        for (int i = 0; i < 50; i++) { // 50 < 2 * 100, does not exceed threshold
            tracker.recordWrite(ch);
        }
        evaluator.evaluate();
        // should NOT have replaced node-2's claim
        assertThat(resolver.getClaim(ch).nodeId()).isEqualTo("node-2");
    }

    @Test
    void claimsWhenExceedingRemoteOwnerByHysteresis() {
        UUID ch = UUID.randomUUID();
        resolver.updateClaim(ch, new OwnershipClaim("node-2", 10));
        for (int i = 0; i < 25; i++) { // 25 > 2 * 10
            tracker.recordWrite(ch);
        }
        evaluator.evaluate();
        assertThat(evaluator.getLocalClaims()).containsKey(ch);
        assertThat(resolver.getClaim(ch).nodeId()).isEqualTo(LOCAL_NODE);
    }

    @Test
    void relinquishesWhenWriteCountDropsToZero() {
        UUID ch = UUID.randomUUID();
        // local node currently claims this channel
        resolver.updateClaim(ch, new OwnershipClaim(LOCAL_NODE, 50));
        evaluator.addLocalClaim(ch, new OwnershipClaim(LOCAL_NODE, 50));

        // tracker has zero writes (window expired or never recorded)
        evaluator.evaluate();

        assertThat(evaluator.getLocalClaims()).doesNotContainKey(ch);
        assertThat(resolver.getClaim(ch)).isNull(); // reverted to hash ring
    }

    @Test
    void doesNotRelinquishWhenStillWriting() {
        UUID ch = UUID.randomUUID();
        resolver.updateClaim(ch, new OwnershipClaim(LOCAL_NODE, 50));
        evaluator.addLocalClaim(ch, new OwnershipClaim(LOCAL_NODE, 50));
        tracker.recordWrite(ch);

        evaluator.evaluate();

        assertThat(evaluator.getLocalClaims()).containsKey(ch);
    }

    @Test
    void skipsChannelsAlreadyOwnedLocally() {
        UUID ch = UUID.randomUUID();
        // find a channel where hash ring assigns to local node
        // we need to generate until we find one
        ConsistentHashRing ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        UUID localOwned = null;
        for (int i = 0; i < 100; i++) {
            UUID candidate = UUID.randomUUID();
            if (ring.owner(candidate).equals(LOCAL_NODE)) {
                localOwned = candidate;
                break;
            }
        }
        if (localOwned != null) {
            for (int i = 0; i < 10; i++) {
                tracker.recordWrite(localOwned);
            }
            evaluator.evaluate();
            // should not create a dynamic claim — already the hash ring owner
            assertThat(evaluator.getLocalClaims()).doesNotContainKey(localOwned);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=OwnershipEvaluatorTest`
Expected: FAIL — `OwnershipEvaluator` class does not exist

- [ ] **Step 3: Implement OwnershipEvaluator**

```java
package io.casehub.qhorus.cluster;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class OwnershipEvaluator {

    private final String localNodeId;
    private final WriteFrequencyTracker tracker;
    private final DynamicOwnershipResolver resolver;
    private final double hysteresisRatio;
    private final int minClaimWrites;
    private final ConcurrentHashMap<UUID, OwnershipClaim> localClaims = new ConcurrentHashMap<>();

    public OwnershipEvaluator(String localNodeId, WriteFrequencyTracker tracker,
                              DynamicOwnershipResolver resolver,
                              double hysteresisRatio, int minClaimWrites) {
        this.localNodeId = localNodeId;
        this.tracker = tracker;
        this.resolver = resolver;
        this.hysteresisRatio = hysteresisRatio;
        this.minClaimWrites = minClaimWrites;
    }

    public void evaluate() {
        tracker.rotateAll();

        // Check relinquishment for channels we currently claim
        var claimIterator = localClaims.entrySet().iterator();
        while (claimIterator.hasNext()) {
            var entry = claimIterator.next();
            UUID channelId = entry.getKey();
            long localCount = tracker.getCount(channelId);
            if (localCount == 0) {
                claimIterator.remove();
                resolver.removeClaim(channelId);
            } else {
                OwnershipClaim updated = new OwnershipClaim(localNodeId, localCount);
                localClaims.put(channelId, updated);
                resolver.updateClaim(channelId, updated);
            }
        }

        // Check for new claims
        for (UUID channelId : tracker.getActiveChannels()) {
            if (localClaims.containsKey(channelId)) {
                continue;
            }
            long localCount = tracker.getCount(channelId);
            if (localCount < minClaimWrites) {
                continue;
            }
            NodeInfo currentOwner = resolver.owner(channelId);
            if (currentOwner.nodeId().equals(localNodeId)) {
                continue;
            }
            OwnershipClaim ownerClaim = resolver.getClaim(channelId);
            long ownerCount = ownerClaim != null ? ownerClaim.writeCount() : 0;
            if (localCount > hysteresisRatio * ownerCount) {
                OwnershipClaim claim = new OwnershipClaim(localNodeId, localCount);
                localClaims.put(channelId, claim);
                resolver.updateClaim(channelId, claim);
            }
        }
    }

    public Map<UUID, OwnershipClaim> getLocalClaims() {
        return Map.copyOf(localClaims);
    }

    void addLocalClaim(UUID channelId, OwnershipClaim claim) {
        localClaims.put(channelId, claim);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=OwnershipEvaluatorTest`
Expected: PASS — all 7 tests green

- [ ] **Step 5: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/OwnershipEvaluator.java cluster/src/test/java/io/casehub/qhorus/cluster/OwnershipEvaluatorTest.java
git commit -m "feat(#475): add OwnershipEvaluator — periodic claim/relinquish with 2x hysteresis

Refs #475

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Integration — wire into ClusterManager, heartbeat, and routing

### Task 5: ClusterManager ownership methods + HeartbeatResponse extension + HeartbeatService integration + WriteRoutingDecorator tracker + RelayProducer wiring

**Files:**
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterManager.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatResponse.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatService.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/OwnershipConfig.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/HeartbeatOwnershipTest.java`
- Modify: `cluster/src/test/java/io/casehub/qhorus/cluster/ClusterManagerTest.java`
- Modify: `cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java`
- Modify: `cluster/src/test/java/io/casehub/qhorus/cluster/HeartbeatServiceTest.java`

**Interfaces:**
- Consumes:
  - `DynamicOwnershipResolver.owner(UUID)`, `.updateClaim()`, `.removeClaim()`, `.clearClaimsForNode()`, `.getAllClaims()`
  - `OwnershipEvaluator.evaluate()`, `.getLocalClaims()`
  - `WriteFrequencyTracker.recordWrite(UUID)`
- Produces:
  - `ClusterManager.setResolver(DynamicOwnershipResolver)` — called by RelayProducer when routing=dynamic
  - `ClusterManager.setEvaluator(OwnershipEvaluator)` — called by RelayProducer
  - `ClusterManager.updateRemoteOwnership(String peerId, Map<UUID, OwnershipClaim>)` — called by HeartbeatService
  - `ClusterManager.getLocalClaims()` — called by HeartbeatService for response
  - `HeartbeatResponse` gains `Map<UUID, OwnershipClaim> ownershipClaims` field
  - `WriteRoutingDecorator` gains `WriteFrequencyTracker tracker` field
  - `OwnershipConfig` — SmallRye ConfigMapping for ownership properties

- [ ] **Step 1: Create OwnershipConfig**

```java
package io.casehub.qhorus.cluster;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.qhorus.relay.ownership")
public interface OwnershipConfig {

    @WithDefault("300")
    int windowSeconds();

    @WithDefault("10")
    int bucketCount();

    @WithDefault("10")
    int evaluationIntervalSeconds();

    @WithDefault("2.0")
    double hysteresisRatio();

    @WithDefault("5")
    int minClaimWrites();
}
```

- [ ] **Step 2: Modify HeartbeatResponse — add ownershipClaims field**

Change from:
```java
public record HeartbeatResponse(String nodeId, Instant timestamp, String ringHash, String status) {}
```
To:
```java
public record HeartbeatResponse(String nodeId, Instant timestamp, String ringHash, String status,
                                Map<UUID, OwnershipClaim> ownershipClaims) {
    public HeartbeatResponse(String nodeId, Instant timestamp, String ringHash, String status) {
        this(nodeId, timestamp, ringHash, status, Map.of());
    }
}
```

Add `import java.util.Map;` and `import java.util.UUID;`.

- [ ] **Step 3: Modify ClusterManager — add ownership methods**

Add fields:
```java
private DynamicOwnershipResolver resolver;
private OwnershipEvaluator evaluator;
```

Add methods:
```java
public void setResolver(DynamicOwnershipResolver resolver) {
    this.resolver = resolver;
}

public void setEvaluator(OwnershipEvaluator evaluator) {
    this.evaluator = evaluator;
}
```

Modify `owner(UUID)`:
```java
public NodeInfo owner(UUID channelId) {
    if (resolver != null) {
        return resolver.owner(channelId);
    }
    String nodeId = ringRef.get().owner(channelId);
    NodeInfo info = configuredPeers.get(nodeId);
    if (info == null) {
        return new NodeInfo(nodeId, nodeId + ":8080");
    }
    return info;
}
```

Add ownership management methods:
```java
public void updateRemoteOwnership(String peerId, Map<UUID, OwnershipClaim> claims) {
    if (resolver == null) return;
    resolver.clearClaimsForNode(peerId);
    for (var entry : claims.entrySet()) {
        resolver.updateClaim(entry.getKey(), entry.getValue());
    }
}

public Map<UUID, OwnershipClaim> getLocalClaims() {
    return evaluator != null ? evaluator.getLocalClaims() : Map.of();
}

public void clearPeerOwnership(String peerId) {
    if (resolver != null) {
        resolver.clearClaimsForNode(peerId);
    }
}
```

Modify `recordMiss` — when peer transitions to DEAD, clear its ownership:
```java
if (newState == NodeState.DEAD) {
    removeFromRing(id);
    clearPeerOwnership(id);
}
```

Modify `handlePeerDeparture` — when peer leaves, clear its ownership:
```java
if (old.state() != NodeState.DEAD) {
    emitEvent(id, old.state(), NodeState.DEAD);
    removeFromRing(id);
    clearPeerOwnership(id);
}
```

- [ ] **Step 4: Modify HeartbeatService — propagate ownership claims**

Update `tick()` to merge remote ownership:
```java
public void tick() {
    String localRingHash = clusterManager.ringHash();
    for (var entry : clusterManager.peerStates().entrySet()) {
        PeerState ps = entry.getValue();
        if (ps.state() == NodeState.DEAD) {
            continue;
        }
        try {
            HeartbeatResponse resp = heartbeatCaller.apply(ps.nodeInfo());
            clusterManager.recordHeartbeat(entry.getKey());
            if (resp != null) {
                if (resp.ringHash() != null && !resp.ringHash().equals(localRingHash)) {
                    LOG.warnf("Ring disagreement with %s — local=%s remote=%s",
                            entry.getKey(), localRingHash, resp.ringHash());
                }
                if (resp.ownershipClaims() != null && !resp.ownershipClaims().isEmpty()) {
                    clusterManager.updateRemoteOwnership(entry.getKey(), resp.ownershipClaims());
                }
            }
        } catch (Exception e) {
            clusterManager.recordMiss(entry.getKey());
        }
    }
}
```

Update `buildLocalResponse()` to include local claims:
```java
public HeartbeatResponse buildLocalResponse() {
    return new HeartbeatResponse(
            clusterManager.nodeId(),
            Instant.now(),
            clusterManager.ringHash(),
            "UP",
            clusterManager.getLocalClaims());
}
```

- [ ] **Step 5: Modify WriteRoutingDecorator — add tracker**

Add tracker field and updated constructor:
```java
private final WriteFrequencyTracker tracker;

public WriteRoutingDecorator(MessageDispatcher delegate,
                             ClusterManager clusterManager,
                             WriteProxyClient proxyClient,
                             boolean routingEnabled,
                             WriteFrequencyTracker tracker) {
    this.delegate = delegate;
    this.clusterManager = clusterManager;
    this.proxyClient = proxyClient;
    this.routingEnabled = routingEnabled;
    this.tracker = tracker;
}
```

Add backward-compatible 4-arg constructor:
```java
public WriteRoutingDecorator(MessageDispatcher delegate,
                             ClusterManager clusterManager,
                             WriteProxyClient proxyClient,
                             boolean routingEnabled) {
    this(delegate, clusterManager, proxyClient, routingEnabled, null);
}
```

Add `tracker.recordWrite()` after successful dispatch in `dispatch()`:
```java
@Override
public DispatchResult dispatch(MessageDispatch dispatch) {
    if (!routingEnabled) {
        return delegate.dispatch(dispatch);
    }
    if (!clusterManager.canServeWrites()) {
        throw new QuorumViolationException("This node is in a minority partition and cannot serve writes");
    }
    NodeInfo owner = clusterManager.owner(dispatch.channelId());
    DispatchResult result;
    if (clusterManager.isLocal(owner)) {
        result = delegate.dispatch(dispatch);
    } else {
        try {
            result = proxyClient.dispatch(owner, dispatch);
        } catch (Exception e) {
            LOG.warnf("Proxy to %s failed, falling back to local dispatch: %s",
                    owner.nodeId(), e.getMessage());
            result = delegate.dispatch(dispatch);
        }
    }
    if (tracker != null) {
        tracker.recordWrite(dispatch.channelId());
    }
    return result;
}
```

- [ ] **Step 6: Modify RelayProducer — wire dynamic ownership components**

Update `clusterManager()` producer method. After creating the manager, check if routing is dynamic and wire the ownership components:
```java
@Produces
@ApplicationScoped
public ClusterManager clusterManager() {
    // ... existing nodeId and peerMap setup ...

    ClusterManager manager = new ClusterManager(nodeId, peerMap, config.virtualNodes(),
            config.heartbeatMissThreshold(), config.quorumEnforced(),
            Clock.systemUTC());

    if ("dynamic".equals(config.routing())) {
        ConsistentHashRing ring = new ConsistentHashRing(peerMap.keySet(), config.virtualNodes());
        DynamicOwnershipResolver resolver = new DynamicOwnershipResolver(ring, peerMap);
        manager.setResolver(resolver);
    }

    return manager;
}
```

Add `@Inject OwnershipConfig ownershipConfig;` field.

Add a producer for `WriteFrequencyTracker` (only when routing=dynamic):
```java
@Produces
@ApplicationScoped
public WriteFrequencyTracker writeFrequencyTracker() {
    if (!"dynamic".equals(config.routing())) {
        return null;
    }
    return new WriteFrequencyTracker(
            ownershipConfig.bucketCount(),
            Duration.ofSeconds(ownershipConfig.windowSeconds() / ownershipConfig.bucketCount()),
            Clock.systemUTC());
}
```

Add import `java.time.Duration;`.

- [ ] **Step 7: Write HeartbeatOwnershipTest**

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneId;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class HeartbeatOwnershipTest {

    private static final Instant NOW = Instant.parse("2026-10-06T12:00:00Z");
    private static final Clock CLOCK = Clock.fixed(NOW, ZoneId.of("UTC"));

    private ClusterManager managerWithPeers(String localNodeId, String... peers) {
        Map<String, String> peerMap = new LinkedHashMap<>();
        for (String p : peers) {
            peerMap.put(p, p + ":8080");
        }
        return new ClusterManager(localNodeId, peerMap, 128, 2, true, CLOCK);
    }

    @Test
    void heartbeatResponseIncludesOwnershipClaims() {
        UUID ch = UUID.randomUUID();
        var claims = Map.of(ch, new OwnershipClaim("node-1", 42));
        var resp = new HeartbeatResponse("node-1", NOW, "abc", "UP", claims);
        assertThat(resp.ownershipClaims()).containsKey(ch);
        assertThat(resp.ownershipClaims().get(ch).writeCount()).isEqualTo(42);
    }

    @Test
    void backwardCompatibleConstructorSetsEmptyClaims() {
        var resp = new HeartbeatResponse("node-1", NOW, "abc", "UP");
        assertThat(resp.ownershipClaims()).isEmpty();
    }

    @Test
    void updateRemoteOwnershipMergesPeerClaims() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        Map<String, String> addresses = new LinkedHashMap<>();
        addresses.put("node-1", "node-1:8080");
        addresses.put("node-2", "node-2:8080");
        addresses.put("node-3", "node-3:8080");
        var resolver = new DynamicOwnershipResolver(ring, addresses);
        mgr.setResolver(resolver);

        UUID ch = UUID.randomUUID();
        mgr.updateRemoteOwnership("node-2", Map.of(ch, new OwnershipClaim("node-2", 100)));

        assertThat(mgr.owner(ch).nodeId()).isEqualTo("node-2");
    }

    @Test
    void peerDeathClearsOwnershipClaims() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        Map<String, String> addresses = new LinkedHashMap<>();
        addresses.put("node-1", "node-1:8080");
        addresses.put("node-2", "node-2:8080");
        addresses.put("node-3", "node-3:8080");
        var resolver = new DynamicOwnershipResolver(ring, addresses);
        mgr.setResolver(resolver);

        UUID ch = UUID.randomUUID();
        mgr.updateRemoteOwnership("node-2", Map.of(ch, new OwnershipClaim("node-2", 50)));
        assertThat(mgr.owner(ch).nodeId()).isEqualTo("node-2");

        // peer goes DEAD (2 misses with threshold=2)
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");

        // channel should revert to hash ring (not necessarily node-2)
        assertThat(resolver.getClaim(ch)).isNull();
    }

    @Test
    void heartbeatServicePropagatesOwnershipOnTick() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2");
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2"), 128);
        Map<String, String> addresses = new LinkedHashMap<>();
        addresses.put("node-1", "node-1:8080");
        addresses.put("node-2", "node-2:8080");
        var resolver = new DynamicOwnershipResolver(ring, addresses);
        mgr.setResolver(resolver);

        UUID ch = UUID.randomUUID();
        var remoteClaims = Map.of(ch, new OwnershipClaim("node-2", 75));
        var heartbeatService = new HeartbeatService(mgr,
                peer -> new HeartbeatResponse("node-2", NOW, mgr.ringHash(), "UP", remoteClaims));

        heartbeatService.tick();

        assertThat(mgr.owner(ch).nodeId()).isEqualTo("node-2");
    }
}
```

- [ ] **Step 8: Update existing tests for backward compatibility**

Update `WriteRoutingDecoratorTest` — existing tests use the 4-arg constructor, which still works. Add one test for tracker recording:

```java
@Test
void recordsWriteToTrackerAfterSuccessfulDispatch() {
    var tracker = new WriteFrequencyTracker(10, Duration.ofSeconds(30),
            Clock.fixed(Instant.parse("2026-10-06T12:00:00Z"), ZoneId.of("UTC")));
    var decorator = new WriteRoutingDecorator(mockDelegate, clusterManager,
            proxyClient, true, tracker);

    UUID channelId = UUID.randomUUID();
    // make owner local
    when(clusterManager.owner(channelId)).thenReturn(new NodeInfo("local", "local:8080"));
    when(clusterManager.isLocal(any())).thenReturn(true);
    when(clusterManager.canServeWrites()).thenReturn(true);
    when(mockDelegate.dispatch(any())).thenReturn(mockResult);

    decorator.dispatch(buildDispatch(channelId));

    assertThat(tracker.getCount(channelId)).isEqualTo(1);
}
```

Add imports for `Duration`, `Clock`, `Instant`, `ZoneId`.

- [ ] **Step 9: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: PASS — all tests green (existing + new)

- [ ] **Step 10: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: PASS — full build green (verify HeartbeatResponse backward compat across modules)

- [ ] **Step 11: Commit**

```bash
git add cluster/
git commit -m "feat(#475): wire dynamic ownership into cluster — ClusterManager, heartbeat, routing decorator

- ClusterManager gains ownership methods (setResolver, updateRemoteOwnership, clearPeerOwnership)
- HeartbeatResponse carries OwnershipClaim map (backward-compatible 4-arg constructor)
- HeartbeatService propagates peer claims on tick
- WriteRoutingDecorator records writes to tracker after dispatch
- RelayProducer creates tracker and resolver when routing=dynamic
- OwnershipConfig for ownership tuning properties

Refs #475

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-07-dynamic-ownership-design.md] — design spec this plan implements
- `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterManager.java` — ownership delegation point
- `cluster/src/main/java/io/casehub/qhorus/cluster/ConsistentHashRing.java` — hash ring fallback
- `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java` — write interception
- `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatService.java` — heartbeat protocol
- `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatResponse.java` — heartbeat payload
- `cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java` — existing config
- `cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java` — CDI wiring
- `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java` — REST endpoint
- `cluster/src/test/java/io/casehub/qhorus/cluster/ClusterManagerTest.java` — test patterns
- Decisions D32-D38
- GitHub #475
