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
