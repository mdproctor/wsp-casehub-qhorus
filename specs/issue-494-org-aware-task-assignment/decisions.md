# Decisions — #494 org-aware task assignment

## D1: Where to apply org preference

**Choice:** Filter-then-fallback in RoutingBridge
**Alternatives:**
- Enrich SelectionContext with org hints — cleaner separation but requires eidos API change and all selector impls must handle it
**Rationale:** Self-contained in qhorus, follows existing RoutingBridge pattern (trust threshold, capacity filtering), no cross-repo API changes needed
**Trade-offs:** Org preference logic lives in qhorus rather than eidos; if other consumers need org-aware selection they'd duplicate the filter logic
**Sources:** RoutingBridge.java:58-109, OrgAwareDispatch interface, issue #494
**Exploration:** quick
**Status:** captured
