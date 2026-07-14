# CaseHub Qhorus — Session Handover

**Date:** 2026-07-14 — Branch `issue-346-websocket-catchup` closed. #346 implemented and pushed.

---

## Immediate Next Step

Pick new work. Two remaining issues from #163 follow-ups: #165 (SmallRye bridge, M/High). No blocking cross-module work.

## What Was Done

Implemented WebSocket catch-up mechanism (#346). `MessageReceivedEvent` gains `Long messageId` as first field with `fromMessage(Message, String)` static factory — eliminates duplication between `MessageObserverDispatcher` and catch-up replay. `CloudEventMapper` uses `messageId` as CloudEvent ID (sequential, dedup-friendly). `ChannelWebSocketEndpoint` gains `@RunOnVirtualThread` catch-up: `lastEventId` query parameter triggers replay via `CrossTenantMessageStore.scan(MessageQuery.poll(...))`, server-side buffering in `WebSocketConnectionRegistry` (`subscribeCatchingUp`/`completeCatchUp`/`tryBufferForCatchUp`/`cancelCatchUp`), truncation detection with configurable max-messages (default 500). Adversarial design review (4 rounds, 11 issues) caught a real race condition — original client-side dedup replaced with server-side buffering to guarantee ordered delivery.

## Cross-Module

**We're blocking:**
- `connectors` — needs Space API for space-aware channel grouping (connectors#67)
- `engine` — needs Space for normative channel layout integration; engine tests need `messageId` update for new `MessageReceivedEvent` constructor
- `blocks` — needs all for end-to-end integration (blocks#49)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #165 | SmallRye Reactive Messaging bridge for MessageObserver | M | High | Alternative to explicit transports — lower priority |

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-14-websocket-catchup-design.md` |
| Landed commit | `5174768c` on main |
| Design review | `~/adr/casehub-qhorus/websocket-catchup-20260714-120059/tracker.md` |
