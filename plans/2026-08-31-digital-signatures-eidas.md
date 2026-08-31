# Digital Signatures (eIDAS Qualified Seals) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #418 — feat: digital signatures (eIDAS qualified seals) for compliance reports
**Issue group:** #418

**Goal:** Add PAdES PDF signing, CAdES detached signatures, and a verification endpoint to the compliance evidence export pipeline, backed by a new platform-level document signing SPI.

**Architecture:** Two repos — `casehub-platform` gains a `DocumentSigningService` SPI in `platform-api` and an EU DSS implementation in a new `platform-signing` module. `casehub-qhorus` gains binary SharedData support, a compliance signing service, pipeline integration (scheduler + REST + storage), and a verification endpoint.

**Tech Stack:** EU DSS 6.x (dss-pades-pdfbox, dss-cades, dss-service), Apache PDFBox 3.x (already in dep graph), BouncyCastle (already in dep graph), RFC 3161 TSA, PKCS#12 keystores

## Global Constraints

- Java 21 on Java 26 JVM, Quarkus 3.32.2
- `casehub-platform-api` version `0.2-SNAPSHOT`
- EU DSS 6.x (latest stable) — verify PDFBox 3.x alignment with OpenHTMLtoPDF
- All NoOp @DefaultBean SPIs return Optional.empty() — never throw
- PKCS#12 keystore passwords resolved via `CredentialResolver`, never in config files
- Profile enforcement: B_T configured + TSA unavailable = fail (no silent downgrade)
- Signing automatic when configured, unsigned pass-through when NoOp
- Flyway domain migrations: V50 (qhorus range)
- Platform repo: `/Users/mdproctor/claude/casehub/platform`
- Qhorus repo: `/Users/mdproctor/claude/casehub/qhorus`
- Use `mcp__intellij-index__*` tools for code navigation and editing, not bash grep

---

## Batch 1: Platform SPI + NoOp Defaults

**Repo:** `casehub-platform`
**After this batch:** Platform-api exposes DocumentSigningService and DocumentVerificationService SPIs with NoOp defaults. Any downstream module can depend on platform-api and inject the SPIs — NoOp returns Optional.empty() when no implementation is deployed.

### Task 1: Document Signing SPI types + NoOp defaults

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/DocumentSigningService.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/DocumentVerificationService.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/SigningIdentity.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/SignedDocument.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/DetachedSignature.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/SigningProfile.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/DocumentVerificationResult.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/VerificationStatus.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/signing/document/CertificateInfo.java`
- Create: `platform/src/main/java/io/casehub/platform/signing/document/NoOpDocumentSigningService.java`
- Create: `platform/src/main/java/io/casehub/platform/signing/document/NoOpDocumentVerificationService.java`
- Test: `platform-api/src/test/java/io/casehub/platform/api/signing/document/DocumentSigningSpiTest.java`

**Interfaces:**
- Produces: `DocumentSigningService.signPdf(byte[], SigningIdentity) → Optional<SignedDocument>`, `DocumentSigningService.signDetached(byte[], SigningIdentity) → Optional<DetachedSignature>`, `DocumentVerificationService.verifyPdf(byte[]) → DocumentVerificationResult`, `DocumentVerificationService.verifyDetached(byte[], byte[]) → DocumentVerificationResult`

- [ ] **Step 1: Write tests for NoOp defaults and SPI contracts**

```java
package io.casehub.platform.api.signing.document;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class DocumentSigningSpiTest {

    @Test
    void signingIdentity_carriesActorAndTenant() {
        var id = new SigningIdentity("system:compliance-signer", "tenant-acme");
        assertThat(id.actorId()).isEqualTo("system:compliance-signer");
        assertThat(id.tenancyId()).isEqualTo("tenant-acme");
    }

    @Test
    void signingProfile_allLevels() {
        assertThat(SigningProfile.values()).containsExactly(
                SigningProfile.B_B, SigningProfile.B_T,
                SigningProfile.B_LT, SigningProfile.B_LTA);
    }

    @Test
    void signingProfile_requiresTimestamp() {
        assertThat(SigningProfile.B_B.requiresTimestamp()).isFalse();
        assertThat(SigningProfile.B_T.requiresTimestamp()).isTrue();
        assertThat(SigningProfile.B_LT.requiresTimestamp()).isTrue();
        assertThat(SigningProfile.B_LTA.requiresTimestamp()).isTrue();
    }

    @Test
    void signedDocument_defensiveCopy() {
        byte[] original = {1, 2, 3};
        var doc = new SignedDocument(original, "CN=Test", java.time.Instant.now(),
                "keyRef123", SigningProfile.B_T);
        original[0] = 99;
        assertThat(doc.signedBytes()[0]).isEqualTo((byte) 1);
    }

    @Test
    void detachedSignature_defensiveCopy() {
        byte[] original = {4, 5, 6};
        var sig = new DetachedSignature(original, "CN=Test", java.time.Instant.now(),
                "keyRef456", SigningProfile.B_T);
        original[0] = 99;
        assertThat(sig.signatureBytes()[0]).isEqualTo((byte) 4);
    }

    @Test
    void verificationStatus_allValues() {
        assertThat(VerificationStatus.values()).containsExactly(
                VerificationStatus.VALID, VerificationStatus.INVALID,
                VerificationStatus.UNSIGNED, VerificationStatus.UNSUPPORTED_FORMAT,
                VerificationStatus.ERROR);
    }

    @Test
    void certificateInfo_claimsQualified() {
        var cert = new CertificateInfo("CN=Seal", "CN=CA",
                java.time.Instant.now(), java.time.Instant.now().plusSeconds(86400), true);
        assertThat(cert.claimsQualified()).isTrue();
    }
}
```

NoOp tests (separate file in platform module):
```java
package io.casehub.platform.signing.document;

import io.casehub.platform.api.signing.document.*;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class NoOpDocumentSigningServiceTest {

    private final NoOpDocumentSigningService service = new NoOpDocumentSigningService();

    @Test
    void signPdf_returnsEmpty() {
        var result = service.signPdf(new byte[]{1}, new SigningIdentity("a", "t"));
        assertThat(result).isEmpty();
    }

    @Test
    void signDetached_returnsEmpty() {
        var result = service.signDetached(new byte[]{1}, new SigningIdentity("a", "t"));
        assertThat(result).isEmpty();
    }
}

class NoOpDocumentVerificationServiceTest {

    private final NoOpDocumentVerificationService service = new NoOpDocumentVerificationService();

    @Test
    void verifyPdf_returnsUnsigned() {
        var result = service.verifyPdf(new byte[]{1});
        assertThat(result.status()).isEqualTo(VerificationStatus.UNSIGNED);
    }

    @Test
    void verifyDetached_returnsUnsigned() {
        var result = service.verifyDetached(new byte[]{1}, new byte[]{2});
        assertThat(result.status()).isEqualTo(VerificationStatus.UNSIGNED);
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/platform-api/pom.xml -Dtest=DocumentSigningSpiTest -pl platform-api
```
Expected: compilation failure — types don't exist yet.

- [ ] **Step 3: Implement SPI types**

`SigningProfile.java`:
```java
package io.casehub.platform.api.signing.document;

public enum SigningProfile {
    B_B, B_T, B_LT, B_LTA;

    public boolean requiresTimestamp() {
        return this != B_B;
    }
}
```

`SigningIdentity.java`:
```java
package io.casehub.platform.api.signing.document;

public record SigningIdentity(String actorId, String tenancyId) {}
```

`SignedDocument.java`:
```java
package io.casehub.platform.api.signing.document;

import java.time.Instant;
import java.util.Arrays;

public record SignedDocument(byte[] signedBytes, String signerDn, Instant signedAt,
                             String keyRef, SigningProfile profile) {
    public SignedDocument {
        signedBytes = Arrays.copyOf(signedBytes, signedBytes.length);
    }
    @Override
    public byte[] signedBytes() {
        return Arrays.copyOf(signedBytes, signedBytes.length);
    }
}
```

`DetachedSignature.java`:
```java
package io.casehub.platform.api.signing.document;

import java.time.Instant;
import java.util.Arrays;

public record DetachedSignature(byte[] signatureBytes, String signerDn, Instant signedAt,
                                 String keyRef, SigningProfile profile) {
    public DetachedSignature {
        signatureBytes = Arrays.copyOf(signatureBytes, signatureBytes.length);
    }
    @Override
    public byte[] signatureBytes() {
        return Arrays.copyOf(signatureBytes, signatureBytes.length);
    }
}
```

`VerificationStatus.java`:
```java
package io.casehub.platform.api.signing.document;

public enum VerificationStatus {
    VALID, INVALID, UNSIGNED, UNSUPPORTED_FORMAT, ERROR
}
```

`CertificateInfo.java`:
```java
package io.casehub.platform.api.signing.document;

import java.time.Instant;

public record CertificateInfo(String subjectDn, String issuerDn,
                               Instant validFrom, Instant validTo,
                               boolean claimsQualified) {}
```

`DocumentVerificationResult.java`:
```java
package io.casehub.platform.api.signing.document;

import java.time.Instant;
import java.util.List;

public record DocumentVerificationResult(
        VerificationStatus status, String signerDn, Instant signedAt,
        String keyRef, SigningProfile detectedProfile,
        List<CertificateInfo> certificateChain, String diagnosticMessage) {

    public static DocumentVerificationResult unsigned() {
        return new DocumentVerificationResult(
                VerificationStatus.UNSIGNED, null, null, null, null, List.of(), null);
    }
}
```

`DocumentSigningService.java`:
```java
package io.casehub.platform.api.signing.document;

import java.util.Optional;

public interface DocumentSigningService {
    Optional<SignedDocument> signPdf(byte[] pdfBytes, SigningIdentity identity);
    Optional<DetachedSignature> signDetached(byte[] data, SigningIdentity identity);
}
```

`DocumentVerificationService.java`:
```java
package io.casehub.platform.api.signing.document;

public interface DocumentVerificationService {
    DocumentVerificationResult verifyPdf(byte[] pdfBytes);
    DocumentVerificationResult verifyDetached(byte[] data, byte[] signature);
}
```

`NoOpDocumentSigningService.java` (in `platform/` module):
```java
package io.casehub.platform.signing.document;

import io.casehub.platform.api.signing.document.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class NoOpDocumentSigningService implements DocumentSigningService {
    @Override
    public Optional<SignedDocument> signPdf(byte[] pdfBytes, SigningIdentity identity) {
        return Optional.empty();
    }
    @Override
    public Optional<DetachedSignature> signDetached(byte[] data, SigningIdentity identity) {
        return Optional.empty();
    }
}
```

`NoOpDocumentVerificationService.java` (in `platform/` module):
```java
package io.casehub.platform.signing.document;

import io.casehub.platform.api.signing.document.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpDocumentVerificationService implements DocumentVerificationService {
    @Override
    public DocumentVerificationResult verifyPdf(byte[] pdfBytes) {
        return DocumentVerificationResult.unsigned();
    }
    @Override
    public DocumentVerificationResult verifyDetached(byte[] data, byte[] signature) {
        return DocumentVerificationResult.unsigned();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/pom.xml -Dtest="DocumentSigningSpiTest,NoOpDocumentSigningServiceTest,NoOpDocumentVerificationServiceTest"
```
Expected: all PASS.

- [ ] **Step 5: Install platform SNAPSHOT**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/platform/pom.xml -DskipTests
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform-api/src/main/java/io/casehub/platform/api/signing/document/ platform/src/main/java/io/casehub/platform/signing/document/ platform-api/src/test/java/io/casehub/platform/api/signing/document/ platform/src/test/java/io/casehub/platform/signing/document/
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#418): DocumentSigningService + DocumentVerificationService SPI with NoOp defaults

Refs casehubio/qhorus#418"
```

---

## Batch 2: Platform DSS Implementation

**Repo:** `casehub-platform`
**After this batch:** A new `casehub-platform-signing` module provides EU DSS-backed PAdES and CAdES signing with PKCS#12 keystore management and RFC 3161 timestamping.

### Task 2: casehub-platform-signing module — KeyStoreManager + DssDocumentSigningService

**Files:**
- Create: `platform-signing/pom.xml`
- Create: `platform-signing/src/main/java/io/casehub/platform/signing/document/DssSigningConfig.java`
- Create: `platform-signing/src/main/java/io/casehub/platform/signing/document/KeyStoreManager.java`
- Create: `platform-signing/src/main/java/io/casehub/platform/signing/document/DssDocumentSigningService.java`
- Modify: `pom.xml` (root — add `platform-signing` module)
- Test: `platform-signing/src/test/java/io/casehub/platform/signing/document/KeyStoreManagerTest.java`
- Test: `platform-signing/src/test/java/io/casehub/platform/signing/document/DssDocumentSigningServiceTest.java`

**Interfaces:**
- Consumes: `DocumentSigningService` (from Task 1), `SigningProfile`, `SignedDocument`, `DetachedSignature`, `SigningIdentity`
- Produces: `DssDocumentSigningService` (CDI bean displacing `NoOpDocumentSigningService`), `KeyStoreManager.toDssKeyEntry()`, `KeyStoreManager.isLoaded()`

- [ ] **Step 1: Create `platform-signing/pom.xml`**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>
  <artifactId>casehub-platform-signing</artifactId>
  <name>CaseHub Platform - Document Signing (EU DSS)</name>

  <properties>
    <dss.version>6.2</dss.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>eu.europa.ec.joinup.sd-dss</groupId>
      <artifactId>dss-pades-pdfbox</artifactId>
      <version>${dss.version}</version>
    </dependency>
    <dependency>
      <groupId>eu.europa.ec.joinup.sd-dss</groupId>
      <artifactId>dss-cades</artifactId>
      <version>${dss.version}</version>
    </dependency>
    <dependency>
      <groupId>eu.europa.ec.joinup.sd-dss</groupId>
      <artifactId>dss-service</artifactId>
      <version>${dss.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.bouncycastle</groupId>
      <artifactId>bcpkix-jdk18on</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

Add `<module>platform-signing</module>` to the root pom.xml `<modules>` section.

**Verify PDFBox version:** After adding the pom, run `mvn dependency:tree -f platform-signing/pom.xml | grep pdfbox` to confirm DSS pulls PDFBox 3.x (same major version as OpenHTMLtoPDF). If a conflict exists, add `<exclusion>` and force the correct version via `<dependencyManagement>`. This is a gate — do not proceed if versions diverge.

- [ ] **Step 2: Write KeyStoreManager tests**

```java
package io.casehub.platform.signing.document;

import org.bouncycastle.asn1.x500.X500Name;
import org.bouncycastle.cert.X509v3CertificateBuilder;
import org.bouncycastle.cert.jcajce.JcaX509CertificateConverter;
import org.bouncycastle.cert.jcajce.JcaX509v3CertificateBuilder;
import org.bouncycastle.operator.ContentSigner;
import org.bouncycastle.operator.jcajce.JcaContentSignerBuilder;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.FileOutputStream;
import java.math.BigInteger;
import java.nio.file.Path;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.KeyStore;
import java.security.cert.X509Certificate;
import java.time.Instant;
import java.util.Date;

import static org.assertj.core.api.Assertions.*;

class KeyStoreManagerTest {

    private static Path keystorePath;
    private static final String ALIAS = "test-seal";
    private static final String PASSWORD = "changeit";

    @TempDir
    static Path tempDir;

    @BeforeAll
    static void createTestKeystore() throws Exception {
        KeyPairGenerator kpg = KeyPairGenerator.getInstance("EC");
        kpg.initialize(256);
        KeyPair kp = kpg.generateKeyPair();

        X500Name dn = new X500Name("CN=Test Seal, O=CaseHub, C=IE");
        ContentSigner signer = new JcaContentSignerBuilder("SHA256withECDSA").build(kp.getPrivate());
        X509v3CertificateBuilder builder = new JcaX509v3CertificateBuilder(
                dn, BigInteger.ONE,
                Date.from(Instant.now().minusSeconds(3600)),
                Date.from(Instant.now().plusSeconds(86400 * 365)),
                dn, kp.getPublic());
        X509Certificate cert = new JcaX509CertificateConverter().getCertificate(builder.build(signer));

        KeyStore ks = KeyStore.getInstance("PKCS12");
        ks.load(null, PASSWORD.toCharArray());
        ks.setKeyEntry(ALIAS, kp.getPrivate(), PASSWORD.toCharArray(), new X509Certificate[]{cert});

        keystorePath = tempDir.resolve("test.p12");
        try (FileOutputStream fos = new FileOutputStream(keystorePath.toFile())) {
            ks.store(fos, PASSWORD.toCharArray());
        }
    }

    @Test
    void load_extractsPrivateKeyAndChain() {
        var mgr = new KeyStoreManager(keystorePath.toString(), PASSWORD, "PKCS12", ALIAS);
        assertThat(mgr.isLoaded()).isTrue();
        assertThat(mgr.toDssKeyEntry()).isNotNull();
        assertThat(mgr.toDssKeyEntry().getCertificate().getSubjectX500Principal().getName())
                .contains("CN=Test Seal");
    }

    @Test
    void load_missingPath_notLoaded() {
        var mgr = new KeyStoreManager(null, null, "PKCS12", null);
        assertThat(mgr.isLoaded()).isFalse();
        assertThat(mgr.toDssKeyEntry()).isNull();
    }

    @Test
    void load_wrongPassword_throws() {
        assertThatThrownBy(() -> new KeyStoreManager(keystorePath.toString(), "wrong", "PKCS12", ALIAS))
                .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void tenantAlias_fallsBackToDefault() {
        var mgr = new KeyStoreManager(keystorePath.toString(), PASSWORD, "PKCS12", ALIAS);
        // tenant-specific alias doesn't exist — falls back to ALIAS
        var entry = mgr.resolveKeyEntry("nonexistent-tenant");
        assertThat(entry).isNotNull();
        assertThat(entry.getCertificate().getSubjectX500Principal().getName())
                .contains("CN=Test Seal");
    }
}
```

- [ ] **Step 3: Implement KeyStoreManager**

```java
package io.casehub.platform.signing.document;

import eu.europa.ec.markt.dss.token.DSSPrivateKeyEntry;
import eu.europa.ec.markt.dss.token.KSPrivateKeyEntry;
import org.jboss.logging.Logger;

import java.io.FileInputStream;
import java.security.KeyStore;
import java.security.PrivateKey;
import java.security.cert.X509Certificate;
import java.util.Arrays;

class KeyStoreManager {

    private static final Logger LOG = Logger.getLogger(KeyStoreManager.class);

    private final boolean loaded;
    private final KeyStore keyStore;
    private final String defaultAlias;

    KeyStoreManager(String path, String password, String type, String alias) {
        if (path == null || path.isBlank()) {
            LOG.warn("No keystore path configured — document signing disabled");
            this.loaded = false;
            this.keyStore = null;
            this.defaultAlias = null;
            return;
        }
        try {
            KeyStore ks = KeyStore.getInstance(type != null ? type : "PKCS12");
            try (var fis = new FileInputStream(path)) {
                ks.load(fis, password != null ? password.toCharArray() : null);
            }
            this.keyStore = ks;
            this.defaultAlias = alias != null ? alias : ks.aliases().nextElement();
            this.loaded = true;
        } catch (Exception e) {
            throw new IllegalStateException("Failed to load keystore from " + path, e);
        }
    }

    boolean isLoaded() { return loaded; }

    DSSPrivateKeyEntry toDssKeyEntry() {
        if (!loaded) return null;
        return resolveKeyEntry(null);
    }

    DSSPrivateKeyEntry resolveKeyEntry(String tenancyId) {
        if (!loaded) return null;
        String alias = tenancyId != null ? tenancyId + "-seal" : defaultAlias;
        try {
            if (!keyStore.containsAlias(alias)) {
                alias = defaultAlias;
            }
            PrivateKey pk = (PrivateKey) keyStore.getKey(alias, null);
            X509Certificate[] chain = Arrays.stream(keyStore.getCertificateChain(alias))
                    .map(X509Certificate.class::cast).toArray(X509Certificate[]::new);
            return new KSPrivateKeyEntry(alias,
                    new KeyStore.PrivateKeyEntry(pk, chain));
        } catch (Exception e) {
            throw new IllegalStateException("Failed to resolve key entry for alias " + alias, e);
        }
    }
}
```

Note: The exact DSS API for `KSPrivateKeyEntry` and `DSSPrivateKeyEntry` must be verified against the DSS 6.x API. The class names may differ — check `eu.europa.esig.dss.token` package. Adjust imports accordingly during implementation.

- [ ] **Step 4: Run KeyStoreManager tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/platform-signing/pom.xml -Dtest=KeyStoreManagerTest
```
Expected: all PASS.

- [ ] **Step 5: Write DssDocumentSigningService tests**

```java
package io.casehub.platform.signing.document;

import io.casehub.platform.api.signing.document.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;

import static org.assertj.core.api.Assertions.*;

class DssDocumentSigningServiceTest {

    static KeyStoreManager keyStoreManager;

    @TempDir
    static Path tempDir;

    @BeforeAll
    static void setup() throws Exception {
        // Same PKCS#12 generation as KeyStoreManagerTest
        // ... (reuse test helper)
        keyStoreManager = new KeyStoreManager(
                keystorePath.toString(), "changeit", "PKCS12", "test-seal");
    }

    @Test
    void signPdf_producesSignedBytes() {
        var service = createService(SigningProfile.B_B, null);
        byte[] unsignedPdf = createMinimalPdf();
        var result = service.signPdf(unsignedPdf, new SigningIdentity("actor", "tenant"));
        assertThat(result).isPresent();
        assertThat(result.get().signedBytes()).hasSizeGreaterThan(unsignedPdf.length);
        assertThat(result.get().signerDn()).contains("CN=Test Seal");
        assertThat(result.get().profile()).isEqualTo(SigningProfile.B_B);
    }

    @Test
    void signDetached_producesCadesSignature() {
        var service = createService(SigningProfile.B_B, null);
        byte[] data = "test report content".getBytes();
        var result = service.signDetached(data, new SigningIdentity("actor", "tenant"));
        assertThat(result).isPresent();
        assertThat(result.get().signatureBytes()).isNotEmpty();
        assertThat(result.get().signerDn()).contains("CN=Test Seal");
    }

    @Test
    void signPdf_btProfile_noTsa_throws() {
        var service = createService(SigningProfile.B_T, null);
        byte[] pdf = createMinimalPdf();
        assertThatThrownBy(() ->
                service.signPdf(pdf, new SigningIdentity("actor", "tenant")))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("TSA unavailable");
    }

    @Test
    void signPdf_notLoaded_returnsEmpty() {
        var emptyMgr = new KeyStoreManager(null, null, "PKCS12", null);
        var service = new DssDocumentSigningService(emptyMgr, SigningProfile.B_B, null, null);
        var result = service.signPdf(new byte[]{1}, new SigningIdentity("a", "t"));
        assertThat(result).isEmpty();
    }

    private DssDocumentSigningService createService(SigningProfile profile, String tsaUrl) {
        return new DssDocumentSigningService(keyStoreManager, profile, tsaUrl, null);
    }

    private byte[] createMinimalPdf() {
        // Generate a minimal valid PDF using PDFBox
        // PDDocument doc = new PDDocument(); doc.addPage(new PDPage()); ByteArrayOutputStream os = ...
        // doc.save(os); doc.close(); return os.toByteArray();
    }
}
```

- [ ] **Step 6: Implement DssDocumentSigningService**

Implement `DssDocumentSigningService` following the spec's pseudocode. Key points:
- Constructor takes `KeyStoreManager`, `SigningProfile`, `tsaUrl`, `tsaTimeout`
- `signPdf()`: create `PAdESSignatureParameters`, set level via `toSignatureLevel()`, configure TSP source if B_T+, use `PAdESService` to sign
- `signDetached()`: create `CAdESSignatureParameters` with `DETACHED` packaging, use `CAdESService`
- Check `keyStoreManager.isLoaded()` first — return `Optional.empty()` if not loaded
- Strict enforcement: B_T+ without TSA → `IllegalStateException`

- [ ] **Step 7: Run tests and commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/platform-signing/pom.xml
```
Expected: all PASS.

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform-signing/ pom.xml
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#418): casehub-platform-signing — EU DSS PAdES + CAdES implementation

Refs casehubio/qhorus#418"
```

### Task 3: DssDocumentVerificationService

**Files:**
- Create: `platform-signing/src/main/java/io/casehub/platform/signing/document/DssDocumentVerificationService.java`
- Test: `platform-signing/src/test/java/io/casehub/platform/signing/document/DssDocumentVerificationServiceTest.java`

**Interfaces:**
- Consumes: `DocumentVerificationService` (from Task 1), `DocumentVerificationResult`, `VerificationStatus`, `CertificateInfo`
- Produces: `DssDocumentVerificationService` (CDI bean displacing `NoOpDocumentVerificationService`)

- [ ] **Step 1: Write sign-then-verify round-trip tests**

```java
package io.casehub.platform.signing.document;

import io.casehub.platform.api.signing.document.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class DssDocumentVerificationServiceTest {

    static DssDocumentSigningService signer;
    static DssDocumentVerificationService verifier;

    @BeforeAll
    static void setup() throws Exception {
        // Reuse PKCS#12 setup from Task 2
        signer = new DssDocumentSigningService(keyStoreManager, SigningProfile.B_B, null, null);
        verifier = new DssDocumentVerificationService();
    }

    @Test
    void verifyPdf_signedPdf_returnsValid() {
        byte[] pdf = createMinimalPdf();
        var signed = signer.signPdf(pdf, new SigningIdentity("a", "t")).orElseThrow();
        var result = verifier.verifyPdf(signed.signedBytes());
        assertThat(result.status()).isEqualTo(VerificationStatus.VALID);
        assertThat(result.signerDn()).contains("CN=Test Seal");
        assertThat(result.certificateChain()).isNotEmpty();
    }

    @Test
    void verifyPdf_unsignedPdf_returnsUnsigned() {
        byte[] pdf = createMinimalPdf();
        var result = verifier.verifyPdf(pdf);
        assertThat(result.status()).isEqualTo(VerificationStatus.UNSIGNED);
    }

    @Test
    void verifyPdf_tamperedPdf_returnsInvalid() {
        byte[] pdf = createMinimalPdf();
        var signed = signer.signPdf(pdf, new SigningIdentity("a", "t")).orElseThrow();
        byte[] tampered = signed.signedBytes().clone();
        tampered[tampered.length - 100] ^= 0xFF;
        var result = verifier.verifyPdf(tampered);
        assertThat(result.status()).isIn(VerificationStatus.INVALID, VerificationStatus.ERROR);
    }

    @Test
    void verifyDetached_validSignature_returnsValid() {
        byte[] data = "test content".getBytes();
        var sig = signer.signDetached(data, new SigningIdentity("a", "t")).orElseThrow();
        var result = verifier.verifyDetached(data, sig.signatureBytes());
        assertThat(result.status()).isEqualTo(VerificationStatus.VALID);
    }

    @Test
    void verifyDetached_tamperedData_returnsInvalid() {
        byte[] data = "test content".getBytes();
        var sig = signer.signDetached(data, new SigningIdentity("a", "t")).orElseThrow();
        var result = verifier.verifyDetached("modified content".getBytes(), sig.signatureBytes());
        assertThat(result.status()).isEqualTo(VerificationStatus.INVALID);
    }
}
```

- [ ] **Step 2: Implement DssDocumentVerificationService**

Use DSS's `SignedDocumentValidator` — it handles format detection, signature extraction, and verification internally. Map DSS's `Reports` to `DocumentVerificationResult`. Extract `CertificateInfo` from the certificate chain in the validation result.

- [ ] **Step 3: Run tests, install SNAPSHOT, commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/platform-signing/pom.xml -Dtest=DssDocumentVerificationServiceTest
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/platform/pom.xml -DskipTests
git -C /Users/mdproctor/claude/casehub/platform add platform-signing/
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#418): DssDocumentVerificationService — sign-then-verify round-trip

Refs casehubio/qhorus#418"
```

---

## Batch 3: Qhorus Foundation

**Repo:** `casehub-qhorus`
**After this batch:** SharedData supports binary content. ComplianceReportSigningService routes signing requests by format. ReportRenderingService extracts renderer selection from the REST resource.

### Task 4: SharedData binary support + V50 migration

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/data/SharedData.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/data/SharedDataEntity.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/data/DataService.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/DataStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/data/JpaDataStore.java` (if exists)
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryDataStore.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V50__shared_data_binary_and_signature_columns.sql`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/storage/ComplianceReportRecord.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/data/SharedDataBinaryTest.java`

**Interfaces:**
- Consumes: `SharedData` record (existing), `DataStore` interface (existing)
- Produces: `SharedData.binaryContent()`, `DataService.storeBinary(key, description, createdBy, byte[], lastChunk)`, `ComplianceReportRecord.signatureStatus`, `.signedAt`, `.signerDn`, `.keyRef`, `.signingProfile`, `.signatureArtefactId`

- [ ] **Step 1: Write test for binary SharedData round-trip**

```java
package io.casehub.qhorus.data;

import io.casehub.qhorus.api.data.SharedData;
import io.casehub.qhorus.runtime.data.DataService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class SharedDataBinaryTest {

    @Inject DataService dataService;

    @Test
    @Transactional
    void storeBinary_roundTrip() {
        byte[] content = new byte[]{1, 2, 3, 4, 5};
        var stored = dataService.storeBinary("test-binary-key", "Binary test",
                "test-actor", content, true);
        assertThat(stored.binaryContent()).isEqualTo(content);
        assertThat(stored.content()).isNull();
        assertThat(stored.sizeBytes()).isEqualTo(5);
        assertThat(stored.complete()).isTrue();

        var retrieved = dataService.getByKey("test-binary-key").orElseThrow();
        assertThat(retrieved.binaryContent()).isEqualTo(content);
        assertThat(retrieved.content()).isNull();
    }

    @Test
    @Transactional
    void storeText_binaryContentIsNull() {
        var stored = dataService.store("test-text-key", "Text test",
                "test-actor", "hello", false, true);
        assertThat(stored.content()).isEqualTo("hello");
        assertThat(stored.binaryContent()).isNull();
    }
}
```

- [ ] **Step 2: Add `binaryContent` to SharedData record**

Add `byte[] binaryContent` field after `content` in the record. Update the Builder with `binaryContent(byte[])` method. Existing 9-arg constructor remains backward-compatible by adding a 10-arg canonical constructor that defaults `binaryContent` to `null`, then delegating from a 9-arg convenience constructor.

- [ ] **Step 3: Update SharedDataEntity**

Add `@Column(name = "binary_content", columnDefinition = "BYTEA") public byte[] binaryContent;` field. Update `fromDomain()` and `toDomain()` to map the new field. Update `@PrePersist` and `@PreUpdate` to compute `sizeBytes` from `binaryContent.length` when binary field is populated. Add XOR validation: `if ((content != null) == (binaryContent != null)) throw new IllegalStateException("Exactly one of content/binaryContent must be non-null");` — but only when at least one is non-null (allow both null during chunked upload init).

- [ ] **Step 4: Add `storeBinary` to DataService**

```java
@Transactional
public SharedData storeBinary(String key, String description, String createdBy,
                               byte[] content, boolean lastChunk) {
    SharedData.Builder b = SharedData.builder(key)
            .binaryContent(content)
            .createdBy(createdBy)
            .complete(lastChunk)
            .sizeBytes(content != null ? content.length : 0);
    if (description != null) b.description(description);
    return dataStore.put(b.build());
}
```

Update `DataStore` interface and `InMemoryDataStore` with the new field mapping.

- [ ] **Step 5: Write V50 migration**

`runtime/src/main/resources/db/qhorus/migration/V50__shared_data_binary_and_signature_columns.sql`:

```sql
-- SharedData binary support
ALTER TABLE shared_data ADD COLUMN binary_content BYTEA;

-- Compliance report signature metadata
ALTER TABLE compliance_report ADD COLUMN signature_status VARCHAR(20) NOT NULL DEFAULT 'UNSIGNED';
ALTER TABLE compliance_report ADD COLUMN signed_at TIMESTAMP;
ALTER TABLE compliance_report ADD COLUMN signer_dn VARCHAR(500);
ALTER TABLE compliance_report ADD COLUMN key_ref VARCHAR(100);
ALTER TABLE compliance_report ADD COLUMN signing_profile VARCHAR(10);
ALTER TABLE compliance_report ADD COLUMN signature_artefact_id UUID;
```

- [ ] **Step 6: Update ComplianceReportRecord entity**

Add the 6 new fields to `ComplianceReportRecord.java` with matching JPA annotations.

- [ ] **Step 7: Run tests and commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -Dtest=SharedDataBinaryTest -pl runtime
```
Expected: PASS.

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/ runtime/ persistence-memory/ compliance-report/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#418): SharedData binary support + V50 signature metadata columns

Refs #418"
```

### Task 5: ComplianceReportSigningService + ReportRenderingService

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/signing/ComplianceReportSigningService.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/signing/SigningResult.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/signing/SignatureStatus.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/ReportRenderingService.java`
- Modify: `compliance-report/pom.xml` (add `casehub-platform-api` dependency — already present, but verify `signing.document` package is accessible)
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/signing/ComplianceReportSigningServiceTest.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/ReportRenderingServiceTest.java`

**Interfaces:**
- Consumes: `DocumentSigningService` (from Task 1), `ReportRenderer` (existing), `ReportFormat` (existing)
- Produces: `ComplianceReportSigningService.sign(byte[], ReportFormat) → SigningResult`, `ComplianceReportSigningService.sign(byte[], ReportFormat, String tenancyId) → SigningResult`, `ReportRenderingService.render(Object, ReportFormat) → byte[]`

- [ ] **Step 1: Write ComplianceReportSigningService tests**

```java
package io.casehub.qhorus.compliance.signing;

import io.casehub.platform.api.signing.document.*;
import io.casehub.qhorus.compliance.model.ReportFormat;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ComplianceReportSigningServiceTest {

    @Test
    void pdf_withSigner_returnsEmbedded() {
        var dss = mock(DocumentSigningService.class);
        when(dss.signPdf(any(), any())).thenReturn(Optional.of(
                new SignedDocument(new byte[]{9}, "CN=Seal", Instant.now(), "kr", SigningProfile.B_T)));
        var service = new ComplianceReportSigningService(dss);
        var result = service.sign(new byte[]{1}, ReportFormat.PDF, "tenant");
        assertThat(result.status()).isEqualTo(SignatureStatus.SIGNED);
        assertThat(result).isInstanceOf(SigningResult.Embedded.class);
    }

    @Test
    void json_withSigner_returnsDetached() {
        var dss = mock(DocumentSigningService.class);
        when(dss.signDetached(any(), any())).thenReturn(Optional.of(
                new DetachedSignature(new byte[]{8}, "CN=Seal", Instant.now(), "kr", SigningProfile.B_T)));
        var service = new ComplianceReportSigningService(dss);
        var result = service.sign(new byte[]{1}, ReportFormat.JSON, "tenant");
        assertThat(result.status()).isEqualTo(SignatureStatus.SIGNED);
        assertThat(result).isInstanceOf(SigningResult.Detached.class);
    }

    @Test
    void html_alwaysUnsigned() {
        var dss = mock(DocumentSigningService.class);
        var service = new ComplianceReportSigningService(dss);
        var result = service.sign(new byte[]{1}, ReportFormat.HTML, "tenant");
        assertThat(result.status()).isEqualTo(SignatureStatus.UNSIGNED);
        verifyNoInteractions(dss);
    }

    @Test
    void noOp_returnsUnsigned() {
        var dss = mock(DocumentSigningService.class);
        when(dss.signPdf(any(), any())).thenReturn(Optional.empty());
        var service = new ComplianceReportSigningService(dss);
        var result = service.sign(new byte[]{1}, ReportFormat.PDF, "tenant");
        assertThat(result.status()).isEqualTo(SignatureStatus.UNSIGNED);
    }
}
```

- [ ] **Step 2: Write ReportRenderingService tests**

```java
package io.casehub.qhorus.compliance.format;

import io.casehub.qhorus.compliance.model.ReportFormat;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class ReportRenderingServiceTest {

    @Test
    void render_selectsCorrectRenderer() {
        var jsonRenderer = mock(ReportRenderer.class);
        when(jsonRenderer.supports(ReportFormat.JSON)).thenReturn(true);
        when(jsonRenderer.render(any())).thenReturn(new byte[]{1, 2});
        var service = new ReportRenderingService(List.of(jsonRenderer));
        byte[] result = service.render("report", ReportFormat.JSON);
        assertThat(result).containsExactly(1, 2);
    }

    @Test
    void render_noRenderer_throws() {
        var service = new ReportRenderingService(List.of());
        assertThatThrownBy(() -> service.render("report", ReportFormat.PDF))
                .isInstanceOf(IllegalStateException.class);
    }
}
```

- [ ] **Step 3: Implement SigningResult, SignatureStatus, ComplianceReportSigningService, ReportRenderingService**

Follow the spec exactly. `ComplianceReportSigningService` has two overloads: 2-arg (reads `InboundTenancyContext`) and 3-arg (explicit tenancyId for scheduler). CDI-free tests use the 3-arg constructor directly.

- [ ] **Step 4: Run tests and commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/compliance-report/pom.xml -Dtest="ComplianceReportSigningServiceTest,ReportRenderingServiceTest"
```
Expected: PASS.

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#418): ComplianceReportSigningService + ReportRenderingService

Refs #418"
```

---

## Batch 4: Pipeline Integration + Verification

**Repo:** `casehub-qhorus`
**After this batch:** Scheduled reports are automatically signed. On-demand reports support `signed=true`. Verification endpoint validates signed reports. Detached signatures downloadable.

### Task 6: Pipeline wiring — scheduler, resource, storage

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportScheduler.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/storage/ComplianceReportStorageService.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/schedule/ComplianceReportGeneratedEvent.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/signing/SigningPipelineTest.java`

**Interfaces:**
- Consumes: `ComplianceReportSigningService.sign()` (from Task 5), `ReportRenderingService.render()` (from Task 5), `DataService.storeBinary()` (from Task 4), `ComplianceReportRecord` signature fields (from Task 4)
- Produces: Full signing pipeline — scheduler calls sign before store, resource supports `signed=true` param

- [ ] **Step 1: Write pipeline integration test**

CDI-free test mocking all services. Verify the scheduler calls `renderingService.render()` → `signingService.sign()` → `storageService.storeWithSignature()` in order. Verify the resource skips signing by default and calls signing when `signed=true`.

- [ ] **Step 2: Add `storeWithSignature` to ComplianceReportStorageService**

Accepts `SigningResult`, stores report body (text or binary depending on format), stores .p7s if detached, populates `ComplianceReportRecord` signature columns.

- [ ] **Step 3: Update ComplianceReportScheduler**

Inject `ReportRenderingService` and `ComplianceReportSigningService`. Change `generateAndStore()` to: render → sign → storeWithSignature.

- [ ] **Step 4: Update ComplianceReportResource**

Add `@QueryParam("signed") @DefaultValue("false") boolean signed` to all on-demand endpoints. Update `renderResponse()` to accept the `signed` flag. When `signed=true` and format is JSON/CSV, return `multipart/mixed` response with report body + .p7s signature.

- [ ] **Step 5: Update ComplianceReportGeneratedEvent**

Add `SignatureStatus signatureStatus` and `UUID signatureArtefactId` fields (both nullable, backward-compatible).

- [ ] **Step 6: Run full compliance-report test suite and commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/compliance-report/pom.xml
```
Expected: all PASS.

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#418): pipeline integration — scheduler, resource, storage signing wiring

Refs #418"
```

### Task 7: Verification endpoint + signature download

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceVerificationResponse.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/api/VerificationEndpointTest.java`

**Interfaces:**
- Consumes: `DocumentVerificationService.verifyPdf()`, `DocumentVerificationService.verifyDetached()` (from Task 1/3), `ComplianceReportRecord.signatureArtefactId` (from Task 4)
- Produces: `POST /api/compliance/verify`, `GET /api/compliance/reports/{id}/verify`, `GET /api/compliance/reports/{id}/signature`

- [ ] **Step 1: Write verification endpoint tests**

```java
package io.casehub.qhorus.compliance.api;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class VerificationEndpointTest {

    @Test
    void verify_unsignedPdf_returnsUnsigned() {
        byte[] unsignedPdf = createMinimalPdf();
        given()
                .multiPart("file", "report.pdf", unsignedPdf, "application/pdf")
                .when().post("/api/compliance/verify")
                .then()
                .statusCode(200)
                .body("status", equalTo("UNSIGNED"));
    }

    @Test
    void signatureDownload_noSignature_returns404() {
        // Create and store an unsigned report, then request its signature
        given()
                .when().get("/api/compliance/reports/{id}/signature", reportId)
                .then()
                .statusCode(404);
    }
}
```

- [ ] **Step 2: Create ComplianceVerificationResponse**

```java
package io.casehub.qhorus.compliance.api;

import java.util.List;

public record ComplianceVerificationResponse(
        String status, String signerDn, String signedAt, String keyRef,
        String detectedProfile, List<CertificateInfoDto> certificateChain,
        String diagnosticMessage) {

    public record CertificateInfoDto(
            String subjectDn, String issuerDn, String validFrom,
            String validTo, boolean claimsQualified) {}
}
```

- [ ] **Step 3: Implement endpoints**

`POST /api/compliance/verify`: accept multipart, detect format, delegate to `DocumentVerificationService`, map to `ComplianceVerificationResponse`.

`GET /api/compliance/reports/{id}/verify`: load stored report from SharedData, delegate to verification service.

`GET /api/compliance/reports/{id}/signature`: load `ComplianceReportRecord`, check `signatureArtefactId`, load .p7s from SharedData, return with `application/pkcs7-signature` content type. 404 if null.

- [ ] **Step 4: Run all tests and commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/compliance-report/pom.xml
```
Expected: all PASS.

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#418): verification endpoint + signature download

Refs #418"
```

- [ ] **Step 5: Run full project build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```
Expected: BUILD SUCCESS — all modules compile and tests pass.

---

## References

- `2026-08-31-digital-signatures-eidas-design.md` — design spec this plan implements
- `decisions.md` (D1-D8) — design decisions
- `SharedData.java` (api/data/) — gains binaryContent field
- `SharedDataEntity.java` (runtime/data/) — gains binary_content column
- `DataService.java` (runtime/data/) — gains storeBinary() method
- `ComplianceReportRecord.java` (compliance-report/storage/) — gains signature columns
- `ComplianceReportResource.java` (compliance-report/api/) — gains verify endpoint + signed param
- `ComplianceReportScheduler.java` (compliance-report/schedule/) — gains signing step
- `ComplianceReportStorageService.java` (compliance-report/storage/) — gains storeWithSignature()
- `ReportRenderer.java` (compliance-report/format/) — renderer interface for RenderingService
- `SigningProvider.java` (platform-api/signing/) — existing SPI, NOT used
- `NoOpPdfGenerator.java` (platform/) — @DefaultBean pattern reference
- `issue-402 design spec` — compliance evidence export architecture
- `issue-417 design spec` — PDF/A-2b rendering
- GitHub #418 — focal issue
