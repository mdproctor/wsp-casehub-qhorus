---
title: "The seam that was half-closed"
date: 2026-05-29
series: casehub-qhorus-dev
entry: mdp01
tags: [casehub, qhorus, store-pattern, panache, testing]
---

The store seam pattern in Qhorus is simple: services inject `*Store` interfaces, JPA implementations run in production, InMemory alternatives activate in tests. Every service follows it — except `WatchdogEvaluationService`, which had three methods calling Panache directly.

```java
// What it looked like — wrong
List<Commitment> pending = Commitment.<Commitment>list(
    "state IN ?1 AND expiresAt IS NOT NULL",
    List.of(CommitmentState.OPEN, CommitmentState.ACKNOWLEDGED));

List<Instance> staleInstances =
    io.casehub.qhorus.runtime.instance.Instance
        .<io.casehub.qhorus.runtime.instance.Instance>list(
            "status = 'stale' AND lastSeen < ?1", cutoff);

long count = Message.count(
    "channelId = ?1 AND messageType != ?2", ch.id, MessageType.EVENT);
```

`CommitmentStore`, `InstanceStore`, and `MessageStore` were all injected in the same class and used elsewhere. These three methods just never got the memo. The InMemory test alternatives were invisible to them.

The first two fixes were mechanical: `CommitmentStore.findAllOpen()` exists with the right semantics; `InstanceQuery.builder().status("stale").staleOlderThan(cutoff).build()` maps directly. No new store methods needed.

The third one required adding `count(MessageQuery)` to `MessageStore`. And that's where it got interesting.

`MessageStore.scan(MessageQuery)` works by building full JPQL — the string starts with `"FROM Message WHERE "` and predicates are appended. Panache's `list()` and `find()` take that full JPQL and run it. So the natural move when writing `count()` is to pass `"WHERE " + predicates`. That's wrong.

Panache's `count(String, Object...)` takes just the predicate fragment. It prepends `FROM Entity WHERE` internally. Pass `"WHERE 1=1 AND channelId = ?1"` and you get `FROM Message WHERE WHERE 1=1 AND channelId = ?1` — a syntax error. The InMemory tests don't catch this; only the JPA integration test does.

The fix is `Message.count(predicates, params)` without the `WHERE` prefix. But the more interesting fix is why both `scan()` and `count()` had separate copies of the predicate-building logic.

Both methods needed to construct `channelId = ?1 AND afterId > ?2 AND sender = ?3...` from a `MessageQuery`. One wrapped it in full JPQL; the other passed it as-is. Factoring them into a shared `MessageQueryJpql` record — a package-private type with a `static from(MessageQuery)` factory — means there's one place to update when `MessageQuery` gains a new field.

```java
record MessageQueryJpql(String where, Object[] params) {
    static MessageQueryJpql from(MessageQuery q) {
        StringBuilder where = new StringBuilder("1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;
        if (q.channelId() != null) {
            where.append(" AND channelId = ?").append(idx++);
            params.add(q.channelId());
        }
        // ... other fields ...
        return new MessageQueryJpql(where.toString(), params.toArray());
    }
}
```

`scan()` prepends `"FROM Message WHERE "` and appends `ORDER BY`. `count()` passes `mq.where()` directly to `Message.count()`. Same predicates, different wrappers.

There's one more subtlety worth noting. `InMemoryMessageStore.count()` must not delegate to `scan()`. `scan()` applies the `limit` field — a poll query with `limit=10` would make `scan(q).size()` return at most 10, giving wrong counts. The implementation has to stream the backing map directly: `store.values().stream().filter(q::matches).count()`. The `matches()` predicate doesn't apply `limit`; it's per-entity filtering only.

The branch also adds a doc comment to `ConfiguredWatchdogAlertRouter.route()`. It ignores the `event` parameter entirely — v1 fan-out delivers every alert to all configured endpoints regardless of which condition fired. That's intentional, but the code gave no hint of it. A single Javadoc explaining the v1 fan-out semantics and how to override it with a non-`@DefaultBean` implementation is the kind of comment that saves the next person an hour.
