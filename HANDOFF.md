# CaseHub Qhorus — Session Handover
**Date:** 2026-06-12 — #249 shipped (ConnectorMeshBridge + informatory role concept); #272 fixed (tokenise API)

---

## What Was Done This Session

**#249 — ConnectorQhorusMeshBridge, closed and landed:**

- `ConnectorQhorusMeshBridge @ApplicationScoped`: implements `ConnectorMeshBridge` SPI; posts STATUS (not EVENT) to `casehub.qhorus.connector-backend.delivery-channel` after each successful MCP connector delivery. Sender `"system:connector:{connectorId}"`. Destination excluded (credential/PII — immutable ledger). ConcurrentHashMap channel cache keyed by tenancyId. Fire-and-forget via ManagedExecutor with sync context capture. Full SPI "must never throw" wrap.
- **Protocol PP-20260608-054090 reframed** — "informatory role, not type" defines observe channel membership. STATUS for content-bearing, EVENT for content-free. Observe channel examples corrected to `allowed_types="EVENT,STATUS"` in `agent-mesh-framework.md` and `normative-channel-layout.md`.
- **Protocol PP-20260612-bd6f8c** — credentials and PII must never appear in `MessageDispatch.content` or any ledger-persisted field.
- **PLATFORM.md** (parent) updated: ConnectorMeshBridge entry corrected (STATUS, not EVENT).
- **fix(#272)** found during work-end: `ActorIdentityProvider.tokenise(String, ActorType)` API change from ledger#130 — 4 call sites in `QhorusLedgerEntryRepository` and `ReactiveLedgerEntryJpaRepository` updated. Stale installed jar had hidden the break.
- Issues filed: #273 (ACL operator docs), #274 (null connectorId guard), connectors#19 (javadoc rewrite), parent#228 (deep-dive sync).

## Immediate Next Step

Run `/work` to pick up the next issue. Main is clean, both remotes current.

## What's Left

- `#267` — add commitment-level tenant isolation test for `A2ATenantScopingTest.getTask()` · XS · Low
- `#268` — minor code quality items from #265/#264/#263 review · XS · Low
- `#273` — ACL operator docs for ConnectorMeshBridge delivery channel (role:system requirement, rate-limit note) · XS · Low
- `#274` — null connectorId guard in `ConnectorQhorusMeshBridge` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| casehub-ledger#126 | Full EVENT telemetry decoupling from message.content | M | Med | Follow-up from #257; not blocking |
| #271 | allowedTypes advisory enforcement (warn, not hard-block) | M | Med | Informatory role concept gives theoretical grounding |
| connectors#19 | ConnectorMeshBridge javadoc rewrite — STATUS, not EVENT | XS | Low | Peer-repo; mechanical |
| parent#228 | casehub-qhorus deep-dive: add ConnectorQhorusMeshBridge | XS | Low | Peer-repo |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-12-mdp01-the-observation-that-couldnt-carry-content.md` |
| Spec | `docs/specs/issue-249-connector-mesh-bridge/2026-06-12-connector-mesh-bridge-design.md` (project) |
| Protocol: informatory role | PP-20260608-054090 |
| Protocol: PII/credential exclusion | PP-20260612-bd6f8c |
| Garden: maven api break masked | GE-20260612-1100fe |
| Garden: computeIfAbsent null | GE-20260612-f6362e |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
