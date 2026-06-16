# A2A SSE Active Model: Keepalive + Timeout (#278) and Live-Stream Integration Test (#277)

**Date:** 2026-06-16 (revised — incorporates spec review)
**Issues:** qhorus#278, qhorus#277
**Branch:** issue-278-sse-keepalive-timeout

---

## Problem

`A2AResource.streamTask()` uses a passive model: the JAX-RS handler registers a
`Consumer<OutboundMessage>` callback and returns. The virtual thread is freed. Events
arrive via `fanOut()` on separate virtual threads, which call the consumer, which calls
`sink.send()`.

This passive model creates three gaps:

1. **No keepalive** — proxies and load balancers drop idle SSE connections. The prior
   spec (§7 of `2026-06-13-a2a-sse-streaming.md`) marked keepalive as "Quarkus handles it."
   That was wrong. No keepalive is sent.
2. **Orphan consumers** — if the task stalls, a client that disconnects never triggers
   consumer deregistration. The consumer and sink stay in `sseStreams` until server restart.
3. **No connection lifetime bound** — there is no maximum duration on an open SSE connection.

---

## Root Cause

The passive model grafts a stateful, long-lived connection onto a handler that exits
immediately. With no thread owning the connection after the handler returns, there is
no natural place to send heartbeats, detect disconnects, or enforce a lifetime bound.

---

## Solution: Active Virtual-Thread Model

An SSE connection is a blocking I/O operation. Java 21 virtual threads exist precisely
for this. The correct architecture: the virtual thread that receives the request stays
alive and owns the connection for its entire lifetime.

The synchronization primitive is a `LinkedBlockingQueue<OutboundMessage>`. The consumer
becomes `queue::offer` — a pure data handoff with no JAX-RS context. All SSE writes
happen on one thread (the virtual thread running the loop), eliminating concurrent-write
concerns.

`queue.poll(heartbeatInterval)` returning `null` is the natural keepalive trigger. The
`sink.isClosed()` check at the top of each iteration is the natural orphan-detection
mechanism. The deadline check is the natural max-duration enforcement.

---

## Design

### §1 — `streamTask()` rewrite

**Thread dispatch:** Remove `@Transactional`. Add `@RunOnVirtualThread`
(`io.smallrye.common.annotation.RunOnVirtualThread` — same package as `@Blocking`;
do not confuse the two).

`@Transactional` was the mechanism that caused RESTEasy Reactive to dispatch the method
on a virtual thread (documented as "load-bearing" in the existing javadoc). Removing it
without a replacement would revert dispatch to the Vert.x I/O thread — `BlockingQueue.poll()`
would block the event loop and freeze the server. `@Blocking` (`io.smallrye.common.annotation.Blocking`)
would use the finite worker thread pool (default 20 threads), exhausting it under any real
SSE load. `@RunOnVirtualThread` creates one Loom virtual thread per request — the correct
model for long-lived blocking I/O. This is the first use of `@RunOnVirtualThread` in the
codebase; the distinction from `@Blocking` matters precisely here.

**Transaction scope:** Use `QuarkusTransaction.requiringNew()` for the two transactional
reads (initial validation + post-registration re-check). Each completes in milliseconds.
The loop runs with no active transaction.

**Private helpers:** `sendStatusEvent` and `sendErrorEvent` are only ever called from
`streamTask()`. Both helpers declare `throws Exception` and use an internal try-finally to
guarantee `sink.close()` regardless of whether the send succeeds, times out, or is
interrupted:

```java
private static void sendStatusEvent(SseEventSink sink, Sse sse, String taskId, String state)
        throws Exception {
    try {
        sink.send(sse.newEventBuilder()
                .name("task_status_update")
                .data("{\"id\":\"%s\",\"status\":{\"state\":\"%s\"},\"final\":true}"
                        .formatted(taskId, state))
                .build())
            .toCompletableFuture().get(5, SECONDS);
    } finally {
        if (!sink.isClosed()) sink.close();
    }
}
```

`sendErrorEvent` is identical in structure. With this, each helper is fully self-contained:
the sink is closed regardless of how `.get()` exits. Callers in steps 1–2 and step 4 are
outside the try-finally, so checked exceptions propagate to JAX-RS error handling.
`streamTask()`'s outer finally sees `isClosed() = true` from the helper and skips the
redundant close — no double-close.

**Stale comment removal:** The existing `sendStatusEvent`/`sendErrorEvent` javadoc says
"Never call sink.close() synchronously after send() — the response may not yet be written."
That concern applied to the old async `thenRun` pattern, where calling close before the
send's `CompletionStage` completed caused `IllegalStateException`. With `.get(5, SECONDS)`
the send completes before close is called — the concern is gone. This comment must be
removed in the implementation.

**Execution flow:**

```
@GET
@Path("/tasks/{id}/stream")
@Produces("text/event-stream")
@RunOnVirtualThread
public void streamTask(@PathParam("id") final String taskId,
                       @Context final SseEventSink sink,
                       @Context final Sse sse) throws Exception {

// [virtual thread, active for connection duration]

  ── outside try-finally ────────────────────────────────────────────────────────
  1. Immediate exits (no tx): A2A disabled, invalid UUID
     → sendErrorEvent (self-contained: try-finally inside closes sink), return
  2. QuarkusTransaction.requiringNew() — validate task exists; read current state
     If not found → sendErrorEvent (self-contained), return
     If terminal  → sendStatusEvent (self-contained), return
  3. consumer = queue::offer      (LinkedBlockingQueue, unbounded — offer() never drops)
     a2aBackend.registerStream(corrId, consumer)
  ── try begins here (immediately after registerStream — covers steps 4 and 5) ──
  4. QuarkusTransaction.requiringNew() — re-check state after registration
     Closes race: message dispatched between step 2 and step 3 is either in the
     queue (consumer already registered) or DB-visible (re-check catches it).
     If now terminal → sendStatusEvent (self-contained), return
                       [finally deregisters consumer and skips close — sink already closed]
  5. LOOP (virtual thread blocks here via queue.poll):
       if sink.isClosed(): break         ← orphan detection: checked every iteration
       remaining = deadline − now; if ≤ 0 → break

       // InterruptedException handled locally — never reaches outer catch
       try { msg = queue.poll(min(heartbeatMs, remaining), MILLISECONDS); }
       catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }

       null (poll timeout)  → sink.send(comment "keepalive")  // fire-and-forget;
                              next iteration's isClosed() check detects broken pipe
       OutboundMessage      → send task_status_update event (fire-and-forget for
                              non-terminal; awaited via .get(5, SECONDS) for terminal)
                              if terminal → break
     catch (Exception e): log at DEBUG   // I/O and runtime failures only;
                                         // InterruptedException never reaches here
  ── finally ────────────────────────────────────────────────────────────────────
  6. a2aBackend.deregisterStream(corrId, consumer)
     if !sink.isClosed() → sink.close()
```

**Try structure — nested:** The outer try covers steps 4 and 5. The inner try covers the
loop body only. There is no catch on the outer try — step 4 JPA or transaction
infrastructure failures propagate directly through the outer try to `finally`, then to
JAX-RS (Quarkus/Narayana will log transaction failures; no additional logging needed here).
Only the loop body has a `catch (Exception e)` to absorb I/O and runtime failures and log
them at DEBUG. The nesting in Java:

```java
try {                                 // outer — steps 4 + 5
    // step 4: re-check tx
    if (terminal) { sendStatusEvent(); return; }  // finally deregisters on return
    try {                             // inner — loop only
        while (true) { ... }
    } catch (Exception e) {
        LOG.debugf(e, "SSE stream error for task %s", taskId);
    }
} finally {
    a2aBackend.deregisterStream(corrId, consumer);
    if (!sink.isClosed()) sink.close();
}
```

**Method signature:** `streamTask()` must declare `throws Exception`. The helpers
(`sendStatusEvent`, `sendErrorEvent`) declare `throws Exception` and are called at steps
1–2 (before the outer try, no surrounding catch) and step 4 (inside the outer try, which
has no catch clause). In all three positions the checked exception is uncaught within the
method body and must be declared on the signature. JAX-RS resource methods in RESTEasy
Reactive may declare `throws Exception` — the VT dispatcher catches and routes it to
JAX-RS error handling.

**Try boundary:** Steps 1–2 execute before the outer try — no consumer is registered yet,
so there is nothing to clean up. The outer try opens immediately after `registerStream()`
in step 3. This is critical: if step 4's `requiringNew()` throws, `finally` still runs and
deregisters the consumer. Without this boundary, the consumer would remain permanently in
`sseStreams`, the queue would accumulate messages from future `post()` calls, and neither
would be GC-eligible until server restart.

**InterruptedException:** caught locally inside the loop by an inner `try/catch` wrapping
`queue.poll()` — the interrupt flag is restored and the loop breaks. It never reaches the
outer `catch (Exception e)`, which handles only I/O and runtime failures.

**Non-terminal send failures:** Non-terminal `sink.send()` calls are fire-and-forget.
A broken pipe on a non-terminal event won't surface until the next `sink.isClosed()`
check (next heartbeat interval — up to `heartbeatIntervalSeconds`). This is acceptable
and is documented explicitly here.

**Terminal send:** Awaited via `.toCompletableFuture().get(5, SECONDS)`. Ensures the
terminal payload reaches the client before `finally` calls `sink.close()`. `.get()` on
a virtual thread parks without blocking an OS thread.

**Timeout behavior:** When the loop exits due to deadline, `finally` calls `sink.close()`
with no preceding event. The client sees a clean connection close and should reconnect.
No new event type is introduced.

### §2 — Config additions to `QhorusConfig.A2a`

The inner interface is named `SseSettings` (not `Sse`) to avoid shadowing the JAX-RS
`jakarta.ws.rs.sse.Sse` type imported in `A2AResource`. Config keys are unchanged:
`casehub.qhorus.a2a.sse.*`.

```java
interface A2a {
    @WithDefault("false")
    boolean enabled();

    SseSettings sse();

    interface SseSettings {
        /** Interval between SSE comment keepalives. Default: 15s. */
        @WithDefault("15")
        int heartbeatIntervalSeconds();

        /**
         * Maximum SSE stream lifetime before server-side close. Default: 1800s (30 min).
         *
         * A2A tasks in a multi-agent coordination context routinely run for minutes to
         * hours. 300s was the initial proposal but forces reconnect logic as a baseline
         * requirement for every client. 1800s is the defensible floor — long enough for
         * most coordinated agent tasks, short enough to bound runaway streams. Operators
         * running longer tasks should increase this value.
         */
        @WithDefault("1800")
        int maxDurationSeconds();
    }
}
```

### §3 — `A2ATaskState` — add `TERMINAL_STATES`

```java
static final Set<String> TERMINAL_STATES = Set.of("completed", "failed", "cancelled");
```

The existing three-way string comparison at lines 257–261 of `A2AResource` (`"completed".equals(...)
|| "failed".equals(...) || "cancelled".equals(...)`) is a call-site for this constant and
must be updated to `TERMINAL_STATES.contains(state)` for consistency.

### §4 — `A2AChannelBackend` — zero changes

The registry stays `ConcurrentHashMap<UUID, Set<Consumer<OutboundMessage>>>`. `post()`,
`registerStream()`, `deregisterStream()`, and `streamCount()` are unchanged. All existing
unit tests pass without modification.

### §5 — `A2AEnabledProfile` — add SSE config overrides

```java
config.put("casehub.qhorus.a2a.sse.heartbeat-interval-seconds", "1");
config.put("casehub.qhorus.a2a.sse.max-duration-seconds", "30");
```

`heartbeat-interval-seconds=1` allows the keepalive test to complete in ~3s in CI
without waiting 15s. `max-duration-seconds=30` keeps stalled-stream tests from hanging.

---

## Testing (#277 + migration from A2AStreamTaskTest)

### `A2AStreamTaskTest` — deleted

All four tests in `A2AStreamTaskTest` cover immediate-close paths using RestAssured.
These are migrated to `A2AStreamIntegrationTest` using `SseEventSource`, which tests
the actual SSE wire protocol rather than RestAssured's body extraction. The RestAssured
versions test that the response body *contains* SSE text — the `SseEventSource` versions
test that the SSE protocol delivers correctly-typed events to a real SSE client.
`A2AStreamTaskTest` is deleted in full; `A2AStreamIntegrationTest` becomes the single
owner of all SSE path coverage.

### `A2AStreamIntegrationTest` — new

`@QuarkusTest @TestProfile(A2AEnabledProfile.class)`

**URL injection:**
```java
@TestHTTPResource("") URI baseUri;
```

**SSE client pattern (all tests):**
```java
Client client = ClientBuilder.newClient();
WebTarget target = client.target(baseUri).path("/a2a/tasks/" + taskId + "/stream");
CountDownLatch latch = new CountDownLatch(1);
CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
try (SseEventSource source = SseEventSource.target(target)
        .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)  // disable auto-reconnect
        .build()) {
    source.register(event -> {
        events.add(event.readData(String.class));
        latch.countDown();
    });
    source.open();
    // ... test-specific assertions
} finally {
    client.close();
}
```

`reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)` disables the default 3s
auto-reconnect. `MILLISECONDS` is used deliberately — `TimeUnit.DAYS.toMillis(Long.MAX_VALUE)`
overflows to a large negative `long` (`Long.MAX_VALUE × 86,400,000` exceeds `Long.MAX_VALUE`),
which may be treated as "reconnect immediately" by the scheduling logic. `MILLISECONDS`
requires no unit conversion and cannot overflow. Without disabling reconnect, after a
terminal event closes the server side, the client reconnects and receives a second
`event:error` (task not found — no COMMAND was re-sent), corrupting the `events` list.

**Coordination pattern for live-stream tests:**
```
1. QuarkusTransaction.requiringNew() → create channel + dispatch COMMAND
2. Open SseEventSource, register listener with CountDownLatch(1)
3. Awaitility.await().atMost(2s).until(() -> a2aBackend.streamCount(corrId) > 0)
4. QuarkusTransaction.requiringNew() → dispatch DONE or DECLINE
5. assertTrue(latch.await(10, TimeUnit.SECONDS))
6. Assert event payload
```

**Test cases:**

| Test | Description | Key assertions |
|------|-------------|----------------|
| `sseStream_taskNotFound_returnsErrorEvent` | Migrated from `A2AStreamTaskTest` | `event:error`, `"final":true` |
| `sseStream_invalidUuid_returnsErrorEvent` | Migrated | `event:error`, `"final":true` |
| `sseStream_alreadyTerminalDone_sendsImmediateFinalEvent` | Migrated | `"state":"completed"`, `"final":true`; no Awaitility needed |
| `sseStream_alreadyTerminalDecline_sendsCancelledEvent` | Migrated | `"state":"cancelled"`, `"final":true` |
| `sseStream_receivesCompletedEvent_whenDoneDispatched` | New — live stream | `"state":"completed"`, `"final":true` |
| `sseStream_receivesCancelledEvent_whenDeclineDispatched` | New — live stream | `"state":"cancelled"`, `"final":true` |
| `sseStream_keepaliveCommentsDoNotTriggerEventHandlers` | New | event handler NOT called after 3s; `streamCount > 0` (connection open) |

**Keepalive test coordination pattern** (test 7 — different from the live-stream pattern):

A COMMAND must be dispatched first: without it, `streamTask()` returns immediately with
"task not found", fires the event handler, and the test self-defeats.

```
1. QuarkusTransaction.requiringNew() → create channel + dispatch COMMAND (establishes task)
   corrId = taskId (no terminal dispatch — task stays in-progress)
2. Open SseEventSource, register event listener (latch NOT used — we assert no event fires)
3. Awaitility.await().atMost(2s).until(() -> a2aBackend.streamCount(corrId) > 0)
4. Thread.sleep(3_000)  // > heartbeat-interval-seconds=1; at least 3 keepalives should fire
5. assertThat(events).isEmpty()                          // comments not delivered to handler
6. assertThat(a2aBackend.streamCount(corrId)).isGreaterThan(0)  // connection still open
// client.close() in finally — triggers server-side sink.isClosed() → loop exits
```

Immediate-close tests (first four) do not need Awaitility — the server closes immediately
and the `SseEventSource` delivers the event in the normal client-receive flow.

---

## Edge Cases

### Race: message dispatched between initial read and consumer registration

Between step 2 (`QuarkusTransaction.requiringNew()` reads non-terminal state) and step 3
(`registerStream()`), a terminal message could be dispatched and missed by the consumer.
The re-check at step 4 catches the DB-visible case: if the message committed before the
re-check transaction starts, the re-check sees terminal state and exits cleanly. The
true residual window is messages dispatched and committed between step 2 and step 3 that
are also not yet DB-visible at step 4's read time — this window is measured in
microseconds. Accepted as an inherent limitation.

Once the consumer (`queue::offer`) is registered in step 3, all subsequent messages go
into the queue regardless of when the re-check runs. The queue absorbs concurrent
dispatches and `poll()` picks them up immediately — there is no "queue vs. re-check" race.

### Server restart

SSE subscriptions do not survive restarts — unchanged from #147.

### Reactive path

`ReactiveA2AResource` is unchanged — no reactive equivalent of this issue is in scope.

---

## Not In Scope

- Reactive path (`ReactiveA2AResource`)
- Persistent A2A channel participation (restart recovery)
- Backpressure for high-frequency message streams

---

## Impact Summary

| Component | Change |
|-----------|--------|
| `A2AResource.streamTask()` | Replace `@Transactional` with `@RunOnVirtualThread`; rewrite handler body with programmatic tx + active loop |
| `A2AResource.sendStatusEvent()` | Synchronous await + explicit close (private helper, VT-only caller) |
| `A2AResource.sendErrorEvent()` | Same as above |
| `QhorusConfig.A2a` | Add `SseSettings` sub-interface (renamed from `Sse` to avoid shadowing JAX-RS `Sse`) |
| `A2ATaskState` | Add `TERMINAL_STATES` string set; update existing terminal check call-site |
| `A2AEnabledProfile` | Add SSE config overrides (`heartbeat-interval-seconds=1`, `max-duration-seconds=30`) |
| `A2AChannelBackend` | **Zero changes** |
| `A2AChannelBackendSseTest` | **Zero changes** |
| `A2AStreamTaskTest` | **Deleted** — all tests migrated |
| `A2AStreamIntegrationTest` | New — 7 test cases covering all SSE paths |
| `runtime/pom.xml` | Add test-scope: `quarkus-rest-client-reactive` (provides `ClientBuilder` + `SseEventSource` impl — `quarkus-rest` is server-only), `awaitility` (managed in Quarkus BOM; no version needed) |
