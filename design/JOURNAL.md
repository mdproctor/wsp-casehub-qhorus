# Design Journal — issue-230-channel-projection-spi

### 2026-06-02 · §Component Structure

Three new types in the `api` module: `MessageView` (read-side DTO in `api/message/`),
`ChannelProjection<S>` (pure left-fold SPI in `api/spi/`), and `ProjectionResult<S>`
(fold result carrier in `api/spi/`). The key layering decision was to introduce `MessageView`
as a proxy for the JPA `Message` entity — `ChannelProjection` must live in `api/spi/` so
consumers depend only on the lightweight `api` module, not `runtime/`. Using the entity
directly would force `ChannelProjection.apply()` to take a `Message` parameter, which is a
JPA type in `runtime/`, breaking the module boundary. `MessageView` is the canonical read-side
projection of a message; it will eventually be useful for timeline endpoints and dashboard
read-models beyond projection alone.

`ChannelProjection<S>` deliberately omits `channelType()`. A future CDI-based registry will
use a `@ChannelBound` qualifier annotation rather than a String return on the interface,
avoiding a breaking change when the registry's dispatch key type needs to change.

### 2026-06-02 · §Services

`ProjectionService` and `ReactiveProjectionService` (build-gated on
`casehub.qhorus.reactive.enabled`) provide four overloads each: full scan, scoped-full,
incremental, and scoped-incremental. All return `ProjectionResult<S>` instead of raw `<S>`
— the result carries `lastMessageId` so the caller can resume the fold from a cursor without
re-reading earlier messages. The incremental overload enforces that `previous.isEmpty()` always
starts from `identity()` regardless of `previous.state()`, making `lastMessageId == null`
an unambiguous "fold from the beginning" signal that cannot be subverted by manually
constructing a `ProjectionResult` with inconsistent fields.

Scope validation (`validateScope()`) rejects two misuses before any store access: a
conflicting `channelId` in the scope (ambiguous intent) and `descending(true)` in the scope
(a right-fold masquerading as a left-fold that silently corrupts state-machine projections).
All reads go through `MessageStore.scan()` — no direct Panache access from services.

### 2026-06-02 · §Persistence Abstraction

`ReactiveMessageStore.stream(MessageQuery)` → `Multi<Message>` is added to enable the
`ReactiveProjectionService` fold without materialising a full `List<Message>` first.
The `ReactiveJpaMessageStore` implementation currently wraps `scan().toMulti()` because
Quarkus 3.32 Hibernate Reactive Panache does not expose `PanacheQuery.stream()`. The
interface is the right shape for future upgrading: when Hibernate Reactive exposes
cursor-backed streaming, only the JPA implementation changes. `InMemoryReactiveMessageStore`
wraps its in-memory list as `Multi.createFrom().iterable()` — correct for consumer unit
tests. A `StubReactiveMessageStore` (`@DefaultBean @ApplicationScoped`) is added to the
runtime test sources to prevent CDI resolution failures when the reactive build property is
enabled in profiles without a reactive datasource.

