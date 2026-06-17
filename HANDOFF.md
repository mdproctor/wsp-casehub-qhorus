# CaseHub Qhorus — Session Handover
**Date:** 2026-06-17 — Active SSE model shipped (#278 closed, #277 closed)

---

## Immediate Next Step

Run `/work` to start #261 (casehub-qhorus-slack-channel module). Main is clean, both repos aligned.

## What's Left

- `#261` — casehub-qhorus-slack-channel module (SlackChannelBackend) · L · Med
  - Requires brainstorming first
  - casehub-connectors-slack-bot 0.2-SNAPSHOT already published to GitHub Packages
- `parent#265` — sync casehub-qhorus deep-dive in parent docs (active SSE model, @RunOnVirtualThread, SseSettings, TERMINAL_STATES, #278 open-issue ref stale) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #261 | casehub-qhorus-slack-channel module | L | Med | Next up; needs brainstorm |
| parent#265 | Sync parent docs deep-dive | XS | Low | Filed this session |

## What Was Done This Session

**#278, #277 closed — active virtual-thread SSE model:**

`streamTask()` rewritten from a passive callback model (register consumer + return) to an
active virtual-thread model (`@RunOnVirtualThread`, `LinkedBlockingQueue.poll()` loop):
- Keepalive: `queue.poll(heartbeatMs)` timeout → named `event: keepalive` (not SSE comments —
  RESTEasy SseEventSource non-compliantly fires handlers for comment-only frames)
- Orphan detection: `sink.isClosed()` checked at top of every iteration
- Max-duration: `casehub.qhorus.a2a.sse.max-duration-seconds` (default 1800s)
- Config: `casehub.qhorus.a2a.sse.heartbeat-interval-seconds` (default 15s)
- `ensureRegistered()` called from `streamTask()` — SSE stream self-registering
- 7 integration tests via `SseEventSource` replacing RestAssured `A2AStreamTaskTest`
- 3 new/revised protocols in `docs/protocols/casehub/`
- 7 garden entries (SSE/VT/queue gotchas and techniques)

Key gotcha: `TrustGateTest.dispatch_command_allowsObligorAboveThreshold` fails on main —
pre-existing, not introduced by this branch (verified by stashing all changes and running
the test on clean main).

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-17-mdp01-three-problems-one-queue.md` |
| Spec | `docs/specs/2026-06-16-sse-keepalive-active-model.md` (promoted to project) |
| Protocols | `docs/protocols/casehub/sse-active-model-virtual-thread.md`, `sse-keepalive-named-event.md`, `sse-sink-async-close.md` (revised) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
