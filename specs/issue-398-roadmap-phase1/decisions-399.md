# Decisions — #399 Cascade Containment

## D1: WatchdogAction enum + persistence

**Choice:** New `WatchdogAction` enum in `api/watchdog/`: ALERT (default), PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE. New `action` field on `Watchdog` record (nullable, default ALERT). V42 migration. `register_watchdog` gains `action` param.
**Alternatives:**
- Action as a separate entity/table — over-engineered for 4 enum values
- Action as String (not enum) — loses type safety, error-prone
**Rationale:** Enum is the natural fit — small fixed set of actions, type-safe, switch-exhaustive. Persisted on the watchdog so each registration controls its own response.
**Trade-offs:** Adding new actions requires an enum value + migration default. Acceptable for a pre-release project.
**Sources:** WatchdogConditionType.java (existing enum pattern), Watchdog.java (record structure), WatchdogEntity (JPA entity)
**Exploration:** quick
**Status:** captured

## D2: Containment execution in WatchdogEvaluationService

**Choice:** Private `executeContainmentAction()` method in `WatchdogEvaluationService`, called from `fireAlert()`. Switches on `w.action()` and calls existing services (ChannelService.pause, ChannelGateway.deregisterBackend).
**Alternatives:**
- Separate ContainmentService — premature abstraction for 3-5 method calls
- CDI event-driven containment (observer pattern) — adds async complexity for synchronous operations
**Rationale:** The containment logic is simple wiring between existing primitives. Keeping it in the evaluation service maintains the single control flow: detect → alert → contain. Can be extracted later if E3/E8 add complexity.
**Trade-offs:** WatchdogEvaluationService grows slightly. Mitigated by the method being self-contained.
**Sources:** WatchdogEvaluationService.java:288-315 (fireAlert pattern), ChannelService.pause(), ChannelGateway.deregisterBackend()
**Exploration:** quick
**Status:** captured

## D3: Agent extraction via AlertContext default method

**Choice:** `default List<String> affectedAgentIds()` on `AlertContext` returning `List.of()`. Overridden by LoopDetectedContext (sender), AgentStaleContext (staleInstanceIds), ContextPressureContext (actorId), EchoChamberContext (participants).
**Alternatives:**
- instanceof pattern matching in containment logic — fragile, grows with each new context type
- Separate AgentExtractor strategy — over-engineered for 4 overrides
**Rationale:** Default method on the sealed interface is the idiomatic Java approach. New context types get the safe default (empty list); agent-specific contexts override.
**Trade-offs:** Couples the "which agents" question to the context type. Correct — the context IS where the agent info lives.
**Sources:** AlertContext.java (sealed interface), LoopDetectedContext.java (sender field), AgentStaleContext.java (staleInstanceIds), ContextPressureContext.java (actorId), EchoChamberContext.java (participants)
**Exploration:** quick
**Status:** captured

## D4: Containment audit trail via EVENT dispatch

**Choice:** After any non-ALERT containment action, dispatch an EVENT to the notification channel with containment telemetry via `telemetryJson()`. Structured JSON with action, condition_type, affected_agents, channel_id.
**Alternatives:**
- Separate audit table — unnecessary; the ledger IS the audit trail
- Log-only — not queryable, not part of the normative record
**Rationale:** EVENTs in the ledger are the established audit mechanism. Containment actions are governance acts — they belong in the normative record alongside the alert that triggered them.
**Trade-offs:** None significant.
**Sources:** QhorusMcpToolsBase.telemetryJson() (existing pattern), LedgerWriteService EVENT handling
**Exploration:** quick
**Status:** captured

## D5: Commitment handling via force-expire

**Choice:** New `CommitmentService.expireByChannel(UUID channelId)` — sets `expiresAt = Instant.now()` on all OPEN/ACKNOWLEDGED commitments for the channel, fires `CommitmentExpiredEvent` for each. Called after PAUSE_CHANNEL or QUARANTINE.
**Alternatives:**
- Let the scheduler pick them up naturally — delayed; paused channel means obligations are unresolvable NOW
- DECLINE all commitments — wrong semantic; the agent didn't decline, the system intervened
**Rationale:** Expiry is the correct semantic — the obligation timed out due to containment, not agent action. Immediate expiry gives downstream systems the signal to act. `CommitmentExpiredEvent` already flows through the notification bridge.
**Trade-offs:** Commitments expire even if the channel is resumed quickly. Acceptable — a paused channel means containment was necessary, and commitments should be re-established explicitly after review.
**Sources:** CommitmentService.expireOverdue() (existing pattern), CommitmentExpiredEvent (existing CDI event), notification-bridge module
**Exploration:** quick
**Status:** captured
