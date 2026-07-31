# REST API for Channel CRUD — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #387 — REST API for channel CRUD
**Issue group:** #387

**Goal:** Add a JAX-RS resource exposing channel lifecycle operations over REST, so consumers like Claudony stop reimplementing the same thin layer.

**Architecture:** Single `ChannelResource` in `runtime/api/` delegates to `ChannelService`. REST-specific request/response records with typed collections (no CSV strings). Channel dual-identity resolution (UUID or slug) at the resource boundary.

**Tech Stack:** Quarkus JAX-RS, Jackson, RestAssured

## Global Constraints

- Always-on — no config gate
- Blocking only — no reactive counterpart
- Connector binding fields excluded from REST API
- Channel `{id}` path param accepts UUID or slug name (dual-identity protocol PP-20260612-ch-dual-id)
- All mutation endpoints return `ChannelResponse` with updated state
- Error body: `{"error": "message"}`

---

### Task 1: ChannelResponse record + toChannelResponse mapper

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/api/ChannelResponseTest.java`

**Interfaces:**
- Consumes: `Channel` record from `io.casehub.qhorus.api.channel.Channel` (fields: `id`, `name`, `description`, `semantic`, `barrierContributors`, `allowedWriters`, `adminInstances`, `rateLimitPerChannel`, `rateLimitPerInstance`, `allowedTypes`, `deniedTypes`, `paused`, `spaceId`, `reviewerInstances`, `protocols`, `protocolParticipants`, `trackDelivery`, `lastActivityAt`)
- Produces: `ChannelResponse` record used by all resource endpoints; `ChannelResponse.from(Channel ch, long messageCount, String spaceName)` static factory

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.qhorus.api;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.api.ChannelResponse;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class ChannelResponseTest {

    @Test
    void fromChannelMapsAllFields() {
        var id = UUID.randomUUID();
        var spaceId = UUID.randomUUID();
        var ch = Channel.builder("test-channel")
                .id(id).description("desc").semantic(ChannelSemantic.BARRIER)
                .barrierContributors(List.of("agent-a", "agent-b"))
                .allowedWriters(List.of("writer-1"))
                .adminInstances(List.of("admin-1"))
                .reviewerInstances(List.of("reviewer-1"))
                .allowedTypes(Set.of(MessageType.QUERY, MessageType.RESPONSE))
                .deniedTypes(Set.of(MessageType.EVENT))
                .rateLimitPerChannel(100).rateLimitPerInstance(10)
                .protocols(List.of("REQUEST_RESPONSE"))
                .protocolParticipants(List.of("agent-a", "agent-b"))
                .trackDelivery(true).spaceId(spaceId)
                .paused(true).lastActivityAt(Instant.parse("2026-07-30T10:00:00Z"))
                .build();

        var response = ChannelResponse.from(ch, 42L, "my-space");

        assertThat(response.channelId()).isEqualTo(id);
        assertThat(response.name()).isEqualTo("test-channel");
        assertThat(response.description()).isEqualTo("desc");
        assertThat(response.semantic()).isEqualTo("BARRIER");
        assertThat(response.messageCount()).isEqualTo(42L);
        assertThat(response.paused()).isTrue();
        assertThat(response.barrierContributors()).containsExactly("agent-a", "agent-b");
        assertThat(response.allowedWriters()).containsExactly("writer-1");
        assertThat(response.adminInstances()).containsExactly("admin-1");
        assertThat(response.reviewerInstances()).containsExactly("reviewer-1");
        assertThat(response.allowedTypes()).containsExactlyInAnyOrder("QUERY", "RESPONSE");
        assertThat(response.deniedTypes()).containsExactlyInAnyOrder("EVENT");
        assertThat(response.rateLimitPerChannel()).isEqualTo(100);
        assertThat(response.rateLimitPerInstance()).isEqualTo(10);
        assertThat(response.protocols()).containsExactly("REQUEST_RESPONSE");
        assertThat(response.protocolParticipants()).containsExactly("agent-a", "agent-b");
        assertThat(response.trackDelivery()).isTrue();
        assertThat(response.spaceId()).isEqualTo(spaceId);
        assertThat(response.spaceName()).isEqualTo("my-space");
        assertThat(response.lastActivityAt()).isEqualTo("2026-07-30T10:00:00Z");
    }

    @Test
    void fromChannelHandlesNulls() {
        var ch = Channel.builder("minimal").build();
        var response = ChannelResponse.from(ch, 0L, null);

        assertThat(response.name()).isEqualTo("minimal");
        assertThat(response.semantic()).isEqualTo("APPEND");
        assertThat(response.barrierContributors()).isEmpty();
        assertThat(response.allowedTypes()).isNull();
        assertThat(response.deniedTypes()).isNull();
        assertThat(response.spaceName()).isNull();
        assertThat(response.lastActivityAt()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelResponseTest -pl runtime`
Expected: FAIL — `ChannelResponse` class does not exist

- [ ] **Step 3: Write the implementation**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java`:

```java
package io.casehub.qhorus.runtime.api;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.message.MessageType;

import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

public record ChannelResponse(
        UUID channelId,
        String name,
        String description,
        String semantic,
        long messageCount,
        String lastActivityAt,
        boolean paused,
        List<String> barrierContributors,
        List<String> allowedWriters,
        List<String> adminInstances,
        List<String> reviewerInstances,
        Set<String> allowedTypes,
        Set<String> deniedTypes,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        UUID spaceId,
        String spaceName,
        List<String> protocols,
        List<String> protocolParticipants,
        Boolean trackDelivery) {

    public static ChannelResponse from(Channel ch, long messageCount, String spaceName) {
        return new ChannelResponse(
                ch.id(), ch.name(), ch.description(),
                ch.semantic() != null ? ch.semantic().name() : null,
                messageCount,
                ch.lastActivityAt() != null ? ch.lastActivityAt().toString() : null,
                ch.paused(),
                ch.barrierContributors(),
                ch.allowedWriters(),
                ch.adminInstances(),
                ch.reviewerInstances(),
                ch.allowedTypes() != null
                        ? ch.allowedTypes().stream().map(MessageType::name).collect(Collectors.toSet())
                        : null,
                ch.deniedTypes() != null
                        ? ch.deniedTypes().stream().map(MessageType::name).collect(Collectors.toSet())
                        : null,
                ch.rateLimitPerChannel(),
                ch.rateLimitPerInstance(),
                ch.spaceId(), spaceName,
                ch.protocols(),
                ch.protocolParticipants(),
                ch.trackDelivery());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelResponseTest -pl runtime`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java runtime/src/test/java/io/casehub/qhorus/api/ChannelResponseTest.java
git commit -m "feat(#387): ChannelResponse record with from(Channel) factory"
```

---

### Task 2: ChannelResource — CRUD endpoints (POST, GET list, GET by id, DELETE)

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/api/ChannelResourceTest.java`

**Interfaces:**
- Consumes: `ChannelResponse.from(Channel, long, String)` from Task 1; `ChannelService.create(ChannelCreateRequest)`, `ChannelService.findById(UUID)`, `ChannelService.findByName(String)`, `ChannelService.listAll()`, `ChannelService.scan(ChannelQuery)`, `ChannelService.delete(UUID, boolean)`; `MessageStore.countByChannel(UUID)` for message count; `SpaceStore.findByIds(Collection<UUID>)` for space name resolution
- Produces: `ChannelResource` JAX-RS resource with `POST /api/channels`, `GET /api/channels`, `GET /api/channels/{id}`, `DELETE /api/channels/{id}`; inner records `CreateChannelRequest`, `ErrorResponse`

- [ ] **Step 1: Write the failing integration test for POST**

```java
package io.casehub.qhorus.api;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ChannelResourceTest {

    @Test
    void createChannelReturns201() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"name": "test-create", "description": "A test channel"}
                """)
            .when().post("/api/channels")
            .then()
            .statusCode(201)
            .body("name", equalTo("test-create"))
            .body("description", equalTo("A test channel"))
            .body("semantic", equalTo("APPEND"))
            .body("channelId", notNullValue())
            .body("paused", equalTo(false));
    }

    @Test
    void createChannelWithInvalidSlugReturns400() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"name": "INVALID SLUG!"}
                """)
            .when().post("/api/channels")
            .then()
            .statusCode(400)
            .body("error", notNullValue());
    }

    @Test
    void listChannelsReturnsCreatedChannels() {
        // Create two channels
        given().contentType(ContentType.JSON)
            .body("""
                {"name": "list-test-a"}
                """)
            .when().post("/api/channels").then().statusCode(201);

        given().contentType(ContentType.JSON)
            .body("""
                {"name": "list-test-b"}
                """)
            .when().post("/api/channels").then().statusCode(201);

        given()
            .when().get("/api/channels")
            .then()
            .statusCode(200)
            .body("findAll { it.name == 'list-test-a' }.size()", equalTo(1))
            .body("findAll { it.name == 'list-test-b' }.size()", equalTo(1));
    }

    @Test
    void listChannelsWithPrefixFilter() {
        given().contentType(ContentType.JSON)
            .body("""
                {"name": "prefix-alpha"}
                """)
            .when().post("/api/channels").then().statusCode(201);

        given()
            .queryParam("prefix", "prefix-")
            .when().get("/api/channels")
            .then()
            .statusCode(200)
            .body("findAll { it.name.startsWith('prefix-') }.size()", greaterThanOrEqualTo(1));
    }

    @Test
    void getChannelByUuid() {
        String channelId = given().contentType(ContentType.JSON)
            .body("""
                {"name": "get-by-uuid"}
                """)
            .when().post("/api/channels")
            .then().statusCode(201)
            .extract().path("channelId");

        given()
            .when().get("/api/channels/{id}", channelId)
            .then()
            .statusCode(200)
            .body("channelId", equalTo(channelId))
            .body("name", equalTo("get-by-uuid"));
    }

    @Test
    void getChannelBySlug() {
        given().contentType(ContentType.JSON)
            .body("""
                {"name": "get-by-slug"}
                """)
            .when().post("/api/channels").then().statusCode(201);

        given()
            .when().get("/api/channels/{id}", "get-by-slug")
            .then()
            .statusCode(200)
            .body("name", equalTo("get-by-slug"));
    }

    @Test
    void getChannelNotFoundReturns404() {
        given()
            .when().get("/api/channels/{id}", "nonexistent-channel-xyz")
            .then()
            .statusCode(404)
            .body("error", notNullValue());
    }

    @Test
    void deleteChannelReturns204() {
        String channelId = given().contentType(ContentType.JSON)
            .body("""
                {"name": "delete-me"}
                """)
            .when().post("/api/channels")
            .then().statusCode(201)
            .extract().path("channelId");

        given()
            .queryParam("force", true)
            .when().delete("/api/channels/{id}", channelId)
            .then()
            .statusCode(204);

        given()
            .when().get("/api/channels/{id}", channelId)
            .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelResourceTest -pl runtime`
Expected: FAIL — `ChannelResource` does not exist, all endpoints return 404

- [ ] **Step 3: Write the resource class with CRUD endpoints**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java`:

```java
package io.casehub.qhorus.runtime.api;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.SpaceStore;
import io.casehub.qhorus.api.store.query.ChannelQuery;
import io.casehub.qhorus.runtime.channel.ChannelService;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.*;
import java.util.stream.Collectors;

@Path("/api/channels")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class ChannelResource {

    @Inject
    ChannelService channelService;

    @Inject
    MessageStore messageStore;

    @Inject
    SpaceStore spaceStore;

    // -- CRUD ---------------------------------------------------------------

    @POST
    public Response create(CreateChannelRequest req) {
        try {
            var builder = ChannelCreateRequest.builder(req.name());
            if (req.description() != null) builder.description(req.description());
            if (req.semantic() != null) builder.semantic(ChannelSemantic.valueOf(req.semantic()));
            if (req.barrierContributors() != null) builder.barrierContributors(req.barrierContributors());
            if (req.allowedWriters() != null) builder.allowedWriters(req.allowedWriters());
            if (req.adminInstances() != null) builder.adminInstances(req.adminInstances());
            if (req.reviewerInstances() != null) builder.reviewerInstances(req.reviewerInstances());
            if (req.allowedTypes() != null) builder.allowedTypes(parseTypes(req.allowedTypes()));
            if (req.deniedTypes() != null) builder.deniedTypes(parseTypes(req.deniedTypes()));
            if (req.protocols() != null) builder.protocols(req.protocols());
            if (req.protocolParticipants() != null) builder.protocolParticipants(req.protocolParticipants());
            if (req.spaceId() != null) builder.spaceId(req.spaceId());
            if (req.trackDelivery() != null) builder.trackDelivery(req.trackDelivery());
            if (req.rateLimitPerChannel() != null) builder.rateLimitPerChannel(req.rateLimitPerChannel());
            if (req.rateLimitPerInstance() != null) builder.rateLimitPerInstance(req.rateLimitPerInstance());

            Channel ch = channelService.create(builder.build());
            return Response.status(Response.Status.CREATED)
                    .entity(toResponse(ch))
                    .build();
        } catch (IllegalArgumentException e) {
            return error(400, e.getMessage());
        }
    }

    @GET
    public List<ChannelResponse> list(
            @QueryParam("prefix") String prefix,
            @QueryParam("spaceId") UUID spaceId,
            @QueryParam("paused") Boolean paused) {

        List<Channel> channels;
        if (prefix != null || spaceId != null || paused != null) {
            var qb = ChannelQuery.builder();
            if (prefix != null) qb.namePrefix(prefix);
            if (spaceId != null) qb.spaceId(spaceId);
            if (paused != null) qb.paused(paused);
            channels = channelService.scan(qb.build());
        } else {
            channels = channelService.listAll();
        }
        return toResponseList(channels);
    }

    @GET
    @Path("/{id}")
    public Response getById(@PathParam("id") String id) {
        Channel ch = resolve(id);
        if (ch == null) return error(404, "Channel not found: " + id);
        return Response.ok(toResponse(ch)).build();
    }

    @DELETE
    @Path("/{id}")
    public Response delete(@PathParam("id") String id,
                           @QueryParam("force") @DefaultValue("false") boolean force) {
        Channel ch = resolve(id);
        if (ch == null) return error(404, "Channel not found: " + id);
        try {
            channelService.delete(ch.id(), force);
            return Response.noContent().build();
        } catch (IllegalStateException e) {
            return error(409, e.getMessage());
        }
    }

    // -- Resolution ---------------------------------------------------------

    private Channel resolve(String idOrName) {
        UUID uuid = tryParseUuid(idOrName);
        if (uuid != null) {
            return channelService.findById(uuid).orElse(null);
        }
        return channelService.findByName(idOrName).orElse(null);
    }

    private static UUID tryParseUuid(String s) {
        try { return UUID.fromString(s); }
        catch (IllegalArgumentException e) { return null; }
    }

    // -- Mapping ------------------------------------------------------------

    private ChannelResponse toResponse(Channel ch) {
        long count = messageStore.countByChannel(ch.id());
        String spaceName = null;
        if (ch.spaceId() != null) {
            var spaces = spaceStore.findByIds(List.of(ch.spaceId()));
            if (!spaces.isEmpty()) spaceName = spaces.get(0).name();
        }
        return ChannelResponse.from(ch, count, spaceName);
    }

    private List<ChannelResponse> toResponseList(List<Channel> channels) {
        Map<UUID, String> spaceNames = Map.of();
        var spaceIds = channels.stream()
                .map(Channel::spaceId).filter(Objects::nonNull)
                .distinct().collect(Collectors.toList());
        if (!spaceIds.isEmpty()) {
            spaceNames = spaceStore.findByIds(spaceIds).stream()
                    .collect(Collectors.toMap(
                            io.casehub.qhorus.api.channel.Space::id,
                            io.casehub.qhorus.api.channel.Space::name));
        }
        Map<UUID, String> finalSpaceNames = spaceNames;
        return channels.stream()
                .map(ch -> ChannelResponse.from(ch,
                        messageStore.countByChannel(ch.id()),
                        ch.spaceId() != null ? finalSpaceNames.get(ch.spaceId()) : null))
                .collect(Collectors.toList());
    }

    private static Set<MessageType> parseTypes(Set<String> names) {
        return names.stream().map(MessageType::valueOf).collect(Collectors.toSet());
    }

    private static Response error(int status, String message) {
        return Response.status(status)
                .entity(new ErrorResponse(message))
                .type(MediaType.APPLICATION_JSON)
                .build();
    }

    // -- Request/Response records -------------------------------------------

    public record CreateChannelRequest(
            String name,
            String description,
            String semantic,
            List<String> barrierContributors,
            List<String> allowedWriters,
            List<String> adminInstances,
            List<String> reviewerInstances,
            Set<String> allowedTypes,
            Set<String> deniedTypes,
            List<String> protocols,
            List<String> protocolParticipants,
            UUID spaceId,
            Boolean trackDelivery,
            Integer rateLimitPerChannel,
            Integer rateLimitPerInstance) {}

    public record ErrorResponse(String error) {}
}
```

- [ ] **Step 4: Check `MessageStore.countByChannel` exists**

Verify this method exists on `MessageStore`. If not, check for an equivalent (`count(UUID channelId)` or similar) and adjust the call. The MCP tools use `messageStore.countByChannel()` — verify the exact signature.

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelResourceTest -pl runtime`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java runtime/src/test/java/io/casehub/qhorus/api/ChannelResourceTest.java
git commit -m "feat(#387): ChannelResource with CRUD endpoints (POST, GET, DELETE)"
```

---

### Task 3: Lifecycle and settings sub-resource endpoints

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/api/ChannelResourceTest.java`

**Interfaces:**
- Consumes: `ChannelService.pause(UUID)`, `ChannelService.resume(UUID)`, `ChannelService.setAllowedWriters(UUID, List<String>)`, `ChannelService.setAdminInstances(UUID, List<String>)`, `ChannelService.setReviewerInstances(UUID, List<String>)`, `ChannelService.setTypeConstraints(UUID, Set<MessageType>, Set<MessageType>)`, `ChannelService.setRateLimits(UUID, Integer, Integer)`, `ChannelService.setProtocols(UUID, List<String>)`, `ChannelService.setProtocolParticipants(UUID, List<String>)`, `ChannelService.setTrackDelivery(UUID, Boolean)`
- Produces: Inner records `TypeConstraintsRequest`, `RateLimitsRequest`, `StringListRequest`, `DeliveryTrackingRequest` on `ChannelResource`

- [ ] **Step 1: Write failing tests for pause/resume**

Add to `ChannelResourceTest`:

```java
@Test
void pauseAndResumeChannel() {
    String channelId = given().contentType(ContentType.JSON)
        .body("""
            {"name": "pause-test"}
            """)
        .when().post("/api/channels")
        .then().statusCode(201)
        .body("paused", equalTo(false))
        .extract().path("channelId");

    given()
        .when().post("/api/channels/{id}/pause", channelId)
        .then()
        .statusCode(200)
        .body("paused", equalTo(true));

    given()
        .when().post("/api/channels/{id}/resume", channelId)
        .then()
        .statusCode(200)
        .body("paused", equalTo(false));
}
```

- [ ] **Step 2: Write failing tests for settings sub-resources**

Add to `ChannelResourceTest`:

```java
@Test
void setAllowedWriters() {
    String channelId = given().contentType(ContentType.JSON)
        .body("""
            {"name": "writers-test"}
            """)
        .when().post("/api/channels")
        .then().statusCode(201).extract().path("channelId");

    given().contentType(ContentType.JSON)
        .body("""
            {"values": ["agent-a", "agent-b"]}
            """)
        .when().put("/api/channels/{id}/allowed-writers", channelId)
        .then()
        .statusCode(200)
        .body("allowedWriters", hasItems("agent-a", "agent-b"));
}

@Test
void setTypeConstraints() {
    String channelId = given().contentType(ContentType.JSON)
        .body("""
            {"name": "types-test"}
            """)
        .when().post("/api/channels")
        .then().statusCode(201).extract().path("channelId");

    given().contentType(ContentType.JSON)
        .body("""
            {"allowedTypes": ["QUERY", "RESPONSE"], "deniedTypes": null}
            """)
        .when().put("/api/channels/{id}/type-constraints", channelId)
        .then()
        .statusCode(200)
        .body("allowedTypes", hasItems("QUERY", "RESPONSE"));
}

@Test
void setRateLimits() {
    String channelId = given().contentType(ContentType.JSON)
        .body("""
            {"name": "rates-test"}
            """)
        .when().post("/api/channels")
        .then().statusCode(201).extract().path("channelId");

    given().contentType(ContentType.JSON)
        .body("""
            {"perChannel": 50, "perInstance": 5}
            """)
        .when().put("/api/channels/{id}/rate-limits", channelId)
        .then()
        .statusCode(200)
        .body("rateLimitPerChannel", equalTo(50))
        .body("rateLimitPerInstance", equalTo(5));
}

@Test
void setProtocols() {
    String channelId = given().contentType(ContentType.JSON)
        .body("""
            {"name": "protocols-test"}
            """)
        .when().post("/api/channels")
        .then().statusCode(201).extract().path("channelId");

    given().contentType(ContentType.JSON)
        .body("""
            {"values": ["REQUEST_RESPONSE"]}
            """)
        .when().put("/api/channels/{id}/protocols", channelId)
        .then()
        .statusCode(200)
        .body("protocols", hasItem("REQUEST_RESPONSE"));
}

@Test
void setDeliveryTracking() {
    String channelId = given().contentType(ContentType.JSON)
        .body("""
            {"name": "tracking-test"}
            """)
        .when().post("/api/channels")
        .then().statusCode(201).extract().path("channelId");

    given().contentType(ContentType.JSON)
        .body("""
            {"enabled": true}
            """)
        .when().put("/api/channels/{id}/delivery-tracking", channelId)
        .then()
        .statusCode(200)
        .body("trackDelivery", equalTo(true));
}

@Test
void settingsEndpointReturns404ForUnknownChannel() {
    given().contentType(ContentType.JSON)
        .body("""
            {"values": ["x"]}
            """)
        .when().put("/api/channels/{id}/allowed-writers", "no-such-channel")
        .then()
        .statusCode(404);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelResourceTest -pl runtime`
Expected: FAIL — endpoints return 404

- [ ] **Step 4: Add lifecycle and settings endpoints to ChannelResource**

Add to `ChannelResource`:

```java
// -- Lifecycle --------------------------------------------------------------

@POST
@Path("/{id}/pause")
public Response pause(@PathParam("id") String id) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    Channel updated = channelService.pause(ch.id());
    return Response.ok(toResponse(updated)).build();
}

@POST
@Path("/{id}/resume")
public Response resume(@PathParam("id") String id) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    Channel updated = channelService.resume(ch.id());
    return Response.ok(toResponse(updated)).build();
}

// -- Settings ---------------------------------------------------------------

@PUT
@Path("/{id}/allowed-writers")
public Response setAllowedWriters(@PathParam("id") String id, StringListRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    Channel updated = channelService.setAllowedWriters(ch.id(), req.values() != null ? req.values() : List.of());
    return Response.ok(toResponse(updated)).build();
}

@PUT
@Path("/{id}/admin-instances")
public Response setAdminInstances(@PathParam("id") String id, StringListRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    Channel updated = channelService.setAdminInstances(ch.id(), req.values() != null ? req.values() : List.of());
    return Response.ok(toResponse(updated)).build();
}

@PUT
@Path("/{id}/reviewer-instances")
public Response setReviewerInstances(@PathParam("id") String id, StringListRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    Channel updated = channelService.setReviewerInstances(ch.id(), req.values() != null ? req.values() : List.of());
    return Response.ok(toResponse(updated)).build();
}

@PUT
@Path("/{id}/type-constraints")
public Response setTypeConstraints(@PathParam("id") String id, TypeConstraintsRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    try {
        Set<MessageType> allowed = req.allowedTypes() != null ? parseTypes(req.allowedTypes()) : null;
        Set<MessageType> denied = req.deniedTypes() != null ? parseTypes(req.deniedTypes()) : null;
        Channel updated = channelService.setTypeConstraints(ch.id(), allowed, denied);
        return Response.ok(toResponse(updated)).build();
    } catch (IllegalArgumentException e) {
        return error(400, e.getMessage());
    }
}

@PUT
@Path("/{id}/rate-limits")
public Response setRateLimits(@PathParam("id") String id, RateLimitsRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    Channel updated = channelService.setRateLimits(ch.id(), req.perChannel(), req.perInstance());
    return Response.ok(toResponse(updated)).build();
}

@PUT
@Path("/{id}/protocols")
public Response setProtocols(@PathParam("id") String id, StringListRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    try {
        Channel updated = channelService.setProtocols(ch.id(), req.values() != null ? req.values() : List.of());
        return Response.ok(toResponse(updated)).build();
    } catch (IllegalArgumentException e) {
        return error(400, e.getMessage());
    }
}

@PUT
@Path("/{id}/protocol-participants")
public Response setProtocolParticipants(@PathParam("id") String id, StringListRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    Channel updated = channelService.setProtocolParticipants(ch.id(), req.values() != null ? req.values() : List.of());
    return Response.ok(toResponse(updated)).build();
}

@PUT
@Path("/{id}/delivery-tracking")
public Response setDeliveryTracking(@PathParam("id") String id, DeliveryTrackingRequest req) {
    Channel ch = resolve(id);
    if (ch == null) return error(404, "Channel not found: " + id);
    channelService.setTrackDelivery(ch.id(), req.enabled());
    // setTrackDelivery returns void — re-fetch
    Channel updated = channelService.findById(ch.id()).orElseThrow();
    return Response.ok(toResponse(updated)).build();
}
```

Add inner request records to `ChannelResource`:

```java
public record TypeConstraintsRequest(Set<String> allowedTypes, Set<String> deniedTypes) {}
public record RateLimitsRequest(Integer perChannel, Integer perInstance) {}
public record StringListRequest(List<String> values) {}
public record DeliveryTrackingRequest(Boolean enabled) {}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelResourceTest -pl runtime`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java runtime/src/test/java/io/casehub/qhorus/api/ChannelResourceTest.java
git commit -m "feat(#387): lifecycle and settings sub-resource endpoints"
```

---

### Task 4: Full build verification + CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — document REST API endpoints in the project structure section

**Interfaces:**
- Consumes: everything from Tasks 1-3

- [ ] **Step 1: Run full module build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS — all existing tests still green

- [ ] **Step 2: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: PASS — no compile errors in dependent modules (examples, testing, etc.)

- [ ] **Step 3: Update CLAUDE.md**

Add `ChannelResource.java` to the project structure under `runtime/api/` and document the REST API surface:

```
│       ├── api/
│           ├── ChannelResource.java     — REST API: POST/GET/DELETE /api/channels, lifecycle actions (pause/resume), settings sub-resources (allowed-writers, admin-instances, reviewer-instances, type-constraints, rate-limits, protocols, protocol-participants, delivery-tracking); always-on, no config gate; channel dual-identity resolution (UUID or slug); REST-specific DTOs with typed collections
│           ├── ChannelResponse.java     — REST response record with typed List/Set fields (no CSV); from(Channel, long, String) static factory
```

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#387): document REST API channel CRUD in CLAUDE.md"
```
