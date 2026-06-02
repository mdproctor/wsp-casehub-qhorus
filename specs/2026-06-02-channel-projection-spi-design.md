# ChannelProjection SPI Design

**Issue:** casehubio/qhorus#230 + casehubio/qhorus#231
**Date:** 2026-06-02  
**Status:** Approved (rev 2 — post code review)

---

## Problem

Consumers (DraftHouse, Claudony) need to derive deterministic read-models from a
channel's message history — a debate vote tally, a review manifest, a work digest.
Each consumer currently implements ad-hoc channel-reading logic locally. The pattern
is universal: left-fold over typed messages → materialised state → output. Qhorus
should own the infrastructure; consumers own the domain logic.

---

## Design

### Core pattern

A `ChannelProjection<S>` is a pure left-fold: an identity element and a step
function. Qhorus reads the message history, folds it via the projection, and returns
a `ProjectionResult<S>` containing the materialised state and the last message ID
seen (needed as a cursor for incremental re-projection). What the consumer does with
the state — render to markdown, serve via REST, compare counts — is not Qhorus's
concern.

---

### New types in `api/`

**`api/message/MessageView.java`** — read-side DTO; the fold function's input.

```java
public record MessageView(
    Long id,
    UUID channelId,
    String sender,
    MessageType type,           // field is type, not messageType — see Mapper section
    String content,
    String correlationId,
    Long inReplyTo,
    String target,
    String artefactRefs,
    ActorType actorType,
    Instant createdAt,
    Instant deadline,
    int replyCount
) {}
```

Fields included: everything a reasonably sophisticated projection needs.  
Fields excluded: `commitmentId` (internal infrastructure UUID),
`acknowledgedAt` (always null in v1).

Note: the entity field is `messageType`; the DTO field is `type` — consistent with
`DispatchResult.type`. The mapper method must document this rename explicitly to
avoid confusion.

`MessageView` is not projection-specific — it is the canonical read-side
representation of a message. Future uses (timeline REST endpoints) may consume it
directly.

---

**`api/spi/ChannelProjection.java`**

```java
/**
 * A pure left-fold over a channel's message history.
 *
 * <p><strong>Contract:</strong>
 * <ul>
 *   <li>{@code identity()} must return a <em>fresh</em> instance on every call.
 *       If {@code S} is mutable (e.g., a HashMap accumulator), returning a cached
 *       singleton creates shared state across concurrent {@code project()} calls.</li>
 *   <li>{@code apply()} must be pure: no external state reads or writes, no side
 *       effects, no thread-local access. Return {@code state} unchanged for messages
 *       this projection does not handle.</li>
 *   <li>{@code apply()} must not throw. If it does (unchecked), the exception
 *       propagates from {@code project()} — partial state is not returned.</li>
 * </ul>
 */
public interface ChannelProjection<S> {
    /** The neutral element — empty initial state before any messages are folded. */
    S identity();

    /** Pure fold step. Return {@code state} unchanged for ignored messages. */
    S apply(S state, MessageView message);
}
```

Two abstract methods — not `@FunctionalInterface`. No `channelType()` in v1; a
future registry will use a `@ChannelBound` qualifier annotation or a
`ChannelBoundProjection` subinterface so the return type is not locked to `String`.

---

**`api/spi/ProjectionResult.java`**

```java
/**
 * The result of a projection fold: materialised state plus the ID of the last
 * message folded. {@code lastMessageId} is null if the channel was empty.
 *
 * <p>Pass this as {@code previous} to the incremental {@code project()} overload
 * to resume folding from the cursor without re-reading earlier messages.
 *
 * <p><strong>Contract:</strong> only pass results obtained from
 * {@code ProjectionService.project()} — never construct manually. A manually
 * constructed instance with a non-null {@code state} and a null {@code lastMessageId}
 * has undefined behaviour in the incremental overload: the service treats
 * {@code lastMessageId == null} as "channel was empty, start from identity()"
 * and will silently ignore the provided {@code state}.
 */
public record ProjectionResult<S>(S state, Long lastMessageId) {

    /** True when the channel was empty — no messages folded. */
    public boolean isEmpty() {
        return lastMessageId == null;
    }
}
```

`ProjectionResult` belongs in `api/spi/` — it is the result type of the SPI,
not a general message type.

---

**No `ProjectionRenderer<S>` in `api/spi/`.** The pattern of "turns state into a
string" is documented in the Javadoc on `ChannelProjection<S>` and in consumer
guides. `Function<S, String>` from the JDK is already named, typed, and composable.
Putting a `@FunctionalInterface` in `api/spi/` that Qhorus never calls gives the
false impression of a managed extension point. Consumers use `Function<S, String>`
or a local `@FunctionalInterface` if they want a named abstraction.

---

### Runtime service — `runtime/message/ProjectionService.java`

Four overloads: full, scoped-full, incremental, scoped-incremental. All return
`ProjectionResult<S>`.

```java
@ApplicationScoped
public class ProjectionService {

    @Inject MessageStore messageStore;
    @Inject QhorusEntityMapper mapper;

    /** Project all messages in the channel from the beginning. */
    public <S> ProjectionResult<S> project(UUID channelId,
                                            ChannelProjection<S> projection) {
        Objects.requireNonNull(channelId, "channelId");
        Objects.requireNonNull(projection, "projection");
        return fold(MessageQuery.builder().channelId(channelId).build(),
                    projection.identity(), null, projection);
    }

    /**
     * Project messages in the channel matching {@code scope}.
     *
     * <p>Scope rules:
     * <ul>
     *   <li>If {@code scope.channelId()} is non-null and differs from
     *       {@code channelId}, {@link IllegalArgumentException} is thrown.</li>
     *   <li>{@code scope.descending(true)} throws — projections always fold
     *       ascending (insertion order). Strip the flag before passing scope.</li>
     * </ul>
     */
    public <S> ProjectionResult<S> project(UUID channelId, MessageQuery scope,
                                            ChannelProjection<S> projection) {
        Objects.requireNonNull(channelId, "channelId");
        Objects.requireNonNull(scope, "scope");
        Objects.requireNonNull(projection, "projection");
        validateScope(channelId, scope);
        var query = scope.toBuilder().channelId(channelId).build();
        return fold(query, projection.identity(), null, projection);
    }

    /**
     * Resume a fold from a previous result. Only messages with
     * {@code id > previous.lastMessageId()} are folded.
     *
     * <p>If {@code previous.isEmpty()} (channel was empty at last projection),
     * a full scan is performed starting from {@code identity()} — {@code previous.state()}
     * is ignored. {@code lastMessageId == null} unambiguously means "fold from scratch":
     * the service enforces this regardless of what is in {@code state}, so that a
     * manually constructed {@code ProjectionResult} with inconsistent fields cannot
     * silently corrupt the fold.
     */
    public <S> ProjectionResult<S> project(UUID channelId,
                                            ProjectionResult<S> previous,
                                            ChannelProjection<S> projection) {
        Objects.requireNonNull(channelId, "channelId");
        Objects.requireNonNull(previous, "previous");
        Objects.requireNonNull(projection, "projection");
        // When isEmpty(), start from identity() regardless of previous.state().
        var initialState = previous.isEmpty() ? projection.identity() : previous.state();
        var query = MessageQuery.builder()
                .channelId(channelId)
                .afterId(previous.lastMessageId())   // null = full scan (empty channel)
                .build();
        return fold(query, initialState, previous.lastMessageId(), projection);
    }

    /** Scoped incremental — scope rules same as the scoped-full overload. */
    public <S> ProjectionResult<S> project(UUID channelId,
                                            ProjectionResult<S> previous,
                                            MessageQuery scope,
                                            ChannelProjection<S> projection) {
        Objects.requireNonNull(channelId, "channelId");
        Objects.requireNonNull(previous, "previous");
        Objects.requireNonNull(scope, "scope");
        Objects.requireNonNull(projection, "projection");
        validateScope(channelId, scope);
        var initialState = previous.isEmpty() ? projection.identity() : previous.state();
        var query = scope.toBuilder()
                .channelId(channelId)
                .afterId(previous.lastMessageId())
                .build();
        return fold(query, initialState, previous.lastMessageId(), projection);
    }

    private <S> ProjectionResult<S> fold(MessageQuery query, S initialState,
                                          Long cursorIn, ChannelProjection<S> projection) {
        var messages = messageStore.scan(query);
        var state = initialState;
        var lastId = cursorIn;
        for (var msg : messages) {
            state = projection.apply(state, mapper.toMessageView(msg));
            lastId = msg.id;
        }
        return new ProjectionResult<>(state, lastId);
    }

    private void validateScope(UUID channelId, MessageQuery scope) {
        if (scope.channelId() != null && !scope.channelId().equals(channelId)) {
            throw new IllegalArgumentException(
                "scope.channelId() conflicts with channelId parameter — remove it from scope");
        }
        if (scope.descending()) {
            throw new IllegalArgumentException(
                "scope.descending(true) breaks fold order — projections always fold ascending");
        }
    }
}
```

---

### Reactive store extension — `ReactiveMessageStore`

`Uni<List<Message>>` from `scan()` materialises everything in memory — that is not
meaningfully reactive. The reactive service requires a new streaming method:

```java
// New method on ReactiveMessageStore
Multi<Message> stream(MessageQuery query);
```

**Implementation semantics by store type:**

- `ReactiveJpaMessageStore.stream()` — backed by `PanacheQuery.stream()`, which uses a
  Hibernate Reactive scrollable cursor. Messages are fetched from the database
  row-by-row (or in small fetch-size batches), not materialised as a full
  `List<Message>` before the fold starts. This is genuinely cursor-backed streaming
  with bounded memory.

- `InMemoryReactiveMessageStore.stream()` — wraps the in-memory list as
  `Multi.createFrom().iterable(list)`. Not lazy, but functionally correct for tests.
  Consumers of the testing module are not using it for large-channel performance.

- `StubReactiveMessageStore.stream()` — throws `UnsupportedOperationException`
  (same contract as other stub methods).

Must be added to `ReactiveMessageStore` interface and all three implementations.

---

### Reactive service — `runtime/message/ReactiveProjectionService.java`

Build-gated on `casehub.qhorus.reactive.enabled`. Uses `stream()` for a genuine
streaming fold — one message fetched and processed at a time, bounded memory.

```java
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class ReactiveProjectionService {

    @Inject ReactiveMessageStore reactiveMessageStore;
    @Inject QhorusEntityMapper mapper;

    public <S> Uni<ProjectionResult<S>> project(UUID channelId,
                                                  ChannelProjection<S> projection) { ... }

    public <S> Uni<ProjectionResult<S>> project(UUID channelId, MessageQuery scope,
                                                  ChannelProjection<S> projection) { ... }

    public <S> Uni<ProjectionResult<S>> project(UUID channelId,
                                                  ProjectionResult<S> previous,
                                                  ChannelProjection<S> projection) { ... }

    public <S> Uni<ProjectionResult<S>> project(UUID channelId,
                                                  ProjectionResult<S> previous,
                                                  MessageQuery scope,
                                                  ChannelProjection<S> projection) { ... }
}
```

The incremental overloads apply the same `isEmpty()` guard as the blocking service:
`previous.isEmpty() ? projection.identity() : previous.state()`.

**Internal fold operator:** `Multi.collect().in()` with a private mutable
accumulator class. `collect().in()` takes `Supplier<C>` (container factory) and
`BiConsumer<C, T>` (mutation), so the container must be mutable — a `FoldAcc<S>`
class with mutable fields, not a record.

```java
// Private mutable accumulator — never exposed outside ReactiveProjectionService
private static final class FoldAcc<S> {
    S state;
    Long lastId;
    FoldAcc(S state, Long lastId) { this.state = state; this.lastId = lastId; }
}

private <S> Uni<ProjectionResult<S>> reactiveFold(MessageQuery query,
                                                    S initialState,
                                                    Long cursorIn,
                                                    ChannelProjection<S> projection) {
    return reactiveMessageStore.stream(query)
        .collect().in(
            () -> new FoldAcc<>(initialState, cursorIn),
            (acc, msg) -> {
                acc.state = projection.apply(acc.state, mapper.toMessageView(msg));
                acc.lastId = msg.id;
            })
        .map(acc -> new ProjectionResult<>(acc.state, acc.lastId));
}
```

`collect().in()` is the correct Mutiny operator for a stateful fold over a `Multi`
with a mutable accumulator. It is NOT `Multi.scan()` (which emits intermediate
running states, not a single final result) and NOT `collect().in(projection::identity,
projection::apply)` (which would not compile — `apply` returns `S`, but `collect().in()`
requires a `BiConsumer<C, T>` with a void mutation, not a `BiFunction`).

The same scope validation applies (duplicated or shared static utility).

---

### Mapper extension — `QhorusEntityMapper.toMessageView(Message)`

New method on the existing `@ApplicationScoped` mapper. Consistent with
`toChannelDetail(Channel, long, Optional<ChannelConnectorBinding>)` (three params —
the two-param variant no longer exists) and `toTimelineEntry(Message)` already there.

```java
// toMessageView: Message.messageType → MessageView.type
// The field rename is intentional — matches DispatchResult.type convention.
// Do not write msg.messageType() in callers; always go through the mapper.
public MessageView toMessageView(Message msg) { ... }
```

The rename `messageType → type` must be called out in the method comment to prevent
callers from accidentally accessing `msg.messageType` and confusing consumers reading
the mapped result.

---

### Scope overload semantics

Consumers use the scoped overload to exclude noise from the fold:

```java
var scope = MessageQuery.builder()
    .excludeTypes(List.of(MessageType.EVENT))
    .build();
var result = projectionService.project(channelId, scope, new VoteTallyProjection());
```

**What scope must NOT contain:**
- `channelId` — enforced by the service (throws if different from the parameter)
- `descending(true)` — throws; fold is always ascending by insertion order

---

### Behavioural invariants to document in Javadoc

**LAST_WRITE channels:** On a LAST_WRITE channel, `messageStore.scan()` returns at
most one message per sender (the current value — history is overwritten in place).
A projection over a LAST_WRITE channel folds the current snapshot only, not history.
This is useful for "who has checked in?" projections; it silently produces wrong
results for "vote tally over time" projections. Consumers must select channel
semantics that match their projection's assumptions.

**Unknown channelId:** A `project()` call on a non-existent `channelId` returns
`new ProjectionResult<>(projection.identity(), null)` — identical to an empty
channel. If the consumer needs to distinguish "channel does not exist" from "channel
has no messages", they must call `ChannelStore.find(channelId)` separately. No
special exception is thrown.

**Threading:** `identity()` must return a fresh instance on every invocation.
`apply()` must be pure — no external state reads or writes. Both constraints are
documented in the `ChannelProjection<S>` Javadoc (see above).

---

### Incremental projection (#231)

The cursor-based overloads (`project(channelId, previous, projection)`) allow a
consumer to resume from a cached result without re-reading earlier messages.

**Usage:**

```java
// First projection — full scan
var result = projectionService.project(channelId, new VoteTallyProjection());
var state = result.state();
var cursor = result.lastMessageId(); // null if channel was empty

// ... later, after new messages arrive ...
var updated = projectionService.project(channelId, result, new VoteTallyProjection());
```

**replyCount staleness warning:** `Message.replyCount` is updated in-place when
replies arrive. An incremental projection that skips earlier messages (via `afterId`)
does not see updated `replyCount` values for those messages. Projections that derive
meaning from `replyCount` (thread depth, participation metrics) must always do a
full scan — not an incremental one — to get accurate counts.

**Empty-channel case:** If `previous.isEmpty()` (the channel had no messages at
last projection), the incremental overload performs a full scan from `identity()`.
This is safe — `MessageQuery.builder().afterId(null)` produces an unbounded query.

---

### Testing

**Unit test — fold logic only (no CDI, no store)**

The fold function is a pure function. Test it directly:

```java
ChannelProjection<VoteState> proj = new VoteTallyProjection();
var state = proj.identity();
state = proj.apply(state, new MessageView(1L, channelId, "alice",
    MessageType.COMMAND, "approve", null, null, null, null,
    ActorType.AGENT, Instant.now(), null, 0));
assertThat(state.approvalCount()).isEqualTo(1);
```

No framework, no store, no CDI. This is the primary test vector for projection logic.

**Integration test — `@QuarkusTest` against H2/JPA**

Runtime `@QuarkusTest` tests use `JpaMessageStore` against H2 (`MODE=PostgreSQL`)
via `application.properties`. The `InMemoryMessageStore` alternative lives in the
`casehub-qhorus-testing` module — that is a consumer dependency (DraftHouse,
Claudony) and is NOT on the runtime test classpath. Runtime integration tests write
messages via `messageStore.put()` or `MessageService.dispatch()` and read back via
`ProjectionService.project()` against the same H2 database.

```java
@QuarkusTest
class ProjectionServiceIT {
    @Inject ProjectionService projectionService;
    @Inject MessageStore messageStore;

    @Test
    @TestTransaction
    void projectsApprovalCount() {
        var channelId = createChannel();
        var m1 = dispatchAndGet(channelId, "alice", MessageType.COMMAND, "approve");
        var m2 = dispatchAndGet(channelId, "bob",   MessageType.COMMAND, "approve");
        dispatchAndGet(channelId,           "carol", MessageType.DECLINE, "not yet");

        var result = projectionService.project(channelId, new VoteTallyProjection());

        assertThat(result.state().approvalCount()).isEqualTo(2);
        assertThat(result.state().declineCount()).isEqualTo(1);
        assertThat(result.lastMessageId()).isNotNull();
    }

    @Test
    @TestTransaction
    void incrementalProjectionOnlyFoldsNewMessages() {
        var channelId = createChannel();
        dispatchAndGet(channelId, "alice", MessageType.COMMAND, "approve");
        dispatchAndGet(channelId, "bob",   MessageType.COMMAND, "approve");

        var result1 = projectionService.project(channelId, new VoteTallyProjection());
        assertThat(result1.state().approvalCount()).isEqualTo(2);

        dispatchAndGet(channelId, "carol", MessageType.DECLINE, "not yet");

        var result2 = projectionService.project(channelId, result1,
                                                  new VoteTallyProjection());
        assertThat(result2.state().approvalCount()).isEqualTo(2);
        assertThat(result2.state().declineCount()).isEqualTo(1);
    }
}
```

---

### Placement summary

| Type | Module | Package |
|------|--------|---------|
| `MessageView` | `api` | `io.casehub.qhorus.api.message` |
| `ChannelProjection<S>` | `api` | `io.casehub.qhorus.api.spi` |
| `ProjectionResult<S>` | `api` | `io.casehub.qhorus.api.spi` |
| `ProjectionService` | `runtime` | `io.casehub.qhorus.runtime.message` |
| `ReactiveProjectionService` | `runtime` | `io.casehub.qhorus.runtime.message` |
| `toMessageView()` | `runtime` | `QhorusEntityMapper` (existing) |
| `stream(MessageQuery)` | `runtime` | `ReactiveMessageStore` + impls |

---

### Platform coherence

**PLATFORM.md** — add to capability ownership table:
> Channel read-model projection (left-fold over message history) | `casehub-qhorus` | `ChannelProjection<S>` SPI + `ProjectionService` in runtime; incremental via `ProjectionResult<S>` cursor

**casehub-qhorus.md deep-dive** — add `MessageView`, `ChannelProjection<S>`,
`ProjectionResult<S>`, `ProjectionService` to Key Abstractions.

**Protocol PP-20260602-b748c9** (`event-log-left-fold-projection`) — to be
authored in `casehub/garden/docs/protocols/universal/` during this session.

---

### What is NOT here

- No `channelType()` on `ChannelProjection<S>` — removed; future registry uses
  a `@ChannelBound` qualifier annotation or `ChannelBoundProjection` subinterface
  so the return type is not locked to `String`.
- No `ProjectionRenderer<S>` in `api/spi/` — not a managed extension point;
  pattern documented in Javadoc; consumers use `Function<S, String>`.
- No MCP tool — requires named-projection registry + render (filed as #232).
- No new Flyway migration — pure computation over existing messages.
- CDI-injected projections (registry layer) deferred to #232 — caller-passed
  and CDI-discovered are complementary; adding CDI discovery later does not
  require changing `ChannelProjection<S>`.

---

### Out of scope — filed issues

- **MCP tool `project_channel`** — casehubio/qhorus#232 (after this branch wraps)
- **DraftHouse migration** — casehubio/drafthouse#31
