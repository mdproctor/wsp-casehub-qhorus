---
layout: post
title: "After the Handoff"
date: 2026-05-17
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [a2a, flyway, commitment-lifecycle]
---

Two bugs today. #151 looked like a two-line change — `DELEGATED` was mapping
to `"completed"` in the A2A task state API when it should map to `"working"`. A
delegated task is still in progress with the delegate; the API was telling external
orchestrators it was done.

I chose to pull the mapping logic out of both resource classes into a shared
`A2ATaskState` utility rather than just fixing the two lines in place — the
duplication was real and the extraction was clean. Bringing Claude in for the
implementation, we ran integration tests for the DELEGATED scenario: send a
QUERY, then a HANDOFF, check the task state. We expected the CommitmentStore path
to fire and return `fromCommitmentState(DELEGATED) → "working"`.

It doesn't. `CommitmentService.findByCorrelationId` returns the most recent active
commitment for a correlationId. After a HANDOFF, the parent commitment transitions
to DELEGATED (terminal) and a child OPEN commitment is created for the delegate —
same correlationId. `findByCorrelationId` returns the child OPEN, not the parent
DELEGATED. The CommitmentStore path fires for OPEN and returns `"submitted"`.

We fixed it with an OPEN guard:

```java
String state = (commitment != null && commitment.state != CommitmentState.OPEN)
        ? A2ATaskState.fromCommitmentState(commitment.state)
        : A2ATaskState.fromMessageHistory(messages);
```

`fromMessageHistory` sees the HANDOFF message at the tail of the history and
correctly returns `"working"`. There was a second discovery in the process:
HANDOFF messages require a non-null `target` argument — all other message types
accept null. Claude hit that in the integration tests, the error message gave
nothing useful, and we found it by inspection.

`A2ATaskState.fromCommitmentState` still has the `DELEGATED → "working"` case.
It's correct, covered by unit tests, and matters if the store implementation
ever changes. But through the current live HTTP path, that branch is unreachable.
One of those things that's right to keep and worth knowing.

#142 was simpler: qhorus ships `V2__add_message_target.sql` in `db/migration/`,
the same classpath location casehub-work uses. When both JARs are on the
classpath, Flyway finds two scripts at version 2 and refuses to start. The fix
was scoping qhorus migrations to `db/migration/qhorus/` and updating
`quarkus.flyway.qhorus.locations=classpath:db/migration/qhorus`. An extension
that already runs on an isolated named datasource should scope its migration
files the same way.

Claude moved the files with shell `mv` and staged only the new directory. The
original V1–V9 files were still tracked — unstaged deletions sitting in the
working tree, invisible to the commit. The build artifact would have shipped
both copies and the classpath conflict would have survived untouched. Claude
caught it in the code review and fixed it with a follow-up commit.
