# HANDOFF — casehub-qhorus

## Last Session

Designed and implemented sub-issue 1 of epic #409 — migrating qhorus MCP tools to a GraphQL-backed hierarchical model. Brainstormed the full migration architecture (9 decisions, light decision review, light spec review), then implemented the first sub-issue: refactoring the `graphql/` module from a single `@McpDomain("qhorus")` to 3 domain-specific packages — `channels/`, `governance/`, `messaging/`. Existing unified resolvers (`QhorusQueryResolver`, `QhorusMutationResolver`, `QhorusSubscriptionResolver`, `QhorusModelEnricher`) deleted; replaced by domain-specific classes with `@McpDomain("channels")`, `@McpDomain("governance")`, `@McpDomain("messaging")`. 24 tests passing. Branch closed, squashed (6 → 3 commits), merged to main, issue #409 closed.

## Immediate Next Step

Epic #409 has 6 remaining sub-issues. Next: sub-issue 2 (messaging domain expansion — add remaining message operations to `MessagingMutationResolver`/new `MessagingQueryResolver`). The design spec at `docs/specs/issue-409-graphql-mcp-migration/` covers all 6 domains with operation tables and API-layer gap analysis.

## Cross-Module

- `casehub-platform` `mcp/` — `GraphQLModelScanner`, `DynamicToolRegistrar` are the platform infrastructure. No platform changes needed for this work.
- `compliance-report/` module still uses `@McpDomain("qhorus")` — sub-issue 7 renames to `@McpDomain("compliance")`.

## References

- `docs/specs/issue-409-graphql-mcp-migration/2026-09-11-graphql-mcp-migration-design.md` — full migration design spec
- `docs/specs/issue-409-graphql-mcp-migration/decisions.md` — D1-D9 validated decisions
- `graphql/src/main/java/io/casehub/qhorus/graphql/channels/` — new channels domain resolvers
- `graphql/src/main/java/io/casehub/qhorus/graphql/governance/` — new governance domain
- `graphql/src/main/java/io/casehub/qhorus/graphql/messaging/` — new messaging domain
