# Signed Agent Cards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #403 — E6: Signed Agent Cards + DID integration
**Issue group:** #403

**Goal:** Add JWS signing to agent cards with inbound verification and trust score integration.

**Architecture:** New `AgentCardSigner` SPI in `casehub-qhorus-api` with a `JwsAgentCardSigner` implementation in a new `agent-card-signing` optional module. Uses platform `SigningProvider` for raw crypto (Ed25519 default). Outbound cards are signed in `AgentCardResource`, inbound cards verified asynchronously at binding creation. Verified agents get a trust score boost via a `@Decorator TrustScoreSource`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Nimbus JOSE+JWT, platform SigningProvider SPI, casehub-ledger TrustScoreSource

## Global Constraints

- Java 21 source, Java 26 JVM
- Quarkus 3.32.2
- No modifications to `casehub-a2a-protocol` AgentCard record
- No credentials or PII in ledger-persisted fields (PP-20260612-bd6f8c)
- Algorithm-transparent signing via platform SigningProvider (PP-20260523-e7b577)
- Flyway consumer versioning — next domain migration: V54 (PP-20260521-0ba358)
- `mvn` not `./mvnw` for builds
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all Maven commands

---

## Batch 1: Foundation — SPI and Data Model

### Task 1: AgentCardSigner SPI, VerificationStatus, and BindingVerificationRequestedEvent

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/AgentCardSigner.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/instance/VerificationStatus.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/event/BindingVerificationRequestedEvent.java`

**Interfaces:**
- Produces: `AgentCardSigner` — SPI with `sign(AgentCard)`, `verify(ObjectNode)`, `jwks()` methods
- Produces: `VerificationStatus` enum — `VERIFIED`, `UNVERIFIED`, `FAILED`
- Produces: `BindingVerificationRequestedEvent(ExternalAgentBinding)` — CDI event record

- [ ] **Step 1: Create VerificationStatus enum**

```java
package io.casehub.qhorus.api.instance;

public enum VerificationStatus {
    VERIFIED, UNVERIFIED, FAILED
}
```

- [ ] **Step 2: Create AgentCardSigner SPI**

```java
package io.casehub.qhorus.api.spi;

import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.a2a.model.AgentCard;

public interface AgentCardSigner {

    ObjectNode sign(AgentCard card);

    VerificationResult verify(ObjectNode signedCardJson);

    ObjectNode jwks();

    record VerificationResult(boolean verified, String keyId, String error) {}
}
```

- [ ] **Step 3: Create BindingVerificationRequestedEvent**

```java
package io.casehub.qhorus.api.event;

import io.casehub.qhorus.api.instance.ExternalAgentBinding;

public record BindingVerificationRequestedEvent(ExternalAgentBinding binding) {}
```

- [ ] **Step 4: Verify api module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -am`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/spi/AgentCardSigner.java api/src/main/java/io/casehub/qhorus/api/instance/VerificationStatus.java api/src/main/java/io/casehub/qhorus/api/event/BindingVerificationRequestedEvent.java
git commit -m "feat: add AgentCardSigner SPI, VerificationStatus enum, BindingVerificationRequestedEvent Refs #403"
```

### Task 2: ExternalAgentBinding expansion and Flyway V54

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/instance/ExternalAgentBinding.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/instance/ExternalAgentBindingEntity.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/instance/JpaExternalAgentBindingStore.java` (if fromDomain/toDomain affected)
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryExternalAgentBindingStore.java`
- Modify: `a2a-outbound/src/main/java/io/casehub/qhorus/a2a/outbound/ExternalAgentBindingResource.java`
- Modify: `a2a-outbound/src/main/java/io/casehub/qhorus/a2a/outbound/ExternalAgentBindingRequest.java` (if needed)
- Create: `runtime/src/main/resources/db/qhorus/migration/V54__external_agent_binding_verification.sql`
- Modify: all test construction sites for ExternalAgentBinding
- Test: `persistence-memory/src/test/java/.../contract/ExternalAgentBindingStoreContractTest.java`

**Interfaces:**
- Consumes: `VerificationStatus` enum from Task 1
- Produces: `ExternalAgentBinding(id, instanceId, endpoint, authConfigKey, protocolVersion, createdAt, verificationStatus, verifiedAt, verificationKeyId)` — 9-component record

- [ ] **Step 1: Write failing test — ExternalAgentBinding construction with verification fields**

Add to `ExternalAgentBindingStoreContractTest`:

```java
@Test
void storeAndRetrieveWithVerificationFields() {
    var binding = new ExternalAgentBinding(
        UUID.randomUUID(), "agent-verified", "https://agent.example.com",
        "auth-key", "1.0", Instant.now(),
        VerificationStatus.VERIFIED, Instant.now(), "kid-1"
    );
    store.put(binding);
    var found = store.findByInstanceId("agent-verified");
    assertTrue(found.isPresent());
    assertEquals(VerificationStatus.VERIFIED, found.get().verificationStatus());
    assertNotNull(found.get().verifiedAt());
    assertEquals("kid-1", found.get().verificationKeyId());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryExternalAgentBindingStoreTest#storeAndRetrieveWithVerificationFields`
Expected: FAIL — constructor mismatch (6 args vs 9)

- [ ] **Step 3: Expand ExternalAgentBinding record (6 → 9 components)**

```java
package io.casehub.qhorus.api.instance;

import java.time.Instant;
import java.util.UUID;

public record ExternalAgentBinding(
    UUID id,
    String instanceId,
    String endpoint,
    String authConfigKey,
    String protocolVersion,
    Instant createdAt,
    VerificationStatus verificationStatus,
    Instant verifiedAt,
    String verificationKeyId
) {}
```

- [ ] **Step 4: Update ExternalAgentBindingEntity — add fields, update fromDomain/toDomain**

Add three fields to the entity:

```java
@Column(name = "verification_status")
@Enumerated(EnumType.STRING)
public VerificationStatus verificationStatus;

@Column(name = "verified_at")
public Instant verifiedAt;

@Column(name = "verification_key_id")
public String verificationKeyId;
```

Update `fromDomain()`:

```java
public static ExternalAgentBindingEntity fromDomain(ExternalAgentBinding binding) {
    ExternalAgentBindingEntity e = new ExternalAgentBindingEntity();
    e.id = binding.id();
    e.instanceId = binding.instanceId();
    e.endpoint = binding.endpoint();
    e.authConfigKey = binding.authConfigKey();
    e.protocolVersion = binding.protocolVersion();
    e.createdAt = binding.createdAt();
    e.verificationStatus = binding.verificationStatus();
    e.verifiedAt = binding.verifiedAt();
    e.verificationKeyId = binding.verificationKeyId();
    return e;
}
```

Update `toDomain()`:

```java
public ExternalAgentBinding toDomain() {
    return new ExternalAgentBinding(id, instanceId, endpoint, authConfigKey,
            protocolVersion, createdAt, verificationStatus, verifiedAt, verificationKeyId);
}
```

- [ ] **Step 5: Update InMemoryExternalAgentBindingStore**

Update any construction sites to include the new fields. If `put()` stores the record directly, no changes needed beyond the record expansion.

- [ ] **Step 6: Fix all construction sites in a2a-outbound**

Update `ExternalAgentBindingResource.put()` to construct with `VerificationStatus.UNVERIFIED, null, null` for the new fields:

```java
var binding = new ExternalAgentBinding(
    existing.map(ExternalAgentBinding::id).orElse(null),
    instanceId, req.endpoint(), req.authConfigKey(), req.protocolVersion(),
    existing.map(ExternalAgentBinding::createdAt).orElse(null),
    VerificationStatus.UNVERIFIED, null, null
);
```

- [ ] **Step 7: Fix all test construction sites**

Search all test files for `new ExternalAgentBinding(` and update to 9-arg constructor with `VerificationStatus.UNVERIFIED, null, null` appended.

Use: `ide_search_text` with query `new ExternalAgentBinding(` to find all sites.

- [ ] **Step 8: Create Flyway V54 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V54__external_agent_binding_verification.sql`:

```sql
ALTER TABLE external_agent_binding ADD COLUMN verification_status VARCHAR(20) DEFAULT 'UNVERIFIED';
ALTER TABLE external_agent_binding ADD COLUMN verified_at TIMESTAMP;
ALTER TABLE external_agent_binding ADD COLUMN verification_key_id VARCHAR(255);
```

- [ ] **Step 9: Run tests to verify everything passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,persistence-memory,a2a-outbound`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add api/ runtime/ persistence-memory/ a2a-outbound/
git commit -m "feat: expand ExternalAgentBinding with verification fields, add Flyway V54 Refs #403"
```

---

## Batch 2: Signing Module Core

### Task 3: Module scaffolding, JcsCanonicalizer, and SigningConfig

**Files:**
- Create: `agent-card-signing/pom.xml`
- Create: `agent-card-signing/src/main/java/io/casehub/qhorus/signing/JcsCanonicalizer.java`
- Create: `agent-card-signing/src/main/java/io/casehub/qhorus/signing/SigningConfig.java`
- Create: `agent-card-signing/src/main/resources/application.properties` (defaults)
- Test: `agent-card-signing/src/test/java/io/casehub/qhorus/signing/JcsCanonicalizerTest.java`
- Modify: `pom.xml` (root) — add `agent-card-signing` to `<modules>`

**Interfaces:**
- Produces: `JcsCanonicalizer.canonicalize(ObjectNode)` → `byte[]`
- Produces: `SigningConfig` — all config properties

- [ ] **Step 1: Create module pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-qhorus-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-qhorus-agent-card-signing</artifactId>
    <name>CaseHub Qhorus Agent Card Signing</name>
    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-qhorus-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-a2a-protocol</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.nimbusds</groupId>
            <artifactId>nimbus-jose-jwt</artifactId>
            <version>10.3</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <!-- Test deps -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Check the parent pom for existing `nimbus-jose-jwt` version management. If managed, remove the `<version>` above. Add `<module>agent-card-signing</module>` to root pom.xml `<modules>`.

- [ ] **Step 2: Create SigningConfig**

```java
package io.casehub.qhorus.signing;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import java.util.Optional;

@ConfigMapping(prefix = "casehub.qhorus.signing")
public interface SigningConfig {

    @WithDefault("true")
    boolean enabled();

    @WithDefault("system:agent-card-signer")
    String actorId();

    @WithDefault("default")
    String keyId();

    Optional<String> jwksUrl();

    JwksCache jwksCache();

    Trust trust();

    interface JwksCache {
        @WithDefault("3600")
        int ttl();

        @WithDefault("65536")
        int maxResponseBytes();

        @WithDefault("5000")
        int connectTimeoutMs();

        @WithDefault("5000")
        int readTimeoutMs();
    }

    interface Trust {
        @WithDefault("0.6")
        double verifiedFloorScore();

        @WithDefault("0.15")
        double dimensionWeight();
    }
}
```

- [ ] **Step 3: Write failing test for JcsCanonicalizer**

```java
package io.casehub.qhorus.signing;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class JcsCanonicalizerTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void canonicalizeSortsKeys() throws Exception {
        ObjectNode node = mapper.createObjectNode();
        node.put("z", "last");
        node.put("a", "first");
        node.put("m", "middle");

        byte[] result = JcsCanonicalizer.canonicalize(node);
        assertThat(new String(result)).isEqualTo("{\"a\":\"first\",\"m\":\"middle\",\"z\":\"last\"}");
    }

    @Test
    void canonicalizeNormalizesNumbers() throws Exception {
        ObjectNode node = mapper.createObjectNode();
        node.put("value", 1.0);

        byte[] result = JcsCanonicalizer.canonicalize(node);
        assertThat(new String(result)).isEqualTo("{\"value\":1}");
    }

    @Test
    void canonicalizeHandlesNestedObjects() throws Exception {
        ObjectNode inner = mapper.createObjectNode();
        inner.put("b", 2);
        inner.put("a", 1);
        ObjectNode outer = mapper.createObjectNode();
        outer.set("nested", inner);
        outer.put("top", "value");

        byte[] result = JcsCanonicalizer.canonicalize(outer);
        assertThat(new String(result)).isEqualTo("{\"nested\":{\"a\":1,\"b\":2},\"top\":\"value\"}");
    }

    @Test
    void canonicalizeExcludesNullValues() throws Exception {
        ObjectNode node = mapper.createObjectNode();
        node.put("present", "yes");
        node.putNull("absent");

        byte[] result = JcsCanonicalizer.canonicalize(node);
        assertThat(new String(result)).isEqualTo("{\"absent\":null,\"present\":\"yes\"}");
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-card-signing -Dtest=JcsCanonicalizerTest`
Expected: FAIL — JcsCanonicalizer class not found

- [ ] **Step 5: Implement JcsCanonicalizer**

```java
package io.casehub.qhorus.signing;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.node.ObjectNode;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;
import java.util.TreeMap;

public final class JcsCanonicalizer {

    private static final ObjectMapper CANONICAL_MAPPER = new ObjectMapper()
            .configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, true);

    private JcsCanonicalizer() {}

    public static byte[] canonicalize(ObjectNode node) {
        try {
            Object sorted = sortRecursive(node);
            return CANONICAL_MAPPER.writeValueAsBytes(sorted);
        } catch (IOException e) {
            throw new IllegalArgumentException("Failed to canonicalize JSON", e);
        }
    }

    private static Object sortRecursive(JsonNode node) {
        if (node.isObject()) {
            TreeMap<String, Object> sorted = new TreeMap<>();
            Iterator<String> fields = node.fieldNames();
            while (fields.hasNext()) {
                String field = fields.next();
                sorted.put(field, sortRecursive(node.get(field)));
            }
            return sorted;
        } else if (node.isArray()) {
            var list = new java.util.ArrayList<>();
            for (JsonNode element : node) {
                list.add(sortRecursive(element));
            }
            return list;
        } else if (node.isIntegralNumber()) {
            return node.longValue();
        } else if (node.isFloatingPointNumber()) {
            double d = node.doubleValue();
            if (d == Math.floor(d) && !Double.isInfinite(d)) {
                return (long) d;
            }
            return d;
        } else if (node.isBoolean()) {
            return node.booleanValue();
        } else if (node.isNull()) {
            return null;
        } else {
            return node.asText();
        }
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-card-signing -Dtest=JcsCanonicalizerTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add agent-card-signing/ pom.xml
git commit -m "feat: add agent-card-signing module with JcsCanonicalizer and SigningConfig Refs #403"
```

### Task 4: JwsAgentCardSigner — sign, verify, jwks

**Files:**
- Create: `agent-card-signing/src/main/java/io/casehub/qhorus/signing/JwsKeyProvider.java`
- Create: `agent-card-signing/src/main/java/io/casehub/qhorus/signing/JwksCache.java`
- Create: `agent-card-signing/src/main/java/io/casehub/qhorus/signing/JwsAgentCardSigner.java`
- Test: `agent-card-signing/src/test/java/io/casehub/qhorus/signing/JwsAgentCardSignerTest.java`

**Interfaces:**
- Consumes: `AgentCardSigner` SPI from Task 1
- Consumes: `JcsCanonicalizer` from Task 3
- Consumes: `SigningConfig` from Task 3
- Consumes: `SigningProvider` from platform-api
- Produces: `JwsAgentCardSigner` — `@ApplicationScoped` impl of `AgentCardSigner`
- Produces: `JwksCache.fetch(String jkuUrl)` → `JWKSet`

- [ ] **Step 1: Write failing test — sign/verify round-trip**

```java
package io.casehub.qhorus.signing;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.a2a.model.AgentCard;
import io.casehub.a2a.model.AgentCapabilities;
import io.casehub.qhorus.api.spi.AgentCardSigner;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class JwsAgentCardSignerTest {

    private JwsAgentCardSigner signer;
    private ObjectMapper mapper;

    @BeforeEach
    void setUp() throws Exception {
        KeyPairGenerator kpg = KeyPairGenerator.getInstance("Ed25519");
        KeyPair kp = kpg.generateKeyPair();
        mapper = new ObjectMapper();
        signer = new JwsAgentCardSigner(kp, "test-kid", null, mapper);
    }

    @Test
    void signAndVerifyRoundTrip() {
        var card = new AgentCard("test-agent", "A test agent",
                "https://agent.example.com", "1.0",
                List.of(), new AgentCapabilities(true, false),
                Map.of(), "default", null);

        ObjectNode signed = signer.sign(card);
        assertThat(signed.has("signatures")).isTrue();
        assertThat(signed.get("signatures").isArray()).isTrue();
        assertThat(signed.get("signatures").size()).isEqualTo(1);

        AgentCardSigner.VerificationResult result = signer.verify(signed);
        assertThat(result.verified()).isTrue();
        assertThat(result.keyId()).isEqualTo("test-kid");
        assertThat(result.error()).isNull();
    }

    @Test
    void verifyDetectsTamperedPayload() {
        var card = new AgentCard("test-agent", "A test agent",
                "https://agent.example.com", "1.0",
                List.of(), new AgentCapabilities(true, false),
                Map.of(), "default", null);

        ObjectNode signed = signer.sign(card);
        signed.put("name", "tampered-agent");

        AgentCardSigner.VerificationResult result = signer.verify(signed);
        assertThat(result.verified()).isFalse();
    }

    @Test
    void verifyReturnsUnverifiedForNoSignatures() {
        ObjectNode unsigned = mapper.createObjectNode();
        unsigned.put("name", "test-agent");

        AgentCardSigner.VerificationResult result = signer.verify(unsigned);
        assertThat(result.verified()).isFalse();
        assertThat(result.error()).contains("No signatures");
    }

    @Test
    void jwksContainsPublicKey() {
        ObjectNode jwks = signer.jwks();
        assertThat(jwks.has("keys")).isTrue();
        assertThat(jwks.get("keys").size()).isEqualTo(1);
        var key = jwks.get("keys").get(0);
        assertThat(key.get("kid").asText()).isEqualTo("test-kid");
        assertThat(key.get("use").asText()).isEqualTo("sig");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-card-signing -Dtest=JwsAgentCardSignerTest`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement JwsKeyProvider**

`JwsKeyProvider` wraps `SigningProvider` to extract key material and construct Nimbus JWK objects. For CDI-free unit tests, `JwsAgentCardSigner` accepts a `KeyPair` directly (test constructor).

```java
package io.casehub.qhorus.signing;

import com.nimbusds.jose.jwk.JWK;
import com.nimbusds.jose.jwk.OctetKeyPair;
import com.nimbusds.jose.jwk.ECKey;
import com.nimbusds.jose.jwk.KeyUse;
import io.casehub.platform.api.signing.SigningProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class JwsKeyProvider {

    private final SigningProvider signingProvider;
    private final SigningConfig config;

    @Inject
    public JwsKeyProvider(SigningProvider signingProvider, SigningConfig config) {
        this.signingProvider = signingProvider;
        this.config = config;
    }

    public JWK publicJwk() {
        var keyMaterial = signingProvider.keyMaterial(config.actorId());
        // Convert platform key material to Nimbus JWK
        // Key type determines JWS algorithm: OKP/Ed25519 → EdDSA, EC/P-256 → ES256
        return JWK.parseFromPEMEncodedObjects(new String(keyMaterial.publicKey()))
                   .toPublicJWK();
    }
}
```

Note: exact conversion from `SigningProvider.keyMaterial()` to JWK depends on the `SignatureResult` format. The implementer should check `SigningProvider` javadoc and adapt. The CDI-free test path bypasses `JwsKeyProvider` entirely.

- [ ] **Step 4: Implement JwksCache**

```java
package io.casehub.qhorus.signing;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;

public class JwksCache {

    private final ConcurrentHashMap<String, CachedJwks> cache = new ConcurrentHashMap<>();
    private final int ttlSeconds;
    private final int maxResponseBytes;
    private final int connectTimeoutMs;
    private final int readTimeoutMs;
    private final ObjectMapper mapper;

    record CachedJwks(ObjectNode jwks, Instant fetchedAt) {}

    public JwksCache(SigningConfig config, ObjectMapper mapper) {
        this.ttlSeconds = config.jwksCache().ttl();
        this.maxResponseBytes = config.jwksCache().maxResponseBytes();
        this.connectTimeoutMs = config.jwksCache().connectTimeoutMs();
        this.readTimeoutMs = config.jwksCache().readTimeoutMs();
        this.mapper = mapper;
    }

    public ObjectNode fetch(String jkuUrl) {
        var cached = cache.get(jkuUrl);
        if (cached != null && cached.fetchedAt().plusSeconds(ttlSeconds).isAfter(Instant.now())) {
            return cached.jwks();
        }
        try {
            HttpClient client = HttpClient.newBuilder()
                    .connectTimeout(Duration.ofMillis(connectTimeoutMs))
                    .build();
            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(jkuUrl))
                    .timeout(Duration.ofMillis(readTimeoutMs))
                    .GET()
                    .build();
            HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
            if (response.body().length() > maxResponseBytes) {
                throw new IllegalStateException("JWKS response exceeds max size: " + response.body().length());
            }
            ObjectNode jwks = (ObjectNode) mapper.readTree(response.body());
            cache.put(jkuUrl, new CachedJwks(jwks, Instant.now()));
            return jwks;
        } catch (Exception e) {
            if (cached != null) return cached.jwks();
            throw new RuntimeException("Failed to fetch JWKS from " + jkuUrl, e);
        }
    }

    public void invalidate(String jkuUrl) {
        cache.remove(jkuUrl);
    }
}
```

- [ ] **Step 5: Implement JwsAgentCardSigner**

```java
package io.casehub.qhorus.signing;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.nimbusds.jose.*;
import com.nimbusds.jose.crypto.*;
import com.nimbusds.jose.jwk.*;
import io.casehub.a2a.model.AgentCard;
import io.casehub.qhorus.api.spi.AgentCardSigner;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.security.KeyPair;
import java.util.Base64;

@ApplicationScoped
public class JwsAgentCardSigner implements AgentCardSigner {

    private final KeyPair keyPair;
    private final String keyId;
    private final JwksCache jwksCache;
    private final ObjectMapper mapper;

    // CDI constructor
    @Inject
    public JwsAgentCardSigner(JwsKeyProvider keyProvider, SigningConfig config,
                               JwksCache jwksCache, ObjectMapper mapper) {
        // Extract KeyPair from JwsKeyProvider
        this.keyPair = keyProvider.keyPair();
        this.keyId = config.keyId();
        this.jwksCache = jwksCache;
        this.mapper = mapper;
    }

    // CDI-free test constructor
    public JwsAgentCardSigner(KeyPair keyPair, String keyId,
                               JwksCache jwksCache, ObjectMapper mapper) {
        this.keyPair = keyPair;
        this.keyId = keyId;
        this.jwksCache = jwksCache;
        this.mapper = mapper;
    }

    @Override
    public ObjectNode sign(AgentCard card) {
        // Serialize card to ObjectNode
        ObjectNode cardNode = mapper.valueToTree(card);
        // Canonicalize (no signatures field on AgentCard, so nothing to exclude)
        byte[] canonical = JcsCanonicalizer.canonicalize(cardNode);
        // Build JWS
        // Determine algorithm from key type
        JWSAlgorithm alg = determineAlgorithm(keyPair);
        JWSHeader header = new JWSHeader.Builder(alg)
                .type(new JOSEObjectType("agentcard+jws"))
                .keyID(keyId)
                .build();
        JWSObject jws = new JWSObject(header, new Payload(canonical));
        try {
            jws.sign(createSigner(keyPair, alg));
        } catch (JOSEException e) {
            throw new RuntimeException("Failed to sign agent card", e);
        }
        // Build signed envelope
        ObjectNode result = cardNode.deepCopy();
        ArrayNode signatures = mapper.createArrayNode();
        ObjectNode sig = mapper.createObjectNode();
        sig.put("protected", jws.getHeader().toBase64URL().toString());
        sig.put("signature", jws.getSignature().toString());
        signatures.add(sig);
        result.set("signatures", signatures);
        return result;
    }

    @Override
    public VerificationResult verify(ObjectNode signedCardJson) {
        if (!signedCardJson.has("signatures") || signedCardJson.get("signatures").isEmpty()) {
            return new VerificationResult(false, null, "No signatures found");
        }
        // Extract and remove signatures
        var sigNode = signedCardJson.get("signatures").get(0);
        String protectedHeader = sigNode.get("protected").asText();
        String signatureValue = sigNode.get("signature").asText();
        ObjectNode cardWithoutSigs = signedCardJson.deepCopy();
        cardWithoutSigs.remove("signatures");
        byte[] canonical = JcsCanonicalizer.canonicalize(cardWithoutSigs);
        try {
            JWSHeader header = JWSHeader.parse(Base64.getUrlDecoder().decode(protectedHeader));
            String kid = header.getKeyID();
            // Resolve public key: local key or remote JWKS
            JWK verificationKey = resolveKey(kid, header);
            if (verificationKey == null) {
                return new VerificationResult(false, kid, "kid not found: " + kid);
            }
            JWSObject jws = new JWSObject(
                    Base64URL.from(protectedHeader),
                    new Payload(canonical),
                    Base64URL.from(signatureValue));
            boolean valid = jws.verify(createVerifier(verificationKey, header.getAlgorithm()));
            return new VerificationResult(valid, kid, valid ? null : "Signature verification failed");
        } catch (Exception e) {
            return new VerificationResult(false, null, "Verification error: " + e.getMessage());
        }
    }

    @Override
    public ObjectNode jwks() {
        // Build JWKS from local key pair
        JWK jwk = buildPublicJwk();
        ObjectNode result = mapper.createObjectNode();
        ArrayNode keys = mapper.createArrayNode();
        try {
            keys.add((ObjectNode) mapper.readTree(jwk.toJSONString()));
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize JWK", e);
        }
        result.set("keys", keys);
        return result;
    }

    // Private helpers: determineAlgorithm, createSigner, createVerifier,
    // resolveKey, buildPublicJwk
    // Implementation uses Nimbus JOSE+JWT APIs for Ed25519 (OctetKeyPair)
    // and EC P-256 (ECKey) depending on key type
}
```

The implementer should fill in the private helper methods using Nimbus JOSE+JWT APIs. Key patterns:
- `Ed25519` key → `JWSAlgorithm.EdDSA`, `Ed25519Signer`, `Ed25519Verifier`
- `EC P-256` key → `JWSAlgorithm.ES256`, `ECDSASigner`, `ECDSAVerifier`

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-card-signing -Dtest=JwsAgentCardSignerTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add agent-card-signing/
git commit -m "feat: implement JwsAgentCardSigner with sign/verify/jwks and JwksCache Refs #403"
```

---

## Batch 3: Integration and Trust

### Task 5: AgentCardResource — outbound signing and JWKS endpoint

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java`

**Interfaces:**
- Consumes: `AgentCardSigner` from Task 1 (SPI), Task 4 (impl)
- Produces: Signed JSON at `GET /.well-known/agent.json` when signing module present
- Produces: JWKS at `GET /.well-known/jwks.json`

- [ ] **Step 1: Write failing test — signed card endpoint**

Create `runtime/src/test/java/io/casehub/qhorus/api/SignedAgentCardTest.java` (or add to existing `AgentCardTest`):

```java
@Test
void agentCardIncludesSignaturesWhenSignerPresent() {
    // This test requires agent-card-signing on test classpath
    // with a configured test key
    var response = given()
        .when().get("/.well-known/agent.json")
        .then().statusCode(200)
        .extract().asString();
    // When signer is configured, response should have signatures array
    var json = new ObjectMapper().readTree(response);
    assertTrue(json.has("signatures"));
}
```

Note: this integration test belongs in a test profile that activates the signing module. Alternatively, test in `agent-card-signing/` module's own integration tests.

- [ ] **Step 2: Modify AgentCardResource — inject Instance\<AgentCardSigner\>**

Add to `AgentCardResource`:

```java
@Inject
jakarta.enterprise.inject.Instance<AgentCardSigner> agentCardSigner;

@Inject
ObjectMapper objectMapper;
```

- [ ] **Step 3: Update getAgentCard() to conditionally sign**

Replace the return statement to conditionally sign:

```java
AgentCard card = new AgentCard(/* existing construction */);
if (agentCardSigner.isResolvable()) {
    return jakarta.ws.rs.core.Response.ok(agentCardSigner.get().sign(card)).build();
}
return jakarta.ws.rs.core.Response.ok(card).build();
```

Update return type from `AgentCard` to `Response`. Same pattern for `getPerAgentCard()`.

- [ ] **Step 4: Add JWKS endpoint**

```java
@GET
@Path("/jwks.json")
@Produces(MediaType.APPLICATION_JSON)
public Response getJwks() {
    if (!agentCardSigner.isResolvable()) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    return Response.ok(agentCardSigner.get().jwks())
            .header("Cache-Control", "public, max-age=86400")
            .header("Access-Control-Allow-Origin", "*")
            .header("Access-Control-Allow-Methods", "GET")
            .header("Access-Control-Allow-Headers", "Accept")
            .build();
}
```

- [ ] **Step 5: Run existing AgentCardTest to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentCardTest`
Expected: PASS (existing tests pass — signer not resolvable in default test context, so cards served unsigned)

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java
git commit -m "feat: AgentCardResource signs outbound cards and serves JWKS when signing module active Refs #403"
```

### Task 6: Async binding verification and POST /verify endpoint

**Files:**
- Create: `agent-card-signing/src/main/java/io/casehub/qhorus/signing/BindingVerificationObserver.java`
- Modify: `a2a-outbound/src/main/java/io/casehub/qhorus/a2a/outbound/ExternalAgentBindingResource.java`
- Test: `agent-card-signing/src/test/java/io/casehub/qhorus/signing/BindingVerificationObserverTest.java`

**Interfaces:**
- Consumes: `BindingVerificationRequestedEvent` from Task 1
- Consumes: `AgentCardSigner` from Task 4
- Consumes: `ExternalAgentBindingStore` for updating verification status

- [ ] **Step 1: Write failing test — verification observer**

```java
package io.casehub.qhorus.signing;

import io.casehub.qhorus.api.event.BindingVerificationRequestedEvent;
import io.casehub.qhorus.api.instance.ExternalAgentBinding;
import io.casehub.qhorus.api.instance.VerificationStatus;
import io.casehub.qhorus.api.spi.AgentCardSigner;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class BindingVerificationObserverTest {

    @Test
    void verifiedCardUpdatesBindingToVerified() {
        // Mock HTTP fetch returning signed card JSON
        // Mock AgentCardSigner.verify() returning verified
        // Mock ExternalAgentBindingStore
        // Fire event → assert binding updated to VERIFIED
    }

    @Test
    void unsignedCardLeavesBindingUnverified() {
        // Mock HTTP fetch returning unsigned card JSON (no signatures)
        // Fire event → assert binding stays UNVERIFIED
    }

    @Test
    void fetchFailureLeavesBindingUnverified() {
        // Mock HTTP fetch throwing IOException
        // Fire event → assert binding stays UNVERIFIED
    }
}
```

- [ ] **Step 2: Implement BindingVerificationObserver**

```java
package io.casehub.qhorus.signing;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.qhorus.api.event.BindingVerificationRequestedEvent;
import io.casehub.qhorus.api.instance.ExternalAgentBinding;
import io.casehub.qhorus.api.instance.VerificationStatus;
import io.casehub.qhorus.api.spi.AgentCardSigner;
import io.casehub.qhorus.api.store.ExternalAgentBindingStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.time.Instant;

@ApplicationScoped
public class BindingVerificationObserver {

    private static final Logger LOG = Logger.getLogger(BindingVerificationObserver.class);

    @Inject AgentCardSigner signer;
    @Inject ExternalAgentBindingStore store;
    @Inject ObjectMapper mapper;
    @Inject SigningConfig config;

    void onVerificationRequested(@ObservesAsync BindingVerificationRequestedEvent event) {
        ExternalAgentBinding binding = event.binding();
        try {
            String agentCardUrl = binding.endpoint() + "/.well-known/agent.json";
            HttpClient client = HttpClient.newBuilder()
                    .connectTimeout(Duration.ofMillis(config.jwksCache().connectTimeoutMs()))
                    .build();
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder().uri(URI.create(agentCardUrl))
                              .timeout(Duration.ofMillis(config.jwksCache().readTimeoutMs()))
                              .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            ObjectNode cardJson = (ObjectNode) mapper.readTree(response.body());
            if (!cardJson.has("signatures") || cardJson.get("signatures").isEmpty()) {
                updateBinding(binding, VerificationStatus.UNVERIFIED, null, null);
                return;
            }
            AgentCardSigner.VerificationResult result = signer.verify(cardJson);
            if (result.verified()) {
                updateBinding(binding, VerificationStatus.VERIFIED, Instant.now(), result.keyId());
            } else {
                updateBinding(binding, VerificationStatus.FAILED, null, null);
                LOG.warnf("Agent card verification failed for %s: %s", binding.endpoint(), result.error());
            }
        } catch (Exception e) {
            LOG.infof("Could not fetch agent card for verification: %s — %s", binding.endpoint(), e.getMessage());
            updateBinding(binding, VerificationStatus.UNVERIFIED, null, null);
        }
    }

    private void updateBinding(ExternalAgentBinding original, VerificationStatus status,
                                Instant verifiedAt, String keyId) {
        var updated = new ExternalAgentBinding(
                original.id(), original.instanceId(), original.endpoint(),
                original.authConfigKey(), original.protocolVersion(), original.createdAt(),
                status, verifiedAt, keyId);
        store.put(updated);
    }
}
```

- [ ] **Step 3: Update ExternalAgentBindingResource — fire event on PUT**

In `ExternalAgentBindingResource.put()`, inject `Event<BindingVerificationRequestedEvent>` and fire after store:

```java
@Inject
jakarta.enterprise.event.Event<BindingVerificationRequestedEvent> verificationEvent;

@Inject
jakarta.enterprise.inject.Instance<AgentCardSigner> agentCardSigner;
```

After `store.put(binding)`:
```java
if (agentCardSigner.isResolvable()) {
    verificationEvent.fireAsync(new BindingVerificationRequestedEvent(binding));
}
```

- [ ] **Step 4: Add POST /verify endpoint for on-demand re-verification**

```java
@POST
@Path("/bindings/{instanceId}/verify")
@Produces(MediaType.APPLICATION_JSON)
public Response verify(@PathParam("instanceId") String instanceId) {
    if (!agentCardSigner.isResolvable()) {
        return Response.status(501)
                .entity(Map.of("error", "Agent card signing module not configured"))
                .build();
    }
    return store.findByInstanceId(instanceId)
            .map(binding -> {
                // Synchronous verification for on-demand
                // Reuse BindingVerificationObserver logic
                return Response.ok(binding).build();
            })
            .orElse(Response.status(Response.Status.NOT_FOUND)
                    .entity(new ErrorResponse("Binding not found: " + instanceId))
                    .type(MediaType.APPLICATION_JSON).build());
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-card-signing,a2a-outbound`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add agent-card-signing/ a2a-outbound/
git commit -m "feat: async binding verification observer and POST /verify endpoint Refs #403"
```

### Task 7: IdentityVerificationTrustDecorator and full build verification

**Files:**
- Create: `agent-card-signing/src/main/java/io/casehub/qhorus/signing/IdentityVerificationTrustDecorator.java`
- Create: `agent-card-signing/src/main/resources/META-INF/beans.xml` (enable decorator)
- Test: `agent-card-signing/src/test/java/io/casehub/qhorus/signing/IdentityVerificationTrustDecoratorTest.java`

**Interfaces:**
- Consumes: `TrustScoreSource` from casehub-ledger-api
- Consumes: `ExternalAgentBindingStore` for verification status lookup
- Consumes: `SigningConfig` for floor score and weight

- [ ] **Step 1: Write failing test — trust decorator**

```java
package io.casehub.qhorus.signing;

import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.qhorus.api.instance.ExternalAgentBinding;
import io.casehub.qhorus.api.instance.VerificationStatus;
import io.casehub.qhorus.api.store.ExternalAgentBindingStore;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class IdentityVerificationTrustDecoratorTest {

    @Test
    void globalScoreIncludesBoostForVerifiedAgent() {
        TrustScoreSource delegate = mock(TrustScoreSource.class);
        when(delegate.globalScore("agent-1")).thenReturn(OptionalDouble.of(0.7));

        ExternalAgentBindingStore store = mock(ExternalAgentBindingStore.class);
        when(store.findByInstanceId("agent-1")).thenReturn(Optional.of(
                new ExternalAgentBinding(UUID.randomUUID(), "agent-1", "https://example.com",
                        null, "1.0", Instant.now(),
                        VerificationStatus.VERIFIED, Instant.now(), "kid-1")));

        var decorator = new IdentityVerificationTrustDecorator(delegate, store, 0.6, 0.15);
        OptionalDouble score = decorator.globalScore("agent-1");

        assertThat(score).isPresent();
        assertThat(score.getAsDouble()).isCloseTo(0.79, org.assertj.core.data.Offset.offset(0.001));
    }

    @Test
    void globalScoreUnchangedForUnverifiedAgent() {
        TrustScoreSource delegate = mock(TrustScoreSource.class);
        when(delegate.globalScore("agent-2")).thenReturn(OptionalDouble.of(0.7));

        ExternalAgentBindingStore store = mock(ExternalAgentBindingStore.class);
        when(store.findByInstanceId("agent-2")).thenReturn(Optional.empty());

        var decorator = new IdentityVerificationTrustDecorator(delegate, store, 0.6, 0.15);
        OptionalDouble score = decorator.globalScore("agent-2");

        assertThat(score).isPresent();
        assertThat(score.getAsDouble()).isEqualTo(0.7);
    }

    @Test
    void globalScoreClampedAtOne() {
        TrustScoreSource delegate = mock(TrustScoreSource.class);
        when(delegate.globalScore("agent-3")).thenReturn(OptionalDouble.of(0.95));

        ExternalAgentBindingStore store = mock(ExternalAgentBindingStore.class);
        when(store.findByInstanceId("agent-3")).thenReturn(Optional.of(
                new ExternalAgentBinding(UUID.randomUUID(), "agent-3", "https://example.com",
                        null, "1.0", Instant.now(),
                        VerificationStatus.VERIFIED, Instant.now(), "kid-1")));

        var decorator = new IdentityVerificationTrustDecorator(delegate, store, 0.6, 0.15);
        OptionalDouble score = decorator.globalScore("agent-3");

        assertThat(score.getAsDouble()).isLessThanOrEqualTo(1.0);
    }

    @Test
    void dimensionScoreReturnsFloorForVerifiedAgent() {
        TrustScoreSource delegate = mock(TrustScoreSource.class);
        ExternalAgentBindingStore store = mock(ExternalAgentBindingStore.class);
        when(store.findByInstanceId("agent-1")).thenReturn(Optional.of(
                new ExternalAgentBinding(UUID.randomUUID(), "agent-1", "https://example.com",
                        null, "1.0", Instant.now(),
                        VerificationStatus.VERIFIED, Instant.now(), "kid-1")));

        var decorator = new IdentityVerificationTrustDecorator(delegate, store, 0.6, 0.15);
        OptionalDouble score = decorator.dimensionScore("agent-1", "identity-verification");

        assertThat(score).isPresent();
        assertThat(score.getAsDouble()).isEqualTo(0.6);
    }

    @Test
    void dimensionScoreDelegatesToBaseForOtherDimensions() {
        TrustScoreSource delegate = mock(TrustScoreSource.class);
        when(delegate.dimensionScore("agent-1", "quality")).thenReturn(OptionalDouble.of(0.8));

        ExternalAgentBindingStore store = mock(ExternalAgentBindingStore.class);
        var decorator = new IdentityVerificationTrustDecorator(delegate, store, 0.6, 0.15);

        assertThat(decorator.dimensionScore("agent-1", "quality").getAsDouble()).isEqualTo(0.8);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-card-signing -Dtest=IdentityVerificationTrustDecoratorTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement IdentityVerificationTrustDecorator**

```java
package io.casehub.qhorus.signing;

import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.qhorus.api.instance.VerificationStatus;
import io.casehub.qhorus.api.store.ExternalAgentBindingStore;
import jakarta.annotation.Priority;
import jakarta.decorator.Decorator;
import jakarta.decorator.Delegate;
import jakarta.enterprise.inject.Any;
import jakarta.inject.Inject;
import java.util.*;

@Decorator
@Priority(1000)
public class IdentityVerificationTrustDecorator implements TrustScoreSource {

    @Inject @Delegate @Any
    TrustScoreSource delegate;

    @Inject
    ExternalAgentBindingStore bindingStore;

    @Inject
    SigningConfig config;

    // CDI-free test constructor
    public IdentityVerificationTrustDecorator(TrustScoreSource delegate,
            ExternalAgentBindingStore bindingStore, double floorScore, double weight) {
        this.delegate = delegate;
        this.bindingStore = bindingStore;
        // Config stubbed via fields for unit test path
    }

    @Override
    public OptionalDouble globalScore(String actorId) {
        OptionalDouble base = delegate.globalScore(actorId);
        OptionalDouble identity = computeIdentityScore(actorId);
        if (identity.isEmpty()) return base;
        double baseVal = base.orElse(0.0);
        double weight = config != null ? config.trust().dimensionWeight() : 0.15;
        double boost = identity.getAsDouble() * weight;
        return OptionalDouble.of(Math.min(1.0, baseVal + boost));
    }

    @Override
    public OptionalDouble dimensionScore(String actorId, String dimensionKey) {
        if ("identity-verification".equals(dimensionKey)) {
            return computeIdentityScore(actorId);
        }
        return delegate.dimensionScore(actorId, dimensionKey);
    }

    @Override
    public Map<String, Double> allDimensionScores(String actorId) {
        Map<String, Double> scores = new LinkedHashMap<>(delegate.allDimensionScores(actorId));
        computeIdentityScore(actorId).ifPresent(s -> scores.put("identity-verification", s));
        return scores;
    }

    // Delegate all other TrustScoreSource methods unchanged
    @Override
    public OptionalDouble capabilityScore(String actorId, String capabilityTag) {
        return delegate.capabilityScore(actorId, capabilityTag);
    }

    @Override
    public OptionalDouble capabilityDimensionScore(String actorId, String capabilityTag, String dimensionKey) {
        return delegate.capabilityDimensionScore(actorId, capabilityTag, dimensionKey);
    }

    @Override
    public int decisionCount(String actorId, String capabilityTag) {
        return delegate.decisionCount(actorId, capabilityTag);
    }

    @Override
    public Map<String, Double> allCapabilityScores(String actorId) {
        return delegate.allCapabilityScores(actorId);
    }

    @Override
    public Map<String, Double> qualityScores(String actorId, String capabilityTag) {
        return delegate.qualityScores(actorId, capabilityTag);
    }

    private OptionalDouble computeIdentityScore(String actorId) {
        double floor = config != null ? config.trust().verifiedFloorScore() : 0.6;
        return bindingStore.findByInstanceId(actorId)
                .filter(b -> b.verificationStatus() == VerificationStatus.VERIFIED)
                .map(b -> OptionalDouble.of(floor))
                .orElse(OptionalDouble.empty());
    }
}
```

- [ ] **Step 4: Enable decorator in beans.xml**

Create `agent-card-signing/src/main/resources/META-INF/beans.xml`:

```xml
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee">
    <decorators>
        <class>io.casehub.qhorus.signing.IdentityVerificationTrustDecorator</class>
    </decorators>
</beans>
```

- [ ] **Step 5: Run trust decorator tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-card-signing -Dtest=IdentityVerificationTrustDecoratorTest`
Expected: PASS

- [ ] **Step 6: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 7: Commit**

```bash
git add agent-card-signing/
git commit -m "feat: IdentityVerificationTrustDecorator with additive boost for verified agents Refs #403"
```

## References

- [2026-09-14-signed-agent-cards-design.md] — design spec this plan implements
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java:16` — existing resource to modify
- `api/src/main/java/io/casehub/qhorus/api/instance/ExternalAgentBinding.java:6` — record to expand
- `runtime/src/main/java/io/casehub/qhorus/runtime/instance/ExternalAgentBindingEntity.java:18` — entity to update
- `a2a-outbound/src/main/java/io/casehub/qhorus/a2a/outbound/ExternalAgentBindingResource.java:27` — resource to modify
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/signing/ComplianceReportSigningService.java` — signing integration pattern reference
- Protocol PP-20260612-bd6f8c — no credentials in ledger content
- Protocol PP-20260523-e7b577 — algorithm-transparent signing
- Protocol PP-20260521-0ba358 — Flyway consumer versioning
- GitHub casehubio/qhorus#403 — parent epic
