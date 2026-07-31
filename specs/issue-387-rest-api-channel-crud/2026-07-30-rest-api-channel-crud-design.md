# REST API for Channel CRUD — Design Spec

**Issue:** casehubio/qhorus#387
**Date:** 2026-07-30
**Status:** approved

## Problem

Qhorus exposes channel management via MCP tools and CDI-injectable services but has no REST API. Applications like Claudony that need to expose channel operations to browser clients proxy through their own REST resources, reimplementing the same thin layer.

## Solution

A JAX-RS resource in `runtime/api/` that maps the full `ChannelService` surface to REST endpoints with proper JSON types.

## Decisions

- **Always-on** — no config gate. Trivial to add later if needed.
- **REST-specific DTOs** — request/response records with `List<String>` and `Set<String>` instead of CSV strings.
- **Action-oriented sub-resources** — pause/resume as POST actions, settings as PUT sub-resources. No single PATCH endpoint.
- **Blocking only** — no reactive counterpart initially.
- **Connector binding excluded** — internal integration concern, not consumer-facing.

## Endpoints

Base path: `/api/channels`

### CRUD

| Method | Path | Service method | Status |
|--------|------|----------------|--------|
| POST | `/api/channels` | `channelService.create()` | 201 |
| GET | `/api/channels` | `channelService.listAll()` / `scan(query)` | 200 |
| GET | `/api/channels/{id}` | `findById()` / `findByName()` | 200 |
| DELETE | `/api/channels/{id}?force=true` | `channelService.delete()` | 204 |

GET list supports query params: `prefix` (name prefix filter), `spaceId` (space filter), `paused` (boolean filter) — mapped to `ChannelQuery`.

### Lifecycle actions

| Method | Path | Service method | Status |
|--------|------|----------------|--------|
| POST | `/api/channels/{id}/pause` | `channelService.pause()` | 200 |
| POST | `/api/channels/{id}/resume` | `channelService.resume()` | 200 |

### Settings sub-resources

| Method | Path | Service method | Status |
|--------|------|----------------|--------|
| PUT | `/api/channels/{id}/allowed-writers` | `setAllowedWriters()` | 200 |
| PUT | `/api/channels/{id}/admin-instances` | `setAdminInstances()` | 200 |
| PUT | `/api/channels/{id}/reviewer-instances` | `setReviewerInstances()` | 200 |
| PUT | `/api/channels/{id}/type-constraints` | `setTypeConstraints()` | 200 |
| PUT | `/api/channels/{id}/rate-limits` | `setRateLimits()` | 200 |
| PUT | `/api/channels/{id}/protocols` | `setProtocols()` | 200 |
| PUT | `/api/channels/{id}/protocol-participants` | `setProtocolParticipants()` | 200 |
| PUT | `/api/channels/{id}/delivery-tracking` | `setTrackDelivery()` | 200 |

All mutation endpoints return `ChannelResponse` with the updated state.

### Channel resolution

The `{id}` path param follows the channel dual-identity protocol: UUID parse first, slug name fallback. Resolution happens once at the resource boundary via a private `resolve(String)` method.

## DTOs

All records in `io.casehub.qhorus.runtime.api`.

### CreateChannelRequest

```java
record CreateChannelRequest(
    String name,                      // required, slug format
    String description,               // optional
    String semantic,                  // optional, defaults to APPEND
    List<String> barrierContributors, // optional
    List<String> allowedWriters,      // optional
    List<String> adminInstances,      // optional
    List<String> reviewerInstances,   // optional
    Set<String> allowedTypes,         // MessageType names, optional
    Set<String> deniedTypes,          // MessageType names, optional
    List<String> protocols,           // optional
    List<String> protocolParticipants,// optional
    UUID spaceId,                     // optional
    Boolean trackDelivery,            // optional
    Integer rateLimitPerChannel,      // optional
    Integer rateLimitPerInstance      // optional
)
```

### Sub-resource request records

Inner records on `ChannelResource` — single-use, not worth top-level files.

```java
record TypeConstraintsRequest(Set<String> allowedTypes, Set<String> deniedTypes)
record RateLimitsRequest(Integer perChannel, Integer perInstance)
record StringListRequest(List<String> values)  // shared by allowed-writers, admin-instances, reviewer-instances, protocol-participants
record DeliveryTrackingRequest(Boolean enabled)
```

### ChannelResponse

```java
record ChannelResponse(
    UUID channelId,
    String name,
    String description,
    String semantic,
    long messageCount,
    String lastActivityAt,
    boolean paused,
    List<String> barrierContributors,
    List<String> allowedWriters,
    List<String> adminInstances,
    List<String> reviewerInstances,
    Set<String> allowedTypes,
    Set<String> deniedTypes,
    Integer rateLimitPerChannel,
    Integer rateLimitPerInstance,
    UUID spaceId,
    String spaceName,
    List<String> protocols,
    List<String> protocolParticipants,
    Boolean trackDelivery
)
```

## Resource class

`ChannelResource` in `io.casehub.qhorus.runtime.api`:
- `@Path("/api/channels")`, `@ApplicationScoped`
- `@Produces(APPLICATION_JSON)`, `@Consumes(APPLICATION_JSON)`
- Injects: `ChannelService`, `MessageStore` (for count), `SpaceStore` (for space name), `CurrentPrincipal`

### Mapping

Private `toResponse(Channel)` method splits CSV strings into typed collections and resolves space name. Alternatively placed on `QhorusEntityMapper` alongside `toChannelDetail()`.

### Error handling

| Exception | HTTP status | When |
|-----------|-------------|------|
| `NotFoundException` | 404 | Channel or space not found |
| `IllegalArgumentException` | 400 | Bad slug, invalid semantic, type constraint overlap |
| `IllegalStateException` | 409 | Delete non-empty without force, space has children |

Resource catches service exceptions and wraps in `WebApplicationException` with `{"error": "message"}` body. No custom exception mapper.

## Testing

`@QuarkusTest` with RestAssured. Tests cover:
- Create channel → verify 201 + response body
- List channels → verify filtering (prefix, spaceId, paused)
- Get by UUID and by slug
- Delete (soft + force)
- Pause/resume lifecycle
- Each settings sub-resource (PUT + verify updated response)
- Error paths: duplicate name (400), not found (404), delete without force (409)

Uses existing H2 test infrastructure. No new test profile needed.

## Files changed

| File | Change |
|------|--------|
| `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java` | New |
| `runtime/src/main/java/io/casehub/qhorus/runtime/api/CreateChannelRequest.java` | New |
| `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java` | New |
| `runtime/src/test/java/io/casehub/qhorus/api/ChannelResourceTest.java` | New |
| `CLAUDE.md` | Document new REST API endpoints |
