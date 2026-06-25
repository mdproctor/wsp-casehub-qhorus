# Design: Event Timestamp, Attestation Context, Per-Connector Normaliser

**Branch:** issue-294-event-timestamp-context-normaliser
**Covers:** #294, #307, #216
**Date:** 2026-06-25

---

## Overview

Three changes delivered sequentially on one branch. Each addresses a distinct gap; no architectural conflicts between them.

1. **#294** — CloudEvent timestamp uses `now()` instead of the message's persist time
2. **#307** — `CommitmentContext` lacks `capabilityTag`, blocking trust-gated attestation policies
3. **#216** — `HumanParticipatingChannelBackend.normaliser()` has no channel context, preventing per-connector normaliser dispatch

---

## #294 — Event timestamp propagation

### Problem

`QhorusCloudEventAdapter.toCloudEvent()` uses `OffsetDateTime.now(ZoneOffset.UTC)` for the CloudEvent `time` field. This is the conversion time, not the message time. Downstream consumers (e.g. `DroolsGanglion.advanceClock()`) use `CloudEvent.getTime()` for temporal reasoning — under load, clock skew proportional to CDI async delivery latency causes incorrect time advancement.

The `ConnectorsCloudEventAdapter` already uses `message.receivedAt()` — Qhorus is the outlier. Garden GE-20260621-629712 rule 5 requires CloudEvent fields to derive from stable semantic fields on the event record.

### Root cause

`MessageReceivedEvent` has no timestamp field. The adapter has nothing to use except `now()`.

### Fix

Add `Instant occurredAt` to `MessageReceivedEvent`. Populate from `Message.createdAt` (set by `@PrePersist`) in `MessageObserverDispatcher.dispatch()`. Use in the adapter instead of `now()`.

### Null-safety

The compact constructor adds `Objects.requireNonNull(occurredAt, "occurredAt")` alongside the existing EVENT content check. The record is public API — callers that construct it (tests in qhorus, claudony, engine) must supply a non-null timestamp. Production path is always safe (`message.createdAt` is set by `@PrePersist`); `MessageObserverDispatcher` uses `message.createdAt != null ? message.createdAt : Instant.now()` as defence-in-depth before passing to the constructor (same pattern as `MessageService.dispatch()` line 276-277).

### CloudEvent data payload

`QhorusCloudEventAdapter.toCloudEvent()` serialises the entire `MessageReceivedEvent` as the CloudEvent `data` field. Adding `occurredAt` means the JSON data payload gains a new `"occurredAt"` key. This is additive (non-breaking) and mirrors the duplication pattern used by `ConnectorsCloudEventAdapter` (envelope `time` + payload `receivedAt`). The CloudEvent `time` field is the envelope-level contract; `occurredAt` in the data payload is the source-of-truth copy.

### Changes

| File | Change |
|------|--------|
| `api/.../MessageReceivedEvent.java` | Add `Instant occurredAt` field after `correlationId`, before `content`; add `Objects.requireNonNull(occurredAt)` in compact constructor |
| `runtime/.../MessageObserverDispatcher.java` | Pass `message.createdAt` (with `Instant.now()` fallback) at construction |
| `runtime/.../ReactiveMessageService.java` | Set `syntheticMsg.createdAt = ctx.occurredAt()` on the synthetic Message (lines 304-314); without this, the reactive path falls back to `Instant.now()` — relocating the bug rather than fixing it |
| `runtime/.../QhorusCloudEventAdapter.java` | `.withTime(event.occurredAt().atOffset(ZoneOffset.UTC))` replaces `OffsetDateTime.now()` |

### Breaking change — call-site inventory

`MessageReceivedEvent` is a record in `qhorus-api`. Adding a field changes the canonical constructor. All callers gain the `occurredAt` argument (`Instant.now()` in tests).

| Repo | File | Sites |
|------|------|-------|
| qhorus | `MessageObserverDispatcher.java` | 1 (production fix site) |
| qhorus | `QhorusCloudEventAdapterTest.java` | 7 |
| qhorus | `MessageObserverDispatcherTest.java` | multiple incl. assertThrows at lines 373/380 |
| claudony | `FleetMessageRelayObserverTest.java` | 3 |
| engine | `InboundWorkItemBridgeGuardTest.java` | 1 |
| engine | `InboundWorkItemBridgeTest.java` | 3 |
| engine | `QhorusMessageSignalBridgeTest.java` | 3 |

---

## #307 — capabilityTag in CommitmentContext

### Problem

`LedgerWriteService.writeAttestation()` extracts `capabilityTag` from `commandEntry.content` AFTER calling `attestationPolicy.attestationFor()`. The policy never sees the capability tag. Trust-gated attestation policies (casehub-devtown) need `capabilityTag` to call `TrustScoreSource.capabilityScore()` instead of `globalScore()`.

Both blocking (`LedgerWriteService`) and reactive (`ReactiveLedgerWriteService`) paths have this issue.

### Root cause

`CommitmentContext` has four fields: `correlationId`, `channelId`, `channelName`, `commitmentId`. No `capabilityTag`.

### Fix

Add `String capabilityTag` to `CommitmentContext`. Extract it from `commandEntry.content` BEFORE constructing `CommitmentContext`. Set `LedgerAttestation.capabilityTag` from `ctx.capabilityTag()` instead of re-extracting. Single extraction, single source of truth.

### 2-arg overload removal

`CommitmentAttestationPolicy` has a backward-compatible 2-arg default method:
```java
default Optional<AttestationOutcome> attestationFor(MessageType terminalType,
        String resolvedActorId) {
    return attestationFor(terminalType, resolvedActorId, null);
}
```

**Remove it.** Zero production callers — both `LedgerWriteService` and `ReactiveLedgerWriteService` call the 3-arg form directly. Zero callers outside qhorus. 13 test callers in `CommitmentAttestationPolicyTest`, all exercising the policy without context. Adding `capabilityTag` to `CommitmentContext` increases what this shim hides — tests using it can never exercise capability-scoped attestation.

Migration: all 13 test sites become `attestationFor(type, actorId, null)`. The `twoArgDefault_delegatesToThreeArg_withNullContext` test is deleted (tests the removed shim). Context remains nullable in the 3-arg form — it's an SPI boundary.

### Changes

| File | Change |
|------|--------|
| `api/.../CommitmentContext.java` | Add `String capabilityTag` field after `commitmentId` |
| `api/.../CommitmentAttestationPolicy.java` | Remove 2-arg `attestationFor` default method |
| `runtime/.../LedgerWriteService.java` | Call `extractCapabilityTag(commandEntry.content)` before `CommitmentContext` construction; pass as 5th arg; set `attestation.capabilityTag = ctx.capabilityTag()` |
| `runtime/.../ReactiveLedgerWriteService.java` | Same restructuring |
| `runtime/test/.../CommitmentAttestationPolicyTest.java` | All 13 sites → 3-arg form with null context; delete `twoArgDefault_delegatesToThreeArg_withNullContext` test |
| All other test constructors of `CommitmentContext` | Add `capabilityTag` argument |

### Protocol update

PP-20260623-77adf0 (`commitment-attestation-policy-null-context`) must be updated: remove reference to the 2-arg overload, document the new `capabilityTag` field. Null-safety guidance remains — the 3-arg form's context parameter is nullable at the SPI boundary. Implementations should treat null `capabilityTag` (or null context) as global scope (`CapabilityTag.GLOBAL`).

### Downstream unblock

After this change, devtown's trust-gated policy can call `TrustScoreSource.capabilityScore(ctx.capabilityTag())` — exactly what #307 describes. Refs casehubio/devtown#13.

---

## #216 — Per-connector InboundNormaliser

### Problem

`HumanParticipatingChannelBackend.normaliser()` is a no-arg method returning a single normaliser for the entire backend instance. `ConnectorChannelBackend` handles all connector types (email, webhook, SMS) but can only return one normaliser. Garden GE-20260517-f28d15 documents this as a known architectural gap.

The gateway already captures the normaliser per-channel in `BackendEntry` (one entry per channel registration). The missing piece is giving the backend the channel ID so it can return a connector-specific normaliser.

### Root cause

`normaliser()` lacks channel context. `ConnectorChannelBackend` registers per-channel via `onChannelInitialised()` but returns the same normaliser for all channels.

### Fix — three parts

#### 3a. SPI change

`HumanParticipatingChannelBackend` (qhorus-api):
- Remove: `default InboundNormaliser normaliser() { return null; }`
- Add: `default InboundNormaliser normaliserFor(UUID channelId) { return null; }`

The parameter is `UUID channelId`, not `ChannelRef`. This is deliberate: `registerBackend(UUID channelId, ...)` only has the UUID — constructing a `ChannelRef` would require a name lookup for zero benefit. Neither `ConnectorChannelBackend` nor `SlackChannelBackend` need the name for normaliser dispatch (both key on UUID from their internal caches). The `open/close/post` SPI methods use `ChannelRef` because backends need the human-readable name for display (Slack thread titles, connector delivery logs). `normaliserFor` is a pure lookup — UUID suffices.

`ChannelGateway.registerBackend()`: change `hb.normaliser()` → `hb.normaliserFor(channelId)`.

`SlackChannelBackend`: rename method, ignore parameter, return same normaliser.

#### 3b. New SPI: ConnectorNormaliser

```java
package io.casehub.qhorus.connector.backend;

/**
 * A normaliser scoped to a specific inbound connector.
 * Implementations are CDI beans discovered by ConnectorChannelBackend
 * and dispatched based on the channel's connector binding.
 */
public interface ConnectorNormaliser extends InboundNormaliser {
    /** The connector ID this normaliser handles (e.g. "email-inbound"). */
    String connectorId();
}
```

Lives in `connector-backend`, not `qhorus-api`. The concept of "connector type" belongs to the connectors bridge. Keyed on connector ID (not connector type) to match the existing `ConnectorKeyStrategy` pattern and the binding's `inboundConnectorId` field — no mapping needed.

#### 3c. ConnectorChannelBackend dispatch

- Inject `Instance<ConnectorNormaliser>` for CDI discovery
- Build `Map<String, ConnectorNormaliser>` at `@PostConstruct`, keyed by `connectorId()`
- **Duplicate detection:** If two `ConnectorNormaliser` beans return the same `connectorId()`, throw `IllegalStateException` at bootstrap naming both classes. Follows the `ProjectionRegistry` pattern (fail-fast at CDI startup, not at first tool call).
- Implement `normaliserFor(UUID channelId)`: look up `CacheEntry.inboundConnectorId` → return matching normaliser or `null` (falls through to system `DefaultInboundNormaliser`)

### Bonus: correlationId passthrough

`ConnectorChannelBackend.route()` currently passes `null` for `correlationId`. Change to pass `msg.metadata().get("correlation-id")` when available. This unblocks metadata-driven threading for any connector without requiring a connector-specific normaliser.

### Changes

| File | Change |
|------|--------|
| `api/.../HumanParticipatingChannelBackend.java` | `normaliser()` → `normaliserFor(UUID channelId)` |
| `runtime/.../ChannelGateway.java` | Pass `channelId` to `normaliserFor()` in `registerBackend()` |
| `slack-channel/.../SlackChannelBackend.java` | Rename method, ignore parameter |
| `connector-backend/.../ConnectorNormaliser.java` | **New interface** |
| `connector-backend/.../ConnectorChannelBackend.java` | Inject `Instance<ConnectorNormaliser>`, dispatch map, `normaliserFor()` impl, correlationId passthrough in `route()` |

### Breaking change — call-site inventory

`normaliser()` → `normaliserFor(UUID channelId)` breaks all `@Override` sites. Same fix pattern as `SlackChannelBackend`: rename, ignore parameter, return same normaliser.

| Repo | File | Sites | Notes |
|------|------|-------|-------|
| qhorus | `SlackChannelBackend.java` | 1 | Production — ignore param, return `slackInboundNormaliser` |
| qhorus | `ConnectorChannelBackend.java` | 1 | Production — new dispatch logic |
| qhorus | `ChannelGatewayTest.java` | 2 (lines 249, 321) | Test inline backends — rename, ignore param |
| qhorus | `ChannelGatewayIntegrationTest.java` | 1 (line 64) | Test inline backend — rename, ignore param |
| qhorus | `ChannelGatewayTest.java` | 0 (line 271) | Comment-only ref — update comment text to `normaliserFor(UUID)` |

No implementations of `HumanParticipatingChannelBackend` exist outside qhorus (verified across claudony, engine, work).

### Design constraints

- **No email normaliser in this branch.** This delivers the mechanism. An `EmailConnectorNormaliser` (with Message-ID → correlationId cache) is a separate issue.
- **No new module.** `ConnectorNormaliser` lives in `connector-backend` — the bridge module.
- **Fallback is explicit.** `normaliserFor()` returns `null` → gateway uses `DefaultInboundNormaliser`.
- **connectorId keying** matches existing `ConnectorKeyStrategy.SENDER_KEYED` pattern.

---

## Cross-cutting

- No Flyway migrations — all changes are to Java records, SPIs, and service logic.
- `MessageReceivedEvent` (#294) and `CommitmentContext` (#307) are both API records — adding fields is a breaking change to consumers. No backward-compatibility shims. The break forces every caller to be explicit.
- Protocol PP-20260623-77adf0 updated for #307.
- Garden GE-20260517-f28d15 invalidation trigger met by #216 (per-channel normaliser registration now exists).
