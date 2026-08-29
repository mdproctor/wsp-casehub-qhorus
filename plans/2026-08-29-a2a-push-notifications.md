# A2A Push Notifications Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #406 — E9: A2A push notifications
**Issue group:** #406

**Goal:** Implement server-side A2A push notifications — when an external client sends `message/send` with a `TaskPushNotificationConfig`, Qhorus pushes `TaskStatusUpdateEvent` back as the task progresses.

**Architecture:** New optional module `a2a-push-notification/` with `PushNotificationBackend` (ChannelBackend, AT_LEAST_ONCE). Non-throwing `post()` filters by correlationId, pushes to registered push URLs. Config CRUD via A2AResource JSON-RPC extension. Persistent JPA configs with three-tier cleanup.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-qhorus-api, casehub-a2a-protocol, casehub-platform-api (CredentialResolver), java.net.http.HttpClient

## Global Constraints

- Java package: `io.casehub.qhorus.a2a.push`
- Module artifactId: `casehub-qhorus-a2a-push-notification`
- Flyway migration V49 in `runtime/src/main/resources/db/qhorus/migration/`
- All IntelliJ MCP for code navigation and editing — no bash grep/Edit on .java files
- CDI-free unit tests for backend, poster, cleanup; `@QuarkusTest` for integration
- `post()` never throws — per-URL health tracking internal to the backend
- Cross-tenant stores for background operations (push backend, cleanup job)
- Test import-qhorus-test.sql required for ledger-enabled tests

---

## Batch 1: Foundation — API types + stores + migration + InMemory

After this batch: store interfaces exist in api/, Flyway migration creates the table, InMemory stores pass contract tests. No runtime behavior yet.

### Task 1: PushNotificationConfig DTO + store interfaces in api/

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/a2a/PushNotificationConfig.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/PushNotificationConfigStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantPushNotificationConfigStore.java`

**Interfaces:**
- Produces: `PushNotificationConfig(UUID id, String taskId, UUID channelId, String url, String token, String authScheme, String authCredentialsRef, String tenancyId, Instant createdAt, Instant lastPushedAt)`
- Produces: `PushNotificationConfigStore.put(PushNotificationConfig)`, `.findById(UUID)`, `.findByTaskId(String)`, `.delete(UUID)`, `.deleteByTaskId(String)`
- Produces: `CrossTenantPushNotificationConfigStore.findByTaskId(String)`, `.findByChannelId(UUID)`, `.activeTaskIds()`, `.findExpired(Instant)`, `.updateLastPushedAt(UUID, Instant)`, `.delete(UUID)`, `.deleteByTaskId(String)`

- [ ] **Step 1: Create PushNotificationConfig record**

```java
package io.casehub.qhorus.api.a2a;

import java.time.Instant;
import java.util.UUID;

public record PushNotificationConfig(
    UUID id,
    String taskId,
    UUID channelId,
    String url,
    String token,
    String authScheme,
    String authCredentialsRef,
    String tenancyId,
    Instant createdAt,
    Instant lastPushedAt
) {}
```

Use `ide_create_file` to create the file.

- [ ] **Step 2: Create PushNotificationConfigStore interface**

```java
package io.casehub.qhorus.api.store;

import io.casehub.qhorus.api.a2a.PushNotificationConfig;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface PushNotificationConfigStore {
    void put(PushNotificationConfig config);
    Optional<PushNotificationConfig> findById(UUID id);
    List<PushNotificationConfig> findByTaskId(String taskId);
    void delete(UUID id);
    void deleteByTaskId(String taskId);
}
```

- [ ] **Step 3: Create CrossTenantPushNotificationConfigStore interface**

```java
package io.casehub.qhorus.api.store;

import io.casehub.qhorus.api.a2a.PushNotificationConfig;
import java.time.Instant;
import java.util.Collection;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public interface CrossTenantPushNotificationConfigStore {
    List<PushNotificationConfig> findByTaskId(String taskId);
    List<PushNotificationConfig> findByChannelId(UUID channelId);
    Set<String> activeTaskIds();
    List<PushNotificationConfig> findExpired(Instant threshold);
    void updateLastPushedAt(UUID id, Instant pushedAt);
    void delete(UUID id);
    void deleteByTaskId(String taskId);
}
```

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C "$PROJECT" add api/src/main/java/io/casehub/qhorus/api/a2a/ api/src/main/java/io/casehub/qhorus/api/store/PushNotificationConfigStore.java api/src/main/java/io/casehub/qhorus/api/store/CrossTenantPushNotificationConfigStore.java
git -C "$PROJECT" commit -m "feat(#406): PushNotificationConfig DTO + store interfaces Refs #406"
```

### Task 2: Flyway migration V49 + InMemory store implementations + contract tests

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V49__push_notification_config.sql`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryPushNotificationConfigStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantPushNotificationConfigStore.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/PushNotificationConfigStoreContractTest.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryPushNotificationConfigStoreTest.java`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryPushNotificationConfigStoreTest.java`

**Interfaces:**
- Consumes: `PushNotificationConfigStore`, `CrossTenantPushNotificationConfigStore`, `PushNotificationConfig`
- Produces: `InMemoryPushNotificationConfigStore`, `InMemoryCrossTenantPushNotificationConfigStore`

- [ ] **Step 1: Create Flyway migration V49**

```sql
-- V49__push_notification_config.sql
CREATE TABLE push_notification_config (
    id UUID PRIMARY KEY,
    task_id VARCHAR(255) NOT NULL,
    channel_id UUID NOT NULL,
    url VARCHAR(1024) NOT NULL,
    token VARCHAR(512),
    auth_scheme VARCHAR(64),
    auth_credentials_ref VARCHAR(255),
    tenancy_id VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    last_pushed_at TIMESTAMP,
    CONSTRAINT uq_push_config_task_url UNIQUE (task_id, url),
    CONSTRAINT fk_push_config_channel FOREIGN KEY (channel_id)
        REFERENCES channel(id) ON DELETE CASCADE
);

CREATE INDEX idx_push_config_task_id ON push_notification_config(task_id);
CREATE INDEX idx_push_config_channel_id ON push_notification_config(channel_id);
```

Write file at `runtime/src/main/resources/db/qhorus/migration/V49__push_notification_config.sql`.

- [ ] **Step 2: Write contract test (failing)**

Create `PushNotificationConfigStoreContractTest` abstract base class following the `ExternalAgentBindingStoreContractTest` pattern. Test methods:
- `put_and_findById`
- `put_and_findByTaskId`
- `put_duplicate_taskId_url_overwrites`
- `delete_by_id`
- `deleteByTaskId`
- `activeTaskIds_returns_distinct_taskIds`
- `findExpired_with_threshold`
- `updateLastPushedAt`

Create `InMemoryPushNotificationConfigStoreTest` concrete runner.

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryPushNotificationConfigStoreTest -q`
Expected: FAIL (implementations don't exist yet)

- [ ] **Step 4: Implement InMemoryPushNotificationConfigStore**

`@Alternative @Priority(1)` implementing both `PushNotificationConfigStore` and `CrossTenantPushNotificationConfigStore`. Uses `ConcurrentHashMap<UUID, PushNotificationConfig>` keyed by id.

Key implementation details:
- `activeTaskIds()`: stream values, collect distinct taskIds into Set
- `findExpired(threshold)`: filter where `COALESCE(lastPushedAt, createdAt)` is before threshold
- `updateLastPushedAt(id, pushedAt)`: replace entry with updated lastPushedAt via record wither/constructor
- No `PanacheEntity` fields — pure record storage (per `inmemory-store-no-entity-mutation-in-session` protocol)

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryPushNotificationConfigStoreTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C "$PROJECT" add runtime/src/main/resources/db/qhorus/migration/V49__push_notification_config.sql persistence-memory/
git -C "$PROJECT" commit -m "feat(#406): V49 migration + InMemory push notification config store Refs #406"
```

## Batch 2: Push backend + poster — core push delivery logic

After this batch: push backend compiles and is unit-tested (CDI-free). No module wiring yet.

### Task 3: Maven module scaffold + PushNotificationConfigEntity + JPA store

**Files:**
- Create: `a2a-push-notification/pom.xml`
- Modify: `pom.xml` (parent — add `<module>a2a-push-notification</module>`)
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/PushNotificationConfigEntity.java`
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/JpaPushNotificationConfigStore.java`
- Create: `a2a-push-notification/src/main/resources/application.properties`

**Interfaces:**
- Consumes: `PushNotificationConfig`, `PushNotificationConfigStore`, `CrossTenantPushNotificationConfigStore`
- Produces: `PushNotificationConfigEntity`, `JpaPushNotificationConfigStore` (CDI bean implementing both store interfaces)

- [ ] **Step 1: Create module pom.xml**

Mirror `a2a-outbound/pom.xml` structure. Dependencies:
- `casehub-qhorus-api` (compile)
- `casehub-a2a-protocol` (compile)
- `casehub-platform-api` (compile, for CredentialResolver)
- `jakarta.enterprise:jakarta.enterprise.cdi-api` (provided)
- `jakarta.persistence:jakarta.persistence-api` (provided)
- `org.jboss.logging:jboss-logging` (provided)
- `casehub-qhorus` (test)
- `casehub-qhorus-persistence-memory` (test)
- `io.quarkus:quarkus-junit5` (test)
- `io.rest-assured:rest-assured` (test)

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>a2a-push-notification</module>` after `<module>push</module>` in the parent pom.

- [ ] **Step 3: Create PushNotificationConfigEntity**

JPA entity following `ExternalAgentBindingEntity` pattern. Maps to `push_notification_config` table. Named PU `qhorus`. `@PrePersist` for id and createdAt defaults. `toDomain()` and `fromDomain()` methods.

- [ ] **Step 4: Create JpaPushNotificationConfigStore**

`@ApplicationScoped` implementing both `PushNotificationConfigStore` and `CrossTenantPushNotificationConfigStore`. Uses `@PersistenceContext(unitName = "qhorus") EntityManager`. Tenant-scoped methods filter by `CurrentPrincipal.tenancyId()`; cross-tenant methods do not.

Key queries:
- `activeTaskIds()`: `SELECT DISTINCT e.taskId FROM PushNotificationConfig e`
- `findExpired(threshold)`: `WHERE COALESCE(e.lastPushedAt, e.createdAt) < :threshold`
- `updateLastPushedAt(id, pushedAt)`: find entity by id, set field, merge

- [ ] **Step 5: Create module application.properties**

Empty for now — configuration comes from the consuming app.

- [ ] **Step 6: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl a2a-push-notification -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C "$PROJECT" add a2a-push-notification/ pom.xml
git -C "$PROJECT" commit -m "feat(#406): a2a-push-notification module scaffold + JPA store Refs #406"
```

### Task 4: PushNotificationPoster + PushNotificationBackend (CDI-free unit tests)

**Files:**
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/PushNotificationPoster.java`
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/PushPostResult.java`
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/PushNotificationBackend.java`
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/UrlHealthState.java`
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/PushNotificationConfig.java` (module config mapping)
- Create: `a2a-push-notification/src/test/java/io/casehub/qhorus/a2a/push/PushNotificationPosterTest.java`
- Create: `a2a-push-notification/src/test/java/io/casehub/qhorus/a2a/push/PushNotificationBackendTest.java`

**Interfaces:**
- Consumes: `ChannelBackend`, `ChannelRef`, `OutboundMessage`, `DeliveryGuarantee`, `ActorType`, `CredentialResolver`, `BackendRegistry`, `CrossTenantPushNotificationConfigStore`, `PushNotificationConfig` (DTO)
- Produces: `PushNotificationPoster.post(String url, String payload, String token, String authScheme, String authCredentialsRef)` → `PushPostResult(boolean success, int statusCode, String error)`
- Produces: `PushNotificationBackend` (ChannelBackend impl)

- [ ] **Step 1: Write PushNotificationPoster test (failing)**

CDI-free test with Mockito. Test cases:
- `post_success_returns_true` — mock HttpClient to return 200
- `post_failure_returns_false_with_status` — mock 500 response
- `post_timeout_returns_false` — mock timeout exception
- `post_with_auth_header_includes_authorization` — verify Authorization header
- `post_with_token_includes_token_in_payload` — verify token in JSON body

- [ ] **Step 2: Implement PushNotificationPoster**

`@ApplicationScoped`. Uses `java.net.http.HttpClient`. Builds JSON payload per the A2A push notification format. Resolves auth via `CredentialResolver` following `A2AOutboundBackend.resolveAuth()` pattern. `authScheme` takes precedence over credential map's `"type"` key.

Inner interface `HttpPoster` for test injection (same pattern as `WebhookMessageObserver.WebhookPoster`).

- [ ] **Step 3: Run poster tests to verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl a2a-push-notification -Dtest=PushNotificationPosterTest -q`

- [ ] **Step 4: Write PushNotificationBackend test (failing)**

CDI-free test with Mockito mocks for store, poster, credential resolver. Test cases:
- `post_nonPushRelevantType_noOp` — EVENT, COMMAND, QUERY, PROPOSE → no poster call
- `post_pushRelevantType_noCorrIdMatch_noOp` — STATUS with unknown correlationId
- `post_status_matchingCorrId_pushesWorkingState` — STATUS → "working"
- `post_done_matchingCorrId_pushesCompletedState_deletesConfig` — terminal cleanup
- `post_failure_matchingCorrId_pushesFailed_deletesConfig`
- `post_decline_matchingCorrId_pushesCanceled_deletesConfig`
- `post_handoff_matchingCorrId_pushesWorking`
- `post_httpFailure_nonTerminal_dropsMessage` — STATUS push fails, no retry
- `post_httpFailure_terminal_tracksPending` — DONE push fails, tracked in memory
- `post_neverThrows_onPosterException` — poster throws RuntimeException, post() returns normally
- `post_urlInBackoff_skipped` — URL within backoff window, no HTTP call
- `post_urlExhausted_configDeleted` — max failures, config deleted
- `post_pendingTerminal_retriedOnNextPost` — terminal retry on subsequent post()
- `post_lazyDbFallback_onCacheMiss` — cache miss → DB query → cache populated

- [ ] **Step 5: Implement PushNotificationBackend**

`@ApplicationScoped` implementing `ChannelBackend`. Key implementation:
- `backendId()` → `"a2a-push"`
- `actorType()` → `ActorType.AGENT`
- `deliveryGuarantee()` → `AT_LEAST_ONCE`
- `open()`, `close()` → no-op
- In-memory: `ConcurrentHashMap<String, UrlHealthState> urlHealth`, `Set<String> activeTaskIds`, `ConcurrentHashMap<String, PendingTerminal> pendingTerminals`
- `post()`: message type filter → cache check (with lazy DB fallback) → per-config loop (health check → map state → post → handle result)
- `onChannelRecovery(@Observes ChannelInitialisedEvent)`: re-register for channels with active configs, startup terminal recovery

- [ ] **Step 6: Run backend tests to verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl a2a-push-notification -Dtest=PushNotificationBackendTest -q`

- [ ] **Step 7: Commit**

```bash
git -C "$PROJECT" add a2a-push-notification/src/
git -C "$PROJECT" commit -m "feat(#406): PushNotificationPoster + PushNotificationBackend with CDI-free tests Refs #406"
```

## Batch 3: A2AResource extensions + AgentCard — protocol surface

After this batch: A2A JSON-RPC push config CRUD works end-to-end. AgentCard advertises `pushNotifications=true` when module is on classpath.

### Task 5: A2AResource push config CRUD + inline message/send + AgentCard capability

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java`
- Create: `a2a-push-notification/src/test/java/io/casehub/qhorus/a2a/push/A2APushNotificationIntegrationTest.java`

**Interfaces:**
- Consumes: `Instance<PushNotificationConfigStore>`, `CommitmentStore.findByCorrelationId(String)`, `PushNotificationConfig`, `BackendRegistry`
- Produces: JSON-RPC methods `pushNotificationConfig/set`, `/get`, `/list`, `/delete` in A2AResource. Inline `pushNotificationConfig` field in `message/send` params.

- [ ] **Step 1: Write integration test (failing)**

`@QuarkusTest @TestTransaction`. Test cases:
- `pushNotificationConfig_set_createsConfig` — JSON-RPC set → config persisted
- `pushNotificationConfig_get_returnsConfig` — get by taskId + configId
- `pushNotificationConfig_list_returnsAllForTask` — list by taskId
- `pushNotificationConfig_delete_removesConfig` — delete by id
- `pushNotificationConfig_set_withoutModule_returns501` — (separate profile without module)
- `messageSend_withInlinePushConfig_createsConfig` — inline push config alongside message dispatch
- `agentCard_pushNotifications_true_whenModulePresent` — capability advertised

- [ ] **Step 2: Add Instance<PushNotificationConfigStore> to A2AResource**

Inject `@Inject Instance<PushNotificationConfigStore> pushNotificationConfigStore`.
Inject `@Inject Instance<BackendRegistry> backendRegistry` (for push backend registration on config create).

- [ ] **Step 3: Add push config dispatch cases to A2AResource.dispatch()**

Extend the switch in `dispatch()`:
```java
case "pushNotificationConfig/set" -> handlePushConfigSet(request);
case "pushNotificationConfig/get" -> handlePushConfigGet(request);
case "pushNotificationConfig/list" -> handlePushConfigList(request);
case "pushNotificationConfig/delete" -> handlePushConfigDelete(request);
```

Each method: check `pushNotificationConfigStore.isResolvable()` → 501 if absent → parse params → delegate to store → return JSON-RPC response.

`handlePushConfigSet`: parse `taskId`, `url`, `token`, `authScheme`, `authCredentialsRef`. If `channelId` absent, resolve from `commitmentStore.findByCorrelationId(taskId)`. Register push backend for channel via `backendRegistry`.

- [ ] **Step 4: Add inline push config to handleMessageSend()**

After successful message dispatch, check for `pushNotificationConfig` in params. If present, create `PushNotificationConfig` and store it. Register push backend for channel.

- [ ] **Step 5: Update AgentCardResource**

Inject `@Inject Instance<PushNotificationConfigStore> pushStore`.
Change: `new AgentCapabilities(true, false)` → `new AgentCapabilities(true, pushStore.isResolvable())`

- [ ] **Step 6: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl a2a-push-notification -Dtest=A2APushNotificationIntegrationTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C "$PROJECT" add runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java a2a-push-notification/src/test/
git -C "$PROJECT" commit -m "feat(#406): A2AResource push config CRUD + AgentCard capability Refs #406"
```

## Batch 4: Cleanup job + full build verification

After this batch: TTL cleanup works. Full `mvn install` passes. Module is production-ready.

### Task 6: PushNotificationCleanupJob + CLAUDE.md update + full build

**Files:**
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/PushNotificationCleanupJob.java`
- Create: `a2a-push-notification/src/main/java/io/casehub/qhorus/a2a/push/PushConfig.java` (Quarkus @ConfigMapping)
- Create: `a2a-push-notification/src/test/java/io/casehub/qhorus/a2a/push/PushNotificationCleanupJobTest.java`
- Modify: `CLAUDE.md` (add module to project structure, document testing conventions)

**Interfaces:**
- Consumes: `CrossTenantPushNotificationConfigStore.findExpired(Instant)`, `.delete(UUID)`

- [ ] **Step 1: Write cleanup job test (failing)**

CDI-free test with `InMemoryCrossTenantPushNotificationConfigStore`. Test cases:
- `cleanup_removesExpiredConfigs` — config with old createdAt and null lastPushedAt → deleted
- `cleanup_preservesActiveConfigs` — config with recent lastPushedAt → kept
- `cleanup_preservesFreshConfigs` — config with recent createdAt → kept
- `cleanup_usesCoalesceLogic` — config with old createdAt but recent lastPushedAt → kept

- [ ] **Step 2: Implement PushNotificationCleanupJob**

`@ApplicationScoped`. `@Scheduled(every = "${casehub.qhorus.a2a.push.cleanup-interval:5m}")`. Injects `CrossTenantPushNotificationConfigStore`. Computes threshold from config TTL, calls `findExpired(threshold)`, deletes each.

- [ ] **Step 3: Create PushConfig ConfigMapping**

```java
@ConfigMapping(prefix = "casehub.qhorus.a2a.push")
public interface PushConfig {
    @WithDefault("true") boolean enabled();
    @WithDefault("PT24H") Duration ttlThreshold();
    @WithDefault("PT5M") Duration cleanupInterval();
    @WithDefault("5") int maxUrlFailures();
    @WithDefault("5000") int httpTimeoutMs();
}
```

- [ ] **Step 4: Run cleanup tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl a2a-push-notification -Dtest=PushNotificationCleanupJobTest -q`
Expected: PASS

- [ ] **Step 5: Update CLAUDE.md**

Add `a2a-push-notification/` to the project structure section. Document:
- Module description and package
- Testing conventions (CDI-free for backend/poster/cleanup, `@QuarkusTest` for integration)
- Config properties
- The non-throwing post() contract
- Consumer package registration requirement

- [ ] **Step 6: Full build verification**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q`
Expected: BUILD SUCCESS, all modules compile and tests pass

- [ ] **Step 7: Commit**

```bash
git -C "$PROJECT" add a2a-push-notification/ CLAUDE.md
git -C "$PROJECT" commit -m "feat(#406): cleanup job + config mapping + CLAUDE.md update Refs #406"
```

## References

- [2026-08-29-a2a-push-notifications-design.md] — design spec this plan implements
- [decisions.md] — 8 design decisions (D1-D8) from brainstorming
- [A2AResource.java:77-95] — existing JSON-RPC dispatch switch
- [AgentCardResource.java:30-49] — agent card capability construction
- [A2AOutboundBackend.java] — ChannelBackend precedent (different failure contract)
- [WebhookMessageObserver.java] — HTTP POST callback pattern
- [ExternalAgentBindingStore.java] — store interface pattern
- [InMemoryExternalAgentBindingStore.java] — InMemory store pattern
- [CrossTenantCommitmentStore.java] — cross-tenant store pattern
- Protocol: `channel-initialised-event-observer-idempotency`
- Protocol: `optional-module-jpa-package-registration`
- Protocol: `scheduled-service-cross-tenant-stores`
- Protocol: `inmemory-store-no-entity-mutation-in-session`
- GitHub #406
