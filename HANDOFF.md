# CaseHub Qhorus — Session Handover
**Date:** 2026-05-21 — #171 CommitmentStore.deleteAll; #174 Flyway path relocation

---

## What Was Done This Session

**#171** — `delete_channel` was missing `commitmentStore.deleteAll(ch.id)` before channel deletion. `fk_commitment_channel` has no CASCADE, so any channel with open commitments would produce an FK violation. Added `deleteAll(UUID channelId)` to `CommitmentStore` interface, `JpaCommitmentStore`, `InMemoryCommitmentStore`, and both MCP tool implementations. TDD throughout. Closed #171.

**#174** — Flyway's recursive classpath scan on `classpath:db/migration` picks up subdirectories, so `db/migration/qhorus/` was visible to any app's default datasource. Moved all 11 migration files to `db/qhorus/migration/`, updated `application.properties` and `FlywayMigrationSchemaTest`. Also corrected Rule 4 in the platform's `flyway-version-range-allocation.md` protocol — the old rule said `db/migration/<module>/` which is still inside the scan root. Closed #174.

**Garden:** 2 entries submitted — obligor/requester semantics (GE-20260521-e39ad1), Flyway recursive subdirectory scan (GE-20260521-effd2f).

## Current State

- **Both repos:** `main` — #171 and #174 merged and pushed to `origin`
- **Tests:** 1107 passing, 44 skipped
- **Next Flyway domain migration:** V11

## Immediate Next Step

Verify **#168** ("reduce selected-alternatives maintenance burden in consumers") — our `@Alternative` removal from reactive service beans in #172 directly addresses it. Check and close if resolved.

## What's Left

- **#168** — verify and close if addressed by #172's @Alternative removal · XS · Low
- **Plan B (#172)** — file issue, convert Category B `@Blocking @Transactional` tools in `ReactiveQhorusMcpTools` to pure `Uni<T>` · L · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #168 | Verify and close — @Alternative removal in #172 | XS | Low | First check |
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| Plan B | Category B reactive tool conversion in ReactiveQhorusMcpTools | L | High | File issue first; spec exists |
| clinical#16 | PiResponseListener workaround removal | S | Low | Unblocked since #154 |
| claudony#117 | ClaudonyChannelBackend | M | Med | Unblocked since qhorus#131 + #153 |

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-21-mdp02-subdirectory-wasnt-scoped.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| #172 spec | `specs/issue-172-reactive-tier-separation/2026-05-19-reactive-tier-separation-design.md` |
