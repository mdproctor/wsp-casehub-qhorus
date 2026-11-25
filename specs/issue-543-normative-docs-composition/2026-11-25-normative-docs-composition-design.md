# Normative Docs Composition Revision — Design Spec

**Issue:** casehubio/qhorus#543
**Date:** 2026-11-25
**Scope:** Revise all normative framework documentation for composition-first framing

---

## Thesis

The current normative documentation tells the story bottom-up: theory
produces a four-layer stack, the stack produces accountability, accountability
produces value. This framing was correct when the normative layer was the
only capability. It is no longer complete.

Three capabilities now exist that compose orthogonally:

- **Communication governance** (template registry + NLP) — mechanically
  extracts structured governance actions from agent prose
- **Normative accountability** (commitments, ledger, trust) — tracks
  obligations with formal lifecycle and tamper-evident record
- **Protocol enforcement** (playbook bridge, channel protocols) — evaluates
  governance rules and gates outcomes on participant agreement

Each is independently useful. Together they close a loop that none can
close alone: **agent prose becomes accountable governance.**

The revised documentation tells this composition story as the thesis,
with the theoretical foundations serving each capability rather than
forming a mandatory bottom-up stack.

---

## Central argument the docs must establish

Conversation without governance is unaccountable. Governance without
perception is manual — a human must read every message and decide what
it means. Perception without accountability is cosmetic — you can parse
"I approve" from prose but nothing tracks whether the obligation was
fulfilled or the approval was authorised.

The three capabilities address these gaps:

| Gap | Capability | What it provides alone |
|-----|-----------|----------------------|
| No perception | Communication governance | LLM prose parsed into structured acts |
| No tracking | Normative accountability | Obligations recorded, causally linked, trust-scored |
| No enforcement | Protocol enforcement | Rules evaluated, outcomes gated on participant agreement |

**Composed:** prose in → structured extraction → protocol evaluation with
handshake gating → enforced governance → auditable ledger entry.

The composition is additive and opt-in. A channel can use any subset.
The normative layer works without templates. Templates work without
enforcement. Enforcement works without the ledger. Each combination
produces value; the full composition produces accountability.

**The accountability argument (D6):** At some point, agent conversation
must result in something actionable or recordable — a function call, an
event, a normalised structure. Without this, LLM conversations produce
no accountability, which is dangerous in regulated environments. The
template registry is the mechanism by which conversation becomes
accountability. The "governance gap" (most messages are reasoning, not
governance acts) is not a limitation of the template registry — it is
the problem statement the entire framework exists to solve.

**The handshake argument (D8):** Governed outcomes are gated, not just
recorded. A vote only counts when the template extraction confirms the
content is a structurally matched vote. An approval only takes effect
when the protocol confirms all required participants have cast
structurally matched approvals. This is the join of perception +
enforcement + commitment lifecycle — without extraction, protocols
can only gate on message count, not on what was actually decided.

---

## Document revisions

### 1. normative-layer.md — Primary revision

**Current structure (590 lines):**
1. What It Is and Why It Matters (~35 lines)
2. The Four Layers (stack diagram, ~60 lines)
3. Theoretical Foundations (~30 lines, long paragraphs)
4. From Status Fields to Accountability (~25 lines)
5. Why Not a Workflow Engine? (~10 lines)
6. What This Enables in Practice (seven questions, ~125 lines)
7. Insurance Claim Example (~145 lines)
8. Production Incident Example (~90 lines)
9. Layered Ecosystem Table (~15 lines)
10. Related Documents (~10 lines)

**Revised structure:**

**Section 1 — What It Is and Why It Matters (revised opener)**

Same thesis — accountability infrastructure for multi-agent AI — but
reframed around three capabilities rather than four theoretical layers.

Key additions to the opening argument:
- Accountability requires perception. An agent that writes "I approved
  this architecture" has performed a governance act. Without mechanical
  extraction, that act exists only as a string in a log. With it, the
  act becomes a structured record: who approved, what was approved,
  the stated rationale — extractable, enforceable, auditable.
- Accountability requires enforcement. An approval that is recorded
  but not gated on authorisation, participant agreement, or protocol
  rules is a rubber stamp — formally present, operationally meaningless.
- The normative layer provides accountability. Communication governance
  provides perception. Protocol enforcement provides the rules. Together
  they produce governed accountability — the first closed loop from
  natural language to auditable, enforced governance.

The "this is not X" contrasts extend:
- "This is not NLP for convenience. It is the perception layer that
  makes agent governance mechanical rather than manual."
- "This is not template matching for parsing. It is the extraction
  that turns prose into accountable acts."

**Section 2 — Three Capabilities That Compose (new, replaces four-layer stack)**

The centrepiece. Each capability gets a subsection establishing its
standalone value and formal grounding:

**Communication Governance.** Templates define protocol vocabulary as
sentence patterns with typed variables. NLP extraction tolerates
grammatical variation (tense, number, determiners). Schema publication
tells agents what sentence shapes the protocol expects. Bidirectional:
the same templates extract fields from prose and generate conformant
prose from fields. The extraction is deterministic and auditable — no
LLM in the parsing loop.

Formal grounding: structural linguistics and lemmatisation theory.
The same convergence argument that applies to speech act types applies
to sentence schemas — formally defined patterns produce convergence
across independently trained LLMs.

Honest limitations: structural matching handles inflection but not
arbitrary paraphrase. Adaptive phrase variation (learning known
variations over time via LLM fallthrough) addresses this architecturally.
Schema publication is the primary mitigation — governance acts are
formulaic by design, not despite it.

**Normative Accountability.** The existing four-layer content (speech
acts, commitments, defeasible reasoning, ledger) restructured as one
capability rather than a mandatory stack. The theory sections serve
this capability — speech act theory grounds the message taxonomy,
deontic logic grounds the commitment lifecycle, etc.

**Protocol Enforcement.** Channel protocols evaluate governance rules
at dispatch time. Playbook bridge connects YAML-defined rules to the
enforcement pipeline. Enforcement modes (advisory/blocking/quarantine)
determine consequence. Governed outcomes are gated on participant
agreement — the template-extracted fields are what the protocol
evaluates, not just message metadata.

**The Composition Pipeline.** Diagram showing the full path with each
step labelled as optional:

```
LLM prose
  → [Communication Governance] Template extraction (optional)
    → [Normative Accountability] Speech act classification + commitment
      → [Protocol Enforcement] Rule evaluation + handshake gating (optional)
        → Enforcement gate (optional)
          → Normative ledger entry (always, when normative layer is active)
```

Each `→` is opt-in. A channel with only normative accountability works.
A channel with governance + accountability but no enforcement works.
The pipeline degrades gracefully — removing a capability reduces what
is governed, not what is possible.

**Section 3 — Formal Foundations (condensed, reframed)**

Same academic content (FIPA, deontic logic, Singh, Winikoff) but
reframed as "why each capability exists" rather than a bottom-up stack.
Each formal tradition grounds a specific capability:

| Tradition | Capability it grounds | What it provides |
|-----------|----------------------|-----------------|
| Speech act theory (Searle, FIPA) | Communication governance + normative accountability | Message types are performatives; sentence patterns are governance vocabularies |
| Deontic logic (Meyer, Governatori) | Normative accountability | Obligation, permission, prohibition — the commitment lifecycle |
| Defeasible reasoning | Normative accountability | Obligations can be transferred, excused, or overridden — formally |
| Social commitment semantics (Singh, Winikoff) | Normative accountability | Commitments are observable, verifiable social contracts |
| Structural linguistics | Communication governance | Lemmatisation, POS tagging, structural matching — NLP extraction |

The convergence argument extends: formal sentence schemas produce the
same convergence across LLMs that formal message types produce across
agent frameworks. Without formal schemas, every team invents its own
governance vocabulary — the same Tower of Babel that KQML produced for
message types.

**Section 4 — From Status Fields to Accountability (mostly unchanged)**

One addition: "Adding NLP parsing is not the same as adding communication
governance." A regex that extracts "approve" from a string is parsing.
A template registry that maps "I approved this architecture because it
handles failover" to `{decision: approve, subject: architecture,
rationale: "it handles failover"}` with typed constraints, NLP tolerance,
and schema publication is governance. The extraction is deterministic,
the fields are typed, the schema is published, and the result feeds
directly into the enforcement pipeline. This is the same distinction as
status fields vs. accountability — applied to perception.

**Section 5 — Why Not a Workflow Engine? (mostly unchanged)**

One addition connecting template governance: a workflow engine requires
the vocabulary to be defined upfront in the process definition. Template
governance allows the vocabulary to be defined declaratively and matched
at runtime — agents discover the governance vocabulary from published
schemas, not from workflow step definitions.

**Section 6 — What This Enables in Practice (revised)**

The seven questions gain template-backed depth. Existing answers show
what the normative layer provides; revised answers show what the full
composition provides.

Example revision for "Who approved this decision?":
- Current: `get_obligation_chain` returns DONE entries.
- Revised: `get_obligation_chain` returns DONE entries with
  template-extracted structured fields — `{decision: approve,
  subject: architecture, rationale: "handles failover"}` — not just
  a prose string. The approval is not an inference from the message
  content; it is a mechanical extraction with typed fields.

New eighth question: *"Does what the agent said match what it claims
to have done?"* Speech act verification via template matching. An agent
sends `type: DONE` but the content is "I need more time" — that's
structurally a STATUS, not a completion. Template matching detects the
mismatch; the enforcement pipeline produces an advisory.

**Section 7 — Insurance Claim Example (extended)**

Specific steps gain template matching annotations woven into the
narrative — not callout boxes, but natural asides in the same style as
the existing layer annotations.

Steps to extend:
- Step 6 (compliance DECLINE): template extracts structured rejection
  fields
- Step 7 (HANDOFF): template extracts delegation target and reason
- Step 10 (payment FAILURE): template extracts payment method and error
- Steps 8-9 (regulatory filing): template extracts reference numbers
  as structured evidence

The governance protocol example (new section) replaces the need to
overload the insurance scenario with template matching detail. The
insurance example gets light annotations; the new example gets the
full pipeline walkthrough.

**Section 8 — Production Incident Example (extended)**

Similar light annotations:
- DECLINE with reason template: structured scope dependency
- FAILURE with cause template: structured failure evidence
- HANDOFF with delegation template: structured transfer

**Section 9 — Governance Protocol Example (new)**

Full pipeline walkthrough using the review/vote/escalate scenario from
the platform TemplateRegistryIntegrationTest. This is the first example
where all three capabilities are load-bearing simultaneously.

Scenario: A multi-agent architecture review with governed voting.

1. Protocol defined as YAML corpus:
   ```yaml
   namespace: protocol:review
   templates:
     vote:
       pattern: "I ${decision} this ${subject}[ because ${rationale}]"
       variables:
         decision: { type: enum, values: [approve, reject, abstain] }
         subject: { type: string }
         rationale: { type: string, optional: true }
     escalate:
       pattern: "Escalating to ${target}[ due to ${reason}]"
       variables:
         target: { type: string }
         reason: { type: string, optional: true }
   ```

2. Channel configured with protocol enforcement:
   - `protocols: [CONTRIBUTION_REQUIRED, protocol:review]`
   - `protocolParticipants: [architect, security-lead, tech-lead]`
   - `enforcementMode: BLOCKING`

3. Agents vote via natural language. NLP tolerates variation:
   - "I approved this architecture" → `{decision: approve}` (past tense)
   - "I reject this proposal because it lacks tests" → `{decision: reject, rationale: "it lacks tests"}`
   - "I approves this design" → `{decision: approve}` (agreement error)

4. Protocol enforcement gates the outcome:
   - CONTRIBUTION_REQUIRED checks all three participants have voted
   - Template extraction confirms each message is a structurally matched vote
   - The outcome (approval, rejection, escalation) is determined by the
     extracted fields, not by counting messages

5. Handshake gating: the approval only takes effect once all participants
   have cast structurally matched votes. Not "has everyone sent a
   message" but "has everyone voted, and do the extracted fields
   constitute agreement."

6. Normative ledger records the full governance trail: each vote as a
   ledger entry with extracted fields, the collective outcome, the
   protocol evaluation result, and the enforcement decision.

7. Bidirectional generation: the infrastructure can produce a governance
   summary from the extracted fields — "The architecture was approved
   by architect and tech-lead (2-1). security-lead voted reject, citing
   lack of tests."

This example demonstrates:
- Communication governance (template extraction with NLP tolerance)
- Normative accountability (commitment tracking, ledger entries)
- Protocol enforcement (contribution requirement, handshake gating)
- The composition (all three working together, each independently removable)

**Section 10 — The Layered Ecosystem (updated)**

The existing table (Qhorus alone → + casehub-ledger → + casehub-work →
+ Claudony → + CaseHub) gains a composition dimension:

| Composition | Without | With |
|---|---|---|
| Communication governance | Governance acts are prose strings — human must interpret | Governance acts are structured records — infrastructure interprets |
| Protocol enforcement | Rules are ad-hoc, metadata-only | Rules are YAML-declared, content-aware, handshake-gated |
| Full composition | Message types tracked, prose ungoverned | Full loop: prose → extraction → enforcement → accountability |

**Section 11 — Related Documents (updated)**

Add references to the template registry module documentation and the
platform epic (#943).

---

### 2. normative-framework.md — Navigation update

**Current:** Reading order starts with agent-mesh-framework.md (developer)
or normative-layer.md (theory).

**Revised:** Reading order reflects the composition-first primary doc:
- "Start here" → normative-layer.md (now composition-first, not theory-first)
- Updated section descriptions reflecting the new structure
- New entry for template registry / communication governance capability
  (references platform, with explanation that the perception layer lives
  in the platform library because it is not specific to qhorus)

---

### 3. normative-summary.md — Critique revision

**Current gaps (5 items):** Channel backend (implemented), cross-channel
correlation (implemented), WorkItem trust integration, extension contract,
empirical hypothesis.

**Revised critique:**

- Perception layer: now addressed by the template registry. The gap was
  "content-level understanding of agent prose" — the template registry
  provides this.
- Paraphrase tolerance: acknowledged as a current limitation, with
  adaptive phrase variation as the architectural response (not "better
  NLP" but "learning variations over time").
- Extension contract (Gap 3): reframed in light of template namespaces.
  The namespace system IS the extension mechanism for communication
  governance vocabularies. The question becomes: how do domain-specific
  governance vocabularies compose with the base normative vocabulary?
- WorkItem trust integration (Gap 2): unchanged.
- Empirical hypothesis (Gap 4): unchanged.
- New gap: Corpus quality tooling — no way to validate a corpus against
  real LLM variation before deployment (platform#974).
- New gap: Hot-update safety — non-atomic namespace replacement
  (platform#972).

Updated "What the Three Documents Establish Together" to include the
composition model and the perception layer.

---

### 4. agent-mesh-framework.md — New section

Add a section: **Template-Based Communication Governance**

Developer-facing guide covering:
- How to define a protocol vocabulary (YAML corpus format)
- How NLP extraction works (regex fast-path, NLP fallback, lemmatisation)
- How to register templates and wire them to channel protocols
- How schema publication works (agent asks "what should I say?")
- How enforcement modes interact with template matching
- How handshake gating works with template-extracted fields
- Example: wiring the review protocol into a channel

This section belongs in the developer guide, not in normative-layer.md.
The primary doc argues why; the developer guide shows how.

---

## Writing approach

The existing doc has a distinctive voice:

- **Thesis-driven:** Each section advances an argument, not a feature list
- **Contrast-heavy:** "This is not X. It is Y." — establishing category
  distinctions
- **Scenario-grounded:** Abstract claims immediately followed by concrete
  scenarios showing what changes
- **Assertive declarations:** Bold opening sentences per section
- **Honest caveats:** Limitations acknowledged directly ("this is a
  hypothesis, not a proven fact")

The revision maintains this voice throughout. New content enters at the
same register — not "we also have a template registry" but "governance
without perception is manual — the same accountability gap that made
logs insufficient for obligations makes unstructured prose insufficient
for governance."

The composition framing must feel inevitable, not retrofitted. The reader
should feel that the doc was always structured around three capabilities
— not that someone added two more boxes to a diagram.

---

## Implementation order

1. normative-layer.md — the primary revision (longest, most critical)
2. normative-framework.md — navigation update (short)
3. normative-summary.md — critique revision (medium)
4. agent-mesh-framework.md — new section (medium)

Each doc committed separately after completion.

---

## References

- normative-layer.md — current primary doc (590 lines)
- normative-framework.md — current navigation index
- normative-summary.md — current critique
- agent-mesh-framework.md — current developer guide
- TemplateRegistryIntegrationTest.java (platform) — integration test showing full extraction pipeline
- UnifiedTemplateRegistry.java (platform) — registry API
- NlpTemplateRegistry.java (platform) — NLP matching tier
- ChannelProtocol.java (qhorus) — protocol enforcement SPI
- ProtocolResolver.java (qhorus) — dynamic protocol resolution
- PlaybookChannelProtocol.java (qhorus) — playbook-to-protocol bridge
- casehubio/platform#943 — template extraction epic
- casehubio/platform#959 — unified template registry
- casehubio/platform#961 — consumer migration to unified registry
- casehubio/platform#969 — YAML corpus format
- casehubio/platform#970 — match playbook step
- casehubio/platform#972 — atomic namespace update (filed from this review)
- casehubio/platform#973 — NearMiss guided rejection (filed from this review)
- casehubio/platform#974 — corpus quality testing tooling (filed from this review)
- Skeptical review (session, 2026-11-25) — adversarial analysis of composition model
