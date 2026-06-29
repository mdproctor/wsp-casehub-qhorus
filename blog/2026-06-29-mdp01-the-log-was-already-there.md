---
layout: post
title: "The Log Was Already There"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [delivery-guarantee, channel-backend, cursor, messaging]
---

Every messaging system that adds delivery guarantees eventually reinvents the same pattern: a durable log, a cursor per consumer, and something that drives consumption forward. Kafka did it with partitions and consumer group offsets. AMQP did it with acknowledgments and redelivery. The question isn't whether to use the pattern — it's whether you notice that the log already exists.

Qhorus had the log. `MessageService.dispatch()` persists every message before `fanOut()` fires. The message store is ordered, queryable, and cursor-capable — `MessageQuery.poll(channelId, afterId, limit)` has been there since the early days. What was missing was the cursor and the consumer loop.

The original issue (#132) proposed three options: at-most-once with client catch-up, retry with exponential backoff, or a dead-letter queue per backend. I rejected all three. The retry approach — the one the issue recommended — only handles transient failures within a single JVM lifetime. If the process restarts between persistence and fanOut, no retry logic can help because the virtual thread never started. The DLQ approach adds a separate table that duplicates what the message store already provides. The right answer was to stop treating the message store as something separate from the delivery mechanism.

The design I landed on splits the world in two. Backends declare `DeliveryGuarantee.BEST_EFFORT` (the default — zero overhead, current fire-and-forget behavior) or `AT_LEAST_ONCE`. For best-effort backends, nothing changes: fanOut delivers in a virtual thread, catches exceptions, logs them, moves on. For at-least-once backends, fanOut skips them entirely. A dedicated delivery pump — signaled post-commit, self-driving until caught up — handles their delivery from the message store via cursor.

The "post-commit" part matters more than I initially appreciated. The adversarial design review caught a race I'd missed: under PostgreSQL READ COMMITTED isolation, signaling the pump inline (right after the DB write but before the transaction commits) wakes the consumer before its query can see the new row. The fix is `TransactionSynchronizationRegistry.afterCompletion(STATUS_COMMITTED)` — the same mechanism `MessageObserverDispatcher` already uses for observer dispatch. The pump always sees committed data.

There's an architectural decision embedded in making the pump the *sole* delivery path for tracked backends. The alternative — have both fanOut and the pump deliver, with deduplication — creates cursor advancement races and the possibility of duplicate Slack messages. By giving each backend exactly one delivery path based on its declared guarantee, there's no concurrency to manage between the two mechanisms. The pump owns sequential delivery; fanOut owns fire-and-forget. They don't overlap.

The cursor itself is minimal: `(channelId, backendId, lastDeliveredId)` — one row per backend per channel, advanced per batch. A scheduled reconciler (30s) scans all cursors as a safety net for JVM restart gaps. In-memory health tracking acts as a circuit breaker — after 10 consecutive failures, the pump stops attempting that backend until the reconciler succeeds or the backend re-registers.

SlackChannelBackend and ConnectorChannelBackend now declare `AT_LEAST_ONCE`. The Slack backend was the original motivation — a lost Slack message is visible to the human who expected it. The connector backend has the same silent-failure problem. A2A and Claudony's panel backend stay best-effort: A2A has SSE with its own catch-up via CommitmentStore, and Claudony polls `check_messages`.

The thing that makes this work cleanly is that `ChannelBackend.deliveryGuarantee()` is a default method returning `BEST_EFFORT`. Every existing backend — across qhorus, claudony, openclaw, drafthouse — compiles and behaves identically without any change. The migration cost for backends that want tracking is one method override.
