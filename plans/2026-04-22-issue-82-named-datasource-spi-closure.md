# Named Datasource Isolation + MessageStore SPI Closure — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Switch Qhorus to the named "qhorus" datasource/persistence unit and close three direct EntityManager bypasses in the MCP tools and WatchdogEvaluationService by routing them through the MessageStore SPI.

**Architecture:** Two new methods added to `MessageStore` and `ReactiveMessageStore` SPI interfaces, implemented in all four store variants. The MCP tools and watchdog service then call these methods instead of `Message.getEntityManager()` directly. Named datasource activated via config + one qualifier annotation.

**Tech Stack:** Quarkus 3.32.2, Hibernate ORM Panache, SmallRye Mutiny, H2 (tests), `io.quarkus.hibernate.orm.PersistenceUnit` CDI qualifier

**Issues:** Epic #82 — closes #83 (SPI) and #84 (named datasource)

---

## File Map

| File | Change |
|---|---|
| `runtime/src/main/java/.../store/MessageStore.java` | Add two methods |
| `runtime/src/main/java/.../store/ReactiveMessageStore.java` | Add two reactive methods |
| `runtime/src/main/java/.../store/jpa/JpaMessageStore.java` | Implement both methods |
| `runtime/src/main/java/.../store/jpa/ReactiveJpaMessageStore.java` | Implement both methods |
| `testing/src/main/java/.../testing/InMemoryMessageStore.java` | Implement both methods |
| `testing/src/main/java/.../testing/InMemoryReactiveMessageStore.java` | Implement both methods |
| `testing/src/test/java/.../testing/contract/MessageStoreContractTest.java` | Add 3 tests + 2 abstract methods |
| `testing/src/test/java/.../testing/contract/InMemoryMessageStoreTest.java` | Implement 2 abstract methods |
| `testing/src/test/java/.../testing/contract/InMemoryReactiveMessageStoreTest.java` | Implement 2 abstract methods |
| `runtime/src/main/java/.../mcp/QhorusMcpTools.java` | Inject `MessageStore`; replace 2 bypasses |
| `runtime/src/main/java/.../mcp/ReactiveQhorusMcpTools.java` | Replace 1 bypass |
| `runtime/src/main/java/.../watchdog/WatchdogEvaluationService.java` | Inject `MessageStore`; replace 1 bypass |
| `runtime/src/main/resources/application.properties` | Add named PU packages + Flyway config |
| `runtime/src/test/resources/application.properties` | Rename datasource keys to `qhorus.*` |
| `runtime/src/main/java/.../ledger/AgentMessageLedgerEntryRepository.java` | Add `@PersistenceUnit("qhorus")` qualifier |
| `adr/0004-named-datasource-isolation.md` | New ADR capturing ledger coupling |
| `adr/INDEX.md` | Add ADR-0004 entry |

Packages abbreviated: `io.quarkiverse.qhorus.runtime` = `...runtime`, `io.quarkiverse.qhorus.testing` = `...testing`

---

## Task 1: Add aggregate methods to MessageStore and ReactiveMessageStore interfaces

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/MessageStore.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveMessageStore.java`

- [ ] **Step 1: Add two methods to `MessageStore`**

Full file after change:

```java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;

public interface MessageStore {
    Message put(Message message);

    Optional<Message> find(Long id);

    List<Message> scan(MessageQuery query);

    void deleteAll(UUID channelId);

    void delete(Long id);

    int countByChannel(UUID channelId);

    Map<UUID, Long> countAllByChannel();

    List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType);
}
```

- [ ] **Step 2: Add two reactive methods to `ReactiveMessageStore`**

Full file after change:

```java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.smallrye.mutiny.Uni;

public interface ReactiveMessageStore {
    Uni<Message> put(Message message);

    Uni<Optional<Message>> find(Long id);

    Uni<List<Message>> scan(MessageQuery query);

    Uni<Void> deleteAll(UUID channelId);

    Uni<Void> delete(Long id);

    Uni<Integer> countByChannel(UUID channelId);

    Uni<Map<UUID, Long>> countAllByChannel();

    Uni<List<String>> distinctSendersByChannel(UUID channelId, MessageType excludedType);
}
```

- [ ] **Step 3: Verify the project does not compile yet**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | grep "error:" | head -20
```

Expected: compile errors in `JpaMessageStore`, `ReactiveJpaMessageStore`, `InMemoryMessageStore`, `InMemoryReactiveMessageStore` — "does not override abstract method".

---

## Task 2: Add contract tests + implement in InMemory stores

**Files:**
- Modify: `testing/src/test/java/io/quarkiverse/qhorus/testing/contract/MessageStoreContractTest.java`
- Modify: `testing/src/test/java/io/quarkiverse/qhorus/testing/contract/InMemoryMessageStoreTest.java`
- Modify: `testing/src/test/java/io/quarkiverse/qhorus/testing/contract/InMemoryReactiveMessageStoreTest.java`
- Modify: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryMessageStore.java`
- Modify: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactiveMessageStore.java`

- [ ] **Step 1: Add abstract methods and three tests to `MessageStoreContractTest`**

Add these imports at the top of the file (after existing imports):
```java
import java.util.Map;
```

Add these abstract methods after the existing abstract method declarations:
```java
protected abstract Map<UUID, Long> countAllByChannel();

protected abstract List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType);
```

Add these test methods after the existing tests:
```java
@Test
void countAllByChannel_returnsCountPerChannel() {
    UUID ch1 = UUID.randomUUID();
    UUID ch2 = UUID.randomUUID();
    put(msg(ch1, "alice", MessageType.REQUEST));
    put(msg(ch1, "bob", MessageType.RESPONSE));
    put(msg(ch2, "carol", MessageType.REQUEST));

    Map<UUID, Long> counts = countAllByChannel();
    assertEquals(2L, counts.get(ch1));
    assertEquals(1L, counts.get(ch2));
}

@Test
void distinctSendersByChannel_excludesSpecifiedType() {
    UUID ch = UUID.randomUUID();
    put(msg(ch, "alice", MessageType.REQUEST));
    put(msg(ch, "bob", MessageType.RESPONSE));
    put(msg(ch, "system", MessageType.EVENT));

    List<String> senders = distinctSendersByChannel(ch, MessageType.EVENT);
    assertTrue(senders.contains("alice"));
    assertTrue(senders.contains("bob"));
    assertFalse(senders.contains("system"));
}

@Test
void distinctSendersByChannel_returnsDistinct() {
    UUID ch = UUID.randomUUID();
    put(msg(ch, "alice", MessageType.REQUEST));
    put(msg(ch, "alice", MessageType.RESPONSE));

    List<String> senders = distinctSendersByChannel(ch, MessageType.EVENT);
    assertEquals(1, senders.size());
    assertEquals("alice", senders.get(0));
}
```

- [ ] **Step 2: Implement `countAllByChannel()` and `distinctSendersByChannel()` in `InMemoryMessageStore`**

Add this import after existing imports:
```java
import java.util.stream.Collectors;
```

Add these two methods at the end of the class (before the closing brace):
```java
@Override
public Map<UUID, Long> countAllByChannel() {
    return store.values().stream()
            .collect(Collectors.groupingBy(m -> m.channelId, Collectors.counting()));
}

@Override
public List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
    return store.values().stream()
            .filter(m -> channelId.equals(m.channelId) && m.messageType != excludedType)
            .map(m -> m.sender)
            .filter(s -> s != null && !s.isBlank())
            .distinct()
            .sorted()
            .toList();
}
```

Also add the `MessageType` import:
```java
import io.quarkiverse.qhorus.runtime.message.MessageType;
```

- [ ] **Step 3: Implement in `InMemoryReactiveMessageStore`**

Add these two methods at the end of the class (before the closing brace):
```java
@Override
public Uni<Map<UUID, Long>> countAllByChannel() {
    return Uni.createFrom().item(() -> delegate.countAllByChannel());
}

@Override
public Uni<List<String>> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
    return Uni.createFrom().item(() -> delegate.distinctSendersByChannel(channelId, excludedType));
}
```

Also add these imports:
```java
import java.util.Map;
import io.quarkiverse.qhorus.runtime.message.MessageType;
```

- [ ] **Step 4: Implement abstract methods in concrete runners**

In `InMemoryMessageStoreTest`, add after the existing `@Override` methods (look for `reset()` and other overrides):
```java
@Override
protected Map<UUID, Long> countAllByChannel() {
    return store.countAllByChannel();
}

@Override
protected List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
    return store.distinctSendersByChannel(channelId, excludedType);
}
```

Add import: `import java.util.Map;`

In `InMemoryReactiveMessageStoreTest`, add the equivalent (wrapping with `.await().indefinitely()`):
```java
@Override
protected Map<UUID, Long> countAllByChannel() {
    return store.countAllByChannel().await().indefinitely();
}

@Override
protected List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
    return store.distinctSendersByChannel(channelId, excludedType).await().indefinitely();
}
```

Add import: `import java.util.Map;`

- [ ] **Step 5: Run contract tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -q 2>&1 | tail -20
```

Expected: BUILD SUCCESS. All `countAllByChannel` and `distinctSendersByChannel` tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/MessageStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveMessageStore.java \
        testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryMessageStore.java \
        testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactiveMessageStore.java \
        testing/src/test/java/io/quarkiverse/qhorus/testing/contract/MessageStoreContractTest.java \
        testing/src/test/java/io/quarkiverse/qhorus/testing/contract/InMemoryMessageStoreTest.java \
        testing/src/test/java/io/quarkiverse/qhorus/testing/contract/InMemoryReactiveMessageStoreTest.java
git commit -m "feat(store): add countAllByChannel + distinctSendersByChannel to MessageStore SPI

Refs #83"
```

---

## Task 3: Implement in JpaMessageStore

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaMessageStore.java`

- [ ] **Step 1: Add two methods to `JpaMessageStore`**

Add these imports:
```java
import java.util.Map;
import java.util.stream.Collectors;
import io.quarkiverse.qhorus.runtime.message.MessageType;
```

Add these methods at the end of the class:
```java
@Override
public Map<UUID, Long> countAllByChannel() {
    @SuppressWarnings("unchecked")
    List<Object[]> rows = Message.getEntityManager()
            .createQuery("SELECT m.channelId, COUNT(m) FROM Message m GROUP BY m.channelId")
            .getResultList();
    return rows.stream().collect(Collectors.toMap(r -> (UUID) r[0], r -> (Long) r[1]));
}

@Override
public List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
    @SuppressWarnings("unchecked")
    List<String> senders = Message.getEntityManager()
            .createQuery("SELECT DISTINCT m.sender FROM Message m "
                    + "WHERE m.channelId = ?1 AND m.messageType != ?2")
            .setParameter(1, channelId)
            .setParameter(2, excludedType)
            .getResultList();
    return senders;
}
```

- [ ] **Step 2: Run runtime tests to confirm the JPA implementation is correct**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -20
```

Expected: BUILD SUCCESS. All 708 existing tests still pass.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaMessageStore.java
git commit -m "feat(store): implement JpaMessageStore aggregate methods

Refs #83"
```

---

## Task 4: Implement in ReactiveJpaMessageStore

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java`

> Note: `ReactiveJpaMessageStore` is `@Alternative` and `@Disabled` pending Docker/PostgreSQL. These methods use the reactive Panache repo to avoid raw session API complexity — correct for `@Disabled` status.

- [ ] **Step 1: Add two methods to `ReactiveJpaMessageStore`**

Add these imports:
```java
import java.util.Map;
import java.util.stream.Collectors;
import io.quarkiverse.qhorus.runtime.message.MessageType;
```

Add these methods at the end of the class:
```java
@Override
public Uni<Map<UUID, Long>> countAllByChannel() {
    return repo.listAll()
            .map(msgs -> msgs.stream()
                    .collect(Collectors.groupingBy(m -> m.channelId, Collectors.counting())));
}

@Override
public Uni<List<String>> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
    return repo.list("channelId = ?1 AND messageType != ?2", channelId, excludedType)
            .map(msgs -> msgs.stream()
                    .map(m -> m.sender)
                    .filter(s -> s != null && !s.isBlank())
                    .distinct()
                    .sorted()
                    .toList());
}
```

- [ ] **Step 2: Confirm the runtime compiles cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS, no errors.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java
git commit -m "feat(store): implement ReactiveJpaMessageStore aggregate methods

Refs #83"
```

---

## Task 5: Close EntityManager bypasses in QhorusMcpTools

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`

- [ ] **Step 1: Add `MessageStore` injection to `QhorusMcpTools`**

Add the import:
```java
import io.quarkiverse.qhorus.runtime.store.MessageStore;
```

Add the field after the existing `@Inject LedgerWriteService ledgerWriteService;` block:
```java
@Inject
MessageStore messageStore;
```

- [ ] **Step 2: Replace the `listChannels()` bypass (around line 287)**

Find this block in `listChannels()`:
```java
// Batch all message counts in one GROUP BY query — avoids N+1
@SuppressWarnings("unchecked")
List<Object[]> countRows = Message.getEntityManager()
        .createQuery("SELECT m.channelId, COUNT(m) FROM Message m GROUP BY m.channelId")
        .getResultList();
Map<UUID, Long> countByChannel = countRows.stream()
        .collect(Collectors.toMap(r -> (UUID) r[0], r -> (Long) r[1]));
```

Replace it with:
```java
Map<UUID, Long> countByChannel = messageStore.countAllByChannel();
```

- [ ] **Step 3: Replace the barrier check bypass (around line 643)**

Find this block (inside the barrier check section):
```java
@SuppressWarnings("unchecked")
List<String> written = Message.getEntityManager()
        .createQuery("SELECT DISTINCT m.sender FROM Message m "
                + "WHERE m.channelId = ?1 AND m.messageType != ?2")
        .setParameter(1, ch.id)
        .setParameter(2, MessageType.EVENT)
        .getResultList();
```

Replace it with:
```java
List<String> written = messageStore.distinctSendersByChannel(ch.id, MessageType.EVENT);
```

- [ ] **Step 4: Verify no `getEntityManager` calls remain in QhorusMcpTools**

```bash
grep -n "getEntityManager" runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java
```

Expected: no output.

- [ ] **Step 5: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -20
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java
git commit -m "refactor(mcp): route aggregate queries through MessageStore SPI in QhorusMcpTools

Refs #83"
```

---

## Task 6: Close EntityManager bypass in ReactiveQhorusMcpTools and WatchdogEvaluationService

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/watchdog/WatchdogEvaluationService.java`

- [ ] **Step 1: Replace bypass in `ReactiveQhorusMcpTools.blockingCheckMessagesBarrier()`**

`messageStore` is already injected as `ReactiveMessageStore`. Find this block in `blockingCheckMessagesBarrier` (around line 752):

```java
@SuppressWarnings("unchecked")
List<String> written = Message.getEntityManager()
        .createQuery("SELECT DISTINCT m.sender FROM Message m "
                + "WHERE m.channelId = ?1 AND m.messageType != ?2")
        .setParameter(1, ch.id)
        .setParameter(2, MessageType.EVENT)
        .getResultList();
```

Replace with:
```java
List<String> written = messageStore.distinctSendersByChannel(ch.id, MessageType.EVENT)
        .await().indefinitely();
```

(This is inside a `@Blocking` private helper — `.await().indefinitely()` is safe here.)

- [ ] **Step 2: Add `MessageStore` injection to `WatchdogEvaluationService`**

Add the import:
```java
import io.quarkiverse.qhorus.runtime.store.MessageStore;
```

Add the field after `@Inject WatchdogStore watchdogStore;`:
```java
@Inject
MessageStore messageStore;
```

- [ ] **Step 3: Replace bypass in `WatchdogEvaluationService` (around line 113)**

Find this block:
```java
@SuppressWarnings("unchecked")
List<String> written = Message.getEntityManager()
        .createQuery("SELECT DISTINCT m.sender FROM Message m "
                + "WHERE m.channelId = ?1 AND m.messageType != ?2")
        .setParameter(1, ch.id)
        .setParameter(2, MessageType.EVENT)
        .getResultList();
```

Replace with:
```java
List<String> written = messageStore.distinctSendersByChannel(ch.id, MessageType.EVENT);
```

- [ ] **Step 4: Verify no `getEntityManager` calls remain in either file**

```bash
grep -rn "getEntityManager" \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/watchdog/WatchdogEvaluationService.java
```

Expected: no output.

- [ ] **Step 5: Verify no `getEntityManager` calls remain anywhere in the MCP or service layer**

```bash
grep -rn "getEntityManager" \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/watchdog/ \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/data/ \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/instance/
```

Expected: no output. (`AgentMessageLedgerEntryRepository` in `ledger/` is the only permitted remaining user — it is the JPA implementation of the ledger interface, not the tool/service layer.)

- [ ] **Step 6: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -20
```

Expected: BUILD SUCCESS. All modules.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/watchdog/WatchdogEvaluationService.java
git commit -m "refactor(mcp,watchdog): close final EntityManager bypasses — SPI fully sealed

Closes #83"
```

---

## Task 7: Named datasource migration

**Files:**
- Modify: `runtime/src/main/resources/application.properties`
- Modify: `runtime/src/test/resources/application.properties`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntryRepository.java`

- [ ] **Step 1: Add named PU and Flyway config to main `application.properties`**

Add these lines to `runtime/src/main/resources/application.properties` (after the existing Qhorus-specific settings):

```properties
# Named Hibernate ORM persistence unit — all Qhorus entities + ledger base class
# (LedgerEntry included because AgentMessageLedgerEntry extends it via JOINED inheritance;
#  see ADR-0004 for the revisit plan when quarkus-ledger adopts its own named PU)
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.quarkiverse.qhorus.runtime,io.quarkiverse.ledger.runtime

# Flyway migrations for the qhorus datasource
quarkus.flyway.qhorus.locations=db/migration
quarkus.flyway.qhorus.migrate-at-start=true
```

- [ ] **Step 2: Replace default datasource config in test `application.properties`**

Replace the entire datasource/Hibernate block in `runtime/src/test/resources/application.properties`.

Before:
```properties
# Test datasource — H2 in-memory
quarkus.datasource.db-kind=h2
quarkus.datasource.username=sa
quarkus.datasource.password=
quarkus.datasource.jdbc.url=jdbc:h2:mem:test;DB_CLOSE_DELAY=-1

# Hibernate creates all tables from entity definitions (ledger base + Qhorus subclass).
# Flyway is not used in tests — quarkus-ledger no longer ships SQL migrations.
quarkus.hibernate-orm.database.generation=drop-and-create

# Disable Hibernate Reactive — tests use H2 JDBC (no reactive datasource available).
# quarkus-hibernate-reactive-panache is in the extension for reactive SPI support,
# but only activates in consumer apps that configure a reactive datasource.
quarkus.datasource.reactive=false
```

After:
```properties
# Named 'qhorus' datasource — H2 in-memory for tests
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:test;DB_CLOSE_DELAY=-1

# Hibernate creates schema from entities; Flyway disabled in tests.
quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
quarkus.flyway.qhorus.migrate-at-start=false
```

- [ ] **Step 3: Add `@PersistenceUnit("qhorus")` qualifier to `AgentMessageLedgerEntryRepository`**

Add this import:
```java
import io.quarkus.hibernate.orm.PersistenceUnit;
```

Change the `EntityManager` injection from:
```java
@Inject
EntityManager em;
```

To:
```java
@Inject
@PersistenceUnit("qhorus")
EntityManager em;
```

- [ ] **Step 4: Run `mvn clean test` — note `clean` is required to pick up new properties**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -q 2>&1 | tail -30
```

Expected: BUILD SUCCESS. If `FastBootHibernateReactivePersistenceProvider` errors appear, add to test `application.properties`:
```properties
quarkus.datasource.qhorus.reactive.enabled=false
```
and re-run.

If schema creation fails (tables not found), verify that `quarkus.hibernate-orm.qhorus.packages` is correctly specified in the main `application.properties` and that both packages are listed.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/resources/application.properties \
        runtime/src/test/resources/application.properties \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntryRepository.java
git commit -m "feat(config): migrate to named datasource 'qhorus' — quarkus.datasource.qhorus.*

Closes #84"
```

---

## Task 8: Write ADR-0004

**Files:**
- Create: `adr/0004-named-datasource-isolation.md`
- Modify: `adr/INDEX.md`

- [ ] **Step 1: Create `adr/0004-named-datasource-isolation.md`**

```markdown
# 0004 — Named Datasource Isolation: quarkus.datasource.qhorus

Date: 2026-04-22
Status: Accepted

## Context and Problem Statement

The Quarkus AI Agent Ecosystem standardises on named datasources and Hibernate
persistence units matching each library's artifact ID. The embedding application's
default datasource must remain free for its own use. Qhorus was using the default
datasource, conflicting with Claudony's own database.

## Decision

Qhorus uses the named persistence unit and datasource "qhorus" for all persistence:

- `quarkus.datasource.qhorus.*` — datasource connection config (provided by consumer)
- `quarkus.hibernate-orm.qhorus.*` — Hibernate ORM persistence unit
- `quarkus.flyway.qhorus.*` — schema migrations

## Ledger Coupling — Revisit Marker

`AgentMessageLedgerEntry extends LedgerEntry` via JPA JOINED inheritance. JPA
requires all entities in an inheritance hierarchy to share a single persistence unit.
`LedgerEntry` (from `quarkus-ledger`, package `io.quarkiverse.ledger.runtime.model`)
is included in the "qhorus" packages config:

```
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=\
  io.quarkiverse.qhorus.runtime,\
  io.quarkiverse.ledger.runtime
```

This means ledger base tables (`ledger_entry`, supplements) live in the Qhorus
datasource — not in a separate ledger datasource.

**Revisit trigger:** when `quarkus-ledger` adopts its own named persistence unit
("ledger"), the JPA inheritance must be replaced with a FK-only reference. At that
point `AgentMessageLedgerEntry` becomes a standalone entity in the "qhorus" PU,
and the `LedgerEntryRepository` methods that query the base `ledger_entry` table
will need a cross-datasource strategy.

## Consequences

**Positive:**
- Default datasource remains free for embedding applications.
- All Qhorus config is clearly namespaced; no collision risk.
- `InMemory*Store` alternatives in the testing module are unaffected — they bypass
  the persistence unit entirely.

**Negative:**
- Ledger base tables currently reside in the "qhorus" datasource, not a
  separate "ledger" datasource.
- Breaking change for consumers: any `quarkus.datasource.*` config must be
  renamed to `quarkus.datasource.qhorus.*`. No external users exist at time
  of decision.

## Embedding App Migration

Replace `quarkus.datasource.*` with `quarkus.datasource.qhorus.*`:

```properties
# Before
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost/mydb

# After
quarkus.datasource.qhorus.db-kind=postgresql
quarkus.datasource.qhorus.jdbc.url=jdbc:postgresql://localhost/mydb
```
```

- [ ] **Step 2: Add entry to `adr/INDEX.md`**

Add the new entry to the index. Look at the existing format in `adr/INDEX.md` and add:
```
| [0004](0004-named-datasource-isolation.md) | Named Datasource Isolation: quarkus.datasource.qhorus | 2026-04-22 | Accepted |
```

- [ ] **Step 3: Commit**

```bash
git add adr/0004-named-datasource-isolation.md adr/INDEX.md
git commit -m "docs(adr): ADR-0004 named datasource isolation + ledger coupling revisit marker

Closes #82"
```

---

## Self-Review Checklist

- [x] **#83 — MessageStore SPI:** Two new methods in both interfaces; all four impls; three bypasses closed; contract tests added.
- [x] **#84 — Named datasource:** Main properties: `quarkus.hibernate-orm.qhorus.packages` + Flyway. Test properties: renamed keys + `migrate-at-start=false`. `AgentMessageLedgerEntryRepository` PU qualifier.
- [x] **ADR-0004:** Ledger coupling documented with explicit revisit trigger.
- [x] **No placeholders:** All code blocks are complete.
- [x] **Type consistency:** `MessageType` used in interface matches `MessageType.EVENT` at call sites. `Map<UUID, Long>` consistent throughout.
- [x] **Reactive store @Disabled:** `ReactiveJpaMessageStore` uses in-memory aggregate approach — acceptable for `@Disabled` status.
- [x] **CLAUDE.md note on `mvn clean`:** Step 4 of Task 7 explicitly calls `mvn clean test` — required for new properties to take effect.
