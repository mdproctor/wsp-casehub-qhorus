---
layout: post
title: "Slack threads and the cache write problem"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [slack, channel-backend, thread-cache, cdi, race-condition]
series: issue-261-slack-channel-backend
---

The Slack channel backend is shipped. #261 is done. But getting there involved
two decisions about timing that weren't in the spec.

The session opened with something else entirely — a CDI regression guard for
a Claudony test failure. The root fix (switching `ReactiveJpaChannelStore.updateLastActivity`
from positional to named JPQL parameters) had already landed. What was missing
was a CI gate to catch the same failure pattern before it surfaces in a consumer again.

The insight from the Claudony analysis was precise: `@Alternative @Priority(1)` from
an external jar is insufficient to override an `@IfBuildProperty`-gated bean when
both are active in the same CDI context. The reactive JPA stores are gated behind
`casehub.qhorus.reactive.enabled=true`, but when that gate passes, `@Priority` alone
doesn't win the CDI selection — the InMemory stores also need to appear in
`quarkus.arc.selected-alternatives`. This isn't documented; we found it by elimination.

I added `StoreCdiAlternativesTest` to `examples/type-system` — the module that has both
`casehub-qhorus` and `casehub-qhorus-testing` on its classpath, matching what a consumer
sees. One of the tests simply asserts `reactiveChannelStore instanceof InMemoryReactiveChannelStore`.
Runs without PostgreSQL and catches exactly the failure mode Claudony hit.

---

The Slack backend itself had been designed three weeks ago (spec r4), so the
implementation was mostly translation. The interesting parts were two ordering decisions.

**Thread continuity across restarts.** Slack replies need a `thread_ts` — the timestamp
of the root message in the thread. Without it, your response appears as a new top-level
message instead of a reply. We cache `(channelId, correlationId) → thread_ts` in memory.
But a server restart wipes the cache. Without it, the next outbound message starts a
fresh Slack thread, silently fragmenting the conversation.

The answer is to back the in-memory cache with DB rows. `SlackThreadCache` (V24) holds
the same mapping persistently. On `ChannelInitialisedEvent`, `SlackChannelBackend` loads
all rows for that channel and pre-warms the cache. The overhead is one SELECT per channel
init — not per message — and one INSERT when a new commitment opens its first thread.

**The write-before-dispatch race.** The subtler problem: when does the cache entry get written?

The natural approach is to write the correlationId mapping after receiving the inbound
Slack message, then dispatch to the gateway. The gateway fires your message handlers.
If the agent is fast — running on a virtual thread, or just quick — it can send a RESPONSE
before the cache write lands, because the write happened after the dispatch call returned.
The `post()` handler looks up the cache, finds nothing, and creates a new top-level Slack
message instead of a thread reply.

The fix: write both the DB row and the in-memory entry *before* calling
`gateway.receiveHumanMessage()`.

```java
if (slackTs != null) {
    threadCacheStore.save(channelRef.id(), corrId, slackTs);          // DB first
    threadCache.computeIfAbsent(channelRef.id(), k -> new ConcurrentHashMap<>())
               .put(corrId, slackTs);                                  // memory second
}
gateway.receiveHumanMessage(channelRef,                                // dispatch last
    new InboundHumanMessage(senderId, content, receivedAt, metadata, corrId.toString(), null));
```

Obvious in hindsight. The natural mental model is receive → process → write state on the
way out. Here, the state needs to be on the way in.

---

Two unexpected gotchas during integration testing.

After a JPQL bulk DELETE in `deleteByChannelId`, `em.find()` returned the deleted entity
from Hibernate's L1 cache. The row was gone from H2; the persistence context still had
it. Switching `findByChannelId` to a JPQL SELECT query was the surgical fix — JPQL queries
go to the DB rather than reading from the first-level cache.

The bigger surprise was `quarkus.hibernate-orm.qhorus.packages`. The qhorus PU only scans
`io.casehub.qhorus.runtime` and `io.casehub.ledger.runtime`. The new slack entities live
in `io.casehub.qhorus.slack`. Without that package in the test configuration, Hibernate
throws `Unknown entity type` even though the jar is Jandex-indexed. Any future optional
qhorus module that ships JPA entities will hit the same wall.

The module builds clean and the issue is closed. The thread cache is the part worth
remembering — not for Slack specifically, but for any system where a correlation mapping
must exist before the first handler fires.
