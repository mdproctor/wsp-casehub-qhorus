## D1: Scope — A2A spec compliance only

**Choice:** Per-task push notifications as defined by the A2A protocol spec (server-side). When an external client sends `message/send` with a `TaskPushNotificationConfig`, Qhorus pushes `TaskStatusUpdateEvent` back as the task progresses.
**Alternatives:**
- General channel push — push channel messages to registered external agents via A2A format. Broader but less protocol-aligned and conflates two concerns.
- Both — unified design covering spec compliance + general push. Larger scope without clear incremental value over separate issues.
**Rationale:** Completes the A2A interop story from #396. Per-task push is the protocol-defined mechanism; general channel push is a different capability that can be a separate issue if needed.
**Trade-offs:** Does not cover pushing arbitrary channel events to external agents — only task-scoped updates triggered by the submitter's push config.
**Sources:** A2A protocol protobuf spec (`a2a.proto`), issue #396 (A2A interop audit), `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`
**Exploration:** quick
**Status:** captured

## D2: Module placement — new optional module

**Choice:** New optional Maven module `a2a-push-notification/` activating by classpath presence.
**Alternatives:**
- Inside runtime/ — simpler dependency graph but adds weight to the always-included runtime module. Push is optional infrastructure, not core.
- Inside a2a-outbound/ — reuses ExternalAgentBinding but conflates outbound (Qhorus as client) with push (Qhorus as server) concerns.
**Rationale:** Follows the established pattern (webhook-observer, kafka-observer, a2a-outbound). Push is optional infrastructure. Keeps runtime lean for deployments that don't need A2A push.
**Trade-offs:** One more Maven module to maintain. Cross-module integration via `Instance<>` adds indirection.
**Sources:** `webhook-observer/`, `kafka-observer/`, `a2a-outbound/` module patterns
**Exploration:** quick
**Status:** captured

## D3: Delivery model — persistent outbox with retry

**Choice:** JPA-backed push outbox table. MessageObserver writes push events to outbox. `@Scheduled` job retries failed pushes with exponential backoff (5s, 30s, 2m, 10m, 1h). Health tracking per endpoint. Entries abandoned after 5 attempts.
**Alternatives:**
- Fire-and-forget (virtual threads) — simple but unreliable. Issue #406 explicitly requests "retry + health tracking."
- Delivery pump adaptation — register push backend per-channel, reuse cursor + retry. Impedance mismatch: pump is per-channel-per-backend with cursor tracking; push is per-task (correlationId-scoped). Would require forcing per-task semantics into per-channel infrastructure.
**Rationale:** A2A push is per-task, not per-channel. The delivery pump's cursor-based model doesn't map to task-scoped events. An outbox gives reliable delivery without the impedance mismatch, while reusing the same retry/health concepts.
**Trade-offs:** New persistence infrastructure (outbox table + scheduled job) rather than reusing the existing delivery pump. Duplicates some retry/health logic.
**Sources:** `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryService.java`, webhook-observer fire-and-forget pattern, A2A spec `TaskPushNotificationConfig`
**Exploration:** quick
**Status:** captured

## D4: Push config storage — new PushNotificationConfig entity

**Choice:** New JPA entity `PushNotificationConfigEntity` in the a2a-push-notification module, stored in the qhorus named PU. Fields: id, taskId (correlationId), channelId, url, token, authScheme, authCredentialsRef, tenancyId, createdAt.
**Alternatives:**
- Extend ExternalAgentBinding — adds pushUrl/pushToken fields to ExternalAgentBinding. Conflates per-agent config (outbound) with per-task config (push). A2A spec is per-task, not per-agent.
- In-memory only — ConcurrentHashMap keyed by correlationId. Simple but lost on restart. Unacceptable for long-running tasks.
**Rationale:** Push configs are per-task (per-correlationId), transient by nature (lifetime of the task), and need to survive restarts. A dedicated entity keeps concerns clean and follows the WebhookRegistrationEntity pattern.
**Trade-offs:** New JPA entity requires Flyway migration. Consumers must register the module's package in `quarkus.hibernate-orm.qhorus.packages`.
**Sources:** `webhook-observer/.../WebhookRegistrationEntity.java`, `api/.../ExternalAgentBinding.java`, A2A spec `TaskPushNotificationConfig(id, task_id, url, token, authentication)`, protocol `optional-module-jpa-package-registration`
**Exploration:** quick
**Status:** captured

## D5: Push trigger — MessageObserver

**Choice:** `PushNotificationObserver` implements `MessageObserver` with `Scope.LOCAL`. Watches all dispatched messages. For messages with correlationIds that have active push configs AND message types that represent task progress (STATUS, DONE, FAILURE, DECLINE, HANDOFF, RESPONSE), writes a `TaskStatusUpdateEvent` to the outbox.
**Alternatives:**
- CDI event observers — observe CommitmentDeclinedEvent, CommitmentExpiredEvent, etc. More targeted but misses STATUS messages (which don't change commitment state) and would require new CDI events for every transition type.
- Commitment state change SPI hook — most precise but requires a new SPI in CommitmentService. STATUS messages don't change commitment state, so they'd be missed.
**Rationale:** MessageObserver fires after commit for all message types. It sees STATUS (which is the most common task update) as well as terminal types. LOCAL scope is correct because the outbox handles dedup — only one node needs to enqueue.
**Trade-offs:** Observer fires for ALL messages, not just push-relevant ones. The correlationId lookup against the push config store adds a query per message dispatch. Mitigated by in-memory cache of active push config correlationIds.
**Sources:** `webhook-observer/.../WebhookMessageObserver.java`, `api/.../MessageObserver.java`, protocol `observer-test-transaction-discipline`, `A2ATaskStateMapper`
**Exploration:** quick
**Status:** captured

## D6: API surface — extend A2AResource

**Choice:** Extend existing `A2AResource` JSON-RPC dispatch with push notification config CRUD methods. Plus inline push config support in `message/send` via the params object. `Instance<PushNotificationConfigStore>` for optional module detection — 501 when absent. `AgentCardResource` sets `pushNotifications = true` when store is resolvable.
**Alternatives:**
- Separate resource in module — new `PushNotificationResource` at its own REST path. Decoupled but splits the A2A protocol surface across two modules. Clients would need to know two endpoints.
- Both JSON-RPC and REST — JSON-RPC in A2AResource plus REST in the module. Most complete but two API surfaces for the same operations.
**Rationale:** Push notification config management is part of the A2A protocol — it belongs with the A2A JSON-RPC dispatch. `Instance<>` is the established pattern for optional feature detection (already used by A2AResource for other optional features). The `PushNotificationConfigStore` interface goes in `api/store/` following the store taxonomy.
**Trade-offs:** A2AResource gains complexity (4 more JSON-RPC methods + inline config parsing). The store interface in api/ means the api module gains awareness of push notification concepts even when the implementation module is absent.
**Sources:** `runtime/.../A2AResource.java`, `runtime/.../AgentCardResource.java`, `api/store/` store pattern
**Exploration:** quick
**Status:** captured
