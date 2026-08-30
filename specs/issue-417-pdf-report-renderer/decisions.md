## D1: PDF library — OpenHTMLtoPDF via HTML→PDF pipeline

**Choice:** OpenHTMLtoPDF (`com.openhtmltopdf:openhtmltopdf-pdfbox`) converts existing HtmlReportRenderer HTML output to PDF. Architecture: report object → HtmlReportRenderer.render() → HTML bytes → OpenHTMLtoPDF → PDF bytes. No duplicate rendering logic.
**Alternatives:**
- iText 7 — most feature-rich, built-in digital signatures, but AGPL 3.0 license is incompatible with library embedding in consumer apps
- Apache PDFBox (direct) — Apache 2.0 license, but low-level API requires manual document layout; OpenHTMLtoPDF uses PDFBox internally anyway
**Rationale:** Reuses existing HTML rendering infrastructure, LGPL 2.1 license compatible, anticipated by the #402 design spec (D3). Digital signatures deferred to #418 with a separate library.
**Trade-offs:** Digital signature support for #418 will need a separate library (PDFBox signatures or BouncyCastle) rather than iText's built-in support. Acceptable — #418 is a separate issue with its own scope.
**Sources:** ReportRenderer.java (compliance-report/format/), HtmlReportRenderer.java, issue-402 design spec D3, OpenHTMLtoPDF GitHub
**Exploration:** quick
**Status:** captured

## D2: PDF/A conformance — PDF/A-2b

**Choice:** PDF/A-2b (ISO 19005-2, basic conformance). Visual reproducibility guarantee for regulatory archival. OpenHTMLtoPDF supports this via `PdfRendererBuilder.usePdfAConformance(PdfAConformance.PDFA_2_B)`.
**Alternatives:**
- PDF/A-3b — allows embedding arbitrary attachments (XML, CSV), but raw JSON is already available via REST API; embedding is redundant complexity
- PDF/A-1b — oldest, most universal, but lacks transparency support and has tighter font constraints that cause rendering issues with HTML→PDF conversion
**Rationale:** Current sweet spot — modern enough for reliable rendering from HTML, widely accepted by EU regulatory bodies, no unnecessary features.
**Trade-offs:** If a regulator specifically requires PDF/A-3b with embedded machine-readable data, this would need upgrading. Unlikely — JSON/CSV REST endpoints serve that need.
**Depends on:** D1 (OpenHTMLtoPDF supports PDF/A-2b natively)
**Sources:** ISO 19005-2, OpenHTMLtoPDF PdfAConformance enum, EU AI Act Article 12
**Exploration:** quick
**Status:** captured

## D3: Module placement — platform PDF service + qhorus adapter

**Choice:** New platform module `casehub-platform-pdf` owns the OpenHTMLtoPDF dependency and exposes a `PdfGenerator` service (HTML bytes → PDF bytes, PDF/A conformance config, document metadata). Qhorus's `PdfReportRenderer` in `compliance-report/` becomes a thin adapter injecting `HtmlReportRenderer` + `PdfGenerator`. SPI interface `PdfGenerator` in `casehub-platform-api` with `@DefaultBean NoOpPdfGenerator` (throws UnsupportedOperationException) so modules can compile against the API without pulling in OpenHTMLtoPDF.
**Alternatives:**
- Qhorus-local in `compliance-report/` — simpler for now, but other repos (ops reports, engine dashboards) would duplicate the OpenHTMLtoPDF dependency and configuration; creates N copies of PDF infrastructure across the ecosystem
**Rationale:** PDF rendering is a general-purpose capability, not qhorus-specific. Any CaseHub module producing HTML reports could need PDF export. Platform placement prevents duplication and centralises font management, PDF/A configuration, and library version control. Follows the established pattern: shared infrastructure in platform, domain composition in domain modules.
**Trade-offs:** Requires a cross-repo change (platform issue + platform PR) before qhorus can consume it. Incremental cost is small — one Maven module with ~2 production classes.
**Depends on:** D1 (library choice), D2 (PDF/A conformance — configured via PdfGenerator)
**Sources:** casehub-platform module patterns, CompliancePostureProvider SPI pattern (api/ + @DefaultBean), NoOpCapabilityHealth
**Exploration:** quick
**Status:** captured

## D4: PdfGenerator SPI — single method with options record

**Choice:** `PdfGenerator.generateFromHtml(byte[] html, PdfOptions options)` returns `byte[]`. `PdfOptions` record carries document metadata (title, author, createdAt) and `PdfAConformance` enum (default `PDFA_2_B`). `PdfOptions.defaults()` static factory for callers that just need basic conversion. Interface lives in `casehub-platform-api`. `@DefaultBean NoOpPdfGenerator` throws `UnsupportedOperationException` — fail-fast when platform-pdf module is absent.
**Alternatives:**
- Builder pattern with `Request`/`Result` types — more ceremony, validation in builder, but overkill for a single conversion operation
**Rationale:** Minimal contract that covers all known use cases. Extensible via adding fields to the options record with defaults — no breaking changes when new metadata fields are needed.
**Trade-offs:** No streaming support — entire HTML and PDF in memory. Acceptable for compliance reports (typically <1MB). If large-document streaming is needed later, a second method can be added without breaking existing callers.
**Depends on:** D1 (library), D2 (conformance enum), D3 (platform placement)
**Sources:** PdfRendererBuilder API (OpenHTMLtoPDF), PdfAConformance enum
**Exploration:** quick
**Status:** captured
