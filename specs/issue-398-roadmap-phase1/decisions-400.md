# Decisions — #400 Active Governance Policies

## D1: Enforcement scope

**Choice:** All advisory sources are subject to enforcement mode by default, with per-channel runtime exclusions to reduce coverage
**Alternatives:**
- Protocols only — simpler but leaves type and correlation advisories permanently informational, limiting compliance coverage
- Design-time fixed subset — removes runtime adaptability; operators can't tune enforcement to their context
**Rationale:** Enforcement should default to maximum coverage. Operators who find specific sources too aggressive for their workload exclude them at runtime. Full range by default, configuration to reduce.
**Trade-offs:** Per-channel exclusions add a field to Channel and a migration column. Tagging advisories with source identifiers requires changes to all three advisory producers.
**Sources:** MessageTypePolicy.java, CorrelationIntegrityChecker.java, ChannelProtocol SPI, MessageService.dispatch() enforcement gate
**Exploration:** quick
**Status:** captured

## D2: Block action

**Choice:** Throw IllegalStateException (consistent with existing hard enforcement)
**Alternatives:**
- Return DispatchResult with blocked=true + blockReason — breaks the contract that a returned DispatchResult means success; every caller must check a new field; messageId would be null
- New EnforcementBlockedException — distinct from existing enforcement throws but adds a new exception type without clear benefit since @WrapBusinessError already handles IllegalStateException
**Rationale:** All existing hard enforcement (paused, ACL, rate limit, trust, COMMAND/QUERY type validation) throws IllegalStateException. Enforcement mode is another hard enforcement gate — same pattern. @WrapBusinessError wraps it into ToolCallException for MCP callers. REST callers get appropriate HTTP error.
**Trade-offs:** No structured violation data in the exception (just a message string). Callers can't distinguish enforcement blocks from other IllegalStateExceptions without parsing the message. Acceptable because enforcement EVENTs provide structured audit data.
**Sources:** MessageService.dispatch() lines 131-217 (existing throw sites), @WrapBusinessError pattern
**Exploration:** quick
**Status:** captured

## D3: Quarantine action

**Choice:** Block (throw) AND contain (pause channel + expire commitments)
**Depends on:** D2 (throw on block)
**Alternatives:**
- Dispatch + contain — let the violating message through but trigger containment as post-dispatch side effect. Preserves the problematic message for diagnosis but defeats the purpose of enforcement.
- Block + contain + audit EVENT — same as chosen but with explicit audit EVENT. Audit EVENT is captured separately in D4.
**Rationale:** QUARANTINE is the most severe enforcement mode. Blocking prevents the violating message; containment prevents future messages on the channel. Uses the same primitives as #399 watchdog containment (channelService.pause(), commitmentService.expireByChannel()). No agent deregistration — that's a watchdog concern (pattern-based detection), not enforcement (single-message-based).
**Trade-offs:** Aggressive — a single protocol violation quarantines the entire channel. Operators who want softer containment use BLOCKING mode instead.
**Sources:** WatchdogEvaluationService.executeContainmentAction(), CommitmentService.expireByChannel(), ChannelService.pause()
**Exploration:** quick
**Status:** captured

## D4: Enforcement audit trail

**Choice:** Dispatch enforcement EVENT (sender="system:enforcement") to the violating channel before throwing
**Depends on:** D2 (throw on block), D3 (quarantine contains)
**Alternatives:**
- No ledger entry — blocked dispatches invisible to auditors; compliance gap for E5
- Standalone ledger entry (no corresponding Message) — breaks the Message↔LedgerEntry 1:1 invariant; more invasive
**Rationale:** Enforcement actions must be auditable. An EVENT dispatched through the normal pipeline creates both a Message and a LedgerEntry, appears in channel timeline, is queryable via list_ledger_entries. Reuses existing infrastructure. EVENT type is exempt from ACL, rate limiting, and enforcement mode — no recursion risk.
**Trade-offs:** Enforcement EVENT is dispatched to a channel that's about to be paused (QUARANTINE) or that just rejected a message (BLOCKING). Order matters: EVENT must be dispatched before pause. System senders must be exempt from enforcement to prevent recursion.
**Sources:** WatchdogEvaluationService.dispatchContainmentEvent() pattern, MessageService.dispatch() EVENT exemptions (lines 170, 183)
**Exploration:** quick
**Status:** captured
