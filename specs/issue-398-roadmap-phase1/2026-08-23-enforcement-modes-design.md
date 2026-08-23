# E3: Active Governance Policies — Enforcement Modes

**Issue:** casehubio/qhorus#400
**Date:** 2026-08-23
**Branch:** issue-398-roadmap-phase1

## Context

Protocol enforcement is currently advisory — violations from all three advisory
sources (MessageTypePolicy, CorrelationIntegrityChecker, ChannelProtocol) produce
`DispatchResult.advisories()` but never block dispatch. Deployers need configurable
enforcement to match their risk tolerance, from informational warnings through
message rejection to full channel containment.

This is Phase 1 Epic 3. It builds on E2 (#399, cascade containment) which
established the containment primitives (pause, expire commitments, deregister).

## Design

### EnforcementMode Enum

```java
public enum EnforcementMode {
    ADVISORY,    // current behavior — advisories returned, message dispatched
    BLOCKING,    // advisories become rejections — message not dispatched, throws
    QUARANTINE   // reject + contain — pause channel, expire commitments, throw
}
```

`ADVISORY` is the default. Existing behavior is unchanged for channels that
don't set an enforcement mode.

### EnforcementBlockedException

```java
public class EnforcementBlockedException extends IllegalStateException {
    private final EnforcementMode mode;
    private final List<String> violationSources;
    private final List<String> violations;
}
```

Extends `IllegalStateException` — backward-compatible with all existing catch
blocks, `@WrapBusinessError` interceptors, and REST exception mappers. Enables
programmatic discrimination (`catch (EnforcementBlockedException)`) for agents
that need different corrective action than for ACL or rate-limit failures.
Same pattern as `MessageTypeViolationException`.

### Channel Record Changes

`Channel` gains two fields:

| Field | Type | Default | Null semantics |
|-------|------|---------|----------------|
| `enforcementMode` | `EnforcementMode` | `ADVISORY` | null = ADVISORY |
| `enforcementExclusions` | `List<String>` | `List.of()` | null = List.of() (no exclusions = full enforcement) |

Backward-compatible constructors default both to null/empty. `Channel.fromRequest()`
and `Channel.toBuilder()` propagate both fields. List normalization follows the
existing pattern: null → `List.of()` in the compact constructor.

### Advisory Tagging

Each advisory source produces a tag identifying its origin. Tags are used by the
enforcement gate to check exclusions.

| Source | Tag | Convention |
|--------|-----|-----------|
| `MessageTypePolicy.advisory()` | `TYPE_POLICY` | New — tag prepended internally |
| `CorrelationIntegrityChecker.check()` | `CORRELATION_INTEGRITY` | New — tag prepended internally |
| `ChannelProtocol.evaluate()` | Protocol name (e.g. `REQUEST_RESPONSE`) | Existing `[PROTOCOL_NAME]` prefix parsed |

Implementation: introduce a package-private `TaggedAdvisory(String source, String message)`
record used within `MessageService.dispatch()`. All three advisory collection sites
produce `TaggedAdvisory` instances. The enforcement gate works with these tagged
objects. After the enforcement gate passes (ADVISORY mode or no enforceable
violations), tagged advisories are flattened to `List<String>` for `DispatchResult`.

`DispatchResult` is unchanged — advisories remain `List<String>`. The tagging is
internal to the dispatch pipeline.

### Enforcement Gate

New enforcement check in `MessageService.dispatch()`, positioned after all three
advisory sources and before LAST_WRITE / persist:

```
Steps 5-8: existing advisory collection (MessageTypePolicy, CorrelationIntegrityChecker, ChannelProtocol)
Step 9 (NEW): Enforcement gate
  - if channel.enforcementMode() == ADVISORY → pass through (current behavior)
  - if dispatch.type() == EVENT → pass through (exempt, prevents recursion)
  - if sender contains ":" → pass through (system sender exempt, consistent with trust gate)
  - if dispatch.type() is obligation-resolving (DONE, FAILURE, DECLINE, RESPONSE) → pass through
    (obligation resolution must never be blocked — ADR-0016; HANDOFF is NOT exempt —
    it transfers obligations via CommitmentService.delegate(), not resolves them)
  - filter tagged advisories: remove any whose source is in channel.enforcementExclusions()
  - if no enforceable violations remain → pass through
  - enforceable violations exist:
    a. enforcementExecutor.execute(channel, dispatch, enforceable violations)
       (REQUIRES_NEW — see EnforcementExecutor below)
    b. throw EnforcementBlockedException with violation details
Step 10: LAST_WRITE / persist / commit / fanOut (existing)
```

The gate method in MessageService: `enforceIfRequired(Channel, List<TaggedAdvisory>, MessageDispatch)`.
It delegates containment and audit to `EnforcementExecutor`.

### EnforcementExecutor (REQUIRES_NEW)

A package-private `@ApplicationScoped` CDI bean, analogous to `ChannelCreateHelper`.
All enforcement side effects execute in `REQUIRES_NEW` transactions so they survive
the outer transaction's rollback when `EnforcementBlockedException` is thrown.

```java
@ApplicationScoped
class EnforcementExecutor {

    @Transactional(REQUIRES_NEW)
    void execute(Channel ch, MessageDispatch dispatch, List<TaggedAdvisory> violations,
                 String tenancyId) {
        // 1. Dispatch enforcement EVENT to violating channel
        messageService.dispatch(MessageDispatch.builder()
                .channelId(ch.id())
                .sender("system:enforcement")
                .type(MessageType.EVENT)
                .telemetry(telemetryJson(...))
                .actorType(ActorType.SYSTEM)
                .tenancyId(tenancyId)
                .build());

        // 2. If QUARANTINE: pause + expire
        if (ch.enforcementMode() == EnforcementMode.QUARANTINE) {
            channelService.pause(ch.id());
            commitmentService.expireByChannel(ch.id());
        }

        // 3. Fire CDI event for external notification
        enforcementBlockedEvent.fireAsync(new EnforcementBlockedEvent(
                ch.id(), ch.name(), ch.enforcementMode(),
                dispatch.sender(), dispatch.type(),
                violations.stream().map(TaggedAdvisory::message).toList(),
                violations.stream().map(TaggedAdvisory::source).distinct().toList()));
    }
}
```

This bean is injected into `MessageService` via CDI — calls go through the CDI
proxy, ensuring the `REQUIRES_NEW` annotation takes effect (unlike a self-call).

Error isolation: the entire `execute()` is wrapped in try-catch in the caller.
If it fails, the `EnforcementBlockedException` is still thrown (message is blocked)
but audit/containment may be incomplete.

### EnforcementBlockedEvent (CDI)

```java
public record EnforcementBlockedEvent(
    UUID channelId, String channelName, EnforcementMode mode,
    String blockedSender, MessageType blockedType,
    List<String> violations, List<String> violationSources) {}
```

Fired async from `EnforcementExecutor`. Flows through `ConnectorAlertBridge` to
external notification systems (Slack, webhooks), consistent with `WatchdogAlertEvent`.
Provides centralized operational monitoring independent of per-channel EVENT audit trail.

### Enforcement EVENT

Dispatched by `EnforcementExecutor` to the violating channel (in REQUIRES_NEW):

```java
MessageDispatch.builder()
    .channelId(channelId)
    .sender("system:enforcement")
    .type(MessageType.EVENT)
    .telemetry(telemetryJson(
        "enforcement_action", mode == QUARANTINE ? "QUARANTINED" : "BLOCKED",
        "violations", violationMessages,
        "violation_sources", violationSources,
        "blocked_sender", dispatch.sender(),
        "blocked_type", dispatch.type().name(),
        "enforcement_mode", mode.name()))
    .actorType(ActorType.SYSTEM)
    .tenancyId(effectiveTenancyId)
    .build()
```

This EVENT goes through the normal dispatch pipeline. It is not subject to
enforcement because `dispatch.type() == EVENT` is exempted in the gate. It is also
exempt because `sender` contains `:`. It creates a Message and MessageLedgerEntry —
fully auditable, queryable via `list_ledger_entries(type_filter="EVENT")`.

### QUARANTINE Containment

Within `EnforcementExecutor.execute()` (REQUIRES_NEW), after the enforcement EVENT:

1. `channelService.pause(channelId)` — prevents further dispatches
2. `commitmentService.expireByChannel(channelId)` — resolves open obligations

Same primitives as #399's `WatchdogEvaluationService.executeContainmentAction()`.
No agent deregistration — enforcement is single-message-based (one sender violated
a protocol); the watchdog handles pattern-based agent concerns.

### ChannelEntity Changes

```java
@Column(name = "enforcement_mode")
private String enforcementMode;      // nullable, null = ADVISORY

@Column(name = "enforcement_exclusions")
private String enforcementExclusions; // nullable, comma-separated tags
```

`toDomain()` maps: `EnforcementMode.valueOf()` with null → `ADVISORY`;
`splitCsv()` for exclusions with null → `List.of()`.

### ChannelCreateRequest Changes

Two new fields:

```java
EnforcementMode enforcementMode     // nullable, null = ADVISORY
List<String> enforcementExclusions   // nullable, null = no exclusions
```

Builder gains `.enforcementMode()` and `.enforcementExclusions()` setters.
Backward-compatible constructor defaults both to null.

### Flyway Migration (V45)

```sql
ALTER TABLE channel ADD COLUMN enforcement_mode VARCHAR(20);
ALTER TABLE channel ADD COLUMN enforcement_exclusions TEXT;
```

Both nullable. No default — null = ADVISORY / no exclusions. No CHECK constraint
on enforcement_mode (enum validation at the Java layer, consistent with other
enum fields like ChannelSemantic).

### MCP Tools

**`set_enforcement_mode(channel, mode)`**
- Validates `mode` is a valid `EnforcementMode` name
- Delegates to `ChannelService.setEnforcementMode(UUID, EnforcementMode)`
- Returns confirmation with current mode

**`set_enforcement_exclusions(channel, exclusions)`**
- `exclusions`: comma-separated source tags
- Validates each tag against known sources: `TYPE_POLICY`, `CORRELATION_INTEGRITY`,
  plus all registered protocol names from `ProtocolRegistry`
- Unknown tags → warning (not error, forward-compatible with custom protocols)
- Delegates to `ChannelService.setEnforcementExclusions(UUID, List<String>)`

**`create_channel`** gains two optional parameters:
- `enforcement_mode` (String, nullable)
- `enforcement_exclusions` (String, nullable, comma-separated)

**`get_channel_enforcement(channel)`** — returns current mode, exclusions, and
lists all available source tags (from ProtocolRegistry + fixed tags).

### REST API

**`PUT /api/channels/{id}/enforcement-mode`**

Request body:
```json
{
  "mode": "BLOCKING",
  "exclusions": ["CORRELATION_INTEGRITY"]
}
```

Both fields optional — omitted fields are unchanged. Delegates to
`ChannelService`. Response: updated `ChannelResponse`.

`ChannelResponse` gains `enforcementMode` and `enforcementExclusions` fields.

### ChannelDetail Changes

`ChannelDetail` (in `api/channel/`) gains two String fields following the existing
CSV-serialized pattern (consistent with `protocols`, `allowedWriters`, etc.):
- `String enforcementMode` — enum name or null (null = ADVISORY)
- `String enforcementExclusions` — comma-separated tags or null

`QhorusEntityMapper.toChannelDetail()` maps from Channel:
`ch.enforcementMode() != ADVISORY ? ch.enforcementMode().name() : null` and
`joinCsv(ch.enforcementExclusions())`. Backward-compatible constructors default
both to null.

### Exemptions Summary

| Exemption | Reason |
|-----------|--------|
| `dispatch.type() == EVENT` | Infrastructure messages; prevents recursion from enforcement EVENTs |
| `sender.contains(":")` | System senders (system:enforcement, system:watchdog, etc.); consistent with trust gate |
| Obligation-resolving types (DONE, FAILURE, DECLINE, RESPONSE) | Obligation resolution must never be blocked (ADR-0016). HANDOFF is NOT exempt — it transfers, not resolves |
| Source in `enforcementExclusions` | Per-channel runtime configuration to reduce coverage |

### What Does NOT Change

- **DispatchResult** — no new fields. Advisories still `List<String>` in ADVISORY mode.
  BLOCKING/QUARANTINE throw before constructing a result.
- **Existing hard enforcement** — paused, ACL, rate limit, trust gate, COMMAND/QUERY
  type validation remain throws. Enforcement mode doesn't affect them.
- **ChannelProtocol SPI** — `evaluate()` still returns `List<String>`. Protocols don't
  know about enforcement mode; the gate handles it.
- **Watchdog system** — independent async detection. Complements enforcement (enforcement
  is synchronous inline, watchdog is asynchronous pattern-based).
- **MessageTypePolicy** interface — `validate()` and `advisory()` signatures unchanged.
  Only the caller (MessageService) tags the advisory output.

## Testing Strategy

1. **EnforcementMode enum** — trivial, no test needed
2. **EnforcementBlockedException** — constructor, field accessors, ISA IllegalStateException
3. **TaggedAdvisory** — package-private record, tested through dispatch integration
4. **Enforcement gate unit tests** (CDI-free):
   - ADVISORY mode: advisories returned, no throw
   - BLOCKING mode: throws EnforcementBlockedException on violation, no throw when advisories empty
   - QUARANTINE mode: throws + executor called (pause + expire)
   - Exclusions: excluded sources don't trigger enforcement
   - EVENT exemption: EVENT type bypasses enforcement
   - System sender exemption: system senders bypass enforcement
   - Obligation-resolving type exemption: DONE/FAILURE/DECLINE/RESPONSE bypass enforcement
   - HANDOFF enforcement: HANDOFF IS subject to enforcement (not exempt — transfers, not resolves)
   - Mixed: some advisories excluded, some enforceable
5. **EnforcementExecutor** (CDI-free with mocks):
   - Dispatches enforcement EVENT with correct telemetry
   - BLOCKING: no pause/expire
   - QUARANTINE: pause + expire called
   - Fires EnforcementBlockedEvent CDI event
   - Error isolation: individual failures don't prevent throw
6. **Integration tests** (`@QuarkusTest`):
   - End-to-end: send violating message to BLOCKING channel, assert throw + EVENT in timeline
   - QUARANTINE: verify channel paused after enforcement
   - Obligation-resolving exemption: DONE on BLOCKING channel still dispatches
   - HANDOFF not exempt: HANDOFF on BLOCKING channel throws EnforcementBlockedException
   - Exclusion: verify excluded protocol doesn't block
   - set_enforcement_mode / set_enforcement_exclusions MCP tools
7. **Migration test** — V45 column existence verified in FlywayMigrationSchemaTest
8. **ChannelCreateRequest** — enforcement fields in builder, backward compat constructors

## Deferred (out of scope for Phase 1)

- **Space-level enforcement inheritance** — channels inherit enforcement mode from Space
  unless explicitly overridden. Useful for deployments with many channels. Phase 2 concern.
- **Threshold-based quarantine** — require N violations within a window before QUARANTINE
  triggers, analogous to watchdog thresholds. Reduces false-positive risk from heuristic
  protocol advisories. Phase 2 concern.

## References

- `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — dispatch pipeline
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java` — validate/advisory split
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/CorrelationIntegrityChecker.java` — structural advisories
- `api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java` — protocol SPI
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/protocol/ProtocolRegistry.java` — protocol discovery
- `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java` — containment pattern (#399)
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateHelper.java` — REQUIRES_NEW precedent
- `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAction.java` — containment enum (#399)
- `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAlertEvent.java` — CDI event pattern
- `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java` — result record (unchanged)
- `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — channel record
- `docs/adr/0016-hybrid-channel-type-enforcement.md` — ADR for type enforcement asymmetry
- casehubio/qhorus#399 — E2 cascade containment (containment primitives)
- casehubio/qhorus#400 — this issue
- [Governance-as-a-Service (arXiv, 2025)](https://arxiv.org/html/2508.18765v2)
