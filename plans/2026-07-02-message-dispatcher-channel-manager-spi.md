# MessageDispatcher and ChannelManager SPI Extraction — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract `MessageDispatcher` and `ChannelManager` service facade interfaces into `casehub-qhorus-api` so Tier 1 consumers can dispatch messages and manage channels without depending on runtime.

**Architecture:** Four new interfaces colocated with their domain types in `api/channel/` and `api/message/`. `ChannelCreateRequest` aligns CSV String fields to `List<String>`. `Channel.splitCsv()` moves to the MCP tool boundary. `FindOrCreateResult` moves to api/. Services add `implements` declarations. `findOrCreateWithBinding` is renamed to `findOrCreate` and generalised to dual-mode (binding-based + name-based) lookup.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny (reactive), JPA/Panache

**Spec:** `specs/issue-315-message-dispatcher-channel-lifecycle-spi/2026-07-02-message-dispatcher-channel-manager-design.md`

## Global Constraints

- Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` (not `./mvnw`)
- No Flyway migrations
- No new dependencies (mutiny already `provided` in api/)
- `api/spi/` is for consumer-provided extension points. These interfaces are consumer-called service facades — they go in domain packages (`api/channel/`, `api/message/`)
- Reactive gating: `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")` on reactive impls
- Worker-pool pattern for blocking SPIs in reactive services: `Uni.createFrom().item(() -> blocking.call()).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`
- TDD: superpowers:test-driven-development before implementing
- java-dev skill for all Java implementation
- All commits reference #315

---

### Task 1: Create service facade interfaces + move FindOrCreateResult

Pure additions — nothing breaks. Establishes the contracts that subsequent tasks implement.

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/ReactiveMessageDispatcher.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/ReactiveChannelManager.java`
- Move: `runtime/.../channel/FindOrCreateResult.java` → `api/.../channel/FindOrCreateResult.java`

**Interfaces:**
- Produces: `MessageDispatcher.dispatch(MessageDispatch) → DispatchResult`
- Produces: `ReactiveMessageDispatcher.dispatch(MessageDispatch) → Uni<DispatchResult>`
- Produces: `ChannelManager` with 9 methods (see spec § ChannelManager)
- Produces: `ReactiveChannelManager` — reactive mirror with `Uni<>` returns
- Produces: `FindOrCreateResult(Channel, boolean)` in `api/channel/`

- [ ] **Step 1: Create `MessageDispatcher.java`**

```java
// api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java
package io.casehub.qhorus.api.message;

public interface MessageDispatcher {
    DispatchResult dispatch(MessageDispatch dispatch);
}
```

- [ ] **Step 2: Create `ReactiveMessageDispatcher.java`**

```java
// api/src/main/java/io/casehub/qhorus/api/message/ReactiveMessageDispatcher.java
package io.casehub.qhorus.api.message;

import io.smallrye.mutiny.Uni;

public interface ReactiveMessageDispatcher {
    Uni<DispatchResult> dispatch(MessageDispatch dispatch);
}
```

- [ ] **Step 3: Create `ChannelManager.java`**

```java
// api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java
package io.casehub.qhorus.api.channel;

import java.util.List;
import java.util.Set;
import java.util.UUID;

import io.casehub.qhorus.api.message.MessageType;

public interface ChannelManager {
    Channel create(ChannelCreateRequest request);
    FindOrCreateResult findOrCreate(ChannelCreateRequest request);
    long delete(UUID channelId, boolean force);
    Channel pause(UUID channelId);
    Channel resume(UUID channelId);

    Channel setTypeConstraints(UUID channelId, Set<MessageType> allowedTypes, Set<MessageType> deniedTypes);
    Channel setRateLimits(UUID channelId, Integer perChannel, Integer perInstance);
    Channel setAllowedWriters(UUID channelId, List<String> allowedWriters);
    Channel setAdminInstances(UUID channelId, List<String> adminInstances);
}
```

- [ ] **Step 4: Create `ReactiveChannelManager.java`**

```java
// api/src/main/java/io/casehub/qhorus/api/channel/ReactiveChannelManager.java
package io.casehub.qhorus.api.channel;

import java.util.List;
import java.util.Set;
import java.util.UUID;

import io.smallrye.mutiny.Uni;
import io.casehub.qhorus.api.message.MessageType;

public interface ReactiveChannelManager {
    Uni<Channel> create(ChannelCreateRequest request);
    Uni<FindOrCreateResult> findOrCreate(ChannelCreateRequest request);
    Uni<Long> delete(UUID channelId, boolean force);
    Uni<Channel> pause(UUID channelId);
    Uni<Channel> resume(UUID channelId);

    Uni<Channel> setTypeConstraints(UUID channelId, Set<MessageType> allowedTypes, Set<MessageType> deniedTypes);
    Uni<Channel> setRateLimits(UUID channelId, Integer perChannel, Integer perInstance);
    Uni<Channel> setAllowedWriters(UUID channelId, List<String> allowedWriters);
    Uni<Channel> setAdminInstances(UUID channelId, List<String> adminInstances);
}
```

- [ ] **Step 5: Move `FindOrCreateResult` to api/channel/**

Delete `runtime/src/main/java/io/casehub/qhorus/runtime/channel/FindOrCreateResult.java`.
Create `api/src/main/java/io/casehub/qhorus/api/channel/FindOrCreateResult.java`:

```java
package io.casehub.qhorus.api.channel;

public record FindOrCreateResult(Channel channel, boolean wasCreated) {}
```

Use IntelliJ `ide_move_file` for the move to update all imports automatically.

- [ ] **Step 6: Build api/ module to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 7: Full build to verify FindOrCreateResult import update**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS (all modules compile with updated import)

- [ ] **Step 8: Commit**

```
feat(#315): add MessageDispatcher, ChannelManager interfaces and move FindOrCreateResult to api

Refs #315
```

---

### Task 2: ChannelCreateRequest CSV→List\<String\> + Channel.fromRequest + splitCsv move

Changes `ChannelCreateRequest.barrierContributors`, `.allowedWriters`, `.adminInstances` from `String` to `List<String>`. Updates `Channel.fromRequest()` to pass-through. Moves `Channel.splitCsv()` to `QhorusMcpToolsBase`. Cascades to ALL callers.

This is atomic — every caller must update in the same commit or compilation fails.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java` — 3 fields + Builder methods: `String` → `List<String>`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — `fromRequest()` pass-through, remove `splitCsv()`
- Modify: `runtime/.../mcp/QhorusMcpToolsBase.java` — add `protected static splitCsv()` helper
- Modify: `runtime/.../mcp/QhorusMcpTools.java` — `create_channel` uses `splitCsv()` at boundary
- Modify: `runtime/.../mcp/ReactiveQhorusMcpTools.java` — same
- Modify: `runtime/.../channel/ChannelService.java` — `setAllowedWriters`/`setAdminInstances` accept `List<String>`
- Modify: `runtime/.../channel/ReactiveChannelService.java` — same
- Modify: ALL callers of `ChannelCreateRequest.builder().allowedWriters(String)` → `List<String>`
- Modify: ALL callers of `Channel.splitCsv()` → use `QhorusMcpToolsBase.splitCsv()` or pass `List<String>` directly

**Interfaces:**
- Consumes: `ChannelCreateRequest` (current String fields)
- Produces: `ChannelCreateRequest` with `List<String>` fields, `QhorusMcpToolsBase.splitCsv()` helper

**Approach:** Use IntelliJ `ide_find_references` to find every caller of the three Builder methods and every reference to `Channel.splitCsv()`. Update all in one sweep. The implementing agent must be exhaustive — a missed call site means a compile failure.

- [ ] **Step 1: Change `ChannelCreateRequest` record fields from `String` to `List<String>`**

In `ChannelCreateRequest.java`:
- Change fields: `String barrierContributors` → `List<String> barrierContributors`, `String allowedWriters` → `List<String> allowedWriters`, `String adminInstances` → `List<String> adminInstances`
- Add `import java.util.List;`
- Add defensive copy in compact constructor: `barrierContributors = barrierContributors != null ? List.copyOf(barrierContributors) : null;` (same pattern as `allowedTypes`)
- Update Builder field types and setter signatures to `List<String>`

- [ ] **Step 2: Update `Channel.fromRequest()` to pass-through**

In `Channel.java`, change `fromRequest()` to pass `req.barrierContributors()`, `req.allowedWriters()`, `req.adminInstances()` directly instead of wrapping in `splitCsv()`. Remove the `splitCsv()` method entirely.

- [ ] **Step 3: Add `splitCsv()` to `QhorusMcpToolsBase`**

```java
protected static List<String> splitCsv(String csv) {
    if (csv == null || csv.isBlank()) return null;
    List<String> result = java.util.Arrays.stream(csv.split(","))
            .map(String::trim)
            .filter(s -> !s.isEmpty())
            .toList();
    return result.isEmpty() ? null : result;
}
```

- [ ] **Step 4: Update `ChannelService.setAllowedWriters` and `setAdminInstances`**

Change signatures from `(UUID, String)` to `(UUID, List<String>)`. Remove `Channel.splitCsv()` calls — pass the list directly to `toBuilder().allowedWriters(list)`.

- [ ] **Step 5: Update `ReactiveChannelService.setAllowedWriters` and `setAdminInstances`**

Same changes as Step 4 for the reactive path.

- [ ] **Step 6: Update ALL callers**

Use IntelliJ `ide_find_references` on `ChannelCreateRequest.Builder.allowedWriters`, `.adminInstances`, `.barrierContributors` to find every call site. Update each from `String` to `List<String>` (or `List.of("value")` for single values).

Key callers to find and update:
- `QhorusMcpTools.createChannel()` — use `splitCsv()` at boundary
- `ReactiveQhorusMcpTools.createChannel()` — same
- `QhorusMcpTools.setChannelWriters()` / `setChannelAdmins()` — use `splitCsv()` at boundary
- `ReactiveQhorusMcpTools` equivalents
- `ConnectorChannelBackend.tryAutoCreate()` — passes `null` for these fields, already correct
- Test fixtures: `MessageServiceTest`, `ChannelServiceTest`, etc.

- [ ] **Step 7: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 8: Commit**

```
refactor(#315): ChannelCreateRequest CSV strings → List<String>, move splitCsv to MCP boundary

Refs #315
```

---

### Task 3: MessageService/ReactiveMessageService implement dispatchers

Simple — add `implements` declarations. No signature changes needed.

**Files:**
- Modify: `runtime/.../message/MessageService.java` — add `implements MessageDispatcher`
- Modify: `runtime/.../message/ReactiveMessageService.java` — add `implements ReactiveMessageDispatcher`
- Test: `runtime/src/test/java/...` — verify `instanceof` for both

**Interfaces:**
- Consumes: `MessageDispatcher`, `ReactiveMessageDispatcher` (from Task 1)

- [ ] **Step 1: Write failing test**

```java
// In an existing test class or a new lightweight test
@Test
void messageServiceImplementsDispatcher() {
    assertThat(messageService).isInstanceOf(MessageDispatcher.class);
}
```

- [ ] **Step 2: Add `implements MessageDispatcher` to `MessageService`**

```java
public class MessageService implements MessageDispatcher {
```

Add import: `import io.casehub.qhorus.api.message.MessageDispatcher;`

- [ ] **Step 3: Add `implements ReactiveMessageDispatcher` to `ReactiveMessageService`**

```java
public class ReactiveMessageService implements ReactiveMessageDispatcher {
```

Add import: `import io.casehub.qhorus.api.message.ReactiveMessageDispatcher;`

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests pass

- [ ] **Step 5: Commit**

```
feat(#315): MessageService implements MessageDispatcher

Refs #315
```

---

### Task 4: ChannelService implements ChannelManager + findOrCreate generalisation

Rename `findOrCreateWithBinding` → `findOrCreate`. Add name-based lookup path. Add `implements ChannelManager`.

**Files:**
- Modify: `runtime/.../channel/ChannelService.java`
- Modify: `connector-backend/.../ConnectorChannelBackend.java` — rename call site
- Modify: `runtime/src/test/.../ChannelServiceFindOrCreateTest.java` — rename + new test for name-based path
- Test: new test for name-based `findOrCreate`

**Interfaces:**
- Consumes: `ChannelManager` (from Task 1), `ChannelCreateRequest` with `List<String>` (from Task 2)
- Produces: `ChannelService implements ChannelManager`

- [ ] **Step 1: Write failing test for name-based findOrCreate**

In `ChannelServiceFindOrCreateTest.java`, add a test for name-based lookup:

```java
@Test
void findOrCreate_nameBasedLookup_createsNew() {
    ChannelCreateRequest req = ChannelCreateRequest.builder("test-name-based").build();
    FindOrCreateResult result = channelService.findOrCreate(req);
    assertThat(result.wasCreated()).isTrue();
    assertThat(result.channel().name()).isEqualTo("test-name-based");
}

@Test
void findOrCreate_nameBasedLookup_findsExisting() {
    ChannelCreateRequest req = ChannelCreateRequest.builder("test-find-existing").build();
    channelService.create(req);
    FindOrCreateResult result = channelService.findOrCreate(req);
    assertThat(result.wasCreated()).isFalse();
    assertThat(result.channel().name()).isEqualTo("test-find-existing");
}
```

- [ ] **Step 2: Rename `findOrCreateWithBinding` → `findOrCreate` and add name-based path**

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public FindOrCreateResult findOrCreate(final ChannelCreateRequest req) {
    if (req.hasConnectorBinding()) {
        // Binding-based lookup — existing path
        Optional<ChannelConnectorBinding> existingBinding = channelBindingStore
                .findByKey(req.inboundConnectorId(), req.externalKey());
        if (existingBinding.isPresent()) {
            Channel existing = channelStore.find(existingBinding.get().channelId())
                    .orElseThrow(() -> new IllegalStateException(
                            "Stale binding: binding exists for key '" + req.externalKey()
                            + "' but referenced channel was deleted"));
            return new FindOrCreateResult(existing, false);
        }
        Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());
        channel = channel.toBuilder().autoCreated(true).build();
        channel = channelStore.put(channel);
        channelBindingStore.put(new ChannelConnectorBinding(
                channel.id(), req.inboundConnectorId(), req.externalKey(),
                req.outboundConnectorId(), req.outboundDestination()));
        return new FindOrCreateResult(channel, true);
    }

    // Name-based lookup
    Optional<Channel> existing = channelStore.findByName(req.name());
    if (existing.isPresent()) {
        return new FindOrCreateResult(existing.get(), false);
    }
    Channel channel = create(req);
    return new FindOrCreateResult(channel, true);
}
```

- [ ] **Step 3: Add `implements ChannelManager` to class declaration**

```java
public class ChannelService implements ChannelManager {
```

Verify all 9 ChannelManager methods have matching signatures. `setAllowedWriters` and `setAdminInstances` were already changed to `List<String>` in Task 2.

- [ ] **Step 4: Update `ConnectorChannelBackend.tryAutoCreate()`**

Rename `channelService.findOrCreateWithBinding(req)` → `channelService.findOrCreate(req)`.

- [ ] **Step 5: Update all test call sites**

In `ChannelServiceFindOrCreateTest.java`: rename all `findOrCreateWithBinding` → `findOrCreate`.

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,connector-backend`
Expected: All tests pass

- [ ] **Step 7: Commit**

```
feat(#315): ChannelService implements ChannelManager, generalise findOrCreate

Refs #315
```

---

### Task 5: ReactiveChannelService implements ReactiveChannelManager + reactive findOrCreate

Add `findOrCreate` to `ReactiveChannelService` (new method — does not exist). Use worker-pool pattern for binding-based path.

**Files:**
- Modify: `runtime/.../channel/ReactiveChannelService.java` — add `implements ReactiveChannelManager`, add `findOrCreate`
- Test: new test for reactive `findOrCreate` (name-based + binding-based)

**Interfaces:**
- Consumes: `ReactiveChannelManager` (from Task 1), `ChannelBindingStore` (blocking, via worker pool)

- [ ] **Step 1: Write failing test for reactive name-based findOrCreate**

Tests are `@Disabled` (require PostgreSQL DevServices). Write the test correctly for when the reactive stack is enabled:

```java
@Test
void findOrCreate_nameBasedLookup_createsNew() {
    ChannelCreateRequest req = ChannelCreateRequest.builder("reactive-name-based").build();
    FindOrCreateResult result = reactiveChannelService.findOrCreate(req)
            .await().indefinitely();
    assertThat(result.wasCreated()).isTrue();
    assertThat(result.channel().name()).isEqualTo("reactive-name-based");
}
```

- [ ] **Step 2: Add `implements ReactiveChannelManager` and inject `ChannelBindingStore`**

```java
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class ReactiveChannelService implements ReactiveChannelManager {

    @Inject ChannelBindingStore channelBindingStore;  // blocking — used via worker pool
```

- [ ] **Step 3: Implement `findOrCreate`**

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public Uni<FindOrCreateResult> findOrCreate(final ChannelCreateRequest req) {
    if (req.hasConnectorBinding()) {
        return Uni.createFrom().item(() ->
                channelBindingStore.findByKey(req.inboundConnectorId(), req.externalKey()))
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
            .flatMap(existingBinding -> {
                if (existingBinding.isPresent()) {
                    return channelStore.find(existingBinding.get().channelId())
                            .map(opt -> opt.orElseThrow(() -> new IllegalStateException(
                                    "Stale binding for key '" + req.externalKey() + "'")))
                            .map(ch -> new FindOrCreateResult(ch, false));
                }
                return create(req).flatMap(channel -> {
                    Channel autoCreated = channel.toBuilder().autoCreated(true).build();
                    return channelStore.put(autoCreated).map(saved -> {
                        Uni.createFrom().item(() -> {
                            channelBindingStore.put(new ChannelConnectorBinding(
                                    saved.id(), req.inboundConnectorId(), req.externalKey(),
                                    req.outboundConnectorId(), req.outboundDestination()));
                            return null;
                        }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
                          .await().indefinitely();
                        return new FindOrCreateResult(saved, true);
                    });
                });
            });
    }

    // Name-based lookup
    return Panache.withTransaction("qhorus", () ->
        channelStore.findByName(req.name())
            .flatMap(existing -> {
                if (existing.isPresent()) {
                    return Uni.createFrom().item(new FindOrCreateResult(existing.get(), false));
                }
                return create(req).map(ch -> new FindOrCreateResult(ch, true));
            }));
}
```

Note: The exact reactive implementation should be verified against the existing `create()` method flow. The implementing agent must read the full `ReactiveChannelService.create()` to align the transaction and gateway patterns.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All passing tests pass (reactive tests remain @Disabled)

- [ ] **Step 5: Commit**

```
feat(#315): ReactiveChannelService implements ReactiveChannelManager with findOrCreate

Refs #315
```

---

### Task 6: Dead code removal + ChannelEntity.fromRequest

Remove `ChannelEntity.fromRequest(ChannelCreateRequest, String)` — zero production callers, only test callers. Update tests to use the canonical `Channel.fromRequest()` → `ChannelEntity.fromDomain()` path.

**Files:**
- Modify: `runtime/.../channel/ChannelEntity.java` — remove `fromRequest()` method
- Modify: `runtime/src/test/.../ChannelFromRequestTest.java` — update tests to use canonical path

- [ ] **Step 1: Verify zero production callers with IntelliJ**

Use `ide_find_references` on `ChannelEntity.fromRequest`. Confirm only test callers.

- [ ] **Step 2: Update `ChannelFromRequestTest` to use canonical path**

Replace `ChannelEntity.fromRequest(req, tenancyId)` with `ChannelEntity.fromDomain(Channel.fromRequest(req, tenancyId))` in each test.

- [ ] **Step 3: Remove `ChannelEntity.fromRequest()`**

Delete the method from `ChannelEntity.java`.

- [ ] **Step 4: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 5: Commit**

```
refactor(#315): remove ChannelEntity.fromRequest() dead code

Refs #315
```

---

### Task 7: CLAUDE.md + documentation update

Update CLAUDE.md project structure section to document the new interfaces and the taxonomy of api/ categories. Update any stale references to `findOrCreateWithBinding`.

**Files:**
- Modify: `CLAUDE.md` — project structure, testing conventions

- [ ] **Step 1: Update CLAUDE.md**

- Add `MessageDispatcher.java` and `ReactiveMessageDispatcher.java` to `api/message/` listing
- Add `ChannelManager.java`, `ReactiveChannelManager.java`, `FindOrCreateResult.java` to `api/channel/` listing
- Add taxonomy table (store / spi / gateway / service facade) to the api/ section
- Update `ChannelService.java` description: `findOrCreateWithBinding` → `findOrCreate`, `setAllowedWriters(UUID, List<String>)`, `implements ChannelManager`
- Update `FindOrCreateResult.java` location from `runtime/channel/` to `api/channel/`
- Update `ChannelCreateRequest` description: `barrierContributors`, `allowedWriters`, `adminInstances` are now `List<String>`
- Remove reference to `ChannelEntity.fromRequest()` if present

- [ ] **Step 2: Commit**

```
docs(#315): update CLAUDE.md for MessageDispatcher and ChannelManager SPIs

Refs #315
```

---

### Task 8: Final verification + platform coherence review

Full build, cross-module breakage check, protocol compliance review.

- [ ] **Step 1: Full clean build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 2: Build examples (profile-gated — catches stale call sites)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -Pwith-llm-examples -f examples/agent-communication/pom.xml`
Expected: BUILD SUCCESS (or compile errors → fix stale call sites)

- [ ] **Step 3: Protocol coherence review**

Verify against:
- `consumer-spi-placement.md` — interfaces correctly placed in domain packages (not `api/spi/`), confirmed not consumer-implemented
- `module-tier-structure.md` — api/ remains Tier 1 (no JPA, no Quarkus runtime). Mutiny `provided` is explicitly permitted.
- `reactive-blocking-spi-worker-pool.md` — reactive `findOrCreate` uses `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` for `ChannelBindingStore` calls
- `qhorus-reactive-gating.md` — `ReactiveChannelManager` beans gated by `@IfBuildProperty`

- [ ] **Step 4: Verify no stale splitCsv references remain in runtime**

Use IntelliJ `ide_search_text` for `Channel.splitCsv` in `runtime/src/main/`. Expected: zero matches (all moved to `QhorusMcpToolsBase`).

- [ ] **Step 5: Code review**

Invoke `superpowers:requesting-code-review`. Any Minor+ finding not fixed this session → create GitHub issue.

- [ ] **Step 6: Documentation sync**

Invoke `implementation-doc-sync`.
