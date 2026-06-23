# casehub-qhorus — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** After casehub-ledger. Must publish before casehub-engine and all application-tier apps.
**Consumed by:** casehub-engine (via SPIs and event listeners), all application-tier harness apps, claudony
**Depends on:** casehub-ledger (MessageLedgerEntry chain).

---

## §1 Introduction and Goals

### Description

`casehub-qhorus` is the peer-to-peer agent communication mesh and governance methodology for multi-agent AI systems. It provides typed speech-act messaging across nine types — COMMAND (formal obligation request), DONE (successful fulfillment with findings), DECLINE (explicit scope refusal), RESPONSE (acknowledgment or information), FAILURE (unsuccessful discharge), HANDOFF (obligation transfer to a named delegate), STATUS (progress update), QUERY (information request), and EVENT (observer-only telemetry) — with commitment lifecycle tracking and automatic audit via `MessageLedgerEntry`.

The project is not middleware. It gives every agent interaction the formal status of an accountable act, grounded in speech act theory, deontic logic, defeasible reasoning, and social commitment semantics. The LLM reasons; the infrastructure enforces, records, and derives.

Core capabilities: `ChannelService` (create, find, list channels per coordination unit), `MessageService` (dispatch all 9 speech-act types; auto-creates Commitment on COMMAND/QUERY; auto-closes Commitment on DONE/DECLINE/FAILURE), `CommitmentService` (open, acknowledge, fulfill, decline, fail, delegate, expire commitments), `MessageLedgerEntry` (per-speech-act immutable audit record — enables tamper-evident review trail), `QhorusExtension` (Quarkus extension wiring datasource + CDI beans), and a suite of SPIs (`ObligorTrustPolicy`, `CommitmentAttestationPolicy`, `InstanceActorIdProvider`, `MessageTypePolicy`) that consumers replace via CDI displacement.

Named datasource `qhorus` — always separate from the default datasource. Always wire `quarkus.datasource.qhorus.*` in any consumer.

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume foundation primitives — channels, typed messages, commitments, ledger |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |
| casehub-engine | Consumes `CommitmentStore.findOpenByObligor` and `ChannelStore.findByIds` for actor state views |
| claudony | Embeds qhorus as the agent communication layer; provides `InstanceActorIdProvider` mapping session→persona |

### Quality Goals

| Goal | Scenario |
|---|---|
| Formal obligation | Every COMMAND creates a Commitment — no agent obligation exists without a machine-tracked record |
| Formal scope refusal | DECLINE is a typed outcome with recorded reason — not a timeout or exception |
| Audit chain | Every speech act produces a MessageLedgerEntry that can be verified independently of the channel state |
| Tenant isolation | Every query filters by tenancyId — data from different tenants never mixes |
| CDI extensibility | All SPIs are displaced via `@DefaultBean` — no fork required for any customisation |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-qhorus:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

### Architectural constraints
- casehub-qhorus has **no dependency on casehub-engine or claudony** — it is the independent communication layer. Domain logic, trust scoring beyond the ObligorTrustPolicy gate, and orchestration belong in consumers.
- All JPA entities live on the **named `qhorus` persistence unit**, never the default PU.
- Flyway migrations are in `db/qhorus/migration/` — not `db/migration/` — to avoid classpath conflicts when multiple modules are on the same classpath.
- The project publishes as a **Quarkus extension** (`api` + `runtime` + `deployment` modules) to give consuming apps zero-configuration CDI wiring.

---

## §3 Context and Scope

### System Context

casehub-qhorus sits at the foundation of the CaseHub platform's agent coordination stack:

```
┌─────────────────────────────────────────────────────────────┐
│                   Application-tier apps                      │
│     (devtown, clinical, life, aml, drafthouse, quarkmind)   │
└────────────────────────┬────────────────────────────────────┘
                         │ embeds
                         ▼
┌──────────────────────────────────┐      ┌──────────────────────┐
│           claudony               │      │   casehub-engine      │
│   (integration layer: embeds     │◄─────│  (orchestration:      │
│    both engine + qhorus)         │      │   case-based reasoning│
└──────────────┬───────────────────┘      └──────────┬───────────┘
               │ depends on                          │ depends on
               ▼                                    ▼
┌──────────────────────────────────────────────────────────────┐
│                    casehub-qhorus                             │
│   (agent communication mesh — this repo)                     │
│   Channels · Messages · Commitments · Ledger · SPIs          │
└──────────────────────────────────┬──────────────────────────┘
                                   │ depends on
                                   ▼
                    ┌─────────────────────────┐
                    │      casehub-ledger       │
                    │  (immutable audit ledger, │
                    │   Merkle hash chain,      │
                    │   trust scoring)          │
                    └─────────────────────────┘
```

**What casehub-qhorus owns:**
- Named channel management (create, find, pause, delete)
- 9-type speech-act dispatch with enforcement (type policy, rate limiting, ACL, trust gate)
- Commitment lifecycle (OPEN → FULFILLED / DECLINED / FAILED / DELEGATED / EXPIRED)
- MessageLedgerEntry write path (one entry per speech act)
- Backend gateway (ChannelBackend SPI, fan-out to all registered backends)
- A2A protocol bridge (POST /a2a/message:send, GET /a2a/tasks/{id}/stream)
- Agent Card (`/.well-known/agent-card.json`)
- Watchdog alerts (condition-based monitoring via WatchdogEvaluationService)
- Multi-tenancy enforcement (X-Tenancy-ID header → CurrentPrincipal → tenant-scoped queries)

**What casehub-qhorus does not own:**
- Trust score computation — delegated to casehub-ledger (Bayesian Beta, EigenTrust)
- Domain business rules — belong in application-tier repos
- Orchestration and case management — belong in casehub-engine
- Human task management — belongs in casehub-work

**Boundary rules:** See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

### External Interfaces

| Interface | Direction | Protocol |
|---|---|---|
| MCP tools (~53) | inbound from LLM agents | Quarkus MCP Server (streamable HTTP) |
| POST /a2a/message:send | inbound from A2A orchestrators | HTTP/JSON |
| GET /a2a/tasks/{id}/stream | outbound to A2A orchestrators | Server-Sent Events |
| GET /.well-known/agent-card.json | outbound to A2A discovery | HTTP/JSON |
| casehub-ledger JAR | outbound dependency | Maven / CDI injection |
| GitHub Packages | publish target | Maven deploy |

---

## §4 Solution Strategy

### Core architectural decisions

**Named datasource `qhorus`**
All JPA entities and repositories use the `qhorus` named persistence unit. This means `quarkus.datasource.qhorus.*` configuration is always required, and `Panache.withTransaction("qhorus", ...)` must be used in reactive code (bare `withTransaction()` routes to the default PU and fails silently in consumers that configure only the named PU). Flyway migrations are in `db/qhorus/migration/` and referenced via `quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration`.

**9-type speech-act taxonomy (ADR-0005)**
The taxonomy is grounded in a four-layer normative framework: speech act theory (illocutionary classification), deontic logic (obligation each type creates or discharges), social commitment semantics (who owes what to whom), and defeasible deontic logic (contrary-to-duty cases, HANDOFF defeat). The nine types cover the obligation lifecycle without overlap or gap — this completeness is a theoretical guarantee, not a design preference.

**`*Store` persistence seam**
Services inject store interfaces (`ChannelStore`, `MessageStore`, `CommitmentStore`, `InstanceStore`, `DataStore`, `WatchdogStore`), never Panache entity statics. This enables CDI-free unit tests via `InMemory*Store` from `casehub-qhorus-testing`, and contract tests via abstract base classes with two concrete runners (blocking and reactive).

**CDI `@DefaultBean` displacement for all SPIs**
Every consumer-facing SPI has a `@DefaultBean` implementation in the runtime module. Consumers replace it by declaring an `@Alternative @Priority(N)` bean — no configuration key, no fork. The four SPIs: `ObligorTrustPolicy`, `CommitmentAttestationPolicy`, `InstanceActorIdProvider`, `MessageTypePolicy`. The CDI principal (`QhorusInboundCurrentPrincipal`) uses plain `@ApplicationScoped` (Tier 2 in the CDI priority ladder) rather than `@DefaultBean` — this avoids `AmbiguousDependencies` when two `@DefaultBean` implementations of the same type are on the classpath.

**`@IfBuildProperty` / `@UnlessBuildProperty` for blocking/reactive bean selection**
`casehub.qhorus.reactive.enabled` is a `BUILD_TIME`-only property declared in `QhorusBuildTimeConfig`. Setting it to `true` activates reactive beans and deactivates their blocking counterparts. The property must not appear in test `application.properties` (causes `SRCFG00050` runtime validation error) — test profiles set it via `getConfigOverrides()`.

**Enforcement gate in `MessageService.dispatch()`**
All channel writes flow through a single gate: paused check → `AllowedWritersPolicy` ACL → `RateLimiter` → `ObligorTrustPolicy` SPI (COMMAND + named non-prefixed target) → `MessageTypePolicy` → LAST_WRITE semantics → `ChannelGateway.fanOut()`. There is no bypass. MCP tools call `dispatch()` and receive the same enforcement as any other caller.

**LedgerWriteService writes for all 9 types**
`LedgerWriteService.record()` fires unconditionally for every `dispatch()` call. Attestations (SOUND/FLAGGED confidence scores) are derived from `MessageType` directly — not from a CommitmentStore query — because a `REQUIRES_NEW` transaction cannot see uncommitted outer-transaction state.

---

## §5 Building Block View

### Module structure

```
casehub-qhorus/
├── api/                    SPI contracts and DTOs — no JPA, no CDI, no runtime deps
├── runtime/                All CDI beans, JPA entities, MCP tools, REST endpoints
├── deployment/             Build-time processors — FeatureBuildItem only
├── testing/                InMemory*Store and InMemoryReactive*Store (@Alternative @Priority(1))
├── connector-backend/      Optional: bridges casehub-connectors InboundMessage events into Qhorus dispatch
├── slack-channel/          Optional: SlackBotClient HumanParticipatingChannelBackend
└── examples/
    ├── type-system/         CI regression tests for 9-type taxonomy (no LLM)
    ├── normative-layout/    Deterministic 3-channel NormativeChannelLayout tests (CI, no LLM)
    └── agent-communication/ Real LLM examples (Jlama) + normative benchmark (Zone 1/2/3); -Pwith-llm-examples
```

### api module

Consumer-facing contracts. No JPA, no CDI, no Quarkus runtime dependencies.

| Package | Contents |
|---|---|
| `api.channel` | `ChannelDetail` (DTO record) |
| `api.instance` | `InstanceInfo` (DTO record) |
| `api.gateway` | `ChannelBackend` hierarchy, `MessageObserver` SPI, `MessageReceivedEvent`, `ChannelInitialisedEvent`, `NormalisedMessage`, `InboundHumanMessage`, `OutboundMessage` |
| `api.message` | `MessageResult`, `MessageType` enum, `CommitmentDeclinedEvent`, `CommitmentExpiredEvent`, `CommitmentState` enum |
| `api.spi` | `ObligorTrustPolicy`, `ObligorTrustContext`, `CommitmentAttestationPolicy`, `InstanceActorIdProvider` |

### runtime module — key packages

| Package | Responsibility |
|---|---|
| `runtime.config` | `QhorusConfig` — all `casehub.qhorus.*` config mapping |
| `runtime.channel` | `Channel` entity, `ChannelService`, `ChannelSemantic`, `AllowedWritersPolicy`, `RateLimiter` |
| `runtime.message` | `Message` entity, `Commitment` entity, `MessageService`, `CommitmentService`, `MessageObserverDispatcher` |
| `runtime.instance` | `Instance` entity, `Capability` entity, `InstanceService` |
| `runtime.data` | `SharedData` entity, `ArtefactClaim` entity, `DataService` |
| `runtime.identity` | `QhorusInboundCurrentPrincipal`, `TenancyContextFilter`, `InboundTenancyContext`, `CrossTenantProducer` |
| `runtime.audit` | `EvidentialChecker` — vocabulary violation detection (I_df: false DONE; I_ec: wrong terminal type) |
| `runtime.ledger` | `MessageLedgerEntry`, `LedgerWriteService`, `QhorusLedgerEntryRepository`, `QhorusSequenceAllocator`, attestation policy impls |
| `runtime.mcp` | `QhorusMcpToolsBase`, `QhorusMcpTools` (~53 `@Tool` methods), `ReactiveQhorusMcpTools` |
| `runtime.gateway` | `ChannelGateway`, `QhorusChannelBackend`, `DefaultInboundNormaliser`, `InProcessMessageBus`, `QhorusCloudEventAdapter` |
| `runtime.watchdog` | `Watchdog` entity, `WatchdogEvaluationService`, `WatchdogScheduler` |
| `runtime.api` | `AgentCardResource`, `A2AResource`, `A2AChannelBackend`, `A2ATaskState`, `A2AActorResolver` |

### deployment module

`QhorusProcessor` — single `@BuildStep` producing `FeatureBuildItem("qhorus")`. Bean selection is handled at the bean level via `@IfBuildProperty` and `@UnlessBuildProperty`, not by the processor.

### optional modules

**connector-backend:** Activated by classpath presence. `ConnectorChannelBackend` bridges `InboundMessage` CDI events from casehub-connectors into Qhorus dispatch. `ConnectorQhorusMeshBridge` posts STATUS back to a configured delivery channel after each connector delivery.

**slack-channel:** Activated by classpath presence. `SlackChannelBackend` is a `HumanParticipatingChannelBackend`. Thread continuity via composite in-memory + DB-backed cache (`SlackThreadCacheStore`). Entities in `io.casehub.qhorus.slack` — consumers must add this package to `quarkus.hibernate-orm.qhorus.packages`.

### testing module

`InMemory*Store` and `InMemoryReactive*Store` with `@Alternative @Priority(1)` — selected over JPA stores in `@QuarkusTest` environments. Key constraint: InMemory store implementations must not mutate `PanacheEntity` fields in-place within a `Panache.withSession()` scope — Hibernate's bytecode enhancement dirty-tracks writes and generates spurious flushes even for entities that were never persisted.

---

## §6 Runtime View

### Scenario 1: COMMAND → DONE commitment lifecycle

```
Agent → send_message(type=COMMAND, correlationId=X) via MCP
  → QhorusMcpTools.sendMessage()
  → MessageService.dispatch(MessageDispatch)
      ├── paused check
      ├── AllowedWritersPolicy ACL
      ├── RateLimiter
      ├── ObligorTrustPolicy SPI (named target → trust check)
      ├── MessageTypePolicy (hard-enforce COMMAND/QUERY; advisory for others)
      ├── messageStore.put(message)
      ├── CommitmentService.open(correlationId, obligor, requester)  ← Commitment OPEN
      ├── LedgerWriteService.record(channel, message)  ← MessageLedgerEntry written
      └── ChannelGateway.fanOut(message)
            ├── QhorusChannelBackend.post()  ← no-op (persistence already done)
            ├── A2AChannelBackend.post()     ← SSE delivery if stream registered
            └── MessageObserver.onMessage()  ← CDI event via InProcessMessageBus

Agent → send_message(type=DONE, correlationId=X) via MCP
  → MessageService.dispatch()
      ├── [enforcement gate]
      ├── messageStore.put(message)
      ├── CommitmentService.fulfill(correlationId)  ← Commitment FULFILLED
      ├── LedgerWriteService.record()
      │     ├── writes MessageLedgerEntry (DONE)
      │     └── writes LedgerAttestation (SOUND, confidence 0.7 default)
      └── ChannelGateway.fanOut()
```

**Key invariant:** The attestation verdict is derived from `MessageType` directly (DONE → SOUND, FAILURE/DECLINE → FLAGGED, RESPONSE used for COMMAND → FLAGGED with lower confidence). No `CommitmentStore` query inside `REQUIRES_NEW` — that would see stale OPEN state before the outer transaction commits.

### Scenario 2: Channel creation and backend self-registration

```
create_channel(name="work", semantic=APPEND) via MCP
  → ChannelService.create(ChannelCreateRequest)
      ├── validates slug, type constraint disjointness
      ├── channelStore.put(channel)  [REQUIRES_NEW]
      └── channelGateway.initChannel(channel)
            ├── registers QhorusChannelBackend as default slot
            └── fires ChannelInitialisedEvent(channelId, channelName)

@Observes ChannelInitialisedEvent  ← any backend that declares this
  → ConnectorChannelBackend.onChannelInitialised()   ← re-registers for its binding
  → SlackChannelBackend.onChannelInitialised()       ← re-registers if binding exists
```

**On startup:** `ChannelGateway` observes `StartupEvent` and calls `initChannel()` for every channel returned by `ChannelService.listAll()`. Each call fires `ChannelInitialisedEvent` — backends self-recover without their own restart logic.

### Scenario 3: Multi-tenancy enforcement

```
HTTP request → TenancyContextFilter (@PreMatching @Priority(100))
  → reads X-Tenancy-ID header
  → InboundTenancyContext.set(tenancyId)  [null/blank → DEFAULT_TENANT_ID]

MessageService.dispatch()
  → messageDispatch.tenancyId() == null
      → resolves from CurrentPrincipal.tenancyId()  ← reads InboundTenancyContext
  → all store queries filter: WHERE tenancy_id = :tenancyId

Background services (WatchdogEvaluationService, ChannelGateway startup):
  → inject @CrossTenant stores via CrossTenantProducer
  → QhorusSystemCurrentPrincipal.isCrossTenantAdmin() == true
  → stores issue queries without tenancy filter
```

---

## §7 Deployment View

**Published artifact:** `io.casehub:casehub-qhorus:0.2-SNAPSHOT` on GitHub Packages (`https://maven.pkg.github.com/casehubio/*`).

**CI:** GitHub Actions. Workflow dispatches downstream builds to casehub-engine, claudony, devtown, drafthouse, and other consumers when the main branch changes. This ensures API breakage is caught at the source rather than at consumer build time.

**Consumer dependency:**
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-qhorus</artifactId>
  <version>${casehub.qhorus.version}</version>
</dependency>
```

**Minimum consumer configuration:**
```properties
quarkus.datasource.qhorus.db-kind=postgresql
quarkus.datasource.qhorus.jdbc.url=jdbc:postgresql://...
quarkus.datasource.qhorus.username=...
quarkus.datasource.qhorus.password=...
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime
quarkus.flyway.qhorus.migrate-at-start=true
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration
```

**Blocking vs. reactive selection** (build-time):
```properties
# Enable reactive stack (PostgreSQL reactive driver required)
casehub.qhorus.reactive.enabled=true
```

**No GraalVM native target** for this foundation module — native image builds are the responsibility of application-tier consumers (e.g. claudony).

**Optional modules** are activated by classpath presence — no additional configuration keys beyond their own `casehub.qhorus.connector-backend.*` or `casehub.qhorus.slack.*` properties.

---

## §8 Crosscutting Concepts

### Protocol references

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |
| EVENT content-free rule | PP-20260608-054090 |
| Observer test transaction discipline | PP-20260608-07daa6 |
| InMemory store no-entity-mutation | PP-20260618-100368 |
| Command obligation verification type check | PP-20260623-fd69f3 |
| Ledger sequence table test init | PP-20260609-e5ac14 |
| Reactive gating property | PP-20260520-a86db7 |

### Anti-patterns

**1. Missing `quarkus.datasource.qhorus.*` configuration**

*Symptom:* `UnsatisfiedResolutionException` at CDI validation, or `NullPointerException` at runtime on `@Default EntityManager`.

*Cause:* All Qhorus JPA entities live on the named `qhorus` persistence unit. Consumers that add `casehub-qhorus` to their classpath without configuring the named datasource get CDI injection failures at startup.

*Fix:* Add the full `quarkus.datasource.qhorus.*` block (db-kind, jdbc.url, username, password) plus `quarkus.hibernate-orm.qhorus.packages` and `quarkus.flyway.qhorus.locations` to `application.properties`.

---

**2. Using bare `Panache.withTransaction(() -> ...)` in reactive code**

*Symptom:* Operations silently target the wrong datasource; entities written by `withTransaction()` are not visible to `channelStore` queries. In consumer apps (e.g. claudony) that configure only the named qhorus datasource, a `NoPersistenceProviderFoundException` is thrown.

*Cause:* Bare `Panache.withTransaction()` routes to the default persistence unit. Qhorus uses the named `qhorus` PU.

*Fix:* Always use `Panache.withTransaction("qhorus", () -> ...)` in reactive code. For blocking code, the named `@PersistenceContext(unitName = "qhorus")` is injected by the store.

---

**3. `ledger_subject_sequence` table not present in test schema**

*Symptom:* `Table "LEDGER_SUBJECT_SEQUENCE" not found` or `H2 syntax error` on the MERGE INTO statement during tests with `casehub.ledger.enabled=true`.

*Cause:* `ledger_subject_sequence` is not a JPA entity — Hibernate `drop-and-create` does not create it. It is created by the Flyway migration only.

*Fix:* Test modules that enable the ledger must include `import-qhorus-test.sql` (one `CREATE TABLE IF NOT EXISTS ledger_subject_sequence ...` statement) and set `quarkus.hibernate-orm.qhorus.sql-load-script=import-qhorus-test.sql`.

---

**4. RESPONSE sent on COMMAND correlationId fulfills commitment without normative flag**

*Symptom:* A COMMAND obligation appears FULFILLED in `CommitmentStore` even though the obligor sent RESPONSE (not DONE, FAILURE, or DECLINE). The infrastructure accepted the message without complaint. No attestation violation is visible until `EvidentialChecker.checkObligation()` is called.

*Cause:* `CommitmentService.fulfill()` is called for any message that carries a matching `correlationId` including RESPONSE. This is correct protocol behaviour — RESPONSE discharges an obligation — but for a COMMAND the semantically correct terminal type is DONE, FAILURE, or DECLINE. Using RESPONSE is a vocabulary error (I_ec violation) that the commitment state machine cannot distinguish.

*Fix:* `StoredCommitmentAttestationPolicy` now writes FLAGGED attestation (confidence 0.3) when RESPONSE is used to close a COMMAND obligation (fixed in #305, PP-20260623-fd69f3). `EvidentialChecker.checkObligation(terminalType, CommitmentContext)` provides the deterministic vocabulary check for callers who need it explicitly.

---

**5. `casehub.qhorus.reactive.enabled` in `application.properties` causes `SRCFG00050`**

*Symptom:* Application fails to start with `SRCFG00050: The config property casehub.qhorus.reactive.enabled with value true is not in the correct format`.

*Cause:* `casehub.qhorus.reactive.enabled` is declared in `QhorusBuildTimeConfig` as `@ConfigRoot(BUILD_TIME)`. SmallRye Config validates that BUILD_TIME-only properties are not present at runtime.

*Fix:* Do not put this property in `application.properties`. For tests, set it via `QuarkusTestProfile.getConfigOverrides()` — this applies the override at augmentation time (Quarkus context restart) without triggering runtime validation.

---

**6. `@DefaultBean` principal causes `AmbiguousDependencies` in consumers**

*Symptom:* Consumer apps fail CDI validation with `AmbiguousDependencies` for `CurrentPrincipal` when `QhorusInboundCurrentPrincipal` is on the classpath alongside `MockCurrentPrincipal`.

*Cause:* If `QhorusInboundCurrentPrincipal` is annotated `@DefaultBean`, two `@DefaultBean` implementations of the same type coexist, and CDI cannot resolve the ambiguity.

*Fix:* `QhorusInboundCurrentPrincipal` uses plain `@ApplicationScoped` (Tier 2 in the CDI priority ladder). It displaces `MockCurrentPrincipal @DefaultBean` automatically and is itself displaced by `@Alternative @Priority(N)` beans. Do not add `quarkus.arc.exclude-types=MockCurrentPrincipal` as a workaround — the plain `@ApplicationScoped` annotation makes it unnecessary.

---

## §9 Journeys and Chapters

Foundation modules use Chapters to track delivery milestones of the foundation capability itself.

---

### Chapter 1 — Mesh Design and Day Zero

**Delivered:** April 2026
**Issues:** design spec, scaffold generation
**What shipped:** The project design spec (`docs/specs/2026-04-13-qhorus-design.md`) — forty pages covering ER diagrams, Flyway schema, full MCP tool surface, and a build roadmap. The scaffold: `pom.xml`, `CLAUDE.md`, module structure. No Java code — vision made concrete enough to build from. The three-project ecosystem was defined on paper: CaseHub (orchestration), Qhorus (communication mesh), Claudony (integration layer).

Key design decisions made before writing code:

- Five channel semantics: APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE
- Artefact refs (`List<UUID>`) on messages with claim/release lifecycle — inline payloads don't scale
- Streamable HTTP transport only (MCP 2025-06-18)
- Named datasource `qhorus` — always separate from default
- No dependency on casehub-engine or claudony

The research pass covered A2A, ACP, AutoGen, LangGraph, OpenAI Swarm, Letta, and CrewAI. The conclusion: most frameworks assume a central orchestrator; Qhorus would sit orthogonal to all of them, not as another orchestration model.

#### Key files
- `docs/specs/2026-04-13-qhorus-design.md` — primary design spec

#### Design decisions
- Seven message types at day zero (expanded to 9 in Chapter 2 — QUERY split from REQUEST, EVENT formalised)
- Quarkus extension pattern chosen from the start — zero-configuration CDI wiring for consumers

#### Status
✅ Complete

---

### Chapter 2 — Speech Act Taxonomy

**Delivered:** April 2026 (2026-04-23)
**Issues:** ADR-0005 (referenced throughout)
**What shipped:** The 9-type speech-act taxonomy and its formal theoretical grounding. QUERY split from REQUEST (COMMAND). Each type maps to exactly one obligation operation: COMMAND/QUERY create, DONE/FAILURE discharge, DECLINE refuses, HANDOFF transfers, STATUS updates, EVENT observes. ADR-0005 documents the completeness argument and the contrary-to-duty cases.

A classification accuracy test (`MessageTaxonomyTest` in `examples/type-system/`) runs in CI with no model — passes natural language scenarios through the schema description and verifies the model selects the expected enum value. This is the empirical validation for the taxonomy's granularity.

The `quarkus-langchain4j-jlama` examples module was added in this phase. A Quarkus 3.32.2 bootstrap serializer bug (`Unsupported value type: [ALL-UNNAMED]`) prevents Jlama from loading — root cause traced to a `quarkus-extension.properties` entry in the Jlama runtime JAR. The examples module was gated behind `-Pwith-llm-examples` pending an upstream fix.

#### Key files
- `api/src/main/java/io/casehub/qhorus/api/message/MessageType.java` — 9-type enum
- `docs/adr/ADR-0005.md` — formal obligation lifecycle argument
- `examples/type-system/` — CI classification regression tests

#### Design decisions
- Four-layer normative framework: speech act theory + deontic logic + social commitment semantics + defeasible deontic logic
- QUERY creates obligation to inform-or-DECLINE; COMMAND creates obligation to execute-or-DECLINE — different semantics, different reply expectations
- EVENT is observer-only; agents must not see their own telemetry as part of their conversational context

#### Status
✅ Complete

---

### Chapter 3 — Commitment Store and Lifecycle

**Delivered:** April 2026 (2026-04-24)
**Issues:** #89–#97
**What shipped:** `CommitmentStore` and `CommitmentService` — a seven-state obligation state machine replacing the thin `PendingReply` record. States: `OPEN → ACKNOWLEDGED → FULFILLED | DECLINED | FAILED | DELEGATED | EXPIRED`. `PendingReply` was deleted entirely (entity, cleanup job, SPI, JPA implementations, InMemory stores, contract tests). 139 new tests.

The delegation constraint surfaced during E2E testing: HANDOFF creates a child Commitment with the same `correlationId` as the parent. The unique constraint on `correlationId` prevented this. Fix: remove the uniqueness constraint; `findByCorrelationId` prefers the non-terminal commitment when multiple records share a key. This is correct — a `correlationId` identifies a *conversation*, not an obligation holder.

The `CommitmentServiceTest` module placement is a structural constraint: unit tests live in `testing/src/test/java/` (not `runtime/src/test/`) to avoid a module cycle. `service.store = store` wires the dependency directly via package-private field access.

#### Key files
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/Commitment.java` — JPA entity
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java` — state machine
- `api/src/main/java/io/casehub/qhorus/api/message/CommitmentState.java` — 7-state enum with `isTerminal()`
- `testing/src/main/java/.../InMemoryCommitmentStore.java` — test double

#### Design decisions
- `correlationId` is a conversation key, not a commitment key — uniqueness constraint removed to support delegation chains
- Unit tests in `testing/` module to avoid cycle with `runtime/`

#### Status
✅ Complete

---

### Chapter 4 — Normative Ledger

**Delivered:** April 2026 (2026-04-27)
**Issues:** #123, #124
**What shipped:** `MessageLedgerEntry` extended from EVENT-only to all 9 message types. `LedgerWriteService.record()` fires unconditionally for every dispatch. `list_ledger_entries` replaces `list_events` as the primary query surface. `LedgerAttestation` entries written on terminal transitions (DONE → SOUND, FAILURE/DECLINE → FLAGGED).

Two SPIs shipped in this chapter:

**`CommitmentAttestationPolicy`** (SPI in `api/spi/`): determines `LedgerAttestation` for DONE/FAILURE/DECLINE. `StoredCommitmentAttestationPolicy` is the `@DefaultBean` impl. The key design constraint: attestation verdict is derived from `MessageType` directly, not from a `CommitmentStore` query — a `REQUIRES_NEW` transaction cannot see the uncommitted outer-transaction state.

**`InstanceActorIdProvider`** (SPI in `api/spi/`): maps instanceId → ledger actorId. Default is identity. Claudony will provide the real mapping from `SessionRegistry` — so trust scores accrue to a persona, not a session-scoped instance ID.

Also shipped: `casehub-ledger` trust score fix — `TrustScoreComputer` had been ignoring the `confidence` field on `LedgerAttestation`, using `recencyWeight` alone. Fixed: `weight = recencyWeight × clamp(confidence, 0, 1)`.

#### Key files
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java` — JPA JOINED subclass
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java` — unconditional write path
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/StoredCommitmentAttestationPolicy.java` — `@DefaultBean`
- `api/src/main/java/io/casehub/qhorus/api/spi/CommitmentAttestationPolicy.java` — SPI
- `api/src/main/java/io/casehub/qhorus/api/spi/InstanceActorIdProvider.java` — SPI

#### Design decisions
- Attestation verdict from `MessageType`, not `CommitmentStore` — avoids REQUIRES_NEW visibility gap
- `causedByEntryId` links DONE/FAILURE/DECLINE/HANDOFF entries to originating COMMAND — causal DAG across 9 types

#### Status
✅ Complete

---

### Chapter 5 — Channel Type Enforcement

**Delivered:** April 2026 (2026-04-28); extended May 2026 (#243, #271)
**Issues:** #122, #243, #247, #246, #271
**What shipped:** `allowedTypes` and `deniedTypes` columns on the channel entity, enforced by `StoredMessageTypePolicy`. Type constraints are stored as sorted canonical CSV (`MessageType.serializeTypes(Set<MessageType>)`). The 3-channel normative layout (`work`, `observe`, `oversight`) became a first-class Qhorus concept — `examples/normative-layout/` ships 27 CI tests (no LLM).

**Hybrid enforcement model** (ADR-0016, #271): COMMAND and QUERY violations are **hard-enforced** — `MessageTypePolicy.validate()` throws `MessageTypeViolationException` and the dispatch is rejected. All other type violations produce **advisories** in `DispatchResult.advisories()` and a WARN log. Denial-first logic applies: denied types are evaluated before allowed types for COMMAND/QUERY.

`ChannelCreateRequest` compact constructor takes typed `Set<MessageType>` (not String) and asserts `allowedTypes ∩ deniedTypes = ∅` — an invalid request throws `IllegalArgumentException` before any DB interaction. MCP tools parse String parameters at the boundary via `MessageType.parseTypes()`.

`set_channel_type_constraints` MCP tool provides full-replacement semantics — passing null for a set clears that constraint.

#### Key files
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java` — `allowedTypes`, `deniedTypes` columns
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` — `setTypeConstraints(UUID, Set<MessageType>, Set<MessageType>)`
- `examples/normative-layout/` — 3-channel layout CI tests
- `docs/adr/ADR-0016.md` — hybrid enforcement decision

#### Design decisions
- Hard enforcement only for COMMAND/QUERY — these create orphan Commitments if the wrong channel accepts them
- Advisory enforcement for STATUS/RESPONSE/EVENT — less severe consequence; operator can observe and decide
- `examples/normative-layout/` as a canonical importable reference — CI-clean, deterministic, no LLM

#### Status
✅ Complete

---

### Chapter 6 — Named Datasource and DB Independence

**Delivered:** April–May 2026 (#172, #177, #142)
**Issues:** #142, #172, #177, #179, #180
**What shipped:** The `qhorus` named persistence unit was established from the start, but this chapter covers the work to make it robust: the Flyway migration path move (`db/migration/qhorus/` → `db/qhorus/migration/` after #142 discovered a classpath collision with casehub-work), the reactive dual-stack (`casehub.qhorus.reactive.enabled` BUILD_TIME property, `@IfBuildProperty`/`@UnlessBuildProperty` bean selection), and alignment of all reactive Panache calls to use `withTransaction("qhorus", ...)`.

**Migration path history:** V1–V9 started in `db/migration/`. After the classpath collision with casehub-work V2, all migrations moved to `db/qhorus/migration/`. Then moved again to `db/migration/qhorus/` (intermediate), and finally settled at `db/qhorus/migration/` (final). Next domain migration: **V25** (see CLAUDE.md for full migration version list).

**Reactive gating:** `QhorusBuildTimeConfig` declares `casehub.qhorus.reactive.enabled` as `@ConfigRoot(BUILD_TIME)`. This makes `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")` reliable — the property is evaluated at augmentation time, not runtime.

**Ledger migration inclusion:** `classpath:db/ledger/migration` added to Flyway locations so qhorus consumers get the ledger schema without depending on casehub-ledger's own Flyway config.

#### Key files
- `runtime/src/main/resources/db/qhorus/migration/` — all Flyway migrations (V1–V24 shipped)
- `deployment/src/main/java/.../QhorusBuildTimeConfig.java` — BUILD_TIME config root
- `deployment/src/main/java/.../QhorusProcessor.java` — feature registration only

#### Design decisions
- `db/qhorus/migration/` namespace prevents classpath conflicts when multiple CaseHub modules coexist
- BUILD_TIME property for reactive gating — `@IfBuildProperty` is unreliable without a `@ConfigRoot(BUILD_TIME)` declaration

#### Status
✅ Complete

---

### Chapter 7 — ObligorTrust SPI and Commitment Gate

**Delivered:** May 2026 (#213, #235)
**Issues:** #213, #235
**What shipped:** `ObligorTrustPolicy` SPI extracted from `MessageService` — the trust gate for COMMAND messages with a named non-prefixed target. `DefaultObligorTrustPolicy @DefaultBean` is the default: returns `true` when `minObligorTrust ≤ 0` (gate disabled), otherwise delegates to `TrustGateService`.

`ObligorTrustContext` record carries `obligorId`, `channelId (UUID)`, `channelName` — passed to `ObligorTrustPolicy.permits()`. Consumers implement the SPI as a `@FunctionalInterface` lambda.

**Reactive parity** (#235): The reactive trust gate (`ReactiveMessageService`) had been calling `trustGateService.meetsThresholdAsync()` directly, bypassing the SPI entirely. Fix: wrap `obligorTrustPolicy.permits(ctx)` in `Uni.createFrom().item(() -> ...).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`. Custom `ObligorTrustPolicy` beans are now honoured in both blocking and reactive paths. `BlockingTierPurityTest` guards against future regression.

#### Key files
- `api/src/main/java/io/casehub/qhorus/api/spi/ObligorTrustPolicy.java` — SPI
- `api/src/main/java/io/casehub/qhorus/api/spi/ObligorTrustContext.java` — context record
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/DefaultObligorTrustPolicy.java` — `@DefaultBean`
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — enforcement gate

#### Design decisions
- Trust gate applies only to COMMAND + named non-prefixed target — other types and prefixed targets (role:X, system:X) are not subject to trust checks
- `minObligorTrust ≤ 0` short-circuits before any JPA call — gate is off by default, opt-in

#### Status
✅ Complete

---

### Chapter 8 — Multi-Tenancy

**Delivered:** June 2026 (#260, #265, #269, #276)
**Issues:** #260, #265, #269, #276
**What shipped:** `tenancyId` column on `Channel`, `Message`, `Commitment`, `Watchdog`, and `MessageLedgerEntry` (V18–V21 migrations). All JPA store queries filter by tenancyId. `TenancyContextFilter` (`@PreMatching @Priority(100)`) reads `X-Tenancy-ID` from HTTP requests and populates `InboundTenancyContext`. `QhorusInboundCurrentPrincipal` reads from `InboundTenancyContext`; null/blank headers resolve to `DEFAULT_TENANT_ID`.

`CrossTenantProducer` produces `@CrossTenant`-qualified stores for background services (`WatchdogEvaluationService`, `ChannelGateway` startup) that must operate across all tenants. `QhorusSystemCurrentPrincipal` (`isCrossTenantAdmin() == true`) is injected by `CrossTenantProducer`.

`X-Tenancy-ID` is **not a security boundary** — it is a routing hint for data isolation in development and multi-tenant staging. Production deployments pair it with OIDC principal extraction (`OidcCurrentPrincipal @Alternative @Priority(100)`).

#### Key files
- `runtime/src/main/java/io/casehub/qhorus/runtime/identity/TenancyContextFilter.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/identity/InboundTenancyContext.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/identity/QhorusInboundCurrentPrincipal.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/identity/CrossTenantProducer.java`
- `runtime/src/main/resources/db/qhorus/migration/V18__tenancy_id_channel.sql` — first tenancy migration

#### Design decisions
- Plain `@ApplicationScoped` for `QhorusInboundCurrentPrincipal` (not `@DefaultBean`) — avoids `AmbiguousDependencies` in consumers with multiple CurrentPrincipal implementations
- Background services use `@CrossTenant` stores explicitly — prevents accidental single-tenant scoping in scheduled tasks

#### Status
✅ Complete

---

### Chapter 9 — A2A Protocol Bridge

**Delivered:** May–June 2026 (#135, #147, #278)
**Issues:** #135, #147, #278
**What shipped:** Full A2A protocol bridge: `POST /a2a/message:send` (sends message, returns task state), `GET /a2a/tasks/{id}` (polls commitment state), `GET /a2a/tasks/{id}/stream` (SSE streaming with keepalives and orphan detection). `A2AChannelBackend` registers as `ChannelBackend "a2a"` and routes SSE delivery via a `ConcurrentHashMap<UUID, Set<Consumer<OutboundMessage>>>` keyed by correlationId.

**SSE active model** (#278): `@RunOnVirtualThread` + `LinkedBlockingQueue<OutboundMessage>`. The `poll(heartbeatMs)` loop drives named keepalive events (`event: keepalive`). `sink.isClosed()` orphan detection. Deadline enforces `max-duration-seconds`. `SseEventSink.send()` returns `CompletionStage<?>` — `sink.close()` must run inside `.thenRun()` on the returned stage, not synchronously (race condition against the async write).

**DECLINE → "cancelled" mapping**: DECLINE is an explicit agent refusal. Mapping to "failed" conflated a decision with an infrastructure error — fixed across all three `A2ATaskState` paths. `A2ATaskState.TERMINAL_TYPES` and `TERMINAL_STATES` constants provide string-level terminal checks.

**Lazy registration constraint**: SSE subscriptions do not survive server restart — `A2AChannelBackend` does not observe `ChannelInitialisedEvent`. `streamTask()` auto-calls `ensureRegistered()` as a mitigation, but this is a known gap (ADR-0013).

`A2AActorResolver` provides a 6-step sender identity resolution chain for A2A `role:"user"`: header → Instance registry → agentCardUrl → persona format → system → default HUMAN.

#### Key files
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AActorResolver.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java`
- `docs/adr/ADR-0013.md` — lazy registration decision
- `docs/adr/ADR-0014.md` — SSE consumer registry design
- `docs/adr/ADR-0015.md` — reactive trust gate worker-pool

#### Design decisions
- SSE runs on virtual threads — allows `@Transactional` for initial DB reads, which commits before the stream stays open (correct semantics for an SSE producer)
- `ConcurrentHashMap.compute()` for deregisterStream — atomic empty-removal prevents TOCTOU orphan where a new consumer is registered and immediately discarded
- DECLINE → "cancelled" (A2A spec has no "declined" state; "cancelled" is the closest semantic match)

#### Status
✅ Complete

---

### Chapter 10 — Channel Projection SPI

**Delivered:** May 2026 (#230, #231, #232, #239, #240)
**Issues:** #230, #231, #232, #239, #240
**What shipped:** `RenderableProjection<S>` SPI — a fold-based view over a channel's message history. `ProjectionRegistry` collects all CDI beans implementing `RenderableProjection<?>` and validates: no null/blank `projectionName()`, no duplicate names (both throw `IllegalStateException` at CDI bootstrap). `ProjectionService` evaluates projections via left-fold from `identity()` state.

`project_channel(channel, projection_name, max_messages?)` MCP tool: renders any registered projection for any channel. `max_messages` bounds fold depth (null = unlimited). `list_projections()` exposes registered names.

`resolveChannel(String channel)` in `QhorusMcpToolsBase` does two-phase UUID parsing (`tryParseUuid` returning null on failure) — both UUID and slug are accepted. Matching `resolveChannelAsync(String) → Uni<Channel>` exists in `ReactiveQhorusMcpTools`.

`ProjectionResult<S>.isEmpty()` is `true` when `lastMessageId == null`. Incremental `project()` performs full scan when passed an empty result — the service enforces this invariant.

#### Key files
- `api/src/main/java/io/casehub/qhorus/api/message/RenderableProjection.java` — SPI (in api module)
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/ProjectionService.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/ProjectionRegistry.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — `projectAndRender()`

#### Design decisions
- Projection SPI in `api` module — consumers can implement projections without a runtime dependency
- `ProjectionRegistry` validates at CDI bootstrap — duplicate names fail fast before any request is served
- Full fold from `identity()` when previous result is empty — no partial state corruption from stale cursors

#### Status
✅ Complete

---

### Chapter 11 — Normative Benchmark

**Delivered:** June 2026 (#296, #297, #298, #299, #303, #304, #305)
**Issues:** #296, #297, #298, #299, #303, #304, #305
**What shipped:** Three-zone normative benchmark in `examples/agent-communication/` (behind `-Pwith-llm-examples`), an `EvidentialChecker` in the runtime module, and `CommitmentAttestationPolicy` update to detect RESPONSE-used-for-COMMAND violations.

**Zone 1** (unstructured baseline): Llama 3.2 1B at temperature 0.1 shows 70–80% cheating on empty-channel and counterfactual variants — the model confidently fabricating COMPLETED responses for definitionally impossible tasks.

**Zone 2** (normative infrastructure): Typed channels with `allowedTypes = COMMAND,STATUS,FAILURE,DECLINE,DONE`. False DONE count falls to zero — but the model shifts to RESPONSE instead of DONE. RESPONSE with a matching `correlationId` fulfills the commitment (`CommitmentState.FULFILLED`). The infrastructure accepted the message. This is a semantic confusion (I_ec violation — wrong terminal type) that the commitment state machine cannot distinguish from correct behavior.

**Zone 3** (`EvidentialChecker`): Two deterministic checks, no LLM:
- `I_df` (false DONE): artefact absent, channel empty, obligation was FAILED
- `I_ec` (wrong terminal type): RESPONSE or QUERY used in place of DONE, FAILURE, DECLINE for a COMMAND obligation

Both fire 100% in testing. Zone 3 is implementation-ready in two seconds.

**`EvidentialChecker`** is published in `io.casehub:casehub-qhorus` (not behind `-Pwith-llm-examples`) — it is runtime infrastructure for any consumer that wants evidential attestation checking, not just a benchmark utility. Two entry points: `check(messageType, content, BenchmarkContext)` (benchmark path, V1–V4 handlers) and `checkObligation(terminalType, CommitmentContext)` (attestation path vocabulary check).

**`StoredCommitmentAttestationPolicy` update** (#305): RESPONSE used to close a COMMAND obligation now writes FLAGGED attestation (confidence 0.3 default, configurable via `casehub.qhorus.attestation.response-confidence`).

The key finding: the normative layer does not make small models reliable. It transforms what failure looks like — from invisible fabrication to structured, queryable semantic confusion. Zone 3 catches the structured thing. Without Zone 2, Zone 3 has nothing to read. Without Zone 3, Zone 2's structured failure goes undetected.

#### Key files
- `runtime/src/main/java/io/casehub/qhorus/runtime/audit/EvidentialChecker.java`
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/StoredCommitmentAttestationPolicy.java` — updated
- `examples/agent-communication/src/test/.../NormativeBenchmarkDemoTest.java`
- `examples/agent-communication/README.md` — annotated results table

#### Design decisions
- `EvidentialChecker` in `runtime.audit` package (not `examples/`) — it is production infrastructure
- `checkObligation()` is type-based, not commitment-store-based — reads message type from channel record, deterministic
- RESPONSE-for-COMMAND produces FLAGGED attestation (not thrown exception) — infrastructure records the semantic gap without breaking the commitment lifecycle

#### Status
✅ Complete

---

## §10 Architectural Decisions

Decisions captured inline in Chapters. Cross-cutting decisions recorded here.

**ADR-0005 — 9-type speech-act taxonomy** (referenced in all chapters)
The taxonomy is theoretically grounded and complete. No obligation type is missing; every obligation has exactly one terminal resolution path. This is the foundational decision — all subsequent decisions about CommitmentService, LedgerWriteService, EvidentialChecker, and the A2A bridge derive from it.

**RESPONSE-on-COMMAND now writes FLAGGED attestation**
PP-20260623-fd69f3 and #305 formalise this: using RESPONSE vocabulary to discharge a COMMAND obligation is a normative violation (I_ec). The infrastructure no longer silently accepts it as equivalent to DONE. The commitment still transitions to FULFILLED (state machine is unchanged), but the LedgerAttestation carries FLAGGED verdict with configurable confidence (default 0.3). `EvidentialChecker.checkObligation()` provides the deterministic check for consumers who want to surface it explicitly.

**LAST_WRITE overwrites skip ledger write** (#195 — tracked)
This is an intentional asymmetry: LAST_WRITE channels are designed for EPHEMERAL state where only the latest value matters. Writing every overwrite to the ledger would fill it with transient state. The gap is documented and tracked.

---

## §11 Quality Requirements

| Quality scenario | Measure |
|---|---|
| **Commitment correctness** | Every COMMAND dispatch produces exactly one `Commitment` in OPEN state. DONE/DECLINE/FAILURE on the same `correlationId` transitions it to a terminal state. `CommitmentStoreContractTest` verifies this for both blocking and reactive stores. |
| **Audit completeness** | Every speech act that passes the enforcement gate produces a `MessageLedgerEntry`. The only exception is LAST_WRITE overwrites (#195 — tracked). `LedgerWriteService.record()` is unconditional. |
| **Tenant isolation** | All JPA store queries carry a `tenancyId` filter. `TenantIsolationTest` verifies that tenant A cannot read tenant B's channels, messages, or commitments. Background services use `@CrossTenant` stores with explicit `isCrossTenantAdmin()` assertion. |
| **CDI extensibility** | Every SPI has a `@DefaultBean` implementation. A consumer can replace any SPI with an `@Alternative @Priority(N)` bean without modifying qhorus source. Verified by consumer integration tests in claudony. |
| **Enforcement gate integrity** | No channel write bypasses `MessageService.dispatch()`. MCP tools, A2A bridge, and connector-backend all route through the same gate. `BlockingTierPurityTest` verifies that `ReactiveMessageService` injects `ObligorTrustPolicy` (SPI) rather than `TrustGateService` (internal). |

---

## §12 Risks and Technical Debt

| Item | Type | Status |
|---|---|---|
| LAST_WRITE overwrites skip ledger write | Technical debt | Tracked (#195) — intentional but creates audit gap for LAST_WRITE channels |
| Reactive integration tests `@Disabled` | Operational gap | Require Podman ≥ 4 GB with PostgreSQL DevServices — skip in CI; reactive correctness verified only locally |
| SSE subscriptions do not survive server restart | Known constraint | Lazy registration — `A2AChannelBackend` does not observe `ChannelInitialisedEvent`. Documented in ADR-0013. Gap reference: qhorus#162 |
| Cross-node fanOut gap | Known constraint | In a multi-node fleet each node has its own channel store and observer set. `InProcessMessageBus` (CDI events) does not cross JVM boundaries. CLUSTER-scoped `MessageObserver` implementations are the SPI hook but no shipped implementation exists. Documented in `docs/messaging-architecture.md` |
| `examples/agent-communication/` compile drift | Operational risk | Profile-gated module — API changes in `runtime/` do not fail CI. Run `mvn test-compile -Pwith-llm-examples` after any API rename or signature change |
| Reactive trust gate (`ReactiveMessageService`) bypassed the SPI | Fixed | #235 — custom `ObligorTrustPolicy` beans were silently ignored in reactive path; fixed May 2026 |
| RESPONSE-for-COMMAND silently fulfilled commitment | Fixed | #305 — now writes FLAGGED attestation; `EvidentialChecker` provides explicit vocabulary check |
| `@DefaultBean` principal caused `AmbiguousDependencies` | Fixed | #276 — `QhorusInboundCurrentPrincipal` changed to plain `@ApplicationScoped` |

---

## §13 Glossary

| Term | Definition |
|---|---|
| `Channel` | Named communication context per coordination unit. Declared semantic (APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE) enforced on every write. May have `allowedTypes` and `deniedTypes` constraints. |
| `ChannelSemantic` | Enum: APPEND (ordered history), COLLECT (all contributors), BARRIER (waits for declared contributors), EPHEMERAL (transient), LAST_WRITE (latest value wins). |
| `Commitment` | Machine-tracked agent obligation. Created on COMMAND/QUERY dispatch. States: OPEN → ACKNOWLEDGED → FULFILLED | DECLINED | FAILED | DELEGATED | EXPIRED. |
| `CommitmentState` | 7-state enum with `isTerminal()`. DELEGATED is terminal for the original obligor — a child OPEN commitment is created for the delegate. |
| `MessageLedgerEntry` | Immutable per-speech-act audit record — JPA JOINED subclass of `LedgerEntry`. One entry per dispatch. Telemetry fields present on EVENT only. |
| `LedgerAttestation` | Trust signal written at terminal commitment transitions: DONE → SOUND (confidence 0.7 default), FAILURE → FLAGGED (0.6), DECLINE → FLAGGED (0.4), RESPONSE-for-COMMAND → FLAGGED (0.3). |
| `COMMAND` | Speech act: formal request creating a Commitment. The obligor must respond with DONE, FAILURE, or DECLINE. |
| `QUERY` | Speech act: information request creating a Commitment. The obligor may respond with RESPONSE or DECLINE. |
| `DONE` | Speech act: successful fulfillment. Closes Commitment as FULFILLED. Writes SOUND attestation. |
| `DECLINE` | Speech act: explicit scope refusal — not an error; closes Commitment as DECLINED. Writes FLAGGED attestation. |
| `HANDOFF` | Speech act: obligation transfer to a named target. Closes parent Commitment as DELEGATED; creates child OPEN Commitment for the delegate. Requires non-null `target`. |
| `FAILURE` | Speech act: unsuccessful discharge. Closes Commitment as FAILED. Writes FLAGGED attestation. |
| `STATUS` | Speech act: progress update. Does not affect Commitment state. |
| `RESPONSE` | Speech act: acknowledgment or information. Closes Commitment as FULFILLED. Writes FLAGGED attestation when used to discharge a COMMAND obligation (I_ec violation). |
| `EVENT` | Speech act: observer-only telemetry. Content is always null — telemetry lives in dedicated `dispatch.telemetry()` field. Never returned to agents polling for messages. |
| `ObligorTrustPolicy` | SPI: determines whether an obligor is trusted before a COMMAND commitment is accepted. `permits(ObligorTrustContext)` returns boolean. `@DefaultBean`: disabled when `minObligorTrust ≤ 0`. |
| `CommitmentAttestationPolicy` | SPI: determines `LedgerAttestation` verdict and confidence for terminal commitment transitions. `@DefaultBean`: `StoredCommitmentAttestationPolicy`. |
| `InstanceActorIdProvider` | SPI: maps instanceId → ledger actorId. Default is identity. Claudony provides the session→persona mapping so trust scores accrue to a persona, not a session. |
| `EvidentialChecker` | `@DefaultBean @ApplicationScoped` in `runtime.audit`. Two entry points: `check()` (benchmark path, V1–V4 handlers) and `checkObligation()` (attestation path vocabulary check for I_ec violations). Deterministic — no LLM required. |
| `RenderableProjection` | SPI: fold-based view over a channel's message history. Registered via CDI; discovered by `ProjectionRegistry`. `project_channel` MCP tool renders any registered projection. |
| `ProjectionRegistry` | `@ApplicationScoped` bean collecting all `RenderableProjection<?>` instances. Validates uniqueness of `projectionName()` at CDI bootstrap — duplicate names throw `IllegalStateException`. |
| `ChannelInitialisedEvent` | CDI event record fired by `ChannelGateway.initChannel()` on channel creation and on startup recovery. Backends observe this to self-register. |
| `BenchmarkContext` | Context passed to `EvidentialChecker.check()` carrying channel state, artefact registry, and message history for the benchmark V1–V4 violation handlers. |
| `BenchmarkViolation` | Result type from `EvidentialChecker.check()` — I_df (false DONE: artefact absent, obligation unfulfilled) or I_ec (wrong terminal type: RESPONSE or QUERY used in place of DONE/FAILURE/DECLINE). |
| Named datasource `qhorus` | The Qhorus persistence unit. Always configured as `quarkus.datasource.qhorus.*`. All Qhorus JPA entities live here. Reactive code must use `Panache.withTransaction("qhorus", ...)`. |
| `InMemoryStore` | Test-double implementations of `*Store` interfaces in `casehub-qhorus-testing`. `@Alternative @Priority(1)` — selected over JPA stores in `@QuarkusTest`. Must not mutate `PanacheEntity` fields in-place within `Panache.withSession()` scope. |
| `*Store` seam | Services inject store interfaces, never Panache entity statics. This enables CDI-free unit tests and contract tests with two concrete runners (blocking, reactive). |
| `DEFAULT_TENANT_ID` | Sentinel tenancyId used when no `X-Tenancy-ID` header is present. Null/blank headers resolve to this value. |

---

*Completed 2026-06-23. All §3–§13 sections written from repository source material, git history, and blog entries.*
