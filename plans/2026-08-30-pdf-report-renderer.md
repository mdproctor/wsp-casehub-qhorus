# PDF Report Renderer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #417 — feat: PDF rendering for compliance evidence export reports
**Issue group:** #417

**Goal:** Add PDF/A-2b rendering to compliance evidence export via a platform-level PDF generation service and a qhorus adapter.

**Architecture:** Two-layer split — platform owns HTML-to-PDF conversion infrastructure (`casehub-platform-pdf` module with `PdfGenerator` SPI in `platform-api`), qhorus owns the compliance-specific `PdfReportRenderer` adapter in `compliance-report/`. The qhorus-local implementation is already working (HtmlToPdfConverter, PdfReportRenderer, HtmlReportRenderer.renderForPdf, fonts, tests, content negotiation, ReportFormat.PDF). This plan extracts the PDF infrastructure to platform and adapts qhorus to consume it.

**Tech Stack:** OpenHTMLtoPDF 1.1.37 (`io.github.openhtmltopdf:openhtmltopdf-pdfbox`), PDFBox (transitive), Liberation Sans + Mono fonts, Java 21, Quarkus 3.32.2.

## Global Constraints

- OpenHTMLtoPDF version `1.1.37` (already in qhorus parent pom `<dependencyManagement>`)
- PDF/A-2b conformance (ISO 19005-2, basic)
- Platform version `0.2-SNAPSHOT` for all `casehub-platform-*` dependencies
- Fonts: Liberation Sans (Regular/Bold/Italic/BoldItalic) + Liberation Mono (Regular/Bold), Apache 2.0
- `@DefaultBean NoOpPdfGenerator` returns `Optional.empty()` — graceful degradation, never throws
- `PdfGenerator.generateFromHtml()` takes `String` html, not `byte[]`

---

## Batch 1: Platform PDF infrastructure

### Task 1: PdfGenerator SPI in platform-api

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/pdf/PdfGenerator.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/pdf/PdfOptions.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/pdf/PdfAConformance.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/pdf/NoOpPdfGenerator.java`
- Test: `platform-api/src/test/java/io/casehub/platform/api/pdf/NoOpPdfGeneratorTest.java`

**Interfaces:**
- Produces: `PdfGenerator.generateFromHtml(String html, PdfOptions options)` returning `Optional<byte[]>`, `PdfOptions.defaults()`, `PdfAConformance.PDFA_2_B`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.platform.api.pdf;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class NoOpPdfGeneratorTest {

    @Test
    void generateFromHtml_returnsEmpty() {
        var noOp = new NoOpPdfGenerator();
        var result = noOp.generateFromHtml("<html></html>", PdfOptions.defaults());
        assertThat(result).isEmpty();
    }

    @Test
    void pdfOptionsDefaults_hasPdfA2b() {
        var opts = PdfOptions.defaults();
        assertThat(opts.conformance()).isEqualTo(PdfAConformance.PDFA_2_B);
        assertThat(opts.title()).isNull();
        assertThat(opts.author()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/platform-api/pom.xml -Dtest=NoOpPdfGeneratorTest -pl .`
Expected: FAIL — classes don't exist yet

- [ ] **Step 3: Create the SPI types**

`PdfAConformance.java`:
```java
package io.casehub.platform.api.pdf;

public enum PdfAConformance {
    PDFA_2_B
}
```

`PdfOptions.java`:
```java
package io.casehub.platform.api.pdf;

import java.time.Instant;

public record PdfOptions(
        String title, String author, Instant createdAt,
        String reportType, PdfAConformance conformance) {
    public static PdfOptions defaults() {
        return new PdfOptions(null, null, null, null, PdfAConformance.PDFA_2_B);
    }
}
```

`PdfGenerator.java`:
```java
package io.casehub.platform.api.pdf;

import java.util.Optional;

public interface PdfGenerator {
    Optional<byte[]> generateFromHtml(String html, PdfOptions options);
}
```

`NoOpPdfGenerator.java`:
```java
package io.casehub.platform.api.pdf;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class NoOpPdfGenerator implements PdfGenerator {
    @Override
    public Optional<byte[]> generateFromHtml(String html, PdfOptions options) {
        return Optional.empty();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/platform-api/pom.xml -Dtest=NoOpPdfGeneratorTest -pl .`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform-api/src/
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#417): PdfGenerator SPI in platform-api  Refs casehubio/qhorus#417"
```

### Task 2: OpenHtmlToPdfGenerator in platform-pdf module

**Files:**
- Create: `platform-pdf/pom.xml`
- Create: `platform-pdf/src/main/java/io/casehub/platform/pdf/OpenHtmlToPdfGenerator.java`
- Create: `platform-pdf/src/main/resources/fonts/*.ttf` (6 font files, copied from qhorus)
- Modify: `pom.xml` (parent — add `platform-pdf` module, add openhtmltopdf to dependencyManagement)
- Test: `platform-pdf/src/test/java/io/casehub/platform/pdf/OpenHtmlToPdfGeneratorTest.java`

**Interfaces:**
- Consumes: `PdfGenerator`, `PdfOptions`, `PdfAConformance` from Task 1
- Produces: `OpenHtmlToPdfGenerator` — `@ApplicationScoped` implementation

- [ ] **Step 1: Create platform-pdf/pom.xml**

Depends on `casehub-platform-api`, `openhtmltopdf-pdfbox`, `quarkus-arc`. Test deps: `quarkus-junit`, `assertj-core`. Jandex plugin for CDI discovery.

- [ ] **Step 2: Add openhtmltopdf to platform parent dependencyManagement and add platform-pdf module**

Add `<module>platform-pdf</module>` to `<modules>`. Add openhtmltopdf `1.1.37` to `<dependencyManagement>` if not already present.

- [ ] **Step 3: Copy font files from qhorus compliance-report to platform-pdf resources**

Copy all 6 `.ttf` from `qhorus/compliance-report/src/main/resources/fonts/` to `platform-pdf/src/main/resources/fonts/`.

- [ ] **Step 4: Write the failing test**

3 tests: `generateFromHtml_producesPdfBytes`, `generateFromHtml_setsDocumentMetadata`, `generateFromHtml_producesMultiplePages`. Same logic as the existing `HtmlToPdfConverterTest` but adapted to the `PdfGenerator` interface (returns `Optional`, takes `PdfOptions`).

- [ ] **Step 5: Implement OpenHtmlToPdfGenerator**

Port from qhorus `HtmlToPdfConverter` — same font loading, same PDF/A-2b conformance, same metadata setting. Adapted to `PdfGenerator` interface with `Optional<byte[]>` return and `PdfOptions` parameter.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/platform-pdf/pom.xml -Dtest=OpenHtmlToPdfGeneratorTest -pl .`
Expected: PASS (3 tests)

- [ ] **Step 7: Run full platform install**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/platform/pom.xml -DskipTests`
Expected: BUILD SUCCESS — SNAPSHOT published to local repo

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform-pdf/ pom.xml
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#417): casehub-platform-pdf with OpenHtmlToPdfGenerator  Refs casehubio/qhorus#417"
```

---

## Batch 2: Qhorus adaptation — consume platform PdfGenerator

### Task 3: Replace HtmlToPdfConverter with platform PdfGenerator

**Files:**
- Delete: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlToPdfConverter.java`
- Delete: `compliance-report/src/main/resources/fonts/` (all 6 .ttf files)
- Delete: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/HtmlToPdfConverterTest.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/PdfReportRenderer.java`
- Modify: `compliance-report/src/test/java/io/casehub/qhorus/compliance/format/PdfReportRendererTest.java`
- Modify: `compliance-report/pom.xml` (replace openhtmltopdf with casehub-platform-pdf)

**Interfaces:**
- Consumes: `PdfGenerator.generateFromHtml(String, PdfOptions)` returning `Optional<byte[]>` from Task 1

- [ ] **Step 1: Update compliance-report/pom.xml**

Replace `openhtmltopdf-pdfbox` dependency with `casehub-platform-pdf` (`0.2-SNAPSHOT`).

- [ ] **Step 2: Update PdfReportRenderer to inject PdfGenerator**

Replace local `HtmlToPdfConverter converter` field with `@Inject PdfGenerator pdfGenerator`. Remove `@PostConstruct init()`. Update `render()` to build `PdfOptions` from `PdfDocumentMetadata` and call `pdfGenerator.generateFromHtml()` with `.orElseThrow()`.

- [ ] **Step 3: Update PdfReportRendererTest to mock PdfGenerator**

Replace `HtmlToPdfConverter` usage with `mock(PdfGenerator.class)`. Test: delegation, content type, `supports(PDF)`, and the `Optional.empty()` → `IllegalStateException` path.

- [ ] **Step 4: Delete HtmlToPdfConverter, its test, and font files**

Remove the 3 files and the `fonts/` directory.

- [ ] **Step 5: Run compliance-report tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/compliance-report/pom.xml -pl .`
Expected: PASS

- [ ] **Step 6: Run full qhorus build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add compliance-report/ pom.xml
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#417): consume platform PdfGenerator, remove local converter  Refs #417"
```

---

## Batch 3: Documentation

### Task 4: Update platform and qhorus documentation

**Files:**
- Modify: platform consumer/contributor guides — add PDF module section
- Modify: platform `ARC42STORIES.MD` — add platform-pdf component
- Modify: qhorus `docs/guides/consumer-guide.md` — document PDF format
- Modify: qhorus `CLAUDE.md` — add platform-pdf dependency note

- [ ] **Step 1: Document platform-pdf in platform guides**

Add section covering: what it does, how to add the dependency, PdfGenerator SPI, PdfOptions, graceful degradation via NoOp.

- [ ] **Step 2: Update platform ARC42STORIES.MD**

Add `platform-pdf` as a new component.

- [ ] **Step 3: Document PDF rendering in qhorus consumer guide**

Document `ReportFormat.PDF`, `Accept: application/pdf`, PDF/A-2b, `casehub-platform-pdf` dependency.

- [ ] **Step 4: Update qhorus CLAUDE.md**

Add `casehub-platform-pdf` dependency note in the compliance-report module context.

- [ ] **Step 5: Commit all doc changes**

Platform:
```bash
git -C /Users/mdproctor/claude/casehub/platform add docs/ ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/platform commit -m "docs(#417): platform-pdf module documentation  Refs casehubio/qhorus#417"
```

Qhorus:
```bash
git -C /Users/mdproctor/claude/casehub/qhorus add docs/ CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#417): PDF rendering documentation  Refs #417"
```

---

## References

- `specs/issue-417-pdf-report-renderer/2026-08-30-pdf-report-renderer-design.md` — design spec
- `specs/issue-417-pdf-report-renderer/decisions.md` — D1-D6 design decisions
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/ReportRenderer.java` — extension point
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlReportRenderer.java:51` — renderForPdf (already implemented)
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/HtmlToPdfConverter.java` — local impl to extract to platform
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/PdfDocumentMetadata.java` — metadata extraction (stays in qhorus)
- `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java:183` — content negotiation (already updated)
- `compliance-report/pom.xml` — module dependencies
- GitHub #417 — feat: PDF rendering for compliance evidence export reports
- GitHub #402 — compliance evidence export (parent design, D3 deferral)
