# A2A Task State Mapping — DELEGATED and HANDOFF Fix

**Issue:** #151  
**Date:** 2026-05-17  
**Status:** Approved

---

## Problem

`toA2AState(CommitmentState.DELEGATED)` returns `"completed"` in both `A2AResource` and
`ReactiveA2AResource`. DELEGATED means the obligation was handed off to a delegate agent via
a HANDOFF message — the task is still in progress, not finished. An external A2A orchestrator
receiving `status.state: "completed"` would terminate polling prematurely.

A secondary inconsistency: `deriveState()` (the CommitmentStore-absent fallback) maps `HANDOFF`
to `"submitted"` via the default branch, while the CommitmentStore path maps `DELEGATED` to
`"completed"`. Two callers with the same message history produce different answers.

Both `toA2AState` and `deriveState` are private statics duplicated identically between the two
resource classes — a future state-mapping change requires four edits.

---

## Design

### New class: `A2ATaskState`

Package-private, no CDI, pure static methods. Lives in
`io.casehub.qhorus.runtime.api`.

```java
static String fromCommitmentState(CommitmentState state)
static String fromMessageHistory(List<Message> messages)
```

`fromCommitmentState` replaces `toA2AState`. `fromMessageHistory` replaces `deriveState`.
Both resource classes delegate to `A2ATaskState`; their existing private methods become
one-line call-throughs so call sites are unchanged.

### Corrected state tables

**CommitmentState → A2A state:**

| CommitmentState | A2A state   | Reason                                      |
|-----------------|-------------|---------------------------------------------|
| `OPEN`          | `submitted` | Received, debtor not yet acknowledged       |
| `ACKNOWLEDGED`  | `working`   | Debtor actively processing                  |
| `FULFILLED`     | `completed` | RESPONSE or DONE received                   |
| `DELEGATED`     | `working`   | **Fixed** — HANDOFF in flight with delegate |
| `DECLINED`      | `failed`    | Debtor refused                              |
| `FAILED`        | `failed`    | Debtor attempted but errored                |
| `EXPIRED`       | `failed`    | Deadline exceeded                           |

**Last MessageType → A2A state (fallback path):**

| Last MessageType              | A2A state   | Reason                              |
|-------------------------------|-------------|-------------------------------------|
| `QUERY`, `COMMAND`, all other | `submitted` | Default                             |
| `STATUS`                      | `working`   | Agent is processing                 |
| `HANDOFF`                     | `working`   | **Fixed** — delegate in progress    |
| `RESPONSE`, `DONE`            | `completed` |                                     |
| `FAILURE`, `DECLINE`          | `failed`    |                                     |

---

## Testing

**`A2ATaskStateTest`** — pure Java unit test, no Quarkus. Exercises every
`CommitmentState` value via `fromCommitmentState` and every relevant `MessageType`
via `fromMessageHistory`. Canonical spec for the state table.

**`A2AGetTaskTest` additions** (integration, existing `@QuarkusTest`):

- `taskWithDelegatedState_isWorking` — sends QUERY via A2A, sends HANDOFF to
  transition Commitment to DELEGATED, asserts `GET /a2a/tasks/{id}` returns `"working"`.
- `taskWithHandoffMessageIsWorking` — inserts HANDOFF on a correlationId with no
  prior QUERY (CommitmentStore absent, fallback path used), asserts `"working"`.
