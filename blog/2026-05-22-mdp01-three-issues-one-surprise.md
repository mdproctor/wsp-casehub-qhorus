---
layout: post
title: "Three small issues and a build that wouldn't start"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
---

Every external backend in the qhorus gateway was solving its own restart recovery
independently. The moment I saw that pattern, I knew the framework was exporting
a problem it should own.

But before that — the build wouldn't compile.

`ActorType` and `ActorTypeResolver` had moved from `io.casehub.ledger.api.model`
to `io.casehub.platform.api.identity`. Fifty-one files importing from the old
package, all broken. Claude ran a Python replacement script across all of them,
added `casehub-platform-api` as an explicit dependency to `api/pom.xml`, and
the build cleared.

With the build fixed, I found that #145 — A2A rate limiting — was already done.
The issue was filed before #135 landed, when `A2AChannelBackend.receive()` called
`messageService.send()` directly and bypassed the rate limiter. After #135, it
routes through `tools.sendMessage()`, which already applies the full pipeline.
I closed it.

For #161, the implementation was straightforward until the code review flagged it.
A LIKE prefix query with a bound parameter:

```java
jpql.append(" AND name LIKE ?").append(idx++);
params.add(prefix + "%");
```

Safely bound — no SQL injection — but SQL still interprets `_` as "any single
character" inside the bound value. `byNamePrefix("case_123")` would match
`case-123/work`. The in-memory path uses `String.startsWith()`, which is exact.
No error, just wrong results with names containing underscores.

The fix: escape metacharacters and declare the escape character in JPQL:

```java
jpql.append(" AND name LIKE ?").append(idx++).append(" ESCAPE '!'");
params.add(escapeLikePrefix(prefix) + "%");

private static String escapeLikePrefix(String prefix) {
    return prefix.replace("!", "!!").replace("%", "!%").replace("_", "!_");
}
```

For #181: the gateway registry is in-memory and empty after restart. Restoring
the default agent backend is trivial — a startup hook reads all persisted channels
and calls `initChannel()` for each. The question was external backends.

Three options: leave each backend to implement its own recovery (status quo).
Persist the registry to the database (correct but significant scope, and it forces
Qhorus to model which external backends are registered — the wrong place for it).
Or fire a CDI event from `initChannel()` that external backends can observe.

I chose the event. `ChannelInitialisedEvent` is a plain record in
`casehub-qhorus-api` — consuming repos observe it without depending on the runtime
JAR. One event, two firing sites: channel creation and startup recovery. The
external backend doesn't need to know which it's responding to.

One edge: synchronous CDI `Event.fire()` propagates observer exceptions to the
caller. A broken observer in a startup loop aborts everything after it. Each
`initChannel()` call in the hook is individually exception-isolated.

ADR-0008 covers the decision.
