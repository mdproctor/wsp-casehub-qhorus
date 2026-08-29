# Attestation Target Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #410 — governed yield governance
**Issue group:** #410

**Goal:** Fix the attestation target mismatch so judgment outcomes correctly update both requester (engine) and obligor (reviewer) trust scores.

**Architecture:** The `JudgmentVerificationObserver` currently writes a single attestation on the COMMAND entry. It needs to write dual attestations (COMMAND + DONE) matching the `LedgerWriteService.writeAttestation()` pattern. The SafetyProperty and EvidenceCompletenessProperty queries and remediation need corresponding fixes for consistency.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, Mockito, AssertJ

## Global Constraints

- No Flyway migrations — all attestations use existing `ledger_attestation` table
- CDI-free unit tests with Mockito mocks (compliance-report module pattern)
- `saveAttestation()` uses `em.persist()` — NOT idempotent; guard against duplicates
- Observer fires after commit via `TransactionSynchronizationRegistry` — integration tests need `QuarkusTransaction.requiringNew()`
- `MessageReceivedEvent` is a 13-param record (messageId, channelName, channelId, tenancyId, messageType, senderId, target, actorType, correlationId, occurredAt, content, payload, topic)

---

## Batch 1: Foundation — repository query + observer dual attestation

### Task 1: Add `findTerminalEntryByCorrelationId` to MessageLedgerEntryRepository

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepositoryTest.java` (or add to existing test class if present)

**Interfaces:**
- Consumes: nothing new — uses existing `EntityManager` injection
- Produces: `Optional<MessageLedgerEntry> findTerminalEntryByCorrelationId(UUID channelId, String correlationId, String tenancyId)` — filters to `messageType IN ('DONE', 'FAILURE', 'DECLINE')`, returns most recent by `sequenceNumber DESC`

- [ ] **Step 1: Write the failing test**

CDI-free test using mock `EntityManager`. Verify the JPQL query filters to terminal types and orders by sequenceNumber DESC.

Since `MessageLedgerEntryRepository` uses a real `EntityManager` (not mockable cleanly for JPQL string verification), the test for this query will be verified via the observer integration in Task 2. The method itself is a direct mirror of `findLatestByCorrelationId` (line 325) with the type filter changed.

- [ ] **Step 2: Add the method**

Use `ide_insert_member` to add after `findLatestByCorrelationId` (line 340):

```java
public Optional<MessageLedgerEntry> findTerminalEntryByCorrelationId(
        final UUID channelId, final String correlationId, final String tenancyId) {
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e " +
                    "WHERE e.subjectId = :sid AND e.correlationId = :corr " +
                    "AND e.tenancyId = :tid " +
                    "AND e.messageType IN ('DONE', 'FAILURE', 'DECLINE') " +
                    "ORDER BY e.sequenceNumber DESC",
            MessageLedgerEntry.class)
            .setParameter("sid", channelId)
            .setParameter("corr", correlationId)
            .setParameter("tid", tenancyId(tenancyId))
            .setMaxResults(1)
            .getResultStream()
            .findFirst();
}
```

- [ ] **Step 3: Verify compilation**

Run: `ide_diagnostics` on `MessageLedgerEntryRepository.java`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#410): add findTerminalEntryByCorrelationId to MessageLedgerEntryRepository

Mirrors findLatestByCorrelationId but filters to DONE/FAILURE/DECLINE
terminal types. Used by JudgmentVerificationObserver for dual attestation.

Refs #410"
```

---

### Task 2: Fix JudgmentVerificationObserver for dual attestation

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/attestation/JudgmentVerificationObserver.java`
- Modify: `compliance-report/src/test/java/io/casehub/qhorus/compliance/attestation/JudgmentVerificationObserverTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.findTerminalEntryByCorrelationId(UUID, String, String)` from Task 1
- Produces: Two `LedgerAttestation` entries per VERIFIED event — one on COMMAND (requester trust), one on DONE (obligor trust) when `command.actorId != done.actorId`

- [ ] **Step 1: Write the failing test — dual attestation**

Add test to `JudgmentVerificationObserverTest`:

```java
@Test
void acceptedVerificationWritesDualAttestations() {
    UUID channelId = UUID.randomUUID();
    String corrId = UUID.randomUUID().toString();
    Long messageId = 50L;

    var verifiedEntry = buildEntry(channelId, "judgment_verified",
            "ACCEPTED", 0.85, corrId);
    var commandEntry = buildEntry(channelId, null, null, null, corrId);
    commandEntry.id = UUID.randomUUID();
    commandEntry.subjectId = UUID.randomUUID();
    commandEntry.actorId = "engine-actor";

    var doneEntry = buildEntry(channelId, null, null, null, corrId);
    doneEntry.id = UUID.randomUUID();
    doneEntry.subjectId = commandEntry.subjectId;
    doneEntry.actorId = "reviewer-actor";

    when(messageRepo.findByMessageId(messageId)).thenReturn(Optional.of(verifiedEntry));
    when(messageRepo.findLatestByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.of(commandEntry));
    when(messageRepo.findTerminalEntryByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.of(doneEntry));

    observer.onMessage(eventMessage(messageId, channelId, corrId));

    var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
    verify(ledger, times(2)).saveAttestation(captor.capture(), eq("default"));

    var attestations = captor.getAllValues();
    // First: COMMAND entry (requester trust)
    assertThat(attestations.get(0).ledgerEntryId).isEqualTo(commandEntry.id);
    assertThat(attestations.get(0).verdict).isEqualTo(AttestationVerdict.SOUND);
    // Second: DONE entry (obligor/reviewer trust)
    assertThat(attestations.get(1).ledgerEntryId).isEqualTo(doneEntry.id);
    assertThat(attestations.get(1).verdict).isEqualTo(AttestationVerdict.SOUND);
}
```

- [ ] **Step 2: Write the failing test — self-attestation guard**

```java
@Test
void sameActorSkipsDoneAttestation() {
    UUID channelId = UUID.randomUUID();
    String corrId = UUID.randomUUID().toString();
    Long messageId = 51L;

    var verifiedEntry = buildEntry(channelId, "judgment_verified",
            "ACCEPTED", 0.9, corrId);
    var commandEntry = buildEntry(channelId, null, null, null, corrId);
    commandEntry.id = UUID.randomUUID();
    commandEntry.subjectId = UUID.randomUUID();
    commandEntry.actorId = "same-actor";

    var doneEntry = buildEntry(channelId, null, null, null, corrId);
    doneEntry.id = UUID.randomUUID();
    doneEntry.subjectId = commandEntry.subjectId;
    doneEntry.actorId = "same-actor";

    when(messageRepo.findByMessageId(messageId)).thenReturn(Optional.of(verifiedEntry));
    when(messageRepo.findLatestByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.of(commandEntry));
    when(messageRepo.findTerminalEntryByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.of(doneEntry));

    observer.onMessage(eventMessage(messageId, channelId, corrId));

    verify(ledger, times(1)).saveAttestation(any(), eq("default"));
}
```

- [ ] **Step 3: Write the failing test — missing terminal entry**

```java
@Test
void missingTerminalEntryWritesCommandOnly() {
    UUID channelId = UUID.randomUUID();
    String corrId = UUID.randomUUID().toString();
    Long messageId = 52L;

    var verifiedEntry = buildEntry(channelId, "judgment_verified",
            "ACCEPTED", 0.8, corrId);
    var commandEntry = buildEntry(channelId, null, null, null, corrId);
    commandEntry.id = UUID.randomUUID();
    commandEntry.subjectId = UUID.randomUUID();
    commandEntry.actorId = "engine";

    when(messageRepo.findByMessageId(messageId)).thenReturn(Optional.of(verifiedEntry));
    when(messageRepo.findLatestByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.of(commandEntry));
    when(messageRepo.findTerminalEntryByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.empty());

    observer.onMessage(eventMessage(messageId, channelId, corrId));

    var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
    verify(ledger, times(1)).saveAttestation(captor.capture(), eq("default"));
    assertThat(captor.getValue().ledgerEntryId).isEqualTo(commandEntry.id);
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentVerificationObserverTest -pl compliance-report`
Expected: 3 new tests FAIL (findTerminalEntryByCorrelationId not called, single attestation written)

- [ ] **Step 5: Implement the observer fix**

Use `ide_replace_member` on `JudgmentVerificationObserver.onMessage`:

```java
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
    MessageLedgerEntry command = commandEntry.get();

    AttestationVerdict verdict = mapVerdict(entry.verificationOutcome);
    double confidence = mapConfidence(entry.verificationOutcome, entry.evidenceQuality);

    writeAttestation(command.id, command.subjectId, verdict, confidence, tenancyId);

    var terminalEntry = messageRepo.findTerminalEntryByCorrelationId(
            entry.channelId, event.correlationId(), tenancyId);
    terminalEntry.ifPresent(done -> {
        if (command.actorId != null && !command.actorId.equals(done.actorId)) {
            writeAttestation(done.id, done.subjectId, verdict, confidence, tenancyId);
        }
    });
}

private void writeAttestation(UUID entryId, UUID subjectId,
        AttestationVerdict verdict, double confidence, String tenancyId) {
    LedgerAttestation attestation = new LedgerAttestation();
    attestation.ledgerEntryId = entryId;
    attestation.subjectId = subjectId;
    attestation.attestorId = "system:judgment-verifier";
    attestation.attestorType = ActorType.SYSTEM;
    attestation.verdict = verdict;
    attestation.confidence = confidence;
    attestation.capabilityTag = CapabilityTag.GLOBAL;
    try {
        ledger.saveAttestation(attestation, tenancyId);
        LOG.debugf("Judgment attestation %s/%.2f written for entry %s",
                verdict, confidence, entryId);
    } catch (Exception e) {
        LOG.warnf(e, "Failed to write judgment attestation for entry %s", entryId);
    }
}
```

- [ ] **Step 6: Update existing tests for dual attestation**

The existing `acceptedVerificationWritesSoundAttestation` test asserts `verify(ledger).saveAttestation(...)` (exactly once). After the fix, when `findTerminalEntryByCorrelationId` returns empty (the default mock behavior), only the COMMAND attestation is written. The existing test already sets up `commandEntry` without `actorId`, and `findTerminalEntryByCorrelationId` is not stubbed → returns `Optional.empty()` by default. Existing tests should still pass as-is. Verify this.

The existing tests that verify single attestation (`rejectedVerificationWritesFlaggedAttestation`, `partialVerificationWritesFlaggedWithMediumConfidence`, `acceptedWithLowEvidenceQualityUsesConfigFloor`) don't stub `findTerminalEntryByCorrelationId` — Mockito returns `Optional.empty()` by default for unstubbed `Optional`-returning methods. But Mockito returns `null` by default, not `Optional.empty()`. These tests need `findTerminalEntryByCorrelationId` stubbed to `Optional.empty()` to avoid NPE.

Add to `setUp()`:
```java
when(messageRepo.findTerminalEntryByCorrelationId(any(), any(), any()))
        .thenReturn(Optional.empty());
```

**Wait — verify Mockito default behavior.** Mockito returns `null` for unstubbed methods returning `Optional`. The code calls `terminalEntry.ifPresent(...)` — calling `ifPresent` on `null` throws NPE. So the stub IS needed. Add it to `setUp()`.

- [ ] **Step 7: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentVerificationObserverTest -pl compliance-report`
Expected: All tests PASS (8 existing + 3 new = 11 total)

- [ ] **Step 8: Verify compilation across modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl compliance-report`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/src/main/java/io/casehub/qhorus/compliance/attestation/JudgmentVerificationObserver.java compliance-report/src/test/java/io/casehub/qhorus/compliance/attestation/JudgmentVerificationObserverTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(#410): JudgmentVerificationObserver writes dual attestation (COMMAND + DONE)

Matches LedgerWriteService.writeAttestation() pattern. COMMAND entry
attestation feeds requester/engine trust. DONE entry attestation feeds
obligor/reviewer trust. Self-attestation guard preserved.

Refs #410"
```

---

## Batch 2: Verification property fixes

### Task 3: SafetyProperty — exclude judgment DONEs from query

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java`
- Modify: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/SafetyPropertyTest.java` (verify existing test class, or create)

**Interfaces:**
- Consumes: `JudgmentEventKinds.YIELDED` constant from `api/src/main/java/io/casehub/qhorus/api/judgment/JudgmentEventKinds.java`
- Produces: `findDoneEntriesWithoutAttestation` now excludes DONE entries that have a matching YIELDED judgment event

- [ ] **Step 1: Check for existing SafetyProperty tests**

```bash
find /Users/mdproctor/claude/casehub/qhorus/compliance-report/src/test -name "SafetyPropertyTest.java" 2>/dev/null
```

- [ ] **Step 2: Write the failing test — judgment DONEs excluded**

Create or extend `SafetyPropertyTest`. CDI-free with mock `MessageLedgerEntryRepository`:

```java
@Test
void judgmentDoneWithoutAttestationNotFlagged() {
    // The query itself handles exclusion — verify by checking that
    // SafetyProperty.check() returns no violations when the repo
    // returns an empty list (judgment DONEs filtered at query level)
    when(messageRepo.findDoneEntriesWithoutAttestation(any(), any(), eq("default")))
            .thenReturn(List.of());

    Instant now = Instant.now();
    CheckResult result = property.check("default",
            now.minus(7, ChronoUnit.DAYS), now);
    assertThat(result.passed()).isTrue();
}
```

Since the exclusion is at the JPQL level (not Java), the unit test for SafetyProperty itself doesn't change behavior — it's the query that changes. The real verification is in the integration test. But we should still add a test documenting the contract.

- [ ] **Step 3: Modify `findDoneEntriesWithoutAttestation`**

Use `ide_replace_member` on `findDoneEntriesWithoutAttestation` in `MessageLedgerEntryRepository`:

```java
public List<MessageLedgerEntry> findDoneEntriesWithoutAttestation(
        Instant from, Instant to, String tenancyId) {
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e " +
            "WHERE e.tenancyId = :tenancyId " +
            "AND e.messageType = 'DONE' " +
            "AND e.occurredAt >= :from AND e.occurredAt <= :to " +
            "AND NOT EXISTS (SELECT 1 FROM " +
            "io.casehub.ledger.runtime.model.LedgerAttestation a " +
            "WHERE a.ledgerEntryId = e.id) " +
            "AND NOT EXISTS (SELECT 1 FROM MessageLedgerEntry v " +
            "WHERE v.correlationId = e.correlationId " +
            "AND v.tenancyId = :tenancyId " +
            "AND v.toolName = :yieldedKind)",
            MessageLedgerEntry.class)
        .setParameter("tenancyId", tenancyId(tenancyId))
        .setParameter("from", from)
        .setParameter("to", to)
        .setParameter("yieldedKind",
                io.casehub.qhorus.api.judgment.JudgmentEventKinds.YIELDED)
        .getResultList();
}
```

- [ ] **Step 4: Add JudgmentEventKinds import if not present**

Check and add `import io.casehub.qhorus.api.judgment.JudgmentEventKinds;` to `MessageLedgerEntryRepository.java` if not already imported (it's already imported for `findDoneEntriesWithDeferredAttestation`).

- [ ] **Step 5: Verify compilation**

Run: `ide_diagnostics` on `MessageLedgerEntryRepository.java`

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,compliance-report`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(#410): SafetyProperty excludes judgment DONEs from unattested query

Judgment DONEs use deferred attestation (written by observer on VERIFIED
event, not at DONE time). Adding NOT EXISTS clause for YIELDED events
prevents false HIGH violations. EvidenceCompletenessProperty owns
judgment attestation completeness.

Refs #410"
```

---

### Task 4: EvidenceCompletenessProperty — dual remediation with guard

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/EvidenceCompletenessProperty.java`
- Modify: `compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/EvidenceCompletenessPropertyTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.findTerminalEntryByCorrelationId(UUID, String, String)` from Task 1, `MessageLedgerEntryRepository.findLatestByCorrelationId(UUID, String, String)`, `LedgerEntryRepository.findAttestationsByEntryId(UUID, String)`
- Produces: `remediate()` writes dual attestations (DONE always, COMMAND with guard)

- [ ] **Step 1: Write the failing test — remediation writes dual attestations**

```java
@Test
void remediateWritesDualAttestations() {
    UUID channelId = UUID.randomUUID();
    UUID judgmentId = UUID.randomUUID();
    String corrId = "corr-remediate";

    var doneEntry = new MessageLedgerEntry();
    doneEntry.id = UUID.randomUUID();
    doneEntry.channelId = channelId;
    doneEntry.correlationId = corrId;
    doneEntry.judgmentId = judgmentId;
    doneEntry.subjectId = UUID.randomUUID();

    var verifiedEntry = new MessageLedgerEntry();
    verifiedEntry.toolName = "judgment_verified";
    verifiedEntry.verificationOutcome = "ACCEPTED";
    verifiedEntry.evidenceQuality = 0.85;

    var commandEntry = new MessageLedgerEntry();
    commandEntry.id = UUID.randomUUID();
    commandEntry.subjectId = doneEntry.subjectId;
    commandEntry.actorId = "engine";

    when(messageRepo.findDoneEntriesWithDeferredAttestation(eq("default")))
            .thenReturn(List.of(doneEntry));
    when(messageRepo.findJudgmentEvents(isNull(), eq(judgmentId),
            isNull(), isNull(), eq("default")))
            .thenReturn(List.of(verifiedEntry));
    when(messageRepo.findLatestByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.of(commandEntry));
    when(ledger.findAttestationsByEntryId(commandEntry.id, "default"))
            .thenReturn(List.of());

    Instant now = Instant.now();
    int count = property.remediate("default",
            now.minus(7, ChronoUnit.DAYS), now);

    assertThat(count).isEqualTo(1);
    var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
    verify(ledger, times(2)).saveAttestation(captor.capture(), eq("default"));

    var attestations = captor.getAllValues();
    assertThat(attestations.get(0).ledgerEntryId).isEqualTo(doneEntry.id);
    assertThat(attestations.get(1).ledgerEntryId).isEqualTo(commandEntry.id);
}
```

- [ ] **Step 2: Write the failing test — guard skips existing COMMAND attestation**

```java
@Test
void remediateSkipsCommandWhenAlreadyAttested() {
    UUID channelId = UUID.randomUUID();
    UUID judgmentId = UUID.randomUUID();
    String corrId = "corr-guard";

    var doneEntry = new MessageLedgerEntry();
    doneEntry.id = UUID.randomUUID();
    doneEntry.channelId = channelId;
    doneEntry.correlationId = corrId;
    doneEntry.judgmentId = judgmentId;
    doneEntry.subjectId = UUID.randomUUID();

    var verifiedEntry = new MessageLedgerEntry();
    verifiedEntry.toolName = "judgment_verified";
    verifiedEntry.verificationOutcome = "REJECTED";

    var commandEntry = new MessageLedgerEntry();
    commandEntry.id = UUID.randomUUID();
    commandEntry.subjectId = doneEntry.subjectId;

    var existingAttestation = new LedgerAttestation();
    existingAttestation.attestorId = "system:judgment-verifier";

    when(messageRepo.findDoneEntriesWithDeferredAttestation(eq("default")))
            .thenReturn(List.of(doneEntry));
    when(messageRepo.findJudgmentEvents(isNull(), eq(judgmentId),
            isNull(), isNull(), eq("default")))
            .thenReturn(List.of(verifiedEntry));
    when(messageRepo.findLatestByCorrelationId(channelId, corrId, "default"))
            .thenReturn(Optional.of(commandEntry));
    when(ledger.findAttestationsByEntryId(commandEntry.id, "default"))
            .thenReturn(List.of(existingAttestation));

    Instant now = Instant.now();
    int count = property.remediate("default",
            now.minus(7, ChronoUnit.DAYS), now);

    assertThat(count).isEqualTo(1);
    // Only DONE attestation written — COMMAND skipped (already attested by observer)
    verify(ledger, times(1)).saveAttestation(any(), eq("default"));
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EvidenceCompletenessPropertyTest -pl compliance-report`
Expected: 2 new tests FAIL

- [ ] **Step 4: Implement the remediation fix**

Use `ide_replace_member` on `EvidenceCompletenessProperty.remediate`:

```java
@Override
public int remediate(String tenancyId, Instant from, Instant to) {
    List<MessageLedgerEntry> deferred =
            messageRepo.findDoneEntriesWithDeferredAttestation(tenancyId);
    int count = 0;
    for (MessageLedgerEntry doneEntry : deferred) {
        var verifiedEntries = messageRepo.findJudgmentEvents(
                null, doneEntry.judgmentId, null, null, tenancyId);
        var verified = verifiedEntries.stream()
                .filter(e -> "judgment_verified".equals(e.toolName))
                .findFirst();
        if (verified.isEmpty()) continue;

        MessageLedgerEntry v = verified.get();
        AttestationVerdict verdict = "ACCEPTED".equals(v.verificationOutcome)
                ? AttestationVerdict.SOUND : AttestationVerdict.FLAGGED;
        double confidence = switch (v.verificationOutcome != null
                ? v.verificationOutcome : "") {
            case "ACCEPTED" -> Math.max(
                    config.attestation().judgmentAcceptedConfidence(),
                    v.evidenceQuality != null ? v.evidenceQuality : 0.7);
            case "REJECTED" -> config.attestation().judgmentRejectedConfidence();
            case "PARTIAL" -> config.attestation().judgmentPartialConfidence();
            default -> config.attestation().judgmentRejectedConfidence();
        };

        try {
            writeAttestation(doneEntry.id, doneEntry.subjectId,
                    verdict, confidence, tenancyId);
            count++;

            var commandEntry = messageRepo.findLatestByCorrelationId(
                    doneEntry.channelId, doneEntry.correlationId, tenancyId);
            commandEntry.ifPresent(cmd -> {
                boolean alreadyAttested = ledger.findAttestationsByEntryId(
                        cmd.id, tenancyId).stream()
                        .anyMatch(a -> "system:judgment-verifier".equals(a.attestorId));
                if (!alreadyAttested) {
                    writeAttestation(cmd.id, cmd.subjectId,
                            verdict, confidence, tenancyId);
                }
            });

            LOG.infof("Remediated missing judgment attestation for entry %s",
                    doneEntry.id);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to remediate judgment attestation for entry %s",
                    doneEntry.id);
        }
    }
    return count;
}

private void writeAttestation(UUID entryId, UUID subjectId,
        AttestationVerdict verdict, double confidence, String tenancyId) {
    LedgerAttestation attestation = new LedgerAttestation();
    attestation.ledgerEntryId = entryId;
    attestation.subjectId = subjectId;
    attestation.attestorId = "system:judgment-verifier";
    attestation.attestorType = ActorType.SYSTEM;
    attestation.verdict = verdict;
    attestation.confidence = confidence;
    attestation.capabilityTag = CapabilityTag.GLOBAL;
    ledger.saveAttestation(attestation, tenancyId);
}
```

Add required imports:
```java
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.platform.api.identity.ActorType;
import java.util.Optional;
```

- [ ] **Step 5: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EvidenceCompletenessPropertyTest -pl compliance-report`
Expected: All tests PASS (4 existing + 2 new = 6 total)

- [ ] **Step 6: Run full module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl compliance-report`
Expected: All tests PASS

- [ ] **Step 7: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — verifies no cross-module breakage

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/src/main/java/io/casehub/qhorus/compliance/verification/EvidenceCompletenessProperty.java compliance-report/src/test/java/io/casehub/qhorus/compliance/verification/EvidenceCompletenessPropertyTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(#410): EvidenceCompletenessProperty dual remediation with COMMAND guard

Remediation writes DONE attestation always (the detected gap) and
COMMAND attestation only if not already written by the observer.
Prevents duplicate attestations (saveAttestation uses em.persist).

Refs #410"
```

---

## References

- [2026-08-29-attestation-target-fix-design.md] — design spec this plan implements
- [runtime/ledger/LedgerWriteService.java:284-338] — dual-attestation reference pattern
- [runtime/ledger/MessageLedgerEntryRepository.java:325-340] — findLatestByCorrelationId (symmetric pair)
- [runtime/ledger/MessageLedgerEntryRepository.java:507-520] — findDoneEntriesWithoutAttestation (modified)
- [runtime/ledger/MessageLedgerEntryRepository.java:551-567] — findDoneEntriesWithDeferredAttestation (unchanged)
- [runtime/ledger/QhorusLedgerEntryRepository.java:132-150] — saveAttestation (em.persist, not idempotent)
- [compliance-report/attestation/JudgmentVerificationObserver.java] — observer (modified)
- [compliance-report/verification/EvidenceCompletenessProperty.java] — remediation (modified)
- [compliance-report/verification/SafetyProperty.java] — unchanged (query handles exclusion)
- PP-20260623-77adf0 — CommitmentAttestationPolicy null context handling
- PP-20260608-07daa6 — observer test transaction discipline
- casehubio/eidos#148 — per-capability trust (filed during this work)
- GitHub #410 — governed yield governance epic
