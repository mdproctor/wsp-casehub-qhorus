# Decisions — #400 Active Governance Policies

## D1: Enforcement scope

**Choice:** All advisory sources are subject to enforcement mode by default, with per-channel runtime exclusions to reduce coverage. Terminal resolution types (DONE, FAILURE, DECLINE, RESPONSE, HANDOFF) are unconditionally exempt — obligation resolution must never be blocked.
**Alternatives:**
- Protocols only — simpler but leaves type and correlation advisories permanently informational, limiting compliance coverage
- Design-time fixed subset — removes runtime adaptability; operators can't tune enforcement to their context
- No terminal type exemption — creates unresolvable obligations when a DONE is blocked on a typed channel (ADR-0016 specifically decided against this)
**Rationale:** Enforcement should default to maximum coverage. Operators who find specific sources too aggressive for their workload exclude them at runtime. Full range by default, configuration to reduce. Terminal types are exempt because blocking a DONE/FAILURE/DECLINE on a typed channel leaves the originating COMMAND/QUERY commitment permanently OPEN with no resolution path.
**Trade-offs:** Per-channel exclusions add a field to Channel and a migration column. Tagging advisories with source identifiers requires changes to all three advisory producers.
**Sources:** MessageTypePolicy.java, CorrelationIntegrityChecker.java, ChannelProtocol SPI, MessageService.dispatch() enforcement gate, ADR-0016 (hybrid-channel-type-enforcement)
**Exploration:** quick
**Status:** revised (R1-03: terminal type exemption added)

## D2: Block action

**Choice:** Throw `EnforcementBlockedException extends IllegalStateException` with structured violation data
**Alternatives:**
- Return DispatchResult with blocked=true + blockReason — breaks the contract that a returned DispatchResult means success; every caller must check a new field; messageId would be null
- Plain IllegalStateException — backward-compatible but callers can't distinguish enforcement blocks from ACL/rate-limit/trust failures without string-parsing. 6+ uses of IllegalStateException in dispatch() already.
**Rationale:** `EnforcementBlockedException extends IllegalStateException` is zero-cost and backward-compatible — existing catch blocks, @WrapBusinessError interceptors, and REST exception mappers all handle it unchanged. Programmatic discrimination via `catch (EnforcementBlockedException)` enables agents to take different corrective action than for ACL or rate-limit failures. Same pattern as `MessageTypeViolationException` for type constraint violations. Carries structured data: enforcement mode, violation sources, violation messages.
**Trade-offs:** One new class file. Callers not aware of the subclass continue to work unchanged.
**Sources:** MessageService.dispatch() lines 131-217 (existing throw sites), @WrapBusinessError pattern, MessageTypeViolationException pattern
**Exploration:** quick
**Status:** revised (R1-04: upgraded from plain IllegalStateException)

## D3: Quarantine action

**Choice:** Block (throw) AND contain (pause channel + expire commitments), with containment actions in REQUIRES_NEW transaction
**Depends on:** D2 (throw on block)
**Alternatives:**
- Dispatch + contain — let the violating message through but trigger containment as post-dispatch side effect. Preserves the problematic message for diagnosis but defeats the purpose of enforcement.
- Containment in same transaction — the throw marks the JTA transaction for rollback, undoing pause and expire. The entire quarantine mechanism is inoperative.
**Rationale:** QUARANTINE is the most severe enforcement mode. Blocking prevents the violating message; containment prevents future messages on the channel. Uses the same primitives as #399 watchdog containment (channelService.pause(), commitmentService.expireByChannel()). No agent deregistration — that's a watchdog concern (pattern-based detection), not enforcement (single-message-based). Containment actions must execute in REQUIRES_NEW to survive the outer transaction's rollback. Extract to a separate CDI bean (EnforcementExecutor) analogous to ChannelCreateHelper.
**Trade-offs:** Aggressive — a single protocol violation quarantines the entire channel. Operators who want softer containment use BLOCKING mode instead. REQUIRES_NEW adds a new bean and transaction boundary.
**Sources:** WatchdogEvaluationService.executeContainmentAction(), CommitmentService.expireByChannel(), ChannelService.pause(), ChannelCreateHelper.createInNewTransaction() (REQUIRES_NEW precedent)
**Exploration:** quick
**Status:** revised (R1-02: REQUIRES_NEW extraction for transaction safety)

## D4: Enforcement audit trail

**Choice:** Dispatch enforcement EVENT to the violating channel (in REQUIRES_NEW) AND fire `EnforcementBlockedEvent` CDI async event for external notification
**Depends on:** D2 (throw on block), D3 (quarantine contains + REQUIRES_NEW)
**Alternatives:**
- No ledger entry — blocked dispatches invisible to auditors; compliance gap for E5
- Standalone ledger entry (no corresponding Message) — breaks the Message↔LedgerEntry 1:1 invariant; more invasive
- Channel EVENT only, no CDI event — enforcement visible only by querying each channel individually; no centralized monitoring; no external notification (Slack, webhooks)
**Rationale:** Enforcement actions must be auditable. The channel EVENT creates a Message+LedgerEntry (audit record, queryable via list_ledger_entries). The CDI event (`EnforcementBlockedEvent`) provides centralized monitoring — flows through ConnectorAlertBridge to external systems (Slack, webhooks), consistent with `WatchdogAlertEvent` pattern. Both execute in REQUIRES_NEW to survive the outer rollback. System senders (containing `:`) are exempt from enforcement entirely (not just EVENTs), consistent with trust gate exemption.
**Trade-offs:** Two notification paths (channel EVENT + CDI event). The channel EVENT provides per-channel audit trail; the CDI event provides operational monitoring. Both are needed for compliance evidence (E5).
**Sources:** WatchdogEvaluationService.dispatchContainmentEvent() pattern, WatchdogAlertEvent CDI pattern, ConnectorAlertBridge, MessageService.dispatch() EVENT exemptions (lines 170, 183)
**Exploration:** quick
**Status:** revised (R1-02: REQUIRES_NEW, R1-06/R1-07: CDI event added, R1-09: system sender exemption)

## D5: Per-channel enforcement configuration (no inheritance)

**Choice:** Enforcement mode is per-channel only; no Space-level or global default inheritance in Phase 1
**Alternatives:**
- Space-level default with per-channel override — reduces operational burden for large deployments; Space entity already supports parentSpaceId for recursive nesting
- Global default with Space and channel overrides — three-tier resolution hierarchy
**Rationale:** Phase 1 validates the enforcement model with explicit per-channel configuration. Inheritance adds resolution complexity (channel overrides Space overrides global) and requires changes to both Space entity and enforcement resolution logic. Per-channel is the minimum viable approach. Once enforcement is proven in production, Space-level inheritance is a natural Phase 2 enhancement.
**Trade-offs:** Operators with many channels must configure each individually. MCP bulk-set operations and channel templates mitigate this. No inheritance means no accidental enforcement propagation — explicit is safer for a new governance mechanism.
**Sources:** Space entity (parentSpaceId), Channel entity (enforcementMode field)
**Exploration:** quick
**Status:** captured (R1-08: implicit decision made explicit)
