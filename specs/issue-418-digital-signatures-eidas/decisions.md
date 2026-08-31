## D1: PDF signing library

**Choice:** EU DSS (Digital Signature Services) — European Commission's reference implementation
**Alternatives:**
- Apache PDFBox `PDSignature` directly — minimal new deps but we own PAdES conformance code (~2000 lines of spec-dense plumbing)
- BouncyCastle CMS + manual PDF integration — maximum control but disproportionate engineering effort
**Rationale:** PAdES conformance is auditor-facing and specification-dense. DSS is the reference implementation by the regulating body itself — PAdES-B/T/LT/LTA profiles, CAdES for detached signatures, built-in TSA client, certificate chain validation. Eliminates the risk of subtle conformance failures that would cause reports to fail regulatory validation.
**Trade-offs:** Heavier dependency (~15 Maven modules, we pull 3-4 selectively: dss-pades-pdfbox, dss-cades, dss-tsl-validation). Uses PDFBox and BouncyCastle internally — PDFBox version compatibility with OpenHTMLtoPDF (from #417) must be verified during implementation (DSS 6.x uses PDFBox 3.x; OpenHTMLtoPDF may target 2.x). If versions conflict, BOM alignment or shade plugin isolation needed.
**Sources:** EU DSS GitHub (ec-europa/dss), ETSI EN 319 142 (PAdES), platform SigningProvider SPI (io.casehub.platform.api.signing)
**Exploration:** quick
**Review:** R1-05 flagged PDFBox version conflict risk — added to trade-offs
**Status:** captured

## D2: Platform document signing — self-contained SPI

**Choice:** New self-contained `DocumentSigningService` SPI in platform-api with implementation in casehub-platform-signing. Owns its own key material configuration (PKCS#12 path via config, certificate chain loaded internally). Does NOT delegate to or adapt the existing `SigningProvider` SPI — they are separate concerns (document sealing vs ledger agent signing).
**Alternatives:**
- Adapter bridging DSS SignatureTokenConnection to SigningProvider — infeasible: SigningProvider returns raw bytes + public key, not X.509 certificate chains that PAdES/CAdES require. Adapter would need direct keystore access anyway, making SigningProvider redundant (R1-01)
- Extend SigningProvider with certificate-aware methods — coherent but couples ledger signing with document signing lifecycle; no existing SigningProvider impls beyond NoOp
- CertificateProvider SPI alongside SigningProvider, compose both — over-decomposed for current needs
**Rationale:** Document signing requires X.509 certificate chains, timestamping, and format-specific packaging — fundamentally different from the raw sign/verify operations that SigningProvider serves. Self-containment means DocumentSigningService can be backed by PKCS#12, cloud KMS, or HSM without any coupling to the ledger's signing path. SigningProvider stays for agent attestation signing (Ed25519 keypairs in the ledger).
**Trade-offs:** Two signing SPIs in the platform (SigningProvider for raw crypto, DocumentSigningService for document sealing). No shared implementation — they serve different domains. Future unification possible but not needed now.
**Depends on:** D1 (EU DSS as the signing library)
**Sources:** platform SigningProvider SPI (io.casehub.platform.api.signing — sign() returns Optional<SignatureResult> with byte[] publicKey, no X.509), EU DSS SignatureTokenConnection, R1-01, R1-02
**Exploration:** quick → revised after decision review
**Status:** revised

## D3: PAdES signature profile — strict mode, no silent fallback

**Choice:** Configurable profile via `casehub.signing.pades-profile` (B_B, B_T, B_LT, B_LTA). Default: B_T. When B_T is configured and TSA is unreachable: **fail and refuse to sign** — do not silently downgrade to B_B. B_B is an explicit configuration choice for deployments that don't need timestamp proof, not a fallback.
**Alternatives:**
- Silent B-B fallback — dangerous: produces documents that appear compliant but lack timestamp proof that the regulatory environment may require (R1-04)
- Always PAdES-B-LTA — maximum archival but requires CRL/OCSP data sources; fails if not configured
- PAdES-B-B only — no timestamp; doesn't meet the timestamping requirement
**Rationale:** A deployer who configures B-T did so because their regulatory environment requires timestamp proof. Silent degradation to B-B is worse than an error — it creates documents that appear compliant but aren't. Fail-loud is the correct behavior for compliance infrastructure.
**Trade-offs:** TSA outage blocks report signing entirely when B-T is configured. Deployers must ensure TSA availability or explicitly choose B-B if timestamps are optional for their jurisdiction. Monitoring/alerting for TSA connectivity is a deployment concern.
**Depends on:** D1 (EU DSS), D2 (DocumentSigningService SPI)
**Sources:** ETSI EN 319 142 (PAdES profiles), EU DSS PAdES module, R1-04
**Exploration:** quick → revised after decision review
**Status:** revised

## D4: Detached signature format for JSON/CSV

**Choice:** CAdES detached signature (.p7s) — PKCS#7/CMS binary format
**Alternatives:**
- JAdES (JSON Advanced Electronic Signatures) — JSON-based, human-readable-ish, but newer standard with less tool support; regulators may lack verification tools
- Raw PKCS#7 via SigningProvider — no timestamp embedding, no certificate chain packaging; doesn't meet the framework goal
**Rationale:** CAdES is the established standard for detached signatures on non-XML data. DSS has full CAdES support with the same profile levels (B-B, B-T, etc.) — zero additional library effort. The .p7s file extension is widely recognized by cryptographic verification tools. Pairs naturally with the report file for download.
**Trade-offs:** Binary format, not human-inspectable. But signature files aren't meant to be human-read — they're machine-verified. Storage: .p7s bytes stored as a second SharedData artefact linked via signature_artefact_id on ComplianceReportRecord. Client retrieval via GET /api/compliance/reports/{id}/signature.
**Depends on:** D1 (EU DSS), D3 (profile levels apply to CAdES too)
**Sources:** ETSI EN 319 122 (CAdES), EU DSS CAdES module
**Exploration:** quick
**Status:** captured

## D5: Signing pipeline — scheduled/stored reports only

**Choice:** Post-render signing for scheduled and stored reports only. Pipeline: render() → ComplianceReportSigningService.sign() → store(). On-demand REST reports (GET /api/compliance/obligations etc.) are returned unsigned — TSA round-trips (200-500ms) are unacceptable for interactive use. Optional `signed=true` query param on REST endpoints for callers who accept the latency.
**Alternatives:**
- Sign every request — unacceptable latency for interactive use
- Post-storage async signing — reports temporarily unsigned, regulatory gap
- Signing inside the renderer — tangles rendering with crypto, breaks single responsibility
**Rationale:** Scheduled reports are the regulatory submission path — they need signatures. On-demand reports are the interactive query path — they need speed. The `signed=true` opt-in covers the rare case where an on-demand report needs a signature (e.g., ad-hoc regulatory request). Graceful degradation: when DocumentSigningService is NoOp (not configured), unsigned reports pass through unchanged — signing is opt-in at the deployment level.
**Trade-offs:** On-demand reports are unsigned by default. Callers who need a signed copy should use the scheduled generation path or the `signed=true` param (with latency warning in API docs). Detached signatures for JSON/CSV require storing two SharedData artefacts (report + .p7s).
**Depends on:** D2 (DocumentSigningService SPI), D4 (CAdES for detached)
**Sources:** ComplianceReportResource.renderResponse(), ComplianceReportScheduler.generateAndStore(), ComplianceReportStorageService, R1-06
**Exploration:** quick → revised after decision review
**Status:** revised

## D6: Verification — platform service + compliance REST endpoint

**Choice:** Platform-level `DocumentVerificationService` SPI in platform-api (alongside DocumentSigningService) with implementation in casehub-platform-signing using DSS's SignedDocumentValidator. Compliance-report module exposes REST endpoint POST /api/compliance/verify that delegates to the platform service. Accepts multipart: report file + optional detached .p7s. Returns VerificationResult: outcome (VALID/INVALID/UNSIGNED/UNSUPPORTED_FORMAT/ERROR), signer identity (subject DN), signed at (timestamp), certificate chain summary, profile level.
**Alternatives:**
- Verification only in compliance-report — if another module needs verification, it must depend on qhorus compliance-report (boundary violation, R1-07)
- Separate endpoints per format — duplicates logic
- Verify by stored report ID only — can't verify externally received reports
**Rationale:** If signing is platform-level infrastructure, verification must be too. Any module that produces signed documents should be able to verify them without depending on qhorus. The compliance-report REST endpoint is a domain-specific convenience layer.
**Trade-offs:** Two layers (platform service + REST endpoint). But the platform service is the reusable primitive; the REST endpoint is the compliance-specific exposure.
**Depends on:** D1 (EU DSS SignedDocumentValidator), D2 (platform-signing module)
**Sources:** EU DSS validation module, ComplianceReportResource, R1-07
**Exploration:** quick → revised after decision review
**Status:** revised

## D7: Scope is "advanced electronic seal" — eIDAS-ready, not eIDAS-qualified

**Choice:** This design produces advanced electronic seals with eIDAS-ready architecture. Qualified status is a deployment-time concern: plug in QTSP-issued certificates, use QSCD-backed key material (HSM/cloud KMS via a DocumentSigningService impl), configure EU Trusted List validation. The code infrastructure supports all of this — qualification is a PKI provisioning and legal decision, not a code decision.
**Alternatives:**
- Build full QTSP integration now — requires legal/compliance decisions outside code scope
- Minimal signing without eIDAS framing — undervalues the architecture
**Rationale:** eIDAS qualification requires a QTSP contract, QSCD hardware, and qualified certificates — none of which are code concerns. The code concern is producing standards-compliant signature formats (PAdES, CAdES) that a QTSP's certificates can be plugged into. This is that code.
**Trade-offs:** Deployers must understand that using a self-signed or non-qualified certificate produces an advanced seal, not a qualified seal. Documentation must be clear on this distinction.
**Sources:** eIDAS Regulation (EU) No 910/2014 Articles 35-36, R1-03
**Exploration:** quick (surfaced by decision review)
**Status:** captured

## D8: Signing is deployment-opt-in via NoOp default

**Choice:** When no DocumentSigningService implementation is deployed (only NoOpDocumentSigningService @DefaultBean), reports are produced unsigned with no error. Signing activates when casehub-platform-signing is on the classpath and a PKCS#12 keystore is configured. Unsigned reports should carry a `signatureStatus: UNSIGNED` field in their metadata for API consumers to distinguish.
**Alternatives:**
- Require signing configuration — fails startup without a keystore; too aggressive for dev/test
- No status field — consumers can't distinguish "not configured" from "signing failed"
**Rationale:** Same pattern as PdfGenerator (NoOp @DefaultBean returns Optional.empty()). Signing is infrastructure that not every deployment needs. The status field gives API consumers a reliable signal.
**Depends on:** D2 (DocumentSigningService SPI), D5 (pipeline position)
**Sources:** NoOpPdfGenerator pattern, R1-15
**Exploration:** quick (surfaced by decision review)
**Status:** captured
