# Per-Backend InboundNormaliser + InboundHumanMessage.inReplyTo

**Date:** 2026-05-25
**Branch:** issue-182-s-xs-batch
**Issues:** Closes #158, closes #159 (inReplyTo part)
**Scale:** M

---

## Problem

`DefaultInboundNormaliser` is an application-wide singleton that hardcodes
`MessageType.QUERY` for every `InboundHumanMessage`, regardless of channel or
context. This breaks the commitment state machine for human replies: a human
responding to a COMMAND gets typed QUERY, which opens a new Commitment instead
of fulfilling the existing one.

The application-wide scope is the root cause, not the QUERY default. A
metadata-key patch on the existing SPI would be safe, but it is still a patch
on top of a wrong abstraction. The normaliser belongs with the backend that
knows the channel's message semantics.

Additionally, `InboundHumanMessage` lacks `inReplyTo` (Long). Without it,
normalisers cannot produce `NormalisedMessage.inReplyTo`, so reply-type
dispatch (RESPONSE, DONE, DECLINE, FAILURE) through the human inbound path is
impossible regardless of how the type is inferred.

---

## Design

### API layer (`casehub-qhorus-api`)

**`InboundHumanMessage`** — add `Long inReplyTo`:

```java
public record InboundHumanMessage(
    String externalSenderId,
    String content,
    Instant receivedAt,
    Map<String, String> metadata,
    String correlationId,
    Long inReplyTo          // message ID being replied to; null for new initiations
)
```

Breaking change: every construction site must supply the new argument.
External consumers (Claudony, openclaw) pass `null` for messages that are not
replies; human UIs that support reply threading pass the message ID.

**`HumanParticipatingChannelBackend`** — add one default method:

```java
public interface HumanParticipatingChannelBackend extends ChannelBackend {
    /**
     * Returns the normaliser for messages received from this backend,
     * or {@code null} to use the system {@link DefaultInboundNormaliser}.
     */
    default InboundNormaliser normaliser() { return null; }
}
```

`null` is the sentinel for "use system default." Existing implementations are
unaffected (default returns null). Backends that need channel-specific type
inference override this method and return their own `InboundNormaliser`
implementation.

`InboundNormaliser` itself is unchanged — `@FunctionalInterface`, takes
`(ChannelRef, InboundHumanMessage)`, returns `NormalisedMessage`.

---

### Runtime layer (`casehub-qhorus`)

**`BackendEntry`** — add `normaliser` component:

```java
record BackendEntry(ChannelBackend backend, String backendType, InboundNormaliser normaliser) {}
```

**`ChannelGateway.registerBackend()`** — extract normaliser at registration,
no API change to callers:

```java
InboundNormaliser normaliser = (backend instanceof HumanParticipatingChannelBackend hb)
        ? hb.normaliser() : null;
entries.add(new BackendEntry(backend, backendType, normaliser));
```

**`ChannelGateway.receiveHumanMessage()`** — resolve effective normaliser:

```java
InboundNormaliser effective = registry.getOrDefault(channel.id(), List.of()).stream()
        .filter(e -> "human_participating".equals(e.backendType()))
        .map(BackendEntry::normaliser)
        .filter(Objects::nonNull)
        .findFirst()
        .orElse(this.normaliser);  // injected @DefaultBean fallback
NormalisedMessage n = effective.normalise(channel, raw);
```

Precedence: backend-declared normaliser > system `DefaultInboundNormaliser`.

**`DefaultInboundNormaliser`** — two new behaviours on top of existing
pass-throughs:

1. **Type from metadata key** — reads `metadata.get("message-type")`,
   case-insensitive parse to `MessageType`; silently falls back to QUERY on
   null, blank, or unrecognised value. This is the escape hatch for backends
   that don't provide a custom normaliser but occasionally need non-QUERY
   types.

2. **`inReplyTo` pass-through** — reads `raw.inReplyTo()` and sets
   `NormalisedMessage.inReplyTo`. Null when the human message is not a reply.

`correlationId` pass-through, `senderInstanceId = Senders.HUMAN`, null
`artefactRefs`, and null `target` remain unchanged. `artefactRefs` and
`target` are left for the remainder of #159.

---

## Protocol compliance

The `MessageDispatch` builder enforces: DONE, DECLINE, FAILURE require both
`inReplyTo` AND `correlationId`; RESPONSE requires `inReplyTo`; HANDOFF
requires `inReplyTo`, `correlationId`, AND `target`. Violations throw
`IllegalArgumentException` at `build()`.

This design satisfies those invariants end-to-end:

| Scenario | type | correlationId | inReplyTo |
|----------|------|--------------|-----------|
| New question | QUERY | null | null |
| Progress update | STATUS | non-null | null |
| Reply to COMMAND | RESPONSE | non-null | non-null (from InboundHumanMessage) |
| Completing a COMMAND | DONE | non-null | non-null |
| Refusing a COMMAND | DECLINE | non-null | non-null |

A backend whose normaliser produces RESPONSE without inReplyTo will cause
`build()` to throw — that is correct behaviour and the backend's responsibility
to fix.

---

## Testing strategy

### Unit tests (plain Java, no Quarkus)

**`DefaultInboundNormaliserTest`** (existing, extend and update):
- `normalise_alwaysReturnsQuery` → rename to `normalise_returns_QUERY_when_no_metadata_key`
- `normalise_remainingNullableFields_areNull` → update: `inReplyTo` now passes through; assert null only when `InboundHumanMessage.inReplyTo` is null
- New: `normalise_uses_message_type_from_metadata` — RESPONSE, DONE, STATUS, EVENT
- New: `normalise_ignores_invalid_message_type_key` — falls back to QUERY
- New: `normalise_passes_inReplyTo_from_InboundHumanMessage` — null and non-null cases
- All existing `InboundHumanMessage` construction sites gain `null` as 6th argument

### Unit tests (Mockito, existing `ChannelGatewayTest`)

- `receiveHumanMessage_uses_backend_normaliser_when_provided`
- `receiveHumanMessage_falls_back_to_system_default_when_backend_normaliser_is_null`
- `receiveHumanMessage_falls_back_to_system_default_when_no_human_participating_backend`
- `registerBackend_extracts_normaliser_from_human_participating_backend`
- All existing `receiveHumanMessage_*` tests — update `InboundHumanMessage`
  construction sites (add `null` for `inReplyTo`)

### Integration test (`@QuarkusTest`, new `ChannelGatewayIntegrationTest`)

- `receiveHumanMessage_RESPONSE_with_inReplyTo_fulfils_commitment` — registers
  a `HumanParticipatingChannelBackend` whose normaliser returns RESPONSE with
  inReplyTo; dispatches a COMMAND; sends a human message via
  `receiveHumanMessage`; asserts the Commitment transitions to FULFILLED.

---

## Out of scope

- `InboundHumanMessage.artefactRefs` and `.target` — remainder of #159;
  add when a backend needs them (issue guidance: do not add speculatively)
- Per-channel scoping for `MessageObserver` — separate concern
- HANDOFF through the human inbound path — HANDOFF requires target; target
  remains null in DefaultInboundNormaliser; backends needing HANDOFF provide
  their own normaliser

---

## Breaking changes

| Artifact | Nature |
|----------|--------|
| `InboundHumanMessage` canonical constructor | Gains `Long inReplyTo` as 6th argument |
| `BackendEntry` | Internal — no external consumers |
| All `new InboundHumanMessage(...)` call sites | Pass `null` for inReplyTo unless threading replies |
