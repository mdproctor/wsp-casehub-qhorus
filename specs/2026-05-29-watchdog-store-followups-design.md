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
    watchdogStore.put(w);   // ← route through store seam
}
```

### Design decision: `put()` vs `updateLastFiredAt()`

The issue offered two options. `WatchdogStore.updateLastFiredAt(UUID, Instant)` is
semantically precise but adds a new method to three implementations (`JpaWatchdogStore`,
`InMemoryWatchdogStore`, `InMemoryReactiveWatchdogStore`) for marginal readability benefit.
`put(w)` is the established upsert pattern and is sufficient — the field mutation on the
preceding line already expresses what changed. No interface extension needed.

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

Replace with: `// Stream directly — scan() applies limit, which would produce wrong counts for unlimited queries.`

The improved wording explains the consequence positively (why we stream directly) rather
than as a prohibition (do not do X).

---

## #209 — WatchdogEvaluationServiceTest improvements

### 1. Explicit channel ID capture

All three test methods currently discard the return value of `channelStore.put(notifCh)`.
`InMemoryChannelStore` mutates the input object's `id` field, making this safe today.
But the `ChannelStore` contract does not guarantee input mutation — the return value is
the authoritative reference to the persisted channel.

Change in every test:
```java
// Before
channelStore.put(notifCh);
// After
notifCh = channelStore.put(notifCh);
```

Apply to all `channelStore.put()` calls in the test class, including `queueCh` in
`evaluateQueueDepth_firesAlert_...`.

### 2. Negative case tests

The existing three tests only verify conditions fire when met. Add three negative cases
matching the issue specification:

- `evaluateApprovalPending_noAlert_whenNoOpenCommitmentsWithExpiry()` — watchdog
  registered, no commitment in store → `messageStore.scan(notifCh)` returns empty
- `evaluateAgentStale_noAlert_whenNoStaleInstances()` — watchdog registered, no
  stale instance in store → no alert
- `evaluateQueueDepth_noAlert_whenBelowThreshold()` — watchdog registered with
  `thresholdCount=5`, only 2 messages in queue channel → no alert

Each negative test asserts `assertTrue(alerts.isEmpty(), "<descriptive message>")`.

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
| `runtime/.../watchdog/WatchdogEvaluationServiceTest.java` | #209 | ID capture, 3 new tests, assertion messages |

No new interfaces, no new methods, no Flyway migrations, no cross-repo impact.
