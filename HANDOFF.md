# CaseHub Qhorus — Session Handover

**Date:** 2026-07-13 — Branch `issue-163-cluster-scoped-observer` closed. #163 implemented and pushed.

---

## Immediate Next Step

Only infrastructure issues remain (#165 SmallRye bridge, open but lower priority after #163 shipped the explicit transport approach). Pick new work or start a new initiative. `drafthouse#102` (redundant `.toString()` on correlationId) is still open as a cross-repo follow-up.

## What Was Done

Implemented CLUSTER-scoped MessageObserver dispatch (#163). Foundation: gave `Scope.CLUSTER` real dispatch semantics — LOCAL fires on dispatching node only, CLUSTER fires on all nodes via `deliverRemote()` → `MessageService.dispatchClusterObservers()`. Fixed LAST_WRITE overwrite path to fire observers (was excluded, creating node-asymmetric behavior). Built three optional transport modules: `kafka-observer` (CloudEvents to Kafka topic, scope LOCAL), `websocket-observer` (real-time push to browser clients, scope CLUSTER), `webhook-observer` (HTTP POST callbacks with HMAC-SHA256, scope CLUSTER). Extracted `CloudEventMapper` as shared utility. Design review caught 12 issues — all verified and fixed (LAST_WRITE asymmetry, tenant isolation via channelId, package visibility, webhook scope change LOCAL→CLUSTER). Deferred items filed as #345 (webhook persistent registrations) and #346 (WebSocket catch-up mechanism).

## Cross-Module

**We're blocking:**
- `connectors` — needs Space API for space-aware channel grouping (connectors#67)
- `engine` — needs Space for normative channel layout integration
- `blocks` — needs all for end-to-end integration (blocks#49)

**Cross-repo follow-ups still open:**
- drafthouse#102 — redundant `.toString()` on correlationId

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #345 | Webhook persistent registrations (JPA) | S | Low | Currently in-memory, lost on restart |
| #346 | WebSocket catch-up mechanism for reconnecting clients | M | Med | Clients miss messages during disconnect |
| #165 | SmallRye Reactive Messaging bridge for MessageObserver | M | High | Alternative to explicit transports — lower priority now |

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-13-cluster-scoped-observer-design.md` |
| Landed commit | `3f9bd83f` on main |
| Design review | `~/adr/qhorus/cluster-scoped-observer-*/tracker.md` |
