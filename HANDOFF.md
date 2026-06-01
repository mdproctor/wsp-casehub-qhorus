# CaseHub Qhorus — Session Handover
**Date:** 2026-06-01 — qhorus#214 auto-channel creation shipped

---

## What Was Done This Session

**Shipped qhorus#214** — auto-channel creation on first contact for `ConnectorChannelBackend`. Full design (brainstorming → spec → two review rounds → implementation plan) then 7-task subagent-driven implementation, code review, and work-end closure. Key design decisions: `AutoChannelPolicy` SPI in `connector-backend` (not `api/spi/`); convention table for SMS/WhatsApp outbound; `ChannelService.findOrCreateWithBinding()` with `REQUIRES_NEW`; winner-only `initChannel()` for race handling. Critical production bug caught in code review: `PSQLException` does not extend `SQLIntegrityConstraintViolationException` so the `isConcurrentInsert()` instanceof check was dead in production. Fixed before merge. Garden: GE-20260601-17fa50. Protocol: PP-20260601-c43112 (bridge-module SPI placement). CLAUDE.md updated: `receiveHumanMessage()` dispatch×2 convention, MeterRegistry delta-assert pattern, V16 next migration.

**Minor findings filed as follow-on issues:** #225 (race-loser `IllegalStateException` should log+discard), #226 (`@ConfigMapping` integration smoke test), #227 (string literal in test).

## Immediate Next Step

**casehubio/parent#125** — 17 protocol ref path-depth fixes + deep-dive enrichment + 2 new protocols + skill update. Python script ready in issue body. S · Low.

## What's Left

- **casehubio/parent#125** — 17 path-depth fixes + deep-dive enrichment + 2 protocols + skill update · S · Low
- **casehubio/parent#124** — tracking; tick off as sub-issues resolve
- **casehubio/ledger#105** — reactive LedgerAttestation persistence · S · Med
- **casehubio/ledger#106** — `Uni<Boolean> TrustGateService.meetsThreshold()` · S · Low
- **qhorus#225, #226, #227** — minor code review findings from #214 · XS · Low (batch)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| parent#125 | Protocol ref path fixes + deep-dive enrichment | S | Low | Python script ready in issue body |
| qhorus#216 | Per-connector InboundNormaliser | S | Med | Unblocked |
| qhorus#225-227 | Minor #214 follow-ons (log+discard, ConfigMapping test, literal) | XS | Low | Batch in one branch |
| qhorus#132 | Delivery guarantees (retry + dead-letter) | L | High | Main feature item |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-01-mdp01-first-contact.md` |
| Auto-channel design spec | `docs/specs/2026-05-31-auto-channel-creation-design.md` (project) |
| Garden: PSQLException instanceof miss | GE-20260601-17fa50 |
| Protocol: bridge-module SPI placement | PP-20260601-c43112 |
| Platform coherence tracking | casehubio/parent#124 |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
