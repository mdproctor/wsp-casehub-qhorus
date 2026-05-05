---
layout: post
title: "Phase 12: Structured Observability and an Unplanned Detour"
date: 2026-04-15
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [mcp, observability, quarkus-ledger, claudony]
---

Phase 12 landed as planned. Every EVENT message through `send_message` now creates an `AgentMessageLedgerEntry` — a JPA subclass of quarkus-ledger's `LedgerEntry` — with `toolName`, `durationMs`, optional `tokenCount`, and optional `contextRefs` capturing what the agent could see at decision time. Two new MCP tools: `list_events` queries the structured audit trail with channel, agent, and time range filters; `get_channel_timeline` gives a chronological interleaved view of all message types for a channel.

The `contextRefs` field is the EU AI Act Article 12 angle — read-set capture, applied to AI agents. Tarkus already uses the same pattern for human reviewers. Same concept, different domain.

One fix worth noting. The `quarkus.ledger.*` config keys were appearing as "Unrecognized configuration key" in startup logs even though the defaults applied correctly. The problem: `@ConfigMapping` alone doesn't register the prefix in the extension descriptor — the `quarkus-extension-processor` looks for `@ConfigRoot`, not `@ConfigMapping`. Adding `@ConfigRoot(phase = ConfigPhase.RUN_TIME)` alongside `@ConfigMapping` tells the processor to emit the prefix. Warnings gone.

Then the unplanned part.

Claudony has `McpServer.java` — 196 lines of hand-rolled JSON-RPC 2.0 at `@Path("/mcp")`. Adding `quarkus-qhorus` as a dependency pulls in `quarkus-mcp-server-http`, which also registers at `/mcp`. Two endpoints at the same path won't work. Before Phase 8 can happen, Claudony's 8 tools need to migrate to `quarkus-mcp-server`.

The migration itself is mechanical: convert each `case` branch to a `@Tool`-annotated method on an `@ApplicationScoped` bean. What wasn't mechanical was the protocol. quarkus-mcp-server implements Streamable HTTP (MCP 2025-06-18), which requires `Accept: application/json, text/event-stream` on every request and a `Mcp-Session-Id` header the server issues on `initialize` and expects back on all subsequent requests. The existing Claudony tests sent plain JSON POSTs without either header. All 10 failed immediately.

Once we understood why, the fix was a `@BeforeEach` that calls `initialize`, captures the session ID header, and a helper that attaches it to every subsequent request. 212 tests passing after.

The migration also surfaced a test structure problem. `McpServerTest` was testing tool logic through the full JSON-RPC protocol stack — every test paid the Streamable HTTP handshake overhead just to call `listSessions()`. The right split is `ClaudonyMcpToolsTest`, which injects `ClaudonyMcpTools` directly via CDI and calls methods with no HTTP, and `McpProtocolTest`, which covers protocol compliance separately. We wrote the briefing; the Claudony Claude implemented it.

557 Qhorus tests passing. Phase 12 closes the last designed phase. Phase 8 — embedding Qhorus in Claudony — is now unblocked.
