# Compliance Evidence Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #402 — E5: Compliance evidence export — EU AI Act audit reports
**Issue group:** #402

**Goal:** Build an optional `compliance-report/` module that composes data from qhorus runtime, casehub-ledger, and casehub-ops (via SPI) into 5 compliance report types, exposed via REST + GraphQL + MCP with scheduled generation.

**Architecture:** New optional Maven module `compliance-report/` containing 5 report composition services, format renderers (JSON/CSV/HTML), REST resources, GraphQL resolvers (@McpDomain), and a scheduled report generator using platform's DigestSchedule. SPI types in `api/` module. Reports stored as SharedData artefacts with a `compliance_report` metadata index.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, SmallRye GraphQL, Jackson, casehub-ledger, casehub-platform-api (DigestSchedule)

## Global Constraints

- Java package: `io.casehub.qhorus.compliance` (compliance-report module), `io.casehub.qhorus.api.spi.compliance` (SPI in api)
- Maven artifact: `casehub-qhorus-compliance-report`
- No dependency on `casehub-ops` — compliance posture via `CompliancePostureProvider` SPI with `@DefaultBean`
- Flyway migrations in `runtime/src/main/resources/db/qhorus/migration/` (V47, V48)
- Reports always stored as JSON internally; re-rendered to requested format on retrieval
- Trust score trajectory requires ledger D1 (snapshot table) — falls back to current score until then
- CSV rendering via StringBuilder + RFC 4180 escaping — no external library
- HTML rendering via StringBuilder + inline CSS — no template engine
- All report records are immutable snapshots with `schemaVersion` (versioned per report type)
- Use `ide_insert_member` / `ide_replace_member` for Java code edits, never bash Edit

---

## Batch 1: Foundation — Module Scaffold + SPI + Runtime Prerequisite

### Task 1: CompliancePostureProvider SPI + Module Scaffold + Migrations

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/compliance/CompliancePostureProvider.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/compliance/CompliancePosture.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/compliance/PostureEntry.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/compliance/PostureStatus.java`
- Create: `compliance-report/pom.xml`
- Modify: `pom.xml` (parent — add `<module>compliance-report</module>`)
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportType.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportFormat.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/AttributionReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/AttributionNode.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/AttributionEdge.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ObligationReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ChannelObligationSummary.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/AgentObligationSummary.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/TrustHistoryReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ActorTrustTrajectory.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/TrustSnapshot.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/AttestationSummaryEntry.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ViolationReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ViolationEntry.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ProvenanceReport.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/posture/NoOpCompliancePostureProvider.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/storage/ComplianceReportRecord.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduleEntity.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportGeneratedEvent.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V47__compliance_report_schedule.sql`
- Create: `runtime/src/main/resources/db/qhorus/migration/V48__compliance_report.sql`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/model/ReportModelTest.java`

**Interfaces:**
- Produces: `CompliancePostureProvider.getPosture(String, Instant, Instant) → CompliancePosture`
- Produces: All model records (AttributionReport, ObligationReport, etc.) used by Tasks 3-8
- Produces: `ReportType` enum (ATTRIBUTION, OBLIGATION, TRUST_HISTORY, VIOLATION, PROVENANCE)
- Produces: `ReportFormat` enum (JSON, CSV, HTML)
- Produces: `ComplianceReportGeneratedEvent` record

- [ ] **Step 1: Create SPI types in api/ module**

Create `CompliancePostureProvider` interface, `CompliancePosture` record, `PostureEntry` record, `PostureStatus` enum in `api/src/main/java/io/casehub/qhorus/api/spi/compliance/`.

```java
// CompliancePostureProvider.java
package io.casehub.qhorus.api.spi.compliance;
public interface CompliancePostureProvider {
    CompliancePosture getPosture(String tenancyId, java.time.Instant from, java.time.Instant to);
}

// CompliancePosture.java
package io.casehub.qhorus.api.spi.compliance;
import java.util.List;
public record CompliancePosture(List<PostureEntry> entries) {
    public static final CompliancePosture EMPTY = new CompliancePosture(List.of());
}

// PostureEntry.java
package io.casehub.qhorus.api.spi.compliance;
import java.time.Instant;
public record PostureEntry(String category, PostureStatus status, String description, String evidence, Instant checkedAt) {}

// PostureStatus.java
package io.casehub.qhorus.api.spi.compliance;
public enum PostureStatus { COMPLIANT, NON_COMPLIANT, PARTIAL, UNKNOWN }
```

- [ ] **Step 2: Create compliance-report module pom.xml**

Create `compliance-report/pom.xml` following the `webhook-observer/pom.xml` pattern. Dependencies: `casehub-qhorus-api`, `casehub-qhorus` (runtime), `casehub-ledger`, `casehub-platform-api`, `casehub-platform-graphql`. Provided: `quarkus-smallrye-graphql`, `quarkus-hibernate-orm`, `jakarta.enterprise.cdi-api`. Test: `quarkus-junit5`, `casehub-qhorus-persistence-memory`, `casehub-platform` (MockCurrentPrincipal), `h2`.

- [ ] **Step 3: Add module to parent pom.xml**

Add `<module>compliance-report</module>` to the parent pom.xml `<modules>` section, after `notification-bridge`.

- [ ] **Step 4: Create all model records**

Create all record types from the spec in `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/`. Include `ReportType` enum, `ReportFormat` enum, and all 5 report records plus their nested types (see spec for exact field definitions). All records are from the spec — transcribe field-for-field.

- [ ] **Step 5: Create JPA entities**

Create `ComplianceReportRecord` (metadata index) and `ComplianceReportScheduleEntity` (schedule config) JPA entities per the spec's schema definitions.

- [ ] **Step 6: Create ComplianceReportGeneratedEvent CDI event record**

```java
package io.casehub.qhorus.compliance.schedule;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import io.casehub.qhorus.compliance.model.ReportType;

public record ComplianceReportGeneratedEvent(
    UUID reportId, ReportType reportType, String tenancyId,
    UUID artefactId, Instant generatedAt, UUID scheduleId,
    String requestedBy, Map<String, String> requestParameters
) {}
```

- [ ] **Step 7: Create NoOpCompliancePostureProvider**

```java
package io.casehub.qhorus.compliance.posture;
import io.casehub.qhorus.api.spi.compliance.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;

@DefaultBean
@ApplicationScoped
public class NoOpCompliancePostureProvider implements CompliancePostureProvider {
    @Override
    public CompliancePosture getPosture(String tenancyId, Instant from, Instant to) {
        return CompliancePosture.EMPTY;
    }
}
```

- [ ] **Step 8: Create Flyway migrations V47 and V48**

Create `runtime/src/main/resources/db/qhorus/migration/V47__compliance_report_schedule.sql` and `V48__compliance_report.sql` per the spec's SQL definitions.

- [ ] **Step 9: Write model record tests**

Verify record construction, null handling, and `CompliancePosture.EMPTY` constant.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ReportModelTest -pl compliance-report`

- [ ] **Step 10: Verify full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl api,compliance-report -am`
Expected: BUILD SUCCESS — module compiles with all dependencies resolved.

- [ ] **Step 11: Commit**

```bash
git -C <PROJECT> add api/src/main/java/io/casehub/qhorus/api/spi/compliance/ compliance-report/ pom.xml runtime/src/main/resources/db/qhorus/migration/V47* runtime/src/main/resources/db/qhorus/migration/V48*
git -C <PROJECT> commit -m "feat(#402): compliance-report module scaffold — SPI, models, migrations, NoOp provider Refs #402"
```

### Task 2: Runtime Query Prereqs — countByOutcome Time-Range + countByStateAndObligor

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/Commitment.java` (JPA store impl)
- Modify: `persistence-memory/src/main/java/.../InMemoryCommitmentStore.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryRepoTest.java`
- Test: `testing/src/test/java/.../CommitmentServiceTest.java` (or new contract test)

**Interfaces:**
- Produces: `countByOutcome(UUID channelId, Instant from, Instant to, String tenancyId) → Map<String, Long>`
- Produces: `CommitmentStore.countByStateAndObligor(String obligor, Instant from, Instant to, String tenancyId) → Map<CommitmentState, Long>` — per-agent obligation breakdown (FULFILLED, FAILED, DECLINED, DELEGATED, OPEN counts)

- [ ] **Step 1: Write the failing test**

Add test methods to `LedgerQueryRepoTest`:
- `countByOutcome_withTimeRange_filtersCorrectly` — messages inside window counted, outside excluded
- `countByOutcome_withTimeRange_emptyWindow_returnsEmptyMap`

```java
@Test
void countByOutcome_withTimeRange_filtersCorrectly() {
    // Setup: send COMMAND at T1, DONE at T2, COMMAND at T3
    // Query: countByOutcome(channelId, T1, T2, tenancyId)
    // Expect: COMMAND=1, DONE=1 (T3 excluded)
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerQueryRepoTest#countByOutcome_withTimeRange_filtersCorrectly -pl runtime`
Expected: FAIL — method does not exist

- [ ] **Step 3: Implement time-range overload**

Add to `MessageLedgerEntryRepository`:

```java
public Map<String, Long> countByOutcome(UUID channelId, Instant from, Instant to, String tenancyId) {
    List<Object[]> rows = em.createQuery(
            "SELECT e.messageType, COUNT(e) FROM MessageLedgerEntry e " +
                    "WHERE e.subjectId = :cid " +
                    "AND e.tenancyId = :tid " +
                    "AND e.occurredAt >= :from " +
                    "AND e.occurredAt <= :to " +
                    "AND e.messageType IN ('COMMAND', 'DONE', 'FAILURE', 'DECLINE', 'HANDOFF') " +
                    "GROUP BY e.messageType",
            Object[].class)
            .setParameter("cid", channelId)
            .setParameter("tid", tenancyId(tenancyId))
            .setParameter("from", from)
            .setParameter("to", to)
            .getResultList();
    Map<String, Long> result = new HashMap<>();
    for (Object[] row : rows) {
        result.put((String) row[0], (Long) row[1]);
    }
    return result;
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerQueryRepoTest#countByOutcome_withTimeRange -pl runtime`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add runtime/
git -C <PROJECT> commit -m "feat(#402): countByOutcome time-range overload for compliance reports Refs #402"
```

---

## Batch 2: Report Services — Attribution + Obligation

### Task 3: AttributionReportService

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/AttributionReportService.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/AttributionReportServiceTest.java`

**Interfaces:**
- Consumes: `CausalGraphService.buildGraph(correlationId, limit, tenancyId) → CausalGraph`
- Consumes: `TrustGateService.currentScore(actorId) → Optional<Double>`
- Consumes: `LedgerEntryRepository.findAttestationsByEntryId(entryId, tenancyId) → List<LedgerAttestation>`
- Consumes: `LedgerVerificationService.treeRoot(subjectId, tenancyId) → String` (returns String, not Optional — null means no Merkle frontier)
- Consumes: `LedgerComplianceReportService.reportForSubject(subjectId, tenancyId) → ComplianceReport` — for ComplianceSupplement enrichment (algorithmRef, confidenceScore, rationale on graph nodes)
- Produces: `generate(String correlationId, int limit, String tenancyId) → AttributionReport`

- [ ] **Step 1: Write failing tests**

CDI-free unit tests with Mockito mocks for all injected services. Test cases:
- `generate_buildsGraphAndEnrichesNodes` — verifies CausalGraph nodes enriched with trust scores and attestation verdicts
- `generate_enrichesWithComplianceSupplement` — nodes carry algorithmRef, confidenceScore, rationale from LedgerComplianceReportService
- `generate_compositesMerkleRoot` — verifies semicolon-separated `subjectId=rootHash` format (treeRoot returns String, null → omitted from composite)
- `generate_unknownCorrelation_returnsEmptyGraph` — empty nodes/edges when CausalGraph returns no data
- `generate_trustScoreFallback_currentScoreWhenSnapshotUnavailable` — uses TrustGateService.currentScore()

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AttributionReportServiceTest -pl compliance-report`

- [ ] **Step 3: Implement AttributionReportService**

`@ApplicationScoped` bean. Injects `CausalGraphService`, `Instance<TrustGateService>`, `Instance<LedgerVerificationService>`, `LedgerEntryRepository`. Maps `CausalGraph.GraphNode` to `AttributionNode` enriching each node with trust score (via `currentScore()`) and attestation verdict (via `findAttestationsByEntryId()`). Builds composite Merkle root from distinct `channelId` values in graph nodes.

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): AttributionReportService — causal graph enrichment with trust + attestations Refs #402"
```

### Task 4: ObligationReportService

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/ObligationReportService.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/ObligationReportServiceTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.countByOutcome(channelId, from, to, tenancyId) → Map<String, Long>` (from Task 2)
- Consumes: `CommitmentStore.countByStateAndObligor(obligor, from, to, tenancyId) → Map<CommitmentState, Long>` (from Task 2)
- Consumes: `CommitmentStore.findOpenByObligor(obligor) → List<Commitment>` (for stillOpen — current state, not time-bounded)
- Consumes: `ChannelStore.findAll() → List<Channel>` (or filtered by tenancy)
- Consumes: `TrustGateService.currentScore(actorId) → Optional<Double>`
- Consumes: `CompliancePostureProvider.getPosture(tenancyId, from, to) → CompliancePosture`
- Consumes: `LedgerVerificationService.treeRoot(subjectId, tenancyId) → String`
- Produces: `generate(UUID channelId, Instant from, Instant to, String actorId, String tenancyId) → ObligationReport`

- [ ] **Step 1: Write failing tests**

CDI-free unit tests. Test cases:
- `generate_perChannel_aggregatesOutcomeCounts` — per-channel fulfillment rates from countByOutcome(channelId, from, to)
- `generate_perAgent_usesCountByStateAndObligor` — per-agent obligation summary from CommitmentStore.countByStateAndObligor()
- `generate_crossChannel_noChannelFilter` — when channelId is null, aggregates across all channels
- `generate_stillOpen_notBoundedToTimeWindow` — stillOpen counts all currently open commitments regardless of creation time
- `generate_stalled_usesFromAsStalenessThreshold` — stalled uses `from` as the age cutoff, not `to`
- `generate_includesPosture_whenProviderReturnsData` — CompliancePosture included in report
- `generate_posture_emptyWhenNoOpProvider` — CompliancePosture.EMPTY from NoOp

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement ObligationReportService**

`@ApplicationScoped` bean. Injects `MessageLedgerEntryRepository`, `CommitmentStore`, `ChannelStore`, `Instance<TrustGateService>`, `Instance<LedgerVerificationService>`, `CompliancePostureProvider`. Aggregates per-channel using `countByOutcome(channelId, from, to)`. Per-agent uses `CommitmentStore.findOpenByObligor()`. Computes fulfillment rate = fulfilled / total.

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): ObligationReportService — per-channel and per-agent fulfillment rates Refs #402"
```

---

## Batch 3: Remaining Reports + PROV-DM

### Task 5: ViolationReportService + TrustHistoryReportService

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/ViolationReportService.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/TrustHistoryReportService.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/ViolationReportServiceTest.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/TrustHistoryReportServiceTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.listEntries(channelId, types, afterSeq, agentId, since, corrId, sortDesc, limit, tenancyId)` — filtered by system:enforcement and system:watchdog senders
- Consumes: `TrustExportService.exportActor(actorId) → Optional<TrustExportPayload>`
- Consumes: `LedgerEntryRepository.summariseAttestationsByActor(actorId, from, to, tenancyId) → AttestationSummary`
- Produces: `ViolationReportService.generate(UUID channelId, Instant from, Instant to, String tenancyId) → ViolationReport`
- Produces: `TrustHistoryReportService.generate(String actorId, Instant from, Instant to, String tenancyId) → TrustHistoryReport`

- [ ] **Step 1: Write ViolationReportService failing tests**

- `generate_extractsEnforcementEvents` — filters ledger entries by sender "system:enforcement", parses telemetry JSON for violation details
- `generate_includesWatchdogAlerts` — includes sender "system:watchdog" entries
- `generate_aggregatesViolationsBySource` — `violationsBySource` map populated correctly
- `generate_countsBlockedAdvisoryQuarantined` — correct categorization by enforcement mode

- [ ] **Step 2: Write TrustHistoryReportService failing tests**

- `generate_returnsCurrentScoreOnly_whenNoSnapshotTable` — D1 fallback, empty trajectory
- `generate_includesAttestationSummary` — attestation entries from `summariseAttestationsByActor`
- `generate_unknownActor_returnsEmptyTrajectory`

- [ ] **Step 3: Run all tests — verify they fail**

- [ ] **Step 4: Implement ViolationReportService**

`@ApplicationScoped`. Queries `MessageLedgerEntryRepository.listEntries()` with `EVENT` type filter, then filters results by sender prefix `"system:enforcement"` and `"system:watchdog"`. Parses `telemetry` field JSON to extract violation details. Categorizes by enforcement mode (BLOCKING → blocked count, ADVISORY → advisory count, QUARANTINE → quarantined count).

- [ ] **Step 5: Implement TrustHistoryReportService**

`@ApplicationScoped`. Uses `TrustExportService.exportActor()` for current score. Trajectory empty (D1 fallback). Attestation summary from `LedgerEntryRepository.summariseAttestationsByActor()`.

- [ ] **Step 6: Run all tests — verify they pass**

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): ViolationReportService + TrustHistoryReportService Refs #402"
```

### Task 6: ProvenanceReportService + PROV-JSON-LD Mapper

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/report/ProvenanceReportService.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/provdm/ProvJsonLdMapper.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/provdm/ProvJsonLdMapperTest.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/report/ProvenanceReportServiceTest.java`

**Interfaces:**
- Consumes: `CausalGraphService.buildGraph(correlationId, limit, tenancyId) → CausalGraph`
- Produces: `ProvJsonLdMapper.toProvJsonLd(CausalGraph) → Map<String, Object>`
- Produces: `ProvenanceReportService.generate(String correlationId, int limit, String tenancyId) → ProvenanceReport`

- [ ] **Step 1: Write ProvJsonLdMapper failing tests**

- `toProvJsonLd_containsCorrectContext` — `@context` with prov, ledger, qhorus, xsd namespaces
- `toProvJsonLd_mapsAgentsToProvAgent` — each unique actorId → `prov:Agent` with `ledger:actor/{actorId}` IRI
- `toProvJsonLd_mapsCommandToProvActivity` — COMMAND/QUERY entries → `prov:Activity`
- `toProvJsonLd_mapsDelegationToActedOnBehalfOf` — HANDOFF → `prov:actedOnBehalfOf`
- `toProvJsonLd_mapsCausalEdgesToWasDerivedFrom` — `causedByEntryId` edges → `prov:wasDerivedFrom`
- `toProvJsonLd_mapsChannelToProvLocation` — channels → `prov:Location` with `qhorus:channel/{id}` IRI
- `toProvJsonLd_sharedIRIsMatchLedgerFormat` — IRI format matches `LedgerProvSerializer` output

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement ProvJsonLdMapper**

Pure function (no CDI). Takes `CausalGraph`, produces `Map<String, Object>` conforming to PROV-JSON-LD structure. Builds `@context`, `agent` map, `activity` map, `entity` map, and relationship maps (`wasInformedBy`, `actedOnBehalfOf`, `wasDerivedFrom`, `atLocation`). Uses `ledger:actor/`, `ledger:activity/`, `ledger:entry/` IRI prefixes shared with `LedgerProvSerializer`. Uses `qhorus:channel/` for channel locations.

- [ ] **Step 4: Implement ProvenanceReportService**

`@ApplicationScoped`. Delegates to `CausalGraphService.buildGraph()`, maps via `ProvJsonLdMapper.toProvJsonLd()`, wraps in `ProvenanceReport`.

- [ ] **Step 5: Run tests — verify they pass**

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): ProvenanceReportService + PROV-JSON-LD mapper Refs #402"
```

---

## Batch 4: Format Renderers + Storage + REST

### Task 7: Format Renderers + Storage Layer

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/ReportRenderer.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/JsonReportRenderer.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/CsvReportRenderer.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlReportRenderer.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/storage/ComplianceReportRecordStore.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/storage/ComplianceReportStorageService.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/CsvReportRendererTest.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/HtmlReportRendererTest.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/storage/ComplianceReportStorageServiceTest.java`

**Interfaces:**
- Produces: `ReportRenderer.render(Object report) → byte[]`
- Produces: `ComplianceReportStorageService.store(ReportType, Object report, ReportFormat, UUID scheduleId, String tenancyId) → ComplianceReportRecord`
- Produces: `ComplianceReportStorageService.retrieve(UUID id) → Optional<byte[]>`

- [ ] **Step 1: Write CsvReportRenderer failing tests**

- `render_attributionReport_oneRowPerNode` — correct CSV columns, RFC 4180 escaping
- `render_obligationReport_twoSections` — channel summary + agent summary with header separator
- `render_handlesEmbeddedCommasAndQuotes` — RFC 4180 edge cases
- `render_handlesNullFields` — nullable fields render as empty

- [ ] **Step 2: Write HtmlReportRenderer failing tests**

- `render_attributionReport_producesValidHtml` — contains `<table>`, `<thead>`, `<tbody>`
- `render_includesPrintFriendlyCss` — contains `@media print` CSS

- [ ] **Step 3: Run tests — verify they fail**

- [ ] **Step 4: Implement renderers**

`ReportRenderer` interface with `contentType()`, `render(Object)`, `supports(ReportFormat)`. `JsonReportRenderer` delegates to Jackson `ObjectMapper`. `CsvReportRenderer` uses `StringBuilder` with RFC 4180 escaping (double-quote fields containing commas, quotes, or newlines). `HtmlReportRenderer` uses `StringBuilder` with inline CSS including `@media print` styles.

- [ ] **Step 5: Implement ComplianceReportStorageService**

`@ApplicationScoped`. `store()`: serializes report to JSON → `DataService.store()` for body → persists `ComplianceReportRecord` with artefact FK. `retrieve()`: loads `ComplianceReportRecord` → fetches body from `DataService.getByUuid()`.

- [ ] **Step 6: Run all tests — verify they pass**

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): format renderers (JSON/CSV/HTML) + report storage layer Refs #402"
```

### Task 8: REST Resources

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceScheduleResource.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/api/ComplianceReportResourceTest.java`

**Interfaces:**
- Consumes: All 5 report services (Tasks 3-6)
- Consumes: Format renderers (Task 7)
- Consumes: ComplianceReportStorageService (Task 7)

- [ ] **Step 1: Write failing REST tests**

`@QuarkusTest` integration tests:
- `getAttribution_returnsJson` — `GET /api/compliance/attribution/{corrId}` with `Accept: application/json`
- `getAttribution_returnsCsv` — same endpoint with `Accept: text/csv`
- `getObligations_withTimeRange` — `GET /api/compliance/obligations?from=...&to=...`
- `getViolations_requiresChannel` — 400 when channel param missing
- `getProvenance_alwaysJsonLd` — ignores Accept header, returns PROV-JSON-LD
- `getStoredReport_reRendersFormat` — stored JSON, retrieved as CSV
- `deleteReport_releasesArtefact` — verify SharedData claim released

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement ComplianceReportResource**

JAX-RS `@Path("/api/compliance")`. On-demand endpoints call report services, render via Accept-header content negotiation, fire `ComplianceReportGeneratedEvent` with `scheduleId=null`. Stored report endpoints delegate to `ComplianceReportStorageService`.

- [ ] **Step 4: Implement ComplianceScheduleResource**

JAX-RS `@Path("/api/compliance/schedules")`. CRUD for `ComplianceReportScheduleEntity`. Validates: `channelId` required when `reportType=VIOLATION`. Serializes `DigestSchedule` to JSON for `scheduleJson` column.

- [ ] **Step 5: Run tests — verify they pass**

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): REST resources — on-demand reports, stored reports, schedule CRUD Refs #402"
```

---

## Batch 5: GraphQL + Scheduling

### Task 9: GraphQL Resolvers

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/ComplianceQueryResolver.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/ComplianceMutationResolver.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/AttributionReportType.java` (+ all other DTO types)
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/graphql/ComplianceQueryResolverTest.java`

**Interfaces:**
- Consumes: All 5 report services
- Consumes: ComplianceReportStorageService, ComplianceReportScheduleStore

- [ ] **Step 1: Create GraphQL DTO types**

Create `AttributionReportType`, `ObligationReportType`, `TrustHistoryReportType`, `ViolationReportType`, `ProvenanceReportType`, `ComplianceReportRecordType`, `ComplianceReportScheduleType`, `ComplianceScheduleInput`, `ComplianceScheduleUpdateInput` in `compliance-report/src/main/java/io/casehub/qhorus/compliance/graphql/dto/`.

- [ ] **Step 2: Write failing GraphQL tests**

`@QuarkusTest`:
- `complianceAttribution_returnsReport` — GraphQL query `{ complianceAttribution(correlationId: "...") { correlationId outcome } }`
- `createComplianceSchedule_mutation` — creates schedule, verifies persisted

- [ ] **Step 3: Implement ComplianceQueryResolver**

`@GraphQLApi @McpDomain("qhorus") @ApplicationScoped`. Thin adapter — each `@Query` method calls the corresponding report service, maps to GraphQL DTO type.

- [ ] **Step 4: Implement ComplianceMutationResolver**

`@GraphQLApi @McpDomain("qhorus") @ApplicationScoped`. CRUD mutations for schedules.

- [ ] **Step 5: Run tests — verify they pass**

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): GraphQL resolvers with @McpDomain for compliance reports Refs #402"
```

### Task 10: Scheduled Report Generation

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduleStore.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduler.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/schedule/ComplianceReportSchedulerTest.java`

**Interfaces:**
- Consumes: `DigestSchedule.isFlushDue(lastRunAt, lastRunAt, now) → boolean`
- Consumes: All report services, ComplianceReportStorageService
- Consumes: `ComplianceReportScheduleStore.findEnabled() → List<ComplianceReportScheduleEntity>`

- [ ] **Step 1: Write failing scheduler tests**

CDI-free unit tests:
- `sweep_generatesDueReport` — schedule with `isFlushDue=true` → report generated + stored
- `sweep_skipsNotDueReport` — schedule with `isFlushDue=false` → no generation
- `sweep_updatesLastRunAt` — after successful generation, `lastRunAt` updated
- `sweep_errorIsolation_continuesToNextSchedule` — one failing schedule doesn't block others
- `sweep_catchesUpAfterDowntime` — `lastRunAt` far in the past → generates on next sweep

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement ComplianceReportScheduleStore**

JPA store for `ComplianceReportScheduleEntity`. Methods: `findEnabled()`, `updateLastRunAt(UUID, Instant)`, `findByTenancy(String)`, `save()`, `delete()`.

- [ ] **Step 4: Implement ComplianceReportScheduler**

`@ApplicationScoped`. `@Scheduled(every = "1h")` sweep. Uses `CrossTenantComplianceReportScheduleStore` (per `scheduled-service-cross-tenant-stores` protocol). Per-schedule try-catch for error isolation. Deserializes `scheduleJson` → `DigestSchedule`, calls `isFlushDue(lastRunAt, lastRunAt, now)`. On due: generates report via the appropriate report service (switch on `reportType`), stores via `ComplianceReportStorageService`, fires `ComplianceReportGeneratedEvent`.

- [ ] **Step 5: Run tests — verify they pass**

- [ ] **Step 6: Verify full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add compliance-report/
git -C <PROJECT> commit -m "feat(#402): scheduled report generation with DigestSchedule + per-schedule error isolation Refs #402"
```

---

## References

- [2026-08-27-compliance-evidence-export-design.md] — design spec this plan implements
- [decisions.md (D1-D12)] — captured design decisions
- [runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CausalGraphService.java] — attribution chain data source
- [runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java:238] — countByOutcome to extend
- [webhook-observer/pom.xml] — optional module pom pattern
- [DigestSchedule.java (casehub-platform-api)] — scheduling semantics
- [LedgerProvSerializer.java (casehub-ledger)] — PROV-JSON-LD IRI format reference
- [casehub-qhorus#402] — focal issue
- [casehubio/ledger#203] — deferred trust score snapshot table
- [casehubio/qhorus#417] — deferred PDF rendering
- [casehubio/qhorus#418] — deferred digital signatures
- [casehubio/qhorus#419] — deferred automated retention
- [scheduled-service-cross-tenant-stores protocol] — scheduler must use CrossTenant stores
- [consumer-spi-placement protocol] — SPI types in api/ module
- [observer-test-transaction-discipline protocol] — QuarkusTransaction.requiringNew() for observer tests
