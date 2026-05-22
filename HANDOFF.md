# CaseHub Qhorus — Session Handover
**Date:** 2026-05-22 — #161/#181 + branch hygiene + squash + sync

---

## What Was Done This Session

Closed `issue-145-181-161-small-items`: #145 (already done via #135), #161 (ChannelQuery.byNamePrefix + LIKE escaping), #181 (ChannelInitialisedEvent + gateway startup recovery). 1494 tests. Full work-end with ADR-0008, 3 protocols, 3 garden entries. Platform migration: ActorType to casehub-platform-api (51 files).

Branch hygiene: deleted 17 stale branches (project + workspace) after verifying content via `git log main --grep` (not `--merged`, which misses squash-rebased branches). ⚠️ User has since said they prefer a **retention policy over deletion** — this is a direction change to carry forward.

Squash: 13 commits → 8 before pushing to upstream. All three remotes (local, origin, upstream) now at same SHA (`c78ffbd`). 37 blog entries published to mdproctor.github.io.

## Immediate Next Step

Address the branch retention policy question — user doesn't want branches deleted, prefers retention. Discuss what the retention policy should look like before next hygiene scan.

Then: `claudony#126` (XS · Low) or `#132` delivery guarantees (L · High).

## What's Left

- **claudony#126** — remove stale `quarkus.arc.selected-alternatives` block · XS · Low
- **claudony#129** — replace local DTO copies with api types · S · Low
- **Plan B** — file issue, convert Category B `@Blocking` tools in `ReactiveQhorusMcpTools` to `Uni<T>` · L · High
- **Branch retention policy** — define what retention looks like instead of deletion · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| Plan B | Category B reactive tool conversion | L | High | File issue first |
| claudony#126 | Remove stale selected-alternatives | XS | Low | |
| claudony#129 | Replace local DTO copies with api types | S | Low | After #175 on casehubio/qhorus |
| #184 | (interrupted — not started) | ? | ? | |

## Notes

- Push workflow: `git push origin main` only; promote to casehubio/qhorus manually
- `git branch --merged` is unreliable after squash-rebase — use `git log main --grep` to verify content
- backup/pre-squash-main-20260522 exists locally (project) — squash backup from this session
- remotes/origin/claude/agent-argument-graphs-DWlFm — unknown provenance remote branch, left untouched

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-22-mdp02-seventeen-branches-unreliable-tool.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
