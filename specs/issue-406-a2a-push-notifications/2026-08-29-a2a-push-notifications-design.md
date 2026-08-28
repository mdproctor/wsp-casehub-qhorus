# A2A Push Notifications — Design Spec

**Issue:** casehubio/qhorus#406
**Branch:** issue-406-a2a-push-notifications
**Date:** 2026-08-29

## Scope

Server-side A2A push notifications per the A2A protocol spec. When an external
A2A client sends `message/send` with a `TaskPushNotificationConfig`, Qhorus
stores the config and pushes `TaskStatusUpdateEvent` notifications as the task
progresses (STATUS, DONE, FAILURE, DECLINE, HANDOFF).

**Excluded:** Client-side push subscription (Qhorus as A2A consumer receiving
push from external agents via `a2a-outbound`) — architecturally distinct,
separate follow-up issue. General channel push (pushing arbitrary channel
events to registered external agents) — different capability, separate issue.

## Module

New optional Maven module: `a2a-push-notification/`

- Package: `io.casehub.qhorus.a2a.push`
- Activates by classpath presence (same pattern as `webhook-observer/`,
  `kafka-observer/`, `a2a-outbound/`)
- Dependencies: `casehub-qhorus-api`, `casehub-a2a-protocol`,
  `casehub-platform-api` (for `CredentialResolver`)

Consumers adding this module must register the package in
`quarkus.hibernate-orm.qhorus.packages` (per `optional-module-jpa-package-registration`
protocol).

## Architecture

### Cross-module integration

```
api/                           runtime/                     a2a-push-notification/
  PushNotificationConfig         A2AResource                  PushNotificationConfigEntity
  PushNotificationConfigStore     (Instance<Store>)           JpaPushNotificationConfigStore
  CrossTenantPushNotifStore       AgentCardResource            PushNotificationBackend
                                   (Instance<Store>)           PushNotificationPoster
                                  A2ATaskStateMapper           PushNotificationCleanupJob
                                   (reused)
```

**Roadmap divergence note:** Issue #406 describes the push backend as a
`MessageObserver`. The design review (D3, standard 3-round) determined that
`ChannelBackend` with `AT_LEAST_ONCE` is the correct pattern: it provides
ordered cursor-based delivery via the existing delivery pump, avoiding a
second delivery mechanism. The key constraint: `post()` must be non-throwing
(unlike `A2AOutboundBackend`) because push has multi-target per-channel
semantics where one dead URL must not stall all others.

`A2AResource` and `AgentCardResource` inject `Instance<PushNotificationConfigStore>`.
When the module is on the classpath, the store is resolvable and push is enabled.
When absent, push JSON-RPC methods return 501, `AgentCapabilities.pushNotifications`
remains `false`.

### Data flow

```
External A2A client                            Qhorus
       |                                         |
       |--message/send + pushNotificationConfig-->|
       |                                         |-- store push config (JPA)
       |                                         |-- dispatch message (creates commitment)
       |                                         |-- register push backend for channel
       |                                         |
       |            (task progresses: STATUS, DONE, FAILURE, ...)
       |                                         |
       |                                         |-- DeliveryService pump
       |                                         |     \-- PushNotificationBackend.post()
       |                                         |           \-- correlationId match? --> push
       |  <--TaskStatusUpdateEvent (HTTP POST)----|
       |                                         |
```

## Components

### 1. PushNotificationConfig (api/store/)

DTO record in `api/`:

```java
public record PushNotificationConfig(
    UUID id,
    String taskId,       // correlationId
    UUID channelId,
    String url,          // push endpoint
    String token,        // verification token (included in push payload)
    String authScheme,   // "Bearer", "Basic", etc.
    String authCredentialsRef, // CredentialResolver key
    String tenancyId,
    Instant createdAt,
    Instant lastPushedAt // updated on each successful push
) {}
```

### 2. PushNotificationConfigStore (api/store/)

Store interface following the standard taxonomy. Two interfaces: tenant-scoped
for request-context operations (A2AResource CRUD), cross-tenant for background
operations (push backend, cleanup job).

```java
public interface PushNotificationConfigStore {
    void put(PushNotificationConfig config);
    Optional<PushNotificationConfig> findById(UUID id);
    List<PushNotificationConfig> findByTaskId(String taskId);
    void delete(UUID id);
    void deleteByTaskId(String taskId);
}

public interface CrossTenantPushNotificationConfigStore {
    List<PushNotificationConfig> findByTaskId(String taskId);
    List<PushNotificationConfig> findByChannelId(UUID channelId);
    Set<String> activeTaskIds();
    List<PushNotificationConfig> findExpired(Instant threshold);
    void updateLastPushedAt(UUID id, Instant pushedAt);
    void delete(UUID id);
    void deleteByTaskId(String taskId);
    List<PushNotificationConfig> findWithTerminalTasks(
        Collection<String> taskIds);
}
```

`CrossTenantPushNotificationConfigStore` is injected by `PushNotificationBackend`
and `PushNotificationCleanupJob` (both run without request context, per
`scheduled-service-cross-tenant-stores` protocol). The `findWithTerminalTasks()`
method supports startup recovery (see section 4).

### 3. PushNotificationConfigEntity (a2a-push-notification/)

JPA entity in the qhorus named PU:

```java
@Entity(name = "PushNotificationConfig")
@Table(name = "push_notification_config",
    uniqueConstraints = @UniqueConstraint(
        name = "uq_push_config_task_url",
        columnNames = {"task_id", "url"}))
public class PushNotificationConfigEntity {
    @Id public UUID id;
    @Column(name = "task_id", nullable = false) public String taskId;
    @Column(name = "channel_id", nullable = false) public UUID channelId;
    @Column(nullable = false, length = 1024) public String url;
    @Column public String token;
    @Column(name = "auth_scheme") public String authScheme;
    @Column(name = "auth_credentials_ref") public String authCredentialsRef;
    @Column(name = "tenancy_id", nullable = false) public String tenancyId;
    @Column(name = "created_at", nullable = false, updatable = false) public Instant createdAt;
    @Column(name = "last_pushed_at") public Instant lastPushedAt;
}
```

Migration V49: `push_notification_config` table with composite unique constraint
on `(task_id, url)`.

### 4. PushNotificationBackend (a2a-push-notification/)

Implements `ChannelBackend` with `DeliveryGuarantee.AT_LEAST_ONCE`.

```java
@ApplicationScoped
public class PushNotificationBackend implements ChannelBackend {
    // backendId: "a2a-push"
    // actorType: ActorType.AGENT  (push recipients are external A2A agents)
    // deliveryGuarantee: AT_LEAST_ONCE
}
```

**AT_LEAST_ONCE semantics clarified:** The `AT_LEAST_ONCE` guarantee means the
delivery pump ensures every message reaches the push backend's `post()` method
in order, with cursor tracking, dedup, and reconciliation. It does NOT mean
every push HTTP delivery succeeds — that is best-effort for non-terminal
messages, retry-with-backoff for terminal. The pump provides ORDER + DEDUP +
RECONCILIATION; the push backend provides per-URL health + selective retry.

**Registration:** The backend registers itself for a channel when a push config
is created for a task on that channel. Uses
`BackendRegistry.registerBackend(channelId, this, "agent")` (backendType
`"agent"`, matching `A2AOutboundBackend` and `A2AChannelBackend`).
On `@Observes ChannelInitialisedEvent`: re-registers for channels that have
active push configs (startup recovery, per `channel-initialised-event-observer-idempotency`
protocol).

**Startup recovery for terminal deliveries:** On `ChannelInitialisedEvent`,
in addition to re-registering for channels with active push configs, the
backend scans for push configs whose tasks have already reached terminal state
(cross-referencing `CrossTenantPushNotificationConfigStore` with
`CommitmentStore.findByCorrelationId()`). For each terminal task, one final
push attempt is made. On success: config deleted. On failure: config left for
TTL cleanup.

**post() contract — non-throwing:**

Unlike `A2AOutboundBackend.post()` which throws on HTTP failure, the push
backend's `post()` never throws. This is critical because push has multi-target
per-channel semantics: multiple clients may register push configs for different
tasks on the same channel. If `post()` threw on one dead push URL, it would
stall the delivery cursor and block all other push URLs on that channel.
After `maxConsecutiveFailures` (default 10), the entire push backend would be
marked unhealthy globally — all channels lose push delivery.

**post() logic:**

1. Check if the message type is push-relevant (STATUS, DONE, FAILURE, DECLINE,
   HANDOFF). If not, return (no-op).
2. Look up push configs by `message.correlationId()` via in-memory cache.
   If no configs exist, return (no-op).
3. For each matching push config:
   a. Check per-URL health — if URL is within backoff window, skip.
   b. Map message to `TaskStatusUpdateEvent` using dedicated mapping
      (not `A2ATaskStateMapper.fromMessageType()` directly — see D8).
   c. POST to the push URL via `PushNotificationPoster`.
   d. On success: update `lastPushedAt`. If terminal message, delete config.
   e. On failure: increment per-URL failure count, apply backoff.
4. Retry pending terminal deliveries (in-memory tracked) before processing
   current message.

**Per-URL health tracking:**

```java
// In-memory, per push backend instance
ConcurrentHashMap<String, UrlHealthState> urlHealth;

record UrlHealthState(
    int failures,
    Instant lastFailure,
    Duration backoffWindow  // 5s, 30s, 2m, 10m, 1h
) {}
```

On HTTP failure: increment failure count, compute exponential backoff.
On subsequent calls: if within backoff window, skip (no-op for that URL).
After configurable max failures (default 5): mark URL as exhausted, delete
the push config, log at WARN level.

**Differentiated delivery by message criticality:**

- **Non-terminal** (STATUS, HANDOFF): best-effort push. Failed deliveries are
  dropped. Task state is progressive — later pushes carry the current state,
  superseding missed intermediate updates.
- **Terminal** (DONE, FAILURE, DECLINE): retried on subsequent `post()` calls.
  When a terminal push fails, the backend tracks `(correlationId, terminalState)`
  in-memory. On each subsequent `post()` call for the same channel, pending
  terminal deliveries are retried before processing the current message.
  On success: push config deleted. On exhaust: push config deleted, client
  must poll `tasks/get`.

**In-memory state is lost on restart.** Push configs persist in JPA; the TTL
cleanup (see Cleanup Lifecycle) eventually removes undelivered configs. Clients
should treat push as supplementary and poll `tasks/get` if no push arrives
within a reasonable timeout.

### 5. PushNotificationPoster (a2a-push-notification/)

HTTP client that posts A2A events to push URLs.

```java
@ApplicationScoped
public class PushNotificationPoster {
    PostResult post(String url, TaskStatusUpdateEvent event,
                    String token, AuthConfig auth);
}
```

- Uses `java.net.http.HttpClient` with configurable timeout
- Adds `Authorization: <scheme> <credentials>` header from resolved auth config
- Includes `token` in the event payload (A2A spec: verification token)
- Returns `PostResult(boolean success, int statusCode, String error)`
- Content-Type: `application/json`
- Non-blocking: runs on calling thread (pump thread), with timeout

### 6. TaskStatusUpdateEvent mapping

Dedicated per-message-type mapping. RESPONSE is excluded because
`A2ATaskStateMapper` has an internal inconsistency: `fromMessageType(RESPONSE)`
returns "working" (default case) while `statePriority(RESPONSE) = 4` maps to
"completed" (same priority as DONE). Rather than encoding either side of this
inconsistency, RESPONSE is excluded — DONE is the unambiguous completion
signal for A2A tasks. The mapper inconsistency should be resolved independently.

| MessageType | A2A task state | Push? |
|-------------|---------------|-------|
| STATUS      | "working"     | Yes   |
| HANDOFF     | "working"     | Yes   |
| DONE        | "completed"   | Yes (terminal) |
| FAILURE     | "failed"      | Yes (terminal) |
| DECLINE     | "canceled"    | Yes (terminal) |
| RESPONSE    | —             | No (excluded, see D8) |
| COMMAND     | —             | No   |
| QUERY       | —             | No   |
| PROPOSE     | —             | No   |
| EVENT       | —             | No   |

Push payload:

```json
{
  "jsonrpc": "2.0",
  "method": "tasks/pushNotification",
  "params": {
    "id": "<push-config-id>",
    "task": {
      "id": "<task-id / correlationId>",
      "contextId": "<channel-id>",
      "status": {
        "state": "<mapped state>",
        "message": "<message content, nullable>"
      }
    }
  }
}
```

### 7. Push config cleanup lifecycle

Three-tier cleanup ensures no config accumulates indefinitely:

1. **Terminal delivery cleanup:** When `post()` successfully delivers a terminal
   message (DONE→"completed", FAILURE→"failed", DECLINE→"canceled"), it deletes
   the push config immediately. This is the normal cleanup path.

2. **TTL cleanup:** `PushNotificationCleanupJob` (`@Scheduled`, configurable
   interval, default 5 minutes) removes push configs where
   `COALESCE(lastPushedAt, createdAt) + threshold < now()`. Default threshold:
   24 hours. `lastPushedAt` is updated on each successful push, so long-running
   tasks that receive periodic STATUS updates are not prematurely cleaned.
   Tasks that register a push config but never receive any messages are cleaned
   after the threshold from `createdAt`.

3. **Client delete:** `tasks/pushNotificationConfig/delete` JSON-RPC method
   per A2A spec. The client can explicitly revoke their push config.

### 8. A2AResource extensions (runtime/)

Four new JSON-RPC methods dispatched by `A2AResource`:

| Method | Action |
|--------|--------|
| `pushNotificationConfig/set` | Create or update a push config for a task |
| `pushNotificationConfig/get` | Get a push config by task ID and config ID |
| `pushNotificationConfig/list` | List all push configs for a task |
| `pushNotificationConfig/delete` | Delete a push config |

All methods check `Instance<PushNotificationConfigStore>.isResolvable()`.
When absent: return JSON-RPC error with code -32601 ("method not found").

**Standalone `pushNotificationConfig/set`:** The request params include
`taskId` and optionally `channelId`. If `channelId` is absent, the server
resolves it from `CommitmentStore.findByCorrelationId(taskId)` → channel of
the originating COMMAND/QUERY. If no commitment is found, return JSON-RPC
error -32602 ("invalid params: unknown task").

**Inline push config in `message/send`:**

The `message/send` params object gains an optional `pushNotificationConfig`
field. When present, A2AResource creates the push config alongside the message
dispatch. This is the primary registration path — clients typically include
push config with their first `message/send` rather than making separate calls.

**AgentCardResource:**

`AgentCardResource.getAgentCard()` sets
`capabilities.pushNotifications = pushNotificationConfigStore.isResolvable()`
(currently hardcoded `false`).

### 9. Authentication

Outbound push requests authenticate using the `authentication` object from
the client's `PushNotificationConfig`:

- Client provides `authScheme` ("Bearer", "Basic") and `authConfigKey` when
  registering the push config. The `authConfigKey` is a `CredentialResolver`
  reference — the client manages credential storage through the platform's
  credential API, not by sending raw tokens in the push config.
- At push time: `CredentialResolver.resolve(authConfigKey)` returns
  `Map<String, String>` — extract `credentials.get("token")` and
  `credentials.getOrDefault("type", "bearer")` (matching the
  `A2AOutboundBackend.resolveAuth()` pattern).
- `authScheme` on the entity is populated from the A2A spec's
  `AuthenticationInfo.scheme` field. When present, it takes precedence over
  the credential map's `"type"` key (spec-defined scheme > convention-derived).
- HTTP header: `Authorization: <scheme> <token>`

No verification handshake on push config registration. No HMAC payload signing
(can be added as a qhorus-specific extension later). These are not required by
the A2A spec.

### 10. In-memory cache and cross-tenant store access

`post()` is called for every message on registered channels. To avoid a DB
query per call, the push backend maintains an in-memory `Set<String>` of
active task IDs (correlationIds) with push configs.

- Populated from `PushNotificationConfigStore.activeTaskIds()` on startup
  and `ChannelInitialisedEvent`
- Updated on push config create/delete
- `post()` checks `activeTaskIds.contains(message.correlationId())` before
  querying the store for full config details

**Multi-node cache coherency:** The in-memory cache is per-JVM. In a multi-node
deployment, a push config created on Node A is invisible in Node B's cache.
Fix: **lazy DB fallback.** When `activeTaskIds` returns a miss, `post()` queries
`crossTenantStore.findByTaskId(correlationId)` before returning no-op. On hit:
add to local cache and proceed with push. This adds one DB query per cache miss
(bounded by distinct correlationIds per channel per batch, not per message).
Cache hits are the hot path — DB fallback is the cold path for cross-node and
post-restart scenarios.

**Cross-tenant store access:** The push backend's `post()` runs in the delivery
pump thread (`ManagedExecutor`) — no request context, no `CurrentPrincipal`.
Per `scheduled-service-cross-tenant-stores` protocol, the backend injects
`CrossTenantPushNotificationConfigStore`. `correlationId` is a UUID — globally
unique across tenants — so `findByTaskId(correlationId)` does not need tenant
scoping. The `activeTaskIds()` cache is cross-tenant by design.

### 11. Configuration

```properties
# Enable/disable push notification module
casehub.qhorus.a2a.push.enabled=true

# TTL cleanup
casehub.qhorus.a2a.push.ttl-threshold=PT24H
casehub.qhorus.a2a.push.cleanup-interval=PT5M

# Per-URL health
casehub.qhorus.a2a.push.max-url-failures=5
casehub.qhorus.a2a.push.http-timeout-ms=5000
```

### 12. Metrics

- `qhorus.push.delivered` — counter per URL domain, incremented on successful push
- `qhorus.push.failed` — counter per failure reason (timeout, 4xx, 5xx, connection)
- `qhorus.push.terminal.pending` — gauge, count of in-memory pending terminal deliveries
- `qhorus.push.urls.exhausted` — counter, URLs deleted after max failures

### 13. Flyway migration

**V49:** `push_notification_config` table

Located in `runtime/src/main/resources/db/qhorus/migration/V49__push_notification_config.sql`.
The table exists unconditionally (same pattern as `external_agent_binding` V42) —
avoids Flyway out-of-order versioning issues when the optional module is added
later.

```sql
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

## Testing strategy

- **CDI-free unit tests:** `PushNotificationBackend` with mocked store, poster,
  and credential resolver. Tests: correlationId filtering, per-URL health
  tracking, backoff logic, terminal delivery cleanup, non-throwing contract.
- **CDI-free unit tests:** `PushNotificationPoster` with mock HTTP server.
  Tests: auth header construction, payload format, timeout handling.
- **CDI-free unit tests:** `PushNotificationCleanupJob` with in-memory store.
  Tests: TTL threshold, lastPushedAt preservation.
- **Integration tests:** (`@QuarkusTest @TestTransaction`) A2AResource
  JSON-RPC methods for push config CRUD. Inline push config in message/send.
- **Integration tests:** End-to-end push delivery via
  `QuarkusTransaction.requiringNew()` (per `observer-test-transaction-discipline`
  protocol — push backend's `post()` is called by the delivery pump after
  commit).
- **Store contract tests:** `PushNotificationConfigStoreContractTest` in
  `persistence-memory/`, concrete runner with `InMemoryPushNotificationConfigStore`.

## References

- A2A protocol protobuf spec `a2a.proto` — `TaskPushNotificationConfig`,
  `AuthenticationInfo`, CRUD methods, `SendMessageConfiguration`,
  `AgentCapabilities.push_notifications`
- `a2a-outbound/.../A2AOutboundBackend.java` — ChannelBackend precedent
  (different failure contract)
- `webhook-observer/.../WebhookMessageObserver.java` — HTTP POST callback pattern
- `webhook-observer/.../WebhookRegistry.java` — persistent registration pattern
- `runtime/.../gateway/DeliveryService.java` — delivery pump
  (cursor starvation analysis)
- `runtime/.../gateway/DeliveryBatchExecutor.java` — sequential iteration,
  cursor stall on throw
- `runtime/.../api/A2AResource.java` — JSON-RPC dispatch
- `runtime/.../api/A2AChannelBackend.java` — SSE streaming, deregistration pattern
- `runtime/.../api/A2ATaskStateMapper.java` — task state mapping
  (RESPONSE inconsistency)
- `api/.../gateway/ChannelBackend.java` — backend interface
- `api/.../instance/ExternalAgentBinding.java` — binding record
- Protocol: `channel-initialised-event-observer-idempotency`
- Protocol: `optional-module-jpa-package-registration`
- Protocol: `ledger-no-credentials-or-pii-in-content`
- Protocol: `observer-test-transaction-discipline`
- Protocol: `scheduled-service-cross-tenant-stores`
