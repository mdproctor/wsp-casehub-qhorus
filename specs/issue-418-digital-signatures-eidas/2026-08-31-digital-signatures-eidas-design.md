# Design: Digital Signatures (eIDAS Qualified Seals) for Compliance Reports

**Issue:** casehubio/qhorus#418
**Date:** 2026-08-31
**Status:** Draft
**Parent:** casehubio/qhorus#402 (compliance evidence export — deferred work)
**Predecessor:** casehubio/qhorus#417 (PDF rendering — completed)

---

## Context

EU AI Act Article 12 requires deployers of high-risk AI systems to maintain auditable records. The compliance-report module (#402) produces those records in JSON, CSV, HTML, and PDF formats. The PDF renderer (#417) delivers PDF/A-2b output with document metadata and page structure.

The missing layer: **cryptographic signatures** that prove report authenticity, integrity, and creation time. Without signatures, a compliance report is an unsigned assertion — it cannot prove it wasn't modified after generation, and its creation time is self-reported.

### Scope: Advanced Electronic Seal — eIDAS-Ready Architecture

This design produces **advanced electronic seals** per eIDAS Regulation (EU) No 910/2014. The architecture supports eIDAS qualification at deployment time — plug in QTSP-issued certificates, QSCD-backed key material (HSM/cloud KMS), and EU Trusted List validation. Qualification is a PKI provisioning and legal decision, not a code decision.

The distinction matters: a self-signed or non-qualified certificate produces a structurally identical signature (PAdES-B-T, CAdES-B-T) but without the legal presumption of integrity that eIDAS qualification provides. The code infrastructure is identical — only the certificate's provenance differs.

### Existing Infrastructure

| Layer | What exists | What this design adds |
|-------|------------|----------------------|
| Platform raw signing | `SigningProvider` SPI — `sign(actorId, byte[])` → `SignatureResult(signature, publicKey, keyRef)`. Ed25519/EC/ML-DSA. `NoOpSigningProvider` @DefaultBean. Used by ledger agent attestation signing. | Nothing — separate concern, unchanged |
| Platform PDF | `PdfGenerator` SPI, `OpenHtmlToPdfGenerator` impl, PDF/A-2b | Nothing — unchanged |
| Platform credentials | `CredentialResolver` SPI — key-value credential lookup | Resolves PKCS#12 keystore password |
| Qhorus compliance-report | 8 report types, 4 renderers (JSON/CSV/HTML/PDF), SharedData storage, scheduler, V47-V48 migrations | Signing step in pipeline, signature metadata on ComplianceReportRecord, verification endpoint |

**SigningProvider is not used by this design.** `SigningProvider.sign()` returns raw bytes and a public key — no X.509 certificate chain, no timestamping, no format-specific packaging. Document signing requires all of those. `DocumentSigningService` (new, this design) is a self-contained SPI that owns its own key material configuration. The two SPIs serve different domains: `SigningProvider` for ledger agent attestation, `DocumentSigningService` for document sealing.

---

## Architecture

Two repos, two new modules:

```
casehub-platform/
├── platform-api/
│   └── src/main/java/io/casehub/platform/api/signing/
│       ├── SigningProvider.java              — existing, unchanged
│       ├── SignatureResult.java             — existing, unchanged
│       ├── SignatureVerifier.java           — existing, unchanged
│       └── document/
│           ├── DocumentSigningService.java   — NEW: SPI interface
│           ├── DocumentVerificationService.java — NEW: SPI interface
│           ├── SigningIdentity.java          — NEW: record
│           ├── SignedDocument.java           — NEW: record
│           ├── DetachedSignature.java        — NEW: record
│           ├── SigningProfile.java           — NEW: enum (B_B, B_T, B_LT, B_LTA)
│           ├── DocumentVerificationResult.java — NEW: record
│           ├── VerificationStatus.java       — NEW: enum
│           ├── NoOpDocumentSigningService.java — NEW: @DefaultBean
│           └── NoOpDocumentVerificationService.java — NEW: @DefaultBean
│
├── platform-signing/                        — NEW module
│   ├── pom.xml
│   └── src/main/java/io/casehub/platform/signing/document/
│       ├── DssDocumentSigningService.java   — @ApplicationScoped EU DSS impl
│       ├── DssDocumentVerificationService.java — @ApplicationScoped EU DSS impl
│       ├── KeyStoreManager.java             — PKCS#12 loader, certificate chain
│       └── DssSigningConfig.java            — @ConfigMapping

casehub-qhorus/
└── compliance-report/
    └── src/main/java/io/casehub/qhorus/compliance/
        ├── signing/
        │   └── ComplianceReportSigningService.java — orchestrates sign step
        ├── format/
        │   └── ReportRenderingService.java        — NEW: CDI-based renderer resolution
        └── api/
            └── ComplianceReportResource.java      — gains verify + signed param
```

### Data Flow

**Scheduled reports (always signed when configured):**
```
ReportService.generate(params)
  → ReportRenderingService.render(report, format)     → byte[] (unsigned PDF/JSON/CSV)
  → ComplianceReportSigningService.sign(bytes, format, tenancyId)
     → DocumentSigningService.signPdf(bytes, identity)        [PDF → PAdES embedded]
     → DocumentSigningService.signDetached(bytes, identity)   [JSON/CSV → CAdES .p7s]
  → ComplianceReportStorageService.storeWithSignature(reportType, signed, format, ...)
     → SharedData: report body (text via content, binary via binaryContent)
     → SharedData: .p7s signature as binaryContent (JSON/CSV only; PDF signature embedded)
     → ComplianceReportRecord: metadata + signature status columns
```

**On-demand reports (unsigned by default, opt-in signing):**
```
GET /api/compliance/obligations?from=X&to=Y
  → unsigned response (fast path, no signing overhead)

GET /api/compliance/obligations?from=X&to=Y&signed=true
  → render → sign → response (200-500ms additional latency from TSA)
  → PDF: response body is signed PDF (PAdES — signature embedded)
  → JSON/CSV: multipart/mixed response (report body + .p7s signature)
```

### SharedData Binary Support (Prerequisite)

`SharedData.content` is `String` (TEXT column). Signed PDFs and .p7s signatures are binary `byte[]` and cannot be stored in a text column. This design adds binary support to SharedData:

```java
// SharedData record — new field
public record SharedData(
    UUID id, String key,
    String content,          // text artefacts (JSON, CSV, HTML)
    byte[] binaryContent,    // binary artefacts (PDF, .p7s) — mutually exclusive with content
    String createdBy, String description,
    boolean complete, long sizeBytes,
    Instant createdAt, Instant updatedAt) {}
```

```java
// DataService — new method
public SharedData storeBinary(String key, String description, String createdBy,
                               byte[] content, boolean lastChunk) { ... }
```

Migration (part of V50): `ALTER TABLE shared_data ADD COLUMN binary_content BYTEA;`

`sizeBytes` in `SharedDataEntity.prePersist()` updated to compute from `binaryContent.length` when the binary field is populated, `content.length()` otherwise. Exactly one of `content` / `binaryContent` is non-null for any given artefact.

Storage routing in `ComplianceReportStorageService.storeWithSignature()`:
- JSON/CSV report body: text → `DataService.store()` (existing path)
- PDF report body (signed or unsigned): binary → `DataService.storeBinary()`
- .p7s detached signature: binary → `DataService.storeBinary()`

---

## Platform SPI: `DocumentSigningService`

```java
package io.casehub.platform.api.signing.document;

public interface DocumentSigningService {

    Optional<SignedDocument> signPdf(byte[] pdfBytes, SigningIdentity identity);

    Optional<DetachedSignature> signDetached(byte[] data, SigningIdentity identity);
}
```

### Supporting Types

```java
public record SigningIdentity(
    String actorId,
    String tenancyId
) {}

public record SignedDocument(
    byte[] signedBytes,
    String signerDn,
    Instant signedAt,
    String keyRef,
    SigningProfile profile
) {}

public record DetachedSignature(
    byte[] signatureBytes,
    String signerDn,
    Instant signedAt,
    String keyRef,
    SigningProfile profile
) {}

public enum SigningProfile {
    B_B, B_T, B_LT, B_LTA
}
```

`SigningIdentity` carries two distinct fields:
- `actorId`: audit metadata identifying who initiated the signing request (e.g., `"system:compliance-signer"`). Recorded in logs and signature metadata for traceability. Not used for key selection.
- `tenancyId`: drives key alias resolution. `KeyStoreManager` resolves the signing certificate using the convention `{tenancyId}-seal` (e.g., `tenant-acme-seal`), falling back to `config.keyAlias()` (shared default) when the tenant-specific alias is absent.

`keyRef` on `SignedDocument` and `DetachedSignature` is the SHA-256 fingerprint of the signing certificate's DER encoding (URL-safe Base64, no padding). This is consistent with the existing `SignatureResult.computeKeyRef()` convention in platform-api. It uniquely identifies the certificate for rotation tracking, audit queries, and key compromise investigation.

### NoOp Defaults

```java
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

Same pattern as `NoOpPdfGenerator` — graceful degradation when `casehub-platform-signing` is absent.

---

## Timestamping (Internal to `platform-signing`)

Timestamping is an implementation detail of `DssDocumentSigningService`, not a platform SPI. `DssDocumentSigningService` creates an `OnlineTSPSource` (EU DSS's RFC 3161 HTTP client) from its configuration (`tsaUrl`, `tsaTimeout`). No `TimestampProvider` interface in platform-api — a custom `DocumentSigningService` implementation (e.g., cloud KMS) handles its own timestamping internally.

When the configured profile is B_T or higher and the TSA is unreachable, **signing fails with an exception** (D3: no silent fallback). When the profile is B_B, the TSA is not called.

---

## Platform SPI: `DocumentVerificationService`

```java
package io.casehub.platform.api.signing.document;

public interface DocumentVerificationService {

    DocumentVerificationResult verifyPdf(byte[] pdfBytes);

    DocumentVerificationResult verifyDetached(byte[] data, byte[] signature);
}

public record DocumentVerificationResult(
    VerificationStatus status,
    String signerDn,
    Instant signedAt,
    String keyRef,
    SigningProfile detectedProfile,
    List<CertificateInfo> certificateChain,
    String diagnosticMessage
) {}

public enum VerificationStatus {
    VALID,
    INVALID,
    UNSIGNED,
    UNSUPPORTED_FORMAT,
    ERROR
}

public record CertificateInfo(
    String subjectDn,
    String issuerDn,
    Instant validFrom,
    Instant validTo,
    boolean claimsQualified
) {}
```

`DssDocumentVerificationService` uses DSS's `SignedDocumentValidator` for algorithm-transparent verification. `claimsQualified` on `CertificateInfo` is `true` when the certificate contains QcStatement OIDs from ETSI EN 319 412 — this is a structural check (the certificate *claims* to be qualified), not a trusted list validation. Without EU Trusted List (LOTL) validation (#426), a self-signed certificate with QcStatement OIDs would also be `claimsQualified=true`. The field name makes this limitation explicit.

---

## Platform Module: `casehub-platform-signing`

### Dependencies

```xml
<dependency>eu.europa.ec.joinup.sd-dss:dss-pades-pdfbox</dependency>
<dependency>eu.europa.ec.joinup.sd-dss:dss-cades</dependency>
<dependency>eu.europa.ec.joinup.sd-dss:dss-service</dependency>
<dependency>eu.europa.ec.joinup.sd-dss:dss-tsl-validation</dependency>  <!-- optional -->
<dependency>io.casehub:casehub-platform-api</dependency>
```

**PDFBox version compatibility:** EU DSS 6.x depends on PDFBox 3.x. Verified: `OpenHtmlToPdfGenerator` (#417) already uses PDFBox 3.x API (`org.apache.pdfbox.Loader.loadPDF()` — introduced in PDFBox 3.x). No version conflict exists — both libraries target PDFBox 3.x.

### Configuration

```java
@ConfigMapping(prefix = "casehub.signing.document")
public interface DssSigningConfig {

    Optional<String> keystorePath();

    Optional<String> keystorePasswordRef();

    Optional<String> keystoreType();  // default PKCS12

    Optional<String> keyAlias();

    SigningProfile profile();  // default B_T

    Optional<String> tsaUrl();

    Optional<Duration> tsaTimeout();  // default 10s

    Optional<Boolean> required();  // default false — when true, startup fails if keystore not configured
}
```

`keystorePasswordRef` is resolved via `CredentialResolver` — the config value is a credential key, not the password itself. No secrets in application.properties.

### KeyStoreManager

```java
@ApplicationScoped
class KeyStoreManager {

    private KeyStore keyStore;
    private PrivateKey privateKey;
    private X509Certificate[] certificateChain;

    @PostConstruct
    void load() {
        // Load PKCS#12 from config.keystorePath()
        // Resolve password via CredentialResolver(config.keystorePasswordRef())
        // Extract private key and certificate chain by config.keyAlias()
    }

    DSSPrivateKeyEntry toDssKeyEntry() {
        return new KSPrivateKeyEntry(keyAlias, new KeyStore.PrivateKeyEntry(
                privateKey, certificateChain));
    }
}
```

`KeyStoreManager` is thread-safe: all mutable fields (`keyStore`, `privateKey`, `certificateChain`) are written once in `@PostConstruct` and never modified thereafter. The `PrivateKey` and `X509Certificate[]` objects passed to DSS signing operations are read-only inputs — DSS does not mutate them. The fields cannot be declared `final` (CDI requires mutable fields for `@PostConstruct` initialization) but are effectively immutable after construction.

Loaded once at startup. Certificate rotation: replace the PKCS#12 file and restart the application. Runtime rotation without restart is deferred (#423).

### DssDocumentSigningService

```java
@ApplicationScoped
public class DssDocumentSigningService implements DocumentSigningService {

    @Inject KeyStoreManager keyStoreManager;
    @Inject DssSigningConfig config;

    @Override
    public Optional<SignedDocument> signPdf(byte[] pdfBytes, SigningIdentity identity) {
        PAdESSignatureParameters params = new PAdESSignatureParameters();
        params.setSignatureLevel(toSignatureLevel(config.profile()));
        params.setSigningCertificate(keyStoreManager.toDssKeyEntry().getCertificate());
        params.setCertificateChain(keyStoreManager.toDssKeyEntry().getCertificateChain());

        if (config.profile().requiresTimestamp()) {
            OnlineTSPSource tspSource = buildTspSource();
            // DSS handles timestamp embedding internally
        }

        PAdESService service = new PAdESService(commonCertificateVerifier());
        // sign and return SignedDocument
    }

    @Override
    public Optional<DetachedSignature> signDetached(byte[] data, SigningIdentity identity) {
        CAdESSignatureParameters params = new CAdESSignatureParameters();
        params.setSignatureLevel(toCadesLevel(config.profile()));
        params.setSignaturePackaging(SignaturePackaging.DETACHED);
        // ... similar pattern with CAdESService
    }
}
```

### Strict Profile Enforcement

```java
private SignatureLevel toSignatureLevel(SigningProfile profile) {
    return switch (profile) {
        case B_B  -> SignatureLevel.PAdES_BASELINE_B;
        case B_T  -> SignatureLevel.PAdES_BASELINE_T;
        case B_LT -> SignatureLevel.PAdES_BASELINE_LT;
        case B_LTA -> SignatureLevel.PAdES_BASELINE_LTA;
    };
}
```

When profile is B_T or higher: if the TSA HTTP call (via DSS's `OnlineTSPSource`) fails or times out, `DssDocumentSigningService` throws `IllegalStateException("TSA unavailable — cannot produce " + profile + " signature")`. No silent downgrade.

When `config.required()` is `true` and `config.keystorePath()` is absent, startup fails immediately — this guards against deployments where signing is intended but misconfigured (e.g., `platform-signing` jar missing from classpath).

---

## Qhorus: ComplianceReportSigningService

```java
package io.casehub.qhorus.compliance.signing;

@ApplicationScoped
public class ComplianceReportSigningService {

    @Inject DocumentSigningService signingService;
    @Inject InboundTenancyContext tenancyContext;

    public SigningResult sign(byte[] reportBytes, ReportFormat format) {
        return sign(reportBytes, format, tenancyContext.tenancyId());
    }

    public SigningResult sign(byte[] reportBytes, ReportFormat format, String tenancyId) {
        SigningIdentity identity = new SigningIdentity(
                "system:compliance-signer", tenancyId);

        return switch (format) {
            case PDF -> {
                var signed = signingService.signPdf(reportBytes, identity);
                yield signed.map(s -> SigningResult.embedded(s.signedBytes(),
                        s.signerDn(), s.signedAt(), s.keyRef(), s.profile()))
                    .orElseGet(() -> {
                        LOG.warn("Producing unsigned PDF — no DocumentSigningService configured");
                        return SigningResult.unsigned(reportBytes);
                    });
            }
            case JSON, CSV -> {
                var sig = signingService.signDetached(reportBytes, identity);
                yield sig.map(s -> SigningResult.detached(reportBytes,
                        s.signatureBytes(), s.signerDn(), s.signedAt(),
                        s.keyRef(), s.profile()))
                    .orElseGet(() -> {
                        LOG.warn("Producing unsigned %s — no DocumentSigningService configured",
                                format);
                        return SigningResult.unsigned(reportBytes);
                    });
            }
            case HTML -> SigningResult.unsigned(reportBytes);
        };
    }
}
```

### SigningResult

```java
public sealed interface SigningResult {

    byte[] reportBytes();
    SignatureStatus status();

    record Embedded(byte[] reportBytes, String signerDn, Instant signedAt,
                    String keyRef, SigningProfile profile) implements SigningResult {
        public SignatureStatus status() { return SignatureStatus.SIGNED; }
    }

    record Detached(byte[] reportBytes, byte[] signatureBytes, String signerDn,
                    Instant signedAt, String keyRef, SigningProfile profile)
            implements SigningResult {
        public SignatureStatus status() { return SignatureStatus.SIGNED; }
    }

    record Unsigned(byte[] reportBytes) implements SigningResult {
        public SignatureStatus status() { return SignatureStatus.UNSIGNED; }
    }

    static SigningResult embedded(...) { ... }
    static SigningResult detached(...) { ... }
    static SigningResult unsigned(byte[] bytes) { return new Unsigned(bytes); }
}

public enum SignatureStatus {
    SIGNED, UNSIGNED
}
```

---

## Pipeline Integration

### ReportRenderingService (New)

Extracted from `ComplianceReportResource`'s inline renderer selection. Both the resource and scheduler inject this service:

```java
@ApplicationScoped
public class ReportRenderingService {

    @Inject Instance<ReportRenderer> renderers;

    public byte[] render(Object report, ReportFormat format) {
        return renderers.stream()
            .filter(r -> r.supports(format))
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("No renderer for " + format))
            .render(report);
    }

    public String contentType(ReportFormat format) {
        return renderers.stream()
            .filter(r -> r.supports(format))
            .findFirst()
            .map(ReportRenderer::contentType)
            .orElse("application/octet-stream");
    }
}
```

Uses CDI `Instance<ReportRenderer>` to discover renderers by format via the existing `ReportRenderer.supports(ReportFormat)` method. Eliminates the duplicated format→renderer resolution logic.

### ComplianceReportScheduler Changes

```java
@Inject ReportRenderingService renderingService;
@Inject ComplianceReportSigningService signingService;

private void generateAndStore(ComplianceReportSchedule schedule, Instant from, Instant now) {
    Object report = switch (schedule.reportType) { ... };  // existing

    byte[] rendered = renderingService.render(report, schedule.format);
    SigningResult signed = signingService.sign(rendered, schedule.format, schedule.tenancyId);

    ComplianceReportRecord record = storageService.storeWithSignature(
            schedule.reportType, signed, schedule.format,
            schedule.id, schedule.tenancyId);

    generatedEvent.fireAsync(new ComplianceReportGeneratedEvent(
            record.id, schedule.reportType, schedule.tenancyId,
            record.artefactId, now, schedule.id,
            "system:compliance-scheduler", Map.of(),
            signed.status(), record.signatureArtefactId));
}
```

### ComplianceReportResource Changes

```java
@Inject ReportRenderingService renderingService;
@Inject ComplianceReportSigningService signingService;

private Response renderResponse(Object report, String accept, boolean signed) {
    ReportFormat format = detectFormat(accept);
    byte[] rendered = renderingService.render(report, format);

    if (signed) {
        SigningResult result = signingService.sign(rendered, format);
        return buildSignedResponse(result, format);
    }
    return Response.ok(rendered)
            .header("Content-Type", renderingService.contentType(format))
            .build();
}
```

On-demand endpoints gain `@QueryParam("signed") @DefaultValue("false") boolean signed`.

**On-demand signed response format:**
- **PDF** (`signed=true`): Response body is the PAdES-signed PDF. Content-Type: `application/pdf`. Transparent — the signature is embedded in the PDF itself.
- **JSON/CSV** (`signed=true`): Response is `multipart/mixed` with two parts:
  - Part 1: report body (`application/json` or `text/csv`), Content-Disposition: `inline; name="report"`
  - Part 2: .p7s signature (`application/pkcs7-signature`), Content-Disposition: `attachment; name="signature"; filename="signature.p7s"`
- **HTML**: `signed=true` ignored (HTML is not signable).

### ComplianceReportStorageService Changes

New method `storeWithSignature(ReportType, SigningResult, ReportFormat, UUID scheduleId, String tenancyId)`:

1. Store report body → SharedData artefact
   - JSON/CSV: `dataService.store()` (text path — rendered bytes decoded to UTF-8 String)
   - PDF: `dataService.storeBinary()` (binary path)
2. If `SigningResult.Detached`: store .p7s → second SharedData artefact via `dataService.storeBinary()`
3. Create `ComplianceReportRecord` with signature metadata columns populated

**Stored report retrieval (`getStoredReport`)** becomes format-aware:

```java
public Response getStoredReport(UUID id) {
    var record = recordStore.findById(id).orElse(null);
    if (record == null) return Response.status(NOT_FOUND).build();

    var data = dataService.getByUuid(record.artefactId).orElse(null);
    if (data == null) return Response.status(NOT_FOUND).build();

    String contentType = renderingService.contentType(record.format);
    if (data.binaryContent() != null) {
        return Response.ok(data.binaryContent()).header("Content-Type", contentType).build();
    }
    return Response.ok(data.content()).header("Content-Type", contentType).build();
}
```

The existing `retrieveJson()` method is renamed to `retrieveContent()` returning a new `StoredReportContent` record:

```java
public record StoredReportContent(byte[] bytes, String contentType, ReportFormat format) {}
```

### ComplianceReportGeneratedEvent Changes

Updated to include signature metadata:

```java
public record ComplianceReportGeneratedEvent(
    UUID reportId, ReportType reportType, String tenancyId,
    UUID artefactId, Instant generatedAt, UUID scheduleId,
    String requestedBy, Map<String, String> requestParameters,
    SignatureStatus signatureStatus,    // NEW
    UUID signatureArtefactId            // NEW — null for embedded/unsigned
) {}
```

---

## Database Migration: V50

SharedData binary support:

```sql
ALTER TABLE shared_data ADD COLUMN binary_content BYTEA;
```

New columns on `compliance_report`:

```sql
ALTER TABLE compliance_report ADD COLUMN signature_status VARCHAR(20) NOT NULL DEFAULT 'UNSIGNED';
ALTER TABLE compliance_report ADD COLUMN signed_at TIMESTAMP;
ALTER TABLE compliance_report ADD COLUMN signer_dn VARCHAR(500);
ALTER TABLE compliance_report ADD COLUMN key_ref VARCHAR(100);
ALTER TABLE compliance_report ADD COLUMN signing_profile VARCHAR(10);
ALTER TABLE compliance_report ADD COLUMN signature_artefact_id UUID;
```

`signature_status`: `SIGNED` or `UNSIGNED`. Default `UNSIGNED` for backward compatibility with existing records.

`signature_artefact_id`: FK to SharedData. Non-null only for detached signatures (JSON/CSV). For PDF, the signature is embedded in the report artefact itself — no separate artefact needed.

No FK constraint on `signature_artefact_id` — follows the existing `artefact_id` pattern (SharedData is in a separate named PU; cross-PU FKs are not supported).

No index on `signature_artefact_id` — the primary lookup is by `compliance_report.id` (PK), not by artefact ID. Reverse lookups (find report by artefact ID) are not part of the current access patterns.

---

## Verification Endpoint

### REST API

```
POST /api/compliance/verify
    Content-Type: multipart/form-data
    Parts:
      - "file": the report file (PDF, JSON, or CSV)
      - "signature": optional .p7s file (required for JSON/CSV, ignored for PDF)
    Returns: ComplianceVerificationResponse
```

```java
public record ComplianceVerificationResponse(
    String status,           // VALID, INVALID, UNSIGNED, UNSUPPORTED_FORMAT, ERROR
    String signerDn,         // null if unsigned
    String signedAt,         // ISO-8601, null if unsigned
    String keyRef,           // null if unsigned
    String detectedProfile,  // B_B, B_T, etc., null if unsigned
    List<CertificateInfoDto> certificateChain,
    String diagnosticMessage // null unless ERROR
) {}
```

**Upload limits:** Maximum file size 10MB (compliance reports are typically <1MB). Enforced via Quarkus `quarkus.http.limits.max-body-size`. The `byte[] pdfBytes` parameter is bounded by this limit. Rate limiting and tenant-scoped access are deployment concerns (reverse proxy / Quarkus rate-limit extension).

### Stored Report Verification

```
GET /api/compliance/reports/{id}/verify
    Returns: ComplianceVerificationResponse
```

Re-verifies a stored report by loading it from SharedData and passing it through `DocumentVerificationService`. This catches post-storage tampering (e.g., SharedData artefact modified directly in the database).

### Signature Download

```
GET /api/compliance/reports/{id}/signature
    Returns: .p7s bytes (application/pkcs7-signature)
    404 if report has no detached signature (PDF or unsigned)
```

---

## Multi-Tenancy

`DssSigningConfig` is flat (single keystore). For multi-tenant deployments where tenants need different signing certificates:

**Phase 1 (this design):** Single keystore with multiple aliases. `SigningIdentity.tenancyId()` maps to a key alias convention: `{tenancyId}-seal` (e.g., `tenant-acme-seal`). `KeyStoreManager` resolves the alias at signing time. If the tenant-specific alias doesn't exist, falls back to `config.keyAlias()` (shared default). `SigningIdentity.actorId()` is audit metadata only — not used for key selection.

**Phase 2 (deferred):** Per-tenant keystore paths via tenant configuration (#425). Not needed until a multi-tenant deployment with separate keystores is required.

---

## GraalVM Native Image

EU DSS uses reflection heavily (PDFBox, BouncyCastle, ICU4J). Following the #417 precedent: JVM-only deployment is an acceptable fallback for compliance signing — it is not a latency-sensitive path. Native image compatibility should be attempted via `native-image-agent` tracing, but is not a gate on this issue.

---

## Testing Strategy

| Component | Test type | Notes |
|-----------|----------|-------|
| `DssDocumentSigningService` | CDI-free unit test | Self-signed test certificate (generated in @BeforeAll); verify signed PDF bytes contain PAdES signature dict; verify CAdES .p7s structure |
| `DssDocumentVerificationService` | CDI-free unit test | Sign-then-verify round-trip; test UNSIGNED detection (unsigned PDF); test INVALID (tampered bytes) |
| `NoOpDocumentSigningService` | CDI-free unit test | Verify returns Optional.empty() |
| `KeyStoreManager` | CDI-free unit test | Load test PKCS#12; verify private key and certificate chain extraction; verify alias resolution with tenancy fallback; verify write-once thread safety (fields not modified after init) |
| Strict profile enforcement | CDI-free unit test | B_T configured + TSA unavailable → IllegalStateException; B_B configured + no TSA → succeeds |
| `config.required=true` startup | CDI-free unit test | Missing keystorePath → startup failure; present keystorePath → success |
| `ReportRenderingService` | CDI-free unit test | Mock renderers; verify format→renderer resolution; verify exception on unsupported format |
| `ComplianceReportSigningService` | CDI-free unit test | Mock DocumentSigningService; verify PDF→embedded, JSON→detached, HTML→unsigned routing |
| `SigningResult` sealed interface | CDI-free unit test | Pattern matching exhaustiveness; status() values |
| Pipeline integration | CDI-free unit test | Mock services; verify scheduler calls sign() before store(); verify on-demand skips signing by default |
| REST verify endpoint | `@QuarkusTest` | Upload signed PDF → VALID; upload unsigned PDF → UNSIGNED; upload JSON + .p7s → VALID |
| REST signed=true param | `@QuarkusTest` | Verify on-demand report with signed=true returns signed bytes |
| Stored report verify | `@QuarkusTest` | Store signed report, GET verify → VALID |
| Signature download | `@QuarkusTest` | Store detached-signed report, GET signature → .p7s bytes; PDF report → 404 |
| V50 migration | `FlywayMigrationSchemaTest` | Verify column additions on compliance_report table |
| SPI displacement | `@QuarkusTest @TestProfile` | NoOpDocumentSigningService displaced by DssDocumentSigningService when platform-signing on classpath |
| PDFBox version compat | Build verification | Compile and run with both OpenHTMLtoPDF and DSS on classpath; verify no NoSuchMethodError. Both already target PDFBox 3.x — this is a smoke test, not a conflict resolution |
| SharedData binary support | CDI-free unit test | Store binary via storeBinary(), retrieve, verify binaryContent round-trip; verify sizeBytes computed correctly |
| Stored report retrieval | `@QuarkusTest` | Store JSON report → retrieve → correct Content-Type; store PDF report → retrieve → correct binary Content-Type |

Test PKCS#12 keystore: generated in test setup with a self-signed certificate (BouncyCastle `X509v3CertificateBuilder`). No real certificates in test resources.

---

## Certificate Lifecycle (Advisory)

Certificate management is a deployment concern, not a code concern. Advisory guidance for deployers:

- **Provisioning:** Obtain a PKCS#12 from your QTSP (for qualified seals) or generate a self-signed certificate (for advanced seals / development)
- **Rotation:** Replace the PKCS#12 file and restart the application. Both old and new signatures remain verifiable — the certificate chain is embedded in each signature
- **Monitoring:** Monitor certificate expiry externally (e.g., `keytool -list -v -keystore seal.p12`). For B-T signatures, the timestamp proves the signature was created before expiry, so signatures remain valid after certificate expiry
- **Future:** Runtime rotation via admin endpoint and certificate expiry alerting are deferred refinements

---

## Deferred Work

| Item | Reason | Issue |
|------|--------|-------|
| Runtime certificate rotation | Requires file watcher or admin reload endpoint | casehubio/qhorus#423 |
| Certificate expiry monitoring/alerting | Deployment concern; integrate with existing monitoring | casehubio/qhorus#424 |
| Per-tenant keystore paths | Not needed until multi-tenant deployment with separate keystores | casehubio/qhorus#425 |
| EU Trusted List (LOTL) validation | Required for full eIDAS qualified verification; deployment config | casehubio/qhorus#426 |
| External signing service (SignServer, cloud HSM) | Different DocumentSigningService impl; architecture supports it | casehubio/qhorus#427 |
| Automated retention with signature preservation | Legal guidance needed on minimum retention periods | casehubio/qhorus#419 |

---

## References

- `PdfReportRenderer.java` (compliance-report/format/) — existing PDF rendering, unchanged
- `ReportRenderer.java` (compliance-report/format/) — renderer interface, unchanged; `supports(ReportFormat)` used by new `ReportRenderingService`
- `ReportRenderingService.java` (compliance-report/format/) — NEW: CDI `Instance<ReportRenderer>` resolver
- `SharedData.java` (api/data/) — gains `binaryContent` field for binary artefact storage
- `DataService.java` (runtime/data/) — gains `storeBinary()` method
- `ComplianceReportRecord.java` (compliance-report/storage/) — gains signature columns
- `ComplianceReportResource.java` (compliance-report/api/) — gains verify endpoint + signed param
- `ComplianceReportScheduler.java` (compliance-report/schedule/) — gains signing step
- `SigningProvider.java` (platform-api/signing/) — existing raw signing SPI, NOT used by this design
- `SignatureVerifier.java` (platform-api/signing/) — existing verification utility, NOT used by this design
- `NoOpPdfGenerator.java` (platform/) — @DefaultBean pattern reference
- `CredentialResolver.java` (platform-api/credentials/) — keystore password resolution
- issue-402 design spec — compliance evidence export architecture
- issue-417 design spec — PDF/A-2b rendering, GraalVM native image precedent
- decisions.md (D1-D8) — all captured design decisions
- EU DSS GitHub (ec-europa/dss) — signing library
- ETSI EN 319 142 — PAdES profiles (B-B, B-T, B-LT, B-LTA)
- ETSI EN 319 122 — CAdES profiles
- eIDAS Regulation (EU) No 910/2014 — Articles 35-36 (electronic seals)
- RFC 3161 — Time-Stamp Protocol
- R1-01 through R1-17 — decision review findings
