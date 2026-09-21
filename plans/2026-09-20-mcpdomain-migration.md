# @McpDomain Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #451 — feat: migrate QhorusMcpTools (113 @Tool ops) to @McpDomain for tri-channel parity
**Issue group:** #451

**Goal:** Migrate all 113 `@Tool` methods from `QhorusMcpTools.java` (2756 lines) to `@McpDomain`-annotated API interfaces across 6 domains, enabling lazy hierarchical MCP discovery via `casehub_activate` and automatic tri-channel (MCP + GraphQL + REST) parity via the platform APT generator.

**Architecture:** Each domain follows the established pattern: a `*Api` interface in `api/spi/<domain>/` annotated with `@McpDomain("<name>")` and `@PlatformQuery`/`@PlatformMutation` on methods, a `*Service` implementation in `graphql/<domain>/` that injects existing API-layer facades (readers, managers, stores), and CDI-free unit tests with Mockito. The platform APT generator (`casehub-platform-graphql-generator`) processes these at compile time to produce `Generated*Resolver.java` (GraphQL) and `Generated*Resource.java` (REST) — no manual resolver or JAX-RS code is needed. As each domain's `*Api` is complete, the corresponding `@Tool` methods are deleted from `QhorusMcpTools.java`. When all domains are done, `QhorusMcpTools.java`, `QhorusMcpToolsBase.java`, and the `@McpServer("qhorus")` annotation are retired.

**Tech Stack:** Java 21, Quarkus 3.32.2, MicroProfile GraphQL, casehub-platform-graphql-generator (APT), JUnit 5, Mockito, AssertJ

## Global Constraints

- Follow the established pattern from `ChannelsApi`/`ChannelsService`, `MessagingApi`/`MessagingService`, `GovernanceApi`/`GovernanceService`
- API interfaces use `@McpDomain("domain")` + `@PlatformQuery`/`@PlatformMutation` — NOT `@Query`/`@Mutation` from MicroProfile GraphQL
- Service implementations are `@ApplicationScoped` CDI beans — inject only API-layer interfaces (not runtime classes)
- Tests are CDI-free with Mockito mocks — no `@QuarkusTest`
- Channel resolution at the API boundary: all channel parameters are `UUID channelId` (not `String channel`) — the MCP-to-API adapter resolves names upstream
- All types referenced in API method signatures must be in `api/` module or platform — never reference `runtime/` classes
- `DomainRegistrationTest` must be updated after each new domain is added
- `mvn install` from root after each batch to verify cross-module compilation

---

## Batch 1: Agents domain (new — 8 ops)

### Task 1: Create AgentsApi interface + AgentsService + tests, remove @Tool methods

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/agents/AgentsApi.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/instance/RegisterResponse.java`
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/agents/AgentsService.java`
- Create: `graphql/src/test/java/io/casehub/qhorus/graphql/agents/AgentsServiceTest.java`
- Modify: `graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` (remove 8 @Tool methods)
- Test: `graphql/src/test/java/io/casehub/qhorus/graphql/agents/AgentsServiceTest.java`

**Interfaces:**
- Consumes: `InstanceStore` (api/store), `PresenceTracker` (api/channel), `ChannelManager` (api/channel), `ChannelReader` (api/channel)
- Produces: `AgentsApi` interface, `RegisterResponse` record, `AgentsService` implementation

**@Tool methods to remove:** `register`, `list_instances`, `get_instance`, `deregister_instance`, `set_presence`, `get_presence`, `get_channel_presence`
Note: `register_instance` (lines 222-245) is a non-`@Tool` overload used internally — it stays until QhorusMcpTools is deleted.

- [ ] **Step 1: Create RegisterResponse record in api/instance/**

```java
package io.casehub.qhorus.api.instance;

import java.util.List;

public record RegisterResponse(
        String instanceId,
        List<String> activeChannelNames,
        List<InstanceInfo> onlineInstances) {}
```

- [ ] **Step 2: Create AgentsApi interface**

```java
package io.casehub.qhorus.api.spi.agents;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.qhorus.api.channel.Presence;
import io.casehub.qhorus.api.instance.InstanceInfo;
import io.casehub.qhorus.api.instance.RegisterResponse;

import java.util.List;
import java.util.UUID;

@McpDomain("agents")
public interface AgentsApi {

    @PlatformQuery("List registered agent instances, optionally filtered by capability")
    List<InstanceInfo> instances(String capability);

    @PlatformQuery("Look up a registered instance by its ID")
    InstanceInfo instance(String instanceId);

    @PlatformQuery("Get presence status for a member")
    Presence presence(String memberId);

    @PlatformQuery("Get presence status for all members of a channel")
    List<Presence> channelPresence(UUID channelId);

    @PlatformMutation("Register an agent instance with capability tags")
    RegisterResponse register(String instanceId, String description,
                              List<String> capabilities, Boolean readOnly);

    @PlatformMutation("Force-remove an agent instance from the registry")
    boolean deregister(String instanceId);

    @PlatformMutation("Report presence status heartbeat (ONLINE, AVAILABLE, BUSY)")
    Presence setPresence(String status, String statusMessage, String memberId);
}
```

- [ ] **Step 3: Write failing test for AgentsService**

```java
package io.casehub.qhorus.graphql.agents;

import io.casehub.qhorus.api.channel.Presence;
import io.casehub.qhorus.api.channel.PresenceStatus;
import io.casehub.qhorus.api.channel.PresenceTracker;
import io.casehub.qhorus.api.instance.Instance;
import io.casehub.qhorus.api.instance.InstanceInfo;
import io.casehub.qhorus.api.store.InstanceStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

class AgentsServiceTest {

    private AgentsService service;
    private InstanceStore instanceStore;
    private PresenceTracker presenceTracker;

    @BeforeEach
    void setUp() {
        instanceStore = mock(InstanceStore.class);
        presenceTracker = mock(PresenceTracker.class);
        service = new AgentsService(instanceStore, presenceTracker);
    }

    @Test
    void instancesReturnsAllWhenNoCapabilityFilter() {
        var inst = new InstanceInfo("agent-1", "desc", "ONLINE",
                List.of("cap1"), Instant.now(), false);
        when(instanceStore.listOnline()).thenReturn(List.of(inst));

        var result = service.instances(null);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).instanceId()).isEqualTo("agent-1");
    }

    @Test
    void instanceByIdReturnsInfo() {
        var inst = new InstanceInfo("agent-1", "desc", "ONLINE",
                List.of("cap1"), Instant.now(), false);
        when(instanceStore.findInfoByInstanceId("agent-1")).thenReturn(Optional.of(inst));

        var result = service.instance("agent-1");

        assertThat(result).isNotNull();
        assertThat(result.instanceId()).isEqualTo("agent-1");
    }

    @Test
    void instanceByIdThrowsWhenNotFound() {
        when(instanceStore.findInfoByInstanceId("missing")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.instance("missing"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void presenceReturnsMemberStatus() {
        var p = new Presence("member-1", PresenceStatus.ONLINE, null, Instant.now());
        when(presenceTracker.getPresence("member-1")).thenReturn(p);

        var result = service.presence("member-1");

        assertThat(result.memberId()).isEqualTo("member-1");
        assertThat(result.status()).isEqualTo(PresenceStatus.ONLINE);
    }
}
```

- [ ] **Step 4: Run tests — verify they fail (AgentsService doesn't exist yet)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=AgentsServiceTest`
Expected: FAIL — compilation error, `AgentsService` class not found

- [ ] **Step 5: Implement AgentsService**

```java
package io.casehub.qhorus.graphql.agents;

import io.casehub.qhorus.api.channel.Presence;
import io.casehub.qhorus.api.channel.PresenceStatus;
import io.casehub.qhorus.api.channel.PresenceTracker;
import io.casehub.qhorus.api.instance.InstanceInfo;
import io.casehub.qhorus.api.instance.RegisterResponse;
import io.casehub.qhorus.api.spi.agents.AgentsApi;
import io.casehub.qhorus.api.store.InstanceStore;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class AgentsService implements AgentsApi {

    private final InstanceStore instanceStore;
    private final PresenceTracker presenceTracker;

    public AgentsService(InstanceStore instanceStore,
                         PresenceTracker presenceTracker) {
        this.instanceStore = instanceStore;
        this.presenceTracker = presenceTracker;
    }

    @Override
    public List<InstanceInfo> instances(String capability) {
        var all = instanceStore.listOnline();
        if (capability == null || capability.isBlank()) {
            return all;
        }
        return all.stream()
                .filter(i -> i.capabilities() != null
                        && i.capabilities().contains(capability))
                .toList();
    }

    @Override
    public InstanceInfo instance(String instanceId) {
        return instanceStore.findInfoByInstanceId(instanceId)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Instance not found: " + instanceId));
    }

    @Override
    public Presence presence(String memberId) {
        return presenceTracker.getPresence(memberId);
    }

    @Override
    public List<Presence> channelPresence(UUID channelId) {
        return presenceTracker.getChannelPresence(channelId);
    }

    @Override
    public RegisterResponse register(String instanceId, String description,
                                     List<String> capabilities, Boolean readOnly) {
        instanceStore.register(instanceId, description,
                capabilities != null ? capabilities : List.of(),
                readOnly != null && readOnly);
        var online = instanceStore.listOnline();
        return new RegisterResponse(instanceId, List.of(), online);
    }

    @Override
    public boolean deregister(String instanceId) {
        instanceStore.markOffline(instanceId);
        return true;
    }

    @Override
    public Presence setPresence(String status, String statusMessage, String memberId) {
        presenceTracker.heartbeat(PresenceStatus.valueOf(status), statusMessage);
        return presenceTracker.getPresence(
                memberId != null ? memberId : "self");
    }
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=AgentsServiceTest`
Expected: PASS

- [ ] **Step 7: Update DomainRegistrationTest**

Add to `DomainRegistrationTest.java`:

```java
import io.casehub.qhorus.api.spi.agents.AgentsApi;

@Test
void agentsSpiAnnotationsPresent() {
    assertDomain(AgentsApi.class, "agents");
}
```

Update `noQhorusDomainRemains()` to include `AgentsApi.class` in the array.

- [ ] **Step 8: Remove @Tool methods from QhorusMcpTools**

Remove these `@Tool`-annotated methods and their non-`@Tool` overloads from `QhorusMcpTools.java`:
- `register` (lines 185-206)
- `listInstances` (lines 249-255)
- `getInstance` (lines 261-267)
- `deregisterInstance` (lines 1754-1764)
- `setPresence` (lines 2375-2383)
- `getPresenceTool` (lines 2386-2389)
- `getChannelPresence` (lines 2392-2396)

Keep `registerInstance` overloads (lines 222-245) — they are non-`@Tool` internal methods still referenced.

- [ ] **Step 9: Verify build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS — no compilation errors from removed methods

- [ ] **Step 10: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql`
Expected: All tests pass including DomainRegistrationTest

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/main/java/io/casehub/qhorus/api/spi/agents/ api/src/main/java/io/casehub/qhorus/api/instance/RegisterResponse.java graphql/src/main/java/io/casehub/qhorus/graphql/agents/ graphql/src/test/java/io/casehub/qhorus/graphql/agents/ graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#451): migrate agents domain to @McpDomain — 7 @Tool ops → AgentsApi

Refs #451"
```

---

## Batch 2: Data domain (new — 11 ops)

### Task 2: Create DataApi interface + DataService + tests, remove @Tool methods

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/data/DataApi.java`
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/data/DataService.java`
- Create: `graphql/src/test/java/io/casehub/qhorus/graphql/data/DataServiceTest.java`
- Modify: `graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` (remove 11 @Tool methods)
- Test: `graphql/src/test/java/io/casehub/qhorus/graphql/data/DataServiceTest.java`

**Interfaces:**
- Consumes: `DataStore` (api/store), `MessageReader` (api/store)
- Produces: `DataApi` interface, `DataService` implementation

**@Tool methods to remove:** `share_artefact`, `begin_artefact`, `append_chunk`, `finalize_artefact`, `get_artefact`, `get_artefact_refs`, `list_artefacts`, `claim_artefact`, `release_artefact`, `is_gc_eligible`, `revoke_artefact`

- [ ] **Step 1: Create DataApi interface**

The API uses the existing `SharedData` record from `api/store/DataStore` for return types. For artefact references, use `ArtefactRef` from `api/message/`.

```java
package io.casehub.qhorus.api.spi.data;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.qhorus.api.message.ArtefactRef;

import java.util.List;
import java.util.UUID;

@McpDomain("data")
public interface DataApi {

    @PlatformQuery("Retrieve a shared artefact by key or UUID")
    SharedData artefact(String key, UUID id);

    @PlatformQuery("Get artefact references attached to a message")
    List<ArtefactRef> artefactRefs(Long messageId);

    @PlatformQuery("List all artefacts with metadata")
    List<SharedData> artefacts();

    @PlatformQuery("Check if an artefact is eligible for garbage collection")
    boolean isGcEligible(UUID artefactId);

    @PlatformMutation("Store a shared artefact by key")
    SharedData shareArtefact(String key, String description, String createdBy, String content);

    @PlatformMutation("Begin a chunked artefact upload")
    SharedData beginArtefact(String key, String description, String createdBy, String content);

    @PlatformMutation("Append a chunk to an in-progress artefact upload")
    SharedData appendChunk(String key, String content);

    @PlatformMutation("Finalize a chunked artefact upload")
    SharedData finalizeArtefact(String key, String content);

    @PlatformMutation("Manually claim an artefact reference")
    void claimArtefact(UUID artefactId, String instanceId);

    @PlatformMutation("Release an artefact claim")
    void releaseArtefact(UUID artefactId, String instanceId);

    @PlatformMutation("Force-delete a shared artefact and release all claims")
    boolean revokeArtefact(UUID artefactId);
}
```

Note: `SharedData` here refers to the existing type from the store layer. Check whether `io.casehub.qhorus.api.store.DataStore` returns a suitable record or whether the runtime `SharedData` entity needs an API-layer DTO. If the entity is runtime-only, create a `io.casehub.qhorus.api.data.Artefact` record as the API return type with a `from(SharedData)` factory.

- [ ] **Step 2: Write failing tests for DataService**

Test `artefacts()`, `artefact(key, id)`, `isGcEligible()`, `shareArtefact()`. Follow the same Mockito pattern as `AgentsServiceTest`.

- [ ] **Step 3: Implement DataService**

Inject `DataStore` and `MessageReader` (for artefactRefs lookup). Each method delegates to the injected store. Follow the established pattern from `AgentsService`.

- [ ] **Step 4: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=DataServiceTest`

- [ ] **Step 5: Update DomainRegistrationTest — add DataApi assertion**

- [ ] **Step 6: Remove 11 @Tool methods from QhorusMcpTools**

Remove: `shareArtefact` (1561), `beginArtefact` (1578), `appendChunk` (1590), `finalizeArtefact` (1600), `getArtefact` (1609), `getArtefactRefs` (1626), `listArtefacts` (1635), `claimArtefact` (1642), `releaseArtefact` (1656), `isGcEligible` (1668), `revokeArtefact` (1676).

- [ ] **Step 7: Verify build + tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#451): migrate data domain to @McpDomain — 11 @Tool ops → DataApi

Refs #451"
```

---

## Batch 3: Governance expansion (add 8 ops to existing)

### Task 3: Expand GovernanceApi with commitment detail + watchdogs + capacity, remove @Tool methods

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/spi/governance/GovernanceApi.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/WatchdogSummary.java` (promote from QhorusMcpToolsBase)
- Modify: `graphql/src/main/java/io/casehub/qhorus/graphql/governance/GovernanceService.java`
- Modify: `graphql/src/test/java/io/casehub/qhorus/graphql/governance/GovernanceServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Interfaces:**
- Consumes: `CommitmentReader` (api/store), `WatchdogStore` (api/store), `ActorCapacityView` (Instance<>)
- Produces: expanded `GovernanceApi`, `WatchdogSummary` API record

**@Tool methods to remove:** `list_pending_commitments`, `list_my_commitments`, `get_commitment`, `register_watchdog`, `list_watchdogs`, `delete_watchdog`, `getActorCapacity`, `listOverloadedActors`, `getRedistributionHistory`

- [ ] **Step 1: Create WatchdogSummary record in api layer**

Promote `QhorusMcpToolsBase.WatchdogSummary` to `io.casehub.qhorus.api.watchdog.WatchdogSummary` (or `api/governance/`). All fields same as the existing record.

- [ ] **Step 2: Expand GovernanceApi with new operations**

Add to existing `GovernanceApi.java`:

```java
@PlatformQuery("List non-terminal commitments across all channels")
List<Commitment> pendingCommitments();

@PlatformQuery("List commitments involving a specific agent on a channel")
List<Commitment> myCommitments(UUID channelId, String sender, String role);

@PlatformQuery("Get commitment by correlationId")
Commitment commitment(String correlationId);

@PlatformQuery("List all registered watchdog conditions")
List<WatchdogSummary> watchdogs();

@PlatformMutation("Register a watchdog condition")
WatchdogSummary registerWatchdog(String conditionType, String targetName,
        Integer thresholdSeconds, Integer thresholdCount,
        Integer similarityPct, String notificationChannel,
        String createdBy, String action);

@PlatformMutation("Delete a watchdog by ID")
boolean deleteWatchdog(UUID watchdogId);
```

Note: Capacity operations (`getActorCapacity`, `listOverloadedActors`, `getRedistributionHistory`) go here too. They require `Instance<ActorCapacityView>` injection with `isResolvable()` guard.

- [ ] **Step 3: Write failing tests**

Test `pendingCommitments()`, `commitment()`, `watchdogs()`, `registerWatchdog()`. Mock `CommitmentReader`, `WatchdogStore`.

- [ ] **Step 4: Expand GovernanceService**

Add implementations for all new GovernanceApi methods. Inject `CommitmentReader`, `WatchdogStore`, `Instance<ActorCapacityView>`.

- [ ] **Step 5: Run tests, verify pass**

- [ ] **Step 6: Remove 9 @Tool methods from QhorusMcpTools**

- [ ] **Step 7: Verify build + commit**

---

## Batch 4: Audit domain (new — 13 ops)

### Task 4: Create AuditApi interface + AuditService + tests, remove @Tool methods

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/audit/AuditApi.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/audit/` — result records for obligation chain, causal chain, stats, telemetry
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/audit/AuditService.java`
- Create: `graphql/src/test/java/io/casehub/qhorus/graphql/audit/AuditServiceTest.java`
- Modify: `graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository` (runtime/ledger — needs API-layer reader), `CausalGraphService`, `PeerAttestationWriter`, `ReviewerResolver`, `LedgerEntryRepository`
- Produces: `AuditApi` interface, audit result records, `AuditService` implementation

**Critical gap:** The ledger query methods (`listLedgerEntries`, `getObligationChain`, `getCausalChain`, etc.) currently use `MessageLedgerEntryRepository` — a runtime class not in the API layer. The spec identifies this as a Tier 3 gap requiring a new `LedgerReader` interface in `api/`. Create `io.casehub.qhorus.api.store.LedgerReader` with the query methods needed, and have `MessageLedgerEntryRepository` implement it (or create a separate impl).

**@Tool methods to remove:** `list_ledger_entries`, `get_obligation_chain`, `get_causal_chain`, `get_causal_graph`, `render_causal_graph`, `list_stalled_obligations`, `get_obligation_stats`, `get_telemetry_summary`, `get_channel_timeline`, `get_obligation_activity`, `list_attestations`, `attest`, `request_peer_review`

- [ ] **Step 1: Create API-layer audit result records**

Promote from `QhorusMcpToolsBase` to `api/audit/`:
- `ObligationChainSummary`
- `CausalChainEntry`
- `StalledObligation`
- `ObligationStats`
- `TelemetrySummary` + `ToolTelemetry`

- [ ] **Step 2: Create LedgerReader interface in api/store/**

```java
package io.casehub.qhorus.api.store;

public interface LedgerReader {
    // Methods extracted from MessageLedgerEntryRepository query surface
    // that the AuditService needs
}
```

The exact method signatures derive from what `QhorusMcpTools` calls on `ledgerRepo` and `messageRepo`. Read the implementations of `listLedgerEntries`, `getObligationChain`, etc. in QhorusMcpTools.java lines 1874-2249 to determine the query contracts.

- [ ] **Step 3: Create AuditApi interface**

All 13 operations with `@PlatformQuery`/`@PlatformMutation` annotations.

- [ ] **Step 4: Write failing tests**

- [ ] **Step 5: Implement AuditService**

The business logic currently in QhorusMcpTools methods (obligation chain computation, stats aggregation, telemetry summarization) moves into AuditService. This is the most complex domain because it contains non-trivial aggregation logic — not just pass-through delegation.

- [ ] **Step 6: Update DomainRegistrationTest**

- [ ] **Step 7: Remove 13 @Tool methods from QhorusMcpTools**

- [ ] **Step 8: Verify build + commit**

---

## Batch 5: Channels expansion — topics, membership, spaces, gateway (add ~25 ops)

### Task 5: Expand ChannelsApi with topic, membership, space, gateway operations

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/spi/channels/ChannelsApi.java`
- Modify: `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsService.java`
- Modify: `graphql/src/test/java/io/casehub/qhorus/graphql/channels/ChannelsServiceQueryTest.java`
- Modify: `graphql/src/test/java/io/casehub/qhorus/graphql/channels/ChannelsServiceMutationTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Interfaces:**
- Consumes: `TopicManager`, `MembershipManager`, `SpaceService` (needs API facade), `ChannelGateway` (needs API facade for backend ops)
- Produces: expanded `ChannelsApi`, expanded `ChannelsService`

**Operations to add (queries):**
- `listTopics(UUID channelId)` → `TopicManager.listTopics()`
- `listMembers(UUID channelId)` → `MembershipManager` / `ChannelMembershipStore`
- `getUnreadCounts(String memberId)` → `UnreadCountProvider`
- `getMessageDeliveryStatus(UUID channelId, Long messageId)` → membership store
- `listSpaces(UUID parentSpaceId)` → `SpaceService`
- `getSpace(UUID spaceId)` → `SpaceService`
- `listSpaceChannels(UUID spaceId)` → `ChannelReader.scan(bySpaceId)`
- `listBackends(UUID channelId)` → `ChannelGateway`

**Operations to add (mutations):**
- `resolveTopic`, `unresolveTopic`, `renameTopic`, `mergeTopics`, `moveTopic` → `TopicManager`
- `joinChannel`, `leaveChannel`, `markChannelRead` → `MembershipManager`
- `createSpace`, `deleteSpace`, `renameSpace`, `updateSpaceDescription`, `moveSpace`, `moveChannelToSpace` → `SpaceService`
- `registerBackend`, `deregisterBackend` → `ChannelGateway`

**@Tool methods to remove:** ~25 methods — all topic, membership, space, and gateway operations listed above.

- [ ] **Step 1: Add all query method signatures to ChannelsApi**

- [ ] **Step 2: Add all mutation method signatures to ChannelsApi**

- [ ] **Step 3: Write tests for topic operations**

- [ ] **Step 4: Write tests for membership operations**

- [ ] **Step 5: Write tests for space operations**

- [ ] **Step 6: Implement all new methods in ChannelsService**

Inject `TopicManager`, `MembershipManager`, additional stores as needed. Each method is a thin delegation to the API-layer interface.

- [ ] **Step 7: Remove ~25 @Tool methods from QhorusMcpTools**

- [ ] **Step 8: Verify build + commit**

---

## Batch 6: Channels expansion — config, summary, projection, capacity, enforcement, routing (add ~26 ops)

### Task 6: Expand ChannelsApi with channel configuration, summary, projection, capacity, and enforcement operations

**Files:**
- Same as Batch 5 (modify ChannelsApi, ChannelsService, tests, QhorusMcpTools)

**Operations to add (queries):**
- `findChannel(String keyword)`, `listChannels()`, `channelDigest(UUID channelId, Integer limit)`
- `checkMessages(UUID channelId, Long afterId, Integer limit, String sender, Boolean includeEvents)` — note: the semantic-specific behavior (APPEND/COLLECT/BARRIER/EPHEMERAL) in QhorusMcpTools.checkMessages() is business logic that needs to live in a service, not the resolver
- `listProtocols()`, `channelProtocols(UUID channelId)`, `channelEnforcement(UUID channelId)`
- `routingConfig(UUID channelId)`, `routingCandidates(String capability, UUID channelId)`
- `channelRedistributionThreshold(UUID channelId)`, `channelRoutingCapacityThreshold(UUID channelId)`
- `listProjections()`, `projectChannel(UUID channelId, String projectionName, Integer maxMessages, String topic)`
- `channelSummary(UUID channelId)` (4 summary operations)

**Operations to add (mutations):**
- `clearChannel`, `forceReleaseChannel`, `updateChannelBinding`
- `setChannelWriters`, `setChannelAdmins`, `setChannelReviewers`, `setChannelRateLimits`, `setChannelTypeConstraints`
- `setDeliveryTracking`, `setChannelProtocols`, `setProtocolParticipants`
- `setEnforcementMode`, `setEnforcementExclusions`
- `setRoutingConfig`, `setChannelRedistributionThreshold`, `setChannelRoutingCapacityThreshold`
- `updateChannelSummary`, `configureChannelSummary`, `triggerChannelSummaryUpdate`

**Critical complexity:** `channelDigest` and `checkMessages` contain significant business logic in their QhorusMcpTools implementations. This logic needs to move into API-layer services (e.g., a `ChannelDigestService` or method on `ChannelReader`) rather than living in the GraphQL service layer.

**@Tool methods to remove:** All remaining channel-related @Tool methods (~26).

- [ ] **Step 1: Add query method signatures to ChannelsApi**

- [ ] **Step 2: Add mutation method signatures to ChannelsApi**

- [ ] **Step 3: Handle business logic migration for channelDigest and checkMessages**

The aggregation logic in `channelDigest()` (lines 1770-1861) needs a new API-layer facade method (e.g., `ChannelReader.digest(UUID, int)`) or a dedicated service. Read the QhorusMcpTools implementation and move the logic to the appropriate API-layer class.

Similarly, `checkMessages` semantic handling needs `ConsumerMessaging` to support semantic-aware history retrieval, or the `ChannelsService` needs to implement the semantic dispatch itself using `ChannelReader.findById()` + `ConsumerMessaging.history()`.

- [ ] **Step 4: Write tests for configuration mutations**

- [ ] **Step 5: Write tests for summary, projection, capacity queries**

- [ ] **Step 6: Implement all methods in ChannelsService**

Inject: `ChannelSummaryService` (needs API facade), `ProjectionRegistry` + `ProjectionService` (need API facades), `ProtocolRegistry`, `RoutingBridge` (needs API facade for `diagnose()`), `ActorCapacityView`.

For runtime classes that don't have API-layer interfaces yet, create them:
- `ChannelSummaryManager` in `api/channel/` (or extend `ChannelManager`)
- `ProjectionReader` in `api/` for `listProjections()` + `project()`
- Methods on `ChannelManager` for the missing mutations (clear, forceRelease, enforcement, routing, capacity thresholds)

- [ ] **Step 7: Remove ~26 @Tool methods from QhorusMcpTools**

- [ ] **Step 8: Verify build + commit**

---

## Batch 7: Cleanup — retire QhorusMcpTools + compliance + messaging gap check

### Task 7: Delete QhorusMcpTools and QhorusMcpToolsBase, refactor compliance, verify completeness

**Files:**
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/spi/compliance/ComplianceApi.java` — change `@McpDomain("qhorus")` to `@McpDomain("compliance")`
- Modify: `graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java` — final state with all 7 domains
- Modify: `runtime/src/test/java/.../ToolOverloadDiscoverabilityTest.java` — remove or update (no more @Tool methods)
- Delete: Any remaining QhorusMcpToolsBase records that were not promoted to API layer

**Preconditions:** All 113 @Tool methods have been migrated in Batches 1-6. QhorusMcpTools should have zero `@Tool` methods remaining.

- [ ] **Step 1: Verify zero @Tool methods remain**

Run: `grep -c '@Tool(' runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
Expected: 0

If any remain, they were missed in previous batches — migrate them before proceeding.

- [ ] **Step 2: Check for remaining references to QhorusMcpTools/QhorusMcpToolsBase**

Use `ide_find_references` on both classes. Any remaining references (other than from test files) need to be redirected to the new API/Service classes.

- [ ] **Step 3: Delete QhorusMcpTools.java and QhorusMcpToolsBase.java**

Use `ide_refactor_safe_delete` to find and handle remaining references.

- [ ] **Step 4: Refactor compliance @McpDomain**

In `ComplianceApi.java`, change:
```java
@McpDomain("qhorus")  →  @McpDomain("compliance")
```

- [ ] **Step 5: Update DomainRegistrationTest to final state**

All 7 domains asserted: channels, messaging, governance, agents, data, audit, compliance.
The `noQhorusDomainRemains()` test must include all `*Api` classes.

- [ ] **Step 6: Update or remove ToolOverloadDiscoverabilityTest**

This test guards against `@Tool` overload collisions in `QhorusMcpTools`. With the class deleted, the test is no longer needed — delete it or repurpose it to verify `@PlatformQuery`/`@PlatformMutation` consistency.

- [ ] **Step 7: Full build + test verification**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and tests pass

- [ ] **Step 8: Verify generated resolvers exist**

Check `graphql/target/generated-sources/annotations/` for all 7 domains:
- `GeneratedAgentsResolver.java`, `GeneratedAgentsResource.java`
- `GeneratedDataResolver.java`, `GeneratedDataResource.java`
- `GeneratedAuditResolver.java`, `GeneratedAuditResource.java`
- (channels, messaging, governance already existed)
- `GeneratedComplianceResolver.java`, `GeneratedComplianceResource.java`

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#451): retire QhorusMcpTools — all 113 ops migrated to @McpDomain

Deletes QhorusMcpTools.java (2756 lines) and QhorusMcpToolsBase.java (603 lines).
All operations now discoverable via casehub_activate across 7 domains:
channels, messaging, governance, agents, data, audit, compliance.

Closes #451"
```

---

## Messaging domain note

The existing `MessagingApi` already covers 14 operations — the full messaging domain from the spec. Verify during Batch 6 (channels expansion) whether `check_messages` belongs in channels (as `channelMessages`) or messaging. The spec places it in channels. If any messaging operations were missed, add them during Batch 6 before cleanup.

---

## References

- [docs/specs/issue-409-graphql-mcp-migration/2026-09-11-graphql-mcp-migration-design.md] — design spec this plan implements
- [docs/specs/issue-409-graphql-mcp-migration/decisions.md] — 9 decisions (D1-D9)
- [docs/protocols/casehub/mcp-tool-channel-resolution-boundary.md] — channel resolution at boundary protocol
- [runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java] — 113 @Tool methods to migrate
- [runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java] — ~25 records + utilities to promote/retire
- [api/src/main/java/io/casehub/qhorus/api/spi/channels/ChannelsApi.java] — established pattern reference
- [graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsService.java] — implementation pattern reference
- [graphql/src/test/java/io/casehub/qhorus/graphql/channels/ChannelsServiceQueryTest.java] — test pattern reference
- [graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java] — domain registration guard
- [GitHub #451] — focal issue
- [GitHub #409] — parent epic (GraphQL/MCP migration)
