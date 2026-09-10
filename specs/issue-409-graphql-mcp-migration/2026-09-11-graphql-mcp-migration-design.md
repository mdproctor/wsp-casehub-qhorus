# Migrate Qhorus MCP Tools to GraphQL-Backed Hierarchical Model

**Issue:** casehubio/qhorus#409
**Date:** 2026-09-11
**Status:** Draft

## Problem

Qhorus has 99 flat `@Tool` methods in `QhorusMcpTools.java` (2761 lines) on the named `@McpServer("qhorus")`. The platform has moved to a GraphQL-backed hierarchical MCP model where operations are organized by domain namespace and loaded on demand via `casehub_action` / `casehub_activate`.

The current tools bypass `@PlatformQuery`/`@PlatformMutation` conventions, bloat the MCP definition registry, and cannot be selectively activated by agents.

## Solution

Migrate all qhorus MCP operations to `@GraphQLApi`-annotated resolvers organized into 6 conceptual domains. Each domain is a `@McpDomain` value discovered by `GraphQLModelScanner` and made available as both:
- A GraphQL HTTP API (via SmallRye GraphQL at `/graphql`)
- MCP tools (via `DynamicToolRegistrar` → `casehub_action` + `casehub_activate`)

Resolvers use MicroProfile GraphQL annotations (`@Query`, `@Mutation`, `@Description`) with `@McpDomain` for domain scoping. `@PlatformQuery`/`@PlatformMutation` are NOT used — those are for non-GraphQL APIs that still want MCP domain registration.

**Note:** Issue #409's body references `@PlatformQuery`/`@PlatformMutation` as the target annotations. The correct annotations are `@Query`/`@Mutation` from MicroProfile GraphQL, as established by the existing resolver pattern and confirmed by the fact that `@PlatformQuery`/`@PlatformMutation` have zero usages in qhorus. The issue body should be updated to match this spec.

## Domain Architecture

Six conceptual domains grouped by agent concern:

### `channels` — Communication infrastructure (~58 ops, largest domain)

Channel lifecycle, configuration, topics, membership, spaces, projections, gateway backends, summaries. This is the largest domain because channels are the central abstraction in qhorus — all sub-features (topics, membership, spaces, projections, gateway) are channel-scoped.

**Size concern (from spec review R1-05):** 58 operations in one `casehub_activate` call may overwhelm agent context windows. If this proves true during the channels sub-issue implementation, split into: `channels` (core CRUD + config, ~33 ops), `topics` (6 ops), `membership` (7 ops), `spaces` (9 ops). The split is deferred because (a) most agents use `casehub_action` dispatch, not activated tools, and (b) the conceptual grouping should be validated empirically before fragmenting.

| Operation | Type | Current @Tool method | API facade |
|-----------|------|---------------------|------------|
| channels | Q | listChannels | ChannelReader.listAll/scan |
| channel | Q | findChannel | ChannelReader.findByName/findById |
| channelMessages | Q | checkMessages | ConsumerMessaging.history |
| channelDigest | Q | channelDigest | (new: ChannelReader.digest) |
| listTopics | Q | listTopics | TopicReader |
| listMembers | Q | listMembers | MembershipReader |
| getUnreadCounts | Q | getUnreadCounts | MembershipReader.unreadCounts |
| getMessageDeliveryStatus | Q | getMessageDeliveryStatus | MembershipReader |
| listSpaces | Q | listSpaces | SpaceStore.list |
| getSpace | Q | getSpace | SpaceStore.findById |
| listSpaceChannels | Q | listSpaceChannels | ChannelReader.scan(bySpaceId) |
| listBackends | Q | listBackends | BackendRegistry |
| channelSummary | Q | get_channel_summary | ChannelSummaryService |
| listProtocols | Q | listProtocols | ProtocolRegistry |
| channelProtocols | Q | getChannelProtocols | ChannelReader |
| channelEnforcement | Q | getChannelEnforcement | ChannelReader |
| routingConfig | Q | getRoutingConfig | ChannelReader |
| routingCandidates | Q | getRoutingCandidates | RoutingBridge.diagnose |
| channelRedistributionThreshold | Q | getChannelRedistributionThreshold | ChannelReader |
| channelRoutingCapacityThreshold | Q | getChannelRoutingCapacityThreshold | ChannelReader |
| listProjections | Q | listProjections | ProjectionRegistry |
| projectChannel | Q | projectChannel | ProjectionService |
| createChannel | M | createChannel | ChannelManager.create |
| deleteChannel | M | deleteChannel | ChannelManager.delete |
| pauseChannel | M | pauseChannel | ChannelManager.pause |
| resumeChannel | M | resumeChannel | ChannelManager.resume |
| clearChannel | M | clearChannel | ChannelManager.clear |
| setChannelWriters | M | setChannelWriters | ChannelManager |
| setChannelAdmins | M | setChannelAdmins | ChannelManager |
| setChannelReviewers | M | setChannelReviewers | ChannelManager |
| setChannelRateLimits | M | setChannelRateLimits | ChannelManager |
| setChannelTypeConstraints | M | setChannelTypeConstraints | ChannelManager |
| setDeliveryTracking | M | setDeliveryTracking | ChannelManager |
| setChannelProtocols | M | setChannelProtocols | ChannelManager |
| setProtocolParticipants | M | setProtocolParticipants | ChannelManager |
| setEnforcementMode | M | setEnforcementMode | ChannelManager |
| setEnforcementExclusions | M | setEnforcementExclusions | ChannelManager |
| setRoutingConfig | M | setRoutingConfig | ChannelManager |
| setChannelRedistributionThreshold | M | setChannelRedistributionThreshold | ChannelManager |
| setChannelRoutingCapacityThreshold | M | setChannelRoutingCapacityThreshold | ChannelManager |
| updateChannelBinding | M | updateChannelBinding | ChannelManager |
| forceReleaseChannel | M | forceReleaseChannel | ChannelManager |
| createSpace | M | createSpace | SpaceStore/SpaceService |
| deleteSpace | M | deleteSpace | SpaceStore/SpaceService |
| renameSpace | M | renameSpace | SpaceStore/SpaceService |
| updateSpaceDescription | M | updateSpaceDescription | SpaceStore/SpaceService |
| moveSpace | M | moveSpace | SpaceStore/SpaceService |
| moveChannelToSpace | M | moveChannelToSpace | ChannelManager |
| registerBackend | M | registerBackend | BackendRegistry |
| deregisterBackend | M | deregisterBackend | BackendRegistry |
| resolveTopic | M | resolveTopic | TopicManager |
| unresolveTopic | M | unresolveTopic | TopicManager |
| renameTopic | M | renameTopic | TopicManager |
| mergeTopics | M | mergeTopics | TopicManager |
| moveTopic | M | moveTopic | TopicManager |
| joinChannel | M | joinChannel | MembershipManager |
| leaveChannel | M | leaveChannel | MembershipManager |
| markChannelRead | M | markChannelRead | MembershipManager |
| updateChannelSummary | M | update_channel_summary | ChannelSummaryService |
| configureChannelSummary | M | configure_channel_summary | ChannelSummaryService |
| triggerChannelSummaryUpdate | M | trigger_channel_summary_update | ChannelSummaryService |

### `messaging` — Message dispatch and retrieval

Sending messages, checking history, replies, reactions, search, wait/approval patterns.

| Operation | Type | Current @Tool method | API facade |
|-----------|------|---------------------|------------|
| message | Q | getMessage | MessageReader |
| replies | Q | getReplies | MessageReader |
| searchMessages | Q | searchMessages | ConsumerMessaging.search |
| reactions | Q | getReactions | ReactionReader |
| reactionsBatch | Q | getReactionsBatch | ReactionReader |
| dispatchMessage | M | sendMessage | MessageDispatcher.dispatch |
| deleteMessage | M | deleteMessage | MessageDispatcher |
| react | M | react | ReactionManager |
| unreact | M | unreact | ReactionManager |
| waitForReply | M | waitForReply | ConsumerMessaging.waitForReply |
| requestApproval | M | requestApproval | ConsumerMessaging |
| respondToApproval | M | respondToApproval | MessageDispatcher |
| cancelWait | M | cancelWait | ConsumerMessaging |

`waitForReply` and `requestApproval` are blocking long-poll operations with SSE keepalives. They stay as mutations for now (they have side effects: creating a polling registration).

**Risk (from spec review R1-07):** SmallRye GraphQL dispatches mutations on Vert.x worker threads. A `waitForReply(timeoutS=300)` blocks a worker for 5 minutes, risking thread pool exhaustion under load. Mitigations: (a) cap GraphQL-layer timeout lower than MCP, (b) return `Uni<WaitResult>` for non-blocking execution. A subscription-based `waitForReply` should be filed as a follow-up issue.

### `governance` — Commitments, watchdogs, enforcement

Obligation lifecycle, condition-based alerts, protocol enforcement, capacity management.

| Operation | Type | Current @Tool method | API facade |
|-----------|------|---------------------|------------|
| commitments | Q | listPendingCommitments | CommitmentReader |
| myCommitments | Q | listMyCommitments | CommitmentReader |
| commitment | Q | getCommitment | CommitmentReader |
| watchdogs | Q | listWatchdogs | WatchdogStore |
| actorCapacity | Q | getActorCapacity | ActorCapacityView |
| overloadedActors | Q | listOverloadedActors | ActorCapacityView |
| redistributionHistory | Q | getRedistributionHistory | (new: CapacityReader) |
| registerWatchdog | M | registerWatchdog | (new: WatchdogManager) |
| deleteWatchdog | M | deleteWatchdog | (new: WatchdogManager) |

### `audit` — Ledger, causal analysis, attestations

Normative ledger queries, causal chain/graph traversal, attestation management, peer review.

| Operation | Type | Current @Tool method | API facade |
|-----------|------|---------------------|------------|
| ledgerEntries | Q | listLedgerEntries | (new: LedgerReader) |
| obligationChain | Q | getObligationChain | (new: LedgerReader) |
| causalChain | Q | getCausalChain | (new: LedgerReader) |
| causalGraph | Q | getCausalGraph | CausalGraphService |
| renderCausalGraph | Q | renderCausalGraph | CausalGraphService |
| stalledObligations | Q | listStalledObligations | (new: LedgerReader) |
| obligationStats | Q | getObligationStats | (new: LedgerReader) |
| telemetrySummary | Q | getTelemetrySummary | (new: LedgerReader) |
| channelTimeline | Q | getChannelTimeline | (new: LedgerReader) |
| obligationActivity | Q | getObligationActivity | (new: LedgerReader) |
| attestations | Q | listAttestations | (new: LedgerReader) |
| attest | M | attest | PeerAttestationWriter |
| requestPeerReview | M | requestPeerReview | ReviewerResolver |

### `agents` — Instance registry, presence

Agent registration, capability discovery, presence tracking.

| Operation | Type | Current @Tool method | API facade |
|-----------|------|---------------------|------------|
| instances | Q | listInstances | InstanceStore |
| instance | Q | getInstance | InstanceStore |
| presence | Q | getPresenceTool | PresenceTracker |
| channelPresence | Q | getChannelPresence | PresenceTracker |
| register | M | register | InstanceStore (or new: InstanceManager) |
| registerInstance | M | registerInstance | InstanceStore (or new: InstanceManager) |
| deregister | M | deregisterInstance | InstanceStore |
| setPresence | M | setPresence | PresenceTracker |

### `data` — Artefact storage

Shared data store with claim/release lifecycle, chunked upload.

| Operation | Type | Current @Tool method | API facade |
|-----------|------|---------------------|------------|
| artefact | Q | getArtefact | DataStore |
| artefactRefs | Q | getArtefactRefs | MessageReader |
| artefacts | Q | listArtefacts | DataStore |
| isGcEligible | Q | isGcEligible | DataStore |
| shareArtefact | M | shareArtefact | DataStore |
| beginArtefact | M | beginArtefact | DataStore |
| appendChunk | M | appendChunk | DataStore |
| finalizeArtefact | M | finalizeArtefact | DataStore |
| claimArtefact | M | claimArtefact | DataStore |
| releaseArtefact | M | releaseArtefact | DataStore |
| revokeArtefact | M | revokeArtefact | DataStore |

### `compliance` — (separate module, own @McpDomain)

EU AI Act compliance evidence export. Already has resolvers in `compliance-report/` module — migrates from `@McpDomain("qhorus")` to `@McpDomain("compliance")`.

## Resolver Class Structure

Each domain produces 2-3 classes in the `graphql/` module:

```
graphql/src/main/java/io/casehub/qhorus/graphql/
├── channels/
│   ├── ChannelsQueryResolver.java      @GraphQLApi @McpDomain("channels")
│   ├── ChannelsMutationResolver.java   @GraphQLApi @McpDomain("channels")
│   └── ChannelsModelEnricher.java      @McpDomain("channels") implements ModelEnricher
├── messaging/
│   ├── MessagingQueryResolver.java     @GraphQLApi @McpDomain("messaging")
│   └── MessagingMutationResolver.java  @GraphQLApi @McpDomain("messaging")
├── governance/
│   ├── GovernanceQueryResolver.java    @GraphQLApi @McpDomain("governance")
│   └── GovernanceMutationResolver.java @GraphQLApi @McpDomain("governance")
├── audit/
│   ├── AuditQueryResolver.java         @GraphQLApi @McpDomain("audit")
│   └── AuditMutationResolver.java      @GraphQLApi @McpDomain("audit")
├── agents/
│   ├── AgentsQueryResolver.java        @GraphQLApi @McpDomain("agents")
│   └── AgentsMutationResolver.java     @GraphQLApi @McpDomain("agents")
├── data/
│   ├── DataQueryResolver.java          @GraphQLApi @McpDomain("data")
│   └── DataMutationResolver.java       @GraphQLApi @McpDomain("data")
├── dto/                                Shared DTOs
│   ├── ChannelType.java
│   ├── MessageType.java
│   ├── CommitmentType.java
│   ├── CreateChannelInput.java
│   ├── DispatchMessageInput.java
│   └── ...
└── QhorusEventPublisher.java           Subscription support
```

Each resolver class follows this pattern:

```java
@GraphQLApi
@McpDomain("channels")
@ApplicationScoped
public class ChannelsQueryResolver {

    @Inject ChannelReader channelReader;
    @Inject TopicReader topicReader;
    @Inject MembershipReader membershipReader;
    // ... other API-layer interfaces

    @Query
    @Description("List channels with optional filtering and pagination")
    public ChannelPage channels(ChannelFilterInput filter, PageInput page) {
        // ... implementation using API-layer interfaces
    }
}
```

## DTO Strategy

- **Output types** (`*Type` records): `static from(domainRecord)` factory methods convert API-layer domain records to GraphQL types. Live in `dto/` shared across domains.
- **Input types** (`*Input` records): GraphQL input objects for mutations with complex parameters. Simple mutations use scalar parameters directly.
- **Filter types** (`*FilterInput` records): For queries with optional filtering (channels, commitments, ledger entries).
- **Page types** (`*Page` records): Wrap `List<*Type>` + `PageInfo` for paginated queries. Use platform's `PageInput`/`PageInfo`.

## API-Layer Gap Analysis

**Important:** The gap is larger than just "create 4 new facades." Many operation-table mappings reference methods that don't yet exist on the listed API interfaces, or reference runtime classes that are not in `api/`. Each domain's sub-issue must verify the exact interface surface before coding resolvers.

### Tier 1 — Partially ready (some facade methods exist, others need adding)

| Domain | Existing interfaces | Missing methods (need adding to existing interfaces) |
|--------|-------------------|------------------------------------------------------|
| channels | `ChannelReader`, `ChannelManager` (partial), `TopicManager`, `TopicReader`, `MembershipManager` (partial), `MembershipReader` (partial) | `ChannelManager` needs: `clear`, `setDeliveryTracking`, `setEnforcementMode/Exclusions`, `setRoutingConfig`, `set*Threshold`, `updateChannelBinding`, `forceReleaseChannel`, `moveChannelToSpace`. `MembershipReader` needs: `unreadCounts`, `deliveryStatus`. |
| messaging | `MessageDispatcher`, `MessageReader` (partial), `ReactionManager`, `ReactionReader` | `ConsumerMessaging` needs: `search`, `waitForReply`, `requestApproval`, `cancelWait`. `MessageReader` needs: `findReplies`. |
| governance | `CommitmentReader` | — |
| agents | `PresenceTracker` | — |
| data | `DataStore` | — |

### Tier 2 — Need new facade interfaces (business logic in runtime services)

| Domain | Runtime class | Action |
|--------|--------------|--------|
| agents | `InstanceService` | Create `InstanceManager` in `api/instance/` |
| governance | `WatchdogEvaluationService` | Create `WatchdogManager` in `api/watchdog/` |
| channels | `ChannelSummaryService` | Create `ChannelSummaryManager` in `api/channel/` or extend `ChannelManager` |
| channels | `SpaceService` | Create `SpaceManager` in `api/channel/` |
| channels | `RoutingBridge.diagnose()` | Create `RoutingDiagnostics` reader in `api/` |
| channels | `ProjectionRegistry`, `ProjectionService` | Create `ProjectionReader` in `api/` |

### Tier 3 — Need new reader interfaces (queries currently internal)

| Domain | Runtime class | Action |
|--------|--------------|--------|
| audit | `MessageLedgerEntryRepository`, `CausalGraphService` | Create `LedgerReader` in `api/store/` — obligation chain, causal chain/graph, stalled obligations, stats, telemetry, timeline, obligation activity, attestations |
| governance | `ActorCapacityView` (runtime) | Promote to `api/` or create `CapacityReader` |
| channels | Digest aggregation logic in `QhorusMcpTools.channelDigest()` | Move aggregation into an API facade method |

### Business logic migration

~25 record types in `QhorusMcpToolsBase` (`ChannelDigest`, `ObligationChainSummary`, `TelemetrySummary`, etc.) contain aggregation business logic in their construction sites. This logic moves into the new API facades — e.g., `LedgerReader.getObligationChain()` returns a domain record, and the GraphQL DTO's `from()` factory maps it. The resolvers stay thin.

## Existing Resolver Migration

The current `graphql/` module has:
- `QhorusQueryResolver` (channels + channelMessages + commitments) → split into `ChannelsQueryResolver` + `GovernanceQueryResolver`
- `QhorusMutationResolver` (createChannel + deleteChannel + pauseChannel + resumeChannel + dispatchMessage) → split into `ChannelsMutationResolver` + `MessagingMutationResolver`
- `QhorusSubscriptionResolver` (channelActivity + channelPresence) → stays in `channels/` package
- `QhorusModelEnricher` → becomes `ChannelsModelEnricher` or shared enricher

Existing DTOs (`ChannelType`, `MessageType`, `CommitmentType`, etc.) move to domain-specific `dto/` packages or stay shared.

The existing `QhorusSubscriptionResolver` (channelActivity, channelPresence) stays in the `channels/` package as `ChannelsSubscriptionResolver` with `@McpDomain("channels")`.

`QhorusEntityMapper` (shared `@ApplicationScoped` mapper used by `QhorusMcpToolsBase` and `QhorusDashboardService`) is replaced by GraphQL DTO `from()` factories. If `QhorusDashboardService` still needs it after migration, retain it as a standalone mapper bean — but it no longer needs to serve the MCP layer.

## MCP Server Transition

**Before:** `@McpServer("qhorus")` on `QhorusMcpTools` — tools on named MCP server.
**After:** `DynamicToolRegistrar` registers tools on the default (application) MCP server via `casehub_action` + `casehub_activate`.

Consumers currently connecting to the named "qhorus" MCP server switch to the default server and use:
- `casehub_model` — discover available domains
- `casehub_activate("channels")` — expand a domain into individual auto-completing tools
- `casehub_action(domain="channels", operation="create", params={...})` — unified dispatch

## QhorusMcpTools Retirement

As each domain migrates:
1. Delete the corresponding `@Tool` methods from `QhorusMcpTools.java`
2. Delete the corresponding utility methods from `QhorusMcpToolsBase.java`
3. When all domains are migrated, delete both classes and the `@McpServer("qhorus")` annotation

Utilities in `QhorusMcpToolsBase` that remain useful (e.g., `resolveChannel` dual-identity resolution) become standalone helper classes or are absorbed into the resolvers' `from()` factories.

## REST Strategy

SmallRye GraphQL automatically exposes a `/graphql` HTTP endpoint. All operations available via GraphQL are also available via REST through this endpoint. Existing JAX-RS resources (`ChannelResource.java`, `CausalGraphResource.java`) can be deprecated once their operations are covered by GraphQL resolvers.

New domain-specific JAX-RS resources are NOT required — the GraphQL HTTP endpoint IS the REST API.

## Testing Strategy

- **Resolver unit tests:** CDI-free tests with mocked API interfaces. Verify input mapping, output type conversion, edge cases.
- **Integration tests:** `@QuarkusTest` tests verifying end-to-end through the GraphQL endpoint. Use SmallRye GraphQL test client.
- **MCP registration tests:** Verify `GraphQLModelScanner` discovers all domains and `DynamicToolRegistrar` registers expected tools. Analogous to existing `ToolOverloadDiscoverabilityTest`.
- **Migration tests:** For each domain, verify the old @Tool method count decreases as resolvers land.

## Execution Plan — Sub-Issues

Each domain is an S or M sub-issue on this epic:

| Order | Domain | Scale | Rationale |
|-------|--------|-------|-----------|
| 1 | channels (refactor existing) | M | Refactor existing resolvers from @McpDomain("qhorus") to @McpDomain("channels"). Establishes the new package structure. |
| 2 | messaging | S | Small domain (13 ops). ConsumerMessaging + MessageDispatcher already exist. |
| 3 | agents | S | Small domain (8 ops). Needs InstanceManager facade. |
| 4 | data | S | Small domain (11 ops). DataStore already exists. |
| 5 | governance | M | Medium domain (9 ops). Needs WatchdogManager and CapacityReader facades. |
| 6 | audit | M | Medium domain (13 ops). Needs LedgerReader facade — most new API work. |
| 7 | compliance (refactor) | XS | Just rename @McpDomain("qhorus") to @McpDomain("compliance"). |

Order rationale: start with the domain that already has resolvers (channels), then domains with existing API facades (messaging, agents, data), then domains needing new facades (governance, audit). Compliance is a trivial rename done last.

## References

- `GraphQLModelScanner.java` — platform MCP, domain discovery via @McpDomain + @GraphQLApi
- `DynamicToolRegistrar.java` — platform MCP, casehub_action/casehub_activate registration
- `QhorusMcpTools.java` — runtime/mcp, 99 @Tool methods to migrate
- `QhorusQueryResolver.java`, `QhorusMutationResolver.java` — existing graphql/ resolvers
- `ChannelManager.java`, `ChannelReader.java` — API facade pattern reference
- Issue casehubio/qhorus#409 — epic issue
- Issue casehubio/qhorus#401 — established the pattern with routing operations
- Platform #240 — Dynamic MCP tool schema
- Platform #228 — MCP hierarchical model
- Decision review R1-01 — revised domain granularity from 15 to 6
- Decision review R1-06 — existing Reader interfaces acknowledgment
- Decision review R1-09 — annotation model clarification
- Decision review R1-10 — @McpServer scope transition
