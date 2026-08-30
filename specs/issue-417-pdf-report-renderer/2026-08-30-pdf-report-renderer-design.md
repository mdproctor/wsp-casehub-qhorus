# Design: PDF Rendering for Compliance Evidence Export Reports

**Issue:** casehubio/qhorus#417
**Date:** 2026-08-30
**Status:** Draft
**Parent:** casehubio/qhorus#402 (compliance evidence export — D3 deferral)

---

## Context

The compliance evidence export module (#402) delivers JSON, CSV, and HTML renderers. D3 in that design explicitly deferred PDF rendering: "browsers can print HTML to PDF natively for internal use. However, browser-printed PDFs lack document metadata, PDF/A compliance, and digital signature support that regulators require for formal submissions."

This issue delivers the proper PDF renderer. Digital signatures (eIDAS qualified seals) remain deferred to #418.

### Existing Infrastructure

The `compliance-report/` module has a clean renderer extension point:

```java
public interface ReportRenderer {
    String contentType();
    byte[] render(Object report);
    boolean supports(ReportFormat format);
}
```

`HtmlReportRenderer` generates structured HTML with print-friendly CSS, per-report-type rendering (Attribution, Obligation, Violation, JudgmentAttribution, JudgmentFulfillment, PropertyVerification), and a generic fallback.

`ComplianceReportResource.renderResponse()` uses `Accept` header content negotiation with hardcoded renderer injection. `ComplianceReportScheduler.generateAndStore()` passes `ReportFormat` from the schedule entity.

---

## Architecture

PDF rendering splits into two layers: platform infrastructure (HTML→PDF conversion, font management) and qhorus composition (report-specific rendering + page structure). D3: PDF rendering is general-purpose infrastructure — platform placement prevents duplication when other repos need it.

### Platform Layer: `casehub-platform-pdf`

New optional platform module owning the OpenHTMLtoPDF dependency and font resources.

**SPI in `casehub-platform-api`:**

```java
package io.casehub.platform.api.pdf;

public interface PdfGenerator {
    Optional<byte[]> generateFromHtml(String html, PdfOptions options);
}

public record PdfOptions(
    String title,
    String author,
    Instant createdAt,
    String reportType,
    PdfAConformance conformance
) {
    public static PdfOptions defaults() {
        return new PdfOptions(null, null, null, null, PdfAConformance.PDFA_2_B);
    }
}

public enum PdfAConformance {
    PDFA_2_B
}
```

`@DefaultBean NoOpPdfGenerator` returns `Optional.empty()` — graceful degradation when platform-pdf module is absent (per R1-12 convention).

**Implementation in `casehub-platform-pdf`:**

```java
@ApplicationScoped
public class OpenHtmlToPdfGenerator implements PdfGenerator {

    private List<FontResource> fonts;

    @PostConstruct
    void loadFonts() {
        // Load bundled Liberation Sans + Mono from classpath
    }

    @Override
    public Optional<byte[]> generateFromHtml(String html, PdfOptions options) {
        PdfRendererBuilder builder = new PdfRendererBuilder();
        builder.usePdfAConformance(PdfAConformance.PDFA_2_B);
        builder.withHtmlContent(html, "/");
        // Register fonts, set document metadata
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        builder.toStream(os);
        builder.run();
        setDocumentMetadata(os.toByteArray(), options);
        return Optional.of(os.toByteArray());
    }
}
```

**Bundled fonts:** Liberation Sans (Regular, Bold, Italic, BoldItalic) + Liberation Mono (Regular, Bold) — Apache 2.0, ~1.4MB total. Liberation Mono for UUIDs, Merkle hashes, correlation IDs — structured data that benefits from monospaced rendering for compliance auditors.

### Qhorus Layer: `PdfReportRenderer` in `compliance-report/`

Thin adapter composing `HtmlReportRenderer` and platform's `PdfGenerator`:

```java
@ApplicationScoped
public class PdfReportRenderer implements ReportRenderer {

    @Inject HtmlReportRenderer htmlRenderer;
    @Inject PdfGenerator pdfGenerator;

    @Override
    public String contentType() {
        return "application/pdf";
    }

    @Override
    public byte[] render(Object report) {
        PdfDocumentMetadata metadata = PdfDocumentMetadata.fromReport(report);
        String html = htmlRenderer.renderForPdf(report, metadata);
        PdfOptions options = new PdfOptions(
            metadata.title(), metadata.author(),
            metadata.createdAt(), metadata.reportType(),
            PdfAConformance.PDFA_2_B);
        return pdfGenerator.generateFromHtml(html, options)
            .orElseThrow(() -> new IllegalStateException(
                "PDF generation unavailable — casehub-platform-pdf not on classpath"));
    }

    @Override
    public boolean supports(ReportFormat format) {
        return format == ReportFormat.PDF;
    }
}
```

**`PdfDocumentMetadata`** — qhorus-local record for extracting report-specific metadata:

```java
public record PdfDocumentMetadata(
    String title,
    String author,
    Instant createdAt,
    String reportType,
    String tenancyId
) {
    public static PdfDocumentMetadata fromReport(Object report) {
        // Extract metadata from report model via pattern matching
    }
}
```

### HtmlReportRenderer Enhancement

New method `renderForPdf(Object report, PdfDocumentMetadata metadata)` adds `@page` CSS rules for PDF-quality output:

```java
public String renderForPdf(Object report, PdfDocumentMetadata metadata) {
    String body = switch (report) {
        case AttributionReport r -> renderAttribution(r);
        case ObligationReport r -> renderObligation(r);
        // ... existing switch cases
        default -> renderGeneric(report);
    };
    return wrapForPdf(body, metadata);
}

private String wrapForPdf(String body, PdfDocumentMetadata metadata) {
    // Wraps body HTML with:
    // - @page rules: margins, page numbers (counter(page)/counter(pages))
    // - Running header: report type + generation timestamp
    // - Running footer: tenant ID + schema version
    // - Liberation Mono font-family for <code>/<tt> elements
}
```

The existing `render()` method is unchanged — browser HTML doesn't need page structure.

---

## Platform Module Structure

```
casehub-platform/
├── platform-api/
│   └── src/main/java/io/casehub/platform/api/pdf/
│       ├── PdfGenerator.java            — SPI interface
│       ├── PdfOptions.java              — options record
│       ├── PdfAConformance.java         — enum
│       └── NoOpPdfGenerator.java        — @DefaultBean (returns Optional.empty())
├── platform-pdf/
│   ├── pom.xml
│   └── src/main/
│       ├── java/io/casehub/platform/pdf/
│       │   └── OpenHtmlToPdfGenerator.java  — @ApplicationScoped impl
│       └── resources/fonts/
│           ├── LiberationSans-Regular.ttf
│           ├── LiberationSans-Bold.ttf
│           ├── LiberationSans-Italic.ttf
│           ├── LiberationSans-BoldItalic.ttf
│           ├── LiberationMono-Regular.ttf
│           └── LiberationMono-Bold.ttf
```

---

## Changes to Existing Code

### `ReportFormat` enum

```java
public enum ReportFormat {
    JSON, CSV, HTML, PDF
}
```

### `ComplianceReportResource.renderResponse()`

Add `application/pdf` content negotiation:

```java
@Inject PdfReportRenderer pdfRenderer;

private Response renderResponse(Object report, String accept) {
    if (accept != null && accept.contains("application/pdf")) {
        return Response.ok(pdfRenderer.render(report))
                .header("Content-Type", "application/pdf").build();
    }
    // ... existing CSV, HTML, JSON branches unchanged
}
```

### `compliance-report/pom.xml`

Add dependency on `casehub-platform-pdf`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-pdf</artifactId>
  <version>${casehub-platform.version}</version>
</dependency>
```

---

## Bundled Fonts

Liberation Sans (Regular, Bold, Italic, BoldItalic) + Liberation Mono (Regular, Bold) bundled in `casehub-platform-pdf/src/main/resources/fonts/`. Apache 2.0 licensed, ~1.4MB total.

- **Liberation Sans** — metrically equivalent to Arial, covers Latin/Cyrillic/Greek. Used for body text, headers, table labels.
- **Liberation Mono** — monospaced, for UUIDs, correlation IDs, Merkle root hashes, trust scores, and entry IDs. Compliance auditors visually verify these values — monospaced rendering aids comparison and detection of transcription errors.

PDF/A-2b requires all fonts embedded. Bundled fonts guarantee reproducible output regardless of deployment environment (Docker containers, CI runners, GraalVM native image).

---

## PDF/A-2b Compliance

PDF/A-2b (ISO 19005-2, basic conformance) requirements:

| Requirement | How satisfied |
|---|---|
| All fonts embedded | Liberation Sans + Mono bundled; registered via `useFont()` |
| No external references | HTML uses inline CSS only; no external stylesheets or images |
| Color profile | sRGB ICC profile embedded by OpenHTMLtoPDF automatically |
| Document metadata | Set via `PdfDocumentMetadata` → PDFBox `PDDocumentInformation` |
| No JavaScript | HTML reports contain no scripts |
| No encryption | Reports are unencrypted |

**Validation:** Tests use Apache PDFBox Preflight to validate generated PDFs conform to PDF/A-2b. Setting the conformance flag is necessary but not sufficient — the validation ensures correct XMP metadata, font embedding, and colour space usage.

---

## Page Structure

Regulatory PDFs require structural elements for auditability:

| Element | Implementation |
|---|---|
| Page numbers | CSS `@page` with `counter(page)` / `counter(pages)` |
| Header | Report type + generation timestamp (running element) |
| Footer | Tenant ID + schema version (running element) |
| Margins | 2cm all sides for print quality |

Classification markings (CONFIDENTIAL, INTERNAL, PUBLIC) are not included — no current requirement. Can be added to the footer via `PdfDocumentMetadata` if deployers need them.

Bookmarks/PDF outline are not generated — OpenHTMLtoPDF supports bookmarks via CSS `-fs-bookmark-level` but the current report HTML structure (flat tables) doesn't have a natural outline hierarchy. Reports with multiple sections (ObligationReport: channels + agents) could benefit; deferred as a refinement.

---

## GraalVM Native Image

OpenHTMLtoPDF uses reflection internally (PDFBox, ICU4J text processing). Native image support requires:

1. **Reflection config:** `reflect-config.json` for OpenHTMLtoPDF and PDFBox classes. Generated via `native-image-agent` tracing during test execution.
2. **Resource registration:** Font files (`.ttf`) and ICC color profile must be registered as native image resources. The `casehub-platform-pdf` deployment module adds `NativeImageResourceBuildItem` entries for `fonts/*.ttf`.
3. **`META-INF/services`:** OpenHTMLtoPDF uses ServiceLoader — entries included automatically by Quarkus's native image support.

Native image compatibility must be verified during implementation — if OpenHTMLtoPDF + PDFBox cannot run in native image without excessive configuration, the library choice (D1) would need revisiting. JVM-only deployment is an acceptable fallback for compliance reporting (not a latency-sensitive path).

---

## Testing Strategy

| Component | Test type | Notes |
|---|---|---|
| `OpenHtmlToPdfGenerator` (platform) | CDI-free unit test | Verify PDF bytes from known HTML; verify document metadata (title, author, createdAt) via PDFBox `PDDocument` API; verify font embedding |
| `NoOpPdfGenerator` (platform) | CDI-free unit test | Verify returns `Optional.empty()` |
| `PdfReportRenderer` (qhorus) | CDI-free unit test | Mock `HtmlReportRenderer` + `PdfGenerator`; verify delegation, content type, `supports(PDF)` |
| `PdfDocumentMetadata.fromReport()` (qhorus) | CDI-free unit test | Verify extraction from each report type |
| `HtmlReportRenderer.renderForPdf()` | CDI-free unit test | Verify `@page` CSS rules present, page number counters, header/footer content |
| PDF/A-2b validation | CDI-free unit test | Use Apache PDFBox Preflight to validate conformance of generated PDF |
| REST content negotiation | `@QuarkusTest` | `Accept: application/pdf` returns PDF bytes with correct Content-Type |
| Scheduled PDF generation | CDI-free unit test | Verify `ReportFormat.PDF` flows through scheduler → storage service |
| Each report type → PDF | CDI-free unit test | Generate PDF for each report type (Attribution, Obligation, Violation, JudgmentAttribution, JudgmentFulfillment, PropertyVerification); verify non-empty, valid PDF header |
| SPI displacement | `@QuarkusTest @TestProfile` | Verify `NoOpPdfGenerator` displaced by `OpenHtmlToPdfGenerator` when platform-pdf on classpath |

---

## Deferred Work

| Item | Reason | Issue |
|------|--------|-------|
| Digital signatures (eIDAS) | Requires PKI infrastructure decisions | casehubio/qhorus#418 |
| PDF/UA accessibility | Compliance exports are internal audit records, not public services; OpenHTMLtoPDF cannot produce tagged PDF | — |
| PDF bookmarks/outline | Reports don't have natural outline hierarchy; refinement when needed | — |
| Platform-api-scope protocol revision | Protocol too conservative for infrastructure capabilities; needs category for general-purpose infrastructure vs domain types | — |
| CJK font support | No current requirement; add font bundles when needed | — |
| Classification markings | No deployer requirement; can be added to footer via metadata | — |

---

## References

- `ReportRenderer.java` (compliance-report/format/) — extension point
- `HtmlReportRenderer.java` (compliance-report/format/) — HTML rendering reused by PDF
- `ComplianceReportResource.java` (compliance-report/api/) — REST content negotiation
- `ComplianceReportScheduler.java` (compliance-report/schedule/) — scheduled generation
- issue-402 design spec — D3 deferral rationale
- issue-402 decisions.md — D3 "PDF deferred — JSON/CSV/HTML first"
- OpenHTMLtoPDF — `com.openhtmltopdf:openhtmltopdf-pdfbox`
- Liberation Fonts — Apache 2.0 font family (Sans + Mono)
- ISO 19005-2 — PDF/A-2b specification
- EU AI Act Article 12 — record-keeping requirements
- European Accessibility Act (Directive 2019/882) — considered, not applicable to internal audit records
- W3C CSS Paged Media Module — @page rules for headers/footers/page numbers
- Decision review R1-11 — single consumer concern; overridden for infrastructure-class capabilities
- Decision review R1-12 — NoOp must degrade gracefully (Optional.empty(), not throw)
- Decision review R1-18 — monospaced font for compliance data
- Decision review R1-21 — page structure for regulatory documents
