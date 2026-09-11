# GraphQL Domain Refactor — Sub-Issue 1 of Epic #409

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #409 — Migrate qhorus MCP tools to GraphQL-backed hierarchical model
**Issue group:** #409

**Goal:** Refactor the existing `graphql/` module from a single `@McpDomain("qhorus")` to three domain-specific packages (`channels`, `governance`, `messaging`), establishing the package structure pattern for all subsequent domain migrations.

**Architecture:** The existing unified resolvers (`QhorusQueryResolver`, `QhorusMutationResolver`, `QhorusSubscriptionResolver`, `QhorusModelEnricher`) split into domain-specific resolver classes under `channels/`, `governance/`, and `messaging/` sub-packages. Each domain gets its own `@McpDomain` annotation value. DTOs remain shared in the existing `dto/` package. Tests mirror the split — each domain resolver gets its own CDI-free unit test with Mockito mocks.

**Tech Stack:** Java 21, Quarkus 3.32.2, MicroProfile GraphQL (`@Query`, `@Mutation`, `@GraphQLApi`), SmallRye GraphQL, `@McpDomain` from casehub-platform-api, Mockito for tests.

## Global Constraints

- Resolvers use `@GraphQLApi` + `@Query`/`@Mutation` (MicroProfile GraphQL), NOT `@PlatformQuery`/`@PlatformMutation`
- Each resolver class annotated with both `@GraphQLApi` and `@McpDomain("<domain-name>")`
- Resolvers inject ONLY api-layer interfaces — never runtime services
- Domain names are bare: `"channels"`, `"governance"`, `"messaging"` (no prefix)
- All edits via IntelliJ MCP (`ide_edit_member`, `ide_insert_member`, `ide_replace_member`, `ide_move_file`, `ide_refactor_rename`)
- Tests are CDI-free unit tests with Mockito (existing pattern)
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql` for running tests

---

## Batch 1: Create domain-specific resolver packages

### Task 1: Create `channels/` domain resolvers

Split channel-specific operations out of the unified resolvers into a new `channels/` sub-package. This task creates the 4 channels resolver/enricher classes.

**Files:**
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsQueryResolver.java`
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsMutationResolver.java`
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsSubscriptionResolver.java`
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsModelEnricher.java`
- Test: `graphql/src/test/java/io/casehub/qhorus/graphql/channels/ChannelsQueryResolverTest.java`
- Test: `graphql/src/test/java/io/casehub/qhorus/graphql/channels/ChannelsMutationResolverTest.java`

**Interfaces:**
- Consumes: `ChannelReader` (api/channel), `ChannelManager` (api/channel), `ConsumerMessaging` (api/message), `CurrentPrincipal` (platform-api), `QhorusEventPublisher` (graphql)
- Produces: `ChannelsQueryResolver.channels(ChannelFilterInput, PageInput) → ChannelPage`, `ChannelsQueryResolver.channel(UUID, String) → ChannelType`, `ChannelsQueryResolver.channelMessages(UUID, Long, Integer) → List<MessageType>`, `ChannelsMutationResolver.createChannel(CreateChannelInput) → ChannelType`, `ChannelsMutationResolver.deleteChannel(UUID, Boolean) → long`, `ChannelsMutationResolver.pauseChannel(UUID) → ChannelType`, `ChannelsMutationResolver.resumeChannel(UUID) → ChannelType`

- [ ] **Step 1: Write ChannelsQueryResolver test**

Create `graphql/src/test/java/io/casehub/qhorus/graphql/channels/ChannelsQueryResolverTest.java`:

```java
package io.casehub.qhorus.graphql.channels;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelReader;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.ConsumerMessaging;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.graphql.dto.ChannelFilterInput;
import io.casehub.qhorus.graphql.dto.ChannelType;
import io.casehub.qhorus.graphql.dto.MessageType;
import io.casehub.platform.graphql.PageInput;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class ChannelsQueryResolverTest {

    private ChannelsQueryResolver resolver;
    private ChannelReader channelReader;
    private ConsumerMessaging consumerMessaging;

    @BeforeEach
    void setUp() {
        channelReader = mock(ChannelReader.class);
        consumerMessaging = mock(ConsumerMessaging.class);
        resolver = new ChannelsQueryResolver();
        resolver.channelReader = channelReader;
        resolver.consumerMessaging = consumerMessaging;
    }

    @Test
    void channelsReturnsPaginatedResults() {
        Channel ch = createChannel("test-channel");
        when(channelReader.scan(any())).thenReturn(List.of(ch));

        var result = resolver.channels(null, null);

        assertThat(result.items()).hasSize(1);
        assertThat(result.items().get(0).name()).isEqualTo("test-channel");
    }

    @Test
    void channelByIdReturnsChannel() {
        UUID id = UUID.randomUUID();
        Channel ch = createChannel("found");
        when(channelReader.findById(id)).thenReturn(Optional.of(ch));

        ChannelType result = resolver.channel(id, null);

        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("found");
    }

    @Test
    void channelThrowsWhenNeitherIdNorName() {
        assertThatThrownBy(() -> resolver.channel(null, null))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void channelMessagesReturnsHistory() {
        UUID channelId = UUID.randomUUID();
        Message msg = createMessage(channelId, 1L);
        when(consumerMessaging.history(channelId, 0L, 50)).thenReturn(List.of(msg));

        List<MessageType> result = resolver.channelMessages(channelId, null, null);

        assertThat(result).hasSize(1);
    }

    private Channel createChannel(String name) {
        return new Channel(UUID.randomUUID(), name, null, ChannelSemantic.APPEND,
                false, null, null, null, null, null, null, null, null,
                null, null, null, null, null, null, null, null, null, null, null, null);
    }

    private Message createMessage(UUID channelId, Long id) {
        return new Message(id, channelId, "sender", io.casehub.qhorus.api.message.MessageType.STATUS,
                "content", null, null, null, null, null, null, null, Instant.now());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ChannelsQueryResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `ChannelsQueryResolver` class does not exist yet.

- [ ] **Step 3: Create ChannelsQueryResolver**

Create `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsQueryResolver.java`:

```java
package io.casehub.qhorus.graphql.channels;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.graphql.PageInfo;
import io.casehub.platform.graphql.PageInput;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelReader;
import io.casehub.qhorus.api.message.ConsumerMessaging;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.store.query.ChannelQuery;
import io.casehub.qhorus.graphql.dto.ChannelFilterInput;
import io.casehub.qhorus.graphql.dto.ChannelPage;
import io.casehub.qhorus.graphql.dto.ChannelType;
import io.casehub.qhorus.graphql.dto.MessageType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.UUID;
import org.eclipse.microprofile.graphql.Description;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Query;

@GraphQLApi
@McpDomain("channels")
@ApplicationScoped
public class ChannelsQueryResolver {

    @Inject ChannelReader channelReader;
    @Inject ConsumerMessaging consumerMessaging;

    @Query
    @Description("List channels with optional filtering and pagination")
    public ChannelPage channels(ChannelFilterInput filter, PageInput page) {
        int offset = page != null && page.offset() != null ? page.offset() : 0;
        int limit = page != null && page.limit() != null ? page.limit() : 20;

        ChannelQuery.Builder queryBuilder = ChannelQuery.builder();
        if (filter != null) {
            if (filter.keyword() != null) queryBuilder.keyword(filter.keyword());
            if (filter.namePrefix() != null) queryBuilder.namePrefix(filter.namePrefix());
            if (filter.semantic() != null) queryBuilder.semantic(filter.semantic());
            if (filter.paused() != null) queryBuilder.paused(filter.paused());
            if (filter.spaceId() != null) queryBuilder.spaceId(filter.spaceId());
        }

        List<Channel> all = channelReader.scan(queryBuilder.build());
        int total = all.size();
        int end = Math.min(offset + limit, total);
        List<ChannelType> items = offset < total
                ? all.subList(offset, end).stream().map(ChannelType::from).toList()
                : List.of();

        boolean hasNext = end < total;
        boolean hasPrevious = offset > 0;
        return new ChannelPage(items, new PageInfo(hasNext, hasPrevious, total, null));
    }

    @Query
    @Description("Retrieve a single channel by ID or name")
    public ChannelType channel(UUID id, String name) {
        if (id != null) {
            return channelReader.findById(id).map(ChannelType::from).orElse(null);
        }
        if (name != null) {
            return channelReader.findByName(name).map(ChannelType::from).orElse(null);
        }
        throw new IllegalArgumentException("Either id or name must be provided");
    }

    @Query
    @Description("Retrieve message history for a channel — cursor-based with afterId")
    public List<MessageType> channelMessages(UUID channelId, Long afterId, Integer limit) {
        long cursor = afterId != null ? afterId : 0;
        int maxMessages = limit != null ? limit : 50;

        List<Message> messages = consumerMessaging.history(channelId, cursor, maxMessages);
        return messages.stream().map(MessageType::from).toList();
    }
}
```

- [ ] **Step 4: Create ChannelsMutationResolver**

Create `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsMutationResolver.java`:

```java
package io.casehub.qhorus.graphql.channels;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelManager;
import io.casehub.qhorus.graphql.dto.ChannelType;
import io.casehub.qhorus.graphql.dto.CreateChannelInput;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.UUID;
import org.eclipse.microprofile.graphql.Description;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Mutation;

@GraphQLApi
@McpDomain("channels")
@ApplicationScoped
public class ChannelsMutationResolver {

    @Inject ChannelManager channelManager;

    @Mutation
    @Description("Create a new communication channel")
    public ChannelType createChannel(CreateChannelInput input) {
        ChannelCreateRequest.Builder builder = ChannelCreateRequest.builder(input.name());
        if (input.description() != null) builder.description(input.description());
        if (input.semantic() != null) builder.semantic(input.semantic());
        if (input.barrierContributors() != null) builder.barrierContributors(input.barrierContributors());
        if (input.allowedWriters() != null) builder.allowedWriters(input.allowedWriters());
        if (input.adminInstances() != null) builder.adminInstances(input.adminInstances());
        if (input.rateLimitPerChannel() != null) builder.rateLimitPerChannel(input.rateLimitPerChannel());
        if (input.rateLimitPerInstance() != null) builder.rateLimitPerInstance(input.rateLimitPerInstance());
        if (input.spaceId() != null) builder.spaceId(input.spaceId());
        if (input.protocols() != null) builder.protocols(input.protocols());

        Channel channel = channelManager.create(builder.build());
        return ChannelType.from(channel);
    }

    @Mutation
    @Description("Delete a channel — optionally force-delete even if it has messages")
    public long deleteChannel(UUID channelId, Boolean force) {
        return channelManager.delete(channelId, force != null && force);
    }

    @Mutation
    @Description("Pause a channel — blocks new message dispatch")
    public ChannelType pauseChannel(UUID channelId) {
        Channel channel = channelManager.pause(channelId);
        return ChannelType.from(channel);
    }

    @Mutation
    @Description("Resume a paused channel — allows message dispatch again")
    public ChannelType resumeChannel(UUID channelId) {
        Channel channel = channelManager.resume(channelId);
        return ChannelType.from(channel);
    }
}
```

- [ ] **Step 5: Create ChannelsSubscriptionResolver**

Create `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsSubscriptionResolver.java` — move from `QhorusSubscriptionResolver` with `@McpDomain("channels")`:

```java
package io.casehub.qhorus.graphql.channels;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.qhorus.graphql.QhorusEventPublisher;
import io.casehub.qhorus.graphql.dto.MessageType;
import io.casehub.qhorus.graphql.dto.PresenceType;
import io.smallrye.graphql.api.Subscription;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.UUID;
import org.eclipse.microprofile.graphql.Description;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Name;

@GraphQLApi
@McpDomain("channels")
@ApplicationScoped
public class ChannelsSubscriptionResolver {

    private final QhorusEventPublisher publisher;

    @Inject
    public ChannelsSubscriptionResolver(QhorusEventPublisher publisher) {
        this.publisher = publisher;
    }

    @Subscription
    @Description("Live channel activity — new messages dispatched to a channel")
    public Multi<MessageType> channelActivity(@Name("channelId") UUID channelId) {
        return publisher.messageStream()
                .filter(event -> event.channelId().equals(channelId))
                .map(event -> new MessageType(
                        event.messageId(),
                        event.channelId(),
                        event.senderId(),
                        event.messageType() != null ? event.messageType().name() : null,
                        event.actorType() != null ? event.actorType().name() : null,
                        event.content(),
                        event.correlationId(),
                        null,
                        0,
                        event.target(),
                        event.topic(),
                        null,
                        event.occurredAt()));
    }

    @Subscription
    @Description("Live presence changes — member status transitions")
    public Multi<PresenceType> channelPresence(@Name("channelId") UUID channelId) {
        return publisher.presenceStream()
                .filter(event -> channelId == null || channelId.equals(event.channelId()))
                .map(PresenceType::fromEvent);
    }
}
```

- [ ] **Step 6: Create ChannelsModelEnricher**

Create `graphql/src/main/java/io/casehub/qhorus/graphql/channels/ChannelsModelEnricher.java`:

```java
package io.casehub.qhorus.graphql.channels;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.ModelEnricher;
import io.casehub.qhorus.api.channel.ChannelReader;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@McpDomain("channels")
@ApplicationScoped
public class ChannelsModelEnricher implements ModelEnricher {

    private final ChannelReader channelReader;

    @Inject
    public ChannelsModelEnricher(ChannelReader channelReader) {
        this.channelReader = channelReader;
    }

    @Override
    public String summary() {
        return "Communication channels — create, query, pause, resume, delete channels. "
                + "Retrieve message history. Subscribe to live channel activity and presence.";
    }

    @Override
    public Map<String, Object> state() {
        int count = channelReader.listAll().size();
        return Map.of("activeChannels", count);
    }
}
```

- [ ] **Step 7: Write ChannelsMutationResolverTest**

Create `graphql/src/test/java/io/casehub/qhorus/graphql/channels/ChannelsMutationResolverTest.java`:

```java
package io.casehub.qhorus.graphql.channels;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelManager;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.graphql.dto.ChannelType;
import io.casehub.qhorus.graphql.dto.CreateChannelInput;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class ChannelsMutationResolverTest {

    private ChannelsMutationResolver resolver;
    private ChannelManager channelManager;

    @BeforeEach
    void setUp() {
        channelManager = mock(ChannelManager.class);
        resolver = new ChannelsMutationResolver();
        resolver.channelManager = channelManager;
    }

    @Test
    void createChannelDelegatesToManager() {
        Channel created = createChannel("new-channel");
        when(channelManager.create(any(ChannelCreateRequest.class))).thenReturn(created);

        CreateChannelInput input = new CreateChannelInput("new-channel", null,
                null, null, null, null, null, null, null, null);
        ChannelType result = resolver.createChannel(input);

        assertThat(result.name()).isEqualTo("new-channel");
    }

    @Test
    void pauseChannelReturnsUpdatedChannel() {
        UUID id = UUID.randomUUID();
        Channel paused = createChannel("paused-ch");
        when(channelManager.pause(id)).thenReturn(paused);

        ChannelType result = resolver.pauseChannel(id);

        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("paused-ch");
    }

    @Test
    void deleteChannelDelegatesToManager() {
        UUID id = UUID.randomUUID();
        when(channelManager.delete(id, true)).thenReturn(5L);

        long result = resolver.deleteChannel(id, true);

        assertThat(result).isEqualTo(5L);
    }

    private Channel createChannel(String name) {
        return new Channel(UUID.randomUUID(), name, null, ChannelSemantic.APPEND,
                false, null, null, null, null, null, null, null, null,
                null, null, null, null, null, null, null, null, null, null, null, null);
    }
}
```

- [ ] **Step 8: Run tests to verify channels domain passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest="ChannelsQueryResolverTest,ChannelsMutationResolverTest"`
Expected: PASS — all tests green.

- [ ] **Step 9: Commit channels domain**

```bash
git add graphql/src/main/java/io/casehub/qhorus/graphql/channels/
git add graphql/src/test/java/io/casehub/qhorus/graphql/channels/
git commit -m "feat(graphql): create channels domain resolvers — @McpDomain(\"channels\") Refs #409"
```

---

### Task 2: Create `governance/` and `messaging/` domain resolvers

Split the commitment query and dispatchMessage mutation into their respective domain packages.

**Files:**
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/governance/GovernanceQueryResolver.java`
- Create: `graphql/src/main/java/io/casehub/qhorus/graphql/messaging/MessagingMutationResolver.java`
- Test: `graphql/src/test/java/io/casehub/qhorus/graphql/governance/GovernanceQueryResolverTest.java`
- Test: `graphql/src/test/java/io/casehub/qhorus/graphql/messaging/MessagingMutationResolverTest.java`

**Interfaces:**
- Consumes: `CommitmentReader` (api/store), `MessageDispatcher` (api/message), `CurrentPrincipal` (platform-api)
- Produces: `GovernanceQueryResolver.commitments(CommitmentFilterInput, PageInput) → CommitmentPage`, `MessagingMutationResolver.dispatchMessage(DispatchMessageInput) → DispatchResultType`

- [ ] **Step 1: Write GovernanceQueryResolverTest**

Create `graphql/src/test/java/io/casehub/qhorus/graphql/governance/GovernanceQueryResolverTest.java`:

```java
package io.casehub.qhorus.graphql.governance;

import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.store.CommitmentReader;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class GovernanceQueryResolverTest {

    private GovernanceQueryResolver resolver;
    private CommitmentReader commitmentReader;

    @BeforeEach
    void setUp() {
        commitmentReader = mock(CommitmentReader.class);
        resolver = new GovernanceQueryResolver();
        resolver.commitmentReader = commitmentReader;
    }

    @Test
    void commitmentsReturnsOpenByDefault() {
        Commitment c = createCommitment(CommitmentState.OPEN);
        when(commitmentReader.findAllOpen()).thenReturn(List.of(c));

        var result = resolver.commitments(null, null);

        assertThat(result.items()).hasSize(1);
    }

    @Test
    void commitmentsFiltersByObligor() {
        Commitment c = createCommitment(CommitmentState.OPEN);
        when(commitmentReader.findOpenByObligor("agent-1")).thenReturn(List.of(c));

        var filter = new io.casehub.qhorus.graphql.dto.CommitmentFilterInput(
                null, null, "agent-1", null);
        var result = resolver.commitments(filter, null);

        assertThat(result.items()).hasSize(1);
    }

    private Commitment createCommitment(CommitmentState state) {
        return new Commitment(UUID.randomUUID(), UUID.randomUUID(), "corr-1",
                "obligor-1", "requester-1", state, null, null, Instant.now(), null, null, null);
    }
}
```

- [ ] **Step 2: Create GovernanceQueryResolver**

Create `graphql/src/main/java/io/casehub/qhorus/graphql/governance/GovernanceQueryResolver.java`:

```java
package io.casehub.qhorus.graphql.governance;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.graphql.PageInfo;
import io.casehub.platform.graphql.PageInput;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.store.CommitmentReader;
import io.casehub.qhorus.graphql.dto.CommitmentFilterInput;
import io.casehub.qhorus.graphql.dto.CommitmentPage;
import io.casehub.qhorus.graphql.dto.CommitmentType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import org.eclipse.microprofile.graphql.Description;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Query;

@GraphQLApi
@McpDomain("governance")
@ApplicationScoped
public class GovernanceQueryResolver {

    @Inject CommitmentReader commitmentReader;

    @Query
    @Description("List commitments with optional filtering by channel, state, obligor, or requester")
    public CommitmentPage commitments(CommitmentFilterInput filter, PageInput page) {
        int offset = page != null && page.offset() != null ? page.offset() : 0;
        int limit = page != null && page.limit() != null ? page.limit() : 20;

        List<Commitment> all = resolveCommitments(filter);
        int total = all.size();
        int end = Math.min(offset + limit, total);
        List<CommitmentType> items = offset < total
                ? all.subList(offset, end).stream().map(CommitmentType::from).toList()
                : List.of();

        boolean hasNext = end < total;
        boolean hasPrevious = offset > 0;
        return new CommitmentPage(items, new PageInfo(hasNext, hasPrevious, total, null));
    }

    private List<Commitment> resolveCommitments(CommitmentFilterInput filter) {
        if (filter == null) {
            return commitmentReader.findAllOpen();
        }
        if (filter.channelId() != null && filter.state() != null) {
            return commitmentReader.findByState(filter.state(), filter.channelId());
        }
        if (filter.channelId() != null && filter.obligor() != null) {
            return commitmentReader.findOpenByObligor(filter.obligor(), filter.channelId());
        }
        if (filter.channelId() != null && filter.requester() != null) {
            return commitmentReader.findOpenByRequester(filter.requester(), filter.channelId());
        }
        if (filter.channelId() != null) {
            return commitmentReader.findByChannel(filter.channelId());
        }
        if (filter.obligor() != null) {
            return commitmentReader.findOpenByObligor(filter.obligor());
        }
        return commitmentReader.findAllOpen();
    }
}
```

- [ ] **Step 3: Write MessagingMutationResolverTest**

Create `graphql/src/test/java/io/casehub/qhorus/graphql/messaging/MessagingMutationResolverTest.java`:

```java
package io.casehub.qhorus.graphql.messaging;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.graphql.dto.DispatchMessageInput;
import io.casehub.qhorus.graphql.dto.DispatchResultType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class MessagingMutationResolverTest {

    private MessagingMutationResolver resolver;
    private MessageDispatcher messageDispatcher;
    private CurrentPrincipal currentPrincipal;

    @BeforeEach
    void setUp() {
        messageDispatcher = mock(MessageDispatcher.class);
        currentPrincipal = mock(CurrentPrincipal.class);
        when(currentPrincipal.actorId()).thenReturn("test-actor");
        when(currentPrincipal.tenancyId()).thenReturn("default");
        resolver = new MessagingMutationResolver();
        resolver.messageDispatcher = messageDispatcher;
        resolver.currentPrincipal = currentPrincipal;
    }

    @Test
    void dispatchMessageBuildsDispatchFromInput() {
        UUID channelId = UUID.randomUUID();
        DispatchResult result = new DispatchResult(1L, "test-channel", "test-actor",
                "STATUS", null, null, null, List.of());
        when(messageDispatcher.dispatch(any(MessageDispatch.class))).thenReturn(result);

        DispatchMessageInput input = new DispatchMessageInput(
                channelId, "STATUS", "hello", null, null, null, null, null);
        DispatchResultType actual = resolver.dispatchMessage(input);

        assertThat(actual).isNotNull();

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        org.mockito.Mockito.verify(messageDispatcher).dispatch(captor.capture());
        MessageDispatch dispatched = captor.getValue();
        assertThat(dispatched.channelId()).isEqualTo(channelId);
        assertThat(dispatched.sender()).isEqualTo("test-actor");
    }
}
```

- [ ] **Step 4: Create MessagingMutationResolver**

Create `graphql/src/main/java/io/casehub/qhorus/graphql/messaging/MessagingMutationResolver.java`:

```java
package io.casehub.qhorus.graphql.messaging;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.graphql.dto.DispatchMessageInput;
import io.casehub.qhorus.graphql.dto.DispatchResultType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.graphql.Description;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Mutation;

@GraphQLApi
@McpDomain("messaging")
@ApplicationScoped
public class MessagingMutationResolver {

    @Inject MessageDispatcher messageDispatcher;
    @Inject CurrentPrincipal currentPrincipal;

    @Mutation
    @Description("Dispatch a typed message to a channel")
    public DispatchResultType dispatchMessage(DispatchMessageInput input) {
        String actorId = currentPrincipal.actorId();
        String tenancyId = currentPrincipal.tenancyId();

        MessageDispatch.Builder builder = MessageDispatch.builder()
                .channelId(input.channelId())
                .sender(actorId)
                .type(io.casehub.qhorus.api.message.MessageType.valueOf(input.type()))
                .actorType(ActorType.HUMAN)
                .tenancyId(tenancyId);

        if (input.content() != null) builder.content(input.content());
        if (input.correlationId() != null) builder.correlationId(input.correlationId());
        if (input.inReplyTo() != null) builder.inReplyTo(input.inReplyTo());
        if (input.target() != null) builder.target(input.target());
        if (input.topic() != null) builder.topic(input.topic());
        if (input.deadline() != null) builder.deadline(input.deadline());

        DispatchResult result = messageDispatcher.dispatch(builder.build());
        return DispatchResultType.from(result);
    }
}
```

- [ ] **Step 5: Run all new domain tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest="GovernanceQueryResolverTest,MessagingMutationResolverTest"`
Expected: PASS

- [ ] **Step 6: Commit governance and messaging domains**

```bash
git add graphql/src/main/java/io/casehub/qhorus/graphql/governance/
git add graphql/src/main/java/io/casehub/qhorus/graphql/messaging/
git add graphql/src/test/java/io/casehub/qhorus/graphql/governance/
git add graphql/src/test/java/io/casehub/qhorus/graphql/messaging/
git commit -m "feat(graphql): create governance and messaging domain resolvers Refs #409"
```

---

## Batch 2: Delete old resolvers and verify domain registration

### Task 3: Remove unified resolvers and update tests

Delete the original unified resolver classes and their tests. The domain-specific resolvers now own all operations.

**Files:**
- Delete: `graphql/src/main/java/io/casehub/qhorus/graphql/QhorusQueryResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `graphql/src/main/java/io/casehub/qhorus/graphql/QhorusMutationResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `graphql/src/main/java/io/casehub/qhorus/graphql/QhorusSubscriptionResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `graphql/src/main/java/io/casehub/qhorus/graphql/QhorusModelEnricher.java` (use `ide_refactor_safe_delete`)
- Delete: `graphql/src/test/java/io/casehub/qhorus/graphql/QhorusQueryResolverTest.java`
- Delete: `graphql/src/test/java/io/casehub/qhorus/graphql/QhorusMutationResolverTest.java`
- Delete: `graphql/src/test/java/io/casehub/qhorus/graphql/QhorusModelEnricherTest.java`

- [ ] **Step 1: Safe-delete old main resolver classes**

Use `ide_refactor_safe_delete` for each:
- `QhorusQueryResolver.java`
- `QhorusMutationResolver.java`
- `QhorusSubscriptionResolver.java`
- `QhorusModelEnricher.java`

If safe-delete reports usages, check each — they should be test-only (which we're also deleting).

- [ ] **Step 2: Delete old test classes**

Delete via bash (test files, not source files):
```bash
rm graphql/src/test/java/io/casehub/qhorus/graphql/QhorusQueryResolverTest.java
rm graphql/src/test/java/io/casehub/qhorus/graphql/QhorusMutationResolverTest.java
rm graphql/src/test/java/io/casehub/qhorus/graphql/QhorusModelEnricherTest.java
```

Keep `QhorusEventPublisherTest.java` — `QhorusEventPublisher` is still shared infrastructure used by `ChannelsSubscriptionResolver`.

- [ ] **Step 3: Run full graphql module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql`
Expected: PASS — all domain-specific tests pass, no references to deleted classes.

- [ ] **Step 4: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — no broken imports across the project. The compliance-report module also compiles (its resolvers still use `@McpDomain("qhorus")` — refactored in a later sub-issue).

- [ ] **Step 5: Verify via diagnostics**

Run `ide_diagnostics` on the `graphql` module to confirm no compilation errors remain.

- [ ] **Step 6: Commit cleanup**

```bash
git add -A graphql/
git commit -m "refactor(graphql): remove unified resolvers — replaced by domain-specific packages Refs #409"
```

---

### Task 4: Domain registration verification test

Write a test that verifies `GraphQLModelScanner` discovers the 3 new domains when the graphql module is on the classpath. This is a compile-time verification — not a full Quarkus integration test.

**Files:**
- Create: `graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java`

**Interfaces:**
- Consumes: `@McpDomain` annotations on resolver classes
- Produces: Test verification that domain annotations are present and correct

- [ ] **Step 1: Write domain annotation verification test**

Create `graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java`:

```java
package io.casehub.qhorus.graphql;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.qhorus.graphql.channels.ChannelsQueryResolver;
import io.casehub.qhorus.graphql.channels.ChannelsMutationResolver;
import io.casehub.qhorus.graphql.channels.ChannelsSubscriptionResolver;
import io.casehub.qhorus.graphql.channels.ChannelsModelEnricher;
import io.casehub.qhorus.graphql.governance.GovernanceQueryResolver;
import io.casehub.qhorus.graphql.messaging.MessagingMutationResolver;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DomainRegistrationTest {

    @Test
    void channelsDomainAnnotationsPresent() {
        assertDomain(ChannelsQueryResolver.class, "channels", true);
        assertDomain(ChannelsMutationResolver.class, "channels", true);
        assertDomain(ChannelsSubscriptionResolver.class, "channels", true);
        assertDomain(ChannelsModelEnricher.class, "channels", false);
    }

    @Test
    void governanceDomainAnnotationsPresent() {
        assertDomain(GovernanceQueryResolver.class, "governance", true);
    }

    @Test
    void messagingDomainAnnotationsPresent() {
        assertDomain(MessagingMutationResolver.class, "messaging", true);
    }

    @Test
    void noQhorusDomainRemains() {
        Class<?>[] classes = {
                ChannelsQueryResolver.class, ChannelsMutationResolver.class,
                ChannelsSubscriptionResolver.class, ChannelsModelEnricher.class,
                GovernanceQueryResolver.class, MessagingMutationResolver.class
        };
        for (Class<?> cls : classes) {
            McpDomain ann = cls.getAnnotation(McpDomain.class);
            assertThat(ann.value())
                    .describedAs("Class %s should not use 'qhorus' domain", cls.getSimpleName())
                    .isNotEqualTo("qhorus");
        }
    }

    private void assertDomain(Class<?> cls, String expectedDomain, boolean expectGraphQLApi) {
        McpDomain mcpDomain = cls.getAnnotation(McpDomain.class);
        assertThat(mcpDomain)
                .describedAs("@McpDomain missing on %s", cls.getSimpleName())
                .isNotNull();
        assertThat(mcpDomain.value()).isEqualTo(expectedDomain);

        if (expectGraphQLApi) {
            assertThat(cls.getAnnotation(GraphQLApi.class))
                    .describedAs("@GraphQLApi missing on %s", cls.getSimpleName())
                    .isNotNull();
        }
    }
```

- [ ] **Step 2: Run the verification test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=DomainRegistrationTest`
Expected: PASS — all 4 tests green.

- [ ] **Step 3: Run full graphql module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql`
Expected: PASS — all tests pass.

- [ ] **Step 4: Commit**

```bash
git add graphql/src/test/java/io/casehub/qhorus/graphql/DomainRegistrationTest.java
git commit -m "test(graphql): domain registration verification — channels, governance, messaging Refs #409"
```

---

## References

- [2026-09-11-graphql-mcp-migration-design.md] — design spec this plan implements
- [graphql/src/main/java/io/casehub/qhorus/graphql/QhorusQueryResolver.java] — existing unified query resolver (being split)
- [graphql/src/main/java/io/casehub/qhorus/graphql/QhorusMutationResolver.java] — existing unified mutation resolver (being split)
- [graphql/src/main/java/io/casehub/qhorus/graphql/QhorusSubscriptionResolver.java] — existing unified subscription resolver (being moved)
- [graphql/src/main/java/io/casehub/qhorus/graphql/QhorusModelEnricher.java] — existing enricher (being moved)
- [mcp/src/main/java/io/casehub/platform/mcp/GraphQLModelScanner.java] — platform scanner that discovers @McpDomain + @GraphQLApi
- [mcp/src/main/java/io/casehub/platform/mcp/DynamicToolRegistrar.java] — platform tool registration from domains
- [GitHub #409] — epic: migrate qhorus MCP tools to GraphQL-backed hierarchical model
- [GitHub #401] — established the GraphQL resolver pattern
