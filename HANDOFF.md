# CaseHub Qhorus — Session Handover

**Date:** 2026-07-20 — #357 implemented and shipped. Channel protocol enforcement SPI — pluggable message sequence validation.

---

## Immediate Next Step

Close epic #350 (Channel Intelligence) — all children done (#355, #357). Then pick from the cross-repo backlog.

## What Was Done

**Implemented #357 — channel protocol enforcement SPI.** Advisory-only enforcement via `ChannelProtocol` SPI in `api/spi/`. `ProtocolRegistry` (ProjectionRegistry pattern). 4 built-in protocols: REQUEST_RESPONSE, TASK_COMPLETION, ROUND_ROBIN, CONTRIBUTION_REQUIRED. Dispatch pipeline position: after CorrelationIntegrityChecker, before LAST_WRITE. Two shared queries per dispatch (findRecent + findOpenByChannelId). Channel gains `protocols` and `protocolParticipants` fields (V39 migration). 4 new MCP tools. Design-reviewed through 4 adversarial rounds (17 issues, all resolved). Pushed to upstream/main.

**Also this session:** closed epics #349 (Coordination Resilience) and #351 (Verification & Trust). Stamped issue-355 project branch.

## What's Left

- Close epic #350 — all children landed · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #358 | Supervisor + friction interventions | L | High | Cross-repo: engine |
| #359 | Summarisation → Qhorus integration | M | Med | Cross-repo: blocks |
| #361 | CBR routing + coordination memory | M | High | Cross-repo: blocks/neocortex |
| #360 | Common ground projection | M | High | Cross-repo: blocks |
| #364 | Convergence detection | M | High | Cross-repo: blocks |

### Epics

| # | Epic | Children |
|---|------|----------|
| #349 | Coordination Resilience | ~~#353~~, ~~#354~~, ~~#362~~, ~~#363~~, ~~#368~~ — **CLOSED** |
| #350 | Channel Intelligence | ~~#355~~, ~~#357~~ — **ready to close** |
| #351 | Verification & Trust | ~~#356~~ — **CLOSED** |
| #352 | Cross-Repo Suggestions | #358, #359, #360, #361, #364 |

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-20-protocol-enforcement-spi-design.md` |
| Design review workspace | `~/adr/casehub-qhorus/protocol-enforcement-spi-20260720-030413/` |
| Previous references | *Unchanged — `git show HEAD~1:HANDOFF.md`* |
