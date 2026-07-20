# Channel Protocol Enforcement SPI — Design Spec

**Issue:** casehubio/qhorus#357
**Date:** 2026-07-20
**Status:** Approved

## Problem

Channels have semantics (BARRIER, COLLECT, etc.) but no message-sequence validation.
Two agents can exchange STATUS messages forever without progress. The commitment
lifecycle enforces COMMAND→terminal, but non-COMMAND interactions (QUERY→RESPONSE,
STATUS exchanges) have no protocol constraints.

PwC demonstrated 7x accuracy gains through structured orchestration — agents
operating within defined protocols dramatically outperform unstructured communication.

The pathology watchdog (#354) validated these concerns empirically. Its conditions —
LOOP_DETECTED, CONVERSATION_STALL, ECHO_CHAMBER, OBLIGATION_FAN_OUT — detect
coordination breakdowns that occur in practice. Protocol enforcement addresses the
same pathologies proactively:

| Watchdog pathology | Protocol mitigation |
|---|---|
| LOOP_DETECTED (same sender monopolising) | ROUND_ROBIN enforces turn-taking; CONTRIBUTION_REQUIRED limits consecutive messages |
| CONVERSATION_STALL (no terminal resolution) | REQUEST_RESPONSE surfaces unanswered QUERYs; TASK_COMPLETION surfaces stalled COMMANDs |
| ECHO_CHAMBER (content relayed without transformation) | CONTRIBUTION_REQUIRED ensures diverse participants contribute |
| OBLIGATION_FAN_OUT (COMMANDs without progress) | TASK_COMPLETION advises when open obligations accumulate |

The watchdog detects and alerts; protocols advise at dispatch time before pathologies
develop. They are complementary — the watchdog fires when protocols are absent or
ignored.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Enforcement mode | All advisory | Protocol violations are coordination quality issues, not security/resource boundaries. Hard-rejection creates brittleness in LLM systems — agents retry on rejection, creating worse loops. Advisory gives actionable feedback for self-correction. Escape hatch: custom `MessageTypePolicy` for hard enforcement. |
| Composability | Multiple protocols per channel | `List<String>` on Channel. Infrastructure cost is marginal (same pattern as `allowedWriters`). Built-in protocols are orthogonal concerns — combining them should be declarative. |
| State management | Stateless — derive from message history | Bounded lookback query (configurable, default 50). No new entity, no sync bugs. Derived state is always consistent with actual history. Performance optimisable later with Caffeine cache if needed. |
| Built-in protocols | All 4 (including CommitmentStore wrappers) | REQUEST_RESPONSE and TASK_COMPLETION are thin wrappers. Uniformity — all constraints visible through one mechanism (`Channel.protocols`). |
| Participants | New `protocolParticipants` field | Dedicated field avoids overloading `barrierContributors`. Nullable — null means derive from distinct senders in the lookback window. ROUND_ROBIN requires explicit participants (rejected at `set_channel_protocols` if null). |
| Round model | No explicit rounds | CONTRIBUTION_REQUIRED uses inter-sender contribution gaps (max consecutive messages from one sender without others contributing). Avoids artificial "round" concept. |
| Registry pattern | ProjectionRegistry pattern | CDI discovery at startup, duplicate name validation, unknown names warned at dispatch. Proven Qhorus pattern. |

## SPI Contract

### ChannelProtocol (api/spi/)

```java
public interface ChannelProtocol {
    String protocolName();
    List<String> evaluate(ProtocolContext context);
}
```

`evaluate()` returns advisory strings (empty list = no violations). Matches
`CorrelationIntegrityChecker.check()` semantics. All protocols share the same
`ProtocolContext` — one lookback query and one commitment query per dispatch,
not per protocol.

**Advisory prefix convention:** all advisory strings MUST start with
`[PROTOCOL_NAME] ` (e.g., `[ROUND_ROBIN] expected 'agent-B' to speak next`).
This enables consumers to attribute advisories to their source protocol.

### ProtocolContext (api/spi/)

```java
public record ProtocolContext(
    UUID channelId,
    String channelName,
    MessageType incomingType,
    String sender,
    String correlationId,
    List<String> protocolParticipants,
    List<MessageView> recentMessages,
    List<Commitment> activeCommitments
) {}
```

`recentMessages` is populated by `MessageService` with a bounded lookback query
before calling protocols. Lookback size configured via
`casehub.qhorus.protocol.lookback-size` (default: 50). EVENT messages excluded.
**Ordering: oldest-first (ascending by ID).** Protocol analysis reads naturally as
a chronological sequence — "consecutive messages from sender A" and "last message
from a participant" are intuitive scanning forward. The store method queries
`ORDER BY id DESC LIMIT N` for efficiency, then reverses before populating the
context.

`activeCommitments` is populated by `CommitmentStore.findOpenByChannelId(channelId)`
— all OPEN or ACKNOWLEDGED commitments for the channel. Pre-queried once in the
dispatch pipeline alongside `recentMessages`. This keeps the SPI contract
self-contained: protocol implementations evaluate from `ProtocolContext` alone,
with no CDI injection required.

`protocolParticipants` comes from `Channel.protocolParticipants()`. When null,
protocols that need participants derive them from distinct senders in the lookback
window. Exception: ROUND_ROBIN requires explicit participants (see §Built-in
Protocols).

## Channel Record Changes

Two new fields on `Channel`:

```java
List<String> protocols              // empty = no protocols (normalised from null)
List<String> protocolParticipants   // empty = derive from membership/history (normalised from null)
```

`protocols` holds protocol names (e.g., `["ROUND_ROBIN", "CONTRIBUTION_REQUIRED"]`).
Validated at dispatch time against `ProtocolRegistry` — unknown names produce an
advisory ("unknown protocol 'X' on channel 'Y', skipped") rather than throwing,
for rollback safety.

`ChannelCreateRequest` gains matching builder methods: `.protocols(List<String>)`
and `.protocolParticipants(List<String>)`.

Compact constructor normalises: null → `List.of()` for both fields (same pattern
as `allowedWriters`, `barrierContributors`, `adminInstances`, `reviewerInstances`).
Empty list means "none configured." The null-preserving pattern used by
`allowedTypes`/`deniedTypes` is not appropriate here — protocols have no semantic
distinction between null and empty.

### Flyway V39

```sql
ALTER TABLE channel ADD COLUMN protocols TEXT;
ALTER TABLE channel ADD COLUMN protocol_participants TEXT;
```

Both nullable, CSV-stored, same pattern as `allowed_writers`.

## ProtocolRegistry

`runtime/message/protocol/ProtocolRegistry.java`, `@ApplicationScoped`.

- Collects `@Any Instance<ChannelProtocol>` at `@PostConstruct`
- Validates no duplicate `protocolName()` values (`IllegalStateException` at startup)
- Validates no null/blank names
- `forProtocols(List<String> names)` — returns matched protocols in declaration
  order, skips unknown names with WARN log
- `allNames()` — returns sorted set for MCP `list_protocols` tool

Package-private `ProtocolRegistry(List<>)` constructor for CDI-free unit tests
(same pattern as `ProjectionRegistry`).

## Dispatch Pipeline Integration

Protocol evaluation runs after `CorrelationIntegrityChecker`, before LAST_WRITE:

```
paused → ACL → rate limit → obligor trust → MessageTypePolicy
       → CorrelationIntegrityChecker → ProtocolEvaluation → LAST_WRITE → persist
```

Position rationale:
- Protocols need the message type to be valid (after MessageTypePolicy)
- Protocols need correlation context (after CorrelationIntegrityChecker)
- Protocols must run before persist — advisories must be in the DispatchResult

In `MessageService.dispatch()`:

```java
if (ch != null && !ch.protocols().isEmpty()) {
    List<ChannelProtocol> active = protocolRegistry.forProtocols(ch.protocols());
    if (!active.isEmpty()) {
        List<MessageView> recent = messageStore.findRecent(ch.id(), config.protocol().lookbackSize());
        List<Commitment> commitments = commitmentStore.findOpenByChannelId(ch.id());
        ProtocolContext ctx = new ProtocolContext(
            ch.id(), ch.name(), dispatch.type(), dispatch.sender(),
            dispatch.correlationId(), ch.protocolParticipants(), recent, commitments);
        for (ChannelProtocol protocol : active) {
            List<String> violations = protocol.evaluate(ctx);
            for (String v : violations) { LOG.warn(v); }
            advisories.addAll(violations);
        }
    }
}
```

`messageStore.findRecent(channelId, limit)` is a new store method — returns last
N messages ordered by ID ASC (oldest-first), excluding EVENTs. The store queries
`ORDER BY id DESC LIMIT N` then reverses, so protocol analysis reads as a
chronological sequence (see §ProtocolContext).

Both queries execute unconditionally when any protocol is active. For channels
using only history-based protocols (ROUND_ROBIN, CONTRIBUTION_REQUIRED), the
commitment query is a no-op cost. This is the explicit trade-off of the
self-contained ProtocolContext design — architectural simplicity over conditional
query optimisation. If profiling shows this is significant, a future optimisation
can conditionally skip the commitment query when no active protocol's
`protocolName()` matches a commitment-dependent set.

Reactive parity: `ReactiveMessageService` gets the same block with
`messageStore.findRecentAsync()` and `commitmentStore.findOpenByChannelIdAsync()`,
composed via `Uni.combine().all().unis(...).asTuple()`.

OTel span event: `qhorus.enforcement.protocol` added after protocol evaluation.

## Built-in Protocols

Four `@ApplicationScoped` beans in `runtime/message/protocol/`.

### REQUEST_RESPONSE

Surfaces open QUERY obligations at dispatch time using `ProtocolContext.activeCommitments`.

- Filters `activeCommitments` to QUERY type
- Advisory on new QUERY: "[REQUEST_RESPONSE] N unanswered QUERYs in channel 'X' —
  consider waiting for responses" (threshold:
  `casehub.qhorus.protocol.request-response.max-open-queries`, default 3)
- Advisory on non-RESPONSE when open QUERYs exist: "[REQUEST_RESPONSE] channel 'X'
  has open QUERYs awaiting RESPONSE"

No CDI injection required — evaluates from ProtocolContext alone.

### TASK_COMPLETION

Surfaces open COMMAND obligations at dispatch time using `ProtocolContext.activeCommitments`.

- Filters `activeCommitments` to COMMAND type
- Advisory on new COMMAND: "[TASK_COMPLETION] N open COMMANDs in channel 'X' —
  consider resolving existing tasks" (threshold:
  `casehub.qhorus.protocol.task-completion.max-open-commands`, default 3)
- Advisory when sender is obligor with open obligation: "[TASK_COMPLETION] you have
  an open obligation in channel 'X' — consider sending DONE/FAILURE/DECLINE"

No CDI injection required — evaluates from ProtocolContext alone.

### ROUND_ROBIN

Enforced turn-taking with explicit participant ordering.

- Turn order from `protocolParticipants` (**required** — `set_channel_protocols`
  rejects ROUND_ROBIN when `protocolParticipants` is empty). Turn-taking without
  a defined order is meaningless; deriving participants from message history produces
  non-deterministic ordering that changes as the lookback window slides.
- Current turn: find the most recent message from a participant in
  `recentMessages` (last participant message in the oldest-first list),
  advance to the next participant in the declared order (wrapping).
  Messages from senders not in `protocolParticipants` are ignored for turn
  determination — system messages, admin actions, and observer agents do not
  advance the turn counter or trigger advisories.
- Advisory on out-of-turn: "[ROUND_ROBIN] expected 'agent-B' to speak next
  in channel 'X', got 'agent-A'"
- Skips evaluation when ≤1 participant or no participant message history

### CONTRIBUTION_REQUIRED

Inter-sender contribution gap detection.

- Scans `recentMessages` for consecutive messages from the same sender
- Advisory when sender has N consecutive messages without all other participants
  contributing: "[CONTRIBUTION_REQUIRED] 'agent-A' has sent N consecutive messages
  in channel 'X' without contributions from: agent-B, agent-C"
  (threshold: `casehub.qhorus.protocol.contribution-required.max-consecutive`, default 2)
- Participants from `protocolParticipants` (falls back to distinct senders from lookback)
- EVENT messages excluded from the scan

**Bounded-lookback limitation:** the lookback window (default 50 messages) creates
a recency bias. If a participant contributed outside the window, the protocol
cannot see that contribution and may fire a false advisory. The `lookback-size`
should be tuned relative to expected participant count and message volume. This
is inherent to the stateless-with-bounded-lookback design — the tradeoff is
correctness at the tail vs. no per-channel state entity.

## Store Changes

### MessageStore

- `findRecent(UUID channelId, int limit)` — last N messages ordered oldest-first
  (ascending by ID), excluding EVENTs. Returns `List<MessageView>`. Implementation
  queries `ORDER BY id DESC LIMIT N` then reverses for chronological analysis.
- `ReactiveMessageStore.findRecentAsync(UUID channelId, int limit)` — reactive
  counterpart returning `Uni<List<MessageView>>`.

### CommitmentStore

- `findOpenByChannelId(UUID channelId)` — all active (OPEN or ACKNOWLEDGED)
  commitments for a channel. "Open" follows the existing `findAllOpen()` convention
  where "open" means `CommitmentState.isActive()`, not just `CommitmentState.OPEN`.
  An ACKNOWLEDGED commitment has received a STATUS but the obligation is still
  outstanding — protocols must see it.
- `ReactiveCommitmentStore.findOpenByChannelIdAsync(UUID channelId)` — reactive
  counterpart.

Both follow existing store patterns: interface in `api/store/`, JPA impl in
`runtime/`, InMemory in `persistence-memory/`, contract tests in
`persistence-memory/src/test/`.

## Config

Under `QhorusConfig`, new `Protocol` sub-interface:

```
casehub.qhorus.protocol.lookback-size=50
casehub.qhorus.protocol.request-response.max-open-queries=3
casehub.qhorus.protocol.task-completion.max-open-commands=3
casehub.qhorus.protocol.contribution-required.max-consecutive=2
```

## MCP Tools

**New tools (4):**

- `list_protocols()` — sorted names from ProtocolRegistry
- `set_channel_protocols(channel, protocols)` — full-replacement, validates names
  against registry. Additional validation:
  - ROUND_ROBIN requires `protocolParticipants` to be non-empty (error if absent)
  - Warns on questionable semantic-protocol combinations (see §Protocol-Semantic
    Interactions below)
  - Warns on redundant compositions (e.g., TASK_COMPLETION on a channel whose
    `allowedTypes` excludes COMMAND)
- `set_protocol_participants(channel, participants)` — full-replacement
- `get_channel_protocols(channel)` — current protocols and participants

**Modified (1):**

- `create_channel` gains `protocols` and `protocol_participants` parameters

Both blocking and reactive tool classes.

## Protocol-Semantic Interactions

Not all protocol-semantic combinations are meaningful. `set_channel_protocols`
produces WARN-level advisories for questionable combinations. It does not
hard-reject — the enforcement system is advisory-only, and hard-rejecting
combinations would reduce extensibility for custom protocols.

| Semantic | + ROUND_ROBIN | + CONTRIBUTION_REQUIRED | + REQUEST_RESPONSE | + TASK_COMPLETION |
|----------|--------------|------------------------|--------------------|-------------------|
| APPEND | Valid | Valid | Valid | Valid |
| BARRIER | WARN: barrier manages contribution flow; turn-taking conflicts with all-at-once collection | WARN: redundant — barrier already requires all `barrierContributors` | Valid | Valid |
| COLLECT | WARN: same conflict as BARRIER | WARN: same redundancy as BARRIER | Valid | Valid |
| LAST_WRITE | WARN: single authoritative writer makes turn-taking meaningless | WARN: single writer makes contribution tracking meaningless | Valid | Valid |
| EPHEMERAL | WARN: messages cleared after read; lookback-based protocols see incomplete history | WARN: same lookback unreliability | WARN: same lookback unreliability | WARN: same lookback unreliability |

REQUEST_RESPONSE and TASK_COMPLETION operate on `activeCommitments` (not the
lookback window), so they are valid with most semantics. They only WARN with
EPHEMERAL because the commitment lifecycle is independent of message ephemerality —
but the advisory text referencing channel names may confuse if messages have been
cleared.

## Testing

- **Protocol beans:** CDI-free unit tests with constructed `ProtocolContext`
  (including test `activeCommitments` for REQUEST_RESPONSE and TASK_COMPLETION).
  No mocks required — protocols are pure functions of ProtocolContext. Each protocol
  tests: no-violation path, single violation, multiple violations, empty channel
  (no history), single participant edge case, non-participant sender (ROUND_ROBIN).

- **ProtocolRegistry:** CDI-free unit tests with package-private constructor.
  Tests: duplicate name rejection, unknown name handling, empty list, ordering.

- **MessageService integration:** `@QuarkusTest` with channels configured with
  protocols, asserting advisories in `DispatchResult`. Uses `@TestTransaction`
  for isolation.

- **Store contract tests:** `MessageStoreContractTest` gains `findRecent` tests.
  `CommitmentStoreContractTest` gains `findOpenByChannelId` tests. Both blocking
  and reactive runners.

- **ToolOverloadDiscoverabilityTest:** updated for new MCP tool methods.

- **Reactive parity:** `ReactiveMessageService` protocol block mirrors blocking
  path. `@Disabled` reactive integration tests (Docker-dependent) follow existing
  pattern.
