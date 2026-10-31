---
layout: post
title: "Governance Follows the Org Chart"
date: 2026-10-31
entry_type: article
subtype: diary
projects: [casehubio/qhorus]
tags: [governance, org-structure, multi-agent, architecture, eidos]
series: issue-500-org-governance-phase1-3
---

# Governance Follows the Org Chart

There's a pattern in every multi-agent framework I've studied — A2A, AutoGen, LangGraph, CrewAI, Swarm — that treats agents as a flat pool of capabilities. Need an analyst? Query the registry, rank by score, dispatch to the winner. The capability graph is the entire world model.

This works until the system gets big enough to have politics.

I don't mean literal politics. I mean the structural reality that ten agents working in the same domain share context, norms, and expectations that a hundred agents across an organisation do not. When you dispatch work to the highest-scoring agent globally, you're optimising for individual competence at the expense of collective coherence. The agent you chose might be brilliant at analysis. It might also have no idea how your team's channels work, what your reporting obligations are, or what "done" means in your operational context.

Human organisations solved this problem thousands of years ago. Work flows through organisational structure. You send it to someone on your team. If nobody on your team can handle it, you escalate to the next level. You don't send it to the best person in the company — you send it to the best person who shares enough context to do the job without a knowledge transfer.

We just started applying this principle to Qhorus, and the implications run deeper than I expected.

## Beyond routing

The first piece was [routing preference](/blog/2026-10-30-mdp01-the-team-you-know) — filtering capability matches to same-unit peers before the selector sees them. That landed in a few hours and the code is clean. But routing is just one of six governance surfaces that should respect org structure.

![Flat vs. org-aware governance — six dimensions affected](images/governance-flat-vs-org.svg)

**Trust computation.** If an analyst's trust score is computed from attestations by every peer in the system, the score reflects performance across contexts that may have nothing in common. The analyst might be excellent in fraud detection and mediocre in compliance analysis — but the global score averages both. Org-scoped trust weights attestations from operational peers more heavily. The trust signal and the routing decision are drawn from the same population.

**Delegation.** A HANDOFF message says "I can't do this, someone else should." In a flat system, any agent can delegate to any other. That's fine in a five-agent system. In a fifty-agent system, unconstrained delegation is how work disappears into corners nobody monitors. Org-aware delegation checks whether the sender is authorised to delegate across unit boundaries — not a hard block, but an advisory that enforcement mode can act on. Within-unit delegation stays unrestricted.

**Capacity redistribution.** When an agent is overloaded and work needs reassigning, the current system scans globally for capacity. With org structure, redistribution prefers same-unit agents first — the same people who already understand the domain and the reporting chain. You widen to other units only when the local team is saturated.

**Protocol inheritance.** Channels created within a team should inherit that team's communication protocols by default. Instead of configuring every channel individually, the organisational unit defines its norms — request-response limits, contribution requirements, round-robin participation — and channels inherit them at creation time. Override when needed, inherit when not.

**Audit context.** When the compliance report asks "show me all governed interactions within Team Alpha", the system currently has to join ledger entries against the org structure at query time. Stamping `orgUnitId` on ledger entries at write time makes this a simple filter. Compliance reporting per organisational boundary becomes a first-class capability.

## The design principle that holds it together

Every one of these dimensions follows the same contract: org-aware when the structure is available, flat when it isn't. The `OrgAwareDispatch` dependency is always nullable. Missing org data means the system behaves exactly as it did before — not that it breaks.

![The nullable/fallback contract applied across all governance dimensions](images/nullable-fallback-pattern.svg)

This is load-bearing. The eidos org module is an optional dependency, not a required one. A Quarkus app that adds `casehub-qhorus` without `casehub-eidos` gets the full communication mesh with flat governance. Add eidos, and governance starts respecting institutional boundaries. No code changes, no configuration, no migration.

The alternative — making org structure a hard dependency everywhere — would be cleaner architecturally and worse practically. Most deployments start small. Requiring org definitions before the system works at all is a barrier to adoption that provides no value until the system is large enough to need it. The nullable pattern means org governance is there when you grow into it.

## What this is really about

The multi-agent ecosystem is converging on capability-based routing as the standard pattern. I think that's necessary but not sufficient. Capability tells you what an agent can do. Org structure tells you what an agent should be doing — which is a governance question, not a capability question.

The agents that share a reporting chain also share the context, protocols, and domain knowledge that make collaboration efficient. Org structure isn't bureaucracy imported into software. It's a proxy for shared understanding, and shared understanding is what makes the difference between a collection of capable agents and a coherent system.
