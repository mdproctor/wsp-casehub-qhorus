# Auto-Channel Creation on First Contact — Design Spec

**Issue:** casehubio/qhorus#214
**Branch:** issue-214-auto-channel-creation
**Date:** 2026-05-31

---

## Problem

`ConnectorChannelBackend` routes inbound messages by looking up a `ChannelConnectorBinding`
for `(inboundConnectorId, externalKey)`. If no binding exists, the message is discarded.
This requires every conversation partner to be pre-provisioned — a manual step that breaks
the "first contact" use case (new SMS senders, first-time email contacts).

---

## Approach

**Approach B — `AutoChannelPolicy` SPI in `connector-backend` with `@DefaultBean` config-driven
implementation.** An `AutoChannelPolicy` bean is consulted on every discard path. If it returns
a spec, the channel and binding are created atomically, the gateway registry is populated, and
the first message is routed. If it returns empty, behaviour is unchanged (discard + warn).

---

## Architecture

Three new types in `connector-backend`, one new method in `ChannelService`.
No new modules. No Flyway migration (V14 `channel_connector_binding` table is sufficient).

```
connector-backend/
  AutoChannelPolicy              ← SPI interface (public)
  AutoChannelSpec                ← record: all params to create a channel + binding (public)
  ConfiguredAutoChannelPolicy    ← @DefaultBean, reads @ConfigMapping (package-private)
  ConnectorAutoChannelConfig     ← @ConfigMapping (package-private)

runtime/ChannelService
  + findOrCreateWithBinding()    ← new @Transactional(REQUIRES_NEW) method
```

### SPI placement rationale

`AutoChannelPolicy` is placed in `connector-backend`, not `api/spi/`, because its parameter
type `InboundMessage` comes from `casehub-connectors-core`. Adding that dependency to `api/`
would violate the `api/` module's intentional lightweight footprint. Per the
cross-foundation-bridge-module-placement protocol, the bridge module owns its own SPIs.
The SPI is still public and consumer-overridable — implementors depend on `connector-backend`.

---

## SPI and Types

```java
public interface AutoChannelPolicy {
    Optional<AutoChannelSpec> onFirstContact(InboundMessage msg, String derivedKey);
}

public record AutoChannelSpec(
    String channelName,
    ChannelSemantic semantic,
    String allowedTypes,        // null = open (no type restriction)
    String outboundConnectorId,
    String outboundDestination
) {}
```

`allowedTypes` is `null` on auto-created channels. GE-20260519-28967d established that
`allowedTypes` restricts inbound types as well as outbound — setting any default would
silently break inbound normaliser results that map to non-listed types. Operators who need
type restrictions implement a custom `AutoChannelPolicy`.

---

## Default Implementation — `ConfiguredAutoChannelPolicy`

### Config shape

```properties
# SMS — convention resolves outbound (no outbound-connector-id needed)
casehub.qhorus.connector.auto-channel.entries."twilio-sms-inbound".enabled=true

# Email — no convention; outbound must be explicit
casehub.qhorus.connector.auto-channel.entries."email-inbound".enabled=true
casehub.qhorus.connector.auto-channel.entries."email-inbound".outbound-connector-id=email

# Optional overrides (all connectors)
casehub.qhorus.connector.auto-channel.entries."<id>".channel-name-pattern=connector/{connectorId}/{derivedKey}
casehub.qhorus.connector.auto-channel.entries."<id>".semantic=APPEND
```

### Outbound connector resolution — hybrid convention

| Connector type | Convention outbound | Reason |
|---|---|---|
| `twilio-sms-inbound` | `twilio-sms` | Protocol-coupled: SMS threading requires same provider/number |
| `whatsapp-inbound` | `whatsapp` | Protocol-coupled: WhatsApp API requires same credential for reply |
| `email-inbound` | *(none)* | Transport-decoupled: outbound SMTP account is a business decision |
| `slack-inbound` | *(none)* | Multi-workspace: explicit mapping required to avoid wrong workspace |

Resolution order:
1. Explicit `outbound-connector-id` in config
2. Convention mapping (SMS, WhatsApp only)
3. `Optional.empty()` + ERROR log naming the missing config key → falls back to discard

### Channel naming

Default pattern: `connector/{connectorId}/{derivedKey}`

Examples:
- `connector/twilio-sms-inbound/+447911123456`
- `connector/email-inbound/alice@example.com`
- `connector/slack-inbound/C01234567`

The name is deterministic and recoverable — the same sender always produces the same channel
name, so a future `findByConnectorKey` miss after a cache flush will reconstruct correctly.
Operators may override via `channel-name-pattern` without a custom `AutoChannelPolicy`.

`outboundDestination` reuses `ConnectorKeyStrategy.deriveKey(msg)` — the same value used
for binding lookup is the reply destination. For sender-keyed connectors (SMS, WhatsApp,
email) this is `externalSenderId`; for channel-keyed connectors (Slack) it is
`externalChannelRef`.

---

## Transactional Flow

### `ChannelService.findOrCreateWithBinding()`

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public Channel findOrCreateWithBinding(
        String connectorId, String externalKey,
        String channelName, ChannelSemantic semantic, String allowedTypes,
        String outboundConnectorId, String outboundDestination)
```

Steps:
1. Recheck `channelBindingStore.findByKey(connectorId, externalKey)` under the new transaction
2. If found: return the existing channel (race winner already committed)
3. If not found: `channelStore.put(channel)` + `channelBindingStore.put(binding)` atomically

`REQUIRES_NEW` commits the channel + binding independently of any caller context. This ensures
the record is durable before `initChannel()` and `receiveHumanMessage()` run. If a subsequent
step fails, the channel persists and future messages route correctly.

### `ConnectorChannelBackend.onInboundMessage()` — updated flow

```
findByConnectorKey(connectorId, key)
  → found: route normally (unchanged)
  → not found:
      autoChannelPolicy.onFirstContact(msg, key)
        → Optional.empty(): WARN + discard (unchanged)
        → AutoChannelSpec present:
            channelService.findOrCreateWithBinding(...)   [REQUIRES_NEW]
              → success (race winner): increment inbound_channels_auto_created_total
              → catches ConstraintViolationException (race loser):
                  findByConnectorKey() again → recovers winner's channel
            channelGateway.initChannel(channel.id, ref)   ← always called, winner or loser
            route via receiveHumanMessage()
```

### Race condition handling

Two concurrent first messages from the same sender:
- Thread A wins DB insert → commits channel + binding → calls `initChannel()` → routes
- Thread B hits unique constraint on `(inbound_connector_id, external_key)`
- Thread B catches `ConstraintViolationException` → calls `findByConnectorKey()` → finds Thread A's channel
- Thread B also calls `initChannel()` — safe to call multiple times; `onChannelInitialised`
  deregisters then re-registers the backend, net result is still registered
- Both messages route; no message dropped; no channel duplicated

`initChannel()` is called by **both winner and loser**. This eliminates a thread-scheduling
hazard: if Thread B reached `receiveHumanMessage()` before Thread A's `initChannel()` had
populated the registry, `fanOut()` would find no backends and silently skip push delivery
(the message is still persisted). Always calling `initChannel()` in `tryAutoCreate()`
regardless of win/loss guarantees the registry is populated before routing. The
`inbound_channels_auto_created_total` counter increments only in the winner path.

### Delivery guarantee

At-most-once for the first message in failure scenarios (JVM crash between
`findOrCreateWithBinding()` committing and `receiveHumanMessage()` executing). The channel
persists and all subsequent messages are delivered normally. This is consistent with
Approach A (at-most-once + client catch-up) from qhorus#132 and acceptable for the first-contact
use case.

---

## Metrics

| Counter | Tags | Meaning |
|---|---|---|
| `inbound_messages_discarded_total` | `connector_id` | Unchanged — increments only when both lookup and policy return empty |
| `inbound_channels_auto_created_total` | `connector_id` | New — increments on successful auto-creation |

---

## Testing

### Unit tests — `ConfiguredAutoChannelPolicyTest` (no Quarkus)

- SMS `enabled=true`, no explicit outbound → convention resolves `"twilio-sms"`, destination = `externalSenderId`
- Email `enabled=true`, explicit `outbound-connector-id=email` → spec with that outbound
- Email `enabled=true`, no outbound config, no convention → `Optional.empty()` + error logged
- `enabled=false` → `Optional.empty()`
- `channel-name-pattern` substitution → `{connectorId}` and `{derivedKey}` replaced
- `semantic=LAST_WRITE` override → spec carries `LAST_WRITE`

### Integration tests — `ConnectorAutoChannelBackendTest` (`@QuarkusTest`, new class)

- **Auto-create on first contact:** unknown sender, SMS policy enabled → `dispatch()` called, binding in store, `inbound_channels_auto_created_total` incremented
- **Second message reuses channel:** same sender → no duplicate channel, `dispatch()` called with same `channelId`
- **Policy disabled → discard:** `enabled=false` → discard counter increments, no channel created
- **outboundDestination wired correctly:** after auto-create, `fanOut()` → `connectorService.send()` called with sender's phone as destination
- **No convention, no config → discard with WARN:** unknown connector type, no explicit outbound → `Optional.empty()`, discard counter increments

### Existing test compatibility

`ConnectorChannelBackendIntegrationTest.unknownSender_noChannelBound_discardCounterIncremented`
relies on discard behaviour. With no `casehub.qhorus.connector.auto-channel.*` entries in test
`application.properties`, `ConfiguredAutoChannelPolicy.entries()` is empty → `Optional.empty()`
for all connectors → test passes unchanged. No mocking needed.

---

## Out of Scope (tracked separately)

- Per-connector `InboundNormaliser` customisation → qhorus#216
- Delivery guarantees (retry, dead-letter) for auto-created channels → qhorus#132
- MCP tools for listing/managing auto-created channels (no new tools needed; existing
  `list_channels`, `get_channel`, `delete_channel` apply)
