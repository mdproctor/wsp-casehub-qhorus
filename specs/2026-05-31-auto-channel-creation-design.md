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

Three new types in `connector-backend`, one new method in `ChannelService`, one column in
the `channel` table (V15 migration). No new modules.

```
connector-backend/
  AutoChannelPolicy              ← SPI interface (public)
  AutoChannelSpec                ← record: all params to create a channel + binding (public)
  ConfiguredAutoChannelPolicy    ← @DefaultBean, reads @ConfigMapping (package-private)
  ConnectorAutoChannelConfig     ← @ConfigMapping (package-private)

runtime/ChannelService
  + findOrCreateWithBinding()    ← new @Transactional(REQUIRES_NEW) method, takes ChannelCreateRequest

runtime/channel/Channel
  + autoCreated boolean          ← V15 migration — distinguishes auto-created from provisioned

testing/InMemoryChannelBindingStore
  put() fix                      ← must throw PersistenceException on duplicate compound key
                                     to faithfully simulate uq_binding_key constraint
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
    Optional<AutoChannelSpec> onFirstContact(InboundMessage msg, String lookupKey);
}

public record AutoChannelSpec(
    String channelName,
    String description,         // e.g. "Auto-created on first contact via twilio-sms-inbound"
    ChannelSemantic semantic,
    String allowedTypes,        // null = open (no type restriction)
    String outboundConnectorId,
    String outboundDestination
) {}
```

**Parameter name:** `lookupKey` not `derivedKey` — from the SPI implementor's perspective this is
the key used for binding lookup; the derivation strategy (`ConnectorKeyStrategy`) is
package-private and invisible to external implementors.

**`allowedTypes` is `null` on auto-created channels.** GE-20260519-28967d established that
`allowedTypes` restricts inbound types as well as outbound — setting any default would
silently break inbound normaliser results that map to non-listed types. Operators who need
type restrictions implement a custom `AutoChannelPolicy`.

---

## Default Implementation — `ConfiguredAutoChannelPolicy`

### Config shape

```java
@ConfigMapping(prefix = "casehub.qhorus.connector.auto-channel")
interface ConnectorAutoChannelConfig {
    Map<String, ConnectorAutoChannelEntry> entries();

    interface ConnectorAutoChannelEntry {
        @WithDefault("false") boolean enabled();        // absent = disabled, not error
        Optional<String> outboundConnectorId();
        Optional<String> channelNamePattern();          // default: "connector/{connectorId}/{lookupKey}"
        Optional<String> semantic();                    // default: APPEND
    }
}
```

`@WithDefault("false")` is required — without it, accessing `enabled()` when the property is
absent throws `NoSuchElementException` at startup for any configured entry lacking the key.

```properties
# SMS — convention resolves outbound (no outbound-connector-id needed)
casehub.qhorus.connector.auto-channel.entries."twilio-sms-inbound".enabled=true

# Email — no convention; outbound must be explicit
casehub.qhorus.connector.auto-channel.entries."email-inbound".enabled=true
casehub.qhorus.connector.auto-channel.entries."email-inbound".outbound-connector-id=email

# Optional overrides (any connector)
casehub.qhorus.connector.auto-channel.entries."<id>".channel-name-pattern=connector/{connectorId}/{lookupKey}
casehub.qhorus.connector.auto-channel.entries."<id>".semantic=APPEND
```

### Outbound connector resolution — hybrid convention

| Connector type | Convention outbound | Reason |
|---|---|---|
| `twilio-sms-inbound` | `"twilio-sms"` | Protocol-coupled: SMS threading requires same provider/number |
| `whatsapp-inbound` | `"whatsapp"` | Protocol-coupled: WhatsApp API requires same credential for reply |
| `email-inbound` | *(none)* | Transport-decoupled: outbound SMTP account is a business decision |
| `slack-inbound` | *(none)* | Multi-workspace: explicit mapping required to avoid wrong workspace |

Convention strings `"twilio-sms"` and `"whatsapp"` are literal outbound connector ID strings.
`casehub-connectors-core` does not yet expose `OutboundConnectorIds` constants — these must
be treated as string literals pending a future constants class, and updated if connector IDs
change.

Resolution order:
1. Explicit `outbound-connector-id` in config
2. Convention mapping (SMS, WhatsApp only)
3. `Optional.empty()` + ERROR log naming the missing config key → falls back to discard

If outbound cannot be resolved, the channel is not created and the discard counter increments.

### Channel naming

Default pattern: `connector/{connectorId}/{lookupKey}`

Examples:
- `connector/twilio-sms-inbound/+447911123456`
- `connector/email-inbound/alice@example.com`
- `connector/slack-inbound/C01234567`

The name is deterministic and recoverable — the same sender always produces the same name.
Operators may override via `channel-name-pattern`.

**Known constraints on `lookupKey` values:**
- `ChannelConnectorBinding.external_key` is `VARCHAR(255)` — a 320-character email address would
  fail DB insertion. The `findByKey()` miss comes first, but `put()` would also fail for keys
  over 255 chars. ConfiguredAutoChannelPolicy should truncate or reject keys exceeding this limit.
- Email addresses containing `+` (e.g. `alice+test@example.com`) are safe in DB storage and
  channel names; avoid using channel names as URI segments without encoding.
- `/` in a connector ID (not currently used in any connector) would corrupt the default pattern
  `connector/{connectorId}/{lookupKey}`. The `channel-name-pattern` config is the escape hatch.

`outboundDestination` reuses `ConnectorKeyStrategy.deriveKey(msg)` — the same value used for
binding lookup is the reply destination. For sender-keyed connectors (SMS, WhatsApp, email) this
is `externalSenderId`; for channel-keyed connectors (Slack) it is `externalChannelRef`.

---

## Transactional Flow

### `ChannelService.findOrCreateWithBinding(ChannelCreateRequest req)`

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public Channel findOrCreateWithBinding(ChannelCreateRequest req)
```

Uses `ChannelCreateRequest` (not raw parameters) — consistent with the existing
`create(ChannelCreateRequest)` API. The method reuses `req.inboundConnectorId()` and
`req.externalKey()` for the binding lookup.

Steps:
1. `channelBindingStore.findByKey(req.inboundConnectorId(), req.externalKey())` — recheck under
   the new transaction (race winner will have committed before this runs)
2. If found: return the existing channel (skip creation)
3. If not found: `channelStore.put(channel)` + `channelBindingStore.put(binding)` atomically

**Why `REQUIRES_NEW` and not `REQUIRED`:** `REQUIRED` would join any existing transaction. If the
caller's transaction later rolls back, the channel creation would also roll back — leaving the
gateway registry populated for a channel that no longer exists. `REQUIRES_NEW` makes the
channel + binding durable independently of the caller context. The commit happens at the CDI
proxy interceptor when the method exits — not inside the method body.

### Exception handling — in the caller (`ConnectorChannelBackend`)

`REQUIRES_NEW` commits when the method exits via the CDI proxy. A unique constraint violation on
`uq_binding_key` therefore surfaces in the **caller** as `jakarta.persistence.PersistenceException`
(wrapping `org.hibernate.exception.ConstraintViolationException`, wrapping
`java.sql.SQLIntegrityConstraintViolationException`). It does not fire inside
`findOrCreateWithBinding()` itself.

The catch belongs in `ConnectorChannelBackend.onInboundMessage()` (or `tryAutoCreate()`), not
inside `findOrCreateWithBinding()`.

A utility method should discriminate constraint violations from other DB errors:

```java
static boolean isConcurrentInsert(PersistenceException ex) {
    Throwable cause = ex.getCause();
    while (cause != null) {
        if (cause instanceof SQLIntegrityConstraintViolationException c) {
            // Check constraint name if available; fall back to message scan
            String msg = c.getMessage() != null ? c.getMessage().toLowerCase() : "";
            return msg.contains("uq_binding_key") || msg.contains("unique");
        }
        cause = cause.getCause();
    }
    return false;
}
```

If `!isConcurrentInsert(ex)` — the exception is a genuine DB error, not a race. Re-throw or
discard + ERROR log. Do not enter the loser recovery path on non-constraint failures.

Channel name also has a unique constraint. A collision there produces a different constraint
violation (on the channel name index, not `uq_binding_key`). `isConcurrentInsert` returning
`false` for that case causes a re-throw, which is correct — a misconfigured
`channel-name-pattern` that strips `{connectorId}` could cause this.

### `ConnectorChannelBackend` — updated flow

```
findByConnectorKey(connectorId, lookupKey)
  → found: route normally (unchanged)
  → not found: tryAutoCreate(msg, lookupKey)
      → autoChannelPolicy.onFirstContact(msg, lookupKey)
          → Optional.empty(): return Optional.empty() — caller discards
          → AutoChannelSpec present:
              try:
                channelService.findOrCreateWithBinding(req)   [REQUIRES_NEW commits]
                meterRegistry.counter("inbound_channels_auto_created_total", ...).increment()
                channelGateway.initChannel(channel.id, ref)
                return Optional.of(channel)
              catch PersistenceException where isConcurrentInsert:
                findByConnectorKey() again → return Optional.of(winner's channel)
                                           → (initChannel NOT called — see race section)
              catch PersistenceException (other):
                LOG.error(...); return Optional.empty() — caller discards
```

### Race condition handling

Two concurrent first messages from the same sender:

- Thread A wins DB insert → `REQUIRES_NEW` commits channel + binding → calls `initChannel()` →
  routes its message
- Thread B's `REQUIRES_NEW` hits unique constraint on `uq_binding_key` at commit time →
  `PersistenceException` thrown to caller → `isConcurrentInsert` returns true → loser path:
  `findByConnectorKey()` → finds Thread A's committed channel → routes Thread B's message

**Thread B does NOT call `initChannel()`.** The winner (Thread A) is solely responsible for
populating the gateway registry. This avoids a double-registration hazard: `onChannelInitialised`
does `deregisterBackend + registerBackend` (two non-atomic operations); if both threads called
`initChannel()` concurrently, interleaving could register the backend twice, causing fanOut to
call `post()` twice per message.

**Consequence for Thread B:** If Thread B's `receiveHumanMessage()` runs before Thread A's
`initChannel()` populates the registry, `fanOut()` finds no backends and the push delivery is
skipped — the message is still persisted. This is within the at-most-once push delivery contract
stated in qhorus#132 and consistent with the existing at-most-once guarantee for all channel
writes. All subsequent messages from the same sender route normally.

### V15 migration — `auto_created` column

```sql
ALTER TABLE channel ADD COLUMN auto_created BOOLEAN NOT NULL DEFAULT FALSE;
```

`ChannelService.findOrCreateWithBinding()` sets `channel.autoCreated = true` before
`channelStore.put()`. Manually provisioned channels (all existing create paths) leave it
`false`. This distinguishes auto-created channels in MCP queries and operator audits without
requiring a new table or join. Adding it now avoids a later migration after the table is in
production.

---

## Metrics

| Counter | Tags | Meaning |
|---|---|---|
| `inbound_messages_discarded_total` | `connector_id` | Unchanged — increments only when both lookup and policy return empty |
| `inbound_channels_auto_created_total` | `connector_id` | New — increments in winner path only |

---

## Testing

### Prerequisite: fix `InMemoryChannelBindingStore.put()`

`InMemoryChannelBindingStore.put()` currently overwrites on duplicate compound key without error.
This must be fixed before the race condition test is valid — the loser path relies on
`PersistenceException` being thrown, and the test store must simulate the `uq_binding_key`
constraint:

```java
@Override
public void put(ChannelConnectorBinding binding) {
    String newKey = compoundKey(binding.inboundConnectorId, binding.externalKey);
    synchronized (this) {
        ChannelConnectorBinding existingById = byChannelId.get(binding.channelId);
        if (existingById != null) {
            byKey.remove(compoundKey(existingById.inboundConnectorId, existingById.externalKey));
        }
        ChannelConnectorBinding existingByKey = byKey.get(newKey);
        if (existingByKey != null && !existingByKey.channelId.equals(binding.channelId)) {
            throw new PersistenceException(
                new SQLIntegrityConstraintViolationException(
                    "Duplicate entry for uq_binding_key: " + newKey));
        }
        byChannelId.put(binding.channelId, binding);
        byKey.put(newKey, binding);
    }
}
```

The entire check-and-put must be inside `synchronized (this)`. Without it, two threads can
both execute `byKey.get(newKey) → null`, both pass the guard, and both insert — the race
test would then see two channels created and fail with a false negative rather than exercising
the loser path.

This fix is in `testing/` and is prerequisite work for this feature's tests.

### Unit tests — `ConfiguredAutoChannelPolicyTest` (no Quarkus)

- SMS `enabled=true`, no explicit outbound → convention resolves `"twilio-sms"`, destination = `externalSenderId`
- Email `enabled=true`, explicit `outbound-connector-id=email` → spec carries that outbound
- Email `enabled=true`, no outbound config, no convention → `Optional.empty()` + error logged
- `enabled=false` → `Optional.empty()`
- `channel-name-pattern` substitution → `{connectorId}` and `{lookupKey}` replaced correctly
- `semantic=LAST_WRITE` override → spec carries `LAST_WRITE`
- Absent connector entry (empty config map) → `Optional.empty()`

### Integration tests — `ConnectorAutoChannelBackendTest` (`@QuarkusTest`, new class)

- **Auto-create on first contact:** unknown sender, SMS policy enabled → `dispatch()` called,
  binding in store, `inbound_channels_auto_created_total` incremented, `channel.autoCreated=true`
- **Second message reuses channel:** same sender again → no duplicate channel, `dispatch()` called
  with same `channelId`, `inbound_channels_auto_created_total` NOT incremented again
- **Policy disabled → discard:** `enabled=false` → discard counter increments, no channel created
- **outboundDestination wired correctly:** after auto-create, `fanOut()` to channel →
  `connectorService.send()` called with sender's phone as destination (verifies cache entry
  populated via `onChannelInitialised`)
- **No convention, no config → discard + ERROR:** connector type unknown to convention, no
  explicit outbound config → `Optional.empty()`, discard counter increments
- **Race condition — concurrent first contact:**

```java
@Test
void concurrentFirstContact_oneChannelCreated_bothMessagesDelivered() throws Exception {
    InboundMessage msg1 = new InboundMessage(TWILIO_SMS, "+447911000001", "+14155550000",
            "first", Instant.now(), Map.of());
    InboundMessage msg2 = new InboundMessage(TWILIO_SMS, "+447911000001", "+14155550000",
            "second", Instant.now(), Map.of());

    var f1 = CompletableFuture.runAsync(() -> backend.onInboundMessage(msg1));
    var f2 = CompletableFuture.runAsync(() -> backend.onInboundMessage(msg2));
    CompletableFuture.allOf(f1, f2).get(5, SECONDS);

    // Exactly one channel + binding created
    assertThat(channelBindingStore.findByKey(TWILIO_SMS, "+447911000001")).isPresent();
    assertThat(channelStore.scan(ChannelQuery.all())).hasSize(1);
    // Both messages dispatched
    verify(messageService, times(2)).dispatch(any());
}
```

### Existing test compatibility

**`ConnectorChannelBackendIntegrationTest`** — no `casehub.qhorus.connector.auto-channel.*`
entries in test `application.properties` → `ConfiguredAutoChannelPolicy.entries()` is empty →
`Optional.empty()` for all connectors → existing discard test passes unchanged.

**`ConnectorChannelBackendTest` (unit test)** — constructs `ConnectorChannelBackend` directly via
the 5-argument constructor. Adding `AutoChannelPolicy` as a new injected field changes this
constructor. Every test in the class will fail to compile. Fix: add `mock(AutoChannelPolicy.class)`
(returning `Optional.empty()` by default) as the 6th constructor argument. Tests exercising
the known-sender path are unaffected; the mock ensures the loser path is never entered.

---

## Out of Scope (tracked separately)

- Per-connector `InboundNormaliser` customisation → qhorus#216
- Delivery guarantees (retry, dead-letter) for auto-created channels → qhorus#132
- Rate-limiting channel creation per connector (flood from unknown senders) → file new issue
- MCP tools for listing/managing auto-created channels (existing `list_channels`, `get_channel`,
  `delete_channel` apply; `autoCreated` flag surfaced via `ChannelDetail`)
