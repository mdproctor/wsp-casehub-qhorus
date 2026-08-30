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

`HtmlReportRenderer` already generates well-structured HTML with print-friendly CSS, per-report-type rendering (Attribution, Obligation, Violation, JudgmentAttribution, JudgmentFulfillment, PropertyVerification), and a generic fallback. The HTML output is the natural input for PDF conversion.

`ComplianceReportResource.renderResponse()` uses `Accept` header content negotiation with hardcoded renderer selection. `ComplianceReportScheduler.generateAndStore()` passes `ReportFormat` from the schedule entity.

---

## Architecture

PDF rendering is a general-purpose capability — not qhorus-specific. Any CaseHub module producing HTML reports could need PDF export. The architecture splits into two layers:

### Platform Layer: `casehub-platform-pdf`

New optional platform module owning the OpenHTMLtoPDF dependency and font resources. Exposes a single service.

**SPI in `casehub-platform-api`:**

```java
package io.casehub.platform.api.pdf;

public interface PdfGenerator {
    byte[] generateFromHtml(byte[] html, PdfOptions options);
}

public record PdfOptions(
    String title,
    String author,
    Instant createdAt,
    PdfAConformance conformance
) {
    public static PdfOptions defaults() {
        return new PdfOptions(null, null, null, PdfAConformance.PDFA_2_B);
    }
}

public enum PdfAConformance {
    PDFA_2_B
}
```

`@DefaultBean NoOpPdfGenerator` in `casehub-platform-api` throws `UnsupportedOperationException` — fail-fast when the PDF module is absent.

**Implementation in `casehub-platform-pdf`:**

```java
@ApplicationScoped
public class OpenHtmlToPdfGenerator implements PdfGenerator {

    private List<FontResource> fonts;

    @PostConstruct
    void loadFonts() {
        // Load bundled Liberation Sans from classpath resources
    }

    @Override
    public byte[] generateFromHtml(byte[] html, PdfOptions options) {
        PdfRendererBuilder builder = new PdfRendererBuilder();
        builder.usePdfAConformance(PdfAConformance.PDFA_2_B);
        builder.withHtmlContent(new String(html, StandardCharsets.UTF_8), "/");
        // Register fonts
        // Set document metadata (title, author, createdAt) via PDFBox API
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        builder.toStream(os);
        builder.run();
        return os.toByteArray();
    }
}
```

**Bundled fonts:** Liberation Sans (Regular, Bold, Italic, BoldItalic) — Apache 2.0 licensed, metrically equivalent to Arial, ~1MB total. Registered at `@PostConstruct`. PDF/A-2b requires all fonts embedded; bundled fonts guarantee reproducible output regardless of deployment environment (Docker containers, CI runners).

**Dependencies:**

```xml
<dependency>
  <groupId>com.openhtmltopdf</groupId>
  <artifactId>openhtmltopdf-pdfbox</artifactId>
</dependency>
```

### Qhorus Layer: `PdfReportRenderer` in `compliance-report/`

Thin adapter composing `HtmlReportRenderer` and `PdfGenerator`:

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
        byte[] html = htmlRenderer.render(report);
        return pdfGenerator.generateFromHtml(html,
            PdfOptions.defaults());
    }

    @Override
    public boolean supports(ReportFormat format) {
        return format == ReportFormat.PDF;
    }
}
```

Report-specific metadata (title, author, timestamp) can be extracted from the report object for richer PDF document properties. This is a refinement — the basic flow works with `PdfOptions.defaults()`.

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
private Response renderResponse(Object report, String accept) {
    if (accept != null && accept.contains("application/pdf")) {
        return Response.ok(pdfRenderer.render(report))
                .header("Content-Type", "application/pdf").build();
    }
    if (accept != null && accept.contains("text/csv")) {
        return Response.ok(csvRenderer.render(report))
                .header("Content-Type", "text/csv").build();
    }
    if (accept != null && accept.contains("text/html")) {
        return Response.ok(htmlRenderer.render(report))
                .header("Content-Type", "text/html").build();
    }
    return Response.ok(jsonRenderer.render(report))
            .header("Content-Type", "application/json").build();
}
```

Inject `PdfReportRenderer pdfRenderer` alongside the existing renderers.

### `ComplianceReportScheduler`

No changes needed — scheduled reports already pass `ReportFormat` from the schedule entity. When a schedule has `format=PDF`, `ComplianceReportStorageService.store()` renders the report using the matching renderer. The storage service already uses CDI `Instance<ReportRenderer>` iteration (or should — verify during implementation).

If `ComplianceReportStorageService` doesn't use renderer selection today (it may store JSON unconditionally), this needs a small change: stored reports should use the schedule's requested format for the initial render, while retrieval re-renders from stored JSON via `Accept` header (per the #402 spec).

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

## Platform Module Structure

```
casehub-platform/
├── platform-api/
│   └── src/main/java/io/casehub/platform/api/pdf/
│       ├── PdfGenerator.java            — SPI interface
│       ├── PdfOptions.java              — options record
│       ├── PdfAConformance.java         — enum
│       └── NoOpPdfGenerator.java        — @DefaultBean (throws UnsupportedOperationException)
├── platform-pdf/
│   ├── pom.xml
│   └── src/main/
│       ├── java/io/casehub/platform/pdf/
│       │   └── OpenHtmlToPdfGenerator.java  — @ApplicationScoped impl
│       └── resources/fonts/
│           ├── LiberationSans-Regular.ttf
│           ├── LiberationSans-Bold.ttf
│           ├── LiberationSans-Italic.ttf
│           └── LiberationSans-BoldItalic.ttf
```

---

## PDF/A-2b Compliance

PDF/A-2b (ISO 19005-2, basic conformance) requirements met by this design:

| Requirement | How satisfied |
|---|---|
| All fonts embedded | Liberation Sans bundled; registered via `useFont()` |
| No external references | HTML uses inline CSS only; no external stylesheets or images |
| Color profile | sRGB ICC profile embedded by OpenHTMLtoPDF automatically |
| Document metadata | Set via `PdfOptions` → PDFBox `PDDocumentInformation` |
| No JavaScript | HTML reports contain no scripts |
| No encryption | Reports are unencrypted |

---

## GraalVM Native Image

OpenHTMLtoPDF uses reflection internally. Native image support requires:

1. **Reflection config:** `reflect-config.json` for OpenHTMLtoPDF and PDFBox classes. Generated via `native-image-agent` tracing or manual registration.
2. **Resource registration:** Font files and ICC color profile must be registered as native image resources. The platform deployment module adds a `@BuildStep` for `NativeImageResourceBuildItem` covering `fonts/*.ttf`.
3. **`META-INF/services`:** OpenHTMLtoPDF uses ServiceLoader — entries must be included in native image.

This follows the established pattern — `QhorusProcessor.registerMigrationResources()` already registers Flyway SQL files for native image.

---

## Testing Strategy

| Component | Test type | Notes |
|---|---|---|
| `OpenHtmlToPdfGenerator` | CDI-free unit test | Verify PDF bytes produced from known HTML input; verify PDF/A-2b metadata (title, author) via PDFBox `PDDocument` API; verify font embedding |
| `PdfReportRenderer` | CDI-free unit test | Mock `HtmlReportRenderer` + `PdfGenerator`; verify delegation and content type |
| `NoOpPdfGenerator` | CDI-free unit test | Verify throws `UnsupportedOperationException` |
| REST content negotiation | `@QuarkusTest` | `Accept: application/pdf` returns PDF bytes with correct Content-Type |
| Scheduled PDF generation | CDI-free unit test | Verify `ReportFormat.PDF` flows through scheduler → storage |
| PDF/A-2b validation | CDI-free unit test | Use Apache PDFBox Preflight to validate generated PDF conforms to PDF/A-2b |
| SPI displacement | `@QuarkusTest @TestProfile` | Verify `NoOpPdfGenerator` displaced by `OpenHtmlToPdfGenerator` when platform-pdf on classpath |

Platform module tests are self-contained — no qhorus dependency. Qhorus integration tests verify the composition.

---

## Deferred Work

| Item | Reason | Issue |
|------|--------|-------|
| Digital signatures (eIDAS) | Requires PKI infrastructure decisions | casehubio/qhorus#418 |
| CJK font support | No current requirement; add font bundles when needed | — |
| Streaming PDF generation | Reports are <1MB; in-memory is sufficient | — |
| PDF/A-3b with embedded attachments | JSON/CSV already served via REST; embedding is redundant | — |

---

## References

- `ReportRenderer.java` (compliance-report/format/) — extension point
- `HtmlReportRenderer.java` (compliance-report/format/) — HTML rendering reused by PDF
- `ComplianceReportResource.java` (compliance-report/api/) — REST content negotiation
- `ComplianceReportScheduler.java` (compliance-report/schedule/) — scheduled generation
- issue-402 design spec — D3 deferral rationale
- issue-402 decisions.md — D3 "PDF deferred — JSON/CSV/HTML first"
- OpenHTMLtoPDF — `com.openhtmltopdf:openhtmltopdf-pdfbox`
- Liberation Sans — Apache 2.0 font family
- ISO 19005-2 — PDF/A-2b specification
- EU AI Act Article 12 — record-keeping requirements
