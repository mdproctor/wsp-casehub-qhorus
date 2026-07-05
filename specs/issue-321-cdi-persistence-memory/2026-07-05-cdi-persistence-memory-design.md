# CDI Persistence-Memory Cleanup + ARC42STORIES.MD Alignment

**Issues:** qhorus#321, qhorus#320
**Date:** 2026-07-05

---

## Problem

Consumers of `casehub-qhorus` (devtown, claudony, openclaw, and others) hit `AmbiguousResolutionException` when running `@QuarkusTest`. The issue has two root causes and a secondary docs drift.

### Root Cause 1 — Duplicate InMemory Stores

The `persistence-memory` extraction (specs/2026-06-30) was intended to MOVE InMemory store implementations from `testing/` to `persistence-memory/`. Instead they were COPIED. Both modules now ship identical `@Alternative @Priority(1)` beans for every store interface. Since `testing` depends on `persistence-memory`, both sets are always on the classpath — CDI sees two alternatives at the same priority for each injection point.

### Root Cause 2 — CrossTenantProducer Concrete-Type Injection

`CrossTenantProducer` injects JPA cross-tenant stores by concrete class (`JpaCrossTenantChannelStore`, etc.), then re-exports them as `@CrossTenant`-qualified interface beans. Even when InMemory alternatives displace the producer's products, CDI still instantiates the producer and tries to satisfy its JPA injection points. In test scenarios without a datasource, this fails.

The producer pattern exists solely to enforce a `@CrossTenant` CDI qualifier that is architecturally redundant (see Analysis below) and an admin assertion that is tautological (`QhorusSystemCurrentPrincipal.isCrossTenantAdmin()` is hardcoded `return true`).

### Secondary — ARC42STORIES.MD §5 Stale (#320)

The api module table in §5 lists 5 packages; 10 actually exist. The module structure section omits `persistence-memory/` and incorrectly describes `testing/` as containing InMemory stores.

---

## Analysis

### Why `@CrossTenant` qualifier is redundant

CDI qualifiers exist to disambiguate beans of the SAME TYPE. `CrossTenantChannelStore` and `ChannelStore` are completely separate interfaces — no shared type hierarchy. The type itself is the distinction. A qualifier on a unique type adds zero discriminating information.

The qualifier forces the `CrossTenantProducer` pattern to exist (someone must bridge unqualified JPA beans to qualified injection points). Removing the qualifier eliminates the producer, which eliminates the concrete-type injection problem.

### Why the admin assertion is a tautology

`CrossTenantProducer` calls `assertCrossTenantAdmin()` which checks `QhorusSystemCurrentPrincipal.isCrossTenantAdmin()`. This method is hardcoded to `return true`. The check can never fail. It exists as a "startup sanity check" but validates a constant.

### Why `@DefaultBean` on InMemory stores is wrong

Issue #321 proposes adding `@DefaultBean` to persistence-memory stores. PLATFORM.md explicitly prohibits this: *"Anti-pattern: labelling an `InMemoryXxx` as `@DefaultBean` — `@DefaultBean` means no-op, not in-memory."* The platform convention reserves `@DefaultBean` for stub/no-op implementations. Working in-memory implementations use `@Alternative @Priority(N)`.

### Engine's `@CrossTenant` is independent

`casehub-engine` has its own `@CrossTenant` qualifier at `io.casehub.engine.common.qualifier.CrossTenant` — completely independent of qhorus's `io.casehub.qhorus.api.qualifier.CrossTenant`. Changes to qhorus's qualifier have zero impact on engine.

---

## Design

### A1. Delete duplicate InMemory stores from `testing/`

Delete all store classes from `testing/src/main/java/io/casehub/qhorus/testing/`:

**Blocking stores (8):**
- `InMemoryChannelStore`, `InMemoryMessageStore`, `InMemoryCommitmentStore`
- `InMemoryInstanceStore`, `InMemoryDataStore`, `InMemoryWatchdogStore`
- `InMemoryChannelBindingStore`, `InMemoryDeliveryCursorStore`

**Reactive stores (6):**
- `InMemoryReactiveChannelStore`, `InMemoryReactiveMessageStore`
- `InMemoryReactiveCommitmentStore`, `InMemoryReactiveInstanceStore`
- `InMemoryReactiveDataStore`, `InMemoryReactiveWatchdogStore`

**CrossTenant stores (4):**
- `InMemoryCrossTenantChannelStore`, `InMemoryCrossTenantCommitmentStore`
- `InMemoryCrossTenantMessageStore`, `InMemoryCrossTenantWatchdogStore`

**Test classes to delete from `testing/src/test/java/`:**
- All `InMemory*StoreTest` and `InMemoryReactive*StoreTest` classes
- `InMemoryStoresDualInterfaceTest` (the one in persistence-memory stays)

**Retained in `testing/`:**
- `RecordingChannelBackend` (gateway test double)
- `MessageLedgerEntryTestFactory` (ledger test factory)
- `CommitmentServiceTest` (lives here to avoid module cycle)
- Any other non-store test utilities

### A2. Eliminate CrossTenantProducer and @CrossTenant from qhorus stores

**Delete:**
- `runtime/.../identity/CrossTenantProducer.java`
- `runtime/.../identity/QhorusSystemCurrentPrincipal.java`
- `runtime/.../qualifier/QhorusSystem.java`
- `runtime/src/test/.../identity/CrossTenantProducerTest.java`

**Remove `@CrossTenant` annotation from store beans:**
- `persistence-memory/InMemoryCrossTenantChannelStore` — remove `@CrossTenant`
- `persistence-memory/InMemoryCrossTenantCommitmentStore` — remove `@CrossTenant`
- `persistence-memory/InMemoryCrossTenantMessageStore` — remove `@CrossTenant`
- `persistence-memory/InMemoryCrossTenantWatchdogStore` — remove `@CrossTenant`

**Update injection points — remove `@CrossTenant` qualifier:**

| File | Change |
|------|--------|
| `ChannelGateway.java` | `@CrossTenant CrossTenantChannelStore` → `CrossTenantChannelStore` |
| `DeliveryBatchExecutor.java` | Remove `@CrossTenant` from `CrossTenantMessageStore` and `CrossTenantChannelStore` |
| `DeliveryService.java` | Remove `@CrossTenant` from `CrossTenantMessageStore` |
| `MessageService.java` | Remove `@CrossTenant` from its cross-tenant injection |
| `WatchdogEvaluationService.java` | Remove `@CrossTenant` from all 4 cross-tenant store injections |

**Delete the annotation class and package:**
- `api/.../qualifier/CrossTenant.java` — verified no external consumer uses the qhorus `@CrossTenant` qualifier (claudony: 0 references; devtown, openclaw: application-tier consumers that interact via MCP tools and standard store interfaces, never via `@CrossTenant`-qualified injection). The annotation is dead after the producer and all qualified injection points are removed.
- `api/src/main/java/io/casehub/qhorus/api/qualifier/` — delete directory. `CrossTenant.java` is the sole file; the package is empty after deletion.

**Resolved TODO:** `QhorusSystemCurrentPrincipal.java` carries an interim marker: *"delete when casehub-platform ships a platform-level system-actor principal with isCrossTenantAdmin()=true."* This spec takes a different path — eliminating the need for a system-actor principal entirely. The cross-tenant stores already implement `CrossTenantChannelStore` (etc.) as distinct interfaces that ignore tenancy filtering at the query level. No system principal is needed to bypass tenancy; the store type itself determines the query behavior. The platform-level principal path is superseded.

### A3. Update `selected-alternatives` in properties files

**Within qhorus — change `io.casehub.qhorus.testing.*` → `io.casehub.qhorus.persistence.memory.*`:**

| File | Entries |
|------|---------|
| `examples/agent-communication/src/main/resources/application.properties` | 5 store references |
| `slack-channel/src/test/resources/application.properties` | 12 store references |

Files already using correct package (no change): `examples/normative-layout`, `examples/type-system`, `connector-backend`.

### B. ARC42STORIES.MD Alignment (#320)

**B1. Module structure (§5)** — add `persistence-memory/` module entry. Update `testing/` description: "Test utilities — RecordingChannelBackend, MessageLedgerEntryTestFactory" (no longer InMemory stores).

**B2. Api module table (§5)** — add missing packages:

| Package | Contents |
|---------|----------|
| `api.channel` | `Channel`, `ChannelDetail`, `ChannelManager`, `ReactiveChannelManager`, `FindOrCreateResult` |
| `api.data` | `SharedData`, `ArtefactClaim` domain types |
| `api.gateway` | `ChannelBackend` hierarchy, `MessageObserver`, `MessageReceivedEvent`, `ChannelInitialisedEvent`, inbound/outbound records |
| `api.instance` | `InstanceInfo` |
| `api.message` | `MessageResult`, `MessageType`, `MessageDispatcher`, `ReactiveMessageDispatcher`, `CommitmentState`, `CommitmentDeclinedEvent`, `CommitmentExpiredEvent` |
| `api.spi` | `CommitmentAttestationPolicy`, `CommitmentContext`, `ObligorTrustPolicy`, `ObligorTrustContext`, `InstanceActorIdProvider` |
| `api.store` | `ChannelStore`, `MessageStore`, `CommitmentStore`, `InstanceStore`, `DataStore`, `WatchdogStore`, `DeliveryCursorStore`, `ChannelBindingStore` + Reactive variants + CrossTenant variants |
| `api.store.query` | `ChannelQuery`, `DataQuery`, `InstanceQuery`, `MessageQuery`, `WatchdogQuery` |
| `api.watchdog` | `Watchdog`, `WatchdogConditionType`, `WatchdogAlertRouter` SPI, `WatchdogAlertEvent`, alert context records (`AlertContext`, `AgentStaleContext`, `BarrierStuckContext`, `ChannelIdleContext`, `QueueDepthContext`, `ApprovalPendingContext`, `AlertDeliveryTarget`) |

**B3. §4 Solution Strategy** — update the `*Store` persistence seam paragraph: change "InMemory*Store from `casehub-qhorus-testing`" to "InMemory*Store from `casehub-qhorus-persistence-memory`".

**B4. §5 runtime module table** — remove `CrossTenantProducer` from the `runtime.identity` row.

**B5. §6 Scenario 3 (Multi-tenancy enforcement)** — remove the `@CrossTenant` / `CrossTenantProducer` / `QhorusSystemCurrentPrincipal` runtime view. Background services inject `CrossTenant*Store` directly (unqualified).

**B6. §8 Chapter 8 (Multi-Tenancy)** — remove `CrossTenantProducer` and `QhorusSystemCurrentPrincipal` from the narrative. Update: background services inject `CrossTenant*Store` interfaces directly; tenancy bypass is a property of the store type, not a CDI qualifier or system principal.

**B7. §11 Quality Requirements** — update Tenant isolation row: "Background services inject `CrossTenant*Store` interfaces directly — tenancy bypass is a property of the store interface contract, not a runtime qualifier." Remove reference to `isCrossTenantAdmin()` assertion.

**B8. §13 Glossary** — update `InMemoryStore` entry: change "`casehub-qhorus-testing`" to "`casehub-qhorus-persistence-memory`". Remove `@CrossTenant` references from glossary if present.

**B9. Protocol PP-20260609-67996e** (`docs/protocols/casehub/scheduled-service-cross-tenant-stores.md`) — update to reflect the new injection pattern. The underlying rule remains valid: `@Scheduled` methods and startup observers must not inject tenant-filtered stores. The mechanism changes:
- Title: remove `@CrossTenant` — becomes "...must use `CrossTenant*Store` interfaces and explicit tenancyId"
- Body: replace "inject `@CrossTenant`-qualified stores (produced by `CrossTenantProducer`) for all reads" with "inject `CrossTenant*Store` interfaces directly for all reads"
- `violation_hint`: update to reference direct `CrossTenant*Store` injection instead of `@CrossTenant`-qualified injection
- Update INDEX.md entry to match new title

**B10. Protocol PP-20260618-131cdf** (`docs/protocols/casehub/reactive-inmemory-store-selected-alternatives.md`) — update module references from `casehub-qhorus-testing` to `casehub-qhorus-persistence-memory`. The rule (reactive consumers must list InMemoryReactive*Store in `selected-alternatives`) is unchanged.
- `applies_to`: change "casehub-qhorus-testing" to "casehub-qhorus-persistence-memory"
- Body: change "InMemoryReactive*Store beans from `casehub-qhorus-testing`" to "from `casehub-qhorus-persistence-memory`"
- Update INDEX.md entry (line 15) to match

**B11. Protocol PP-20260618-100368** (`docs/protocols/casehub/inmemory-store-no-entity-mutation-in-session.md`) — update module reference from `casehub-qhorus-testing` to `casehub-qhorus-persistence-memory`. The rule (InMemory stores must not mutate PanacheEntity fields in session scope) is unchanged.
- `applies_to`: change "casehub-qhorus-testing" to "casehub-qhorus-persistence-memory"
- Update INDEX.md entry (line 81) to match

**B12. Living documentation** — update `casehub-qhorus-testing` → `casehub-qhorus-persistence-memory` references in:
- `docs/DESIGN.md` (lines 31, 221, 238, 252): module table and 3 body references attributing InMemory stores to `testing/`
- `CLAUDE.md` (line 355): change "adds `casehub-qhorus-testing` (or `casehub-qhorus-persistence-memory` directly)" to "adds `casehub-qhorus-persistence-memory`" — the parenthetical alternative is now the only path
- Note: `CLAUDE.md` lines 367, 386 reference `RecordingChannelBackend` and `MessageLedgerEntryTestFactory` in `casehub-qhorus-testing` — these are correct (both utilities are retained in `testing/`)

### C. Verification

1. `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` — full build of all modules
2. `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -Pwith-llm-examples` — verify `examples/agent-communication` compiles (profile-gated, not exercised by standard build)
3. All tests must pass: runtime, persistence-memory, testing, connector-backend, slack-channel, examples/type-system, examples/normative-layout
4. No CDI ambiguity errors in any module
5. Documentation sweep: search all `.md` files for `casehub-qhorus-testing` and verify that any remaining references correctly refer to utilities retained in `testing/` (RecordingChannelBackend, MessageLedgerEntryTestFactory, CommitmentServiceTest) — not to InMemory stores

### Cross-repo follow-up (out of scope — issues to file during implementation)

| Consumer | What changes | Nature |
|----------|-------------|--------|
| Claudony (9 test files) | `import io.casehub.qhorus.testing.InMemory*` → `io.casehub.qhorus.persistence.memory.InMemory*` | Mechanical |
| Devtown (2 properties files) | Remove ghost `exclude-types` for `io.casehub.qhorus.testing.InMemory*`, remove `CrossTenantProducer` exclude, remove `persistence-memory` excludes from production properties | Mechanical |
| OpenClaw (1 test file) | Same import change as Claudony | Mechanical |

---

## CDI Resolution After Fix

**Production** (only runtime on classpath):
```
@Inject ChannelStore           → JpaChannelStore @ApplicationScoped
@Inject CrossTenantChannelStore → JpaCrossTenantChannelStore @ApplicationScoped
```

**Tests** (runtime + persistence-memory on classpath):
```
@Inject ChannelStore           → InMemoryChannelStore @Alternative @Priority(1) wins
@Inject CrossTenantChannelStore → InMemoryCrossTenantChannelStore @Alternative @Priority(1) wins
```

No qualifiers. No producers. No exclude-types. CDI priority resolution works as designed.
