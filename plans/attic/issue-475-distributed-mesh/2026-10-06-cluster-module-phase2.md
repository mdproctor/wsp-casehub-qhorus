# Cluster Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #475 — epic: distributed qhorus mesh — standalone service with clustering
**Issue group:** #475

**Goal:** Build `casehub-qhorus-cluster`, a new Maven module that adds
multi-node clustering to qhorus via consistent hash ring, heartbeat
failure detection, and write routing decorators.

**Architecture:** CDI decorators on `MessageDispatcher` and `ChannelManager`
transparently route writes to the channel-owning node via consistent hash
ring. Internal node-to-node RPC uses Quarkus REST Client. Heartbeat service
detects node failures and triggers ring recalculation. All beans gated
behind `casehub.qhorus.cluster.enabled=true`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Quarkus REST Client Jackson,
quarkus-scheduler, SHA-256 consistent hashing

## Global Constraints

- Java 21 source level (running on Java 26 JVM)
- `groupId: io.casehub`, `artifactId: casehub-qhorus-cluster`, version `${project.version}`
- Package: `io.casehub.qhorus.cluster`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All cluster beans gated by `@IfBuildProperty(name = "casehub.qhorus.cluster.enabled", stringValue = "true", enableIfMissing = false)`
- Tests are CDI-free unit tests (Mockito + AssertJ) unless noted otherwise
- Commits reference #475: `Refs #475`

---

## Batch 1: Foundation — ConsistentHashRing + ClusterConfig + Module Scaffold

After this batch: the module exists, compiles, and has a fully tested
hash ring implementation. No CDI, no networking — pure algorithm.

### Task 1: Module scaffold and ConsistentHashRing

**Files:**
- Create: `cluster/pom.xml`
- Modify: `pom.xml` (parent — add `<module>cluster</module>`)
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ConsistentHashRing.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/NodeInfo.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/NodeState.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/ConsistentHashRingTest.java`

**Interfaces:**
- Consumes: nothing (first task)
- Produces:
  - `ConsistentHashRing(Set<String> members, int virtualNodes)` — constructor
  - `String owner(UUID channelId)` — returns nodeId owning this channel
  - `ConsistentHashRing withNode(String nodeId)` — returns new ring with node added
  - `ConsistentHashRing withoutNode(String nodeId)` — returns new ring with node removed
  - `Set<String> members()` — current ring members
  - `String ringHash()` — SHA-256 of sorted member list
  - `NodeInfo(String nodeId, String address)` — record
  - `NodeState` — enum: `ALIVE`, `SUSPECT`, `DEAD`

- [ ] **Step 1: Create module pom.xml**

Create `cluster/pom.xml`:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-qhorus-cluster</artifactId>
  <name>CaseHub Qhorus Cluster</name>
  <description>Multi-node clustering for qhorus mesh — hash ring, heartbeat, write routing</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 2: Register module in parent pom**

In `pom.xml` (project root), add `<module>cluster</module>` after the
`<module>mesh</module>` entry:

```xml
    <module>mesh</module>
    <module>cluster</module>
    <module>examples</module>
```

- [ ] **Step 3: Write ConsistentHashRing failing tests**

Create `cluster/src/test/java/io/casehub/qhorus/cluster/ConsistentHashRingTest.java`:

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class ConsistentHashRingTest {

    @Test
    void ownerIsDeterministic() {
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        UUID channelId = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");
        String owner1 = ring.owner(channelId);
        String owner2 = ring.owner(channelId);
        assertThat(owner1).isEqualTo(owner2);
    }

    @Test
    void allChannelsHaveAnOwner() {
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        for (int i = 0; i < 100; i++) {
            assertThat(ring.owner(UUID.randomUUID())).isIn("node-1", "node-2", "node-3");
        }
    }

    @Test
    void distributionIsReasonablyEven() {
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        Map<String, Integer> counts = new HashMap<>();
        int total = 10_000;
        for (int i = 0; i < total; i++) {
            counts.merge(ring.owner(UUID.randomUUID()), 1, Integer::sum);
        }
        double expected = total / 3.0;
        for (int count : counts.values()) {
            assertThat((double) count).isCloseTo(expected, withinPercentage(15));
        }
    }

    @Test
    void withNodeAddsToRing() {
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2"), 128);
        var expanded = ring.withNode("node-3");
        assertThat(expanded.members()).containsExactlyInAnyOrder("node-1", "node-2", "node-3");
        assertThat(ring.members()).containsExactlyInAnyOrder("node-1", "node-2");
    }

    @Test
    void withoutNodeRemovesFromRing() {
        var ring = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        var shrunk = ring.withoutNode("node-2");
        assertThat(shrunk.members()).containsExactlyInAnyOrder("node-1", "node-3");
    }

    @Test
    void onlyFractionOfChannelsMoveOnNodeRemoval() {
        var ring3 = new ConsistentHashRing(Set.of("node-1", "node-2", "node-3"), 128);
        var ring2 = ring3.withoutNode("node-2");
        List<UUID> channels = new ArrayList<>();
        for (int i = 0; i < 1000; i++) channels.add(UUID.randomUUID());
        long moved = channels.stream()
                .filter(ch -> !ring3.owner(ch).equals(ring2.owner(ch)))
                .count();
        assertThat(moved).isBetween(200L, 500L);
    }

    @Test
    void ringHashIsDeterministic() {
        var ring1 = new ConsistentHashRing(Set.of("node-1", "node-2"), 128);
        var ring2 = new ConsistentHashRing(Set.of("node-2", "node-1"), 128);
        assertThat(ring1.ringHash()).isEqualTo(ring2.ringHash());
    }

    @Test
    void ringHashChangesWithMembership() {
        var ring2 = new ConsistentHashRing(Set.of("node-1", "node-2"), 128);
        var ring3 = ring2.withNode("node-3");
        assertThat(ring2.ringHash()).isNotEqualTo(ring3.ringHash());
    }

    @Test
    void singleNodeOwnsEverything() {
        var ring = new ConsistentHashRing(Set.of("node-1"), 128);
        for (int i = 0; i < 100; i++) {
            assertThat(ring.owner(UUID.randomUUID())).isEqualTo("node-1");
        }
    }

    @Test
    void emptyRingThrows() {
        assertThatThrownBy(() -> new ConsistentHashRing(Set.of(), 128))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml`
Expected: Compilation error (classes don't exist yet)

- [ ] **Step 5: Create NodeInfo, NodeState, and ConsistentHashRing**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/NodeInfo.java`:

```java
package io.casehub.qhorus.cluster;

public record NodeInfo(String nodeId, String address) {
    public NodeInfo {
        if (nodeId == null || nodeId.isBlank()) {
            throw new IllegalArgumentException("nodeId must not be blank");
        }
        if (address == null || address.isBlank()) {
            throw new IllegalArgumentException("address must not be blank");
        }
    }
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/NodeState.java`:

```java
package io.casehub.qhorus.cluster;

public enum NodeState {
    ALIVE, SUSPECT, DEAD
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ConsistentHashRing.java`:

```java
package io.casehub.qhorus.cluster;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;
import java.util.stream.Collectors;

public final class ConsistentHashRing {

    private final NavigableMap<Long, String> ring;
    private final int virtualNodes;
    private final Set<String> members;
    private final String ringHash;

    public ConsistentHashRing(Set<String> members, int virtualNodes) {
        if (members == null || members.isEmpty()) {
            throw new IllegalArgumentException("Ring must have at least one member");
        }
        this.virtualNodes = virtualNodes;
        this.members = Set.copyOf(members);
        this.ring = buildRing(this.members, virtualNodes);
        this.ringHash = computeRingHash(this.members);
    }

    public String owner(UUID channelId) {
        long hash = hash(channelId.toString().getBytes(StandardCharsets.UTF_8));
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
        return entry != null ? entry.getValue() : ring.firstEntry().getValue();
    }

    public ConsistentHashRing withNode(String nodeId) {
        var expanded = new HashSet<>(members);
        expanded.add(nodeId);
        return new ConsistentHashRing(expanded, virtualNodes);
    }

    public ConsistentHashRing withoutNode(String nodeId) {
        var shrunk = new HashSet<>(members);
        shrunk.remove(nodeId);
        return new ConsistentHashRing(shrunk, virtualNodes);
    }

    public Set<String> members() {
        return members;
    }

    public String ringHash() {
        return ringHash;
    }

    static long hash(byte[] data) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] digest = md.digest(data);
            long h = 0;
            for (int i = 0; i < 8; i++) {
                h = (h << 8) | (digest[i] & 0xFF);
            }
            return h;
        } catch (NoSuchAlgorithmException e) {
            throw new AssertionError("SHA-256 not available", e);
        }
    }

    private static NavigableMap<Long, String> buildRing(Set<String> members, int vNodes) {
        NavigableMap<Long, String> ring = new TreeMap<>();
        for (String nodeId : members) {
            for (int i = 0; i < vNodes; i++) {
                byte[] key = (nodeId + "#" + i).getBytes(StandardCharsets.UTF_8);
                ring.put(hash(key), nodeId);
            }
        }
        return Collections.unmodifiableNavigableMap(ring);
    }

    private static String computeRingHash(Set<String> members) {
        String sorted = members.stream().sorted().collect(Collectors.joining(","));
        byte[] digest;
        try {
            digest = MessageDigest.getInstance("SHA-256")
                    .digest(sorted.getBytes(StandardCharsets.UTF_8));
        } catch (NoSuchAlgorithmException e) {
            throw new AssertionError("SHA-256 not available", e);
        }
        StringBuilder sb = new StringBuilder(digest.length * 2);
        for (byte b : digest) {
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml`
Expected: All 10 tests PASS

- [ ] **Step 7: Commit**

```bash
git add cluster/ pom.xml
git commit -m "feat(#475): add casehub-qhorus-cluster module with ConsistentHashRing

Refs #475"
```

---

## Batch 2: ClusterManager + HeartbeatService

After this batch: peer state machine works, ring recalculation on
membership change, heartbeat miss detection — all unit tested.

### Task 2: PeerState + ClusterManager

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/PeerState.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterManager.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterMembershipEvent.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/QuorumViolationException.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/ClusterManagerTest.java`

**Interfaces:**
- Consumes: `ConsistentHashRing`, `NodeInfo`, `NodeState` (from Task 1)
- Produces:
  - `PeerState(NodeInfo nodeInfo, NodeState state, java.time.Instant lastHeartbeat, int missCount)` — record
  - `RelayConfig` — `@ConfigMapping(prefix = "casehub.qhorus.cluster")`
  - `ClusterManager.owner(UUID channelId) → NodeInfo`
  - `ClusterManager.isLocal(NodeInfo) → boolean`
  - `ClusterManager.canServeWrites() → boolean`
  - `ClusterManager.recordHeartbeat(String nodeId)`
  - `ClusterManager.recordMiss(String nodeId)`
  - `ClusterManager.handlePeerDeparture(String nodeId)`
  - `ClusterManager.nodeId() → String`
  - `ClusterManager.ringHash() → String`
  - `ClusterManager.clusterSize() → int`
  - `ClusterManager.expectedSize() → int`
  - `ClusterManager.peerStates() → Map<String, PeerState>`
  - `ClusterMembershipEvent(String nodeId, NodeState previousState, NodeState newState, java.time.Instant timestamp)` — record
  - `QuorumViolationException extends RuntimeException`

- [ ] **Step 1: Write ClusterManager failing tests**

Create `cluster/src/test/java/io/casehub/qhorus/cluster/ClusterManagerTest.java`:

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class ClusterManagerTest {

    private static final Instant NOW = Instant.parse("2026-10-06T12:00:00Z");
    private Clock clock;

    @BeforeEach
    void setUp() {
        clock = Clock.fixed(NOW, ZoneId.of("UTC"));
    }

    private ClusterManager managerWithPeers(String localNodeId, String... peers) {
        Map<String, String> peerMap = new LinkedHashMap<>();
        for (String p : peers) {
            peerMap.put(p, p + ":8080");
        }
        return new ClusterManager(localNodeId, peerMap, 128, 2, true, clock);
    }

    @Test
    void ownerReturnsSameNodeForSameChannel() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        UUID ch = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");
        assertThat(mgr.owner(ch).nodeId()).isEqualTo(mgr.owner(ch).nodeId());
    }

    @Test
    void isLocalReturnsTrueForLocalNode() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        UUID ch = UUID.randomUUID();
        NodeInfo owner = mgr.owner(ch);
        if (owner.nodeId().equals("node-1")) {
            assertThat(mgr.isLocal(owner)).isTrue();
        } else {
            assertThat(mgr.isLocal(owner)).isFalse();
        }
    }

    @Test
    void canServeWritesWithAllAlive() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        assertThat(mgr.canServeWrites()).isTrue();
    }

    @Test
    void canServeWritesAfterOneDead() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");
        assertThat(mgr.canServeWrites()).isTrue();
    }

    @Test
    void cannotServeWritesAfterTwoDead() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-3");
        mgr.recordMiss("node-3");
        assertThat(mgr.canServeWrites()).isFalse();
    }

    @Test
    void heartbeatRecoversDeadNode() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");
        assertThat(mgr.peerStates().get("node-2").state()).isEqualTo(NodeState.DEAD);
        mgr.recordHeartbeat("node-2");
        assertThat(mgr.peerStates().get("node-2").state()).isEqualTo(NodeState.ALIVE);
    }

    @Test
    void missTransitionsToSuspectThenDead() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        assertThat(mgr.peerStates().get("node-2").state()).isEqualTo(NodeState.ALIVE);
        mgr.recordMiss("node-2");
        assertThat(mgr.peerStates().get("node-2").state()).isEqualTo(NodeState.SUSPECT);
        mgr.recordMiss("node-2");
        assertThat(mgr.peerStates().get("node-2").state()).isEqualTo(NodeState.DEAD);
    }

    @Test
    void deadNodeRemovedFromRing() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        UUID ch = UUID.randomUUID();
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");
        assertThat(mgr.owner(ch).nodeId()).isIn("node-1", "node-3");
    }

    @Test
    void recoveredNodeReaddedToRing() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");
        mgr.recordHeartbeat("node-2");
        boolean hasNode2 = false;
        for (int i = 0; i < 1000; i++) {
            if (mgr.owner(UUID.randomUUID()).nodeId().equals("node-2")) {
                hasNode2 = true;
                break;
            }
        }
        assertThat(hasNode2).isTrue();
    }

    @Test
    void handlePeerDepartureRemovesImmediately() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        mgr.handlePeerDeparture("node-2");
        assertThat(mgr.peerStates().get("node-2").state()).isEqualTo(NodeState.DEAD);
    }

    @Test
    void quorumDisabledWhenSingleNode() {
        var mgr = managerWithPeers("node-1", "node-1");
        assertThat(mgr.canServeWrites()).isTrue();
    }

    @Test
    void shutdownSendsLeaveToAllAlivePeers() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        List<String> leaveSent = new ArrayList<>();
        mgr.shutdown(node -> leaveSent.add(node.nodeId()));
        assertThat(leaveSent).containsExactlyInAnyOrder("node-2", "node-3");
    }

    @Test
    void shutdownSkipsDeadPeers() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");
        List<String> leaveSent = new ArrayList<>();
        mgr.shutdown(node -> leaveSent.add(node.nodeId()));
        assertThat(leaveSent).containsExactly("node-3");
    }

    @Test
    void membershipEventsRecorded() {
        var mgr = managerWithPeers("node-1", "node-1", "node-2", "node-3");
        mgr.recordMiss("node-2");
        mgr.recordMiss("node-2");
        List<ClusterMembershipEvent> events = mgr.drainEvents();
        assertThat(events).hasSize(2);
        assertThat(events.get(0).newState()).isEqualTo(NodeState.SUSPECT);
        assertThat(events.get(1).newState()).isEqualTo(NodeState.DEAD);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml -Dtest=ClusterManagerTest`
Expected: Compilation error

- [ ] **Step 3: Create supporting types and ClusterManager**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/PeerState.java`:

```java
package io.casehub.qhorus.cluster;

import java.time.Instant;

public record PeerState(NodeInfo nodeInfo, NodeState state, Instant lastHeartbeat, int missCount) {}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterMembershipEvent.java`:

```java
package io.casehub.qhorus.cluster;

import java.time.Instant;

public record ClusterMembershipEvent(
        String nodeId,
        NodeState previousState,
        NodeState newState,
        Instant timestamp) {}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/QuorumViolationException.java`:

```java
package io.casehub.qhorus.cluster;

public class QuorumViolationException extends RuntimeException {
    public QuorumViolationException(String message) {
        super(message);
    }
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterManager.java`:

```java
package io.casehub.qhorus.cluster;

import java.time.Clock;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicReference;

public class ClusterManager {

    private final String localNodeId;
    private final Map<String, NodeInfo> configuredPeers;
    private final int virtualNodes;
    private final int missThreshold;
    private final boolean quorumEnforced;
    private final Clock clock;
    private final AtomicReference<ConsistentHashRing> ringRef;
    private final ConcurrentHashMap<String, PeerState> peerStates = new ConcurrentHashMap<>();
    private final ConcurrentLinkedQueue<ClusterMembershipEvent> pendingEvents = new ConcurrentLinkedQueue<>();

    public ClusterManager(String localNodeId, Map<String, String> peers,
                          int virtualNodes, int missThreshold,
                          boolean quorumEnforced, Clock clock) {
        this.localNodeId = localNodeId;
        this.virtualNodes = virtualNodes;
        this.missThreshold = missThreshold;
        this.quorumEnforced = quorumEnforced;
        this.clock = clock;

        this.configuredPeers = new LinkedHashMap<>();
        for (var entry : peers.entrySet()) {
            configuredPeers.put(entry.getKey(), new NodeInfo(entry.getKey(), entry.getValue()));
        }

        Set<String> memberIds = new HashSet<>(peers.keySet());
        this.ringRef = new AtomicReference<>(new ConsistentHashRing(memberIds, virtualNodes));

        Instant now = clock.instant();
        for (String peerId : peers.keySet()) {
            if (!peerId.equals(localNodeId)) {
                peerStates.put(peerId, new PeerState(
                        configuredPeers.get(peerId), NodeState.ALIVE, now, 0));
            }
        }
    }

    public NodeInfo owner(UUID channelId) {
        String nodeId = ringRef.get().owner(channelId);
        NodeInfo info = configuredPeers.get(nodeId);
        if (info == null) {
            return new NodeInfo(nodeId, nodeId + ":8080");
        }
        return info;
    }

    public boolean isLocal(NodeInfo node) {
        return localNodeId.equals(node.nodeId());
    }

    public boolean canServeWrites() {
        if (!quorumEnforced || configuredPeers.size() <= 1) {
            return true;
        }
        long reachable = 1; // self
        for (PeerState ps : peerStates.values()) {
            if (ps.state() != NodeState.DEAD) {
                reachable++;
            }
        }
        return reachable > configuredPeers.size() / 2;
    }

    public void recordHeartbeat(String nodeId) {
        peerStates.computeIfPresent(nodeId, (id, old) -> {
            Instant now = clock.instant();
            if (old.state() == NodeState.DEAD) {
                emitEvent(id, old.state(), NodeState.ALIVE);
                readdToRing(id);
            } else if (old.state() == NodeState.SUSPECT) {
                emitEvent(id, old.state(), NodeState.ALIVE);
            }
            return new PeerState(old.nodeInfo(), NodeState.ALIVE, now, 0);
        });
    }

    public void recordMiss(String nodeId) {
        peerStates.computeIfPresent(nodeId, (id, old) -> {
            int newMissCount = old.missCount() + 1;
            NodeState newState;
            if (newMissCount >= missThreshold) {
                newState = NodeState.DEAD;
            } else {
                newState = NodeState.SUSPECT;
            }
            if (newState != old.state()) {
                emitEvent(id, old.state(), newState);
                if (newState == NodeState.DEAD) {
                    removeFromRing(id);
                }
            }
            return new PeerState(old.nodeInfo(), newState, old.lastHeartbeat(), newMissCount);
        });
    }

    public void handlePeerDeparture(String nodeId) {
        peerStates.computeIfPresent(nodeId, (id, old) -> {
            if (old.state() != NodeState.DEAD) {
                emitEvent(id, old.state(), NodeState.DEAD);
                removeFromRing(id);
            }
            return new PeerState(old.nodeInfo(), NodeState.DEAD, old.lastHeartbeat(), old.missCount());
        });
    }

    public String nodeId() { return localNodeId; }
    public String ringHash() { return ringRef.get().ringHash(); }
    public int clusterSize() { return (int) peerStates.values().stream().filter(p -> p.state() != NodeState.DEAD).count() + 1; }
    public int expectedSize() { return configuredPeers.size(); }
    public Map<String, PeerState> peerStates() { return Map.copyOf(peerStates); }

    public void shutdown(java.util.function.Consumer<NodeInfo> leaveSender) {
        for (var entry : peerStates.entrySet()) {
            if (entry.getValue().state() != NodeState.DEAD) {
                try {
                    leaveSender.accept(entry.getValue().nodeInfo());
                } catch (Exception ignored) {}
            }
        }
    }

    public List<ClusterMembershipEvent> drainEvents() {
        List<ClusterMembershipEvent> events = new ArrayList<>();
        ClusterMembershipEvent e;
        while ((e = pendingEvents.poll()) != null) {
            events.add(e);
        }
        return events;
    }

    private void emitEvent(String nodeId, NodeState previous, NodeState next) {
        pendingEvents.add(new ClusterMembershipEvent(nodeId, previous, next, clock.instant()));
    }

    private void removeFromRing(String nodeId) {
        ringRef.updateAndGet(ring -> {
            if (ring.members().contains(nodeId)) {
                return ring.withoutNode(nodeId);
            }
            return ring;
        });
    }

    private void readdToRing(String nodeId) {
        ringRef.updateAndGet(ring -> {
            if (!ring.members().contains(nodeId)) {
                return ring.withNode(nodeId);
            }
            return ring;
        });
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml -Dtest=ClusterManagerTest`
Expected: All 12 tests PASS

- [ ] **Step 5: Commit**

```bash
git add cluster/
git commit -m "feat(#475): add ClusterManager with peer state machine and quorum check

Refs #475"
```

### Task 3: HeartbeatService

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatResponse.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatService.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/HeartbeatServiceTest.java`

**Interfaces:**
- Consumes: `ClusterManager.recordHeartbeat()`, `ClusterManager.recordMiss()`, `ClusterManager.peerStates()`, `ClusterManager.ringHash()`, `NodeInfo`, `PeerState`, `NodeState` (from Task 2)
- Produces:
  - `HeartbeatResponse(String nodeId, java.time.Instant timestamp, String ringHash, String status)` — record
  - `HeartbeatService.tick()` — polls all alive/suspect peers
  - `HeartbeatService.buildLocalResponse() → HeartbeatResponse`

- [ ] **Step 1: Write HeartbeatService failing tests**

Create `cluster/src/test/java/io/casehub/qhorus/cluster/HeartbeatServiceTest.java`:

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import java.time.*;
import java.util.*;
import java.util.function.Function;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class HeartbeatServiceTest {

    private ClusterManager clusterManager;
    private Function<NodeInfo, HeartbeatResponse> heartbeatCaller;
    private HeartbeatService service;
    private static final Instant NOW = Instant.parse("2026-10-06T12:00:00Z");

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        clusterManager = Mockito.spy(new ClusterManager(
                "node-1",
                Map.of("node-1", "node-1:8080", "node-2", "node-2:8080", "node-3", "node-3:8080"),
                128, 2, true,
                Clock.fixed(NOW, ZoneId.of("UTC"))));
        heartbeatCaller = mock(Function.class);
        service = new HeartbeatService(clusterManager, heartbeatCaller);
    }

    @Test
    void tickRecordsHeartbeatOnSuccess() {
        var resp = new HeartbeatResponse("node-2", NOW, clusterManager.ringHash(), "UP");
        when(heartbeatCaller.apply(any())).thenReturn(resp);
        service.tick();
        verify(clusterManager).recordHeartbeat("node-2");
        verify(clusterManager).recordHeartbeat("node-3");
    }

    @Test
    void tickRecordsMissOnFailure() {
        when(heartbeatCaller.apply(any())).thenThrow(new RuntimeException("connection refused"));
        service.tick();
        verify(clusterManager).recordMiss("node-2");
        verify(clusterManager).recordMiss("node-3");
    }

    @Test
    void tickDoesNotPollDeadNodes() {
        clusterManager.recordMiss("node-2");
        clusterManager.recordMiss("node-2");
        assertThat(clusterManager.peerStates().get("node-2").state()).isEqualTo(NodeState.DEAD);
        reset(heartbeatCaller);
        var resp = new HeartbeatResponse("node-3", NOW, clusterManager.ringHash(), "UP");
        when(heartbeatCaller.apply(any())).thenReturn(resp);
        service.tick();
        verify(heartbeatCaller, times(1)).apply(any());
    }

    @Test
    void buildLocalResponseReturnsNodeInfo() {
        HeartbeatResponse resp = service.buildLocalResponse();
        assertThat(resp.nodeId()).isEqualTo("node-1");
        assertThat(resp.ringHash()).isEqualTo(clusterManager.ringHash());
        assertThat(resp.status()).isEqualTo("UP");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml -Dtest=HeartbeatServiceTest`
Expected: Compilation error

- [ ] **Step 3: Create HeartbeatResponse and HeartbeatService**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatResponse.java`:

```java
package io.casehub.qhorus.cluster;

import java.time.Instant;

public record HeartbeatResponse(String nodeId, Instant timestamp, String ringHash, String status) {}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatService.java`:

```java
package io.casehub.qhorus.cluster;

import java.time.Instant;
import java.util.function.Function;

public class HeartbeatService {

    private static final org.jboss.logging.Logger LOG = org.jboss.logging.Logger.getLogger(HeartbeatService.class);

    private final ClusterManager clusterManager;
    private final Function<NodeInfo, HeartbeatResponse> heartbeatCaller;

    public HeartbeatService(ClusterManager clusterManager,
                            Function<NodeInfo, HeartbeatResponse> heartbeatCaller) {
        this.clusterManager = clusterManager;
        this.heartbeatCaller = heartbeatCaller;
    }

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
                if (resp != null && resp.ringHash() != null
                        && !resp.ringHash().equals(localRingHash)) {
                    LOG.warnf("Ring disagreement with %s — local=%s remote=%s",
                            entry.getKey(), localRingHash, resp.ringHash());
                }
            } catch (Exception e) {
                clusterManager.recordMiss(entry.getKey());
            }
        }
    }

    public HeartbeatResponse buildLocalResponse() {
        return new HeartbeatResponse(
                clusterManager.nodeId(),
                Instant.now(),
                clusterManager.ringHash(),
                "UP");
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml -Dtest=HeartbeatServiceTest`
Expected: All 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add cluster/
git commit -m "feat(#475): add HeartbeatService with peer polling and failure detection

Refs #475"
```

---

## Batch 3: Write Routing — Decorators + Internal RPC

After this batch: write routing works end-to-end — decorators intercept
dispatch and channel mutations, proxy to remote nodes via internal HTTP,
quorum enforcement active. Also includes the preAssignedId API change.

### Task 4: PreAssignedId on ChannelCreateRequest + Channel.fromRequest

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java` — add `preAssignedId` field + builder method
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — `fromRequest()` uses preAssignedId when set
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateHelper.java` — pass preAssignedId through
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequestBuilderTest.java` — add test for preAssignedId

**Interfaces:**
- Consumes: existing `ChannelCreateRequest`, `Channel.fromRequest()`, `ChannelCreateHelper`
- Produces: `ChannelCreateRequest.preAssignedId() → UUID` (nullable), `ChannelCreateRequest.Builder.preAssignedId(UUID) → Builder`

- [ ] **Step 1: Write failing test for preAssignedId**

Add to `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequestBuilderTest.java`:

```java
@Test
void preAssignedIdIsUsedWhenSet() {
    UUID preId = UUID.fromString("12345678-1234-1234-1234-123456789abc");
    var req = ChannelCreateRequest.builder("test-channel")
            .preAssignedId(preId)
            .build();
    assertThat(req.preAssignedId()).isEqualTo(preId);
}

@Test
void preAssignedIdDefaultsToNull() {
    var req = ChannelCreateRequest.builder("test-channel").build();
    assertThat(req.preAssignedId()).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelCreateRequestBuilderTest`
Expected: Compilation error (preAssignedId doesn't exist)

- [ ] **Step 3: Add preAssignedId to ChannelCreateRequest**

Add `UUID preAssignedId` as the 24th parameter to the canonical record constructor
(after `outboundDestination`). Add a builder field and setter. Add backward-compatible
constructors that pass `null` for the new field. Update `build()` to pass the field.

In `Channel.fromRequest()`: replace `UUID.randomUUID()` with
`req.preAssignedId() != null ? req.preAssignedId() : UUID.randomUUID()`.

In `ChannelCreateHelper.createWithBinding()`: replace `UUID.randomUUID()` with
the same pattern using `req.preAssignedId()`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelCreateRequestBuilderTest`
Expected: PASS

Then run full build to verify no compilation breaks:
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add api/ runtime-core/ runtime/
git commit -m "feat(#475): add preAssignedId to ChannelCreateRequest for cluster routing

Refs #475"
```

### Task 5: WriteRoutingDecorator + ChannelManagerDecorator

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ChannelManagerDecorator.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/ChannelManagerDecoratorTest.java`

**Interfaces:**
- Consumes: `ClusterManager.owner()`, `ClusterManager.isLocal()`, `ClusterManager.canServeWrites()` (from Task 2), `MessageDispatcher`, `ChannelManager` (from qhorus API), `ChannelCreateRequest.preAssignedId()` (from Task 4)
- Produces:
  - `WriteRoutingDecorator implements MessageDispatcher` — `@Decorator`
  - `ChannelManagerDecorator implements ChannelManager` — `@Decorator`
  - `WriteProxyClient` — wraps internal HTTP client, methods: `dispatch(NodeInfo, MessageDispatch) → DispatchResult`, `createChannel(NodeInfo, ChannelCreateRequest) → Channel`, `deleteChannel(NodeInfo, UUID, boolean) → long`, `pauseChannel(NodeInfo, UUID) → Channel`, `resumeChannel(NodeInfo, UUID) → Channel`

- [ ] **Step 1: Write WriteRoutingDecorator failing tests**

Create `cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.message.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class WriteRoutingDecoratorTest {

    private MessageDispatcher delegate;
    private ClusterManager clusterManager;
    private WriteProxyClient proxyClient;
    private WriteRoutingDecorator decorator;
    private static final UUID CHANNEL_ID = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");

    @BeforeEach
    void setUp() {
        delegate = mock(MessageDispatcher.class);
        clusterManager = mock(ClusterManager.class);
        proxyClient = mock(WriteProxyClient.class);
        decorator = new WriteRoutingDecorator(delegate, clusterManager, proxyClient);
    }

    @Test
    void delegatesLocallyWhenOwner() {
        var localNode = new NodeInfo("node-1", "node-1:8080");
        when(clusterManager.canServeWrites()).thenReturn(true);
        when(clusterManager.owner(CHANNEL_ID)).thenReturn(localNode);
        when(clusterManager.isLocal(localNode)).thenReturn(true);
        var dispatch = MessageDispatch.builder().channelId(CHANNEL_ID)
                .sender("agent-1").type(MessageType.STATUS).content("test").build();
        var expectedResult = new DispatchResult(1L, CHANNEL_ID, "agent-1",
                MessageType.STATUS, null, null, null, null, null, null, null, 0, null);
        when(delegate.dispatch(dispatch)).thenReturn(expectedResult);

        DispatchResult result = decorator.dispatch(dispatch);

        assertThat(result).isEqualTo(expectedResult);
        verify(delegate).dispatch(dispatch);
        verifyNoInteractions(proxyClient);
    }

    @Test
    void proxiesToRemoteWhenNotOwner() {
        var remoteNode = new NodeInfo("node-2", "node-2:8080");
        when(clusterManager.canServeWrites()).thenReturn(true);
        when(clusterManager.owner(CHANNEL_ID)).thenReturn(remoteNode);
        when(clusterManager.isLocal(remoteNode)).thenReturn(false);
        var dispatch = MessageDispatch.builder().channelId(CHANNEL_ID)
                .sender("agent-1").type(MessageType.STATUS).content("test").build();
        var expectedResult = new DispatchResult(1L, CHANNEL_ID, "agent-1",
                MessageType.STATUS, null, null, null, null, null, null, null, 0, null);
        when(proxyClient.dispatch(remoteNode, dispatch)).thenReturn(expectedResult);

        DispatchResult result = decorator.dispatch(dispatch);

        assertThat(result).isEqualTo(expectedResult);
        verify(proxyClient).dispatch(remoteNode, dispatch);
        verifyNoInteractions(delegate);
    }

    @Test
    void throwsOnQuorumViolation() {
        when(clusterManager.canServeWrites()).thenReturn(false);
        var dispatch = MessageDispatch.builder().channelId(CHANNEL_ID)
                .sender("agent-1").type(MessageType.STATUS).content("test").build();

        assertThatThrownBy(() -> decorator.dispatch(dispatch))
                .isInstanceOf(QuorumViolationException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml -Dtest=WriteRoutingDecoratorTest`
Expected: Compilation error

- [ ] **Step 3: Create WriteProxyClient and decorators**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;

public class WriteProxyClient {

    public DispatchResult dispatch(NodeInfo target, MessageDispatch dispatch) {
        throw new UnsupportedOperationException("HTTP proxy not wired — requires InternalMeshClient");
    }

    public Channel createChannel(NodeInfo target, ChannelCreateRequest request) {
        throw new UnsupportedOperationException("HTTP proxy not wired — requires InternalMeshClient");
    }

    public long deleteChannel(NodeInfo target, java.util.UUID channelId, boolean force) {
        throw new UnsupportedOperationException("HTTP proxy not wired — requires InternalMeshClient");
    }

    public Channel pauseChannel(NodeInfo target, java.util.UUID channelId) {
        throw new UnsupportedOperationException("HTTP proxy not wired — requires InternalMeshClient");
    }

    public Channel resumeChannel(NodeInfo target, java.util.UUID channelId) {
        throw new UnsupportedOperationException("HTTP proxy not wired — requires InternalMeshClient");
    }
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;

public class WriteRoutingDecorator implements MessageDispatcher {

    private final MessageDispatcher delegate;
    private final ClusterManager clusterManager;
    private final WriteProxyClient proxyClient;

    public WriteRoutingDecorator(MessageDispatcher delegate,
                                 ClusterManager clusterManager,
                                 WriteProxyClient proxyClient) {
        this.delegate = delegate;
        this.clusterManager = clusterManager;
        this.proxyClient = proxyClient;
    }

    @Override
    public DispatchResult dispatch(MessageDispatch dispatch) {
        if (!clusterManager.canServeWrites()) {
            throw new QuorumViolationException("This node is in a minority partition and cannot serve writes");
        }
        NodeInfo owner = clusterManager.owner(dispatch.channelId());
        if (clusterManager.isLocal(owner)) {
            return delegate.dispatch(dispatch);
        }
        return proxyClient.dispatch(owner, dispatch);
    }
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ChannelManagerDecorator.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.channel.*;
import io.casehub.qhorus.api.message.MessageType;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

public class ChannelManagerDecorator implements ChannelManager {

    private final ChannelManager delegate;
    private final ClusterManager clusterManager;
    private final WriteProxyClient proxyClient;

    public ChannelManagerDecorator(ChannelManager delegate,
                                   ClusterManager clusterManager,
                                   WriteProxyClient proxyClient) {
        this.delegate = delegate;
        this.clusterManager = clusterManager;
        this.proxyClient = proxyClient;
    }

    @Override
    public Channel create(ChannelCreateRequest request) {
        if (!clusterManager.canServeWrites()) {
            throw new QuorumViolationException("minority partition");
        }
        UUID channelId = request.preAssignedId() != null
                ? request.preAssignedId() : UUID.randomUUID();
        var withId = ChannelCreateRequest.builder(request.name())
                .description(request.description())
                .semantic(request.semantic())
                .barrierContributors(request.barrierContributors())
                .allowedWriters(request.allowedWriters())
                .adminInstances(request.adminInstances())
                .rateLimitPerChannel(request.rateLimitPerChannel())
                .rateLimitPerInstance(request.rateLimitPerInstance())
                .allowedTypes(request.allowedTypes())
                .deniedTypes(request.deniedTypes())
                .spaceId(request.spaceId())
                .reviewerInstances(request.reviewerInstances())
                .protocols(request.protocols())
                .protocolParticipants(request.protocolParticipants())
                .trackDelivery(request.trackDelivery())
                .enforcementMode(request.enforcementMode())
                .enforcementExclusions(request.enforcementExclusions())
                .routingTrustThreshold(request.routingTrustThreshold())
                .metadata(request.metadata())
                .preAssignedId(channelId)
                .build();
        NodeInfo owner = clusterManager.owner(channelId);
        if (clusterManager.isLocal(owner)) {
            return delegate.create(withId);
        }
        return proxyClient.createChannel(owner, withId);
    }

    @Override
    public FindOrCreateResult findOrCreate(ChannelCreateRequest request) {
        return delegate.findOrCreate(request);
    }

    @Override
    public long delete(UUID channelId, boolean force) {
        if (!clusterManager.canServeWrites()) {
            throw new QuorumViolationException("minority partition");
        }
        NodeInfo owner = clusterManager.owner(channelId);
        if (clusterManager.isLocal(owner)) {
            return delegate.delete(channelId, force);
        }
        return proxyClient.deleteChannel(owner, channelId, force);
    }

    @Override
    public Channel pause(UUID channelId) {
        NodeInfo owner = clusterManager.owner(channelId);
        if (clusterManager.isLocal(owner)) {
            return delegate.pause(channelId);
        }
        return proxyClient.pauseChannel(owner, channelId);
    }

    @Override
    public Channel resume(UUID channelId) {
        NodeInfo owner = clusterManager.owner(channelId);
        if (clusterManager.isLocal(owner)) {
            return delegate.resume(channelId);
        }
        return proxyClient.resumeChannel(owner, channelId);
    }

    // All config mutations route by channelId (D15: route ALL channel mutations)
    private Channel routeChannelMutation(UUID channelId, java.util.function.Function<UUID, Channel> localAction) {
        NodeInfo owner = clusterManager.owner(channelId);
        if (clusterManager.isLocal(owner)) {
            return localAction.apply(channelId);
        }
        throw new UnsupportedOperationException("Remote config mutation proxy not yet wired");
    }

    @Override public Channel setTypeConstraints(UUID id, Set<MessageType> a, Set<MessageType> d) { return routeChannelMutation(id, i -> delegate.setTypeConstraints(i, a, d)); }
    @Override public Channel setRateLimits(UUID id, Integer pc, Integer pi) { return routeChannelMutation(id, i -> delegate.setRateLimits(i, pc, pi)); }
    @Override public Channel setAllowedWriters(UUID id, List<String> w) { return routeChannelMutation(id, i -> delegate.setAllowedWriters(i, w)); }
    @Override public Channel setAdminInstances(UUID id, List<String> a) { return routeChannelMutation(id, i -> delegate.setAdminInstances(i, a)); }
    @Override public Channel setReviewerInstances(UUID id, List<String> r) { return routeChannelMutation(id, i -> delegate.setReviewerInstances(i, r)); }
    @Override public Channel setProtocols(UUID id, List<String> p) { return routeChannelMutation(id, i -> delegate.setProtocols(i, p)); }
    @Override public Channel setProtocolParticipants(UUID id, List<String> p) { return routeChannelMutation(id, i -> delegate.setProtocolParticipants(i, p)); }
    @Override public Channel setEnforcementMode(UUID id, EnforcementMode m) { return routeChannelMutation(id, i -> delegate.setEnforcementMode(i, m)); }
    @Override public Channel setEnforcementExclusions(UUID id, List<String> e) { return routeChannelMutation(id, i -> delegate.setEnforcementExclusions(i, e)); }
    @Override public Channel setRoutingTrustThreshold(UUID id, Double t) { return routeChannelMutation(id, i -> delegate.setRoutingTrustThreshold(i, t)); }
    @Override public Channel setRedistributionCapacityThreshold(UUID id, Double t) { return routeChannelMutation(id, i -> delegate.setRedistributionCapacityThreshold(i, t)); }
    @Override public Channel setRoutingCapacityThreshold(UUID id, Double t) { return routeChannelMutation(id, i -> delegate.setRoutingCapacityThreshold(i, t)); }
    @Override public Channel setPolicyOverrides(UUID id, Map<String, String> o) { return routeChannelMutation(id, i -> delegate.setPolicyOverrides(i, o)); }
    @Override public void setTrackDelivery(UUID id, Boolean t) { routeChannelMutation(id, i -> { delegate.setTrackDelivery(i, t); return null; }); }
    @Override public void updateLastActivity(UUID id, String t) { delegate.updateLastActivity(id, t); }
}
```

- [ ] **Step 4: Write ChannelManagerDecorator tests**

Create `cluster/src/test/java/io/casehub/qhorus/cluster/ChannelManagerDecoratorTest.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.channel.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;

class ChannelManagerDecoratorTest {

    private ChannelManager delegate;
    private ClusterManager clusterManager;
    private WriteProxyClient proxyClient;
    private ChannelManagerDecorator decorator;

    @BeforeEach
    void setUp() {
        delegate = mock(ChannelManager.class);
        clusterManager = mock(ClusterManager.class);
        proxyClient = mock(WriteProxyClient.class);
        decorator = new ChannelManagerDecorator(delegate, clusterManager, proxyClient);
    }

    @Test
    void createRoutesToOwner() {
        var localNode = new NodeInfo("node-1", "node-1:8080");
        when(clusterManager.canServeWrites()).thenReturn(true);
        when(clusterManager.owner(any(UUID.class))).thenReturn(localNode);
        when(clusterManager.isLocal(localNode)).thenReturn(true);
        var req = ChannelCreateRequest.builder("test-ch").build();
        var channel = mock(Channel.class);
        when(delegate.create(any())).thenReturn(channel);

        Channel result = decorator.create(req);

        assertThat(result).isEqualTo(channel);
        verify(delegate).create(argThat(r -> r.preAssignedId() != null));
    }

    @Test
    void deleteRoutesToOwnerByChannelId() {
        UUID channelId = UUID.randomUUID();
        var remoteNode = new NodeInfo("node-2", "node-2:8080");
        when(clusterManager.canServeWrites()).thenReturn(true);
        when(clusterManager.owner(channelId)).thenReturn(remoteNode);
        when(clusterManager.isLocal(remoteNode)).thenReturn(false);
        when(proxyClient.deleteChannel(remoteNode, channelId, true)).thenReturn(5L);

        long result = decorator.delete(channelId, true);

        assertThat(result).isEqualTo(5L);
        verify(proxyClient).deleteChannel(remoteNode, channelId, true);
    }

    @Test
    void throwsOnQuorumViolation() {
        when(clusterManager.canServeWrites()).thenReturn(false);
        var req = ChannelCreateRequest.builder("test-ch").build();
        assertThatThrownBy(() -> decorator.create(req))
                .isInstanceOf(QuorumViolationException.class);
    }
}
```

- [ ] **Step 5: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add cluster/ api/
git commit -m "feat(#475): add WriteRoutingDecorator and ChannelManagerDecorator

Refs #475"
```

---

## Batch 4: Internal HTTP + Health Endpoints

After this batch: internal RPC endpoints serve dispatch/heartbeat/leave
requests. Health and topology endpoints provide cluster observability.
Full module is feature-complete.

### Task 6: InternalMeshResource + ClusterHealthResource

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalDispatchRequest.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalChannelRequest.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshClient.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/LeaveRequest.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResource.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResponse.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/TopologyResponse.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/InternalDispatchRequestTest.java`

**Interfaces:**
- Consumes: `MessageDispatch`, `DispatchResult`, `ClusterManager`, `HeartbeatService.buildLocalResponse()` (from Tasks 2-3), `MessageDispatcher` (from qhorus API), `ChannelManager` (from qhorus API)
- Produces:
  - `InternalDispatchRequest` — record with `from(MessageDispatch)` and `toMessageDispatch()` methods
  - `InternalMeshResource` — JAX-RS `@Path("/internal")` with POST dispatch, GET heartbeat, POST leave
  - `ClusterHealthResource` — JAX-RS `@Path("/health/cluster")` and `@Path("/admin/topology")`
  - `LeaveRequest(String nodeId)` — record
  - `ClusterHealthResponse`, `TopologyResponse` — response records

- [ ] **Step 1: Write InternalDispatchRequest round-trip test**

Create `cluster/src/test/java/io/casehub/qhorus/cluster/InternalDispatchRequestTest.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.*;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class InternalDispatchRequestTest {

    @Test
    void roundTripPreservesAllFields() {
        UUID channelId = UUID.randomUUID();
        var original = MessageDispatch.builder()
                .channelId(channelId)
                .sender("agent-1")
                .type(MessageType.COMMAND)
                .content("do something")
                .correlationId("corr-1")
                .target("role:worker")
                .actorType(ActorType.AGENT)
                .topic("general")
                .build();

        var request = InternalDispatchRequest.from(original);
        var restored = request.toMessageDispatch();

        assertThat(restored.channelId()).isEqualTo(channelId);
        assertThat(restored.sender()).isEqualTo("agent-1");
        assertThat(restored.type()).isEqualTo(MessageType.COMMAND);
        assertThat(restored.content()).isEqualTo("do something");
        assertThat(restored.correlationId()).isEqualTo("corr-1");
        assertThat(restored.target()).isEqualTo("role:worker");
        assertThat(restored.topic()).isEqualTo("general");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml -Dtest=InternalDispatchRequestTest`
Expected: Compilation error

- [ ] **Step 3: Create InternalDispatchRequest**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/InternalDispatchRequest.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.*;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record InternalDispatchRequest(
        UUID channelId, String sender, String type, String content,
        String payload, String correlationId, Long inReplyTo,
        List<ArtefactRef> artefactRefs, String target,
        String subjectId, String causedByEntryId, String actorType,
        String deadline, String telemetry, String tenancyId, String topic,
        String correctsMessageId, boolean retraction) {

    public static InternalDispatchRequest from(MessageDispatch d) {
        return new InternalDispatchRequest(
                d.channelId(), d.sender(),
                d.type() != null ? d.type().name() : null,
                d.content(), d.payload(), d.correlationId(), d.inReplyTo(),
                d.artefactRefs(), d.target(),
                d.subjectId() != null ? d.subjectId().toString() : null,
                d.causedByEntryId() != null ? d.causedByEntryId().toString() : null,
                d.actorType() != null ? d.actorType().name() : null,
                d.deadline() != null ? d.deadline().toString() : null,
                d.telemetry(), d.tenancyId(), d.topic(),
                d.correctsMessageId() != null ? d.correctsMessageId().toString() : null,
                d.retraction());
    }

    public MessageDispatch toMessageDispatch() {
        return new MessageDispatch(
                channelId, sender,
                type != null ? MessageType.valueOf(type) : null,
                content, payload, correlationId, inReplyTo,
                artefactRefs, target,
                subjectId != null ? UUID.fromString(subjectId) : null,
                causedByEntryId != null ? UUID.fromString(causedByEntryId) : null,
                actorType != null ? ActorType.valueOf(actorType) : null,
                deadline != null ? Instant.parse(deadline) : null,
                telemetry, tenancyId, topic,
                correctsMessageId != null ? Long.parseLong(correctsMessageId) : null,
                retraction);
    }
}
```

- [ ] **Step 4: Create remaining types and resources**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/LeaveRequest.java`:

```java
package io.casehub.qhorus.cluster;

public record LeaveRequest(String nodeId) {}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/InternalChannelRequest.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;

import java.util.*;

public record InternalChannelRequest(
        String name, String description, String semantic,
        List<String> barrierContributors, List<String> allowedWriters,
        List<String> adminInstances, Integer rateLimitPerChannel,
        Integer rateLimitPerInstance, Set<String> allowedTypes,
        Set<String> deniedTypes, UUID spaceId, UUID preAssignedId) {

    public static InternalChannelRequest from(ChannelCreateRequest req) {
        return new InternalChannelRequest(
                req.name(), req.description(),
                req.semantic() != null ? req.semantic().name() : null,
                req.barrierContributors(), req.allowedWriters(),
                req.adminInstances(), req.rateLimitPerChannel(),
                req.rateLimitPerInstance(),
                req.allowedTypes() != null ? MessageType.serializeTypes(req.allowedTypes()).transform(s -> Set.of(s.split(","))) : null,
                req.deniedTypes() != null ? MessageType.serializeTypes(req.deniedTypes()).transform(s -> Set.of(s.split(","))) : null,
                req.spaceId(), req.preAssignedId());
    }

    public ChannelCreateRequest toChannelCreateRequest() {
        return ChannelCreateRequest.builder(name)
                .description(description)
                .semantic(semantic != null ? ChannelSemantic.valueOf(semantic) : null)
                .barrierContributors(barrierContributors)
                .allowedWriters(allowedWriters)
                .adminInstances(adminInstances)
                .rateLimitPerChannel(rateLimitPerChannel)
                .rateLimitPerInstance(rateLimitPerInstance)
                .spaceId(spaceId)
                .preAssignedId(preAssignedId)
                .build();
    }
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshClient.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.message.DispatchResult;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;

import java.util.UUID;

@Path("/internal")
@RegisterRestClient(configKey = "internal-mesh")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface InternalMeshClient {

    @POST @Path("/dispatch")
    DispatchResult dispatch(InternalDispatchRequest request);

    @POST @Path("/channel")
    Channel createChannel(InternalChannelRequest request);

    @POST @Path("/channel/{id}/delete")
    long deleteChannel(@PathParam("id") UUID channelId, @QueryParam("force") boolean force);

    @POST @Path("/channel/{id}/pause")
    Channel pauseChannel(@PathParam("id") UUID channelId);

    @POST @Path("/channel/{id}/resume")
    Channel resumeChannel(@PathParam("id") UUID channelId);

    @GET @Path("/heartbeat")
    HeartbeatResponse heartbeat();

    @POST @Path("/leave")
    void leave(LeaveRequest request);
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResponse.java`:

```java
package io.casehub.qhorus.cluster;

import java.util.Map;

public record ClusterHealthResponse(
        String nodeId, String status, int clusterSize, int expectedSize,
        String ringHash, Map<String, PeerState> peers) {}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/TopologyResponse.java`:

```java
package io.casehub.qhorus.cluster;

import java.util.List;
import java.util.Map;

public record TopologyResponse(List<NodeSummary> nodes) {
    public record NodeSummary(String nodeId, String address, NodeState status,
                               java.time.Instant lastHeartbeat) {}
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatcher;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;

@Path("/internal")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class InternalMeshResource {

    private final MessageDispatcher messageDispatcher;
    private final HeartbeatService heartbeatService;
    private final ClusterManager clusterManager;

    public InternalMeshResource(MessageDispatcher messageDispatcher,
                                 HeartbeatService heartbeatService,
                                 ClusterManager clusterManager) {
        this.messageDispatcher = messageDispatcher;
        this.heartbeatService = heartbeatService;
        this.clusterManager = clusterManager;
    }

    @POST @Path("/dispatch")
    public DispatchResult dispatch(InternalDispatchRequest request) {
        return messageDispatcher.dispatch(request.toMessageDispatch());
    }

    @GET @Path("/heartbeat")
    public HeartbeatResponse heartbeat() {
        return heartbeatService.buildLocalResponse();
    }

    @POST @Path("/leave")
    public void leave(LeaveRequest request) {
        clusterManager.handlePeerDeparture(request.nodeId());
    }
}
```

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResource.java`:

```java
package io.casehub.qhorus.cluster;

import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import java.util.*;
import java.util.stream.Collectors;

@Path("/")
@Produces(MediaType.APPLICATION_JSON)
public class ClusterHealthResource {

    private final ClusterManager clusterManager;

    public ClusterHealthResource(ClusterManager clusterManager) {
        this.clusterManager = clusterManager;
    }

    @GET @Path("/health/cluster")
    public ClusterHealthResponse health() {
        return new ClusterHealthResponse(
                clusterManager.nodeId(),
                clusterManager.canServeWrites() ? "UP" : "DEGRADED",
                clusterManager.clusterSize(),
                clusterManager.expectedSize(),
                clusterManager.ringHash(),
                clusterManager.peerStates());
    }

    @GET @Path("/admin/topology")
    public TopologyResponse topology() {
        List<TopologyResponse.NodeSummary> nodes = new ArrayList<>();
        nodes.add(new TopologyResponse.NodeSummary(
                clusterManager.nodeId(), "self", NodeState.ALIVE, null));
        for (var entry : clusterManager.peerStates().entrySet()) {
            PeerState ps = entry.getValue();
            nodes.add(new TopologyResponse.NodeSummary(
                    entry.getKey(), ps.nodeInfo().address(),
                    ps.state(), ps.lastHeartbeat()));
        }
        return new TopologyResponse(nodes);
    }
}
```

- [ ] **Step 5: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f cluster/pom.xml`
Expected: All tests PASS

- [ ] **Step 6: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 7: Commit**

```bash
git add cluster/
git commit -m "feat(#475): add InternalMeshResource and ClusterHealthResource

Refs #475"
```

---

## Batch 5: ClusterConfig + CDI Wiring

After this batch: the cluster module is fully wired with Quarkus CDI.
`@IfBuildProperty` gates all beans. Config mapping reads env vars.
The mesh module can add this as a dependency and clustering activates
by setting `casehub.qhorus.cluster.enabled=true`.

### Task 7: ClusterConfig + CDI bean producers

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterConfig.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterProducer.java`
- Modify: `cluster/pom.xml` — add `quarkus-arc`, `quarkus-rest-jackson`, `quarkus-scheduler` dependencies
- Create: `cluster/src/main/resources/application.properties` — default config

**Interfaces:**
- Consumes: all types from Tasks 1-6
- Produces:
  - `RelayConfig` — `@ConfigMapping`
  - `RelayProducer` — CDI producer for `ClusterManager`, `HeartbeatService`, `WriteRoutingDecorator`, `ChannelManagerDecorator`, `WriteProxyClient`

- [ ] **Step 1: Create ClusterConfig**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterConfig.java`:

```java
package io.casehub.qhorus.cluster;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

import java.time.Duration;
import java.util.List;
import java.util.Optional;

@ConfigMapping(prefix = "casehub.qhorus.cluster")
public interface ClusterConfig {

    @WithDefault("false")
    boolean enabled();

    Optional<List<String>> peers();

    Optional<String> nodeId();

    @WithDefault("128")
    int virtualNodes();

    @WithDefault("3s")
    Duration heartbeatInterval();

    @WithDefault("2")
    int heartbeatMissThreshold();

    @WithDefault("30s")
    Duration drainTimeout();

    @WithDefault("true")
    boolean quorumEnforced();

    @WithDefault("10s")
    Duration proxyTimeout();
}
```

- [ ] **Step 2: Create ClusterProducer**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterProducer.java`:

```java
package io.casehub.qhorus.cluster;

import io.quarkus.arc.properties.IfBuildProperty;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;
import java.net.InetAddress;
import java.time.Clock;
import java.util.*;

@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.cluster.enabled", stringValue = "true", enableIfMissing = false)
public class ClusterProducer {

    @Inject ClusterConfig config;

    @Produces @ApplicationScoped
    public ClusterManager clusterManager() {
        String nodeId = config.nodeId().orElseGet(() -> {
            try { return InetAddress.getLocalHost().getHostName(); }
            catch (Exception e) { return "unknown"; }
        });

        List<String> peerList = config.peers().orElse(List.of());
        if (peerList.isEmpty()) {
            peerList = List.of(nodeId + ":8080");
        }

        Map<String, String> peerMap = new LinkedHashMap<>();
        for (String peer : peerList) {
            String[] parts = peer.split(":", 2);
            if (parts.length != 2) {
                throw new IllegalStateException("Malformed peer address: " + peer + " — expected host:port");
            }
            String id = parts[0];
            if (peerMap.containsKey(id)) {
                throw new IllegalStateException("Duplicate node ID in peer list: " + id);
            }
            peerMap.put(id, peer);
        }

        return new ClusterManager(nodeId, peerMap, config.virtualNodes(),
                config.heartbeatMissThreshold(), config.quorumEnforced(),
                Clock.systemUTC());
    }

    @Produces @ApplicationScoped
    public WriteProxyClient writeProxyClient() {
        return new WriteProxyClient();
    }
}
```

- [ ] **Step 3: Update pom.xml with Quarkus dependencies**

Add to `cluster/pom.xml` dependencies:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-scheduler</artifactId>
    </dependency>
    <dependency>
      <groupId>io.smallrye</groupId>
      <artifactId>smallrye-config-core</artifactId>
    </dependency>
```

- [ ] **Step 4: Create default application.properties**

Create `cluster/src/main/resources/application.properties`:

```properties
# Cluster defaults — override via environment variables
casehub.qhorus.cluster.enabled=false
```

- [ ] **Step 5: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add cluster/ pom.xml
git commit -m "feat(#475): add ClusterConfig and CDI producer with @IfBuildProperty gating

Refs #475"
```

---

## References

- [2026-10-06-cluster-module-design.md] — Phase 2 design spec
- [2026-10-06-distributed-mesh-design.md] — overall distributed mesh architecture
- [decisions.md] — D12-D17 Phase 2 implementation decisions
- [api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java] — decorator target interface
- [api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java] — decorator target interface
- [api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java] — preAssignedId modification target
- [api/src/main/java/io/casehub/qhorus/api/channel/Channel.java:133] — Channel.fromRequest() UUID generation
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateHelper.java] — channel creation flow
- [postgres-broadcaster/src/main/java/.../PostgresChannelActivityBroadcaster.java] — cross-node delivery reference
- [GitHub #475] — epic: distributed qhorus mesh
