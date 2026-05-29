---
layout: post
title: "Five small fixes and three wrong assumptions"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [qhorus, trust-gate, watchdog, a2a, cleanup]
---

The issues were labelled XS and S. Five of them, all supposedly straightforward.
I worked through each one, and three of them had a wrong assumption embedded in
the design I'd sketched before touching code.

**The first was the trust gate target format.** The goal was simple: before
accepting a COMMAND, check the obligor's trust score. `TrustGateService.meetsThreshold(actorId, minTrust)` takes a plain actor identifier. The obvious implementation
calls it with `dispatch.target()`. The problem is that `dispatch.target()` isn't
always a plain actor ID — it can be `"role:specialist"` or `"capability:writer"`.
Those are role/capability prefixes used for broadcast addressing. No record in
`ActorTrustScoreRepository` will ever match `"role:specialist"` as an actorId, so
`meetsThreshold` returns `false` and every COMMAND to a role target gets rejected
silently when the gate is on.

The fix is a colon guard — skip the trust check if the target contains a colon.
Not elegant, but it matches the platform's existing naming convention: only role
and capability prefixes use colons; plain instance IDs don't. It works. It's documented.
And #213 tracks the eventual cleanup — an `ObligorTrustPolicy` SPI that resolves
targets properly rather than using a string heuristic.

**The second was BARRIER_STUCK test design.** I wrote the positive test case using
EVENT messages as barrier contributions: agent-x posts an EVENT, agent-y doesn't,
BARRIER_STUCK should fire for agent-y. It wouldn't have worked.
`JpaMessageStore.distinctSendersByChannel(channelId, MessageType.EVENT)` uses EVENT
as the *excluded* type — it returns senders of non-EVENT messages. Agents contribute
to a barrier with STATUS or COMMAND messages, not EVENTs. The positive test would
have fired for the wrong reason (both agents missing rather than just one), and
the negative test (both post EVENTs → no alert) would have failed entirely.

Caught before the tests were written. The correct setup uses `MessageType.STATUS`
for contributions. Obvious in retrospect; not obvious from the interface signature.

**The third was the ledger datasource in test profiles.** The trust gate integration
tests need `ActorTrustScore` rows in the database. Straightforward enough — until
the first test run. Claude came back with `UnknownNamedQueryException` on every
`ActorTrustScoreRepository` call.

The cause: `QuarkusTestProfile.getConfigOverrides()` causes a Quarkus context restart
when the profile class differs from the previous test class. On restart,
`application.properties` is not re-read — only the overrides are applied.
`casehub.ledger.datasource=qhorus` normally sits in `application.properties` and
routes `@LedgerPersistenceUnit` to the named qhorus persistence unit. Without it
in `getConfigOverrides()`, the restarted context routes `@LedgerPersistenceUnit`
to the default PU, where the ledger's named queries aren't registered.

The property itself is undocumented. We found it only by reading
`LedgerEntityManagerProducer.java`. It's now in the casehub protocols as a standing
rule for any test profile that restarts the context and touches ledger JPA.

---

The other two were genuinely small. The JSON error body injection in A2AResource
was string concatenation where a structured record should go — `ErrorResponse`
record, two minutes, done. The `fromMessageHistory` ordering bug was a real problem
but a clean fix: replace `messages.getLast()` (last-wins) with a max-priority
reduce over all messages, so a QUERY arriving after a DONE no longer regresses
state to "submitted."

And the dedup guard in `registerBackend()` got something I'd missed in the original
design: the check-then-add wasn't inside `synchronized(entries)`. Two threads could
both pass the backendId check and both add, defeating the dedup entirely. Now the
synchronized block covers all three branches.

The batch is closed. All 1236 tests pass.
