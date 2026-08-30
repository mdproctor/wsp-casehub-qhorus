## D1: PDF library — OpenHTMLtoPDF via HTML→PDF pipeline

**Choice:** OpenHTMLtoPDF (`com.openhtmltopdf:openhtmltopdf-pdfbox`) converts existing HtmlReportRenderer HTML output to PDF. Architecture: report object → HtmlReportRenderer.render() → HTML bytes → OpenHTMLtoPDF → PDF bytes. No duplicate rendering logic.
**Alternatives:**
- iText 7 — most feature-rich, built-in digital signatures, but AGPL 3.0 license is incompatible with library embedding in consumer apps
- Apache PDFBox (direct) — Apache 2.0 license, but low-level API requires manual document layout; OpenHTMLtoPDF uses PDFBox internally anyway; pdfbox-layout/pdfbox-table add higher-level APIs but are less maintained than OpenHTMLtoPDF itself
**Rationale:** Reuses existing HTML rendering infrastructure, LGPL 2.1 license compatible, anticipated by the #402 design spec (D3). Digital signatures deferred to #418 with a separate library. Direct PDFBox rendering (R1-04) would produce slightly better semantic PDF structure but at significant implementation cost — the HTML output is structurally simple (tables + summary paragraphs) and OpenHTMLtoPDF handles it correctly.
**Trade-offs:** Digital signature support for #418 will need a separate library (PDFBox signatures or BouncyCastle) rather than iText's built-in support. CSS support limited to CSS 2.1 with partial CSS3 — sufficient for the current report styling but constrains future visual enhancements. Streaming PDF generation is architecturally impossible for PDF/A (cross-reference table requirement), so the in-memory approach is not a deferrable trade-off but a format constraint.
**Sources:** ReportRenderer.java (compliance-report/format/), HtmlReportRenderer.java, issue-402 design spec D3, OpenHTMLtoPDF GitHub
**Exploration:** quick
**Status:** revised — expanded alternatives (R1-04 direct PDFBox), clarified CSS and streaming constraints (R1-06, R1-16)

## D2: PDF/A conformance — PDF/A-2b

**Choice:** PDF/A-2b (ISO 19005-2, basic conformance). Visual reproducibility guarantee for regulatory archival. OpenHTMLtoPDF supports this via `PdfRendererBuilder.usePdfAConformance(PdfAConformance.PDFA_2_B)`.
**Alternatives:**
- PDF/A-3b — allows embedding arbitrary attachments (XML, CSV), but raw JSON is already available via REST API; embedding is redundant complexity
- PDF/A-1b — oldest, most universal, but lacks transparency support and has tighter font constraints that cause rendering issues with HTML→PDF conversion
- PDF/A-2a (accessible conformance) — adds tagged PDF requirement for screen readers; OpenHTMLtoPDF cannot produce tagged PDF structure, so this is not achievable with D1's library choice
**Rationale:** Current sweet spot — modern enough for reliable rendering from HTML, widely accepted by EU regulatory bodies, no unnecessary features. PDF/UA accessibility (R1-08) is a valid concern for public-facing documents but compliance evidence exports are internal audit records, not public services under the European Accessibility Act. If accessibility becomes a requirement, it would require a different library approach (D1 revision).
**Trade-offs:** If a regulator specifically requires PDF/A-3b with embedded machine-readable data, this would need upgrading. Unlikely — JSON/CSV REST endpoints serve that need. PDF/A-2b conformance must be validated in tests using veraPDF or Apache PDFBox Preflight — setting the conformance flag is necessary but not sufficient (R1-09).
**Depends on:** D1 (OpenHTMLtoPDF supports PDF/A-2b natively)
**Sources:** ISO 19005-2, OpenHTMLtoPDF PdfAConformance enum, EU AI Act Article 12, European Accessibility Act (Directive 2019/882)
**Exploration:** quick
**Status:** revised — added PDF/A-2a and PDF/UA as considered alternatives (R1-08), added validation requirement (R1-09)

## D3: Module placement — qhorus compliance-report local

**Choice:** `PdfReportRenderer` and `HtmlToPdfConverter` (package-private) live in `compliance-report/src/main/java/io/casehub/qhorus/compliance/format/` alongside the existing renderers. OpenHTMLtoPDF dependency owned by `compliance-report/pom.xml`. Fonts bundled in `compliance-report/src/main/resources/fonts/`.
**Alternatives:**
- Platform `casehub-platform-pdf` module — shared HTML→PDF service usable by any CaseHub repo; rejected because only one consumer exists today. Platform-api-scope protocol: "implement in domain repo first, extract to platform when a second consumer materialises." Cross-repo coordination cost (platform issue + PR + release) is disproportionate for ~2 classes serving one module.
**Rationale:** Single consumer (compliance-report). No other CaseHub repo currently produces HTML reports needing PDF export. When a second consumer appears, extract `HtmlToPdfConverter` to platform — the internal API is designed for clean extraction (String in, byte[] out, options record). The @DefaultBean NoOp-that-throws pattern (R1-12) also violates established platform convention where all NoOps degrade gracefully.
**Trade-offs:** If ops or engine need PDF later, they'll need to either extract to platform or duplicate. Duplication is the signal to extract — not speculation about future need.
**Depends on:** D1 (library choice)
**Sources:** platform-api-scope protocol (R1-11), platform-ownership-check protocol, existing renderer placement in compliance-report/format/
**Exploration:** quick
**Status:** revised — reversed from platform placement to qhorus-local per R1-11 (single consumer, speculative extraction)

## D4: HtmlToPdfConverter — package-private, String parameter

**Choice:** Package-private `HtmlToPdfConverter` class in `compliance-report/format/` with `byte[] convert(String html, PdfDocumentMetadata metadata)`. `PdfDocumentMetadata` record carries title, author, createdAt, reportType. Used only by `PdfReportRenderer`. Not an SPI — no cross-module contract needed with qhorus-local placement.
**Alternatives:**
- Platform SPI with `byte[]` html parameter — rejected with D3 revision; byte[] obscures charset contract (R1-15), and platform placement is premature
- Builder pattern — overkill for a single conversion operation within one package
**Rationale:** Package-private keeps the API surface minimal. String parameter eliminates charset ambiguity. Metadata record maps report model fields to PDF document properties (Title, Author, Subject, CreationDate) for regulatory traceability (R1-22).
**Trade-offs:** Package-private means extraction to platform later requires promoting to public. Trivial refactor.
**Depends on:** D1 (library), D2 (conformance), D3 (local placement)
**Sources:** PdfRendererBuilder API (OpenHTMLtoPDF), PDDocumentInformation (PDFBox)
**Exploration:** quick
**Status:** revised — demoted from platform SPI to package-private per D3 revision; String parameter per R1-15; metadata mapping per R1-22

## D5: Font handling — bundled Liberation Sans + Mono

**Choice:** Bundle Liberation Sans (Regular, Bold, Italic, BoldItalic) and Liberation Mono (Regular, Bold) inside `compliance-report/src/main/resources/fonts/`. ~1.4MB total, Apache 2.0 licensed. Register at converter construction via OpenHTMLtoPDF's `PdfRendererBuilder.useFont()`. Liberation Mono for UUIDs, correlation IDs, Merkle hashes, and trust scores — structured data that benefits from monospaced rendering.
**Alternatives:**
- System font discovery — rendering varies by environment; unreliable for reproducible regulatory documents
- Sans-serif only — monospaced data (hashes, UUIDs) harder to verify visually for compliance auditors (R1-18)
**Rationale:** PDF/A-2b requires all fonts embedded. Bundled fonts guarantee reproducible output. Liberation family covers Latin/Cyrillic/Greek scripts. Monospaced font for data fields improves readability of the exact values that make these reports valuable.
**Trade-offs:** ~1.4MB added to the module jar. CJK scripts not covered — would need additional font bundles if needed.
**Depends on:** D1 (OpenHTMLtoPDF font API), D2 (PDF/A requires embedding), D3 (compliance-report owns fonts)
**Sources:** Liberation Fonts (GitHub), OpenHTMLtoPDF font registration API, PDF/A-2b font embedding requirement
**Exploration:** quick
**Status:** revised — added Liberation Mono per R1-18; moved from platform to compliance-report per D3 revision

## D6: Page structure for regulatory documents

**Choice:** Enhance `HtmlReportRenderer` CSS with `@page` rules for PDF-quality output: page numbers (CSS `counter(page)` / `counter(pages)`), header with report type and generation timestamp, footer with tenant ID and schema version. The HTML renderer gains a `renderForPdf(Object report, PdfDocumentMetadata metadata)` method that wraps the existing per-type rendering with page structure markup. The standard `render()` method is unchanged — browser HTML doesn't need page structure.
**Alternatives:**
- PDF-only page injection via PDFBox API post-conversion — technically possible but fragile (coordinate-based text placement over rendered content) and duplicates information already available in the report model
- Separate HTML template for PDF — unnecessary duplication when CSS @page rules handle the structural differences
**Rationale:** Regulatory PDFs need page numbers, headers/footers, and document identifiers for auditability (R1-21). CSS `@page` is the OpenHTMLtoPDF-supported mechanism. A dedicated `renderForPdf()` method keeps the standard HTML output clean for browser viewing while adding the page structure PDF needs.
**Trade-offs:** `HtmlReportRenderer` gains awareness of the PDF use case via the new method. This is acceptable — the renderer already knows about report types via the switch expression, and the PDF-specific method is additive (no changes to existing render() behaviour).
**Depends on:** D1 (OpenHTMLtoPDF CSS @page support), D3 (local placement — renderer modification is straightforward)
**Sources:** OpenHTMLtoPDF @page CSS support, CSS Paged Media Module (W3C), R1-21 (page structure requirement)
**Exploration:** quick
**Status:** captured
