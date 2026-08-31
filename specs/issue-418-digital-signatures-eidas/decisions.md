## D1: PDF signing library

**Choice:** EU DSS (Digital Signature Services) — European Commission's reference implementation
**Alternatives:**
- Apache PDFBox `PDSignature` directly — minimal new deps but we own PAdES conformance code (~2000 lines of spec-dense plumbing)
- BouncyCastle CMS + manual PDF integration — maximum control but disproportionate engineering effort
**Rationale:** PAdES conformance is auditor-facing and specification-dense. DSS is the reference implementation by the regulating body itself — PAdES-B/T/LT/LTA profiles, CAdES for detached signatures, built-in TSA client, certificate chain validation. Eliminates the risk of subtle conformance failures that would cause reports to fail regulatory validation.
**Trade-offs:** Heavier dependency (~15 Maven modules, we pull 3-4 selectively: dss-pades-pdfbox, dss-cades, dss-tsl-validation). Uses PDFBox and BouncyCastle internally (already in dep graph via OpenHTMLtoPDF).
**Sources:** EU DSS GitHub (ec-europa/dss), ETSI EN 319 142 (PAdES), platform SigningProvider SPI (io.casehub.platform.api.signing)
**Exploration:** quick
**Status:** captured
