*Updated: casehub-devtown#13 closed — removed from backlog.*

# CaseHub Qhorus — Session Handover
**Date:** 2026-06-27 — #294, #307, #216 event timestamp, attestation context, per-connector normaliser

---

## Immediate Next Step

Main is clean. Both remotes at `80e0eb8`. Three issues closed (#294, #307, #216).

GitHub Packages auth status unknown — was expired last session. If `mvn install` fails resolving `casehub-platform:0.2-SNAPSHOT`, check `~/.m2/settings.xml` GitHub token.

Cross-repo follow-up needed: `MessageReceivedEvent` constructor changed (added `Instant occurredAt`). Claudony (3 test sites) and engine (7 test sites) will fail at next compile — mechanical fix (`Instant.now()` as 7th arg).

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #287 | `casehub-qhorus-desiredstate` NodeDriftChecker bridge | — | — | — |
| #169 | Extract persistence-memory/ module from testing/ | M | Low | Standalone |

## What Was Done This Session

**CloudEvent timestamp (#294):** Added `Instant occurredAt` to `MessageReceivedEvent` with `requireNonNull` in compact constructor. Populated from `Message.createdAt` in `MessageObserverDispatcher`. Fixed reactive path (`syntheticMsg.createdAt = ctx.occurredAt()`). Adapter uses `event.occurredAt()` instead of `OffsetDateTime.now()`.

**Attestation capabilityTag (#307):** Added `String capabilityTag` as 5th field on `CommitmentContext`. Extracted from COMMAND content BEFORE `attestationFor()` — single extraction, single source of truth. Removed 2-arg `attestationFor` backward-compat overload (zero production callers). Updated protocol PP-20260623-77adf0.

**Per-connector normaliser (#216):** Renamed `normaliser()` → `normaliserFor(UUID channelId)` on `HumanParticipatingChannelBackend`. New `ConnectorNormaliser` SPI in connector-backend with CDI discovery and duplicate detection. `ConnectorChannelBackend.route()` now passes `correlation-id` from metadata.

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-06-25-event-timestamp-context-normaliser-design.md` |
| Blog entry | workspace `blog/2026-06-27-mdp01-when-now-is-the-bug.md` |
| Previous session | `git show HEAD~1:HANDOFF.md` |
