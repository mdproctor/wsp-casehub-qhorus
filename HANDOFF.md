# CaseHub Qhorus — Session Handover
**Date:** 2026-06-13 — Large S/XS batch + A2A SSE streaming closed (#276, #275, #273, #250, #238, #247, #246, #242, #235, #234, #147)

---

## What Was Done This Session

**11 issues closed in one batch (issue-276-sxs-batch):**

- **#276** `QhorusInboundCurrentPrincipal`: `@DefaultBean` → plain `@ApplicationScoped` — fixes CDI ambiguity in consumer apps. Removed `quarkus.arc.exclude-types=MockCurrentPrincipal` from all 4 test configs.
- **#275** `CommitmentService.decline()` null guard removed; recording `Event<CommitmentDeclinedEvent>` wired in CDI-free tests.
- **#247/#246** `ChannelCreateRequest.allowedTypes/deniedTypes` changed from `String` to `Set<MessageType>`; `MessageType.serializeTypes()` added; `setTypeConstraints()` typed; MCP tools parse at boundary.
- **#250** `CommitmentService.extendDeadline()` + reactive mirror.
- **#235** `ReactiveMessageService` trust gate now calls `ObligorTrustPolicy.permits()` via `Infrastructure.getDefaultWorkerPool()` — custom policy beans honoured in reactive path.
- **#147** A2A SSE streaming: `GET /a2a/tasks/{id}/stream`, `Consumer<OutboundMessage>` registry in `A2AChannelBackend`, DECLINE→"cancelled" fix across all `A2ATaskState` paths, async `sink.close()` via `thenRun`.
- **#242, #234** — already implemented; closed with comments.
- **#273, #238** — docs/protocol; closed.

**ADRs:** 0013 (A2A lazy registration), 0014 (SSE consumer registry), 0015 (reactive trust gate worker-pool)

**Protocols:** sse-sink-async-close, a2a-decline-maps-to-cancelled, reactive-blocking-spi-worker-pool

**Garden:** 4 entries — two-@DefaultBean-ambiguity, sse-sink-close-async, fanout-pre-commit-timing, transactional-sse-void-commits

## Immediate Next Step

Run `/work` to pick up the next issue. Main is clean, all repos aligned.

## What's Left

- `#277` — SSE live-stream integration test (COMMAND → stream → DONE → completed event) · S · Med
- `#278` — SSE keepalive + server-side timeout for orphaned consumers in A2AChannelBackend · M · Med
- `parent#241` — sync casehub-qhorus deep-dive in parent docs (A2A SSE, Set<MessageType>, DECLINE→cancelled, #235) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #271 | allowedTypes/deniedTypes advisory enforcement (warn not hard-block) | M | Med | Informatory role concept gives theoretical grounding |
| casehub-ledger#126 | Full EVENT telemetry decoupling | M | Med | Unblocks content-bearing EVENTs; unowned backlog |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-13-mdp01-ten-issues-one-stream.md` |
| Garden: 4 new entries | GE-20260613-9e0a5b, c29bb8, 6527d0, a5983e |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
