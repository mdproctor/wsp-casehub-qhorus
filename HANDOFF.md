# CaseHub Qhorus — Session Handover

**Date:** 2026-07-13 — Branch `issue-343-watchdog-test-fix-and-topic-projections` closed. #343 (already fixed), #338 implemented and pushed.

---

## Immediate Next Step

Epic #328 backlog is clear — all conversation model enrichment issues closed. Only infrastructure-level issues remain (#163, #165). Pick new work or start a new epic.

## What Was Done

Closed #343 without code changes — the consistent test failure (ChannelDigestTest.digestShowsResolvedTopicStatus, mislabeled as WatchdogAlertE2ETest in the issue title) was already fixed by the CDI proxy field-access fix in #337. Implemented #338: topic-aware projections. Fixed a silent bug in MessageQueryJpql (JPA stores ignored the topic field — in-memory stores filtered correctly). Added MessageView.topic field. Added topic parameter to project_channel MCP tool with blank normalization and reactive parity. Design-reviewed (3 rounds, 12 issues, all verified, $9.84). Garden entry GE-20260713-26f881 for the MessageQueryJpql parity gap.

## Cross-Module

**We're blocking:**
- `connectors` — needs Space API for space-aware channel grouping (connectors#67)
- `engine` — needs Space for normative channel layout integration
- `blocks` — needs all for end-to-end integration (blocks#49)

**Cross-repo follow-ups still open:**
- claudony#169 — OutboundMessage correlationId UUID→String
- drafthouse#102 — redundant `.toString()` on correlationId

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #165 | SmallRye Reactive Messaging bridge for MessageObserver | M | High | Infrastructure — Kafka/AMQP bridge |
| #163 | CLUSTER-scoped MessageObserver — Kafka, WebSocket, Webhook | L | High | Cross-node observer delivery |

## References

| What | Path |
|------|------|
| Blog | `blog/2026-07-13-mdp02-silent-filter-missing-field.md` |
| Garden | `GE-20260713-26f881` — MessageQueryJpql parity gap |
| Spec | `specs/2026-07-13-topic-aware-projections-design.md` |
