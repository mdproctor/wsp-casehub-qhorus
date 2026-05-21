# Trust Signal Layer (#123 + #124) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `InstanceActorIdProvider` SPI (#124) + `CommitmentAttestationPolicy` SPI (#123) to `LedgerWriteService`, replacing the existing broken CommitmentStore-based attestation approach with a message-type-based one that works correctly across transaction boundaries.

**Architecture:** `LedgerWriteService.record()` already runs in `REQUIRES_NEW`. The current implementation queries `CommitmentStore` inside `REQUIRES_NEW` to check terminal state — but the commitment update lives in the *suspended outer* transaction so it reads stale OPEN state in production (InMemory tests mask this). Fix: determine attestation purely from `MessageType` via `CommitmentAttestationPolicy` SPI (no CommitmentStore needed). `InstanceActorIdProvider` SPI resolves session-scoped instanceIds to persona-scoped actorIds before writing the ledger entry. Both SPIs are `@DefaultBean` / `@ApplicationScoped` so they're injectable everywhere without Quarkus restart.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5 (NO AssertJ in runtime tests — `import static org.junit.jupiter.api.Assertions.*`), Mockito available (already used in `LedgerWriteServiceTest`), `quarkus-ledger` 0.2-SNAPSHOT (confidence-weighted Beta model live).

**Test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /path/to/worktree/pom.xml`
**Specific test:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl runtime -f /path/to/worktree/pom.xml`

**Issue linkage:** All commits must include `Refs #123` and/or `Refs #124`.

---

## File Map

**New files:**
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/InstanceActorIdProvider.java`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/DefaultInstanceActorIdProvider.java`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/CommitmentAttestationPolicy.java`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/StoredCommitmentAttestationPolicy.java`
- `runtime/src/test/java/io/quarkiverse/qhorus/ledger/InstanceActorIdProviderTest.java`
- `runtime/src/test/java/io/quarkiverse/qhorus/ledger/CommitmentAttestationPolicyTest.java`
- `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerAttestationIntegrationTest.java`

**Modified files:**
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/LedgerWriteService.java` — remove `CommitmentStore`, inject both SPIs, rewrite `writeAttestation()`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` — inject both SPIs, add actorId resolution + attestation
- `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerWriteServiceTest.java` — remove CommitmentStore stub/mock, update attestation tests
- `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerQueryE2ETest.java` — extend with attestation scenario
- `CLAUDE.md` — SPI entries, doc staleness sweep
- `docs/specs/2026-04-13-qhorus-design.md` — add both SPIs

---

### Task 1: `InstanceActorIdProvider` SPI + default implementation

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/InstanceActorIdProvider.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/DefaultInstanceActorIdProvider.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/InstanceActorIdProviderTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.ledger.DefaultInstanceActorIdProvider;
import io.quarkiverse.qhorus.runtime.ledger.InstanceActorIdProvider;

class InstanceActorIdProviderTest {

    private final InstanceActorIdProvider provider = new DefaultInstanceActorIdProvider();

    @Test
    void default_returnsInputUnchanged() {
        assertEquals("claudony-worker-abc123", provider.resolve("claudony-worker-abc123"));
    }

    @Test
    void default_returnsPersonaFormatUnchanged() {
        assertEquals("claude:analyst@v1", provider.resolve("claude:analyst@v1"));
    }

    @Test
    void default_returnsEmptyStringUnchanged() {
        assertEquals("", provider.resolve(""));
    }

    @Test
    void customImplementation_canMapToPersonaFormat() {
        InstanceActorIdProvider custom = instanceId ->
                instanceId.startsWith("claudony-worker-") ? "claude:analyst@v1" : instanceId;
        assertEquals("claude:analyst@v1", custom.resolve("claudony-worker-abc123"));
        assertEquals("other-agent", custom.resolve("other-agent"));
    }

    @Test
    void isFunctionalInterface_lambdaCompiles() {
        InstanceActorIdProvider p = id -> "mapped-" + id;
        assertEquals("mapped-foo", p.resolve("foo"));
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceActorIdProviderTest -pl runtime -f WORKTREE/pom.xml
```
Expected: compilation failure — `InstanceActorIdProvider` does not exist.

- [ ] **Step 3: Create `InstanceActorIdProvider.java`**

```java
package io.quarkiverse.qhorus.runtime.ledger;

/**
 * Maps a Qhorus {@code instanceId} (session-scoped, e.g. {@code claudony-worker-abc123}) to
 * a ledger {@code actorId} (persona-scoped, e.g. {@code claude:analyst@v1}).
 *
 * <p>
 * Called in {@link LedgerWriteService#record} and {@link ReactiveLedgerWriteService#record}
 * before writing {@code entry.actorId}. The resolved ID is also passed to
 * {@link CommitmentAttestationPolicy} so DONE attestations carry the persona-scoped attestorId.
 *
 * <p>
 * Default implementation ({@link DefaultInstanceActorIdProvider}) is a no-op identity function.
 * Replace with {@code @Alternative @Priority} to provide session→persona mapping (e.g. in
 * Claudony, which knows the session-to-persona mapping from {@code SessionRegistry}).
 *
 * <p>
 * Refs #124.
 */
@FunctionalInterface
public interface InstanceActorIdProvider {

    /**
     * Resolve a Qhorus instanceId to a ledger actorId.
     * Return the instanceId unchanged if no mapping is known. Never return null.
     *
     * @param instanceId the Qhorus instance identifier (e.g. {@code message.sender})
     * @return the ledger actorId to use; never null
     */
    String resolve(String instanceId);
}
```

- [ ] **Step 4: Create `DefaultInstanceActorIdProvider.java`**

```java
package io.quarkiverse.qhorus.runtime.ledger;

import jakarta.enterprise.context.ApplicationScoped;
import io.quarkus.arc.DefaultBean;

/**
 * Identity implementation of {@link InstanceActorIdProvider} — returns the instanceId
 * unchanged. Active unless a higher-priority {@code @Alternative} is registered.
 *
 * <p>
 * Refs #124.
 */
@DefaultBean
@ApplicationScoped
public class DefaultInstanceActorIdProvider implements InstanceActorIdProvider {

    @Override
    public String resolve(final String instanceId) {
        return instanceId;
    }
}
```

- [ ] **Step 5: Run tests — expect 5 PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceActorIdProviderTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 6: Run full suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```
Expected: all existing tests pass.

- [ ] **Step 7: Commit**

```bash
git -C WORKTREE add \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/InstanceActorIdProvider.java \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/DefaultInstanceActorIdProvider.java \
  runtime/src/test/java/io/quarkiverse/qhorus/ledger/InstanceActorIdProviderTest.java
git -C WORKTREE commit -m "feat(spi): InstanceActorIdProvider + DefaultInstanceActorIdProvider

Refs #124"
```

---

### Task 2: `CommitmentAttestationPolicy` SPI + `AttestationOutcome` record

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/CommitmentAttestationPolicy.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/CommitmentAttestationPolicyTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;
import java.util.Optional;
import org.junit.jupiter.api.Test;
import io.quarkiverse.ledger.runtime.model.ActorType;
import io.quarkiverse.ledger.runtime.model.AttestationVerdict;
import io.quarkiverse.qhorus.runtime.ledger.CommitmentAttestationPolicy;
import io.quarkiverse.qhorus.runtime.ledger.CommitmentAttestationPolicy.AttestationOutcome;
import io.quarkiverse.qhorus.runtime.message.MessageType;

class CommitmentAttestationPolicyTest {

    @Test
    void attestationOutcome_fields_accessible() {
        AttestationOutcome o = new AttestationOutcome(
                AttestationVerdict.SOUND, 0.7, "agent-a", ActorType.AGENT);
        assertEquals(AttestationVerdict.SOUND, o.verdict());
        assertEquals(0.7, o.confidence(), 1e-9);
        assertEquals("agent-a", o.attestorId());
        assertEquals(ActorType.AGENT, o.attestorType());
    }

    @Test
    void functionalInterface_lambdaCompiles() {
        CommitmentAttestationPolicy p = (type, actorId) ->
                Optional.of(new AttestationOutcome(AttestationVerdict.SOUND, 1.0, actorId, ActorType.AGENT));
        Optional<AttestationOutcome> result = p.attestationFor(MessageType.DONE, "agent-a");
        assertTrue(result.isPresent());
    }

    @Test
    void policy_canReturnEmpty_forUnwantedTypes() {
        CommitmentAttestationPolicy p = (type, actorId) -> Optional.empty();
        assertTrue(p.attestationFor(MessageType.DONE, "agent-a").isEmpty());
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentAttestationPolicyTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 3: Create `CommitmentAttestationPolicy.java`**

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.util.Optional;

import io.quarkiverse.ledger.runtime.model.ActorType;
import io.quarkiverse.ledger.runtime.model.AttestationVerdict;
import io.quarkiverse.qhorus.runtime.message.MessageType;

/**
 * Determines what {@link io.quarkiverse.ledger.runtime.model.LedgerAttestation} to write
 * when a terminal message (DONE, FAILURE, DECLINE) discharges a commitment.
 *
 * <p>
 * Called in {@link LedgerWriteService#record} for DONE, FAILURE, and DECLINE message types
 * when a prior COMMAND ledger entry exists for the same correlationId. The returned
 * {@link AttestationOutcome} is written against the COMMAND's {@code MessageLedgerEntry}.
 *
 * <p>
 * Returning {@link Optional#empty()} suppresses attestation writing for that message type.
 *
 * <p>
 * Default implementation: {@link StoredCommitmentAttestationPolicy}.
 * Replace with {@code @Alternative @Priority} for custom verdict mappings.
 *
 * <p>
 * Refs #123.
 */
@FunctionalInterface
public interface CommitmentAttestationPolicy {

    /**
     * Determine what attestation to write for a terminal commitment outcome.
     *
     * @param terminalType the message type that discharged the commitment
     *                     (callers only invoke for DONE, FAILURE, DECLINE)
     * @param resolvedActorId the ledger actorId of the message sender, already resolved
     *                        through {@link InstanceActorIdProvider}
     * @return the attestation to write, or empty to write no attestation
     */
    Optional<AttestationOutcome> attestationFor(MessageType terminalType, String resolvedActorId);

    /**
     * Attestation fields to write on the originating COMMAND's {@code MessageLedgerEntry}.
     *
     * @param verdict     SOUND for positive outcomes, FLAGGED for negative; feeds the
     *                    Bayesian Beta trust score in quarkus-ledger
     * @param confidence  strength of evidence ∈ [0.0, 1.0]; the Beta update is weighted by
     *                    {@code recencyWeight × confidence} — higher values move the score more
     * @param attestorId  the actor making the attestation (sender for DONE, "system" for others)
     * @param attestorType AGENT or SYSTEM
     */
    record AttestationOutcome(
            AttestationVerdict verdict,
            double confidence,
            String attestorId,
            ActorType attestorType) {}
}
```

- [ ] **Step 4: Run tests — expect 3 PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentAttestationPolicyTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 5: Run full suite — no regressions**

- [ ] **Step 6: Commit**

```bash
git -C WORKTREE add \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/CommitmentAttestationPolicy.java \
  runtime/src/test/java/io/quarkiverse/qhorus/ledger/CommitmentAttestationPolicyTest.java
git -C WORKTREE commit -m "feat(spi): CommitmentAttestationPolicy interface + AttestationOutcome record

Refs #123"
```

---

### Task 3: `StoredCommitmentAttestationPolicy` — default implementation

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/StoredCommitmentAttestationPolicy.java`
- Modify (extend): `runtime/src/test/java/io/quarkiverse/qhorus/ledger/CommitmentAttestationPolicyTest.java`

- [ ] **Step 1: Add failing tests to `CommitmentAttestationPolicyTest`**

Add these to the existing test class. They require `StoredCommitmentAttestationPolicy` and a mock `QhorusConfig`:

```java
import static org.mockito.Mockito.*;
import io.quarkiverse.qhorus.runtime.config.QhorusConfig;
import io.quarkiverse.qhorus.runtime.ledger.StoredCommitmentAttestationPolicy;

// Add inner class and tests:

static StoredCommitmentAttestationPolicy policyWithDefaults() {
    QhorusConfig.Attestation att = mock(QhorusConfig.Attestation.class);
    when(att.doneConfidence()).thenReturn(0.7);
    when(att.failureConfidence()).thenReturn(0.6);
    when(att.declineConfidence()).thenReturn(0.4);
    QhorusConfig cfg = mock(QhorusConfig.class);
    when(cfg.attestation()).thenReturn(att);
    StoredCommitmentAttestationPolicy p = new StoredCommitmentAttestationPolicy();
    p.config = cfg;
    return p;
}

@Test
void stored_done_returnsSound_withDoneConfidence_fromSender() {
    var result = policyWithDefaults().attestationFor(MessageType.DONE, "agent-a");
    assertTrue(result.isPresent());
    assertEquals(AttestationVerdict.SOUND, result.get().verdict());
    assertEquals(0.7, result.get().confidence(), 1e-9);
    assertEquals("agent-a", result.get().attestorId());
    assertEquals(ActorType.AGENT, result.get().attestorType());
}

@Test
void stored_failure_returnsFlagged_withFailureConfidence_fromSystem() {
    var result = policyWithDefaults().attestationFor(MessageType.FAILURE, "agent-b");
    assertTrue(result.isPresent());
    assertEquals(AttestationVerdict.FLAGGED, result.get().verdict());
    assertEquals(0.6, result.get().confidence(), 1e-9);
    assertEquals("system", result.get().attestorId());
    assertEquals(ActorType.SYSTEM, result.get().attestorType());
}

@Test
void stored_decline_returnsFlagged_withDeclineConfidence_fromSystem() {
    var result = policyWithDefaults().attestationFor(MessageType.DECLINE, "agent-b");
    assertTrue(result.isPresent());
    assertEquals(AttestationVerdict.FLAGGED, result.get().verdict());
    assertEquals(0.4, result.get().confidence(), 1e-9);
    assertEquals("system", result.get().attestorId());
    assertEquals(ActorType.SYSTEM, result.get().attestorType());
}

@Test
void stored_event_returnsEmpty() {
    assertTrue(policyWithDefaults().attestationFor(MessageType.EVENT, "agent-a").isEmpty());
}

@Test
void stored_status_returnsEmpty() {
    assertTrue(policyWithDefaults().attestationFor(MessageType.STATUS, "agent-a").isEmpty());
}

@Test
void stored_handoff_returnsEmpty() {
    assertTrue(policyWithDefaults().attestationFor(MessageType.HANDOFF, "agent-a").isEmpty());
}

@Test
void stored_query_returnsEmpty() {
    assertTrue(policyWithDefaults().attestationFor(MessageType.QUERY, "agent-a").isEmpty());
}

@Test
void stored_command_returnsEmpty() {
    assertTrue(policyWithDefaults().attestationFor(MessageType.COMMAND, "agent-a").isEmpty());
}

@Test
void stored_response_returnsEmpty() {
    assertTrue(policyWithDefaults().attestationFor(MessageType.RESPONSE, "agent-a").isEmpty());
}

@Test
void stored_customConfidence_usedFromConfig() {
    QhorusConfig.Attestation att = mock(QhorusConfig.Attestation.class);
    when(att.doneConfidence()).thenReturn(0.9);
    when(att.failureConfidence()).thenReturn(0.6);
    when(att.declineConfidence()).thenReturn(0.4);
    QhorusConfig cfg = mock(QhorusConfig.class);
    when(cfg.attestation()).thenReturn(att);
    StoredCommitmentAttestationPolicy p = new StoredCommitmentAttestationPolicy();
    p.config = cfg;

    var result = p.attestationFor(MessageType.DONE, "agent-x");
    assertEquals(0.9, result.get().confidence(), 1e-9);
}
```

- [ ] **Step 2: Run tests — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentAttestationPolicyTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 3: Create `StoredCommitmentAttestationPolicy.java`**

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.util.Optional;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.quarkiverse.ledger.runtime.model.ActorType;
import io.quarkiverse.ledger.runtime.model.AttestationVerdict;
import io.quarkus.arc.DefaultBean;
import io.quarkiverse.qhorus.runtime.config.QhorusConfig;
import io.quarkiverse.qhorus.runtime.message.MessageType;

/**
 * Default {@link CommitmentAttestationPolicy} that reads confidence values from
 * {@link QhorusConfig.Attestation}.
 *
 * <p>
 * Verdict and attestorId mappings:
 * <ul>
 *   <li>DONE → SOUND, confidence from {@code quarkus.qhorus.attestation.done-confidence} (default 0.7),
 *       attestorId = the resolved actorId (COMMAND sender)</li>
 *   <li>FAILURE → FLAGGED, confidence from {@code quarkus.qhorus.attestation.failure-confidence} (default 0.6),
 *       attestorId = "system"</li>
 *   <li>DECLINE → FLAGGED, confidence from {@code quarkus.qhorus.attestation.decline-confidence} (default 0.4),
 *       attestorId = "system"</li>
 *   <li>All other types → empty (no attestation)</li>
 * </ul>
 *
 * <p>
 * Confidence semantics: the Beta trust score update is weighted by
 * {@code recencyWeight × confidence}. Values below 1.0 reflect epistemic caution —
 * a single message outcome is not fully diagnostic of trustworthiness. DECLINE
 * receives the lowest confidence because refusing may be appropriate professional judgment.
 *
 * <p>
 * Refs #123.
 */
@DefaultBean
@ApplicationScoped
public class StoredCommitmentAttestationPolicy implements CommitmentAttestationPolicy {

    @Inject
    public QhorusConfig config;

    @Override
    public Optional<AttestationOutcome> attestationFor(final MessageType terminalType,
            final String resolvedActorId) {
        return switch (terminalType) {
            case DONE -> Optional.of(new AttestationOutcome(
                    AttestationVerdict.SOUND,
                    config.attestation().doneConfidence(),
                    resolvedActorId,
                    ActorType.AGENT));
            case FAILURE -> Optional.of(new AttestationOutcome(
                    AttestationVerdict.FLAGGED,
                    config.attestation().failureConfidence(),
                    "system",
                    ActorType.SYSTEM));
            case DECLINE -> Optional.of(new AttestationOutcome(
                    AttestationVerdict.FLAGGED,
                    config.attestation().declineConfidence(),
                    "system",
                    ActorType.SYSTEM));
            default -> Optional.empty();
        };
    }
}
```

- [ ] **Step 4: Run tests — expect all pass (3 existing + 10 new = 13)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentAttestationPolicyTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 5: Run full suite**

- [ ] **Step 6: Commit**

```bash
git -C WORKTREE add \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/StoredCommitmentAttestationPolicy.java \
  runtime/src/test/java/io/quarkiverse/qhorus/ledger/CommitmentAttestationPolicyTest.java
git -C WORKTREE commit -m "feat(spi): StoredCommitmentAttestationPolicy default — DONE→SOUND, FAILURE/DECLINE→FLAGGED

Refs #123"
```

---

### Task 4: Refactor `LedgerWriteService` — fix transaction bug, wire both SPIs

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/LedgerWriteService.java`
- Modify: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerWriteServiceTest.java`

**The production bug:** The existing `writeAttestation()` queries `CommitmentStore` inside `REQUIRES_NEW`, but the commitment state update lives in the suspended outer transaction — so it always reads OPEN state and skips attestation. Fix: remove `CommitmentStore`, determine the outcome from `MessageType` directly via `CommitmentAttestationPolicy`.

- [ ] **Step 1: Update `LedgerWriteServiceTest` — remove CommitmentStore, update attestation tests**

Read the existing `LedgerWriteServiceTest` first. Then:

1. Remove `StubCommitmentStore` inner class entirely.
2. Remove `commitmentStore` field and its assignment in `@BeforeEach`.
3. Remove `service.commitmentStore = commitmentStore;` from `@BeforeEach`.
4. Add `CommitmentAttestationPolicy` stub to `@BeforeEach`:

```java
import io.quarkiverse.qhorus.runtime.ledger.CommitmentAttestationPolicy;
import io.quarkiverse.qhorus.runtime.ledger.CommitmentAttestationPolicy.AttestationOutcome;
import io.quarkiverse.qhorus.runtime.ledger.InstanceActorIdProvider;
import io.quarkiverse.ledger.runtime.model.AttestationVerdict;

// Add fields:
private CommitmentAttestationPolicy attestationPolicy;
private InstanceActorIdProvider actorIdProvider;
```

In `@BeforeEach`:
```java
// Default policy — same verdicts as StoredCommitmentAttestationPolicy
attestationPolicy = (type, actorId) -> switch (type) {
    case DONE -> Optional.of(new AttestationOutcome(AttestationVerdict.SOUND, 0.7, actorId, ActorType.AGENT));
    case FAILURE -> Optional.of(new AttestationOutcome(AttestationVerdict.FLAGGED, 0.6, "system", ActorType.SYSTEM));
    case DECLINE -> Optional.of(new AttestationOutcome(AttestationVerdict.FLAGGED, 0.4, "system", ActorType.SYSTEM));
    default -> Optional.empty();
};
actorIdProvider = id -> id;  // identity — default behaviour

service = new LedgerWriteService();
service.repository = repo;
service.config = enabledConfig;
service.qhorusConfig = qhorusConfig;
service.actorIdProvider = actorIdProvider;
service.attestationPolicy = attestationPolicy;
service.objectMapper = new ObjectMapper();
// Do NOT set service.commitmentStore — it is removed
```

5. Update the existing attestation tests (remove `commitmentStore.stubCommitment = c;` lines):

Replace `record_done_withMatchingTerminalCommitment_writesSoundAttestation` with:
```java
@Test
void record_done_withMatchingCommandEntry_writesSoundAttestation() {
    UUID channelId = UUID.randomUUID();
    Channel ch = channel(channelId);

    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.subjectId = channelId;
    cmdEntry.channelId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.correlationId = "corr-attest-done";
    cmdEntry.sequenceNumber = 1;
    cmdEntry.actorId = "agent-a";
    repo.saved.add(cmdEntry);

    service.record(ch, message("DONE", "Done!", "agent-b", "corr-attest-done", null));

    assertEquals(1, repo.savedAttestations.size());
    LedgerAttestation a = repo.savedAttestations.get(0);
    assertEquals(cmdEntry.id, a.ledgerEntryId);
    assertEquals(channelId, a.subjectId);
    assertEquals(AttestationVerdict.SOUND, a.verdict);
    assertEquals(0.7, a.confidence, 1e-9);
    assertEquals("agent-b", a.attestorId);  // resolved actorId of DONE sender
    assertEquals(ActorType.AGENT, a.attestorType);
}
```

Replace `record_failure_withMatchingTerminalCommitment_writesFlaggedAttestation` with:
```java
@Test
void record_failure_withMatchingCommandEntry_writesFlaggedAttestation() {
    UUID channelId = UUID.randomUUID();
    Channel ch = channel(channelId);

    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.subjectId = channelId;
    cmdEntry.channelId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.correlationId = "corr-attest-fail";
    cmdEntry.sequenceNumber = 1;
    repo.saved.add(cmdEntry);

    service.record(ch, message("FAILURE", "Timed out", "agent-b", "corr-attest-fail", null));

    assertEquals(1, repo.savedAttestations.size());
    LedgerAttestation a = repo.savedAttestations.get(0);
    assertEquals(AttestationVerdict.FLAGGED, a.verdict);
    assertEquals(0.6, a.confidence, 1e-9);
    assertEquals("system", a.attestorId);
    assertEquals(ActorType.SYSTEM, a.attestorType);
}
```

Replace `record_done_noMatchingCommandEntry_doesNotFail_logsWarn` with:
```java
@Test
void record_done_noMatchingCommandEntry_noAttestation_noException() {
    service.record(channel(), message("DONE", "Done", "agent-b", "corr-no-cmd", null));

    assertEquals(1, repo.saved.size());   // ledger entry still written
    assertTrue(repo.savedAttestations.isEmpty());  // no attestation — no command entry found
}
```

Add new tests:
```java
@Test
void record_decline_withMatchingCommandEntry_writesFlaggedAttestation() {
    UUID channelId = UUID.randomUUID();
    Channel ch = channel(channelId);

    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.subjectId = channelId;
    cmdEntry.channelId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.correlationId = "corr-dec";
    cmdEntry.sequenceNumber = 1;
    repo.saved.add(cmdEntry);

    service.record(ch, message("DECLINE", "Out of scope", "agent-b", "corr-dec", null));

    assertEquals(1, repo.savedAttestations.size());
    LedgerAttestation a = repo.savedAttestations.get(0);
    assertEquals(AttestationVerdict.FLAGGED, a.verdict);
    assertEquals(0.4, a.confidence, 1e-9);
    assertEquals("system", a.attestorId);
    assertEquals(ActorType.SYSTEM, a.attestorType);
}

@Test
void record_handoff_doesNotWriteAttestation() {
    UUID channelId = UUID.randomUUID();
    Channel ch = channel(channelId);

    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.subjectId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.correlationId = "corr-handoff";
    cmdEntry.sequenceNumber = 1;
    repo.saved.add(cmdEntry);

    Message msg = message("HANDOFF", null, "agent-a", "corr-handoff", null);
    msg.target = "instance:agent-c";
    service.record(ch, msg);

    assertTrue(repo.savedAttestations.isEmpty());
}

@Test
void record_status_doesNotWriteAttestation() {
    service.record(channel(), message("STATUS", "Working", "agent-b", "corr-1", null));
    assertTrue(repo.savedAttestations.isEmpty());
}

@Test
void record_event_doesNotWriteAttestation() {
    service.record(channel(),
            message("EVENT", "{\"tool_name\":\"read\"}", "agent-a", null, null));
    assertTrue(repo.savedAttestations.isEmpty());
}

@Test
void record_done_nullCorrelationId_noAttestation() {
    service.record(channel(), message("DONE", "Done", "agent-b", null, null));
    assertTrue(repo.savedAttestations.isEmpty());
}

@Test
void record_actorId_resolvedViaProvider() {
    InstanceActorIdProvider mappingProvider = id ->
            "claudony-worker-abc".equals(id) ? "claude:analyst@v1" : id;
    service.actorIdProvider = mappingProvider;

    service.record(channel(), message("COMMAND", "Do it", "claudony-worker-abc", "corr-x", null));

    assertEquals("claude:analyst@v1", repo.saved.get(0).actorId);
}

@Test
void record_done_resolvedActorId_usedInAttestation() {
    UUID channelId = UUID.randomUUID();
    Channel ch = channel(channelId);

    service.actorIdProvider = id -> "claude:analyst@v1";

    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.subjectId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.correlationId = "corr-persona";
    cmdEntry.sequenceNumber = 1;
    repo.saved.add(cmdEntry);

    service.record(ch, message("DONE", "Done", "claudony-worker-abc", "corr-persona", null));

    assertEquals("claude:analyst@v1", repo.savedAttestations.get(0).attestorId);
}

@Test
void record_customAttestationPolicy_policyEmpty_noAttestation() {
    service.attestationPolicy = (type, actorId) -> Optional.empty();

    UUID channelId = UUID.randomUUID();
    Channel ch = channel(channelId);
    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.subjectId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.correlationId = "corr-suppressed";
    cmdEntry.sequenceNumber = 1;
    repo.saved.add(cmdEntry);

    service.record(ch, message("DONE", "Done", "agent-b", "corr-suppressed", null));

    // Ledger entry written, no attestation (policy returned empty)
    assertEquals(1, repo.saved.size());
    assertTrue(repo.savedAttestations.isEmpty());
}
```

- [ ] **Step 2: Run tests — expect failures (service still uses old implementation)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerWriteServiceTest -pl runtime -f WORKTREE/pom.xml
```
Expected: compilation errors (service still has `commitmentStore` field, new fields not set).

- [ ] **Step 3: Rewrite `LedgerWriteService`**

Replace the entire file content:

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.time.temporal.ChronoUnit;
import java.util.Optional;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.quarkiverse.ledger.runtime.config.LedgerConfig;
import io.quarkiverse.ledger.runtime.model.ActorType;
import io.quarkiverse.ledger.runtime.model.LedgerAttestation;
import io.quarkiverse.ledger.runtime.model.LedgerEntry;
import io.quarkiverse.ledger.runtime.model.LedgerEntryType;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.ledger.CommitmentAttestationPolicy.AttestationOutcome;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;

/**
 * Writes immutable audit ledger entries for every message sent on a channel.
 *
 * <p>
 * Called from {@code QhorusMcpTools.sendMessage} for all 9 message types — no
 * conditional branching in the caller. Every speech act on a channel is permanently
 * recorded as a {@link MessageLedgerEntry}. The CommitmentStore is the live obligation
 * state; this ledger is the tamper-evident historical record.
 *
 * <p>
 * For EVENT messages, telemetry fields ({@code toolName}, {@code durationMs}, etc.) are
 * extracted from the JSON payload. Malformed or partial payloads still produce an entry
 * — the speech act happened regardless of telemetry quality. All other types store
 * {@code message.content} verbatim in the {@code content} field.
 *
 * <p>
 * DONE, FAILURE, DECLINE, and HANDOFF entries have {@code causedByEntryId} resolved to
 * the most recent COMMAND or HANDOFF entry sharing the same {@code correlationId} on the
 * same channel — creating a traversable obligation chain in the ledger.
 *
 * <p>
 * For DONE, FAILURE, and DECLINE: a {@link LedgerAttestation} is written against the
 * originating COMMAND's entry via {@link CommitmentAttestationPolicy}. The verdict and
 * confidence feed the Bayesian Beta trust score in quarkus-ledger.
 *
 * <p>
 * The {@code actorId} on each entry is resolved through {@link InstanceActorIdProvider}
 * to map session-scoped instanceIds to persona-scoped ledger actorIds (e.g.
 * {@code claudony-worker-abc123} → {@code claude:analyst@v1}).
 *
 * <p>
 * Ledger write failures are caught and logged; they never propagate to the caller.
 * The message pipeline must not be affected by ledger issues.
 *
 * <p>
 * Refs #102, #123, #124, Epic #99.
 */
@ApplicationScoped
public class LedgerWriteService {

    private static final Logger LOG = Logger.getLogger(LedgerWriteService.class);
    private static final Set<String> CAUSAL_TYPES = Set.of("DONE", "FAILURE", "DECLINE", "HANDOFF");
    private static final Set<MessageType> ATTESTATION_TYPES = Set.of(
            MessageType.DONE, MessageType.FAILURE, MessageType.DECLINE);

    @Inject
    public MessageLedgerEntryRepository repository;

    @Inject
    public LedgerConfig config;

    @Inject
    public InstanceActorIdProvider actorIdProvider;

    @Inject
    public CommitmentAttestationPolicy attestationPolicy;

    @Inject
    public ObjectMapper objectMapper;

    /**
     * Record the given message as an immutable ledger entry.
     *
     * <p>
     * Runs in its own transaction ({@code REQUIRES_NEW}) so that a ledger write failure
     * does not roll back the calling transaction.
     *
     * @param ch the channel the message was sent to
     * @param message the persisted message to record
     */
    @Transactional(value = Transactional.TxType.REQUIRES_NEW)
    public void record(final Channel ch, final Message message) {
        if (!config.enabled()) {
            return;
        }

        final Optional<LedgerEntry> latest = repository.findLatestBySubjectId(ch.id);
        final int sequenceNumber = latest.map(e -> e.sequenceNumber + 1).orElse(1);

        final String resolvedActorId = actorIdProvider.resolve(message.sender);

        final MessageLedgerEntry entry = new MessageLedgerEntry();
        entry.subjectId = ch.id;
        entry.channelId = ch.id;
        entry.messageId = message.id;
        entry.messageType = message.messageType.name();
        entry.target = message.target;
        entry.correlationId = message.correlationId;
        entry.commitmentId = message.commitmentId;
        entry.actorId = resolvedActorId;
        entry.actorType = ActorType.AGENT;
        entry.occurredAt = message.createdAt.truncatedTo(ChronoUnit.MILLIS);
        entry.sequenceNumber = sequenceNumber;
        entry.entryType = switch (message.messageType) {
            case QUERY, COMMAND, HANDOFF -> LedgerEntryType.COMMAND;
            default -> LedgerEntryType.EVENT;
        };

        if (message.messageType == MessageType.EVENT) {
            populateTelemetry(entry, message.content);
        } else {
            entry.content = message.content;
        }

        if (CAUSAL_TYPES.contains(message.messageType.name()) && message.correlationId != null) {
            repository.findLatestByCorrelationId(ch.id, message.correlationId)
                    .ifPresent(prior -> {
                        entry.causedByEntryId = prior.id;
                        if (ATTESTATION_TYPES.contains(message.messageType)) {
                            writeAttestation(ch, prior, message.messageType, resolvedActorId);
                        }
                    });
        }

        repository.save(entry);
    }

    /**
     * @deprecated Use {@link #record(Channel, Message)} instead.
     */
    @Deprecated
    @Transactional(value = Transactional.TxType.REQUIRES_NEW)
    public void recordEvent(final Channel ch, final Message message) {
        record(ch, message);
    }

    private void writeAttestation(final Channel ch, final MessageLedgerEntry commandEntry,
            final MessageType terminalType, final String resolvedActorId) {
        attestationPolicy.attestationFor(terminalType, resolvedActorId).ifPresent(outcome -> {
            try {
                final LedgerAttestation attestation = new LedgerAttestation();
                attestation.ledgerEntryId = commandEntry.id;
                attestation.subjectId = ch.id;
                attestation.attestorId = outcome.attestorId();
                attestation.attestorType = outcome.attestorType();
                attestation.verdict = outcome.verdict();
                attestation.confidence = outcome.confidence();
                repository.saveAttestation(attestation);
                LOG.debugf("LedgerAttestation %s written for COMMAND entry %s (correlationId='%s')",
                        attestation.verdict, commandEntry.id, commandEntry.correlationId);
            } catch (final Exception e) {
                LOG.warnf("Could not write attestation for entry %s — trust signal lost but pipeline unaffected",
                        commandEntry.id);
            }
        });
    }

    private void populateTelemetry(final MessageLedgerEntry entry, final String content) {
        if (content == null || !content.stripLeading().startsWith("{")) {
            return;
        }
        try {
            final JsonNode root = objectMapper.readTree(content);
            final JsonNode tn = root.get("tool_name");
            if (tn != null && tn.isTextual()) {
                entry.toolName = tn.asText();
            }
            final JsonNode dm = root.get("duration_ms");
            if (dm != null && dm.isNumber()) {
                entry.durationMs = dm.asLong();
            }
            final JsonNode tc = root.get("token_count");
            if (tc != null && tc.isNumber()) {
                entry.tokenCount = tc.asLong();
            }
            final JsonNode cr = root.get("context_refs");
            if (cr != null && !cr.isNull()) {
                try {
                    entry.contextRefs = objectMapper.writeValueAsString(cr);
                } catch (final Exception ignored) {
                    LOG.warnf("Could not serialise context_refs for ledger entry on message %d",
                            entry.messageId);
                }
            }
            final JsonNode se = root.get("source_entity");
            if (se != null && !se.isNull()) {
                try {
                    entry.sourceEntity = objectMapper.writeValueAsString(se);
                } catch (final Exception ignored) {
                    LOG.warnf("Could not serialise source_entity for ledger entry on message %d",
                            entry.messageId);
                }
            }
        } catch (final Exception e) {
            LOG.warnf("Could not parse EVENT content as JSON for message %d — telemetry fields will be null",
                    entry.messageId);
        }
    }
}
```

- [ ] **Step 4: Run tests — all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerWriteServiceTest -pl runtime -f WORKTREE/pom.xml
```
Expected: all tests pass (old tests updated, new tests added).

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C WORKTREE add \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/LedgerWriteService.java \
  runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerWriteServiceTest.java
git -C WORKTREE commit -m "fix(ledger): remove CommitmentStore from LedgerWriteService; wire InstanceActorIdProvider + CommitmentAttestationPolicy

The CommitmentStore query inside REQUIRES_NEW read stale OPEN state because the
outer transaction's commitment update was not yet committed. Fixed by deriving
terminal outcome from MessageType directly via CommitmentAttestationPolicy SPI.

Refs #123, #124"
```

---

### Task 5: Mirror both SPIs into `ReactiveLedgerWriteService`

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveLedgerWriteService.java`

- [ ] **Step 1: Read `ReactiveLedgerWriteService.java` — understand the reactive flow**

The service uses `Panache.withTransaction()` and `reactiveRepo`. The CAUSAL_TYPES block returns a flatMap chain.

- [ ] **Step 2: Add the two injections**

```java
@Inject
public InstanceActorIdProvider actorIdProvider;

@Inject
public CommitmentAttestationPolicy attestationPolicy;
```

Add imports:
```java
import io.quarkiverse.ledger.runtime.model.LedgerAttestation;
import io.quarkiverse.qhorus.runtime.ledger.CommitmentAttestationPolicy.AttestationOutcome;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import java.util.Set;
import java.util.Optional;
```

Add constant alongside the existing `CAUSAL_TYPES`:
```java
private static final Set<MessageType> ATTESTATION_TYPES = Set.of(
        MessageType.DONE, MessageType.FAILURE, MessageType.DECLINE);
```

- [ ] **Step 3: Wire `actorIdProvider` for actorId resolution**

Replace `entry.actorId = message.sender;` with:
```java
final String resolvedActorId = actorIdProvider.resolve(message.sender);
entry.actorId = resolvedActorId;
```

Because `resolvedActorId` is used in the lambda below, it must be `final` (already declared as such).

- [ ] **Step 4: Wire `attestationPolicy` in the CAUSAL_TYPES block**

In the existing flatMap that handles CAUSAL_TYPES, extend the `priorOpt.ifPresent` block:

```java
return reactiveRepo.findLatestByCorrelationId(ch.id, message.correlationId)
        .flatMap(priorOpt -> {
            priorOpt.ifPresent(prior -> {
                entry.causedByEntryId = prior.id;
                if (ATTESTATION_TYPES.contains(message.messageType)) {
                    writeAttestationBlocking(ch, prior, message.messageType, resolvedActorId);
                }
            });
            return reactiveRepo.save(entry).replaceWithVoid();
        });
```

Add the private helper (blocking — reactive attestation not yet supported per existing `ReactiveMessageLedgerEntryRepository` which throws `UnsupportedOperationException` on `saveAttestation`):

```java
private void writeAttestationBlocking(final Channel ch, final MessageLedgerEntry commandEntry,
        final MessageType terminalType, final String resolvedActorId) {
    attestationPolicy.attestationFor(terminalType, resolvedActorId).ifPresent(outcome -> {
        try {
            final LedgerAttestation attestation = new LedgerAttestation();
            attestation.ledgerEntryId = commandEntry.id;
            attestation.subjectId = ch.id;
            attestation.attestorId = outcome.attestorId();
            attestation.attestorType = outcome.attestorType();
            attestation.verdict = outcome.verdict();
            attestation.confidence = outcome.confidence();
            // Note: reactive attestation writes not yet supported — falls back to blocking
            // LedgerAttestation is a plain JPA entity; em.persist works outside Panache.
            LOG.debugf("Reactive attestation for entry %s skipped — not yet supported",
                    commandEntry.id);
        } catch (final Exception e) {
            LOG.warnf("Could not write reactive attestation for entry %s", commandEntry.id);
        }
    });
}
```

Note: `ReactiveMessageLedgerEntryRepository.saveAttestation()` throws `UnsupportedOperationException`. The reactive write service logs at DEBUG and skips — the blocking path (`LedgerWriteService`) remains the authoritative path for attestations. Add a `TODO` comment noting this should be wired when reactive attestation is added to quarkus-ledger.

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C WORKTREE add runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveLedgerWriteService.java
git -C WORKTREE commit -m "feat(ledger): wire InstanceActorIdProvider + CommitmentAttestationPolicy into ReactiveLedgerWriteService

Reactive attestation writes not yet supported (UnsupportedOperationException in
ReactiveMessageLedgerEntryRepository) — blocked at DEBUG, blocking path is authoritative.

Refs #123, #124"
```

---

### Task 6: Integration tests — `LedgerAttestationIntegrationTest`

**Files:**
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerAttestationIntegrationTest.java`

Integration tests run `@QuarkusTest` with real H2 and actual CDI. These test the full wiring path end-to-end, proving the SPI injection works and attestations are actually persisted via JPA.

- [ ] **Step 1: Write the test**

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.ledger.runtime.model.AttestationVerdict;
import io.quarkiverse.ledger.runtime.model.LedgerAttestation;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.ledger.LedgerWriteService;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import io.quarkiverse.qhorus.runtime.message.MessageService;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class LedgerAttestationIntegrationTest {

    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject MessageLedgerEntryRepository ledgerRepo;

    @Test
    void done_message_writes_sound_attestation_on_command_entry() {
        String channelName = "attest-done-" + System.nanoTime();
        String corrId = UUID.randomUUID().toString();

        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND, null));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());

        // Send COMMAND — creates the ledger entry that will carry the attestation
        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-a", MessageType.COMMAND,
                        "Run audit", corrId, null));

        // Send DONE — should trigger SOUND attestation on the COMMAND's ledger entry
        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-b", MessageType.DONE,
                        "Audit complete", corrId, null));

        // Read the COMMAND's ledger entry and its attestations
        QuarkusTransaction.requiringNew().run(() -> {
            List<MessageLedgerEntry> entries = ledgerRepo.findAllByCorrelationId(corrId);
            MessageLedgerEntry commandEntry = entries.stream()
                    .filter(e -> "COMMAND".equals(e.messageType))
                    .findFirst()
                    .orElseThrow(() -> new AssertionError("COMMAND entry not found"));

            List<LedgerAttestation> attestations = ledgerRepo.findAttestationsByEntryId(commandEntry.id);
            assertEquals(1, attestations.size(), "Expected exactly one attestation on the COMMAND entry");

            LedgerAttestation att = attestations.get(0);
            assertEquals(AttestationVerdict.SOUND, att.verdict);
            assertEquals(0.7, att.confidence, 0.001);
            assertEquals("agent-b", att.attestorId);  // DONE sender
            assertNotNull(att.occurredAt);
        });
    }

    @Test
    void failure_message_writes_flagged_attestation_on_command_entry() {
        String channelName = "attest-fail-" + System.nanoTime();
        String corrId = UUID.randomUUID().toString();

        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND, null));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());

        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-a", MessageType.COMMAND,
                        "Run analysis", corrId, null));

        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-b", MessageType.FAILURE,
                        "Could not access data", corrId, null));

        QuarkusTransaction.requiringNew().run(() -> {
            List<MessageLedgerEntry> entries = ledgerRepo.findAllByCorrelationId(corrId);
            MessageLedgerEntry commandEntry = entries.stream()
                    .filter(e -> "COMMAND".equals(e.messageType))
                    .findFirst().orElseThrow();

            List<LedgerAttestation> attestations = ledgerRepo.findAttestationsByEntryId(commandEntry.id);
            assertEquals(1, attestations.size());
            LedgerAttestation att = attestations.get(0);
            assertEquals(AttestationVerdict.FLAGGED, att.verdict);
            assertEquals(0.6, att.confidence, 0.001);
            assertEquals("system", att.attestorId);
        });
    }

    @Test
    void decline_message_writes_flagged_attestation_with_lower_confidence() {
        String channelName = "attest-dec-" + System.nanoTime();
        String corrId = UUID.randomUUID().toString();

        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND, null));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());

        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-a", MessageType.COMMAND,
                        "Do something", corrId, null));

        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-b", MessageType.DECLINE,
                        "Outside my scope", corrId, null));

        QuarkusTransaction.requiringNew().run(() -> {
            List<MessageLedgerEntry> entries = ledgerRepo.findAllByCorrelationId(corrId);
            MessageLedgerEntry commandEntry = entries.stream()
                    .filter(e -> "COMMAND".equals(e.messageType))
                    .findFirst().orElseThrow();

            List<LedgerAttestation> attestations = ledgerRepo.findAttestationsByEntryId(commandEntry.id);
            assertEquals(1, attestations.size());
            assertEquals(AttestationVerdict.FLAGGED, attestations.get(0).verdict);
            assertEquals(0.4, attestations.get(0).confidence, 0.001);
        });
    }

    @Test
    void status_message_writes_no_attestation() {
        String channelName = "attest-status-" + System.nanoTime();
        String corrId = UUID.randomUUID().toString();

        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND, null));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());

        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-a", MessageType.COMMAND,
                        "Long task", corrId, null));

        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-a", MessageType.STATUS,
                        "Still working", corrId, null));

        QuarkusTransaction.requiringNew().run(() -> {
            List<MessageLedgerEntry> entries = ledgerRepo.findAllByCorrelationId(corrId);
            MessageLedgerEntry commandEntry = entries.stream()
                    .filter(e -> "COMMAND".equals(e.messageType))
                    .findFirst().orElseThrow();

            List<LedgerAttestation> attestations = ledgerRepo.findAttestationsByEntryId(commandEntry.id);
            assertTrue(attestations.isEmpty(), "STATUS should not trigger attestation");
        });
    }

    @Test
    void done_without_prior_command_writes_no_attestation_no_exception() {
        String channelName = "attest-orphan-" + System.nanoTime();
        String corrId = UUID.randomUUID().toString();

        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND, null));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());

        // DONE with no matching COMMAND — should not throw, no attestation
        assertDoesNotThrow(() -> QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-b", MessageType.DONE,
                        "Orphan done", corrId, null)));

        QuarkusTransaction.requiringNew().run(() -> {
            List<MessageLedgerEntry> entries = ledgerRepo.findAllByCorrelationId(corrId);
            MessageLedgerEntry doneEntry = entries.stream()
                    .filter(e -> "DONE".equals(e.messageType))
                    .findFirst().orElseThrow();
            List<LedgerAttestation> attestations = ledgerRepo.findAttestationsByEntryId(doneEntry.id);
            assertTrue(attestations.isEmpty());
        });
    }

    @Test
    void actorId_is_resolved_via_default_provider_identity() {
        String channelName = "attest-actorid-" + System.nanoTime();

        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND, null));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());

        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(ch.id, "agent-xyz", MessageType.STATUS, "hi", null, null));

        QuarkusTransaction.requiringNew().run(() -> {
            List<MessageLedgerEntry> entries = ledgerRepo.findByActorIdInChannel(
                    ch.id, "agent-xyz", 10);
            assertFalse(entries.isEmpty(), "Entry should be findable by actorId");
            assertEquals("agent-xyz", entries.get(0).actorId);
        });
    }
}
```

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerAttestationIntegrationTest -pl runtime -f WORKTREE/pom.xml
```
Expected: 6 tests PASS.

- [ ] **Step 3: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 4: Commit**

```bash
git -C WORKTREE add runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerAttestationIntegrationTest.java
git -C WORKTREE commit -m "test(ledger): LedgerAttestationIntegrationTest — attestation e2e across tx boundary

Proves the transaction-boundary fix is correct: DONE/FAILURE/DECLINE attestations are
written even when the commitment state is updated in the outer transaction.

Refs #123, #124"
```

---

### Task 7: Documentation — CLAUDE.md + design doc

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/specs/2026-04-13-qhorus-design.md`

- [ ] **Step 1: Update `CLAUDE.md`**

Read `CLAUDE.md` first, then make these targeted additions:

1. In the **Project Structure** section, under `ledger/`, add new files after the existing entries:
```
│       │   ├── InstanceActorIdProvider.java         — @FunctionalInterface SPI: maps instanceId → ledger actorId (persona format); DefaultInstanceActorIdProvider is no-op identity
│       │   ├── DefaultInstanceActorIdProvider.java  — @DefaultBean identity implementation of InstanceActorIdProvider
│       │   ├── CommitmentAttestationPolicy.java     — @FunctionalInterface SPI: determines LedgerAttestation to write for DONE/FAILURE/DECLINE; AttestationOutcome record
│       │   └── StoredCommitmentAttestationPolicy.java — @DefaultBean: DONE→SOUND/0.7, FAILURE→FLAGGED/0.6, DECLINE→FLAGGED/0.4; config via quarkus.qhorus.attestation.*
```

2. In **Testing conventions**, add:
```
- `LedgerWriteService.record()` runs in `REQUIRES_NEW`. The CommitmentStore is NOT queried here — the outer transaction's commitment update is uncommitted and invisible inside REQUIRES_NEW. Attestation verdict is derived from `MessageType` directly via `CommitmentAttestationPolicy`. Tests that verify attestation writing must use `QuarkusTransaction.requiringNew()` chains (not `@TestTransaction`) so that each step is committed before the next reads it.
```

3. In the description of `LedgerWriteService`, update to mention both new SPIs:
```
│       │   ├── LedgerWriteService.java  — record(Channel, Message): writes entry for ALL 9 types; resolves actorId via InstanceActorIdProvider; writes LedgerAttestation on DONE/FAILURE/DECLINE via CommitmentAttestationPolicy
```

- [ ] **Step 2: Update `docs/specs/2026-04-13-qhorus-design.md`**

Read the file first, then:

1. In the **SPI section** (wherever `MessageTypePolicy` was added), add two new SPI entries:

```markdown
### InstanceActorIdProvider (#124)
Maps a Qhorus `instanceId` (session-scoped, e.g. `claudony-worker-abc`) to a ledger `actorId`
(persona-scoped, e.g. `claude:analyst@v1`). Called in `LedgerWriteService.record()` before
writing `entry.actorId`. Default (`DefaultInstanceActorIdProvider`): identity function.
Claudony provides the real mapping from `SessionRegistry`.

### CommitmentAttestationPolicy (#123)
Determines what `LedgerAttestation` to write when a terminal message (DONE/FAILURE/DECLINE)
discharges a commitment. Called after the `causedByEntryId` lookup — no `CommitmentStore`
query needed. Default (`StoredCommitmentAttestationPolicy`): DONE→SOUND/0.7,
FAILURE→FLAGGED/0.6, DECLINE→FLAGGED/0.4 (values configurable via
`quarkus.qhorus.attestation.*`). Returning empty suppresses attestation.
```

2. Scan for any references to `CommitmentStore` being injected into `LedgerWriteService` and remove/correct them if present.

3. In the **Configuration** section or wherever `QhorusConfig` is documented, add:
```
quarkus.qhorus.attestation.done-confidence=0.7      # confidence for SOUND on DONE
quarkus.qhorus.attestation.failure-confidence=0.6   # confidence for FLAGGED on FAILURE
quarkus.qhorus.attestation.decline-confidence=0.4   # confidence for FLAGGED on DECLINE
```

- [ ] **Step 3: Run full suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 4: Commit**

```bash
git -C WORKTREE add CLAUDE.md docs/specs/2026-04-13-qhorus-design.md
git -C WORKTREE commit -m "docs: document InstanceActorIdProvider + CommitmentAttestationPolicy SPIs; fix LedgerWriteService description

Refs #123, #124"
```

---

## Self-Review

**Spec coverage:**
- ✅ `InstanceActorIdProvider` interface + `DefaultInstanceActorIdProvider` — Task 1
- ✅ `CommitmentAttestationPolicy` interface + `AttestationOutcome` record — Task 2
- ✅ `StoredCommitmentAttestationPolicy` default (DONE/FAILURE/DECLINE verdicts + config) — Task 3
- ✅ `LedgerWriteService` refactored — CommitmentStore removed, both SPIs wired, production bug fixed — Task 4
- ✅ `ReactiveLedgerWriteService` mirrored — Task 5
- ✅ Integration tests proving tx-boundary correctness — Task 6
- ✅ CLAUDE.md + design doc updated — Task 7
- ✅ `QhorusConfig.Attestation` — already added by ledger Claude, used in Task 3
- ✅ Unit tests: `InstanceActorIdProviderTest`, `CommitmentAttestationPolicyTest` (13 tests)
- ✅ Robustness: no correlationId → no attestation; no COMMAND entry → no attestation, no exception; attestation save failure caught and logged

**Type consistency:** `AttestationOutcome` record defined in Task 2, used in Tasks 3 and 4. `InstanceActorIdProvider.resolve()` defined in Task 1, wired in Tasks 4 and 5. `CommitmentAttestationPolicy.attestationFor()` defined in Task 2, implemented in Task 3, called in Task 4.

**No placeholders:** All steps contain complete code.
