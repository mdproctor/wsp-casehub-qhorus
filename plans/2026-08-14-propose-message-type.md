# PROPOSE Message Type Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #395 — Add PROPOSE message type — commissive speech act for negotiation protocols
**Issue group:** #395

**Goal:** Add PROPOSE as the 10th message type with distinct fulfillment semantics (RESPONSE does not auto-fulfill)

**Architecture:** PROPOSE is a new obligation-creating entry point into the existing 7-state commitment lifecycle. It differs from COMMAND only in that RESPONSE does not fulfill. All other commitment transitions (DONE, DECLINE, FAILURE, HANDOFF, STATUS) behave identically. No Flyway migration needed — MessageType stored as String, Commitment already has `messageType` column.

**Tech Stack:** Java 21, Quarkus 3.32.2, H2 (tests), Maven

## Global Constraints

- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn test -Dtest=ClassName -pl runtime` for targeted tests
- All commits reference `Refs #395`
- After API module changes, run `mvn install` from project root before testing runtime
- `@TestTransaction` + `REQUIRES_NEW` gotcha: ledger writes persist after rollback. Set up channels and messages inside `@Test` body.
- CDI-free unit tests must set `service.tracingConfig` to a disabled-tracing implementation
- `ToolOverloadDiscoverabilityTest` will fail if public non-`@Tool` overloads share a name with a `@Tool` method

---

### Task 1: MessageType enum and Builder validation (api module)

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageType.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java`

**Interfaces:**
- Produces: `MessageType.PROPOSE` enum value with `requiresCorrelationId()=true`, `requiresContent()=true`, `requiresTarget()=false`, `isTerminal()=false`, `isAgentVisible()=true`
- Produces: Builder.build() throws on null correlationId or null/blank content for PROPOSE

- [ ] **Step 1: Add PROPOSE to MessageType enum**

Add between FAILURE and EVENT (before EVENT, which is the perlocutionary outlier):

```java
/** Offer conditional commitment. Sender binds to action contingent on receiver's acceptance. Commissive. Carries correlation_id. */
PROPOSE,
/** Observer-only telemetry. NOT delivered to agent context. */
EVENT;
```

Update method implementations — add PROPOSE to return expressions:

`requiresCorrelationId()`: `return this == QUERY || this == COMMAND || this == PROPOSE;`
`requiresContent()`: `return this == DECLINE || this == FAILURE || this == PROPOSE;`

The other three methods need no change — PROPOSE returns the default:
- `isAgentVisible()` → `this != EVENT` (PROPOSE is not EVENT → true ✓)
- `requiresTarget()` → `this == HANDOFF` (PROPOSE is not HANDOFF → false ✓)
- `isTerminal()` → `this == HANDOFF || this == DONE || this == FAILURE` (PROPOSE not in set → false ✓)

- [ ] **Step 2: Add PROPOSE validation to Builder.build()**

In `MessageDispatch.java` Builder.build(), add a new case before the `default` arm:

```java
case PROPOSE -> {
    if (correlationId == null) {
        throw new IllegalArgumentException("PROPOSE requires correlationId for commitment tracking");
    }
    if (content == null || content.isBlank()) {
        throw new IllegalArgumentException("PROPOSE requires content (proposal terms)");
    }
}
```

- [ ] **Step 3: Run full api module build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
feat(#395): add PROPOSE to MessageType enum and Builder validation

Refs #395
```

---

### Task 2: MessageService dispatch — commitment lifecycle (runtime module)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java:324-373`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` (Commitment interface)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/ProposeDispatchTest.java` (new)

**Interfaces:**
- Consumes: `MessageType.PROPOSE` from Task 1
- Produces: PROPOSE opens commitments; RESPONSE on PROPOSE does not fulfill; default propose deadline config

- [ ] **Step 1: Write failing test — PROPOSE opens a commitment**

Create `ProposeDispatchTest.java` as a CDI-free unit test (same pattern as existing MessageService unit tests). Set up MessageService with mocked stores, disabled tracing, and a stub CommitmentService.

```java
@Test
void propose_opens_commitment() {
    var dispatch = MessageDispatch.builder(channelId, "proposer", MessageType.PROPOSE)
            .content("I will do X if you agree")
            .correlationId("corr-1")
            .actorType(ActorType.AGENT)
            .build();

    messageService.dispatch(dispatch);

    verify(commitmentService).open(any(), eq("corr-1"), eq(channelId),
            eq(MessageType.PROPOSE), eq("proposer"), isNull(), isNull());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProposeDispatchTest#propose_opens_commitment -pl runtime`
Expected: FAIL — PROPOSE not in the commitment ID generation guard

- [ ] **Step 3: Add PROPOSE to commitment ID generation guard**

In `MessageService.java` ~line 324, change:

```java
// Before:
final UUID commitmentId = (dispatch.correlationId() != null &&
        (dispatch.type() == MessageType.COMMAND || dispatch.type() == MessageType.QUERY))
        ? UUID.randomUUID() : null;

// After:
final UUID commitmentId = (dispatch.correlationId() != null &&
        (dispatch.type() == MessageType.COMMAND || dispatch.type() == MessageType.QUERY
         || dispatch.type() == MessageType.PROPOSE))
        ? UUID.randomUUID() : null;
```

- [ ] **Step 4: Add PROPOSE to commitment open switch**

In `MessageService.java` ~line 354, change:

```java
// Before:
case QUERY, COMMAND -> {

// After:
case QUERY, COMMAND, PROPOSE -> {
```

- [ ] **Step 5: Add default propose deadline to QhorusConfig**

In `QhorusConfig.java` Commitment interface (~line 111), add:

```java
@WithDefault("") // absent by default
Optional<Duration> defaultProposeDeadline();
```

- [ ] **Step 6: Add default deadline logic in the open branch**

In `MessageService.java`, inside the `case QUERY, COMMAND, PROPOSE` block, after the existing QUERY default deadline logic, add:

```java
if (effectiveDeadline == null && dispatch.type() == MessageType.PROPOSE) {
    var defaultDl = config.commitment().defaultProposeDeadline();
    if (defaultDl.isPresent()) {
        effectiveDeadline = Instant.now().plus(defaultDl.get());
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProposeDispatchTest#propose_opens_commitment -pl runtime`
Expected: PASS

- [ ] **Step 8: Write failing test — RESPONSE on PROPOSE does not fulfill**

```java
@Test
void response_on_propose_does_not_fulfill() {
    // Set up: a PROPOSE commitment exists
    when(commitmentStore.findByCorrelationId("corr-1"))
            .thenReturn(Optional.of(new Commitment(
                    UUID.randomUUID(), "corr-1", channelId, MessageType.PROPOSE,
                    "proposer", "responder", CommitmentState.OPEN,
                    null, null, null, null, null, "default", Instant.now())));

    var dispatch = MessageDispatch.builder(channelId, "responder", MessageType.RESPONSE)
            .content("counter-proposal")
            .correlationId("corr-1")
            .inReplyTo(1L)
            .actorType(ActorType.AGENT)
            .build();

    messageService.dispatch(dispatch);

    verify(commitmentService, never()).fulfill("corr-1");
}
```

- [ ] **Step 9: Run test to verify it fails**

Expected: FAIL — current code calls `fulfill()` for all RESPONSE dispatches

- [ ] **Step 10: Split RESPONSE from DONE in the commitment switch**

In `MessageService.java` ~line 368, change:

```java
// Before:
case RESPONSE, DONE -> commitmentService.fulfill(dispatch.correlationId());

// After:
case DONE -> commitmentService.fulfill(dispatch.correlationId());
case RESPONSE -> {
    var commitment = commitmentStore.findByCorrelationId(dispatch.correlationId());
    if (commitment.isPresent() && commitment.get().messageType() != MessageType.PROPOSE) {
        commitmentService.fulfill(dispatch.correlationId());
    }
}
```

- [ ] **Step 11: Write test — RESPONSE on COMMAND still fulfills**

```java
@Test
void response_on_command_still_fulfills() {
    when(commitmentStore.findByCorrelationId("corr-2"))
            .thenReturn(Optional.of(new Commitment(
                    UUID.randomUUID(), "corr-2", channelId, MessageType.COMMAND,
                    "commander", "executor", CommitmentState.OPEN,
                    null, null, null, null, null, "default", Instant.now())));

    var dispatch = MessageDispatch.builder(channelId, "executor", MessageType.RESPONSE)
            .content("done")
            .correlationId("corr-2")
            .inReplyTo(2L)
            .actorType(ActorType.AGENT)
            .build();

    messageService.dispatch(dispatch);

    verify(commitmentService).fulfill("corr-2");
}
```

- [ ] **Step 12: Run all ProposeDispatchTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProposeDispatchTest -pl runtime`
Expected: ALL PASS

- [ ] **Step 13: Commit**

```
feat(#395): PROPOSE commitment lifecycle — open, RESPONSE non-fulfillment, default deadline

Refs #395
```

---

### Task 3: StoredMessageTypePolicy and CorrelationIntegrityChecker (runtime module)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java:13-41`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/CorrelationIntegrityChecker.java:47-76`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/ProposeEnforcementTest.java` (new)

**Interfaces:**
- Consumes: `MessageType.PROPOSE` from Task 1

- [ ] **Step 1: Write failing test — PROPOSE is hard-enforced on typed channels**

```java
@Test
void propose_hard_enforced_on_channel_with_denied_types() {
    var channel = channelWithDeniedTypes(Set.of(MessageType.PROPOSE));
    var policy = new StoredMessageTypePolicy();

    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(channel, MessageType.PROPOSE));
}

@Test
void propose_advisory_returns_null() {
    var channel = channelWithDeniedTypes(Set.of(MessageType.PROPOSE));
    var policy = new StoredMessageTypePolicy();

    assertNull(policy.advisory(channel, MessageType.PROPOSE));
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — PROPOSE falls through the early-return guard in validate()

- [ ] **Step 3: Update validate() and advisory()**

In `StoredMessageTypePolicy.java`:

validate() line 14: `if (type != MessageType.COMMAND && type != MessageType.QUERY && type != MessageType.PROPOSE) return;`

advisory() line 28: `if (type == MessageType.COMMAND || type == MessageType.QUERY || type == MessageType.PROPOSE) return null;`

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS

- [ ] **Step 5: Write test — RESPONSE on PROPOSE generates no advisory in CorrelationIntegrityChecker**

```java
@Test
void response_on_propose_generates_no_resolution_advisory() {
    when(commitmentStore.findByCorrelationId("corr-1"))
            .thenReturn(Optional.of(new Commitment(
                    UUID.randomUUID(), "corr-1", channelId, MessageType.PROPOSE,
                    "proposer", "responder", CommitmentState.OPEN,
                    null, null, null, null, null, "default", Instant.now())));

    var dispatch = MessageDispatch.builder(channelId, "responder", MessageType.RESPONSE)
            .content("counter-proposal")
            .correlationId("corr-1")
            .inReplyTo(1L)
            .actorType(ActorType.AGENT)
            .build();

    List<String> advisories = checker.check(dispatch, channelId);
    assertTrue(advisories.stream().noneMatch(a -> a.contains("RESPONSE used to resolve")));
}
```

- [ ] **Step 6: Update CorrelationIntegrityChecker**

In `CorrelationIntegrityChecker.java` line 57, the existing check only fires for COMMAND. PROPOSE needs no advisory for RESPONSE. The existing code already doesn't flag RESPONSE-on-PROPOSE (it only flags RESPONSE-on-COMMAND). Verify this by running the test — it should pass without code changes. If the test passes, no code change is needed here.

If a PROPOSE-specific advisory is wanted for truly wrong types (e.g. using DONE/FAILURE to resolve a QUERY), the existing QUERY check at line 60-62 already handles that. PROPOSE resolution matching follows the COMMAND pattern — DONE/DECLINE/FAILURE are all valid.

- [ ] **Step 7: Run all enforcement tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProposeEnforcementTest -pl runtime`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat(#395): PROPOSE hard-enforcement in StoredMessageTypePolicy, correlation integrity check

Refs #395
```

---

### Task 4: QhorusMcpTools — documentation and artefact release guard (runtime module)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java:863-1003`

**Interfaces:**
- Consumes: `MessageType.PROPOSE` from Task 1, commitment store from Task 2

- [ ] **Step 1: Update send_message @Tool description**

In QhorusMcpTools.java, find the `@Tool` annotation on `sendMessage` (~line 863). Update the description string to include PROPOSE in all relevant parameter descriptions:
- type parameter: add PROPOSE to the list
- correlation_id: change "auto-generated for QUERY and COMMAND" to "auto-generated for QUERY, COMMAND, and PROPOSE"
- deadline: change "Only meaningful for QUERY and COMMAND" to "Only meaningful for QUERY, COMMAND, and PROPOSE"

- [ ] **Step 2: Fix artefact claim release guard**

In QhorusMcpTools.java ~line 983, add a PROPOSE check to the artefact release condition:

```java
// Before:
if (dispatchResult.correlationId() != null && (msgType == MessageType.RESPONSE || msgType == MessageType.DONE
        || msgType == MessageType.DECLINE || msgType == MessageType.FAILURE)) {

// After — exclude RESPONSE when the commitment is for a PROPOSE:
boolean isCommitmentResolving = msgType == MessageType.DONE
        || msgType == MessageType.DECLINE || msgType == MessageType.FAILURE;
if (!isCommitmentResolving && msgType == MessageType.RESPONSE && dispatchResult.correlationId() != null) {
    // RESPONSE resolves QUERY/COMMAND but not PROPOSE — check commitment type
    var commitment = commitmentStore.findByCorrelationId(dispatchResult.correlationId());
    isCommitmentResolving = commitment.isEmpty()
            || commitment.get().messageType() != MessageType.PROPOSE;
}
if (dispatchResult.correlationId() != null && isCommitmentResolving) {
```

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS (compile + tests pass across all modules)

- [ ] **Step 4: Commit**

```
feat(#395): send_message PROPOSE documentation, artefact release guard for PROPOSE commitments

Refs #395
```

---

### Task 5: Notification bridge — PROPOSED kind (notification-bridge module)

**Files:**
- Modify: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusObligationEvent.java`
- Modify: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/NotificationBridgeObserver.java:39-44`
- Modify: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusSubscriptionBootstrap.java`
- Test: `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/NotificationBridgeObserverTest.java` (existing — add test)

**Interfaces:**
- Consumes: `MessageType.PROPOSE` from Task 1

- [ ] **Step 1: Add PROPOSED to Kind enum**

In `QhorusObligationEvent.java`, add `PROPOSED` to the `Kind` enum after `ASSIGNED`:

```java
public enum Kind {
    ASSIGNED, PROPOSED, FULFILLED, FAILED, DECLINED, EXPIRED
}
```

Update `type()` method if it constructs the event type string from the kind.

- [ ] **Step 2: Add PROPOSE case to NotificationBridgeObserver.onMessage()**

In `NotificationBridgeObserver.java` line 39-44, add PROPOSE:

```java
switch (event.messageType()) {
    case COMMAND -> fireAssigned(event);
    case PROPOSE -> fireProposed(event);
    case DONE -> fireResolved(event, QhorusObligationEvent.Kind.FULFILLED);
    case FAILURE -> fireResolved(event, QhorusObligationEvent.Kind.FAILED);
    default -> {}
}
```

Add `fireProposed()` method — same structure as `fireAssigned()` but with `Kind.PROPOSED`:

```java
private void fireProposed(MessageReceivedEvent event) {
    Optional<Commitment> commitment = commitmentStore.findByCorrelationId(event.correlationId());
    if (commitment.isEmpty()) {
        LOG.debugf("No commitment for correlationId=%s — skipping PROPOSE notification", event.correlationId());
        return;
    }
    String obligor = commitment.get().obligor();
    if (obligor == null || obligor.isBlank()) {
        return;
    }
    fire(new QhorusObligationEvent(
            QhorusObligationEvent.Kind.PROPOSED,
            event.tenancyId(),
            obligor,
            commitment.get().requester(),
            event.channelId(),
            event.channelName(),
            event.senderId(),
            event.correlationId(),
            truncate(event.content(), MAX_CONTENT_LENGTH)));
}
```

- [ ] **Step 3: Add PROPOSED subscription to QhorusSubscriptionBootstrap**

Add a 6th default subscription for the PROPOSED kind alongside the existing 5 (ASSIGNED, FULFILLED, FAILED, DECLINED, EXPIRED). Follow the same pattern.

- [ ] **Step 4: Write test**

In `NotificationBridgeObserverTest.java`, add:

```java
@Test
void propose_fires_proposed_event() {
    when(commitmentStore.findByCorrelationId("corr-1"))
            .thenReturn(Optional.of(commitment("proposer", "receiver")));

    observer.onMessage(event(MessageType.PROPOSE, "corr-1", "proposer"));

    verify(dataSource).add(argThat(e ->
            ((QhorusObligationEvent) e).kind() == QhorusObligationEvent.Kind.PROPOSED));
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#395): notification bridge PROPOSED kind for PROPOSE message type

Refs #395
```

---

### Task 6: MessageTaxonomyTest and full build verification (examples module)

**Files:**
- Modify: `examples/type-system/src/test/java/io/casehub/qhorus/examples/taxonomy/MessageTaxonomyTest.java`

**Interfaces:**
- Consumes: `MessageType.PROPOSE` from Task 1

- [ ] **Step 1: Update MessageTaxonomyTest**

Add PROPOSE to the enum value count assertion (9 → 10). Add method assertions:

```java
@Test
void propose_methods() {
    assertThat(MessageType.PROPOSE.isAgentVisible()).isTrue();
    assertThat(MessageType.PROPOSE.requiresCorrelationId()).isTrue();
    assertThat(MessageType.PROPOSE.requiresContent()).isTrue();
    assertThat(MessageType.PROPOSE.requiresTarget()).isFalse();
    assertThat(MessageType.PROPOSE.isTerminal()).isFalse();
}
```

- [ ] **Step 2: Run type-system tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/type-system`
Expected: ALL PASS

- [ ] **Step 3: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile, all tests pass. This catches any exhaustive switch statements in other modules that break with the new enum value.

- [ ] **Step 4: Commit**

```
feat(#395): MessageTaxonomyTest coverage for PROPOSE

Refs #395
```

---

### Task 7: Doc revisions (project docs)

**Files:**
- Modify: `docs/adr/0005-message-type-taxonomy-theoretical-foundation.md`
- Modify: `docs/normative-layer.md`
- Modify: `docs/guides/consumer-guide.md`
- Modify: `docs/guides/contributor-guide.md`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: All prior tasks

- [ ] **Step 1: Amend ADR-0005**

Add an "Amendment" section at the end (before References):

```markdown
## Amendment: PROPOSE and the Commissive Gap (2026-08-14)

STATUS was originally classified as a commissive. On review, STATUS is
assertive — it extends an existing obligation window ("I am still working
on it") rather than creating a new conditional commitment. A Searlean
commissive requires the speaker to bind themselves to a future course of
action contingent on some condition.

PROPOSE fills this gap: "I will do X if you agree to Y" is a textbook
conditional commissive. It opens a commitment (like COMMAND) but RESPONSE
does not auto-fulfill — only DONE (explicit acceptance) fulfills.

### Stopping criterion

A new type is justified only when it occupies a unique cell in the
Searle-category × deontic-effect matrix:

| Searle category | Type | Deontic effect |
|---|---|---|
| Directive (epistemic) | QUERY | Receiver obligation to inform |
| Directive (action) | COMMAND | Receiver obligation to execute |
| Assertive | RESPONSE, DECLINE, STATUS | Truth commitment / obligation extension |
| Commissive | PROPOSE | Conditional sender-obligation; receiver responds |
| Declaration | HANDOFF, DONE, FAILURE | Institutional reality change |
| Perlocutionary | EVENT | No deontic footprint |

Future candidates must demonstrate an empty cell in this matrix.
```

Update the "Complete Type Definitions" table to add PROPOSE and correct STATUS from "Commissive" to "Assertive".

Update the "Completeness Argument" section — change "9-type" to "10-type" and add PROPOSE as lifecycle state 1b (Created — conditional).

- [ ] **Step 2: Update normative-layer.md**

Add PROPOSE to the speech act coverage table. Add a section on commissive semantics.

- [ ] **Step 3: Update consumer-guide.md**

Add PROPOSE to the message type reference table. Add usage example for negotiation.

- [ ] **Step 4: Update contributor-guide.md**

Update type system internals section. Document RESPONSE non-fulfillment mechanism.

- [ ] **Step 5: Update CLAUDE.md**

Search for "9-type" and update to "10-type". Add PROPOSE to the MessageType enum description. Add `default-propose-deadline` to config documentation.

- [ ] **Step 6: Commit**

```
docs(#395): amend ADR-0005 completeness argument, update guides for PROPOSE

Refs #395
```
