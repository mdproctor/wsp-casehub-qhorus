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

## D3: Module placement — platform PDF service + qhorus adapter

**Choice:** New platform module `casehub-platform-pdf` owns the OpenHTMLtoPDF dependency and exposes a `PdfGenerator` service (String HTML → byte[] PDF, PDF/A conformance config, document metadata). Qhorus's `PdfReportRenderer` in `compliance-report/` is a thin adapter injecting `HtmlReportRenderer` + `PdfGenerator`. SPI interface `PdfGenerator` in `casehub-platform-api` with `@DefaultBean NoOpPdfGenerator` returning `Optional.empty()` (graceful degradation, not throwing — per R1-12). Fonts bundled in `casehub-platform-pdf/src/main/resources/fonts/`.
**Alternatives:**
- Qhorus-local in `compliance-report/` — fewer moving parts, but PDF rendering is infrastructure (like HTTP clients or JSON serialization), not a domain type. Keeping it local guarantees duplication when a second consumer appears; the question is when, not if. R1-11 cited platform-api-scope protocol ("wait for second consumer"), but that protocol is too conservative for obviously general-purpose infrastructure — it conflates domain types that *might* be reusable with infrastructure capabilities that *are* reusable. Protocol revision tracked separately.
**Rationale:** PDF rendering is general-purpose infrastructure. HTML→PDF conversion with font management, PDF/A conformance, and document metadata is the same operation regardless of which module produces the HTML. Platform placement prevents duplication, centralises font management and library version control, and gives every CaseHub module PDF capability without taking on OpenHTMLtoPDF as a direct dependency.
**Trade-offs:** Cross-repo coordination cost (platform issue + PR + release before qhorus can consume). Acceptable for infrastructure — the platform exists to absorb this cost.
**Depends on:** D1 (library choice), D2 (PDF/A conformance — configured via PdfGenerator)
**Sources:** casehub-platform module patterns, CompliancePostureProvider SPI pattern (api/ + @DefaultBean), NoOpCapabilityHealth (graceful degradation)
**Exploration:** quick
**Status:** revised — restored platform placement (user override of R1-11; protocol too conservative for infrastructure); NoOp returns Optional.empty() per R1-12

## D4: PdfGenerator SPI — single method with String parameter and options record

**Choice:** `PdfGenerator.generateFromHtml(String html, PdfOptions options)` returns `Optional<byte[]>`. `PdfOptions` record carries document metadata (title, author, createdAt, reportType) and `PdfAConformance` enum (default `PDFA_2_B`). `PdfOptions.defaults()` static factory for callers that just need basic conversion. Interface lives in `casehub-platform-api`. `@DefaultBean NoOpPdfGenerator` returns `Optional.empty()` — graceful degradation when platform-pdf module is absent (per R1-12).
**Alternatives:**
- `byte[]` html parameter — obscures charset contract (R1-15); String eliminates ambiguity
- Builder pattern — overkill for a single conversion operation
- Package-private local class — rejected with D3 restoration to platform placement
**Rationale:** Minimal contract covering all known use cases. String parameter is the natural Java representation for HTML text. Optional return enables callers to handle PDF-unavailable gracefully (e.g., omit `application/pdf` from content negotiation). Extensible via adding fields to the options record with defaults.
**Trade-offs:** In-memory only — entire HTML and PDF held in memory. Acceptable for compliance reports (typically <1MB). Streaming is architecturally impossible for PDF/A (R1-16).
**Depends on:** D1 (library), D2 (conformance enum), D3 (platform placement)
**Sources:** PdfRendererBuilder API (OpenHTMLtoPDF), PDDocumentInformation (PDFBox)
**Exploration:** quick
**Status:** revised — String parameter per R1-15; Optional return per R1-12; restored to platform SPI per D3 restoration

## D5: Font handling — bundled Liberation Sans + Mono

**Choice:** Bundle Liberation Sans (Regular, Bold, Italic, BoldItalic) and Liberation Mono (Regular, Bold) inside `casehub-platform-pdf/src/main/resources/fonts/`. ~1.4MB total, Apache 2.0 licensed. Register at `@PostConstruct` via OpenHTMLtoPDF's `PdfRendererBuilder.useFont()`. Liberation Mono for UUIDs, correlation IDs, Merkle hashes, and trust scores — structured data that benefits from monospaced rendering.
**Alternatives:**
- System font discovery — rendering varies by environment; unreliable for reproducible regulatory documents
- Sans-serif only — monospaced data (hashes, UUIDs) harder to verify visually for compliance auditors (R1-18)
**Rationale:** PDF/A-2b requires all fonts embedded. Bundled fonts guarantee reproducible output. Liberation family covers Latin/Cyrillic/Greek scripts. Monospaced font for data fields improves readability of the exact values that make these reports valuable.
**Trade-offs:** ~1.4MB added to the module jar. CJK scripts not covered — would need additional font bundles if needed.
**Depends on:** D1 (OpenHTMLtoPDF font API), D2 (PDF/A requires embedding), D3 (platform module owns fonts)
**Sources:** Liberation Fonts (GitHub), OpenHTMLtoPDF font registration API, PDF/A-2b font embedding requirement
**Exploration:** quick
**Status:** revised — added Liberation Mono per R1-18; fonts in platform-pdf per D3 restoration

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
