# HANDOFF — casehub-qhorus

## Last Session (2026-11-03)

**Branch:** `issue-509-channel-visibility-semantics` — implementation and tests complete.

### What Was Built

Added `ChannelVisibility` (PUBLIC, PRIVATE, DIRECT) as a typed enum field on `Channel`, with:

- **DIRECT channels:** deterministic `dm/{sorted-a}/{sorted-b}` naming, exactly 2 immutable members, count-based join guard in `ChannelMembershipService`
- **Group channels:** `group/{slug}` naming with `boundGroupId` (first-class UUID field, indexed) linking to eidos org units
- **VisibilityManager interface** (4 methods): `getOrCreateDirect`, `findDirectChannelsFor`, `createGroupChannel`, `findGroupChannel`
- **Creation-time defaults:** PUBLIC→BROADCAST/OPEN, PRIVATE/DIRECT→APPEND/ENFORCE (overridable)
- **REST API:** `/api/visibility/direct`, `/api/visibility/group`, `PUT /api/channels/{id}/visibility`, `GET /api/channels?visibility=X`
- **BroadcastMembershipManager** stamps new broadcast channels with `visibility=PUBLIC`
- **Flyway V61:** `visibility VARCHAR(20)` + `bound_group_id UUID` columns with indexes

### Tests (46 total, all pass)

| Test class | Location | Count | Type |
|------------|----------|-------|------|
| `ChannelMembershipServiceDirectGuardTest` | `testing/src/test/` | 9 | CDI-free |
| `VisibilityServiceTest` | `runtime/src/test/` | 15 | CDI-free (Mockito) |
| `ChannelVisibilityTest` | `runtime/src/test/` | 13 | @QuarkusTest |
| `VisibilityResourceTest` | `runtime/src/test/` | 9 | @QuarkusTest |

### Bugs Found and Fixed During Testing

| Bug | Fix |
|-----|-----|
| DIRECT join guard fired before existing-member check, blocking role updates | Moved existing-member lookup before DIRECT guard |
| `JpaChannelStore.scan()` missing visibility/boundGroupId JPQL clauses | Added filter clauses to JPA query builder |
| `VisibilityResource` POST methods missing `@Transactional` | Added `@Transactional` to `createDirect` and `createGroup` |
| `VisibilityService` missing no-args constructor for CDI proxying | Added `protected` no-args constructor |
| `CreateChannelRequest` missing visibility/boundGroupId fields | Added fields + wired in `ChannelCore.create()` |
| `ChannelConfigBootstrapTest` StubChannelConfig missing new methods | Added `visibility()` and `boundGroupId()` stubs |

### Code Review Findings — All Addressed

| # | Fixed | Description |
|---|-------|-------------|
| C1 | Yes | REST path collision — VisibilityResource moved to `/api/visibility` |
| C2 | Yes | Spring RuntimeAutoConfiguration uses 4-arg constructor with ChannelStore |
| H1 | Yes | ChannelConfigBootstrap wires visibility/boundGroupId/participationPolicy from YAML |
| H2 | Yes | ChannelDetail + QhorusEntityMapper updated with visibility/boundGroupId |
| M1 | **Yes** | 46 tests written across 4 test classes |
| M2 | Yes | Test name renamed to `semanticDefaultsToNull` |

### What's Left

Branch is ready for `work end`. All implementation batches complete, all tests pass, all code review findings addressed.

### Design Artifacts (workspace)

- Spec: `specs/issue-509-channel-visibility-semantics/2026-11-02-channel-visibility-semantics-design.md`
- Decisions: `specs/issue-509-channel-visibility-semantics/decisions.md` (7 decisions)
- Plan: `plans/2026-11-03-channel-visibility-semantics.md`
- Blog: `blog/2026-11-03-mdp01-when-agents-need-mailboxes.md`
