# Breaking Changes — Full Platform Sweep Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Execute all deferred breaking changes across the casehubio platform in a single coordinated push — module splits, MCP surface redesign, bug fixes — before publishing snapshots.

**Architecture:** Five repos change in dependency order: quarkus-ledger first (others depend on it), then quarkus-qhorus (MCP + api split), then quarkus-work (bug fixes + migration), then claudony (bug fixes + updated imports), then casehub-engine (updated imports). No snapshots published until all repos are green.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven multi-module, Quarkiverse parent, H2 (test), GitHub Packages (SNAPSHOT distribution)

**Constraint:** Do NOT run `mvn deploy` or `mvn install` until Task 31 (final install gate). casehub-engine consumers must not pick up broken snapshots mid-flight.

**Build command (all repos):** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests` (quick check) or `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test` (full test run). Use `-pl <module>` for targeted builds.

---

## Repo locations

| Repo | Path |
|---|---|
| quarkus-ledger | `~/claude/quarkus-ledger` |
| quarkus-qhorus | `~/claude/quarkus-qhorus` |
| quarkus-work | `~/claude/quarkus-work` |
| claudony | `~/claude/claudony` |
| casehub-engine | (path TBD — ask user) |

---

## Phase 1 — quarkus-ledger #73: api/jpa module split

**Why first:** quarkus-qhorus depends on quarkus-ledger. Split ledger first so qhorus can depend on `quarkus-ledger-api` only.

**Split content:**

`quarkus-ledger-api` (new module — no JPA, no Hibernate, no datasource):
- Abstract POJO `LedgerEntry` (copy fields from current `@Entity`, strip JPA annotations)
- POJO `LedgerAttestation` (copy fields, strip `@Entity`)
- POJO `ActorTrustScore` (copy fields, strip `@Entity`)
- Interfaces: `LedgerEntryRepository`, `ReactiveLedgerEntryRepository`, `ActorTrustScoreRepository`
- SPIs: `LedgerTraceIdProvider`, `ActorTypeResolver`
- Enums: `ActorType`, `LedgerEntryType`, `AttestationVerdict`
- Supplement types: `LedgerSupplement`, `ComplianceSupplement`, `ProvenanceSupplement`, `LedgerSupplementSerializer`

`quarkus-ledger` (existing runtime — adds JPA back):
- All current `@Entity` classes renamed to `*Entity` suffix: `LedgerEntryEntity`, `LedgerAttestationEntity`, `ActorTrustScoreEntity` — extend the api POJOs
- JPA repository impls: `JpaLedgerEntryRepository`, `JpaActorTrustScoreRepository`
- Flyway migrations, services, config — unchanged

### Task 1: Create quarkus-ledger-api Maven module

**Files:**
- Create: `~/claude/quarkus-ledger/api/pom.xml`
- Modify: `~/claude/quarkus-ledger/pom.xml` (add `<module>api</module>`)

- [ ] **Step 1: Add api module to parent pom**

In `~/claude/quarkus-ledger/pom.xml`, change:
```xml
<modules>
  <module>runtime</module>
  <module>deployment</module>
  <module>examples</module>
</modules>
```
to:
```xml
<modules>
  <module>api</module>
  <module>runtime</module>
  <module>deployment</module>
  <module>examples</module>
</modules>
```

- [ ] **Step 2: Create api/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.quarkiverse.ledger</groupId>
    <artifactId>quarkus-ledger-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>quarkus-ledger-api</artifactId>
  <name>Quarkus Ledger - API</name>
  <description>Domain model and SPI interfaces — no JPA, safe for any module to depend on</description>

  <dependencies>
    <!-- Jackson for supplement serialization only — no JPA -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <scope>provided</scope>
    </dependency>
    <!-- Mutiny for reactive SPI -->
    <dependency>
      <groupId>io.smallrye.reactive</groupId>
      <artifactId>mutiny</artifactId>
      <scope>provided</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 3: Create directory structure**

```bash
mkdir -p ~/claude/quarkus-ledger/api/src/main/java/io/quarkiverse/ledger/api/{model,repository,spi,supplement}
```

- [ ] **Step 4: Verify parent pom builds with new module**

```bash
cd ~/claude/quarkus-ledger && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl api validate
```
Expected: BUILD SUCCESS

---

### Task 2: Move domain model to api (POJO versions)

**Files:**
- Create: `api/src/main/java/io/quarkiverse/ledger/api/model/LedgerEntry.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/model/LedgerAttestation.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/model/ActorTrustScore.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/model/ActorType.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/model/LedgerEntryType.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/model/AttestationVerdict.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/model/ActorIdentity.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/supplement/` (all supplement types)

- [ ] **Step 1: Copy enums to api**

Copy `ActorType`, `LedgerEntryType`, `AttestationVerdict` from `runtime/.../model/` to `api/.../model/` — change package to `io.quarkiverse.ledger.api.model`. These are pure enums, no changes needed.

- [ ] **Step 2: Create POJO LedgerEntry in api**

Read current `runtime/src/main/java/io/quarkiverse/ledger/runtime/model/LedgerEntry.java`. Create `api/src/main/java/io/quarkiverse/ledger/api/model/LedgerEntry.java` as an abstract POJO — copy all fields as public instance variables, strip all JPA annotations (`@Entity`, `@Table`, `@Column`, `@Id`, `@Inheritance`, `@DiscriminatorColumn`, `@EntityListeners`, etc.), keep non-JPA logic:

```java
package io.quarkiverse.ledger.api.model;

import java.time.Instant;
import java.util.UUID;

public abstract class LedgerEntry {
    public Long id;
    public String actorId;
    public String channelId;
    public String correlationId;
    public String causedByEntryId;
    public String sha256;
    public Long sequenceNumber;
    public Instant recordedAt;
    public LedgerEntryType entryType;
    public String traceId;
    // (copy remaining fields from current LedgerEntry, no annotations)
}
```
(Check actual fields in the existing source and copy them all.)

- [ ] **Step 3: Create POJO LedgerAttestation in api**

Read `runtime/.../model/LedgerAttestation.java`. Create `api/.../model/LedgerAttestation.java` — same pattern: copy all fields, strip @Entity/@Table/@Column etc.:

```java
package io.quarkiverse.ledger.api.model;

import java.time.Instant;
import java.util.UUID;

public class LedgerAttestation {
    public UUID id;
    public Long entryId;
    public AttestationVerdict verdict;
    public Double confidence;
    public String attestorId;
    public Instant attestedAt;
    // (copy remaining fields from current LedgerAttestation)
}
```

- [ ] **Step 4: Create POJO ActorTrustScore in api**

Same pattern — copy from `runtime/.../model/ActorTrustScore.java`, strip JPA:

```java
package io.quarkiverse.ledger.api.model;

import java.time.Instant;
import java.util.UUID;

public class ActorTrustScore {
    public UUID id;
    public String actorId;
    public String actorType;
    public double alpha;
    public double beta;
    public Instant updatedAt;
    // (copy remaining fields)
}
```

- [ ] **Step 5: Move supplement types to api**

Copy all classes from `runtime/.../model/supplement/` to `api/.../model/supplement/` — change package. These reference only Jackson (no JPA).

- [ ] **Step 6: Move ActorTypeResolver to api**

Copy `runtime/.../model/ActorTypeResolver.java` to `api/.../model/ActorTypeResolver.java` — change package. Pure utility, no deps.

---

### Task 3: Move SPI interfaces to api

**Files:**
- Create: `api/src/main/java/io/quarkiverse/ledger/api/repository/LedgerEntryRepository.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/repository/ReactiveLedgerEntryRepository.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/repository/ActorTrustScoreRepository.java`
- Create: `api/src/main/java/io/quarkiverse/ledger/api/spi/LedgerTraceIdProvider.java`

- [ ] **Step 1: Move LedgerTraceIdProvider**

Copy `runtime/.../service/LedgerTraceIdProvider.java` to `api/.../spi/LedgerTraceIdProvider.java` — change package to `io.quarkiverse.ledger.api.spi`. No other changes (pure interface with Optional<String> return).

- [ ] **Step 2: Move repository interfaces**

Copy `runtime/.../repository/LedgerEntryRepository.java` to `api/.../repository/LedgerEntryRepository.java` — change package, update imports to use `io.quarkiverse.ledger.api.model.*`.

Copy `runtime/.../repository/ReactiveLedgerEntryRepository.java` to `api/.../repository/` — same treatment.

Copy `runtime/.../repository/ActorTrustScoreRepository.java` to `api/.../repository/` — same treatment.

- [ ] **Step 3: Build api module alone**

```bash
cd ~/claude/quarkus-ledger && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl api compile
```
Expected: BUILD SUCCESS with no JPA imports in api module.

---

### Task 4: Update quarkus-ledger runtime to depend on api and rename JPA entities

**Files:**
- Modify: `runtime/pom.xml` (add dependency on api)
- Modify: `runtime/.../model/LedgerEntry.java` (extend api.LedgerEntry, strip duplicated fields)
- Modify: `runtime/.../model/LedgerAttestation.java` (same)
- Modify: `runtime/.../model/ActorTrustScore.java` (same)
- Modify: all runtime consumers of old model classes (update imports)

- [ ] **Step 1: Add api dependency to runtime pom**

In `runtime/pom.xml`, add:
```xml
<dependency>
  <groupId>io.quarkiverse.ledger</groupId>
  <artifactId>quarkus-ledger-api</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Update runtime LedgerEntry to extend api LedgerEntry**

Keep `@Entity`, `@Table`, `@Inheritance`, `@DiscriminatorColumn`, `@EntityListeners` on the class.  
Add `extends io.quarkiverse.ledger.api.model.LedgerEntry`.  
Remove all fields now inherited from the api superclass (keeping only JPA-specific overrides via `@Column` annotations).  
Add `@Column` annotations as field-level annotations on inherited fields using `@AttributeOverride` if needed.

- [ ] **Step 3: Update LedgerAttestation similarly**

Add `extends io.quarkiverse.ledger.api.model.LedgerAttestation`, keep `@Entity @Table`, use `@AttributeOverride` or `@Column` on inherited fields.

- [ ] **Step 4: Update ActorTrustScore similarly**

Same pattern.

- [ ] **Step 5: Update all runtime imports**

Search for `import io.quarkiverse.ledger.runtime.model.*` across runtime and update:
- `LedgerEntry` → `io.quarkiverse.ledger.api.model.LedgerEntry` (or keep runtime if it's the JPA entity)
- `LedgerAttestation` → api version
- `ActorTrustScore` → api version
- `ActorType`, `LedgerEntryType`, `AttestationVerdict` → api enums
- `ActorTypeResolver` → api utility
- `LedgerTraceIdProvider` → api SPI
- `LedgerEntryRepository` → api repository interface

- [ ] **Step 6: Update runtime repository implementations**

`JpaLedgerEntryRepository` now implements `io.quarkiverse.ledger.api.repository.LedgerEntryRepository`. Update return types to use api POJO types (the JPA entity IS the api POJO via inheritance, so casts work).

- [ ] **Step 7: Remove duplicated classes from runtime**

Delete the originals from `runtime/.../model/`: enums now in api, supplement types now in api, `ActorTypeResolver` now in api, `LedgerTraceIdProvider` now in api.

Delete the originals from `runtime/.../repository/`: `LedgerEntryRepository.java`, `ReactiveLedgerEntryRepository.java`, `ActorTrustScoreRepository.java` (interfaces moved to api; JPA implementations stay).

- [ ] **Step 8: Build and test ledger**

```bash
cd ~/claude/quarkus-ledger && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 9: Commit**

```bash
cd ~/claude/quarkus-ledger
git add -A
git commit -m "refactor: split into quarkus-ledger-api and quarkus-ledger (#73)

Move domain model (LedgerEntry, LedgerAttestation, ActorTrustScore as POJOs),
repository SPI interfaces, LedgerTraceIdProvider, ActorTypeResolver, and enums
to new quarkus-ledger-api module. JPA entities extend api model types.
Consumers needing only SPIs now depend on quarkus-ledger-api without JPA.

Refs #73"
```

---

## Phase 2 — quarkus-work bug fixes (#71, #72, #70)

### Task 5: Fix duplicate migration V1001 (#71)

**Files:**
- Investigate: `~/claude/quarkus-work/quarkus-work-ledger/src/main/resources/db/migration/`

- [ ] **Step 1: Check the duplicate**

```bash
find ~/claude/quarkus-work -name "V1001*.sql" | sort
cat ~/claude/quarkus-work/quarkus-work-ledger/src/main/resources/db/migration/V1001__actor_trust_score.sql
```

Compare with `~/claude/quarkus-ledger/runtime/src/main/resources/db/migration/V1001__actor_trust_score.sql`.

- [ ] **Step 2: Determine Flyway location config**

```bash
grep -r "flyway\|migration" ~/claude/quarkus-work/quarkus-work-ledger/src/main/resources/application.properties 2>/dev/null
```

If quarkus-work uses a separate Flyway location (different datasource), the two V1001s don't conflict. If same location, delete the quarkus-work copy and rely on quarkus-ledger's migration.

- [ ] **Step 3: Delete stale copy if safe**

If quarkus-work's V1001 is reachable by the same Flyway scanner as quarkus-ledger's, delete it:
```bash
rm ~/claude/quarkus-work/quarkus-work-ledger/src/main/resources/db/migration/V1001__actor_trust_score.sql
```

If quarkus-work uses a separate Flyway location, update the quarkus-work copy to match the current quarkus-ledger schema (add UUID pk, score_type, scope_key columns if missing).

- [ ] **Step 4: Run quarkus-work tests**

```bash
cd ~/claude/quarkus-work && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```
Expected: no Flyway duplicate-script or table-already-exists errors.

- [ ] **Step 5: Commit**

```bash
cd ~/claude/quarkus-work
git add -A
git commit -m "fix: resolve duplicate V1001 actor_trust_score migration (#71)

Refs #71"
```

---

### Task 6: Fix JSON escaping and null guard in quarkus-work (#72)

**Files:**
- Modify: `~/claude/quarkus-work/quarkus-work-ledger/src/main/java/io/quarkiverse/work/ledger/...` (find class with `buildDecisionContext()`)

- [ ] **Step 1: Find the affected class**

```bash
grep -rn "buildDecisionContext\|String.format.*json\|eventSuffix" ~/claude/quarkus-work --include="*.java" | grep -v test
```

- [ ] **Step 2: Fix JSON building with Jackson**

Replace `String.format()`-based JSON construction with Jackson `ObjectMapper`:

```java
// Before (insecure):
private String buildDecisionContext(String field1, String field2) {
    return String.format("{\"field1\": \"%s\", \"field2\": \"%s\"}", field1, field2);
}

// After (safe):
private static final ObjectMapper MAPPER = new ObjectMapper();

private String buildDecisionContext(String field1, String field2) {
    try {
        return MAPPER.writeValueAsString(Map.of("field1", field1, "field2", field2));
    } catch (JsonProcessingException e) {
        throw new RuntimeException("Failed to build decision context", e);
    }
}
```

- [ ] **Step 3: Fix null guard on eventSuffix()**

```bash
grep -n "eventSuffix" ~/claude/quarkus-work --include="*.java" -r
```

Add null check:
```java
// Before:
String suffix = eventSuffix();
EVENT_META.get(suffix)[0];  // NPE if suffix is null

// After:
String suffix = eventSuffix();
if (suffix == null) return; // or throw, depending on intent
EVENT_META.get(suffix)[0];
```

- [ ] **Step 4: Fix TrustScoreComputerTest expectations**

```bash
cd ~/claude/quarkus-work && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TrustScoreComputerTest 2>&1 | tail -40
```

The test expects score=1.0 for unattested and score=0.0 for flagged, but the Bayesian Beta model returns 0.5 for no evidence. Update the test expectations to match the algorithm:

```java
// Before (wrong expectation):
assertThat(score).isEqualTo(1.0);

// After (correct for Beta(1,1) prior = 0.5 mean):
assertThat(score).isCloseTo(0.5, within(0.01));
```

(Read the actual test to confirm exact assertions, then fix them to match Beta model outputs.)

- [ ] **Step 5: Run tests**

```bash
cd ~/claude/quarkus-work && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
cd ~/claude/quarkus-work
git add -A
git commit -m "fix: JSON escaping via Jackson, null guard on eventSuffix, fix TrustScoreComputerTest expectations (#72)

Refs #72"
```

---

### Task 7: Migrate quarkus-work to TrustGateService (#70)

**Files:**
- Modify: `~/claude/quarkus-work/quarkus-work-ledger/src/main/java/io/quarkiverse/work/ledger/api/ActorTrustResource.java`
- Modify: `~/claude/quarkus-work/quarkus-work-examples/src/main/java/io/quarkiverse/work/examples/queue/DocumentQueueScenario.java`

- [ ] **Step 1: Check TrustGateService API**

```bash
cat ~/claude/quarkus-ledger/runtime/src/main/java/io/quarkiverse/ledger/runtime/service/TrustGateService.java
```

- [ ] **Step 2: Update ActorTrustResource to use TrustGateService**

Replace `@Inject ActorTrustScoreRepository trustScoreRepo` with `@Inject TrustGateService trustGateService`.  
Replace direct entity field access with `trustGateService` API calls.

- [ ] **Step 3: Update DocumentQueueScenario to use TrustGateService**

Replace `trustScoreRepo.findByActorId(...)` call with equivalent `TrustGateService` method.

- [ ] **Step 4: Run tests**

```bash
cd ~/claude/quarkus-work && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```

- [ ] **Step 5: Commit**

```bash
cd ~/claude/quarkus-work
git add -A
git commit -m "refactor: migrate ActorTrustScoreRepository direct access to TrustGateService (#70)

Refs #70"
```

---

## Phase 3 — quarkus-qhorus MCP surface changes (#121)

### Task 8: #121-A — Artefact terminology rename

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: test files that reference `share_data`, `get_shared_data`, `list_shared_data`

- [ ] **Step 1: Rename tool methods in QhorusMcpTools**

In `QhorusMcpTools.java`:
- `@Tool(name = "share_data", ...)` → `@Tool(name = "share_artefact", ...)`; method `shareData(...)` → `shareArtefact(...)`
- `@Tool(name = "get_shared_data", ...)` → `@Tool(name = "get_artefact", ...)`; method `getSharedData(...)` → `getArtefact(...)`
- `@Tool(name = "list_shared_data", ...)` → `@Tool(name = "list_artefacts", ...)`; method `listSharedData(...)` → `listArtefacts()`
- Update `artefact_refs` description in `send_message`: change "from share_data" to "from share_artefact"

- [ ] **Step 2: Same renames in ReactiveQhorusMcpTools**

Same three renames in the reactive mirror class.

- [ ] **Step 3: Find and fix test references**

```bash
grep -rn "share_data\|get_shared_data\|list_shared_data" runtime/src/test examples/*/src/test --include="*.java" -l
```

Update all test files to use new names.

- [ ] **Step 4: Run tests**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(mcp): rename share_data/get_shared_data/list_shared_data to artefact terminology (#121-A)

Breaking: share_data → share_artefact, get_shared_data → get_artefact,
list_shared_data → list_artefacts. Consistent with claim/release/revoke_artefact.

Closes (partial) #121, Refs #119"
```

---

### Task 9: #121-H — Consolidate list_pending_waits + list_pending_approvals

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/CommitmentStore.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveCommitmentStore.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaCommitmentStore.java`
- Modify: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryCommitmentStore.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Add findAllOpen() to CommitmentStore**

In `CommitmentStore.java`, add:
```java
/** All OPEN or ACKNOWLEDGED commitments across all channels. */
List<Commitment> findAllOpen();
```

In `ReactiveCommitmentStore.java`, add:
```java
Uni<List<Commitment>> findAllOpen();
```

- [ ] **Step 2: Implement in JpaCommitmentStore**

```java
@Override
public List<Commitment> findAllOpen() {
    return Commitment.<Commitment>find(
        "state IN ?1 ORDER BY expiresAt ASC NULLS LAST",
        List.of(CommitmentState.OPEN, CommitmentState.ACKNOWLEDGED)
    ).list();
}
```

- [ ] **Step 3: Implement in InMemoryCommitmentStore**

```java
@Override
public List<Commitment> findAllOpen() {
    return store.values().stream()
        .filter(c -> c.state == CommitmentState.OPEN || c.state == CommitmentState.ACKNOWLEDGED)
        .sorted(Comparator.comparing(c -> c.expiresAt != null ? c.expiresAt : Instant.MAX))
        .toList();
}
```

- [ ] **Step 4: Add list_pending_commitments tool, remove old tools**

In `QhorusMcpTools.java`:

**Remove** `listPendingWaits()` (@Tool `list_pending_waits`) and `listPendingApprovals()` (@Tool `list_pending_approvals`).

**Add** new tool:
```java
@Tool(name = "list_pending_commitments", description = "List non-terminal commitments across all channels. "
    + "type_filter: 'waits' = all blocked OPEN/ACKNOWLEDGED; 'approvals' = approval-gate subset; omit for all. "
    + "Returns oldest first.")
@Transactional
public List<CommitmentDetail> listPendingCommitments(
        @ToolArg(name = "type_filter", description = "Optional: 'waits' or 'approvals'", required = false) String typeFilter) {
    List<Commitment> all = commitmentStore.findAllOpen();
    Stream<Commitment> stream = all.stream();
    if ("approvals".equalsIgnoreCase(typeFilter)) {
        // approvals are COMMAND commitments awaiting human respond_to_approval
        stream = stream.filter(c -> c.requiresApproval != null && c.requiresApproval);
    }
    Instant now = Instant.now();
    return stream.map(CommitmentDetail::from).toList();
}
```

(Check `Commitment` fields to determine the approval filter predicate — use whatever field `list_pending_approvals` was using to filter.)

**Also remove** references to `PendingWaitSummary` and `ApprovalSummary` records in `QhorusMcpToolsBase` if they are no longer used.

- [ ] **Step 5: Mirror in ReactiveQhorusMcpTools**

Same removal + add of reactive version using `commitmentStore.findAllOpen()` returning `Uni<List<CommitmentDetail>>`.

- [ ] **Step 6: Update respond_to_approval description** (it referenced `list_pending_approvals`):

Change description to reference `list_pending_commitments(type_filter='approvals')`.

- [ ] **Step 7: Run tests**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing
```

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(mcp): consolidate list_pending_waits/approvals into list_pending_commitments (#121-H)

Breaking: remove list_pending_waits and list_pending_approvals.
New: list_pending_commitments(type_filter?) with optional 'waits'/'approvals' scoping.

Closes (partial) #121, Refs #119"
```

---

### Task 10: #121-G — Observer consolidation into read_only flag

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/instance/Instance.java`
- Modify: `runtime/src/main/resources/db/migration/` (add new Flyway script)
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Delete: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/instance/ObserverRegistry.java`
- Modify: test files

- [ ] **Step 1: Add readOnly to Instance entity**

In `Instance.java`, add:
```java
@Column(name = "read_only", nullable = false)
public boolean readOnly;
```

- [ ] **Step 2: Add Flyway migration**

Find the highest existing migration number:
```bash
ls runtime/src/main/resources/db/migration/ | sort | tail -3
```

Create `V{N+1}__instance_read_only.sql`:
```sql
ALTER TABLE instance ADD COLUMN read_only BOOLEAN NOT NULL DEFAULT FALSE;
```

- [ ] **Step 3: Add read_only param to register tool**

In `QhorusMcpTools.java`, update `register(...)` method to add:
```java
@ToolArg(name = "read_only", description = "When true, registers as a passive observer — can read messages/events but cannot send. Default false.", required = false) Boolean readOnly
```

In the method body, set `instance.readOnly = readOnly != null && readOnly;`

Update description to mention read_only option.

- [ ] **Step 4: Add include_events to check_messages**

In `QhorusMcpTools.java`, update `checkMessages(...)` method:
```java
@ToolArg(name = "include_events", description = "When true, include EVENT type messages in results (default false). Set true for read-only observer instances.", required = false) Boolean includeEvents
```

In the query/filter logic, if `includeEvents` is true, include MESSAGE_TYPE = EVENT in the results. Currently EVENTs are excluded — add the condition:
```java
boolean showEvents = includeEvents != null && includeEvents;
// In message filter: skip EVENT unless showEvents is true
if (!showEvents && MessageType.EVENT.equals(msg.type)) continue;
```

- [ ] **Step 5: Remove the three observer tools**

In `QhorusMcpTools.java`, remove:
- `registerObserver(...)` (@Tool `register_observer`)
- `deregisterObserver(...)` (@Tool `deregister_observer`)
- `readObserverEvents(...)` (@Tool `read_observer_events`)

In `ReactiveQhorusMcpTools.java`, same three removals.

- [ ] **Step 6: Delete ObserverRegistry**

The `ObserverRegistry` is used only by the three deleted tools. Delete it:
```bash
rm runtime/src/main/java/io/quarkiverse/qhorus/runtime/instance/ObserverRegistry.java
```

- [ ] **Step 7: Remove ObserverRegistry injection from MCP tool classes**

Remove `@Inject ObserverRegistry observerRegistry;` from `QhorusMcpTools` and `ReactiveQhorusMcpTools`.

- [ ] **Step 8: Update InstanceStore if needed**

Check if `InstanceStore` has methods for observers — if so, remove them or they were only for the registry (which was in-memory).

- [ ] **Step 9: Update test files**

```bash
grep -rn "register_observer\|deregister_observer\|read_observer_events\|ObserverRegistry" runtime/src/test examples/*/src/test --include="*.java" -l
```

Remove or update tests that call removed tools. Add a test for `register` with `read_only=true` and verify `check_messages` with `include_events=true` returns EVENT messages to the read-only instance.

- [ ] **Step 10: Run tests**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing
```

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(mcp): consolidate observer model into read_only flag on register (#121-G)

Breaking: remove register_observer, deregister_observer, read_observer_events.
New: register(read_only=true) creates a passive observer instance.
New: check_messages(include_events=true) includes EVENT messages in results.
ObserverRegistry deleted — observer state now persisted in Instance table.

Closes (partial) #121, Refs #119"
```

---

### Task 11: #121-D — 3-step chunked upload API

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/data/DataService.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: test files

- [ ] **Step 1: Add begin/appendChunk/finalize to DataService**

In `DataService.java`, add three new methods (using the existing `store()` internally):

```java
public SharedData begin(String key, String createdBy, String description) {
    return store(key, description, createdBy, "", false, false);
}

public SharedData appendChunk(String key, String chunk) {
    SharedData data = getByKey(key)
        .orElseThrow(() -> new IllegalArgumentException("No artefact found for key: " + key));
    if (data.complete) {
        throw new IllegalStateException("Artefact '" + key + "' is already finalized");
    }
    return store(key, data.description, data.createdBy, chunk, true, false);
}

public SharedData finalize(String key) {
    SharedData data = getByKey(key)
        .orElseThrow(() -> new IllegalArgumentException("No artefact found for key: " + key));
    if (data.complete) {
        throw new IllegalStateException("Artefact '" + key + "' is already finalized");
    }
    data.complete = true;
    return dataStore.put(data);
}
```

- [ ] **Step 2: Remove append/last_chunk params from share_artefact**

In `QhorusMcpTools.shareArtefact(...)` (renamed in Task 8), remove:
- `@ToolArg(name = "append", ...)` parameter
- `@ToolArg(name = "last_chunk", ...)` parameter

Change the method body to always call `dataService.store(key, description, createdBy, content, false, true)` — single-shot, complete=true.

Update tool description: remove chunked upload mention, say "for large content use begin_artefact + append_chunk + finalize_artefact".

- [ ] **Step 3: Add the 3 new tools to QhorusMcpTools**

```java
@Tool(name = "begin_artefact", description = "Start a multi-chunk artefact upload. "
    + "Follow with one or more append_chunk calls, then finalize_artefact.")
@Transactional
public ArtefactDetail beginArtefact(
        @ToolArg(name = "key", description = "Unique key for this artefact") String key,
        @ToolArg(name = "created_by", description = "Owner instance identifier") String createdBy,
        @ToolArg(name = "description", description = "Human-readable description", required = false) String description) {
    return toArtefactDetail(dataService.begin(key, createdBy, description));
}

@Tool(name = "append_chunk", description = "Append a content chunk to an in-progress artefact. "
    + "Artefact must have been started with begin_artefact and not yet finalized.")
@Transactional
public ArtefactDetail appendChunk(
        @ToolArg(name = "key", description = "Artefact key (as given to begin_artefact)") String key,
        @ToolArg(name = "content", description = "Content chunk to append") String content) {
    try {
        return toArtefactDetail(dataService.appendChunk(key, content));
    } catch (IllegalArgumentException | IllegalStateException e) {
        throw new IllegalArgumentException(toolError(e));
    }
}

@Tool(name = "finalize_artefact", description = "Mark a chunked artefact as complete. "
    + "After this call, get_artefact will return the full assembled content.")
@Transactional
public ArtefactDetail finalizeArtefact(
        @ToolArg(name = "key", description = "Artefact key to finalize") String key) {
    try {
        return toArtefactDetail(dataService.finalize(key));
    } catch (IllegalArgumentException | IllegalStateException e) {
        throw new IllegalArgumentException(toolError(e));
    }
}
```

- [ ] **Step 4: Mirror all changes in ReactiveQhorusMcpTools**

Same `shareArtefact` simplification (remove append/last_chunk), plus 3 new reactive tool methods returning `Uni<ArtefactDetail>`.

- [ ] **Step 5: Write tests**

In the appropriate test file, add:
```java
@Test
void chunkedUpload_threeStepApi() {
    beginArtefact("myKey", "test-agent", "Test artefact");
    appendChunk("myKey", "first chunk");
    appendChunk("myKey", " second chunk");
    finalizeArtefact("myKey");
    
    ArtefactDetail detail = getArtefact(null, null); // by key
    assertThat(detail.complete()).isTrue();
    assertThat(detail.content()).isEqualTo("first chunk second chunk");
}

@Test
void appendChunk_onFinalizedArtefact_throws() {
    shareArtefact("finKey", null, "test-agent", "content");
    assertThatThrownBy(() -> appendChunk("finKey", "more"))
        .hasMessageContaining("already finalized");
}
```

- [ ] **Step 6: Run tests**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(mcp): replace append/last_chunk flags with begin_artefact+append_chunk+finalize_artefact (#121-D)

Breaking: remove append and last_chunk params from share_artefact.
New: begin_artefact(key, created_by), append_chunk(key, content), finalize_artefact(key).

Closes (partial) #121, Refs #119"
```

---

### Task 12: #121-C — Auto-claim/release on send_message

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

**Design:** When `send_message` includes `artefact_refs`, auto-call `dataService.claim(uuid, instanceUuid)` for each ref, where `instanceUuid` is looked up from the sender string via `instanceStore`. When DONE/FAILURE/DECLINE is sent with a `correlationId`, look up the original COMMAND/QUERY message for that correlationId, find its `artefactRefs`, and auto-release each.

- [ ] **Step 1: Add private helper to auto-claim**

In `QhorusMcpTools.java`, add a private helper:
```java
private void autoClaimArtefacts(List<String> artefactRefs, String sender) {
    if (artefactRefs == null || artefactRefs.isEmpty()) return;
    Optional<Instance> senderInstance = instanceStore.findByInstanceId(sender);
    if (senderInstance.isEmpty()) return; // unregistered callers skip auto-claim
    UUID instanceUuid = senderInstance.get().id;
    for (String ref : artefactRefs) {
        try {
            dataService.claim(UUID.fromString(ref), instanceUuid);
        } catch (Exception e) {
            // log but don't fail the send — artefact may not exist yet or already claimed
        }
    }
}
```

- [ ] **Step 2: Add private helper to auto-release**

```java
private void autoReleaseArtefactsForCorrelation(String correlationId, String sender) {
    if (correlationId == null) return;
    Optional<Instance> senderInstance = instanceStore.findByInstanceId(sender);
    if (senderInstance.isEmpty()) return;
    UUID instanceUuid = senderInstance.get().id;
    // find the original COMMAND or QUERY message for this correlationId
    messageStore.findOriginalForCorrelation(correlationId).ifPresent(original -> {
        if (original.artefactRefs == null || original.artefactRefs.isBlank()) return;
        for (String ref : original.artefactRefs.split(",")) {
            try {
                dataService.release(UUID.fromString(ref.trim()), instanceUuid);
            } catch (Exception e) {
                // log but don't fail the send
            }
        }
    });
}
```

- [ ] **Step 3: Add findOriginalForCorrelation to MessageStore**

In `MessageStore.java`, add:
```java
/** Find the first QUERY or COMMAND message with the given correlationId (the obligation source). */
Optional<Message> findOriginalForCorrelation(String correlationId);
```

Implement in `JpaMessageStore`:
```java
@Override
public Optional<Message> findOriginalForCorrelation(String correlationId) {
    return Message.<Message>find(
        "correlationId = ?1 AND type IN ?2",
        correlationId, List.of(MessageType.QUERY, MessageType.COMMAND)
    ).firstResultOptional();
}
```

Implement in `InMemoryMessageStore`:
```java
@Override
public Optional<Message> findOriginalForCorrelation(String correlationId) {
    return store.values().stream()
        .filter(m -> correlationId.equals(m.correlationId)
            && (MessageType.QUERY.equals(m.type) || MessageType.COMMAND.equals(m.type)))
        .findFirst();
}
```

- [ ] **Step 4: Hook auto-claim into sendMessage**

In `QhorusMcpTools.sendMessage(...)`, after the message is saved:
```java
Message saved = messageService.send(...);
// auto-claim artefact refs
autoClaimArtefacts(artefactRefs, sender);
```

- [ ] **Step 5: Hook auto-release into sendMessage for terminal types**

In `QhorusMcpTools.sendMessage(...)`, before returning, add:
```java
MessageType msgType = MessageType.valueOf(type.toUpperCase());
if (msgType == MessageType.DONE || msgType == MessageType.FAILURE || msgType == MessageType.DECLINE) {
    autoReleaseArtefactsForCorrelation(correlationId, sender);
}
```

- [ ] **Step 6: Mirror in ReactiveQhorusMcpTools**

Add reactive equivalents of auto-claim and auto-release (using Uni chains).

- [ ] **Step 7: Write tests**

```java
@Test
void sendMessage_withArtefactRefs_autoClaims() {
    String key = "test-artefact-" + UUID.randomUUID();
    ArtefactDetail artefact = shareArtefact(key, null, "sender-agent", "content");
    register("sender-agent", "Agent", null, null);
    
    // Before send: not claimed
    assertThat(tools.isGcEligible(artefact.id())).isTrue();
    
    // Send COMMAND with artefact_refs
    sendMessage("work", "sender-agent", "COMMAND", "do thing", 
                null, null, List.of(artefact.id()), null, null);
    
    // After send: auto-claimed, no longer GC-eligible
    assertThat(tools.isGcEligible(artefact.id())).isFalse();
}

@Test
void sendMessage_DONE_autoReleasesArtefacts() {
    String key = "test-artefact-" + UUID.randomUUID();
    ArtefactDetail artefact = shareArtefact(key, null, "sender-agent", "content");
    register("sender-agent", "Agent", null, null);
    
    MessageResult cmd = sendMessage("work", "sender-agent", "COMMAND", "do thing",
                                   null, null, List.of(artefact.id()), null, null);
    assertThat(tools.isGcEligible(artefact.id())).isFalse();
    
    // Send DONE on same correlationId
    sendMessage("work", "sender-agent", "DONE", "done", 
                cmd.correlationId(), null, null, null, null);
    
    // Released
    assertThat(tools.isGcEligible(artefact.id())).isTrue();
}
```

- [ ] **Step 8: Run tests**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing
```

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(mcp): auto-claim/release artefacts on send_message with artefact_refs (#121-C)

Breaking (behaviour): artefact_refs in send_message now auto-claims on behalf of sender.
DONE/FAILURE/DECLINE auto-releases previously claimed artefacts for the correlationId.
Explicit claim_artefact/release_artefact still available for manual override.

Closes (partial) #121, Refs #119"
```

---

### Task 13: #127 — delete_channel admin guard

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Add caller_instance_id to delete_channel**

In `QhorusMcpTools.deleteChannel(...)`, add:
```java
@ToolArg(name = "caller_instance_id", description = "Instance ID of the caller. Required when the channel has an admin_instances list.", required = false) String callerInstanceId
```

In the method body, before deletion, add admin check (same pattern as `pauseChannel`):
```java
Channel channel = channelService.findByName(channelName)
    .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
if (channel.adminInstances != null && !channel.adminInstances.isBlank()) {
    if (callerInstanceId == null || callerInstanceId.isBlank()) {
        return toolError("caller_instance_id required: channel '" + channelName + "' has admin_instances set");
    }
    Set<String> admins = Set.of(channel.adminInstances.split(","));
    if (!admins.contains(callerInstanceId.trim())) {
        return toolError("caller '" + callerInstanceId + "' is not in admin_instances for channel '" + channelName + "'");
    }
}
```

- [ ] **Step 2: Mirror in ReactiveQhorusMcpTools**

Same parameter + admin check.

- [ ] **Step 3: Write test**

```java
@Test
void deleteChannel_withAdminList_rejectsNonAdmin() {
    createChannel("protected", "desc", null, null, null, "admin-agent", null, null, null);
    
    assertThatThrownBy(() -> tools.deleteChannel("protected", false, "non-admin"))
        .hasMessageContaining("not in admin_instances");
}

@Test
void deleteChannel_withAdminList_acceptsAdmin() {
    createChannel("protected2", "desc", null, null, null, "admin-agent", null, null, null);
    String result = tools.deleteChannel("protected2", false, "admin-agent");
    assertThat(result).contains("deleted");
}
```

- [ ] **Step 4: Run tests and commit**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
git add -A
git commit -m "feat(mcp): add caller_instance_id + admin guard to delete_channel (#127)

Refs #127, Refs #119"
```

---

## Phase 4 — quarkus-qhorus #128: api/jpa module split

**Split content (quarkus-qhorus-api, no JPA, no Panache):**
- Store interfaces: all 6 blocking (`ChannelStore`, `MessageStore`, `InstanceStore`, `DataStore`, `WatchdogStore`, `CommitmentStore`) + 6 reactive
- SPI interfaces: `MessageTypePolicy`, `CommitmentAttestationPolicy`, `InstanceActorIdProvider`, `DefaultInstanceActorIdProvider`
- Enums: `ChannelSemantic`, `MessageType`, `CommitmentState`
- Domain POJO base classes: `Channel`, `Message`, `Commitment`, `Instance`, `SharedData`, `ArtefactClaim`, `Capability`, `Watchdog` (no JPA annotations — JPA entities in runtime will extend these)
- Query types: `DataQuery`, `MessageQuery` etc.

### Task 14: Create quarkus-qhorus-api Maven module

**Files:**
- Create: `api/pom.xml`
- Modify: root `pom.xml` (add `<module>api</module>` before runtime)

- [ ] **Step 1: Add api module to root pom**

In root `pom.xml`, change `<modules>` to:
```xml
<modules>
  <module>api</module>
  <module>runtime</module>
  <module>deployment</module>
  <module>testing</module>
  <module>examples</module>
</modules>
```

- [ ] **Step 2: Create api/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.quarkiverse.qhorus</groupId>
    <artifactId>quarkus-qhorus-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>quarkus-qhorus-api</artifactId>
  <name>Quarkus Qhorus - API</name>
  <description>Domain model, store SPI interfaces, and extension SPIs — no JPA, safe for any module to depend on</description>

  <dependencies>
    <!-- Mutiny for reactive store interfaces -->
    <dependency>
      <groupId>io.smallrye.reactive</groupId>
      <artifactId>mutiny</artifactId>
      <scope>provided</scope>
    </dependency>
    <!-- quarkus-ledger-api for MessageLedgerEntry base type -->
    <dependency>
      <groupId>io.quarkiverse.ledger</groupId>
      <artifactId>quarkus-ledger-api</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 3: Create directory structure**

```bash
mkdir -p ~/claude/quarkus-qhorus/api/src/main/java/io/quarkiverse/qhorus/api/{channel,message,instance,data,store,spi}
```

---

### Task 15: Move domain POJOs and store interfaces to api

- [ ] **Step 1: Create domain POJOs in api (no JPA)**

For each entity class in `runtime/.../channel/`, `runtime/.../message/`, `runtime/.../instance/`, `runtime/.../data/`:

Create a POJO version in `api/src/main/java/io/quarkiverse/qhorus/api/{package}/` with:
- Package `io.quarkiverse.qhorus.api.{channel|message|instance|data}`
- Copy all fields from the JPA entity, strip all JPA annotations
- Keep enums (`ChannelSemantic`, `MessageType`, `CommitmentState`, `CommitmentState`) in api

Example for `Channel`:
```java
// api/src/main/java/io/quarkiverse/qhorus/api/channel/Channel.java
package io.quarkiverse.qhorus.api.channel;

import java.time.Instant;
import java.util.UUID;

public class Channel {
    public UUID id;
    public String name;
    public String description;
    public ChannelSemantic semantic;
    public String barrierContributors;
    public boolean paused;
    public String allowedWriters;
    public String adminInstances;
    public Integer rateLimitPerChannel;
    public Integer rateLimitPerInstance;
    public String allowedTypes;
    public Instant createdAt;
}
```

Repeat for: `Message`, `Commitment`, `Instance`, `Capability`, `SharedData`, `ArtefactClaim`, `Watchdog`.

- [ ] **Step 2: Move store interfaces to api**

Move all 12 store interface files from `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/` to `api/src/main/java/io/quarkiverse/qhorus/api/store/`.

Change package to `io.quarkiverse.qhorus.api.store`.
Update all imports to use `io.quarkiverse.qhorus.api.*` types.

Keep the `jpa/` subdirectory (JPA implementations) in runtime — do not move those.

- [ ] **Step 3: Move SPI interfaces to api**

Move from `runtime/.../ledger/`:
- `InstanceActorIdProvider.java` → `api/.../spi/InstanceActorIdProvider.java`
- `DefaultInstanceActorIdProvider.java` → `api/.../spi/DefaultInstanceActorIdProvider.java`

Move from `runtime/.../message/`:
- `MessageTypePolicy.java` → `api/.../spi/MessageTypePolicy.java`
- `CommitmentAttestationPolicy.java` → `api/.../spi/CommitmentAttestationPolicy.java`
- `AttestationOutcome.java` → `api/.../spi/AttestationOutcome.java`

Update packages accordingly.

- [ ] **Step 4: Build api alone**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl api compile
```
Expected: BUILD SUCCESS (no JPA imports).

---

### Task 16: Update runtime to depend on api and extend api POJOs

**Files:**
- Modify: `runtime/pom.xml` (add api dependency)
- Modify: all JPA entity files (extend api POJO, strip duplicated fields)
- Modify: all services, MCP tools (update imports)

- [ ] **Step 1: Add api dependency to runtime pom**

```xml
<dependency>
  <groupId>io.quarkiverse.qhorus</groupId>
  <artifactId>quarkus-qhorus-api</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Update JPA entities to extend api POJOs**

For each entity (Channel, Message, Commitment, Instance, SharedData, ArtefactClaim, Capability, Watchdog):

1. Add `extends io.quarkiverse.qhorus.api.{package}.{ClassName}` to the class declaration
2. Remove all fields that are now inherited from the api POJO (keep only JPA-specific superclass calls and @PrePersist/@PreUpdate hooks)
3. Keep all JPA annotations (`@Entity`, `@Table`, `@Column`, etc.) on the class and use `@AttributeOverride` where column name customization is needed

Example for Channel:
```java
@Entity
@Table(name = "channel", uniqueConstraints = @UniqueConstraint(name = "uq_channel_name", columnNames = "name"))
@AttributeOverride(name = "id", column = @Column(name = "id"))
public class Channel extends io.quarkiverse.qhorus.api.channel.Channel
        implements PanacheEntityBase {
    // No fields needed — all inherited from api.Channel
    // Keep only JPA lifecycle hooks:
    @PrePersist
    void prePersist() { ... }
}
```

- [ ] **Step 3: Update all imports throughout runtime**

Services, MCP tools, and store implementations that imported `io.quarkiverse.qhorus.runtime.{channel|message|instance|data}.*` should now import from `io.quarkiverse.qhorus.api.*` for domain types.

Store implementations import from `io.quarkiverse.qhorus.api.store.*`.

SPI references update to `io.quarkiverse.qhorus.api.spi.*`.

- [ ] **Step 4: Update testing module to depend on api, not runtime**

In `testing/pom.xml`:
- Change `quarkus-qhorus` dependency scope or replace with `quarkus-qhorus-api` for InMemory*Store interface implementations
- InMemory*Store classes now import store interfaces and domain types from api

- [ ] **Step 5: Build all modules**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Run all tests**

```bash
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```
Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "refactor: split into quarkus-qhorus-api and quarkus-qhorus (#128)

New module quarkus-qhorus-api: domain POJOs (Channel, Message, Commitment, Instance,
SharedData, ArtefactClaim, Watchdog, Capability), all store interfaces (blocking + reactive),
SPI interfaces (MessageTypePolicy, CommitmentAttestationPolicy, InstanceActorIdProvider).
JPA entities in runtime extend api POJOs. Testing module depends on api only.

Closes #128, Refs #119"
```

---

## Phase 5 — claudony bug fixes (#95 / #72)

### Task 17: Fix ClaudonyLedgerEventCapture exception swallowing (#95, #72)

**Files:**
- Modify: `~/claude/claudony/claudony-app/src/main/java/dev/claudony/ledger/ClaudonyLedgerEventCapture.java` (or equivalent path)

- [ ] **Step 1: Find the file**

```bash
find ~/claude/claudony -name "ClaudonyLedgerEventCapture.java" -path "*/main/*"
```

- [ ] **Step 2: Fix silent exception swallowing**

Read the current file. Find `catch (Exception e)` that only logs. Change to propagate:

```java
// Before:
} catch (Exception e) {
    log.error("Failed to capture ledger event", e);
    // silently continues
}

// After:
} catch (RuntimeException e) {
    throw e; // propagate so caller knows the write failed
} catch (Exception e) {
    throw new RuntimeException("Failed to capture ledger event", e);
}
```

- [ ] **Step 3: Fix nextSequenceNumber() race condition**

Read current implementation. Replace MAX() query + result+1 pattern with a database-level sequence or optimistic lock:

```java
// Before (racy):
Long max = entityManager.createQuery("SELECT MAX(e.sequenceNumber) FROM ClaudonyLedgerEvent e WHERE e.caseId = :caseId", Long.class)
    .setParameter("caseId", caseId)
    .getSingleResult();
return (max == null ? 0L : max) + 1;

// After (safe — use SELECT FOR UPDATE or INSERT with conflict handling):
// Option A: rely on DB sequence column (if schema supports it)
// Option B: use @Version optimistic locking on the entity
// Option C: use NEXTVAL from a named sequence
// Check what casehub-engine does for the same concern and mirror it exactly.
```

Read `casehub-engine` equivalent implementation and match it. The fix from `casehub-engine` is authoritative.

- [ ] **Step 4: Run claudony tests**

```bash
cd ~/claude/claudony && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```

- [ ] **Step 5: Commit**

```bash
cd ~/claude/claudony
git add -A
git commit -m "fix: ClaudonyLedgerEventCapture exception propagation and sequence race condition (#95)

Remove silent catch that swallowed DB failures. Fix nextSequenceNumber() to use
safe concurrent strategy matching casehub-engine implementation.

Closes #95, Refs #72"
```

---

## Phase 6 — Update all consumers to new module APIs

### Task 18: Update claudony to new quarkus-qhorus and quarkus-ledger apis

**Files:**
- Modify: `~/claude/claudony/claudony-casehub/pom.xml`
- Modify: `~/claude/claudony/claudony-app/pom.xml`
- Modify: Java source files that import from old package paths

- [ ] **Step 1: Update pom.xml dependencies**

In `claudony-casehub/pom.xml` and `claudony-app/pom.xml`:

Add `quarkus-qhorus-api` as a dependency (or update `quarkus-qhorus` to also pull api transitively — check whether the runtime pom exports api).

Add `quarkus-ledger-api` dependency where ledger SPIs are used.

- [ ] **Step 2: Update Java imports**

```bash
grep -rn "import io.quarkiverse.qhorus.runtime" ~/claude/claudony --include="*.java" -l
grep -rn "import io.quarkiverse.ledger.runtime" ~/claude/claudony --include="*.java" -l
```

For each file:
- `io.quarkiverse.qhorus.runtime.channel.ChannelSemantic` → `io.quarkiverse.qhorus.api.channel.ChannelSemantic`
- `io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools` stays in runtime — no change
- `io.quarkiverse.ledger.runtime.model.*` → `io.quarkiverse.ledger.api.model.*`
- `io.quarkiverse.ledger.runtime.service.LedgerTraceIdProvider` → `io.quarkiverse.ledger.api.spi.LedgerTraceIdProvider`

- [ ] **Step 3: Update test files**

Same import updates for test files.

- [ ] **Step 4: Build claudony**

```bash
cd ~/claude/claudony && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```

- [ ] **Step 5: Commit claudony**

```bash
cd ~/claude/claudony
git add -A
git commit -m "refactor: update imports for quarkus-qhorus-api and quarkus-ledger-api module split

Refs casehubio/qhorus#128, casehubio/ledger#73"
```

---

### Task 19: Update casehub-engine to new module APIs

**Files:**
- Modify: relevant pom.xml files in casehub-engine
- Modify: Java source files with old import paths
- Remove: `NoOpLedgerEntryRepository` workarounds (if they exist)

- [ ] **Step 1: Find casehub-engine path**

Ask user for the path, or check `~/claude/casehub/engine` or `~/claude/casehub-poc`.

- [ ] **Step 2: Update pom.xml**

Replace `quarkus-ledger` with `quarkus-ledger-api` in modules that only need SPIs (not JPA persistence). Keep `quarkus-ledger` (full) only in modules that need actual JPA persistence.

- [ ] **Step 3: Remove NoOpLedgerEntryRepository workarounds**

```bash
find ~/claude/casehub/engine -name "NoOpLedgerEntryRepository.java" -path "*/test/*"
```

If found, delete them — they were only needed because importing quarkus-ledger pulled in JPA. With `quarkus-ledger-api`, this is no longer needed.

- [ ] **Step 4: Update imports**

Same pattern as claudony — update to api package paths.

- [ ] **Step 5: Build and test**

```bash
cd ~/claude/casehub/engine && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```

- [ ] **Step 6: Commit casehub-engine**

```bash
cd ~/claude/casehub/engine
git add -A
git commit -m "refactor: depend on quarkus-ledger-api and quarkus-qhorus-api for SPI-only consumers

Eliminates NoOpLedgerEntryRepository workarounds — api modules carry no JPA requirement.

Refs casehubio/ledger#73, casehubio/qhorus#128"
```

---

## Phase 7 — Install snapshots and close issues

### Task 20: Install all snapshots to local Maven repo (not deploy)

- [ ] **Step 1: Install in dependency order**

```bash
# Install quarkus-ledger (api + runtime)
cd ~/claude/quarkus-ledger && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install

# Install quarkus-qhorus (api + runtime + testing)
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install

# Install quarkus-work
cd ~/claude/quarkus-work && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install

# Install claudony
cd ~/claude/claudony && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install

# Build casehub-engine
cd ~/claude/casehub/engine && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: all BUILD SUCCESS.

- [ ] **Step 2: Deploy to GitHub Packages when all pass**

Only run deploy when the user confirms all builds pass locally:
```bash
cd ~/claude/quarkus-ledger && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn deploy -DskipTests
cd ~/claude/quarkus-qhorus && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn deploy -DskipTests
```

---

### Task 21: Close GitHub issues

- [ ] **Close #121** with comment: "All breaking items (A, C, D, G, H) implemented. B decided no-change."
- [ ] **Close #127** with commit reference
- [ ] **Close #128** with commit reference
- [ ] **Close quarkus-ledger #73** with commit reference
- [ ] **Close quarkus-ledger #71** with commit reference
- [ ] **Close quarkus-work #70** with commit reference
- [ ] **Close claudony #95** with commit reference

```bash
gh issue close 121 --repo casehubio/qhorus --comment "All breaking changes shipped: A (artefact rename), C (auto-claim), D (3-step chunked upload), G (observer consolidation), H (list_pending_commitments). B closed as no-change (semantic split preserved). Closes #119 MCP consistency epic."
gh issue close 119 --repo casehubio/qhorus --comment "All planned consistency changes shipped as part of platform-wide breaking change window."
gh issue close 127 --repo casehubio/qhorus --comment "delete_channel admin guard implemented."
gh issue close 128 --repo casehubio/qhorus --comment "quarkus-qhorus-api module created."
gh issue close 73 --repo casehubio/ledger --comment "quarkus-ledger-api module created."
gh issue close 71 --repo casehubio/ledger --comment "Duplicate V1001 migration resolved."
gh issue close 70 --repo casehubio/ledger --comment "quarkus-work migrated to TrustGateService."
gh issue close 95 --repo casehubio/claudony --comment "Exception propagation and sequence race condition fixed."
```

---

## Self-Review

**Spec coverage check:**

| Requirement | Task |
|---|---|
| quarkus-ledger #73 module split | Tasks 1–4 |
| quarkus-work #71 duplicate migration | Task 5 |
| quarkus-work #72 JSON + null + test fixes | Task 6 |
| quarkus-work #70 TrustGateService migration | Task 7 |
| qhorus #121-A artefact rename | Task 8 |
| qhorus #121-H commitment consolidation | Task 9 |
| qhorus #121-G observer consolidation | Task 10 |
| qhorus #121-D chunked upload | Task 11 |
| qhorus #121-C auto-claim/release | Task 12 |
| qhorus #127 delete_channel admin guard | Task 13 |
| qhorus #128 module split | Tasks 14–16 |
| claudony #95 bug fixes | Task 17 |
| claudony import updates | Task 18 |
| casehub-engine import updates | Task 19 |
| Install + deploy + close issues | Tasks 20–21 |

**Notes for implementer:**
- Task 16 (qhorus JPA entity extension) is the most complex step — JPA @AttributeOverride can be tricky. If mapping problems arise, use `@MappedSuperclass` on the api POJO instead of plain POJO (requires api to accept jakarta.persistence as a provided dep).
- Task 12 (auto-claim) assumes `instanceStore.findByInstanceId(String)` exists — verify this method name matches the actual InstanceStore interface.
- For casehub-engine path, ask user before Task 19.
- All `mvn` commands require `JAVA_HOME=$(/usr/libexec/java_home -v 26)` prefix on this machine.
