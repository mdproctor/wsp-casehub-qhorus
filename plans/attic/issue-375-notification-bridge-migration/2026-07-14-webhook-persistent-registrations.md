# Webhook Persistent Registrations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #345 — Webhook observer: persistent registrations (JPA)
**Issue group:** #345

**Goal:** Add JPA-backed durable webhook registrations to the webhook-observer module so subscriptions survive restarts, replacing plaintext `secret` with `secretRef` via `CredentialResolver`.

**Architecture:** The in-memory `ConcurrentHashMap` remains the hot-path lookup. JPA provides durability — registrations load into memory on startup, and `register()`/`deregister()` write-through to the store. `ChannelClosedEvent` (new CDI event in the API module) triggers cleanup. `secretRef` replaces `secret` throughout, resolved via `CredentialResolver` at POST time.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (Hibernate ORM, `qhorus` named PU), H2 (tests), Flyway V35

## Global Constraints

- Flyway V35 is the next domain migration slot
- `SIGNING_SECRET = "signing-secret"` must be added to `CredentialPropertyKeys` in `casehub-platform-api` (cross-repo prerequisite)
- `quarkus.hibernate-orm.qhorus.packages` must include `io.casehub.qhorus.webhook` in any consumer (per PP-20260618-d9aeef)
- `TenancyConstants.DEFAULT_TENANT_ID = "278776f9-e1b0-46fb-9032-8bddebdcf9ce"` — UUID string, not `"DEFAULT"`
- Pre-release — breaking changes cost nothing

---

### Task 1: Cross-repo prerequisite — SIGNING_SECRET in CredentialPropertyKeys

**Files:**
- Modify: `casehub-platform-api` — `io.casehub.platform.api.credentials.CredentialPropertyKeys` (in `~/claude/casehub/platform/api`)

**Interfaces:**
- Produces: `CredentialPropertyKeys.SIGNING_SECRET` — `public static final String SIGNING_SECRET = "signing-secret";`

This is a cross-repo change to `casehub-platform-api`. After adding the constant, run `mvn install` on the platform-api module so the SNAPSHOT jar is available locally for qhorus.

- [ ] **Step 1: Add the constant**

```java
public static final String SIGNING_SECRET = "signing-secret";
```

Add after the existing `API_KEY` constant in `CredentialPropertyKeys`.

- [ ] **Step 2: Build and install**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f ~/claude/casehub/platform/api/pom.xml clean install -DskipTests
```

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/casehub/platform/api add src/main/java/io/casehub/platform/api/credentials/CredentialPropertyKeys.java
git -C ~/claude/casehub/platform/api commit -m "feat(#345): add SIGNING_SECRET to CredentialPropertyKeys

Refs casehubio/qhorus#345"
```

---

### Task 2: ChannelClosedEvent in the API module + fire from ChannelGateway

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelClosedEvent.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` — `closeChannel()` method + new field + constructor
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayClosedEventTest.java`

**Interfaces:**
- Produces: `ChannelClosedEvent(UUID channelId, String channelName)` — CDI event record in `io.casehub.qhorus.api.gateway`
- Produces: `ChannelGateway.closeChannel()` fires `ChannelClosedEvent` after closing all backends

- [ ] **Step 1: Write the failing test**

CDI-free unit test. Construct a `ChannelGateway` with a recording `Event<ChannelClosedEvent>`, call `closeChannel()`, assert the event was fired with the correct channelId and channelName.

```java
package io.casehub.qhorus.runtime.gateway;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.ChannelClosedEvent;
import io.casehub.qhorus.api.gateway.ChannelRef;

class ChannelGatewayClosedEventTest {

    @Test
    void closeChannelFiresClosedEvent() {
        var firedEvents = new ArrayList<ChannelClosedEvent>();
        // ChannelGateway has package-private fields; construct via the CDI constructor
        // then set the channelClosedEvents field directly
        var gateway = TestChannelGatewayBuilder.withClosedEventRecorder(firedEvents);

        UUID channelId = UUID.randomUUID();
        gateway.closeChannel(channelId, new ChannelRef(channelId, "test-ch"));

        assertThat(firedEvents).hasSize(1);
        assertThat(firedEvents.get(0).channelId()).isEqualTo(channelId);
        assertThat(firedEvents.get(0).channelName()).isEqualTo("test-ch");
    }

    @Test
    void closeChannelFiresEventAfterBackendsAreClosed() {
        var firedEvents = new ArrayList<ChannelClosedEvent>();
        var closeOrder = new ArrayList<String>();
        var gateway = TestChannelGatewayBuilder.withClosedEventRecorder(firedEvents);

        UUID channelId = UUID.randomUUID();
        // Register a recording backend
        gateway.registerBackend(channelId,
                new RecordingBackend("test-backend", closeOrder),
                "agent");
        gateway.closeChannel(channelId, new ChannelRef(channelId, "test-ch"));

        // Backend closed first, then event fired
        assertThat(closeOrder).containsExactly("test-backend:closed");
        assertThat(firedEvents).hasSize(1);
    }
}
```

The `TestChannelGatewayBuilder` helper and `RecordingBackend` are test infrastructure — construct a `ChannelGateway` with null/no-op dependencies except the fields under test. Use the existing `ChannelGatewayTest` pattern for reference.

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelGatewayClosedEventTest -f pom.xml
```

Expected: compilation failure — `ChannelClosedEvent` does not exist.

- [ ] **Step 3: Create ChannelClosedEvent record**

Create `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelClosedEvent.java`:

```java
package io.casehub.qhorus.api.gateway;

import java.util.UUID;

/**
 * Fired by ChannelGateway after all backends have been closed for a channel.
 * Enables observers (e.g. webhook registry, external integrations) to clean up
 * channel-specific state without implementing ChannelBackend.
 *
 * Refs #345.
 */
public record ChannelClosedEvent(UUID channelId, String channelName) {}
```

- [ ] **Step 4: Add Event<ChannelClosedEvent> to ChannelGateway**

Add a new field `final Event<ChannelClosedEvent> channelClosedEvents` to `ChannelGateway`. Add it to the constructor. Fire after the backend close loop in `closeChannel()`:

```java
public void closeChannel(UUID channelId, ChannelRef ref) {
    List<BackendEntry> entries = registry.remove(channelId);
    if (entries != null) {
        for (BackendEntry e : entries) {
            try {
                e.backend().close(ref);
            } catch (Exception ex) {
                LOG.errorf(ex, "Error closing backend %s on channel %s",
                        e.backend().backendId(), channelId);
            }
        }
    }
    channelClosedEvents.fire(new ChannelClosedEvent(channelId, ref.name()));
}
```

- [ ] **Step 5: Create test helpers, run tests**

Build `TestChannelGatewayBuilder` as a package-private helper in the test package that constructs `ChannelGateway` with a recording `Event<ChannelClosedEvent>` implementation (simple list-add lambda wrapper). Null/no-op for all other constructor params (the test only exercises `closeChannel` which doesn't touch them).

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelGatewayClosedEventTest -f pom.xml
```

Expected: PASS

- [ ] **Step 6: Verify existing tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f pom.xml
```

The new constructor parameter may break existing CDI-free `ChannelGateway` test constructors — fix by adding a null `channelClosedEvents` argument to those call sites.

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/gateway/ChannelClosedEvent.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayClosedEventTest.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/gateway/TestChannelGatewayBuilder.java
git commit -m "feat(#345): ChannelClosedEvent — fired by ChannelGateway.closeChannel()

Mirrors ChannelInitialisedEvent pattern. Enables observers to react to
channel deletion without implementing ChannelBackend.

Refs #345"
```

---

### Task 3: Flyway V35 migration

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V35__webhook_registration.sql`

**Interfaces:**
- Produces: `webhook_registration` table with columns `id`, `channel_id`, `url`, `secret_ref`, `headers`, `tenancy_id`, `created_at`

- [ ] **Step 1: Create V35 migration**

```sql
CREATE TABLE webhook_registration (
    id UUID PRIMARY KEY,
    channel_id UUID,
    url VARCHAR(2048) NOT NULL,
    secret_ref VARCHAR(255),
    headers TEXT,
    tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT uq_webhook_url_channel_tenant UNIQUE (url, channel_id, tenancy_id),
    CONSTRAINT fk_webhook_channel FOREIGN KEY (channel_id) REFERENCES channel(id)
);

CREATE UNIQUE INDEX uq_webhook_global ON webhook_registration (url, tenancy_id)
    WHERE channel_id IS NULL;
```

No `ON DELETE CASCADE` — explicit cleanup via `ChannelClosedEvent` observer (per design review R1-05).

- [ ] **Step 2: Commit**

```bash
git add runtime/src/main/resources/db/qhorus/migration/V35__webhook_registration.sql
git commit -m "feat(#345): V35 migration — webhook_registration table

Partial unique index for global webhooks (NULL ≠ NULL in SQL UNIQUE).
No CASCADE — explicit cleanup via ChannelClosedEvent.

Refs #345"
```

---

### Task 4: MapToJsonConverter + WebhookRegistrationEntity + WebhookRegistrationStore

**Files:**
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/MapToJsonConverter.java`
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistrationEntity.java`
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistrationStore.java`
- Modify: `webhook-observer/pom.xml` — add JPA dependencies
- Create: `webhook-observer/src/test/resources/application.properties` — H2 config for @QuarkusTest
- Create: `webhook-observer/src/test/resources/import-webhook-test.sql` — ledger_subject_sequence
- Test: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookRegistrationStoreTest.java`
- Test: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookFlywaySchemaTest.java`

**Interfaces:**
- Consumes: V35 migration (Task 3)
- Produces: `WebhookRegistrationEntity` — JPA entity with `@PersistenceUnit("qhorus")`
- Produces: `WebhookRegistrationStore` — `findById`, `findByChannelId`, `findGlobal`, `findAll`, `save`, `delete`, `deleteByChannelId`
- Produces: `MapToJsonConverter` — `AttributeConverter<Map<String,String>, String>`

- [ ] **Step 1: Add JPA dependencies to webhook-observer/pom.xml**

Add these dependencies:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-api</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
```

Add test dependencies:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-h2</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-persistence-memory</artifactId>
    <version>${project.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform</artifactId>
    <version>0.2-SNAPSHOT</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5-mockito</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Create test application.properties**

`webhook-observer/src/test/resources/application.properties`:

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.qhorus.persistence.memory.InMemoryChannelStore,\
  io.casehub.qhorus.persistence.memory.InMemoryMessageStore,\
  io.casehub.qhorus.persistence.memory.InMemoryInstanceStore,\
  io.casehub.qhorus.persistence.memory.InMemoryDataStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore,\
  io.casehub.qhorus.persistence.memory.InMemoryWatchdogStore,\
  io.casehub.qhorus.persistence.memory.InMemoryDeliveryCursorStore,\
  io.casehub.qhorus.persistence.memory.InMemoryChannelBindingStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantChannelStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantMessageStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantCommitmentStore,\
  io.casehub.qhorus.persistence.memory.InMemoryCrossTenantWatchdogStore

quarkus.datasource.db-kind=h2
quarkus.datasource.username=sa
quarkus.datasource.password=
quarkus.datasource.jdbc.url=jdbc:h2:mem:webhook-test;DB_CLOSE_DELAY=-1

quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:webhook-test;DB_CLOSE_DELAY=-1
quarkus.datasource.qhorus.reactive=false

quarkus.hibernate-orm.packages=io.casehub.qhorus.runtime.config
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime,io.casehub.qhorus.webhook
quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
quarkus.flyway.qhorus.migrate-at-start=false
quarkus.hibernate-orm.qhorus.sql-load-script=import-webhook-test.sql

casehub.ledger.enabled=true
casehub.qhorus.delivery.enabled=false

quarkus.http.test-port=0
```

- [ ] **Step 3: Create import-webhook-test.sql**

`webhook-observer/src/test/resources/import-webhook-test.sql`:

```sql
CREATE TABLE IF NOT EXISTS ledger_subject_sequence (
    subject_id VARCHAR(255) NOT NULL PRIMARY KEY,
    next_sequence BIGINT NOT NULL DEFAULT 1
);
```

- [ ] **Step 4: Write failing store tests**

`WebhookRegistrationStoreTest.java`:

```java
package io.casehub.qhorus.webhook;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class WebhookRegistrationStoreTest {

    @Inject WebhookRegistrationStore store;

    @Test
    @TestTransaction
    void saveAndFindById() {
        var entity = new WebhookRegistrationEntity();
        entity.id = UUID.randomUUID();
        entity.url = "https://example.com/hook";
        entity.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        entity.createdAt = Instant.now();

        store.save(entity);
        var found = store.findById(entity.id);

        assertThat(found).isPresent();
        assertThat(found.get().url).isEqualTo("https://example.com/hook");
    }

    @Test
    @TestTransaction
    void findByChannelIdReturnsOnlyChannelSpecific() {
        UUID channelId = UUID.randomUUID();
        var entity = new WebhookRegistrationEntity();
        entity.id = UUID.randomUUID();
        entity.channelId = channelId;
        entity.url = "https://example.com/specific";
        entity.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        entity.createdAt = Instant.now();
        store.save(entity);

        var global = new WebhookRegistrationEntity();
        global.id = UUID.randomUUID();
        global.url = "https://example.com/global";
        global.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        global.createdAt = Instant.now();
        store.save(global);

        assertThat(store.findByChannelId(channelId)).hasSize(1);
        assertThat(store.findByChannelId(channelId).get(0).url).isEqualTo("https://example.com/specific");
    }

    @Test
    @TestTransaction
    void findGlobalReturnsNullChannelIdOnly() {
        var global = new WebhookRegistrationEntity();
        global.id = UUID.randomUUID();
        global.url = "https://example.com/global";
        global.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        global.createdAt = Instant.now();
        store.save(global);

        var specific = new WebhookRegistrationEntity();
        specific.id = UUID.randomUUID();
        specific.channelId = UUID.randomUUID();
        specific.url = "https://example.com/specific";
        specific.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        specific.createdAt = Instant.now();
        store.save(specific);

        assertThat(store.findGlobal()).hasSize(1);
        assertThat(store.findGlobal().get(0).url).isEqualTo("https://example.com/global");
    }

    @Test
    @TestTransaction
    void findAllIsCrossTenant() {
        var t1 = new WebhookRegistrationEntity();
        t1.id = UUID.randomUUID();
        t1.url = "https://t1.com/hook";
        t1.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        t1.createdAt = Instant.now();
        store.save(t1);

        var t2 = new WebhookRegistrationEntity();
        t2.id = UUID.randomUUID();
        t2.url = "https://t2.com/hook";
        t2.tenancyId = "other-tenant";
        t2.createdAt = Instant.now();
        store.save(t2);

        // findAll is cross-tenant — returns both
        assertThat(store.findAll()).hasSizeGreaterThanOrEqualTo(2);
    }

    @Test
    @TestTransaction
    void deleteRemovesEntity() {
        var entity = new WebhookRegistrationEntity();
        entity.id = UUID.randomUUID();
        entity.url = "https://example.com/hook";
        entity.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        entity.createdAt = Instant.now();
        store.save(entity);

        assertThat(store.delete(entity.id)).isTrue();
        assertThat(store.findById(entity.id)).isEmpty();
    }

    @Test
    @TestTransaction
    void deleteByChannelIdRemovesAllForChannel() {
        UUID channelId = UUID.randomUUID();
        var e1 = new WebhookRegistrationEntity();
        e1.id = UUID.randomUUID();
        e1.channelId = channelId;
        e1.url = "https://a.com/hook";
        e1.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        e1.createdAt = Instant.now();
        store.save(e1);

        var e2 = new WebhookRegistrationEntity();
        e2.id = UUID.randomUUID();
        e2.channelId = channelId;
        e2.url = "https://b.com/hook";
        e2.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        e2.createdAt = Instant.now();
        store.save(e2);

        store.deleteByChannelId(channelId);
        assertThat(store.findByChannelId(channelId)).isEmpty();
    }

    @Test
    @TestTransaction
    void headersSerializedAsJson() {
        var entity = new WebhookRegistrationEntity();
        entity.id = UUID.randomUUID();
        entity.url = "https://example.com/hook";
        entity.headers = Map.of("Authorization", "Bearer tok", "X-Custom", "val");
        entity.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        entity.createdAt = Instant.now();
        store.save(entity);

        var found = store.findById(entity.id).orElseThrow();
        assertThat(found.headers).containsEntry("Authorization", "Bearer tok");
        assertThat(found.headers).containsEntry("X-Custom", "val");
    }
}
```

- [ ] **Step 5: Write WebhookFlywaySchemaTest**

```java
package io.casehub.qhorus.webhook;

import static org.assertj.core.api.Assertions.assertThat;

import java.sql.Connection;
import java.sql.DriverManager;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

class WebhookFlywaySchemaTest {

    private static final String JDBC_URL =
            "jdbc:h2:mem:webhook_flyway_schema_test_" + System.nanoTime()
                    + ";MODE=PostgreSQL;DB_CLOSE_DELAY=-1";

    @BeforeAll
    static void migrate() {
        Flyway.configure()
                .dataSource(JDBC_URL, "sa", "")
                .locations("classpath:db/qhorus/migration", "classpath:db/ledger/migration")
                .load()
                .migrate();
    }

    @Test
    void v35_webhook_registration_table_exists() throws Exception {
        try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "");
             var rs = conn.getMetaData().getTables(null, null, "WEBHOOK_REGISTRATION", new String[]{"TABLE"})) {
            assertThat(rs.next())
                    .as("webhook_registration table must exist — created by V35 migration")
                    .isTrue();
        }
    }

    @Test
    void v35_has_unique_constraint_on_url_channel_tenant() throws Exception {
        try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "")) {
            var rs = conn.getMetaData().getIndexInfo(null, null, "WEBHOOK_REGISTRATION", true, false);
            boolean foundUrl = false;
            while (rs.next()) {
                String colName = rs.getString("COLUMN_NAME");
                if ("URL".equalsIgnoreCase(colName)) foundUrl = true;
            }
            rs.close();
            assertThat(foundUrl).as("UNIQUE index including URL must exist").isTrue();
        }
    }
}
```

- [ ] **Step 6: Create MapToJsonConverter**

```java
package io.casehub.qhorus.webhook;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;

import java.util.Map;

@Converter
public class MapToJsonConverter implements AttributeConverter<Map<String, String>, String> {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final TypeReference<Map<String, String>> TYPE_REF = new TypeReference<>() {};

    @Override
    public String convertToDatabaseColumn(Map<String, String> map) {
        if (map == null || map.isEmpty()) return null;
        try {
            return MAPPER.writeValueAsString(map);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize headers map", e);
        }
    }

    @Override
    public Map<String, String> convertToEntityAttribute(String json) {
        if (json == null || json.isBlank()) return Map.of();
        try {
            return MAPPER.readValue(json, TYPE_REF);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to deserialize headers map", e);
        }
    }
}
```

- [ ] **Step 7: Create WebhookRegistrationEntity**

```java
package io.casehub.qhorus.webhook;

import jakarta.persistence.Column;
import jakarta.persistence.Convert;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

@Entity
@Table(name = "webhook_registration")
public class WebhookRegistrationEntity {

    @Id
    public UUID id;

    @Column(name = "channel_id")
    public UUID channelId;

    @Column(nullable = false, length = 2048)
    public String url;

    @Column(name = "secret_ref")
    public String secretRef;

    @Convert(converter = MapToJsonConverter.class)
    public Map<String, String> headers;

    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId;

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;
}
```

- [ ] **Step 8: Create WebhookRegistrationStore**

```java
package io.casehub.qhorus.webhook;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.hibernate.orm.PersistenceUnit;

@ApplicationScoped
public class WebhookRegistrationStore {

    @Inject
    @PersistenceUnit("qhorus")
    EntityManager em;

    @Inject
    CurrentPrincipal currentPrincipal;

    public Optional<WebhookRegistrationEntity> findById(UUID id) {
        return em.createQuery(
                "FROM WebhookRegistrationEntity e WHERE e.id = :id AND e.tenancyId = :tid",
                WebhookRegistrationEntity.class)
                .setParameter("id", id)
                .setParameter("tid", currentPrincipal.tenancyId())
                .getResultStream().findFirst();
    }

    public List<WebhookRegistrationEntity> findByChannelId(UUID channelId) {
        return em.createQuery(
                "FROM WebhookRegistrationEntity e WHERE e.channelId = :cid AND e.tenancyId = :tid",
                WebhookRegistrationEntity.class)
                .setParameter("cid", channelId)
                .setParameter("tid", currentPrincipal.tenancyId())
                .getResultList();
    }

    public List<WebhookRegistrationEntity> findGlobal() {
        return em.createQuery(
                "FROM WebhookRegistrationEntity e WHERE e.channelId IS NULL AND e.tenancyId = :tid",
                WebhookRegistrationEntity.class)
                .setParameter("tid", currentPrincipal.tenancyId())
                .getResultList();
    }

    /** Cross-tenant — used only by startup reload. */
    public List<WebhookRegistrationEntity> findAll() {
        return em.createQuery(
                "FROM WebhookRegistrationEntity e",
                WebhookRegistrationEntity.class)
                .getResultList();
    }

    @Transactional
    public void save(WebhookRegistrationEntity entity) {
        em.merge(entity);
    }

    @Transactional
    public boolean delete(UUID id) {
        return em.createQuery("DELETE FROM WebhookRegistrationEntity e WHERE e.id = :id AND e.tenancyId = :tid")
                .setParameter("id", id)
                .setParameter("tid", currentPrincipal.tenancyId())
                .executeUpdate() > 0;
    }

    /** Cross-tenant — channel deletion is authoritative. */
    @Transactional
    public void deleteByChannelId(UUID channelId) {
        em.createQuery("DELETE FROM WebhookRegistrationEntity e WHERE e.channelId = :cid")
                .setParameter("cid", channelId)
                .executeUpdate();
    }
}
```

- [ ] **Step 9: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook-observer -f pom.xml
```

Expected: all store tests and Flyway schema tests pass.

- [ ] **Step 10: Commit**

```bash
git add webhook-observer/
git add runtime/src/main/resources/db/qhorus/migration/V35__webhook_registration.sql
git commit -m "feat(#345): WebhookRegistrationEntity, store, MapToJsonConverter, V35

JPA entity with qhorus named PU. Store injects CurrentPrincipal for
tenant-scoped queries. findAll() and deleteByChannelId() are cross-tenant.
MapToJsonConverter for headers field. V35 creates webhook_registration
table with partial unique index for global webhooks.

Refs #345"
```

---

### Task 5: WebhookRegistration DTO + WebhookRegistry refactor + WebhookMessageObserver credential resolution

**Files:**
- Modify: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistration.java`
- Modify: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistry.java`
- Modify: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistryResource.java`
- Modify: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookMessageObserver.java`
- Modify: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookObserverConfig.java`
- Modify: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookRegistryTest.java`
- Modify: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookMessageObserverTest.java`
- Create: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookRegistryIntegrationTest.java`

**Interfaces:**
- Consumes: `WebhookRegistrationEntity` (Task 4), `WebhookRegistrationStore` (Task 4), `ChannelClosedEvent` (Task 2), `CredentialResolver` (platform-api), `CredentialPropertyKeys.SIGNING_SECRET` (Task 1)
- Produces: Fully wired `WebhookRegistry` with JPA persistence, tenant-scoped lookup, and startup reload
- Produces: `WebhookMessageObserver` with `CredentialResolver` integration

This is the core integration task — all the pieces come together here.

- [ ] **Step 1: Update WebhookRegistration record (DTO)**

Replace the existing record with the new shape:

```java
package io.casehub.qhorus.webhook;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

public record WebhookRegistration(
        UUID id,
        UUID channelId,
        String tenancyId,
        String url,
        String secretRef,
        Map<String, String> headers,
        Instant createdAt) {

    public WebhookRegistration {
        if (url == null || url.isBlank()) {
            throw new IllegalArgumentException("Webhook URL must not be blank");
        }
        if (headers == null) {
            headers = Map.of();
        }
    }
}
```

- [ ] **Step 2: Update WebhookRegistryTest (CDI-free unit test)**

Replace the existing test to work with the new `WebhookRegistration` shape and the tenant-scoped `globalHooks` map. This is CDI-free — tests the in-memory lookup logic only:

```java
package io.casehub.qhorus.webhook;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class WebhookRegistryTest {

    private WebhookRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new WebhookRegistry();
    }

    @Test
    void registerAndLookupByChannel() {
        UUID channelId = UUID.randomUUID();
        var reg = registry.registerInMemory(channelId, "t1", "https://example.com/hook", null, Map.of());

        assertThat(reg.id()).isNotNull();
        assertThat(registry.findByChannelId(channelId)).hasSize(1);
    }

    @Test
    void globalWebhookScopedByTenant() {
        registry.registerInMemory(null, "t1", "https://example.com/global", null, Map.of());
        registry.registerInMemory(null, "t2", "https://other.com/global", null, Map.of());

        UUID anyChannel = UUID.randomUUID();

        assertThat(registry.findForChannel(anyChannel, "t1")).hasSize(1);
        assertThat(registry.findForChannel(anyChannel, "t1").iterator().next().url())
                .isEqualTo("https://example.com/global");
        assertThat(registry.findForChannel(anyChannel, "t2")).hasSize(1);
    }

    @Test
    void channelSpecificPlusGlobal() {
        UUID channelId = UUID.randomUUID();
        registry.registerInMemory(channelId, "t1", "https://example.com/specific", null, Map.of());
        registry.registerInMemory(null, "t1", "https://example.com/global", null, Map.of());

        assertThat(registry.findForChannel(channelId, "t1")).hasSize(2);
    }

    @Test
    void deregisterRemoves() {
        UUID channelId = UUID.randomUUID();
        var reg = registry.registerInMemory(channelId, "t1", "https://example.com/hook", null, Map.of());

        assertThat(registry.deregisterInMemory(reg.id())).isTrue();
        assertThat(registry.findByChannelId(channelId)).isEmpty();
    }

    @Test
    void deregisterUnknownReturnsFalse() {
        assertThat(registry.deregisterInMemory(UUID.randomUUID())).isFalse();
    }

    @Test
    void deregisterGlobalWebhook() {
        var reg = registry.registerInMemory(null, "t1", "https://example.com/global", null, Map.of());

        assertThat(registry.deregisterInMemory(reg.id())).isTrue();
        assertThat(registry.findForChannel(UUID.randomUUID(), "t1")).isEmpty();
    }

    @Test
    void listAllByTenantFilters() {
        registry.registerInMemory(UUID.randomUUID(), "t1", "https://a.com", null, Map.of());
        registry.registerInMemory(null, "t2", "https://b.com", null, Map.of());

        assertThat(registry.listAll("t1")).hasSize(1);
        assertThat(registry.listAll("t2")).hasSize(1);
    }

    @Test
    void crossTenantIsolationForGlobalHooks() {
        registry.registerInMemory(null, "tenant-a", "https://a.com/hook", null, Map.of());
        registry.registerInMemory(null, "tenant-b", "https://b.com/hook", null, Map.of());

        UUID channelId = UUID.randomUUID();
        assertThat(registry.findForChannel(channelId, "tenant-a")).hasSize(1);
        assertThat(registry.findForChannel(channelId, "tenant-a").iterator().next().url())
                .isEqualTo("https://a.com/hook");
    }

    @Test
    void removeChannelCleansUp() {
        UUID channelId = UUID.randomUUID();
        registry.registerInMemory(channelId, "t1", "https://a.com", null, Map.of());
        registry.registerInMemory(channelId, "t1", "https://b.com", null, Map.of());

        registry.removeChannel(channelId);

        assertThat(registry.findByChannelId(channelId)).isEmpty();
    }
}
```

- [ ] **Step 3: Rewrite WebhookRegistry**

Full rewrite — becomes the coordination layer between in-memory cache and JPA store:

```java
package io.casehub.qhorus.webhook;

import java.time.Instant;
import java.util.Collection;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.qhorus.api.gateway.ChannelClosedEvent;
import io.quarkus.runtime.StartupEvent;

@ApplicationScoped
public class WebhookRegistry {

    private static final Logger LOG = Logger.getLogger(WebhookRegistry.class);

    private final ConcurrentHashMap<UUID, Set<WebhookRegistration>> channelHooks = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Set<WebhookRegistration>> globalHooks = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<UUID, WebhookRegistration> byId = new ConcurrentHashMap<>();

    @Inject
    WebhookRegistrationStore store;

    WebhookRegistry() {}

    void onStart(@Observes StartupEvent ev) {
        for (WebhookRegistrationEntity e : store.findAll()) {
            registerInMemory(e.channelId, e.tenancyId, e.url, e.secretRef, e.headers == null ? Map.of() : e.headers, e.id, e.createdAt);
        }
        LOG.infof("Loaded %d webhook registration(s) from database", byId.size());
    }

    public WebhookRegistration register(UUID channelId, String tenancyId, String url, String secretRef, Map<String, String> headers) {
        var entity = new WebhookRegistrationEntity();
        entity.id = UUID.randomUUID();
        entity.channelId = channelId;
        entity.url = url;
        entity.secretRef = secretRef;
        entity.headers = headers;
        entity.tenancyId = tenancyId;
        entity.createdAt = Instant.now();
        store.save(entity);
        return registerInMemory(channelId, tenancyId, url, secretRef, headers, entity.id, entity.createdAt);
    }

    WebhookRegistration registerInMemory(UUID channelId, String tenancyId, String url, String secretRef, Map<String, String> headers) {
        return registerInMemory(channelId, tenancyId, url, secretRef, headers, UUID.randomUUID(), Instant.now());
    }

    private WebhookRegistration registerInMemory(UUID channelId, String tenancyId, String url, String secretRef, Map<String, String> headers, UUID id, Instant createdAt) {
        var reg = new WebhookRegistration(id, channelId, tenancyId, url, secretRef, headers, createdAt);
        byId.put(reg.id(), reg);
        if (channelId == null) {
            globalHooks.computeIfAbsent(tenancyId, k -> ConcurrentHashMap.newKeySet()).add(reg);
        } else {
            channelHooks.computeIfAbsent(channelId, k -> ConcurrentHashMap.newKeySet()).add(reg);
        }
        return reg;
    }

    public boolean deregister(UUID registrationId) {
        store.delete(registrationId);
        return deregisterInMemory(registrationId);
    }

    boolean deregisterInMemory(UUID registrationId) {
        WebhookRegistration reg = byId.remove(registrationId);
        if (reg == null) return false;
        if (reg.channelId() == null) {
            globalHooks.computeIfPresent(reg.tenancyId(), (k, hooks) -> {
                hooks.remove(reg);
                return hooks.isEmpty() ? null : hooks;
            });
        } else {
            channelHooks.computeIfPresent(reg.channelId(), (k, hooks) -> {
                hooks.remove(reg);
                return hooks.isEmpty() ? null : hooks;
            });
        }
        return true;
    }

    public Set<WebhookRegistration> findForChannel(UUID channelId, String tenancyId) {
        Set<WebhookRegistration> result = new HashSet<>();
        Set<WebhookRegistration> globals = globalHooks.get(tenancyId);
        if (globals != null) result.addAll(globals);
        Set<WebhookRegistration> specific = channelHooks.get(channelId);
        if (specific != null) result.addAll(specific);
        return result;
    }

    public Set<WebhookRegistration> findByChannelId(UUID channelId) {
        return channelHooks.getOrDefault(channelId, Set.of());
    }

    public Collection<WebhookRegistration> listAll(String tenancyId) {
        return byId.values().stream()
                .filter(r -> tenancyId.equals(r.tenancyId()))
                .collect(Collectors.toList());
    }

    void removeChannel(UUID channelId) {
        Set<WebhookRegistration> removed = channelHooks.remove(channelId);
        if (removed != null) {
            for (WebhookRegistration r : removed) {
                byId.remove(r.id());
            }
        }
    }

    void onChannelClosed(@Observes ChannelClosedEvent event) {
        removeChannel(event.channelId());
        store.deleteByChannelId(event.channelId());
    }
}
```

- [ ] **Step 4: Update WebhookRegistryResource**

```java
package io.casehub.qhorus.webhook;

import java.util.Collection;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.ws.rs.DELETE;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.Response;

import io.casehub.platform.api.identity.CurrentPrincipal;

@Path("/qhorus/webhooks")
public class WebhookRegistryResource {

    @Inject WebhookRegistry registry;
    @Inject CurrentPrincipal currentPrincipal;

    public record RegisterRequest(UUID channelId, String url, String secretRef, Map<String, String> headers) {}

    @POST
    public Response register(RegisterRequest request) {
        if (request.url() == null || request.url().isBlank()) {
            return Response.status(400).entity("url is required").build();
        }
        WebhookRegistration reg = registry.register(
                request.channelId(), currentPrincipal.tenancyId(),
                request.url(), request.secretRef(),
                request.headers() != null ? request.headers() : Map.of());
        return Response.status(201).entity(reg).build();
    }

    @DELETE
    @Path("/{id}")
    public Response deregister(@PathParam("id") UUID id) {
        if (registry.deregister(id)) {
            return Response.noContent().build();
        }
        return Response.status(404).build();
    }

    @GET
    public Collection<WebhookRegistration> list(@QueryParam("channelId") UUID channelId) {
        if (channelId != null) {
            return registry.findByChannelId(channelId);
        }
        return registry.listAll(currentPrincipal.tenancyId());
    }
}
```

- [ ] **Step 5: Update WebhookMessageObserver**

Inject `CredentialResolver`. Change `onMessage()` to pass `tenancyId` to `findForChannel()`. Resolve `secretRef` via `CredentialResolver` at POST time. Resolution failure skips the POST entirely:

```java
// In the onMessage method:
Set<WebhookRegistration> hooks = registry.findForChannel(event.channelId(), event.tenancyId());

// In the poster lambda / post method, before HMAC computation:
if (hook.secretRef() != null) {
    String secret;
    try {
        Map<String, String> creds = credentialResolver.resolve(hook.secretRef());
        secret = creds.get(CredentialPropertyKeys.SIGNING_SECRET);
        if (secret == null || secret.isBlank()) {
            LOG.errorf("Credential %s missing signing-secret key — skipping webhook POST to %s",
                    hook.secretRef(), hook.url());
            continue; // or return from poster
        }
    } catch (Exception e) {
        LOG.errorf("Failed to resolve credential %s — skipping webhook POST to %s: %s",
                hook.secretRef(), hook.url(), e.getMessage());
        continue;
    }
    builder.header("X-Qhorus-Signature", hmacSha256(secret, body));
}
```

Update the `WebhookPoster` interface and constructor to inject `CredentialResolver`.

- [ ] **Step 6: Update WebhookMessageObserverTest**

Update to use `secretRef` instead of `secret`, add a stub `CredentialResolver`, test resolution failure skips POST:

```java
// Key new tests:
@Test
void secretRefResolvesViaCredentialResolver() {
    UUID channelId = UUID.randomUUID();
    registry.registerInMemory(channelId, "t1", "https://example.com/hook", "my-webhook-cred", Map.of());

    observer.onMessage(new MessageReceivedEvent(
            "test-channel", channelId, "t1",
            MessageType.STATUS, "agent-1", null,
            Instant.now(), "hello", null));

    assertThat(posts).hasSize(1);
    assertThat(posts.get(0).secret()).isEqualTo("resolved-signing-secret");
}

@Test
void missingCredentialSkipsPost() {
    UUID channelId = UUID.randomUUID();
    registry.registerInMemory(channelId, "t1", "https://example.com/hook", "missing-cred", Map.of());

    observer.onMessage(new MessageReceivedEvent(
            "test-channel", channelId, "t1",
            MessageType.STATUS, "agent-1", null,
            Instant.now(), "hello", null));

    assertThat(posts).isEmpty();
}

@Test
void noSecretRefPostsWithoutSignature() {
    UUID channelId = UUID.randomUUID();
    registry.registerInMemory(channelId, "t1", "https://example.com/hook", null, Map.of());

    observer.onMessage(new MessageReceivedEvent(
            "test-channel", channelId, "t1",
            MessageType.STATUS, "agent-1", null,
            Instant.now(), "hello", null));

    assertThat(posts).hasSize(1);
    assertThat(posts.get(0).secret()).isNull();
}
```

- [ ] **Step 7: Write WebhookRegistryIntegrationTest**

`@QuarkusTest` testing the full flow — register via `WebhookRegistry`, verify store persistence, restart reload, `ChannelClosedEvent` cleanup:

```java
@QuarkusTest
class WebhookRegistryIntegrationTest {

    @Inject WebhookRegistry registry;
    @Inject WebhookRegistrationStore store;

    @Test
    void registerPersistsToStore() {
        var reg = QuarkusTransaction.requiringNew().call(() ->
                registry.register(UUID.randomUUID(), "278776f9-e1b0-46fb-9032-8bddebdcf9ce",
                        "https://integration-test.com/hook", null, Map.of()));

        var found = QuarkusTransaction.requiringNew().call(() ->
                store.findById(reg.id()));
        assertThat(found).isPresent();
    }

    @Test
    void deregisterRemovesFromStore() {
        var reg = QuarkusTransaction.requiringNew().call(() ->
                registry.register(null, "278776f9-e1b0-46fb-9032-8bddebdcf9ce",
                        "https://deregister-test.com/hook", null, Map.of()));

        QuarkusTransaction.requiringNew().run(() -> registry.deregister(reg.id()));

        var found = QuarkusTransaction.requiringNew().call(() ->
                store.findById(reg.id()));
        assertThat(found).isEmpty();
    }
}
```

- [ ] **Step 8: Run all webhook-observer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook-observer -f pom.xml
```

Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git add webhook-observer/
git commit -m "feat(#345): persistent webhook registrations — registry, DTO, observer

WebhookRegistry coordinates in-memory cache with JPA store. Startup
reload from store.findAll(). ChannelClosedEvent triggers cleanup.
secretRef replaces secret — resolved via CredentialResolver at POST
time. Resolution failure skips POST entirely (fail-closed). Global
hooks keyed by tenancyId for cross-tenant isolation.

Refs #345"
```

---

### Task 6: Full build verification + CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — add webhook-observer conventions

- [ ] **Step 1: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml
```

Must pass with zero errors. Fix any compilation issues in other modules (e.g. `examples/` modules that depend on changed API types).

- [ ] **Step 2: Update CLAUDE.md**

Add to the project structure section the webhook-observer JPA details, the V35 migration, and the `ChannelClosedEvent`. Add to the testing conventions:
- `WebhookRegistry` is `@ApplicationScoped` — in-memory state does not roll back with `@TestTransaction`. Use unique URLs per test.
- `ChannelClosedEvent` mirrors `ChannelInitialisedEvent` — fired after backends close, not before.
- V35 is now the latest domain migration; next is V36.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#345): update CLAUDE.md with webhook persistence conventions

Refs #345"
```

---

## Task Dependency Graph

```
Task 1 (SIGNING_SECRET) ──┐
                           ├── Task 5 (integration — registry + observer + resource)
Task 2 (ChannelClosedEvent)┤
                           │
Task 3 (V35 migration) ───┤
                           │
Task 4 (entity + store) ──┘
                           
Task 5 ──── Task 6 (full build + CLAUDE.md)
```

Tasks 1, 2, 3 are independent of each other. Task 4 depends on Task 3 (migration must exist for Flyway schema test). Task 5 integrates everything. Task 6 is the final verification.
