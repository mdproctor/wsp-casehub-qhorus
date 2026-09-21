# HANDOFF — casehub-qhorus

## Last Session

Completed Batch 7a of the @McpDomain migration (issue #451). QhorusMcpTools is now fully stripped of MCP exposure — all tool discovery is exclusively via @McpDomain APIs.

**Batch 7a (MCP de-registration):** Stripped all `@McpServer`, `@Tool` (52), `@ToolArg` (195), and `@WrapBusinessError` annotations from `QhorusMcpTools.java`. The class remains as an inert `@ApplicationScoped` CDI bean used by 87 test files as test infrastructure. Fixed `ToolCallException` → `IllegalArgumentException`/`IllegalStateException` across 23+ test files (SSR + manual fixups). Added `ComplianceApi` to `DomainRegistrationTest` (7 domains verified). Deleted `ToolOverloadDiscoverabilityTest`. Removed obsolete HTTP-level MCP tool tests from `ToolErrorHandlingTest`. Added missing `deleteSummary()` to `ChannelSummaryManager`/`ChannelSummaryService`. Fixed pre-existing `ChannelSummaryServiceTest.setSummary_advancesCursor` mock issue.

**Build status:** 2038 runtime tests pass (0 failures). All modules succeed except pre-existing `compliance-report` `@HandWrittenEndpoint` build failure (unrelated).

**What remains for Batch 7b (follow-up issue):**
- Migrate 87 test files from `QhorusMcpTools` to direct service/store calls
- Delete `QhorusMcpTools.java` and `QhorusMcpToolsBase.java` entirely
- This is a large but low-risk refactoring task — all MCP tool exposure is already via @McpDomain

## References

- `plans/2026-09-20-mcpdomain-migration.md` — implementation plan (7 batches)
- `docs/specs/issue-409-graphql-mcp-migration/` — design spec + decisions
