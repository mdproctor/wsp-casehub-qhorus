# ChannelProjection SPI Design

**Issue:** casehubio/qhorus#230  
**Date:** 2026-06-02  
**Status:** Approved

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
the materialised state `<S>`. What the consumer does with `<S>` — render to markdown,
serve via REST, compare counts — is not Qhorus's concern.

### New types in `api/`

**`api/message/MessageView.java`** — read-side DTO; the fold function's input.

```java
public record MessageView(
    Long id,
    UUID channelId,
    String sender,
    MessageType type,
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
`acknowledgedAt` (always null in v1 — add when v2 ACK mechanism ships).

`MessageView` is not projection-specific — it is the canonical read-side
representation of a message. Future uses (timeline REST endpoints, dashboard
read models) may consume it directly.

**`api/spi/ChannelProjection.java`** — the SPI.

```java
public interface ChannelProjection<S> {
    /** The neutral element — empty initial state before any messages are folded. */
    S identity();

    /**
     * Pure fold step. Must not throw or mutate external state.
     * Return {@code state} unchanged for messages this projection ignores.
     */
    S apply(S state, MessageView message);

    /**
     * Channel type tag. Currently informational — reserved for a future
     * registry that dispatches projections by channel name/pattern.
     * Return a channel name prefix (e.g. {@code "work."}) or {@code "*"} for all.
     */
    String channelType();
}
```

Not `@FunctionalInterface` — three abstract methods.

**`api/spi/ProjectionRenderer.java`** — consumer-side convention.

```java
@FunctionalInterface
public interface ProjectionRenderer<S> {
    String render(S state);
}
```

The service never calls this. It names the "turns state into a string" pattern so
DraftHouse and Claudony have a first-class abstraction to implement. Consumers call
`renderer.render(service.project(channelId, projection))` themselves.

### Runtime service

**`runtime/message/ProjectionService.java`**

```java
@ApplicationScoped
public class ProjectionService {

    @Inject MessageStore messageStore;
    @Inject QhorusEntityMapper mapper;

    /** Project all messages in the channel. */
    public <S> S project(UUID channelId, ChannelProjection<S> projection) {
        return project(channelId, MessageQuery.builder().build(), projection);
    }

    /**
     * Project messages matching {@code scope} within the channel.
     * The service always enforces {@code channelId} — scope adds orthogonal
     * filters (type exclusions, sender filter, afterId cursor, etc.).
     * The channelId in scope, if set, is overridden.
     */
    public <S> S project(UUID channelId, MessageQuery scope,
                          ChannelProjection<S> projection) {
        var query = scope.toBuilder().channelId(channelId).build();
        var state = projection.identity();
        for (var msg : messageStore.scan(query)) {
            state = projection.apply(state, mapper.toMessageView(msg));
        }
        return state;
    }
}
```

**`runtime/message/ReactiveProjectionService.java`** — reactive parity
(build-gated on `casehub.qhorus.reactive.enabled`).

```java
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class ReactiveProjectionService {

    @Inject ReactiveMessageStore reactiveMessageStore;
    @Inject QhorusEntityMapper mapper;

    public <S> Uni<S> project(UUID channelId, ChannelProjection<S> projection) {
        return project(channelId, MessageQuery.builder().build(), projection);
    }

    public <S> Uni<S> project(UUID channelId, MessageQuery scope,
                               ChannelProjection<S> projection) {
        var query = scope.toBuilder().channelId(channelId).build();
        return reactiveMessageStore.scan(query)
            .map(messages -> {
                var state = projection.identity();
                for (var msg : messages) {
                    state = projection.apply(state, mapper.toMessageView(msg));
                }
                return state;
            });
    }
}
```

### Mapper extension

`QhorusEntityMapper.toMessageView(Message)` — new method on the existing mapper.
Consistent with `toChannelDetail(Channel, long)` and `toTimelineEntry(Message)`
already there. No new mapper class.

### Scope overload semantics

The two-parameter overload (`channelId, scope, projection`) allows consumers to
exclude noise from the fold:

```java
// Exclude EVENT telemetry from a vote-tally projection
var scope = MessageQuery.builder()
    .excludeTypes(List.of(MessageType.EVENT))
    .build();
var tally = projectionService.project(channelId, scope, new VoteTallyProjection());
```

The service always enforces channelId — scope cannot project a different channel.
Scope's `channelId` field, if set, is silently overridden. This is consistent
with how `MessageQuery` is used elsewhere.

---

## Error handling

`apply()` must not throw. If it does (unchecked), the exception propagates from
`project()` — no catch-and-continue. Partial state is not returned. The contract
is: implement `apply()` defensively; return `state` unchanged for messages you
cannot handle.

No timeout or result-count guard in v1. Very large channels are a future concern
(tracked in #231 — incremental/cursor projection).

---

## Testing

### Unit test pattern (no CDI)

```java
ChannelProjection<VoteState> proj = new VoteTallyProjection();
var state = proj.identity();
state = proj.apply(state, new MessageView(1L, channelId, "alice",
    MessageType.COMMAND, "approve", null, null, null, null,
    ActorType.AGENT, Instant.now(), null, 0));
assertThat(state.approvalCount()).isEqualTo(1);
```

The fold logic is a pure function — no framework, no CDI, no store needed.

### Integration test pattern (`@QuarkusTest`)

```java
@QuarkusTest
class ProjectionServiceIT {
    @Inject ProjectionService projectionService;
    @Inject QhorusMcpTools tools;  // or messageService.dispatch() directly

    @Test
    @TestTransaction
    void projectsApprovalCount() {
        var channelId = createChannel();
        sendMessage(channelId, "alice", MessageType.COMMAND, "approve");
        sendMessage(channelId, "bob",   MessageType.COMMAND, "approve");
        sendMessage(channelId, "carol", MessageType.DECLINE, "not yet");

        var state = projectionService.project(channelId, new VoteTallyProjection());

        assertThat(state.approvalCount()).isEqualTo(2);
        assertThat(state.declineCount()).isEqualTo(1);
    }
}
```

Uses `InMemoryMessageStore` (`@Alternative @Priority(1)`) automatically in
`@QuarkusTest`. No test-specific store wiring needed.

### Testing the scoped overload

```java
var scope = MessageQuery.builder()
    .excludeTypes(List.of(MessageType.EVENT))
    .build();
var state = projectionService.project(channelId, scope, new VoteTallyProjection());
// Only QUERYs/COMMANDs/etc. folded — EVENTs excluded
```

---

## Placement summary

| Type | Module | Package |
|------|--------|---------|
| `MessageView` | `api` | `io.casehub.qhorus.api.message` |
| `ChannelProjection<S>` | `api` | `io.casehub.qhorus.api.spi` |
| `ProjectionRenderer<S>` | `api` | `io.casehub.qhorus.api.spi` |
| `ProjectionService` | `runtime` | `io.casehub.qhorus.runtime.message` |
| `ReactiveProjectionService` | `runtime` | `io.casehub.qhorus.runtime.message` |
| `toMessageView()` | `runtime` | `QhorusEntityMapper` (existing) |

---

## Platform coherence

**PLATFORM.md** — add to capability ownership table:
> Channel read-model projection (left-fold over message history) | `casehub-qhorus` | `ChannelProjection<S>` SPI + `ProjectionService` in runtime

**casehub-qhorus.md deep-dive** — add `MessageView`, `ChannelProjection<S>`,
`ProjectionRenderer<S>`, `ProjectionService` to Key Abstractions.

**Protocol PP-20260602-b748c9** (`event-log-left-fold-projection`) — to be
authored in `casehub/garden/docs/protocols/universal/` during this session.

---

## Out of scope — filed as follow-on issues

- **Incremental/cursor projection** (re-project from `afterId` without full
  replay) — drives live SSE updates. Filed as casehubio/qhorus#231.
- **MCP tool `project_channel`** — requires named-projection registry + render;
  not possible with generic `<S>`. Filed as casehubio/qhorus#232.
- **DraftHouse migration** (replace local file-parser with `DebateChannelProjection`)
  — consume this SPI. Filed as casehubio/drafthouse#31.

---

## What is NOT here

- No registry — caller always passes the projection explicitly.
- No service-side render — `ProjectionRenderer<S>` is consumer-side only.
- No new Flyway migration — pure computation over existing messages.
- No new MCP tools in this issue.
