# CaseHub Qhorus — Session Handover
**Date:** 2026-05-21 — git-squash + #179/#180 + blog catch-up

---

## What Was Done This Session

git-squash on full 325-commit history found 40 candidates; merge commits in the topology blocked full execution — squashed only the clean linear section (326→321, pushed). PR #178 merged. Push workflow clarified: `git push origin main` only.

Completed #179/#180: add `classpath:db/ledger/migration` to Flyway locations; rename V1003→V2000 (V1004 also taken by ledger JAR); improved `FlywayMigrationSchemaTest` to use both locations directly. 1108 tests passing. Branch closed.

Blog catch-up: all 35 qhorus entries now published to mdproctor.github.io (4 backlogged entries from May 15–20 were missing).

## Immediate Next Step

`claudony#126` — remove stale `quarkus.arc.selected-alternatives` block (22 entries, XS · Low). Or start #132 delivery guarantees.

## What's Left

- **claudony#126** — remove stale `quarkus.arc.selected-alternatives` block · XS · Low
- **claudony#129** — replace `ChannelView`/`InstanceView`/`HumanMessageResult` with api types · S · Low
- **Plan B** — file issue, convert Category B `@Blocking` tools in `ReactiveQhorusMcpTools` to `Uni<T>` · L · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| Plan B | Category B reactive tool conversion | L | High | File issue first |
| claudony#126 | Remove stale selected-alternatives | XS | Low | |
| claudony#129 | Replace local DTO copies with api types | S | Low | After #175 on casehubio/qhorus |

## Notes

- 3 squash backup branches deletable: `backup/pre-squash-main-20260507`, `backup/pre-squash-v1-main-20260508`, `backup/pre-squash-main-20260521`
- Stale merged epic branches deletable: `epic-142`, `epic-153`, `epic-154`, `epic-a2a`, `feat/commitment-store`
- Flyway convention: V1000–V1999 = ledger base, V2000+ = consumer joins (PP-20260521-0ba358)
- Push to `origin` (mdproctor/qhorus) only — promote to casehubio/qhorus manually

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-21-mdp05-jar-disagreed-with-protocol.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
