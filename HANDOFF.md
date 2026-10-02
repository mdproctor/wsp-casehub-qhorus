# HANDOFF — casehub-qhorus

## Last Session

Completed Batch 3 of qhorus-mesh (#465) and designed the AgentProvider bridge. Session covered implementation (mesh module scaffold + MCP tools), architecture correction (shim dropped — Claude Code connects via SSE directly), and full brainstorming cycle for the agent integration layer.

**Implementation completed:**
- Fixed build break (28-arg backward-compat Channel constructor for compliance-report)
- Scaffolded `mesh/` module — standalone Quarkus app, H2 file, MCP HTTP+SSE, health check
- Implemented 7 MCP tools in `MeshMcpTools` (register, deregister, send, check, create, list, discover)
- Added `InstanceService.register()` overload with metadata parameter
- Updated CLAUDE.md with mesh module in project structure

**Design completed:**
- Dropped the connection shim (Task 8) — relay exposes SSE directly, no bridge needed
- Brainstormed AgentProvider bridge — 11 decisions, standard review (3 rounds each for decisions + spec)
- Wrote implementation plan: 5 batches, 5 tasks

## Immediate Next Step

Execute the AgentProvider bridge plan: `plans/2026-10-02-agent-provider-bridge.md`. Start with Batch 1 (AgentChannelBinding + SpeechActMapper). The plan creates a new `agent-bridge/` module at the repo root.

**Key design decisions to keep in mind:**
- Bridge is `agent-bridge/` module (not in `mesh/` — mesh is an app, bridge is a library)
- ChannelBackend (AT_LEAST_ONCE) for delivery, virtual thread async invocation
- Sender-based loop guard only (indirect loops deferred to #468)
- MeshMcpTools migrates to @McpDomain pattern (MeshApi + MeshService)
- `casehub-platform` already has AgentProvider SPI with 7 backends — bridge connects those to channels

## References

- `specs/qhorus-mesh-agent-provider/2026-10-02-agent-provider-bridge-design.md` — full design spec
- `specs/qhorus-mesh-agent-provider/decisions.md` — 11 design decisions
- `plans/2026-10-02-agent-provider-bridge.md` — implementation plan (5 tasks)
- `specs/qhorus-mesh/2026-10-01-qhorus-mesh-design.md` — original mesh design
- claudony #246/#247 — fleet deployment (future consumer of bridge)
