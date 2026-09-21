# HANDOFF — casehub-qhorus

## Last Session

Completed Batches 5 and 6 of the @McpDomain migration (issue #451). Two sessions' worth of progress in one sitting — 6/7 batches now done.

**Batch 5 (Channels-Part1):** Migrated 25 @Tool ops covering topics, membership, spaces, and gateway. Created `SpaceManager` API facade, extended `TopicManager` with `move()` and actor-id overloads, extended `MembershipManager` with `listMembers()`/`markRead()`. Promoted 5 result records from `QhorusMcpToolsBase` to API layer. Created 3 new test files (24 tests). Fixed 32 runtime test failures (26 new from removed methods + 6 pre-existing `ToolCallException` assertion mismatches from Batches 2-3).

**Batch 6 (Channels-Part2):** Added ~25 config/summary/projection/capacity/enforcement/routing operations to `ChannelsApi` (now 57 total ops). Created 4 new API facades: `ChannelSummaryManager`, `ProjectionReader`, `ProtocolReader`, `RoutingDiagnostics`. Created 4 runtime adapters. Extended `ChannelManager` with 7 new methods. Created 2 new test files (18 tests). Removed 33 @Tool methods initially, but restored test-infrastructure methods (`checkMessages`, `listChannels`, `findChannel`, `channelDigest`, `projectChannel`) because 49 test classes depend on them as test helpers — these will be removed with QhorusMcpTools in Batch 7.

**Lesson learned:** `checkMessages` and `listChannels` are used pervasively as test infrastructure, not just as tool-specific tests. 253 test failures when removed prematurely. These must be migrated to direct service calls when QhorusMcpTools is deleted.

**Key observation from this session:** Fork-based subagents don't inherit hook enforcement (e.g. `intellij-first.sh`), so they fall back to bash grep. For code-heavy refactoring, inline work with IntelliJ MCP tools is faster and more reliable than forking.

## Immediate Next Step

**Batch 7 (Cleanup):** Retire `QhorusMcpTools.java` and `QhorusMcpToolsBase.java` entirely. This is the largest remaining task — 49 test classes need migrating from `tools.checkMessages()`/`tools.listChannels()` etc. to direct service calls. Do this inline with IntelliJ tools, not forked. Also: change `ComplianceApi` from `@McpDomain("qhorus")` to `@McpDomain("compliance")`, update `DomainRegistrationTest` to final 7-domain state, remove `ToolOverloadDiscoverabilityTest`.

Pre-existing: `compliance-report` module has `@HandWrittenEndpoint` build failure (unrelated to this work).

## References

- `plans/2026-09-20-mcpdomain-migration.md` — implementation plan (7 batches)
- `docs/specs/issue-409-graphql-mcp-migration/` — design spec + decisions
