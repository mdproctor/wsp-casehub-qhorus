# CaseHub Qhorus — Session Handover
**Date:** 2026-06-12 — S/XS batch closed (#269, #268, #267, #274, #245, #241, #229, #254, #251)

---

## What Was Done This Session

**S/XS batch (9 issues, all closed):**

- **#269** `QhorusInboundCurrentPrincipal` changed from `@Alternative @Priority(1)` to `@DefaultBean` — resolves CDI ambiguity when `casehub-platform-testing` (`FixedCurrentPrincipal`) is on the classpath. All four test `application.properties` updated with `quarkus.arc.exclude-types=MockCurrentPrincipal`.
- **#254** `ChannelService.create()` now calls `channelGateway.initChannel()` — runtime-created channels visible to ChannelBackend dispatch. Earlier attempt (direct CDI event fire) caught in code review: it skipped `registry.computeIfAbsent()` step 1. Redundant `initChannel()` calls removed from MCP tools.
- **#251** `CommitmentDeclinedEvent` CDI record in `api/`; `CommitmentService.decline()` fires it. Null guard on field (CDI-free unit tests don't wire it). Filed #275 for cleaner alternative.
- **#229** V22 Flyway: `CREATE INDEX idx_commitment_obligor ON commitment (obligor)`.
- **#245** Already fixed in #243 — closed without code change.
- Remaining: #268 IOException, #241 dead glob, #274 null connectorId guard, #267 tenant isolation test — all mechanical, all closed.

**Garden:** GE-20260609-9ee2ad revised to `resolved` (the entry had predicted this exact fix).

## Immediate Next Step

Run `/work` to pick up the next issue. Main is clean.

## What's Left

- `#273` — ACL operator docs for ConnectorMeshBridge delivery channel (role:system requirement) · XS · Low
- `#275` — null guard cleanup on CommitmentService.declinedEvents · XS · Low
- connectors#19 — ConnectorMeshBridge javadoc rewrite (EVENT→STATUS) · XS · Low
- parent#228 — casehub-qhorus deep-dive sync (ConnectorQhorusMeshBridge, ChannelService.create()) · XS · Low
- parent#230 — casehub-qhorus deep-dive sync (@DefaultBean, CommitmentDeclinedEvent) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| casehub-ledger#126 | Full EVENT telemetry decoupling | M | Med | Unblocks EVENT content-bearing; unowned backlog |
| #271 | allowedTypes advisory enforcement (warn not hard-block) | M | Med | Informatory role concept gives theoretical grounding |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-12-mdp02-nine-fixes-two-surprises.md` |
| Protocol: @DefaultBean + exclude-types pattern | see CLAUDE.md testing conventions |
| Protocol: PII/credential exclusion | PP-20260612-bd6f8c |
| Garden: initChannel resolved | GE-20260609-9ee2ad |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
