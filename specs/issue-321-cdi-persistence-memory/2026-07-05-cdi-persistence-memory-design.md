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

**Deprecate (do not delete) the annotation class:**
- `api/.../qualifier/CrossTenant.java` — add `@Deprecated` with javadoc explaining the type-based injection pattern. Retained because external code may reference it; removing it is a separate cross-repo issue.

### A3. Update `selected-alternatives` in properties files

**Within qhorus — change `io.casehub.qhorus.testing.*` → `io.casehub.qhorus.persistence.memory.*`:**

| File | Entries |
|------|---------|
| `examples/agent-communication/src/main/resources/application.properties` | 5 store references |
| `slack-channel/src/test/resources/application.properties` | 12 store references (fix duplicate `InMemoryWatchdogStore`) |

Files already using correct package (no change): `examples/normative-layout`, `examples/type-system`, `connector-backend`.

### B. ARC42STORIES.MD §5 Alignment (#320)

**B1. Module structure** — add `persistence-memory/` module entry. Update `testing/` description: "Test utilities — RecordingChannelBackend, MessageLedgerEntryTestFactory" (no longer InMemory stores).

**B2. Api module table** — add missing packages:

| Package | Contents |
|---------|----------|
| `api.channel` | `Channel`, `ChannelDetail`, `ChannelManager`, `ReactiveChannelManager`, `FindOrCreateResult` |
| `api.data` | `SharedData`, `ArtefactClaim` domain types |
| `api.gateway` | `ChannelBackend` hierarchy, `MessageObserver`, `MessageReceivedEvent`, `ChannelInitialisedEvent`, inbound/outbound records |
| `api.instance` | `InstanceInfo` |
| `api.message` | `MessageResult`, `MessageType`, `MessageDispatcher`, `ReactiveMessageDispatcher`, `CommitmentState`, `CommitmentDeclinedEvent`, `CommitmentExpiredEvent` |
| `api.qualifier` | `@CrossTenant` (deprecated) |
| `api.spi` | `CommitmentAttestationPolicy`, `CommitmentContext`, `ObligorTrustPolicy`, `ObligorTrustContext`, `InstanceActorIdProvider` |
| `api.store` | `ChannelStore`, `MessageStore`, `CommitmentStore`, `InstanceStore`, `DataStore`, `WatchdogStore`, `DeliveryCursorStore`, `ChannelBindingStore` + Reactive variants + CrossTenant variants |
| `api.store.query` | `ChannelQuery`, `DataQuery`, `InstanceQuery`, `MessageQuery`, `WatchdogQuery` |
| `api.watchdog` | `Watchdog`, `WatchdogConditionType`, `WatchdogAlertRouter` SPI, `WatchdogAlertEvent`, alert context records (`AlertContext`, `AgentStaleContext`, `BarrierStuckContext`, `ChannelIdleContext`, `QueueDepthContext`, `ApprovalPendingContext`, `AlertDeliveryTarget`) |

### C. Verification

1. `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` — full build of all modules
2. All tests must pass: runtime, persistence-memory, testing, connector-backend, slack-channel, examples/type-system, examples/normative-layout
3. No CDI ambiguity errors in any module

### Cross-repo follow-up (out of scope — issues filed)

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
