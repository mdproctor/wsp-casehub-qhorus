# Design: Org-Aware Task Assignment

**Issue:** casehubio/qhorus#494
**Branch:** issue-494-org-aware-task-assignment
**Date:** 2026-10-30

## Summary

When `RoutingBridge` resolves a `role:X` target to a specific agent, prefer agents in the same organisational unit as the requesting agent before widening to the full registry.

## Mechanism

In `RoutingBridge.resolve()`, after `agentRegistry.find(byCapability)` returns all matching agents:

1. Call `orgAwareDispatch.teamMembers(dispatch.sender(), tenancyId)` to get same-unit peer IDs
2. Filter the capability matches to only those whose `agentId` is in the team members set
3. If filtered list is non-empty, pass it to `agentSelector.select()`
4. If filtered list is empty (no team member has the capability), fall back to the full unfiltered match list

This mirrors the existing pattern — trust threshold and capacity checks already happen after selection. Org preference happens before selection, narrowing the candidate pool.

## Changes

### RoutingBridge (runtime-core)

- Add `OrgAwareDispatch` as a constructor parameter (nullable — no-op when eidos org module not on classpath)
- In `resolve()`, between `agentRegistry.find()` and `agentSelector.select()`, insert the filter-then-fallback logic
- In `diagnose()`, include org membership info in `CandidateInfo` for observability

### RuntimeBeans (runtime)

- Add `Instance<OrgAwareDispatch>` to the `routingBridge()` CDI producer, resolve-or-null pattern (same as `AgentRegistry`, `AgentSelector`)

### RuntimeAutoConfiguration (runtime-spring)

- Add `Optional<OrgAwareDispatch>` to the Spring `RoutingBridge` bean factory method

### RoutingBridgeTest (runtime)

- Test: team members preferred when available
- Test: fallback to full list when no team member qualifies
- Test: null `OrgAwareDispatch` (eidos not on classpath) — current behaviour unchanged
- Test: empty `teamMembers()` return — current behaviour unchanged
- Test: sender not registered in org — `teamMembers()` returns empty, fallback works

### Ledger metadata

`RoutingOutcome` already records `strategyName`. When org filtering applies, set to `"eidos-org-preferred"` (vs current `"eidos-simple"`). No migration needed — `strategyName` is a nullable VARCHAR on `MessageLedgerEntry`.

## Non-goals

- No `AgentSelector` API changes
- No `SelectionContext` changes
- No new config properties — org preference is always active when `OrgAwareDispatch` is available
- No new Flyway migration

## References

- `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/RoutingBridge.java` — integration point
- `runtime/src/main/java/io/casehub/qhorus/runtime/RuntimeBeans.java:282-294` — CDI producer
- `io.casehub.eidos.api.OrgAwareDispatch` — eidos API contract
- `io.casehub.eidos.api.AgentSelector` — unchanged, receives narrowed candidate list
- `runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingBridgeTest.java` — existing test patterns
