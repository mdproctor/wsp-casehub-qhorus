# CaseHub Qhorus — Session Handover
**Date:** 2026-05-21 — issue audit + #149 #175 #176 #177

---

## What Was Done This Session

Backlog audit: closed stale/resolved issues (#144 already shipped, #152 false finding, #168 resolved by #172, #149 log null-guard fix). Then three S-scale refactors: moved `ChannelDetail`/`InstanceInfo`/`MessageResult` from `QhorusMcpToolsBase` inner records to `casehub-qhorus-api` (35 files); extracted `toTimelineEntry`/`toChannelDetail` to `QhorusEntityMapper` CDI bean; aligned all `Panache.withTransaction()` calls to named `"qhorus"` PU across 5 reactive services (16 call sites). All on PR #178 against upstream.

Also confirmed closed: `clinical#16` and `claudony#117` — both already done. Filed `claudony#126` (remove stale `selected-alternatives` block) and `claudony#129` (replace local DTO copies with canonical api types).

**Tests:** 1107 passing, 44 skipped. Both repos on `main`.

## Immediate Next Step

Merge PR #178 upstream — it's been on `mdproctor:main` since the session started and now carries 4 commits (#149, #175, #176, #177 + CLAUDE.md update).

## What's Left

- **claudony#126** — remove stale `quarkus.arc.selected-alternatives` block (22 entries, dead config since #172) · XS · Low
- **claudony#129** — replace `ChannelView`/`InstanceView`/`HumanMessageResult` in `QhorusDashboardService` with canonical api types from #175 · S · Low
- **Plan B (#172)** — file issue, convert Category B `@Blocking @Transactional` tools in `ReactiveQhorusMcpTools` to pure `Uni<T>` · L · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| PR #178 | Merge to upstream casehubio/qhorus | XS | Low | Ready now |
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| Plan B | Category B reactive tool conversion | L | High | File issue first |
| claudony#126 | Remove stale selected-alternatives | XS | Low | |
| claudony#129 | Replace local DTO copies with api types | S | Low | After #175 lands |

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-21-mdp03-inner-records-api-boundaries.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| PR #178 | https://github.com/casehubio/qhorus/pull/178 |
