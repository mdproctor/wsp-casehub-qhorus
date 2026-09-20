# HANDOFF — casehub-qhorus

## Last Session

Completed Batch 4 (Audit domain) of the @McpDomain migration. Created `AuditApi` + `AuditService` with 13 operations covering ledger queries, obligation chains, causal graphs, telemetry summaries, attestations, and peer review. This was the most complex batch — required a new `LedgerReader` API facade (12 query methods wrapping `MessageLedgerEntryRepository`), `CausalGraphReader` (wraps `CausalGraphService`), plus `PeerAttestor` and `ReviewerProvider` SPI interfaces. Created runtime adapter classes (`LedgerReaderAdapter`, `CausalGraphReaderAdapter`, `PeerAttestorAdapter`) and made `ReviewerResolver` implement `ReviewerProvider` directly. Promoted 10 records from `QhorusMcpToolsBase` to `api/audit/`. Updated 3 test files (10 assertions) from `ToolCallException` → `IllegalArgumentException` since `@WrapBusinessError` only fires on `@Tool` methods. 6 domains now registered: channels, messaging, governance, agents, data, audit.

Key pattern established this session: runtime adapters wire existing implementations to new API interfaces, so `AuditService` (in graphql module) can inject API-layer types only. The adapters handle entity → record conversion.

Pre-existing failures: `SharedDataToolTest` (3) and `WatchdogDisabledTest` (3) from Batches 2-3 — same `ToolCallException` → `IllegalArgumentException` issue. `compliance-report` module has `@HandWrittenEndpoint` build failure unrelated to this work.

## Immediate Next Step

Batch 5: Expand `ChannelsApi` + `ChannelsService` with ~25 ops covering topics, membership, spaces, and gateway operations. These are mostly thin delegations to existing `TopicManager`, `MembershipManager`, `SpaceService`, and `ChannelGateway` — simpler than Batch 4. May need API facades for runtime classes that don't have API-layer interfaces yet.

## References

- `plans/2026-09-20-mcpdomain-migration.md` — implementation plan (7 batches)
- `docs/specs/issue-409-graphql-mcp-migration/` — design spec + decisions
