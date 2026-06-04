# CaseHub Qhorus — Session Handover
**Date:** 2026-06-04 — qhorus#243 denied_types enforcement shipped + branch hygiene

---

## What Was Done This Session

**Shipped qhorus#243** — full `denied_types` enforcement: `MessageType.parseTypes()` on the api enum, `ChannelCreateRequest` compact constructor as D1 enforcement gate (type validation + overlap check), `StoredMessageTypePolicy` denial-first, `ChannelService`/`ReactiveChannelService` 10-arg overloads, `AutoChannelSpec.deniedTypes`, `create_channel` MCP `denied_types` ToolArg. Pre-existing bug fixed in `ReactiveQhorusMcpTools` (destructured `ChannelCreateRequest` back to named params, losing `deniedTypes`). Two protocols filed: PP-20260604-c19f7c (D1 gate), PP-20260604-55a0aa (Flyway naming dependency). Garden: GE-20260604-942686 (find-or-create counter gotcha), GE-20260604-b3afd6 (compact constructor technique).

**Branch hygiene:** reconciled upstream divergence (force-pushed to casehubio), closed 13 branches, promoted 5 specs, published 1 blog fix. Both mdproctor/qhorus and casehubio/qhorus now at same HEAD (`b28a67a`).

**Root cause found (not yet fixed):** qhorus#248 — `ConnectorChannelBackend.tryAutoCreate()` increments the `inbound_channels_auto_created_total` counter unconditionally after `findOrCreateWithBinding()` returns. The method has two silent success paths: create-new and find-existing. Counter fires for both. Fix: `FindOrCreateResult(Channel channel, boolean wasCreated)` return type; only count when `wasCreated == true`.

## Immediate Next Step

Publish `0.2-SNAPSHOT` to GitHub Packages so Claudony can consume it:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn deploy
```
Then start `issue-248-findorcreate-counter-fix` to fix the `tryAutoCreate()` counter bug.

## Cross-Module

**We're blocking:**
- `claudony` — needs `0.2-SNAPSHOT` published to GitHub Packages to complete claudony#142 (oversight channel fix). Denied_types is shipped in code; snapshot not yet published.

## What's Left

- **qhorus#248** — `findOrCreateWithBinding()` counter fires on find-existing path; `FindOrCreateResult` pattern needed · S · Low
- **qhorus#244** — `update_channel` MCP doesn't expose `deniedTypes` yet · S · Low
- **parent#163** — `/oversight` deep-dive doc stale (still shows `allowedTypes=COMMAND,RESPONSE`; should be `deniedTypes=EVENT`) · XS · Low
- **qhorus#236** — slug enforcement on channel names · M · Low
- **qhorus#237** — MCP tool migration from channel_name to UUID-or-slug · L · Low
- **qhorus#238** — dual-identity protocol · S · Low
- **qhorus#239** — `project_channel` output size bound · S · Low
- **qhorus#240** — `list_projections` MCP tool · XS · Low
- **ledger#114** — lightweight mode (paused, stack depth 1) · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | `mvn deploy` — publish 0.2-SNAPSHOT | XS | Low | Blocking claudony#142 |
| qhorus#248 | findOrCreateWithBinding counter fix | S | Low | Root cause known; FindOrCreateResult pattern |
| qhorus#244 | update_channel denied_types | S | Low | Same pattern as create_channel |
| qhorus#240 | list_projections MCP tool | XS | Low | Trivial: ProjectionRegistry.registeredNames() |
| qhorus#236 | Slug enforcement on channel names | M | Low | V17 migration + ChannelService validation |
| ledger#114 | Lightweight outcome-tracking mode | L | Med | Paused — resume from stack |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-04-mdp01-the-gate-and-two-holes.md` |
| denied_types spec (rev 3) | `docs/specs/2026-06-03-denied-types-enforcement-design.md` |
| Garden: find-or-create counter | GE-20260604-942686 |
| Garden: compact constructor technique | GE-20260604-b3afd6 |
| Protocol: D1 enforcement gate | PP-20260604-c19f7c |
| Protocol: Flyway naming dependency | PP-20260604-55a0aa |
| Deferred issues | qhorus#244, #248, #236, #237, #238, #239, #240 |
| Ledger paused branch | ledger `issue-114-lightweight-mode` (stack depth 1) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
