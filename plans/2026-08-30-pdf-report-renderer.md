# PDF Report Renderer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #417 — feat: PDF rendering for compliance evidence export reports
**Issue group:** #417

**Goal:** Add PDF/A-2b rendering to the compliance evidence export module, enabling regulatory-grade PDF output for all report types.

**Architecture:** `PdfReportRenderer` composes `HtmlReportRenderer` (enhanced with `@page` CSS for PDF) with a package-private `HtmlToPdfConverter` wrapping OpenHTMLtoPDF. All code lives in `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/`. Bundled Liberation Sans + Mono fonts in `compliance-report/src/main/resources/fonts/`.

**Tech Stack:** OpenHTMLtoPDF (`com.openhtmltopdf:openhtmltopdf-pdfbox`), Apache PDFBox (transitive — used for metadata and test validation), Liberation Fonts (Apache 2.0)

## Global Constraints

- Java 21 source, Java 26 JVM, Quarkus 3.32.2
- PDF/A-2b conformance (ISO 19005-2, basic)
- OpenHTMLtoPDF version managed in parent pom `<dependencyManagement>`
- All fonts embedded — bundled Liberation Sans + Mono, no system font dependency
- No changes to existing `render()` method on `HtmlReportRenderer` — browser HTML path unchanged
- `ComplianceReportStorageService.store()` always stores JSON — `ReportFormat.PDF` on a schedule is metadata only (audit trail); the stored body is always JSON, re-rendered on retrieval

---

## Batch 1: Core PDF conversion infrastructure

### Task 1: PdfDocumentMetadata record + HtmlToPdfConverter

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/PdfDocumentMetadata.java`
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlToPdfConverter.java`
- Create: `compliance-report/src/main/resources/fonts/LiberationSans-Regular.ttf`
- Create: `compliance-report/src/main/resources/fonts/LiberationSans-Bold.ttf`
- Create: `compliance-report/src/main/resources/fonts/LiberationSans-Italic.ttf`
- Create: `compliance-report/src/main/resources/fonts/LiberationSans-BoldItalic.ttf`
- Create: `compliance-report/src/main/resources/fonts/LiberationMono-Regular.ttf`
- Create: `compliance-report/src/main/resources/fonts/LiberationMono-Bold.ttf`
- Modify: `pom.xml` (parent — add OpenHTMLtoPDF version property + dependencyManagement)
- Modify: `compliance-report/pom.xml` (add openhtmltopdf-pdfbox dependency)
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/HtmlToPdfConverterTest.java`

**Interfaces:**
- Produces: `PdfDocumentMetadata(String title, String author, Instant createdAt, String reportType, String tenancyId)` record with `static PdfDocumentMetadata fromReport(Object report)` factory
- Produces: `HtmlToPdfConverter` (package-private) with `byte[] convert(String html, PdfDocumentMetadata metadata)`

- [ ] **Step 1: Add OpenHTMLtoPDF dependency to parent pom**

Add version property and dependencyManagement entry to `pom.xml` (project root):

```xml
<!-- In <properties> -->
<openhtmltopdf.version>1.1.24</openhtmltopdf.version>

<!-- In <dependencyManagement><dependencies> -->
<dependency>
  <groupId>com.openhtmltopdf</groupId>
  <artifactId>openhtmltopdf-pdfbox</artifactId>
  <version>${openhtmltopdf.version}</version>
</dependency>
```

Add compile dependency to `compliance-report/pom.xml`:

```xml
<dependency>
  <groupId>com.openhtmltopdf</groupId>
  <artifactId>openhtmltopdf-pdfbox</artifactId>
</dependency>
```

- [ ] **Step 2: Download Liberation fonts**

Download Liberation Sans (Regular, Bold, Italic, BoldItalic) and Liberation Mono (Regular, Bold) TTF files from https://github.com/liberationfonts/liberation-fonts/releases and place in `compliance-report/src/main/resources/fonts/`.

Verify font files exist:
```bash
ls compliance-report/src/main/resources/fonts/
```
Expected: 6 `.ttf` files (~1.4MB total)

- [ ] **Step 3: Write PdfDocumentMetadata record**

```java
package io.casehub.qhorus.compliance.format;

import io.casehub.qhorus.compliance.model.AttributionReport;
import io.casehub.qhorus.compliance.model.JudgmentAttributionReport;
import io.casehub.qhorus.compliance.model.JudgmentFulfillmentReport;
import io.casehub.qhorus.compliance.model.ObligationReport;
import io.casehub.qhorus.compliance.model.ViolationReport;

import java.time.Instant;

public record PdfDocumentMetadata(
        String title,
        String author,
        Instant createdAt,
        String reportType,
        String tenancyId) {

    public static PdfDocumentMetadata fromReport(Object report) {
        Instant now = Instant.now();
        return switch (report) {
            case AttributionReport r -> new PdfDocumentMetadata(
                    "Attribution Report — " + r.correlationId(),
                    "CaseHub Compliance", r.generatedAt(), "ATTRIBUTION", null);
            case ObligationReport r -> new PdfDocumentMetadata(
                    "Obligation Fulfillment Report",
                    "CaseHub Compliance", r.generatedAt(), "OBLIGATION", null);
            case ViolationReport r -> new PdfDocumentMetadata(
                    "Violation Report — " + r.channelName(),
                    "CaseHub Compliance", r.generatedAt(), "VIOLATION", null);
            case JudgmentAttributionReport r -> new PdfDocumentMetadata(
                    "Judgment Attribution Report — " + r.judgmentId(),
                    "CaseHub Compliance", r.generatedAt(), "JUDGMENT_ATTRIBUTION", null);
            case JudgmentFulfillmentReport r -> new PdfDocumentMetadata(
                    "Judgment Fulfillment Report",
                    "CaseHub Compliance", r.generatedAt(), "JUDGMENT_FULFILLMENT", null);
            default -> new PdfDocumentMetadata(
                    "Compliance Report", "CaseHub Compliance", now,
                    report.getClass().getSimpleName(), null);
        };
    }
}
```

- [ ] **Step 4: Write failing test for HtmlToPdfConverter**

```java
package io.casehub.qhorus.compliance.format;

import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.PDDocumentInformation;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;

class HtmlToPdfConverterTest {

    final HtmlToPdfConverter converter = new HtmlToPdfConverter();

    @Test
    void convert_simpleHtml_producesPdfBytes() {
        String html = "<!DOCTYPE html><html><head><style>"
                + "body { font-family: 'Liberation Sans', sans-serif; }"
                + "</style></head><body><h1>Test</h1>"
                + "<table><tr><td>Value</td></tr></table></body></html>";
        var metadata = new PdfDocumentMetadata(
                "Test Report", "Test Author", Instant.now(), "TEST", "tenant-1");

        byte[] pdf = converter.convert(html, metadata);

        assertThat(pdf).isNotEmpty();
        assertThat(pdf[0]).isEqualTo((byte) '%');
        assertThat(pdf[1]).isEqualTo((byte) 'P');
        assertThat(pdf[2]).isEqualTo((byte) 'D');
        assertThat(pdf[3]).isEqualTo((byte) 'F');
    }

    @Test
    void convert_setsDocumentMetadata() throws IOException {
        String html = "<!DOCTYPE html><html><head></head><body>Content</body></html>";
        var metadata = new PdfDocumentMetadata(
                "Obligation Report", "CaseHub Compliance",
                Instant.parse("2026-08-30T12:00:00Z"), "OBLIGATION", "tenant-1");

        byte[] pdf = converter.convert(html, metadata);

        try (PDDocument doc = PDDocument.load(pdf)) {
            PDDocumentInformation info = doc.getDocumentInformation();
            assertThat(info.getTitle()).isEqualTo("Obligation Report");
            assertThat(info.getAuthor()).isEqualTo("CaseHub Compliance");
        }
    }

    @Test
    void convert_embedsFonts() throws IOException {
        String html = "<!DOCTYPE html><html><head><style>"
                + "body { font-family: 'Liberation Sans', sans-serif; }"
                + "</style></head><body><p>Font test</p></body></html>";
        var metadata = new PdfDocumentMetadata(
                "Test", "Test", Instant.now(), "TEST", null);

        byte[] pdf = converter.convert(html, metadata);

        try (PDDocument doc = PDDocument.load(pdf)) {
            assertThat(doc.getPages().getCount()).isGreaterThanOrEqualTo(1);
            var fonts = doc.getPage(0).getResources().getFontNames();
            assertThat(fonts).isNotNull();
        }
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=HtmlToPdfConverterTest -pl compliance-report -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: compilation failure — `HtmlToPdfConverter` does not exist

- [ ] **Step 6: Implement HtmlToPdfConverter**

```java
package io.casehub.qhorus.compliance.format;

import com.openhtmltopdf.pdfboxout.PdfRendererBuilder;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.PDDocumentInformation;

import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.Calendar;
import java.util.List;
import java.util.TimeZone;

class HtmlToPdfConverter {

    private static final List<FontDef> FONTS = List.of(
            new FontDef("LiberationSans-Regular.ttf", "Liberation Sans", 400, false),
            new FontDef("LiberationSans-Bold.ttf", "Liberation Sans", 700, false),
            new FontDef("LiberationSans-Italic.ttf", "Liberation Sans", 400, true),
            new FontDef("LiberationSans-BoldItalic.ttf", "Liberation Sans", 700, true),
            new FontDef("LiberationMono-Regular.ttf", "Liberation Mono", 400, false),
            new FontDef("LiberationMono-Bold.ttf", "Liberation Mono", 700, false));

    byte[] convert(String html, PdfDocumentMetadata metadata) {
        try {
            ByteArrayOutputStream os = new ByteArrayOutputStream();
            PdfRendererBuilder builder = new PdfRendererBuilder();
            builder.usePdfAConformance(PdfRendererBuilder.PdfAConformance.PDFA_2_B);
            builder.withHtmlContent(html, "/");
            builder.toStream(os);

            for (FontDef font : FONTS) {
                try (InputStream is = getClass().getResourceAsStream("/fonts/" + font.file())) {
                    if (is != null) {
                        builder.useFont(() -> getClass().getResourceAsStream("/fonts/" + font.file()),
                                font.family(), font.weight(),
                                font.italic()
                                        ? com.openhtmltopdf.outputdevice.helper.BaseRendererBuilder.FontStyle.ITALIC
                                        : com.openhtmltopdf.outputdevice.helper.BaseRendererBuilder.FontStyle.NORMAL,
                                true);
                    }
                }
            }

            builder.run();
            byte[] pdfBytes = os.toByteArray();
            return setMetadata(pdfBytes, metadata);
        } catch (Exception e) {
            throw new IllegalStateException("PDF generation failed", e);
        }
    }

    private byte[] setMetadata(byte[] pdfBytes, PdfDocumentMetadata metadata) {
        try (PDDocument doc = PDDocument.load(pdfBytes)) {
            PDDocumentInformation info = new PDDocumentInformation();
            if (metadata.title() != null) info.setTitle(metadata.title());
            if (metadata.author() != null) info.setAuthor(metadata.author());
            if (metadata.reportType() != null) info.setSubject(metadata.reportType());
            if (metadata.createdAt() != null) {
                Calendar cal = Calendar.getInstance(TimeZone.getTimeZone("UTC"));
                cal.setTimeInMillis(metadata.createdAt().toEpochMilli());
                info.setCreationDate(cal);
            }
            doc.setDocumentInformation(info);
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            doc.save(out);
            return out.toByteArray();
        } catch (Exception e) {
            throw new IllegalStateException("Failed to set PDF metadata", e);
        }
    }

    private record FontDef(String file, String family, int weight, boolean italic) {}
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=HtmlToPdfConverterTest -pl compliance-report -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: 3 tests PASS

- [ ] **Step 8: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/format/PdfDocumentMetadata.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlToPdfConverter.java
git add compliance-report/src/main/resources/fonts/
git add compliance-report/src/test/java/io/casehub/qhorus/compliance/format/HtmlToPdfConverterTest.java
git add compliance-report/pom.xml pom.xml
git commit -m "feat(#417): HtmlToPdfConverter with Liberation fonts and PDF/A-2b

Adds package-private HtmlToPdfConverter using OpenHTMLtoPDF to convert
HTML to PDF/A-2b. Bundles Liberation Sans + Mono fonts for reproducible
rendering. PdfDocumentMetadata record extracts title/author/timestamp
from report model objects.

Refs #417"
```

---

## Batch 2: HtmlReportRenderer PDF enhancement + PdfReportRenderer

### Task 2: HtmlReportRenderer.renderForPdf() with @page CSS

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlReportRenderer.java`
- Modify: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/HtmlReportRendererTest.java`

**Interfaces:**
- Consumes: `PdfDocumentMetadata(String title, String author, Instant createdAt, String reportType, String tenancyId)`
- Produces: `String renderForPdf(Object report, PdfDocumentMetadata metadata)` — public method on `HtmlReportRenderer`

- [ ] **Step 1: Write failing test for renderForPdf()**

Add to `HtmlReportRendererTest.java`:

```java
@Test
void renderForPdf_includesPageCssRules() {
    var report = new AttributionReport(
            "corr-1", null, 0, List.of(), null, "OPEN",
            List.of(), List.of(), null, Instant.now(), 1);
    var metadata = new PdfDocumentMetadata(
            "Attribution Report", "CaseHub Compliance",
            Instant.now(), "ATTRIBUTION", "tenant-1");

    String html = renderer.renderForPdf(report, metadata);

    assertThat(html).contains("@page");
    assertThat(html).contains("counter(page)");
    assertThat(html).contains("counter(pages)");
    assertThat(html).contains("Attribution Report");
}

@Test
void renderForPdf_includesHeaderAndFooter() {
    var report = new ObligationReport(
            Instant.now(), Instant.now(), List.of(), List.of(),
            0, 0, 0, 0, 0, 0, 0, 0.0, null, null, Instant.now(), 1);
    var metadata = new PdfDocumentMetadata(
            "Obligation Report", "CaseHub Compliance",
            Instant.parse("2026-08-30T12:00:00Z"), "OBLIGATION", "tenant-1");

    String html = renderer.renderForPdf(report, metadata);

    assertThat(html).contains("OBLIGATION");
    assertThat(html).contains("tenant-1");
}

@Test
void renderForPdf_containsReportBody() {
    var node = new AttributionNode(
            "e1", "ch1", "channel-a", "COMMAND", "actor-1",
            "2026-08-27T10:00:00Z", "do X", null, 0,
            0.85, "SOUND", null, null, null);
    var report = new AttributionReport(
            "corr-1", "e1", 1, List.of("channel-a"), 500L, "FULFILLED",
            List.of(node), List.of(), "root", Instant.now(), 1);
    var metadata = new PdfDocumentMetadata(
            "Attribution Report", "CaseHub Compliance",
            Instant.now(), "ATTRIBUTION", null);

    String html = renderer.renderForPdf(report, metadata);

    assertThat(html).contains("channel-a");
    assertThat(html).contains("COMMAND");
    assertThat(html).contains("<table>");
}

@Test
void render_unchanged_noPdfCss() {
    var report = new AttributionReport(
            "corr-1", null, 0, List.of(), null, "OPEN",
            List.of(), List.of(), null, Instant.now(), 1);

    String html = new String(renderer.render(report), java.nio.charset.StandardCharsets.UTF_8);

    assertThat(html).doesNotContain("@page");
    assertThat(html).doesNotContain("counter(page)");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=HtmlReportRendererTest -pl compliance-report -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: compilation failure — `renderForPdf` method does not exist

- [ ] **Step 3: Implement renderForPdf()**

Add to `HtmlReportRenderer.java`:

```java
public String renderForPdf(Object report, PdfDocumentMetadata metadata) {
    String body = switch (report) {
        case AttributionReport r -> renderAttribution(r);
        case ObligationReport r -> renderObligation(r);
        case ViolationReport r -> renderViolation(r);
        case JudgmentAttributionReport r -> renderJudgmentAttribution(r);
        case JudgmentFulfillmentReport r -> renderJudgmentFulfillment(r);
        default -> renderGeneric(report);
    };
    return wrapForPdf(body, metadata);
}

private String wrapForPdf(String bodyContent, PdfDocumentMetadata metadata) {
    String title = metadata.title() != null ? esc(metadata.title()) : "Compliance Report";
    String reportType = metadata.reportType() != null ? esc(metadata.reportType()) : "";
    String timestamp = metadata.createdAt() != null ? metadata.createdAt().toString() : "";
    String tenant = metadata.tenancyId() != null ? esc(metadata.tenancyId()) : "";

    return "<!DOCTYPE html>\n<html><head><meta charset=\"UTF-8\">\n<title>"
            + title + "</title>\n<style>\n" + CSS + "\n"
            + PDF_PAGE_CSS
            + "\n</style>\n</head><body>\n"
            + "<div class=\"header-content\">" + reportType + " — " + timestamp + "</div>\n"
            + "<div class=\"footer-content\">Tenant: " + tenant + " | Page <span class=\"page-number\"></span> of <span class=\"page-count\"></span></div>\n"
            + "<h1>" + title + "</h1>\n"
            + bodyContent
            + "\n</body></html>";
}
```

Add the `PDF_PAGE_CSS` constant alongside the existing `CSS` constant:

```java
private static final String PDF_PAGE_CSS = """
        @page {
            margin: 2cm;
            @top-center {
                content: element(running-header);
            }
            @bottom-center {
                content: element(running-footer);
            }
            @bottom-right {
                content: "Page " counter(page) " of " counter(pages);
                font-size: 9pt;
                font-family: 'Liberation Sans', sans-serif;
            }
        }
        .header-content {
            position: running(running-header);
            font-size: 9pt;
            font-family: 'Liberation Sans', sans-serif;
            color: #666;
            text-align: center;
        }
        .footer-content {
            position: running(running-footer);
            font-size: 8pt;
            font-family: 'Liberation Sans', sans-serif;
            color: #999;
            text-align: center;
        }
        code, tt, .mono {
            font-family: 'Liberation Mono', monospace;
        }
        """;
```

The `wrapForPdf` method replaces the `header()` call used by `render()`. The body rendering methods (`renderAttribution`, etc.) are reused unchanged — they produce the table/content HTML without the `<html>` wrapper.

**Note:** The existing private render methods (`renderAttribution`, `renderObligation`, etc.) each call `header()` and `footer()` internally. To reuse them from `renderForPdf()`, extract the body content. The simplest approach: the existing render methods stay as-is for `render()`, and `renderForPdf()` calls them, then strips the `<!DOCTYPE html>...<body>` wrapper and `</body></html>` suffix — or better, extract the body-only rendering into private helper methods and have both `render()` and `renderForPdf()` call them.

Choose the extraction approach: refactor each render method (e.g. `renderAttribution`) to return body-only HTML, then have the existing `render()` call `header() + renderAttribution() + footer()` while `renderForPdf()` calls `wrapForPdf(renderAttribution(), metadata)`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=HtmlReportRendererTest -pl compliance-report -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: all tests PASS (including existing tests — `render()` behaviour unchanged)

- [ ] **Step 5: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlReportRenderer.java
git add compliance-report/src/test/java/io/casehub/qhorus/compliance/format/HtmlReportRendererTest.java
git commit -m "feat(#417): HtmlReportRenderer.renderForPdf() with @page CSS

Adds renderForPdf(report, metadata) method with CSS @page rules for
page numbers, running headers (report type + timestamp), and running
footers (tenant ID). Liberation Mono font-family for code/mono elements.
Existing render() method unchanged.

Refs #417"
```

### Task 3: PdfReportRenderer + ReportFormat.PDF

**Files:**
- Create: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/PdfReportRenderer.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportFormat.java`
- Test: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/PdfReportRendererTest.java`

**Interfaces:**
- Consumes: `HtmlReportRenderer.renderForPdf(Object report, PdfDocumentMetadata metadata)` → `String`
- Consumes: `HtmlToPdfConverter.convert(String html, PdfDocumentMetadata metadata)` → `byte[]`
- Consumes: `PdfDocumentMetadata.fromReport(Object report)` → `PdfDocumentMetadata`
- Produces: `PdfReportRenderer` implements `ReportRenderer` — `contentType()` = `"application/pdf"`, `supports(ReportFormat.PDF)` = true

- [ ] **Step 1: Add PDF to ReportFormat enum**

Modify `compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportFormat.java`:

```java
public enum ReportFormat {
    JSON, CSV, HTML, PDF
}
```

- [ ] **Step 2: Write failing test for PdfReportRenderer**

```java
package io.casehub.qhorus.compliance.format;

import io.casehub.qhorus.compliance.model.AttributionNode;
import io.casehub.qhorus.compliance.model.AttributionReport;
import io.casehub.qhorus.compliance.model.ReportFormat;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class PdfReportRendererTest {

    final HtmlReportRenderer htmlRenderer = new HtmlReportRenderer();
    final PdfReportRenderer renderer;

    PdfReportRendererTest() {
        renderer = new PdfReportRenderer();
        renderer.htmlRenderer = htmlRenderer;
        renderer.init();
    }

    @Test
    void contentType_isPdf() {
        assertThat(renderer.contentType()).isEqualTo("application/pdf");
    }

    @Test
    void supports_pdf() {
        assertThat(renderer.supports(ReportFormat.PDF)).isTrue();
        assertThat(renderer.supports(ReportFormat.HTML)).isFalse();
        assertThat(renderer.supports(ReportFormat.JSON)).isFalse();
        assertThat(renderer.supports(ReportFormat.CSV)).isFalse();
    }

    @Test
    void render_attributionReport_producesPdf() {
        var node = new AttributionNode(
                "e1", "ch1", "channel-a", "COMMAND", "actor-1",
                "2026-08-27T10:00:00Z", "do X", null, 0,
                0.85, "SOUND", null, null, null);
        var report = new AttributionReport(
                "corr-1", "e1", 1, List.of("channel-a"), 500L, "FULFILLED",
                List.of(node), List.of(), "root", Instant.now(), 1);

        byte[] pdf = renderer.render(report);

        assertThat(pdf).isNotEmpty();
        assertThat(new String(pdf, 0, 4)).isEqualTo("%PDF");
    }

    @Test
    void render_obligationReport_producesPdf() {
        var report = new io.casehub.qhorus.compliance.model.ObligationReport(
                Instant.now(), Instant.now(), List.of(), List.of(),
                10, 8, 1, 1, 0, 0, 0, 0.8, null, null, Instant.now(), 1);

        byte[] pdf = renderer.render(report);

        assertThat(pdf).isNotEmpty();
        assertThat(new String(pdf, 0, 4)).isEqualTo("%PDF");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PdfReportRendererTest -pl compliance-report -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: compilation failure — `PdfReportRenderer` does not exist

- [ ] **Step 4: Implement PdfReportRenderer**

```java
package io.casehub.qhorus.compliance.format;

import io.casehub.qhorus.compliance.model.ReportFormat;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

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

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PdfReportRendererTest -pl compliance-report -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: all 4 tests PASS

- [ ] **Step 6: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/format/PdfReportRenderer.java
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/model/ReportFormat.java
git add compliance-report/src/test/java/io/casehub/qhorus/compliance/format/PdfReportRendererTest.java
git commit -m "feat(#417): PdfReportRenderer + ReportFormat.PDF

PdfReportRenderer composes HtmlReportRenderer.renderForPdf() with
HtmlToPdfConverter to produce PDF/A-2b output. Adds PDF value to
ReportFormat enum.

Refs #417"
```

---

## Batch 3: REST content negotiation + full build verification

### Task 4: Wire PDF into ComplianceReportResource + full build

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java`

**Interfaces:**
- Consumes: `PdfReportRenderer` — `@ApplicationScoped`, injectable

- [ ] **Step 1: Add PdfReportRenderer injection and content negotiation**

Add to `ComplianceReportResource.java`:

```java
@Inject PdfReportRenderer pdfRenderer;
```

Modify `renderResponse()`:

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

- [ ] **Step 2: Run compliance-report module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl compliance-report -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: all tests PASS (existing + new)

- [ ] **Step 3: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS across all modules. This catches any compile errors in `examples/` modules that may reference `ReportFormat` (they shouldn't, but the full build verifies).

- [ ] **Step 4: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java
git commit -m "feat(#417): PDF content negotiation in ComplianceReportResource

Accept: application/pdf now returns PDF/A-2b rendered compliance reports.
Existing JSON, CSV, and HTML paths unchanged.

Closes #417"
```

---

## References

- `2026-08-30-pdf-report-renderer-design.md` — design spec this plan implements
- `ReportRenderer.java` (compliance-report/format/:5) — extension point interface
- `HtmlReportRenderer.java` (compliance-report/format/:17) — HTML renderer to enhance
- `ComplianceReportResource.java` (compliance-report/api/:31) — REST content negotiation
- `ComplianceReportStorageService.java` (compliance-report/storage/:16) — stores JSON always, format is metadata
- `HtmlReportRendererTest.java` (compliance-report format test) — existing test patterns
- `ReportFormat.java` (compliance-report/model/:3) — enum to extend
- `ObligationReport.java` (compliance-report/model/:8) — report record example
- `decisions.md` — D1-D6 design decisions
- GitHub #417 — feat: PDF rendering for compliance evidence export reports
- GitHub #402 — parent issue (compliance evidence export)
