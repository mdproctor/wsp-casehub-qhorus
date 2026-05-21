# CaseHub Qhorus — Session Handover
**Date:** 2026-05-21 — git-squash exploration + #179/#180 Flyway migration fix

---

## What Was Done This Session

git-squash analysis on full 325-commit history found 40 candidates, but merge commits in the topology blocked execution — squashed only the clean linear section (326→321 commits, pushed). PR #178 merged. Push workflow clarified: `git push origin main` only; casehubio/qhorus promoted manually.

Completed #179/#180: add `classpath:db/ledger/migration` to Flyway locations; rename V1003→V2000 (V1004 also taken by ledger JAR — confirmed by test failure); improved `FlywayMigrationSchemaTest` to use both locations directly. 1108 tests passing. Branch closed.

## Immediate Next Step

Install qhorus snapshot to other modules that depend on it (claudony, aml) — `mvn install` already run locally. Next discrete work: `#132` delivery guarantees for backends, or `claudony#126`/`claudony#129` cleanup.

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

- 3 squash backup branches exist: `backup/pre-squash-main-20260507`, `backup/pre-squash-v1-main-20260508`, `backup/pre-squash-main-20260521` — deletable
- Stale merged epic branches on project: `epic-142`, `epic-153`, `epic-154`, `epic-a2a`, `feat/commitment-store` (4 weeks) — deletable
- Flyway versioning protocol: V1000–V1999 = ledger base, V2000+ = consumer joins (PP-20260521-0ba358)

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-21-mdp05-jar-disagreed-with-protocol.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
