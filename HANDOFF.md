# Session Handover — 2026-10-08

## What happened

Completed #488 (cluster hardening epic) — all three sub-tasks landed on main as `339a5464`. Closed #476 (cache CDI wiring, completed within #488). Labeled all open issues with scale/complexity.

- **Split-brain safety**: `ProxyFallbackEvent` CDI event + configurable fail-fast via `casehub.qhorus.relay.proxy-fallback` (local|fail). `WriteRoutingDecorator` fires event on every proxy failure; `fail` mode throws `ProxyDispatchException`.
- **Cache CDI wiring (#476)**: `CacheCdiWiringTest`, `CacheDisabledTest`, `CacheHealthResourceTest`. Changed `enableIfMissing` from `true` to `false` — cache now requires explicit opt-in. Garden entry GE-20261008-6575cd captures the `enableIfMissing` gotcha.
- **E2E test**: `cross_node_dispatch_via_proxy` — sends from node-b to channel on node-a, verifies via shared PostgreSQL.

## Decisions

- `enableIfMissing=false` for cache module — consistent with relay, correct for optional library
- `ProxyFallbackEvent` stays in `cluster/` package (not `api/`) — cluster-internal for now, promote if external consumers need it

## Next action

#484 (distributed mesh audit and E2E testing) — the last open child of #475. The E2E harness and first test exist; remaining work is extending coverage and running the code audit.

## References

| Artifact | Location |
|----------|----------|
| Project commits | `fccfba6f`..`339a5464` on main (5 commits) |
| Spec | `docs/specs/issue-488-cluster-hardening/` |
| Diary | `docs/blog/2026-10-08-mdp01-hardening-the-cluster-layer.md` |
| Garden | GE-20261008-6575cd (enableIfMissing gotcha) |

## Project state

- Both repos on main, branch `issue-488-cluster-hardening` closed and stamped
- Build: green (105 cluster tests, 38 cache tests)
- #475 epic: 12/13 children closed, #484 remains
