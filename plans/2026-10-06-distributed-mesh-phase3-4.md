# Distributed Mesh Phase 3-4 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #475 — epic: distributed qhorus mesh — standalone service with clustering
**Issue group:** #475

**Goal:** Add REST API endpoints for agent and message operations (Phase 3),
then wire the mesh module for PostgreSQL production deployment with cluster
module integration (Phase 4). This enables Level 2 (dedicated server) and
Level 3 (relays) from the topology ladder.

**Architecture:** REST resources follow the existing pattern: JAX-RS
`@Path` resource delegates to a `*Core` POJO in `runtime-core`, which
calls service-layer beans. The mesh module (`MeshApp`) gains PostgreSQL
config with env-var substitution, cluster module dependency, and a
Dockerfile for container deployment.

**Tech Stack:** Java 21, Quarkus 3.32.2, JAX-RS (quarkus-rest-jackson),
PostgreSQL (quarkus-jdbc-postgresql), SSE (quarkus-rest), Docker

## Global Constraints

- Java 21 source level (running on Java 26 JVM)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- REST resources in `runtime/src/main/java/io/casehub/qhorus/runtime/api/`
- Core POJOs in `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/`
- Resources delegate to `*Core` classes (not directly to services)
- Commits reference #475: `Refs #475`
- Tests: `@QuarkusTest` for REST integration, CDI-free for core logic

---

## Batch 1: InstanceResource — agent registration via REST

After this batch: agents can register, deregister, and discover peers
via REST endpoints. Level 2 (dedicated server) can accept agent
connections without MCP.

### Task 1: InstanceCore + InstanceResource

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/InstanceCore.java`
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/RegisterInstanceRequest.java`
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/InstanceResponse.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/InstanceResource.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/api/InstanceResourceTest.java`

**Interfaces:**
- Consumes: `InstanceService.register()`, `InstanceService.deregister()`, `InstanceStore.scan()`, `InstanceStore.findByInstanceId()` (from qhorus runtime)
- Produces:
  - `POST /api/instances` — register agent, returns `InstanceResponse`
  - `DELETE /api/instances/{instanceId}` — deregister agent
  - `GET /api/instances` — list agents, optional `?capability=` filter
  - `GET /api/instances/{instanceId}` — get single agent
  - `InstanceCore` — POJO delegate with `register()`, `deregister()`, `list()`, `get()`

- [ ] **Step 1: Write RegisterInstanceRequest and InstanceResponse records**

Create `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/RegisterInstanceRequest.java`:

```java
package io.casehub.qhorus.runtime.api.core;

import java.util.List;
import java.util.Map;

public record RegisterInstanceRequest(
        String instanceId,
        String description,
        List<String> capabilities,
        Map<String, String> metadata) {}
```

Create `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/InstanceResponse.java`:

```java
package io.casehub.qhorus.runtime.api.core;

import io.casehub.qhorus.api.instance.Instance;

import java.time.Instant;
import java.util.List;
import java.util.Map;

public record InstanceResponse(
        String instanceId,
        String description,
        String status,
        Instant lastSeen,
        List<String> capabilities,
        Map<String, String> metadata) {

    public static InstanceResponse from(Instance inst, List<String> capabilities) {
        return new InstanceResponse(
                inst.instanceId(), inst.description(), inst.status(),
                inst.lastSeen(), capabilities, inst.metadata());
    }
}
```

- [ ] **Step 2: Write InstanceCore**

Create `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/InstanceCore.java`:

```java
package io.casehub.qhorus.runtime.api.core;

import io.casehub.qhorus.api.instance.Instance;
import io.casehub.qhorus.api.store.InstanceStore;
import io.casehub.qhorus.api.store.query.InstanceQuery;
import io.casehub.qhorus.runtime.instance.InstanceService;

import java.util.List;
import java.util.NoSuchElementException;

public class InstanceCore {

    private final InstanceService instanceService;
    private final InstanceStore instanceStore;

    public InstanceCore(InstanceService instanceService, InstanceStore instanceStore) {
        this.instanceService = instanceService;
        this.instanceStore = instanceStore;
    }

    public InstanceResponse register(RegisterInstanceRequest req) {
        Instance inst = instanceService.register(
                req.instanceId(), req.description(),
                req.capabilities() != null ? req.capabilities() : List.of(),
                null, false, req.metadata());
        List<String> caps = instanceStore.findCapabilities(inst.id());
        return InstanceResponse.from(inst, caps);
    }

    public void deregister(String instanceId) {
        instanceService.deregister(instanceId);
    }

    public List<InstanceResponse> list(String capability) {
        List<Instance> instances;
        if (capability != null && !capability.isBlank()) {
            instances = instanceStore.scan(InstanceQuery.byCapability(capability));
        } else {
            instances = instanceStore.scan(InstanceQuery.all());
        }
        return instances.stream()
                .map(inst -> InstanceResponse.from(inst,
                        instanceStore.findCapabilities(inst.id())))
                .toList();
    }

    public InstanceResponse get(String instanceId) {
        Instance inst = instanceStore.findByInstanceId(instanceId)
                .orElseThrow(() -> new NoSuchElementException(
                        "Instance not found: " + instanceId));
        List<String> caps = instanceStore.findCapabilities(inst.id());
        return InstanceResponse.from(inst, caps);
    }
}
```

- [ ] **Step 3: Write InstanceResource**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/api/InstanceResource.java`:

```java
package io.casehub.qhorus.runtime.api;

import io.casehub.qhorus.runtime.api.core.ErrorResponse;
import io.casehub.qhorus.runtime.api.core.InstanceCore;
import io.casehub.qhorus.runtime.api.core.InstanceResponse;
import io.casehub.qhorus.runtime.api.core.RegisterInstanceRequest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.DELETE;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.List;
import java.util.NoSuchElementException;

@Path("/api/instances")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class InstanceResource {

    @Inject InstanceCore core;

    @POST
    public Response register(RegisterInstanceRequest request) {
        try {
            InstanceResponse resp = core.register(request);
            return Response.status(Response.Status.CREATED).entity(resp).build();
        } catch (IllegalArgumentException e) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(new ErrorResponse(e.getMessage())).build();
        }
    }

    @DELETE
    @Path("/{instanceId}")
    public Response deregister(@PathParam("instanceId") String instanceId) {
        try {
            core.deregister(instanceId);
            return Response.noContent().build();
        } catch (NoSuchElementException e) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(new ErrorResponse(e.getMessage())).build();
        }
    }

    @GET
    public List<InstanceResponse> list(@QueryParam("capability") String capability) {
        return core.list(capability);
    }

    @GET
    @Path("/{instanceId}")
    public Response get(@PathParam("instanceId") String instanceId) {
        try {
            return Response.ok(core.get(instanceId)).build();
        } catch (NoSuchElementException e) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(new ErrorResponse(e.getMessage())).build();
        }
    }
}
```

- [ ] **Step 4: Write integration test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/api/InstanceResourceTest.java`:

```java
package io.casehub.qhorus.runtime.api;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class InstanceResourceTest {

    @Test
    void registerAndGetInstance() {
        String body = """
                {"instanceId": "test-agent-1", "description": "Test agent",
                 "capabilities": ["summarise", "translate"]}
                """;

        given().contentType(ContentType.JSON).body(body)
                .when().post("/api/instances")
                .then().statusCode(201)
                .body("instanceId", equalTo("test-agent-1"))
                .body("description", equalTo("Test agent"))
                .body("capabilities", hasItems("summarise", "translate"));

        given().when().get("/api/instances/test-agent-1")
                .then().statusCode(200)
                .body("instanceId", equalTo("test-agent-1"));
    }

    @Test
    void listInstancesWithCapabilityFilter() {
        String agent1 = """
                {"instanceId": "filter-agent-1", "description": "Agent 1",
                 "capabilities": ["summarise"]}
                """;
        String agent2 = """
                {"instanceId": "filter-agent-2", "description": "Agent 2",
                 "capabilities": ["translate"]}
                """;

        given().contentType(ContentType.JSON).body(agent1)
                .when().post("/api/instances").then().statusCode(201);
        given().contentType(ContentType.JSON).body(agent2)
                .when().post("/api/instances").then().statusCode(201);

        given().queryParam("capability", "summarise")
                .when().get("/api/instances")
                .then().statusCode(200)
                .body("size()", greaterThanOrEqualTo(1))
                .body("instanceId", hasItem("filter-agent-1"));
    }

    @Test
    void deregisterInstance() {
        String body = """
                {"instanceId": "deregister-agent", "description": "To be removed",
                 "capabilities": []}
                """;
        given().contentType(ContentType.JSON).body(body)
                .when().post("/api/instances").then().statusCode(201);

        given().when().delete("/api/instances/deregister-agent")
                .then().statusCode(204);

        given().when().get("/api/instances/deregister-agent")
                .then().statusCode(404);
    }

    @Test
    void getNonExistentInstanceReturns404() {
        given().when().get("/api/instances/does-not-exist")
                .then().statusCode(404);
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=InstanceResourceTest`
Expected: All 4 tests PASS

- [ ] **Step 6: Commit**

```bash
git add runtime-core/ runtime/
git commit -m "feat(#475): add InstanceResource — agent registration via REST

POST /api/instances, GET /api/instances, GET /api/instances/{id},
DELETE /api/instances/{id}. Follows the Core+Resource delegate pattern.

Refs #475"
```

---

## Batch 2: Message listing — read messages via REST

After this batch: agents can read channel messages with pagination via
`GET /api/channels/{id}/messages`. Combined with the existing
`POST /api/channels/{id}/messages`, the full message read/write cycle
works over REST.

### Task 2: Message listing endpoint on ChannelResource

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/ChannelCore.java` — add `listMessages()` method
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java` — add `GET /{id}/messages` endpoint
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/MessageResponse.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/api/MessageListingTest.java`

**Interfaces:**
- Consumes: `MessageStore.scan(MessageQuery)`, `ChannelService.findByName()` / `findById()` (existing)
- Produces:
  - `GET /api/channels/{id}/messages?afterId=&limit=&topic=&type=` — paginated message list
  - `MessageResponse` — REST DTO for messages
  - `ChannelCore.listMessages()` — delegate method

- [ ] **Step 1: Write MessageResponse record**

Create `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/MessageResponse.java`:

```java
package io.casehub.qhorus.runtime.api.core;

import com.fasterxml.jackson.annotation.JsonInclude;
import io.casehub.qhorus.api.message.ArtefactRef;
import io.casehub.qhorus.api.message.Message;

import java.time.Instant;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record MessageResponse(
        Long id,
        String sender,
        String type,
        String content,
        String payload,
        String correlationId,
        Long inReplyTo,
        String target,
        String topic,
        List<ArtefactRef> artefactRefs,
        Instant createdAt) {

    public static MessageResponse from(Message msg) {
        return new MessageResponse(
                msg.id(), msg.sender(), msg.messageType().name(),
                msg.content(), msg.payload(), msg.correlationId(),
                msg.inReplyTo(), msg.target(), msg.topic(),
                msg.artefactRefs(), msg.createdAt());
    }
}
```

- [ ] **Step 2: Add listMessages to ChannelCore**

Add method to `ChannelCore.java`:

```java
public List<MessageResponse> listMessages(String channelRef, Long afterId,
                                           Integer limit, String topic, String type) {
    Channel ch = resolveChannel(channelRef);
    MessageQuery.Builder qb = MessageQuery.builder().channelId(ch.id());
    if (afterId != null) qb.afterId(afterId);
    if (limit != null) qb.limit(limit); else qb.limit(50);
    if (topic != null && !topic.isBlank()) qb.topic(topic);
    if (type != null && !type.isBlank()) qb.excludeType(MessageType.valueOf(type.toUpperCase()));
    return messageStore.scan(qb.build()).stream()
            .map(MessageResponse::from)
            .toList();
}
```

Note: `resolveChannel()` already exists in `ChannelCore` — dual-identity resolution (UUID or slug).

- [ ] **Step 3: Add GET endpoint to ChannelResource**

Add to `ChannelResource.java` in the Messages section:

```java
@GET
@Path("/{id}/messages")
public List<io.casehub.qhorus.runtime.api.core.MessageResponse> listMessages(
        @PathParam("id") String id,
        @QueryParam("afterId") Long afterId,
        @QueryParam("limit") @DefaultValue("50") Integer limit,
        @QueryParam("topic") String topic,
        @QueryParam("type") String type) {
    return core.listMessages(id, afterId, limit, topic, type);
}
```

- [ ] **Step 4: Write integration test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/api/MessageListingTest.java`:

```java
package io.casehub.qhorus.runtime.api;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class MessageListingTest {

    @Test
    void postAndListMessages() {
        // Create channel
        given().contentType(ContentType.JSON)
                .body("{\"name\": \"msg-list-test\", \"semantic\": \"APPEND\"}")
                .when().post("/api/channels")
                .then().statusCode(201);

        // Post a message
        given().contentType(ContentType.JSON)
                .body("{\"sender\": \"agent-1\", \"type\": \"STATUS\", \"actorType\": \"AGENT\", \"content\": \"hello\"}")
                .when().post("/api/channels/msg-list-test/messages")
                .then().statusCode(200);

        // List messages
        given().when().get("/api/channels/msg-list-test/messages")
                .then().statusCode(200)
                .body("size()", greaterThanOrEqualTo(1))
                .body("[0].sender", equalTo("agent-1"))
                .body("[0].content", equalTo("hello"));
    }

    @Test
    void listMessagesWithPagination() {
        given().contentType(ContentType.JSON)
                .body("{\"name\": \"msg-page-test\", \"semantic\": \"APPEND\"}")
                .when().post("/api/channels").then().statusCode(201);

        for (int i = 0; i < 3; i++) {
            given().contentType(ContentType.JSON)
                    .body("{\"sender\": \"agent-1\", \"type\": \"STATUS\", \"actorType\": \"AGENT\", \"content\": \"msg-" + i + "\"}")
                    .when().post("/api/channels/msg-page-test/messages")
                    .then().statusCode(200);
        }

        // Get first page (limit 2)
        var firstPage = given().queryParam("limit", 2)
                .when().get("/api/channels/msg-page-test/messages")
                .then().statusCode(200)
                .body("size()", equalTo(2))
                .extract().jsonPath();

        // Get second page using afterId
        Long lastId = firstPage.getLong("[1].id");
        given().queryParam("afterId", lastId)
                .when().get("/api/channels/msg-page-test/messages")
                .then().statusCode(200)
                .body("size()", greaterThanOrEqualTo(1));
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageListingTest`
Expected: All 2 tests PASS

- [ ] **Step 6: Commit**

```bash
git add runtime-core/ runtime/
git commit -m "feat(#475): add GET /api/channels/{id}/messages — paginated message listing

afterId pagination, limit, topic and type filtering. Completes the
REST message read/write cycle alongside the existing POST endpoint.

Refs #475"
```

---

## Batch 3: Mesh wiring — PostgreSQL, cluster module, Dockerfile

After this batch: the mesh module runs as a standalone container with
PostgreSQL, cluster module on the classpath, and production-ready
configuration via environment variables. Supports Level 2 (dedicated
server) and Level 3 (relays) from the topology ladder.

### Task 3: Mesh module PostgreSQL + cluster dependencies

**Files:**
- Modify: `mesh/pom.xml` — add PostgreSQL driver, cluster module, postgres-broadcaster dependencies
- Modify: `mesh/src/main/resources/application.properties` — PostgreSQL config with env-var substitution, relay config
- Test: Verify `mvn clean install` compiles and mesh module starts

**Interfaces:**
- Consumes: `casehub-qhorus-cluster` module, `casehub-qhorus-postgres-broadcaster` module
- Produces: Production-ready mesh configuration that reads from env vars

- [ ] **Step 1: Update mesh pom.xml**

Add to `mesh/pom.xml` dependencies (after `casehub-qhorus-agent-bridge`):

```xml
    <!-- Cluster module — relay infrastructure + optional routing -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-cluster</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- Cross-node delivery via PostgreSQL LISTEN/NOTIFY -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-postgres-broadcaster</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- PostgreSQL driver — production database -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-postgresql</artifactId>
    </dependency>

    <!-- Reactive PG for postgres-broadcaster LISTEN/NOTIFY -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-reactive-pg-client</artifactId>
    </dependency>
```

Keep `quarkus-jdbc-h2` for the `dev` profile (local development fallback).

- [ ] **Step 2: Rewrite mesh application.properties**

Replace `mesh/src/main/resources/application.properties`:

```properties
# ── Mesh Relay Server ──────────────────────────────────────
quarkus.http.port=${QHORUS_HTTP_PORT:9741}

# ── PostgreSQL (production) ────────────────────────────────
%prod.quarkus.datasource.qhorus.db-kind=postgresql
%prod.quarkus.datasource.qhorus.jdbc.url=jdbc:postgresql://${QHORUS_DB_HOST:localhost}:${QHORUS_DB_PORT:5432}/${QHORUS_DB_NAME:qhorus}
%prod.quarkus.datasource.qhorus.username=${QHORUS_DB_USER:qhorus}
%prod.quarkus.datasource.qhorus.password=${QHORUS_DB_PASSWORD:}
%prod.quarkus.datasource.qhorus.reactive.url=postgresql://${QHORUS_DB_HOST:localhost}:${QHORUS_DB_PORT:5432}/${QHORUS_DB_NAME:qhorus}

%prod.quarkus.datasource.db-kind=postgresql
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://${QHORUS_DB_HOST:localhost}:${QHORUS_DB_PORT:5432}/${QHORUS_DB_NAME:qhorus}
%prod.quarkus.datasource.username=${QHORUS_DB_USER:qhorus}
%prod.quarkus.datasource.password=${QHORUS_DB_PASSWORD:}

%prod.quarkus.flyway.qhorus.migrate-at-start=true
%prod.quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration

# ── H2 (dev mode) ─────────────────────────────────────────
%dev.quarkus.datasource.qhorus.db-kind=h2
%dev.quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:${user.home}/.qhorus/mesh;AUTO_SERVER=TRUE
%dev.quarkus.datasource.qhorus.username=sa
%dev.quarkus.datasource.qhorus.password=

%dev.quarkus.datasource.db-kind=h2
%dev.quarkus.datasource.jdbc.url=jdbc:h2:file:${user.home}/.qhorus/mesh;AUTO_SERVER=TRUE
%dev.quarkus.datasource.username=sa
%dev.quarkus.datasource.password=

# ── Hibernate ──────────────────────────────────────────────
quarkus.hibernate-orm.datasource=qhorus
quarkus.hibernate-orm.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime,io.casehub.ledger.jpa
%prod.quarkus.hibernate-orm.database.generation=none
%dev.quarkus.hibernate-orm.database.generation=drop-and-create

# ── Relay config (Level 3+) ────────────────────────────────
casehub.qhorus.relay.enabled=${CASEHUB_QHORUS_RELAY_ENABLED:false}
casehub.qhorus.relay.peers=${CASEHUB_QHORUS_RELAY_PEERS:}
casehub.qhorus.relay.node-id=${CASEHUB_QHORUS_RELAY_NODE_ID:}
casehub.qhorus.relay.routing=${CASEHUB_QHORUS_RELAY_ROUTING:none}

# ── Stale instance cleanup ─────────────────────────────────
casehub.qhorus.cleanup.stale-instance-seconds=300
```

- [ ] **Step 3: Verify full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile, mesh module starts in dev mode

- [ ] **Step 4: Commit**

```bash
git add mesh/
git commit -m "feat(#475): wire mesh module for PostgreSQL, cluster, and postgres-broadcaster

Production PostgreSQL config via env vars, H2 fallback for dev mode.
Cluster module and postgres-broadcaster on classpath — relay config
maps to CASEHUB_QHORUS_RELAY_* env vars. Flyway migration at startup.

Refs #475"
```

### Task 4: Dockerfile

**Files:**
- Create: `mesh/src/main/docker/Dockerfile.jvm`

**Interfaces:**
- Consumes: mesh module JAR from `mvn package`
- Produces: Container image `casehub-qhorus-mesh:latest`

- [ ] **Step 1: Create Dockerfile**

Create `mesh/src/main/docker/Dockerfile.jvm`:

```dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-21-runtime:1.20

ENV LANGUAGE='en_US:en'

COPY --chown=185 target/quarkus-app/lib/ /deployments/lib/
COPY --chown=185 target/quarkus-app/*.jar /deployments/
COPY --chown=185 target/quarkus-app/app/ /deployments/app/
COPY --chown=185 target/quarkus-app/quarkus/ /deployments/quarkus/

EXPOSE 9741

ENV JAVA_OPTS_APPEND="-Dquarkus.http.host=0.0.0.0"
ENV JAVA_APP_JAR="/deployments/quarkus-run.jar"

ENTRYPOINT [ "/opt/jboss/container/java/run/run-java.sh" ]
```

- [ ] **Step 2: Verify Docker build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -pl mesh -DskipTests`
Then: `docker build -f mesh/src/main/docker/Dockerfile.jvm -t casehub-qhorus-mesh:latest mesh/`
Expected: Image builds successfully

- [ ] **Step 3: Commit**

```bash
git add mesh/src/main/docker/
git commit -m "feat(#475): add Dockerfile for mesh relay container image

JVM-mode container using UBI9 OpenJDK 21 base. Exposes port 9741.
All config via environment variables.

Refs #475"
```

---

## References

- [2026-10-06-distributed-mesh-consolidated.md] — consolidated design spec (§3, §9, §11)
- [decisions.md] — D18-D24 topology and HA decisions
- [runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java] — existing REST resource pattern
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/ChannelCore.java] — Core delegate pattern
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/MessagePostRequest.java] — existing request DTO
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java] — agent registration service
- [mesh/src/main/java/io/casehub/qhorus/mesh/MeshService.java] — existing MCP mesh tools
- [mesh/pom.xml] — current mesh module dependencies
- [mesh/src/main/resources/application.properties] — current mesh config
- [GitHub #475] — epic: distributed qhorus mesh
