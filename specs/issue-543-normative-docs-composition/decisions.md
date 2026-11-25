# Decisions — Normative Docs Composition Revision

## D1: Document framing

**Choice:** Composition-first — lead with three independent capabilities and their composition. Theory supports why each exists, not the other way around.
**Alternatives:**
- Theory-first + composition section — keeps existing narrative, adds composition as a new section. Risk: reads as bolted-on.
- Parallel tracks — two reading paths (theory / practitioner). Risk: doc proliferation, harder to maintain.
**Rationale:** The composition model is the thesis now. The theory served as the thesis when only accountability existed; with three capabilities, the thesis is their composition.
**Trade-offs:** Restructuring a well-regarded 590-line doc risks losing the persuasive flow that works.
**Sources:** normative-layer.md (current), platform template-registry module, qhorus playbook-bridge module
**Exploration:** quick
**Status:** captured

## D2: Document strategy

**Choice:** Revise normative-layer.md in place — one authoritative document, restructured around composition.
**Alternatives:**
- New primary + keep existing — avoids touching the existing doc, but creates two docs that partially overlap.
- New primary + retire existing — cleanest but loses git history and inbound links.
**Rationale:** The goal is a cohesively whole story. Two documents telling overlapping stories is the opposite.
**Trade-offs:** A bad revision makes the existing doc worse. Quality bar is high.
**Sources:** normative-framework.md (navigation), normative-summary.md (critique)
**Exploration:** quick
**Status:** captured

## D3: Worked examples

**Choice:** Both — extend existing insurance claim / production incident examples with template matching annotations, AND add a new governance protocol example (review/vote/escalate).
**Alternatives:**
- Extend existing only — natural, but doesn't show the full composition pipeline end-to-end.
- New example only — leaves existing examples incomplete.
**Rationale:** Extending existing examples shows template matching is not a separate feature but part of how governance always worked. The new example demonstrates the full pipeline where all three capabilities are load-bearing simultaneously.
**Trade-offs:** Longer doc. The new example adds ~200-300 lines.
**Sources:** TemplateRegistryIntegrationTest.java (platform), insurance claim scenario (normative-layer.md)
**Exploration:** quick
**Status:** captured

## D4: Revision scope

**Choice:** All normative docs — normative-layer.md, normative-framework.md, normative-summary.md, agent-mesh-framework.md.
**Alternatives:**
- normative-layer.md only — focused but leaves supporting docs inconsistent.
- Primary + framework index — partial coherence.
**Rationale:** The user wants a joined-up, complete story. Partial revision leaves gaps that undermine the composition narrative.
**Trade-offs:** Larger scope, more writing, more risk of inconsistency during revision.
**Sources:** normative-framework.md (navigation), normative-summary.md (critique), agent-mesh-framework.md (developer guide)
**Exploration:** quick
**Status:** captured

## D5: Quality constraint

**Choice:** The revision must feel cohesively whole — not bolted or patched. The composition framing should read as the natural starting point, not a retrofit.
**Rationale:** Direct user instruction. The existing doc has a distinctive thesis-driven voice (assertive, contrast-heavy, scenario-grounded). New content must match this register.
**Trade-offs:** Slower writing. Cannot just insert sections — must rewrite for flow.
**Sources:** User direction
**Exploration:** quick
**Status:** captured

## D6: Template governance is not optional convenience — it is the accountability mechanism

**Choice:** The docs frame template matching as the mechanism by which agent conversation becomes accountable — not as an optional enhancement for structured governance acts only. At some point, prose must result in something actionable (function call/event) or recordable (normalised structure). Without it, LLM conversations produce no accountability — which is dangerous in regulated environments.
**Alternatives:**
- Frame template matching as a "precision instrument for the 5-10% of messages that are governance acts" — treats the other 90% as inherently ungoverned.
**Rationale:** The skeptical review identified the "governance gap" (90% of messages are reasoning/discussion, not governance acts). The user's position: that gap IS the problem. Agents conversing freely with no accountable outcome is the status quo the normative layer exists to change. Template matching closes that gap — conversation is free, but governed outcomes must be robust and unambiguous.
**Trade-offs:** Sets a higher bar for the template registry — it must eventually handle the breadth of how agents express governance, not just exact template shapes.
**Sources:** Skeptical review finding #2, user direction, platform adaptive phrase variation work
**Exploration:** quick (via adversarial review)
**Status:** captured

## D7: Adaptive phrase variation addresses the paraphrase cliff

**Choice:** Platform is building adaptive phrase variation with LLM fallthrough to learn known variations over time. The docs should acknowledge the paraphrase limitation honestly while noting it is being addressed architecturally, not just with better NLP.
**Alternatives:**
- Accept structural matching as the ceiling — paraphrases always require schema conformance.
**Rationale:** The skeptical review's #1 finding (paraphrase cliff) is real but being addressed. The composition model's strength is that the matching tier is pluggable — NLP today, adaptive variation tomorrow, semantic similarity later.
**Trade-offs:** The adaptive path is in progress, not shipped. Docs should be honest about current state.
**Sources:** Skeptical review finding #1, platform work in progress
**Exploration:** quick (via adversarial review)
**Status:** captured

## D8: Governed outcomes as handshake gating

**Choice:** The docs frame template-extracted governance acts as part of protocol handshake gating — the governed outcome only takes effect once all participants have agreed. Not "has everyone sent a message" but "has everyone cast a structurally matched vote, and do the extracted fields constitute agreement."
**Alternatives:**
- Frame template matching as extraction-only, with protocol enforcement as a separate concern.
**Rationale:** This is the natural join of template matching + protocol enforcement + commitment lifecycle. The template registry extracts the structured governance act. The protocol checks participant coverage and threshold. The commitment records the collective agreement. Without the extraction layer, the protocol can only gate on message count, not on what was actually decided.
**Trade-offs:** Raises the bar for protocol definitions — they must understand template-extracted fields, not just message metadata.
**Sources:** Existing qhorus protocols (CONTRIBUTION_REQUIRED, BARRIER, protocolParticipants), user direction
**Exploration:** quick
**Status:** captured
