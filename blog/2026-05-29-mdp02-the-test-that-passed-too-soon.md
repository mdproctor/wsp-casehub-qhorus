---
layout: post
title: "The test that passed too soon"
date: 2026-05-29
entry_type: note
subtype: diary
projects: [qhorus]
tags: [casehub, qhorus, store-pattern, jpa, testing]
---

The previous entry closed three Panache static calls that were bypassing the store seam in `WatchdogEvaluationService`. What it didn't close: line 100, where `w.lastFiredAt = now` directly mutated the entity field.

After the fix it's two lines:

```java
w.lastFiredAt = now;
watchdogStore.put(w);
```

The protocol requires it. Services route all persistence through the `*Store` interface — they don't rely on JPA dirty-tracking to clean up. It works today because JPA does clean it up, within the `@Transactional` boundary. A non-JPA store implementation would silently stop persisting the debounce timestamp and watchdogs would fire on every evaluation cycle, indefinitely.

The interesting part is what happens when you try to test it.

The natural test for debounce correctness is: call `evaluateAll()` twice, assert one alert. If `lastFiredAt` isn't persisted, the second call fires again. Clear invariant, clear test.

The problem: in a `@QuarkusTest @TestTransaction` context, this test passes before the fix is applied. JPA's `FlushModeType.AUTO` triggers a flush before the second `Watchdog.list()` JPQL query, pushing the dirty `lastFiredAt` into the session. The second scan reads the updated value. The debounce works — not because the store was called, but because JPA was covering the gap.

You cannot use a `@TestTransaction` integration test to distinguish between "store-seam compliance" and "JPA dirty-tracking happens to work here." Both produce the same observable result. The debounce test documents the behavioral invariant correctly — two calls within the window produce one alert — but it can't prove the mechanism. The fix is a structural rule, not a behavioral one, and structural rules don't always have structural tests at the integration layer.

The code review caught something else. The AGENT_STALE negative test was structured wrong: it added no instances to the store, which tests that an empty store returns nothing — trivially true regardless of how the query filter is implemented. Claude flagged this: a useful negative test needs a non-matching instance, one with `status = "online"`. That exercises the predicate, not the empty-list fast path. Easy to miss when writing from a happy-path mental model.

The comment on `InMemoryMessageStore.count()` had the same character of flaw. "For unlimited queries" implied that delegating to `scan()` would be safe when a limit is set — which isn't true. `scan()` enforces `query.limit()` regardless of whether the query requests a limit. The fix removed the qualifier; the comment now just says what happens: `scan()` would truncate the count.

Small things. But a test that doesn't fail when it should is worse than no test — it teaches the wrong lesson.
