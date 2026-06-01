# Auto-Channel Creation on First Contact — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When `ConnectorChannelBackend` receives an inbound message for an unknown sender, automatically create a Qhorus channel + binding and route the first message through, instead of discarding it.

**Architecture:** An `AutoChannelPolicy` SPI (in `connector-backend`) is consulted when no binding is found. A `@DefaultBean ConfiguredAutoChannelPolicy` reads `@ConfigMapping` per-connector config and applies a convention table for protocol-coupled connectors (SMS, WhatsApp). `ChannelService.findOrCreateWithBinding()` creates channel + binding atomically in a `REQUIRES_NEW` transaction; race losers catch `PersistenceException`, verify it is a constraint violation, and recover by finding the winner's channel.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JTA (Narayana), Hibernate ORM, H2 (tests), JUnit 5, Mockito, AssertJ.

**Spec:** `specs/2026-05-31-auto-channel-creation-design.md`
**Branch:** `issue-214-auto-channel-creation`
**Project root:** `/Users/mdproctor/claude/casehub/qhorus`

---

## File Map

| Action | File | What changes |
|--------|------|-------------|
| Modify | `testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java` | `put()` enforces unique compound key (synchronized) |
| Create | `runtime/src/main/resources/db/qhorus/migration/V15__add_channel_auto_created.sql` | New column `auto_created` on `channel` table |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java` | Add `autoCreated` boolean field |
| Modify | `runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java` | Assert `auto_created` column exists |
| Create | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelPolicy.java` | SPI interface |
| Create | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelSpec.java` | Immutable spec record |
| Create | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelConfig.java` | `@ConfigMapping` per-connector config |
| Create | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicy.java` | `@DefaultBean` config-driven SPI impl |
| Create | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java` | Unit tests for policy |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` | Add `findOrCreateWithBinding(ChannelCreateRequest)` |
| Modify | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java` | Inject policy, add `tryAutoCreate()`, `isConcurrentInsert()`, update `onInboundMessage()` |
| Modify | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendTest.java` | Add `AutoChannelPolicy` mock to constructor |
| Create | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelBackendTest.java` | Integration tests |

**Maven commands:**
```bash
# Full build (run after each task to catch cross-module issues)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests

# Run a specific test
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl <module>

# Run all tests in a module
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>

# Module names: testing, runtime, connector-backend
```

---

## Task 1: Fix `InMemoryChannelBindingStore.put()` — enforce unique constraint

The in-memory store currently overwrites on duplicate compound key silently. The race-condition integration test requires it to throw `PersistenceException` (matching the JPA `uq_binding_key` constraint), so the loser path in `ConnectorChannelBackend` can be exercised.

**Files:**
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java`

- [ ] **Step 1: Write a failing test for duplicate-key enforcement**

Add to (or create if needed) `testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelBindingStoreTest.java`:

```java
package io.casehub.qhorus.testing;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;
import jakarta.persistence.PersistenceException;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;

class InMemoryChannelBindingStoreTest {

    private InMemoryChannelBindingStore store = new InMemoryChannelBindingStore();

    private ChannelConnectorBinding binding(UUID channelId, String connId, String key) {
        ChannelConnectorBinding b = new ChannelConnectorBinding();
        b.channelId = channelId;
        b.inboundConnectorId = connId;
        b.externalKey = key;
        b.outboundConnectorId = "twilio-sms";
        b.outboundDestination = "+1";
        return b;
    }

    @Test
    void put_duplicateCompoundKey_differentChannelId_throws() {
        UUID channelA = UUID.randomUUID();
        UUID channelB = UUID.randomUUID();
        store.put(binding(channelA, "sms", "+447911000001"));

        assertThatThrownBy(() -> store.put(binding(channelB, "sms", "+447911000001")))
                .isInstanceOf(PersistenceException.class)
                .hasMessageContaining("uq_binding_key");
    }

    @Test
    void put_sameChannelId_updatesExisting() {
        UUID channelId = UUID.randomUUID();
        ChannelConnectorBinding first = binding(channelId, "sms", "+447911000001");
        first.outboundDestination = "+1111";
        store.put(first);

        ChannelConnectorBinding updated = binding(channelId, "sms", "+447911000001");
        updated.outboundDestination = "+2222";
        store.put(updated);  // should NOT throw — same channelId = update

        assertThat(store.findByKey("sms", "+447911000001"))
                .isPresent()
                .get()
                .extracting(b -> b.outboundDestination)
                .isEqualTo("+2222");
    }
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryChannelBindingStoreTest -pl testing
```

Expected: `put_duplicateCompoundKey_differentChannelId_throws` FAILS — no exception thrown.

- [ ] **Step 3: Fix `InMemoryChannelBindingStore.put()`**

Replace the `put()` method in `testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java`:

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
            throw new jakarta.persistence.PersistenceException(
                new java.sql.SQLIntegrityConstraintViolationException(
                    "Duplicate entry for uq_binding_key: " + newKey));
        }
        byChannelId.put(binding.channelId, binding);
        byKey.put(newKey, binding);
    }
}
```

Add the needed imports at the top of the class (the existing file uses `ConcurrentHashMap` etc.; add these):
```java
// (already present if the class compiles — these are in java.persistence and java.sql)
```

- [ ] **Step 4: Run tests to confirm both pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryChannelBindingStoreTest -pl testing
```

Expected: both tests PASS.

- [ ] **Step 5: Full build to catch any regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelBindingStoreTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(#214): InMemoryChannelBindingStore.put() throws PersistenceException on duplicate compound key"
```

---

## Task 2: V15 migration + `Channel.autoCreated` field

Adds the `auto_created` column that distinguishes auto-created channels from manually provisioned ones.

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V15__add_channel_auto_created.sql`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java`

- [ ] **Step 1: Write the Flyway schema test assertion (failing)**

Open `runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java` and add this test at the end of the class (before the closing `}`):

```java
@Test
void channelAutoCreatedColumnExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "")) {
        var rs = conn.getMetaData().getColumns(null, null, "CHANNEL", "AUTO_CREATED");
        assertTrue(rs.next(),
                "channel.auto_created column must exist — added by V15 migration");
        rs.close();
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FlywayMigrationSchemaTest -pl runtime
```

Expected: `channelAutoCreatedColumnExists` FAILS — column not found.

- [ ] **Step 3: Create the V15 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V15__add_channel_auto_created.sql`:

```sql
ALTER TABLE channel
    ADD COLUMN auto_created BOOLEAN NOT NULL DEFAULT FALSE;
```

- [ ] **Step 4: Add `autoCreated` field to `Channel` entity**

In `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`, add after the `paused` field:

```java
/** True when this channel was auto-created by ConnectorChannelBackend on first contact. */
@Column(name = "auto_created", nullable = false)
public boolean autoCreated = false;
```

- [ ] **Step 5: Run the schema test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FlywayMigrationSchemaTest -pl runtime
```

Expected: all tests PASS including `channelAutoCreatedColumnExists`.

- [ ] **Step 6: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    runtime/src/main/resources/db/qhorus/migration/V15__add_channel_auto_created.sql \
    runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java \
    runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#214): V15 migration — add channel.auto_created column"
```

---

## Task 3: `AutoChannelPolicy` SPI and `AutoChannelSpec` record

Pure types — no tests needed (no behaviour). These must exist before any code that references them compiles.

**Files:**
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelPolicy.java`
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelSpec.java`

- [ ] **Step 1: Create `AutoChannelSpec` record**

Create `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelSpec.java`:

```java
package io.casehub.qhorus.connector.backend;

import io.casehub.qhorus.api.channel.ChannelSemantic;

/**
 * Specification returned by {@link AutoChannelPolicy} describing the channel to create
 * on first contact from an unknown sender.
 *
 * <p>{@code allowedTypes} null = open channel (no message type restriction).
 * {@code description} should be human-readable, e.g. "Auto-created on first contact via twilio-sms-inbound".
 */
public record AutoChannelSpec(
        String channelName,
        String description,
        ChannelSemantic semantic,
        String allowedTypes,
        String outboundConnectorId,
        String outboundDestination
) {}
```

- [ ] **Step 2: Create `AutoChannelPolicy` interface**

Create `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelPolicy.java`:

```java
package io.casehub.qhorus.connector.backend;

import java.util.Optional;

import io.casehub.connectors.InboundMessage;

/**
 * SPI — determines whether and how to auto-create a Qhorus channel on first contact
 * from an unknown connector sender.
 *
 * <p>The {@code @DefaultBean} implementation is {@link ConfiguredAutoChannelPolicy},
 * which reads per-connector config. Displace it by providing your own
 * {@code @ApplicationScoped} bean in the consuming application.
 *
 * <p>Placement: {@code connector-backend} (not {@code api/spi/}) because the parameter
 * type {@code InboundMessage} comes from {@code casehub-connectors-core}, which the
 * {@code api} module does not depend on.
 *
 * @param msg        the inbound message that triggered the first-contact path
 * @param lookupKey  the key used for binding lookup (derived by {@code ConnectorKeyStrategy});
 *                   will become the channel's {@code externalKey} in the binding
 */
public interface AutoChannelPolicy {
    Optional<AutoChannelSpec> onFirstContact(InboundMessage msg, String lookupKey);
}
```

- [ ] **Step 3: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -pl connector-backend
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelPolicy.java \
    connector-backend/src/main/java/io/casehub/qhorus/connector/backend/AutoChannelSpec.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#214): AutoChannelPolicy SPI and AutoChannelSpec record"
```

---

## Task 4: `ConfiguredAutoChannelPolicy` — unit tests + implementation

TDD for the default SPI implementation: config-driven with convention mapping for protocol-coupled connectors.

**Files:**
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelConfig.java`
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicy.java`
- Create: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java`:

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.Map;
import java.util.Optional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundMessage;
import io.casehub.qhorus.api.channel.ChannelSemantic;

class ConfiguredAutoChannelPolicyTest {

    private ConnectorAutoChannelConfig config;
    private ConnectorAutoChannelConfig.ConnectorAutoChannelEntry smsEntry;
    private ConfiguredAutoChannelPolicy policy;

    private InboundMessage smsMsg(String sender) {
        return new InboundMessage(InboundConnectorIds.TWILIO_SMS, sender,
                "+14155550000", "hello", Instant.now(), Map.of());
    }

    private InboundMessage emailMsg(String sender) {
        return new InboundMessage(InboundConnectorIds.EMAIL, sender,
                "support@company.com", "hello", Instant.now(), Map.of());
    }

    @BeforeEach
    void setUp() {
        config = mock(ConnectorAutoChannelConfig.class);
        smsEntry = mock(ConnectorAutoChannelConfig.ConnectorAutoChannelEntry.class);
        when(smsEntry.enabled()).thenReturn(true);
        when(smsEntry.outboundConnectorId()).thenReturn(Optional.empty());
        when(smsEntry.channelNamePattern()).thenReturn(Optional.empty());
        when(smsEntry.semantic()).thenReturn(Optional.empty());
        when(config.entries()).thenReturn(Map.of(InboundConnectorIds.TWILIO_SMS, smsEntry));
        policy = new ConfiguredAutoChannelPolicy(config);
    }

    @Test
    void sms_enabled_conventionResolvesOutbound() {
        Optional<AutoChannelSpec> result = policy.onFirstContact(smsMsg("+447911000001"), "+447911000001");

        assertThat(result).isPresent();
        AutoChannelSpec spec = result.get();
        assertThat(spec.outboundConnectorId()).isEqualTo("twilio-sms");
        assertThat(spec.outboundDestination()).isEqualTo("+447911000001");
        assertThat(spec.semantic()).isEqualTo(ChannelSemantic.APPEND);
        assertThat(spec.allowedTypes()).isNull();
    }

    @Test
    void sms_enabled_defaultChannelName() {
        Optional<AutoChannelSpec> result = policy.onFirstContact(smsMsg("+447911000001"), "+447911000001");

        assertThat(result).isPresent();
        assertThat(result.get().channelName())
                .isEqualTo("connector/twilio-sms-inbound/+447911000001");
    }

    @Test
    void sms_enabled_descriptionMentionsConnectorAndSender() {
        Optional<AutoChannelSpec> result = policy.onFirstContact(smsMsg("+447911000001"), "+447911000001");

        assertThat(result).isPresent();
        assertThat(result.get().description())
                .contains("twilio-sms-inbound")
                .contains("+447911000001");
    }

    @Test
    void sms_customPattern_substitutesTokens() {
        when(smsEntry.channelNamePattern()).thenReturn(Optional.of("sms/{lookupKey}"));

        Optional<AutoChannelSpec> result = policy.onFirstContact(smsMsg("+447911000001"), "+447911000001");

        assertThat(result).isPresent();
        assertThat(result.get().channelName()).isEqualTo("sms/+447911000001");
    }

    @Test
    void sms_semanticOverride_appliesConfiguredSemantic() {
        when(smsEntry.semantic()).thenReturn(Optional.of("LAST_WRITE"));

        Optional<AutoChannelSpec> result = policy.onFirstContact(smsMsg("+447911"), "+447911");

        assertThat(result).isPresent();
        assertThat(result.get().semantic()).isEqualTo(ChannelSemantic.LAST_WRITE);
    }

    @Test
    void sms_disabled_returnsEmpty() {
        when(smsEntry.enabled()).thenReturn(false);

        assertThat(policy.onFirstContact(smsMsg("+447911"), "+447911")).isEmpty();
    }

    @Test
    void unknownConnector_noEntry_returnsEmpty() {
        InboundMessage slackMsg = new InboundMessage("slack-inbound", "U12345",
                "C67890", "hi", Instant.now(), Map.of());
        assertThat(policy.onFirstContact(slackMsg, "C67890")).isEmpty();
    }

    @Test
    void email_enabled_explicitOutbound_returnsSpec() {
        ConnectorAutoChannelConfig.ConnectorAutoChannelEntry emailEntry =
                mock(ConnectorAutoChannelConfig.ConnectorAutoChannelEntry.class);
        when(emailEntry.enabled()).thenReturn(true);
        when(emailEntry.outboundConnectorId()).thenReturn(Optional.of("email"));
        when(emailEntry.channelNamePattern()).thenReturn(Optional.empty());
        when(emailEntry.semantic()).thenReturn(Optional.empty());
        when(config.entries()).thenReturn(Map.of(InboundConnectorIds.EMAIL, emailEntry));

        Optional<AutoChannelSpec> result = policy.onFirstContact(
                emailMsg("alice@example.com"), "alice@example.com");

        assertThat(result).isPresent();
        assertThat(result.get().outboundConnectorId()).isEqualTo("email");
        assertThat(result.get().outboundDestination()).isEqualTo("alice@example.com");
    }

    @Test
    void email_enabled_noOutboundConfig_noConvention_returnsEmpty() {
        ConnectorAutoChannelConfig.ConnectorAutoChannelEntry emailEntry =
                mock(ConnectorAutoChannelConfig.ConnectorAutoChannelEntry.class);
        when(emailEntry.enabled()).thenReturn(true);
        when(emailEntry.outboundConnectorId()).thenReturn(Optional.empty());
        when(emailEntry.channelNamePattern()).thenReturn(Optional.empty());
        when(emailEntry.semantic()).thenReturn(Optional.empty());
        when(config.entries()).thenReturn(Map.of(InboundConnectorIds.EMAIL, emailEntry));

        // Email has no convention — should return empty + log error
        assertThat(policy.onFirstContact(emailMsg("alice@example.com"), "alice@example.com"))
                .isEmpty();
    }

    @Test
    void whatsapp_enabled_conventionResolvesOutbound() {
        ConnectorAutoChannelConfig.ConnectorAutoChannelEntry waEntry =
                mock(ConnectorAutoChannelConfig.ConnectorAutoChannelEntry.class);
        when(waEntry.enabled()).thenReturn(true);
        when(waEntry.outboundConnectorId()).thenReturn(Optional.empty());
        when(waEntry.channelNamePattern()).thenReturn(Optional.empty());
        when(waEntry.semantic()).thenReturn(Optional.empty());
        when(config.entries()).thenReturn(Map.of(InboundConnectorIds.WHATSAPP, waEntry));

        InboundMessage waMsg = new InboundMessage(InboundConnectorIds.WHATSAPP,
                "+44791100001", "+14155550000", "hi", Instant.now(), Map.of());

        Optional<AutoChannelSpec> result = policy.onFirstContact(waMsg, "+44791100001");

        assertThat(result).isPresent();
        assertThat(result.get().outboundConnectorId()).isEqualTo("whatsapp");
    }
}
```

- [ ] **Step 2: Run to confirm all tests fail (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConfiguredAutoChannelPolicyTest -pl connector-backend
```

Expected: COMPILATION ERROR — `ConnectorAutoChannelConfig` and `ConfiguredAutoChannelPolicy` not found.

- [ ] **Step 3: Create `ConnectorAutoChannelConfig`**

Create `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelConfig.java`:

```java
package io.casehub.qhorus.connector.backend;

import java.util.Map;
import java.util.Optional;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.qhorus.connector.auto-channel")
interface ConnectorAutoChannelConfig {

    /** Keyed by inbound connector ID, e.g. {@code "twilio-sms-inbound"}. */
    Map<String, ConnectorAutoChannelEntry> entries();

    interface ConnectorAutoChannelEntry {
        /** Whether auto-channel creation is active for this connector. Default: false (absent = disabled). */
        @WithDefault("false") boolean enabled();

        /** Outbound connector ID for replies. Required for email, Slack; optional for SMS, WhatsApp (convention applies). */
        Optional<String> outboundConnectorId();

        /**
         * Channel name pattern. Tokens: {@code {connectorId}}, {@code {lookupKey}}.
         * Default: {@code connector/{connectorId}/{lookupKey}}.
         */
        Optional<String> channelNamePattern();

        /** Channel semantic. Default: APPEND. */
        Optional<String> semantic();
    }
}
```

- [ ] **Step 4: Create `ConfiguredAutoChannelPolicy`**

Create `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicy.java`:

```java
package io.casehub.qhorus.connector.backend;

import java.util.Map;
import java.util.Optional;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundMessage;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.smallrye.config.ConfigMapping;
import org.jboss.logging.Logger;

@io.quarkus.arc.DefaultBean
@ApplicationScoped
class ConfiguredAutoChannelPolicy implements AutoChannelPolicy {

    private static final Logger LOG = Logger.getLogger(ConfiguredAutoChannelPolicy.class);

    // Convention table: protocol-coupled connectors where inbound and outbound
    // must use the same provider (SMS threading rules, WhatsApp API contract).
    private static final Map<String, String> OUTBOUND_CONVENTION = Map.of(
            InboundConnectorIds.TWILIO_SMS, "twilio-sms",
            InboundConnectorIds.WHATSAPP,   "whatsapp"
    );

    private final ConnectorAutoChannelConfig config;

    @Inject
    ConfiguredAutoChannelPolicy(ConnectorAutoChannelConfig config) {
        this.config = config;
    }

    @Override
    public Optional<AutoChannelSpec> onFirstContact(InboundMessage msg, String lookupKey) {
        ConnectorAutoChannelConfig.ConnectorAutoChannelEntry entry =
                config.entries().get(msg.connectorId());
        if (entry == null || !entry.enabled()) {
            return Optional.empty();
        }

        String outboundConnectorId = entry.outboundConnectorId()
                .or(() -> Optional.ofNullable(OUTBOUND_CONVENTION.get(msg.connectorId())))
                .orElse(null);

        if (outboundConnectorId == null) {
            LOG.errorf("auto-channel enabled for connector '%s' but no outbound-connector-id configured " +
                       "and no convention applies — add casehub.qhorus.connector.auto-channel.entries.\"%s\"" +
                       ".outbound-connector-id to application.properties",
                       msg.connectorId(), msg.connectorId());
            return Optional.empty();
        }

        String outboundDestination = ConnectorKeyStrategy.deriveKey(msg);

        String channelName = entry.channelNamePattern()
                .map(p -> p.replace("{connectorId}", msg.connectorId())
                            .replace("{lookupKey}", lookupKey))
                .orElse("connector/" + msg.connectorId() + "/" + lookupKey);

        ChannelSemantic semantic = entry.semantic()
                .map(ChannelSemantic::valueOf)
                .orElse(ChannelSemantic.APPEND);

        String description = "Auto-created on first contact via " + msg.connectorId()
                + " from " + lookupKey;

        return Optional.of(new AutoChannelSpec(
                channelName, description, semantic, null,
                outboundConnectorId, outboundDestination));
    }
}
```

- [ ] **Step 5: Run unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConfiguredAutoChannelPolicyTest -pl connector-backend
```

Expected: all tests PASS.

- [ ] **Step 6: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelConfig.java \
    connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicy.java \
    connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#214): ConfiguredAutoChannelPolicy — config-driven default with SMS/WhatsApp convention"
```

---

## Task 5: `ChannelService.findOrCreateWithBinding()` — TDD

The new transactional method that creates channel + binding atomically under `REQUIRES_NEW`. The recheck inside the transaction prevents concurrent threads from both creating a channel.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelServiceFindOrCreateTest.java`

- [ ] **Step 1: Write failing tests**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelServiceFindOrCreateTest.java`:

```java
package io.casehub.qhorus.runtime.channel;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.testing.InMemoryChannelBindingStore;
import io.casehub.qhorus.testing.InMemoryChannelStore;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class ChannelServiceFindOrCreateTest {

    @Inject ChannelService channelService;
    @Inject InMemoryChannelStore channelStore;
    @Inject InMemoryChannelBindingStore channelBindingStore;

    @BeforeEach
    void setUp() {
        channelStore.clear();
        channelBindingStore.clear();
    }

    private ChannelCreateRequest smsRequest(String senderPhone) {
        return new ChannelCreateRequest(
                "connector/twilio-sms-inbound/" + senderPhone,
                "Auto-created on first contact via twilio-sms-inbound from " + senderPhone,
                ChannelSemantic.APPEND,
                null, null, null, null, null, null,
                InboundConnectorIds.TWILIO_SMS, senderPhone, "twilio-sms", senderPhone);
    }

    @Test
    void createsChannelAndBindingWhenNotFound() {
        Channel result = channelService.findOrCreateWithBinding(smsRequest("+447911000001"));

        assertThat(result).isNotNull();
        assertThat(result.name).isEqualTo("connector/twilio-sms-inbound/+447911000001");
        assertThat(result.autoCreated).isTrue();
        assertThat(channelStore.scan(io.casehub.qhorus.runtime.store.query.ChannelQuery.all())).hasSize(1);
        assertThat(channelBindingStore.findByKey(InboundConnectorIds.TWILIO_SMS, "+447911000001")).isPresent();
    }

    @Test
    void returnsExistingChannelWhenAlreadyCreated() {
        Channel first = channelService.findOrCreateWithBinding(smsRequest("+447911000001"));
        Channel second = channelService.findOrCreateWithBinding(smsRequest("+447911000001"));

        assertThat(second.id).isEqualTo(first.id);
        assertThat(channelStore.scan(io.casehub.qhorus.runtime.store.query.ChannelQuery.all())).hasSize(1);
    }

    @Test
    void setsAutoCreatedTrue() {
        Channel result = channelService.findOrCreateWithBinding(smsRequest("+447911000002"));
        assertThat(result.autoCreated).isTrue();
    }

    @Test
    void differentSenders_createSeparateChannels() {
        channelService.findOrCreateWithBinding(smsRequest("+447911000001"));
        channelService.findOrCreateWithBinding(smsRequest("+447911000002"));

        assertThat(channelStore.scan(io.casehub.qhorus.runtime.store.query.ChannelQuery.all())).hasSize(2);
    }
}
```

- [ ] **Step 2: Run to confirm test fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelServiceFindOrCreateTest -pl runtime
```

Expected: COMPILATION ERROR — `findOrCreateWithBinding` not found.

- [ ] **Step 3: Add `findOrCreateWithBinding()` to `ChannelService`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`, add after the existing `create(ChannelCreateRequest)` method (around line 118):

```java
/**
 * Finds an existing channel by connector binding key, or creates one atomically if not found.
 *
 * <p>Runs in a new transaction ({@code REQUIRES_NEW}) so the channel and binding are
 * committed before {@link io.casehub.qhorus.runtime.gateway.ChannelGateway#initChannel}
 * fires — regardless of any outer transaction context. The commit happens at the CDI proxy
 * boundary when this method returns, not inside the method body.
 *
 * <p>If two threads race on the same {@code req.inboundConnectorId()} + {@code req.externalKey()},
 * one wins the DB insert; the other receives a {@link jakarta.persistence.PersistenceException}
 * wrapping a unique-constraint violation on {@code uq_binding_key}. The caller must catch that
 * exception and retry {@link #findByConnectorKey} to obtain the winner's channel.
 *
 * @param req must have {@link ChannelCreateRequest#hasConnectorBinding()} == true
 * @return the found or newly created channel, with {@code autoCreated} set to {@code true}
 *         for new channels
 */
@Transactional(jakarta.transaction.Transactional.TxType.REQUIRES_NEW)
public Channel findOrCreateWithBinding(ChannelCreateRequest req) {
    if (!req.hasConnectorBinding()) {
        throw new IllegalArgumentException("findOrCreateWithBinding requires a connector binding");
    }
    // Recheck under the new transaction — race winner will have committed before this runs.
    Optional<Channel> existing = channelBindingStore
            .findByKey(req.inboundConnectorId(), req.externalKey())
            .flatMap(b -> channelStore.find(b.channelId));
    if (existing.isPresent()) {
        return existing.get();
    }

    Channel channel = new Channel();
    channel.name = req.name();
    channel.description = req.description();
    channel.semantic = req.semantic();
    channel.barrierContributors = req.barrierContributors();
    channel.allowedWriters = (req.allowedWriters() == null || req.allowedWriters().isBlank())
            ? null : req.allowedWriters();
    channel.adminInstances = (req.adminInstances() == null || req.adminInstances().isBlank())
            ? null : req.adminInstances();
    channel.rateLimitPerChannel = req.rateLimitPerChannel();
    channel.rateLimitPerInstance = req.rateLimitPerInstance();
    channel.allowedTypes = (req.allowedTypes() == null || req.allowedTypes().isBlank())
            ? null : req.allowedTypes();
    channel.autoCreated = true;
    channelStore.put(channel);

    ChannelConnectorBinding binding = new ChannelConnectorBinding();
    binding.channelId = channel.id;
    binding.inboundConnectorId = req.inboundConnectorId();
    binding.externalKey = req.externalKey();
    binding.outboundConnectorId = req.outboundConnectorId();
    binding.outboundDestination = req.outboundDestination();
    channelBindingStore.put(binding);

    return channel;
}
```

Add the import at the top if not already present:
```java
import jakarta.transaction.Transactional;
```

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelServiceFindOrCreateTest -pl runtime
```

Expected: all 4 tests PASS.

- [ ] **Step 5: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java \
    runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelServiceFindOrCreateTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#214): ChannelService.findOrCreateWithBinding() — REQUIRES_NEW atomic channel+binding creation"
```

---

## Task 6: Update `ConnectorChannelBackend` — inject policy, `tryAutoCreate()`, `onInboundMessage()`

Wires the policy into the backend. Adds `tryAutoCreate()` with `isConcurrentInsert()` discrimination, and updates `onInboundMessage()` to use it.

**Files:**
- Modify: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java`
- Modify: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendTest.java`

- [ ] **Step 1: Update `ConnectorChannelBackendTest` constructor (existing tests must still pass)**

In `ConnectorChannelBackendTest.java`, add `AutoChannelPolicy` mock to `setUp()`. The mock returns `Optional.empty()` by default (Mockito default for Optional-returning methods), so existing tests are unaffected.

Replace the `setUp()` method:

```java
@BeforeEach
void setUp() {
    gateway = mock(ChannelGateway.class);
    channelService = mock(ChannelService.class);
    bindingStore = mock(ChannelBindingStore.class);
    connectorService = mock(ConnectorService.class);
    AutoChannelPolicy autoChannelPolicy = mock(AutoChannelPolicy.class);
    // Mockito default: Optional-returning methods return Optional.empty()
    backend = new ConnectorChannelBackend(gateway, channelService, bindingStore,
            connectorService, new SimpleMeterRegistry(), autoChannelPolicy);
}
```

Add the import:
```java
import io.casehub.qhorus.connector.backend.AutoChannelPolicy;
```

- [ ] **Step 2: Run existing unit tests to confirm they still pass (will fail — constructor not updated yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConnectorChannelBackendTest -pl connector-backend
```

Expected: COMPILATION ERROR — constructor not matching yet.

- [ ] **Step 3: Update `ConnectorChannelBackend`**

Replace the full contents of `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java`:

```java
package io.casehub.qhorus.connector.backend;

import java.sql.SQLIntegrityConstraintViolationException;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.persistence.PersistenceException;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.InboundMessage;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.ChannelInitialisedEvent;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.HumanParticipatingChannelBackend;
import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;
import io.micrometer.core.instrument.MeterRegistry;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ConnectorChannelBackend implements HumanParticipatingChannelBackend {

    private static final Logger LOG = Logger.getLogger(ConnectorChannelBackend.class);
    private static final String BACKEND_ID = "connector-human";

    private final ChannelGateway gateway;
    private final ChannelService channelService;
    private final ChannelBindingStore bindingStore;
    private final ConnectorService connectorService;
    private final MeterRegistry meterRegistry;
    private final AutoChannelPolicy autoChannelPolicy;

    private final ConcurrentHashMap<UUID, CacheEntry> cache = new ConcurrentHashMap<>();

    @Inject
    public ConnectorChannelBackend(
            final ChannelGateway gateway,
            final ChannelService channelService,
            final ChannelBindingStore bindingStore,
            final ConnectorService connectorService,
            final MeterRegistry meterRegistry,
            final AutoChannelPolicy autoChannelPolicy) {
        this.gateway = gateway;
        this.channelService = channelService;
        this.bindingStore = bindingStore;
        this.connectorService = connectorService;
        this.meterRegistry = meterRegistry;
        this.autoChannelPolicy = autoChannelPolicy;
    }

    @Override
    public String backendId() {
        return BACKEND_ID;
    }

    @Override
    public ActorType actorType() {
        return ActorType.HUMAN;
    }

    @Override
    public void open(final ChannelRef channel, final Map<String, String> metadata) {
        // no-op — registration is driven by ChannelInitialisedEvent
    }

    @Override
    public void close(final ChannelRef channel) {
        cache.remove(channel.id());
    }

    public void onChannelInitialised(@Observes final ChannelInitialisedEvent event) {
        UUID channelId = event.channelId();
        bindingStore.findByChannelId(channelId).ifPresentOrElse(binding -> {
            cache.put(channelId, new CacheEntry(
                    binding.inboundConnectorId,
                    binding.externalKey,
                    binding.outboundConnectorId,
                    binding.outboundDestination));
            gateway.deregisterBackend(channelId, BACKEND_ID);
            gateway.registerBackend(channelId, this, "human_participating");
        }, () -> {
            // no binding — not a connector-backed channel, skip
        });
    }

    /**
     * Receives an inbound message via CDI async event delivery.
     *
     * <p>Returns {@code CompletionStage<Void>} so that callers using
     * {@code Event.fireAsync().toCompletableFuture().join()} reliably wait for this
     * observer to finish before asserting.
     */
    public CompletionStage<Void> onInboundMessage(@ObservesAsync final InboundMessage msg) {
        String lookupKey = ConnectorKeyStrategy.deriveKey(msg);

        channelService.findByConnectorKey(msg.connectorId(), lookupKey)
                .or(() -> tryAutoCreate(msg, lookupKey))
                .ifPresentOrElse(
                        channel -> route(channel, msg),
                        () -> {
                            LOG.warnf("No channel for connector=%s key=%s — discarding",
                                    msg.connectorId(), lookupKey);
                            meterRegistry.counter("inbound_messages_discarded_total",
                                    "connector_id", msg.connectorId()).increment();
                        });

        return CompletableFuture.completedFuture(null);
    }

    private Optional<Channel> tryAutoCreate(InboundMessage msg, String lookupKey) {
        Optional<AutoChannelSpec> specOpt = autoChannelPolicy.onFirstContact(msg, lookupKey);
        if (specOpt.isEmpty()) {
            return Optional.empty();
        }
        AutoChannelSpec spec = specOpt.get();
        ChannelCreateRequest req = new ChannelCreateRequest(
                spec.channelName(),
                spec.description(),
                spec.semantic(),
                null, null, null, null, null,
                spec.allowedTypes(),
                msg.connectorId(),
                lookupKey,
                spec.outboundConnectorId(),
                spec.outboundDestination());
        try {
            Channel channel = channelService.findOrCreateWithBinding(req);
            meterRegistry.counter("inbound_channels_auto_created_total",
                    "connector_id", msg.connectorId()).increment();
            gateway.initChannel(channel.id, new ChannelRef(channel.id, channel.name));
            return Optional.of(channel);
        } catch (PersistenceException ex) {
            if (isConcurrentInsert(ex)) {
                // Race loser: winner's REQUIRES_NEW committed; find their channel.
                // initChannel() is NOT called here — winner already fired it.
                // Thread B's push delivery may miss if winner's initChannel() hasn't run yet;
                // message is still persisted (at-most-once push delivery contract).
                return channelService.findByConnectorKey(msg.connectorId(), lookupKey)
                        .map(Optional::of)
                        .orElseThrow(() -> new IllegalStateException(
                                "Race recovery failed: uq_binding_key violated but channel not found"
                                + " for connector=" + msg.connectorId() + " key=" + lookupKey));
            }
            LOG.errorf(ex, "DB error auto-creating channel for connector=%s key=%s — discarding",
                    msg.connectorId(), lookupKey);
            return Optional.empty();
        }
    }

    /**
     * Returns true if {@code ex} is a unique-constraint violation on the binding key
     * (uq_binding_key), indicating a concurrent first-contact race. Returns false for
     * other DB errors (connection failure, schema errors, etc.) which must not silently
     * enter the race-loser recovery path.
     */
    static boolean isConcurrentInsert(PersistenceException ex) {
        Throwable cause = ex;
        while (cause != null) {
            if (cause instanceof SQLIntegrityConstraintViolationException c) {
                String msg = c.getMessage() != null ? c.getMessage().toLowerCase() : "";
                return msg.contains("uq_binding_key") || msg.contains("unique");
            }
            cause = cause.getCause();
        }
        return false;
    }

    private void route(Channel channel, InboundMessage msg) {
        gateway.receiveHumanMessage(
                new ChannelRef(channel.id, channel.name),
                new InboundHumanMessage(
                        msg.externalSenderId(),
                        msg.content(),
                        msg.receivedAt(),
                        msg.metadata(),
                        null,
                        null));
    }

    @Override
    public void post(final ChannelRef channel, final OutboundMessage message) {
        CacheEntry entry = cache.get(channel.id());
        if (entry == null) {
            LOG.errorf("No cache entry for channel %s (%s) — cannot post outbound message",
                    channel.id(), channel.name());
            return;
        }
        String title = OutboundTitle.forConnector(entry.outboundConnectorId(), channel);
        try {
            connectorService.send(entry.outboundConnectorId(),
                    new ConnectorMessage(entry.outboundDestination(), title, message.content()));
        } catch (IllegalArgumentException ex) {
            LOG.errorf(ex, "Failed to send via connector %s to channel %s (%s)",
                    entry.outboundConnectorId(), channel.id(), channel.name());
        }
    }

    /** Package-private test helper — reads the discarded message counter for a connector. */
    double discardedCount(final String connectorId) {
        return meterRegistry.counter("inbound_messages_discarded_total",
                "connector_id", connectorId).count();
    }

    /** Package-private test helper — reads the auto-created channel counter for a connector. */
    double autoCreatedCount(final String connectorId) {
        return meterRegistry.counter("inbound_channels_auto_created_total",
                "connector_id", connectorId).count();
    }

    private record CacheEntry(
            String inboundConnectorId,
            String externalKey,
            String outboundConnectorId,
            String outboundDestination) {}
}
```

- [ ] **Step 4: Run the unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConnectorChannelBackendTest -pl connector-backend
```

Expected: all existing tests PASS.

- [ ] **Step 5: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java \
    connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#214): ConnectorChannelBackend — AutoChannelPolicy integration, tryAutoCreate(), isConcurrentInsert()"
```

---

## Task 7: Integration tests — `ConnectorAutoChannelBackendTest`

Verifies the full end-to-end auto-channel creation flow including: first contact, idempotency, discard when disabled, outbound routing, and concurrent first contact (race).

**Files:**
- Create: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelBackendTest.java`

- [ ] **Step 1: Create the integration test class**

Create `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelBackendTest.java`:

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundMessage;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.store.query.ChannelQuery;
import io.casehub.qhorus.testing.InMemoryChannelBindingStore;
import io.casehub.qhorus.testing.InMemoryChannelStore;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class ConnectorAutoChannelBackendTest {

    @Inject ConnectorChannelBackend backend;
    @Inject ChannelService channelService;
    @Inject ChannelGateway gateway;
    @Inject InMemoryChannelStore channelStore;
    @Inject InMemoryChannelBindingStore channelBindingStore;
    @Inject Event<InboundMessage> inboundMessageEvent;

    @InjectMock MessageService messageService;
    @InjectMock ConnectorService connectorService;
    @InjectMock AutoChannelPolicy autoChannelPolicy;

    private static final String SENDER = "+447911000099";
    private static final String CONNECTOR = InboundConnectorIds.TWILIO_SMS;

    @BeforeEach
    void setUp() {
        channelStore.clear();
        channelBindingStore.clear();
    }

    @AfterEach
    void tearDown() {
        // Clean up any auto-created channels from the gateway registry
        channelStore.scan(ChannelQuery.all()).forEach(ch ->
                gateway.closeChannel(ch.id, new ChannelRef(ch.id, ch.name)));
    }

    private InboundMessage smsMsg(String sender, String content) {
        return new InboundMessage(CONNECTOR, sender, "+14155550000", content, Instant.now(), Map.of());
    }

    private AutoChannelSpec smsSpec(String sender) {
        return new AutoChannelSpec(
                "connector/" + CONNECTOR + "/" + sender,
                "Auto-created on first contact via " + CONNECTOR + " from " + sender,
                ChannelSemantic.APPEND,
                null,
                "twilio-sms",
                sender);
    }

    // -------------------------------------------------------------------------
    // Auto-create on first contact
    // -------------------------------------------------------------------------

    @Test
    void firstContact_policyEnabled_channelCreatedAndMessageRouted() {
        when(autoChannelPolicy.onFirstContact(any(), eq(SENDER)))
                .thenReturn(Optional.of(smsSpec(SENDER)));

        backend.onInboundMessage(smsMsg(SENDER, "hello"));

        // Channel and binding created
        assertThat(channelBindingStore.findByKey(CONNECTOR, SENDER)).isPresent();
        assertThat(channelStore.scan(ChannelQuery.all())).hasSize(1);
        assertThat(channelStore.scan(ChannelQuery.all()).get(0).autoCreated).isTrue();

        // Message dispatched
        verify(messageService).dispatch(argThat(d ->
                "human:" + SENDER equals d.sender()
                && "hello".equals(d.content())));

        // Counter incremented
        assertThat(backend.autoCreatedCount(CONNECTOR)).isEqualTo(1.0);
    }

    @Test
    void secondMessage_sameSender_reusesChannel_noNewChannelCreated() {
        when(autoChannelPolicy.onFirstContact(any(), eq(SENDER)))
                .thenReturn(Optional.of(smsSpec(SENDER)));

        // First message — auto-creates
        backend.onInboundMessage(smsMsg(SENDER, "first"));
        UUID channelId = channelStore.scan(ChannelQuery.all()).get(0).id;
        // Wire the gateway so second message routes via known channel
        gateway.initChannel(channelId, new ChannelRef(channelId, "connector/" + CONNECTOR + "/" + SENDER));

        // Second message — finds existing channel via findByConnectorKey
        backend.onInboundMessage(smsMsg(SENDER, "second"));

        assertThat(channelStore.scan(ChannelQuery.all())).hasSize(1);
        // autoCreatedCount still 1 — not incremented on second message
        assertThat(backend.autoCreatedCount(CONNECTOR)).isEqualTo(1.0);
        verify(messageService, times(2)).dispatch(any());
    }

    @Test
    void policyDisabled_returnsEmpty_messageDiscarded() {
        when(autoChannelPolicy.onFirstContact(any(), any())).thenReturn(Optional.empty());
        double before = backend.discardedCount(CONNECTOR);

        backend.onInboundMessage(smsMsg(SENDER, "hello"));

        assertThat(backend.discardedCount(CONNECTOR)).isGreaterThan(before);
        verify(messageService, never()).dispatch(any());
        assertThat(channelStore.scan(ChannelQuery.all())).isEmpty();
    }

    // -------------------------------------------------------------------------
    // Outbound routing after auto-creation
    // -------------------------------------------------------------------------

    @Test
    void afterAutoCreate_fanOut_sendsToSenderPhone() {
        when(autoChannelPolicy.onFirstContact(any(), eq(SENDER)))
                .thenReturn(Optional.of(smsSpec(SENDER)));

        backend.onInboundMessage(smsMsg(SENDER, "hello"));

        UUID channelId = channelStore.scan(ChannelQuery.all()).get(0).id;
        OutboundMessage reply = new OutboundMessage(UUID.randomUUID(), "agent",
                MessageType.RESPONSE, "reply text", null, null, ActorType.AGENT);
        gateway.fanOut(channelId, "connector/" + CONNECTOR + "/" + SENDER, reply);

        // Virtual-thread dispatch — brief timeout
        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService, timeout(1000).atLeastOnce()).send(eq("twilio-sms"), captor.capture());
        assertThat(captor.getValue().destination()).isEqualTo(SENDER);
        assertThat(captor.getValue().body()).isEqualTo("reply text");
    }

    // -------------------------------------------------------------------------
    // Race condition — concurrent first contact
    // -------------------------------------------------------------------------

    @Test
    void concurrentFirstContact_oneBindingCreated_bothMessagesDelivered() throws Exception {
        when(autoChannelPolicy.onFirstContact(any(), eq(SENDER)))
                .thenReturn(Optional.of(smsSpec(SENDER)));

        InboundMessage msg1 = smsMsg(SENDER, "first");
        InboundMessage msg2 = smsMsg(SENDER, "second");

        CompletableFuture<Void> f1 = CompletableFuture.runAsync(
                () -> backend.onInboundMessage(msg1));
        CompletableFuture<Void> f2 = CompletableFuture.runAsync(
                () -> backend.onInboundMessage(msg2));
        CompletableFuture.allOf(f1, f2).get(5, TimeUnit.SECONDS);

        // Exactly one binding (unique constraint enforced)
        assertThat(channelBindingStore.findByKey(CONNECTOR, SENDER)).isPresent();
        // Both messages dispatched
        verify(messageService, times(2)).dispatch(any());
        // Auto-created counter: exactly 1 (only winner increments)
        assertThat(backend.autoCreatedCount(CONNECTOR)).isEqualTo(1.0);
    }
}
```

Fix the typo in line: `"human:" + SENDER equals d.sender()` → use `.equals()`:

Actually, that line should be:
```java
("human:" + SENDER).equals(d.sender())
```

Update the test assertion in `firstContact_policyEnabled_channelCreatedAndMessageRouted`:
```java
verify(messageService).dispatch(argThat(d ->
        ("human:" + SENDER).equals(d.sender())
        && "hello".equals(d.content())));
```

- [ ] **Step 2: Run integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConnectorAutoChannelBackendTest -pl connector-backend
```

Expected: all tests PASS. If `concurrentFirstContact` is flaky (timing), try running a few times. The `synchronized` block in `InMemoryChannelBindingStore` makes the race deterministic.

- [ ] **Step 3: Run all connector-backend tests to check no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend
```

Expected: all tests PASS.

- [ ] **Step 4: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorAutoChannelBackendTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#214): ConnectorAutoChannelBackendTest — auto-create, idempotency, discard, fanOut, race condition"
```

---

## Self-Review Checklist

Spec coverage:
- [x] `AutoChannelPolicy` SPI + `AutoChannelSpec` → Task 3
- [x] `ConfiguredAutoChannelPolicy` + `ConnectorAutoChannelConfig` → Task 4
- [x] `ChannelService.findOrCreateWithBinding()` → Task 5
- [x] `ConnectorChannelBackend` update → Task 6
- [x] `isConcurrentInsert()` utility → Task 6 (inside backend)
- [x] `InMemoryChannelBindingStore.put()` fix → Task 1
- [x] V15 migration + `Channel.autoCreated` → Task 2
- [x] `FlywayMigrationSchemaTest` update → Task 2
- [x] `ConfiguredAutoChannelPolicyTest` unit tests → Task 4
- [x] `ConnectorChannelBackendTest` constructor update → Task 6
- [x] `ConnectorAutoChannelBackendTest` integration tests → Task 7
- [x] Race condition test → Task 7 (`concurrentFirstContact_...`)
- [x] `@WithDefault("false")` in config → Task 4 (`ConnectorAutoChannelConfig`)
- [x] `autoCreated` counter (winner only) → Task 6 (backend `tryAutoCreate()`)
- [x] `initChannel()` winner-only → Task 6 (loser comment in code)
- [x] Exception discrimination (`isConcurrentInsert`) → Task 6

All spec requirements covered.
