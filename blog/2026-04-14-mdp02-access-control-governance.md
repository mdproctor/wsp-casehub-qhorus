---
layout: post
title: "Locking Down the Mesh"
date: 2026-04-14
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [quarkus, mcp, multi-agent, access-control, tdd]
---

Phase 11 started with housekeeping. The previous session had shipped all the HITL work but left fifteen child issues from phases 6, 9, and 10 still open on GitHub. I closed those, then built four issues under epic #45: write permissions, admin role, rate limiting, and observer mode. Claude and I worked through all four in one session, strict TDD throughout — test first, watch it fail, implement the minimum to pass.

## Write Permissions — Reusing the Read-Side Logic

The first feature (#46) added an `allowed_writers` column to `Channel`. Entries are comma-separated: bare instance IDs, `capability:tag` patterns, or `role:name` patterns. `send_message` checks the list before accepting; EVENT messages bypass it entirely.

The matching logic was nearly free. The read-side dispatch already resolved capability and role patterns against a sender's registered tags. We extracted the same logic onto the write path as `isAllowedWriter` — a private helper that iterates the ACL, checks the sender's tags, and returns a boolean. New tool: `set_channel_writers` for post-creation updates. Twenty-three tests.

## Admin Role — The @Tool Signature Problem

Issue #47 added `admin_instances` — instance IDs permitted to invoke `pause_channel`, `resume_channel`, `force_release_channel`, and `clear_channel`. When the list is set, callers not in it are rejected.

Adding a `caller_instance_id` parameter to existing tools would break every existing test. Two `@Tool` annotations with the same name aren't allowed. The solution: non-`@Tool` convenience overloads that delegate:

```java
/** Convenience overload — no caller identity (open governance assumed). */
public ChannelDetail pauseChannel(String channelName) {
    return pauseChannel(channelName, null);
}

@Tool(name = "pause_channel", ...)
public ChannelDetail pauseChannel(String channelName, String callerInstanceId) { ... }
```

The `@Tool` annotation lives on the full signature; old tests call the one-arg version without modification. We used this pattern across all four Phase 11 features — `createChannel` grew from four to eight parameters with zero test churn.

## Rate Limiting — The UUID in the Error Message

Issue #48 added per-channel and per-instance message throttling. The `RateLimiter` bean tracks send timestamps in in-memory `ConcurrentHashMap<UUID, Deque<Instant>>` windows, pruned on the request path. No scheduler, no database writes.

First test run produced one failure. The test asserted:

```java
assertTrue(ex.getMessage().contains("rl-ch-3"), "error should name the channel");
```

The actual message: `"Rate limit exceeded for channel 'f96dd562-7627-4e63-a41b-...'"`. The `check()` method only received the UUID — it had no channel name. Threading `channelName` through alongside `channelId` fixed it. Obvious after the fact; the test assertion on the exact error text is what surfaced it. A manual smoke test would have missed it entirely.

## Observer Mode — When check_messages Says Nothing

Issue #49 added read-only observers. `register_observer` stores subscriptions in an `ObserverRegistry` bean — no `Instance` row, excluded from `list_instances`, no write access. Registered observers call `read_observer_events` to receive EVENT messages.

The E2E test sent five messages to an APPEND channel — three agent messages, two EVENTs — then called `check_messages` and expected five back. It got three. The reason: `check_messages` excludes EVENT messages from all delivery paths by design, an invariant documented in CLAUDE.md as "agent context is never polluted with telemetry." EVENTs live in the database; they just don't come back through the standard read path. The test was asserting the wrong thing. `read_observer_events` is the correct path for EVENTs, and once the assertion targeted that, it passed immediately.

Phase 11 closes with 521 tests passing. Four new MCP tools, three new Flyway migrations (V5–V7), two new in-memory beans. Phase 12 — structured observability — is next.
