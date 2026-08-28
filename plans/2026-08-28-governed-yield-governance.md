# Governed Yield Governance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #411 — judgment commitment type (reframed: composition pattern + attestation feedback)
**Issue group:** #411, #412, #414

**Goal:** Governed yields compose with existing 10-type taxonomy — no new message types. Add judgment attestation feedback via deferred attestation policy + verification observer, and formal verification properties as offline audit tools in the compliance-report module.

**Architecture:** `JudgmentCommitmentAttestationPolicy` extends `StoredCommitmentAttestationPolicy` to return `Optional.empty()` for judgment DONE (deferring attestation). `JudgmentVerificationObserver` (MessageObserver) writes the sole attestation when VERIFIED events land. Five `VerificationProperty` predicate classes check liveness, safety, deadlock freedom, fairness, and evidence completeness against the ledger, surfaced as `PROPERTY_VERIFICATION` report type.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger API (AttestationVerdict, LedgerAttestation, LedgerEntryRepository)

## Global Constraints

- No Flyway migrations — all data in existing tables
- No new MessageType enum values — ADR-0005 stopping criterion holds
- Judgment confidence values configurable via `casehub.qhorus.attestation.*`
- Verification properties are trace checkers (observed history), not model checkers (system proofs)
- `compliance-report` module contains all new classes (attestation policy, observer, verification properties)
- All code navigation via IntelliJ MCP (`ide_find_class`, `ide_find_references`); structural edits via `ide_edit_member`/`ide_insert_member`/`ide_replace_member`

---

## Batch 1: Judgment Attestation Feedback

### Task 1: JudgmentCommitmentAttestationPolicy — deferred attestation for judgment commitments

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/attestation/JudgmentCommitmentAttestationPolicy.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/attestation/JudgmentCommitmentAttestationPolicyTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java` — add `hasJudgmentEvent()`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` — add judgment confidence config

**Interfaces:**
- Consumes: `StoredCommitmentAttestationPolicy` (parent class, `runtime/ledger/`), `CommitmentContext` (api/spi/), `MessageLedgerEntryRepository.hasJudgmentEvent()`
- Produces: `JudgmentCommitmentAttestationPolicy` — overrides `attestationFor()` to return `Optional.empty()` for judgment DONE. Later tasks reference: `config.attestation().judgmentAcceptedConfidence()`, `.judgmentRejectedConfidence()`, `.judgmentPartialConfidence()`

- [ ] **Step 1: Add `hasJudgmentEvent()` to `MessageLedgerEntryRepository`**

Use `ide_insert_member` to add:

```java
public boolean hasJudgmentEvent(String correlationId, String tenancyId) {
    return getEntityManager().createQuery(
            "SELECT COUNT(e) FROM MessageLedgerEntry e " +
            "WHERE e.correlationId = :corrId " +
            "AND e.tenancyId = :tenancyId " +
            "AND e.toolName = :yieldedKind", Long.class)
        .setParameter("corrId", correlationId)
        .setParameter("tenancyId", tenancyId != null ? tenancyId
                : io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID)
        .setParameter("yieldedKind",
                io.casehub.qhorus.api.judgment.JudgmentEventKinds.YIELDED)
        .getSingleResult() > 0;
}
```

- [ ] **Step 2: Add judgment confidence config to `QhorusConfig.Attestation`**

Use `ide_edit_member` on the `Attestation` interface in `QhorusConfig.java` to add after the existing `responseConfidence()`:

```java
@WithDefault("0.7")
double judgmentAcceptedConfidence();

@WithDefault("0.3")
double judgmentRejectedConfidence();

@WithDefault("0.5")
double judgmentPartialConfidence();
```

- [ ] **Step 3: Write failing test for the policy**

Create test file. CDI-free, Mockito mocks.

```java
package io.casehub.qhorus.compliance.attestation;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.CommitmentAttestationPolicy.AttestationOutcome;
import io.casehub.qhorus.api.spi.CommitmentContext;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class JudgmentCommitmentAttestationPolicyTest {

    private JudgmentCommitmentAttestationPolicy policy;
    private MessageLedgerEntryRepository messageRepo;
    private CurrentPrincipal principal;
    private QhorusConfig config;

    @BeforeEach
    void setUp() {
        policy = new JudgmentCommitmentAttestationPolicy();
        messageRepo = mock(MessageLedgerEntryRepository.class);
        principal = mock(CurrentPrincipal.class);
        config = mock(QhorusConfig.class, RETURNS_DEEP_STUBS);

        policy.messageRepo = messageRepo;
        policy.currentPrincipal = principal;
        policy.config = config;
        policy.evidentialChecker = null;

        when(principal.tenancyId()).thenReturn("default");
        when(config.attestation().doneConfidence()).thenReturn(0.7);
    }

    @Test
    void doneOnJudgmentCommitmentReturnsEmpty() {
        var ctx = new CommitmentContext("corr-1", UUID.randomUUID(), "ch", null, null, null, null, null);
        when(messageRepo.hasJudgmentEvent("corr-1", "default")).thenReturn(true);

        Optional<AttestationOutcome> result = policy.attestationFor(MessageType.DONE, "actor-1", ctx);

        assertThat(result).isEmpty();
    }

    @Test
    void doneOnNonJudgmentCommitmentDelegatesToParent() {
        var ctx = new CommitmentContext("corr-2", UUID.randomUUID(), "ch", null, null, null, null, null);
        when(messageRepo.hasJudgmentEvent("corr-2", "default")).thenReturn(false);

        Optional<AttestationOutcome> result = policy.attestationFor(MessageType.DONE, "actor-1", ctx);

        assertThat(result).isPresent();
        assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
    }

    @Test
    void failureAlwaysDelegatesToParent() {
        var ctx = new CommitmentContext("corr-3", UUID.randomUUID(), "ch", null, null, null, null, null);
        when(messageRepo.hasJudgmentEvent("corr-3", "default")).thenReturn(true);

        Optional<AttestationOutcome> result = policy.attestationFor(MessageType.FAILURE, "actor-1", ctx);

        assertThat(result).isPresent();
        assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.FLAGGED);
    }

    @Test
    void nullCorrelationIdDelegatesToParent() {
        var ctx = new CommitmentContext(null, UUID.randomUUID(), "ch", null, null, null, null, null);

        Optional<AttestationOutcome> result = policy.attestationFor(MessageType.DONE, "actor-1", ctx);

        assertThat(result).isPresent();
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentCommitmentAttestationPolicyTest -pl compliance-report`
Expected: FAIL — class does not exist yet

- [ ] **Step 5: Implement `JudgmentCommitmentAttestationPolicy`**

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/attestation/JudgmentCommitmentAttestationPolicy.java`:

```java
package io.casehub.qhorus.compliance.attestation;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.CommitmentContext;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import io.casehub.qhorus.runtime.ledger.StoredCommitmentAttestationPolicy;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Optional;

@ApplicationScoped
public class JudgmentCommitmentAttestationPolicy extends StoredCommitmentAttestationPolicy {

    @Inject
    public MessageLedgerEntryRepository messageRepo;

    @Inject
    public CurrentPrincipal currentPrincipal;

    @Override
    public Optional<AttestationOutcome> attestationFor(final MessageType terminalType,
            final String resolvedActorId, final CommitmentContext context) {
        if (terminalType == MessageType.DONE && isJudgmentCommitment(context)) {
            return Optional.empty();
        }
        return super.attestationFor(terminalType, resolvedActorId, context);
    }

    private boolean isJudgmentCommitment(final CommitmentContext ctx) {
        if (ctx == null || ctx.correlationId() == null) return false;
        String tenancyId = currentPrincipal.tenancyId();
        return messageRepo.hasJudgmentEvent(ctx.correlationId(), tenancyId);
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentCommitmentAttestationPolicyTest -pl compliance-report`
Expected: PASS (all 4 tests)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/src runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#411): judgment attestation policy — defer DONE attestation for judgment commitments

Extends StoredCommitmentAttestationPolicy to return Optional.empty() for
DONE on judgment commitments (detected via YIELDED event on same correlationId).
FAILURE/DECLINE always delegate to parent — expired judgments get standard
trust penalty. Adds hasJudgmentEvent() repository method and judgment
confidence config properties.

Refs #411"
```

---

### Task 2: JudgmentVerificationObserver — write attestation on VERIFIED events

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/attestation/JudgmentVerificationObserver.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/attestation/JudgmentVerificationObserverTest.java`

**Interfaces:**
- Consumes: `MessageObserver` SPI (api/gateway/), `MessageReceivedEvent` (api/gateway/), `LedgerEntryRepository.saveAttestation()`, `MessageLedgerEntryRepository.findByMessageId()`, `.findLatestByCorrelationId()`, `JudgmentEventKinds.VERIFIED`, `QhorusConfig.Attestation.judgmentAcceptedConfidence()` etc. (from Task 1)
- Produces: `JudgmentVerificationObserver` — writes `LedgerAttestation` with `attestorId="system:judgment-verifier"`, `AttestationVerdict.SOUND` or `FLAGGED`, configurable confidence.

- [ ] **Step 1: Write failing test**

CDI-free, Mockito mocks. The observer queries `MessageLedgerEntryRepository` for the ledger entry (since `MessageReceivedEvent` does not carry telemetry columns and EVENT `content` is enforced null).

```java
package io.casehub.qhorus.compliance.attestation;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class JudgmentVerificationObserverTest {

    private JudgmentVerificationObserver observer;
    private LedgerEntryRepository ledger;
    private MessageLedgerEntryRepository messageRepo;
    private QhorusConfig config;

    @BeforeEach
    void setUp() {
        observer = new JudgmentVerificationObserver();
        ledger = mock(LedgerEntryRepository.class);
        messageRepo = mock(MessageLedgerEntryRepository.class);
        config = mock(QhorusConfig.class, RETURNS_DEEP_STUBS);

        observer.ledger = ledger;
        observer.messageRepo = messageRepo;
        observer.config = config;

        when(config.attestation().judgmentAcceptedConfidence()).thenReturn(0.7);
        when(config.attestation().judgmentRejectedConfidence()).thenReturn(0.3);
        when(config.attestation().judgmentPartialConfidence()).thenReturn(0.5);
    }

    @Test
    void acceptedVerificationWritesSoundAttestation() {
        UUID channelId = UUID.randomUUID();
        String corrId = UUID.randomUUID().toString();
        Long messageId = 42L;

        var verifiedEntry = buildEntry(channelId, "judgment_verified",
                "ACCEPTED", 0.85, corrId);
        var commandEntry = buildEntry(channelId, null, null, null, corrId);
        commandEntry.id = UUID.randomUUID();
        commandEntry.subjectId = UUID.randomUUID();

        when(messageRepo.findByMessageId(messageId)).thenReturn(Optional.of(verifiedEntry));
        when(messageRepo.findLatestByCorrelationId(channelId, corrId, "default"))
                .thenReturn(Optional.of(commandEntry));

        var event = new MessageReceivedEvent(messageId, "ch", channelId,
                "default", MessageType.EVENT, "engine", null, null,
                corrId, Instant.now(), null, null, null);
        observer.onMessage(event);

        var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
        verify(ledger).saveAttestation(captor.capture(), eq("default"));
        var att = captor.getValue();
        assertThat(att.verdict).isEqualTo(AttestationVerdict.SOUND);
        assertThat(att.confidence).isEqualTo(0.85);
        assertThat(att.attestorId).isEqualTo("system:judgment-verifier");
        assertThat(att.ledgerEntryId).isEqualTo(commandEntry.id);
    }

    @Test
    void rejectedVerificationWritesFlaggedAttestation() {
        UUID channelId = UUID.randomUUID();
        String corrId = UUID.randomUUID().toString();
        Long messageId = 43L;

        var verifiedEntry = buildEntry(channelId, "judgment_verified",
                "REJECTED", null, corrId);
        var commandEntry = buildEntry(channelId, null, null, null, corrId);
        commandEntry.id = UUID.randomUUID();
        commandEntry.subjectId = UUID.randomUUID();

        when(messageRepo.findByMessageId(messageId)).thenReturn(Optional.of(verifiedEntry));
        when(messageRepo.findLatestByCorrelationId(channelId, corrId, "default"))
                .thenReturn(Optional.of(commandEntry));

        var event = new MessageReceivedEvent(messageId, "ch", channelId,
                "default", MessageType.EVENT, "engine", null, null,
                corrId, Instant.now(), null, null, null);
        observer.onMessage(event);

        var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
        verify(ledger).saveAttestation(captor.capture(), eq("default"));
        assertThat(captor.getValue().verdict).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(captor.getValue().confidence).isEqualTo(0.3);
    }

    @Test
    void nonEventMessageIgnored() {
        var event = new MessageReceivedEvent(1L, "ch", UUID.randomUUID(),
                "default", MessageType.COMMAND, "agent", null, null,
                null, Instant.now(), "hello", null, null);
        observer.onMessage(event);
        verifyNoInteractions(ledger);
    }

    @Test
    void nonVerifiedEventIgnored() {
        Long messageId = 44L;
        var entry = buildEntry(UUID.randomUUID(), "judgment_yielded",
                null, null, "corr");
        when(messageRepo.findByMessageId(messageId)).thenReturn(Optional.of(entry));

        var event = new MessageReceivedEvent(messageId, "ch", UUID.randomUUID(),
                "default", MessageType.EVENT, "engine", null, null,
                "corr", Instant.now(), null, null, null);
        observer.onMessage(event);
        verifyNoInteractions(ledger);
    }

    private MessageLedgerEntry buildEntry(UUID channelId, String toolName,
            String verificationOutcome, Double evidenceQuality, String corrId) {
        var entry = new MessageLedgerEntry();
        entry.channelId = channelId;
        entry.toolName = toolName;
        entry.verificationOutcome = verificationOutcome;
        entry.evidenceQuality = evidenceQuality;
        entry.correlationId = corrId;
        entry.tenancyId = "default";
        return entry;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentVerificationObserverTest -pl compliance-report`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement `JudgmentVerificationObserver`**

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/attestation/JudgmentVerificationObserver.java`:

```java
package io.casehub.qhorus.compliance.attestation;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.judgment.JudgmentEventKinds;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class JudgmentVerificationObserver implements MessageObserver {

    private static final Logger LOG = Logger.getLogger(JudgmentVerificationObserver.class);

    @Inject public LedgerEntryRepository ledger;
    @Inject public MessageLedgerEntryRepository messageRepo;
    @Inject public QhorusConfig config;

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (event.messageType() != MessageType.EVENT) return;
        if (event.messageId() == null) return;

        var verifiedEntry = messageRepo.findByMessageId(event.messageId());
        if (verifiedEntry.isEmpty()) return;
        MessageLedgerEntry entry = verifiedEntry.get();
        if (!JudgmentEventKinds.VERIFIED.equals(entry.toolName)) return;

        String tenancyId = event.tenancyId();
        var commandEntry = messageRepo.findLatestByCorrelationId(
                entry.channelId, event.correlationId(), tenancyId);
        if (commandEntry.isEmpty()) return;

        MessageLedgerEntry target = commandEntry.get();
        AttestationVerdict verdict = mapVerdict(entry.verificationOutcome);
        double confidence = mapConfidence(entry.verificationOutcome, entry.evidenceQuality);

        LedgerAttestation attestation = new LedgerAttestation();
        attestation.ledgerEntryId = target.id;
        attestation.subjectId = target.subjectId;
        attestation.attestorId = "system:judgment-verifier";
        attestation.attestorType = ActorType.SYSTEM;
        attestation.verdict = verdict;
        attestation.confidence = confidence;
        attestation.capabilityTag = CapabilityTag.GLOBAL;

        try {
            ledger.saveAttestation(attestation, tenancyId);
            LOG.debugf("Judgment attestation %s/%.2f written for entry %s (correlationId=%s)",
                    verdict, confidence, target.id, event.correlationId());
        } catch (Exception e) {
            LOG.warnf(e, "Failed to write judgment attestation for entry %s", target.id);
        }
    }

    private AttestationVerdict mapVerdict(String verificationOutcome) {
        if ("ACCEPTED".equals(verificationOutcome)) return AttestationVerdict.SOUND;
        return AttestationVerdict.FLAGGED;
    }

    private double mapConfidence(String verificationOutcome, Double evidenceQuality) {
        return switch (verificationOutcome != null ? verificationOutcome : "") {
            case "ACCEPTED" -> Math.max(
                    config.attestation().judgmentAcceptedConfidence(),
                    evidenceQuality != null ? evidenceQuality : 0.7);
            case "REJECTED" -> config.attestation().judgmentRejectedConfidence();
            case "PARTIAL" -> config.attestation().judgmentPartialConfidence();
            default -> config.attestation().judgmentRejectedConfidence();
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentVerificationObserverTest -pl compliance-report`
Expected: PASS (all 4 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/src
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#411): judgment verification observer — write attestation on VERIFIED events

MessageObserver picks up VERIFIED events, queries ledger entry for
telemetry columns, finds the COMMAND entry via correlationId, writes
attestation with configurable confidence (ACCEPTED→SOUND, REJECTED→FLAGGED).

Refs #411"
```

---

## Batch 2: Formal Verification Properties — Foundation + First Two Properties

### Task 3: Verification property interfaces + Liveness + Safety properties

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/VerificationProperty.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/RemediatingProperty.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/CheckResult.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/PropertyViolation.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/LivenessProperty.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/SafetyProperty.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/LivenessPropertyTest.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/SafetyPropertyTest.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CommitmentStore.java` — add `findOpenOlderThan()`

**Interfaces:**
- Consumes: `CommitmentStore` (api/store/), `MessageLedgerEntryRepository` (runtime/ledger/), `LedgerEntryRepository` (casehub-ledger)
- Produces: `VerificationProperty` interface, `RemediatingProperty` interface, `CheckResult`, `PropertyViolation` records. `LivenessProperty.check()`, `SafetyProperty.check()`.

- [ ] **Step 1: Create property interfaces and records**

Create `VerificationProperty.java`:
```java
package io.casehub.qhorus.compliance.verification;

import java.time.Instant;

public interface VerificationProperty {
    String name();
    String ctlFormula();
    String description();
    CheckResult check(String tenancyId, Instant from, Instant to);
}
```

Create `RemediatingProperty.java`:
```java
package io.casehub.qhorus.compliance.verification;

import java.time.Instant;

public interface RemediatingProperty extends VerificationProperty {
    int remediate(String tenancyId, Instant from, Instant to);
}
```

Create `CheckResult.java`:
```java
package io.casehub.qhorus.compliance.verification;

import java.util.List;

public record CheckResult(List<PropertyViolation> violations, int remediationsAvailable) {
    public boolean passed() { return violations.isEmpty(); }
}
```

Create `PropertyViolation.java`:
```java
package io.casehub.qhorus.compliance.verification;

import java.time.Instant;

public record PropertyViolation(
    String propertyName,
    String description,
    String evidence,
    Instant occurredAt,
    String severity
) {}
```

- [ ] **Step 2: Add `findOpenOlderThan()` to `CommitmentStore`**

Use `ide_insert_member` on `CommitmentStore.java`:
```java
List<io.casehub.qhorus.api.message.Commitment> findOpenOlderThan(Instant cutoff, String tenancyId);
```

Implement in `JpaCommitmentStore`, `InMemoryCommitmentStore`, and `CrossTenantCommitmentStore`. JPA query:
```sql
FROM CommitmentEntity c WHERE c.state = 'OPEN' AND c.createdAt < :cutoff AND c.tenancyId = :tenancyId
```

- [ ] **Step 3: Write failing tests for LivenessProperty and SafetyProperty**

`LivenessPropertyTest.java` — mock `CommitmentStore`, verify OPEN commitments older than threshold are violations.

`SafetyPropertyTest.java` — mock `MessageLedgerEntryRepository`, verify DONE entries without attestation are violations. New repository method: `findDoneEntriesWithoutAttestation(Instant from, Instant to, String tenancyId)`.

- [ ] **Step 4: Implement LivenessProperty and SafetyProperty**

`LivenessProperty`: queries `commitmentStore.findOpenOlderThan(cutoff, tenancyId)` where `cutoff = Instant.now().minus(livenessThreshold)`. Config: `casehub.qhorus.verification.liveness-threshold` (default `PT24H`).

`SafetyProperty`: queries `messageRepo.findDoneEntriesWithoutAttestation(from, to, tenancyId)`. New repository method joins `message_ledger_entry` LEFT JOIN `ledger_attestation` WHERE attestation IS NULL AND messageType = 'DONE'.

- [ ] **Step 5: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="LivenessPropertyTest,SafetyPropertyTest" -pl compliance-report`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src compliance-report/src runtime/src persistence-memory/src
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#414): verification property interfaces + liveness and safety properties

VerificationProperty/RemediatingProperty interfaces. LivenessProperty checks
for commitments OPEN beyond threshold. SafetyProperty checks for DONE entries
missing attestation. Adds findOpenOlderThan() to CommitmentStore.

Refs #414"
```

---

### Task 4: Deadlock freedom, Fairness, and Evidence completeness properties

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/DeadlockFreedomProperty.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/FairnessProperty.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/EvidenceCompletenessProperty.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/DeadlockFreedomPropertyTest.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/FairnessPropertyTest.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/EvidenceCompletenessPropertyTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java` — add `findRoutingEntries()`, `findDoneEntriesWithDeferredAttestation()`

**Interfaces:**
- Consumes: `CommitmentStore.findAllByCorrelationId()` (existing), `MessageLedgerEntryRepository.findJudgmentEvents()` (existing from #413), `MessageLedgerEntryRepository.findPendingJudgments()` (existing from #413), `LedgerEntryRepository.saveAttestation()`, `QhorusConfig`
- Produces: Three property implementations. `EvidenceCompletenessProperty` also implements `RemediatingProperty`.

- [ ] **Step 1: Add new repository methods**

`findRoutingEntries(Instant from, Instant to, String tenancyId)` — returns entries WHERE `routing_selected_agent IS NOT NULL`.

`findDoneEntriesWithDeferredAttestation(String tenancyId)` — DONE entries that have a matching VERIFIED event (same correlationId) but no attestation. Joins `message_ledger_entry` with `ledger_attestation`.

- [ ] **Step 2: Write failing tests for all three properties**

`DeadlockFreedomPropertyTest`: Create commitment chains with cycles, verify detection. Reuses `CommitmentStore.findAllByCorrelationId()` logic.

`FairnessPropertyTest`: Build routing entries with concentrated and uniform distributions. Verify Gini coefficient computation flags concentration above threshold (default 0.5). Config: `casehub.qhorus.verification.fairness-threshold` (default 0.5 — lowered from original 0.7 per review finding R1-16).

`EvidenceCompletenessPropertyTest`: Verify pending judgments detected. Verify `check()` is side-effect-free. Verify `remediate()` writes missing attestations.

- [ ] **Step 3: Implement all three properties**

`DeadlockFreedomProperty`: For each HANDOFF commitment in the time window, walk the delegation chain via `findAllByCorrelationId()`. Detect repeated obligors (cycle). Same logic as `WatchdogEvaluationService.evaluateCircularDelegation()` but sweeps ALL correlationIds, not just watched channels.

`FairnessProperty`: Query routing entries, group by `routing_selected_agent`, compute Gini coefficient. Entries with `routing_candidate_count == 1` excluded (monopoly is forced, not unfair). Config threshold default 0.5.

`EvidenceCompletenessProperty`: Two sub-checks: (1) pending judgments — YIELDED with no VERIFIED/ESCALATED (reuses `findPendingJudgments`); (2) deferred attestation — DONE entries with VERIFIED event but no attestation. `remediate()` writes missing attestations using same mapping as `JudgmentVerificationObserver`.

- [ ] **Step 4: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="DeadlockFreedomPropertyTest,FairnessPropertyTest,EvidenceCompletenessPropertyTest" -pl compliance-report`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/src runtime/src
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#414): deadlock freedom, fairness, and evidence completeness properties

DeadlockFreedomProperty sweeps all HANDOFF chains for cycles. FairnessProperty
computes Gini coefficient over routing selection distribution (threshold 0.5).
EvidenceCompletenessProperty checks for pending judgments and deferred attestations
with remediate() for recovery.

Refs #414"
```

---

## Batch 3: Report Integration + Issue Closures

### Task 5: PropertyVerificationReport type + service + scheduler + REST/GraphQL

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/PropertyVerificationReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/PropertyResult.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/PropertyVerificationService.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/PropertyVerificationReportType.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/PropertyResultType.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/PropertyViolationType.java`
- Create: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/PropertyVerificationServiceTest.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportType.java` — add `PROPERTY_VERIFICATION`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduler.java` — add case
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java` — add endpoint
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/ComplianceQueryResolver.java` — add query
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/CsvReportRenderer.java` — add case
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlReportRenderer.java` — add case

**Interfaces:**
- Consumes: All 5 `VerificationProperty` implementations (Task 3-4), `ComplianceReportStorageService`, `LedgerVerificationService`
- Produces: `PropertyVerificationReport` model, `PROPERTY_VERIFICATION` report type, REST endpoint `GET /api/compliance/property-verification`, GraphQL query `compliancePropertyVerification`

- [ ] **Step 1: Create report model records**

`PropertyVerificationReport.java`:
```java
package io.casehub.qhorus.compliance.model;

import io.casehub.qhorus.compliance.verification.PropertyViolation;
import java.time.Instant;
import java.util.List;

public record PropertyVerificationReport(
    Instant from, Instant to,
    List<PropertyResult> results,
    int totalProperties, int passed, int violated,
    int remediationsApplied,
    String merkleRoot, Instant generatedAt, int schemaVersion
) {}
```

`PropertyResult.java`:
```java
package io.casehub.qhorus.compliance.model;

import io.casehub.qhorus.compliance.verification.PropertyViolation;
import java.util.List;

public record PropertyResult(
    String propertyName, String ctlFormula,
    boolean passed, int violationCount,
    int remediationsApplied,
    List<PropertyViolation> violations
) {}
```

- [ ] **Step 2: Write failing test for PropertyVerificationService**

CDI-free. Mock `Instance<VerificationProperty>`.

- [ ] **Step 3: Implement PropertyVerificationService**

Iterates all discovered `VerificationProperty` beans, calls `check()`, optionally calls `remediate()` on `RemediatingProperty` instances, assembles `PropertyVerificationReport`.

- [ ] **Step 4: Add `PROPERTY_VERIFICATION` to `ReportType` enum**

Use `ide_edit_member` on `ReportType.java`:
```java
ATTRIBUTION, OBLIGATION, TRUST_HISTORY, VIOLATION, PROVENANCE,
JUDGMENT_ATTRIBUTION, JUDGMENT_FULFILLMENT, PROPERTY_VERIFICATION
```

- [ ] **Step 5: Add scheduler case, REST endpoint, GraphQL query, renderers**

Follow existing patterns from `JudgmentFulfillmentReportService` and `ComplianceReportScheduler.generateAndStore()`.

REST: `GET /api/compliance/property-verification?from=...&to=...`
GraphQL: `compliancePropertyVerification(from: String!, to: String!): PropertyVerificationReportType`
Scheduler: `case PROPERTY_VERIFICATION ->` in `generateAndStore()`.

- [ ] **Step 6: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl compliance-report`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/src
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#414): property verification report type + service + REST/GraphQL/scheduler

PropertyVerificationService orchestrates all VerificationProperty beans.
PROPERTY_VERIFICATION report type with scheduled generation, REST endpoint,
GraphQL query, and CSV/HTML renderers.

Refs #414"
```

---

### Task 6: Issue closures + full build verification + doc updates

**Files:**
- Modify: `docs/adr/0005-message-type-taxonomy-theoretical-foundation.md` — add judgment composition rationale
- Modify: `CLAUDE.md` — document judgment attestation and verification properties

**Interfaces:**
- Consumes: Everything from Tasks 1-5
- Produces: Closed issues #412, #414. Updated ADR-0005.

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 2: Update ADR-0005 — add judgment composition rationale**

Append to the Completeness Argument section:

```markdown
### Governed Yield Composition

Governed yields (judgment requests, evidence verification, trust feedback) compose
with existing types rather than adding new ones. A judgment request is a COMMAND
(directive); evidence delivery is DONE (declaration); verification tracking is EVENT
(perlocutionary). The dual-lifecycle separation — commitment tracks obligation
discharge, attestation tracks quality — is intentional: obligation fulfillment and
quality assessment are orthogonal dimensions.

See `docs/specs/issue-411-judgment-commitment-type/2026-08-28-governed-yield-governance-design.md`.
```

- [ ] **Step 3: Update CLAUDE.md with judgment attestation and verification docs**

Add entries for `JudgmentCommitmentAttestationPolicy`, `JudgmentVerificationObserver`, and the 5 verification properties.

- [ ] **Step 4: Close #412 on GitHub**

```bash
gh issue close 412 --repo casehubio/qhorus --comment "Covered by #401 (RoutingBridge). Judgment caller routing = COMMAND with target: \"role:reviewer\" → RoutingBridge resolves via trust scores. Per-channel routingTrustThreshold gates minimum quality. Engine-side integration tracked on engine#996."
```

- [ ] **Step 5: File eidos issue for per-capability trust**

```bash
gh issue create --repo casehubio/eidos --title "feat: per-capability trust differentiation in AgentSelector" --body "AgentSelector should support per-capability trust scores — an agent trusted for code_review may not be trusted for legal_review. Currently trust scores are per-actor only. Consumed by qhorus RoutingBridge for judgment caller selection. Refs casehubio/qhorus#412, casehubio/qhorus#410." --label enhancement
```

- [ ] **Step 6: Commit doc updates**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add docs/adr/ CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#411): ADR-0005 judgment composition rationale + CLAUDE.md updates

Documents why governed yields compose with existing types rather than
adding new ones. Updates CLAUDE.md with judgment attestation and
verification property documentation.

Refs #411, Closes #412"
```

---

## References

- [2026-08-28-governed-yield-governance-design.md](../specs/issue-411-judgment-commitment-type/2026-08-28-governed-yield-governance-design.md) — design spec this plan implements
- [decisions.md](../specs/issue-411-judgment-commitment-type/decisions.md) — D1-D9 design decisions
- `api/src/main/java/io/casehub/qhorus/api/spi/CommitmentAttestationPolicy.java` — SPI interface
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/StoredCommitmentAttestationPolicy.java:46` — parent class to extend
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java:284` — writeAttestation method
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java` — existing query methods
- `api/src/main/java/io/casehub/qhorus/api/judgment/JudgmentEventKinds.java` — telemetry contract constants
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportType.java` — existing 7 report types
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduler.java` — scheduler pattern
- `docs/adr/0005-message-type-taxonomy-theoretical-foundation.md` — stopping criterion, Searle matrix
- `docs/specs/issue-395-propose-message-type/2026-08-14-propose-message-type-design.md` — PROPOSE precedent
- `docs/specs/issue-401-reputation-routing/2026-08-25-reputation-routing-design.md` — RoutingBridge architecture
- `docs/specs/issue-413-judgment-compliance-evidence/2026-08-27-judgment-compliance-evidence-design.md` — telemetry contract
- PP-20260623-fd69f3 — COMMAND obligation verification type check protocol
- GitHub #410 — governed yield epic
- GitHub #411, #412, #414 — child issues
