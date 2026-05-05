---
layout: post
title: "Phases 1–5: Building the Foundation"
date: 2026-04-14
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [mcp, panache, semantics, wait-for-reply, artefacts]
---

The next session was long. We went from zero Java to five complete phases in one pass — core data model, MCP tools, channel semantics, wait_for_reply, and artefact lifecycle. 193 tests by the end. Two critical production bugs found and fixed along the way.

Phase 1 was the foundation: five entity groups (Channel, Message, Instance, SharedData, PendingReply, plus Capability and ArtefactClaim), Flyway V1 schema, four services. Panache active record throughout — public fields, `@PrePersist` for UUIDs, no Lombok. The schema is straightforward, but getting the JPQL right for the more complex queries took iteration.

Phase 2 was the MCP surface — 14 `@Tool` methods on a single `@ApplicationScoped` bean, 8 return-type records nested as public statics. The record-per-tool approach keeps the return types explicit and stable. I'd go back and forth on this later (ADR-0001), but the pattern set here held.

Phase 3 was the interesting one: channel semantic enforcement. LAST_WRITE, EPHEMERAL, COLLECT, and BARRIER each have distinct dispatch logic in `check_messages`. BARRIER was the trickiest — it blocks until all declared contributors have posted, then delivers and clears atomically. Getting the contributor tracking right without introducing race conditions took a few iterations.

Code review after Phase 3 turned up two bugs worth naming. The first: `DataService.claim()` wasn't checking for an existing claim before inserting. Double-claim by the same instance would succeed, creating two rows. When the instance released once, the GC eligibility check — `claimCount == 0` — would still be satisfied, making the artefact GC-eligible while it was still claimed. Silent data loss. Fixed with an idempotency check before the insert.

The second: `wait_for_reply` was accepting `instance_id` as a parameter but was comparing it against `Message.sender` using UUID format. Human-readable agent names like `"planning-agent"` weren't matching. Claude flagged this during the review; the fix was removing the UUID assumption and doing a direct string comparison.

Phase 4 was wait_for_reply: a `PendingReply` row, SSE keepalives every 30s, a cleanup job for expired entries, and cancellation via `cancel_wait`. The polling loop is straightforward but the concurrency test required `ManagedExecutor` — raw `ExecutorService` loses the Quarkus CDI context, so `@Transactional` silently breaks on background threads. That one cost an hour.

Phase 5 wired `artefact_refs` through the MCP message flow — `send_message` accepts a list of UUIDs, `MessageSummary` returns them, and they're stored as a comma-separated string on the `Message` row. Not elegant, but it works and migrations are cheap.

Between phases we also spent time on the framework landscape. The multi-agent framework comparison — AutoGen, LangGraph, CrewAI, Swarm, Letta, A2A, ACP, and more — ended up as `docs/multi-agent-framework-comparison.md`. Writing it clarified where Qhorus fits: it's infrastructure, not a framework. It doesn't orchestrate; it provides the mesh that orchestrators use.

One architecture decision worth recording: I'd originally planned `request_approval` as Phase 10 — a human-in-the-loop approval gate built into Qhorus. Mid-session I moved it out. Approval gates are task lifecycle management, not agent communication. That's a different problem domain, and mixing them would make Qhorus more complex without making it more useful. The approval gate moved to `quarkus-tarkus` (later quarkus-workitems), which became a separate project that day.

By the end of this session Qhorus had a working MCP surface, five channel semantics enforced, correlation tracking, artefact lifecycle, and 193 tests. The foundation was solid enough to build everything else on.
