# CaseHub Qhorus — The Split That Fixed The Seam

**Date:** 2026-06-07
**Type:** phase-update

---

## What I was trying to achieve: eliminate a constraint violation triggered by cross-dtype ledger writes

The bug was filed as #253: when a consuming application writes any non-qhorus
`LedgerEntry` subclass to the same `subjectId` before dispatching a qhorus message,
the ledger write service assigns `sequenceNumber=1` — colliding with the domain
entry that already owns seq=1, and firing `IDX_LEDGER_ENTRY_SUBJECT_SEQ`. The
casehub-aml team hit it in production.

The companion issue was #252: after `resolveChannelAsync` resolves a `Channel`
entity, six service methods were performing a second `findByName()` lookup
internally. Pure redundancy, filed after the #237 rename work made the pattern
visible.

## What we believed going in: the fix would be a one-liner

The issue description was precise. `findLatestBySubjectId` queries
`FROM MessageLedgerEntry` instead of `FROM LedgerEntry`. Change one string,
done. I was wrong about the scope, though not wrong about the fix.

When Claude and I actually read the class, we found three violations of the same
pattern — `findLatestBySubjectId`, `findBySubjectId`, and `findEntryById` all
scoped to the qhorus subtype. The reactive mirror had the same three bugs plus
eight more (Panache's typed repository makes every query automatically dtype-scoped).
And there was an unsafe `(MessageLedgerEntry)` cast that would throw
`ClassCastException` if a domain entry ever appeared as a `causedByEntryId`.

That's not a one-liner. That's a structural problem.

## The naming-the-split session, and five rounds of review before a line of code

`MessageLedgerEntryRepository` had two incompatible contracts:
it implemented `LedgerEntryRepository` (which requires cross-dtype visibility)
while also serving as qhorus's specialised query layer (which requires dtype-scoped
queries). Every query added to the class defaulted to the qhorus pattern. The bugs
weren't accidents — the class design guaranteed them.

The fix was to split it: `LedgerEntryJpaRepository` for cross-dtype operations,
`MessageLedgerEntryRepository` for qhorus-scoped queries. I didn't want to just
fix the symptoms.

What surprised me was how long the design spec took to get right. We ran five
rounds of code review on the spec before writing a line of implementation code —
finding everything from CDI ambiguity risks to stub shared-state breakage to
missing `entryType` fields in test setup. I've done a lot of spec-then-implement
cycles, but I don't think I've had a spec go through five structured review passes.
By the end, the implementation was genuinely mechanical.

## What we built: two repos, two stubs, instanceof guards, and a clean protocol

We created `LedgerEntryJpaRepository` — all JPQL uses `FROM LedgerEntry`,
`em.find(LedgerEntry.class, id)` for primary-key lookups, plain `em.persist()`
for writes (sequence assignment stays in `LedgerWriteService` for now, tracked
in #256). The reactive counterpart follows the same pattern but uses
`repo.getSession()` for raw Mutiny JPQL instead of Panache's typed repo.

`MessageLedgerEntryRepository` lost `implements LedgerEntryRepository` and
`@Priority(10)`. It kept eleven qhorus-specific methods. Both write services
got two injections instead of one. The unsafe cast in `writeAttestation()` became
an `instanceof` pattern guard.

For tests, we used a shared-list pattern: both stubs receive the same
`List<LedgerEntry>` instance by constructor, so `save()` writes and
`findByMessageId()` reads from the same state. There's a note in the stub about
the inherited `em` field being null in CDI-free tests — future test authors will
hit that NPE without it.

The #252 work was clean by comparison. Six methods in each service changed from
`String name` to `UUID channelId`, twelve MCP call sites updated to pass `ch.id`.
The pattern was already established by `setTypeConstraints` — we just applied it
uniformly.

We captured a new protocol (PP-20260607-d83ba5): any `LedgerEntryRepository`
implementation must use `FROM LedgerEntry` in all JPQL. Updated the existing
channel-resolution protocol to reflect that service mutation methods now receive
`ch.id`, not `ch.name`.

## Where things stand

Both issues are merged. The `IDX_LEDGER_ENTRY_SUBJECT_SEQ` violation is fixed.
The double-lookup is gone. 1416 tests pass.

The longer-term work is tracked in #255 and #256: eventually `LedgerWriteService`
should delegate sequence assignment to `LedgerSequenceAllocator` in the library,
which would let us drop `LedgerEntryJpaRepository` entirely and use the library's
`JpaLedgerEntryRepository` directly. That's a separate session.
