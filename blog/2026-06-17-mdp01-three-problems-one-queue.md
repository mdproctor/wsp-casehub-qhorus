---
layout: post
title: "Three Problems, One Queue"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [sse, virtual-threads, quarkus, a2a]
series: issue-278-sse-keepalive-timeout
---

The A2A SSE endpoint had what looked like three separate problems: no keepalive (proxies drop idle connections), orphaned consumers (disconnected clients leaving their entries in the registry forever), and no maximum connection lifetime. The obvious approach is to solve each separately — a scheduler for keepalives, lazy cleanup for orphans, another scheduler for timeouts. We started down that path, then stopped.

The root issue was simpler. `streamTask()` registered a callback and returned. The virtual thread was freed. Everything downstream was working against a fundamental mismatch: SSE is a long-lived, stateful connection, but we'd built it as a short-lived handler. Without a thread owning the connection, there's nowhere natural to put keepalives, nowhere to watch for a closed sink, nowhere to enforce a deadline.

The fix was to stop returning. The virtual thread that receives the SSE request should hold the connection for its entire lifetime. Java 21 virtual threads are built for exactly this — one per connection, parked in a blocking wait, not consuming an OS thread.

The synchronisation primitive that makes it clean is `LinkedBlockingQueue.poll(heartbeatMs)`. When the poll times out, send a keepalive. When it returns a message, push the SSE event. Check `sink.isClosed()` at the top of each iteration — orphan detection at near-zero cost. Track a deadline — max-duration enforcement. Three concerns, one loop, no scheduler state.

```java
while (true) {
    if (sink.isClosed()) break;
    long remaining = deadline - System.currentTimeMillis();
    if (remaining <= 0) break;

    OutboundMessage msg = queue.poll(Math.min(heartbeatMs, remaining), MILLISECONDS);

    if (msg == null) {
        sink.send(sse.newEventBuilder().name("keepalive").data("").build());
        continue;
    }
    // handle message
}
```

The consumer — what `A2AChannelBackend` calls when a message arrives — becomes `queue::offer`. The backend is unchanged.

One thing we didn't anticipate: RESTEasy's `SseEventSource` client fires the registered event handler for SSE comment lines. SSE comments are supposed to be silently ignored — that's in the spec. RESTEasy doesn't honour it. The integration test asserting keepalive comments don't reach the event handler failed because they did, arriving as empty-name, empty-data events. We switched to named events (`event: keepalive`) instead — same wire effect for proxy keepalive, filterable by name in test handlers.

Several things came up in review that I wouldn't have caught on my own. The try-finally scope was wrong — the consumer registration happened before the outer try block, which meant if a follow-up transaction threw, the consumer leaked into the registry permanently with no virtual thread to drain it. The method needed `throws Exception` on the signature since the helpers it calls declare it. `Set.of().contains(null)` throws NPE — unlike `HashSet`, which returns false — so the state reference needed a non-null default. `TimeUnit.DAYS.toMillis(Long.MAX_VALUE)` overflows to a negative long, which could cause "disabled reconnect" to behave as "reconnect immediately."

There was also a misdiagnosis early on about what keepalives actually need to do. The assumption was that a virtual thread staying alive keeps the connection open. It does — for the server. Proxies watch for TCP inactivity, not server process health. No bytes flowing means the connection gets killed regardless of what the server is doing. The keepalive event sends actual bytes, which is what matters to a load balancer.
