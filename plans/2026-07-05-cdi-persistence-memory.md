# CDI Persistence-Memory Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #321 — CDI ambiguity: persistence-memory stores need fixing
**Issue group:** #321, #320

**Goal:** Eliminate CDI ambiguity between testing/ and persistence-memory/ InMemory stores, remove the redundant CrossTenantProducer pattern, and align ARC42STORIES.MD with current code.

**Architecture:** Delete duplicate InMemory stores from testing/ (keeping only test utilities). Eliminate `CrossTenantProducer`, `@CrossTenant` qualifier, `QhorusSystemCurrentPrincipal`, and `@QhorusSystem` — consumers inject `CrossTenant*Store` interfaces directly via CDI type resolution. Update properties files, protocols, and documentation to reflect the new structure.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (ArC), Maven

## Global Constraints

- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All modules must compile and pass tests after each task
- `persistence-memory/` stores keep `@Alternative @Priority(1)` — no `@DefaultBean` (platform convention)
- Cross-repo consumer migration is out of scope — file issues only
- ARC42STORIES.MD lives in workspace at `/Users/mdproctor/claude/public/casehub/qhorus/ARC42STORIES.MD`
- Project repo is `/Users/mdproctor/claude/casehub/qhorus`
- Protocols live at `docs/protocols/casehub/` in the project repo

---

### Task 1: Delete Duplicate InMemory Stores from testing/

**Files:**
- Delete (18 main sources): `testing/src/main/java/io/casehub/qhorus/testing/InMemory*.java` (all InMemory store files — NOT `RecordingChannelBackend.java`, NOT `MessageLedgerEntryTestFactory.java`)
- Delete (22 test sources): all `testing/src/test/java/io/casehub/qhorus/testing/InMemory*.java` and all `testing/src/test/java/io/casehub/qhorus/testing/contract/*.java`
- Retain: `testing/src/test/java/io/casehub/qhorus/runtime/message/CommitmentServiceTest.java`
- Modify: `testing/src/test/java/io/casehub/qhorus/runtime/message/CommitmentServiceTest.java` — update import
- Modify: `examples/agent-communication/src/main/resources/application.properties` — lines 25-30
- Modify: `slack-channel/src/test/resources/application.properties` — lines 5-17

**Interfaces:**
- Consumes: `casehub-qhorus-persistence-memory` (testing already depends on it in pom.xml)
- Produces: Clean CDI graph — one alternative per store interface, no ambiguity

- [ ] **Step 1: Delete duplicate InMemory store sources from testing/src/main/java/**

Delete these 18 files (keep `gateway/RecordingChannelBackend.java` and `MessageLedgerEntryTestFactory.java`):

```
testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryCommitmentStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantChannelStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantCommitmentStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantMessageStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantWatchdogStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryDataStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryDeliveryCursorStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryInstanceStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveCommitmentStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveDataStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveInstanceStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveWatchdogStore.java
testing/src/main/java/io/casehub/qhorus/testing/InMemoryWatchdogStore.java
```

- [ ] **Step 2: Delete duplicate test files from testing/src/test/java/**

Delete these 22 test files (keep `CommitmentServiceTest.java`):

```
testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelBindingStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryCommitmentStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryDataStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryInstanceStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryMessageStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryReactiveCommitmentStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryReactiveDataStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryReactiveInstanceStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryReactiveWatchdogStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryStoresDualInterfaceTest.java
testing/src/test/java/io/casehub/qhorus/testing/InMemoryWatchdogStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelBindingStoreContractTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelStoreContractTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/CommitmentStoreContractTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/DataStoreContractTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/DeliveryCursorStoreContractTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/InMemoryDeliveryCursorStoreTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/InstanceStoreContractTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/MessageStoreContractTest.java
testing/src/test/java/io/casehub/qhorus/testing/contract/WatchdogStoreContractTest.java
```

- [ ] **Step 3: Update CommitmentServiceTest import**

In `testing/src/test/java/io/casehub/qhorus/runtime/message/CommitmentServiceTest.java`, change any import of `io.casehub.qhorus.testing.InMemoryCommitmentStore` to `io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore`. If the test imports other InMemory stores from the testing package, update those too.

- [ ] **Step 4: Update examples/agent-communication properties**

In `examples/agent-communication/src/main/resources/application.properties`, replace lines 25-30:

Change every `io.casehub.qhorus.testing.` prefix to `io.casehub.qhorus.persistence.memory.`:
```
quarkus.arc.selected-alternatives=\
  io.casehub.qhorus.persistence.memory.InMemoryChannelStore,\
  io.casehub.qhorus.persistence.memory.InMemoryMessageStore,\
  io.casehub.qhorus.persistence.memory.InMemoryInstanceStore,\
  io.casehub.qhorus.persistence.memory.InMemoryDataStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore
```

- [ ] **Step 5: Update slack-channel test properties**

In `slack-channel/src/test/resources/application.properties`, replace lines 5-17:

Change every `io.casehub.qhorus.testing.` prefix to `io.casehub.qhorus.persistence.memory.`:
```
quarkus.arc.selected-alternatives=\
  io.casehub.qhorus.persistence.memory.InMemoryChannelStore,\
  io.casehub.qhorus.persistence.memory.InMemoryMessageStore,\
  io.casehub.qhorus.persistence.memory.InMemoryInstanceStore,\
  io.casehub.qhorus.persistence.memory.InMemoryDataStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore,\
  io.casehub.qhorus.persistence.memory.InMemoryWatchdogStore,\
  io.casehub.qhorus.persistence.memory.InMemoryDeliveryCursorStore,\
  io.casehub.qhorus.persistence.memory.InMemoryChannelBindingStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantChannelStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantMessageStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantCommitmentStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantWatchdogStore
```

- [ ] **Step 6: Build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`

Expected: BUILD SUCCESS — all modules compile, all tests pass, no CDI ambiguity errors.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#321): delete duplicate InMemory stores from testing module

Remove 18 duplicate InMemory store classes from casehub-qhorus-testing
that were copied (not moved) during the persistence-memory extraction.
Update selected-alternatives in agent-communication and slack-channel
properties to reference casehub-qhorus-persistence-memory.

The testing module now contains only test utilities:
RecordingChannelBackend, MessageLedgerEntryTestFactory.

Refs #321"
```

---

### Task 2: Eliminate CrossTenantProducer and @CrossTenant

**Files:**
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/identity/CrossTenantProducer.java`
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/identity/QhorusSystemCurrentPrincipal.java`
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/qualifier/QhorusSystem.java`
- Delete: `runtime/src/test/java/io/casehub/qhorus/identity/CrossTenantProducerTest.java`
- Delete: `api/src/main/java/io/casehub/qhorus/api/qualifier/CrossTenant.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` — lines 23, 52
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java` — lines 20, 34, 35
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryService.java` — lines 28, 84
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — lines 25, 57-58
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java` — lines 17, 64-77
- Modify: 4 persistence-memory CrossTenant stores — remove `@CrossTenant` annotation and import

**Interfaces:**
- Consumes: Task 1 completed (no duplicate stores)
- Produces: Clean injection — `@Inject CrossTenantChannelStore` resolves via CDI type, no qualifier

- [ ] **Step 1: Delete CrossTenantProducer and supporting classes**

Delete these 5 files:
```
api/src/main/java/io/casehub/qhorus/api/qualifier/CrossTenant.java
runtime/src/main/java/io/casehub/qhorus/runtime/identity/CrossTenantProducer.java
runtime/src/main/java/io/casehub/qhorus/runtime/identity/QhorusSystemCurrentPrincipal.java
runtime/src/main/java/io/casehub/qhorus/runtime/qualifier/QhorusSystem.java
runtime/src/test/java/io/casehub/qhorus/identity/CrossTenantProducerTest.java
```

Delete the now-empty `api/src/main/java/io/casehub/qhorus/api/qualifier/` directory.

- [ ] **Step 2: Remove @CrossTenant from persistence-memory CrossTenant stores**

In each of these 4 files, remove the `@CrossTenant` annotation and its import (`import io.casehub.qhorus.api.qualifier.CrossTenant`):

1. `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantChannelStore.java` — line 25 (`@CrossTenant`), import line
2. `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantCommitmentStore.java` — line 26 (`@CrossTenant`), import line
3. `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantMessageStore.java` — line 27 (`@CrossTenant`), import line
4. `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantWatchdogStore.java` — line 23 (`@CrossTenant`), import line

- [ ] **Step 3: Remove @CrossTenant from ChannelGateway**

In `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`:
- Delete line 23: `import io.casehub.qhorus.api.qualifier.CrossTenant;`
- Line 52: change `@CrossTenant CrossTenantChannelStore crossTenantChannelStore,` to `CrossTenantChannelStore crossTenantChannelStore,`

- [ ] **Step 4: Remove @CrossTenant from DeliveryBatchExecutor**

In `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java`:
- Delete line 20: `import io.casehub.qhorus.api.qualifier.CrossTenant;`
- Line 34: change `@Inject @CrossTenant CrossTenantMessageStore messageStore;` to `@Inject CrossTenantMessageStore messageStore;`
- Line 35: change `@Inject @CrossTenant CrossTenantChannelStore channelStore;` to `@Inject CrossTenantChannelStore channelStore;`
- Also check the constructor (lines 41-42) for `@CrossTenant` params and remove if present

- [ ] **Step 5: Remove @CrossTenant from DeliveryService**

In `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryService.java`:
- Delete line 28: `import io.casehub.qhorus.api.qualifier.CrossTenant;`
- Line 84: change `@CrossTenant CrossTenantMessageStore messageStore,` to `CrossTenantMessageStore messageStore,`

- [ ] **Step 6: Remove @CrossTenant from MessageService**

In `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`:
- Delete line 25: `import io.casehub.qhorus.api.qualifier.CrossTenant;`
- Lines 57-58: change `@Inject @CrossTenant\n    CrossTenantChannelStore crossTenantChannelStore;` to `@Inject\n    CrossTenantChannelStore crossTenantChannelStore;`

- [ ] **Step 7: Remove @CrossTenant from WatchdogEvaluationService**

In `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`:
- Delete line 17: `import io.casehub.qhorus.api.qualifier.CrossTenant;`
- Lines 64-65: remove `@CrossTenant` from `CrossTenantChannelStore` injection
- Lines 68-69: remove `@CrossTenant` from `CrossTenantMessageStore` injection
- Lines 72-73: remove `@CrossTenant` from `CrossTenantCommitmentStore` injection
- Lines 76-77: remove `@CrossTenant` from `CrossTenantWatchdogStore` injection

- [ ] **Step 8: Build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`

Expected: BUILD SUCCESS — all modules compile, all tests pass. CrossTenant stores now resolved by type alone.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#321): eliminate CrossTenantProducer and @CrossTenant qualifier

Delete CrossTenantProducer, QhorusSystemCurrentPrincipal, @QhorusSystem,
and @CrossTenant annotation. The qualifier was redundant — CrossTenant*Store
interfaces are distinct types that CDI resolves without a qualifier. The
producer's admin assertion was tautological (hardcoded true).

Runtime injection points now use plain @Inject CrossTenantChannelStore
(no qualifier). JPA stores satisfy in production; InMemory @Alternative
@Priority(1) stores win in tests.

Refs #321"
```

---

### Task 3: Documentation Updates (B1-B12)

**Files:**
- Modify: `/Users/mdproctor/claude/public/casehub/qhorus/ARC42STORIES.MD` — §4, §5, §6, §8, §11, §13
- Modify: `docs/protocols/casehub/scheduled-service-cross-tenant-stores.md` (PP-20260609-67996e)
- Modify: `docs/protocols/casehub/reactive-inmemory-store-selected-alternatives.md` (PP-20260618-131cdf)
- Modify: `docs/protocols/casehub/inmemory-store-no-entity-mutation-in-session.md` (PP-20260618-100368)
- Modify: `docs/protocols/casehub/INDEX.md` — update protocol titles/references
- Modify: `docs/DESIGN.md` — lines 31, 221, 238, 252
- Modify: `CLAUDE.md` — line 355

**Interfaces:**
- Consumes: Tasks 1-2 completed (code matches docs)
- Produces: All documentation consistent with new code structure

- [ ] **Step 1: Update ARC42STORIES.MD §5 module structure**

In `/Users/mdproctor/claude/public/casehub/qhorus/ARC42STORIES.MD`, find the module structure code block after `## §5 Building Block View`.

Add `persistence-memory/` entry:
```
├── persistence-memory/         InMemory*Store and InMemoryReactive*Store (@Alternative @Priority(1)); zero-config ephemeral installs and test isolation
```

Update `testing/` entry:
```
├── testing/                Test utilities — RecordingChannelBackend, MessageLedgerEntryTestFactory
```

- [ ] **Step 2: Update ARC42STORIES.MD §5 api module table**

Replace the existing 5-row api module table with the complete 9-row table (omit `api.qualifier` — empty after deletion):

| Package | Contents |
|---|---|
| `api.channel` | `Channel`, `ChannelDetail`, `ChannelManager`, `ReactiveChannelManager`, `FindOrCreateResult` |
| `api.data` | `SharedData`, `ArtefactClaim` domain types |
| `api.gateway` | `ChannelBackend` hierarchy, `MessageObserver` SPI, `MessageReceivedEvent`, `ChannelInitialisedEvent`, inbound/outbound records |
| `api.instance` | `InstanceInfo` |
| `api.message` | `MessageResult`, `MessageType`, `MessageDispatcher`, `ReactiveMessageDispatcher`, `CommitmentState`, `CommitmentDeclinedEvent`, `CommitmentExpiredEvent` |
| `api.spi` | `CommitmentAttestationPolicy`, `CommitmentContext`, `ObligorTrustPolicy`, `ObligorTrustContext`, `InstanceActorIdProvider` |
| `api.store` | `ChannelStore`, `MessageStore`, `CommitmentStore`, `InstanceStore`, `DataStore`, `WatchdogStore`, `DeliveryCursorStore`, `ChannelBindingStore` + Reactive + CrossTenant variants |
| `api.store.query` | `ChannelQuery`, `DataQuery`, `InstanceQuery`, `MessageQuery`, `WatchdogQuery` |
| `api.watchdog` | `Watchdog`, `WatchdogConditionType`, `WatchdogAlertRouter` SPI, `WatchdogAlertEvent`, alert context records |

- [ ] **Step 3: Update ARC42STORIES.MD §4, §5 runtime, §6, §8, §11, §13**

For each section, apply the changes specified in the design spec B3-B8:
- **§4**: Change "InMemory*Store from `casehub-qhorus-testing`" to "from `casehub-qhorus-persistence-memory`"
- **§5 runtime table**: Remove `CrossTenantProducer` from the `runtime.identity` row
- **§6 Scenario 3**: Remove `@CrossTenant` / `CrossTenantProducer` runtime view. Background services inject `CrossTenant*Store` directly
- **§8**: Remove `CrossTenantProducer` and `QhorusSystemCurrentPrincipal` from narrative
- **§11**: Update tenant isolation row — remove `isCrossTenantAdmin()` reference
- **§13**: Update `InMemoryStore` glossary entry — change `casehub-qhorus-testing` to `casehub-qhorus-persistence-memory`

Read each section first with `grep -n` to find exact lines before editing.

- [ ] **Step 4: Update protocol PP-20260609-67996e**

In `docs/protocols/casehub/scheduled-service-cross-tenant-stores.md`:
- Title: remove `@CrossTenant` reference — becomes "must use `CrossTenant*Store` interfaces and explicit tenancyId"
- Body: replace "@CrossTenant-qualified stores (produced by CrossTenantProducer)" with "CrossTenant*Store interfaces directly"
- `violation_hint`: update to reference direct injection pattern
- Update `docs/protocols/casehub/INDEX.md` entry to match new title

- [ ] **Step 5: Update protocol PP-20260618-131cdf**

In `docs/protocols/casehub/reactive-inmemory-store-selected-alternatives.md`:
- `applies_to`: change `casehub-qhorus-testing` to `casehub-qhorus-persistence-memory`
- Body: change "InMemoryReactive*Store beans from `casehub-qhorus-testing`" to "from `casehub-qhorus-persistence-memory`"
- Update INDEX.md entry

- [ ] **Step 6: Update protocol PP-20260618-100368**

In `docs/protocols/casehub/inmemory-store-no-entity-mutation-in-session.md`:
- `applies_to`: change `casehub-qhorus-testing` to `casehub-qhorus-persistence-memory`
- Update INDEX.md entry

- [ ] **Step 7: Update docs/DESIGN.md**

In `docs/DESIGN.md`, update 4 references (lines 31, 221, 238, 252):
- Change `casehub-qhorus-testing` to `casehub-qhorus-persistence-memory` where the reference is about InMemory stores
- Leave references to `casehub-qhorus-testing` that correctly refer to retained utilities (RecordingChannelBackend, MessageLedgerEntryTestFactory)

- [ ] **Step 8: Update CLAUDE.md**

In `CLAUDE.md`, line 355:
- Change "adds `casehub-qhorus-testing` (or `casehub-qhorus-persistence-memory` directly)" to "adds `casehub-qhorus-persistence-memory`"
- Lines 367, 386 reference RecordingChannelBackend and MessageLedgerEntryTestFactory in testing — these are correct, leave them

- [ ] **Step 9: Commit documentation to both repos**

Project repo (protocols, DESIGN.md, CLAUDE.md):
```bash
git -C /Users/mdproctor/claude/casehub/qhorus add docs/protocols/ docs/DESIGN.md CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#321,#320): update protocols, DESIGN.md, CLAUDE.md for persistence-memory cleanup

Update 3 protocols to reference casehub-qhorus-persistence-memory
instead of casehub-qhorus-testing. Remove @CrossTenant references from
scheduled-service-cross-tenant-stores protocol. Update DESIGN.md and
CLAUDE.md InMemory store module references.

Refs #321, #320"
```

Workspace repo (ARC42STORIES.MD):
```bash
git -C /Users/mdproctor/claude/public/casehub/qhorus add ARC42STORIES.MD
git -C /Users/mdproctor/claude/public/casehub/qhorus commit -m "docs(#320): align ARC42STORIES.MD with current api/ and module structure

Update §4, §5, §6, §8, §11, §13: add persistence-memory/ module, expand
api module table from 5 to 9 packages, remove CrossTenantProducer references,
update InMemory store module attribution.

Refs #320"
```

---

### Task 4: Full Verification and Cross-Repo Issues

**Files:**
- No modifications — verification only

**Interfaces:**
- Consumes: Tasks 1-3 completed
- Produces: Confirmed green build, filed cross-repo issues

- [ ] **Step 1: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 2: Verify agent-communication compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -Pwith-llm-examples -f /Users/mdproctor/claude/casehub/qhorus/examples/agent-communication/pom.xml`

Expected: BUILD SUCCESS (profile-gated module, not exercised by standard build).

- [ ] **Step 3: Documentation sweep**

Search all `.md` files for stale references:
```bash
grep -rn "casehub-qhorus-testing" /Users/mdproctor/claude/casehub/qhorus/ --include="*.md"
```

Verify every remaining reference correctly refers to retained utilities (RecordingChannelBackend, MessageLedgerEntryTestFactory, CommitmentServiceTest) — not to InMemory stores.

Also verify:
```bash
grep -rn "@CrossTenant" /Users/mdproctor/claude/casehub/qhorus/ --include="*.java"
```

Expected: zero matches (all `@CrossTenant` annotations removed).

- [ ] **Step 4: File cross-repo migration issues**

Create GitHub issues for each consumer that needs mechanical migration:

1. **claudony** — 9 test files: change `import io.casehub.qhorus.testing.InMemory*` to `io.casehub.qhorus.persistence.memory.InMemory*`
2. **devtown** — remove ghost `exclude-types` for `io.casehub.qhorus.testing.InMemory*`, remove `CrossTenantProducer` exclude, remove `persistence-memory` excludes from production properties
3. **openclaw** — 1 test file: same import change as claudony

```bash
gh issue create --repo casehubio/claudony --title "chore: update qhorus InMemory store imports after persistence-memory cleanup" --body "casehubio/qhorus#321 removed duplicate InMemory stores from casehub-qhorus-testing. 9 test files import from the old package. Mechanical change: io.casehub.qhorus.testing.InMemory* → io.casehub.qhorus.persistence.memory.InMemory*"
gh issue create --repo casehubio/devtown --title "chore: clean up qhorus exclude-types after persistence-memory cleanup" --body "casehubio/qhorus#321 eliminated CDI ambiguity and CrossTenantProducer. Remove ghost exclude-types for io.casehub.qhorus.testing.InMemory*, CrossTenantProducer, and persistence-memory excludes from production properties."
gh issue create --repo casehubio/openclaw --title "chore: update qhorus InMemory store import after persistence-memory cleanup" --body "casehubio/qhorus#321 removed duplicate InMemory stores from casehub-qhorus-testing. 1 test file imports from old package. Mechanical: io.casehub.qhorus.testing.InMemory* → io.casehub.qhorus.persistence.memory.InMemory*"
```

- [ ] **Step 5: Update issue #321 title**

The original issue title says "need @DefaultBean for test displacement" — this is the wrong fix. Update:

```bash
gh issue edit 321 --repo casehubio/qhorus --title "chore: CDI ambiguity — duplicate InMemory stores in testing/ and redundant CrossTenantProducer"
```
