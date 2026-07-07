# CaseHub Qhorus — Session Handover
*Updated: parent#353, openclaw#62 closed — removed from backlog.*
**Date:** 2026-07-06 — #322, #325, #326 closed; #323 closed (already fixed by #319).

---

## Immediate Next Step

Main is clean. Both remotes at `72fd4a3f`. Four issues closed this session.

Cross-repo follow-up issues filed and pending:
- life#58 — remove persistence-memory Maven exclusion workaround
- claudony#169 — update OutboundMessage construction in 2 test files (correlationId UUID→String)
- drafthouse#102 — remove redundant .toString() on OutboundMessage.correlationId()

Prior session follow-ups still open:
- claudony#168 — migrate FleetMessageRelayObserver to postgres-broadcaster

## What Was Done This Session

**OutboundMessage.correlationId UUID→String (#325):** Fixed type mismatch at the root — 3 duplicated parseCorrelationUuid methods deleted, latent IllegalArgumentException in DeliveryBatchExecutor fixed. A2A SSE case-normalization for String keys. SlackThreadCacheId migrated via Flyway V27. Design-reviewed (2 rounds, 13 issues, all resolved).

**Testing CDI fix (#322):** persistence-memory compile→test scope in testing/pom.xml. 5 sibling modules given direct deps.

**Broadcaster exponential backoff (#325):** 1s→60s capped backoff, AtomicBoolean reconnection guard, shutdown leak prevention, stale closeHandler guard.

**Test sleeps→Awaitility (#325):** 4 Thread.sleep(100) replaced in ChannelGatewayDeliverRemoteTest.

**Binding-first putIfAbsent (#326):** Eliminated findOrCreateWithBinding race — binding is the coordination primitive, created before the channel. No orphan channels, no TOCTOU. Catch-and-retry backup for belt-and-suspenders.

**allowedWriters NPE (#323):** Investigated, found already fixed by #319 compact constructor normalization. Claudony needs SNAPSHOT rebuild. Closed.

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-06-testing-cdi-fix-and-review-cleanup.md` |
| Garden entries | `GE-20260706-16293f` (Maven transitive scope), `GE-20260706-b56877` (Collections.synchronizedSet) |
| Cross-repo issues | life#58, claudony#169, drafthouse#102 |
| Previous session | `git show HEAD~1:HANDOFF.md` |
