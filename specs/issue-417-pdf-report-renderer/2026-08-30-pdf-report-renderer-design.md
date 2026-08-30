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

All PDF rendering lives in `compliance-report/` — alongside the existing renderers. No platform module (D3 — single consumer today; extract when a second consumer materialises).

### Data Flow

```
Report model → HtmlReportRenderer.renderForPdf(report, metadata) → HTML String
    → HtmlToPdfConverter.convert(html, metadata) → PDF byte[]
```

The `renderForPdf()` method wraps the existing per-type HTML rendering with `@page` CSS rules for page numbers, headers, and footers. The standard `render()` method is unchanged.

### New Classes

**`PdfReportRenderer`** — `@ApplicationScoped`, implements `ReportRenderer`:

```java
@ApplicationScoped
public class PdfReportRenderer implements ReportRenderer {

    @Inject HtmlReportRenderer htmlRenderer;

    HtmlToPdfConverter converter;

    @PostConstruct
    void init() {
        converter = new HtmlToPdfConverter();
    }

    @Override
    public String contentType() {
        return "application/pdf";
    }

    @Override
    public byte[] render(Object report) {
        PdfDocumentMetadata metadata = PdfDocumentMetadata.fromReport(report);
        String html = htmlRenderer.renderForPdf(report, metadata);
        return converter.convert(html, metadata);
    }

    @Override
    public boolean supports(ReportFormat format) {
        return format == ReportFormat.PDF;
    }
}
```

**`HtmlToPdfConverter`** — package-private, not a CDI bean:

```java
class HtmlToPdfConverter {

    private static final List<FontResource> FONTS = loadFonts();

    byte[] convert(String html, PdfDocumentMetadata metadata) {
        PdfRendererBuilder builder = new PdfRendererBuilder();
        builder.usePdfAConformance(PdfAConformance.PDFA_2_B);
        builder.withHtmlContent(html, "/");

        for (FontResource font : FONTS) {
            builder.useFont(font.supplier(), font.family(),
                font.weight(), font.style(), true);
        }

        ByteArrayOutputStream os = new ByteArrayOutputStream();
        builder.toStream(os);
        builder.run();

        setDocumentMetadata(os.toByteArray(), metadata);
        return os.toByteArray();
    }

    private void setDocumentMetadata(byte[] pdf, PdfDocumentMetadata metadata) {
        // PDFBox PDDocument: set Title, Author, Subject, CreationDate
    }

    private static List<FontResource> loadFonts() {
        // Load Liberation Sans (Regular, Bold, Italic, BoldItalic)
        // + Liberation Mono (Regular, Bold) from classpath
    }
}
```

**`PdfDocumentMetadata`** — record:

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

Add OpenHTMLtoPDF dependency:

```xml
<dependency>
  <groupId>com.openhtmltopdf</groupId>
  <artifactId>openhtmltopdf-pdfbox</artifactId>
</dependency>
```

Version managed by the parent pom `<dependencyManagement>` section.

---

## Bundled Fonts

Liberation Sans (Regular, Bold, Italic, BoldItalic) + Liberation Mono (Regular, Bold) bundled in `compliance-report/src/main/resources/fonts/`. Apache 2.0 licensed, ~1.4MB total.

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
2. **Resource registration:** Font files (`.ttf`) and ICC color profile must be registered as native image resources. The `compliance-report` deployment module (or the existing `QhorusProcessor`) adds `NativeImageResourceBuildItem` entries for `fonts/*.ttf`.
3. **`META-INF/services`:** OpenHTMLtoPDF uses ServiceLoader — entries included automatically by Quarkus's native image support.

Native image compatibility must be verified during implementation — if OpenHTMLtoPDF + PDFBox cannot run in native image without excessive configuration, the library choice (D1) would need revisiting. JVM-only deployment is an acceptable fallback for compliance reporting (not a latency-sensitive path).

---

## Testing Strategy

| Component | Test type | Notes |
|---|---|---|
| `HtmlToPdfConverter` | CDI-free unit test | Verify PDF bytes from known HTML; verify document metadata (title, author, createdAt) via PDFBox `PDDocument` API; verify font embedding |
| `PdfReportRenderer` | CDI-free unit test | Mock `HtmlReportRenderer`; verify delegation, content type, `supports(PDF)` |
| `PdfDocumentMetadata.fromReport()` | CDI-free unit test | Verify extraction from each report type |
| `HtmlReportRenderer.renderForPdf()` | CDI-free unit test | Verify `@page` CSS rules present, page number counters, header/footer content |
| PDF/A-2b validation | CDI-free unit test | Use Apache PDFBox Preflight to validate conformance of generated PDF |
| REST content negotiation | `@QuarkusTest` | `Accept: application/pdf` returns PDF bytes with correct Content-Type |
| Scheduled PDF generation | CDI-free unit test | Verify `ReportFormat.PDF` flows through scheduler → storage service |
| Each report type → PDF | CDI-free unit test | Generate PDF for each report type (Attribution, Obligation, Violation, JudgmentAttribution, JudgmentFulfillment, PropertyVerification); verify non-empty, valid PDF header |

---

## Deferred Work

| Item | Reason | Issue |
|------|--------|-------|
| Digital signatures (eIDAS) | Requires PKI infrastructure decisions | casehubio/qhorus#418 |
| PDF/UA accessibility | Compliance exports are internal audit records, not public services; OpenHTMLtoPDF cannot produce tagged PDF | — |
| PDF bookmarks/outline | Reports don't have natural outline hierarchy; refinement when needed | — |
| Platform extraction | Extract `HtmlToPdfConverter` to platform when a second consumer appears | — |
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
- Decision review R1-11 — single consumer, platform extraction premature
- Decision review R1-18 — monospaced font for compliance data
- Decision review R1-21 — page structure for regulatory documents
