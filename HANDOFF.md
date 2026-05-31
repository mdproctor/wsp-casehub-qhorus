# CaseHub Qhorus — Session Handover
**Date:** 2026-05-31 — batch of 11 issues cleared + platform coherence investigation

---

## What Was Done This Session

**Batch cleared and merged.** Branch `issue-213-obligor-trust-policy-spi` closed: #213 (ObligorTrustPolicy SPI), #215 (ChannelInitialisedEvent on binding update), #217 (connector binding MCP tools), #203 (drafthouse in CI), #202 (normaliser telemetry EVENT), #183 (recovered flag), #164 (per-channel MessageObserver filter), #166 (JTA post-commit dispatch). Issues #150/#148/#146 closed — already resolved by prior work. 8 squashed commits on casehubio/qhorus main.

**Platform coherence investigation.** casehub-drafthouse reported wrong MessageType enum. Root cause: `docs/DESIGN.md` in qhorus still shows the pre-ADR-0005 vocabulary (`REQUEST | RESPONSE | STATUS | HANDOFF | DONE | EVENT`). Also found: 17 protocol ref path-depth errors across parent, 2 cross-repo source refs, missing protocols. Filed casehubio/parent#124 (tracking, full context), casehubio/qhorus#224 (DESIGN.md fix + drafthouse integration pattern), casehubio/parent#125 (protocol fixes + deep-dive enrichment + 2 new protocols + skill update).

## Immediate Next Step

**For qhorus:** Fix `docs/DESIGN.md` — casehubio/qhorus#224. Two edits: line 100 (MessageType table) and line 251 (REQUEST → QUERY/COMMAND). XS.

**For drafthouse:** Use the integration pattern in qhorus#224. Single APPEND channel, QUERY/RESPONSE, `MessageObserver` with `channels()` filter. No custom backend SPI needed.

## What's Left

- **casehubio/qhorus#224** — DESIGN.md stale MessageType fix · XS · Low
- **casehubio/parent#125** — 17 path-depth fixes + deep-dive enrichment + 2 protocols + skill update · S · Low
- **casehubio/parent#124** — tracking; checklist to tick off as sub-issues resolve
- **casehub-ledger** — source code refs → GitHub URLs (low priority, can be deferred)
- **casehub-ledger#105** — reactive LedgerAttestation persistence · S · Med
- **casehub-ledger#106** — `Uni<Boolean> TrustGateService.meetsThreshold()` · S · Low
- **casehub-qhorus#221** — CDI async wiring test for ConnectorChannelBackend · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#224 | Fix DESIGN.md stale MessageType enum | XS | Low | Unblocks cross-repo consumers |
| parent#125 | Protocol ref path fixes + deep-dive enrichment | S | Low | Python script ready in issue body |
| qhorus#221 | CDI async wiring test for ConnectorChannelBackend | S | Med | Unblocked |
| qhorus#216 | Per-connector InboundNormaliser | S | Med | Unblocked |
| qhorus#214 | Auto-channel creation on first contact | S | High | Needs #215/#217 ✅ |
| qhorus#132 | Delivery guarantees (retry + dead-letter) | L | High | Main feature item |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-30-mdp06-clearing-the-queue.md` |
| Platform coherence tracking | casehubio/parent#124 |
| DESIGN.md stale fix + drafthouse context | casehubio/qhorus#224 |
| Protocol + deep-dive fixes (actionable) | casehubio/parent#125 |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
