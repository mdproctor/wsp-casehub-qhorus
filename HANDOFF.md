# HANDOFF — casehub-qhorus

## Last Session

Migrated 23 `@Tool` methods from `QhorusMcpTools` to `@McpDomain` interfaces across 3 domains (agents, data, governance). Created `InstanceManager` and `DataManager` API-layer interfaces with runtime implementations. `@Tool` annotations stripped but Java methods kept as public non-`@Tool` helpers — tests in different packages still call them directly. Pattern is locked: `*Api` interface in `api/spi/`, `*Service` in `graphql/`, CDI-free Mockito tests, `DomainRegistrationTest` guard. Compliance-report module has a pre-existing build failure (`@HandWrittenEndpoint` check) unrelated to this work. Three capacity operations (getActorCapacity, listOverloadedActors, getRedistributionHistory) deferred from governance to Batch 6 due to cross-module `ActorCapacityView` dependency.

## Immediate Next Step

Batch 4: Create `AuditApi` + `AuditService` — needs a new `LedgerReader` facade in `api/store/` to expose `MessageLedgerEntryRepository` queries. Aggregation logic (obligation chain, telemetry summary) moves from QhorusMcpTools into the service.

## References

- `plans/2026-09-20-mcpdomain-migration.md` — implementation plan (7 batches)
- `docs/specs/issue-409-graphql-mcp-migration/` — design spec + decisions
- `blog/2026-09-20-mdp01-from-113-tools-to-six-domains.md` — session diary
