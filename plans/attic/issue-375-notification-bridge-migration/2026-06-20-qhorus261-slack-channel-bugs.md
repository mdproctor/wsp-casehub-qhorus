# casehub-qhorus-slack-channel Bug Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 8 known implementation bugs in `casehub-qhorus-slack-channel` (qhorus#261) against the r27 spec, plus one fix in `casehub-connectors` (connectors#22).

**Architecture:** The existing implementation at git sha `fc05f29` has 8 correctness bugs against the r27 spec (in `wsp-casehub-qhorus` branch `issue-261-slack-channel-backend`, file `specs/issue-261-slack-channel-backend/2026-06-17-slack-channel-backend-design.md`). Each task fixes one bug: write the failing test, verify it fails, apply the minimal fix, verify it passes, commit. Bugs are independent and can be fixed in order.

**Tech Stack:** Java 21 on JVM 26, Quarkus 3.32.2, Panache ORM, JUnit 5, Mockito, H2

**Build commands:**
- qhorus module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-channel`
- connectors module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook`
- Full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

**Work branch:** `issue-261-slack-channel-backend` in both `casehub/qhorus` and `wsp-casehub-qhorus`

---

## Task 1: connectors#22 — Filter Slack event subtypes

**Files:**
- Modify: `casehub-connectors/webhook/src/main/java/io/casehub/connectors/webhook/SlackInboundConnector.java`
- Modify: `casehub-connectors/webhook/src/test/java/io/casehub/connectors/webhook/SlackInboundConnectorTest.java`

**Context:** `parseMessages()` filters bot messages and non-`message` types but not event subtypes. `message_changed` (edits), `message_deleted`, and `channel_join` all have `type="message"` and no `bot_id`, so they pass through and generate unintended Qhorus COMMAND/QUERY messages for every Slack edit. This must be fixed before shipping to a real workspace.

- [ ] **Step 1: Write the failing test**

Add to `SlackInboundConnectorTest` (look at existing test for `parseMessages` in that file):

```java
@Test
void messageChangedSubtype_isFiltered() {
    // Simulate a Slack message_changed event (subtype present)
    String body = """
        {"type":"event_callback","team_id":"T123","event":{
            "type":"message","subtype":"message_changed",
            "message":{"text":"edited","user":"U1","ts":"1.2"},
            "channel":"C1","ts":"1.2"}}""";
    WebhookRequest req = new WebhookRequest(HttpMethod.POST, body, Map.of(
        "x-slack-request-timestamp", String.valueOf(Instant.now().getEpochSecond()),
        "x-slack-signature", signatureFor(body, signingSecret)));
    WebhookResult result = connector.handle(req);
    assertThat(result).isInstanceOf(WebhookResult.Ignored.class);
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/mdproctor/claude/casehub/connectors
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackInboundConnectorTest#messageChangedSubtype_isFiltered -pl webhook
```
Expected: FAIL (result is `Delivered`, not `Ignored`)

- [ ] **Step 3: Add the subtype filter**

In `SlackInboundConnector.java`, inside `parseMessages()`, add this immediately after the `!\"message\".equals(event.getString(\"type\", null))` guard:

```java
// Filter Slack message subtypes (message_changed, message_deleted, channel_join, etc.)
// Only primary message events (no subtype) should enter the inbound pipeline.
if (event.getString("subtype", null) != null) return messages;
```

The full guard block becomes:
```java
if (event.containsKey("bot_id")) return messages;
if (!"message".equals(event.getString("type", null))) return messages;
if (event.getString("subtype", null) != null) return messages;  // ADD THIS LINE
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackInboundConnectorTest -pl webhook
```
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add webhook/src/main/java/io/casehub/connectors/webhook/SlackInboundConnector.java webhook/src/test/java/io/casehub/connectors/webhook/SlackInboundConnectorTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "fix(slack-inbound): filter message subtypes (edited/deleted/join) — Closes #22"
```

---

## Task 2: @ObservesAsync → @Observes on onChannelInitialised()

**Files:**
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java`
- Modify: `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java`

**Context:** `ChannelGateway.initChannel()` uses synchronous `channelInitialisedEvents.fire()` (line 82 of ChannelGateway.java). CDI routes synchronous `fire()` exclusively to `@Observes` observers. `@ObservesAsync` is for `fireAsync()` only — with the wrong annotation, `onChannelInitialised()` never executes, so no channel ever registers with the backend. This is the most critical bug.

- [ ] **Step 1: Write unit tests for onChannelInitialised() logic**

These tests call the method directly (CDI-free). They also catch any regression in the method body.

Add to `SlackChannelBackendTest`:

```java
@Test
void onChannelInitialised_withBinding_populatesCachesAndRegisters() {
    UUID channelId = UUID.randomUUID();
    String slackChId = "C456";
    SlackBotBinding binding = new SlackBotBinding();
    binding.channelId = channelId;
    binding.slackChannelId = slackChId;
    binding.workspaceId = "T456";
    binding.createdAt = Instant.now();

    when(bindingStore.findByChannelId(channelId)).thenReturn(Optional.of(binding));
    when(threadCacheStore.findByChannelId(channelId)).thenReturn(List.of());

    ChannelInitialisedEvent event = new ChannelInitialisedEvent(channelId, "my-channel", false);
    backend.onChannelInitialised(event);

    assertThat(backend.bindingCache).containsKey(channelId);
    assertThat(backend.slackToChannel).containsKey(slackChId);
    verify(gateway).registerBackend(eq(channelId), eq(backend), eq("human_participating"));
}

@Test
void onChannelInitialised_withoutBinding_doesNothing() {
    UUID channelId = UUID.randomUUID();
    when(bindingStore.findByChannelId(channelId)).thenReturn(Optional.empty());

    backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "no-binding", false));

    assertThat(backend.bindingCache).doesNotContainKey(channelId);
    verify(gateway, never()).registerBackend(any(), any(), any());
}

@Test
void onChannelInitialised_restartRecovery_loadsThreadCacheFromDb() {
    UUID channelId = UUID.randomUUID();
    UUID corrId = UUID.randomUUID();
    String threadTs = "1718567890.123456";

    SlackBotBinding binding = new SlackBotBinding();
    binding.channelId = channelId;
    binding.slackChannelId = "C789";
    binding.workspaceId = "T789";
    binding.createdAt = Instant.now();

    SlackThreadCache entry = new SlackThreadCache();
    entry.id = new SlackThreadCacheId(channelId, corrId);
    entry.threadTs = threadTs;

    when(bindingStore.findByChannelId(channelId)).thenReturn(Optional.of(binding));
    when(threadCacheStore.findByChannelId(channelId)).thenReturn(List.of(entry));

    backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "recovery-channel", true));

    assertThat(backend.threadCache.get(channelId)).containsEntry(corrId, threadTs);
}
```

- [ ] **Step 2: Run tests to verify they pass** (method logic is correct, only annotation is wrong)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackChannelBackendTest -pl slack-channel
```
Expected: new tests PASS (they call the method directly, bypassing CDI routing)

- [ ] **Step 3: Fix the annotation**

In `SlackChannelBackend.java`, line ~97, change:

```java
// BEFORE:
public void onChannelInitialised(@ObservesAsync ChannelInitialisedEvent event) {

// AFTER:
public void onChannelInitialised(@Observes ChannelInitialisedEvent event) {
```

Also update imports: remove `import jakarta.enterprise.event.ObservesAsync;`, add `import jakarta.enterprise.event.Observes;`.

- [ ] **Step 4: Run all backend tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackChannelBackendTest -pl slack-channel
```
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(slack-channel): @Observes on onChannelInitialised (was @ObservesAsync — gateway uses synchronous fire()) — Refs #261"
```

---

## Task 3: UNIQUE(slack_channel_id) in V23 DDL

**Files:**
- Modify: `slack-channel/src/main/resources/db/qhorus/migration/V23__slack_bot_binding.sql`

**Context:** Without `UNIQUE(slack_channel_id)`, two Qhorus channels can bind to the same Slack channel ID. When both initialise, `slackToChannel["C1"]` is overwritten by the second binding. The first channel silently stops receiving inbound Slack messages.

- [ ] **Step 1: Write the schema test**

Find or create `SlackFlywaySchemaTest.java` in `slack-channel/src/test/java/io/casehub/qhorus/slack/`. Add (or add to existing schema test):

```java
@Test
void v23_slack_bot_binding_has_unique_slack_channel_id() throws Exception {
    // Verify the UNIQUE constraint on slack_channel_id exists
    try (Connection conn = ds.getConnection();
         ResultSet rs = conn.getMetaData().getIndexInfo(null, null, "slack_bot_binding", true, false)) {
        boolean found = false;
        while (rs.next()) {
            String colName = rs.getString("COLUMN_NAME");
            if ("slack_channel_id".equalsIgnoreCase(colName)) {
                found = true;
            }
        }
        assertThat(found).as("UNIQUE index on slack_channel_id must exist").isTrue();
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackFlywaySchemaTest#v23_slack_bot_binding_has_unique_slack_channel_id -pl slack-channel
```
Expected: FAIL — no unique index found

- [ ] **Step 3: Add the constraint to V23**

Edit `V23__slack_bot_binding.sql`:

```sql
CREATE TABLE slack_bot_binding (
    channel_id       UUID         NOT NULL,
    slack_channel_id VARCHAR(32)  NOT NULL,
    workspace_id     VARCHAR(32)  NOT NULL,
    created_at       TIMESTAMP    NOT NULL,
    CONSTRAINT pk_slack_bot_binding      PRIMARY KEY (channel_id),
    CONSTRAINT fk_slack_binding_channel  FOREIGN KEY (channel_id) REFERENCES channel(id),
    CONSTRAINT uq_slack_bot_slack_id     UNIQUE (slack_channel_id)
);
```

- [ ] **Step 4: Run to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackFlywaySchemaTest -pl slack-channel
```
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add slack-channel/src/main/resources/db/qhorus/migration/V23__slack_bot_binding.sql slack-channel/src/test/java/io/casehub/qhorus/slack/SlackFlywaySchemaTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(slack-channel): add UNIQUE(slack_channel_id) to V23 DDL — Refs #261"
```

---

## Task 4: Fix rootTs for unknown thread replies in onInboundMessage()

**Files:**
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java`
- Modify: `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java`

**Context:** When a human replies in a Slack thread with no active anchor (unknown thread), the code generates a new corrId and stores `slackTs` (the reply's own timestamp) as the anchor. Slack requires `thread_ts` to equal the ROOT message's timestamp. Passing a reply's ts causes the Slack API to fail or create unexpected nesting. Fix: use `slackThreadTs` as rootTs for thread replies.

- [ ] **Step 1: Write the failing test**

Add to `SlackChannelBackendTest`:

```java
@Test
void onInboundMessage_unknownThreadReply_anchorsWithThreadRootTs() throws Exception {
    String slackChannelId = "C123ABC";
    ChannelRef channelRef = new ChannelRef(channelId, "test-channel");
    backend.slackToChannel.put(slackChannelId, channelRef);

    String slackTs = "1718567890.999999";       // reply's own ts
    String slackThreadTs = "1718567890.111111"; // thread root ts (different from slackTs)

    Map<String, String> meta = Map.of(
        "slack-ts", slackTs,
        "slack-thread-ts", slackThreadTs
    );

    // No existing anchor → findCorrelationId returns empty
    when(threadCacheStore.findCorrelationId(channelId, slackThreadTs)).thenReturn(Optional.empty());

    io.casehub.connectors.InboundMessage msg =
        new io.casehub.connectors.InboundMessage(
            io.casehub.connectors.InboundConnectorIds.SLACK_INBOUND,
            "U123", slackChannelId, "hello", Instant.now(), meta);

    backend.onInboundMessage(msg).toCompletableFuture().join();

    // CRITICAL: rootTs must be slackThreadTs (root), NOT slackTs (reply)
    verify(threadCacheStore).save(eq(channelId), any(UUID.class), eq(slackThreadTs));
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackChannelBackendTest#onInboundMessage_unknownThreadReply_anchorsWithThreadRootTs -pl slack-channel
```
Expected: FAIL — `save()` was called with `slackTs` (reply ts), not `slackThreadTs` (root ts)

- [ ] **Step 3: Fix the rootTs computation**

In `SlackChannelBackend.java`, in the `onInboundMessage()` method, find the `if (corrIdStr == null)` block and replace the anchor-write section:

```java
// BEFORE (wrong):
if (slackTs != null) {
    threadCacheStore.save(channelRef.id(), corrId, slackTs);
    threadCache.computeIfAbsent(channelRef.id(), k -> new ConcurrentHashMap<>())
            .put(corrId, slackTs);
}

// AFTER (correct):
// For thread replies: rootTs = slackThreadTs (the thread root, NOT the reply's own ts).
// For top-level messages: rootTs = slackTs (this message IS the root).
// Slack's thread_ts parameter must equal the root message's timestamp.
String rootTs = (slackThreadTs != null && !slackThreadTs.equals(slackTs)) ? slackThreadTs : slackTs;
if (rootTs != null) {
    // In-memory first (never throws), then DB best-effort with ORDERING INVARIANT
    threadCache.computeIfAbsent(channelRef.id(), k -> new ConcurrentHashMap<>())
            .put(corrId, rootTs);
    try {
        threadCacheStore.save(channelRef.id(), corrId, rootTs);
    } catch (Exception e) {
        LOG.warnf(e, "Thread anchor DB write failed for channel=%s corrId=%s — " +
                "in-memory anchor intact; restart recovery disabled for this corrId",
                channelRef.id(), corrId);
    }
}
```

- [ ] **Step 4: Run to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackChannelBackendTest -pl slack-channel
```
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(slack-channel): rootTs=slackThreadTs for unknown thread replies (not slackTs) — Refs #261"
```

---

## Task 5: Slash-command COMMAND detection in SlackInboundNormaliser

**Files:**
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackInboundNormaliser.java`
- Modify: `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackInboundNormaliserTest.java`

**Context:** The normaliser currently only produces RESPONSE or QUERY. Messages starting with `/` (e.g. `/approve`, `/cancel`) should produce COMMAND type so agents can handle slash commands distinctly from freeform queries.

- [ ] **Step 1: Write the failing tests**

Add to `SlackInboundNormaliserTest`:

```java
@Test
void slashCommand_producesCommand() {
    InboundHumanMessage msg = new InboundHumanMessage(
            "U123", "/approve", Instant.now(), Map.of("slack-ts", "1.1"), null, null);
    NormalisedMessage result = normaliser.normalise(channel, msg);
    assertThat(result.type()).isEqualTo(MessageType.COMMAND);
}

@Test
void slashCommand_nullContent_doesNotThrowNpe_producesQuery() {
    // Defensive: SlackInboundConnector always provides non-null content via getString("text","")
    // but guard against future callers.
    InboundHumanMessage msg = new InboundHumanMessage(
            "U123", null, Instant.now(), Map.of("slack-ts", "1.1"), null, null);
    NormalisedMessage result = normaliser.normalise(channel, msg);
    assertThat(result.type()).isEqualTo(MessageType.QUERY); // null content → QUERY, no NPE
}

@Test
void slashCommand_emptyString_producesQuery() {
    // Empty string (media-only Slack messages) → QUERY
    InboundHumanMessage msg = new InboundHumanMessage(
            "U123", "", Instant.now(), Map.of("slack-ts", "1.1"), null, null);
    NormalisedMessage result = normaliser.normalise(channel, msg);
    assertThat(result.type()).isEqualTo(MessageType.QUERY);
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackInboundNormaliserTest -pl slack-channel
```
Expected: `slashCommand_producesCommand` FAILS (produces QUERY instead of COMMAND)

- [ ] **Step 3: Add COMMAND detection to normaliser**

Replace the type inference block in `SlackInboundNormaliser.java`:

```java
// BEFORE:
MessageType type = (slackThreadTs != null && !slackThreadTs.equals(slackTs)
        && raw.correlationId() != null)
        ? MessageType.RESPONSE
        : MessageType.QUERY;

// AFTER:
final MessageType type;
if (content != null && content.startsWith("/")) {
    type = MessageType.COMMAND;            // slash command — agent-executable action
} else if (slackThreadTs != null && !slackThreadTs.equals(slackTs)
           && raw.correlationId() != null) {
    type = MessageType.RESPONSE;           // thread reply with active anchor
} else {
    type = MessageType.QUERY;              // top-level message or no-anchor thread reply
}
```

- [ ] **Step 4: Run to verify all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackInboundNormaliserTest -pl slack-channel
```
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add slack-channel/src/main/java/io/casehub/qhorus/slack/SlackInboundNormaliser.java slack-channel/src/test/java/io/casehub/qhorus/slack/SlackInboundNormaliserTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(slack-channel): add slash-command COMMAND detection to SlackInboundNormaliser — Refs #261"
```

---

## Task 6: Add evict() method; fix SlackBindingResource.delete() encapsulation

**Files:**
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java`
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackBindingResource.java`
- Modify: `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java`

**Context:** `SlackBindingResource.delete()` accesses `backend.bindingCache`, `backend.slackToChannel`, and `backend.threadCache` directly. The spec defines a package-private `evict(UUID channelId)` method to encapsulate this. Direct field access couples `SlackBindingResource` to internal implementation details and would break if the map structure changes.

- [ ] **Step 1: Write the failing test**

Add to `SlackChannelBackendTest`:

```java
@Test
void evict_removesFromAllInMemoryMaps() {
    UUID corrId = UUID.randomUUID();
    String slackChannelId = "C999";
    SlackBotBinding binding = new SlackBotBinding();
    binding.channelId = channelId;
    binding.slackChannelId = slackChannelId;
    binding.workspaceId = "T999";
    binding.createdAt = Instant.now();

    // Populate all maps
    backend.bindingCache.put(channelId, binding);
    backend.slackToChannel.put(slackChannelId, channelRef);
    backend.threadCache.computeIfAbsent(channelId, k -> new java.util.concurrent.ConcurrentHashMap<>())
            .put(corrId, "1.1");

    backend.evict(channelId);

    assertThat(backend.bindingCache).doesNotContainKey(channelId);
    assertThat(backend.slackToChannel).doesNotContainKey(slackChannelId);
    assertThat(backend.threadCache).doesNotContainKey(channelId);
}
```

This test will FAIL because `evict()` doesn't exist yet.

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackChannelBackendTest#evict_removesFromAllInMemoryMaps -pl slack-channel
```
Expected: FAIL — compilation error (method `evict` not found)

- [ ] **Step 3: Add evict() to SlackChannelBackend**

Add this method to `SlackChannelBackend.java` (alongside close() and open()):

```java
/**
 * Evicts in-memory routing state for the channel.
 * Called from SlackBindingResource.delete() on admin unbinding.
 * DB thread cache rows are preserved — TTL cleanup handles them;
 * in-flight fanOut() calls that already captured the binding can still complete.
 */
void evict(UUID channelId) {
    SlackBotBinding binding = bindingCache.remove(channelId);
    if (binding != null) slackToChannel.remove(binding.slackChannelId);
    threadCache.remove(channelId);
}
```

- [ ] **Step 4: Fix SlackBindingResource.delete() to use evict()**

Replace the `delete()` body in `SlackBindingResource.java`:

```java
@DELETE
@Path("/{channelId}")
public Response delete(@PathParam("channelId") UUID channelId) {
    backend.evict(channelId);
    gateway.deregisterBackend(channelId, SlackChannelBackend.BACKEND_ID);
    bindingStore.deleteByChannelId(channelId);
    return Response.noContent().build();
}
```

- [ ] **Step 5: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackChannelBackendTest -pl slack-channel
```
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java slack-channel/src/main/java/io/casehub/qhorus/slack/SlackBindingResource.java slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(slack-channel): add evict() method; SlackBindingResource.delete() no longer accesses backend fields directly — Refs #261"
```

---

## Task 7: Fix PUT flow — reorder validations, add ChannelConnectorBinding check, blank token guard, evict+deleteAll before save

**Files:**
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackBindingResource.java`
- Modify: `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackBindingResourceTest.java` (create if missing)

**Context:** The current `put()` method has three bugs:
1. It saves the binding BEFORE checking channel existence → orphaned binding on 404
2. It has no mutual exclusion check for existing `ChannelConnectorBinding` → DuplicateParticipatingBackendException surfaces as HTTP 500
3. It validates the credential key but not whether the resolved value is blank → empty token accepted at bind time, fails silently at delivery

The correct order: channel check → binding conflict check → credential + blank check → evict+deleteAll → save → initChannel

Also need to inject `ChannelBindingStore` for the mutual exclusion check, and call `evict()` + `threadCacheStore.deleteAllByChannelId()` before save to handle rebind cleanly.

- [ ] **Step 1: Write the failing tests**

Create `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackBindingResourceTest.java`:

```java
package io.casehub.qhorus.slack;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

import java.util.Optional;
import java.util.UUID;

import jakarta.ws.rs.core.Response;

import org.eclipse.microprofile.config.Config;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;

class SlackBindingResourceTest {

    private SlackBotBindingStore bindingStore;
    private ChannelService channelService;
    private ChannelGateway gateway;
    private SlackChannelBackend backend;
    private ChannelBindingStore channelBindingStore;
    private SlackThreadCacheStore threadCacheStore;
    private Config config;
    private SlackBindingResource resource;

    private final UUID channelId = UUID.randomUUID();
    private final String workspaceId = "T123ABC";
    private final String slackChannelId = "C123ABC";
    private final String credKey = "casehub.qhorus.slack-channel.credentials." + workspaceId;

    @BeforeEach
    void setUp() {
        bindingStore = mock(SlackBotBindingStore.class);
        channelService = mock(ChannelService.class);
        gateway = mock(ChannelGateway.class);
        backend = mock(SlackChannelBackend.class);
        channelBindingStore = mock(ChannelBindingStore.class);
        threadCacheStore = mock(SlackThreadCacheStore.class);
        config = mock(Config.class);

        resource = new SlackBindingResource(
                bindingStore, channelService, gateway, backend, channelBindingStore, threadCacheStore, config);

        // Default: channel exists, no existing generic binding, valid non-blank token
        Channel ch = new Channel();
        ch.id = channelId;
        ch.name = "test-channel";
        when(channelService.findById(channelId)).thenReturn(Optional.of(ch));
        when(channelBindingStore.findByChannelId(channelId)).thenReturn(Optional.empty());
        when(config.getValue(credKey, String.class)).thenReturn("xoxb-valid-token");
    }

    @Test
    void put_channelNotFound_returns404() {
        when(channelService.findById(channelId)).thenReturn(Optional.empty());

        Response r = resource.put(channelId, new SlackBindingRequest(workspaceId, slackChannelId));

        assertThat(r.getStatus()).isEqualTo(404);
        verify(bindingStore, never()).save(any());
    }

    @Test
    void put_channelConnectorBindingExists_returns409() {
        when(channelBindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(new ChannelConnectorBinding()));

        Response r = resource.put(channelId, new SlackBindingRequest(workspaceId, slackChannelId));

        assertThat(r.getStatus()).isEqualTo(409);
        verify(bindingStore, never()).save(any());
    }

    @Test
    void put_credentialKeyMissing_returns400() {
        when(config.getValue(credKey, String.class))
                .thenThrow(new java.util.NoSuchElementException("No config value"));

        Response r = resource.put(channelId, new SlackBindingRequest(workspaceId, slackChannelId));

        assertThat(r.getStatus()).isEqualTo(400);
        verify(bindingStore, never()).save(any());
    }

    @Test
    void put_credentialBlank_returns400() {
        when(config.getValue(credKey, String.class)).thenReturn("");  // blank token

        Response r = resource.put(channelId, new SlackBindingRequest(workspaceId, slackChannelId));

        assertThat(r.getStatus()).isEqualTo(400);
        verify(bindingStore, never()).save(any());
    }

    @Test
    void put_validRequest_evictsBeforeSaveAndReturns200() {
        Response r = resource.put(channelId, new SlackBindingRequest(workspaceId, slackChannelId));

        assertThat(r.getStatus()).isEqualTo(200);
        // evict must be called before save to clean stale routing state
        var inOrder = inOrder(backend, bindingStore);
        inOrder.verify(backend).evict(channelId);
        inOrder.verify(bindingStore).save(any(SlackBotBinding.class));
    }

    @Test
    void delete_returns204_idempotent() {
        Response r = resource.delete(channelId);
        assertThat(r.getStatus()).isEqualTo(204);
    }
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackBindingResourceTest -pl slack-channel
```
Expected: Compilation error (constructor doesn't match) and multiple test failures

- [ ] **Step 3: Fix SlackBindingResource.java**

Replace the entire class with the correct implementation:

```java
package io.casehub.qhorus.slack;

import java.time.Instant;
import java.util.NoSuchElementException;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import org.eclipse.microprofile.config.Config;

import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.gateway.DuplicateParticipatingBackendException;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;

/**
 * Manages Slack bot bindings — associates a Qhorus channel with a Slack channel.
 * No auth annotations — network isolation is the current security boundary.
 *
 * <p>put() is intentionally NOT @Transactional. See spec Known Limitations for
 * the split-transaction rationale and why @Transactional would break the
 * DuplicateParticipatingBackendException catch pattern.
 */
@Path("/slack-channel/bindings")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class SlackBindingResource {

    private final SlackBotBindingStore bindingStore;
    private final ChannelService channelService;
    private final ChannelGateway gateway;
    private final SlackChannelBackend backend;
    private final ChannelBindingStore channelBindingStore;
    private final SlackThreadCacheStore threadCacheStore;
    private final Config config;

    public SlackBindingResource(SlackBotBindingStore bindingStore,
                                ChannelService channelService,
                                ChannelGateway gateway,
                                SlackChannelBackend backend,
                                ChannelBindingStore channelBindingStore,
                                SlackThreadCacheStore threadCacheStore,
                                Config config) {
        this.bindingStore = bindingStore;
        this.channelService = channelService;
        this.gateway = gateway;
        this.backend = backend;
        this.channelBindingStore = channelBindingStore;
        this.threadCacheStore = threadCacheStore;
        this.config = config;
    }

    @PUT
    @Path("/{channelId}")
    public Response put(@PathParam("channelId") UUID channelId, SlackBindingRequest req) {
        // Step 1: Channel must exist
        var channel = channelService.findById(channelId).orElse(null);
        if (channel == null) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity("Channel not found: " + channelId).build();
        }

        // Step 2: No existing generic ChannelConnectorBinding (mutual exclusion)
        if (channelBindingStore.findByChannelId(channelId).isPresent()) {
            return Response.status(Response.Status.CONFLICT)
                    .entity("Channel already has a generic connector binding — cannot add Slack bot binding")
                    .build();
        }

        // Step 3: Credential must exist and be non-blank
        String credKey = "casehub.qhorus.slack-channel.credentials." + req.workspaceId();
        String token;
        try {
            token = config.getValue(credKey, String.class);
        } catch (NoSuchElementException e) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity("Credential key not configured: " + credKey).build();
        }
        if (token.isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity("Credential " + credKey + " is configured but blank").build();
        }

        // Step 4: Clean stale in-memory routing (safe no-op for fresh binds; handles rebind)
        backend.evict(channelId);
        threadCacheStore.deleteAllByChannelId(channelId);

        // Step 5: Persist new binding
        SlackBotBinding binding = new SlackBotBinding();
        binding.channelId = channelId;
        binding.slackChannelId = req.slackChannelId();
        binding.workspaceId = req.workspaceId();
        binding.createdAt = Instant.now();
        bindingStore.save(binding);

        // Step 6: Fire ChannelInitialisedEvent so backend self-registers
        // Catch DuplicateParticipatingBackendException inside this method (before @Transactional
        // boundary) to allow bindingStore.deleteByChannelId() to undo the save cleanly.
        // See spec Known Limitations for the split-transaction rationale.
        try {
            gateway.initChannel(channelId, new ChannelRef(channelId, channel.name));
        } catch (DuplicateParticipatingBackendException e) {
            bindingStore.deleteByChannelId(channelId);
            return Response.status(Response.Status.CONFLICT)
                    .entity("Channel already has a participating backend: " + e.getMessage()).build();
        }

        return Response.ok(SlackBindingDto.from(channelId, binding)).build();
    }

    @GET
    @Path("/{channelId}")
    public SlackBindingDto get(@PathParam("channelId") UUID channelId) {
        return bindingStore.findByChannelId(channelId)
                .map(b -> SlackBindingDto.from(channelId, b))
                .orElseThrow(NotFoundException::new);
    }

    @DELETE
    @Path("/{channelId}")
    public Response delete(@PathParam("channelId") UUID channelId) {
        backend.evict(channelId);
        gateway.deregisterBackend(channelId, SlackChannelBackend.BACKEND_ID);
        bindingStore.deleteByChannelId(channelId);
        // DB thread cache rows intentionally NOT deleted — TTL cleanup handles them.
        // After evict(), post() returns early on "no binding" and never reads thread cache.
        return Response.noContent().build();
    }
}
```

- [ ] **Step 4: Run to verify all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SlackBindingResourceTest,SlackChannelBackendTest -pl slack-channel
```
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add slack-channel/src/main/java/io/casehub/qhorus/slack/SlackBindingResource.java slack-channel/src/test/java/io/casehub/qhorus/slack/SlackBindingResourceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(slack-channel): PUT validates channel/binding before save, adds blank-token check, fixes rebind cleanup — Refs #261"
```

---

## Task 8: Inject Config for testable resolveToken()

**Files:**
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java`
- Modify: `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java`

**Context:** `resolveToken()` uses `ConfigProvider.getConfig()` (static access), which prevents CDI injection and forces the fragile anonymous subclass pattern in tests. The correct approach: inject `org.eclipse.microprofile.config.Config` via constructor — standard CDI, directly mockable in unit tests.

- [ ] **Step 1: Update the test setUp() to use injected Config**

In `SlackChannelBackendTest`, update `setUp()` to remove `mockToken()` usage and inject a mock Config:

```java
private Config config;

@BeforeEach
void setUp() {
    bindingStore = mock(SlackBotBindingStore.class);
    threadCacheStore = mock(SlackThreadCacheStore.class);
    slackBotClient = mock(SlackBotClient.class);
    gateway = mock(ChannelGateway.class);
    config = mock(Config.class);

    backend = new SlackChannelBackend(
            bindingStore, threadCacheStore, slackBotClient,
            new SlackInboundNormaliser(), gateway, config);

    SlackBotBinding binding = binding();
    backend.bindingCache.put(channelId, binding);
    // Default: token resolves correctly
    when(config.getValue("casehub.qhorus.slack-channel.credentials." + workspaceId, String.class))
            .thenReturn(token);
}
```

Replace all calls to `mockToken()` in existing tests with `when(config.getValue(...)).thenReturn(token)` in `setUp()`. Delete the `mockToken()` helper entirely.

- [ ] **Step 2: Run to verify it fails (compilation)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl slack-channel
```
Expected: FAIL — `SlackChannelBackend` constructor doesn't accept 6 arguments

- [ ] **Step 3: Update SlackChannelBackend to inject Config**

Add `Config config` as a constructor parameter and update `resolveToken()`:

```java
// Add import at top:
import org.eclipse.microprofile.config.Config;

// In the field declarations:
private final Config config;

// Updated constructor:
public SlackChannelBackend(SlackBotBindingStore bindingStore,
                           SlackThreadCacheStore threadCacheStore,
                           SlackBotClient slackBotClient,
                           SlackInboundNormaliser slackInboundNormaliser,
                           ChannelGateway gateway,
                           Config config) {
    this.bindingStore = bindingStore;
    this.threadCacheStore = threadCacheStore;
    this.slackBotClient = slackBotClient;
    this.slackInboundNormaliser = slackInboundNormaliser;
    this.gateway = gateway;
    this.config = config;
}

// Updated resolveToken():
String resolveToken(String workspaceId) {
    return config.getValue("casehub.qhorus.slack-channel.credentials." + workspaceId, String.class);
}
```

Remove: `import org.eclipse.microprofile.config.ConfigProvider;`

- [ ] **Step 4: Run all slack-channel tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-channel
```
Expected: ALL PASS

- [ ] **Step 5: Full module build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl api,runtime,slack-channel,testing
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(slack-channel): inject Config via constructor for testable resolveToken() — Refs #261"
```

---

## Final verification

- [ ] **Run the full slack-channel test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl slack-channel
```
Expected: ALL PASS

- [ ] **Run the full qhorus build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
Expected: BUILD SUCCESS

- [ ] **Push both repos**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus push
git -C /Users/mdproctor/claude/casehub/connectors push
git -C /Users/mdproctor/claude/public/casehub/qhorus push
```

---

## Self-review: spec coverage check

| Spec requirement | Task |
|---|---|
| @Observes on onChannelInitialised() | Task 2 |
| UNIQUE(slack_channel_id) in V23 | Task 3 |
| rootTs = slackThreadTs for unknown thread replies | Task 4 |
| Slash-command → COMMAND in normaliser | Task 5 |
| evict() method on SlackChannelBackend | Task 6 |
| SlackBindingResource.delete() uses evict() | Task 6 |
| PUT: channel existence check before save | Task 7 |
| PUT: ChannelConnectorBinding mutual exclusion → 409 | Task 7 |
| PUT: blank token → 400 | Task 7 |
| PUT: evict+deleteAll before save (rebind) | Task 7 |
| Injected Config for resolveToken() | Task 8 |
| connectors#22 subtype filter | Task 1 |
