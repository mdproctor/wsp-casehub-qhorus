# CaseHub Qhorus — Session Handover

**Date:** 2026-07-13 — Branch `issue-337-topic-digest-and-fixes` closed. #337, #341, #342 implemented and pushed.

---

## Immediate Next Step

All three issues closed. Epic #328 backlog is clear. Pick from What's Next — #338 (topic-aware projections) is the natural continuation. Start with `/work` when ready.

## What Was Done

Three issues on one branch: telemetryJson() utility eliminates string-concatenated JSON in 5 MCP tool sites (#341). CommitmentContext enriched with artefactUuid, expectedToken, content — V1/V4 evidential checks now fire through the attestation path, StoredCommitmentAttestationPolicy downgrades DONE to FLAGGED on violations (#342). get_channel_digest includes topicBreakdown with per-topic message count, last activity, and resolved status (#337). Also fixed pre-existing CDI proxy field-access bug in resolveTopic/unresolveTopic and filed #343 for WatchdogAlertE2ETest consistent failure.

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
| #338 | Topic-aware projections | M | Med | Natural pair with #337 |
| #343 | WatchdogAlertE2ETest consistent failure | S | Med | Filed this session |

## References

| What | Path |
|------|------|
| Blog | `blog/2026-07-13-mdp01-three-issues-one-proxy-bug.md` |
| Protocol updated | `docs/protocols/casehub/commitment-attestation-policy-null-context.md` |
