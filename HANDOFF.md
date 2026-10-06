# Session Handover — 2026-10-06

## What happened

Landed #443/#444 (correction/retraction modeling + message-scoped erasure) via `land-443-444` branch — 16 commits fast-forward merged to main. Fixed compilation errors from ledger package reorganisation, migrated erasure tests from deleted `QhorusMcpTools` to `QhorusTestHelper`, registered `correct_message`/`retract_message`/`erase_message_content` via `@McpDomain` on `MessagingApi`.

Fixed CDI `LedgerEntryRepository` ambiguity caused by upstream `casehub-ledger` SNAPSHOT removing `@Alternative` from JPA implementations. Added `quarkus.arc.exclude-types` across 13 module `application.properties` files. Full build green — 3154 tests, 0 failures.

Designed distributed qhorus mesh (#475) — standalone service with clustering. 11 design decisions captured. Code trace revealed 3 subsystems that break under multi-node writes (Merkle chain, LAST_WRITE, corrections). Adopted hybrid hash-ring + DB-lock model.

## Key decisions

- Hybrid write model: hash ring routes writes to channel owner (performance), DB-level locks as safety net for edge cases (correctness)
- Shared PostgreSQL, not distributed database — write-ownership partitioning only
- Multi-protocol transport: REST+SSE, MCP-over-SSE, A2A, WebSocket — all facades over one service layer
- OIDC auth primary, API key fallback
- Non-Java SDKs deferred — REST/GraphQL is the universal client interface

## Next action

Execute Phase 1 plan — 3 surgical runtime changes: `SELECT FOR UPDATE` on Merkle frontier, `@Version` on `MessageEntity`, `findByCorrelationIdForUpdate` for commitments.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-475-distributed-mesh/2026-10-06-distributed-mesh-design.md` |
| Decisions | `specs/issue-475-distributed-mesh/decisions.md` |
| Phase 1 plan | `plans/2026-10-06-distributed-mesh-phase1-runtime-safety-net.md` |
| Diary entry | `blog/2026-10-06-mdp01-the-lock-that-doesnt-lock.md` |
| Epic issue | casehubio/qhorus#475 |

## Project state

- **Project branch:** `main` — 18 commits ahead of `origin/main` (unpushed)
- **Workspace branch:** `issue-475-distributed-mesh`
- Build: green (`mvn clean test` — all modules pass)
