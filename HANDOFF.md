# CaseHub Qhorus — Session Handover
**Date:** 2026-06-03 — qhorus#232 project_channel MCP tool shipped

---

## What Was Done This Session

**Shipped qhorus#232** — `project_channel(channel, projection_name) -> String` MCP tool.
`RenderableProjection<S>` SPI in `api/spi/` extends `ChannelProjection<S>` with `projectionName()`
(no CDI qualifier — follows `backendId()` pattern) and `render(ProjectionResult<S>)` (not `render(S)` —
`isEmpty()` is the only reliable empty-channel signal). `ProjectionRegistry` is `@ApplicationScoped`,
iterates `@Any Instance<RenderableProjection<?>>` at construction, validates null/blank/duplicate names
at startup. `resolveChannel()` and `projectAndRender()` helpers in `QhorusMcpToolsBase`; `ProjectionRegistry`
injected in concrete tool classes. Two code review bugs fixed: null name guard, two-phase UUID parsing.

Also completed: ledger#105+#106 (reactive attestation + TrustGate parity), qhorus#225-227 (connector review fixes).
Channel identity brainstorm filed #236 (slug enforcement), #237 (MCP UUID migration), #238 (dual-identity protocol).
Garden: GE-20260603-8f582a (Java nested wildcard inference failure). Protocol: PP-20260603-fa1bf0 (CDI registry startup validation).

**Pending:** upstream (casehubio/qhorus) diverged — local main is 34 commits ahead, upstream has 30 distinct
commits. Direct push rejected. Needs reconciliation next session before upstream delivery.

## Immediate Next Step

Reconcile the `upstream/main` divergence: investigate whether the 30 upstream commits are genuinely distinct
or rebased copies, then either `git pull --rebase upstream main` carefully or force-push to upstream. File
this as a tracking issue or handle at session start.

## What's Left

- **upstream push reconciliation** — 34 local commits vs 30 upstream commits; diverged histories need merge · M · Med
- **qhorus#236** — slug enforcement on channel names · M · Low
- **qhorus#237** — MCP tool migration from channel_name to UUID-or-slug · L · Low
- **qhorus#238** — dual-identity protocol · S · Low
- **qhorus#239** — `project_channel` output size bound · S · Low
- **qhorus#240** — `list_projections` MCP tool · XS · Low
- **ledger#114** — lightweight mode (paused, stack depth 1) · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | upstream/main reconciliation | M | Med | Blocking blessed repo delivery |
| qhorus#240 | `list_projections` MCP tool | XS | Low | Trivial: `ProjectionRegistry.registeredNames()` |
| qhorus#236 | Slug enforcement on channel names | M | Low | V16 migration + ChannelService validation |
| ledger#114 | Lightweight outcome-tracking mode | L | Med | Paused — resume from stack |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-03-mdp03-registry-by-name.md` |
| project_channel spec (v3) | `specs/2026-06-03-project-channel-mcp-design.md` |
| Garden: nested wildcard inference | GE-20260603-8f582a |
| Protocol: CDI registry startup validation | PP-20260603-fa1bf0 |
| Deferred issues | qhorus#236, #237, #238, #239, #240 |
| Ledger paused branch | ledger `issue-114-lightweight-mode` (stack depth 1) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
