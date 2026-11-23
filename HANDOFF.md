# HANDOFF — casehub-qhorus

## Last Session (2026-11-22)

Implemented and landed #528 (playbook protocol bridge) and #529 (deadline timer). 7 tasks across 5 batches: ProtocolResolver SPI in api/, playbook-bridge module scaffold with ProtocolDefinitionStore (JPA + migration V62), PlaybookProtocolOutput validator, PlaybookProtocolResolver with CompilationCache and PlaybookExecutor abstraction, ProtocolDeadlineScheduler with Consumer-based fire action, PlaybookProtocolLifecycleHandler, REST endpoint with CDI producers. Code review caught 4 CDI wiring issues (scheduler missing @ApplicationScoped, registry not injected, unused ChannelManager, tenancy filtering gap) — fixed 2 inline, filed 4 issues (#534-#537). Squashed 9→6 commits. Landed on main, published.

## Queue State

#532 is active but depends on deferred #531 (platform prerequisite). #533 (CLAUDE.md update) is the next actionable item. Queue: #532, #533, #534, #535, #536, #537.

## Cross-Module

- casehubio/qhorus#531 — `PlaybookRunnerCore.run(String, PlaybookRunConfig)` overload needed in casehub-platform. Deferred. Blocks #532 and final PlaybookExecutor wiring.

## Open Follow-ups

- #534 — REST endpoint should manage channel.protocols list on PUT/DELETE
- #535 — JpaProtocolDefinitionStore queries need tenancy filtering
- #536 — @QuarkusTest integration and recovery tests for playbook-bridge
- #537 — Use proper YAML parser for deadline metadata

## References

- `specs/issue-528-playbook-protocol-bridge/2026-11-20-playbook-protocol-bridge-design.md`
- `plans/2026-11-20-playbook-protocol-bridge.md`
- `blog/2026-11-20-mdp01-the-bridge-that-closes-the-loop.md`
