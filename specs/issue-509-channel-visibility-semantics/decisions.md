# Decisions — Channel Visibility Semantics (#509)

## D1: DIRECT channel naming scheme

**Choice:** Sorted slug path — `dm/{sorted-a}/{sorted-b}`
**Alternatives:**
- Hash-based name (`dm/{hash(sorted(a,b))}`) — works with any ID format but opaque, lookup requires hash computation
- Metadata lookup (arbitrary slug + `dm.participants` metadata) — flexible but loses deterministic naming, requires metadata query
**Rationale:** Readable, deterministic, slug-valid. Leverages existing `findByName` for deduplication. Instance IDs are already valid slug segments in the Qhorus ecosystem.
**Trade-offs:** Requires instance IDs to be valid slug segments. UUIDs as instance IDs would not work — but Qhorus instance IDs are already human-readable slugs.
**Sources:** `api/channel/ChannelSlugValidator.java` — slug validation rules; `api/store/ChannelStore.java` — `findByName()` for dedup
**Exploration:** quick
**Status:** captured

## D2: Group-bound channel naming and binding

**Choice:** groupName slug + metadata — `group/{groupName-slug}` with `metadata["boundGroupId"] = groupId`
**Alternatives:**
- groupId hash in name (`group/{hash(groupId)}`) — works regardless of groupName format but opaque
- groupName only, no metadata — loses groupId binding, can't lookup by UUID
**Rationale:** Readable channel names. `ChannelQuery.byMetadata("boundGroupId", groupId)` provides UUID-based lookup. groupName comes from eidos OrgUnit which uses human-readable names.
**Trade-offs:** groupName must be convertible to a valid slug. If eidos OrgUnit names contain non-slug characters, the caller must slugify.
**Sources:** `api/store/query/ChannelQuery.java` — `byMetadata(key, value)` factory method already exists
**Exploration:** quick

## D3: DIRECT membership immutability enforcement

**Choice:** Guard in `ChannelMembershipService.join()/leave()`
**Alternatives:**
- Separate DirectChannelService — more explicit but splits membership logic across two services
- Store-layer guard — pushes domain logic into persistence layer
**Rationale:** Single enforcement point. All callers (MCP tools, REST, internal, gateway auto-join) hit the same guard. Consistent with how `participationPolicy` is enforced in `MessageService.dispatch()` — domain rules live in service layer.
**Trade-offs:** Adds visibility awareness to ChannelMembershipService. Requires injecting a channel lookup (to read visibility) into the membership service.
**Sources:** `runtime-core/channel/ChannelMembershipService.java` — current `join()` implementation
**Exploration:** quick
**Status:** captured

## D4: ChannelVisibility storage

**Choice:** Typed enum field on `Channel` record + `ChannelEntity` column
**Alternatives:**
- Metadata convention (`metadata["visibility"]`) — no schema change but string-typed, not queryable or enforceable at compile time
**Rationale:** Typed field enables compile-time safety, DB-level queries (e.g. `WHERE visibility = 'DIRECT'`), and validation in the compact constructor. Consistent with `participationPolicy`, `enforcementMode`, and `semantic` — all enum fields on Channel.
**Trade-offs:** Adds 1 field to the 31-field Channel record. Requires V59 Flyway migration. Another backward-compat constructor.
**Sources:** `api/channel/Channel.java` — record with existing enum fields; `runtime/channel/ChannelEntity.java` — JPA entity
**Exploration:** quick
**Status:** captured

## D5: API surface for visibility operations

**Choice:** New `VisibilityManager` interface in `api/channel/`
**Alternatives:**
- Extend ChannelManager — all channel ops in one interface but grows an already-large interface (18 methods)
- Split queries on ChannelReader, mutations on ChannelManager — splits visibility logic across two existing interfaces
**Rationale:** Keeps ChannelManager focused on core channel CRUD/config. `VisibilityManager` owns the visibility-specific convenience operations: `getOrCreateDirect`, `createGroupChannel`, `addMember`, `removeMember`, `findDirectChannelsFor`, `findGroupChannel`, `findByVisibility`. `ChannelService` implements both interfaces.
**Trade-offs:** Another interface to inject. Consumers that want both general and visibility ops inject both `ChannelManager` and `VisibilityManager`.
**Sources:** `api/channel/ChannelManager.java` — 18 existing methods; `api/channel/MembershipManager.java` — 6 methods
**Exploration:** quick
**Status:** captured

## D6: Default semantics application

**Choice:** Creation-time defaults, overridable
**Alternatives:**
- Strict invariants — locks semantic and participationPolicy to visibility; maximum safety but rigid
- Creation defaults + soft validation — advisory warnings on post-creation conflicts; adds complexity for marginal benefit
**Rationale:** When `ChannelCreateRequest` specifies a visibility, semantic and participationPolicy default to the visibility's conventions if not explicitly set. Caller can override for legitimate use cases (e.g. a PRIVATE channel with BARRIER semantic). No post-creation enforcement — once created, the channel behaves according to its actual settings, not its visibility category.
**Trade-offs:** A channel could be marked DIRECT but have its participation policy changed to OPEN post-creation — visibility becomes descriptive rather than prescriptive. The membership guard (D3) still enforces the 2-member invariant regardless of participationPolicy.
**Sources:** Issue #509 R6 — "Caller can override"
**Exploration:** quick
**Status:** captured

## D7: Existing channel migration strategy

**Choice:** Null = unset (no visibility constraints)
**Alternatives:**
- Default to PUBLIC — retroactive semantic assignment; simpler model but may surprise existing deployments
**Rationale:** Purely opt-in. Existing channels have `visibility = null`. Visibility guards check `visibility != null` before enforcing — null channels behave exactly as they do today. Zero migration risk.
**Trade-offs:** Two channel populations coexist (visibility-aware and legacy). Queries like `findByVisibility(PUBLIC)` won't return legacy channels that are semantically public but lack the field.
**Sources:** `runtime/channel/ChannelEntity.java` — nullable column pattern (matches `participationPolicy`, `enforcementMode`)
**Exploration:** quick
**Status:** captured
