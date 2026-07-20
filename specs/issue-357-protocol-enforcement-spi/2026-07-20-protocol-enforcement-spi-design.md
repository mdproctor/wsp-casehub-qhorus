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

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Enforcement mode | All advisory | Protocol violations are coordination quality issues, not security/resource boundaries. Hard-rejection creates brittleness in LLM systems — agents retry on rejection, creating worse loops. Advisory gives actionable feedback for self-correction. Escape hatch: custom `MessageTypePolicy` for hard enforcement. |
| Composability | Multiple protocols per channel | `List<String>` on Channel. Infrastructure cost is marginal (same pattern as `allowedWriters`). Built-in protocols are orthogonal concerns — combining them should be declarative. |
| State management | Stateless — derive from message history | Bounded lookback query (configurable, default 50). No new entity, no sync bugs. Derived state is always consistent with actual history. Performance optimisable later with Caffeine cache if needed. |
| Built-in protocols | All 4 (including CommitmentStore wrappers) | REQUEST_RESPONSE and TASK_COMPLETION are thin wrappers. Uniformity — all constraints visible through one mechanism (`Channel.protocols`). |
| Participants | New `protocolParticipants` field | Dedicated field avoids overloading `barrierContributors`. Nullable — null means derive from membership/history. |
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
`ProtocolContext` — one lookback query per dispatch, not per protocol.

### ProtocolContext (api/spi/)

```java
public record ProtocolContext(
    UUID channelId,
    String channelName,
    MessageType incomingType,
    String sender,
    String correlationId,
    List<String> protocolParticipants,
    List<MessageView> recentMessages
) {}
```

`recentMessages` is populated by `MessageService` with a bounded lookback query
before calling protocols. Lookback size configured via
`casehub.qhorus.protocol.lookback-size` (default: 50). EVENT messages excluded.

`protocolParticipants` comes from `Channel.protocolParticipants()`. When null,
protocols that need participants derive them from distinct senders in the lookback
window.

## Channel Record Changes

Two new fields on `Channel`:

```java
List<String> protocols              // nullable — null = no protocols
List<String> protocolParticipants   // nullable — null = derive from membership/history
```

`protocols` holds protocol names (e.g., `["ROUND_ROBIN", "CONTRIBUTION_REQUIRED"]`).
Validated at dispatch time against `ProtocolRegistry` — unknown names produce an
advisory ("unknown protocol 'X' on channel 'Y', skipped") rather than throwing,
for rollback safety.

`ChannelCreateRequest` gains matching builder methods: `.protocols(List<String>)`
and `.protocolParticipants(List<String>)`.

Compact constructor normalises: null → `List.of()` for both fields (same pattern
as `allowedWriters`).

### Flyway V38

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
if (ch != null && ch.protocols() != null && !ch.protocols().isEmpty()) {
    List<ChannelProtocol> active = protocolRegistry.forProtocols(ch.protocols());
    if (!active.isEmpty()) {
        List<MessageView> recent = messageStore.findRecent(ch.id(), config.protocol().lookbackSize());
        ProtocolContext ctx = new ProtocolContext(
            ch.id(), ch.name(), dispatch.type(), dispatch.sender(),
            dispatch.correlationId(), ch.protocolParticipants(), recent);
        for (ChannelProtocol protocol : active) {
            List<String> violations = protocol.evaluate(ctx);
            for (String v : violations) { LOG.warn(v); }
            advisories.addAll(violations);
        }
    }
}
```

`messageStore.findRecent(channelId, limit)` is a new store method — returns last
N messages ordered by ID DESC, excluding EVENTs.

Reactive parity: `ReactiveMessageService` gets the same block with
`messageStore.findRecentAsync()` returning `Uni<List<MessageView>>`.

OTel span event: `qhorus.enforcement.protocol` added after protocol evaluation.

## Built-in Protocols

Four `@ApplicationScoped` beans in `runtime/message/protocol/`.

### REQUEST_RESPONSE

Surfaces open QUERY obligations at dispatch time. Thin wrapper over CommitmentStore.

- Queries `CommitmentStore.findOpenByChannelId(channelId)` filtered to QUERY type
- Advisory on new QUERY: "N unanswered QUERYs in channel 'X' — consider waiting
  for responses" (threshold: `casehub.qhorus.protocol.request-response.max-open-queries`,
  default 3)
- Advisory on non-RESPONSE when open QUERYs exist: "channel 'X' has open QUERYs
  awaiting RESPONSE"

### TASK_COMPLETION

Surfaces open COMMAND obligations at dispatch time. Thin wrapper over CommitmentStore.

- Queries `CommitmentStore.findOpenByChannelId(channelId)` filtered to COMMAND type
- Advisory on new COMMAND: "N open COMMANDs in channel 'X' — consider resolving
  existing tasks" (threshold: `casehub.qhorus.protocol.task-completion.max-open-commands`,
  default 3)
- Advisory when sender is obligor with open obligation: "you have an open obligation
  in channel 'X' — consider sending DONE/FAILURE/DECLINE"

### ROUND_ROBIN

Enforced turn-taking derived from message history.

- Turn order from `protocolParticipants` (falls back to distinct senders from lookback)
- Current turn: find last non-EVENT message in `recentMessages`, advance to next
  participant (wrapping)
- Advisory on out-of-turn: "protocol violation: expected 'agent-B' to speak next
  in channel 'X', got 'agent-A'"
- Skips evaluation when ≤1 participant or no message history

### CONTRIBUTION_REQUIRED

Inter-sender contribution gap detection.

- Scans `recentMessages` for consecutive messages from the same sender
- Advisory when sender has N consecutive messages without all other participants
  contributing: "protocol violation: 'agent-A' has sent N consecutive messages
  in channel 'X' without contributions from: agent-B, agent-C"
  (threshold: `casehub.qhorus.protocol.contribution-required.max-consecutive`, default 2)
- Participants from `protocolParticipants` (falls back to distinct senders from lookback)
- EVENT messages excluded from the scan

## Store Changes

### MessageStore

- `findRecent(UUID channelId, int limit)` — last N messages by ID DESC, excluding
  EVENTs. Returns `List<MessageView>`.
- `ReactiveMessageStore.findRecentAsync(UUID channelId, int limit)` — reactive
  counterpart returning `Uni<List<MessageView>>`.

### CommitmentStore

- `findOpenByChannelId(UUID channelId)` — all OPEN commitments for a channel.
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
  against registry
- `set_protocol_participants(channel, participants)` — full-replacement
- `get_channel_protocols(channel)` — current protocols and participants

**Modified (1):**

- `create_channel` gains `protocols` and `protocol_participants` parameters

Both blocking and reactive tool classes.

## Testing

- **Protocol beans:** CDI-free unit tests with constructed `ProtocolContext` and
  mock `CommitmentStore` (for REQUEST_RESPONSE and TASK_COMPLETION). Each protocol
  tests: no-violation path, single violation, multiple violations, empty channel
  (no history), single participant edge case.

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
