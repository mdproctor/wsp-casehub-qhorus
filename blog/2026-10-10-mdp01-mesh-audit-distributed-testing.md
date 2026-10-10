---
entry_type: note
subtype: diary
title: "The quorum hole and the distributed testing push"
date: 2026-10-10
author: mdp
issue: 484
tags: [distributed, cluster, e2e, testing, audit]
---

# The quorum hole and the distributed testing push

The distributed mesh has been building up over seven phases — cluster management, ownership routing, cache coherence, postgres-broadcaster for cross-node delivery. All of it wired with CDI-free POJOs and unit tests. The unit tests proved each component worked. What they couldn't prove was whether the components worked *together*, across real network boundaries, with real PostgreSQL, and real container restarts.

That's what issue #484 was for: audit the wiring, improve the error handling, and write the e2e tests that would catch the distributed failures the unit tests can't see.

## The decorator with a hole

The audit started with tracing every write path into `MessageService.dispatch()` — REST, MCP, A2A, connector, internal mesh. The enforcement gate (rate limiting, ACL, protocol enforcement, quorum check) is the single convergence point. Every path should go through it.

They did. Except one.

`ChannelManagerDecorator.findOrCreate()` delegated directly to the underlying service without a quorum check. Every other method in the decorator — `create()`, `delete()`, `pause()`, `resume()`, and a dozen config mutations — had the guard. But `findOrCreate` was the original pass-through from before quorum was introduced, and when the concern was added to the other methods, this one was overlooked.

The name itself was part of the camouflage. "Find or create" sounds read-heavy — you're *finding* something, and only incidentally creating it if it doesn't exist. But the create path is a write, and in a minority partition, that write should be rejected. A partitioned node could create channels via `findOrCreate` while `create()` on the same node would correctly throw `QuorumViolationException`.

Claude's design review subagent flagged it. The fix was three lines — a quorum check at the top of the method — but the finding was worth a garden entry because the pattern is universal. Any decorator that adds a cross-cutting concern must audit every interface method, including the ones that were pass-throughs before the concern existed. Time makes the gap invisible: a reviewer seeing the decorator with twelve correctly guarded methods would assume completeness.

## Typed exceptions for the proxy path

The cross-node proxy path (`WriteProxyClient`) was throwing generic `RuntimeException` for everything — timeouts, auth failures, 500s from the remote node. All of them landed in the same catch block in `WriteRoutingDecorator`, logged at WARN, and fell back to local dispatch.

That meant an auth misconfiguration (wrong `X-Internal-Secret`) looked identical to a transient network timeout in the logs. We split it into three exception types: `ProxyTimeoutException` for network-level failures (WARN — transient, expected during ownership transfer), `ProxyAuthException` for 401/403 (ERROR — misconfiguration that needs immediate attention), and the existing `ProxyDispatchException` for everything else.

The refactoring also extracted a `fallbackToLocal()` helper that eliminated three identical catch-block copies in the decorator. The typed exceptions plus the extracted helper made the dispatch path's failure taxonomy visible in the code structure, not just in log messages.

## Thirteen scenarios across four test classes

The e2e infrastructure was already in place from the earlier phases — `ClusterTestHarness` spinning up PostgreSQL and N mesh containers via Testcontainers on Podman. Three test classes existed: dispatch routing, node failure detection, and quorum enforcement. We added four more.

**Ownership transfer** tests the dynamic ownership evaluator end-to-end. Twenty messages from node-b trigger a claim that exceeds the hysteresis threshold. A new `/health/cluster/ownership/{channelId}` endpoint distinguishes hash-ring default ownership from dynamic claims, so the test can assert that ownership transferred via a claim — not that the hash ring happened to assign it. The test then verifies that messages from node-a are proxied to node-b (the new owner), and that ownership reverts to hash-ring when writes stop.

**Node failure fallback** fills the gap the existing `NodeFailureE2ETest` left. The original test proved the cluster detects a dead node and that a restarted node rejoins. But it never tested what happens to *dispatches* during the outage. The new test stops node-a, verifies node-b accepts writes via fallback-to-local, then restarts node-a and verifies that ownership claims reconstruct via heartbeat propagation.

**Cache coherence** validates the postgres-broadcaster path. Messages dispatched on node-a should appear in node-b's in-memory cache via `LISTEN/NOTIFY` → `CachePopulationObserver`. The test checks cache stats, rapid message convergence, and cache invalidation when a channel is deleted.

**Channel creation routing** tests `preAssignedId` — the mechanism that lets the cluster route channel creation to the correct owner node before the channel exists in the database. A UUID is pre-assigned, hashed to determine the owner, and the creation request is proxied if needed. The test also fires concurrent `findOrCreate` from both nodes to verify idempotency across the shared database.

## Native image: the opt-in validation path

The mesh containers run on JVM mode with 256MB heap. We added a native image variant — `Dockerfile.native` on UBI9-minimal with 128MB — activated via `-Pwith-e2e-native`. The harness selects between JVM and native mode via a `mesh.container.mode` system property that switches the Docker build context.

This is opt-in, not the default. Native builds take minutes and native-only bugs (reflection registration, Vert.x/reactive issues) are a different class of problem from what the e2e suite is designed to catch. But having the path available means native image validation is one Maven flag away, not a separate project.

## What this opens up

The e2e suite now covers the full distributed lifecycle: creation, dispatch, ownership transfer, failure, recovery, cache coherence. The mesh module can be tested under realistic conditions without deploying to Kubernetes. The native image path means GraalVM compatibility can be validated on the same infrastructure.

The next gap is performance characterisation — the e2e tests prove correctness but say nothing about latency under load, ownership transfer convergence time, or cache hit rates. That's a separate concern and a separate issue.
