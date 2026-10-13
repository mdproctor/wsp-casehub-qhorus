# YAML Channel Configuration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #464 — feat: channel configuration in app.yaml — YAML-declared channels
**Issue group:** #464

**Goal:** Allow channels to be declared in `application.yaml` under `casehub.qhorus.channels` and automatically created/drift-detected at startup.

**Architecture:** SmallRye `@ConfigMapping` with `Map<String, ChannelConfig>` in `QhorusConfig`. A `ChannelConfigBootstrap` bean observes `StartupEvent`, creates missing channels via `findOrCreate()`, and logs drift warnings for existing channels whose state diverges from the declared config.

**Tech Stack:** Java 21, Quarkus 3.32.2, SmallRye Config, CDI

## Global Constraints

- All fields except `semantic` are `Optional<>` — channels can be declared with just a name (map key) and defaults
- `allowedTypes`/`deniedTypes` use `List<MessageType>` in config (SmallRye `Set<T>` unreliable) — converted to `Set` at bootstrap boundary
- `space` is a name string resolved to UUID via `SpaceService.findByName()` — not a UUID in YAML
- Per-channel try-catch in bootstrap follows PP-20260522-c3a8c1 (startup exception isolation)
- Spring counterpart `QhorusConfigProperties` must be updated for every config interface change
- `ChannelConfigBootstrap` lives in `runtime/` (CDI annotations), not `runtime-core/` (POJOs)

---

## Batch 1: Config interface and bootstrap with creation

### Task 1: Add ChannelConfig to QhorusConfig and Spring counterpart

**Repo:** `casehub-qhorus`

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java`
- Modify: `runtime-spring/src/main/java/io/casehub/qhorus/runtime/spring/config/QhorusConfigProperties.java`

**Interfaces:**
- Consumes: `ChannelSemantic` (existing enum), `MessageType` (existing enum), `EnforcementMode` (existing enum)
- Produces: `QhorusConfig.ChannelConfig` interface, `QhorusConfig.ChannelConfig.ConnectorConfig` interface, `QhorusConfigProperties.ChannelConfigProps` class, `QhorusConfigProperties.ConnectorConfigProps` class

- [ ] **Step 1: Add ChannelConfig and ConnectorConfig interfaces to QhorusConfig**

Add after the `Instance` interface block (line ~31) in `QhorusConfig.java`. Use `ide_insert_member` to add `channels()` method and the two inner interfaces:

```java
Map<String, ChannelConfig> channels();

interface ChannelConfig {
    @WithDefault("APPEND")
    ChannelSemantic semantic();

    Optional<String> description();

    Optional<List<String>> allowedWriters();
    Optional<List<String>> adminInstances();
    Optional<List<String>> barrierContributors();
    Optional<List<String>> reviewerInstances();

    Optional<List<MessageType>> allowedTypes();
    Optional<List<MessageType>> deniedTypes();

    Optional<List<String>> protocols();
    Optional<List<String>> protocolParticipants();
    Optional<EnforcementMode> enforcementMode();
    Optional<List<String>> enforcementExclusions();

    OptionalInt rateLimitPerChannel();
    OptionalInt rateLimitPerInstance();

    Optional<Double> routingTrustThreshold();

    Optional<Boolean> trackDelivery();

    Optional<String> space();
    Optional<Map<String, String>> metadata();

    Optional<ConnectorConfig> connector();

    interface ConnectorConfig {
        String inboundConnectorId();
        String externalKey();
        String outboundConnectorId();
        String outboundDestination();
    }
}
```

Add required imports: `java.util.Map`, `java.util.Optional`, `java.util.OptionalInt`, `java.util.List`, `io.smallrye.config.WithDefault`.

- [ ] **Step 2: Add Spring counterpart classes to QhorusConfigProperties**

Add `ChannelConfigProps` and `ConnectorConfigProps` inner classes plus field/getter/setter. Use `ide_insert_member`:

```java
private Map<String, ChannelConfigProps> channels = new java.util.LinkedHashMap<>();

@Override public Map<String, ChannelConfig> channels() { return Map.copyOf(channels); }
public void setChannels(Map<String, ChannelConfigProps> v) { this.channels = v; }

public static class ChannelConfigProps implements ChannelConfig {
    private ChannelSemantic semantic = ChannelSemantic.APPEND;
    private String description;
    private List<String> allowedWriters;
    private List<String> adminInstances;
    private List<String> barrierContributors;
    private List<String> reviewerInstances;
    private List<MessageType> allowedTypes;
    private List<MessageType> deniedTypes;
    private List<String> protocols;
    private List<String> protocolParticipants;
    private EnforcementMode enforcementMode;
    private List<String> enforcementExclusions;
    private Integer rateLimitPerChannel;
    private Integer rateLimitPerInstance;
    private Double routingTrustThreshold;
    private Boolean trackDelivery;
    private String space;
    private Map<String, String> metadata;
    private ConnectorConfigProps connector;

    @Override public ChannelSemantic semantic() { return semantic; }
    @Override public Optional<String> description() { return Optional.ofNullable(description); }
    @Override public Optional<List<String>> allowedWriters() { return Optional.ofNullable(allowedWriters); }
    @Override public Optional<List<String>> adminInstances() { return Optional.ofNullable(adminInstances); }
    @Override public Optional<List<String>> barrierContributors() { return Optional.ofNullable(barrierContributors); }
    @Override public Optional<List<String>> reviewerInstances() { return Optional.ofNullable(reviewerInstances); }
    @Override public Optional<List<MessageType>> allowedTypes() { return Optional.ofNullable(allowedTypes); }
    @Override public Optional<List<MessageType>> deniedTypes() { return Optional.ofNullable(deniedTypes); }
    @Override public Optional<List<String>> protocols() { return Optional.ofNullable(protocols); }
    @Override public Optional<List<String>> protocolParticipants() { return Optional.ofNullable(protocolParticipants); }
    @Override public Optional<EnforcementMode> enforcementMode() { return Optional.ofNullable(enforcementMode); }
    @Override public Optional<List<String>> enforcementExclusions() { return Optional.ofNullable(enforcementExclusions); }
    @Override public OptionalInt rateLimitPerChannel() { return rateLimitPerChannel != null ? OptionalInt.of(rateLimitPerChannel) : OptionalInt.empty(); }
    @Override public OptionalInt rateLimitPerInstance() { return rateLimitPerInstance != null ? OptionalInt.of(rateLimitPerInstance) : OptionalInt.empty(); }
    @Override public Optional<Double> routingTrustThreshold() { return Optional.ofNullable(routingTrustThreshold); }
    @Override public Optional<Boolean> trackDelivery() { return Optional.ofNullable(trackDelivery); }
    @Override public Optional<String> space() { return Optional.ofNullable(space); }
    @Override public Optional<Map<String, String>> metadata() { return Optional.ofNullable(metadata); }
    @Override public Optional<ConnectorConfig> connector() { return Optional.ofNullable(connector); }

    // setters for Spring binding
    public void setSemantic(ChannelSemantic v) { this.semantic = v; }
    public void setDescription(String v) { this.description = v; }
    public void setAllowedWriters(List<String> v) { this.allowedWriters = v; }
    public void setAdminInstances(List<String> v) { this.adminInstances = v; }
    public void setBarrierContributors(List<String> v) { this.barrierContributors = v; }
    public void setReviewerInstances(List<String> v) { this.reviewerInstances = v; }
    public void setAllowedTypes(List<MessageType> v) { this.allowedTypes = v; }
    public void setDeniedTypes(List<MessageType> v) { this.deniedTypes = v; }
    public void setProtocols(List<String> v) { this.protocols = v; }
    public void setProtocolParticipants(List<String> v) { this.protocolParticipants = v; }
    public void setEnforcementMode(EnforcementMode v) { this.enforcementMode = v; }
    public void setEnforcementExclusions(List<String> v) { this.enforcementExclusions = v; }
    public void setRateLimitPerChannel(Integer v) { this.rateLimitPerChannel = v; }
    public void setRateLimitPerInstance(Integer v) { this.rateLimitPerInstance = v; }
    public void setRoutingTrustThreshold(Double v) { this.routingTrustThreshold = v; }
    public void setTrackDelivery(Boolean v) { this.trackDelivery = v; }
    public void setSpace(String v) { this.space = v; }
    public void setMetadata(Map<String, String> v) { this.metadata = v; }
    public void setConnector(ConnectorConfigProps v) { this.connector = v; }
}

public static class ConnectorConfigProps implements ChannelConfig.ConnectorConfig {
    private String inboundConnectorId;
    private String externalKey;
    private String outboundConnectorId;
    private String outboundDestination;

    @Override public String inboundConnectorId() { return inboundConnectorId; }
    @Override public String externalKey() { return externalKey; }
    @Override public String outboundConnectorId() { return outboundConnectorId; }
    @Override public String outboundDestination() { return outboundDestination; }

    public void setInboundConnectorId(String v) { this.inboundConnectorId = v; }
    public void setExternalKey(String v) { this.externalKey = v; }
    public void setOutboundConnectorId(String v) { this.outboundConnectorId = v; }
    public void setOutboundDestination(String v) { this.outboundDestination = v; }
}
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -f /path/to/qhorus/pom.xml -Dno-format`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 4: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java runtime-spring/src/main/java/io/casehub/qhorus/runtime/spring/config/QhorusConfigProperties.java
git commit -m "feat(config): add ChannelConfig to QhorusConfig — YAML channel declarations Refs #464"
```

### Task 2: ChannelConfigBootstrap — create channels from config

**Repo:** `casehub-qhorus`

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrap.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrapTest.java`

**Interfaces:**
- Consumes: `QhorusConfig.channels()` → `Map<String, ChannelConfig>`, `ChannelService.findOrCreate(ChannelCreateRequest)` → `FindOrCreateResult`, `SpaceService.findByName(String)` → `Optional<Space>`
- Produces: `ChannelConfigBootstrap` — `@ApplicationScoped` bean, `onStartup(@Observes StartupEvent)`, `processChannel(String, ChannelConfig)`, `toRequest(String, ChannelConfig)` → `ChannelCreateRequest`

- [ ] **Step 1: Write failing tests for channel creation from config**

Create CDI-free test class. Construct `ChannelConfigBootstrap` with Mockito mocks for `ChannelService` and `SpaceService`, and a test `QhorusConfig` implementation that returns a controlled channel map.

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.*;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.channel.ChannelService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class ChannelConfigBootstrapTest {

    private ChannelService channelService;
    private ChannelConfigBootstrap bootstrap;

    @BeforeEach
    void setUp() {
        channelService = mock(ChannelService.class);
    }

    @Test
    void emptyConfigDoesNothing() {
        bootstrap = new ChannelConfigBootstrap(
                configWithChannels(Map.of()), channelService, null);
        bootstrap.onStartup(null);
        verifyNoInteractions(channelService);
    }

    @Test
    void singleChannelCreated() {
        var channelConfig = minimalChannelConfig(ChannelSemantic.APPEND);
        bootstrap = new ChannelConfigBootstrap(
                configWithChannels(Map.of("reviews", channelConfig)),
                channelService, null);
        when(channelService.findOrCreate(any()))
                .thenReturn(new FindOrCreateResult(
                        stubChannel("reviews"), true));

        bootstrap.onStartup(null);

        var captor = ArgumentCaptor.forClass(ChannelCreateRequest.class);
        verify(channelService).findOrCreate(captor.capture());
        assertThat(captor.getValue().name()).isEqualTo("reviews");
        assertThat(captor.getValue().semantic()).isEqualTo(ChannelSemantic.APPEND);
    }

    @Test
    void exceptionInOneChannelDoesNotBlockOthers() {
        var config = configWithChannels(Map.of(
                "channel-a", minimalChannelConfig(ChannelSemantic.APPEND),
                "channel-b", minimalChannelConfig(ChannelSemantic.BROADCAST)));
        bootstrap = new ChannelConfigBootstrap(config, channelService, null);
        when(channelService.findOrCreate(argThat(r -> r.name().equals("channel-a"))))
                .thenThrow(new RuntimeException("boom"));
        when(channelService.findOrCreate(argThat(r -> r.name().equals("channel-b"))))
                .thenReturn(new FindOrCreateResult(
                        stubChannel("channel-b"), true));

        bootstrap.onStartup(null);

        verify(channelService, times(2)).findOrCreate(any());
    }

    // Helper methods: configWithChannels(), minimalChannelConfig(), stubChannel()
    // — construct test implementations of the config interfaces
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelConfigBootstrapTest -f /path/to/qhorus/pom.xml -Dno-format`
Expected: FAIL — class not found

- [ ] **Step 3: Implement ChannelConfigBootstrap**

Create the bean in `runtime/src/main/java/.../runtime/channel/ChannelConfigBootstrap.java` using `ide_create_file`:

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.*;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.config.QhorusConfig.ChannelConfig;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.*;

@ApplicationScoped
public class ChannelConfigBootstrap {

    private static final Logger LOG = Logger.getLogger(ChannelConfigBootstrap.class);

    private final QhorusConfig config;
    private final ChannelService channelService;
    private final SpaceService spaceService;

    @Inject
    public ChannelConfigBootstrap(QhorusConfig config,
                                   ChannelService channelService,
                                   SpaceService spaceService) {
        this.config = config;
        this.channelService = channelService;
        this.spaceService = spaceService;
    }

    void onStartup(@Observes StartupEvent event) {
        var channels = config.channels();
        if (channels.isEmpty()) return;

        int created = 0, existing = 0;
        for (var entry : channels.entrySet()) {
            try {
                var result = processChannel(entry.getKey(), entry.getValue());
                if (result.wasCreated()) created++;
                else existing++;
            } catch (Exception e) {
                LOG.warnf("Failed to bootstrap channel '%s': %s",
                          entry.getKey(), e.getMessage());
            }
        }
        LOG.infof("Channel config bootstrap: %d channels processed, %d created, %d existing",
                  channels.size(), created, existing);
    }

    FindOrCreateResult processChannel(String name, ChannelConfig cfg) {
        var request = toRequest(name, cfg);
        return channelService.findOrCreate(request);
    }

    ChannelCreateRequest toRequest(String name, ChannelConfig cfg) {
        var builder = ChannelCreateRequest.builder(name)
                .semantic(cfg.semantic());
        cfg.description().ifPresent(builder::description);
        cfg.allowedWriters().ifPresent(builder::allowedWriters);
        cfg.adminInstances().ifPresent(builder::adminInstances);
        cfg.barrierContributors().ifPresent(builder::barrierContributors);
        cfg.reviewerInstances().ifPresent(builder::reviewerInstances);
        cfg.allowedTypes().ifPresent(list -> builder.allowedTypes(new LinkedHashSet<>(list)));
        cfg.deniedTypes().ifPresent(list -> builder.deniedTypes(new LinkedHashSet<>(list)));
        cfg.protocols().ifPresent(builder::protocols);
        cfg.protocolParticipants().ifPresent(builder::protocolParticipants);
        cfg.enforcementMode().ifPresent(builder::enforcementMode);
        cfg.enforcementExclusions().ifPresent(builder::enforcementExclusions);
        if (cfg.rateLimitPerChannel().isPresent())
            builder.rateLimitPerChannel(cfg.rateLimitPerChannel().getAsInt());
        if (cfg.rateLimitPerInstance().isPresent())
            builder.rateLimitPerInstance(cfg.rateLimitPerInstance().getAsInt());
        cfg.routingTrustThreshold().ifPresent(builder::routingTrustThreshold);
        cfg.trackDelivery().ifPresent(builder::trackDelivery);
        cfg.metadata().ifPresent(builder::metadata);

        // Space resolution
        cfg.space().ifPresent(spaceName -> {
            try {
                var space = spaceService.findByName(spaceName);
                space.ifPresentOrElse(
                        s -> builder.spaceId(s.id()),
                        () -> LOG.warnf("Channel '%s': space '%s' not found — skipping space assignment", name, spaceName));
            } catch (IllegalStateException e) {
                LOG.warnf("Channel '%s': space '%s' is ambiguous — skipping space assignment", name, spaceName);
            }
        });

        // Connector binding
        cfg.connector().ifPresent(conn -> {
            builder.inboundConnectorId(conn.inboundConnectorId());
            builder.externalKey(conn.externalKey());
            builder.outboundConnectorId(conn.outboundConnectorId());
            builder.outboundDestination(conn.outboundDestination());
        });

        return builder.build();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelConfigBootstrapTest -f /path/to/qhorus/pom.xml -Dno-format`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrap.java runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrapTest.java
git commit -m "feat(channel): add ChannelConfigBootstrap — create channels from YAML config at startup Refs #464"
```

---

## Batch 2: Drift detection and integration test

### Task 3: Drift detection for existing channels

**Repo:** `casehub-qhorus`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrap.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrapTest.java`

**Interfaces:**
- Consumes: `Channel` record (29 fields — compared against `ChannelConfig`), `FindOrCreateResult.wasCreated()` (skip drift if newly created)
- Produces: `ChannelDrift` record (channelName, field, declared, actual), `detectDrift(String, ChannelConfig, Channel)` → `List<ChannelDrift>`

- [ ] **Step 1: Write failing tests for drift detection**

Add to existing test class:

```java
@Test
void existingChannelWithNoDrift() {
    var channelConfig = channelConfigWith(ChannelSemantic.APPEND, "A review channel");
    var existingChannel = channelWith("reviews", ChannelSemantic.APPEND, "A review channel");
    bootstrap = new ChannelConfigBootstrap(
            configWithChannels(Map.of("reviews", channelConfig)),
            channelService, null);
    when(channelService.findOrCreate(any()))
            .thenReturn(new FindOrCreateResult(existingChannel, false));

    bootstrap.onStartup(null);
    // no drift warnings — assert via log capture or return value
}

@Test
void existingChannelWithSemanticDrift() {
    var channelConfig = channelConfigWith(ChannelSemantic.COLLECT, "Reviews");
    var existingChannel = channelWith("reviews", ChannelSemantic.APPEND, "Reviews");
    bootstrap = new ChannelConfigBootstrap(
            configWithChannels(Map.of("reviews", channelConfig)),
            channelService, null);
    when(channelService.findOrCreate(any()))
            .thenReturn(new FindOrCreateResult(existingChannel, false));

    var drifts = bootstrap.processChannelWithDrift("reviews", channelConfig);
    assertThat(drifts).hasSize(1);
    assertThat(drifts.get(0).field()).isEqualTo("semantic");
    assertThat(drifts.get(0).declared()).isEqualTo("COLLECT");
    assertThat(drifts.get(0).actual()).isEqualTo("APPEND");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelConfigBootstrapTest -f /path/to/qhorus/pom.xml -Dno-format`
Expected: FAIL — method not found

- [ ] **Step 3: Implement drift detection**

Add `ChannelDrift` record and `detectDrift()` method to `ChannelConfigBootstrap`. Use `ide_insert_member`:

```java
record ChannelDrift(String channelName, String field, String declared, String actual) {}

List<ChannelDrift> detectDrift(String name, ChannelConfig cfg, Channel existing) {
    var drifts = new ArrayList<ChannelDrift>();

    if (cfg.semantic() != existing.semantic())
        drifts.add(new ChannelDrift(name, "semantic",
                cfg.semantic().name(), existing.semantic().name()));

    cfg.description().ifPresent(d -> {
        if (!Objects.equals(d, existing.description()))
            drifts.add(new ChannelDrift(name, "description", d,
                    existing.description()));
    });

    cfg.allowedWriters().ifPresent(declared -> {
        if (!declared.equals(existing.allowedWriters()))
            drifts.add(new ChannelDrift(name, "allowedWriters",
                    declared.toString(), existing.allowedWriters().toString()));
    });

    // ... same pattern for all compared fields:
    // adminInstances, barrierContributors, reviewerInstances,
    // allowedTypes, deniedTypes, protocols, protocolParticipants,
    // enforcementMode, enforcementExclusions, rateLimitPerChannel,
    // rateLimitPerInstance, routingTrustThreshold, trackDelivery

    return drifts;
}
```

Update `processChannel()` to call `detectDrift()` when `!result.wasCreated()` and log each drift as WARN.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelConfigBootstrapTest -f /path/to/qhorus/pom.xml -Dno-format`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrap.java runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrapTest.java
git commit -m "feat(channel): add drift detection for YAML-declared channels Refs #464"
```

### Task 4: Integration test — channels created from application.yaml

**Repo:** `casehub-qhorus`

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrapIntegrationTest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigProfile.java`

**Interfaces:**
- Consumes: `ChannelService.findByName(String)` → `Optional<Channel>`, `ChannelConfigBootstrap` (auto-runs on startup)
- Produces: Integration test proving end-to-end YAML → channel creation

- [ ] **Step 1: Write the TestProfile**

```java
package io.casehub.qhorus.runtime.channel;

import io.quarkus.test.junit.QuarkusTestProfile;
import java.util.*;

public class ChannelConfigProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        Map<String, String> config = new HashMap<>();
        // Declare two channels
        config.put("casehub.qhorus.channels.it-reviews.semantic", "APPEND");
        config.put("casehub.qhorus.channels.it-reviews.description", "Integration test reviews");
        config.put("casehub.qhorus.channels.it-status.semantic", "BROADCAST");
        config.put("casehub.qhorus.channels.it-status.description", "Integration test status");
        // Required datasource overrides for profile restart
        config.put("quarkus.datasource.qhorus.db-kind", "h2");
        config.put("quarkus.datasource.qhorus.jdbc.url", "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1");
        config.put("quarkus.datasource.qhorus.username", "sa");
        config.put("quarkus.datasource.qhorus.password", "");
        config.put("quarkus.datasource.qhorus.reactive", "false");
        config.put("quarkus.hibernate-orm.qhorus.database.generation", "drop-and-create");
        return config;
    }
}
```

- [ ] **Step 2: Write the integration test**

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(ChannelConfigProfile.class)
class ChannelConfigBootstrapIntegrationTest {

    @Inject
    ChannelService channelService;

    @Test
    void channelsDeclaredInConfigExistAfterStartup() {
        var reviews = channelService.findByName("it-reviews");
        assertThat(reviews).isPresent();
        assertThat(reviews.get().semantic()).isEqualTo(ChannelSemantic.APPEND);
        assertThat(reviews.get().description()).isEqualTo("Integration test reviews");

        var status = channelService.findByName("it-status");
        assertThat(status).isPresent();
        assertThat(status.get().semantic()).isEqualTo(ChannelSemantic.BROADCAST);
    }
}
```

- [ ] **Step 3: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelConfigBootstrapIntegrationTest -f /path/to/qhorus/pom.xml -Dno-format`
Expected: PASS

- [ ] **Step 4: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /path/to/qhorus/pom.xml -Dno-format`
Expected: PASS — 0 regressions

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrapIntegrationTest.java runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelConfigProfile.java
git commit -m "test(channel): add integration test for YAML channel bootstrap Refs #464"
```

---

## References

- [specs/issue-464-yaml-channel-config/2026-10-13-yaml-channel-config-design.md] — design spec
- [api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java] — builder API, compact constructor validation
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java:72] — findOrCreate() method
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java] — existing @ConfigMapping
- [api/src/main/java/io/casehub/qhorus/api/channel/Channel.java] — Channel record (29 fields)
- PP-20260522-c3a8c1 — startup exception isolation protocol
- PP-20260522-e5e527 — ChannelInitialisedEvent idempotency protocol
- GitHub #464 — focal issue
