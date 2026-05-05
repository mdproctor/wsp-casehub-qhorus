# Issue #74 — Reactive Store Layer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `Reactive*Store` interfaces (5 domains), `ReactiveJpa*Store` implementations, and `InMemoryReactive*Store` test helpers — the complete reactive store layer for the dual-stack design.

**Architecture:** Reactive interfaces mirror blocking `*Store` but return `Uni<T>`. `ReactiveJpa*Store` beans are `@Alternative` and use `PanacheRepositoryBase` helpers + `@WithTransaction`. `InMemoryReactive*Store` wraps `InMemory*Store` via `Uni.createFrom().item(...)` with no logic duplication.

**Tech Stack:** Java 21, Quarkus 3.32.2, quarkus-hibernate-reactive-panache, SmallRye Mutiny, JUnit 5, AssertJ

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl runtime,testing -am`

---

## File Map

**New in `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/`:**
- `ReactiveChannelStore.java`, `ReactiveMessageStore.java`, `ReactiveInstanceStore.java`, `ReactiveDataStore.java`, `ReactiveWatchdogStore.java`

**New in `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/`:**
- `ChannelReactivePanacheRepo.java`, `MessageReactivePanacheRepo.java`, `InstanceReactivePanacheRepo.java`, `CapabilityReactivePanacheRepo.java`, `SharedDataReactivePanacheRepo.java`, `ArtefactClaimReactivePanacheRepo.java`, `WatchdogReactivePanacheRepo.java`
- `ReactiveJpaChannelStore.java`, `ReactiveJpaMessageStore.java`, `ReactiveJpaInstanceStore.java`, `ReactiveJpaDataStore.java`, `ReactiveJpaWatchdogStore.java`

**New in `testing/src/main/java/io/quarkiverse/qhorus/testing/`:**
- `InMemoryReactiveChannelStore.java`, `InMemoryReactiveMessageStore.java`, `InMemoryReactiveInstanceStore.java`, `InMemoryReactiveDataStore.java`, `InMemoryReactiveWatchdogStore.java`

**New in `testing/src/test/java/io/quarkiverse/qhorus/testing/`:**
- `InMemoryReactiveChannelStoreTest.java`, `InMemoryReactiveMessageStoreTest.java`, `InMemoryReactiveInstanceStoreTest.java`, `InMemoryReactiveDataStoreTest.java`, `InMemoryReactiveWatchdogStoreTest.java`

**New in `runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/`:**
- `ReactiveStoreTestProfile.java`, `ReactiveJpaChannelStoreTest.java`, `ReactiveJpaMessageStoreTest.java`, `ReactiveJpaInstanceStoreTest.java`, `ReactiveJpaDataStoreTest.java`, `ReactiveJpaWatchdogStoreTest.java`

**Modified:**
- `runtime/pom.xml` — add `vertx-jdbc-client` test dep

---

## Task 1: Five Reactive*Store Interfaces

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveChannelStore.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveMessageStore.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveInstanceStore.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveDataStore.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveWatchdogStore.java`

- [ ] **Step 1: Write ReactiveChannelStore**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveChannelStore.java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;
import io.smallrye.mutiny.Uni;

public interface ReactiveChannelStore {
    Uni<Channel> put(Channel channel);
    Uni<Optional<Channel>> find(UUID id);
    Uni<Optional<Channel>> findByName(String name);
    Uni<List<Channel>> scan(ChannelQuery query);
    Uni<Void> delete(UUID id);
}
```

- [ ] **Step 2: Write ReactiveMessageStore**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveMessageStore.java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.smallrye.mutiny.Uni;

public interface ReactiveMessageStore {
    Uni<Message> put(Message message);
    Uni<Optional<Message>> find(Long id);
    Uni<List<Message>> scan(MessageQuery query);
    Uni<Void> deleteAll(UUID channelId);
    Uni<Void> delete(Long id);
    Uni<Integer> countByChannel(UUID channelId);
}
```

- [ ] **Step 3: Write ReactiveInstanceStore**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveInstanceStore.java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;
import io.smallrye.mutiny.Uni;

public interface ReactiveInstanceStore {
    Uni<Instance> put(Instance instance);
    Uni<Optional<Instance>> find(UUID id);
    Uni<Optional<Instance>> findByInstanceId(String instanceId);
    Uni<List<Instance>> scan(InstanceQuery query);
    Uni<Void> putCapabilities(UUID instanceId, List<String> tags);
    Uni<Void> deleteCapabilities(UUID instanceId);
    Uni<List<String>> findCapabilities(UUID instanceId);
    Uni<Void> delete(UUID id);
}
```

- [ ] **Step 4: Write ReactiveDataStore**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveDataStore.java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.query.DataQuery;
import io.smallrye.mutiny.Uni;

public interface ReactiveDataStore {
    Uni<SharedData> put(SharedData data);
    Uni<Optional<SharedData>> find(UUID id);
    Uni<Optional<SharedData>> findByKey(String key);
    Uni<List<SharedData>> scan(DataQuery query);
    Uni<ArtefactClaim> putClaim(ArtefactClaim claim);
    Uni<Void> deleteClaim(UUID artefactId, UUID instanceId);
    Uni<Integer> countClaims(UUID artefactId);
    Uni<Void> delete(UUID id);
}
```

- [ ] **Step 5: Write ReactiveWatchdogStore**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/ReactiveWatchdogStore.java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;
import io.smallrye.mutiny.Uni;

public interface ReactiveWatchdogStore {
    Uni<Watchdog> put(Watchdog watchdog);
    Uni<Optional<Watchdog>> find(UUID id);
    Uni<List<Watchdog>> scan(WatchdogQuery query);
    Uni<Void> delete(UUID id);
}
```

- [ ] **Step 6: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/Reactive*.java
git commit -m "feat(store): Reactive*Store interfaces for all 5 domains

Refs #74"
```

---

## Task 2: Seven Panache Repo Helpers

These are package-private `@Alternative @ApplicationScoped` adapters that expose Hibernate Reactive Panache operations. They are `@Alternative` so they only activate when selected via `quarkus.arc.selected-alternatives` — this prevents Hibernate Reactive from booting in blocking-only deployments.

**Files** (all in `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/`):
- Create: `ChannelReactivePanacheRepo.java`
- Create: `MessageReactivePanacheRepo.java`
- Create: `InstanceReactivePanacheRepo.java`
- Create: `CapabilityReactivePanacheRepo.java`
- Create: `SharedDataReactivePanacheRepo.java`
- Create: `ArtefactClaimReactivePanacheRepo.java`
- Create: `WatchdogReactivePanacheRepo.java`

- [ ] **Step 1: Write all seven repo helpers**

```java
// ChannelReactivePanacheRepo.java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

@Alternative
@ApplicationScoped
class ChannelReactivePanacheRepo implements PanacheRepositoryBase<Channel, UUID> {
}
```

```java
// MessageReactivePanacheRepo.java
package io.quarkiverse.qhorus.runtime.store.jpa;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

@Alternative
@ApplicationScoped
class MessageReactivePanacheRepo implements PanacheRepositoryBase<Message, Long> {
}
```

```java
// InstanceReactivePanacheRepo.java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

@Alternative
@ApplicationScoped
class InstanceReactivePanacheRepo implements PanacheRepositoryBase<Instance, UUID> {
}
```

```java
// CapabilityReactivePanacheRepo.java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.quarkiverse.qhorus.runtime.instance.Capability;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

@Alternative
@ApplicationScoped
class CapabilityReactivePanacheRepo implements PanacheRepositoryBase<Capability, UUID> {
}
```

```java
// SharedDataReactivePanacheRepo.java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

@Alternative
@ApplicationScoped
class SharedDataReactivePanacheRepo implements PanacheRepositoryBase<SharedData, UUID> {
}
```

```java
// ArtefactClaimReactivePanacheRepo.java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

@Alternative
@ApplicationScoped
class ArtefactClaimReactivePanacheRepo implements PanacheRepositoryBase<ArtefactClaim, UUID> {
}
```

```java
// WatchdogReactivePanacheRepo.java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

@Alternative
@ApplicationScoped
class WatchdogReactivePanacheRepo implements PanacheRepositoryBase<Watchdog, UUID> {
}
```

- [ ] **Step 2: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/*ReactivePanacheRepo.java
git commit -m "feat(store): reactive Panache repo helpers for all 7 entity types

Refs #74"
```

---

## Task 3: ReactiveJpaChannelStore and ReactiveJpaMessageStore

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaChannelStore.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java`

- [ ] **Step 1: Write ReactiveJpaChannelStore**

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.store.ReactiveChannelStore;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;
import io.quarkus.hibernate.reactive.panache.common.WithTransaction;
import io.smallrye.mutiny.Uni;

@Alternative
@ApplicationScoped
public class ReactiveJpaChannelStore implements ReactiveChannelStore {

    @Inject
    ChannelReactivePanacheRepo repo;

    @Override
    @WithTransaction
    public Uni<Channel> put(Channel channel) {
        return repo.persist(channel);
    }

    @Override
    public Uni<Optional<Channel>> find(UUID id) {
        return repo.findById(id).map(Optional::ofNullable);
    }

    @Override
    public Uni<Optional<Channel>> findByName(String name) {
        return repo.find("name", name).firstResult().map(Optional::ofNullable);
    }

    @Override
    public Uni<List<Channel>> scan(ChannelQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Channel WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

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

        return repo.list(jpql.toString(), params.toArray());
    }

    @Override
    @WithTransaction
    public Uni<Void> delete(UUID id) {
        return repo.deleteById(id).replaceWithVoid();
    }
}
```

- [ ] **Step 2: Write ReactiveJpaMessageStore**

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.store.ReactiveMessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.hibernate.reactive.panache.common.WithTransaction;
import io.smallrye.mutiny.Uni;

@Alternative
@ApplicationScoped
public class ReactiveJpaMessageStore implements ReactiveMessageStore {

    @Inject
    MessageReactivePanacheRepo repo;

    @Override
    @WithTransaction
    public Uni<Message> put(Message message) {
        return repo.persist(message);
    }

    @Override
    public Uni<Optional<Message>> find(Long id) {
        return repo.findById(id).map(Optional::ofNullable);
    }

    @Override
    public Uni<List<Message>> scan(MessageQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Message WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.channelId() != null) {
            jpql.append(" AND channelId = ?").append(idx++);
            params.add(q.channelId());
        }
        if (q.afterId() != null) {
            jpql.append(" AND id > ?").append(idx++);
            params.add(q.afterId());
        }
        if (q.sender() != null) {
            jpql.append(" AND sender = ?").append(idx++);
            params.add(q.sender());
        }
        if (q.target() != null) {
            jpql.append(" AND target = ?").append(idx++);
            params.add(q.target());
        }
        if (q.inReplyTo() != null) {
            jpql.append(" AND inReplyTo = ?").append(idx++);
            params.add(q.inReplyTo());
        }
        if (q.excludeTypes() != null && !q.excludeTypes().isEmpty()) {
            jpql.append(" AND messageType NOT IN ?").append(idx++);
            params.add(q.excludeTypes());
        }
        if (q.contentPattern() != null) {
            jpql.append(" AND LOWER(content) LIKE ?").append(idx++);
            params.add("%" + q.contentPattern().toLowerCase() + "%");
        }
        jpql.append(" ORDER BY id ASC");

        return repo.list(jpql.toString(), params.toArray())
                .map(results -> q.limit() != null && results.size() > q.limit()
                        ? results.subList(0, q.limit())
                        : results);
    }

    @Override
    @WithTransaction
    public Uni<Void> deleteAll(UUID channelId) {
        return repo.delete("channelId", channelId).replaceWithVoid();
    }

    @Override
    @WithTransaction
    public Uni<Void> delete(Long id) {
        return repo.deleteById(id).replaceWithVoid();
    }

    @Override
    public Uni<Integer> countByChannel(UUID channelId) {
        return repo.count("channelId", channelId).map(Long::intValue);
    }
}
```

- [ ] **Step 3: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaChannelStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java
git commit -m "feat(store): ReactiveJpaChannelStore + ReactiveJpaMessageStore

Refs #74"
```

---

## Task 4: ReactiveJpaInstanceStore, ReactiveJpaDataStore, ReactiveJpaWatchdogStore

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaInstanceStore.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaDataStore.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaWatchdogStore.java`

- [ ] **Step 1: Write ReactiveJpaInstanceStore**

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.quarkiverse.qhorus.runtime.instance.Capability;
import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.ReactiveInstanceStore;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;
import io.quarkus.hibernate.reactive.panache.common.WithTransaction;
import io.smallrye.mutiny.Uni;

@Alternative
@ApplicationScoped
public class ReactiveJpaInstanceStore implements ReactiveInstanceStore {

    @Inject
    InstanceReactivePanacheRepo instanceRepo;

    @Inject
    CapabilityReactivePanacheRepo capRepo;

    @Override
    @WithTransaction
    public Uni<Instance> put(Instance instance) {
        return instanceRepo.persist(instance);
    }

    @Override
    public Uni<Optional<Instance>> find(UUID id) {
        return instanceRepo.findById(id).map(Optional::ofNullable);
    }

    @Override
    public Uni<Optional<Instance>> findByInstanceId(String instanceId) {
        return instanceRepo.find("instanceId", instanceId).firstResult().map(Optional::ofNullable);
    }

    @Override
    public Uni<List<Instance>> scan(InstanceQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Instance WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.status() != null) {
            jpql.append(" AND status = ?").append(idx++);
            params.add(q.status());
        }
        if (q.staleOlderThan() != null) {
            jpql.append(" AND lastSeen < ?").append(idx++);
            params.add(q.staleOlderThan());
        }
        if (q.capability() != null) {
            jpql.append(" AND id IN (SELECT c.instanceId FROM Capability c WHERE c.tag = ?").append(idx++).append(")");
            params.add(q.capability());
        }

        return instanceRepo.list(jpql.toString(), params.toArray());
    }

    @Override
    @WithTransaction
    public Uni<Void> putCapabilities(UUID instanceId, List<String> tags) {
        return capRepo.delete("instanceId", instanceId)
                .flatMap(ignored -> {
                    List<Uni<Capability>> persists = tags.stream()
                            .map(tag -> {
                                Capability c = new Capability();
                                c.instanceId = instanceId;
                                c.tag = tag;
                                return capRepo.persist(c);
                            })
                            .toList();
                    return Uni.join().all(persists).andCollectFailures();
                })
                .replaceWithVoid();
    }

    @Override
    @WithTransaction
    public Uni<Void> deleteCapabilities(UUID instanceId) {
        return capRepo.delete("instanceId", instanceId).replaceWithVoid();
    }

    @Override
    public Uni<List<String>> findCapabilities(UUID instanceId) {
        return capRepo.list("instanceId", instanceId)
                .map(caps -> caps.stream().map(c -> c.tag).toList());
    }

    @Override
    @WithTransaction
    public Uni<Void> delete(UUID id) {
        return capRepo.delete("instanceId", id)
                .flatMap(ignored -> instanceRepo.deleteById(id))
                .replaceWithVoid();
    }
}
```

- [ ] **Step 2: Write ReactiveJpaDataStore**

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.ReactiveDataStore;
import io.quarkiverse.qhorus.runtime.store.query.DataQuery;
import io.quarkus.hibernate.reactive.panache.common.WithTransaction;
import io.smallrye.mutiny.Uni;

@Alternative
@ApplicationScoped
public class ReactiveJpaDataStore implements ReactiveDataStore {

    @Inject
    SharedDataReactivePanacheRepo dataRepo;

    @Inject
    ArtefactClaimReactivePanacheRepo claimRepo;

    @Override
    @WithTransaction
    public Uni<SharedData> put(SharedData data) {
        return dataRepo.persist(data);
    }

    @Override
    public Uni<Optional<SharedData>> find(UUID id) {
        return dataRepo.findById(id).map(Optional::ofNullable);
    }

    @Override
    public Uni<Optional<SharedData>> findByKey(String key) {
        return dataRepo.find("key", key).firstResult().map(Optional::ofNullable);
    }

    @Override
    public Uni<List<SharedData>> scan(DataQuery q) {
        StringBuilder jpql = new StringBuilder("FROM SharedData WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.createdBy() != null) {
            jpql.append(" AND createdBy = ?").append(idx++);
            params.add(q.createdBy());
        }
        if (q.complete() != null) {
            jpql.append(" AND complete = ?").append(idx++);
            params.add(q.complete());
        }

        return dataRepo.list(jpql.toString(), params.toArray());
    }

    @Override
    @WithTransaction
    public Uni<ArtefactClaim> putClaim(ArtefactClaim claim) {
        return claimRepo.persist(claim);
    }

    @Override
    @WithTransaction
    public Uni<Void> deleteClaim(UUID artefactId, UUID instanceId) {
        return claimRepo.delete("artefactId = ?1 AND instanceId = ?2", artefactId, instanceId)
                .replaceWithVoid();
    }

    @Override
    public Uni<Integer> countClaims(UUID artefactId) {
        return claimRepo.count("artefactId", artefactId).map(Long::intValue);
    }

    @Override
    @WithTransaction
    public Uni<Void> delete(UUID id) {
        return claimRepo.delete("artefactId", id)
                .flatMap(ignored -> dataRepo.deleteById(id))
                .replaceWithVoid();
    }
}
```

- [ ] **Step 3: Write ReactiveJpaWatchdogStore**

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.quarkiverse.qhorus.runtime.store.ReactiveWatchdogStore;
import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;
import io.quarkus.hibernate.reactive.panache.common.WithTransaction;
import io.smallrye.mutiny.Uni;

@Alternative
@ApplicationScoped
public class ReactiveJpaWatchdogStore implements ReactiveWatchdogStore {

    @Inject
    WatchdogReactivePanacheRepo repo;

    @Override
    @WithTransaction
    public Uni<Watchdog> put(Watchdog watchdog) {
        return repo.persist(watchdog);
    }

    @Override
    public Uni<Optional<Watchdog>> find(UUID id) {
        return repo.findById(id).map(Optional::ofNullable);
    }

    @Override
    public Uni<List<Watchdog>> scan(WatchdogQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Watchdog WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.conditionType() != null) {
            jpql.append(" AND conditionType = ?").append(idx++);
            params.add(q.conditionType());
        }

        return repo.list(jpql.toString(), params.toArray());
    }

    @Override
    @WithTransaction
    public Uni<Void> delete(UUID id) {
        return repo.deleteById(id).replaceWithVoid();
    }
}
```

- [ ] **Step 4: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Existing tests still green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`, 717 tests passing (or more if reactive tests were already added).

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaInstanceStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaDataStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/ReactiveJpaWatchdogStore.java
git commit -m "feat(store): ReactiveJpaInstanceStore + ReactiveJpaDataStore + ReactiveJpaWatchdogStore

Refs #74"
```

---

## Task 5: InMemoryReactive*Store + Unit Tests

`InMemoryReactive*Store` holds an internal `InMemory*Store` delegate. No CDI injection — the delegate is created by the default constructor, making the class directly instantiable in unit tests. The `@Inject` constructor variant makes it injectable in Quarkus integration test contexts.

**Files:**
- Create: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactiveChannelStore.java`
- Create: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactiveMessageStore.java`
- Create: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactiveInstanceStore.java`
- Create: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactiveDataStore.java`
- Create: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactiveWatchdogStore.java`
- Create: `testing/src/test/java/io/quarkiverse/qhorus/testing/InMemoryReactiveChannelStoreTest.java`
- Create: `testing/src/test/java/io/quarkiverse/qhorus/testing/InMemoryReactiveMessageStoreTest.java`
- Create: `testing/src/test/java/io/quarkiverse/qhorus/testing/InMemoryReactiveInstanceStoreTest.java`
- Create: `testing/src/test/java/io/quarkiverse/qhorus/testing/InMemoryReactiveDataStoreTest.java`
- Create: `testing/src/test/java/io/quarkiverse/qhorus/testing/InMemoryReactiveWatchdogStoreTest.java`

- [ ] **Step 1: Write InMemoryReactiveChannelStore**

```java
package io.quarkiverse.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.store.ReactiveChannelStore;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveChannelStore implements ReactiveChannelStore {

    private final InMemoryChannelStore delegate = new InMemoryChannelStore();

    @Override
    public Uni<Channel> put(Channel channel) {
        return Uni.createFrom().item(() -> delegate.put(channel));
    }

    @Override
    public Uni<Optional<Channel>> find(UUID id) {
        return Uni.createFrom().item(() -> delegate.find(id));
    }

    @Override
    public Uni<Optional<Channel>> findByName(String name) {
        return Uni.createFrom().item(() -> delegate.findByName(name));
    }

    @Override
    public Uni<List<Channel>> scan(ChannelQuery query) {
        return Uni.createFrom().item(() -> delegate.scan(query));
    }

    @Override
    public Uni<Void> delete(UUID id) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.delete(id));
    }

    public void clear() {
        delegate.clear();
    }
}
```

- [ ] **Step 2: Write InMemoryReactiveMessageStore**

```java
package io.quarkiverse.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.store.ReactiveMessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveMessageStore implements ReactiveMessageStore {

    private final InMemoryMessageStore delegate = new InMemoryMessageStore();

    @Override
    public Uni<Message> put(Message message) {
        return Uni.createFrom().item(() -> delegate.put(message));
    }

    @Override
    public Uni<Optional<Message>> find(Long id) {
        return Uni.createFrom().item(() -> delegate.find(id));
    }

    @Override
    public Uni<List<Message>> scan(MessageQuery query) {
        return Uni.createFrom().item(() -> delegate.scan(query));
    }

    @Override
    public Uni<Void> deleteAll(UUID channelId) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.deleteAll(channelId));
    }

    @Override
    public Uni<Void> delete(Long id) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.delete(id));
    }

    @Override
    public Uni<Integer> countByChannel(UUID channelId) {
        return Uni.createFrom().item(() -> delegate.countByChannel(channelId));
    }

    public void clear() {
        delegate.clear();
    }
}
```

- [ ] **Step 3: Write InMemoryReactiveInstanceStore**

```java
package io.quarkiverse.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.ReactiveInstanceStore;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveInstanceStore implements ReactiveInstanceStore {

    private final InMemoryInstanceStore delegate = new InMemoryInstanceStore();

    @Override
    public Uni<Instance> put(Instance instance) {
        return Uni.createFrom().item(() -> delegate.put(instance));
    }

    @Override
    public Uni<Optional<Instance>> find(UUID id) {
        return Uni.createFrom().item(() -> delegate.find(id));
    }

    @Override
    public Uni<Optional<Instance>> findByInstanceId(String instanceId) {
        return Uni.createFrom().item(() -> delegate.findByInstanceId(instanceId));
    }

    @Override
    public Uni<List<Instance>> scan(InstanceQuery query) {
        return Uni.createFrom().item(() -> delegate.scan(query));
    }

    @Override
    public Uni<Void> putCapabilities(UUID instanceId, List<String> tags) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.putCapabilities(instanceId, tags));
    }

    @Override
    public Uni<Void> deleteCapabilities(UUID instanceId) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.deleteCapabilities(instanceId));
    }

    @Override
    public Uni<List<String>> findCapabilities(UUID instanceId) {
        return Uni.createFrom().item(() -> delegate.findCapabilities(instanceId));
    }

    @Override
    public Uni<Void> delete(UUID id) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.delete(id));
    }

    public void clear() {
        delegate.clear();
    }
}
```

- [ ] **Step 4: Write InMemoryReactiveDataStore**

```java
package io.quarkiverse.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.ReactiveDataStore;
import io.quarkiverse.qhorus.runtime.store.query.DataQuery;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveDataStore implements ReactiveDataStore {

    private final InMemoryDataStore delegate = new InMemoryDataStore();

    @Override
    public Uni<SharedData> put(SharedData data) {
        return Uni.createFrom().item(() -> delegate.put(data));
    }

    @Override
    public Uni<Optional<SharedData>> find(UUID id) {
        return Uni.createFrom().item(() -> delegate.find(id));
    }

    @Override
    public Uni<Optional<SharedData>> findByKey(String key) {
        return Uni.createFrom().item(() -> delegate.findByKey(key));
    }

    @Override
    public Uni<List<SharedData>> scan(DataQuery query) {
        return Uni.createFrom().item(() -> delegate.scan(query));
    }

    @Override
    public Uni<ArtefactClaim> putClaim(ArtefactClaim claim) {
        return Uni.createFrom().item(() -> delegate.putClaim(claim));
    }

    @Override
    public Uni<Void> deleteClaim(UUID artefactId, UUID instanceId) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.deleteClaim(artefactId, instanceId));
    }

    @Override
    public Uni<Integer> countClaims(UUID artefactId) {
        return Uni.createFrom().item(() -> delegate.countClaims(artefactId));
    }

    @Override
    public Uni<Void> delete(UUID id) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.delete(id));
    }

    public void clear() {
        delegate.clear();
    }
}
```

- [ ] **Step 5: Write InMemoryReactiveWatchdogStore**

```java
package io.quarkiverse.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkiverse.qhorus.runtime.store.ReactiveWatchdogStore;
import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveWatchdogStore implements ReactiveWatchdogStore {

    private final InMemoryWatchdogStore delegate = new InMemoryWatchdogStore();

    @Override
    public Uni<Watchdog> put(Watchdog watchdog) {
        return Uni.createFrom().item(() -> delegate.put(watchdog));
    }

    @Override
    public Uni<Optional<Watchdog>> find(UUID id) {
        return Uni.createFrom().item(() -> delegate.find(id));
    }

    @Override
    public Uni<List<Watchdog>> scan(WatchdogQuery query) {
        return Uni.createFrom().item(() -> delegate.scan(query));
    }

    @Override
    public Uni<Void> delete(UUID id) {
        return Uni.createFrom().voidItem().invoke(() -> delegate.delete(id));
    }

    public void clear() {
        delegate.clear();
    }
}
```

- [ ] **Step 6: Write InMemoryReactiveChannelStoreTest**

No Quarkus needed — instantiate directly, `.await().indefinitely()` to unwrap.

```java
package io.quarkiverse.qhorus.testing;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;

class InMemoryReactiveChannelStoreTest {

    private final InMemoryReactiveChannelStore store = new InMemoryReactiveChannelStore();

    @BeforeEach
    void reset() {
        store.clear();
    }

    @Test
    void put_assignsIdAndReturns() {
        Channel ch = channel("rx-put-" + UUID.randomUUID(), ChannelSemantic.APPEND);

        Channel saved = store.put(ch).await().indefinitely();

        assertThat(saved.id).isNotNull();
        assertThat(saved.name).isEqualTo(ch.name);
    }

    @Test
    void find_returnsEmpty_whenNotFound() {
        var result = store.find(UUID.randomUUID()).await().indefinitely();
        assertThat(result).isEmpty();
    }

    @Test
    void find_returnsChannel_whenExists() {
        Channel ch = channel("rx-find-" + UUID.randomUUID(), ChannelSemantic.COLLECT);
        store.put(ch).await().indefinitely();

        var found = store.find(ch.id).await().indefinitely();

        assertThat(found).isPresent();
        assertThat(found.get().semantic).isEqualTo(ChannelSemantic.COLLECT);
    }

    @Test
    void findByName_returnsChannel_whenExists() {
        Channel ch = channel("named-" + UUID.randomUUID(), ChannelSemantic.APPEND);
        store.put(ch).await().indefinitely();

        var found = store.findByName(ch.name).await().indefinitely();

        assertThat(found).isPresent();
    }

    @Test
    void scan_byPaused_returnsOnlyPaused() {
        Channel active = channel("active-" + UUID.randomUUID(), ChannelSemantic.APPEND);
        Channel paused = channel("paused-" + UUID.randomUUID(), ChannelSemantic.APPEND);
        paused.paused = true;
        store.put(active).await().indefinitely();
        store.put(paused).await().indefinitely();

        List<Channel> results = store.scan(ChannelQuery.pausedOnly()).await().indefinitely();

        assertThat(results).anyMatch(c -> c.name.equals(paused.name));
        assertThat(results).noneMatch(c -> c.name.equals(active.name));
    }

    @Test
    void delete_removesChannel() {
        Channel ch = channel("rx-del-" + UUID.randomUUID(), ChannelSemantic.APPEND);
        store.put(ch).await().indefinitely();

        store.delete(ch.id).await().indefinitely();

        assertThat(store.find(ch.id).await().indefinitely()).isEmpty();
    }

    private Channel channel(String name, ChannelSemantic semantic) {
        Channel ch = new Channel();
        ch.name = name;
        ch.semantic = semantic;
        return ch;
    }
}
```

- [ ] **Step 7: Write InMemoryReactiveMessageStoreTest**

```java
package io.quarkiverse.qhorus.testing;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;

class InMemoryReactiveMessageStoreTest {

    private final InMemoryReactiveMessageStore store = new InMemoryReactiveMessageStore();

    @BeforeEach
    void reset() {
        store.clear();
    }

    @Test
    void put_assignsIdAndReturns() {
        Message m = message(UUID.randomUUID(), "alice");
        Message saved = store.put(m).await().indefinitely();
        assertThat(saved.id).isNotNull();
    }

    @Test
    void find_returnsEmpty_whenNotFound() {
        assertThat(store.find(999L).await().indefinitely()).isEmpty();
    }

    @Test
    void scan_byChannel_returnsMatchingMessages() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        store.put(message(ch1, "alice")).await().indefinitely();
        store.put(message(ch2, "bob")).await().indefinitely();

        List<Message> results = store.scan(MessageQuery.forChannel(ch1)).await().indefinitely();

        assertThat(results).hasSize(1);
        assertThat(results.get(0).sender).isEqualTo("alice");
    }

    @Test
    void deleteAll_removesAllForChannel() {
        UUID ch = UUID.randomUUID();
        store.put(message(ch, "a")).await().indefinitely();
        store.put(message(ch, "b")).await().indefinitely();

        store.deleteAll(ch).await().indefinitely();

        assertThat(store.countByChannel(ch).await().indefinitely()).isZero();
    }

    @Test
    void countByChannel_countsCorrectly() {
        UUID ch = UUID.randomUUID();
        store.put(message(ch, "x")).await().indefinitely();
        store.put(message(ch, "y")).await().indefinitely();

        assertThat(store.countByChannel(ch).await().indefinitely()).isEqualTo(2);
    }

    private Message message(UUID channelId, String sender) {
        Message m = new Message();
        m.channelId = channelId;
        m.sender = sender;
        m.messageType = MessageType.request;
        m.content = "hello";
        return m;
    }
}
```

- [ ] **Step 8: Write InMemoryReactiveInstanceStoreTest**

```java
package io.quarkiverse.qhorus.testing;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;

class InMemoryReactiveInstanceStoreTest {

    private final InMemoryReactiveInstanceStore store = new InMemoryReactiveInstanceStore();

    @BeforeEach
    void reset() {
        store.clear();
    }

    @Test
    void put_assignsIdAndReturns() {
        Instance inst = instance("agent-" + UUID.randomUUID());
        Instance saved = store.put(inst).await().indefinitely();
        assertThat(saved.id).isNotNull();
    }

    @Test
    void findByInstanceId_returnsEmpty_whenNotFound() {
        assertThat(store.findByInstanceId("missing").await().indefinitely()).isEmpty();
    }

    @Test
    void putCapabilities_andFindCapabilities() {
        Instance inst = instance("cap-agent-" + UUID.randomUUID());
        store.put(inst).await().indefinitely();

        store.putCapabilities(inst.id, List.of("code-review", "planning")).await().indefinitely();

        List<String> caps = store.findCapabilities(inst.id).await().indefinitely();
        assertThat(caps).containsExactlyInAnyOrder("code-review", "planning");
    }

    @Test
    void deleteCapabilities_clearsAll() {
        Instance inst = instance("dc-agent-" + UUID.randomUUID());
        store.put(inst).await().indefinitely();
        store.putCapabilities(inst.id, List.of("a", "b")).await().indefinitely();

        store.deleteCapabilities(inst.id).await().indefinitely();

        assertThat(store.findCapabilities(inst.id).await().indefinitely()).isEmpty();
    }

    @Test
    void scan_byCapability_returnsMatchingInstances() {
        Instance a = instance("a-" + UUID.randomUUID());
        Instance b = instance("b-" + UUID.randomUUID());
        store.put(a).await().indefinitely();
        store.put(b).await().indefinitely();
        store.putCapabilities(a.id, List.of("search")).await().indefinitely();

        List<Instance> results = store.scan(InstanceQuery.byCapability("search")).await().indefinitely();

        assertThat(results).hasSize(1);
        assertThat(results.get(0).instanceId).isEqualTo(a.instanceId);
    }

    private Instance instance(String instanceId) {
        Instance i = new Instance();
        i.instanceId = instanceId;
        i.status = "online";
        return i;
    }
}
```

- [ ] **Step 9: Write InMemoryReactiveDataStoreTest**

```java
package io.quarkiverse.qhorus.testing;

import static org.assertj.core.api.Assertions.*;

import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;

class InMemoryReactiveDataStoreTest {

    private final InMemoryReactiveDataStore store = new InMemoryReactiveDataStore();

    @BeforeEach
    void reset() {
        store.clear();
    }

    @Test
    void put_assignsIdAndReturns() {
        SharedData data = sharedData("key-" + UUID.randomUUID());
        SharedData saved = store.put(data).await().indefinitely();
        assertThat(saved.id).isNotNull();
    }

    @Test
    void findByKey_returnsEmpty_whenNotFound() {
        assertThat(store.findByKey("missing").await().indefinitely()).isEmpty();
    }

    @Test
    void findByKey_returnsData_whenExists() {
        SharedData data = sharedData("lookup-" + UUID.randomUUID());
        store.put(data).await().indefinitely();

        assertThat(store.findByKey(data.key).await().indefinitely()).isPresent();
    }

    @Test
    void putClaim_andCountClaims() {
        SharedData data = sharedData("claim-key-" + UUID.randomUUID());
        store.put(data).await().indefinitely();

        UUID instanceId = UUID.randomUUID();
        ArtefactClaim claim = new ArtefactClaim();
        claim.artefactId = data.id;
        claim.instanceId = instanceId;
        store.putClaim(claim).await().indefinitely();

        assertThat(store.countClaims(data.id).await().indefinitely()).isEqualTo(1);
    }

    @Test
    void deleteClaim_reduceCount() {
        SharedData data = sharedData("dc-key-" + UUID.randomUUID());
        store.put(data).await().indefinitely();

        UUID inst1 = UUID.randomUUID();
        ArtefactClaim c = new ArtefactClaim();
        c.artefactId = data.id;
        c.instanceId = inst1;
        store.putClaim(c).await().indefinitely();

        store.deleteClaim(data.id, inst1).await().indefinitely();

        assertThat(store.countClaims(data.id).await().indefinitely()).isZero();
    }

    private SharedData sharedData(String key) {
        SharedData d = new SharedData();
        d.key = key;
        d.createdBy = "test-agent";
        return d;
    }
}
```

- [ ] **Step 10: Write InMemoryReactiveWatchdogStoreTest**

```java
package io.quarkiverse.qhorus.testing;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;

class InMemoryReactiveWatchdogStoreTest {

    private final InMemoryReactiveWatchdogStore store = new InMemoryReactiveWatchdogStore();

    @BeforeEach
    void reset() {
        store.clear();
    }

    @Test
    void put_assignsIdAndReturns() {
        Watchdog w = watchdog("threshold");
        Watchdog saved = store.put(w).await().indefinitely();
        assertThat(saved.id).isNotNull();
    }

    @Test
    void find_returnsEmpty_whenNotFound() {
        assertThat(store.find(UUID.randomUUID()).await().indefinitely()).isEmpty();
    }

    @Test
    void scan_byConditionType_returnsMatching() {
        store.put(watchdog("threshold")).await().indefinitely();
        store.put(watchdog("pattern")).await().indefinitely();

        List<Watchdog> results = store.scan(WatchdogQuery.byConditionType("threshold")).await().indefinitely();

        assertThat(results).hasSize(1);
        assertThat(results.get(0).conditionType).isEqualTo("threshold");
    }

    @Test
    void delete_removesWatchdog() {
        Watchdog w = watchdog("del-type");
        store.put(w).await().indefinitely();

        store.delete(w.id).await().indefinitely();

        assertThat(store.find(w.id).await().indefinitely()).isEmpty();
    }

    private Watchdog watchdog(String conditionType) {
        Watchdog w = new Watchdog();
        w.conditionType = conditionType;
        w.targetName = "test-target";
        w.notificationChannel = "test-alerts";
        return w;
    }
}
```

- [ ] **Step 11: Build and run tests (testing module)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -q 2>&1 | tail -10
```
Expected: BUILD SUCCESS, 5 new test classes passing.

- [ ] **Step 12: Commit**

```bash
git add testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryReactive*.java \
        testing/src/test/java/io/quarkiverse/qhorus/testing/InMemoryReactive*StoreTest.java
git commit -m "feat(testing): InMemoryReactive*Store wrappers + unit tests (all 5 domains)

Refs #74"
```

---

## Task 6: ReactiveJpa Integration Tests

These tests start a full Quarkus context with the reactive datasource enabled and the `ReactiveJpa*Store` alternatives selected. The test profile overrides `quarkus.datasource.reactive=false` (set in test `application.properties`) and activates the reactive beans via `quarkus.arc.selected-alternatives`.

**Step 0: Add vertx-jdbc-client test dependency to runtime/pom.xml.** This provides the Vert.x JDBC bridge that backs reactive H2 in tests.

**Files:**
- Modify: `runtime/pom.xml`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/ReactiveStoreTestProfile.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/ReactiveJpaChannelStoreTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/ReactiveJpaMessageStoreTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/ReactiveJpaInstanceStoreTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/ReactiveJpaDataStoreTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/ReactiveJpaWatchdogStoreTest.java`

- [ ] **Step 1: Add vertx-jdbc-client to runtime/pom.xml**

In `runtime/pom.xml`, inside the `<dependencies>` block, after the `quarkus-junit5-mockito` test dependency, add:

```xml
    <dependency>
      <groupId>io.vertx</groupId>
      <artifactId>vertx-jdbc-client</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Write ReactiveStoreTestProfile**

```java
package io.quarkiverse.qhorus.store.reactive;

import java.util.Map;

import io.quarkus.test.junit.QuarkusTestProfile;

public class ReactiveStoreTestProfile implements QuarkusTestProfile {

    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of(
                "quarkus.datasource.reactive", "true",
                "quarkus.datasource.reactive.url", "h2:mem:qhorus-reactive-test",
                "quarkus.arc.selected-alternatives",
                String.join(",",
                        "io.quarkiverse.qhorus.runtime.store.jpa.ChannelReactivePanacheRepo",
                        "io.quarkiverse.qhorus.runtime.store.jpa.ReactiveJpaChannelStore",
                        "io.quarkiverse.qhorus.runtime.store.jpa.MessageReactivePanacheRepo",
                        "io.quarkiverse.qhorus.runtime.store.jpa.ReactiveJpaMessageStore",
                        "io.quarkiverse.qhorus.runtime.store.jpa.InstanceReactivePanacheRepo",
                        "io.quarkiverse.qhorus.runtime.store.jpa.CapabilityReactivePanacheRepo",
                        "io.quarkiverse.qhorus.runtime.store.jpa.ReactiveJpaInstanceStore",
                        "io.quarkiverse.qhorus.runtime.store.jpa.SharedDataReactivePanacheRepo",
                        "io.quarkiverse.qhorus.runtime.store.jpa.ArtefactClaimReactivePanacheRepo",
                        "io.quarkiverse.qhorus.runtime.store.jpa.ReactiveJpaDataStore",
                        "io.quarkiverse.qhorus.runtime.store.jpa.WatchdogReactivePanacheRepo",
                        "io.quarkiverse.qhorus.runtime.store.jpa.ReactiveJpaWatchdogStore"));
    }
}
```

- [ ] **Step 3: Write ReactiveJpaChannelStoreTest**

Tests run on the Vert.x event loop via `@RunOnVertxContext`. Mutations wrap in `Panache.withTransaction()` since `@WithTransaction` operates at the method boundary, not test boundary.

```java
package io.quarkiverse.qhorus.store.reactive;

import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.store.ReactiveChannelStore;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.vertx.RunOnVertxContext;
import io.quarkus.test.vertx.UniAsserter;

@QuarkusTest
@TestProfile(ReactiveStoreTestProfile.class)
class ReactiveJpaChannelStoreTest {

    @Inject
    ReactiveChannelStore store;

    @Test
    @RunOnVertxContext
    void put_persistsChannelAndAssignsId(UniAsserter asserter) {
        Channel ch = new Channel();
        ch.name = "rx-put-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;

        asserter.assertThat(
                () -> Panache.withTransaction(() -> store.put(ch)),
                saved -> {
                    assertNotNull(saved.id);
                    assertEquals(ChannelSemantic.APPEND, saved.semantic);
                });
    }

    @Test
    @RunOnVertxContext
    void find_returnsEmpty_whenNotFound(UniAsserter asserter) {
        asserter.assertThat(
                () -> store.find(UUID.randomUUID()),
                opt -> assertTrue(opt.isEmpty()));
    }

    @Test
    @RunOnVertxContext
    void findByName_returnsChannel_whenExists(UniAsserter asserter) {
        Channel ch = new Channel();
        ch.name = "rx-named-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.COLLECT;

        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(ch)))
                .assertThat(
                        () -> store.findByName(ch.name),
                        opt -> {
                            assertTrue(opt.isPresent());
                            assertEquals(ChannelSemantic.COLLECT, opt.get().semantic);
                        });
    }

    @Test
    @RunOnVertxContext
    void scan_pausedOnly_returnsOnlyPaused(UniAsserter asserter) {
        Channel active = new Channel();
        active.name = "rx-active-" + UUID.randomUUID();
        active.semantic = ChannelSemantic.APPEND;
        active.paused = false;

        Channel paused = new Channel();
        paused.name = "rx-paused-" + UUID.randomUUID();
        paused.semantic = ChannelSemantic.APPEND;
        paused.paused = true;

        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(active)))
                .execute(() -> Panache.withTransaction(() -> store.put(paused)))
                .assertThat(
                        () -> store.scan(ChannelQuery.pausedOnly()),
                        results -> {
                            assertTrue(results.stream().anyMatch(c -> c.name.equals(paused.name)));
                            assertTrue(results.stream().noneMatch(c -> c.name.equals(active.name)));
                        });
    }

    @Test
    @RunOnVertxContext
    void delete_removesChannel(UniAsserter asserter) {
        Channel ch = new Channel();
        ch.name = "rx-del-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;

        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(ch)))
                .execute(() -> store.delete(ch.id))
                .assertThat(
                        () -> store.find(ch.id),
                        opt -> assertTrue(opt.isEmpty()));
    }
}
```

- [ ] **Step 4: Write ReactiveJpaMessageStoreTest**

```java
package io.quarkiverse.qhorus.store.reactive;

import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.store.ReactiveMessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.vertx.RunOnVertxContext;
import io.quarkus.test.vertx.UniAsserter;

@QuarkusTest
@TestProfile(ReactiveStoreTestProfile.class)
class ReactiveJpaMessageStoreTest {

    @Inject
    ReactiveMessageStore store;

    @Test
    @RunOnVertxContext
    void put_assignsIdAndReturns(UniAsserter asserter) {
        Message m = message(UUID.randomUUID(), "alice");
        asserter.assertThat(
                () -> Panache.withTransaction(() -> store.put(m)),
                saved -> assertNotNull(saved.id));
    }

    @Test
    @RunOnVertxContext
    void find_returnsEmpty_whenNotFound(UniAsserter asserter) {
        asserter.assertThat(
                () -> store.find(Long.MAX_VALUE),
                opt -> assertTrue(opt.isEmpty()));
    }

    @Test
    @RunOnVertxContext
    void scan_byChannel_returnsMatchingMessages(UniAsserter asserter) {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        Message m1 = message(ch1, "alice");
        Message m2 = message(ch2, "bob");

        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(m1)))
                .execute(() -> Panache.withTransaction(() -> store.put(m2)))
                .assertThat(
                        () -> store.scan(MessageQuery.forChannel(ch1)),
                        results -> {
                            assertEquals(1, results.size());
                            assertEquals("alice", results.get(0).sender);
                        });
    }

    @Test
    @RunOnVertxContext
    void countByChannel_returnsCorrectCount(UniAsserter asserter) {
        UUID ch = UUID.randomUUID();
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(message(ch, "x"))))
                .execute(() -> Panache.withTransaction(() -> store.put(message(ch, "y"))))
                .assertThat(
                        () -> store.countByChannel(ch),
                        count -> assertEquals(2, count));
    }

    @Test
    @RunOnVertxContext
    void deleteAll_removesAllMessagesForChannel(UniAsserter asserter) {
        UUID ch = UUID.randomUUID();
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(message(ch, "a"))))
                .execute(() -> Panache.withTransaction(() -> store.put(message(ch, "b"))))
                .execute(() -> store.deleteAll(ch))
                .assertThat(
                        () -> store.countByChannel(ch),
                        count -> assertEquals(0, count));
    }

    private Message message(UUID channelId, String sender) {
        Message m = new Message();
        m.channelId = channelId;
        m.sender = sender;
        m.messageType = MessageType.request;
        m.content = "hello";
        return m;
    }
}
```

- [ ] **Step 5: Write ReactiveJpaInstanceStoreTest**

```java
package io.quarkiverse.qhorus.store.reactive;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.ReactiveInstanceStore;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.vertx.RunOnVertxContext;
import io.quarkus.test.vertx.UniAsserter;

@QuarkusTest
@TestProfile(ReactiveStoreTestProfile.class)
class ReactiveJpaInstanceStoreTest {

    @Inject
    ReactiveInstanceStore store;

    @Test
    @RunOnVertxContext
    void put_assignsIdAndReturns(UniAsserter asserter) {
        Instance inst = instance("rx-agent-" + UUID.randomUUID());
        asserter.assertThat(
                () -> Panache.withTransaction(() -> store.put(inst)),
                saved -> assertNotNull(saved.id));
    }

    @Test
    @RunOnVertxContext
    void putCapabilities_andFindCapabilities(UniAsserter asserter) {
        Instance inst = instance("cap-rx-" + UUID.randomUUID());
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(inst)))
                .execute(() -> store.putCapabilities(inst.id, List.of("search", "plan")))
                .assertThat(
                        () -> store.findCapabilities(inst.id),
                        caps -> {
                            assertTrue(caps.contains("search"));
                            assertTrue(caps.contains("plan"));
                        });
    }

    @Test
    @RunOnVertxContext
    void scan_byCapability_returnsMatchingInstances(UniAsserter asserter) {
        Instance a = instance("rx-cap-a-" + UUID.randomUUID());
        Instance b = instance("rx-cap-b-" + UUID.randomUUID());
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(a)))
                .execute(() -> Panache.withTransaction(() -> store.put(b)))
                .execute(() -> store.putCapabilities(a.id, List.of("review")))
                .assertThat(
                        () -> store.scan(InstanceQuery.byCapability("review")),
                        results -> {
                            assertEquals(1, results.size());
                            assertEquals(a.instanceId, results.get(0).instanceId);
                        });
    }

    private Instance instance(String instanceId) {
        Instance i = new Instance();
        i.instanceId = instanceId;
        i.status = "online";
        return i;
    }
}
```

- [ ] **Step 6: Write ReactiveJpaDataStoreTest**

```java
package io.quarkiverse.qhorus.store.reactive;

import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.ReactiveDataStore;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.vertx.RunOnVertxContext;
import io.quarkus.test.vertx.UniAsserter;

@QuarkusTest
@TestProfile(ReactiveStoreTestProfile.class)
class ReactiveJpaDataStoreTest {

    @Inject
    ReactiveDataStore store;

    @Test
    @RunOnVertxContext
    void put_assignsIdAndReturns(UniAsserter asserter) {
        SharedData data = sharedData("rx-key-" + UUID.randomUUID());
        asserter.assertThat(
                () -> Panache.withTransaction(() -> store.put(data)),
                saved -> assertNotNull(saved.id));
    }

    @Test
    @RunOnVertxContext
    void findByKey_returnsData_whenExists(UniAsserter asserter) {
        SharedData data = sharedData("rx-lookup-" + UUID.randomUUID());
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(data)))
                .assertThat(
                        () -> store.findByKey(data.key),
                        opt -> assertTrue(opt.isPresent()));
    }

    @Test
    @RunOnVertxContext
    void putClaim_andCountClaims(UniAsserter asserter) {
        SharedData data = sharedData("rx-claim-" + UUID.randomUUID());
        UUID instanceId = UUID.randomUUID();
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(data)))
                .execute(() -> {
                    ArtefactClaim claim = new ArtefactClaim();
                    claim.artefactId = data.id;
                    claim.instanceId = instanceId;
                    return Panache.withTransaction(() -> store.putClaim(claim));
                })
                .assertThat(
                        () -> store.countClaims(data.id),
                        count -> assertEquals(1, count));
    }

    private SharedData sharedData(String key) {
        SharedData d = new SharedData();
        d.key = key;
        d.createdBy = "test-rx";
        return d;
    }
}
```

- [ ] **Step 7: Write ReactiveJpaWatchdogStoreTest**

```java
package io.quarkiverse.qhorus.store.reactive;

import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.store.ReactiveWatchdogStore;
import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.vertx.RunOnVertxContext;
import io.quarkus.test.vertx.UniAsserter;

@QuarkusTest
@TestProfile(ReactiveStoreTestProfile.class)
class ReactiveJpaWatchdogStoreTest {

    @Inject
    ReactiveWatchdogStore store;

    @Test
    @RunOnVertxContext
    void put_assignsIdAndReturns(UniAsserter asserter) {
        Watchdog w = watchdog("threshold");
        asserter.assertThat(
                () -> Panache.withTransaction(() -> store.put(w)),
                saved -> assertNotNull(saved.id));
    }

    @Test
    @RunOnVertxContext
    void find_returnsEmpty_whenNotFound(UniAsserter asserter) {
        asserter.assertThat(
                () -> store.find(UUID.randomUUID()),
                opt -> assertTrue(opt.isEmpty()));
    }

    @Test
    @RunOnVertxContext
    void scan_byConditionType_returnsMatching(UniAsserter asserter) {
        Watchdog w1 = watchdog("threshold");
        Watchdog w2 = watchdog("pattern");
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(w1)))
                .execute(() -> Panache.withTransaction(() -> store.put(w2)))
                .assertThat(
                        () -> store.scan(WatchdogQuery.byConditionType("threshold")),
                        results -> {
                            assertEquals(1, results.size());
                            assertEquals("threshold", results.get(0).conditionType);
                        });
    }

    @Test
    @RunOnVertxContext
    void delete_removesWatchdog(UniAsserter asserter) {
        Watchdog w = watchdog("del-type");
        asserter
                .execute(() -> Panache.withTransaction(() -> store.put(w)))
                .execute(() -> store.delete(w.id))
                .assertThat(
                        () -> store.find(w.id),
                        opt -> assertTrue(opt.isEmpty()));
    }

    private Watchdog watchdog(String conditionType) {
        Watchdog w = new Watchdog();
        w.conditionType = conditionType;
        w.targetName = "test-target";
        w.notificationChannel = "test-alerts";
        return w;
    }
}
```

- [ ] **Step 8: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing -q 2>&1 | tail -15
```

Expected: BUILD SUCCESS. All existing 717 tests pass plus the new reactive store tests.

If the reactive profile fails to initialise (e.g. "no reactive datasource configured"), check:
1. `vertx-jdbc-client` appears in `mvn dependency:list -pl runtime | grep vertx-jdbc`
2. The H2 reactive URL format — Quarkus may need `jdbc:h2:mem:...` instead of `h2:mem:...`; adjust `ReactiveStoreTestProfile` accordingly and re-run.

- [ ] **Step 9: Commit**

```bash
git add runtime/pom.xml \
        runtime/src/test/java/io/quarkiverse/qhorus/store/reactive/
git commit -m "test(store): ReactiveJpa*Store integration tests + reactive test profile

Refs #74"
```

- [ ] **Step 10: Close issue**

```bash
gh issue close 74 --repo mdproctor/quarkus-qhorus \
  --comment "All deliverables complete: 5 Reactive*Store interfaces, 7 Panache repo helpers, 5 ReactiveJpa*Store implementations (@Alternative), 5 InMemoryReactive*Store wrappers, 10 unit tests, 5 integration tests."
```

---

## Self-Review

**Spec coverage check:**
- ✅ 5 `Reactive*Store` interfaces under `runtime/store/` — Tasks 1
- ✅ 5 `ReactiveJpa*Store` implementations under `runtime/store/jpa/` — Tasks 3–4
- ✅ All reactive impls `@Alternative`; blocking `Jpa*Store` unchanged — verified in Tasks 3–4
- ✅ Existing 717 tests still green — Task 4 Step 5 verification
- ✅ InMemoryReactive*Store (per HANDOFF.md scope) — Task 5
- ✅ TDD: unit tests before/alongside implementation for InMemory stores — Task 5
- ✅ Integration tests for ReactiveJpa stores — Task 6
- ✅ All commits reference Issue #74

**Placeholder scan:** None found. All steps contain complete code.

**Type consistency check:**
- `ReactiveChannelStore.put(Channel)` → `Uni<Channel>` — matches in `ReactiveJpaChannelStore` and `InMemoryReactiveChannelStore` ✅
- `ReactiveMessageStore.find(Long)` → `Uni<Optional<Message>>` — Message PK is `Long`, consistent across interface, impl, test ✅
- `ReactiveInstanceStore.putCapabilities(UUID, List<String>)` → `Uni<Void>` — matches impl (uses `CapabilityReactivePanacheRepo`) ✅
- `ReactiveDataStore.countClaims(UUID)` → `Uni<Integer>` — impl uses `.map(Long::intValue)` ✅
- All `@Alternative` classes are in `store/jpa/` package, consistent with `quarkus.arc.selected-alternatives` entries in `ReactiveStoreTestProfile` ✅
