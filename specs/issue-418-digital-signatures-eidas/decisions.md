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

## D2: Platform signing SPI composition

**Choice:** New `DocumentSigningService` SPI in platform-api, implementation in casehub-platform-signing, delegates raw crypto to existing `SigningProvider` via a DSS `SignatureTokenConnection` adapter
**Alternatives:**
- Bypass SigningProvider, load PKCS#12 directly in DSS — simpler but creates two parallel signing paths in the platform, making SigningProvider irrelevant for document signing
- Extend SigningProvider with certificate-aware methods — mixes abstraction levels, forces all existing impls to update
**Rationale:** Preserves clean layering. SigningProvider stays focused on raw cryptographic operations (HSM-backable). DocumentSigningService handles format packaging (PAdES/CAdES). The DSS SignatureTokenConnection adapter bridges them — DSS calls SigningProvider for the actual sign operation, gets certificate chain from the keystore.
**Trade-offs:** One more SPI interface in platform-api. The adapter layer adds a small indirection. But the separation means swapping from PKCS#12 to cloud KMS only requires a new SigningProvider impl, not touching any document signing code.
**Depends on:** D1 (EU DSS as the signing library)
**Sources:** platform SigningProvider SPI (io.casehub.platform.api.signing), EU DSS SignatureTokenConnection interface
**Exploration:** quick
**Status:** captured

## D3: PAdES signature profile level

**Choice:** PAdES-B-T default (with TSA), PAdES-B-B fallback (without TSA). Configurable via `casehub.signing.pades-profile` for deployers who want LT or LTA — DSS supports all levels with the same code path.
**Alternatives:**
- Always PAdES-B-LTA — maximum archival but requires CRL/OCSP data sources; fails if not configured
- PAdES-B-B only — no timestamp; doesn't meet the timestamping requirement
**Rationale:** B-T proves signature time (required for regulatory submissions) without requiring CRL/OCSP infrastructure. LTA is the gold standard for archival but requires revocation data sources that may not be configured. Making the profile configurable means deployers with full PKI infrastructure can upgrade to LTA without code changes.
**Trade-offs:** B-T signatures do not survive certificate expiry + CRL unavailability the way LTA does. Deployers who need indefinite archival must configure CRL/OCSP sources and set the profile to LTA.
**Depends on:** D1 (EU DSS), D2 (DocumentSigningService SPI)
**Sources:** ETSI EN 319 142 (PAdES profiles), EU DSS PAdES module
**Exploration:** quick
**Status:** captured

## D4: Detached signature format for JSON/CSV

**Choice:** CAdES detached signature (.p7s) — PKCS#7/CMS binary format
**Alternatives:**
- JAdES (JSON Advanced Electronic Signatures) — JSON-based, human-readable-ish, but newer standard with less tool support; regulators may lack verification tools
- Raw PKCS#7 via SigningProvider — no timestamp embedding, no certificate chain packaging; doesn't meet the framework goal
**Rationale:** CAdES is the established standard for detached signatures on non-XML data. DSS has full CAdES support with the same profile levels (B-B, B-T, etc.) — zero additional library effort. The .p7s file extension is widely recognized by cryptographic verification tools. Pairs naturally with the report file for download.
**Trade-offs:** Binary format, not human-inspectable. But signature files aren't meant to be human-read — they're machine-verified.
**Depends on:** D1 (EU DSS), D3 (profile levels apply to CAdES too)
**Sources:** ETSI EN 319 122 (CAdES), EU DSS CAdES module
**Exploration:** quick
**Status:** captured

## D5: Signing pipeline position

**Choice:** Post-render signing service — render() → ComplianceReportSigningService.sign() → store(). New service in compliance-report module orchestrates: receives rendered bytes + format, calls DocumentSigningService (platform SPI), returns signed bytes (PDF) or original bytes + detached signature (JSON/CSV).
**Alternatives:**
- Signing inside the renderer — tangles rendering with crypto, breaks single responsibility, every renderer needs signing awareness
- Post-storage async signing — reports temporarily unsigned, regulatory gap for on-demand reports
**Rationale:** Signing is logically separate from rendering. The post-render position means: (1) re-signing on certificate rotation doesn't require re-rendering, (2) renderers stay format-focused, (3) both on-demand (REST) and scheduled paths call the same signing service. Graceful degradation: when DocumentSigningService returns empty (NoOp), the unsigned report passes through unchanged.
**Trade-offs:** One extra pipeline step. Detached signatures for JSON/CSV require storing two artefacts (report + .p7s) — the signing service returns a SignedReport record containing both.
**Depends on:** D2 (DocumentSigningService SPI), D4 (CAdES for detached)
**Sources:** ComplianceReportResource.renderResponse(), ComplianceReportScheduler.generateAndStore(), ComplianceReportStorageService
**Exploration:** quick
**Status:** captured
