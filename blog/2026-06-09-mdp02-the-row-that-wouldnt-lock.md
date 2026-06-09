---
layout: post
title: "The Row That Wouldn't Lock"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [h2, cdi, sequence-allocation, merkle, quarkus, concurrent]
---

The plan was tidy. Move sequence allocation out of `LedgerWriteService.record()` —
where it was a racy SELECT-then-persist — and into `save()`, where it would use
the same MERGE SQL that casehub-ledger's `LedgerSequenceAllocator` already used
internally. Eliminate the TOCTOU race, drop a cross-dtype query from the critical
path, unlock the library class.

The MERGE itself was straightforward. The problems started when we ran tests with
concurrent writers.

H2 doesn't serialise concurrent `MERGE INTO ... WHEN NOT MATCHED THEN INSERT`
statements the way PostgreSQL does. In PostgreSQL, the MERGE acquires a row-level
lock before inserting — the second concurrent transaction blocks until the first
commits. H2 has no equivalent. Two concurrent `REQUIRES_NEW` transactions can both
evaluate `WHEN NOT MATCHED` before either commits, both attempt to INSERT the same
`subject_id` primary key, and one of them explodes with a PK violation.

The naive fix — `@Transactional(REQUIRES_NEW)` on `save()`, combined with
`synchronized(this)` — doesn't work. The CDI `@Transactional` interceptor commits
the transaction after the method body returns, not inside it. `synchronized` releases
when the method body returns. There's a real race window between "lock released"
and "REQUIRES_NEW committed," and H2's MERGE can slip into it.

The fix that works: call the MERGE through a *separate CDI bean* that carries
`@Transactional(REQUIRES_NEW)`. The caller holds `synchronized(this)` across that
call. When the allocator's method returns, the CDI proxy has committed the REQUIRES_NEW
— the row is in the database — before the calling class's lock releases. T2 blocks
on the lock, acquires it after T1's commit, runs the MERGE, sees `WHEN MATCHED`,
increments. No PK violation.

`QhorusSequenceAllocator` is now that separate bean. `QhorusLedgerEntryRepository.save()`
holds `synchronized(this)` while calling it.

---

The second surprise was CDI. The original plan was to activate casehub-ledger's
`JpaLedgerEntryRepository` (which is `@Alternative`) via `quarkus.arc.selected-alternatives`
in `application.properties`. This reliably works in Quarkus application projects. In
a Quarkus *extension* — which is what qhorus is — it silently did nothing. CDI
validation failed with `UnsatisfiedResolutionException` regardless of whether the JAR
had a Jandex index, regardless of explicit `quarkus.index-dependency` config.

The CDI spec says `@Alternative` doesn't propagate to subclasses. So:

```java
@ApplicationScoped
class QhorusLedgerEntryRepository extends JpaLedgerEntryRepository {
    // inherits everything — save(), all query methods, all @Inject fields
    // NOT @Alternative → DEFAULT CDI bean
}
```

That's it. A non-`@Alternative` subclass is a DEFAULT bean that CDI discovers from
the extension's own class scan. No config required. Same pattern for the Merkle
frontier repository.

`LedgerEntryJpaRepository` — the qhorus-owned intermediate class that had been
accumulating TODOs since #253 — is now deleted.

---

The reactive path gained its first Merkle chain in the same session. The blocking
path (`JpaLedgerEntryRepository.save()`) had always computed a leaf hash and updated
the frontier — that's what #255's activation gave us. The reactive `save()` had
always been a plain `session.persist()`. We added the MERGE sequence, actorId
tokenisation before the leafHash (the canonical bytes include actorId; tokenising
after the digest would make blocking and reactive hashes diverge), and the frontier
update via `session.createMutationQuery()`.

---

The timeline fix (#262) turned out to have two separate problems. The blocking
`getChannelTimeline()` was doing one `findByMessageId()` per EVENT in the result
window — N+1, bounded by the 200-row page cap, but wrong. The reactive equivalent
wasn't doing any ledger lookup at all. EVENT messages in the reactive timeline all
showed null telemetry, silently, with no error. One `findByMessageIds(Collection<Long>)`
batch query and a pre-built map fixed both.

The thing about `LedgerSequenceAllocator` is that it almost works in H2. The SQL is
fine; the table semantics are fine; the locking just isn't there for concurrent
new-row inserts. That gap only shows up at test time if you have a barrier test with
concurrent writers — and then only after the MERGE is on the critical write path.
Before #256, `LedgerWriteService` did a SELECT before calling `save()`, so `save()`
was just `em.persist()`. The race existed there too, but it was silent: duplicate
sequence numbers instead of an exception. At least the exception made it diagnosable.
