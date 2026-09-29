# HANDOFF — casehub-qhorus

## Last Session (2026-09-29)

Triaged parent#515 items. Closed parent#480 (Pattern 2 migration already done — all 7 SPI interfaces have @McpDomain + @PlatformQuery/@PlatformMutation). Added qhorus-rest-spring to reactor (parent#495) — generator plugin now published, 8 Spring MVC controllers generated and compiling.

**Branch:** `issue-495-rest-spring-reactor` — 1 commit (pom.xml module addition)

**Pre-existing issues found:**
- Stale ledger-api in slot's .m2 — LedgerMerkleFrontierRepository missing. Fixed by installing ledger API to slot's local repo (`-Dmaven.repo.local=...`)
- Test compilation errors in compliance-report from stale ComplianceReport/LedgerDecisionRecord record signatures (upstream ledger changes). Not #495 scope.
- runtime-spring remains excluded from reactor — 3 compile errors in generated auto-config (A2ACore$TransactionRunner missing, MessageLedgerEntryRepository not in core, QhorusTracingConfigSpring→DeliveryConfig type mismatch). Needs core extraction work.

## Immediate Next Step

Close branch issue-495-rest-spring-reactor via work-end. Then file an issue for the runtime-spring compile errors.

## References

- parent#515 — Spring deployment completion epic
- parent#480 — Pattern 2 migration (closed — already done)
- parent#495 — Spring REST controllers (closed — rest-spring in reactor)
