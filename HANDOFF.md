# CaseHub Qhorus — Session Handover
**Date:** 2026-07-06 — #322 closed (testing CDI fix), #325 closed (review cleanup).

---

## Immediate Next Step

Main is clean. Both remotes at `4c177454`. Issues #322, #325 merged and closed.

Cross-repo follow-up issues filed and pending:
- life#58 — remove persistence-memory Maven exclusion workaround (no longer needed)
- claudony#169 — update OutboundMessage construction in 2 test files (correlationId UUID→String)
- drafthouse#102 — remove redundant .toString() on OutboundMessage.correlationId()

Prior session follow-ups still open:
- parent#353 — update PLATFORM.md for cross-node delivery
- claudony#168 — migrate FleetMessageRelayObserver to postgres-broadcaster
- openclaw#62 — 1 test file persistence-memory import change

## What Was Done This Session

**OutboundMessage.correlationId UUID→String (#325):** Fixed the type mismatch at the root — 3 duplicated parseCorrelationUuid methods deleted, latent IllegalArgumentException in DeliveryBatchExecutor fixed. Cascade through A2A (case-normalized String keys), Slack (SlackThreadCacheId + V27 migration), and ~15 test files. Design-reviewed: A2A case-sensitivity regression and broadcaster connection leak caught and fixed.

**Testing CDI fix (#322):** Changed persistence-memory from compile to test scope in testing/pom.xml. Added direct test-scope deps to 5 sibling modules that relied on the transitive path.

**Broadcaster exponential backoff (#325):** Replaced fixed 5s retry with 1s→60s capped backoff. Added AtomicBoolean reconnection guard and shutdown leak prevention.

**Test sleeps→Awaitility (#325):** 4 Thread.sleep(100) replaced with Awaitility polling in ChannelGatewayDeliverRemoteTest.

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-06-testing-cdi-fix-and-review-cleanup.md` |
| Garden entry | `GE-20260706-16293f` (Maven transitive scope narrowing gotcha) |
| Cross-repo issues | life#58, claudony#169, drafthouse#102 |
| Previous session | `git show HEAD~1:HANDOFF.md` |
