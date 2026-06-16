# A2A SSE Active Model: Keepalive + Timeout (#278) and Live-Stream Integration Test (#277)

**Date:** 2026-06-16
**Issues:** qhorus#278, qhorus#277
**Branch:** issue-278-sse-keepalive-timeout

---

## Problem

`A2AResource.streamTask()` uses a passive model: the JAX-RS handler registers a
`Consumer<OutboundMessage>` callback and returns. The virtual thread is freed. Events
arrive via `fanOut()` on separate virtual threads, which call the consumer, which calls
`sink.send()`.

This passive model creates three gaps:

1. **No keepalive** — proxies and load balancers drop idle SSE connections after N seconds
   (commonly 30–120s). The A2A spec §6 previously marked keepalive as "Quarkus handles it".
   That was wrong. No keepalive is sent.
2. **Orphan consumers** — if the task stalls (no DONE/FAILURE/DECLINE ever dispatched),
   a client that disconnects never triggers consumer deregistration. The consumer and sink
   stay in `sseStreams` until server restart.
3. **No connection lifetime bound** — there is no maximum duration on an open SSE connection.

Both Option A (global scheduler in `A2AChannelBackend`) and Option B (`AtomicBoolean`
keepalive loop with concurrent SSE writes from two threads) treat these as separate problems
to bolt on. They are not separate — they stem from a single root cause.

---

## Root Cause

The passive model grafts a stateful, long-lived connection onto a handler that exits
immediately. With no thread owning the connection after the handler returns, there is
no natural place to send heartbeats, detect disconnects, or enforce a lifetime bound.

---

## Solution: Active Virtual-Thread Model

An SSE connection is a blocking I/O operation. Java 21 virtual threads exist precisely
for this. The correct architecture is an **active model**: the virtual thread that
receives the request stays alive and owns the connection for its entire lifetime.

The synchronization primitive is a `BlockingQueue<OutboundMessage>`. The consumer becomes
`queue::offer` — a pure data handoff with no JAX-RS context. All SSE writes happen on
one thread (the virtual thread running the loop), eliminating concurrent-write concerns.

`queue.poll(heartbeatInterval)` returning `null` is the natural keepalive trigger. The
`sink.isClosed()` check at the top of each iteration is the natural orphan-detection
mechanism. The deadline check is the natural max-duration enforcement.

---

## Design

### §1 — `streamTask()` rewrite

Remove `@Transactional`. Replace with `QuarkusTransaction.requiringNew()` scoped to just
the validation reads. The short-lived transaction commits immediately; the loop runs with
no active transaction.

```
streamTask() [virtual thread, active for connection duration]:

  1. Immediate exits: A2A disabled, invalid UUID
  2. QuarkusTransaction.requiringNew() → validate task exists, check if already terminal
  3. If terminal → sendStatusEvent, return
  4. consumer = queue::offer
     a2aBackend.registerStream(corrId, consumer)
  5. QuarkusTransaction.requiringNew() → re-check state (close dispatch-during-registration race)
     If now terminal → deregister, sendStatusEvent, return
  6. LOOP (virtual thread blocks here):
       remaining = deadline − now; if <= 0 → break
       msg = queue.poll(min(heartbeatMs, remaining))
       null → send keepalive comment if sink open and not past deadline; continue
       msg  → send task_status_update event
              if terminal → await send completion; break
  7. finally: deregisterStream; if !sink.isClosed() → sink.close()
```

Key properties:
- **Single-threaded SSE writes** — no concurrent `sink.send()` from multiple threads
- **No `AtomicBoolean`/`AtomicReference`/`whenComplete` chains** — state is local to the loop
- **Keepalive** — `poll()` timeout drives it naturally
- **Orphan detection** — `sink.isClosed()` checked every heartbeat interval
- **Max duration** — deadline checked every iteration
- **Terminal send awaited** — `.toCompletableFuture().get(5, SECONDS)` before `finally` closes
  the sink; on a virtual thread, `.get()` parks without blocking an OS thread
- **Exception boundary** — the entire loop body (including the awaited terminal send) is
  wrapped in `try/catch (Exception e)`; `finally` runs cleanup regardless of how the loop exits

### §2 — Config additions to `QhorusConfig.A2a`

```java
interface A2a {
    @WithDefault("false")
    boolean enabled();

    Sse sse();

    interface Sse {
        /** Interval between SSE comment keepalives. Default: 15s. */
        @WithDefault("15")
        int heartbeatIntervalSeconds();

        /** Maximum SSE stream lifetime before server-side close. Default: 300s. */
        @WithDefault("300")
        int maxDurationSeconds();
    }
}
```

Config keys: `casehub.qhorus.a2a.sse.heartbeat-interval-seconds`,
`casehub.qhorus.a2a.sse.max-duration-seconds`.

### §3 — `A2AChannelBackend` — zero changes

The registry stays `ConcurrentHashMap<UUID, Set<Consumer<OutboundMessage>>>`. The consumer
is `queue::offer` — a one-liner that replaces the 15-line `AtomicReference`-based lambda.
`post()`, `registerStream()`, `deregisterStream()`, and `streamCount()` are unchanged.
All existing unit tests pass without modification.

### §4 — `A2ATaskState` — add `TERMINAL_STATES`

The re-check path needs to compare state strings against terminal values:

```java
static final Set<String> TERMINAL_STATES = Set.of("completed", "failed", "cancelled");
```

### §5 — `A2AEnabledProfile` — add SSE config overrides

Integration tests that exercise keepalive need a short heartbeat interval:

```java
config.put("casehub.qhorus.a2a.sse.heartbeat-interval-seconds", "1");
config.put("casehub.qhorus.a2a.sse.max-duration-seconds", "30");
```

---

## Testing (#277)

New class: `A2AStreamIntegrationTest` — `@QuarkusTest @TestProfile(A2AEnabledProfile.class)`.

### Coordination pattern (all tests)

```
1. QuarkusTransaction.requiringNew() → create channel + dispatch COMMAND (committed)
2. Open SseEventSource on background thread; register event listener with CountDownLatch
3. Awaitility.await().atMost(2s).until(() -> a2aBackend.streamCount(corrId) > 0)
4. QuarkusTransaction.requiringNew() → dispatch terminal message (committed)
5. assertTrue(latch.await(10, TimeUnit.SECONDS))
6. Assert event payload
7. client.close()
```

### Test cases

| Test | Setup | Dispatch | Assert |
|------|-------|----------|--------|
| `sseStream_receivesCompletedEvent_whenDoneDispatched` | COMMAND | DONE | `"state":"completed","final":true` |
| `sseStream_receivesCancelledEvent_whenDeclineDispatched` | COMMAND | DECLINE | `"state":"cancelled","final":true` |
| `sseStream_keepaliveCommentsDoNotTriggerEventHandlers` | COMMAND | none | event handler not called after 3s; connection still open (streamCount > 0) |
| `sseStream_alreadyTerminalTask_sendsImmediateFinalEvent` | COMMAND + DONE (before stream opens) | none | `"state":"completed","final":true` immediately; no Awaitility on streamCount |

The keepalive test verifies SSE comment lines (`: keepalive`) are not delivered to
`SseEventSource` event handlers — comments are ignored by the JAX-RS SSE client per spec.
It uses `heartbeat-interval-seconds=1` (from `A2AEnabledProfile`) so it completes in CI
without a long wait.

### SSE client pattern

```java
Client client = ClientBuilder.newClient();
WebTarget target = client.target(uri).path("/a2a/tasks/" + taskId + "/stream");
CountDownLatch latch = new CountDownLatch(1);
CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
try (SseEventSource source = SseEventSource.target(target).build()) {
    source.register(event -> {
        events.add(event.readData(String.class));
        latch.countDown();
    });
    source.open();
    // Awaitility sync + dispatch + latch.await() here
} finally {
    client.close();
}
```

---

## Edge Cases

### Race: message dispatched between initial read and consumer registration

Between `QuarkusTransaction.requiringNew()` (reads non-terminal state) and
`a2aBackend.registerStream()`, a terminal message could be dispatched and missed. The loop
would then wait until timeout.

**Mitigation:** a second `QuarkusTransaction.requiringNew()` immediately after
`registerStream()` re-reads the state. If terminal, the consumer is deregistered and the
terminal event is sent immediately. This closes most of the race window at minimal cost.
A residual window remains (between the re-check and the first `queue.poll()`), accepted
as an inherent limitation.

### Timeout behavior

When the loop exits due to deadline, `finally` calls `sink.close()` with no preceding
event. The client sees a clean connection close and should reconnect. No new event type
is introduced — the connection close is the signal.

### `sink.send()` for keepalive comments

Fire-and-forget. If the send fails, `sink.isClosed()` becomes true on the next iteration.

### `sink.send()` for terminal events

Awaited: `.toCompletableFuture().get(5, SECONDS)`. Ensures the terminal payload reaches
the client before `sink.close()` fires in `finally`.

### Server restart

SSE subscriptions do not survive restarts — unchanged from #147, documented in §6 of the
original SSE spec.

### Reactive path

`ReactiveA2AResource` is unchanged — no reactive equivalent of this issue is in scope.

---

## Not In Scope

- Reactive path (`ReactiveA2AResource`)
- Persistent A2A channel participation (restart recovery) — tracked separately
- Backpressure for high-frequency message streams

---

## Impact Summary

| Component | Change |
|-----------|--------|
| `A2AResource.streamTask()` | Rewrite: remove `@Transactional`, add programmatic tx + active loop |
| `QhorusConfig.A2a` | Add `Sse` sub-interface with two config properties |
| `A2ATaskState` | Add `TERMINAL_STATES` string set |
| `A2AEnabledProfile` | Add SSE config overrides for tests |
| `A2AChannelBackend` | **Zero changes** |
| `A2AChannelBackendSseTest` | **Zero changes** |
| `A2AStreamIntegrationTest` | New — four test cases |
