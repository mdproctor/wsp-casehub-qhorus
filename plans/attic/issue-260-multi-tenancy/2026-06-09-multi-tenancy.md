# Multi-Tenancy Implementation Plan — qhorus #260

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `tenancy_id` scoping to Channel, Message, Commitment, and Watchdog so every agent in every tenant sees only its own data.

**Architecture:** JPA stores inject `CurrentPrincipal` and filter all reads by `tenancyId`; writes are stamped by the creation gate (ChannelService, MessageService). `MessageService.dispatch()` resolves the effective tenancyId (from `currentPrincipal` for HTTP callers, from `dispatch.tenancyId()` for system callers like the watchdog). Cross-tenant stores (no filter) are produced via `@CrossTenant` CDI qualifier and used by ChannelGateway startup and WatchdogEvaluationService.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache ORM (blocking + reactive), Flyway, H2 (tests), CDI qualifiers, `casehub-platform-api` `CurrentPrincipal`

**Test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
**Full build:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

### Task 1: Flyway migrations V18–V21 + FlywayMigrationSchemaTest

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V18__add_channel_tenancy.sql`
- Create: `runtime/src/main/resources/db/qhorus/migration/V19__add_message_tenancy.sql`
- Create: `runtime/src/main/resources/db/qhorus/migration/V20__add_commitment_tenancy.sql`
- Create: `runtime/src/main/resources/db/qhorus/migration/V21__add_watchdog_tenancy.sql`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java`

- [ ] **Step 1: Write failing schema tests**

Add four new test methods to `FlywayMigrationSchemaTest` (inside the existing class, after the last `@Test`):

```java
@Test
void channelTenancyIdColumnExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "");
         var rs = conn.getMetaData().getColumns(null, null, "CHANNEL", "TENANCY_ID")) {
        assertTrue(rs.next(), "channel.tenancy_id must exist — added by V18");
        rs.close();
    }
}

@Test
void channelNameTenancyUniqueConstraintExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "");
         var stmt = conn.prepareStatement(
                 "SELECT COUNT(*) FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS " +
                 "WHERE TABLE_NAME = 'CHANNEL' AND CONSTRAINT_NAME = 'UQ_CHANNEL_NAME_TENANCY'");
         var rs = stmt.executeQuery()) {
        rs.next();
        assertTrue(rs.getInt(1) >= 1, "uq_channel_name_tenancy constraint must exist on channel");
    }
}

@Test
void messageTenancyIdColumnExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "");
         var rs = conn.getMetaData().getColumns(null, null, "QHORUS_MESSAGE", "TENANCY_ID")) {
        assertTrue(rs.next(), "qhorus_message.tenancy_id must exist — added by V19");
        rs.close();
    }
}

@Test
void commitmentTenancyIdColumnExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "");
         var rs = conn.getMetaData().getColumns(null, null, "COMMITMENT", "TENANCY_ID")) {
        assertTrue(rs.next(), "commitment.tenancy_id must exist — added by V20");
        rs.close();
    }
}

@Test
void watchdogTenancyIdColumnExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "");
         var rs = conn.getMetaData().getColumns(null, null, "WATCHDOG", "TENANCY_ID")) {
        assertTrue(rs.next(), "watchdog.tenancy_id must exist — added by V21");
        rs.close();
    }
}
```

- [ ] **Step 2: Run to confirm failures**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FlywayMigrationSchemaTest
```
Expected: 5 failures (columns don't exist yet).

- [ ] **Step 3: Create V18 migration**

`runtime/src/main/resources/db/qhorus/migration/V18__add_channel_tenancy.sql`:
```sql
ALTER TABLE channel
    ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';

ALTER TABLE channel DROP CONSTRAINT uq_channel_name;
ALTER TABLE channel ADD CONSTRAINT uq_channel_name_tenancy UNIQUE (tenancy_id, name);
```

- [ ] **Step 4: Create V19 migration**

`runtime/src/main/resources/db/qhorus/migration/V19__add_message_tenancy.sql`:
```sql
ALTER TABLE qhorus_message
    ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
```

- [ ] **Step 5: Create V20 migration**

`runtime/src/main/resources/db/qhorus/migration/V20__add_commitment_tenancy.sql`:
```sql
ALTER TABLE commitment
    ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
```

- [ ] **Step 6: Create V21 migration**

`runtime/src/main/resources/db/qhorus/migration/V21__add_watchdog_tenancy.sql`:
```sql
ALTER TABLE watchdog
    ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
```

- [ ] **Step 7: Run schema tests — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FlywayMigrationSchemaTest
```
Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/resources/db/qhorus/migration/V18__add_channel_tenancy.sql \
  runtime/src/main/resources/db/qhorus/migration/V19__add_message_tenancy.sql \
  runtime/src/main/resources/db/qhorus/migration/V20__add_commitment_tenancy.sql \
  runtime/src/main/resources/db/qhorus/migration/V21__add_watchdog_tenancy.sql \
  runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): Flyway V18-V21 — tenancy_id columns on channel, message, commitment, watchdog"
```

---

### Task 2: CDI qualifiers + QhorusSystemCurrentPrincipal

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/qualifier/CrossTenant.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/qualifier/QhorusSystem.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/identity/QhorusSystemCurrentPrincipal.java`

- [ ] **Step 1: Create @CrossTenant qualifier**

`api/src/main/java/io/casehub/qhorus/api/qualifier/CrossTenant.java`:
```java
package io.casehub.qhorus.api.qualifier;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import jakarta.inject.Qualifier;

/**
 * Marks injection points that require cross-tenant data access.
 * Produced by {@code CrossTenantProducer} — only injectable by classes
 * that have been explicitly granted cross-tenant access.
 *
 * <p>Convention-based marker — enforced by code review. See protocol PP-20260520-e6a5f0.
 */
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
public @interface CrossTenant {}
```

- [ ] **Step 2: Create @QhorusSystem qualifier**

`runtime/src/main/java/io/casehub/qhorus/runtime/qualifier/QhorusSystem.java`:
```java
package io.casehub.qhorus.runtime.qualifier;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import jakarta.inject.Qualifier;

/**
 * Selects the qhorus-internal system-actor CurrentPrincipal implementation.
 * Used by CrossTenantProducer to inject QhorusSystemCurrentPrincipal specifically,
 * without displacing the @DefaultBean mock.
 */
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
public @interface QhorusSystem {}
```

- [ ] **Step 3: Create QhorusSystemCurrentPrincipal**

`runtime/src/main/java/io/casehub/qhorus/runtime/identity/QhorusSystemCurrentPrincipal.java`:
```java
package io.casehub.qhorus.runtime.identity;

import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.qhorus.runtime.qualifier.QhorusSystem;

/**
 * Engine-internal system-actor CurrentPrincipal. Always isCrossTenantAdmin().
 *
 * <p>Not @DefaultBean — never replaces MockCurrentPrincipal. Accessed only
 * via @QhorusSystem qualifier from CrossTenantProducer.
 *
 * <p>Interim: delete when casehub-platform ships a platform-level system-actor
 * principal with isCrossTenantAdmin()=true. Update CrossTenantProducer to inject
 * the platform implementation instead.
 */
@ApplicationScoped
@QhorusSystem
public class QhorusSystemCurrentPrincipal implements CurrentPrincipal {

    @Override
    public String actorId() {
        return "system:qhorus";
    }

    @Override
    public Set<String> groups() {
        return Set.of();
    }

    @Override
    public String tenancyId() {
        return TenancyConstants.DEFAULT_TENANT_ID;
    }

    @Override
    public boolean isCrossTenantAdmin() {
        return true;
    }
}
```

- [ ] **Step 4: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  api/src/main/java/io/casehub/qhorus/api/qualifier/CrossTenant.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/qualifier/QhorusSystem.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/identity/QhorusSystemCurrentPrincipal.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): @CrossTenant/@QhorusSystem qualifiers + QhorusSystemCurrentPrincipal"
```

---

### Task 3: Cross-tenant store interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantChannelStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantMessageStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantCommitmentStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantWatchdogStore.java`

- [ ] **Step 1: Create CrossTenantChannelStore**

`api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantChannelStore.java`:
```java
package io.casehub.qhorus.api.spi;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.casehub.qhorus.runtime.channel.Channel;

/**
 * Cross-tenant channel access — no tenancy filter applied.
 * Produced via @CrossTenant CDI qualifier by CrossTenantProducer.
 * Inject only in system-scoped beans (ChannelGateway startup, WatchdogEvaluationService).
 */
public interface CrossTenantChannelStore {
    List<Channel> listAll();
    Optional<Channel> findById(UUID id);
    Optional<Channel> findByNameAndTenancy(String name, String tenancyId);
}
```

- [ ] **Step 2: Create CrossTenantMessageStore**

`api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantMessageStore.java`:
```java
package io.casehub.qhorus.api.spi;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.query.MessageQuery;

/**
 * Cross-tenant message access — no tenancy filter applied.
 * Produced via @CrossTenant CDI qualifier by CrossTenantProducer.
 */
public interface CrossTenantMessageStore {
    List<Message> scan(MessageQuery query);
    int countByChannel(UUID channelId);
    List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType);
    Optional<Message> findLastMessage(UUID channelId);
}
```

- [ ] **Step 3: Create CrossTenantCommitmentStore**

`api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantCommitmentStore.java`:
```java
package io.casehub.qhorus.api.spi;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import io.casehub.qhorus.runtime.message.Commitment;

/**
 * Cross-tenant commitment access — no tenancy filter applied.
 * Produced via @CrossTenant CDI qualifier by CrossTenantProducer.
 */
public interface CrossTenantCommitmentStore {
    List<Commitment> findAllOpen();
    List<Commitment> findOpenByChannel(UUID channelId);
    void expireOverdue(Instant cutoff);
}
```

- [ ] **Step 4: Create CrossTenantWatchdogStore**

`api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantWatchdogStore.java`:
```java
package io.casehub.qhorus.api.spi;

import java.util.List;

import io.casehub.qhorus.runtime.watchdog.Watchdog;

/**
 * Cross-tenant watchdog access — no tenancy filter applied.
 * Produced via @CrossTenant CDI qualifier by CrossTenantProducer.
 * Used only by WatchdogEvaluationService (scheduler thread, no request context).
 */
public interface CrossTenantWatchdogStore {
    List<Watchdog> listAll();
}
```

- [ ] **Step 5: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantChannelStore.java \
  api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantMessageStore.java \
  api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantCommitmentStore.java \
  api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantWatchdogStore.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): cross-tenant store interfaces in api/spi/"
```

---

### Task 4: CrossTenantProducer (TDD)

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/identity/CrossTenantProducer.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/identity/CrossTenantProducerTest.java`

- [ ] **Step 1: Write failing test**

`runtime/src/test/java/io/casehub/qhorus/identity/CrossTenantProducerTest.java`:
```java
package io.casehub.qhorus.identity;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.api.spi.CrossTenantChannelStore;
import io.casehub.qhorus.api.spi.CrossTenantCommitmentStore;
import io.casehub.qhorus.api.spi.CrossTenantMessageStore;
import io.casehub.qhorus.api.spi.CrossTenantWatchdogStore;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class CrossTenantProducerTest {

    @Inject @CrossTenant CrossTenantChannelStore channelStore;
    @Inject @CrossTenant CrossTenantMessageStore messageStore;
    @Inject @CrossTenant CrossTenantCommitmentStore commitmentStore;
    @Inject @CrossTenant CrossTenantWatchdogStore watchdogStore;

    @Test
    void crossTenantChannelStore_isProduced() {
        assertThat(channelStore).isNotNull();
    }

    @Test
    void crossTenantMessageStore_isProduced() {
        assertThat(messageStore).isNotNull();
    }

    @Test
    void crossTenantCommitmentStore_isProduced() {
        assertThat(commitmentStore).isNotNull();
    }

    @Test
    void crossTenantWatchdogStore_isProduced() {
        assertThat(watchdogStore).isNotNull();
    }
}
```

- [ ] **Step 2: Run to confirm failures**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CrossTenantProducerTest
```
Expected: CDI `UnsatisfiedResolutionException` — no `@CrossTenant` beans produced yet.

- [ ] **Step 3: Create JPA cross-tenant store implementations**

`runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantChannelStore.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.qhorus.api.spi.CrossTenantChannelStore;
import io.casehub.qhorus.runtime.channel.Channel;

@ApplicationScoped
public class JpaCrossTenantChannelStore implements CrossTenantChannelStore {

    @Override
    public List<Channel> listAll() {
        return Channel.listAll();
    }

    @Override
    public Optional<Channel> findById(UUID id) {
        return Channel.findByIdOptional(id);
    }

    @Override
    public Optional<Channel> findByNameAndTenancy(String name, String tenancyId) {
        return Channel.find("name = ?1 AND tenancyId = ?2", name, tenancyId).firstResultOptional();
    }
}
```

`runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantMessageStore.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.CrossTenantMessageStore;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.query.MessageQuery;

@ApplicationScoped
public class JpaCrossTenantMessageStore implements CrossTenantMessageStore {

    @Override
    public List<Message> scan(MessageQuery q) {
        MessageQueryJpql mq = MessageQueryJpql.from(q);
        String jpql = "FROM Message WHERE " + mq.where()
                + (q.descending() ? " ORDER BY id DESC" : " ORDER BY id ASC");
        if (q.limit() != null) {
            return Message.find(jpql, mq.params()).page(0, q.limit()).list();
        }
        return Message.list(jpql, mq.params());
    }

    @Override
    public int countByChannel(UUID channelId) {
        return (int) Message.count("channelId", channelId);
    }

    @Override
    public List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
        @SuppressWarnings("unchecked")
        List<String> result = Message.getEntityManager()
                .createQuery("SELECT DISTINCT m.sender FROM Message m "
                        + "WHERE m.channelId = ?1 AND m.messageType != ?2 ORDER BY m.sender")
                .setParameter(1, channelId)
                .setParameter(2, excludedType)
                .getResultList();
        return result;
    }

    @Override
    public Optional<Message> findLastMessage(UUID channelId) {
        return Message.<Message>find("channelId = ?1 ORDER BY id DESC", channelId)
                .page(0, 1)
                .firstResultOptional();
    }
}
```

`runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantCommitmentStore.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.qhorus.api.spi.CrossTenantCommitmentStore;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.CommitmentState;

@ApplicationScoped
public class JpaCrossTenantCommitmentStore implements CrossTenantCommitmentStore {

    @Inject
    CommitmentPanacheRepo repo;

    @Override
    public List<Commitment> findAllOpen() {
        return repo.list("state IN ?1 ORDER BY expiresAt ASC NULLS LAST",
                List.of(CommitmentState.OPEN, CommitmentState.ACKNOWLEDGED));
    }

    @Override
    public List<Commitment> findOpenByChannel(UUID channelId) {
        return repo.list("channelId = ?1 AND state NOT IN ?2", channelId, terminalStates());
    }

    @Override
    @Transactional
    public void expireOverdue(Instant cutoff) {
        repo.update("state = ?1 WHERE expiresAt < ?2 AND state NOT IN ?3",
                CommitmentState.EXPIRED, cutoff, terminalStates());
    }

    private List<CommitmentState> terminalStates() {
        return List.of(CommitmentState.FULFILLED, CommitmentState.DECLINED,
                CommitmentState.FAILED, CommitmentState.DELEGATED, CommitmentState.EXPIRED);
    }
}
```

`runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantWatchdogStore.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.qhorus.api.spi.CrossTenantWatchdogStore;
import io.casehub.qhorus.runtime.watchdog.Watchdog;

@ApplicationScoped
public class JpaCrossTenantWatchdogStore implements CrossTenantWatchdogStore {

    @Override
    public List<Watchdog> listAll() {
        return Watchdog.listAll();
    }
}
```

- [ ] **Step 4: Create CrossTenantProducer**

`runtime/src/main/java/io/casehub/qhorus/runtime/identity/CrossTenantProducer.java`:
```java
package io.casehub.qhorus.runtime.identity;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.api.spi.CrossTenantChannelStore;
import io.casehub.qhorus.api.spi.CrossTenantCommitmentStore;
import io.casehub.qhorus.api.spi.CrossTenantMessageStore;
import io.casehub.qhorus.api.spi.CrossTenantWatchdogStore;
import io.casehub.qhorus.runtime.qualifier.QhorusSystem;
import io.casehub.qhorus.runtime.store.jpa.JpaCrossTenantChannelStore;
import io.casehub.qhorus.runtime.store.jpa.JpaCrossTenantCommitmentStore;
import io.casehub.qhorus.runtime.store.jpa.JpaCrossTenantMessageStore;
import io.casehub.qhorus.runtime.store.jpa.JpaCrossTenantWatchdogStore;

/**
 * Produces @CrossTenant-qualified cross-tenant store beans.
 *
 * <p>The @QhorusSystem SystemCurrentPrincipal check is a contract assertion:
 * if isCrossTenantAdmin() ever returns false, this producer fails at startup
 * rather than silently granting cross-tenant access.
 */
@ApplicationScoped
public class CrossTenantProducer {

    @Inject @QhorusSystem QhorusSystemCurrentPrincipal systemPrincipal;
    @Inject JpaCrossTenantChannelStore channelStore;
    @Inject JpaCrossTenantMessageStore messageStore;
    @Inject JpaCrossTenantCommitmentStore commitmentStore;
    @Inject JpaCrossTenantWatchdogStore watchdogStore;

    @Produces @CrossTenant @ApplicationScoped
    public CrossTenantChannelStore produceChannelStore() {
        assertCrossTenantAdmin();
        return channelStore;
    }

    @Produces @CrossTenant @ApplicationScoped
    public CrossTenantMessageStore produceMessageStore() {
        assertCrossTenantAdmin();
        return messageStore;
    }

    @Produces @CrossTenant @ApplicationScoped
    public CrossTenantCommitmentStore produceCommitmentStore() {
        assertCrossTenantAdmin();
        return commitmentStore;
    }

    @Produces @CrossTenant @ApplicationScoped
    public CrossTenantWatchdogStore produceWatchdogStore() {
        assertCrossTenantAdmin();
        return watchdogStore;
    }

    private void assertCrossTenantAdmin() {
        if (!systemPrincipal.isCrossTenantAdmin()) {
            throw new IllegalStateException(
                    "QhorusSystemCurrentPrincipal.isCrossTenantAdmin() must return true — qhorus#260");
        }
    }
}
```

- [ ] **Step 5: Run test — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CrossTenantProducerTest
```
Expected: 4 tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/identity/CrossTenantProducer.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantChannelStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantMessageStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantCommitmentStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantWatchdogStore.java \
  runtime/src/test/java/io/casehub/qhorus/identity/CrossTenantProducerTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): CrossTenantProducer + JPA cross-tenant stores"
```

---

### Task 5: Entity tenancyId fields

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/Message.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/Commitment.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/Watchdog.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java`

- [ ] **Step 1: Add tenancyId to Channel**

In `Channel.java`, add after the `autoCreated` field (before `createdAt`):
```java
@Column(name = "tenancy_id", nullable = false, updatable = false)
public String tenancyId;
```
Also add import: `import io.casehub.platform.api.identity.TenancyConstants;` — needed later for testing.

- [ ] **Step 2: Add tenancyId to Message**

`Message.java` is in `runtime/src/main/java/io/casehub/qhorus/runtime/message/Message.java`.

Add after `actorType` field:
```java
@Column(name = "tenancy_id", nullable = false, updatable = false)
public String tenancyId;
```

- [ ] **Step 3: Add tenancyId to Commitment**

`Commitment.java` is in `runtime/src/main/java/io/casehub/qhorus/runtime/message/Commitment.java`.

Add after the existing fields (before `@PrePersist` if present):
```java
@Column(name = "tenancy_id", nullable = false, updatable = false)
public String tenancyId;
```

- [ ] **Step 4: Add tenancyId to Watchdog**

In `Watchdog.java`, add after `createdBy`:
```java
@Column(name = "tenancy_id", nullable = false, updatable = false)
public String tenancyId;
```

- [ ] **Step 5: Add tenancyId to MessageLedgerEntry**

In `MessageLedgerEntry.java`, add after `sourceEntity` (last field):
```java
/** Tenant this message belongs to. Populated from MessageDispatch.tenancyId() at write time. */
@Column(name = "tenancy_id")
public String tenancyId;
```

Note: `nullable = true` here — ledger entries from before #260 have no tenancyId. The V2000 migration pre-dates this; a separate migration to add the column to `message_ledger_entry` is tracked in casehubio/ledger#127.

- [ ] **Step 6: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 7: Run full test suite — confirm existing tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all existing tests pass (schema uses `drop-and-create` in tests; new columns get created).

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/Message.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/Commitment.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/Watchdog.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): tenancyId fields on Channel, Message, Commitment, Watchdog, MessageLedgerEntry"
```

---

### Task 6: InMemory cross-tenant stores + null-guard in InMemory put()

**Files:**
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantChannelStore.java`
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantMessageStore.java`
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantCommitmentStore.java`
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantWatchdogStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryCommitmentStore.java`

- [ ] **Step 1: Create InMemoryCrossTenantChannelStore**

`testing/src/main/java/io/casehub/qhorus/testing/InMemoryCrossTenantChannelStore.java`:
```java
package io.casehub.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.api.spi.CrossTenantChannelStore;
import io.casehub.qhorus.runtime.channel.Channel;

/**
 * Test alternative: returns all channels regardless of tenant.
 * Displaces CrossTenantProducer-produced bean in @QuarkusTest contexts.
 */
@Alternative
@Priority(1)
@ApplicationScoped
@CrossTenant
public class InMemoryCrossTenantChannelStore implements CrossTenantChannelStore {

    private final InMemoryChannelStore delegate;

    public InMemoryCrossTenantChannelStore(InMemoryChannelStore delegate) {
        this.delegate = delegate;
    }

    // CDI requires a no-arg constructor
    InMemoryCrossTenantChannelStore() {
        this.delegate = null;
    }

    @Override
    public List<Channel> listAll() {
        return delegate != null ? delegate.scanAll() : List.of();
    }

    @Override
    public Optional<Channel> findById(UUID id) {
        return delegate != null ? delegate.findCrossTenant(id) : Optional.empty();
    }

    @Override
    public Optional<Channel> findByNameAndTenancy(String name, String tenancyId) {
        if (delegate == null) return Optional.empty();
        return delegate.scanAll().stream()
                .filter(ch -> name.equals(ch.name) && tenancyId.equals(ch.tenancyId))
                .findFirst();
    }
}
```

**Note:** `InMemoryChannelStore` needs two new package-private methods: `scanAll()` returning all entries (no filter), and `findCrossTenant(UUID id)`. Add them in the next sub-step.

- [ ] **Step 2: Add scanAll() and findCrossTenant() to InMemoryChannelStore**

In `InMemoryChannelStore.java`, add:
```java
// Used by InMemoryCrossTenantChannelStore — bypasses any future tenant filter
List<Channel> scanAll() {
    return List.copyOf(store.values());
}

Optional<Channel> findCrossTenant(UUID id) {
    return Optional.ofNullable(store.get(id));
}
```

Also add a null-guard in `put()` to avoid GE-20260601-a35fb3 (null tenancyId silent mismatch). After `store.put(channel.id, channel)`:
```java
if (channel.tenancyId == null) {
    channel.tenancyId = io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;
}
```

- [ ] **Step 3: Create remaining three InMemory cross-tenant stores**

`InMemoryCrossTenantMessageStore.java` — delegates to `InMemoryMessageStore`:
```java
package io.casehub.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.api.spi.CrossTenantMessageStore;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.query.MessageQuery;

@Alternative @Priority(1) @ApplicationScoped @CrossTenant
public class InMemoryCrossTenantMessageStore implements CrossTenantMessageStore {

    private final InMemoryMessageStore delegate;

    public InMemoryCrossTenantMessageStore(InMemoryMessageStore delegate) {
        this.delegate = delegate;
    }

    InMemoryCrossTenantMessageStore() { this.delegate = null; }

    @Override public List<Message> scan(MessageQuery q) {
        return delegate != null ? delegate.scan(q) : List.of();
    }
    @Override public int countByChannel(UUID channelId) {
        return delegate != null ? delegate.countByChannel(channelId) : 0;
    }
    @Override public List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
        return delegate != null ? delegate.distinctSendersByChannel(channelId, excludedType) : List.of();
    }
    @Override public Optional<Message> findLastMessage(UUID channelId) {
        return delegate != null ? delegate.findLastMessage(channelId) : Optional.empty();
    }
}
```

`InMemoryCrossTenantCommitmentStore.java`:
```java
package io.casehub.qhorus.testing;

import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.api.spi.CrossTenantCommitmentStore;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.CommitmentState;

@Alternative @Priority(1) @ApplicationScoped @CrossTenant
public class InMemoryCrossTenantCommitmentStore implements CrossTenantCommitmentStore {

    private final InMemoryCommitmentStore delegate;

    public InMemoryCrossTenantCommitmentStore(InMemoryCommitmentStore delegate) {
        this.delegate = delegate;
    }

    InMemoryCrossTenantCommitmentStore() { this.delegate = null; }

    @Override public List<Commitment> findAllOpen() {
        return delegate != null ? delegate.findAllOpen() : List.of();
    }
    @Override public List<Commitment> findOpenByChannel(UUID channelId) {
        return delegate != null ? delegate.findOpenByObligor(null, channelId) : List.of();
    }
    @Override public void expireOverdue(Instant cutoff) {
        if (delegate != null) delegate.expireOverdue(cutoff);
    }
}
```

`InMemoryCrossTenantWatchdogStore.java`:
```java
package io.casehub.qhorus.testing;

import java.util.List;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.api.spi.CrossTenantWatchdogStore;
import io.casehub.qhorus.runtime.watchdog.Watchdog;

@Alternative @Priority(1) @ApplicationScoped @CrossTenant
public class InMemoryCrossTenantWatchdogStore implements CrossTenantWatchdogStore {

    private final InMemoryWatchdogStore delegate;

    public InMemoryCrossTenantWatchdogStore(InMemoryWatchdogStore delegate) {
        this.delegate = delegate;
    }

    InMemoryCrossTenantWatchdogStore() { this.delegate = null; }

    @Override public List<Watchdog> listAll() {
        return delegate != null ? delegate.scanAll() : List.of();
    }
}
```

Also add `scanAll()` to `InMemoryWatchdogStore` in the testing module:
```java
List<Watchdog> scanAll() {
    return List.copyOf(store.values());
}
```

- [ ] **Step 4: Add expireOverdue to InMemoryCommitmentStore (if not already there)**

Search `InMemoryCommitmentStore` for `expireOverdue`. If absent, add:
```java
public void expireOverdue(Instant cutoff) {
    store.values().stream()
        .filter(c -> c.expiresAt != null && c.expiresAt.isBefore(cutoff) && !c.state.isTerminal())
        .forEach(c -> c.state = CommitmentState.EXPIRED);
}
```

- [ ] **Step 5: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl testing -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add testing/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): InMemory cross-tenant stores + null tenancyId guard in put()"
```

---

### Task 7: JpaChannelStore tenant filtering (TDD: tenant isolation test — channels)

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/store/TenantIsolationTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java`

- [ ] **Step 1: Write failing test**

`runtime/src/test/java/io/casehub/qhorus/store/TenantIsolationTest.java`:
```java
package io.casehub.qhorus.store;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.store.ChannelStore;
import io.casehub.qhorus.runtime.store.query.ChannelQuery;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import org.mockito.Mockito;

/**
 * Verifies tenant isolation: data created in tenant A must not appear in tenant B's queries.
 * Uses @InjectMock CurrentPrincipal to switch tenant context between operations.
 */
@QuarkusTest
class TenantIsolationTest {

    private static final String TENANT_A = TenancyConstants.DEFAULT_TENANT_ID;
    private static final String TENANT_B = "tenant-b-" + UUID.randomUUID();

    @Inject ChannelStore channelStore;
    @InjectMock CurrentPrincipal currentPrincipal;

    @Test
    @Transactional
    void channel_createdInTenantA_notVisibleToTenantB() {
        // Arrange: create channel as tenant A
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_A);
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = "isolation-test-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;
        ch.tenancyId = TENANT_A;
        channelStore.put(ch);

        // Act: query as tenant B
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_B);
        Optional<Channel> found = channelStore.find(ch.id);
        var list = channelStore.scan(ChannelQuery.builder().build());

        // Assert: not visible to tenant B
        assertThat(found).isEmpty();
        assertThat(list).noneMatch(c -> c.id.equals(ch.id));
    }

    @Test
    @Transactional
    void channel_findByName_scopedToTenant() {
        String name = "shared-name-" + UUID.randomUUID();

        // Create same-named channel in tenant A
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_A);
        Channel chA = new Channel();
        chA.id = UUID.randomUUID();
        chA.name = name;
        chA.semantic = ChannelSemantic.APPEND;
        chA.tenancyId = TENANT_A;
        channelStore.put(chA);

        // Create same-named channel in tenant B
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_B);
        Channel chB = new Channel();
        chB.id = UUID.randomUUID();
        chB.name = name;
        chB.semantic = ChannelSemantic.APPEND;
        chB.tenancyId = TENANT_B;
        channelStore.put(chB);

        // Each tenant finds only their own channel
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_A);
        Optional<Channel> foundA = channelStore.findByName(name);
        assertThat(foundA).isPresent().get().extracting(c -> c.id).isEqualTo(chA.id);

        Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_B);
        Optional<Channel> foundB = channelStore.findByName(name);
        assertThat(foundB).isPresent().get().extracting(c -> c.id).isEqualTo(chB.id);
    }
}
```

- [ ] **Step 2: Run to confirm failures**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenantIsolationTest
```
Expected: failures — channel from tenant A visible to tenant B (no filter yet).

- [ ] **Step 3: Modify JpaChannelStore to inject CurrentPrincipal and filter reads**

Full replacement of `JpaChannelStore.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.store.ChannelStore;
import io.casehub.qhorus.runtime.store.query.ChannelQuery;

@ApplicationScoped
public class JpaChannelStore implements ChannelStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    @Transactional
    public Channel put(Channel channel) {
        channel.persistAndFlush();
        return channel;
    }

    @Override
    public Optional<Channel> find(UUID id) {
        return Channel.find("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId())
                .firstResultOptional();
    }

    @Override
    public Optional<Channel> findByName(String name) {
        return Channel.find("name = ?1 AND tenancyId = ?2", name, currentPrincipal.tenancyId())
                .firstResultOptional();
    }

    @Override
    public List<Channel> scan(ChannelQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Channel WHERE tenancyId = ?1");
        List<Object> params = new ArrayList<>();
        params.add(currentPrincipal.tenancyId());
        int idx = 2;

        if (q.paused() != null) {
            jpql.append(" AND paused = ?").append(idx++);
            params.add(q.paused());
        }
        if (q.semantic() != null) {
            jpql.append(" AND semantic = ?").append(idx++);
            params.add(q.semantic());
        }
        if (q.namePattern() != null) {
            jpql.append(" AND name LIKE ?").append(idx++);
            params.add(q.namePattern().replace("*", "%"));
        }
        if (q.namePrefix() != null) {
            jpql.append(" AND name LIKE ?").append(idx++).append(" ESCAPE '!'");
            params.add(escapeLikePrefix(q.namePrefix()) + "%");
        }

        return Channel.list(jpql.toString(), params.toArray());
    }

    @Override
    @Transactional
    public void delete(UUID id) {
        Channel.delete("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId());
    }

    @Override
    @Transactional
    public void updateLastActivity(UUID channelId) {
        // UUID is globally unique — no tenant filter needed here; channel was already
        // validated as belonging to this tenant before dispatch reached this point.
        Channel.update("lastActivityAt = ?1 WHERE id = ?2", Instant.now(), channelId);
    }

    @Override
    public List<Channel> findByIds(Collection<UUID> ids) {
        if (ids == null || ids.isEmpty()) return List.of();
        return Channel.list("id IN ?1 AND tenancyId = ?2",
                new ArrayList<>(ids), currentPrincipal.tenancyId());
    }

    private static String escapeLikePrefix(String prefix) {
        return prefix.replace("!", "!!").replace("%", "!%").replace("_", "!_");
    }
}
```

- [ ] **Step 4: Run channel isolation tests — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenantIsolationTest#channel*
```
Expected: both channel tests pass.

- [ ] **Step 5: Run full test suite — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java \
  runtime/src/test/java/io/casehub/qhorus/store/TenantIsolationTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): JpaChannelStore tenant filtering + TenantIsolationTest"
```

---

### Task 8: JpaMessageStore + MessageQueryJpql tenant filtering (TDD)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageQueryJpql.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/store/TenantIsolationTest.java`

- [ ] **Step 1: Add message isolation test to TenantIsolationTest**

Add to `TenantIsolationTest`:
```java
@Inject MessageStore messageStore;

@Test
@Transactional
void message_createdInTenantA_notVisibleToTenantB() {
    Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_A);
    // Create a channel first (required FK)
    Channel ch = new Channel();
    ch.id = UUID.randomUUID();
    ch.name = "msg-isolation-" + UUID.randomUUID();
    ch.semantic = ChannelSemantic.APPEND;
    ch.tenancyId = TENANT_A;
    channelStore.put(ch);

    Message m = new Message();
    m.channelId = ch.id;
    m.sender = "agent-a";
    m.messageType = io.casehub.qhorus.api.message.MessageType.QUERY;
    m.tenancyId = TENANT_A;
    messageStore.put(m);

    // Tenant B cannot see tenant A's message
    Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_B);
    var list = messageStore.scan(
        io.casehub.qhorus.runtime.store.query.MessageQuery.builder().channelId(ch.id).build());
    assertThat(list).isEmpty();

    int count = messageStore.countByChannel(ch.id);
    assertThat(count).isZero();
}
```

Add required imports to the test:
```java
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.MessageStore;
```

- [ ] **Step 2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenantIsolationTest#message*
```
Expected: tenant B sees tenant A's message (no filter yet).

- [ ] **Step 3: Add tenant-aware factory method to MessageQueryJpql**

In `MessageQueryJpql.java`, add a new factory method alongside the existing `from(MessageQuery q)`:
```java
static MessageQueryJpql from(MessageQuery q, String tenancyId) {
    StringBuilder where = new StringBuilder("tenancyId = ?1");
    List<Object> params = new ArrayList<>();
    params.add(tenancyId);
    int idx = 2;

    if (q.channelId() != null) {
        where.append(" AND channelId = ?").append(idx++);
        params.add(q.channelId());
    }
    if (q.afterId() != null) {
        where.append(" AND id > ?").append(idx++);
        params.add(q.afterId());
    }
    if (q.sender() != null) {
        where.append(" AND sender = ?").append(idx++);
        params.add(q.sender());
    }
    if (q.target() != null) {
        where.append(" AND target = ?").append(idx++);
        params.add(q.target());
    }
    if (q.inReplyTo() != null) {
        where.append(" AND inReplyTo = ?").append(idx++);
        params.add(q.inReplyTo());
    }
    if (q.messageType() != null) {
        where.append(" AND messageType = ?").append(idx++);
        params.add(q.messageType());
    }
    if (q.correlationId() != null) {
        where.append(" AND correlationId = ?").append(idx++);
        params.add(q.correlationId());
    }
    if (q.excludeTypes() != null && !q.excludeTypes().isEmpty()) {
        where.append(" AND messageType NOT IN ?").append(idx++);
        params.add(q.excludeTypes());
    }
    if (q.contentPattern() != null) {
        where.append(" AND LOWER(content) LIKE ?").append(idx++);
        params.add("%" + q.contentPattern().toLowerCase() + "%");
    }

    return new MessageQueryJpql(where.toString(), params.toArray());
}
```

- [ ] **Step 4: Modify JpaMessageStore to inject CurrentPrincipal and filter reads**

Full replacement of `JpaMessageStore.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.MessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;

@ApplicationScoped
public class JpaMessageStore implements MessageStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    @Transactional
    public Message put(Message message) {
        message.persistAndFlush();
        return message;
    }

    @Override
    public Optional<Message> find(Long id) {
        return Message.find("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId())
                .firstResultOptional();
    }

    @Override
    public List<Message> scan(MessageQuery q) {
        MessageQueryJpql mq = MessageQueryJpql.from(q, currentPrincipal.tenancyId());
        String jpql = "FROM Message WHERE " + mq.where()
                + (q.descending() ? " ORDER BY id DESC" : " ORDER BY id ASC");
        if (q.limit() != null) {
            return Message.find(jpql, mq.params()).page(0, q.limit()).list();
        }
        return Message.list(jpql, mq.params());
    }

    @Override
    @Transactional
    public void deleteAll(UUID channelId) {
        Message.delete("channelId = ?1 AND tenancyId = ?2", channelId, currentPrincipal.tenancyId());
    }

    @Override
    @Transactional
    public void delete(Long id) {
        Message.delete("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId());
    }

    @Override
    public int countByChannel(UUID channelId) {
        return (int) Message.count("channelId = ?1 AND tenancyId = ?2",
                channelId, currentPrincipal.tenancyId());
    }

    @Override
    public long count(MessageQuery q) {
        MessageQueryJpql mq = MessageQueryJpql.from(q, currentPrincipal.tenancyId());
        return Message.count(mq.where(), mq.params());
    }

    @Override
    public Map<UUID, Long> countAllByChannel() {
        @SuppressWarnings("unchecked")
        List<Object[]> rows = Message.getEntityManager()
                .createQuery("SELECT m.channelId, COUNT(m) FROM Message m "
                        + "WHERE m.tenancyId = ?1 GROUP BY m.channelId")
                .setParameter(1, currentPrincipal.tenancyId())
                .getResultList();
        return rows.stream().collect(Collectors.toMap(r -> (UUID) r[0], r -> (Long) r[1]));
    }

    @Override
    public List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
        @SuppressWarnings("unchecked")
        List<String> result = Message.getEntityManager()
                .createQuery("SELECT DISTINCT m.sender FROM Message m "
                        + "WHERE m.channelId = ?1 AND m.messageType != ?2 "
                        + "AND m.tenancyId = ?3 ORDER BY m.sender")
                .setParameter(1, channelId)
                .setParameter(2, excludedType)
                .setParameter(3, currentPrincipal.tenancyId())
                .getResultList();
        return result;
    }

    @Override
    public Optional<Message> findLastMessage(UUID channelId) {
        return Message.<Message>find(
                "channelId = ?1 AND tenancyId = ?2 ORDER BY id DESC",
                channelId, currentPrincipal.tenancyId())
                .page(0, 1)
                .firstResultOptional();
    }
}
```

- [ ] **Step 5: Run message isolation test — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenantIsolationTest
```
Expected: all isolation tests pass.

- [ ] **Step 6: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageQueryJpql.java \
  runtime/src/test/java/io/casehub/qhorus/store/TenantIsolationTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): JpaMessageStore + MessageQueryJpql tenant filtering"
```

---

### Task 9: JpaCommitmentStore + JpaWatchdogStore tenant filtering

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaWatchdogStore.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/store/TenantIsolationTest.java`

- [ ] **Step 1: Add commitment isolation test**

Add to `TenantIsolationTest`:
```java
@Inject CommitmentStore commitmentStore;

@Test
@Transactional
void commitment_createdInTenantA_notVisibleToTenantB() {
    Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_A);
    UUID channelId = UUID.randomUUID();
    io.casehub.qhorus.runtime.message.Commitment c =
            new io.casehub.qhorus.runtime.message.Commitment();
    c.id = UUID.randomUUID();
    c.correlationId = "corr-" + UUID.randomUUID();
    c.channelId = channelId;
    c.obligor = "agent-a";
    c.requester = "agent-req";
    c.state = io.casehub.qhorus.runtime.message.CommitmentState.OPEN;
    c.messageType = io.casehub.qhorus.api.message.MessageType.COMMAND;
    c.tenancyId = TENANT_A;
    commitmentStore.save(c);

    Mockito.when(currentPrincipal.tenancyId()).thenReturn(TENANT_B);
    var open = commitmentStore.findAllOpen();
    assertThat(open).noneMatch(x -> x.id.equals(c.id));
}
```

Add import: `import io.casehub.qhorus.runtime.store.CommitmentStore;`

- [ ] **Step 2: Modify JpaCommitmentStore**

Add `@Inject CurrentPrincipal currentPrincipal;` and add `AND tenancyId = ?N` to all read queries. Below is the complete replacement for `JpaCommitmentStore.java` — only query strings change; `save()` and transactional delete methods are unchanged structurally:

Key changes (all `repo.find/list` calls gain `AND tenancyId = ?N`):
```java
@ApplicationScoped
public class JpaCommitmentStore implements CommitmentStore {

    @Inject CommitmentPanacheRepo repo;
    @Inject CurrentPrincipal currentPrincipal;

    @Override @Transactional
    public Commitment save(Commitment c) {
        if (c.id == null) { repo.persist(c); } else { c = repo.getEntityManager().merge(c); }
        return c;
    }

    @Override
    public Optional<Commitment> findById(UUID id) {
        return repo.find("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId())
                .firstResultOptional();
    }

    @Override @Transactional
    public Optional<Commitment> findByCorrelationId(String correlationId) {
        String tenancy = currentPrincipal.tenancyId();
        return repo.find("correlationId = ?1 AND tenancyId = ?2 ORDER BY createdAt DESC",
                        correlationId, tenancy)
                .list().stream()
                .filter(c -> !c.state.isTerminal())
                .findFirst()
                .or(() -> repo.find("correlationId = ?1 AND tenancyId = ?2 ORDER BY createdAt DESC",
                        correlationId, tenancy).firstResultOptional());
    }

    @Override
    public List<Commitment> findOpenByObligor(String obligor, UUID channelId) {
        return repo.list("obligor = ?1 AND channelId = ?2 AND state NOT IN ?3 AND tenancyId = ?4",
                obligor, channelId, terminalStates(), currentPrincipal.tenancyId());
    }

    @Override
    public List<Commitment> findOpenByObligor(String obligor) {
        if (obligor == null) return List.of();
        return repo.list("obligor = ?1 AND state NOT IN ?2 AND tenancyId = ?3",
                obligor, terminalStates(), currentPrincipal.tenancyId());
    }

    @Override
    public List<Commitment> findOpenByRequester(String requester, UUID channelId) {
        return repo.list("requester = ?1 AND channelId = ?2 AND state NOT IN ?3 AND tenancyId = ?4",
                requester, channelId, terminalStates(), currentPrincipal.tenancyId());
    }

    @Override
    public List<Commitment> findByState(CommitmentState state, UUID channelId) {
        return repo.list("state = ?1 AND channelId = ?2 AND tenancyId = ?3",
                state, channelId, currentPrincipal.tenancyId());
    }

    @Override
    public List<Commitment> findExpiredBefore(Instant cutoff) {
        return repo.list("expiresAt < ?1 AND state NOT IN ?2 AND tenancyId = ?3",
                cutoff, terminalStates(), currentPrincipal.tenancyId());
    }

    @Override
    public List<Commitment> findAllOpen() {
        return repo.list("state IN ?1 AND tenancyId = ?2 ORDER BY expiresAt ASC NULLS LAST",
                List.of(CommitmentState.OPEN, CommitmentState.ACKNOWLEDGED),
                currentPrincipal.tenancyId());
    }

    @Override @Transactional
    public void deleteById(UUID id) { repo.deleteById(id); }

    @Override @Transactional
    public long deleteAll(UUID channelId) {
        return repo.delete("channelId = ?1", channelId);
    }

    @Override @Transactional
    public long deleteExpiredBefore(Instant cutoff) {
        return repo.delete("expiresAt < ?1 AND state NOT IN ?2", cutoff, terminalStates());
    }

    private List<CommitmentState> terminalStates() {
        return List.of(CommitmentState.FULFILLED, CommitmentState.DECLINED,
                CommitmentState.FAILED, CommitmentState.DELEGATED, CommitmentState.EXPIRED);
    }
}
```

- [ ] **Step 3: Modify JpaWatchdogStore**

Add `@Inject CurrentPrincipal currentPrincipal;` and filter `scan()` and `find()`:
```java
@ApplicationScoped
public class JpaWatchdogStore implements WatchdogStore {

    @Inject CurrentPrincipal currentPrincipal;

    @Override @Transactional
    public Watchdog put(Watchdog watchdog) {
        watchdog.persistAndFlush();
        return watchdog;
    }

    @Override
    public Optional<Watchdog> find(UUID id) {
        return Watchdog.find("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId())
                .firstResultOptional();
    }

    @Override
    public List<Watchdog> scan(WatchdogQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Watchdog WHERE tenancyId = ?1");
        List<Object> params = new ArrayList<>();
        params.add(currentPrincipal.tenancyId());
        int idx = 2;

        if (q.conditionType() != null) {
            jpql.append(" AND conditionType = ?").append(idx++);
            params.add(q.conditionType());
        }

        return Watchdog.list(jpql.toString(), params.toArray());
    }

    @Override @Transactional
    public void delete(UUID id) {
        Watchdog.delete("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId());
    }
}
```

- [ ] **Step 4: Run isolation tests + full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenantIsolationTest
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCommitmentStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaWatchdogStore.java \
  runtime/src/test/java/io/casehub/qhorus/store/TenantIsolationTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): JpaCommitmentStore + JpaWatchdogStore tenant filtering"
```

---

### Task 10: ChannelService + ReactiveChannelService tenancyId stamping

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java`

- [ ] **Step 1: Add CurrentPrincipal injection to ChannelService**

In `ChannelService.java`, add:
```java
@Inject
CurrentPrincipal currentPrincipal;
```

Add import: `import io.casehub.platform.api.identity.CurrentPrincipal;`

- [ ] **Step 2: Stamp tenancyId in ChannelService.populateChannel()**

In the existing `private Channel populateChannel(ChannelCreateRequest req)` method, add one line before the `return channel;`:
```java
channel.tenancyId = currentPrincipal.tenancyId();
```

- [ ] **Step 3: Add CurrentPrincipal injection to ReactiveChannelService**

`ReactiveChannelService` has its own `private static Channel populateChannel(ChannelCreateRequest req)` (it's `static`). Change it to `private Channel populateChannel(ChannelCreateRequest req)` (remove `static`) and inject `CurrentPrincipal`:

```java
@Inject
CurrentPrincipal currentPrincipal;
```

In `ReactiveChannelService.populateChannel()`, add before `return channel;`:
```java
channel.tenancyId = currentPrincipal.tenancyId();
```

- [ ] **Step 4: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): ChannelService + ReactiveChannelService stamp tenancyId at creation"
```

---

### Task 11: QhorusMcpTools — register_watchdog and list_watchdogs tenancyId

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Add CurrentPrincipal injection to QhorusMcpToolsBase (or QhorusMcpTools directly)**

Check whether `QhorusMcpToolsBase` already injects `CurrentPrincipal`. If not, add in `QhorusMcpTools.java`:
```java
@Inject
CurrentPrincipal currentPrincipal;
```

- [ ] **Step 2: Stamp tenancyId in register_watchdog**

In `QhorusMcpTools.registerWatchdog()`, add after `w.createdBy = createdBy;`:
```java
w.tenancyId = currentPrincipal.tenancyId();
```

- [ ] **Step 3: Filter list_watchdogs by tenant**

Replace `Watchdog.<Watchdog>listAll()` with a tenant-filtered query:
```java
Watchdog.<Watchdog>list("tenancyId = ?1", currentPrincipal.tenancyId())
```

- [ ] **Step 4: Apply same changes to ReactiveQhorusMcpTools.registerWatchdog() and listWatchdogs()**

In `ReactiveQhorusMcpTools`, find the reactive `register_watchdog` tool and add `w.tenancyId = currentPrincipal.tenancyId();` before `w.persist()`. Update the reactive `list_watchdogs` to filter by tenant.

- [ ] **Step 5: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): MCP register_watchdog + list_watchdogs tenant scoped"
```

---

### Task 12: MessageReceivedEvent SPI break + MessageObserverDispatcher

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/MessageReceivedEvent.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java`

- [ ] **Step 1: Add tenancyId to MessageReceivedEvent**

`MessageReceivedEvent` is a record. Replace its declaration with:
```java
public record MessageReceivedEvent(
        String channelName,
        UUID channelId,
        String tenancyId,
        MessageType messageType,
        String senderId,
        String correlationId,
        String content) {}
```

This is a breaking change. All call sites that construct `MessageReceivedEvent` must add `tenancyId` as the 3rd argument.

- [ ] **Step 2: Fix MessageObserverDispatcher constructor call**

In `MessageObserverDispatcher.dispatch()`, the event construction currently is:
```java
final MessageReceivedEvent event = new MessageReceivedEvent(
        channelName, channelId,
        message.messageType, message.sender,
        message.correlationId, content);
```

The method signature must accept tenancyId. Add `tenancyId` parameter to both `dispatch()` overloads:

```java
static void dispatch(final String channelName, final UUID channelId,
        final String tenancyId,
        final Message message,
        final Iterable<? extends Instance.Handle<MessageObserver>> handles) {
    dispatch(channelName, channelId, tenancyId, message, handles, null);
}

static void dispatch(final String channelName, final UUID channelId,
        final String tenancyId,
        final Message message,
        final Iterable<? extends Instance.Handle<MessageObserver>> handles,
        final TransactionSynchronizationRegistry tsr) {
    final String content = message.messageType == MessageType.EVENT ? null : message.content;
    final MessageReceivedEvent event = new MessageReceivedEvent(
            channelName, channelId, tenancyId,
            message.messageType, message.sender,
            message.correlationId, content);
    // ... rest of method unchanged ...
}
```

- [ ] **Step 3: Fix MessageService call site**

`MessageService.dispatch()` calls `MessageObserverDispatcher.dispatch(...)`. Find and update the call to pass `message.tenancyId` as the third argument (after `dispatch.channelId()`):

The current call is:
```java
MessageObserverDispatcher.dispatch(
        ch != null ? ch.name : null, dispatch.channelId(), message, observers.handles(), tsr);
```

Replace with:
```java
MessageObserverDispatcher.dispatch(
        ch != null ? ch.name : null, dispatch.channelId(),
        message.tenancyId,
        message, observers.handles(), tsr);
```

- [ ] **Step 4: Fix ReactiveMessageService call site (if it calls dispatch)**

Find any call to `MessageObserverDispatcher.dispatch` in `ReactiveMessageService` and add `message.tenancyId` as 3rd argument.

- [ ] **Step 5: Fix InProcessMessageBus (constructs MessageReceivedEvent)**

`InProcessMessageBus` observes `MessageReceivedEvent` — it does NOT construct it, so it's unaffected. Verify by searching for `new MessageReceivedEvent` — should only be in `MessageObserverDispatcher`.

- [ ] **Step 6: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime -q
```
Expected: BUILD SUCCESS. If other call sites exist, fix them.

- [ ] **Step 7: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass. Update any test that constructs `MessageReceivedEvent` directly to add `tenancyId`.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  api/src/main/java/io/casehub/qhorus/api/gateway/MessageReceivedEvent.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): MessageReceivedEvent gains tenancyId — SPI break; MessageObserverDispatcher updated"
```

---

### Task 13: MessageDispatch 14th field + MessageService tenancyId resolution

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/message/MessageDispatchTenancyTest.java`

- [ ] **Step 1: Write failing test**

`runtime/src/test/java/io/casehub/qhorus/message/MessageDispatchTenancyTest.java`:
```java
package io.casehub.qhorus.message;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.store.ChannelStore;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MessageDispatchTenancyTest {

    @Inject MessageService messageService;
    @Inject ChannelStore channelStore;
    @InjectMock CurrentPrincipal currentPrincipal;

    private Channel createChannel(String tenancyId) {
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(tenancyId);
        // Use ChannelService to stamp tenancyId
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = "dispatch-test-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;
        ch.tenancyId = tenancyId;
        // Put directly to bypass ChannelService (avoid circular) — tenancyId already set
        io.quarkus.narayana.jta.runtime.TransactionUtils.runInTransaction(() -> channelStore.put(ch));
        return ch;
    }

    @Test
    void dispatch_normPath_usesCurrentPrincipalTenancy() {
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(TenancyConstants.DEFAULT_TENANT_ID);
        Mockito.when(currentPrincipal.actorId()).thenReturn("test-actor");
        Channel ch = createChannel(TenancyConstants.DEFAULT_TENANT_ID);

        var result = messageService.dispatch(MessageDispatch.builder()
                .channelId(ch.id)
                .sender("agent-1")
                .type(MessageType.QUERY)
                .actorType(ActorType.AGENT)
                .build());

        assertThat(result).isNotNull();
    }

    @Test
    void dispatch_explicitTenancyId_usedAsIs() {
        String explicitTenancy = TenancyConstants.DEFAULT_TENANT_ID;
        Channel ch = createChannel(explicitTenancy);
        // No CurrentPrincipal needed — explicit tenancyId set on dispatch
        Mockito.when(currentPrincipal.tenancyId()).thenReturn("any-other-tenant");

        var result = messageService.dispatch(MessageDispatch.builder()
                .channelId(ch.id)
                .sender("system:watchdog")
                .type(MessageType.STATUS)
                .actorType(ActorType.SYSTEM)
                .tenancyId(explicitTenancy)
                .build());

        assertThat(result).isNotNull();
    }

    @Test
    void dispatch_crossTenantAttempt_rejected() {
        String channelTenancy = TenancyConstants.DEFAULT_TENANT_ID;
        Channel ch = createChannel(channelTenancy);
        String differentTenancy = "other-tenant-" + UUID.randomUUID();
        Mockito.when(currentPrincipal.tenancyId()).thenReturn(differentTenancy);

        assertThatThrownBy(() -> messageService.dispatch(MessageDispatch.builder()
                .channelId(ch.id)
                .sender("intruder")
                .type(MessageType.QUERY)
                .actorType(ActorType.AGENT)
                .build()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Cross-tenant dispatch rejected");
    }
}
```

- [ ] **Step 2: Add tenancyId as 14th field to MessageDispatch**

In `MessageDispatch.java`, the record declaration currently ends at `String telemetry`. Add `String tenancyId` as the 14th component:

```java
public record MessageDispatch(
        UUID channelId,
        String sender,
        MessageType type,
        String content,
        String correlationId,
        Long inReplyTo,
        String artefactRefs,
        String target,
        UUID subjectId,
        UUID causedByEntryId,
        ActorType actorType,
        Instant deadline,
        String telemetry,
        String tenancyId) {
```

In `Builder`, add field and method:
```java
private String tenancyId;
public Builder tenancyId(String v) { this.tenancyId = v; return this; }
```

In `Builder.build()`, pass `tenancyId` as the 14th argument to the canonical constructor (after `telemetry`).

- [ ] **Step 3: Fix all canonical constructor call sites**

Search for `new MessageDispatch(` across the codebase. Any call using the 13-arg constructor now needs a 14th `null` argument (tenancyId). Run:
```bash
grep -rn "new MessageDispatch(" /Users/mdproctor/claude/casehub/qhorus/runtime/src
```
Fix each occurrence by appending `, null)` before the closing `)`. Callers using `MessageDispatch.builder()` need no changes.

- [ ] **Step 4: Modify MessageService.dispatch() — tenancyId resolution + CrossTenantChannelStore**

At the top of `MessageService.dispatch()`, add these injections to the class:
```java
@Inject
CurrentPrincipal currentPrincipal;

@CrossTenant
@Inject
CrossTenantChannelStore crossTenantChannelStore;
```

Add imports:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.api.spi.CrossTenantChannelStore;
```

Replace the existing channel lookup at the start of `dispatch()`:
```java
// was: final Channel ch = channelService.findById(dispatch.channelId()).orElse(null);
final String effectiveTenancyId = dispatch.tenancyId() != null
        ? dispatch.tenancyId()
        : currentPrincipal.tenancyId();

final Channel ch = crossTenantChannelStore.findById(dispatch.channelId()).orElse(null);

if (ch != null && !effectiveTenancyId.equals(ch.tenancyId)) {
    throw new IllegalArgumentException(
            "Cross-tenant dispatch rejected: caller tenant=" + effectiveTenancyId
            + ", channel tenant=" + ch.tenancyId);
}
```

Then in the "Normal insert" section where `message` is constructed, add:
```java
message.tenancyId = effectiveTenancyId;
```
(add this line after `message.deadline = dispatch.deadline();`)

- [ ] **Step 5: Run failing tests to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageDispatchTenancyTest
```
Expected: tests pass.

- [ ] **Step 6: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java \
  runtime/src/test/java/io/casehub/qhorus/message/MessageDispatchTenancyTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): MessageDispatch.tenancyId + MessageService enforcement gate"
```

---

### Task 14: LedgerWriteService — MessageLedgerEntry tenancyId population

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java`

- [ ] **Step 1: Stamp tenancyId on MessageLedgerEntry in record()**

In `LedgerWriteService.record()`, after the block that sets `entry.sequenceNumber`, add:
```java
entry.tenancyId = dispatch.tenancyId();
```

`dispatch.tenancyId()` is non-null at this point because `MessageService.dispatch()` resolves and stamps it on the `MessageDispatch` before calling `ledgerWriteService.record()`.

Wait — `dispatch` passed to `record()` is the original `MessageDispatch` object, which may have `tenancyId=null` for normal callers (the service resolved it locally but didn't create a new dispatch). Fix: pass `effectiveTenancyId` separately to `record()`, OR create a resolved dispatch.

**Correct approach:** Pass `effectiveTenancyId` to `record()` by updating its signature.

Update `LedgerWriteService.record()` signature:
```java
@Transactional(value = Transactional.TxType.REQUIRES_NEW)
public LedgerWriteOutcome record(final MessageDispatch dispatch,
        final Long messageId,
        @Nullable final UUID commitmentId,
        final Instant occurredAt,
        final String tenancyId) {
```

And add: `entry.tenancyId = tenancyId;`

Update the call site in `MessageService.dispatch()`:
```java
final LedgerWriteOutcome ledgerOutcome =
        ledgerWriteService.record(dispatch, messageId, storedCommitmentId, occurredAt, effectiveTenancyId);
```

Also update `ReactiveLedgerWriteService.record()` the same way (add `tenancyId` param) and stamp it on the entry.

Find any other call sites to `ledgerWriteService.record()` and add the `tenancyId` argument (pass `TenancyConstants.DEFAULT_TENANT_ID` where context is unavailable, or the message's tenancyId if available).

- [ ] **Step 2: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass. Update any tests that call `ledgerWriteService.record()` directly to add the tenancyId argument.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): LedgerWriteService stamps tenancyId on MessageLedgerEntry"
```

---

### Task 15: WatchdogEvaluationService cross-tenant redesign

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`

- [ ] **Step 1: Replace tenant-filtered injections with cross-tenant equivalents**

In `WatchdogEvaluationService`, change:
```java
// Remove:
@Inject ChannelService channelService;
@Inject WatchdogStore watchdogStore;
@Inject MessageStore messageStore;
@Inject CommitmentStore commitmentStore;

// Add:
@CrossTenant @Inject CrossTenantChannelStore crossTenantChannelStore;
@CrossTenant @Inject CrossTenantMessageStore crossTenantMessageStore;
@CrossTenant @Inject CrossTenantCommitmentStore crossTenantCommitmentStore;
@CrossTenant @Inject CrossTenantWatchdogStore crossTenantWatchdogStore;
```

Add imports for the cross-tenant types and qualifier.

- [ ] **Step 2: Update evaluateAll() to use cross-tenant watchdog store**

```java
List<Watchdog> watchdogs = crossTenantWatchdogStore.listAll();
```

- [ ] **Step 3: Update evaluateBarrierStuck() — replace channelService.listAll() and messageStore calls**

```java
// was: channelService.listAll().stream()
List<Channel> barriers = crossTenantChannelStore.listAll().stream()
        .filter(ch -> ch.semantic == ChannelSemantic.BARRIER)
        .filter(ch -> "*".equals(w.targetName) || ch.name.equals(w.targetName))
        .filter(ch -> ch.lastActivityAt == null || ch.lastActivityAt.isBefore(cutoff) || threshold == 0)
        .toList();
// ...
// was: messageStore.distinctSendersByChannel(...)
List<String> written = crossTenantMessageStore.distinctSendersByChannel(ch.id, MessageType.EVENT);
```

- [ ] **Step 4: Update evaluateApprovalPending() — replace commitmentStore**

```java
// was: commitmentStore.findAllOpen()
List<Commitment> pending = crossTenantCommitmentStore.findAllOpen()
```

- [ ] **Step 5: Update evaluateChannelIdle() and evaluateQueueDepth()**

```java
// evaluateChannelIdle: was channelService.listAll()
List<Channel> idle = crossTenantChannelStore.listAll().stream()...

// evaluateQueueDepth: was channelService.listAll() and messageStore.count()
List<Channel> channels = crossTenantChannelStore.listAll().stream()...
long count = crossTenantMessageStore.countByChannel(ch.id);  // Note: no MessageQuery needed
```

For `evaluateQueueDepth`, `crossTenantMessageStore` doesn't have `count(MessageQuery)`. Use `countByChannel(ch.id)` which counts all messages regardless of type. If the existing behaviour specifically excluded EVENTs, use `scan()` + `size()` or add a count method. Check the original: it used `MessageQuery.builder().channelId(ch.id).excludeTypes(List.of(MessageType.EVENT)).build()`. Use `crossTenantMessageStore.scan(MessageQuery.builder().channelId(ch.id).excludeTypes(List.of(MessageType.EVENT)).build()).size()` as a fallback (or add `count(MessageQuery)` to `CrossTenantMessageStore`).

Add `long count(MessageQuery query)` to `CrossTenantMessageStore` interface and `JpaCrossTenantMessageStore`:
```java
// CrossTenantMessageStore:
long count(MessageQuery query);

// JpaCrossTenantMessageStore:
@Override
public long count(MessageQuery q) {
    MessageQueryJpql mq = MessageQueryJpql.from(q);
    return Message.count(mq.where(), mq.params());
}
```

Then in `evaluateQueueDepth`:
```java
long count = crossTenantMessageStore.count(
        MessageQuery.builder().channelId(ch.id).excludeTypes(List.of(MessageType.EVENT)).build());
```

- [ ] **Step 6: Update fireAlert() — replace channelService.findByName() and messageService.dispatch()**

```java
private void fireAlert(Watchdog w, String summary, AlertContext context, Instant now) {
    alertEvents.fireAsync(new WatchdogAlertEvent(
            w.id, w.targetName, w.notificationChannel, summary, now, context));

    // Use cross-tenant lookup + tenancyId from watchdog entity (GE-20260531-446fea)
    Optional<Channel> notifChannel = crossTenantChannelStore
            .findByNameAndTenancy(w.notificationChannel, w.tenancyId);
    if (notifChannel.isEmpty()) {
        return;
    }
    messageService.dispatch(MessageDispatch.builder()
            .channelId(notifChannel.get().id)
            .sender("system:watchdog")
            .type(MessageType.STATUS)
            .content(summary)
            .actorType(ActorType.SYSTEM)
            .tenancyId(w.tenancyId)       // explicit tenancyId — no request context in scheduler
            .build());
}
```

- [ ] **Step 7: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass. The `WatchdogEvaluationServiceTest` uses `InMemory*` stores via `@InjectMock` — those now need to be the cross-tenant variants. Update the test if needed.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java \
  api/src/main/java/io/casehub/qhorus/api/spi/CrossTenantMessageStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantMessageStore.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): WatchdogEvaluationService uses cross-tenant stores; fireAlert passes tenancyId"
```

---

### Task 16: ChannelGateway startup recovery

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`

- [ ] **Step 1: Replace channelService.listAll() with cross-tenant store**

In `ChannelGateway`, add injection:
```java
@CrossTenant
@Inject
CrossTenantChannelStore crossTenantChannelStore;
```

Find the `@Observes StartupEvent` method (around line 87 per the search results):
```java
// was: for (Channel ch : channelService.listAll()) {
for (Channel ch : crossTenantChannelStore.listAll()) {
```

- [ ] **Step 2: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: all pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#260): ChannelGateway startup recovery uses CrossTenantChannelStore"
```

---

### Task 17: Full build + PLATFORM.md update

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`

- [ ] **Step 1: Full project build including examples**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
Expected: BUILD SUCCESS across all modules (runtime, api, testing, deployment, examples).

- [ ] **Step 2: Update PLATFORM.md**

In the `casehub-qhorus` Capability Ownership entry, update the `MessageDispatch` builder description from "13 fields" to "14 fields (adds `tenancyId`)". Also add to the "Agent-to-agent messaging" row note that dispatch is now tenant-scoped.

- [ ] **Step 3: Commit PLATFORM.md to parent repo**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: qhorus MessageDispatch gains tenancyId (14th field) — refs casehubio/qhorus#260"
```

- [ ] **Step 4: Final qhorus project commit for CLAUDE.md**

Update `CLAUDE.md` in the project repo:
- Update "Next domain migration: V18" → "Next domain migration: V22"
- Update MessageDispatch description to reference 14 fields and tenancyId

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#260): update CLAUDE.md — V22 next migration, MessageDispatch 14 fields"
```

---

## Self-Review

**Spec coverage:**
- ✅ V18-V21 migrations (Task 1)
- ✅ @CrossTenant + @QhorusSystem qualifiers (Task 2)
- ✅ QhorusSystemCurrentPrincipal (Task 2)
- ✅ CrossTenantProducer + guard (Task 4)
- ✅ Cross-tenant store interfaces in api/spi/ (Task 3)
- ✅ JPA cross-tenant stores (Task 4)
- ✅ InMemory cross-tenant stores + null-guard (Task 6)
- ✅ Entity tenancyId fields (Task 5)
- ✅ JpaChannelStore tenant filtering (Task 7)
- ✅ JpaMessageStore + MessageQueryJpql (Task 8)
- ✅ JpaCommitmentStore + JpaWatchdogStore (Task 9)
- ✅ ChannelService + ReactiveChannelService stamping (Task 10)
- ✅ QhorusMcpTools watchdog tenancyId (Task 11)
- ✅ MessageReceivedEvent SPI break (Task 12)
- ✅ MessageObserverDispatcher update (Task 12)
- ✅ MessageDispatch 14th field (Task 13)
- ✅ MessageService.dispatch() tenancyId resolution + cross-tenant validation (Task 13)
- ✅ LedgerWriteService tenancyId (Task 14)
- ✅ WatchdogEvaluationService cross-tenant redesign (Task 15)
- ✅ ChannelGateway startup recovery (Task 16)
- ✅ TenantIsolationTest (Tasks 7-9)
- ✅ CrossTenantProducerTest (Task 4)
- ✅ MessageDispatchTenancyTest (Task 13)
- ✅ PLATFORM.md + CLAUDE.md update (Task 17)

**Placeholder scan:** No TBDs. All code blocks are complete.

**Type consistency:** `effectiveTenancyId` resolved in Task 13 and passed to `LedgerWriteService.record()` in Task 14 — consistent. `CrossTenantChannelStore.findByNameAndTenancy()` defined in Task 3, implemented in Task 4, used in Task 15 — consistent.
