# Session Handover — 2026-10-08

## What happened

Fixed three cluster module bugs (#485, #486, #487) discovered during #484 E2E testing. All landed on main.

- **#485**: `HeartbeatService.tick()` skipped DEAD peers entirely — added reduced-rate probing every 5th tick
- **#486 + #487 (shared root cause)**: `RelayProducer` produced `WriteRoutingDecorator` as `MessageDispatcher` alternative, but `ChannelCore` injects `ConsumerMessaging` — bypassed quorum enforcement and cross-node routing. Fix: `RoutingConsumerMessaging` wraps the decorator for `ConsumerMessaging` injection. Added `QuorumViolationExceptionMapper` (HTTP 503).

## Decisions

- #486 and #487 combined in one commit — same root cause (CDI type mismatch)
- Created follow-up epic #488 (cluster hardening: cross-node E2E test, split-brain safety, cache wiring)

## Next action

Start #488 — begin with the cross-node dispatch E2E test (XS/Low, validates #486/#487 fix end-to-end).

## References

| Artifact | Location |
|----------|----------|
| Project commits | `da22128d`, `bc905d20`, `49d3f4ad` on main |
| Follow-up epic | casehubio/qhorus#488 |

## Project state

- **Both repos on main**, branch `issue-485-cluster-bug-fixes` closed and stamped
- Build: green (full build, 101 cluster tests pass)
