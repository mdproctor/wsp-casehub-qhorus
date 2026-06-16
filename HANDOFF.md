# CaseHub Qhorus — Session Handover
**Date:** 2026-06-16 — Advisory type enforcement shipped (#266 closed (already done), #271 closed)

---

## Immediate Next Step

Run `/work` to start #261 (Slack channel module). Main is clean, both repos aligned.

## What's Left

- `#261` — casehub-qhorus-slack-channel module (SlackChannelBackend) · L · Med
  - Requires brainstorming first (per session instructions)
  - casehub-connectors-slack-bot 0.2-SNAPSHOT already published to GitHub Packages
- `#277` — SSE live-stream integration test · S · Med
- `#278` — SSE keepalive + server-side timeout for orphaned consumers · M · Med
- `parent#241` — sync casehub-qhorus deep-dive in parent docs (A2A SSE, Set<MessageType>, DECLINE→cancelled, #235, #271 advisory enforcement) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #261 | casehub-qhorus-slack-channel module | L | Med | Next up; needs brainstorm |
| #271 | allowedTypes/deniedTypes advisory — **CLOSED** | — | — | Shipped this session |
| #266 | MessageReceivedEvent migration — **CLOSED** | — | — | Already complete before session |

## What Was Done This Session

**#271 closed — hybrid channel type enforcement:**

`StoredMessageTypePolicy` now discriminates by normative weight:
- COMMAND/QUERY violations: hard-enforced (both call `commitmentService.open()`; advisory dispatch + LLM correction creates orphan Commitments)
- All other types: advisory (`DispatchResult.advisories()` populated; WARN logged; dispatch proceeds)

Key additions:
- `MessageTypePolicy.advisory(Channel, MessageType) → String` default method
- `DispatchResult.advisories` field `@JsonInclude(NON_EMPTY)`
- `MessageService` Logger + validate()-then-advisory() calling sequence
- `ReactiveMessageService` AtomicReference advisory capture across worker-pool lambda boundaries
- Client-side `validate()` call removed from MCP tools (MessageService is single gate)
- ADR-0016 in `docs/adr/`; protocol PP-20260604-a7ad99 updated in garden
- 3 garden entries: `@Tool` returns record (not Map), AtomicReference Mutiny chain, MessageService missing Logger

**#266 closed:** Migration was already complete before session — just closed the issue.

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-16-mdp01-when-advisory-makes-orphans.md` |
| ADR | `docs/adr/0016-hybrid-channel-type-enforcement.md` |
| Spec | `docs/specs/2026-06-15-advisory-type-enforcement-design.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
