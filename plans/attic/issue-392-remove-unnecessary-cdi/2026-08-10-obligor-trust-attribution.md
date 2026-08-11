# Obligor Trust Attribution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #373 — Trust model: obligor-targeted trust attribution for attestation outcomes
**Issue group:** #391, #373

**Goal:** Write a second attestation on the terminal entry (DONE/FAILURE/DECLINE/RESPONSE) so the obligor's trust score is affected by attestation outcomes.

**Architecture:** One file change in `LedgerWriteService`. After the existing attestation on the COMMAND entry (feeds requester trust), write a second attestation on the terminal entry (feeds obligor trust). Self-attestation guard prevents duplicates when requester == obligor.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger LedgerAttestation model

## Global Constraints

- Zero changes to casehub-ledger — model, queries, and trust computation pipeline stay unchanged
- Existing COMMAND attestations unchanged — requester trust attribution preserved
- `LedgerWriteService` runs in `REQUIRES_NEW` — no CommitmentStore queries

---

### Task 1: Dual attestation with self-attestation guard

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/ledger/LedgerWriteServiceTest.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.saveAttestation(LedgerAttestation, String)` (existing)
- Produces: Second `LedgerAttestation` per terminal message, linked to terminal entry ID

- [ ] **Step 1: Write failing test — dual attestation on DONE**

Add to `LedgerWriteServiceTest.java`. This tests that a DONE message produces TWO attestations — one on the COMMAND entry (existing) and one on the terminal DONE entry (new).

```java
@Test
void record_done_writesDualAttestation_commandAndTerminalEntries() {
    UUID          channelId = UUID.randomUUID();
    ChannelEntity ch        = channel(channelId);

    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.messageId = 80L;
    cmdEntry.subjectId = channelId;
    cmdEntry.channelId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.actorId = "requester-a";
    cmdEntry.correlationId = "corr-dual-attest";
    cmdEntry.sequenceNumber = 1;
    sharedEntries.add(cmdEntry);

    recordWithReplyTo("DONE", "Report delivered", "obligor-b",
            "corr-dual-attest", null, ch, cmdEntry.messageId);

    assertEquals(2, ledgerStub.savedAttestations.size(),
            "Expected 2 attestations: one on COMMAND entry, one on terminal DONE entry");

    LedgerAttestation cmdAttestation = ledgerStub.savedAttestations.get(0);
    assertEquals(cmdEntry.id, cmdAttestation.ledgerEntryId,
            "First attestation should target the COMMAND entry");
    assertEquals(AttestationVerdict.SOUND, cmdAttestation.verdict);

    LedgerAttestation terminalAttestation = ledgerStub.savedAttestations.get(1);
    assertNotEquals(cmdEntry.id, terminalAttestation.ledgerEntryId,
            "Second attestation should target the terminal entry, not the COMMAND");
    assertEquals(AttestationVerdict.SOUND, terminalAttestation.verdict);
    assertEquals(0.7, terminalAttestation.confidence, 1e-9);
    assertEquals(channelId, terminalAttestation.subjectId);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -pl runtime -Dtest=LedgerWriteServiceTest#record_done_writesDualAttestation_commandAndTerminalEntries -Dno-format`
Expected: FAIL — `assertEquals(2, ledgerStub.savedAttestations.size())` fails with actual = 1

- [ ] **Step 3: Write failing test — self-attestation guard**

Add to `LedgerWriteServiceTest.java`. When the COMMAND sender and terminal sender are the same actor, only one attestation should be written (no duplicate).

```java
@Test
void record_done_sameSenderAsCommand_writesOnlyOneAttestation() {
    UUID          channelId = UUID.randomUUID();
    ChannelEntity ch        = channel(channelId);

    MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
    cmdEntry.id = UUID.randomUUID();
    cmdEntry.messageId = 81L;
    cmdEntry.subjectId = channelId;
    cmdEntry.channelId = channelId;
    cmdEntry.messageType = "COMMAND";
    cmdEntry.actorId = "same-agent";
    cmdEntry.correlationId = "corr-self-attest";
    cmdEntry.sequenceNumber = 1;
    sharedEntries.add(cmdEntry);

    recordWithReplyTo("DONE", "Self-completed", "same-agent",
            "corr-self-attest", null, ch, cmdEntry.messageId);

    assertEquals(1, ledgerStub.savedAttestations.size(),
            "Self-attestation guard: same actor on COMMAND and DONE → only COMMAND attestation");
    assertEquals(cmdEntry.id, ledgerStub.savedAttestations.get(0).ledgerEntryId);
}
```

- [ ] **Step 4: Run both new tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -pl runtime -Dtest=LedgerWriteServiceTest#record_done_writesDualAttestation_commandAndTerminalEntries+record_done_sameSenderAsCommand_writesOnlyOneAttestation -Dno-format`
Expected: First test FAILS (only 1 attestation). Second test PASSES (already produces 1 attestation — this is the existing behavior we want to preserve).

- [ ] **Step 5: Implement dual attestation in LedgerWriteService**

Modify `LedgerWriteService.java`:

1. Change `record()` to pass `entry.id` (the terminal entry's ID) to `writeAttestation()`.
2. Change `writeAttestation()` signature to accept `terminalEntryId`.
3. After the existing attestation write on `priorMsg.id`, write a second attestation on `terminalEntryId` if `resolvedActorId` differs from `priorMsg.actorId`.

In `record()`, change the `writeAttestation` call (around line 241):

```java
if (hasAttestation) {
    writeAttestation(resolvedSubjectId, resolvedCausedByEntryId, entry.id,
            dispatch.type(), resolvedActorId, tenancyId, commitmentId,
            dispatch.content(), dispatch.artefactRefs());
}
```

Change `writeAttestation` signature and body:

```java
private void writeAttestation(final UUID subjectId, final UUID causedByEntryId,
                              final UUID terminalEntryId,
                              final MessageType terminalType, final String resolvedActorId, final String tenancyId,
                              final UUID commitmentId, final String terminalContent,
                              final java.util.List<io.casehub.qhorus.api.message.ArtefactRef> terminalArtefactRefs) {
    try {
        ledger.findEntryById(causedByEntryId, tenancyId).ifPresent(prior -> {
            if (!(prior instanceof MessageLedgerEntry priorMsg)) {return;}
            if (!"COMMAND".equals(priorMsg.messageType)
                && !"HANDOFF".equals(priorMsg.messageType)) {return;}
            final String capabilityTag = extractCapabilityTag(priorMsg.content);
            final UUID   artefactUuid  = extractFirstArtefactUuid(terminalArtefactRefs);
            final String expectedToken = extractVerificationToken(priorMsg.content);
            final CommitmentContext ctx = new CommitmentContext(
                    priorMsg.correlationId, priorMsg.channelId, null, commitmentId, capabilityTag,
                    artefactUuid, expectedToken, terminalContent);
            attestationPolicy.attestationFor(terminalType, resolvedActorId, ctx)
                             .ifPresent(outcome -> {
                                 // Attestation on COMMAND entry — feeds requester trust
                                 final LedgerAttestation attestation = new LedgerAttestation();
                                 attestation.ledgerEntryId = priorMsg.id;
                                 attestation.subjectId     = subjectId;
                                 attestation.attestorId    = outcome.attestorId();
                                 attestation.attestorType  = outcome.attestorType();
                                 attestation.verdict       = outcome.verdict();
                                 attestation.confidence    = outcome.confidence();
                                 attestation.capabilityTag = ctx.capabilityTag();
                                 ledger.saveAttestation(attestation, tenancyId);
                                 LOG.debugf("LedgerAttestation %s written for COMMAND entry %s"
                                            + " (correlationId='%s', capability='%s')",
                                            attestation.verdict, priorMsg.id,
                                            priorMsg.correlationId, attestation.capabilityTag);

                                 // Attestation on terminal entry — feeds obligor trust
                                 if (!resolvedActorId.equals(priorMsg.actorId)) {
                                     final LedgerAttestation obligorAttestation = new LedgerAttestation();
                                     obligorAttestation.ledgerEntryId = terminalEntryId;
                                     obligorAttestation.subjectId     = subjectId;
                                     obligorAttestation.attestorId    = outcome.attestorId();
                                     obligorAttestation.attestorType  = outcome.attestorType();
                                     obligorAttestation.verdict       = outcome.verdict();
                                     obligorAttestation.confidence    = outcome.confidence();
                                     obligorAttestation.capabilityTag = ctx.capabilityTag();
                                     ledger.saveAttestation(obligorAttestation, tenancyId);
                                     LOG.debugf("LedgerAttestation %s written for terminal entry %s"
                                                + " (obligor='%s', capability='%s')",
                                                obligorAttestation.verdict, terminalEntryId,
                                                resolvedActorId, obligorAttestation.capabilityTag);
                                 }
                             });
        });
    } catch (final Exception e) {
        LOG.warnf(e, "Could not write attestation for entry %s"
                     + " — trust signal lost but ledger entry is safe",
                  causedByEntryId);
    }
}
```

- [ ] **Step 6: Run new tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -pl runtime -Dtest=LedgerWriteServiceTest#record_done_writesDualAttestation_commandAndTerminalEntries+record_done_sameSenderAsCommand_writesOnlyOneAttestation -Dno-format`
Expected: BOTH PASS

- [ ] **Step 7: Update existing attestation tests for new count**

Several existing tests assert `assertEquals(1, ledgerStub.savedAttestations.size())`. These now produce 2 attestations when requester ≠ obligor. Update:

In `record_done_withMatchingCommandEntry_writesSoundAttestation`:
- Change `assertEquals(1, ledgerStub.savedAttestations.size())` → `assertEquals(2, ledgerStub.savedAttestations.size())`
- Existing assertion `assertEquals(cmdEntry.id, a.ledgerEntryId)` stays (verifies COMMAND attestation)

In `record_failure_withMatchingCommandEntry_writesFlaggedAttestation`:
- Change `assertEquals(1, ledgerStub.savedAttestations.size())` → `assertEquals(2, ledgerStub.savedAttestations.size())`

In `record_decline_withMatchingCommandEntry_writesFlaggedAttestation`:
- Change `assertEquals(1, ledgerStub.savedAttestations.size())` → `assertEquals(2, ledgerStub.savedAttestations.size())`

In `record_response_withMatchingCommandEntry_writesFlaggedAttestation`:
- Change `assertEquals(1, ledgerStub.savedAttestations.size())` → `assertEquals(2, ledgerStub.savedAttestations.size())`

Tests that assert `assertTrue(ledgerStub.savedAttestations.isEmpty())` are unaffected (no attestation produced at all).

- [ ] **Step 8: Run full LedgerWriteServiceTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -pl runtime -Dtest=LedgerWriteServiceTest -Dno-format`
Expected: ALL PASS

- [ ] **Step 9: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -pl runtime -Dno-format`
Expected: ALL PASS (no regressions from other tests asserting attestation counts)

- [ ] **Step 10: Update CLAUDE.md**

Add to the `LedgerWriteService.record()` testing conventions bullet:

> `LedgerWriteService.writeAttestation()` writes TWO attestations per terminal message when requester ≠ obligor: one on the COMMAND entry (requester trust) and one on the terminal entry (obligor trust). Self-attestation guard: when `resolvedActorId.equals(priorMsg.actorId)`, only the COMMAND attestation is written. Refs #373.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java runtime/src/test/java/io/casehub/qhorus/ledger/LedgerWriteServiceTest.java CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#373): obligor trust attribution via dual attestation

Write a second attestation on the terminal entry (DONE/FAILURE/DECLINE/RESPONSE)
so the obligor's trust score is affected by attestation outcomes. Self-attestation
guard skips the second write when requester == obligor.

Closes #373

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```
