# Design Spec — #456 ARC42STORIES.MD Migration

## Summary

Migrate qhorus from `docs/DESIGN.md` (434 lines, significantly stale) to
`ARC42STORIES.MD` at project root, matching the foundation-tier template
(`casehub-work/ARC42STORIES.MD`) with a full retrospective chapter log
covering the project's entire build history.

## Scope

- Create `ARC42STORIES.MD` at `/qhorus/ARC42STORIES.MD` (project repo root)
- Write all sections §1-§9 with content updated to current state
- Full retrospective: ~50 chapter entries at casehub-work detail level
- Delete `docs/DESIGN.md` after migration
- Update CLAUDE.md to reference ARC42STORIES.MD instead of DESIGN.md

## Out of Scope

- No code changes (docs-only)
- No Flyway migrations
- No changes to other repos
- DESIGN.md's references to external comparison docs (`agent-protocol-comparison.md`,
  `multi-agent-framework-comparison.md`) stay in place — those are standalone documents

---

## Preamble

```markdown
# CaseHub Qhorus — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** Foundation — depends on `casehub-platform-api` and `casehub-ledger`; no other casehubio dependencies
**Consumed by:** `claudony` (integration layer)
**Depends on:** `casehub-platform-api` (compile, `api/` module), `casehub-ledger` (compile, runtime + optional modules)
```

---

## §1 Introduction and Goals

**Source:** CLAUDE.md "What This Project Is" + DESIGN.md overview

Content:
- Description: governance methodology for multi-agent AI, speech act theory foundation
- Stakeholders: Quarkus app developer, consumer repos (claudony), AI agents, platform team
- Quality Goals: SLA correctness analog → commitment lifecycle correctness,
  isolation (no casehub-engine dependency), zero-datasource testing
- Artifact Schema: inherited from profile (issues, ADRs, garden entries, protocols, specs)

---

## §2 Constraints

**Source:** CLAUDE.md "Build and Test" + DESIGN.md "Technology Stack"

Content:
- Platform: Java 21 (language level on Java 26 JVM), Quarkus 3.32.2, GraalVM 25, quarkus-mcp-server 1.11.1
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Architectural: named 'qhorus' datasource, Flyway at `db/qhorus/migration/`,
  casehub-ledger dependency, `casehub.qhorus` config prefix
- Testing: H2 MODE=PostgreSQL, `@TestTransaction`, named PU schema generation

---

## §3 Context and Scope

**Source:** CLAUDE.md "Ecosystem Context" diagram + DESIGN.md references

Content:
- C4 System Context diagram showing qhorus, claudony, casehub-engine, casehub-ledger,
  casehub-platform-api, external A2A agents
- Boundary Rules: what qhorus explicitly does NOT do (orchestrate cases, provision agents,
  implement trust scoring — those belong to engine, claudony, ledger respectively)
- Platform references per foundation-tier profile

---

## §4 Solution Strategy — Layer Taxonomy

**Source:** New — derived from module structure in CLAUDE.md

Foundation modules define their own layer taxonomy. Qhorus's layers represent
the internal architectural concerns added incrementally across ~50 build chapters:

| Layer | Concern |
|---|---|
| L1 Core Domain | Channel, Message, Instance, SharedData entities; CRUD services; store interfaces; 5 channel semantics (APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE) |
| L2 Speech Act Protocol | 10-type message taxonomy (ADR-0005); commitment lifecycle (7 states: OPEN→ACKNOWLEDGED→FULFILLED/DECLINED/FAILED/DELEGATED/EXPIRED); correlation integrity; dispatch pipeline; ObligorTrustPolicy |
| L3 Normative Audit | MessageLedgerEntry (all 10 types); CommitmentAttestationPolicy SPI; EvidentialChecker (Zone 1-3); peer attestation; AttestorCredibilityPolicy; CausalGraphService |
| L4 Channel Features | Topics (create, merge, move); reactions (idempotent); spaces (recursive nesting); membership (join/leave, unread counts); presence (Caffeine-based heartbeat); channel summaries (SummaryUpdateHook SPI); projections (left-fold SPI); delivery tracking (per-participant) |
| L5 Enforcement & Safety | Watchdog framework (8 condition types); ChannelProtocol SPI (4 built-in protocols); enforcement modes (ADVISORY/BLOCKING/QUARANTINE); DispatchAdvisory pipeline; containment actions (PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE); channel policy overrides |
| L6 Transport Layer | ChannelGateway (backend-agnostic fan-out); backend impls (qhorus-internal, connector, Slack, A2A outbound, push notification); MessageObserver SPI (LOCAL/CLUSTER scope); observer impls (in-process CDI, Kafka, WebSocket, Webhook); DeliveryService (AT_LEAST_ONCE pump); postgres broadcaster (LISTEN/NOTIFY) |
| L7 API Surface | MCP tools (QhorusTestHelper — blocking); REST resources (channels, A2A JSON-RPC, agent cards, causal graphs, compliance); per-agent cards at `/.well-known/agents/{instanceId}.json` |
| L8 Compliance & Enterprise | Compliance report module (EU AI Act, 8 report types, PDF/A-2b + digital signatures); agent card signing (JWS Ed25519, JWKS endpoint); notification bridge (platform subscription engine); routing bridge (reputation-aware, role: targets); capacity signals |

### Chapter Sequencing Rationale

Key hard dependencies — order is non-negotiable:

- Core Domain (L1) before Speech Act Protocol (L2): commitment lifecycle requires channel + message entities
- Speech Act Protocol (L2) before Normative Audit (L3): ledger entries record commitment state transitions
- Core Domain (L1) before Transport Layer (L6): gateway fans out persisted messages
- Speech Act Protocol (L2) before Enforcement & Safety (L5): protocol enforcement evaluates message types and commitment state
- Normative Audit (L3) before Compliance & Enterprise (L8): compliance reports query ledger entries

---

## §5 Building Block View

**Source:** CLAUDE.md "Project Structure" (authoritative 28-module layout)

Content:
- C4 Container diagram showing all module boundaries and layer membership
- Module Index table (same format as casehub-work): folder, artifact, type, purpose
- 28 modules: api/, runtime/, deployment/, connector-backend/, slack-channel/,
  persistence-memory/, testing/, compliance-report/, notification-bridge/,
  a2a-outbound/, a2a-push-notification/, kafka-observer/, websocket-observer/,
  webhook-observer/, agent-card-signing/, postgres-broadcaster/,
  examples/type-system/, examples/normative-layout/, examples/agent-communication/
- API Interface Taxonomy (4 categories): store, spi, gateway, service facades
- Key domain model tables (Channel, Message, Instance, SharedData, Commitment,
  Watchdog, Space, Topic, Reaction, ChannelMembership, ChannelSummary)

---

## §6 Runtime View

**Source:** CLAUDE.md testing conventions + DESIGN.md services section

Content — key scenarios:
1. Message dispatch pipeline (MessageService.dispatch enforcement gate):
   paused check → AllowedWritersPolicy → RateLimiter → RoutingBridge →
   ObligorTrustPolicy → MessageTypePolicy → CorrelationIntegrityChecker →
   ProtocolRegistry → EnforcementGate → LAST_WRITE → persist → fanOut →
   LedgerWriteService.record → MessageObserverDispatcher
2. Commitment lifecycle (COMMAND → OPEN → DONE → FULFILLED + attestation)
3. A2A task streaming (POST /a2a message/send → SSE stream → task state mapping)
4. Watchdog evaluation cycle (scheduled → evaluateAll → condition checks → containment)
5. Channel gateway fan-out (persist → fanOut → per-backend delivery → delivery pump)

---

## §7 Deployment View

**Source:** DESIGN.md deployment section (adapted) + CLAUDE.md module structure

Content:
- C4 Deployment diagram (Quarkus nodes + PostgreSQL + optional Kafka)
- Deployment Variants table:
  - Minimal: casehub-qhorus + H2
  - Standard: + PostgreSQL
  - Full audit: + casehub-ledger
  - Multi-node: + postgres-broadcaster
  - Full platform: + all optional modules

---

## §8 Crosscutting Concepts

**Source:** CLAUDE.md testing conventions + DESIGN.md key invariants

Content:
- Convention References table (Flyway scoping, CDI displacement, SPI placement,
  persistence backend priority — referencing protocols by name)
- Anti-patterns section (inline, per profile requirement):
  1. `@TestTransaction` + `afterCompletion` → MessageObserver silently skipped
  2. InMemory store entity mutation in Panache session → phantom dirty flushes
  3. `LedgerWriteService.record()` REQUIRES_NEW → stale commitment state visibility
  4. Missing `import-qhorus-test.sql` → ledger_subject_sequence table absent

---

## §9 Journeys and Chapters

### §9.1 Journey Overview

| Journey | Description | Chapters | Status |
|---|---|---|---|
| J1 — Core Communication Mesh | Foundational mesh: entities, MCP tools, semantics, correlation, artefacts, addressing, agent card, A2A, HITL, ACL, ledger, persistence, reactive | C1–C14 | ✅ Complete |
| J2 — Normative Governance | Commitment lifecycle, speech act taxonomy, normative audit, attestation, evidential checking, multi-tenancy | C15–C20 | ✅ Complete |
| J3 — Channel Gateway & Transport | Backend-agnostic fan-out, connector/Slack backends, observer SPI, Kafka/WS/Webhook, delivery service, postgres broadcaster | C21–C27 | ✅ Complete |
| J4 — Channel Intelligence | Topics, reactions, artefact refs, membership, spaces, presence, summaries, projections, delivery tracking | C28–C37 | ✅ Complete |
| J5 — Operational Safety | Watchdog framework, coordination pathology conditions, context pressure, protocol enforcement, enforcement modes, containment | C38–C44 | ✅ Complete |
| J6 — Trust, Compliance & Routing | Peer attestation, credibility, causal graphs, PROPOSE type, routing bridge, compliance reports, agent card signing, A2A outbound, push notifications, notification bridge, channel policy | C45–C55 | ✅ Complete |

### §9.2 Chapter Index

Full index table with columns: #, Chapter, Journey, Key issues, Status

### §9.3+ Journey Details

Each journey section contains:
- Journey overview (1 paragraph)
- Per-chapter entries at casehub-work detail level:
  - **What this delivers** (before/after narrative, ~200 words)
  - **Accountability gaps closed** (specific gaps, or "None")
  - **Layer Impact** table (H/M/L per layer)

### Chapter Research Strategy

Each chapter entry requires:
1. **CLAUDE.md** — primary feature documentation (richest technical detail)
2. **Git log** — `git log --oneline --all --grep="<issue-number>"` for dates, commit messages
3. **GitHub issues** — `gh issue view <N>` for descriptions, labels, cross-references

### Proposed Chapter Breakdown

**J1 — Core Communication Mesh (C1-C14)** — maps directly to DESIGN.md phases 1-15:

| # | Chapter | Key Issues | Layers |
|---|---|---|---|
| C1 | Core Data Model + Services | Phase 1 | L1 |
| C2 | MCP Tool Surface | Phase 2, #8-#11 | L7 |
| C3 | Channel Semantics | Phase 3 | L1 |
| C4 | Correlation + wait_for_reply | Phase 4 | L2 |
| C5 | Artefacts | Phase 5 | L1 |
| C6 | Addressing | Phase 6 | L1, L2 |
| C7 | Agent Card | Phase 7 | L7 |
| C8 | A2A Compatibility | Phase 9 | L7 |
| C9 | Human-in-the-Loop Controls | Phase 10 | L1, L5 |
| C10 | Access Control + Governance | Phase 11 | L1 |
| C11 | Structured Observability | Phase 12 | L3 |
| C12 | Persistence Abstraction | Phase 13, ADR-0002 | L1 |
| C13 | Reactive Dual-Stack | Phase 14, #74-#80, ADR-0003 | L1 |
| C14 | Documentation | Phase 15, #81 | cross-cutting |

**J2 — Normative Governance (C15-C20):**

| # | Chapter | Key Issues | Layers |
|---|---|---|---|
| C15 | Commitment Lifecycle | #121 | L2 |
| C16 | Speech Act Taxonomy | ADR-0005 | L2 |
| C17 | Normative Audit Ledger | #123, #179 | L3 |
| C18 | Commitment Attestation SPI | #123, #304 | L3 |
| C19 | Evidential Checker | #303 | L3 |
| C20 | Multi-Tenancy | #260, #265 | cross-cutting |

**J3 — Channel Gateway & Transport (C21-C27):**

| # | Chapter | Key Issues | Layers |
|---|---|---|---|
| C21 | Channel Gateway | #135 | L6 |
| C22 | Connector Backend | #216 | L6 |
| C23 | Slack Channel Backend | #261 | L6 |
| C24 | MessageObserver SPI | #163 | L6 |
| C25 | Transport Observers | #279, #294 | L6 |
| C26 | Delivery Service | #380 | L6 |
| C27 | Postgres Broadcaster | — | L6 |

**J4 — Channel Intelligence (C28-C37):**

| # | Chapter | Key Issues | Layers |
|---|---|---|---|
| C28 | Topics | #329, #335, #336 | L4 |
| C29 | Reactions | #330 | L4 |
| C30 | Artefact References | #331 | L1, L4 |
| C31 | Channel Membership | #332, #379 | L4 |
| C32 | Spaces | #334 | L4 |
| C33 | Presence | #333 | L4 |
| C34 | Channel Summaries | #355 | L4 |
| C35 | Channel Projections | #232 | L4 |
| C36 | Delivery Tracking | #376, #380 | L4, L6 |
| C37 | Chunked Artefact Upload | — | L1 |

**J5 — Operational Safety (C38-C44):**

| # | Chapter | Key Issues | Layers |
|---|---|---|---|
| C38 | Watchdog Framework | #121 (initial) | L5 |
| C39 | Coordination Pathology Watchdogs | #354 | L5 |
| C40 | Context Pressure + Delivery Lag | #363, #381 | L5 |
| C41 | Circular Delegation Watchdog | #368 | L5 |
| C42 | Protocol Enforcement SPI | #357 | L5 |
| C43 | Enforcement Modes | #400 | L5 |
| C44 | Watchdog Containment Actions | #399 | L5 |

**J6 — Trust, Compliance & Routing (C45-C55):**

| # | Chapter | Key Issues | Layers |
|---|---|---|---|
| C45 | Peer Attestation | #356 | L3 |
| C46 | Attestor Credibility | #371 | L3 |
| C47 | Causal Graph Service | #398 | L3 |
| C48 | PROPOSE Message Type | #395 | L2 |
| C49 | Reputation-Aware Routing | #401 | L8 |
| C50 | Compliance Reports | #411, #414, #417, #418 | L8 |
| C51 | Agent Card Signing | #403 | L8 |
| C52 | A2A Outbound + Push Notifications | #396, #406 | L6, L8 |
| C53 | Notification Bridge | #375 | L8 |
| C54 | Capacity Signals | #434 | L8 |
| C55 | Channel Policy Overrides | #455 | L5 |

---

## Implementation Strategy

### Phase 1 — §1-§8 (structural sections)

Write directly from CLAUDE.md + DESIGN.md content. These sections are
straightforward content migration and update — no historical research needed.

Estimated output: ~400 lines

### Phase 2 — §9 (retrospective chapters)

Research and write journey-by-journey using parallel subagents:
- Each agent receives one journey's chapter list with issue numbers
- Agent researches: CLAUDE.md feature docs, git log dates/commits, GitHub issue descriptions
- Agent drafts chapter entries in casehub-work format
- Assembly: merge all journey outputs, review for consistency, finalize

Estimated output: ~2000-3000 lines (55 chapters × 40-60 lines each)

### Phase 3 — Cleanup

- Delete `docs/DESIGN.md`
- Update CLAUDE.md references
- Final consistency review

---

## Layer × Chapter Matrix

To be populated during implementation — same format as casehub-work
(H/M/L/— per layer per chapter).

---

## References

- `docs/DESIGN.md` — existing stale design document (being replaced)
- `casehub-work/ARC42STORIES.MD` — foundation-tier template
- `../parent/docs/arc42stories-casehub-profile.md` — CaseHub ARC42Stories profile
- `CLAUDE.md` — authoritative project conventions and feature documentation
- `docs/adr/` — architecture decision records
- `docs/specs/2026-04-13-qhorus-design.md` — original design specification
