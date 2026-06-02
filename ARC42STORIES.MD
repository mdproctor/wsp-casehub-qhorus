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

`casehub-qhorus` is the peer-to-peer agent communication mesh. It provides typed speech-act messaging — COMMAND (formal obligation request), DONE (successful fulfillment with findings), DECLINE (scope refusal), and RESPONSE (acknowledgment) — with commitment lifecycle tracking and automatic audit via `MessageLedgerEntry`.

Core capabilities: `ChannelService` (create, find, list channels per coordination unit), `MessageService` (dispatch COMMAND/DONE/DECLINE/RESPONSE; auto-creates Commitment on COMMAND; auto-closes Commitment on DONE/DECLINE), `CommitmentService` (open, acknowledge, fulfill, decline, fail, expire commitments), `MessageLedgerEntry` (per-speech-act immutable audit record — enables C4's tamper-evident review trail), `QhorusExtension` (Quarkus extension wiring datasource + CDI beans), `ObligorTrustPolicy` SPI (domain-defined trust gate for commitment acceptance).

Named datasource `qhorus` — separate from the default datasource. Always wire `quarkus.datasource.qhorus.*` in any consumer.

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| Formal obligation | Every COMMAND creates a Commitment — no agent obligation exists without a machine-tracked record |
| Formal scope refusal | DECLINE is a typed outcome with recorded reason — not a timeout or exception |
| Audit chain | Every speech act produces a MessageLedgerEntry that can be verified independently of the channel state |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-qhorus:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** casehub-qhorus owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

🔲 Core architectural patterns and module structure — complete in dedicated session from DESIGN.md and repo architecture.

---

## §5 Building Block View

🔲 C4Container diagram and module structure — complete in dedicated session.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

🔲 Anti-patterns inline (Symptom → Cause → Fix) — complete in dedicated session. Must be present per profile rules; do not defer to AGENTIC-HARNESS-GUIDE.md reference alone.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

🔲 Cross-cutting decisions — complete in dedicated session.

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `Channel` | Named communication context per coordination unit (e.g. pr-review-{N}/work). COMMAND/DONE/DECLINE messages are scoped to a channel. |
| `Commitment` | Machine-tracked agent obligation created on COMMAND dispatch. States: OPEN → ACKNOWLEDGED → FULFILLED | DECLINED | FAILED | EXPIRED. |
| `MessageLedgerEntry` | Immutable per-speech-act audit record — enables tracing every agent interaction in the ledger chain. |
| `COMMAND` | Speech act: formal request creating a Commitment. Target field is the Commitment obligor. |
| `DECLINE` | Speech act: formal scope refusal — not an error; closes the Commitment with reason. |
| `ObligorTrustPolicy` | SPI for domain-defined trust gate — evaluate minimum trust threshold before accepting a Commitment obligor. |

---

*Bootstrap document — generated 2026-06-02. §3–§9 require a dedicated session to complete from repository source material.*
