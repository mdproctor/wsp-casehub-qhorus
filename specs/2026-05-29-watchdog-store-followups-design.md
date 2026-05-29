# Design: Watchdog Store Seam Followups (#207, #208, #209)

**Date:** 2026-05-29  
**Issues:** casehubio/qhorus#207, #208, #209  
**Branch:** issue-207-watchdog-store-followups  
**Scope:** XS/S cleanup items filed during code review of #205

---

## #207 — Route `evaluateAll()` mutation through WatchdogStore

### Problem

`WatchdogEvaluationService.evaluateAll()` mutates `w.lastFiredAt = now` directly on the
Watchdog entity (line 100) without calling through `WatchdogStore`. This violates the
store-seam protocol (PP-20260529-eb19c3): all persistence state changes in qhorus services
must flow through the injected `*Store` interface, not via direct entity mutation.

It works today because:
- JPA dirty-checking picks up the mutation within the `@Transactional` boundary
- `InMemoryWatchdogStore.scan()` returns object references, so mutation is immediately visible

But any future `WatchdogStore` implementation that does not use JPA managed entities (e.g.
a JDBC or outbox-backed store) would silently drop the debounce timestamp, causing watchdogs
to fire on every evaluation cycle indefinitely.

### Fix

After the existing `w.lastFiredAt = now;` line, add an explicit store call:

```java
if (fired) {
    w.lastFiredAt = now;
    watchdogStore.put(w);   // route through store seam; return value intentionally discarded — w is not used again in this iteration
}
```

The return value of `put(w)` is intentionally discarded — `w` is not referenced again after
this point in the loop iteration.

### Side effect: N flushes per evaluation cycle

`JpaWatchdogStore.put()` calls `watchdog.persistAndFlush()`. For a managed entity (which
every `w` in the loop is, since it came from `Watchdog.list()` within the same `@Transactional`
boundary), `persist()` is a no-op per JPA spec, but `flush()` synchronizes the **entire**
persistence context — not just `w`. In a loop where N watchdogs fire in one cycle, this
produces N full-context flushes instead of one at commit.

This is accepted as-is for the following reasons:
- Watchdog evaluations fire at a configured scheduler interval (default 60s), not on the
  hot path
- The number of simultaneously-firing watchdogs is expected to be small (typically 0–2)
- The alternative (`WatchdogStore.updateLastFiredAt(UUID, Instant)`) adds interface and
  implementation churn for marginal runtime benefit at this scale
- Correctness over performance: the principal concern is store seam compliance

If the watchdog count grows significantly, a bulk-update method can be added then.

### Design decision: `put()` vs `updateLastFiredAt()`

`WatchdogStore.updateLastFiredAt(UUID, Instant)` is semantically precise but adds a new
method to three implementations (`JpaWatchdogStore`, `InMemoryWatchdogStore`,
`InMemoryReactiveWatchdogStore`) for marginal readability benefit over `put(w)`.
`put(w)` is the established upsert pattern and is sufficient.

---

## #208 — Javadoc and comment consistency

### 1. `ReactiveMessageStore.count(MessageQuery)`

`MessageStore.count(MessageQuery)` documents its intentional `long` return type
(vs the legacy `int countByChannel()`). The reactive counterpart `ReactiveMessageStore.count()`
has no Javadoc and should mirror the blocking version's explanation.

Add:
```java
/**
 * Count messages matching the given query. Intentionally {@code long}
 * (Panache count semantics) unlike the legacy {@code int countByChannel}.
 */
Uni<Long> count(MessageQuery query);
```

### 2. `InMemoryMessageStore.count()` comment wording

Current: `// Do NOT delegate to scan() — scan() applies limit, giving wrong counts.`

Replace with: `// Stream directly — scan() enforces query.limit(), which would truncate the count.`

The qualifier "for unlimited queries" (from the first draft) was removed — `scan()` applies
`query.limit()` to all queries regardless of whether limit is set, so delegation would produce
wrong results for any query, not just unlimited ones.

---

## #209 — WatchdogEvaluationServiceTest improvements

### 1. Explicit channel ID capture

All three test methods currently discard the return value of `channelStore.put(notifCh)`.
`InMemoryChannelStore` mutates the input object's `id` field, making this safe today.
But the `ChannelStore` contract does not guarantee input mutation — the return value is
the authoritative reference to the persisted channel.

Change in every test where the channel ID is subsequently used:
```java
// Before
channelStore.put(notifCh);
// After
notifCh = channelStore.put(notifCh);
```

Apply to `channelStore.put(notifCh)` and `channelStore.put(queueCh)` — these IDs appear
in `MessageQuery.forChannel(notifCh.id)` and `m.channelId = queueCh.id` respectively.

The remaining store calls — `watchdogStore.put(w)`, `commitmentStore.save(c)`,
`instanceStore.put(inst)`, `messageStore.put(m)` — have their return values safely
discarded because those IDs are never referenced in assertions or subsequent store calls.
This distinction is intentional, not an oversight.

### 2. Negative case tests

The existing three tests only verify conditions fire when met. Add four negative cases:

**a. `evaluateApprovalPending_noAlert_whenOpenCommitmentHasNoExpiry()`**

Set up a watchdog and a commitment with `expiresAt = null`. The service's
`filter(c -> c.expiresAt != null)` predicate should exclude it, producing no alert.
This exercises the filter logic, not just the trivially-empty-store case.

```java
Commitment c = new Commitment();
c.state = CommitmentState.OPEN;
c.expiresAt = null;  // no expiry — should be excluded by filter
// ... other fields ...
commitmentStore.save(c);
watchdogService.evaluateAll();
assertTrue(alerts.isEmpty(), "APPROVAL_PENDING must not fire when commitment has no expiresAt");
```

**b. `evaluateAgentStale_noAlert_whenNoStaleInstances()`**

Set up watchdog, no stale instances in store → no alert.

**c. `evaluateQueueDepth_noAlert_whenBelowThreshold()`**

Set up queue channel + watchdog with `thresholdCount=5`, add 2 non-EVENT messages → no alert.

**d. `evaluateAll_debounce_preventsRefireWithinWindow()`**

This is the principal behavioral guarantee of #207 — that `lastFiredAt` is persisted
correctly so `isDebounced()` prevents re-firing. A fix can be read and judged correct,
but without this test there is no executable verification.

Test design:
1. Set up watchdog with triggering condition (e.g. QUEUE_DEPTH exceeded)
2. Call `evaluateAll()` → assert one alert message in notification channel
3. Call `evaluateAll()` again (condition still met, but within debounce window)
4. Assert still only one alert — second call was debounced

Note: the watchdog's `thresholdSeconds` controls the debounce window. Set it to a value
that's comfortably larger than the elapsed time between the two `evaluateAll()` calls
(e.g. `thresholdSeconds = 300`). The `isDebounced()` method checks
`lastFiredAt.isAfter(now.minusSeconds(windowSeconds))`, so any positive threshold will
debounce an immediate re-call.

### 3. Richer assertion messages

Replace all current `assertFalse(alerts.isEmpty(), "X watchdog should fire alert")`
messages with condition-specific descriptions, e.g.:

```java
assertFalse(alerts.isEmpty(),
    "APPROVAL_PENDING should trigger when open commitment has expiresAt set and threshold=0");
```

---

## Files changed

| File | Issue | Change |
|------|-------|--------|
| `runtime/.../watchdog/WatchdogEvaluationService.java` | #207 | +1 line: `watchdogStore.put(w)` |
| `runtime/.../store/ReactiveMessageStore.java` | #208 | +4 lines Javadoc |
| `testing/.../InMemoryMessageStore.java` | #208 | comment wording |
| `runtime/.../watchdog/WatchdogEvaluationServiceTest.java` | #209 | ID capture, 4 new tests (3 negative + 1 debounce), assertion messages |

No new interfaces, no new methods, no Flyway migrations, no cross-repo impact.

---

## Out of scope — follow-up issues

Two structural gaps surfaced during review, out of scope for this branch:

- **BARRIER_STUCK / CHANNEL_IDLE zero test coverage** — no positive or negative tests exist
  for these two condition types in `WatchdogEvaluationServiceTest`.
- **`evaluateBarrierStuck()` NPE risk** — `ch.lastActivityAt.isBefore(cutoff)` has a null
  dereference if a channel was created but never received a message and `lastActivityAt` is null.
