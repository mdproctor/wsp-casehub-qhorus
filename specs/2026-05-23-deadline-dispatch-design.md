# Deadline Dispatch Design

**Branch:** issue-192-deadline-dispatch  
**Issue:** casehubio/qhorus#192  
**Date:** 2026-05-23  

---

## Problem

`Message.deadline` exists as a column since V1, and `CommitmentService.open()` accepts it as a
parameter — but it is always `null`. The field is never populated through any code path, making
Layer 3 temporal accountability (Watchdog, deadline enforcement) silently broken for all
COMMAND/QUERY obligations.

Two root causes:

1. **`MessageDispatch` has no `deadline` field.** `MessageService.dispatch()` never sets
   `message.deadline` — every COMMAND and QUERY enters the commitment store without a deadline.
2. **`ReactiveMessageService.send()` has no `deadline` parameter.** Callers cannot propagate a
   deadline through the reactive path. This is the immediate blocker for claudony#135 (which adds
   `deadline` as a first-class SPI parameter to `postToChannel()`).

Additionally, `ReactiveMessageService.send()` uses flat positional parameters rather than
`MessageDispatch`, diverging from the blocking service and making future maintenance harder.
`QhorusDashboardService.sendHumanMessage()` performs a manual paused check before calling
`send()`, duplicating enforcement in a caller — a violation of PP-20260523-a08b97.

---

## Design

### 1. `MessageDispatch` — add `deadline` field (api module)

Add `Instant deadline` as a nullable record component. Add `.deadline(Instant v)` builder method.
No validation: deadline is optional for all 9 message types and meaningful only for COMMAND and
QUERY (the commitment-opening types). The builder field defaults to `null`; all existing
`MessageDispatch.Builder` call sites compile unchanged.

`build()` passes `deadline` to the record constructor — the only line that changes in the builder.

### 2. `MessageService.dispatch()` — set `message.deadline` (blocking path fix)

Add one line immediately before `messageStore.put(message)`:

```java
message.deadline = dispatch.deadline();
```

This makes `deadline` flow through to `commitmentService.open()` for the first time in the
blocking path. No other changes.

### 3. `ReactiveMessageService` — replace `send()` with `dispatch(MessageDispatch)`

Replace the 9-flat-param `send(...)` returning `Uni<Message>` with
`dispatch(MessageDispatch dispatch)` returning `Uni<DispatchResult>`.

**Rationale:** `ReactiveMessageService` mirrors `MessageService` (reactive-service-build-gating
protocol). Using `MessageDispatch` ensures every caller is explicit about every field and prevents
positional drift. Returning `Uni<DispatchResult>` matches the blocking service's return type and
gives callers ledger metadata when it becomes available after #193 adds ledger writes.

**Implementation inside `Panache.withTransaction("qhorus", ...)`:**

1. Paused check — fetch channel, throw `IllegalStateException` if `ch.paused`. This check moves
   here from `QhorusDashboardService.sendHumanMessage()`, where it is currently a caller-level
   enforcement duplicate.
2. Construct `Message` from `dispatch.*` fields, including `message.deadline = dispatch.deadline()`.
3. `messageStore.put(message)`.
4. Observer dispatch via `MessageObserverDispatcher`.
5. Commitment state machine (unchanged from current `send()`).
6. Reply-count update (unchanged).
7. `ch.lastActivityAt = Instant.now()`.
8. Construct and return `DispatchResult` — ledger fields (`ledgerEntryId`, `subjectId`,
   `causedByEntryId`) are `null` until #193 adds ledger writes to the reactive path.
   `parentReplyCount` is computed from the reply-count update above.

All other enforcement (writer ACL, rate limiting, type policy, LAST_WRITE, fanOut) is deferred to
issue #193 (enforcement parity).

### 4. Caller updates (mechanical)

**`QhorusDashboardService.sendHumanMessage()`**

- Remove manual paused check (now inside `ReactiveMessageService.dispatch()`).
- Build `MessageDispatch` with `actorType = HUMAN`, no correlationId, no deadline.
- Call `messageService.dispatch(dispatch)`.
- Map `DispatchResult` fields to `HumanMessageResult`. `parentReplyCount` comes from
  `DispatchResult.parentReplyCount()` — more accurate than the hardcoded `0` it currently returns.

**`ClaudonyReactiveCaseChannelProvider.postToChannel()`** (claudony repo)

- Build `MessageDispatch` from SPI parameters. `deadline` is `null` now; it will carry a real
  value once claudony#135 updates the `ReactiveCaseChannelProvider` SPI to add `deadline` as a
  first-class parameter.
- The `extractCorrelationId(content)` hack remains until claudony#135 also adds `correlationId`
  to the SPI.

**`ReactiveMessageServiceTest`**

- Update to build `MessageDispatch` objects rather than calling `send()` with flat params.

**All other `MessageDispatch.Builder` call sites in qhorus**

- No changes required — `deadline` defaults to `null` in the builder.

---

## Scope boundaries

| In scope | Out of scope |
|----------|-------------|
| `MessageDispatch.deadline` field | Enforcing deadline constraints at dispatch time |
| Blocking path: `message.deadline = dispatch.deadline()` | Watchdog / expiry evaluation changes |
| Reactive path: `dispatch(MessageDispatch)` → `Uni<DispatchResult>` | Reactive writer ACL, rate limiting, type policy (#193) |
| Paused check in reactive `dispatch()` | Reactive fanOut (#193) |
| Caller updates (dashboard, Claudony, tests) | Reactive ledger write (#193) |
| | Populating deadline from channel config default |

---

## Protocol coherence

- **PP-20260523-a08b97** (enforcement gate): paused check moves from caller
  (`sendHumanMessage`) into `ReactiveMessageService.dispatch()`. ✅
- **PP-20260522-3dca14** (builder validation): no change to `build()` validation matrix.
  `deadline` has no cross-field constraints. ✅
- **reactive-service-build-gating**: `dispatch(MessageDispatch)` mirrors `dispatch(MessageDispatch)`
  in the blocking service. ✅
- **PLATFORM.md boundary rule** (no caller-level enforcement): `sendHumanMessage()` paused check
  violation resolved. ✅

---

## Files changed

**qhorus repo:**
- `api/src/.../message/MessageDispatch.java` — add `deadline`, builder method, update `build()`
- `runtime/src/.../message/MessageService.java` — one line: `message.deadline = dispatch.deadline()`
- `runtime/src/.../message/ReactiveMessageService.java` — replace `send()` with `dispatch(MessageDispatch)`
- `runtime/src/.../dashboard/QhorusDashboardService.java` — remove paused check, use `MessageDispatch`
- `runtime/src/test/.../service/ReactiveMessageServiceTest.java` — update to `MessageDispatch.builder()`

**claudony repo:**
- `casehub/src/.../ClaudonyReactiveCaseChannelProvider.java` — build `MessageDispatch` in `postToChannel()`
- `casehub/src/test/.../ClaudonyReactiveCaseChannelProviderTest.java` — update test setup
