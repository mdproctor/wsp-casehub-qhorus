# Judgment Compliance Evidence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #413 — feat: judgment compliance evidence for E5 audit reports
**Issue group:** #413

**Goal:** Extend the E5 compliance export module with two judgment-specific report types (JUDGMENT_ATTRIBUTION, JUDGMENT_FULFILLMENT) that consume judgment provenance EVENTs from the engine.

**Architecture:** Four dedicated nullable columns on `MessageLedgerEntry` (V2004 migration) store judgment-specific telemetry extracted from EVENTs. Compile-time contract constants (`JudgmentEventKinds`) in `casehub-qhorus-api` define the telemetry field names. Two new report services in `compliance-report/` query judgment data using SQL aggregation and compose with existing `CausalGraphService` and trust score infrastructure. Full API exposure via REST, GraphQL, and MCP.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Flyway, SmallRye GraphQL

## Global Constraints

- Migration version: V2004 (in `runtime/src/main/resources/db/qhorus/migration/`)
- JPQL: `FROM MessageLedgerEntry` (not `LedgerEntry`) per `ledger-entry-repository-cross-dtype-jpql` protocol
- Query methods use `JudgmentEventKinds` constants (IN clause), not LIKE
- `domainContentBytes()` uses tagged suffix `|J:` for backward compatibility — non-judgment entries hash unchanged
- `evidence_quality` range: 0.0-1.0 with CHECK constraint in DDL and validation in extraction
- All CDI-free tests construct `MessageLedgerEntry` via `MessageLedgerEntryTestFactory`
- IntelliJ MCP required for all code navigation and editing

---

## Batch 1: Foundation — MessageLedgerEntry ready for judgment data

### Task 1: Contract constants, migration, entity fields, telemetry extraction, test factory

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/judgment/JudgmentEventKinds.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V2004__judgment_compliance_columns.sql`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java` — add 4 fields + update domainContentBytes()
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java:385-429` — extend populateTelemetry()
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/MessageLedgerEntryTestFactory.java` — add judgment helper
- Test: `runtime/src/test/java/io/casehub/qhorus/ledger/JudgmentTelemetryTest.java`

**Interfaces:**
- Produces: `JudgmentEventKinds.YIELDED`, `.RESPONDED`, `.VERIFIED`, `.ESCALATED`, `.TOOL_NAME_PREFIX` — used by query methods in Task 2 and report services in Tasks 3-4
- Produces: `MessageLedgerEntry.judgmentId` (UUID), `.judgmentType` (String), `.verificationOutcome` (String), `.evidenceQuality` (Double) — used by all downstream tasks
- Produces: `MessageLedgerEntryTestFactory.judgmentEvent(...)` — used by test tasks

- [ ] **Step 1: Create JudgmentEventKinds constants**

Create `api/src/main/java/io/casehub/qhorus/api/judgment/JudgmentEventKinds.java`:

```java
package io.casehub.qhorus.api.judgment;

import java.util.List;

public final class JudgmentEventKinds {
    public static final String YIELDED = "judgment_yielded";
    public static final String RESPONDED = "judgment_responded";
    public static final String VERIFIED = "judgment_verified";
    public static final String ESCALATED = "judgment_escalated";
    public static final String TOOL_NAME_PREFIX = "judgment_";

    public static final List<String> ALL = List.of(YIELDED, RESPONDED, VERIFIED, ESCALATED);
    public static final List<String> TERMINAL = List.of(VERIFIED, ESCALATED);

    private JudgmentEventKinds() {}
}
```

- [ ] **Step 2: Create V2004 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V2004__judgment_compliance_columns.sql`:

```sql
ALTER TABLE message_ledger_entry ADD COLUMN judgment_id UUID;
ALTER TABLE message_ledger_entry ADD COLUMN judgment_type VARCHAR(100);
ALTER TABLE message_ledger_entry ADD COLUMN verification_outcome VARCHAR(20);
ALTER TABLE message_ledger_entry ADD COLUMN evidence_quality DOUBLE PRECISION;

ALTER TABLE message_ledger_entry ADD CONSTRAINT chk_evidence_quality
    CHECK (evidence_quality IS NULL OR (evidence_quality >= 0 AND evidence_quality <= 1));

CREATE INDEX idx_mle_tenancy_toolname ON message_ledger_entry(tenancy_id, tool_name);
```

- [ ] **Step 3: Add judgment fields to MessageLedgerEntry**

Use `ide_insert_member` to add four fields after `routingCandidateCount` (line 96) in `MessageLedgerEntry.java`:

```java
@Column(name = "judgment_id")
public UUID judgmentId;

@Column(name = "judgment_type", length = 100)
public String judgmentType;

@Column(name = "verification_outcome", length = 20)
public String verificationOutcome;

@Column(name = "evidence_quality")
public Double evidenceQuality;
```

- [ ] **Step 4: Update domainContentBytes() with tagged suffix**

Use `ide_replace_member` to replace the `domainContentBytes()` method in `MessageLedgerEntry.java`. The existing 14-field `String.join("|", ...)` remains unchanged. Append a tagged judgment suffix ONLY when any judgment field is non-null:

```java
@Override
protected byte[] domainContentBytes() {
    String canonical = String.join("|",
        channelId     != null ? channelId.toString()     : "",
        messageId     != null ? messageId.toString()     : "",
        messageType   != null ? messageType              : "",
        target        != null ? target                   : "",
        content       != null ? content                  : "",
        correlationId != null ? correlationId            : "",
        commitmentId  != null ? commitmentId.toString()  : "",
        topic         != null ? topic                    : "",
        toolName      != null ? toolName                 : "",
        durationMs    != null ? durationMs.toString()    : "",
        tokenCount    != null ? tokenCount.toString()    : "",
        contextRefs      != null ? contextRefs                 : "",
        sourceEntity     != null ? sourceEntity                : "",
        contextWindowPct != null ? contextWindowPct.toString() : ""
    );
    if (judgmentId != null || judgmentType != null
            || verificationOutcome != null || evidenceQuality != null) {
        canonical += "|J:"
            + (judgmentId != null ? judgmentId.toString() : "") + "|"
            + (judgmentType != null ? judgmentType : "") + "|"
            + (verificationOutcome != null ? verificationOutcome : "") + "|"
            + (evidenceQuality != null ? String.valueOf(evidenceQuality) : "");
    }
    return canonical.getBytes(StandardCharsets.UTF_8);
}
```

- [ ] **Step 5: Extend populateTelemetry() in LedgerWriteService**

Use `ide_replace_member` to replace the `populateTelemetry` method in `LedgerWriteService.java`. Add judgment field extraction after the existing `contextWindowPct` block (before the closing catch), plus dual-storage fallback:

```java
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
        final JsonNode cwp = root.get("context_window_pct");
        if (cwp != null && cwp.isNumber()) {
            entry.contextWindowPct = cwp.asInt();
        }

        // judgment fields
        final JsonNode ji = root.get("judgment_id");
        if (ji != null && ji.isTextual()) {
            try {
                entry.judgmentId = UUID.fromString(ji.asText());
            } catch (IllegalArgumentException ignored) {
                LOG.warnf("Invalid judgment_id UUID for message %d", entry.messageId);
            }
        }
        final JsonNode jt = root.get("judgment_type");
        if (jt != null && jt.isTextual()) {
            entry.judgmentType = jt.asText();
        }
        final JsonNode vo = root.get("verification_outcome");
        if (vo != null && vo.isTextual()) {
            entry.verificationOutcome = vo.asText();
        }
        final JsonNode eq = root.get("evidence_quality");
        if (eq != null && eq.isNumber()) {
            double val = eq.asDouble();
            entry.evidenceQuality = (val >= 0 && val <= 1) ? val : null;
        }

        } catch (final Exception e) {
        LOG.warnf("Could not parse EVENT content as JSON for message %d — telemetry fields will be null",
                entry.messageId);
    }
}
```

- [ ] **Step 6: Add judgment helper to MessageLedgerEntryTestFactory**

Use `ide_insert_member` to add a static method after the existing `entry(UUID, Long, ...)` method:

```java
public static MessageLedgerEntry judgmentEvent(String toolName, UUID judgmentId,
        String judgmentType, UUID channelId, String correlationId) {
    MessageLedgerEntry e = entry(channelId, null, "EVENT", channelId, correlationId);
    e.toolName = toolName;
    e.judgmentId = judgmentId;
    e.judgmentType = judgmentType;
    e.entryType = LedgerEntryType.EVENT;
    return e;
}
```

- [ ] **Step 7: Write failing tests for telemetry extraction and domainContentBytes**

Create `runtime/src/test/java/io/casehub/qhorus/ledger/JudgmentTelemetryTest.java`:

```java
package io.casehub.qhorus.ledger;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.qhorus.api.judgment.JudgmentEventKinds;
import io.casehub.qhorus.runtime.ledger.LedgerWriteService;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.nio.charset.StandardCharsets;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class JudgmentTelemetryTest {

    private LedgerWriteService service;

    @BeforeEach
    void setUp() {
        service = new LedgerWriteService();
        service.objectMapper = new ObjectMapper();
    }

    @Test
    void extractsJudgmentFieldsFromTelemetry() throws Exception {
        var entry = new MessageLedgerEntry();
        entry.messageId = 1L;
        var judgmentId = UUID.randomUUID();
        String telemetry = """
            {"tool_name": "judgment_verified", "judgment_id": "%s",
             "judgment_type": "code_review", "verification_outcome": "ACCEPTED",
             "evidence_quality": 0.85}
            """.formatted(judgmentId);

        invokePopulateTelemetry(entry, telemetry);

        assertThat(entry.toolName).isEqualTo(JudgmentEventKinds.VERIFIED);
        assertThat(entry.judgmentId).isEqualTo(judgmentId);
        assertThat(entry.judgmentType).isEqualTo("code_review");
        assertThat(entry.verificationOutcome).isEqualTo("ACCEPTED");
        assertThat(entry.evidenceQuality).isEqualTo(0.85);
    }

    @Test
    void rejectsEvidenceQualityOutOfRange() throws Exception {
        var entry = new MessageLedgerEntry();
        entry.messageId = 2L;
        String telemetry = """
            {"tool_name": "judgment_responded", "evidence_quality": 1.5}
            """;

        invokePopulateTelemetry(entry, telemetry);

        assertThat(entry.evidenceQuality).isNull();
    }

    @Test
    void dualStorageFallbackPopulatesExistingColumns() throws Exception {
        var entry = new MessageLedgerEntry();
        entry.messageId = 3L;
        var judgmentId = UUID.randomUUID();
        String telemetry = """
            {"tool_name": "judgment_yielded", "judgment_id": "%s",
             "judgment_type": "code_review"}
            """.formatted(judgmentId);

        invokePopulateTelemetry(entry, telemetry);

        assertThat(entry.sourceEntity).isEqualTo("code_review");
        assertThat(entry.contextRefs).isEqualTo(judgmentId.toString());
    }

    @Test
    void domainContentBytesUnchangedForNonJudgmentEntries() {
        var entry = new MessageLedgerEntry();
        entry.channelId = UUID.randomUUID();
        entry.messageId = 1L;
        entry.messageType = "COMMAND";

        byte[] before = entry.domainContentBytes();

        entry.judgmentId = null;
        entry.judgmentType = null;
        entry.verificationOutcome = null;
        entry.evidenceQuality = null;

        byte[] after = entry.domainContentBytes();

        assertThat(after).isEqualTo(before);
    }

    @Test
    void domainContentBytesIncludesJudgmentSuffix() {
        var entry = new MessageLedgerEntry();
        entry.channelId = UUID.randomUUID();
        entry.messageId = 1L;
        entry.messageType = "EVENT";
        entry.judgmentId = UUID.randomUUID();
        entry.judgmentType = "code_review";
        entry.verificationOutcome = "ACCEPTED";
        entry.evidenceQuality = 0.85;

        String content = new String(entry.domainContentBytes(), StandardCharsets.UTF_8);

        assertThat(content).contains("|J:");
        assertThat(content).contains(entry.judgmentId.toString());
        assertThat(content).contains("code_review");
        assertThat(content).contains("ACCEPTED");
        assertThat(content).contains("0.85");
    }

    @Test
    void domainContentBytesNoCollisionWithPartialFields() {
        var entry1 = new MessageLedgerEntry();
        entry1.channelId = UUID.randomUUID();
        entry1.messageId = 1L;
        entry1.judgmentType = "code_review";
        entry1.verificationOutcome = "ACCEPTED";

        var entry2 = new MessageLedgerEntry();
        entry2.channelId = entry1.channelId;
        entry2.messageId = 1L;
        entry2.judgmentType = "code_reviewACCEPTED";

        assertThat(entry1.domainContentBytes()).isNotEqualTo(entry2.domainContentBytes());
    }

    private void invokePopulateTelemetry(MessageLedgerEntry entry, String telemetry) throws Exception {
        var method = LedgerWriteService.class.getDeclaredMethod(
                "populateTelemetry", MessageLedgerEntry.class, String.class);
        method.setAccessible(true);
        method.invoke(service, entry, telemetry);
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentTelemetryTest -pl runtime`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/judgment/
git add runtime/src/main/resources/db/qhorus/migration/V2004__judgment_compliance_columns.sql
git add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java
git add testing/src/main/java/io/casehub/qhorus/testing/MessageLedgerEntryTestFactory.java
git add runtime/src/test/java/io/casehub/qhorus/ledger/JudgmentTelemetryTest.java
git commit -m "feat(#413): judgment telemetry foundation — constants, migration, entity fields, extraction

JudgmentEventKinds constants in casehub-qhorus-api. V2004 migration adds
judgment_id, judgment_type, verification_outcome, evidence_quality columns
with CHECK constraint and (tenancy_id, tool_name) index. domainContentBytes()
uses tagged |J: suffix for Merkle backward compatibility. populateTelemetry()
extracts judgment fields with dual-storage fallback.

Refs #413"
```

---

## Batch 2: Report Services — judgment reports functional

### Task 2: MessageLedgerEntryRepository judgment query methods

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java` — add 3 query methods
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/JudgmentQueryTest.java`

**Interfaces:**
- Consumes: `JudgmentEventKinds.ALL`, `.TERMINAL`, `.YIELDED`, `.VERIFIED` from Task 1
- Consumes: `MessageLedgerEntry.judgmentId`, `.judgmentType`, `.verificationOutcome`, `.evidenceQuality` from Task 1
- Produces: `findJudgmentEvents(UUID channelId, UUID judgmentId, Instant from, Instant to, String tenancyId)` → `List<MessageLedgerEntry>` — used by JudgmentAttributionReportService (Task 3)
- Produces: `countJudgmentOutcomes(Instant from, Instant to, String tenancyId)` → `List<Object[]>` [judgmentType, toolName, verificationOutcome, count] — used by JudgmentFulfillmentReportService (Task 4). Includes both VERIFIED and ESCALATED terminal kinds so the service can populate escalated counts.
- Produces: `findPendingJudgments(String tenancyId)` → `List<MessageLedgerEntry>` — used by JudgmentFulfillmentReportService (Task 4)

- [ ] **Step 1: Write failing tests for query methods**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/JudgmentQueryTest.java`:

```java
package io.casehub.qhorus.runtime.ledger;

import io.casehub.qhorus.api.judgment.JudgmentEventKinds;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestTransaction
class JudgmentQueryTest {

    @Inject MessageLedgerEntryRepository repo;

    @Test
    void findJudgmentEventsFiltersByJudgmentId() {
        var channelId = UUID.randomUUID();
        var judgmentId = UUID.randomUUID();
        var otherJudgmentId = UUID.randomUUID();

        persistJudgmentEvent(channelId, judgmentId, JudgmentEventKinds.YIELDED, null, null);
        persistJudgmentEvent(channelId, judgmentId, JudgmentEventKinds.VERIFIED, "ACCEPTED", null);
        persistJudgmentEvent(channelId, otherJudgmentId, JudgmentEventKinds.YIELDED, null, null);

        var results = repo.findJudgmentEvents(null, judgmentId, null, null, null);

        assertThat(results).hasSize(2);
        assertThat(results).allMatch(e -> e.judgmentId.equals(judgmentId));
    }

    @Test
    void findJudgmentEventsFiltersByTimeRange() {
        var channelId = UUID.randomUUID();
        var judgmentId = UUID.randomUUID();
        var now = Instant.now();

        var e1 = persistJudgmentEvent(channelId, judgmentId, JudgmentEventKinds.YIELDED, null, null);
        e1.occurredAt = now.minus(2, ChronoUnit.HOURS);

        var e2 = persistJudgmentEvent(channelId, judgmentId, JudgmentEventKinds.VERIFIED, "ACCEPTED", null);
        e2.occurredAt = now.minus(1, ChronoUnit.HOURS);

        var results = repo.findJudgmentEvents(null, null,
                now.minus(3, ChronoUnit.HOURS), now, null);

        assertThat(results).hasSize(2);
    }

    @Test
    void countJudgmentOutcomesGroupsByTypeAndOutcome() {
        var channelId = UUID.randomUUID();
        var now = Instant.now();

        persistVerifiedEvent(channelId, "code_review", "ACCEPTED", now.minus(1, ChronoUnit.HOURS));
        persistVerifiedEvent(channelId, "code_review", "ACCEPTED", now.minus(30, ChronoUnit.MINUTES));
        persistVerifiedEvent(channelId, "code_review", "REJECTED", now.minus(20, ChronoUnit.MINUTES));
        persistVerifiedEvent(channelId, "quality_check", "ACCEPTED", now.minus(10, ChronoUnit.MINUTES));

        var results = repo.countJudgmentOutcomes(
                now.minus(2, ChronoUnit.HOURS), now, null);

        assertThat(results).hasSize(3); // code_review/ACCEPTED, code_review/REJECTED, quality_check/ACCEPTED
    }

    @Test
    void findPendingJudgmentsReturnsYieldedWithNoTerminal() {
        var channelId = UUID.randomUUID();
        var resolvedId = UUID.randomUUID();
        var pendingId = UUID.randomUUID();

        persistJudgmentEvent(channelId, resolvedId, JudgmentEventKinds.YIELDED, null, null);
        persistJudgmentEvent(channelId, resolvedId, JudgmentEventKinds.VERIFIED, "ACCEPTED", null);
        persistJudgmentEvent(channelId, pendingId, JudgmentEventKinds.YIELDED, null, null);

        var results = repo.findPendingJudgments(null);

        assertThat(results).hasSize(1);
        assertThat(results.getFirst().judgmentId).isEqualTo(pendingId);
    }

    private MessageLedgerEntry persistJudgmentEvent(UUID channelId, UUID judgmentId,
            String toolName, String verificationOutcome, Double evidenceQuality) {
        var e = new MessageLedgerEntry();
        e.subjectId = channelId;
        e.channelId = channelId;
        e.messageType = "EVENT";
        e.toolName = toolName;
        e.judgmentId = judgmentId;
        e.judgmentType = "code_review";
        e.verificationOutcome = verificationOutcome;
        e.evidenceQuality = evidenceQuality;
        e.sequenceNumber = 1;
        e.entryType = io.casehub.ledger.api.model.LedgerEntryType.EVENT;
        e.actorId = "test-actor";
        e.actorType = io.casehub.platform.api.identity.ActorType.AGENT;
        e.actorRole = "test-role";
        e.occurredAt = Instant.now();
        e.tenancyId = io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;
        // persist via EntityManager — repo.save() has REQUIRES_NEW complications
        return e;
    }

    private void persistVerifiedEvent(UUID channelId, String judgmentType,
            String outcome, Instant occurredAt) {
        var e = persistJudgmentEvent(channelId, UUID.randomUUID(),
                JudgmentEventKinds.VERIFIED, outcome, null);
        e.judgmentType = judgmentType;
        e.occurredAt = occurredAt;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentQueryTest -pl runtime`
Expected: FAIL — methods don't exist yet

- [ ] **Step 3: Implement query methods**

Use `ide_insert_member` to add three methods to `MessageLedgerEntryRepository.java`:

```java
public List<MessageLedgerEntry> findJudgmentEvents(
        UUID channelId, UUID judgmentId, Instant from, Instant to, String tenancyId) {
    String tid = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    String jpql = "FROM MessageLedgerEntry e WHERE e.tenancyId = :tenancyId"
        + " AND e.toolName IN :kinds";
    if (channelId != null) jpql += " AND e.channelId = :channelId";
    if (judgmentId != null) jpql += " AND e.judgmentId = :judgmentId";
    if (from != null) jpql += " AND e.occurredAt >= :from";
    if (to != null) jpql += " AND e.occurredAt <= :to";
    jpql += " ORDER BY e.occurredAt ASC";

    var query = em.createQuery(jpql, MessageLedgerEntry.class)
        .setParameter("tenancyId", tid)
        .setParameter("kinds", JudgmentEventKinds.ALL);
    if (channelId != null) query.setParameter("channelId", channelId);
    if (judgmentId != null) query.setParameter("judgmentId", judgmentId);
    if (from != null) query.setParameter("from", from);
    if (to != null) query.setParameter("to", to);
    return query.getResultList();
}

public List<Object[]> countJudgmentOutcomes(Instant from, Instant to, String tenancyId) {
    String tid = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    return em.createQuery(
        "SELECT e.judgmentType, e.toolName, e.verificationOutcome, COUNT(e)"
        + " FROM MessageLedgerEntry e"
        + " WHERE e.tenancyId = :tenancyId"
        + " AND e.toolName IN :terminalKinds"
        + " AND e.occurredAt >= :from AND e.occurredAt <= :to"
        + " GROUP BY e.judgmentType, e.toolName, e.verificationOutcome", Object[].class)
        .setParameter("tenancyId", tid)
        .setParameter("terminalKinds", JudgmentEventKinds.TERMINAL)
        .setParameter("from", from)
        .setParameter("to", to)
        .getResultList();
}

public List<MessageLedgerEntry> findPendingJudgments(String tenancyId) {
    String tid = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    return em.createQuery(
        "FROM MessageLedgerEntry e WHERE e.tenancyId = :tenancyId"
        + " AND e.toolName = :yieldedKind"
        + " AND NOT EXISTS (SELECT 1 FROM MessageLedgerEntry v"
        + "   WHERE v.tenancyId = :tenancyId"
        + "   AND v.judgmentId = e.judgmentId"
        + "   AND v.toolName IN :terminalKinds)"
        + " ORDER BY e.occurredAt ASC", MessageLedgerEntry.class)
        .setParameter("tenancyId", tid)
        .setParameter("yieldedKind", JudgmentEventKinds.YIELDED)
        .setParameter("terminalKinds", JudgmentEventKinds.TERMINAL)
        .getResultList();
}
```

Add import for `JudgmentEventKinds`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentQueryTest -pl runtime`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java
git add runtime/src/test/java/io/casehub/qhorus/runtime/ledger/JudgmentQueryTest.java
git commit -m "feat(#413): judgment query methods — findJudgmentEvents, countOutcomes, findPending

SQL aggregation for fulfillment report (no bulk loading). IN clause with
JudgmentEventKinds constants for compile-time contract enforcement.
findPendingJudgments unbounded — follows ObligationReport.stillOpen pattern.

Refs #413"
```

### Task 3: JudgmentAttributionReport model + service + test

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentAttributionReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentEvent.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/JudgmentAttributionReportService.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/JudgmentAttributionReportServiceTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.findJudgmentEvents(UUID, UUID, Instant, Instant, String)` from Task 2
- Consumes: `CausalGraphService.buildGraph(String, int, String)` from existing runtime
- Consumes: `TrustGateService.currentScore(String)` from casehub-ledger
- Consumes: `LedgerVerificationService.treeRoot(UUID, String)` from casehub-ledger
- Produces: `JudgmentAttributionReportService.generate(UUID judgmentId, int limit, String tenancyId)` → `JudgmentAttributionReport` — used by REST/GraphQL endpoints (Task 5)

- [ ] **Step 1: Create model records**

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentEvent.java`:

```java
package io.casehub.qhorus.compliance.model;

import java.time.Instant;

public record JudgmentEvent(
    String eventKind,
    String actorId,
    Instant occurredAt,
    Double evidenceQuality,
    String verificationOutcome,
    String escalationReason,
    Double trustScoreAtTime,
    Long durationMs
) {}
```

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentAttributionReport.java`:

```java
package io.casehub.qhorus.compliance.model;

import java.time.Instant;
import java.util.List;

public record JudgmentAttributionReport(
    String judgmentId,
    String judgmentType,
    int channelCount,
    List<String> channels,
    String correlationId,
    String verificationOutcome,
    Long totalDurationMs,
    List<JudgmentEvent> events,
    List<AttributionNode> causalNodes,
    List<AttributionEdge> causalEdges,
    String merkleRoot,
    Instant generatedAt,
    int schemaVersion
) {}
```

- [ ] **Step 2: Write failing test for JudgmentAttributionReportService**

Create `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/JudgmentAttributionReportServiceTest.java`:

```java
package io.casehub.qhorus.compliance.report;

import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.qhorus.api.judgment.JudgmentEventKinds;
import io.casehub.qhorus.runtime.ledger.CausalGraphService;
import io.casehub.qhorus.runtime.ledger.CausalGraphService.CausalGraph;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.OptionalDouble;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class JudgmentAttributionReportServiceTest {

    private JudgmentAttributionReportService service;
    private MessageLedgerEntryRepository mockRepo;
    private CausalGraphService mockCausalGraph;
    private TrustGateService mockTrustGate;

    @BeforeEach
    void setUp() {
        service = new JudgmentAttributionReportService();
        mockRepo = mock(MessageLedgerEntryRepository.class);
        mockCausalGraph = mock(CausalGraphService.class);
        mockTrustGate = mock(TrustGateService.class);

        service.ledgerRepo = mockRepo;
        service.causalGraphService = mockCausalGraph;
        service.trustGateServiceInstance = mockInstance(mockTrustGate);
        service.verificationServiceInstance = mockInstance(null);
    }

    @Test
    void generatesReportFromJudgmentEvents() {
        var judgmentId = UUID.randomUUID();
        var channelId = UUID.randomUUID();
        var correlationId = UUID.randomUUID().toString();
        var now = Instant.now();

        var yielded = buildEntry(JudgmentEventKinds.YIELDED, judgmentId, channelId,
                correlationId, now.minus(2, ChronoUnit.MINUTES), "agent-engine");
        var responded = buildEntry(JudgmentEventKinds.RESPONDED, judgmentId, channelId,
                correlationId, now.minus(1, ChronoUnit.MINUTES), "agent-reviewer");
        responded.evidenceQuality = 0.85;
        var verified = buildEntry(JudgmentEventKinds.VERIFIED, judgmentId, channelId,
                correlationId, now, "agent-engine");
        verified.verificationOutcome = "ACCEPTED";

        when(mockRepo.findJudgmentEvents(isNull(), eq(judgmentId), isNull(), isNull(), isNull()))
                .thenReturn(List.of(yielded, responded, verified));
        when(mockCausalGraph.buildGraph(eq(correlationId), anyInt(), isNull()))
                .thenReturn(new CausalGraph(correlationId, null, 1, List.of(channelId.toString()),
                        null, "FULFILLED", false, List.of(), List.of()));
        when(mockTrustGate.currentScore(anyString()))
                .thenReturn(OptionalDouble.of(0.9));

        var report = service.generate(judgmentId, 200, null);

        assertThat(report.judgmentId()).isEqualTo(judgmentId.toString());
        assertThat(report.judgmentType()).isEqualTo("code_review");
        assertThat(report.verificationOutcome()).isEqualTo("ACCEPTED");
        assertThat(report.events()).hasSize(3);
        assertThat(report.events().get(0).eventKind()).isEqualTo(JudgmentEventKinds.YIELDED);
        assertThat(report.events().get(2).eventKind()).isEqualTo(JudgmentEventKinds.VERIFIED);
        assertThat(report.schemaVersion()).isEqualTo(1);
    }

    @Test
    void returnsEmptyReportWhenNoEventsFound() {
        when(mockRepo.findJudgmentEvents(any(), any(), any(), any(), any()))
                .thenReturn(List.of());

        var report = service.generate(UUID.randomUUID(), 200, null);

        assertThat(report.events()).isEmpty();
        assertThat(report.causalNodes()).isEmpty();
    }

    private MessageLedgerEntry buildEntry(String toolName, UUID judgmentId, UUID channelId,
            String correlationId, Instant occurredAt, String actorId) {
        var e = new MessageLedgerEntry();
        e.subjectId = channelId;
        e.channelId = channelId;
        e.messageType = "EVENT";
        e.toolName = toolName;
        e.judgmentId = judgmentId;
        e.judgmentType = "code_review";
        e.correlationId = correlationId;
        e.occurredAt = occurredAt;
        e.actorId = actorId;
        e.sequenceNumber = 1;
        e.entryType = io.casehub.ledger.api.model.LedgerEntryType.EVENT;
        e.actorType = io.casehub.platform.api.identity.ActorType.AGENT;
        e.actorRole = "test-role";
        e.tenancyId = "DEFAULT";
        return e;
    }

    @SuppressWarnings("unchecked")
    private <T> Instance<T> mockInstance(T value) {
        Instance<T> instance = mock(Instance.class);
        if (value != null) {
            when(instance.isResolvable()).thenReturn(true);
            when(instance.get()).thenReturn(value);
        } else {
            when(instance.isResolvable()).thenReturn(false);
        }
        return instance;
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentAttributionReportServiceTest -pl compliance-report`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement JudgmentAttributionReportService**

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/JudgmentAttributionReportService.java`:

```java
package io.casehub.qhorus.compliance.report;

import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.qhorus.api.judgment.JudgmentEventKinds;
import io.casehub.qhorus.compliance.model.AttributionEdge;
import io.casehub.qhorus.compliance.model.AttributionNode;
import io.casehub.qhorus.compliance.model.JudgmentAttributionReport;
import io.casehub.qhorus.compliance.model.JudgmentEvent;
import io.casehub.qhorus.runtime.ledger.CausalGraphService;
import io.casehub.qhorus.runtime.ledger.CausalGraphService.CausalGraph;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.OptionalDouble;
import java.util.Set;
import java.util.TreeSet;
import java.util.UUID;

@ApplicationScoped
public class JudgmentAttributionReportService {

    @Inject MessageLedgerEntryRepository ledgerRepo;
    @Inject CausalGraphService causalGraphService;
    @Inject Instance<TrustGateService> trustGateServiceInstance;
    @Inject LedgerEntryRepository ledgerEntryRepository;
    @Inject Instance<LedgerVerificationService> verificationServiceInstance;

    public JudgmentAttributionReport generate(UUID judgmentId, int limit, String tenancyId) {
        List<MessageLedgerEntry> entries = ledgerRepo.findJudgmentEvents(
                null, judgmentId, null, null, tenancyId);

        if (entries.isEmpty()) {
            return new JudgmentAttributionReport(
                    judgmentId.toString(), null, 0, List.of(), null, null, null,
                    List.of(), List.of(), List.of(), null, Instant.now(), 1);
        }

        MessageLedgerEntry yielded = entries.stream()
                .filter(e -> JudgmentEventKinds.YIELDED.equals(e.toolName))
                .findFirst().orElse(entries.getFirst());

        String judgmentType = yielded.judgmentType;
        String correlationId = yielded.correlationId;

        String verificationOutcome = entries.stream()
                .filter(e -> JudgmentEventKinds.VERIFIED.equals(e.toolName))
                .map(e -> e.verificationOutcome)
                .findFirst().orElse(null);

        Long totalDurationMs = null;
        if (entries.size() >= 2) {
            Instant first = entries.getFirst().occurredAt;
            Instant last = entries.getLast().occurredAt;
            if (first != null && last != null) {
                totalDurationMs = Duration.between(first, last).toMillis();
            }
        }

        List<JudgmentEvent> events = entries.stream()
                .map(e -> toJudgmentEvent(e))
                .toList();

        List<AttributionNode> causalNodes = List.of();
        List<AttributionEdge> causalEdges = List.of();
        Set<String> channels = new TreeSet<>();

        if (correlationId != null) {
            CausalGraph graph = causalGraphService.buildGraph(correlationId, limit, tenancyId);
            causalNodes = graph.nodes().stream()
                    .map(n -> enrichNode(n, tenancyId))
                    .toList();
            causalEdges = graph.edges().stream()
                    .map(e -> new AttributionEdge(e.from(), e.to(), e.type(), e.elapsedMs()))
                    .toList();
            graph.channels().forEach(channels::add);
        }

        entries.stream()
                .map(e -> e.channelId)
                .filter(java.util.Objects::nonNull)
                .map(UUID::toString)
                .forEach(channels::add);

        String merkleRoot = buildMerkleRoot(channels, tenancyId);

        return new JudgmentAttributionReport(
                judgmentId.toString(), judgmentType,
                channels.size(), new ArrayList<>(channels), correlationId,
                verificationOutcome, totalDurationMs,
                events, causalNodes, causalEdges, merkleRoot, Instant.now(), 1);
    }

    private JudgmentEvent toJudgmentEvent(MessageLedgerEntry e) {
        Double trustScore = null;
        if (trustGateServiceInstance.isResolvable()) {
            OptionalDouble score = trustGateServiceInstance.get().currentScore(e.actorId);
            if (score.isPresent()) {
                trustScore = score.getAsDouble();
            }
        }
        return new JudgmentEvent(
                e.toolName, e.actorId, e.occurredAt,
                e.evidenceQuality, e.verificationOutcome,
                null, trustScore, e.durationMs);
    }

    private AttributionNode enrichNode(CausalGraphService.GraphNode node, String tenancyId) {
        UUID entryId = UUID.fromString(node.entryId());
        Double trustScore = null;
        if (trustGateServiceInstance.isResolvable()) {
            OptionalDouble score = trustGateServiceInstance.get().currentScore(node.actorId());
            if (score.isPresent()) trustScore = score.getAsDouble();
        }
        String attestationVerdict = null;
        var attestations = ledgerEntryRepository.findAttestationsByEntryId(entryId, tenancyId);
        if (!attestations.isEmpty()) {
            attestationVerdict = attestations.getLast().verdict.name();
        }
        return new AttributionNode(
                node.entryId(), node.channelId(), node.channelName(),
                node.messageType(), node.actorId(), node.occurredAt(),
                node.content(), node.causedByEntryId(), node.depth(),
                trustScore, attestationVerdict, null, null, null);
    }

    private String buildMerkleRoot(Set<String> channels, String tenancyId) {
        if (!verificationServiceInstance.isResolvable()) {
            return null;
        }
        LedgerVerificationService service = verificationServiceInstance.get();
        List<String> parts = new ArrayList<>();
        for (String chId : channels) {
            try {
                String root = service.treeRoot(UUID.fromString(chId), tenancyId);
                parts.add(chId + "=" + root);
            } catch (Exception e) {
                // No frontier — skip
            }
        }
        return parts.isEmpty() ? null : String.join(";", parts);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentAttributionReportServiceTest -pl compliance-report`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentEvent.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentAttributionReport.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/report/JudgmentAttributionReportService.java
git add compliance-report/src/test/java/io/casehub/qhorus/compliance/report/JudgmentAttributionReportServiceTest.java
git commit -m "feat(#413): JudgmentAttributionReport — per-judgment provenance chain

Model records + service. Composes CausalGraphService for message flow,
judgment EVENTs for lifecycle timeline, trust scores for enrichment.
Multi-channel support via channelCount + channels list.

Refs #413"
```

### Task 4: JudgmentFulfillmentReport model + service + test

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentFulfillmentReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentTypeSummary.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/CallerSummary.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/JudgmentFulfillmentReportService.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/JudgmentFulfillmentReportServiceTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.countJudgmentOutcomes(Instant, Instant, String)` from Task 2
- Consumes: `MessageLedgerEntryRepository.findPendingJudgments(String)` from Task 2
- Consumes: `MessageLedgerEntryRepository.findJudgmentEvents(UUID, UUID, Instant, Instant, String)` from Task 2 — for response time computation
- Consumes: `TrustGateService.currentScore(String)` from casehub-ledger
- Produces: `JudgmentFulfillmentReportService.generate(Instant from, Instant to, String judgmentType, String actorId, String tenancyId)` → `JudgmentFulfillmentReport` — used by REST/GraphQL endpoints (Task 5)

- [ ] **Step 1: Create model records**

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentTypeSummary.java`:

```java
package io.casehub.qhorus.compliance.model;

public record JudgmentTypeSummary(
    String judgmentType,
    int total, int accepted, int rejected, int escalated, int pending,
    double acceptanceRate, double averageResponseTimeMs, double averageEvidenceQuality
) {}
```

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/CallerSummary.java`:

```java
package io.casehub.qhorus.compliance.model;

public record CallerSummary(
    String actorId,
    int total, int accepted, int rejected, int escalated, int pending,
    double acceptanceRate, double averageResponseTimeMs, double averageEvidenceQuality,
    Double currentTrustScore
) {}
```

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentFulfillmentReport.java`:

```java
package io.casehub.qhorus.compliance.model;

import java.time.Instant;
import java.util.List;

public record JudgmentFulfillmentReport(
    Instant from, Instant to,
    List<JudgmentTypeSummary> byType,
    List<CallerSummary> byCaller,
    int totalJudgments, int accepted, int rejected, int escalated, int pending,
    double overallAcceptanceRate, double averageResponseTimeMs, double averageEvidenceQuality,
    String merkleRoot, Instant generatedAt, int schemaVersion
) {}
```

- [ ] **Step 2: Write failing test**

Create `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/JudgmentFulfillmentReportServiceTest.java` — CDI-free with mocked repository. Tests should verify: per-type aggregation, per-caller aggregation, acceptance rate computation, pending detection (unbounded), evidence quality averaging with null exclusion.

- [ ] **Step 3: Implement JudgmentFulfillmentReportService**

Create `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/JudgmentFulfillmentReportService.java`. Service uses:
- `countJudgmentOutcomes()` for time-bounded outcome aggregation
- `findPendingJudgments()` for unbounded pending count
- `findJudgmentEvents()` with RESPONDED filter for response time and evidence quality computation
- `TrustGateService.currentScore()` for caller enrichment

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JudgmentFulfillmentReportServiceTest -pl compliance-report`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentTypeSummary.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/model/CallerSummary.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/model/JudgmentFulfillmentReport.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/report/JudgmentFulfillmentReportService.java
git add compliance-report/src/test/java/io/casehub/qhorus/compliance/report/JudgmentFulfillmentReportServiceTest.java
git commit -m "feat(#413): JudgmentFulfillmentReport — per-type and per-caller aggregation

SQL aggregation via countJudgmentOutcomes. Pending unbounded (all currently
open, follows ObligationReport.stillOpen pattern). Response time from
occurredAt timestamps. Evidence quality averaged with null exclusion.

Refs #413"
```

---

## Batch 3: API Surfaces — fully exposed

### Task 5: ReportType enum, GraphQL DTOs, resolvers, REST endpoints

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportType.java` — add 2 entries
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/JudgmentAttributionReportType.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/JudgmentFulfillmentReportType.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/JudgmentEventType.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/JudgmentTypeSummaryType.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/CallerSummaryType.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/ComplianceQueryResolver.java` — add 2 queries
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java` — add 2 endpoints
- Test: existing `ComplianceQueryResolverTest.java` extended

**Interfaces:**
- Consumes: `JudgmentAttributionReportService.generate(UUID, int, String)` from Task 3
- Consumes: `JudgmentFulfillmentReportService.generate(Instant, Instant, String, String, String)` from Task 4

- [ ] **Step 1: Add JUDGMENT_ATTRIBUTION and JUDGMENT_FULFILLMENT to ReportType**

Use `ide_edit_member` to add entries to `ReportType.java`:

```java
public enum ReportType {
    ATTRIBUTION, OBLIGATION, TRUST_HISTORY, VIOLATION, PROVENANCE,
    JUDGMENT_ATTRIBUTION, JUDGMENT_FULFILLMENT
}
```

- [ ] **Step 2: Create GraphQL DTOs**

Create each DTO following the `AttributionReportType.from()` pattern — `@Type` annotation, static `from()` method mapping from model record.

- [ ] **Step 3: Add GraphQL queries to ComplianceQueryResolver**

Use `ide_insert_member` to add two query methods:

```java
@Query
@Description("Generate a judgment attribution report — provenance chain for a single judgment exchange")
public JudgmentAttributionReportType complianceJudgmentAttribution(String judgmentId, Integer limit) {
    int depth = limit != null ? limit : 200;
    return JudgmentAttributionReportType.from(
            judgmentAttributionService.generate(UUID.fromString(judgmentId), depth, currentPrincipal.tenancyId()));
}

@Query
@Description("Generate a judgment fulfillment report — per-type and per-caller acceptance rates for a time window")
public JudgmentFulfillmentReportType complianceJudgmentFulfillment(String from, String to, String judgmentType, String actorId) {
    Instant f = from != null ? Instant.parse(from) : Instant.now().minus(30, ChronoUnit.DAYS);
    Instant t = to != null ? Instant.parse(to) : Instant.now();
    return JudgmentFulfillmentReportType.from(
            judgmentFulfillmentService.generate(f, t, judgmentType, actorId, currentPrincipal.tenancyId()));
}
```

Inject the two new service fields.

- [ ] **Step 4: Add REST endpoints to ComplianceReportResource**

Use `ide_insert_member` to add two endpoints:

```java
@GET
@Path("/judgment-attribution/{judgmentId}")
public Response getJudgmentAttribution(
        @PathParam("judgmentId") String judgmentId,
        @QueryParam("limit") @DefaultValue("200") int limit,
        @HeaderParam("Accept") @DefaultValue("application/json") String accept) {
    var report = judgmentAttributionService.generate(
            UUID.fromString(judgmentId), limit, tenancyContext.tenancyId());
    return renderResponse(report, accept);
}

@GET
@Path("/judgment-fulfillment")
public Response getJudgmentFulfillment(
        @QueryParam("from") String from,
        @QueryParam("to") String to,
        @QueryParam("judgmentType") String judgmentType,
        @QueryParam("actorId") String actorId,
        @HeaderParam("Accept") @DefaultValue("application/json") String accept) {
    Instant fromInstant = from != null ? Instant.parse(from) : Instant.now().minus(30, java.time.temporal.ChronoUnit.DAYS);
    Instant toInstant = to != null ? Instant.parse(to) : Instant.now();
    var report = judgmentFulfillmentService.generate(
            fromInstant, toInstant, judgmentType, actorId, tenancyContext.tenancyId());
    return renderResponse(report, accept);
}
```

Inject the two new service fields.

- [ ] **Step 5: Run full build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportType.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/ComplianceQueryResolver.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java
git commit -m "feat(#413): judgment report API surfaces — REST, GraphQL, MCP

ReportType gains JUDGMENT_ATTRIBUTION + JUDGMENT_FULFILLMENT. GraphQL DTOs
with @McpDomain for automatic MCP tool generation. REST endpoints follow
existing compliance resource pattern.

Refs #413"
```

### Task 6: Scheduler case, CSV/HTML renderers

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduler.java:48-69` — add JUDGMENT_FULFILLMENT case
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceScheduleResource.java` — add JUDGMENT_ATTRIBUTION rejection
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/CsvReportRenderer.java:25-33` — add judgment cases
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlReportRenderer.java:32-40` — add judgment cases
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/JudgmentRendererTest.java`

**Interfaces:**
- Consumes: `JudgmentAttributionReport`, `JudgmentFulfillmentReport` from Tasks 3-4
- Consumes: `JudgmentFulfillmentReportService.generate(...)` from Task 4

- [ ] **Step 1: Add JUDGMENT_FULFILLMENT to scheduler switch**

Use `ide_replace_member` on `ComplianceReportScheduler.generateAndStore()`:

```java
private void generateAndStore(ComplianceReportSchedule schedule, Instant from, Instant now) {
    Object report = switch (schedule.reportType) {
        case OBLIGATION -> obligationService.generate(
                schedule.channelId, from, now, null, schedule.tenancyId);
        case VIOLATION -> {
            if (schedule.channelId == null) {
                throw new IllegalStateException("VIOLATION schedule requires channelId");
            }
            yield violationService.generate(schedule.channelId, from, now, schedule.tenancyId);
        }
        case JUDGMENT_FULFILLMENT -> judgmentFulfillmentService.generate(
                from, now, null, null, schedule.tenancyId);
        default -> throw new IllegalStateException(
                "Scheduled generation not supported for " + schedule.reportType);
    };
    // ... rest unchanged
}
```

Inject `JudgmentFulfillmentReportService judgmentFulfillmentService`.

- [ ] **Step 2: Add JUDGMENT_ATTRIBUTION schedule rejection**

Add validation in the schedule creation endpoint (ComplianceScheduleResource or wherever schedule creation is handled) to reject `JUDGMENT_ATTRIBUTION`:

```java
if (input.reportType() == ReportType.JUDGMENT_ATTRIBUTION) {
    throw new IllegalArgumentException("JUDGMENT_ATTRIBUTION is on-demand only — not schedulable");
}
```

- [ ] **Step 3: Add judgment cases to CsvReportRenderer.render()**

Use `ide_replace_member` to add cases to the switch in `render()`:

```java
case JudgmentAttributionReport r -> renderJudgmentAttribution(r);
case JudgmentFulfillmentReport r -> renderJudgmentFulfillment(r);
```

Add rendering methods:

```java
private String renderJudgmentAttribution(JudgmentAttributionReport report) {
    StringBuilder sb = new StringBuilder();
    sb.append("eventKind,actorId,occurredAt,evidenceQuality,verificationOutcome,escalationReason,trustScore,durationMs\n");
    for (var e : report.events()) {
        sb.append(String.join(",",
            escape(e.eventKind()), escape(e.actorId()),
            escape(e.occurredAt() != null ? e.occurredAt().toString() : ""),
            e.evidenceQuality() != null ? String.valueOf(e.evidenceQuality()) : "",
            escape(e.verificationOutcome() != null ? e.verificationOutcome() : ""),
            escape(e.escalationReason() != null ? e.escalationReason() : ""),
            e.trustScoreAtTime() != null ? String.valueOf(e.trustScoreAtTime()) : "",
            e.durationMs() != null ? String.valueOf(e.durationMs()) : ""
        )).append("\n");
    }
    return sb.toString();
}

private String renderJudgmentFulfillment(JudgmentFulfillmentReport report) {
    StringBuilder sb = new StringBuilder();
    sb.append("judgmentType,total,accepted,rejected,escalated,pending,acceptanceRate,avgResponseTimeMs,avgEvidenceQuality\n");
    for (var t : report.byType()) {
        sb.append(String.join(",",
            escape(t.judgmentType()), String.valueOf(t.total()),
            String.valueOf(t.accepted()), String.valueOf(t.rejected()),
            String.valueOf(t.escalated()), String.valueOf(t.pending()),
            String.valueOf(t.acceptanceRate()),
            String.valueOf(t.averageResponseTimeMs()),
            String.valueOf(t.averageEvidenceQuality())
        )).append("\n");
    }
    sb.append("\nactorId,total,accepted,rejected,escalated,pending,acceptanceRate,avgResponseTimeMs,avgEvidenceQuality,trustScore\n");
    for (var c : report.byCaller()) {
        sb.append(String.join(",",
            escape(c.actorId()), String.valueOf(c.total()),
            String.valueOf(c.accepted()), String.valueOf(c.rejected()),
            String.valueOf(c.escalated()), String.valueOf(c.pending()),
            String.valueOf(c.acceptanceRate()),
            String.valueOf(c.averageResponseTimeMs()),
            String.valueOf(c.averageEvidenceQuality()),
            c.currentTrustScore() != null ? String.valueOf(c.currentTrustScore()) : ""
        )).append("\n");
    }
    return sb.toString();
}
```

- [ ] **Step 4: Add judgment cases to HtmlReportRenderer.render()**

Same pattern — add cases to the switch and rendering methods.

- [ ] **Step 5: Write renderer tests**

Create `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/JudgmentRendererTest.java` — CDI-free tests verifying CSV and HTML output structure for both judgment report types.

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS, ALL TESTS PASS

- [ ] **Step 7: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduler.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/format/CsvReportRenderer.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlReportRenderer.java
git add compliance-report/src/test/java/io/casehub/qhorus/compliance/format/JudgmentRendererTest.java
git commit -m "feat(#413): judgment scheduler + renderers — CSV, HTML, scheduled generation

JUDGMENT_FULFILLMENT schedulable (lastRun→now window). JUDGMENT_ATTRIBUTION
on-demand only — schedule creation rejected. CSV/HTML renderers follow
existing section patterns.

Closes #413"
```

---

## References

- `2026-08-27-judgment-compliance-evidence-design.md` — design spec
- `MessageLedgerEntry.java:99-118` — domainContentBytes() implementation
- `LedgerWriteService.java:385-429` — populateTelemetry() implementation
- `ComplianceReportScheduler.java:48-69` — generateAndStore() switch
- `CsvReportRenderer.java:24-34` — render() dispatch switch
- `HtmlReportRenderer.java:32-40` — render() dispatch switch
- `ComplianceReportResource.java:45-70` — REST endpoint pattern
- `ComplianceQueryResolver.java:44-58` — GraphQL query pattern
- `AttributionReportType.java` — GraphQL DTO from() pattern
- `MessageLedgerEntryTestFactory.java` — test factory pattern
- `V2003__ledger_routing_metadata.sql` — migration naming convention
- `decisions.md` (D1-D4) — design decisions
- GitHub #413 — focal issue
- GitHub #410 — parent epic
- `ledger-entry-repository-cross-dtype-jpql` protocol — JPQL FROM constraint
- `observer-test-transaction-discipline` protocol — test transaction discipline
