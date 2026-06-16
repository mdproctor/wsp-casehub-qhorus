# Advisory Channel Type Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Change `StoredMessageTypePolicy` so COMMAND/QUERY violations remain hard-enforced (they create Commitments; wrong-channel advisory dispatch causes orphan obligations) while all other type violations become advisory — logged and returned in `DispatchResult.advisories()`, but dispatch proceeds.

**Architecture:** Add an `advisory()` default method to `MessageTypePolicy` alongside the existing `validate()` SAM. `StoredMessageTypePolicy.validate()` hard-enforces COMMAND/QUERY only; `advisory()` returns violation text for all other types. `DispatchResult` gains an `advisories` field. `MessageService` (and reactive mirror) call `validate()` then `advisory()` on every dispatch, thread the advisory into `DispatchResult`. Remove the redundant `validate()` pre-call from MCP tools — `MessageService` is the single gate.

**Tech Stack:** Java 21, Quarkus 3.32.2, JBoss Logging, Mutiny, JUnit 5 / AssertJ, `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn`

---

## File Map

| File | Change |
|---|---|
| `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java` | Add `advisories` field |
| `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageTypePolicy.java` | Add `advisory()` default method |
| `runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java` | Rewrite `validate()` (COMMAND/QUERY only) + implement `advisory()` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` | Add Logger; update type-policy block; thread `advisories` into both `DispatchResult` constructions |
| `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` | Add `AtomicReference`; update `.invoke()` type-policy step; thread `advisoriesRef.get()` into both `DispatchResult` constructions |
| `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` | Remove `validate()` call at line 571; update `@Tool` description for `set_channel_type_constraints` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` | Remove `validate()` call at line 751; update `@Tool` description |
| `runtime/src/test/java/io/casehub/qhorus/message/StoredMessageTypePolicyTest.java` | Update flipping tests; add new hybrid-enforcement tests |
| `runtime/src/test/java/io/casehub/qhorus/message/MessageServiceTypeEnforcementTest.java` | Flip `serverSide_violation_messageContainsChannelAndType()` |
| `runtime/src/test/java/io/casehub/qhorus/channel/ChannelServiceTest.java` | Flip `dispatch_deniedType_throwsViolation()` |
| `runtime/src/test/java/io/casehub/qhorus/runtime/message/DeniedTypesMcpTest.java` | Flip `sendMessage_deniedType_throwsToolCallException()` |
| `runtime/src/test/java/io/casehub/qhorus/mcp/ChannelAllowedTypesTest.java` | Rename `sendMessage_rejectsDisallowedType_clientSide()`; flip `violationError_mentionsChannelAndType()` |
| `examples/normative-layout/src/test/java/.../NormativeLayoutTypeEnforcementTest.java` | Flip `oversightChannel_rejectsEvent_serverSide()` and `violationException_messageContainsChannelNameAndType()` |
| `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java` | Add `List.of()` to `DispatchResult` constructor at line 251 |
| `connector-backend/src/test/java/.../ConnectorQhorusMeshBridgeTest.java` | Add `List.of()` to `DispatchResult` constructor at line 224 |

---

## Task 1: Add `advisory()` to `MessageTypePolicy`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageTypePolicy.java`

- [ ] **Step 1: Open `MessageTypePolicy.java` and verify current content**

The file currently has one abstract method `void validate(Channel channel, MessageType type)` and the `@FunctionalInterface` annotation.

- [ ] **Step 2: Add the `advisory()` default method**

Replace the entire file with:

```java
package io.casehub.qhorus.runtime.message;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.MessageTypeViolationException;
import io.casehub.qhorus.runtime.channel.Channel;

@FunctionalInterface
public interface MessageTypePolicy {

    /**
     * Hard-block gate. Throw {@link MessageTypeViolationException} to reject; return normally
     * to permit. {@link StoredMessageTypePolicy} hard-enforces only COMMAND and QUERY
     * violations — these are the only types that call {@code commitmentService.open()};
     * advisory dispatch on the wrong channel creates orphan Commitments when the LLM corrects.
     *
     * <p>Custom policies that need hard enforcement for additional types override this method.
     */
    void validate(Channel channel, MessageType type);

    /**
     * Advisory evaluation. Returns a human-readable violation description when the type
     * violates the channel's declared constraints and the type is not COMMAND or QUERY
     * (those are hard-enforced by {@link #validate}). Returns {@code null} when permitted
     * or when the type is obligation-creating.
     *
     * <p>Never throws for well-formed channel configurations. For malformed
     * {@code allowedTypes}/{@code deniedTypes} values (unknown type names), propagates
     * {@link IllegalArgumentException} from {@link MessageType#parseTypes} — an impossible
     * condition in production since {@code ChannelCreateRequest} validates at creation time.
     *
     * <p><strong>Calling contract for custom implementations:</strong> when a custom policy
     * provides only {@code validate()} and leaves {@code advisory()} as the default null,
     * the calling sequence is: {@code validate()} → may throw →
     * {@code advisory()} → null (no advisory logged, because advisory() is called only
     * after validate() returns normally; if validate() throws, advisory() is never invoked).
     * This is the correct hard-enforcement-only mode.
     *
     * <p>Default: {@code null} — no advisory; defers entirely to {@code validate()}.
     */
    default String advisory(Channel channel, MessageType type) {
        return null;
    }
}
```

- [ ] **Step 3: Build just the runtime module to confirm it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: `BUILD SUCCESS`

---

## Task 2: Rewrite `StoredMessageTypePolicy` with TDD

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/message/StoredMessageTypePolicyTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java`

### Step 2a — Update existing tests that flip, and write new tests

- [ ] **Step 1: Update `multipleTypes_rejectsUnlisted()` — EVENT not obligation-creating**

Find:
```java
@Test
void multipleTypes_rejectsUnlisted() {
    Channel ch = channel("QUERY,COMMAND,RESPONSE");
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
}
```

Replace with:
```java
@Test
void multipleTypes_rejectsUnlisted() {
    Channel ch = channel("QUERY,COMMAND,RESPONSE");
    // validate() is a no-op for EVENT (not obligation-creating)
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
    // advisory() fires instead
    assertNotNull(policy.advisory(ch, MessageType.EVENT));
}
```

- [ ] **Step 2: Update `deniedType_onOpenChannel_isRejected()` — EVENT denied**

Find:
```java
@Test
void deniedType_onOpenChannel_isRejected() {
    Channel ch = channelWithDenied(null, "EVENT");
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
}
```

Replace with:
```java
@Test
void deniedType_onOpenChannel_isRejected() {
    Channel ch = channelWithDenied(null, "EVENT");
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
    String adv = policy.advisory(ch, MessageType.EVENT);
    assertNotNull(adv);
    assertTrue(adv.contains("denies"), "Advisory should mention denial: " + adv);
}
```

- [ ] **Step 3: Update `deniedType_exceptionMessageIndicatesDenial()` — EVENT denied**

Find:
```java
@Test
void deniedType_exceptionMessageIndicatesDenial() {
    Channel ch = channelWithDenied(null, "EVENT");
    ch.name = "case-abc/oversight";
    MessageTypeViolationException ex = assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
    assertTrue(ex.getMessage().contains("denies"), ...);
    assertTrue(ex.getMessage().contains("case-abc/oversight"));
    assertTrue(ex.getMessage().contains("EVENT"));
}
```

Replace with:
```java
@Test
void deniedType_exceptionMessageIndicatesDenial() {
    Channel ch = channelWithDenied(null, "EVENT");
    ch.name = "case-abc/oversight";
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
    String adv = policy.advisory(ch, MessageType.EVENT);
    assertNotNull(adv);
    assertTrue(adv.contains("denies"), "Expected 'denies' in advisory: " + adv);
    assertTrue(adv.contains("case-abc/oversight"), "Expected channel name in advisory: " + adv);
    assertTrue(adv.contains("EVENT"), "Expected type in advisory: " + adv);
}
```

- [ ] **Step 4: Update `unknownTypeName_throwsIllegalArgument()` — test via `advisory()` now**

Find:
```java
@Test
void unknownTypeName_throwsIllegalArgument() {
    Channel ch = channel("RUBBISH");
    assertThrows(IllegalArgumentException.class,
            () -> policy.validate(ch, MessageType.EVENT));
}
```

Replace with:
```java
@Test
void unknownTypeName_throwsIllegalArgument() {
    // validate() is a no-op for EVENT; advisory() calls parseTypes("RUBBISH") which throws IAE
    Channel ch = channel("RUBBISH");
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
    assertThrows(IllegalArgumentException.class,
            () -> policy.advisory(ch, MessageType.EVENT));
}
```

- [ ] **Step 5: Update `nullDeniedTypes_hasNoEffect()` — flip second assertion only**

Find:
```java
@Test
void nullDeniedTypes_hasNoEffect() {
    Channel ch = channelWithDenied("QUERY", null);
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.QUERY));
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
}
```

Replace with:
```java
@Test
void nullDeniedTypes_hasNoEffect() {
    Channel ch = channelWithDenied("QUERY", null);
    // QUERY is obligation-creating — hard-enforced; allowedTypes="QUERY" so no violation
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.QUERY));
    // EVENT is not obligation-creating — validate() no-op; advisory() fires
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
    assertNotNull(policy.advisory(ch, MessageType.EVENT));
}
```

- [ ] **Step 6: Add new tests for hybrid enforcement at the end of the test class**

```java
// ── Hybrid enforcement — COMMAND/QUERY hard, others advisory ────────────

@Test
void validate_command_onEventOnlyChannel_throws() {
    Channel ch = channel("EVENT");
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.COMMAND));
}

@Test
void validate_query_onEventOnlyChannel_throws() {
    Channel ch = channel("EVENT");
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.QUERY));
}

@Test
void validate_status_onEventOnlyChannel_doesNotThrow() {
    Channel ch = channel("EVENT");
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.STATUS));
}

@Test
void validate_event_onDeniedChannel_doesNotThrow() {
    Channel ch = channelWithDenied(null, "EVENT");
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
}

@Test
void advisory_commandOnEventOnlyChannel_returnsNull() {
    // COMMAND is hard-enforced; advisory() returns null for obligation-creating types
    Channel ch = channel("EVENT");
    assertNull(policy.advisory(ch, MessageType.COMMAND));
}

@Test
void advisory_statusOnEventOnlyChannel_returnsText() {
    Channel ch = channel("EVENT");
    ch.name = "case/observe";
    String adv = policy.advisory(ch, MessageType.STATUS);
    assertNotNull(adv);
    assertTrue(adv.contains("case/observe"), "Advisory should contain channel name: " + adv);
    assertTrue(adv.contains("STATUS"), "Advisory should contain type: " + adv);
}

@Test
void advisory_eventOnDeniedChannel_returnsText() {
    Channel ch = channelWithDenied(null, "EVENT");
    ch.name = "case/oversight";
    String adv = policy.advisory(ch, MessageType.EVENT);
    assertNotNull(adv);
    assertTrue(adv.contains("denies"), "Advisory should mention denial: " + adv);
}

@Test
void advisory_commandOnOpenChannel_returnsNull() {
    Channel ch = channel(null);
    assertNull(policy.advisory(ch, MessageType.COMMAND));
}
```

- [ ] **Step 7: Run the policy tests — they should all FAIL** (implementation not updated yet)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StoredMessageTypePolicyTest -pl runtime -q 2>&1 | tail -20
```

Expected: multiple failures (validate() still throws for EVENT; advisory() method doesn't exist yet)

### Step 2b — Implement the new `StoredMessageTypePolicy`

- [ ] **Step 8: Replace `StoredMessageTypePolicy.java` with the hybrid implementation**

```java
package io.casehub.qhorus.runtime.message;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.MessageTypeViolationException;
import io.casehub.qhorus.runtime.channel.Channel;

@ApplicationScoped
public class StoredMessageTypePolicy implements MessageTypePolicy {

    /**
     * Hard-enforces COMMAND and QUERY only — both call commitmentService.open(); advisory
     * dispatch on the wrong channel creates orphan Commitments. No-op for all other types.
     */
    @Override
    public void validate(Channel channel, MessageType type) {
        if (type != MessageType.COMMAND && type != MessageType.QUERY) return;
        // Denial-first: explicit denial wins over allowedTypes
        if (channel.deniedTypes != null && !channel.deniedTypes.isBlank()) {
            if (MessageType.parseTypes(channel.deniedTypes).contains(type)) {
                throw MessageTypeViolationException.denied(channel.name, type, channel.deniedTypes);
            }
        }
        // Open channel (no allowedTypes restriction) passes after denial check
        if (channel.allowedTypes == null || channel.allowedTypes.isBlank()) return;
        if (!MessageType.parseTypes(channel.allowedTypes).contains(type)) {
            throw new MessageTypeViolationException(channel.name, type, channel.allowedTypes);
        }
    }

    /**
     * Advisory for non-obligation-creating types only. Returns null for COMMAND and QUERY
     * (those are hard-enforced by validate()). Denial-first: denial wins over allowedTypes.
     */
    @Override
    public String advisory(Channel channel, MessageType type) {
        if (type == MessageType.COMMAND || type == MessageType.QUERY) return null;
        if (channel.deniedTypes != null && !channel.deniedTypes.isBlank()) {
            if (MessageType.parseTypes(channel.deniedTypes).contains(type)) {
                return "Type advisory: channel '" + channel.name
                        + "' explicitly denies " + type
                        + " — denied: [" + channel.deniedTypes + "]. Message dispatched.";
            }
        }
        if (channel.allowedTypes == null || channel.allowedTypes.isBlank()) return null;
        if (!MessageType.parseTypes(channel.allowedTypes).contains(type)) {
            return "Type advisory: channel '" + channel.name
                    + "' allows [" + channel.allowedTypes + "] only, received " + type
                    + ". Message dispatched.";
        }
        return null;
    }
}
```

- [ ] **Step 9: Run policy tests — they should all PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StoredMessageTypePolicyTest -pl runtime -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all tests green.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageTypePolicy.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java \
  runtime/src/test/java/io/casehub/qhorus/message/StoredMessageTypePolicyTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#271): hybrid type enforcement — COMMAND/QUERY hard, others advisory

Refs #271"
```

---

## Task 3: Add `advisories` field to `DispatchResult` and fix all construction sites

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java` (line 251)
- Modify: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorQhorusMeshBridgeTest.java` (line 224)

- [ ] **Step 1: Update `DispatchResult.java`**

```java
package io.casehub.qhorus.api.message;

import java.util.List;
import java.util.UUID;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record DispatchResult(
        Long messageId,
        UUID channelId,
        String sender,
        MessageType type,
        String correlationId,
        Long inReplyTo,
        @JsonInclude(JsonInclude.Include.NON_EMPTY) List<UUID> artefactRefs,
        String target,
        UUID ledgerEntryId,
        UUID subjectId,
        UUID causedByEntryId,
        int parentReplyCount,
        @JsonInclude(JsonInclude.Include.NON_EMPTY) List<String> advisories
) {
    public DispatchResult {
        artefactRefs = artefactRefs == null ? List.of() : List.copyOf(artefactRefs);
        advisories   = advisories   == null ? List.of() : List.copyOf(advisories);
    }
}
```

- [ ] **Step 2: Fix `QhorusDashboardServiceTest.java` line 251**

Find (line 251):
```java
DispatchResult dr = new DispatchResult(42L, ch.id, "human:alice", MessageType.STATUS,
```

The full call at line 251-252 will now fail to compile (12 args, now needs 13). Add `List.of()` as the final argument. Find the complete constructor call and add the trailing arg. The original ends with `null, null, null, 0)` — change to `null, null, null, 0, List.of())`.

- [ ] **Step 3: Fix `ConnectorQhorusMeshBridgeTest.java` line 224**

Find (line 223-226):
```java
private static DispatchResult dummyResult() {
    return new DispatchResult(1L, UUID.randomUUID(), "system:connector:slack",
            MessageType.STATUS, null, null, null, null, null, null, null, 0);
}
```

Change to:
```java
private static DispatchResult dummyResult() {
    return new DispatchResult(1L, UUID.randomUUID(), "system:connector:slack",
            MessageType.STATUS, null, null, null, null, null, null, null, 0, List.of());
}
```

Also verify `List` is imported in that file; add `import java.util.List;` if not present.

- [ ] **Step 4: Build to confirm the API module compiles and the two test files compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test-compile -pl api,runtime,connector-backend -q 2>&1 | tail -20
```

Expected: `BUILD SUCCESS` (the two production DispatchResult construction sites in MessageService and ReactiveMessageService are not yet updated; they will fail at this step — that's fine, proceed)

If there are other compilation errors pointing to construction sites not in this plan, fix them now (add `List.of()` as the final arg).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java \
  connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorQhorusMeshBridgeTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#271): add advisories field to DispatchResult

Refs #271"
```

---

## Task 4: Update `MessageService.dispatch()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`

- [ ] **Step 1: Add Logger field**

After the class declaration, add (before the first `@Inject`):
```java
private static final Logger LOG = Logger.getLogger(MessageService.class);
```

Add the import at the top of the file:
```java
import org.jboss.logging.Logger;
```

- [ ] **Step 2: Replace the type-policy block (lines ~189-192)**

Current code (line ~189-192):
```java
        // ── Type policy ───────────────────────────────────────────────────────────────────
        if (ch != null) {
            messageTypePolicy.validate(ch, dispatch.type());
        }
```

Replace with:
```java
        // ── Type policy ───────────────────────────────────────────────────────────────────
        List<String> advisories = List.of();
        if (ch != null) {
            // Hard gate: throws MessageTypeViolationException for COMMAND/QUERY violations.
            // No-op for all other types — they cannot create orphan Commitments.
            messageTypePolicy.validate(ch, dispatch.type());
            // Advisory: logs warning for non-COMMAND/QUERY violations; null for COMMAND/QUERY.
            final String adv = messageTypePolicy.advisory(ch, dispatch.type());
            if (adv != null) {
                LOG.warn(adv);
                advisories = List.of(adv);
            }
        }
```

Also add `import java.util.List;` if not already imported.

- [ ] **Step 3: Fix the LAST_WRITE `DispatchResult` construction (~line 227)**

Current (12 args, the last param is `0`):
```java
                    return new DispatchResult(last.id, ch.id, last.sender,
                            last.messageType, last.correlationId, last.inReplyTo,
                            ArtefactRefParser.parse(last.artefactRefs), last.target,
                            null, null, null, 0);
```

Replace with:
```java
                    return new DispatchResult(last.id, ch.id, last.sender,
                            last.messageType, last.correlationId, last.inReplyTo,
                            ArtefactRefParser.parse(last.artefactRefs), last.target,
                            null, null, null, 0, advisories);
```

- [ ] **Step 4: Fix the normal `DispatchResult` construction (~line 331)**

Find the normal-path `DispatchResult` construction at line ~331 (end of `dispatch()`, after the fan-out and rate-limit steps). It ends with `parentReplyCount)`. Add `, advisories)` — change `parentReplyCount)` to `parentReplyCount, advisories)`.

- [ ] **Step 5: Build runtime module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#271): update MessageService — Logger, advisory dispatch, thread advisories to DispatchResult

Refs #271"
```

---

## Task 5: Update `ReactiveMessageService.dispatch()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`

- [ ] **Step 1: Declare `AtomicReference` before the Uni chain**

Add `import java.util.concurrent.atomic.AtomicReference;` at the top.

In `dispatch()`, before the reactive chain starts, add:
```java
final AtomicReference<List<String>> advisoriesRef = new AtomicReference<>(List.of());
```

- [ ] **Step 2: Replace the type-policy `.invoke()` step (lines 219-224)**

Current:
```java
                .invoke(ch -> {
                    // Phase 1e: Type policy (sync)
                    if (ch != null) {
                        messageTypePolicy.validate(ch, dispatch.type());
                    }
                })
```

Replace with:
```java
                .invoke(ch -> {
                    // Phase 1e: Type policy (sync)
                    if (ch != null) {
                        messageTypePolicy.validate(ch, dispatch.type()); // hard gate
                        final String adv = messageTypePolicy.advisory(ch, dispatch.type());
                        if (adv != null) {
                            LOG.warn(adv);
                            advisoriesRef.set(List.of(adv));
                        }
                    }
                })
```

- [ ] **Step 3: Fix the LAST_WRITE/overwrite `DispatchResult` construction (~line 250-258)**

Current (12 args ending with `null, null, null, 0`):
```java
                                                                 new DispatchResult(
                                                                         last.id, ch.id, last.sender,
                                                                         last.messageType,
                                                                         last.correlationId,
                                                                         last.inReplyTo,
                                                                         ArtefactRefParser.parse(
                                                                                 last.artefactRefs),
                                                                         last.target,
                                                                         null, null, null, 0)));
```

Add `advisoriesRef.get()` as the 13th arg: change `null, null, null, 0)` to `null, null, null, 0, advisoriesRef.get())`.

- [ ] **Step 4: Fix the normal `DispatchResult` construction (lines 337-349)**

The construction is inside the outer chain's `.flatMap().map()` lambda (Phase 3/4 of dispatch), NOT inside `doNormalInsert()` — so `advisoriesRef` is in scope. Find:

```java
                                    return new DispatchResult(
                                            ctx.messageId(),
                                            dispatch.channelId(),
                                            dispatch.sender(),
                                            dispatch.type(),
                                            dispatch.correlationId(),
                                            dispatch.inReplyTo(),
                                            ArtefactRefParser.parse(dispatch.artefactRefs()),
                                            dispatch.target(),
                                            lo.entryId(),
                                            lo.subjectId(),
                                            lo.causedByEntryId(),
                                            ctx.replyCount());
```

Add `, advisoriesRef.get()` before the closing `)`:
```java
                                    return new DispatchResult(
                                            ctx.messageId(),
                                            dispatch.channelId(),
                                            dispatch.sender(),
                                            dispatch.type(),
                                            dispatch.correlationId(),
                                            dispatch.inReplyTo(),
                                            ArtefactRefParser.parse(dispatch.artefactRefs()),
                                            dispatch.target(),
                                            lo.entryId(),
                                            lo.subjectId(),
                                            lo.causedByEntryId(),
                                            ctx.replyCount(),
                                            advisoriesRef.get());
```

- [ ] **Step 5: Build runtime module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#271): update ReactiveMessageService — AtomicReference advisory capture

Refs #271"
```

---

## Task 6: Update MCP tools

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Remove `validate()` from `QhorusMcpTools.sendMessage()` (lines 570-571)**

Remove these two lines:
```java
        // Type policy — client-side early rejection (MessageService.dispatch() enforces server-side)
        messageTypePolicy.validate(ch, msgType);
```

- [ ] **Step 2: Update the `@Tool` description for `set_channel_type_constraints` in `QhorusMcpTools` (lines 323-329)**

Replace the `description = "..."` string (current text includes "Denial wins at dispatch time" and references PP-20260604-a7ad99) with:

```java
    @Tool(name = "set_channel_type_constraints",
            description = "Replace the allowed_types and denied_types constraints on an existing channel. "
                    + "This is a full-replacement operation: both fields are overwritten on every call. "
                    + "Pass null for a field to clear the constraint; pass the current value to preserve it. "
                    + "Constraint enforcement is type-discriminated: COMMAND and QUERY are hard-enforced "
                    + "(a violation throws and the message is not dispatched — these types create Commitments; "
                    + "wrong-channel dispatch creates orphan obligations). All other types are advisory: "
                    + "a violation warning is returned in the advisories field of the dispatch result and "
                    + "the message is dispatched. Denial wins over allowed_types when both are set. "
                    + "Constraints are prospective only — messages already in the channel are unaffected.")
```

- [ ] **Step 3: Remove `validate()` from `ReactiveQhorusMcpTools.sendMessage()` (lines 750-751)**

Remove:
```java
        // Type policy — client-side early rejection (MessageService.dispatch() enforces server-side)
        messageTypePolicy.validate(ch, msgType);
```

- [ ] **Step 4: Update the `@Tool` description for `set_channel_type_constraints` in `ReactiveQhorusMcpTools` (line 312)**

Apply the same description replacement as Step 2.

- [ ] **Step 5: Build runtime module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#271): remove client-side validate() from MCP tools; update @Tool descriptions

Refs #271"
```

---

## Task 7: Fix integration tests that flip

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/message/MessageServiceTypeEnforcementTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/channel/ChannelServiceTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/message/DeniedTypesMcpTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/ChannelAllowedTypesTest.java`
- Modify: `examples/normative-layout/src/test/java/io/casehub/qhorus/examples/normativelayout/NormativeLayoutTypeEnforcementTest.java`

- [ ] **Step 1: Fix `MessageServiceTypeEnforcementTest.serverSide_violation_messageContainsChannelAndType()`**

Current (lines 253-269): dispatches EVENT to QUERY+COMMAND channel, asserts `MessageTypeViolationException` with channel name and "EVENT".

Replace with:
```java
@Test
void serverSide_violation_messageContainsChannelAndType() {
    String name = "server-msg-" + System.nanoTime();
    UUID channelId = createChannel(name, Set.of(MessageType.QUERY, MessageType.COMMAND));

    // EVENT is not obligation-creating — dispatch succeeds with advisory
    DispatchResult result = QuarkusTransaction.requiringNew().call(() -> messageService.dispatch(
            MessageDispatch.builder()
                    .channelId(channelId)
                    .sender("agent-1")
                    .type(MessageType.EVENT)
                    .telemetry("{}")
                    .actorType(ActorTypeResolver.resolve("agent-1"))
                    .build()));

    assertFalse(result.advisories().isEmpty(), "Expected advisory for EVENT on constrained channel");
    String adv = result.advisories().get(0);
    assertTrue(adv.contains(name), "Expected channel name in advisory: " + adv);
    assertTrue(adv.contains("EVENT"), "Expected type in advisory: " + adv);
}
```

- [ ] **Step 2: Fix `ChannelServiceTest.dispatch_deniedType_throwsViolation()` (line 187-206)**

Current: creates channel with `deniedTypes={EVENT}`, dispatches EVENT, asserts `MessageTypeViolationException`.

Replace the `assertThrows` block with:
```java
        // EVENT is not obligation-creating — dispatch succeeds with advisory
        DispatchResult[] result = new DispatchResult[1];
        QuarkusTransaction.requiringNew().run(() ->
                result[0] = messageService.dispatch(MessageDispatch.builder()
                        .channelId(chId[0])
                        .sender("telemetry-agent")
                        .type(MessageType.EVENT)
                        .telemetry("{\"tool\":\"search\"}")
                        .actorType(ActorTypeResolver.resolve("telemetry-agent"))
                        .build()));
        assertFalse(result[0].advisories().isEmpty(), "Expected advisory for denied EVENT");
```

Also add the import `import io.casehub.qhorus.api.message.DispatchResult;` if not present.

- [ ] **Step 3: Fix `DeniedTypesMcpTest.sendMessage_deniedType_throwsToolCallException()`**

Current: sends EVENT via MCP to `deniedTypes=EVENT` channel, asserts `ToolCallException`.

Replace with:
```java
@Test
@TestTransaction
void sendMessage_deniedType_returnsAdvisory() {
    tools.createChannel(
            "oversight-denied", "Oversight channel",
            null, null, null, null, null, null,
            null, "EVENT",
            null, null, null, null);

    // EVENT is not obligation-creating — dispatch succeeds with advisory
    DispatchResult result = tools.sendMessage("oversight-denied", "telemetry-agent", "event",
            null, null, null, null, null, null, null, null);

    assertFalse(result.advisories().isEmpty(), "Expected advisory for denied EVENT");
    String adv = result.advisories().get(0);
    assertTrue(adv.contains("denies"), "Advisory should mention denial: " + adv);
    assertTrue(adv.contains("EVENT"), "Advisory should name the type: " + adv);
}
```

- [ ] **Step 4: Fix `ChannelAllowedTypesTest.violationError_mentionsChannelAndType()`**

Current (lines 77-84): sends EVENT to QUERY+COMMAND channel via MCP, asserts exception with "EVENT".

Replace with:
```java
@Test
@TestTransaction
void advisory_mentionsChannelAndType() {
    String name = "oversight-block-" + System.nanoTime();
    tools.createChannel(name,  "Governance",  "APPEND",
            null,  null,  null,  null,  null,  "QUERY,COMMAND",  null,  null,  null,  null,  null);
    // EVENT is not obligation-creating — dispatch succeeds with advisory
    DispatchResult result = tools.sendMessage(name, "agent-1", "EVENT", "{\"tool\":\"read\"}",
            null, null, null, null, null, null, null);
    assertFalse(result.advisories().isEmpty(), "Expected advisory for EVENT on constrained channel");
    assertTrue(result.advisories().get(0).contains("EVENT"), "Advisory should name the type");
}
```

- [ ] **Step 5: Rename `ChannelAllowedTypesTest.sendMessage_rejectsDisallowedType_clientSide()`**

The validate() call has moved server-side. Rename to `sendMessage_rejectsDisallowedType_serverSide()` (the test body is correct — QUERY on EVENT-only still throws, just now from MessageService). Update the method name only.

- [ ] **Step 6: Fix `NormativeLayoutTypeEnforcementTest.oversightChannel_rejectsEvent_serverSide()`**

Current (lines 93-106): dispatches EVENT to oversight channel, asserts `MessageTypeViolationException`.

Replace the `assertThatThrownBy` block with:
```java
    @Test
    void oversightChannel_advisesEvent_serverSide() {
        SecureCodeReviewScenario s = scenario("enf-4-");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        DispatchResult[] result = new DispatchResult[1];
        QuarkusTransaction.requiringNew().run(() -> {
            result[0] = messageService.dispatch(MessageDispatch.builder()
                    .channelId(s.oversightChannel().id)
                    .sender("agent-x")
                    .type(MessageType.EVENT)
                    .telemetry("{\"tool\":\"blocked\"}")
                    .actorType(ActorTypeResolver.resolve("agent-x"))
                    .build());
        });
        assertThat(result[0].advisories()).isNotEmpty();
    }
```

- [ ] **Step 7: Fix `NormativeLayoutTypeEnforcementTest.violationException_messageContainsChannelNameAndType()`**

Current (lines 207-225): dispatches STATUS to observe channel, asserts `MessageTypeViolationException` with channel name and "STATUS".

Replace with:
```java
    @Test
    void advisory_messageContainsChannelNameAndType() {
        SecureCodeReviewScenario s = scenario("enf-8-");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        DispatchResult[] result = new DispatchResult[1];
        QuarkusTransaction.requiringNew().run(() -> {
            result[0] = messageService.dispatch(MessageDispatch.builder()
                    .channelId(s.observeChannel().id)
                    .sender("agent-x")
                    .type(MessageType.STATUS)
                    .content("still working")
                    .actorType(ActorTypeResolver.resolve("agent-x"))
                    .build());
        });
        assertThat(result[0].advisories()).isNotEmpty();
        String adv = result[0].advisories().get(0);
        assertThat(adv).contains(s.observeChannel().name);
        assertThat(adv).contains("STATUS");
    }
```

- [ ] **Step 8: Run all the affected test classes to verify the flips pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="MessageServiceTypeEnforcementTest,ChannelServiceTest,DeniedTypesMcpTest,ChannelAllowedTypesTest" \
  -pl runtime -q 2>&1 | tail -20
```

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="NormativeLayoutTypeEnforcementTest" \
  -pl examples/normative-layout -q 2>&1 | tail -20
```

Expected: `BUILD SUCCESS` for both.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/test/java/io/casehub/qhorus/message/MessageServiceTypeEnforcementTest.java \
  runtime/src/test/java/io/casehub/qhorus/channel/ChannelServiceTest.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/message/DeniedTypesMcpTest.java \
  runtime/src/test/java/io/casehub/qhorus/mcp/ChannelAllowedTypesTest.java \
  examples/normative-layout/src/test/java/io/casehub/qhorus/examples/normativelayout/NormativeLayoutTypeEnforcementTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#271): flip advisory test assertions across runtime and examples

Refs #271"
```

---

## Task 8: Full build and test run

- [ ] **Step 1: Run the full project build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q 2>&1 | tail -30
```

Expected: `BUILD SUCCESS`. If any tests fail, investigate — the spec enumerates all expected flip sites.

- [ ] **Step 2: Verify `ToolOverloadDiscoverabilityTest` still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="ToolOverloadDiscoverabilityTest" -pl runtime -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

---

## Task 9: Protocol, CLAUDE.md, and ADR

**Files:**
- Modify: `~/claude/casehub/garden/docs/protocols/casehub/channel-type-policy-invariant.md`
- Modify: `/Users/mdproctor/claude/casehub/qhorus/CLAUDE.md`

- [ ] **Step 1: Update protocol `channel-type-policy-invariant.md`**

Replace the body (preserve frontmatter) with:

```markdown
Type enforcement on Qhorus channels is discriminated by normative weight.

**COMMAND and QUERY — hard-enforced.**
Both types call `commitmentService.open()`. Advisory dispatch on the wrong channel followed by
LLM correction creates orphan OPEN Commitments — stalled permanently with no mechanism to
distinguish them from genuine governance failures. Hard enforcement is normatively correct for
these types: a directive that cannot be honoured on a channel where no agent responds is not a
valid speech act.

**All other types — advisory.**
STATUS, EVENT, DONE, FAILURE, DECLINE, RESPONSE, HANDOFF do not call `commitmentService.open()`.
Advisory dispatch for these types produces an accurate audit entry (WARN log + ledger entry +
`DispatchResult.advisories()`) without orphan risk. Hard enforcement for non-obligation-creating
types erases constraint violations from the ledger; advisory enforcement makes them visible.

The `observe`/`oversight`/`work` normative layout examples remain valid:
- `observe` (`allowedTypes=EVENT`): COMMAND and QUERY on observe are still hard-blocked.
  EVENT (and STATUS etc.) on other channels with allowedTypes=EVENT trigger advisories.
- `oversight` (`deniedTypes=EVENT`): EVENT on oversight triggers an advisory (EVENTs are
  already excluded from `pollAfter` defaults; the advisory record is more informative than
  a hard block).
- `work` (open): no constraints; no enforcement.

`allowedTypes` and `deniedTypes` remain valid channel configuration. They declare intent and
drive enforcement at the appropriate strength for each type category.
```

- [ ] **Step 2: Update two stale notes in `CLAUDE.md`**

Find: `"MessageTypePolicy is injected into both QhorusMcpTools.sendMessage() (client-side early rejection) and MessageService.dispatch() (server-side enforcement)."`

Replace the relevant clause with: `"MessageService.dispatch() is the single enforcement point for MessageTypePolicy. The injection in QhorusMcpTools is retained but the validate() call is removed — MCP tools no longer do client-side early rejection."`

Find: `"StoredMessageTypePolicy.validate() runs denial-first: if channel.deniedTypes contains the type, MessageTypeViolationException.denied() is thrown"`

Replace with: `"StoredMessageTypePolicy.validate() hard-enforces COMMAND and QUERY only (both create Commitments; wrong-channel advisory dispatch causes orphan obligations). advisory() returns warning text for all other types. validate() runs denial-first for COMMAND/QUERY."`

- [ ] **Step 3: Create ADR**

Run the `adr` skill to create an ADR titled "Why channel type enforcement is type-discriminated" with the rationale: obligation-creating types (COMMAND, QUERY) create Commitments; advisory dispatch on wrong channel followed by LLM correction creates orphan Commitments; hard enforcement is normatively correct for those types; all others are advisory to produce complete audit records.

- [ ] **Step 4: Commit**

```bash
git -C ~/claude/casehub/garden add docs/protocols/casehub/channel-type-policy-invariant.md
git -C ~/claude/casehub/garden commit -m "protocol: update channel-type-policy-invariant — hybrid enforcement model"

git -C /Users/mdproctor/claude/casehub/qhorus add CLAUDE.md docs/adr/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#271): update CLAUDE.md, ADR for hybrid type enforcement

Closes #271"
```

---

## Task 10: Final verification

- [ ] **Step 1: Full build from root**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`

- [ ] **Step 2: Confirm issue is closed**

```bash
gh issue view 271 --repo casehubio/qhorus --json state,title --jq '"#\(.number) [\(.state)] \(.title)"'
```

Expected: `#271 [CLOSED] arch: make allowedTypes/deniedTypes advisory — warn not hard-block`
