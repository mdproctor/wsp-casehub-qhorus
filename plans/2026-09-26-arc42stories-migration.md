# ARC42STORIES.MD Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax
> for tracking.

**Focal issue:** #456 — docs: migrate DESIGN.md to ARC42STORIES.MD format
**Issue group:** #456

**Goal:** Create a comprehensive ARC42STORIES.MD at the project root with full retrospective chapter entries covering qhorus's entire build history, matching the casehub-work foundation-tier template quality.

**Architecture:** Three-phase approach — write structural sections (§1-§8) first from existing CLAUDE.md/DESIGN.md content, then research and write ~55 retrospective chapter entries for §9 using parallel subagents per journey, then assemble and clean up. The document follows the casehub ARC42Stories profile for foundation-tier modules.

**Tech Stack:** Markdown, Mermaid diagrams, GitHub CLI for issue research, git log for timeline data

## Global Constraints

- Docs-only — no code changes, no Flyway migrations, no test changes
- All content writes go to the project repo (`/Users/mdproctor/claude/casehub/slots/205/qhorus`)
- Follow casehub-work/ARC42STORIES.MD template format exactly
- Follow arc42stories-casehub-profile.md for foundation-tier conventions
- Commit after every substantial write (wip: prefix)
- Config prefix is `casehub.qhorus` (not `quarkus.qhorus` — DESIGN.md had this wrong)

---

## Batch 1: Structural sections (§1-§8)

### Task 1: Write ARC42STORIES.MD preamble through §8

Write the complete ARC42STORIES.MD file with all structural sections. This is a single large write because §1-§8 are all derived from existing content (CLAUDE.md + DESIGN.md) with no historical research needed. The chapter sections (§9) will be added in subsequent batches.

**Files:**
- Create: `ARC42STORIES.MD` (project repo root)

**Interfaces:**
- Consumes: `CLAUDE.md` (project conventions, module structure, ecosystem context), `docs/DESIGN.md` (technology stack, domain model, services), `casehub-work/ARC42STORIES.MD` (template), `../parent/docs/arc42stories-casehub-profile.md` (profile)
- Produces: ARC42STORIES.MD with §1-§8 complete, §9 placeholder structure ready for chapter injection

- [ ] **Step 1: Read source material**

Read the following files for content extraction:
- `CLAUDE.md` — "What This Project Is", "Project Structure", "Build and Test", "Ecosystem Context", "CaseHub Naming", testing conventions
- `docs/DESIGN.md` — Technology Stack table, Domain Model tables, Services section, key invariants, Configuration section
- `casehub-work/ARC42STORIES.MD` — template structure for §1-§8
- `../parent/docs/arc42stories-casehub-profile.md` — foundation-tier preamble, default conventions, anti-pattern requirements

- [ ] **Step 2: Write ARC42STORIES.MD with §1-§8**

Create the file at the project repo root with all structural sections. Content mapping:

**Preamble:** Foundation-tier template from spec (Build position, Consumed by, Depends on).

**§1 Introduction and Goals:**
- Description: from CLAUDE.md "What This Project Is" — governance methodology, speech act theory, deontic logic, defeasible reasoning, social commitment semantics. Not middleware.
- Stakeholders table: Quarkus app developer, consumer repo (claudony), AI agents, platform team, A2A ecosystem (5 rows)
- Quality Goals table: (1) Commitment lifecycle correctness, (2) Isolation — no casehub-engine dependency, (3) Zero-datasource unit testing via InMemoryStores
- Artifact Schema: inherited from profile — issues, ADRs, garden entries, protocols, specs

**§2 Constraints:**
- Platform table: Java 21 on JVM 26, Quarkus 3.32.2, GraalVM 25, quarkus-mcp-server 1.11.1
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Architectural constraints: named 'qhorus' datasource, Flyway at `db/qhorus/migration/`, casehub-ledger dependency, `casehub.qhorus` config prefix, H2 MODE=PostgreSQL for tests

**§3 Context and Scope:**
- C4 System Context Mermaid diagram showing: qhorus (central), claudony (consumer), casehub-engine (indirect consumer via claudony), casehub-ledger (dependency — LedgerEntry base class), casehub-platform-api (dependency — ActorType, CurrentPrincipal), external A2A agents (via /.well-known/agent.json)
- Boundary Rules: what qhorus does NOT do — orchestrate cases (engine), provision agents (claudony), implement trust scoring (ledger), manage human tasks (casehub-work)
- Platform references: dependency-map, repos/qhorus.md

**§4 Solution Strategy:**
- Layer Taxonomy table (L1-L8 from spec)
- Chapter Sequencing Rationale (hard dependencies from spec)
- Mermaid flowchart showing journey dependencies

**§5 Building Block View:**
- C4 Container Mermaid diagram showing all modules grouped by layer
- Module Index table (28 rows): folder, artifact, type, purpose — content from CLAUDE.md "Project Structure"
- API Interface Taxonomy: 4 categories from CLAUDE.md (store, spi, gateway, service facades)
- Domain Model section: key entity tables (Channel, Message, Commitment, Instance, SharedData, Watchdog, Space, Topic, Reaction, ChannelMembership, ChannelSummary) — updated from DESIGN.md Domain Model with current fields

**§6 Runtime View:**
5 scenarios from spec:
1. Message dispatch pipeline — full enforcement gate sequence from CLAUDE.md MessageService.dispatch documentation
2. Commitment lifecycle — COMMAND → OPEN → DONE → FULFILLED with attestation write
3. A2A task streaming — POST /a2a message/send → SSE → task state mapping
4. Watchdog evaluation cycle — scheduled → evaluateAll → condition checks → containment
5. Channel gateway fan-out — persist → fanOut → per-backend delivery → pump

**§7 Deployment View:**
- C4 Deployment Mermaid diagram (Quarkus nodes + PostgreSQL + optional Kafka/WebSocket)
- Deployment Variants table (6 variants from spec)

**§8 Crosscutting Concepts:**
- Convention References table referencing protocols by name (from profile defaults + qhorus-specific protocols in `docs/protocols/casehub/`)
- Anti-patterns section (inline per profile requirement, Symptom → Cause → Fix format):
  1. `@TestTransaction` + `afterCompletion(STATUS_COMMITTED)` → observer silently skipped (GE-20260608-038af4)
  2. InMemory store mutation in Panache session → phantom dirty flush (PP-20260618-100368)
  3. `LedgerWriteService.record()` REQUIRES_NEW → stale commitment state in outer tx
  4. Missing `import-qhorus-test.sql` → `ledger_subject_sequence` table absent in test modules

**§9 placeholder:** Write the §9.1 Journey Overview table and §9.2 Chapter Index table headers. Leave chapter entries empty — Batch 2 fills them.

- [ ] **Step 3: Verify document structure**

Check:
- Preamble has all required foundation-tier fields
- All 9 section numbers present (§1-§9)
- Mermaid diagram syntax is valid
- §8 anti-patterns are inline (not just references, per profile)
- §9 has journey overview table and empty chapter index table

- [ ] **Step 4: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): create ARC42STORIES.MD — §1-§8 structural sections

Migrates content from docs/DESIGN.md to foundation-tier ARC42STORIES format.
§9 chapter entries to follow.

Refs #456"
```

---

## Batch 2: §9 retrospective chapters (research + write)

### Task 2: Research and write Journey 1 chapters (C1-C14)

Research the 14 chapters of Journey 1 — Core Communication Mesh. This journey covers the original 15 DESIGN.md phases. DESIGN.md's "Build Roadmap" section has phase summaries; supplement with CLAUDE.md feature documentation and git log for dates.

**Files:**
- Modify: `ARC42STORIES.MD` (append §9.3 journey section)

**Interfaces:**
- Consumes: DESIGN.md "Build Roadmap" (phase summaries, test counts), CLAUDE.md (channel semantics, MCP tools, persistence abstraction, reactive stack), git log (dates, issue numbers)
- Produces: §9.3 with 14 chapter entries in casehub-work format

- [ ] **Step 1: Research dates and issue numbers**

For each chapter's key issues, run:
```bash
# Phase dates from early git history
git -C "$PROJECT" log --oneline --reverse --format="%as %s" | grep -i "phase [0-9]" | head -20

# ADR dates
git -C "$PROJECT" log --oneline --all --format="%as %s" -- "docs/adr/" | head -10

# Issue details for specific issues
gh issue view 74 --repo casehubio/qhorus --json title,createdAt --jq '.title + " | " + .createdAt' 2>/dev/null
gh issue view 81 --repo casehubio/qhorus --json title,createdAt --jq '.title + " | " + .createdAt' 2>/dev/null
```

- [ ] **Step 2: Write §9.3 — Journey 1 chapters**

Write the journey overview paragraph and all 14 chapter entries. Each entry follows this template exactly:

```markdown
### Chapter C<N> — <Title>

**Journey:** J1 — Core Communication Mesh | **Sequence:** <N> of 14 | **Status:** ✅
**Delivered:** <YYYY-MM-DD> | **Issues:** <issue refs>

**What this delivers**
Before this chapter, <what was missing>. After: <what exists now and what it enables>.

**Accountability gaps closed**
- <specific gap> → <how it's addressed> (or "None")

**Layer Impact**
| Layer | Delta |
|---|---|
| <layer> | <H/M/L> |
```

Content sources per chapter:
- C1 (Core Data Model): DESIGN.md Phase 1, CLAUDE.md entity descriptions
- C2 (MCP Tool Surface): DESIGN.md Phase 2, CLAUDE.md MCP tool section (now ~50 tools, was 14)
- C3 (Channel Semantics): DESIGN.md Phase 3, CLAUDE.md ChannelSemantic enum
- C4 (Correlation): DESIGN.md Phase 4 (PendingReply — since replaced by Commitment)
- C5 (Artefacts): DESIGN.md Phase 5, CLAUDE.md SharedData/ArtefactClaim
- C6 (Addressing): DESIGN.md Phase 6, CLAUDE.md target field (3 modes)
- C7 (Agent Card): DESIGN.md Phase 7, CLAUDE.md AgentCardResource
- C8 (A2A): DESIGN.md Phase 9, CLAUDE.md A2AResource
- C9 (HITL): DESIGN.md Phase 10, CLAUDE.md pause/resume/watchdog
- C10 (ACL): DESIGN.md Phase 11, CLAUDE.md allowed_writers/admin_instances/RateLimiter
- C11 (Observability): DESIGN.md Phase 12, CLAUDE.md MessageLedgerEntry
- C12 (Persistence): DESIGN.md Phase 13, CLAUDE.md store pattern, ADR-0002
- C13 (Reactive): DESIGN.md Phase 14, CLAUDE.md reactive stores/services, ADR-0003
- C14 (Documentation): DESIGN.md Phase 15

- [ ] **Step 3: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): add §9.3 Journey 1 chapters (C1-C14)

Core Communication Mesh — 14 chapters covering the foundational mesh from
entities through reactive dual-stack.

Refs #456"
```

### Task 3: Research and write Journey 2 chapters (C15-C20)

Research the 6 chapters of Journey 2 — Normative Governance. This journey covers the commitment lifecycle, speech act taxonomy, normative audit, and multi-tenancy.

**Files:**
- Modify: `ARC42STORIES.MD` (append §9.4 journey section)

**Interfaces:**
- Consumes: CLAUDE.md (commitment lifecycle, MessageType, ledger, attestation, tenancy), GitHub issues (#121, #123, #179, #260, #265, #303, #304, ADR-0005), git log (dates)
- Produces: §9.4 with 6 chapter entries

- [ ] **Step 1: Research issues and dates**

```bash
for N in 121 123 179 260 265 303 304; do
  gh issue view $N --repo casehubio/qhorus --json title,createdAt,closedAt --jq "\"#$N: \" + .title + \" | created: \" + .createdAt + \" | closed: \" + (.closedAt // \"open\")" 2>/dev/null
done
```

Also check ADR-0005 date:
```bash
git -C "$PROJECT" log --oneline --format="%as %s" -- "docs/adr/0005*" | head -3
```

- [ ] **Step 2: Write §9.4 — Journey 2 chapters**

6 chapters. Content sources per chapter:
- C15 (Commitment Lifecycle): CLAUDE.md Commitment entity, CommitmentService state machine, CommitmentState enum (7 states)
- C16 (Speech Act Taxonomy): CLAUDE.md MessageType (10 types), ADR-0005
- C17 (Normative Audit Ledger): CLAUDE.md MessageLedgerEntry, LedgerWriteService.record() for all 10 types, QhorusLedgerEntryRepository
- C18 (Commitment Attestation SPI): CLAUDE.md CommitmentAttestationPolicy, StoredCommitmentAttestationPolicy, CommitmentContext
- C19 (Evidential Checker): CLAUDE.md EvidentialChecker, BenchmarkContext, BenchmarkViolation, Zone 1-3
- C20 (Multi-Tenancy): CLAUDE.md CurrentPrincipal, InboundTenancyContext, TenancyContextFilter, X-Tenancy-ID

- [ ] **Step 3: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): add §9.4 Journey 2 chapters (C15-C20)

Normative Governance — 6 chapters covering commitment lifecycle through
multi-tenancy.

Refs #456"
```

### Task 4: Research and write Journey 3 chapters (C21-C27)

Research the 7 chapters of Journey 3 — Channel Gateway & Transport.

**Files:**
- Modify: `ARC42STORIES.MD` (append §9.5 journey section)

**Interfaces:**
- Consumes: CLAUDE.md (ChannelGateway, backends, MessageObserver SPI, delivery service, observers), GitHub issues (#135, #163, #216, #261, #279, #294, #380), git log (dates)
- Produces: §9.5 with 7 chapter entries

- [ ] **Step 1: Research issues and dates**

```bash
for N in 135 163 216 261 279 294 380; do
  gh issue view $N --repo casehubio/qhorus --json title,createdAt,closedAt --jq "\"#$N: \" + .title + \" | created: \" + .createdAt + \" | closed: \" + (.closedAt // \"open\")" 2>/dev/null
done
```

- [ ] **Step 2: Write §9.5 — Journey 3 chapters**

7 chapters. Content sources:
- C21 (Channel Gateway): CLAUDE.md ChannelGateway, AgentChannelBackend, HumanParticipatingChannelBackend, HumanObserverChannelBackend
- C22 (Connector Backend): CLAUDE.md connector-backend module, ConnectorChannelBackend, ConnectorNormaliser SPI
- C23 (Slack Channel Backend): CLAUDE.md slack-channel module, SlackChannelBackend, thread cache
- C24 (MessageObserver SPI): CLAUDE.md MessageObserver (Scope LOCAL/CLUSTER), MessageReceivedEvent, MessageObserverDispatcher
- C25 (Transport Observers): CLAUDE.md kafka-observer, websocket-observer, webhook-observer modules, CloudEventMapper
- C26 (Delivery Service): CLAUDE.md DeliveryService, DeliveryBatchExecutor, DeliveryCursor, AT_LEAST_ONCE
- C27 (Postgres Broadcaster): CLAUDE.md postgres-broadcaster module

- [ ] **Step 3: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): add §9.5 Journey 3 chapters (C21-C27)

Channel Gateway & Transport — 7 chapters covering backend-agnostic fan-out
through postgres broadcaster.

Refs #456"
```

### Task 5: Research and write Journey 4 chapters (C28-C37)

Research the 10 chapters of Journey 4 — Channel Intelligence.

**Files:**
- Modify: `ARC42STORIES.MD` (append §9.6 journey section)

**Interfaces:**
- Consumes: CLAUDE.md (topics, reactions, artefact refs, membership, spaces, presence, summaries, projections, delivery tracking), GitHub issues (#232, #329-#336, #355, #376, #379, #380), git log (dates)
- Produces: §9.6 with 10 chapter entries

- [ ] **Step 1: Research issues and dates**

```bash
for N in 232 329 330 331 332 333 334 335 336 355 376 379 380; do
  gh issue view $N --repo casehubio/qhorus --json title,createdAt,closedAt --jq "\"#$N: \" + .title + \" | created: \" + .createdAt + \" | closed: \" + (.closedAt // \"open\")" 2>/dev/null
done
```

- [ ] **Step 2: Write §9.6 — Journey 4 chapters**

10 chapters. Content sources:
- C28 (Topics): CLAUDE.md TopicEntity, TopicService, merge, move
- C29 (Reactions): CLAUDE.md ReactionEntity, ReactionService, idempotent react/unreact
- C30 (Artefact References): CLAUDE.md ArtefactRef, ArtefactType, SelectionScope, ArtefactRefListConverter
- C31 (Channel Membership): CLAUDE.md ChannelMembership, MemberRole, UnreadCount, auto-join
- C32 (Spaces): CLAUDE.md Space, SpaceService, recursive nesting, MAX_DEPTH=10
- C33 (Presence): CLAUDE.md PresenceService, PresenceStatus, Caffeine cache, heartbeat
- C34 (Channel Summaries): CLAUDE.md ChannelSummary, SummaryUpdateHook SPI, ChannelSummaryScheduler
- C35 (Channel Projections): CLAUDE.md ChannelProjection, ProjectionResult, ProjectionService, RenderableProjection
- C36 (Delivery Tracking): CLAUDE.md track_delivery, lastDeliveredMessageId, per-participant retry
- C37 (Chunked Artefact Upload): CLAUDE.md begin_artefact/append_chunk/finalize_artefact

- [ ] **Step 3: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): add §9.6 Journey 4 chapters (C28-C37)

Channel Intelligence — 10 chapters covering topics through chunked uploads.

Refs #456"
```

### Task 6: Research and write Journey 5 chapters (C38-C44)

Research the 7 chapters of Journey 5 — Operational Safety.

**Files:**
- Modify: `ARC42STORIES.MD` (append §9.7 journey section)

**Interfaces:**
- Consumes: CLAUDE.md (watchdog framework, conditions, protocol enforcement, enforcement modes, containment), GitHub issues (#121, #354, #357, #363, #368, #381, #399, #400), git log (dates)
- Produces: §9.7 with 7 chapter entries

- [ ] **Step 1: Research issues and dates**

```bash
for N in 354 357 363 368 381 399 400; do
  gh issue view $N --repo casehubio/qhorus --json title,createdAt,closedAt --jq "\"#$N: \" + .title + \" | created: \" + .createdAt + \" | closed: \" + (.closedAt // \"open\")" 2>/dev/null
done
```

- [ ] **Step 2: Write §9.7 — Journey 5 chapters**

7 chapters. Content sources:
- C38 (Watchdog Framework): CLAUDE.md Watchdog entity, WatchdogEvaluationService, WatchdogScheduler, WatchdogConditionType
- C39 (Coordination Pathology): CLAUDE.md LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER, JaccardSimilarity
- C40 (Context Pressure + Delivery Lag): CLAUDE.md CONTEXT_PRESSURE, DELIVERY_LAG conditions
- C41 (Circular Delegation): CLAUDE.md CIRCULAR_DELEGATION, chain detection
- C42 (Protocol Enforcement SPI): CLAUDE.md ChannelProtocol, ProtocolRegistry, REQUEST_RESPONSE, TASK_COMPLETION, ROUND_ROBIN, CONTRIBUTION_REQUIRED
- C43 (Enforcement Modes): CLAUDE.md ADVISORY/BLOCKING/QUARANTINE, EnforcementExecutor, EnforcementBlockedException, DispatchAdvisory, Severity, SuggestedAction, ProtocolEvaluationEvent
- C44 (Containment Actions): CLAUDE.md WatchdogAction (ALERT, PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE), executeContainmentAction

- [ ] **Step 3: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): add §9.7 Journey 5 chapters (C38-C44)

Operational Safety — 7 chapters covering watchdog framework through
containment actions.

Refs #456"
```

### Task 7: Research and write Journey 6 chapters (C45-C55)

Research the 11 chapters of Journey 6 — Trust, Compliance & Routing.

**Files:**
- Modify: `ARC42STORIES.MD` (append §9.8 journey section)

**Interfaces:**
- Consumes: CLAUDE.md (peer attestation, credibility, causal graphs, PROPOSE, routing, compliance, signing, A2A outbound, push, notification bridge, policy overrides), GitHub issues (#356, #371, #375, #395, #396, #398, #401, #403, #406, #411, #414, #417, #418, #434, #455), git log (dates)
- Produces: §9.8 with 11 chapter entries

- [ ] **Step 1: Research issues and dates**

```bash
for N in 356 371 375 395 396 398 401 403 406 411 414 417 418 434 455; do
  gh issue view $N --repo casehubio/qhorus --json title,createdAt,closedAt --jq "\"#$N: \" + .title + \" | created: \" + .createdAt + \" | closed: \" + (.closedAt // \"open\")" 2>/dev/null
done
```

- [ ] **Step 2: Write §9.8 — Journey 6 chapters**

11 chapters. Content sources:
- C45 (Peer Attestation): CLAUDE.md PeerAttestationWriter, ReviewerResolver, PeerReviewAutoTrigger, PeerReviewResponseHandler
- C46 (Attestor Credibility): CLAUDE.md AttestorCredibilityPolicy, AgreementCredibilityPolicy, CollusionAwareCredibilityPolicy
- C47 (Causal Graph Service): CLAUDE.md CausalGraphService, GraphNode, GraphEdge, CausalGraph
- C48 (PROPOSE Message Type): CLAUDE.md MessageType.PROPOSE, commissive speech act, 10th type
- C49 (Reputation-Aware Routing): CLAUDE.md RoutingBridge, role: targets, AgentRegistry/AgentSelector, trust thresholds
- C50 (Compliance Reports): CLAUDE.md compliance-report module, 8 report types, PDF/A-2b, digital signatures, property verification
- C51 (Agent Card Signing): CLAUDE.md agent-card-signing module, JWS Ed25519, JcsCanonicalizer, JWKS
- C52 (A2A Outbound + Push): CLAUDE.md a2a-outbound module, a2a-push-notification module, PushNotificationBackend
- C53 (Notification Bridge): CLAUDE.md notification-bridge module, QhorusObligationEvent, SubscribableEvent
- C54 (Capacity Signals): CLAUDE.md CommitmentCountCapacitySource, redistribution/routing thresholds
- C55 (Channel Policy Overrides): CLAUDE.md Channel.policyOverrides, merge semantics, V55

- [ ] **Step 3: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): add §9.8 Journey 6 chapters (C45-C55)

Trust, Compliance & Routing — 11 chapters covering peer attestation through
channel policy overrides.

Refs #456"
```

---

## Batch 3: Assembly and index

### Task 8: Complete §9.2 Chapter Index and Layer × Chapter matrix

After all journey sections are written, back-fill the §9.2 Chapter Index table and add the Layer × Chapter matrix.

**Files:**
- Modify: `ARC42STORIES.MD` (update §9.2 chapter index, add matrix after §9.1)

**Interfaces:**
- Consumes: All chapter entries from Tasks 2-7
- Produces: Complete §9.2 table (55 rows) + Layer × Chapter matrix

- [ ] **Step 1: Build the Chapter Index table**

Populate the §9.2 table from all written chapters. Format:

```markdown
| # | Chapter | Journey | Key issues | Status |
|---|---|---|---|---|
| C1 | Core Data Model + Services | Core Communication Mesh | Phase 1 | ✅ |
| ... | ... | ... | ... | ... |
| C55 | Channel Policy Overrides | Trust, Compliance & Routing | #455 | ✅ |
```

- [ ] **Step 2: Build the Layer × Chapter matrix**

Create the cross-reference matrix showing H/M/L impact per layer per chapter. Use the Layer Impact tables from each chapter entry as the source. Split across multiple tables if needed (same format as casehub-work — one table per journey segment).

- [ ] **Step 3: Add the Mermaid journey flow diagram**

Create a Mermaid flowchart showing chapter sequencing across journeys (same style as casehub-work — C1→C2→...→C55 with journey grouping and green fill for completed).

- [ ] **Step 4: Commit**

```bash
git -C "$PROJECT" add ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#456): complete §9.2 chapter index + layer matrix

55 chapters indexed across 6 journeys. Layer × Chapter matrix populated.

Refs #456"
```

---

## Batch 4: Cleanup

### Task 9: Delete DESIGN.md and update references

Remove the stale DESIGN.md and update all references to point to ARC42STORIES.MD.

**Files:**
- Delete: `docs/DESIGN.md`
- Modify: `CLAUDE.md` (update design document reference)

**Interfaces:**
- Consumes: Complete ARC42STORIES.MD
- Produces: Clean project with no stale design doc and updated references

- [ ] **Step 1: Check for references to DESIGN.md**

```bash
grep -r "DESIGN.md" "$PROJECT" --include="*.md" --include="*.java" -l
```

Review each hit — update references that should point to ARC42STORIES.MD, leave references in git commit messages or historical docs alone.

- [ ] **Step 2: Update CLAUDE.md**

In CLAUDE.md, find the "Design Document" section and update:
- Change `docs/specs/2026-04-13-qhorus-design.md` reference to note ARC42STORIES.MD as the primary architecture record
- Add reference to ARC42STORIES.MD
- Remove or update the `docs/DESIGN.md` reference
- Set `HAS_ARC42STORIES=yes` context for future sessions

- [ ] **Step 3: Delete docs/DESIGN.md**

```bash
git -C "$PROJECT" rm docs/DESIGN.md
```

- [ ] **Step 4: Final consistency review**

Read the complete ARC42STORIES.MD end-to-end. Check:
- All 55 chapter entries present and formatted consistently
- Layer Impact tables use consistent layer names (L1-L8)
- Journey overview table matches actual chapter ranges
- Chapter index matches actual chapter entries
- No TODO/TBD/placeholder text remains
- Mermaid diagrams have valid syntax
- All issue references use `#N` format
- Anti-patterns in §8 are inline (not just references)

- [ ] **Step 5: Commit**

```bash
git -C "$PROJECT" add -A
git -C "$PROJECT" commit -m "docs(#456): delete DESIGN.md, update CLAUDE.md references

ARC42STORIES.MD supersedes DESIGN.md per arc42stories-casehub-profile.md.

Closes #456"
```

---

## References

- `specs/issue-456-arc42stories-migration/2026-09-26-arc42stories-migration-design.md` — design spec
- `docs/DESIGN.md` — existing stale design document (source material)
- `CLAUDE.md` — authoritative project conventions (source material)
- `casehub-work/ARC42STORIES.MD` — foundation-tier template
- `../parent/docs/arc42stories-casehub-profile.md` — CaseHub ARC42Stories profile
- `docs/adr/` — architecture decision records (source material)
- GitHub #456 — focal issue
