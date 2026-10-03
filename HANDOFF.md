# HANDOFF — casehub-qhorus

## Last Session

Completed Batch 3 of qhorus-mesh (#465) and designed the AgentProvider bridge. Session covered implementation (mesh module scaffold + MCP tools), architecture correction (shim dropped — Claude Code connects via SSE directly), and full brainstorming cycle for the agent integration layer.

**Implementation completed (on branch `issue-465-qhorus-mesh`):**
- Fixed build break (28-arg backward-compat Channel constructor for compliance-report)
- Scaffolded `mesh/` module — standalone Quarkus app, H2 file, MCP HTTP+SSE, health check
- Implemented 7 MCP tools in `MeshMcpTools` (register, deregister, send, check, create, list, discover)
- Added `InstanceService.register()` overload with metadata parameter
- Updated CLAUDE.md with mesh module in project structure

**Design completed (on workspace branch `issue-465-qhorus-mesh`):**
- Dropped the connection shim (Task 8) — relay exposes SSE directly, no bridge needed
- Brainstormed AgentProvider bridge — 11 decisions, standard review (3 rounds each for decisions + spec)
- Wrote implementation plan: 5 batches, 5 tasks

## Resume Instructions

Branch is paused. Run `work resume` to restore it. Both repos will switch from main to `issue-465-qhorus-mesh`. The .plan and all specs/plans are on that branch.

## Remaining Work

All work is on branch `issue-465-qhorus-mesh`, issue casehubio/qhorus#465.

### From original mesh plan (`plans/2026-10-01-qhorus-mesh.md`)

| Task | Status | Description |
|------|--------|-------------|
| Task 1: Channel.metadata | Done | V56 migration, record field, entity round-trip |
| Task 2: ChannelQuery.byMetadata | Done | JPA + InMemory filtering |
| Task 3: ChannelCreateRequest.metadata + REST | Done | setMetadata merge semantics, REST endpoint |
| Task 4: Instance.metadata | Done | V57 migration, record field, entity round-trip |
| Task 5: InstanceQuery.byMetadata | Done | JPA + InMemory filtering |
| Task 6: Maven module scaffold | Done | mesh/ module, H2, MCP, health check |
| Task 7: Core MCP tools | Done | 7 tools: register, deregister, send, check, create, list, discover |
| Task 8: Connection shim | Dropped | Relay exposes SSE directly — no shim needed |

### From AgentProvider bridge plan (`plans/2026-10-02-agent-provider-bridge.md`)

| Batch | Task | Status | Description |
|-------|------|--------|-------------|
| 1 | AgentChannelBinding + SpeechActMapper | TODO | Binding record with builder; AgentEvent → MessageType mapping |
| 2 | AgentProviderBackend | TODO | ChannelBackend (AT_LEAST_ONCE), sender loop guard, target routing, virtual thread async delivery, semaphore concurrency control |
| 3 | AgentBridgeService | TODO | Lifecycle SPI — create/destroy/list bindings, AgentBackend key resolution, persistent session management |
| 4 | MeshApi migration | TODO | Migrate MeshMcpTools @Tool → MeshApi @McpDomain + MeshService impl |
| 5 | Integration test | TODO | Wire agent-bridge into mesh, COMMAND → agent → RESPONSE end-to-end |

### Key design decisions

- Bridge is `agent-bridge/` module at repo root (not in `mesh/` — mesh is an app, bridge is a library)
- ChannelBackend (AT_LEAST_ONCE) for delivery, virtual thread async invocation
- Sender-based loop guard only (indirect loops deferred to #468)
- MeshMcpTools migrates to @McpDomain pattern (MeshApi + MeshService)
- `casehub-platform` already has AgentProvider SPI with 7 backends (claude, openai, gemini, gemini-cli, codex, langchain4j, ollama) — bridge connects those to channels
- Cache-aware prompt structuring: stable context in systemPrompt (cached), per-message content in query()
- Commitment-aware terminal messages: RESPONSE fulfills COMMAND/QUERY, RESPONSE+DONE for PROPOSE
- Binding persistence is ephemeral (ConcurrentHashMap, no JPA) — consumers recreate on startup

## References

- `specs/qhorus-mesh-agent-provider/2026-10-02-agent-provider-bridge-design.md` — full design spec (11 decisions, 3 review rounds)
- `specs/qhorus-mesh-agent-provider/decisions.md` — 11 design decisions with rationale
- `plans/2026-10-02-agent-provider-bridge.md` — implementation plan (5 batches, 5 tasks)
- `specs/qhorus-mesh/2026-10-01-qhorus-mesh-design.md` — original mesh design
- `plans/2026-10-01-qhorus-mesh.md` — original mesh implementation plan (Tasks 1-7 done, Task 8 dropped)
- claudony #246/#247 — fleet deployment (future consumer of bridge)
