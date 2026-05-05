---
layout: post
title: "From Addressing to Human Control"
date: 2026-04-14
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [quarkus, mcp, multi-agent, hitl, a2a]
---

Today I set out to close phases 6 through 10 of Qhorus — addressing, A2A compatibility, and human-in-the-loop controls. Forty-two commits later, the test count sits at 439.

The addressing work (Phase 6) ended up cleaner than I expected. The design question was where to enforce targeting: at write time or read time. Read-side filtering made more sense — a single private method handles all three dispatch modes without touching the message write path:

```java
private boolean isVisibleToReader(Message m, String readerInstanceId) {
    if (readerInstanceId == null || readerInstanceId.isBlank()) return true;
    if (m.messageType == MessageType.EVENT) return true; // telemetry is always broadcast
    if (m.target == null) return true;
    if (m.target.equals("instance:" + readerInstanceId)) return true;
    if (m.target.startsWith("capability:") || m.target.startsWith("role:")) {
        List<String> tags = instanceService.findCapabilityTagsForInstance(readerInstanceId);
        return tags.contains(m.target);
    }
    return false;
}
```

The convention: agents register with qualified tags — `"capability:code-review"` or `"role:reviewer"` — and the full target string is the lookup key. `instance:alice` matches directly. EVENT messages bypass the filter because observer telemetry has no business being directed at anyone in particular.

One design decision worth noting: if Alice sends to `role:reviewer`, both Alice and Bob see the message — but the BARRIER still requires each to independently write their own contribution. The broadcast is a visibility filter, not a fan-out. The BARRIER cares who sent a message, not who can read it.

Phase 9 added A2A compatibility — `POST /a2a/message:send` and `GET /a2a/tasks/{id}`, disabled by default behind a config flag. Claude and I hit one gotcha immediately: RestAssured URL-encodes the colon in `message:send` to `%3A`. Claude spotted it from the test log — the request URL was `http://localhost:8081/a2a/message%3Asend`. RFC 3986 permits colons in path segments after the first, but RestAssured doesn't trust it. The fix is `given().urlEncodingEnabled(false)` on every call to that path.

Phase 10 was the longest stretch. Eight HITL tool groups: pause/resume channels, an approval gate with discovery tools (`list_pending_approvals`, `respond_to_approval`), wait cancellation, force-release for stuck BARRIERs, artefact revocation, message deletion, channel clearing, instance deregistration, a channel digest for human dashboards, and watchdog alerting.

## Testing Friction — Three Patterns We Won't Forget

`@TestTransaction` plus RestAssured HTTP doesn't work. The test transaction is uncommitted when the HTTP handler fires — the handler runs in a separate transaction and sees nothing. We learned this on the A2A tests and it applies everywhere you mix injected calls with RestAssured.

For concurrent testing — cancel-while-blocking, specifically `cancel_wait` while `wait_for_reply` is in its poll loop — raw `ExecutorService` loses Quarkus CDI context, silently breaking `@Transactional`. The fix is `@Inject ManagedExecutor` from MicroProfile Context Propagation. It propagates CDI context to spawned threads. Not documented anywhere I could find easily.

The watchdog work introduced a pattern I'll use again. Extract all evaluation logic into a separate `@ApplicationScoped` service. The `@Scheduled` method becomes a one-liner that delegates to it. Tests inject the service and call `evaluateAll()` directly — no scheduler timing, fully deterministic. Obvious in hindsight; not where most people start.

Phases 11 and 12 remain: access control and structured observability. The mesh is already useful enough to embed.
