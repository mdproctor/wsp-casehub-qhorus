## D1: Scope — A2A spec compliance only (server-side push)

**Choice:** Per-task push notifications as defined by the A2A protocol spec (server-side). When an external client sends `message/send` with a `TaskPushNotificationConfig`, Qhorus pushes `TaskStatusUpdateEvent` back as the task progresses. Client-side push subscription (Qhorus as A2A client receiving push from external agents via `a2a-outbound`) is explicitly excluded — it requires push registration in `A2AOutboundBackend`, which is a separate capability tracked for a follow-up issue.
**Alternatives:**
- General channel push — push channel messages to registered external agents via A2A format. Broader but less protocol-aligned and conflates two concerns.
- Both — unified design covering spec compliance + general push. Larger scope without clear incremental value over separate issues.
- Server + client push — include Qhorus-as-consumer push registration in A2AOutboundBackend. Conflates two independent capabilities with different integration points.
**Rationale:** Completes the A2A interop story from #396. Per-task push is the protocol-defined mechanism; general channel push is a different capability that can be a separate issue if needed. Client-side push subscription is architecturally distinct (requires `A2AOutboundBackend` extensions, client-side push URL registration, and inbound event handling) and is better scoped as a follow-up.
**Trade-offs:** Does not cover pushing arbitrary channel events to external agents — only task-scoped updates triggered by the submitter's push config. Does not cover receiving push notifications from external agents that Qhorus invokes via a2a-outbound.
**Sources:** A2A protocol protobuf spec (`a2a.proto`), issue #396 (A2A interop audit), `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`
**Exploration:** quick
**Status:** revised — R1-04: added explicit scope exclusion for client-side push subscription with rationale

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

## D3: Delivery model — ChannelBackend with AT_LEAST_ONCE, non-throwing post()

**Choice:** `PushNotificationBackend` implements `ChannelBackend` with `DeliveryGuarantee.AT_LEAST_ONCE`. The delivery pump calls `post()` for every message on registered channels. `post()` filters by correlationId — messages whose correlationId matches an active push config trigger an HTTP POST of a `TaskStatusUpdateEvent` to the configured URL; non-matching messages are a no-op. **Critically, `post()` never throws.** HTTP delivery failures are caught internally; the cursor always advances. The push backend manages per-URL health tracking independently of the pump's per-backend circuit breaker.
**Failure contract — differs from A2AOutboundBackend:** `A2AOutboundBackend.post()` throws on HTTP failure — acceptable because each channel typically routes to one external agent, so cursor starvation affects only that channel's one target. Push notifications have multiple independent targets per channel (different clients, different push URLs, different correlationIds). If `post()` threw, one dead push URL would stall the cursor and block all other push URLs on the same channel. After `maxConsecutiveFailures` (default 10), the entire push backend would be marked unhealthy globally — all channels lose push delivery. Therefore `post()` catches failures internally and returns normally.
**Per-URL health tracking within the push backend:** The push backend maintains a `ConcurrentHashMap<String, UrlHealthState>` tracking per-URL failure count, last failure timestamp, and exponential backoff window. On HTTP failure: increment per-URL failure count, apply backoff. On subsequent `post()` calls: if a URL is within its backoff window, skip (no-op). After a configurable max failures per URL, mark the URL as exhausted — delete the push config and log.
**Differentiated delivery by message criticality:**
- Non-terminal messages (STATUS, HANDOFF): best-effort push. Failed deliveries are dropped. Task state is progressive — later pushes carry the current state, superseding missed intermediate updates.
- Terminal messages (DONE, FAILURE, DECLINE): retried on subsequent `post()` calls. When a terminal push fails, the backend tracks `(correlationId, terminalState)` in-memory. On each subsequent `post()` call for the same channel, pending terminal deliveries are retried before processing the current message. On success: push config deleted (D4 tier 1). On exhaust: push config deleted, client must poll.
**AT_LEAST_ONCE declaration is for pump participation:** The `DeliveryGuarantee` enum controls how the pump handles the backend — `AT_LEAST_ONCE` means cursor-based delivery via `DeliveryBatchExecutor`, not fire-and-forget from `fanOut()`. The backend's internal HTTP delivery semantics are separate: best-effort for non-terminal, retry-with-backoff for terminal.
**Alternatives:**
- JPA-backed outbox with `@Scheduled` retry — duplicates the delivery pump's infrastructure. Creates a second delivery mechanism within qhorus.
- Throwing post() (A2AOutboundBackend pattern) — causes cursor starvation when multiple push URLs coexist on the same channel. One dead URL blocks all push delivery on that channel and, after `maxConsecutiveFailures`, globally.
- Fire-and-forget (virtual threads) — simple but unreliable. Issue #406 explicitly requests "retry + health tracking."
**Rationale:** The ChannelBackend pattern (R1-02) is architecturally correct — the pump must not be duplicated. But the failure contract must differ from `A2AOutboundBackend`: push has multi-target per-channel semantics where cursor starvation from one target is unacceptable. The non-throwing `post()` with internal per-URL health tracking preserves the pump's cursor management (ordering, dedup, reconciliation) while isolating per-URL failures. Terminal retry ensures critical state transitions are delivered with reasonable effort.
**Trade-offs:** In-memory per-URL health and pending-terminal state are lost on restart. Push configs persist in JPA; the TTL cleanup (D4) eventually removes undelivered configs. Clients should treat push as supplementary and poll `tasks/get` if no push arrives within a reasonable timeout. Persistent terminal retry (flag on PushNotificationConfigEntity) can be added as a future enhancement if restart-resilient terminal delivery is required.
**Sources:** `A2AOutboundBackend.java` (ChannelBackend precedent — different failure contract), `DeliveryBatchExecutor.deliverBatch()` lines 125-154 (sequential iteration, cursor stall on throw), `DeliveryService.recordFailure()` lines 270-278 (per-backendId global unhealthy marking), `DeliveryService.processChannel()` lines 162-168 (unhealthy skip on all channels), `ChannelGateway.fanOut()` (tracked backend skip), ADR-0018 (platform SPI — does not apply)
**Exploration:** quick
**Status:** revised — R1-02/R1-03: adopted ChannelBackend pattern, eliminated outbox. R2-02: non-throwing post() with per-URL health tracking; differentiated delivery by message criticality. Cursor starvation eliminated.

## D4: Push config storage — PushNotificationConfig entity with cleanup lifecycle

**Choice:** New JPA entity `PushNotificationConfigEntity` in the a2a-push-notification module, stored in the qhorus named PU. Fields: id, taskId (correlationId), channelId, url, token, authScheme, authCredentialsRef, tenancyId, createdAt, lastPushedAt. Cleanup lifecycle: (1) When the push backend delivers a terminal-state event (DONE → "completed", FAILURE → "failed", DECLINE → "canceled"), it deletes the push config after successful delivery. (2) A `@Scheduled` TTL cleanup job removes push configs where `COALESCE(lastPushedAt, createdAt) + threshold < now()` — this preserves configs for long-running tasks that receive periodic STATUS updates while cleaning up genuinely abandoned configs. The `lastPushedAt` field is updated on each successful push delivery. (3) Client-initiated deletion via `tasks/pushNotificationConfig/delete` JSON-RPC method per the A2A spec.
**Alternatives:**
- Extend ExternalAgentBinding — adds pushUrl/pushToken fields to ExternalAgentBinding. Conflates per-agent config (outbound) with per-task config (push). A2A spec is per-task, not per-agent.
- In-memory only — ConcurrentHashMap keyed by correlationId. Simple but lost on restart. Unacceptable for long-running tasks.
**Rationale:** Push configs are per-task (per-correlationId), transient by nature (lifetime of the task), and need to survive restarts. A dedicated entity keeps concerns clean and follows the WebhookRegistrationEntity pattern. The three-tier cleanup ensures no config accumulates indefinitely: terminal delivery cleans up the normal path, TTL catches abandoned configs, and client delete handles explicit revocation. The push backend's `post()` already processes terminal messages — cleaning up the config at that point is natural and requires no additional events or infrastructure.
**Trade-offs:** New JPA entity requires Flyway migration. Consumers must register the module's package in `quarkus.hibernate-orm.qhorus.packages`. TTL cleanup job adds a `@Scheduled` task, but this is cleanup, not delivery — no duplication with the pump.
**Sources:** `webhook-observer/.../WebhookRegistrationEntity.java`, `api/.../ExternalAgentBinding.java`, A2A spec `TaskPushNotificationConfig(id, task_id, url, token, authentication)`, protocol `optional-module-jpa-package-registration`, `CommitmentState.isTerminal()` (FULFILLED, DECLINED, FAILED, DELEGATED, EXPIRED), `A2AChannelBackend` SSE deregistration pattern (consumers self-deregister on terminal events)
**Exploration:** quick
**Status:** revised — R1-06: added three-tier cleanup lifecycle (terminal delivery, TTL, client delete). R2-03: added lastPushedAt field; TTL uses COALESCE(lastPushedAt, createdAt) to prevent premature deletion of configs for long-running tasks.

## D5: Push trigger — ChannelBackend.post()

**Choice:** `PushNotificationBackend.post()` is the push trigger, called by `ChannelGateway.fanOut()` via the delivery pump. `post()` looks up the message's correlationId in the push config store (in-memory cache backed by JPA). If a matching config exists and the message type is push-relevant (STATUS, DONE, FAILURE, DECLINE, HANDOFF), it maps the message type to an A2A task state and POSTs a `TaskStatusUpdateEvent` to the configured push URL. RESPONSE is excluded — its correct A2A state mapping ("completed" vs "working") depends on context that `fromMessageType()` and `statePriority()` disagree on; DONE is the unambiguous completion signal.
**Alternatives:**
- MessageObserver writing to outbox — fires for all messages, writes to separate persistence layer. Requires its own scheduled retry. Moot with ChannelBackend adoption (D3 revision).
- CDI event observers — CommitmentStateChangedEvent etc. `CommitmentStateChangedEvent` is defined but not fired in production code. Misses STATUS messages. Would require new CDI events for each transition type.
- Commitment state change SPI hook — most precise but requires a new SPI in CommitmentService. STATUS messages don't change commitment state.
**Rationale:** With D3 revised to use ChannelBackend, `post()` IS the trigger — no separate observer, no outbox, no event bus. The delivery pump calls `post()` for each message on the channel; the push backend filters by correlationId. Push-relevant message types are the subset that represent task state transitions. RESPONSE is excluded because `A2ATaskStateMapper.fromMessageType(RESPONSE)` returns "working" while `statePriority(RESPONSE) = 4` → "completed" — a mapping inconsistency that should be resolved in the mapper itself, not papered over in the push backend.
**Trade-offs:** `post()` is called for every message on registered channels. The correlationId lookup adds a query per message dispatch, mitigated by in-memory cache of active push config correlationIds.
**Sources:** `A2AOutboundBackend.post()` (selective filtering precedent), `ChannelGateway.fanOut()` (delivery path), `A2ATaskStateMapper.fromMessageType()` / `statePriority()` (RESPONSE mapping inconsistency), `A2AChannelBackend.post()` (SSE dispatch pattern)
**Exploration:** quick
**Status:** revised — R1-07: changed from MessageObserver to ChannelBackend.post() following D3 revision. R1-11: RESPONSE excluded from push-relevant types pending mapper consistency fix.

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

## D7: Outbound push authentication — A2A spec authentication object via CredentialResolver

**Choice:** Outbound push requests authenticate using the `authentication` object from the client's `PushNotificationConfig`. The client provides `authScheme` and credentials when registering the push config. Qhorus resolves credentials via `CredentialResolver` (matching the `A2AOutboundBackend.resolveAuth()` pattern) and attaches them to the outbound HTTP POST. Supported auth types: bearer token, API key. No verification handshake or URL reachability check on push config registration — the A2A spec does not require it, and the registration endpoint is already behind the platform's auth layer. No HMAC payload signing — the A2A spec's `authentication` field authenticates the request to the target; tamper evidence is a separate concern not defined by the protocol.
**Alternatives:**
- Inline credentials (store token directly in PushNotificationConfigEntity) — simpler but bypasses the platform credential store. Credentials in a JPA entity are a security smell.
- URL verification handshake on registration — POST a challenge to the push URL before accepting the config. More secure but not required by A2A spec, adds latency to registration, and the push URL may not be ready when the config is registered.
- HMAC payload signing (like WebhookMessageObserver) — adds tamper evidence via `X-Qhorus-Signature` header. Useful but orthogonal to A2A spec compliance. Can be added as a qhorus-specific extension later if needed.
**Rationale:** The `A2AOutboundBackend.resolveAuth()` pattern is proven: `CredentialResolver.resolve(authConfigKey)` → `AuthConfig(type, null, token)`. The push backend follows the same pattern. The A2A spec defines authentication as a property of `PushNotificationConfig` — the client tells the server how to authenticate when pushing. This is the standard webhook authentication model. URL validation would reject configs where the push endpoint is deployed after registration (common in CI/CD pipelines). HMAC signing is valuable but is a spec extension, not a compliance requirement.
**Sources:** `A2AOutboundBackend.resolveAuth()`, `CredentialResolver`, `WebhookMessageObserver.hmacSha256()` (HMAC precedent — deferred), A2A spec `PushNotificationConfig.authentication`
**Exploration:** quick (surfaced by review R1-09)
**Status:** captured

## D8: Task state mapping for push events — per-message-type, RESPONSE excluded

**Choice:** Push notifications use a dedicated state mapping for each message type: STATUS → "working", HANDOFF → "working", DONE → "completed", FAILURE → "failed", DECLINE → "canceled". RESPONSE is excluded from push-relevant message types entirely. The `TaskStatusUpdateEvent` carries the mapped state plus the message content/payload as the event body.
**Alternatives:**
- Use `A2ATaskStateMapper.fromMessageType()` directly — maps RESPONSE → "working" (default case), but this loses the completion signal for QUERY obligations.
- Use `A2ATaskStateMapper.fromMessageHistory()` — most accurate aggregate state but requires reading the full message history in post(), adding a query per push. Also inherits the RESPONSE priority-4 → "completed" mapping, which conflates RESPONSE with DONE.
- Use `A2ATaskStateMapper.fromCommitmentState()` — accurate commitment-level state but requires querying the commitment store in post().
**Rationale:** `A2ATaskStateMapper` has an internal inconsistency: `fromMessageType(RESPONSE)` → "working" but `statePriority(RESPONSE) = 4` → "completed". Rather than encoding either side of this inconsistency into the push backend, RESPONSE is excluded. DONE is the unambiguous completion signal for A2A tasks. If a QUERY obligation is fulfilled by RESPONSE, the commitment transitions to FULFILLED, but the push backend does not need to signal this separately — the client can query task state if needed. The mapping inconsistency in `A2ATaskStateMapper` should be resolved independently (it also affects SSE streaming via `fromMessageHistory()`).
**Sources:** `A2ATaskStateMapper.java` (fromMessageType vs statePriority divergence), `CommitmentState` (RESPONSE → FULFILLED), A2A spec `TaskStatusUpdateEvent`
**Exploration:** quick (surfaced by review R1-11)
**Status:** captured
