# Design Journal — issue-314-store-spi-to-api

## 2026-06-30 — Session 1: Structural architecture + partial implementation

### Design decisions

1. **Domain records own clean names** — `Channel`, `Message`, `Commitment` etc. are immutable Java records in `api/`. JPA entities renamed to `*Entity` suffix in `runtime/`. Rationale: domain model IS the concept; persistence is an implementation detail.

2. **CSV entity fields → typed collections** — Entity fields like `allowedTypes` (CSV String) become `Set<MessageType>` on domain records. Conversion happens at the JPA boundary via `fromDomain()`/`toDomain()`. Consistent with `ChannelCreateRequest` which already uses typed sets.

3. **`Channel.fromRequest()` factory on the domain record** — generates UUID and timestamps, replacing `@PrePersist` as the primary source. Entity `@PrePersist` remains as JPA safety net.

4. **`@Entity(name = "OldName")` on all renamed entities** — preserves JPQL entity names. Discovered necessity when IntelliJ incorrectly renamed JPQL string literals.

5. **ADR-0017 needed** — reverses ADR-0002 "No Info record layer". The original rationale ("Panache entities are POJOs") no longer holds — entities carry JPA annotations that force persistence-memory/ to depend on full runtime/.

### What was implemented

- **Task 1** ✅ — 10 JPA entities renamed to `*Entity` via IntelliJ refactor (253 files, committed)
- **Task 2** ✅ — 9 domain records created in api/ with builders, TDD (63 files, committed)
- **Task 3** ✅ — `fromDomain()`/`toDomain()` conversion on all 9 entities, round-trip tests (37 files, committed)
- **Task 4** 🟡 — Store SPIs (18) + Query types (5) moved to api/store/. SPI signatures updated to domain records. JpaChannelStore updated as reference. **Not compiling** — remaining JPA stores, InMemory stores, services, and tests need the same mechanical entity→record boundary updates.

### Design review

8-round adversarial design review ($22.80). 18 issues raised, 15 verified (spec updated), 3 accepted. Key review-driven improvements: typed CSV fields, `Capability` entity inclusion, `@Entity(name=...)` convention, cross-tenant store signatures, cross-repo consumer audit (found casehub-ops, drafthouse, clinical consumers), Maven dependency inversion as primary outcome statement, Mutiny `provided` dep for reactive SPIs.

### Remaining work

The pattern is established (see `JpaChannelStore` reference implementation). Remaining is mechanical — apply the same fromDomain/toDomain boundary conversion to:
- 17 more JPA stores (7 blocking, 6 reactive, 4 cross-tenant)
- ~19 InMemory stores in persistence-memory/ (change to use domain records directly)
- ~9 service files (change entity field access to record accessors)
- ~95 test files
- persistence-memory/pom.xml (dep from casehub-qhorus → casehub-qhorus-api)
- FindOrCreateResult (ChannelEntity → Channel)
- testing/ duplicate cleanup (Task 5)
- CLAUDE.md, ADR-0017, cross-repo issues (Task 6)
