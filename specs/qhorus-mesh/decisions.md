# Qhorus Mesh — Design Decisions

## D1: Transport

**Choice:** SSE MCP — each Claude Code session connects to the relay via MCP SSE transport
**Alternatives:**
- REST via bash/WebFetch — works immediately but more verbose per call, no native tool integration
- CLI wrapper + REST — cleaner than raw curl but still REST underneath, extra maintenance
**Rationale:** SSE MCP gives sessions native tool access (send_message, check_messages, etc.) indistinguishable from built-in tools. Zero friction per-call.
**Trade-offs:** Requires an MCP server process; more setup than raw REST
**Sources:** Claude Code MCP architecture, Qhorus REST endpoints (ChannelResource, AgentCardResource)
**Exploration:** quick
**Status:** captured

## D2: Server Scope

**Choice:** New lightweight module (qhorus-mesh) — purpose-built relay node
**Alternatives:**
- Use Claudony — full feature set but pulls in CaseHub engine, connectors, UI. Overkill.
- Standalone runtime profile — adds MCP tools back to qhorus runtime, slightly bloats the library
**Rationale:** Purpose-built for local LLM mesh. Fast startup, minimal dependencies. Native image target. Does not bloat the library or require Claudony.
**Trade-offs:** New module to build and maintain
**Sources:** Qhorus module structure, Claudony integration architecture
**Exploration:** quick
**Status:** captured

## D3: Identity Model

**Choice:** Path-based identity with searchable key/value metadata on both Channel and Instance
**Alternatives:**
- Random session UUID with capabilities — more flexible but less discoverable
- Slot-based identity — less useful for canonical repo sessions not in slots
**Rationale:** Path works consistently across canonical repos, slots, and clones. Metadata enables structured discovery ("who's working on qhorus?") without overloading capability tags.
**Trade-offs:** Metadata is a new core feature to build; path-based IDs are long strings
**Sources:** Slot structure (ctx.py), Instance capabilities model, Channel policyOverrides pattern
**Exploration:** quick (user-driven — pushed back on initial proposal, identified the consistency requirement)
**Status:** captured

## D4: Metadata — Core Feature

**Choice:** Add searchable `Map<String, String> metadata` to both Channel and Instance in qhorus core
**Alternatives:**
- Channel only — use existing capability tags on Instance as stopgap (but tags aren't key/value)
- Separate MetadataEntry table — more normalized but adds join complexity
**Rationale:** Both entities need metadata for the relay use case. Channel metadata for context (project, purpose, scope). Instance metadata for session identity (repo, slot, branch, issue). Same storage pattern as policyOverrides. Benefits all of Qhorus, not just the mesh.
**Trade-offs:** Two new nullable columns, two new query filters, two migrations
**Sources:** Channel.policyOverrides pattern, ChannelQuery/InstanceQuery filter patterns
**Exploration:** quick
**Status:** captured

## D5: Architecture — Local Relay with Location Transparency

**Choice:** Each machine runs a local Qhorus relay node. LLMs always connect to localhost. The relay handles routing — local messages stay local, remote messages are forwarded. The LLM does not know or care if the peer is local or remote.
**Alternatives:**
- Direct to central server — each LLM connects directly; no local relay. Simple but no location transparency, no offline capability.
- No relay, file-based — shared directory structure. Zero infrastructure but reinvents the wheel.
**Rationale:** Location transparency means the LLM's MCP config never changes regardless of deployment topology. The relay is cheap (native image, <100ms startup, ~50MB RAM). Offline resilience for co-located LLMs.
**Trade-offs:** Always-on process; server lifecycle management
**Sources:** ChannelGateway local vs. remote delivery, MessageObserver.Scope (LOCAL/CLUSTER), postgres-broadcaster
**Exploration:** quick (user-driven — proposed the relay pattern, asked about local vs. remote masking)
**Status:** captured

## D6: Persistence

**Choice:** H2 file-based (`~/.qhorus/mesh.mv.db`), SPI-swappable
**Alternatives:**
- PostgreSQL via Podman — enables postgres-broadcaster but heavier
- In-memory only — fastest but loses state on restart
**Rationale:** Storage is SPI — H2 is fine for now, others added later. Zero config, survives restarts, no container runtime needed.
**Trade-offs:** H2 can't federate via postgres-broadcaster; future federation needs a different transport
**Sources:** Qhorus store SPI pattern (ChannelStore, MessageStore, etc.), persistence-memory module
**Exploration:** quick
**Status:** captured

## D7: Federation Model — Selective Subscription with Lazy Replication

**Choice:** Local relay acts as a cache. Selective channel subscription (only channels with local members). Window-based recent message cache, on-demand fetch for older messages with time+size LRU eviction. Search always delegates to central. Full channel listings + metadata replicated locally.
**Alternatives:**
- Full replication (Matrix model) — every relay has complete room history. Too heavy.
- No local cache (IRC model) — ephemeral relay. Loses messages on disconnect.
- Strong consistency (NATS RAFT) — blocks writes without quorum. Wrong for disconnected relays.
**Rationale:** CDN model for chat channels. Recent messages always warm, old messages fetched on demand. Local relay stays small (bounded by cache window + eviction). Central server is source of truth for full history.
**Trade-offs:** Search requires central server connectivity; evicted messages need re-fetch
**Sources:** NATS JetStream, Matrix protocol, IRC federation, Eventuate CRDT framework, Apache Pekko replicated event sourcing
**Exploration:** deep-analysis (internet search for federation patterns, CRDT frameworks, distributed messaging)
**Status:** captured

## D8: Connection Shim — Stdio MCP Bridge

**Choice:** A stdio MCP process configured once in Claude Code's settings.json. On launch, checks for running local relay and bridges to it (stdio↔SSE proxy). If no relay found, starts embedded Qhorus in-process. LLM gets identical tools either way.
**Alternatives:**
- SSE-only (no shim) — simpler but requires relay always running; no fallback
- Manual config switching — user changes MCP URL based on deployment. Error-prone.
**Rationale:** Zero-config from the LLM's perspective. MCP config never changes. Benefits from local relay when present, works standalone when not.
**Trade-offs:** Shim is an extra component to build; stdio↔SSE bridge adds a hop
**Sources:** Claude Code MCP settings.json structure, stdio vs SSE MCP transport
**Exploration:** quick (user-driven — "must be fairly simple for the MCP client to not care if it's remote or local")
**Status:** captured

## D9: Channel Model

**Choice:** Ad-hoc, LLM-created. Channels created as needed — per-issue, per-topic, or ad-hoc. Channel metadata makes them discoverable. No auto-creation.
**Alternatives:**
- Auto-created per project family — simpler discovery but less flexible
- Hierarchical (family > project > issue) via Spaces — structured but over-engineered for initial use case
**Rationale:** Mirrors how humans use Slack/IRC: create a channel when you need one. Metadata-based discovery replaces hierarchical structure. YAGNI on auto-creation.
**Trade-offs:** LLMs must create channels explicitly; no guaranteed channel structure
**Sources:** ChannelService.create(), Space model, ChannelCreateRequest builder
**Exploration:** quick
**Status:** captured
