# CaseHub Qhorus — Session Handover
**Date:** 2026-05-31 — #213 closed + 10 batch issues resolved and merged to casehubio/qhorus main

---

## What Was Done This Session

**Batch of 11 issues cleared** on branch `issue-213-obligor-trust-policy-spi`:
- **#213 (M)**: `ObligorTrustPolicy` SPI extracted from `MessageService` — `ObligorTrustContext(obligorId, channelId, channelName)` in `api/spi/`; `DefaultObligorTrustPolicy` in `runtime/message/`
- **#215 (XS)**: `ChannelService.updateConnectorBinding()` fires `ChannelInitialisedEvent` on binding update
- **#217 (S)**: `create_channel` extended with 4 connector binding params; new `update_channel_binding` MCP tool
- **#203 (XS)**: drafthouse added to CI dispatch chain in publish.yml
- **#150, #148, #146**: closed without code — all resolved by prior #135 work
- **#202 (S)**: normaliser telemetry EVENT after every `receiveHumanMessage`
- **#183 (XS)**: `recovered` flag on `ChannelInitialisedEvent` (startup recovery = true, all else = false)
- **#164 (S)**: `MessageObserver.channels()` per-channel exact-name filter
- **#166 (S)**: `MessageObserverDispatcher` defers to JTA post-commit via TSR; STATUS_ACTIVE gate prevents Narayana "state 1" error in `@TestTransaction` tests

Squashed to 8 clean commits. Merged to `casehubio/qhorus` main.

Also: `consumer-spi-placement` protocol and `jta-tsr-status-active-gate` protocol added to casehub parent. ADR-0012 (JTA post-commit dispatch). 3 garden entries + 1 revise.

## Immediate Next Step

Pick up the next issue — run `/work` to start a new branch. The What's Next queue below reflects the remaining open issues.

## What's Left

- **casehub-ledger#105** — reactive `LedgerAttestation` persistence · S · Med
- **casehub-ledger#106** — `Uni<Boolean> TrustGateService.meetsThreshold()` · S · Low
- **#221** — CDI async wiring test for `@ObservesAsync InboundMessage` (coverage gap) · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #221 | CDI async wiring test for ConnectorChannelBackend.onInboundMessage | S | Med | Unblocked |
| #216 | Per-connector InboundNormaliser — email threading, type inference | S | Med | Unblocked |
| #214 | Auto-channel creation on first contact from external connector | S | High | Needs #215/#217 ✅ |
| #213 | ObligorTrustPolicy SPI | M | Med | ✅ Done this session |
| #132 | Delivery guarantees (retry + dead-letter) | L | High | Main feature item |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-30-mdp06-clearing-the-queue.md` |
| ObligorTrustPolicy spec | `specs/2026-05-30-obligor-trust-policy-spi.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
