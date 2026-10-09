## D1: Work ordering — audit → improve → e2e

**Choice:** Sequential: audit first (identifies gaps), improvement second (fixes gaps), e2e third (validates the fixed system)
**Alternatives:**
- Interleaved (e2e first to catch issues, then audit) — risks writing e2e tests against code that changes during audit
- Parallel (audit + e2e simultaneously) — no dependency benefit, harder to coordinate
**Rationale:** Each phase feeds the next — audit findings drive improvements, improvements must be validated by e2e
**Trade-offs:** None significant — this is the natural dependency order
**Sources:** Issue #484 scope definition
**Exploration:** quick
**Status:** captured

## D2: E2E test infrastructure — extend existing harness

**Choice:** Extend the existing ClusterTestHarness with new helper methods; each test class gets its own cluster lifecycle (PER_CLASS)
**Alternatives:**
- Shared cluster across all test classes — faster but fragile (ordering deps, leaked state, harder debugging)
- Compose-based infrastructure — more realistic but harder to integrate with JUnit lifecycle
**Rationale:** The per-class pattern is proven by 3 existing test classes. The harness is clean and just needs helper methods for ownership, cache stats, and channel creation assertions.
**Trade-offs:** Each test class pays ~30-60s startup cost for its own Podman cluster. Acceptable for 6 test classes total.
**Sources:** ClusterTestHarness.java, DispatchRoutingE2ETest.java, NodeFailureE2ETest.java, QuorumEnforcementE2ETest.java
**Exploration:** quick
**Status:** captured

## D3: Native image approach — separate Dockerfile + conditional profile

**Choice:** Add `Dockerfile.native` alongside the existing JVM Dockerfile. Add Maven profile `-Pwith-e2e-native` that builds native image first and swaps the Docker image used by ClusterTestHarness.
**Alternatives:**
- Replace JVM with native everywhere — faster containers but native builds take ~5min and native image issues block the entire e2e suite
**Rationale:** JVM-mode stays the fast path for CI. Native mode is opt-in validation that catches reflection/serialization issues without gating the primary e2e suite.
**Trade-offs:** Two container images to maintain. Native image build adds ~5min when activated. Possible native-only bugs won't be caught in default CI.
**Sources:** Dockerfile (e2e-cluster/src/test/resources/), mesh/pom.xml, GraalVM 25 on this machine
**Exploration:** quick
**Status:** captured

## D4: Error handling — typed ProxyException hierarchy

**Choice:** Create `ProxyTimeoutException` and `ProxyAuthException` alongside the existing `ProxyDispatchException`. WriteProxyClient maps HTTP status codes to specific exception types.
**Alternatives:**
- Status-code-aware RuntimeException — simpler but callers can't programmatically distinguish failure modes
**Rationale:** The fail-fast path already uses `ProxyDispatchException`, so we're extending an established pattern. Typed exceptions let callers react differently to auth failures (misconfiguration, log loudly) vs timeouts (transient, fallback silently).
**Trade-offs:** 2 new exception classes. Marginal complexity increase in WriteProxyClient's catch blocks.
**Sources:** WriteProxyClient.java, ProxyDispatchException.java, WriteRoutingDecorator.java
**Exploration:** quick
**Status:** captured
