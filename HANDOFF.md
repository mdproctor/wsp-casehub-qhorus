# HANDOFF — casehub-qhorus

## Last Session (2026-11-03)

**Branch:** `issue-509-channel-visibility-semantics` — implementation complete, tests outstanding.

### What Was Built

Added `ChannelVisibility` (PUBLIC, PRIVATE, DIRECT) as a typed enum field on `Channel`, with:

- **DIRECT channels:** deterministic `dm/{sorted-a}/{sorted-b}` naming, exactly 2 immutable members, count-based join guard in `ChannelMembershipService`
- **Group channels:** `group/{slug}` naming with `boundGroupId` (first-class UUID field, indexed) linking to eidos org units
- **VisibilityManager interface** (4 methods): `getOrCreateDirect`, `findDirectChannelsFor`, `createGroupChannel`, `findGroupChannel`
- **Creation-time defaults:** PUBLIC→BROADCAST/OPEN, PRIVATE/DIRECT→APPEND/ENFORCE (overridable)
- **REST API:** `/api/visibility/direct`, `/api/visibility/group`, `PUT /api/channels/{id}/visibility`, `GET /api/channels?visibility=X`
- **BroadcastMembershipManager** stamps new broadcast channels with `visibility=PUBLIC`
- **Flyway V61:** `visibility VARCHAR(20)` + `bound_group_id UUID` columns with indexes

### Code Review Findings — All Addressed

| # | Fixed | Description |
|---|-------|-------------|
| C1 | Yes | REST path collision — VisibilityResource moved to `/api/visibility` |
| C2 | Yes | Spring RuntimeAutoConfiguration uses 4-arg constructor with ChannelStore |
| H1 | Yes | ChannelConfigBootstrap wires visibility/boundGroupId/participationPolicy from YAML |
| H2 | Yes | ChannelDetail + QhorusEntityMapper updated with visibility/boundGroupId |
| M1 | **No** | Zero test files written — this is the outstanding work |
| M2 | Yes | Test name renamed to `semanticDefaultsToNull` |

### What's Left

**M1: Tests.** No tests were written. The next session should write:

1. CDI-free unit tests in `testing/` — `ChannelMembershipServiceDirectGuardTest`, `VisibilityServiceTest`
2. `@QuarkusTest` integration tests in `runtime/` — `ChannelVisibilityTest`, `VisibilityResourceTest`
3. **Known issue:** `OrgUnitIdLedgerTest` (from #505) blocks JUnit discovery after `mvn clean`. Workaround: `mvn install -pl api,runtime-core,persistence-memory,testing,runtime -DskipTests` before running tests.

### Design Artifacts (workspace)

- Spec: `specs/issue-509-channel-visibility-semantics/2026-11-02-channel-visibility-semantics-design.md`
- Decisions: `specs/issue-509-channel-visibility-semantics/decisions.md` (7 decisions)
- Plan: `plans/2026-11-03-channel-visibility-semantics.md`
- Blog: `blog/2026-11-03-mdp01-when-agents-need-mailboxes.md`
