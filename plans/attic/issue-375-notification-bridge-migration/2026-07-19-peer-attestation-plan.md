# Peer Attestation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #356 — feat: peer attestation — multi-agent verification on ledger entries
**Issue group:** #356

**Goal:** Enable agents to formally attest to other agents' work via ENDORSED/CHALLENGED verdicts on ledger entries, with automatic review triggering and structured response handling.

**Architecture:** Layered from a root primitive (PeerAttestationWriter) through MCP tools (attest, request_peer_review, list_attestations) to automation (auto-trigger after DONE, structured response parsing). No new entities — uses existing LedgerAttestation with unused ENDORSED/CHALLENGED verdicts. Channel gains `reviewerInstances` field.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger 0.2-SNAPSHOT, Panache, Flyway

## Global Constraints

- Pre-release platform — breaking changes are free
- Blocking stack only (v1) — observers gated with `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "false", enableIfMissing = true)`
- All MCP tools gated with `@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)`
- CDI-free unit tests: `tracingConfig` must be set to a disabled-tracing implementation
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` for runtime tests
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` for full build after API changes
- Commit messages: `feat(#356): <description>`

---

### Task 1: Channel `reviewerInstances` field — API, entity, migration, stores

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ReactiveChannelManager.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V37__channel_reviewer_instances.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelReviewerInstancesTest.java`

**Interfaces:**
- Produces: `Channel.reviewerInstances()` → `List<String>`, `ChannelService.setReviewerInstances(UUID, List<String>)` → `Channel`, `ChannelCreateRequest.Builder.reviewerInstances(List<String>)`

- [ ] **Step 1: Write the failing test**

```java
// ChannelReviewerInstancesTest.java — CDI-free unit test
@Test
void channel_record_includes_reviewer_instances() {
    var ch = Channel.builder("test-ch")
                    .reviewerInstances(List.of("reviewer-a", "reviewer-b"))
                    .build();
    assertThat(ch.reviewerInstances()).containsExactly("reviewer-a", "reviewer-b");
}

@Test
void channel_record_defaults_reviewer_instances_to_empty() {
    var ch = Channel.builder("test-ch").build();
    assertThat(ch.reviewerInstances()).isEmpty();
}

@Test
void channel_create_request_includes_reviewer_instances() {
    var req = ChannelCreateRequest.builder("test-ch")
                                  .reviewerInstances(List.of("rev-1"))
                                  .build();
    assertThat(req.reviewerInstances()).containsExactly("rev-1");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelReviewerInstancesTest -pl runtime`
Expected: FAIL — `reviewerInstances` method does not exist

- [ ] **Step 3: Add `reviewerInstances` to Channel record**

Add the field after `spaceId` in the Channel record. Update the compact constructor to normalize null → `List.of()`. Update `fromRequest()` to pass `req.reviewerInstances()`. Update `toBuilder()` to include `.reviewerInstances(reviewerInstances)`. Update `Builder` with the field, setter method, and `build()` call.

- [ ] **Step 4: Add `reviewerInstances` to ChannelCreateRequest record**

Add the field after `spaceId`. Update the compact constructor null normalization. Add the backward-compatible constructor (pre-reviewerInstances arity) defaulting to `null`. Update Builder with field, setter, and build().

- [ ] **Step 5: Add `reviewerInstances` to ChannelDetail record**

Add `String reviewerInstances` field after `spaceName`.

- [ ] **Step 6: Add `reviewerInstances` to ChannelEntity**

Add JPA column:
```java
@Column(name = "reviewer_instances", columnDefinition = "TEXT")
public String reviewerInstances;
```
Update `fromDomain()` to map `joinCsv(channel.reviewerInstances())`.
Update `toDomain()` to pass `splitCsv(reviewerInstances)`.

- [ ] **Step 7: Update QhorusEntityMapper.toChannelDetail()**

Add `joinCsv(ch.reviewerInstances())` to the ChannelDetail construction.

- [ ] **Step 8: Add `setReviewerInstances` to ChannelManager and ChannelService**

In `ChannelManager.java` (api):
```java
Channel setReviewerInstances(UUID channelId, List<String> reviewerInstances);
```

In `ChannelService.java` (runtime), follow the `setAllowedWriters` pattern:
```java
@Override
@Transactional
public Channel setReviewerInstances(UUID channelId, List<String> reviewerInstances) {
    Channel ch = channelStore.find(channelId).orElseThrow(() ->
            new IllegalArgumentException("Channel not found: " + channelId));
    return channelStore.put(ch.toBuilder().reviewerInstances(reviewerInstances).build());
}
```

Add the same to `ReactiveChannelManager` and `ReactiveChannelService` with `Uni<Channel>` return.

- [ ] **Step 9: Add MCP tool `set_channel_reviewers`**

In `QhorusMcpTools.java`, follow `set_channel_writers` pattern:
```java
@Tool(name = "set_channel_reviewers", description = "Update the reviewer list on an existing channel. "
        + "Reviewers receive automatic peer review QUERYs after DONE messages.")
@Transactional
public ChannelDetail setChannelReviewers(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "reviewer_ids", description = "Comma-separated reviewer instance IDs. Null = disable auto-review.", required = false) String reviewerIds) {
    Channel resolved = resolveChannel(channel);
    Channel ch       = channelService.setReviewerInstances(resolved.id(), splitCsv(reviewerIds));
    return toChannelDetail(ch, messageStore.countByChannel(ch.id()));
}
```

Add `reviewer_ids` parameter to `create_channel` tool. Add same tools to `ReactiveQhorusMcpTools`.

- [ ] **Step 10: Create Flyway migration V37**

```sql
-- V37__channel_reviewer_instances.sql
ALTER TABLE channel ADD COLUMN reviewer_instances TEXT;
```

- [ ] **Step 11: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: PASS — full build including all modules

- [ ] **Step 12: Commit**

```bash
git add -A && git commit -m "feat(#356): channel reviewerInstances field, migration V37, MCP tools"
```

---

### Task 2: Config — peer attestation confidence values

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/config/QhorusConfigPeerAttestationTest.java`

**Interfaces:**
- Produces: `QhorusConfig.Attestation.peerEndorsedConfidence()` → `double` (0.4), `QhorusConfig.Attestation.peerChallengedConfidence()` → `double` (0.5)

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class QhorusConfigPeerAttestationTest {
    @Inject QhorusConfig config;

    @Test
    void peer_confidence_defaults() {
        assertThat(config.attestation().peerEndorsedConfidence()).isEqualTo(0.4);
        assertThat(config.attestation().peerChallengedConfidence()).isEqualTo(0.5);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusConfigPeerAttestationTest -pl runtime`
Expected: FAIL — methods do not exist

- [ ] **Step 3: Add config methods to QhorusConfig.Attestation**

In `QhorusConfig.java`, add to the `Attestation` interface:
```java
@WithDefault("0.4")
double peerEndorsedConfidence();

@WithDefault("0.5")
double peerChallengedConfidence();
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusConfigPeerAttestationTest -pl runtime`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(#356): peer attestation confidence config defaults"
```

---

### Task 3: PeerAttestationWriter — the root primitive

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/PeerAttestationWriter.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/PeerAttestationWriterTest.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.findEntryById(UUID, String)`, `LedgerEntryRepository.saveAttestation(LedgerAttestation, String)`, `InstanceStore.findByInstanceId(String)`, `QhorusConfig.Attestation.peerEndorsedConfidence()`, `QhorusConfig.Attestation.peerChallengedConfidence()`
- Produces: `PeerAttestationWriter.write(UUID ledgerEntryId, AttestationVerdict verdict, String evidence, String attestorId, String tenancyId)` → `LedgerAttestation`

- [ ] **Step 1: Write failing tests**

```java
// PeerAttestationWriterTest.java — CDI-free unit test
class PeerAttestationWriterTest {
    private PeerAttestationWriter writer;
    private StubLedgerEntryRepository ledger; // from runtime test fixtures
    private InMemoryInstanceStore instanceStore;

    @BeforeEach
    void setUp() {
        writer = new PeerAttestationWriter();
        ledger = new StubLedgerEntryRepository();
        instanceStore = new InMemoryInstanceStore();
        writer.ledger = ledger;
        writer.instanceStore = instanceStore;
        writer.config = stubConfig(0.4, 0.5);
        writer.objectMapper = new ObjectMapper();
    }

    @Test
    void write_endorsed_creates_attestation() {
        // set up a COMMAND entry in the stub ledger
        // register an instance as the attestor
        // call writer.write(entryId, ENDORSED, "good work", attestorId, tenancyId)
        // assert attestation saved with verdict=ENDORSED, attestorRole="peer-reviewer",
        //   attestorType=AGENT, confidence=0.4
    }

    @Test
    void write_challenged_uses_challenged_confidence() { ... }
    @Test
    void write_rejects_sound_verdict() { ... }
    @Test
    void write_rejects_flagged_verdict() { ... }
    @Test
    void write_rejects_missing_entry() { ... }
    @Test
    void write_rejects_non_command_entry() { ... }
    @Test
    void write_rejects_self_attestation() { ... }
    @Test
    void write_rejects_unregistered_attestor() { ... }
    @Test
    void write_sets_subject_id_from_entry() { ... }
    @Test
    void write_extracts_capability_tag_from_entry_content() { ... }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerAttestationWriterTest -pl runtime`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement PeerAttestationWriter**

```java
@ApplicationScoped
class PeerAttestationWriter {
    @Inject public LedgerEntryRepository ledger;
    @Inject public InstanceStore instanceStore;
    @Inject public QhorusConfig config;
    @Inject public ObjectMapper objectMapper;

    LedgerAttestation write(UUID ledgerEntryId, AttestationVerdict verdict,
                            String evidence, String attestorId, String tenancyId) {
        // validate verdict is ENDORSED or CHALLENGED
        // find entry, guard COMMAND/HANDOFF
        // guard self-attestation: attestorId != entry.actorId
        // guard attestor registered: instanceStore.findByInstanceId(attestorId)
        // extract capabilityTag from entry content (reuse pattern from StoredCommitmentAttestationPolicy)
        // build LedgerAttestation, set attestorRole="peer-reviewer", confidence from config
        // save and return
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerAttestationWriterTest -pl runtime`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(#356): PeerAttestationWriter — root primitive with validation"
```

---

### Task 4: PeerReviewRequestedEvent — CDI event in api/spi/

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/PeerReviewRequestedEvent.java`

**Interfaces:**
- Produces: `PeerReviewRequestedEvent(UUID ledgerEntryId, UUID channelId, String tenancyId)` — CDI event record

- [ ] **Step 1: Create the event record**

```java
package io.casehub.qhorus.api.spi;

import java.util.UUID;

public record PeerReviewRequestedEvent(
        UUID ledgerEntryId,
        UUID channelId,
        String tenancyId) {}
```

- [ ] **Step 2: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat(#356): PeerReviewRequestedEvent CDI event in api/spi"
```

---

### Task 5: ReviewerResolver — fallback chain

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReviewerResolver.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/ReviewerResolverTest.java`

**Interfaces:**
- Consumes: `ChannelStore.find(UUID)`, `InstanceService.findByCapability(String)`, `PeerReviewRequestedEvent` CDI event
- Produces: `ReviewerResolver.resolve(UUID channelId, List<String> explicitReviewerIds, UUID ledgerEntryId, String tenancyId)` → `List<String>`

- [ ] **Step 1: Write failing tests**

```java
class ReviewerResolverTest {
    @Test void explicit_reviewers_returned_directly() { ... }
    @Test void channel_config_used_when_no_explicit() { ... }
    @Test void capability_routing_used_when_no_channel_config() { ... }
    @Test void cdi_event_fired_when_all_layers_empty() { ... }
    @Test void explicit_wins_over_channel_config() { ... }
}
```

CDI-free: mock ChannelStore, InstanceService, Event<PeerReviewRequestedEvent>.

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement ReviewerResolver**

```java
@ApplicationScoped
class ReviewerResolver {
    @Inject ChannelStore channelStore;
    @Inject InstanceService instanceService;
    @Inject Event<PeerReviewRequestedEvent> reviewRequestedEvent;

    List<String> resolve(UUID channelId, List<String> explicitReviewerIds,
                         UUID ledgerEntryId, String tenancyId) {
        if (explicitReviewerIds != null && !explicitReviewerIds.isEmpty()) {
            return explicitReviewerIds;
        }
        // channel config
        var ch = channelStore.find(channelId);
        if (ch.isPresent() && !ch.get().reviewerInstances().isEmpty()) {
            return ch.get().reviewerInstances();
        }
        // capability routing
        var capable = instanceService.findByCapability("peer-reviewer");
        if (!capable.isEmpty()) {
            return capable.stream().map(i -> i.instanceId).toList();
        }
        // CDI event — external consumer handles
        reviewRequestedEvent.fireAsync(new PeerReviewRequestedEvent(ledgerEntryId, channelId, tenancyId));
        return List.of();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(#356): ReviewerResolver — 4-layer fallback chain"
```

---

### Task 6: MCP tools — attest, list_attestations, request_peer_review

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/mcp/PeerAttestationMcpToolsTest.java`

**Interfaces:**
- Consumes: `PeerAttestationWriter.write()`, `ReviewerResolver.resolve()`, `LedgerEntryRepository.findAttestationsByEntryId()`, `MessageService.dispatch()`
- Produces: MCP tools `attest`, `list_attestations`, `request_peer_review`

- [ ] **Step 1: Write failing integration test**

```java
@QuarkusTest
class PeerAttestationMcpToolsTest {
    @Inject QhorusMcpTools tools;

    @Test
    void attest_writes_endorsed_attestation() {
        // create channel, register instances, dispatch COMMAND, dispatch DONE
        // call tools.attest(entryId.toString(), "ENDORSED", "verified output")
        // verify attestation exists via tools.listAttestations(entryId.toString())
    }

    @Test
    void attest_rejects_sound_verdict() { ... }
    @Test
    void list_attestations_returns_policy_and_peer() { ... }
    @Test
    void request_peer_review_sends_query_to_reviewers() { ... }
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement `attest` tool in QhorusMcpTools**

```java
@Tool(name = "attest", description = "Record a peer attestation (ENDORSED or CHALLENGED) "
        + "on a COMMAND or HANDOFF ledger entry.")
@Transactional
public Map<String, Object> attest(
        @ToolArg(name = "entry_id", description = "UUID of the COMMAND/HANDOFF ledger entry") String entryId,
        @ToolArg(name = "verdict", description = "ENDORSED or CHALLENGED") String verdict,
        @ToolArg(name = "evidence", description = "Free-text evidence for the attestation") String evidence) {
    UUID id = UUID.fromString(entryId);
    AttestationVerdict v = AttestationVerdict.valueOf(verdict.toUpperCase());
    String tenancyId = currentPrincipal.tenancyId();
    String attestorId = currentPrincipal.actorId();
    LedgerAttestation a = peerAttestationWriter.write(id, v, evidence, attestorId, tenancyId);
    return Map.of("attestation_id", a.id, "entry_id", id, "verdict", v.name(), "attestor_id", attestorId);
}
```

- [ ] **Step 4: Implement `list_attestations` tool**

```java
@Tool(name = "list_attestations", description = "List all attestations (policy and peer) on a ledger entry.")
public List<Map<String, Object>> listAttestations(
        @ToolArg(name = "entry_id", description = "UUID of the ledger entry") String entryId) {
    UUID id = UUID.fromString(entryId);
    String tenancyId = currentPrincipal.tenancyId();
    return ledger.findAttestationsByEntryId(id, tenancyId).stream()
            .map(a -> Map.<String, Object>of(
                    "attestation_id", a.id,
                    "verdict", a.verdict.name(),
                    "attestor_id", a.attestorId,
                    "attestor_role", a.attestorRole != null ? a.attestorRole : "policy",
                    "evidence", a.evidence != null ? a.evidence : "",
                    "confidence", a.confidence,
                    "occurred_at", a.occurredAt.toString()))
            .toList();
}
```

- [ ] **Step 5: Implement `request_peer_review` tool**

```java
@Tool(name = "request_peer_review", description = "Send peer review QUERYs to reviewers for a COMMAND/HANDOFF entry.")
@Transactional
public Map<String, Object> requestPeerReview(
        @ToolArg(name = "entry_id", description = "UUID of the COMMAND/HANDOFF ledger entry") String entryId,
        @ToolArg(name = "reviewer_ids", description = "Comma-separated reviewer instance IDs. If omitted, resolved from channel config / capability routing.", required = false) String reviewerIds,
        @ToolArg(name = "channel", description = "Channel for the review QUERYs. Defaults to the entry's channel.", required = false) String channel) {
    // resolve entry, validate COMMAND/HANDOFF
    // look up completion_content from most recent terminal entry on same correlationId
    // resolve reviewers via ReviewerResolver
    // for each reviewer, dispatch QUERY with peer_review content
    // return list of sent reviews
}
```

- [ ] **Step 6: Add same tools to ReactiveQhorusMcpTools (blocking wrappers)**

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerAttestationMcpToolsTest -pl runtime`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add -A && git commit -m "feat(#356): MCP tools — attest, list_attestations, request_peer_review"
```

---

### Task 7: PeerReviewAutoTrigger — DONE → review QUERY observer

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/PeerReviewAutoTrigger.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/PeerReviewAutoTriggerTest.java`

**Interfaces:**
- Consumes: `MessageReceivedEvent`, `MessageLedgerEntryRepository.findEarliestWithSubjectByCorrelationId()`, `ReviewerResolver.resolve()`, `MessageDispatcher.dispatch()`
- Produces: Review QUERYs dispatched on DONE events (side effect)

- [ ] **Step 1: Write failing tests**

```java
class PeerReviewAutoTriggerTest {
    // CDI-free: mock messageRepo, reviewerResolver, messageDispatcher
    @Test void fires_review_query_after_done() { ... }
    @Test void ignores_non_done_messages() { ... }
    @Test void ignores_done_with_null_correlation_id() { ... }
    @Test void ignores_when_entry_is_not_command() { ... }
    @Test void noop_when_no_reviewers_found() { ... }
    @Test void uses_entry_channel_id_not_event_channel_id() { ... }
    @Test void each_review_query_gets_own_correlation_id() { ... }
    @Test void sender_is_entry_actor_id() { ... }
    @Test void query_content_includes_peer_review_json() { ... }
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement PeerReviewAutoTrigger**

```java
@ApplicationScoped
@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)
class PeerReviewAutoTrigger implements MessageObserver {
    @Inject MessageLedgerEntryRepository messageRepo;
    @Inject ReviewerResolver reviewerResolver;
    @Inject MessageDispatcher messageDispatcher;
    @Inject ObjectMapper objectMapper;

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (event.messageType() != MessageType.DONE) return;
        if (event.correlationId() == null) return;

        var entry = messageRepo.findEarliestWithSubjectByCorrelationId(
                event.correlationId(), event.tenancyId()).orElse(null);
        if (entry == null) return;
        if (!"COMMAND".equals(entry.messageType) && !"HANDOFF".equals(entry.messageType)) return;

        var reviewers = reviewerResolver.resolve(
                entry.channelId, List.of(), entry.id, event.tenancyId());
        if (reviewers.isEmpty()) return;

        for (String reviewerId : reviewers) {
            // build peer_review JSON content
            // dispatch QUERY with own UUID correlationId, sender = entry.actorId
        }
    }

    @Override
    public Scope scope() { return Scope.LOCAL; }
}
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(#356): PeerReviewAutoTrigger — DONE → review QUERY observer"
```

---

### Task 8: PeerReviewResponseHandler — structured RESPONSE → attestation

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/PeerReviewResponseHandler.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/PeerReviewResponseHandlerTest.java`

**Interfaces:**
- Consumes: `MessageReceivedEvent`, `PeerAttestationWriter.write()`
- Produces: LedgerAttestation entries from structured RESPONSE content (side effect)

- [ ] **Step 1: Write failing tests**

```java
class PeerReviewResponseHandlerTest {
    // CDI-free: mock peerAttestationWriter
    @Test void parses_structured_response_and_writes_attestation() { ... }
    @Test void ignores_non_response_messages() { ... }
    @Test void ignores_response_without_peer_review_response_key() { ... }
    @Test void ignores_response_with_null_content() { ... }
    @Test void falls_back_silently_on_malformed_json() { ... }
    @Test void uses_config_confidence_not_response_confidence() { ... }
    @Test void extracts_verdict_and_evidence_from_response() { ... }
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement PeerReviewResponseHandler**

```java
@ApplicationScoped
@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)
class PeerReviewResponseHandler implements MessageObserver {
    private static final Logger LOG = Logger.getLogger(PeerReviewResponseHandler.class);
    @Inject PeerAttestationWriter peerAttestationWriter;
    @Inject ObjectMapper objectMapper;

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (event.messageType() != MessageType.RESPONSE) return;
        if (event.content() == null) return;

        try {
            var root = objectMapper.readTree(event.content());
            var reviewResponse = root.get("peer_review_response");
            if (reviewResponse == null) return;

            var entryId = UUID.fromString(reviewResponse.get("ledger_entry_id").asText());
            var verdict = AttestationVerdict.valueOf(reviewResponse.get("verdict").asText().toUpperCase());
            var evidence = reviewResponse.has("evidence") ? reviewResponse.get("evidence").asText() : null;

            peerAttestationWriter.write(entryId, verdict, evidence,
                    event.senderId(), event.tenancyId());
        } catch (Exception e) {
            LOG.warnf(e, "Could not parse peer_review_response from RESPONSE on channel %s — "
                    + "use attest() tool to record attestation manually", event.channelName());
        }
    }

    @Override
    public Scope scope() { return Scope.LOCAL; }
}
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(#356): PeerReviewResponseHandler — structured RESPONSE → attestation"
```

---

### Task 9: Integration test — full round-trip

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/ledger/PeerAttestationIntegrationTest.java`

**Interfaces:**
- Consumes: All components from Tasks 1-8

- [ ] **Step 1: Write integration test**

```java
@QuarkusTest
class PeerAttestationIntegrationTest {
    @Inject QhorusMcpTools tools;
    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;

    @Test
    void full_round_trip_command_done_review_attest() {
        // 1. Create channel with reviewerInstances
        // 2. Register requester + reviewer instances (reviewer with "peer-reviewer" capability)
        // 3. Dispatch COMMAND via requester
        // 4. Dispatch DONE via obligor (different agent)
        // 5. Verify auto-trigger sent a review QUERY
        // 6. Dispatch structured RESPONSE with peer_review_response JSON
        // 7. Verify attestation was auto-written (list_attestations shows ENDORSED)
        // 8. Verify both policy and peer attestations coexist
    }

    @Test
    void explicit_attest_without_review_query() {
        // 1. Create channel, register instances, dispatch COMMAND + DONE
        // 2. Call attest() directly — no review QUERY needed
        // 3. Verify ENDORSED attestation exists
    }
}
```

Uses `QuarkusTransaction.requiringNew()` for dispatch per observer transaction discipline.

- [ ] **Step 2: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerAttestationIntegrationTest -pl runtime`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat(#356): peer attestation integration test — full round-trip"
```

---

### Task 10: CLAUDE.md update + full build verification

**Files:**
- Modify: `CLAUDE.md`
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/ToolOverloadDiscoverabilityTest.java` (verify no overload conflicts)

- [ ] **Step 1: Update CLAUDE.md with new conventions**

Add peer attestation documentation to the testing conventions section:
- PeerAttestationWriter, ReviewerResolver, PeerReviewAutoTrigger, PeerReviewResponseHandler
- New MCP tools: attest, list_attestations, request_peer_review, set_channel_reviewers
- Channel.reviewerInstances field
- Config: casehub.qhorus.attestation.peer-endorsed-confidence, peer-challenged-confidence
- V37 migration

- [ ] **Step 2: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: PASS — all modules compile and test

- [ ] **Step 3: Verify ToolOverloadDiscoverabilityTest passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ToolOverloadDiscoverabilityTest -pl runtime`
Expected: PASS — no public non-@Tool overloads share names with @Tool methods

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "docs(#356): CLAUDE.md — peer attestation conventions and testing notes"
```
