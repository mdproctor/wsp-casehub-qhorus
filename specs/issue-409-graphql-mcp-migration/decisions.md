## D1: Domain model — ~6 conceptual domains (REVISED from 15)

**Choice:** Group operations into ~6 conceptual domains by agent concern, not 15 entity-based domains:
- `channels` — channel CRUD, topics, membership, spaces, projections, gateway, summaries
- `messaging` — send, check, replies, reactions, search, wait_for_reply, approval
- `governance` — commitments, watchdogs, enforcement, protocols, capacity
- `audit` — ledger entries, causal graphs, attestations, peer review
- `agents` — instances, presence, capabilities, registration
- `data` — artefacts, shared data, chunked upload
**Alternatives:**
- 15 entity-based domains — one per issue table row. Agent working with channels needs to activate 5 separate domains. Fragments discovery.
- Single "qhorus" domain — no granularity, loads all ~99 tools at once
**Rationale:** Groups operations by how an agent thinks (communicating, governing, auditing), not by internal entity model. Each domain is substantial (10-35 ops), justifying activation overhead. `casehub_activate` loads one domain for a coherent concern.
**Trade-offs:** Larger resolver classes than 15-domain model. Some operations span concerns (e.g., channel enforcement could be governance or channels). Resolved by placing at the primary usage point.
**Sources:** `GraphQLModelScanner.java`, `DynamicToolRegistrar.java`, decision review R1-01
**Exploration:** quick → revised after review
**Status:** revised

## D2: Domain naming convention — bare names

**Choice:** Bare domain names — `"channels"`, `"messaging"`, `"governance"`, `"audit"`, `"agents"`, `"data"` — without prefix.
**Alternatives:**
- Prefixed ("qhorus-channels") — unambiguous but verbose tool names
- Dotted hierarchy ("qhorus.channels") — dots in tool names may confuse MCP clients
**Rationale:** With 6 conceptual domains, names are descriptive enough to be collision-resistant ("governance", "audit" are qhorus-specific). Tool names like `channels_create`, `governance_listWatchdogs` are clean and readable.
**Trade-offs:** Bare names like "data" are generic. Mitigated by pre-release platform where naming is controlled.
**Sources:** `DynamicToolRegistrar.registerOperationTool()` — tool name format is `domain + "_" + op.name()`
**Exploration:** quick
**Status:** captured

## D3: Migration strategy — remove @Tool methods immediately

**Choice:** Delete `@Tool` methods from `QhorusMcpTools` as each domain's GraphQL resolver lands. No deprecation period.
**Alternatives:**
- Deprecate then remove — transition window but doubles maintenance surface
- Keep both permanently — maximum compatibility but permanent duplication
**Rationale:** Pre-release platform — breaking changes cost nothing. Maintaining dual registration (old @Tool + new GraphQL) creates confusion about which path is authoritative. Clean removal reduces the 2761-line file incrementally until it can be deleted.
**Trade-offs:** Consumers (Claudony) must update tool calls as each domain migrates. Coordinate via same-branch or same-session updates.
**Sources:** CLAUDE.md maturity stage: pre-release
**Exploration:** quick
**Status:** captured

## D4: Resolver organization — split query/mutation per domain

**Choice:** Each domain gets separate query and mutation resolver classes (e.g., `ChannelsQueryResolver` + `ChannelsMutationResolver`).
**Alternatives:**
- Single resolver per domain — fewer files but unwieldy for large domains (channels has ~35 ops)
- Mixed threshold (≤6 single, >6 split) — pragmatic but inconsistent
**Rationale:** Follows the established pattern. With ~6 domains of 10-35 ops each, all domains are large enough to justify the split. Clear read/write separation.
**Trade-offs:** 12+ resolver classes. Acceptable at this domain size.
**Sources:** Existing `QhorusQueryResolver.java`, `QhorusMutationResolver.java`
**Exploration:** quick
**Status:** captured

## D5: Scope — full design, batched execution

**Choice:** This spec designs the complete migration of all 6 domains (architecture, naming, DTOs, API facade gaps). Execution spawns S/M sub-issues per domain, implemented one at a time.
**Alternatives:**
- First batch only — faster to first value but risks inconsistency across batches
- Architecture only — most thorough but slowest, each domain needing its own brainstorm
**Rationale:** Full design ensures consistent patterns across all domains. Sub-issues per domain keep execution incremental and reviewable.
**Trade-offs:** Larger spec to maintain. If patterns evolve during execution, earlier domains may need retrofitting. Mitigated by pre-release.
**Sources:** Issue #409 domain table, #401 established pattern
**Exploration:** quick
**Status:** captured

## D6: API-layer access — use existing readers/stores, create facades only where business logic exists

**Choice:** GraphQL resolvers inject the existing API-layer interfaces: Reader interfaces in `api/store/` for reads, Manager interfaces in `api/channel/`/`api/message/` for writes with business logic, stores directly for simple CRUD. New facade interfaces created ONLY where runtime business logic needs exposure.
**Alternatives:**
- Create facades for every domain — boilerplate for domains where the facade is a pass-through
- Inject runtime services — violates API-layer separation
**Rationale:** Reader interfaces already exist for most domains (`CommitmentReader`, `MessageReader`, `ReactionReader`, `TopicReader`, `MembershipReader`). Manager interfaces exist where business logic warrants them (`ChannelManager`, `MessageDispatcher`, `TopicManager`, `ReactionManager`, `MembershipManager`, `PresenceTracker`). Stores in `api/store/` are already consumer-facing contracts. No need to wrap them in pass-through facades.
**Trade-offs:** Some resolvers inject stores directly (e.g., `InstanceStore` for instance registration). Acceptable because stores are in the API module — the dependency direction is correct.
**Depends on:** D1 (domain grouping determines which facades are needed)
**Sources:** Existing `CommitmentReader`, `ChannelReader`, `TopicReader` in `api/store/`; decision review R1-06
**Exploration:** quick → revised after review
**Status:** revised

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

## D8: Annotation model — @GraphQLApi + @Query/@Mutation (MicroProfile GraphQL)

**Choice:** New resolvers use `@GraphQLApi` + `@Query`/`@Mutation` from MicroProfile GraphQL, NOT `@PlatformQuery`/`@PlatformMutation`.
**Alternatives:**
- `@PlatformQuery`/`@PlatformMutation` on interfaces — designed for non-GraphQL APIs that still want MCP domain registration
**Rationale:** `@GraphQLApi` + `@Query`/`@Mutation` provides BOTH a GraphQL HTTP endpoint AND MCP tools via `GraphQLModelScanner`. The existing resolvers use this path and it works. `@PlatformQuery`/`@PlatformMutation` is the alternative for APIs that want MCP without GraphQL — not the target here.
**Trade-offs:** None. This is the established path.
**Sources:** `GraphQLModelScanner.java` scan paths; existing `QhorusQueryResolver.java`, `QhorusMutationResolver.java`; decision review R1-09
**Exploration:** quick (review-prompted)
**Status:** captured

## D9: MCP server scope — tools move from @McpServer("qhorus") to default server

**Choice:** After migration, tools are dynamically registered on the default (application) MCP server via `casehub_action`/`casehub_activate`. The named `@McpServer("qhorus")` annotation is retired with `QhorusMcpTools`.
**Alternatives:**
- Keep named server — would require `DynamicToolRegistrar` changes to register on named servers
**Rationale:** The hierarchical MCP model (`casehub_action` + domain activation) replaces the per-library named server model. Consumers connect to the application's default MCP server and use `casehub_activate` for domain-scoped tool loading. This is the platform direction.
**Trade-offs:** Consumers currently connecting to the named "qhorus" MCP server must switch to the default server. Pre-release — acceptable.
**Sources:** `DynamicToolRegistrar.java` — `toolManager.newTool("casehub_action")` registers on default server; `QhorusMcpTools.java` line 79: `@McpServer("qhorus")`; decision review R1-10
**Exploration:** quick (review-prompted)
**Status:** captured
