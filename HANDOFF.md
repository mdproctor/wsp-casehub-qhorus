# CaseHub Qhorus — Session Handover
**Date:** 2026-06-05 — channel slug enforcement (#236) shipped

---

## What Was Done This Session

**Shipped qhorus#236 — channel slug enforcement:**

- `ChannelSlugValidator` — public class owning all slug validation (`validateSlugPath`, `isValidSegment`, `tryParseUuid`); UUID-shaped names explicitly rejected; `SEGMENT_PATTERN = [a-z][a-z0-9]*(-[a-z0-9]+)*`
- `ChannelCreateRequest` compact constructor — slug validation as first gate, before connector binding and type checks; `Channel.name` gets `@Column(updatable = false)` for ORM immutability
- `QhorusMcpToolsBase.resolveChannel()` — flipped to name-first; private `tryParseUuid()` replaced by `ChannelSlugValidator.tryParseUuid()`; `create_channel` and reactive counterpart both have slug-rules description
- `ConfiguredAutoChannelPolicy` — `sanitiseSegment()` (8-hex SHA-256 hash, unconditional, for external keys), `slugifyConnectorId()` (hash-free, for developer IDs), `validatePattern()` startup gate via `{...}` → `a` substitution
- V17 Flyway migration — `chk_channel_name_slug` CHECK constraint via `REGEXP_LIKE` (H2 + PostgreSQL compatible)
- `FlywayMigrationSchemaTest` — verifies V17 constraint exists
- Test fixes across runtime and connector-backend for sanitised name format

Garden: GE-20260605-9636fd (self-catching exception), GE-20260605-0ffc19 (H2 SIMILAR TO in CHECK)
Protocol: PP-20260605-8013d4 (auto-channel key sanitisation)

## Immediate Next Step

Run `mvn deploy` via CI to publish updated `0.2-SNAPSHOT` to GitHub Packages (Claudony is waiting for `set_channel_type_constraints` and the new slug-conformant channel names).

Then: claudony#142 oversight channel config fix (now unblocked by `set_channel_type_constraints`).

## Cross-Module

**We're blocking:**
- `claudony` — CI needs to publish `0.2-SNAPSHOT` first; then claudony#142 can proceed

## What's Left

- **qhorus#237** — MCP tool migration from channel_name to UUID-or-slug parameter name · L · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#237 | MCP tool parameter migration: channel_name → channel | L | Low | PP-20260604-dualid compliance |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-05-mdp03-names-that-mean-something.md` |
| Design spec | `docs/specs/2026-06-04-channel-slug-enforcement-design.md` (project) |
| Garden: self-catching exception | GE-20260605-9636fd |
| Garden: H2 SIMILAR TO in CHECK | GE-20260605-0ffc19 |
| Protocol: auto-channel key sanitisation | PP-20260605-8013d4 |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
