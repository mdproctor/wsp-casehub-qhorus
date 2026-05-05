---
layout: post
title: "Day Zero: Designing a Multi-Agent Mesh"
date: 2026-04-13
type: day-zero
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [architecture, mcp, design, quarkus]
---

I'd been running `cross-claude-mcp` for a while — a Node.js server that let Claude agents share channels, pass messages, and coordinate on shared artefacts. It worked, but it wasn't production-grade. No native image, no proper lifecycle, no Quarkus. When I started thinking seriously about what the Quarkus Native AI agent ecosystem should look like, Qhorus was the obvious starting point.

The name is from Horus — the Egyptian god of the sky, associated with communication between worlds. The idea: a peer-to-peer mesh that any Quarkus app can join, where agents get channels, typed messages, shared data, and presence without knowing about each other in advance.

Before writing any code I wanted to understand the landscape. I spent time with A2A (Google's agent-to-agent protocol), ACP (IBM/LF AI's agent communication protocol), AutoGen, LangGraph, OpenAI Swarm, Letta, and CrewAI. The comparison was illuminating. Most frameworks assume a central orchestrator — one agent coordinates, others execute. Swarm is the exception; it passes control entirely between agents. Letta is interesting for its memory model and its tagging system for agent roles. A2A is the most relevant for interoperability but it's protocol, not infrastructure.

What I wanted was something that sits orthogonal to all of them: a communication mesh that any framework could talk through, not another orchestration model. Qhorus shouldn't care whether the agents using it are AutoGen agents, Claude sessions, or something else entirely.

The design decisions came quickly once I had that frame.

**Channel semantics** were the most important choice. Not all channels behave the same way — a coordination channel where the last value wins (LAST_WRITE) is fundamentally different from an accumulation channel where everyone contributes (COLLECT) or a barrier channel that waits for all declared contributors (BARRIER). I settled on five: APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE. Each declared at channel creation, enforced on every write.

**Message types** were seven: request, response, status, handoff, done, event. `event` is special — it's observer-only, never returned to agents polling for messages. The distinction matters: agents shouldn't see their own telemetry as part of their context.

**Transport:** Streamable HTTP only. MCP 2025-06-18 deprecated the old SSE approach. `quarkus-mcp-server` 1.11.1 fixed sampling and elicitation in native image — they were silently broken in earlier versions. I documented this as a hard requirement.

**Artefacts:** inline payloads don't scale. `artefact_refs: List<UUID>` on messages, with a claim/release lifecycle for GC. Agents reference data by UUID; the payload lives in SharedData.

The three-project ecosystem came together on paper: CaseHub for orchestration (case-based reasoning, human task management), Qhorus for the communication mesh (this), Claudony as the integration layer that embeds both. Each is independent; Qhorus has no dependency on either of the others.

Claude generated the scaffold from the spec in a single pass — `pom.xml`, `CLAUDE.md`, `docs/specs/2026-04-13-qhorus-design.md`. The design document ran to forty pages: ER diagrams, Flyway schema, the full MCP tool surface, a complete build roadmap. No Java code yet — just the vision made concrete enough to build from. That felt right. Starting with code before the design is clear is how you end up refactoring for weeks.

The one thing I got wrong on day zero: I underestimated how much the ledger integration would grow. I had "structured observability" as a single phase in the roadmap. It eventually became its own dependency — `quarkus-ledger` — with its own DESIGN.md and its own set of surprises. But that's for later.
