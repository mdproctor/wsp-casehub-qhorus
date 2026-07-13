# CaseHub Qhorus — Session Handover

**Date:** 2026-07-13 — Branch `issue-344-channel-create-request-compat` closed. #344 fixed and pushed.

---

## Immediate Next Step

Epic #328 backlog clear — all conversation model enrichment issues closed, plus #344 (cross-repo compat fix). Only infrastructure-level issues remain (#163, #165). Pick new work or start a new epic.

## What Was Done

Fixed #344: added a 14-param backward-compatible constructor to `ChannelCreateRequest` that defaults `spaceId` to `null`. This unblocks downstream consumers (clinical `ProtocolDeviationService`, ops `ChannelProvisionHandler`) broken by the 15th param added in #334. CLAUDE.md updated with compat constructor documentation. One of 8 SNAPSHOT breakages tracked by engine#719 — the qhorus-specific one is now resolved.

## Cross-Module

*Updated: claudony#169 closed — removed from backlog.*

**We're blocking:**
- `connectors` — needs Space API for space-aware channel grouping (connectors#67)
- `engine` — needs Space for normative channel layout integration
- `blocks` — needs all for end-to-end integration (blocks#49)

**Cross-repo follow-ups still open:**
- drafthouse#102 — redundant `.toString()` on correlationId

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #165 | SmallRye Reactive Messaging bridge for MessageObserver | M | High | Infrastructure — Kafka/AMQP bridge |
| #163 | CLUSTER-scoped MessageObserver — Kafka, WebSocket, Webhook | L | High | Cross-node observer delivery |

## References

| What | Path |
|------|------|
| Fix commit | `6ab760db` on main |
