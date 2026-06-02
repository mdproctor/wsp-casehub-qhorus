# CaseHub Qhorus — Session Handover
**Date:** 2026-06-02 — qhorus#230 + #231 ChannelProjection SPI shipped

---

## What Was Done This Session

**Shipped qhorus#230 + #231** — `ChannelProjection<S>` SPI + `ProjectionService` (four
overloads: full, scoped, incremental, scoped-incremental). `ProjectionResult<S>` carries
state + lastMessageId cursor — eliminates the race in incremental re-projection (#231 included).
Three code-review rounds drove significant API changes: `channelType()` removed (YAGNI,
locks String return type), `ProjectionRenderer<S>` removed from api/spi/ (not an SPI if
the service never calls it), six bugs fixed. `MessageView` DTO in api/message/ (cannot
use JPA Message entity in api/spi/). `ReactiveProjectionService` uses `collect().in()`
with mutable `FoldAcc<S>` (Mutiny BiConsumer constraint). PanacheQuery.stream() absent in
Quarkus 3.32 — documented honestly. Garden: GE-20260602-fd2a31 (git squash base SHA),
GE-20260602-c38360 (PanacheQuery.stream() missing), GE-20260602-488fa9 (Mutiny BiConsumer).
Protocol: PP-20260602-b748c9 (event-log-left-fold-projection). CLAUDE.md: 3 new testing
conventions (StubReactiveMessageStore, ProjectionService put() pattern, ProjectionResult
isEmpty() guard). parent#142/#143 filed for PLATFORM.md + deep-dive doc updates.
#232 (MCP tool project_channel) deferred — requires registry + render.

## Immediate Next Step

Run `/work` to pick up **qhorus#216** (per-connector InboundNormaliser) or batch
**qhorus#225-227** (minor #214 follow-ons). Both unblocked.

## What's Left

- **casehubio/ledger#105** — reactive LedgerAttestation persistence · S · Med
- **casehubio/ledger#106** — `Uni<Boolean> TrustGateService.meetsThreshold()` · S · Low
- **qhorus#225, #226, #227** — minor code review findings from #214 · XS · Low (batch)
- **qhorus#232** — `project_channel` MCP tool (registry + render) — after next wrap

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#216 | Per-connector InboundNormaliser | S | Med | Unblocked |
| qhorus#225-227 | Minor #214 follow-ons (log+discard, ConfigMapping test, literal) | XS | Low | Batch in one branch |
| qhorus#232 | project_channel MCP tool — registry + render | M | Med | After wrap |
| qhorus#132 | Delivery guarantees (retry + dead-letter) | L | High | Main feature item |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-02-mdp02-the-fold.md` |
| ChannelProjection design spec | `docs/specs/2026-06-02-channel-projection-spi-design.md` (project) |
| Garden: git squash base SHA | GE-20260602-fd2a31 |
| Garden: PanacheQuery.stream() missing | GE-20260602-c38360 |
| Garden: Mutiny collect().in() BiConsumer | GE-20260602-488fa9 |
| Protocol: left-fold projection | PP-20260602-b748c9 |
| Peer doc issues | parent#142 (PLATFORM.md), parent#143 (qhorus deep-dive) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
