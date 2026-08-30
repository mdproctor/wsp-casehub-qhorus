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
