# Issue #79 — Activation Build Step + Reactive REST Resources

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the reactive dual-stack toggle — `quarkus.qhorus.reactive.enabled=true` at build time activates `ReactiveQhorusMcpTools` and the two reactive REST resources; default (`false`) leaves the blocking stack unchanged.

**Architecture:** Three changes work together: (1) `quarkus.qhorus.reactive.enabled` added to both runtime `QhorusConfig` and a new build-time `QhorusBuildConfig` in the deployment module; (2) `@IfBuildProperty` / `@UnlessBuildProperty` annotations on the four conflicting beans (`QhorusMcpTools`, `AgentCardResource`, `A2AResource`, `ReactiveQhorusMcpTools`); (3) `QhorusProcessor` gains a `@BuildStep(onlyIf = ReactiveEnabled.class)` that marks reactive beans as unremovable. Non-conflicting reactive services (`ReactiveChannelService`, etc.) need no changes — they are different types and coexist harmlessly. The 666 blocking tests continue to pass because the default build property value is `false`.

**Tech Stack:** Java 21, Quarkus 3.32.2, `@IfBuildProperty` / `@UnlessBuildProperty` (quarkus-arc), `@ConfigRoot(phase = BUILD_TIME)` (deployment module), `UnremovableBeanBuildItem` (quarkus-arc-deployment), SmallRye Mutiny `Uni<T>`.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,deployment -q`

---

## File Map

**New in `deployment/`:**
- `deployment/src/main/java/io/quarkiverse/qhorus/deployment/QhorusBuildConfig.java` — build-time config

**New in `runtime/`:**
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveAgentCardResource.java`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveA2AResource.java`

**Modified:**
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/config/QhorusConfig.java` — add `Reactive reactive()`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java` — add `@UnlessBuildProperty`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — replace `@Alternative` with `@IfBuildProperty`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/AgentCardResource.java` — add `@UnlessBuildProperty`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/A2AResource.java` — add `@UnlessBuildProperty`
- `deployment/src/main/java/io/quarkiverse/qhorus/deployment/QhorusProcessor.java` — add build step

---

## Task 1: Config + build step

**Files:**
- Create: `deployment/src/main/java/io/quarkiverse/qhorus/deployment/QhorusBuildConfig.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/config/QhorusConfig.java`
- Modify: `deployment/src/main/java/io/quarkiverse/qhorus/deployment/QhorusProcessor.java`

- [ ] **Step 1: Verify baseline**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run:" | tail -1
```
Expected: `Tests run: 666, Failures: 0`

- [ ] **Step 2: Create QhorusBuildConfig**

```java
// deployment/src/main/java/io/quarkiverse/qhorus/deployment/QhorusBuildConfig.java
package io.quarkiverse.qhorus.deployment;

import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

/**
 * Build-time configuration for Qhorus. Properties here are fixed at build time
 * and cannot be overridden at runtime.
 */
@ConfigMapping(prefix = "quarkus.qhorus")
@ConfigRoot(phase = ConfigPhase.BUILD_TIME)
public interface QhorusBuildConfig {

    /** Reactive dual-stack settings. */
    Reactive reactive();

    interface Reactive {
        /**
         * When true, activates the reactive dual-stack: ReactiveQhorusMcpTools,
         * ReactiveAgentCardResource, and ReactiveA2AResource replace their blocking
         * counterparts. Default: false (blocking stack active).
         */
        @WithDefault("false")
        boolean enabled();
    }
}
```

- [ ] **Step 3: Add Reactive interface to QhorusConfig**

In `QhorusConfig.java`, add after the existing `Watchdog watchdog()` method:

```java
    /** Reactive dual-stack settings (build-time fixed — read QhorusBuildConfig at build time). */
    Reactive reactive();

    interface Reactive {
        /**
         * When true, the reactive dual-stack is active (set at build time via
         * quarkus.qhorus.reactive.enabled). Runtime reads of this value reflect
         * what was set when the application was compiled.
         */
        @WithDefault("false")
        boolean enabled();
    }
```

- [ ] **Step 4: Add ReactiveEnabled + build step to QhorusProcessor**

Replace the entire `QhorusProcessor.java` with:

```java
package io.quarkiverse.qhorus.deployment;

import java.util.function.BooleanSupplier;

import io.quarkiverse.qhorus.runtime.api.ReactiveAgentCardResource;
import io.quarkiverse.qhorus.runtime.mcp.ReactiveQhorusMcpTools;
import io.quarkus.arc.deployment.UnremovableBeanBuildItem;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

/**
 * Quarkus build-time processor for the Qhorus extension.
 * Registers the "qhorus" feature and, when reactive is enabled, ensures
 * reactive beans are not pruned by Arc's unused-bean removal.
 */
class QhorusProcessor {

    private static final String FEATURE = "qhorus";

    @BuildStep
    FeatureBuildItem feature() {
        return new FeatureBuildItem(FEATURE);
    }

    /**
     * When {@code quarkus.qhorus.reactive.enabled=true}, marks reactive MCP tool and
     * REST resource beans as unremovable so Arc does not prune them.
     * The actual activation is handled by {@code @IfBuildProperty} on each bean class.
     */
    @BuildStep(onlyIf = ReactiveEnabled.class)
    UnremovableBeanBuildItem markReactiveBeans() {
        return UnremovableBeanBuildItem.beanTypes(
                ReactiveQhorusMcpTools.class,
                ReactiveAgentCardResource.class);
    }

    /** Activates when {@code quarkus.qhorus.reactive.enabled=true}. */
    static final class ReactiveEnabled implements BooleanSupplier {

        QhorusBuildConfig config;

        @Override
        public boolean getAsBoolean() {
            return config.reactive().enabled();
        }
    }
}
```

- [ ] **Step 5: Compile deployment module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl deployment -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`. If not, check that `QhorusBuildConfig`, `ReactiveQhorusMcpTools`, and `ReactiveAgentCardResource` are all importable from the deployment module.

- [ ] **Step 6: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run:" | tail -1
```
Expected: `Tests run: 666, Failures: 0`

- [ ] **Step 7: Commit**

```bash
git add deployment/src/main/java/io/quarkiverse/qhorus/deployment/QhorusBuildConfig.java \
        deployment/src/main/java/io/quarkiverse/qhorus/deployment/QhorusProcessor.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/config/QhorusConfig.java
git commit -m "$(cat <<'EOF'
feat(deploy): reactive activation build step + config

QhorusBuildConfig: build-time quarkus.qhorus.reactive.enabled property.
QhorusProcessor: @BuildStep(onlyIf=ReactiveEnabled) marks reactive beans
unremovable when the flag is set. QhorusConfig gains Reactive interface
for runtime awareness of the build-time setting.

Refs #79, Refs #73
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: @IfBuildProperty / @UnlessBuildProperty on conflicting beans

The four beans that would conflict if both stacks are active simultaneously each get a build-property condition. Non-conflicting reactive services (`ReactiveChannelService`, `ReactiveMessageService`, etc.) are different types and need no changes.

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/AgentCardResource.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/A2AResource.java`

- [ ] **Step 1: Add @UnlessBuildProperty to QhorusMcpTools**

In `QhorusMcpTools.java`, add the import and annotation:

Import to add:
```java
import io.quarkus.arc.properties.UnlessBuildProperty;
```

Add annotation directly above `@WrapBusinessError(...)`:
```java
@UnlessBuildProperty(name = "quarkus.qhorus.reactive.enabled", stringValue = "true",
        enableIfMissing = true)
@WrapBusinessError({ IllegalArgumentException.class, IllegalStateException.class })
@ApplicationScoped
public class QhorusMcpTools extends QhorusMcpToolsBase {
```

- [ ] **Step 2: Replace @Alternative with @IfBuildProperty on ReactiveQhorusMcpTools**

In `ReactiveQhorusMcpTools.java`:

Import to add:
```java
import io.quarkus.arc.properties.IfBuildProperty;
```

Import to remove:
```java
import jakarta.enterprise.inject.Alternative;
```

Change class declaration from:
```java
@WrapBusinessError({ IllegalArgumentException.class, IllegalStateException.class })
@Alternative
@ApplicationScoped
public class ReactiveQhorusMcpTools extends QhorusMcpToolsBase {
```
To:
```java
@WrapBusinessError({ IllegalArgumentException.class, IllegalStateException.class })
@IfBuildProperty(name = "quarkus.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class ReactiveQhorusMcpTools extends QhorusMcpToolsBase {
```

- [ ] **Step 3: Add @UnlessBuildProperty to AgentCardResource**

In `AgentCardResource.java`, add the import and annotation:

Import to add:
```java
import io.quarkus.arc.properties.UnlessBuildProperty;
```

Add annotation above `@Path("/.well-known")`:
```java
@UnlessBuildProperty(name = "quarkus.qhorus.reactive.enabled", stringValue = "true",
        enableIfMissing = true)
@Path("/.well-known")
@ApplicationScoped
public class AgentCardResource {
```

- [ ] **Step 4: Add @UnlessBuildProperty to A2AResource**

In `A2AResource.java`, add the import and annotation:

Import to add:
```java
import io.quarkus.arc.properties.UnlessBuildProperty;
```

Add annotation above `@Path("/a2a")`:
```java
@UnlessBuildProperty(name = "quarkus.qhorus.reactive.enabled", stringValue = "true",
        enableIfMissing = true)
@Path("/a2a")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class A2AResource {
```

- [ ] **Step 5: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run:" | tail -1
```
Expected: `Tests run: 666, Failures: 0` — the default build property is `false` so `QhorusMcpTools`, `AgentCardResource`, `A2AResource` all remain active (via `enableIfMissing = true`).

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/AgentCardResource.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/A2AResource.java
git commit -m "$(cat <<'EOF'
feat(api,mcp): @IfBuildProperty/@UnlessBuildProperty on conflicting beans

QhorusMcpTools + AgentCardResource + A2AResource: inactive when
quarkus.qhorus.reactive.enabled=true (enableIfMissing=true keeps them
active in all tests and default deployments).
ReactiveQhorusMcpTools: @Alternative replaced with @IfBuildProperty —
only registered when reactive.enabled=true.

Refs #79, Refs #73
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: ReactiveAgentCardResource + ReactiveA2AResource + verify + commit

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveAgentCardResource.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveA2AResource.java`

- [ ] **Step 1: Create ReactiveAgentCardResource**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveAgentCardResource.java
package io.quarkiverse.qhorus.runtime.api;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.qhorus.runtime.config.QhorusConfig;
import io.quarkus.arc.properties.IfBuildProperty;
import io.smallrye.mutiny.Uni;

/**
 * Reactive mirror of {@link AgentCardResource} — active only when
 * {@code quarkus.qhorus.reactive.enabled=true}.
 * Returns {@code Uni<Response>} so the Vert.x event loop is not blocked.
 */
@IfBuildProperty(name = "quarkus.qhorus.reactive.enabled", stringValue = "true")
@Path("/.well-known")
@ApplicationScoped
public class ReactiveAgentCardResource {

    @Inject
    QhorusConfig config;

    @GET
    @Path("/agent-card.json")
    @Produces(MediaType.APPLICATION_JSON)
    public Uni<Response> getAgentCard() {
        QhorusConfig.AgentCard cfg = config.agentCard();
        AgentCardResource.AgentCard card = new AgentCardResource.AgentCard(
                cfg.name(),
                cfg.description(),
                cfg.url().orElse(""),
                cfg.version(),
                buildSkills(),
                new AgentCardResource.AgentCapabilities(true, true));
        return Uni.createFrom().item(Response.ok(card).build());
    }

    private List<AgentCardResource.AgentSkill> buildSkills() {
        return List.of(
                new AgentCardResource.AgentSkill(
                        "channel-messaging",
                        "Channel Messaging",
                        "Send and receive typed messages on named channels with declared semantics"
                                + " (APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE)"),
                new AgentCardResource.AgentSkill(
                        "shared-data",
                        "Shared Data Store",
                        "Store and retrieve large artefacts by key with UUID references,"
                                + " claim/release lifecycle, and chunked streaming"),
                new AgentCardResource.AgentSkill(
                        "presence",
                        "Agent Presence",
                        "Register agents with capability tags and discover online peers"
                                + " by capability tag or role broadcast"),
                new AgentCardResource.AgentSkill(
                        "wait-for-reply",
                        "Correlation-based Wait",
                        "Wait for a response with a specific correlation ID —"
                                + " safe under concurrent requests via UUID-keyed PendingReply"));
    }
}
```

- [ ] **Step 2: Create ReactiveA2AResource**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveA2AResource.java
package io.quarkiverse.qhorus.runtime.api;

import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.mcp.server.ToolCallException;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.config.QhorusConfig;
import io.quarkiverse.qhorus.runtime.mcp.ReactiveQhorusMcpTools;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageService;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkus.arc.properties.IfBuildProperty;
import io.smallrye.common.annotation.Blocking;
import io.smallrye.mutiny.Uni;

/**
 * Reactive mirror of {@link A2AResource} — active only when
 * {@code quarkus.qhorus.reactive.enabled=true}.
 *
 * <p>
 * {@code POST /a2a/message:send} uses {@link ReactiveQhorusMcpTools} and returns
 * {@code Uni<Response>}. {@code GET /a2a/tasks/{id}} uses {@code @Blocking} with
 * the blocking message / channel services because {@code findAllByCorrelationId}
 * is not yet exposed via the reactive service layer.
 *
 * @see A2AResource
 */
@IfBuildProperty(name = "quarkus.qhorus.reactive.enabled", stringValue = "true")
@Path("/a2a")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class ReactiveA2AResource {

    private static final Response A2A_DISABLED = Response
            .status(Response.Status.NOT_IMPLEMENTED)
            .entity("{\"error\":\"A2A endpoint is disabled. Set quarkus.qhorus.a2a.enabled=true to activate.\"}")
            .type(MediaType.APPLICATION_JSON)
            .build();

    @Inject
    QhorusConfig config;

    @Inject
    ReactiveQhorusMcpTools tools;

    @Inject
    MessageService messageService;

    @Inject
    ChannelService channelService;

    @POST
    @Path("/message:send")
    @Consumes(MediaType.APPLICATION_JSON)
    public Uni<Response> sendMessage(A2AResource.SendMessageRequest request) {
        if (!config.a2a().enabled()) {
            return Uni.createFrom().item(A2A_DISABLED);
        }

        if (request == null || request.message() == null) {
            return Uni.createFrom().item(error400("message is required"));
        }
        A2AResource.A2AMessage msg = request.message();

        if (msg.contextId() == null || msg.contextId().isBlank()) {
            return Uni.createFrom().item(error400("message.contextId (channel name) is required"));
        }
        if (msg.parts() == null || msg.parts().isEmpty()) {
            return Uni.createFrom().item(error400("message.parts must contain at least one text part"));
        }
        String text = msg.parts().stream()
                .filter(p -> "text".equals(p.kind()) && p.text() != null)
                .map(A2AResource.A2APart::text)
                .findFirst()
                .orElse(null);
        if (text == null) {
            return Uni.createFrom().item(
                    error400("message.parts must contain at least one text part with kind=text"));
        }

        String correlationId = (msg.taskId() != null && !msg.taskId().isBlank())
                ? msg.taskId()
                : UUID.randomUUID().toString();
        String sender = (msg.role() != null && !msg.role().isBlank()) ? msg.role() : "agent";

        final String finalCorrelationId = correlationId;
        final String finalContextId = msg.contextId();

        return tools.sendMessage(finalContextId, sender, "request", text,
                        finalCorrelationId, null, null, null)
                .map(ignored -> {
                    A2AResource.Task task = new A2AResource.Task(
                            finalCorrelationId, finalContextId,
                            new A2AResource.TaskStatus("submitted"), null);
                    return Response.ok(new A2AResource.SendMessageResponse(task)).build();
                })
                .onFailure(IllegalArgumentException.class)
                .recoverWithItem(e -> error400(e.getMessage()))
                .onFailure(ToolCallException.class)
                .recoverWithItem(e -> {
                    String m = e.getCause() != null ? e.getCause().getMessage() : e.getMessage();
                    return error400(m);
                });
    }

    @GET
    @Path("/tasks/{id}")
    @Blocking
    public Uni<Response> getTask(@PathParam("id") String taskId) {
        if (!config.a2a().enabled()) {
            return Uni.createFrom().item(A2A_DISABLED);
        }

        return Uni.createFrom().item(() -> {
            List<Message> messages = messageService.findAllByCorrelationId(taskId);
            if (messages.isEmpty()) {
                return Response.status(Response.Status.NOT_FOUND)
                        .entity("{\"error\":\"Task not found: " + taskId + "\"}")
                        .type(MediaType.APPLICATION_JSON)
                        .build();
            }

            Channel channel = channelService.findById(messages.get(0).channelId)
                    .orElseThrow(() -> new IllegalStateException(
                            "Channel not found for task " + taskId));

            String state = deriveState(messages);

            List<A2AResource.A2AMessage> history = messages.stream()
                    .map(m -> new A2AResource.A2AMessage(
                            m.sender,
                            m.content != null
                                    ? List.of(new A2AResource.A2APart("text", m.content))
                                    : List.of(),
                            null,
                            m.correlationId,
                            channel.name))
                    .toList();

            return Response.ok(
                    new A2AResource.Task(taskId, channel.name, new A2AResource.TaskStatus(state),
                            history))
                    .build();
        });
    }

    private static String deriveState(List<Message> messages) {
        for (Message m : messages) {
            if (m.messageType == MessageType.RESPONSE || m.messageType == MessageType.DONE) {
                return "completed";
            }
        }
        for (Message m : messages) {
            if (m.messageType == MessageType.STATUS) {
                return "working";
            }
        }
        return "submitted";
    }

    private static Response error400(String message) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity("{\"error\":\"" + message + "\"}")
                .type(MediaType.APPLICATION_JSON)
                .build();
    }
}
```

- [ ] **Step 3: Compile and run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run:" | tail -1
```
Expected: `Tests run: 666, Failures: 0`

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveAgentCardResource.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/api/ReactiveA2AResource.java
git commit -m "$(cat <<'EOF'
feat(api): ReactiveAgentCardResource + ReactiveA2AResource

Both active only when quarkus.qhorus.reactive.enabled=true (@IfBuildProperty).
AgentCard: pure Uni<Response> returning the same card payload.
A2A sendMessage: reactive chain via ReactiveQhorusMcpTools.sendMessage().
A2A getTask: @Blocking + blocking services (findAllByCorrelationId not yet
in reactive service layer).

Closes #79, Refs #73
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- ✅ `quarkus.qhorus.reactive.enabled=true` activates full reactive stack — `@IfBuildProperty` on `ReactiveQhorusMcpTools`, `ReactiveAgentCardResource`, `ReactiveA2AResource`; `@UnlessBuildProperty` on `QhorusMcpTools`, `AgentCardResource`, `A2AResource`
- ✅ Default `false` leaves blocking stack active — `enableIfMissing = true` on `@UnlessBuildProperty`, and `@IfBuildProperty` only registers reactive beans when property = "true"
- ✅ `ReactiveAgentCardResource` implemented — Task 3
- ✅ `ReactiveA2AResource` implemented — Task 3
- ✅ A2A optional module guard preserved — `if (!config.a2a().enabled()) return A2A_DISABLED`
- ✅ `@BuildStep(onlyIf = ReactiveEnabled.class)` pattern — Task 1, `QhorusProcessor`
- ✅ Config property in `QhorusConfig` as build-time fixed — `QhorusBuildConfig` in deployment + `Reactive reactive()` in runtime `QhorusConfig`

**Placeholder scan:** None.

**Type consistency:**
- `ReactiveA2AResource.sendMessage` calls `tools.sendMessage(...)` returning `Uni<ReactiveQhorusMcpToolsBase.MessageResult>` — the return value is discarded (`.map(ignored -> ...)`) so the generic type doesn't matter
- `ReactiveAgentCardResource.buildSkills()` returns `List<AgentCardResource.AgentSkill>` — correct, reusing parent class records
- `A2AResource.SendMessageRequest`, `A2AResource.A2AMessage`, etc. — reused from `A2AResource` (no duplication)

**What is NOT in this plan (deferred):**
- Reactive JPA integration tests for the reactive stack — requires Docker/PostgreSQL; Issue #80
- `findAllByCorrelationId` in `ReactiveMessageService` — deferred; `ReactiveA2AResource.getTask()` uses `@Blocking` + blocking service as a documented stopgap
