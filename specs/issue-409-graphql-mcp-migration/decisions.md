## D1: Domain model — separate @McpDomain per namespace

**Choice:** Each logical domain (channels, messages, instances, ledger, etc.) becomes its own `@McpDomain` value, not a single `@McpDomain("qhorus")`.
**Alternatives:**
- Single "qhorus" domain — simpler but `casehub_activate` loads all ~99 tools at once with no granularity
- Nested sub-domains ("qhorus.channels") — requires scanner changes and dots in tool names confuse some MCP clients
**Rationale:** Separate domains match the issue's domain table structure, give agents granular tool loading via `casehub_activate`, and align with the platform's hierarchical MCP model where domains are the unit of discovery.
**Trade-offs:** Existing `@McpDomain("qhorus")` resolvers must be refactored to the new domain names. Slightly more cognitive overhead for consumers who must know domain names.
**Sources:** `GraphQLModelScanner.java`, `DynamicToolRegistrar.java`, issue #409 domain table
**Exploration:** quick
**Status:** captured

## D2: Domain naming convention — bare names

**Choice:** Bare domain names — `"channels"`, `"messages"`, `"ledger"`, etc. — without prefix.
**Alternatives:**
- Prefixed ("qhorus-channels") — unambiguous but verbose tool names like `qhorus-channels_create`
- Dotted hierarchy ("qhorus.channels") — middle ground but dots in tool names may confuse MCP clients
**Rationale:** Short, clean tool names like `channels_create`. Collision risk is negligible — qhorus is the only communication mesh in the platform. Domain names are already implicitly scoped by the deployment.
**Trade-offs:** If another library uses the same domain name (e.g., "channels"), tools collide. Mitigated by pre-release platform where naming is controlled.
**Sources:** `DynamicToolRegistrar.registerOperationTool()` — tool name format is `domain + "_" + op.name()`
**Exploration:** quick
**Status:** captured

## D3: Migration strategy — remove @Tool methods immediately

**Choice:** Delete `@Tool` methods from `QhorusMcpTools` as each domain's GraphQL resolver lands. No deprecation period.
**Alternatives:**
- Deprecate then remove — transition window but doubles maintenance surface
- Keep both permanently — maximum compatibility but permanent duplication
**Rationale:** Pre-release platform — breaking changes cost nothing. Maintaining dual registration (old @Tool + new GraphQL) creates confusion about which path is authoritative. Clean removal reduces the 2761-line file incrementally until it can be deleted.
**Trade-offs:** Consumers (Claudony) must update tool calls as each domain migrates. Since execution is batched, this happens in coordinated chunks.
**Sources:** CLAUDE.md maturity stage: pre-release
**Exploration:** quick
**Status:** captured

## D4: Resolver organization — split query/mutation per domain

**Choice:** Each domain gets separate query and mutation resolver classes (e.g., `ChannelsQueryResolver` + `ChannelsMutationResolver`).
**Alternatives:**
- Single resolver per domain — fewer files but unwieldy for large domains (channels has ~18 ops)
- Mixed threshold (≤6 single, >6 split) — pragmatic but inconsistent
**Rationale:** Follows the established pattern in the existing `graphql/` module (`QhorusQueryResolver` + `QhorusMutationResolver`). Clear read/write separation. Scales uniformly regardless of domain size.
**Trade-offs:** More files for small domains (presence has 3 ops but still gets 2 resolver classes). Acceptable — consistency beats minimalism.
**Sources:** Existing `QhorusQueryResolver.java`, `QhorusMutationResolver.java`
**Exploration:** quick
**Status:** captured

## D5: Scope — full design, batched execution

**Choice:** This spec designs the complete migration of all ~15 domains (architecture, naming, DTOs, API facade gaps). Execution spawns S/M sub-issues per domain, implemented one at a time.
**Alternatives:**
- First batch only — faster to first value but risks inconsistency across batches
- Architecture only — most thorough but slowest, each domain needing its own brainstorm
**Rationale:** Full design ensures consistent patterns across all domains. The architectural decisions (D1-D4) apply uniformly. Sub-issues per domain keep execution incremental and reviewable.
**Trade-offs:** Larger spec to maintain. If patterns evolve during execution, earlier domains may need retrofitting. Mitigated by pre-release: changes propagate freely.
**Sources:** Issue #409 domain table, #401 established pattern
**Exploration:** quick
**Status:** captured

## D6: API-layer facades — create incrementally for each domain

**Choice:** For each domain, create read/write API facade interfaces in `api/` (e.g., `InstanceManager`, `LedgerReader`, `ProjectionReader`). Runtime module implements them. GraphQL resolvers inject only API-layer interfaces.
**Alternatives:**
- Resolvers inject stores directly — stores are in api/ but resolvers would contain business logic that belongs in services
- Resolvers inject runtime services — violates API-layer separation, creates tight coupling between graphql/ and runtime/
**Rationale:** Follows the established pattern (ChannelManager interface in api/, ChannelService impl in runtime/). Keeps the graphql module's dependency graph clean — it depends only on api/, never on runtime/. Business logic stays in service implementations.
**Trade-offs:** More interfaces and implementations to create upfront. For domains with simple CRUD, the facade may be thin. Acceptable — consistency and separation justify the boilerplate.
**Depends on:** D1 (separate domains determine which facades are needed)
**Sources:** Existing `ChannelManager.java`, `ChannelService.java`, `ChannelReader.java` pattern
**Exploration:** quick
**Status:** captured

## D7: Compliance domain — own @McpDomain("compliance")

**Choice:** The compliance-report module's resolvers migrate from `@McpDomain("qhorus")` to `@McpDomain("compliance")`.
**Alternatives:**
- Keep under "qhorus" — contradicts D1 (separate domains)
- Nested "qhorus-compliance" — contradicts D2 (bare names)
**Rationale:** Compliance is a distinct concern (EU AI Act reports, verification, property checks, signing). Agents should activate it independently. Consistent with D1.
**Trade-offs:** None significant. The compliance module is already a separate Maven module.
**Depends on:** D1 (separate domains), D2 (bare names)
**Sources:** `ComplianceQueryResolver.java`, `ComplianceMutationResolver.java`
**Exploration:** quick
**Status:** captured
