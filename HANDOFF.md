# CaseHub Qhorus — Session Handover
**Date:** 2026-05-20 — #172 reactive gating property rename; #173 getChannelTimeline store bypass fixed; all closed epics on main

---

## What Was Done This Session

**#173 (quick fix):** `getChannelTimeline()` in `ReactiveQhorusMcpTools` was calling `Message.<Message>find(...)` (raw Panache) instead of `blockingMessageStore.scan()` — bypassed `InMemoryMessageStore` in tests, breaking Claudony. Fixed with one injection + one call. Installed to local .m2.

**#172 (reactive gating property rename):**
- Renamed gating property from `quarkus.datasource.qhorus.reactive` → `casehub.qhorus.reactive.enabled` across all 28 runtime beans
- Added `QhorusBuildTimeConfig` (`@ConfigRoot BUILD_TIME`) in deployment module to formally declare the property — makes `@IfBuildProperty` reliable for a custom property in extension context
- Removed `@Alternative` from reactive beans that used it only as a gating workaround (not for legitimate CDI selection)
- Added `BlockingTierPurityTest` — 7 reflection checks that blocking service beans have no `Uni<T>` methods
- Protocol PP-20260520-a86db7 filed: qhorus uses `@IfBuildProperty` per-bean, not `ExcludedTypeBuildItem`

**Key finding:** `ExcludedTypeBuildItem` from a `@BuildStep` in the deployment module is silently not invoked during Quarkus 3.32.2 workspace test augmentation — the method exists in bytecode, is listed in `quarkus-build-steps.list`, but never runs. No error. Garden entries GE-20260520-48e1d4 and GE-20260520-c52767 capture this and the related `SRCFG00050` trap.

**Also:** Closed #167, #160, #157 on GitHub — they were resolved in the previous epic but never explicitly closed. Deleted stale `feat/issue-88-message-type-redesign` scaffold branch. Merged `issue-172-reactive-tier-separation` to main; pushed to origin.

**Side-effects:**
- `casehub.qhorus.reactive.enabled` must NOT appear in `application.properties` — BUILD_TIME-only property causes `SRCFG00050` at runtime. Only set via `@TestProfile.getConfigOverrides()` for reactive tests.
- `#168` ("reduce selected-alternatives burden") — may be closeable now that `@Alternative` is removed from reactive service beans. Needs verification.

## Current State

- **Both repos:** `main` — issue-172 merged and pushed to `origin` (`mdproctor/qhorus`)
- **Epic branches retained:** epic-142, epic-153, epic-154, epic-a2a-lifecycle-cleanup, issue-172 (all with `EPIC-CLOSED.md`)
- **Tests:** 1093 passing, 44 skipped (matching baseline)
- **Plan B for #172:** Category B `@Blocking @Transactional` tool conversion to pure `Uni<T>` — no issue filed yet; spec at `specs/issue-172-reactive-tier-separation/2026-05-19-reactive-tier-separation-design.md` covers the design
- **Next Flyway domain migration:** V11

## Immediate Next Step

Check whether **#168** ("reduce selected-alternatives maintenance burden in consumers") can be closed — our `@Alternative` removal from reactive service beans directly addresses it. Then pick up **#132** (delivery guarantees).

## What's Left

- **#171** — `delete_channel` must call `commitmentStore.deleteAll(channelId)` before channel deletion (no CASCADE on fk_commitment_channel) · S · Low
- **Plan B (#172)** — file an issue and implement Category B reactive tool conversion in `ReactiveQhorusMcpTools` (remove `@Blocking @Transactional`, convert to `Uni<T>`) · L · High
- **#168** — verify and close if addressed by this session's changes · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| #171 | delete_channel commitments cascade fix | S | Low | Filed last session |
| Plan B | Category B reactive tool conversion in ReactiveQhorusMcpTools | L | High | File issue first; spec exists |
| clinical#16 | PiResponseListener workaround removal | S | Low | Unblocked since #154 |
| claudony#117 | ClaudonyChannelBackend | M | Med | Unblocked since qhorus#131 + #153 |

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-20-mdp01-the-buildstep-that-wasnt.md` |
| #172 spec | `specs/issue-172-reactive-tier-separation/2026-05-19-reactive-tier-separation-design.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
