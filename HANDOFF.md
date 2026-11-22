# HANDOFF — casehub-qhorus

## Last Session (2026-11-22)

Implemented and landed #528 (playbook protocol bridge), #529 (deadline timer), and #531 (PlaybookRunnerExecutor adapter). Platform#931 landed the `run(String, PlaybookRunConfig)` overload; the qhorus adapter wires it via `Instance<PlaybookRunnerCore>` with availability guard. Code review caught CDI wiring issue (`PlaybookRunnerCore` is not a CDI bean — changed to `Instance<>` pattern). All three issues closed, published.

## Queue State

#533 (CLAUDE.md update) is next. Queue: #533, #534, #535, #536, #537. #532 (ChannelCreateRequest integration) is deferred but no longer blocked — can be un-deferred now that #531 is done.

## Open Follow-ups

- #534 — REST endpoint should manage channel.protocols list on PUT/DELETE
- #535 — JpaProtocolDefinitionStore queries need tenancy filtering
- #536 — @QuarkusTest integration and recovery tests for playbook-bridge
- #537 — Use proper YAML parser for deadline metadata

## References

- `specs/issue-528-playbook-protocol-bridge/2026-11-20-playbook-protocol-bridge-design.md`
- `plans/2026-11-20-playbook-protocol-bridge.md`
- `blog/2026-11-20-mdp01-the-bridge-that-closes-the-loop.md`
