# CaseHub Qhorus — Session Handover
**Date:** 2026-05-30 — #219 closed, connector bridge merged to casehubio/qhorus main

---

## What Was Done This Session

**#219 merged.** `casehub-qhorus-connector-backend` ships — bridges `InboundMessage` CDI events from casehub-connectors into Qhorus channels via `HumanParticipatingChannelBackend`. Key deliverables: `ChannelConnectorBinding` JPA entity + V14 migration, `ChannelBindingStore` store seam (blocking + InMemory), `ConnectorChannelBackend` (sync `@Observes ChannelInitialisedEvent` for cache, async `@ObservesAsync InboundMessage` for routing), `ConnectorKeyStrategy`, `OutboundTitle`.

Option B mapper refactor: `QhorusEntityMapper` is now a pure transformer (no store injection). `ChannelBindingStore` moved to `QhorusMcpToolsBase` with single-item and batch `toChannelDetail` overloads. `list_channels` pre-loads `findAll()` to eliminate N+1. `QhorusDashboardService.listChannels()` uses `runSubscriptionOn(worker-pool)` for the blocking `findAll()` call. `NativeImageResourcePatternsBuildItem` now registers both `db/qhorus/migration/*.sql` and `db/ledger/migration/*.sql` (LedgerProcessor does not self-register — GE-20260530-0dc6de).

PR #222 had conflicts post-work-end (squash-merge orphan from #193 — GE-20260422-ceb229 revised). Fixed with `git rebase --onto origin/main 8e84605`. `casehub-qhorus 0.2-SNAPSHOT` installed locally.

Filed: casehubio/qhorus#221 (CDI async wiring gap for `@ObservesAsync InboundMessage` — currently untested).

## Immediate Next Step

Pick up the S/XS queue — run `/work` and start with #215 (fire `ChannelInitialisedEvent` on connector binding update, XS). Then #217, #203, #150, #148, #146, #202, #183, #164, #166 in turn. After S/XS batch, proceed to #213 (M).

## What's Left

- **casehub-ledger#105** — reactive `LedgerAttestation` persistence · S · Med _(ledger work)_
- **casehub-ledger#106** — `Uni<Boolean> TrustGateService.meetsThreshold()` · S · Low _(ledger work)_
- **claudony#135** — `deadline` + `correlationId` first-class in `postToChannel()` SPI · S · Low _(Claudony; qhorus ready)_
- **#221** — CDI async wiring test for `@ObservesAsync InboundMessage` (coverage gap) · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #215 | Fire `ChannelInitialisedEvent` on connector binding update — enables cache refresh | XS | Low | Unblocked by #219; enables #217 |
| #217 | MCP tools for creating/updating channels with connector bindings | S | Low | Needs #215 first |
| #203 | Add drafthouse to CI dispatch chain | XS | Low | Unblocked |
| #150 | A2A batch cleanup — JSON error injection, deriveState ordering, ensureRegistered race | XS | Low | Unblocked |
| #148 | LAST_WRITE channel semantics for A2A inbound | S | Low | Unblocked |
| #146 | Artefact claim/release lifecycle for A2A inbound | S | Low | Unblocked |
| #202 | Normaliser telemetry EVENT on receiveHumanMessage | S | Low | Unblocked |
| #183 | Add `recovered` flag to `ChannelInitialisedEvent` | XS | Low | Now applicable (ConnectorChannelBackend is the first real observer that needs it) |
| #164 | Per-channel subscription scope on MessageObserver | S | Med | Unblocked |
| #166 | Dispatch MessageObserver after transaction commit via JTA | S | Med | Unblocked |
| #213 | ObligorTrustPolicy SPI — replace colon heuristic in trust gate | M | Med | After S/XS batch |
| #132 | Delivery guarantees (retry + dead-letter) | L | High | Main feature item |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-30-mdp05-humans-in-the-mesh.md` |
| Connector backend spec | `specs/2026-05-30-connector-channel-backend-completion.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
