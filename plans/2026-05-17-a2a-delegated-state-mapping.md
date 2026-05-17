# A2A DELEGATED/HANDOFF State Mapping Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `DELEGATED` → `"working"` and `HANDOFF` → `"working"` in A2A task state derivation, and eliminate the duplicated mapping logic between `A2AResource` and `ReactiveA2AResource`.

**Architecture:** Extract a new package-private `A2ATaskState` class with two static methods (`fromCommitmentState`, `fromMessageHistory`). Both resource classes replace their identical private statics with direct calls to `A2ATaskState`. Tests: pure-Java unit test covers all enum/type values; two new integration tests in the existing `A2AGetTaskTest` cover the live paths.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, RestAssured. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

**Issue:** #151

---

## File Map

| Action | Path |
|--------|------|
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java` |
| Create | `runtime/src/test/java/io/casehub/qhorus/runtime/api/A2ATaskStateTest.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java` |
| Modify | `runtime/src/test/java/io/casehub/qhorus/api/A2AGetTaskTest.java` |

---

## Task 1: Write failing unit tests for `fromCommitmentState`

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/api/A2ATaskStateTest.java`

- [ ] **Step 1.1: Create the test file**

```java
package io.casehub.qhorus.runtime.api;

import static org.junit.jupiter.api.Assertions.assertEquals;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageType;

class A2ATaskStateTest {

    // -----------------------------------------------------------------------
    // fromCommitmentState
    // -----------------------------------------------------------------------

    @Test
    void openIsSubmitted() {
        assertEquals("submitted", A2ATaskState.fromCommitmentState(CommitmentState.OPEN));
    }

    @Test
    void acknowledgedIsWorking() {
        assertEquals("working", A2ATaskState.fromCommitmentState(CommitmentState.ACKNOWLEDGED));
    }

    @Test
    void fulfilledIsCompleted() {
        assertEquals("completed", A2ATaskState.fromCommitmentState(CommitmentState.FULFILLED));
    }

    @Test
    void delegatedIsWorking() {
        assertEquals("working", A2ATaskState.fromCommitmentState(CommitmentState.DELEGATED));
    }

    @Test
    void declinedIsFailed() {
        assertEquals("failed", A2ATaskState.fromCommitmentState(CommitmentState.DECLINED));
    }

    @Test
    void failedIsFailed() {
        assertEquals("failed", A2ATaskState.fromCommitmentState(CommitmentState.FAILED));
    }

    @Test
    void expiredIsFailed() {
        assertEquals("failed", A2ATaskState.fromCommitmentState(CommitmentState.EXPIRED));
    }
}
```

- [ ] **Step 1.2: Run — expect compilation failure (class does not exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2ATaskStateTest
```

Expected: `COMPILATION ERROR — cannot find symbol: class A2ATaskState`

---

## Task 2: Create `A2ATaskState` with `fromCommitmentState`

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java`

- [ ] **Step 2.1: Create the class with `fromCommitmentState` only**

```java
package io.casehub.qhorus.runtime.api;

import java.util.List;

import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageType;

class A2ATaskState {

    static String fromCommitmentState(CommitmentState state) {
        return switch (state) {
            case FULFILLED -> "completed";
            case DELEGATED, ACKNOWLEDGED -> "working";
            case FAILED, DECLINED, EXPIRED -> "failed";
            case OPEN -> "submitted";
        };
    }
}
```

- [ ] **Step 2.2: Run — expect all 7 fromCommitmentState tests to pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2ATaskStateTest
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`

---

## Task 3: Add failing unit tests for `fromMessageHistory`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/api/A2ATaskStateTest.java`

- [ ] **Step 3.1: Append the `fromMessageHistory` tests to the test class (before the closing `}`)**

```java
    // -----------------------------------------------------------------------
    // fromMessageHistory
    // -----------------------------------------------------------------------

    @Test
    void emptyHistoryIsSubmitted() {
        assertEquals("submitted", A2ATaskState.fromMessageHistory(List.of()));
    }

    @Test
    void lastQueryIsSubmitted() {
        assertEquals("submitted", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.QUERY))));
    }

    @Test
    void lastCommandIsSubmitted() {
        assertEquals("submitted", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.COMMAND))));
    }

    @Test
    void lastEventIsSubmitted() {
        assertEquals("submitted", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.EVENT))));
    }

    @Test
    void lastStatusIsWorking() {
        assertEquals("working", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.STATUS))));
    }

    @Test
    void lastHandoffIsWorking() {
        assertEquals("working", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.HANDOFF))));
    }

    @Test
    void lastResponseIsCompleted() {
        assertEquals("completed", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.RESPONSE))));
    }

    @Test
    void lastDoneIsCompleted() {
        assertEquals("completed", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.DONE))));
    }

    @Test
    void lastFailureIsFailed() {
        assertEquals("failed", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.FAILURE))));
    }

    @Test
    void lastDeclineIsFailed() {
        assertEquals("failed", A2ATaskState.fromMessageHistory(List.of(msg(MessageType.DECLINE))));
    }

    @Test
    void lastMessageDeterminesState() {
        // earlier messages don't matter — only the last one
        assertEquals("working", A2ATaskState.fromMessageHistory(
                List.of(msg(MessageType.QUERY), msg(MessageType.STATUS))));
    }

    private static Message msg(MessageType type) {
        Message m = new Message();
        m.messageType = type;
        return m;
    }
```

- [ ] **Step 3.2: Run — expect `fromMessageHistory` tests to fail (method not defined yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2ATaskStateTest
```

Expected: `COMPILATION ERROR — cannot find symbol: method fromMessageHistory`

---

## Task 4: Implement `fromMessageHistory` in `A2ATaskState`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java`

- [ ] **Step 4.1: Add `fromMessageHistory` to `A2ATaskState`**

The complete file after the addition:

```java
package io.casehub.qhorus.runtime.api;

import java.util.List;

import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageType;

class A2ATaskState {

    static String fromCommitmentState(CommitmentState state) {
        return switch (state) {
            case FULFILLED -> "completed";
            case DELEGATED, ACKNOWLEDGED -> "working";
            case FAILED, DECLINED, EXPIRED -> "failed";
            case OPEN -> "submitted";
        };
    }

    static String fromMessageHistory(List<Message> messages) {
        MessageType lastType = null;
        for (Message m : messages) {
            lastType = m.messageType;
        }
        if (lastType == null) return "submitted";
        return switch (lastType) {
            case RESPONSE, DONE -> "completed";
            case FAILURE, DECLINE -> "failed";
            case STATUS, HANDOFF -> "working";
            default -> "submitted";
        };
    }
}
```

- [ ] **Step 4.2: Run — expect all 18 unit tests to pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2ATaskStateTest
```

Expected: `Tests run: 18, Failures: 0, Errors: 0`

---

## Task 5: Write failing integration tests for DELEGATED and HANDOFF

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/api/A2AGetTaskTest.java`

These tests encode the correct expected state. They will FAIL against the current resource code (which still has the old private statics returning `"completed"` for DELEGATED).

- [ ] **Step 5.1: Add two tests to `A2AGetTaskTest` in the `CommitmentStore-based state` section**

```java
    @Test
    void taskWithDelegatedState_isWorking() {
        tools.createChannel("a2a-gt-del-1", "Test", "APPEND", null, null, null, null, null, null);
        String taskId = UUID.randomUUID().toString();

        // QUERY via A2A creates an OPEN commitment
        sendA2A("a2a-gt-del-1", "user", "please delegate this", taskId);

        // Agent sends HANDOFF — commitment transitions to DELEGATED
        tools.sendMessage("a2a-gt-del-1", "agent", "handoff", "delegating to specialist", taskId,
                null, null, null, null);

        given()
                .when().get(TASKS_PATH + taskId)
                .then()
                .statusCode(200)
                .body("status.state", equalTo("working"));
    }

    @Test
    void taskWithHandoffMessageIsWorking() {
        tools.createChannel("a2a-gt-del-2", "Test", "APPEND", null, null, null, null, null, null);
        String taskId = UUID.randomUUID().toString();

        // HANDOFF sent directly without a prior QUERY — no commitment created.
        // If CommitmentService gracefully skips with no matching commitment,
        // deriveState() is invoked → HANDOFF → "working".
        // If CommitmentService creates/transitions a commitment, toA2AState(DELEGATED)
        // is invoked → "working". Either path is correct after the fix.
        tools.sendMessage("a2a-gt-del-2", "agent", "handoff", "delegated", taskId,
                null, null, null, null);

        given()
                .when().get(TASKS_PATH + taskId)
                .then()
                .statusCode(200)
                .body("status.state", equalTo("working"));
    }
```

- [ ] **Step 5.2: Run — expect the two new tests to FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2AGetTaskTest
```

Expected: `taskWithDelegatedState_isWorking` FAILS with `expected: "working" but was: "completed"`

---

## Task 6: Refactor `A2AResource` and `ReactiveA2AResource` to use `A2ATaskState`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java`

- [ ] **Step 6.1: Replace private statics in `A2AResource`**

Replace the `toA2AState` and `deriveState` methods entirely. Also update line ~156 to call `A2ATaskState` directly:

Change the state derivation line (currently ~156):
```java
// before
String state = (commitment != null) ? toA2AState(commitment.state) : deriveState(messages);
```
```java
// after
String state = (commitment != null)
        ? A2ATaskState.fromCommitmentState(commitment.state)
        : A2ATaskState.fromMessageHistory(messages);
```

Remove the two private static methods (`toA2AState` and `deriveState`) entirely.

- [ ] **Step 6.2: Replace private statics in `ReactiveA2AResource`**

Apply the same change at the equivalent state derivation line (~161):
```java
// before
String state = (commitment != null) ? toA2AState(commitment.state) : deriveState(messages);
```
```java
// after
String state = (commitment != null)
        ? A2ATaskState.fromCommitmentState(commitment.state)
        : A2ATaskState.fromMessageHistory(messages);
```

Remove the two private static methods (`toA2AState` and `deriveState`) from `ReactiveA2AResource` entirely.

- [ ] **Step 6.3: Run the full `A2AGetTaskTest` — all tests including the two new ones must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2AGetTaskTest
```

Expected: all tests pass, including `taskWithDelegatedState_isWorking` and `taskWithHandoffMessageIsWorking`

- [ ] **Step 6.4: Run the full unit test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2ATaskStateTest
```

Expected: `Tests run: 18, Failures: 0, Errors: 0`

---

## Task 7: Full suite and commit

- [ ] **Step 7.1: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: all tests pass (88 skipped — reactive JPA, Docker-only)

- [ ] **Step 7.2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/api/A2ATaskStateTest.java \
  runtime/src/test/java/io/casehub/qhorus/api/A2AGetTaskTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(#151): DELEGATED and HANDOFF map to 'working' in A2A state; extract A2ATaskState"
```
