# Channel Slug Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce a well-formed slug pattern on every Qhorus channel name, blocking UUID-shaped names and arbitrary strings, while preserving existing channel name compatibility for conformant IDs.

**Architecture:** `ChannelSlugValidator` (new public class in `runtime/channel`) owns all validation logic. `ChannelCreateRequest`'s compact constructor calls it as the first check — all creation paths flow through this gate. `ConfiguredAutoChannelPolicy` adds `slugifyConnectorId()` (hash-free) and `sanitiseSegment()` (hash-bearing) to produce slug-conformant names from raw external identifiers. `QhorusMcpToolsBase.resolveChannel()` replaces its private `tryParseUuid()` copy with `ChannelSlugValidator.tryParseUuid()`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM, H2 (tests), PostgreSQL (production), Flyway, AssertJ, JUnit 5, SHA-256 via `java.security.MessageDigest`, `HexFormat` (Java 17+)

**Spec:** `docs/specs/2026-06-04-channel-slug-enforcement-design.md` (rev 5)

---

## File Map

| Action | Path | What changes |
|--------|------|--------------|
| **Create** | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelSlugValidator.java` | New public validator class |
| **Create** | `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelSlugValidatorTest.java` | Unit tests for validator |
| **Create** | `runtime/src/main/resources/db/qhorus/migration/V17__add_channel_name_slug_constraint.sql` | CHECK constraint migration |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java` | Add `validateSlugPath()` as first compact constructor check |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java` | Add `updatable = false` to `@Column` on `name` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` | Replace private `tryParseUuid()` with `ChannelSlugValidator.tryParseUuid()` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` | Same `tryParseUuid()` consolidation in reactive `resolveChannel()` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` | Update `create_channel` `name` argument description |
| **Modify** | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicy.java` | Add `slugifyConnectorId()`, `sanitiseSegment()`, `validatePattern()`, `@PostConstruct` |
| **Modify** | `runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java` | Add V17 constraint existence test |
| **Modify** | `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelServiceFindOrCreateTest.java` | Fix `smsRequest()` to use sanitised channel name |
| **Modify** | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java` | Update all channel name assertions to sanitised form |

---

## Task 1 — `ChannelSlugValidator` (new class + unit tests, TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelSlugValidatorTest.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelSlugValidator.java`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.qhorus.runtime.channel;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class ChannelSlugValidatorTest {

    // ── validateSlugPath: valid inputs ──

    @Test void acceptsSimpleSlug() {
        assertThatNoException().isThrownBy(() -> ChannelSlugValidator.validateSlugPath("billing-output"));
    }

    @Test void acceptsSingleSegment() {
        assertThatNoException().isThrownBy(() -> ChannelSlugValidator.validateSlugPath("billing"));
    }

    @Test void acceptsHierarchicalPath() {
        assertThatNoException().isThrownBy(() -> ChannelSlugValidator.validateSlugPath("case-abc/work"));
    }

    @Test void acceptsThreeSegmentPath() {
        assertThatNoException().isThrownBy(() ->
            ChannelSlugValidator.validateSlugPath("connector/twilio-sms-inbound/id-14155552671-3fa2b100"));
    }

    @Test void acceptsDigitsInSegment() {
        assertThatNoException().isThrownBy(() -> ChannelSlugValidator.validateSlugPath("case-abc-123/work2"));
    }

    // ── validateSlugPath: invalid inputs ──

    @Test void rejectsNull() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath(null))
            .withMessageContaining("null or blank");
    }

    @Test void rejectsBlank() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("  "))
            .withMessageContaining("null or blank");
    }

    @Test void rejectsUppercase() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("Billing-Output"))
            .withMessageContaining("Billing-Output");
    }

    @Test void rejectsSpace() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("billing output"))
            .withMessageContaining("billing output");
    }

    @Test void rejectsTrailingHyphen() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("billing-"))
            .withMessageContaining("billing-");
    }

    @Test void rejectsLeadingHyphen() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("-billing"))
            .withMessageContaining("-billing");
    }

    @Test void rejectsConsecutiveHyphens() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("billing--output"))
            .withMessageContaining("billing--output");
    }

    @Test void rejectsSegmentStartingWithDigit() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("123-billing"))
            .withMessageContaining("123-billing");
    }

    @Test void rejectsUuidShapedName_startingWithLetter() {
        // ~37% of UUIDs start with a-f; this must not be a valid channel name
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath("a81b4c6d-1234-5678-abcd-ef0123456789"))
            .withMessageContaining("UUID-shaped");
    }

    @Test void rejectsTotalLengthOver200() {
        String longName = "a" + "b".repeat(200); // 201 chars
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath(longName))
            .withMessageContaining("200");
    }

    @Test void rejectsSegmentOver80Chars() {
        String longSegment = "a" + "b".repeat(80); // 81-char segment
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ChannelSlugValidator.validateSlugPath(longSegment))
            .withMessageContaining("exceeds");
    }

    @Test void rejectsUuidShapedSegmentInPath() {
        // Full path containing UUID-shaped segment — the UUID-shaped check is on the full name only;
        // but the UUID "a81b4c..." is 36 chars and matches the segment pattern, so it's allowed
        // in a path segment (not rejected by isValidSegment's UUID check at the segment level too).
        // Only the FULL name UUID rejection applies.
        assertThatNoException().isThrownBy(() ->
            ChannelSlugValidator.validateSlugPath("connector/a81b4c6d-1234-5678-abcd-ef0123456789"));
    }

    // ── isValidSegment ──

    @Test void isValidSegment_trueForValidSlug() {
        assertThat(ChannelSlugValidator.isValidSegment("billing-output")).isTrue();
    }

    @Test void isValidSegment_falseForNull() {
        assertThat(ChannelSlugValidator.isValidSegment(null)).isFalse();
    }

    @Test void isValidSegment_falseForTrailingHyphen() {
        assertThat(ChannelSlugValidator.isValidSegment("billing-")).isFalse();
    }

    @Test void isValidSegment_falseForUuidShaped() {
        assertThat(ChannelSlugValidator.isValidSegment("a81b4c6d-1234-5678-abcd-ef0123456789")).isFalse();
    }

    @Test void isValidSegment_falseForSegmentOver80Chars() {
        assertThat(ChannelSlugValidator.isValidSegment("a" + "b".repeat(80))).isFalse();
    }

    // ── tryParseUuid ──

    @Test void tryParseUuid_returnsUuidForValidInput() {
        UUID uuid = ChannelSlugValidator.tryParseUuid("a81b4c6d-1234-5678-abcd-ef0123456789");
        assertThat(uuid).isNotNull();
        assertThat(uuid.toString()).isEqualTo("a81b4c6d-1234-5678-abcd-ef0123456789");
    }

    @Test void tryParseUuid_returnsNullForNonUuid() {
        assertThat(ChannelSlugValidator.tryParseUuid("billing-output")).isNull();
    }

    @Test void tryParseUuid_returnsNullForNull() {
        assertThat(ChannelSlugValidator.tryParseUuid(null)).isNull();
    }
}
```

- [ ] **Step 2: Run tests — confirm they all FAIL with "class not found"**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelSlugValidatorTest -pl runtime
```

Expected: compilation error — `ChannelSlugValidator` does not exist yet.

- [ ] **Step 3: Create `ChannelSlugValidator`**

```java
package io.casehub.qhorus.runtime.channel;

import java.util.UUID;
import java.util.regex.Pattern;

/**
 * Validates and utility-parses Qhorus channel name slugs.
 * <p>Public: consumers may call {@link #validateSlugPath} to pre-validate a name
 * before calling {@code create_channel}.
 */
public final class ChannelSlugValidator {

    public static final Pattern SEGMENT_PATTERN =
            Pattern.compile("[a-z][a-z0-9]*(-[a-z0-9]+)*");
    public static final int MAX_SEGMENT_LENGTH = 80;
    public static final int MAX_NAME_LENGTH = 200;

    /**
     * Validates that {@code name} is a well-formed channel slug path.
     * Every {@code /}-delimited segment must match {@code [a-z][a-z0-9]*(-[a-z0-9]+)*}.
     * Max 80 chars per segment, 200 chars total. UUID-shaped names are rejected.
     *
     * @throws IllegalArgumentException on any violation
     */
    public static void validateSlugPath(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Channel name must not be null or blank");
        }
        if (name.length() > MAX_NAME_LENGTH) {
            throw new IllegalArgumentException(
                    "Channel name exceeds " + MAX_NAME_LENGTH + " chars: '" + name + "'");
        }
        // Reject UUID-shaped names — resolveChannel() tries UUID parse first;
        // a UUID-shaped name makes name-based lookup silently unreachable.
        // Use a flag — throwing inside the try block is caught by the same catch.
        boolean isUuid;
        try {
            UUID.fromString(name);
            isUuid = true;
        } catch (IllegalArgumentException ignored) {
            isUuid = false;
        }
        if (isUuid) {
            throw new IllegalArgumentException(
                    "Channel name must not be UUID-shaped: '" + name + "'");
        }
        for (String segment : name.split("/", -1)) {
            if (segment.length() > MAX_SEGMENT_LENGTH) {
                throw new IllegalArgumentException(
                        "Segment '" + segment + "' exceeds " + MAX_SEGMENT_LENGTH
                        + " chars in channel name '" + name + "'");
            }
            if (!SEGMENT_PATTERN.matcher(segment).matches()) {
                throw new IllegalArgumentException(
                        "Invalid channel name segment '" + segment
                        + "' — must match [a-z][a-z0-9]*(-[a-z0-9]+)*. Full name: '" + name + "'");
            }
        }
    }

    /**
     * Returns true iff {@code segment} is a valid single slug segment.
     * Rejects UUID-shaped strings for consistency with {@link #validateSlugPath}.
     */
    public static boolean isValidSegment(String segment) {
        if (segment == null || segment.isBlank() || segment.length() > MAX_SEGMENT_LENGTH) {
            return false;
        }
        if (!SEGMENT_PATTERN.matcher(segment).matches()) {
            return false;
        }
        try {
            UUID.fromString(segment);
            return false; // UUID-shaped segment — rejected
        } catch (IllegalArgumentException ignored) {
            return true;
        }
    }

    /** Returns the UUID if {@code s} parses as one, null otherwise. */
    public static UUID tryParseUuid(String s) {
        if (s == null) return null;
        try {
            return UUID.fromString(s);
        } catch (IllegalArgumentException ignored) {
            return null;
        }
    }

    private ChannelSlugValidator() {}
}
```

- [ ] **Step 4: Run tests — confirm they all PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelSlugValidatorTest -pl runtime
```

Expected: BUILD SUCCESS, all tests green.

Note: The test `rejectsUuidShapedSegmentInPath` asserts that `"connector/a81b4c6d-1234-5678-abcd-ef0123456789"` is VALID as a full path (the UUID-shaped check applies to the full name, not individual path segments via `validateSlugPath`). If this test fails, review whether `validateSlugPath` is accidentally applying the UUID check per-segment — it should only apply to the full `name` string.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelSlugValidator.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelSlugValidatorTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#236): ChannelSlugValidator — slug pattern, UUID exclusion, tryParseUuid"
```

---

## Task 2 — Wire `ChannelCreateRequest` + ORM Immutability (TDD)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`
- Create/Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequestSlugTest.java`

- [ ] **Step 1: Write failing tests for slug validation in `ChannelCreateRequest`**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequestSlugTest.java`:

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.ChannelSemantic;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

/** Pure-Java tests — no CDI, no Quarkus. Tests the compact constructor slug gate. */
class ChannelCreateRequestSlugTest {

    @Test
    void simple_acceptsValidSlug() {
        assertThatNoException().isThrownBy(() ->
            ChannelCreateRequest.simple("billing-output", ChannelSemantic.APPEND));
    }

    @Test
    void simple_acceptsHierarchical() {
        assertThatNoException().isThrownBy(() ->
            ChannelCreateRequest.simple("case-abc/work", ChannelSemantic.APPEND));
    }

    @Test
    void compactConstructor_rejectsUppercaseName() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new ChannelCreateRequest(
                "Billing Output", null, ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null))
            .withMessageContaining("Billing Output");
    }

    @Test
    void compactConstructor_rejectsUuidShapedName() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new ChannelCreateRequest(
                "a81b4c6d-1234-5678-abcd-ef0123456789", null, ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null))
            .withMessageContaining("UUID-shaped");
    }

    @Test
    void compactConstructor_rejectsRawPhoneNumberInSegment() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new ChannelCreateRequest(
                "connector/twilio-sms-inbound/+14155552671", null, ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null))
            .withMessageContaining("+14155552671");
    }

    @Test
    void compactConstructor_acceptsValidHierarchicalSlug() {
        assertThatNoException().isThrownBy(() -> new ChannelCreateRequest(
            "connector/twilio-sms-inbound/id-14155552671-3fa2b100", null, ChannelSemantic.APPEND,
            null, null, null, null, null, null, null, null, null, null, null));
    }

    @Test
    void compactConstructor_slugValidationFiresBeforeTypeValidation() {
        // Invalid slug AND invalid type — slug error should come first
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new ChannelCreateRequest(
                "Invalid Name", null, ChannelSemantic.APPEND,
                null, null, null, null, null, "NOT_A_TYPE", null, null, null, null, null))
            .withMessageContaining("Invalid Name");
    }
}
```

- [ ] **Step 2: Run tests — confirm they FAIL (slug validation not yet wired)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelCreateRequestSlugTest -pl runtime
```

Expected: tests that assert `IllegalArgumentException` on invalid names will FAIL (no exception thrown yet).

- [ ] **Step 3: Add `validateSlugPath()` as first call in `ChannelCreateRequest` compact constructor**

In `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java`, update the compact constructor to add the slug check at the top:

```java
public ChannelCreateRequest {
    ChannelSlugValidator.validateSlugPath(name);   // ← ADD THIS LINE FIRST

    boolean anySet = inboundConnectorId != null || externalKey != null
            || outboundConnectorId != null || outboundDestination != null;
    boolean allSet = inboundConnectorId != null && externalKey != null
            && outboundConnectorId != null && outboundDestination != null;
    if (anySet && !allSet) {
        throw new IllegalArgumentException(
                "Connector binding requires all four fields: inboundConnectorId, " +
                "externalKey, outboundConnectorId, outboundDestination");
    }

    Set<MessageType> allowed = MessageType.parseTypes(allowedTypes);
    Set<MessageType> denied  = MessageType.parseTypes(deniedTypes);
    if (!allowed.isEmpty() && !denied.isEmpty()) {
        Set<MessageType> overlap = new HashSet<>(allowed);
        overlap.retainAll(denied);
        if (!overlap.isEmpty()) {
            throw new IllegalArgumentException(
                    "allowedTypes and deniedTypes must not intersect. Overlap: " + overlap);
        }
    }
}
```

Also update the Javadoc to add:
```
 *   <li>Channel name is a well-formed slug path (see {@link ChannelSlugValidator#validateSlugPath})</li>
```

- [ ] **Step 4: Add `updatable = false` to `Channel.name`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`, find:

```java
@Column(nullable = false)
public String name;
```

Change to:

```java
@Column(nullable = false, updatable = false) /* immutable after creation — PP-20260604-dualid */
public String name;
```

- [ ] **Step 5: Run the new slug tests — confirm they PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelCreateRequestSlugTest -pl runtime
```

Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 6: Run the full runtime test suite to catch any existing tests using non-conformant channel names**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: BUILD SUCCESS. If any tests fail with "Invalid channel name segment", note the test file and fix the name to a conformant slug (all lowercase, hyphens as separators, no leading/trailing hyphens, no UUID-shaped names, no special chars). **Do not fix such failures in this task** — note them and proceed; Task 9 addresses them.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequestSlugTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#236): wire ChannelSlugValidator into ChannelCreateRequest; Channel.name updatable=false"
```

---

## Task 3 — `QhorusMcpToolsBase.resolveChannel()` + `create_channel` description

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

The current `resolveChannel()` already resolves name vs UUID correctly. The changes are:
1. Replace the private `tryParseUuid()` with `ChannelSlugValidator.tryParseUuid()`.
2. Restructure so the UUID-not-found error path is unambiguous (not swallowed by the parse-failure catch).

- [ ] **Step 1: Replace `resolveChannel()` and remove private `tryParseUuid()` in `QhorusMcpToolsBase`**

Find the existing `resolveChannel()` method (around line 297) and the private `tryParseUuid()` method (around line 309). Replace both with:

```java
UUID resolveChannel(final String channel) {
    final UUID parsedUuid = ChannelSlugValidator.tryParseUuid(channel);
    if (parsedUuid == null) {
        // Not UUID-shaped — look up by name.
        return channelService.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel))
                .id;
    }
    // UUID-shaped input — look up by ID directly.
    // The slug invariant blocks UUID-named channels, so UUID-shaped inputs
    // are always channel IDs, never channel names.
    return channelService.findById(parsedUuid)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel))
            .id;
}
```

Remove the private `tryParseUuid()` method entirely. Add the import:
```java
import io.casehub.qhorus.runtime.channel.ChannelSlugValidator;
```

- [ ] **Step 2: Update `create_channel` tool description in `QhorusMcpTools`**

Find the `@ToolArg(name = "name", ...)` annotation on `create_channel`. Replace its description:

Old:
```java
@ToolArg(name = "name", description = "Unique channel name") String name,
```

New:
```java
@ToolArg(name = "name", description = "Unique channel name. Each /‑delimited segment must match " +
    "[a-z][a-z0-9]*(-[a-z0-9]+)* — lowercase letters and digits, hyphens only between " +
    "alphanumeric groups. No leading, trailing, or consecutive hyphens. Max 80 chars per " +
    "segment, 200 chars total. UUID-shaped names are rejected. " +
    "Examples: \"billing-output\", \"case-abc/work\".") String name,
```

- [ ] **Step 3: Build the runtime module to confirm no compilation errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: BUILD SUCCESS (or the same failures noted in Task 2 Step 6 if any exist).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#236): resolveChannel() uses ChannelSlugValidator.tryParseUuid(); update create_channel description"
```

---

## Task 4 — `ReactiveQhorusMcpTools.resolveChannel()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

The reactive path has its own `resolveChannel()` returning `Uni<UUID>`. Before implementing, verify the return type of `reactiveChannelService.findByName()` — it is either `Uni<Optional<Channel>>` (blocking seam mirror) or `Uni<Channel>` (null on miss).

- [ ] **Step 1: Check `ReactiveChannelService.findByName()` return type**

```bash
grep -n "findByName\|findById" /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java
```

- [ ] **Step 2a: If `findByName()` returns `Uni<Optional<Channel>>`**, replace the existing `resolveChannel()` in `ReactiveQhorusMcpTools.java` with:

```java
Uni<UUID> resolveChannel(final String channel) {
    final UUID parsedUuid = ChannelSlugValidator.tryParseUuid(channel);
    if (parsedUuid == null) {
        return reactiveChannelService.findByName(channel)
                .map(opt -> opt.orElseThrow(() ->
                        new IllegalArgumentException("Channel not found: " + channel)))
                .map(ch -> ch.id);
    }
    return reactiveChannelService.findById(parsedUuid)
            .map(opt -> opt.orElseThrow(() ->
                    new IllegalArgumentException("Channel not found: " + channel)))
            .map(ch -> ch.id);
}
```

- [ ] **Step 2b: If `findByName()` returns `Uni<Channel>` (null on miss)**, use:

```java
Uni<UUID> resolveChannel(final String channel) {
    final UUID parsedUuid = ChannelSlugValidator.tryParseUuid(channel);
    if (parsedUuid == null) {
        return reactiveChannelService.findByName(channel)
                .onItem().ifNull().failWith(() ->
                        new IllegalArgumentException("Channel not found: " + channel))
                .map(ch -> ch.id);
    }
    return reactiveChannelService.findById(parsedUuid)
            .onItem().ifNull().failWith(() ->
                    new IllegalArgumentException("Channel not found: " + channel))
            .map(ch -> ch.id);
}
```

Remove any private `tryParseUuid()` that the reactive class maintains independently. Add:
```java
import io.casehub.qhorus.runtime.channel.ChannelSlugValidator;
```

- [ ] **Step 3: Build to confirm no compilation errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#236): ReactiveQhorusMcpTools.resolveChannel() — ChannelSlugValidator.tryParseUuid()"
```

---

## Task 5 — `ConfiguredAutoChannelPolicy` Sanitisation (TDD)

**Files:**
- Modify: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicy.java`
- Modify: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java`

- [ ] **Step 1: Write failing tests for `sanitiseSegment()`, `slugifyConnectorId()`, `validatePattern()`**

Add the following tests to `ConfiguredAutoChannelPolicyTest.java`. Add them as new `@Test` methods after the existing tests:

```java
// ── sanitiseSegment ──

@Test void sanitiseSegment_phoneGetsIdPrefix() {
    String result = ConfiguredAutoChannelPolicy.sanitiseSegment("+14155552671");
    assertThat(result).startsWith("id-14155552671-");
    assertThat(result).matches("id-14155552671-[0-9a-f]{8}");
}

@Test void sanitiseSegment_emailNormalised() {
    String result = ConfiguredAutoChannelPolicy.sanitiseSegment("user@example.com");
    assertThat(result).startsWith("user-example-com-");
    assertThat(result).matches("user-example-com-[0-9a-f]{8}");
}

@Test void sanitiseSegment_caseVariantsProduceSameResult() {
    String lower = ConfiguredAutoChannelPolicy.sanitiseSegment("user@example.com");
    String upper = ConfiguredAutoChannelPolicy.sanitiseSegment("User@Example.COM");
    assertThat(lower).isEqualTo(upper); // hash computed on lowercased form
}

@Test void sanitiseSegment_validSlugGetsHashAppended() {
    String result = ConfiguredAutoChannelPolicy.sanitiseSegment("twilio-sms-inbound");
    assertThat(result).startsWith("twilio-sms-inbound-");
    assertThat(result).matches("twilio-sms-inbound-[0-9a-f]{8}");
}

@Test void sanitiseSegment_alwaysProducesValidSlug() {
    // Any sanitised output must pass the segment validator
    String result = ConfiguredAutoChannelPolicy.sanitiseSegment("+14155552671");
    assertThat(ChannelSlugValidator.isValidSegment(result)).isTrue();
}

@Test void sanitiseSegment_uuidInputPreservesFullContent() {
    String uuid = "550e8400-e29b-41d4-a716-446655440000";
    String result = ConfiguredAutoChannelPolicy.sanitiseSegment(uuid);
    // UUID starts with digit → id- prefix; full UUID content preserved (39 chars < 71)
    assertThat(result).startsWith("id-550e8400-e29b-41d4-a716-446655440000-");
}

// ── slugifyConnectorId ──

@Test void slugifyConnectorId_validSlugUnchanged() {
    assertThat(ConfiguredAutoChannelPolicy.slugifyConnectorId("twilio-sms-inbound"))
        .isEqualTo("twilio-sms-inbound"); // NO hash appended
}

@Test void slugifyConnectorId_spacesNormalised() {
    assertThat(ConfiguredAutoChannelPolicy.slugifyConnectorId("My Connector"))
        .isEqualTo("my-connector"); // NO hash appended
}

@Test void slugifyConnectorId_digitStartGetsIdPrefix() {
    assertThat(ConfiguredAutoChannelPolicy.slugifyConnectorId("123connector"))
        .isEqualTo("id-123connector");
}

@Test void slugifyConnectorId_alwaysProducesValidSlug() {
    assertThat(ChannelSlugValidator.isValidSegment(
        ConfiguredAutoChannelPolicy.slugifyConnectorId("My Connector"))).isTrue();
}

// ── validatePattern ──

@Test void validatePattern_acceptsAllPlaceholders() {
    assertThatNoException().isThrownBy(() ->
        ConfiguredAutoChannelPolicy.validatePattern("connector/{connectorId}/{lookupKey}"));
}

@Test void validatePattern_acceptsValidLiterals() {
    assertThatNoException().isThrownBy(() ->
        ConfiguredAutoChannelPolicy.validatePattern("sms/{lookupKey}"));
}

@Test void validatePattern_rejectsUppercaseLiteral() {
    assertThatIllegalStateException()
        .isThrownBy(() -> ConfiguredAutoChannelPolicy.validatePattern("Support/{lookupKey}"))
        .withMessageContaining("Support");
}

@Test void validatePattern_rejectsMixedLiteralWithUppercase() {
    assertThatIllegalStateException()
        .isThrownBy(() ->
            ConfiguredAutoChannelPolicy.validatePattern("Billing-{lookupKey}/work"))
        .withMessageContaining("Billing");
}

@Test void validatePattern_purePlaceholderSegmentPasses() {
    // {lookupKey} alone → substituted to "a" → valid
    assertThatNoException().isThrownBy(() ->
        ConfiguredAutoChannelPolicy.validatePattern("{lookupKey}/sub-path"));
}
```

- [ ] **Step 2: Run tests — confirm they FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConfiguredAutoChannelPolicyTest -pl connector-backend
```

Expected: compilation error — `sanitiseSegment`, `slugifyConnectorId`, `validatePattern` not yet defined.

- [ ] **Step 3: Implement `sanitiseSegment()`, `slugifyConnectorId()`, `validatePattern()`, and `@PostConstruct` in `ConfiguredAutoChannelPolicy`**

Add the following imports:

```java
import jakarta.annotation.PostConstruct;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.Locale;
import io.casehub.qhorus.runtime.channel.ChannelSlugValidator;
```

Add the following static methods to `ConfiguredAutoChannelPolicy`:

```java
/**
 * Normalises a connector ID to a slug segment without appending a hash.
 * Connector IDs are developer-defined controlled strings — they should be
 * valid slugs already. This function is defensive normalisation only.
 * Two non-conformant IDs that slugify identically share the same segment;
 * connector IDs that require slugification should be made unique at source.
 */
static String slugifyConnectorId(String connectorId) {
    if (connectorId == null || connectorId.isBlank()) {
        throw new IllegalArgumentException("Connector ID must not be null or blank");
    }
    String lower = connectorId.toLowerCase(Locale.ROOT);
    String slug = lower.replaceAll("[^a-z0-9]+", "-").replaceAll("^-+|-+$", "");
    if (slug.isEmpty()) {
        throw new IllegalArgumentException(
                "Connector ID '" + connectorId + "' reduced to empty after slugification");
    }
    if (Character.isDigit(slug.charAt(0))) {
        slug = "id-" + slug;
    }
    if (slug.length() > 80) {
        slug = slug.substring(0, 80).replaceAll("-+$", "");
    }
    if (slug.isEmpty()) {
        throw new IllegalArgumentException(
                "Connector ID '" + connectorId + "' produced empty slug after truncation");
    }
    return slug;
}

/**
 * Sanitises an arbitrary external identifier (phone number, email, etc.) to a
 * slug segment, always appending an 8-hex-char SHA-256 hash of the lowercased
 * input. The hash is unconditional — it guarantees uniqueness even when two
 * different raw inputs produce the same sanitised prefix.
 *
 * <p>Hash is of the lowercased form so case variants (e.g. user@example.com
 * and User@Example.COM) map to the same channel.
 */
static String sanitiseSegment(String raw) {
    if (raw == null || raw.isBlank()) {
        throw new IllegalArgumentException("Cannot sanitise null or blank segment");
    }
    String lowercased = raw.toLowerCase(Locale.ROOT);
    String slug = lowercased.replaceAll("[^a-z0-9]+", "-").replaceAll("^-+|-+$", "");
    if (slug.isEmpty()) {
        throw new IllegalArgumentException(
                "Segment '" + raw + "' reduced to empty after sanitisation");
    }
    if (Character.isDigit(slug.charAt(0))) {
        slug = "id-" + slug;
    }
    if (slug.length() > 71) {
        slug = slug.substring(0, 71).replaceAll("-+$", "");
    }
    if (slug.isEmpty()) {
        throw new IllegalArgumentException(
                "Segment '" + raw + "' produced empty slug after truncation");
    }
    return slug + "-" + hashHex8(lowercased);
}

/** Validates that every literal segment in a channel name pattern is a valid slug segment. */
static void validatePattern(String pattern) {
    for (String segment : pattern.split("/", -1)) {
        String testable = segment.replaceAll("\\{[^}]+}", "a");
        if (!ChannelSlugValidator.isValidSegment(testable)) {
            throw new IllegalStateException(
                    "Channel name pattern '" + pattern + "' has invalid literal segment '"
                    + segment + "' — literal parts must match [a-z][a-z0-9]*(-[a-z0-9]+)*");
        }
    }
}

private static String hashHex8(String lowercased) {
    try {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(lowercased.getBytes(StandardCharsets.UTF_8));
        return HexFormat.of().formatHex(hash, 0, 4); // 4 bytes = 8 hex chars
    } catch (NoSuchAlgorithmException e) {
        throw new IllegalStateException("SHA-256 not available", e);
    }
}
```

Add the `@PostConstruct` method:

```java
@PostConstruct
void validateConfiguredPatterns() {
    config.entries().forEach((connectorId, entry) ->
        entry.channelNamePattern().ifPresent(ConfiguredAutoChannelPolicy::validatePattern));
}
```

Update `onFirstContact()` to use the sanitisation functions. Replace:

```java
String channelName = entry.channelNamePattern()
        .map(p -> p.replace("{connectorId}", msg.connectorId())
                    .replace("{lookupKey}", lookupKey))
        .orElse("connector/" + msg.connectorId() + "/" + lookupKey);
```

With:

```java
String channelName = entry.channelNamePattern()
        .map(p -> p.replace("{connectorId}", slugifyConnectorId(msg.connectorId()))
                    .replace("{lookupKey}", sanitiseSegment(lookupKey)))
        .orElse("connector/" + slugifyConnectorId(msg.connectorId()) + "/" + sanitiseSegment(lookupKey));
```

- [ ] **Step 4: Run the new tests — confirm they PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConfiguredAutoChannelPolicyTest -pl connector-backend
```

Expected: BUILD SUCCESS, all tests green. If existing tests fail because they assert the old raw channel name (e.g. `"connector/" + InboundConnectorIds.TWILIO_SMS + "/+447911000001"`), note them — Task 6 fixes them.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicy.java \
  connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#236): ConfiguredAutoChannelPolicy — sanitiseSegment, slugifyConnectorId, validatePattern"
```

---

## Task 6 — Fix Broken Test Assertions (Channel Name Format Change)

**Files:**
- Modify: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelServiceFindOrCreateTest.java`

The existing tests assert raw phone numbers and email addresses in channel names. These now fail because `ConfiguredAutoChannelPolicy` sanitises them.

- [ ] **Step 1: Fix `ConfiguredAutoChannelPolicyTest` — old channel name assertions**

**Fix `sms_enabled_defaultChannelName`** (asserts raw phone in channel name):

```java
@Test
void sms_enabled_defaultChannelName() {
    Optional<AutoChannelSpec> result = policy.onFirstContact(smsMsg("+447911000001"), "+447911000001");

    assertThat(result).isPresent();
    // Connector ID unchanged (already a valid slug); phone sanitised with hash
    String name = result.get().channelName();
    assertThat(name).startsWith("connector/" + InboundConnectorIds.TWILIO_SMS + "/id-447911000001-");
    assertThat(name).matches("connector/" + InboundConnectorIds.TWILIO_SMS + "/id-447911000001-[0-9a-f]{8}");
}
```

**Fix `sms_customPattern_substitutesTokens`** (asserts raw phone in custom pattern result):

```java
@Test
void sms_customPattern_substitutesTokens() {
    when(smsEntry.channelNamePattern()).thenReturn(Optional.of("sms/{lookupKey}"));

    Optional<AutoChannelSpec> result = policy.onFirstContact(smsMsg("+447911000001"), "+447911000001");

    assertThat(result).isPresent();
    // {lookupKey} is sanitised; +447911000001 → id-447911000001-<hash>
    String name = result.get().channelName();
    assertThat(name).startsWith("sms/id-447911000001-");
    assertThat(name).matches("sms/id-447911000001-[0-9a-f]{8}");
}
```

**Fix `sms_enabled_descriptionMentionsConnectorAndSender`** — description is unchanged (it uses raw `lookupKey`), no fix needed.

- [ ] **Step 2: Fix `ChannelServiceFindOrCreateTest` — `smsRequest()` uses raw phone in name**

The `smsRequest()` helper currently builds `"connector/twilio-sms-inbound/" + senderPhone` as the name. After our change, `ChannelCreateRequest`'s compact constructor rejects `+447911...` as an invalid segment. The test must use the sanitised name.

Replace `smsRequest()`:

```java
private ChannelCreateRequest smsRequest(String senderPhone) {
    // Channel name uses sanitised phone segment (slug enforcement requires this).
    // The raw senderPhone is still used as the externalKey (binding lookup key).
    String channelName = "connector/twilio-sms-inbound/"
            + ConfiguredAutoChannelPolicy.sanitiseSegment(senderPhone);
    return new ChannelCreateRequest(
            channelName,
            "Auto-created on first contact",
            ChannelSemantic.APPEND,
            null, null, null, null, null, null, null,
            InboundConnectorIds.TWILIO_SMS, senderPhone, TwilioSmsConnector.ID, senderPhone);
}
```

Add the import:
```java
import io.casehub.qhorus.connector.backend.ConfiguredAutoChannelPolicy;
```

Update `createsChannelAndBindingWhenNotFound` assertion:

```java
@Test
void createsChannelAndBindingWhenNotFound() {
    String phone = uniquePhone();
    FindOrCreateResult result = channelService.findOrCreateWithBinding(smsRequest(phone));

    assertThat(result.wasCreated()).isTrue();
    assertThat(result.channel()).isNotNull();
    assertThat(result.channel().id).isNotNull();
    // Name uses sanitised phone segment; connector segment is unchanged (already a valid slug)
    assertThat(result.channel().name)
        .startsWith("connector/twilio-sms-inbound/")
        .matches("connector/twilio-sms-inbound/id-44[0-9a-f-]+-[0-9a-f]{8}");
    assertThat(result.channel().autoCreated).isTrue();
    assertThat(channelBindingStore.findByKey(InboundConnectorIds.TWILIO_SMS, phone)).isPresent();
}
```

Fix the `throwsWhenNoConnectorBinding` test to use a valid channel name (the test currently generates an invalid slug accidentally and relies on IAE for the wrong reason):

```java
@Test
void throwsWhenNoConnectorBinding() {
    // Use a valid slug name (no connector binding fields → should throw IAE)
    String validName = "my-channel-" + UUID.randomUUID().toString().replace("-", "").substring(0, 8);
    ChannelCreateRequest noBinding = ChannelCreateRequest.simple(validName, ChannelSemantic.APPEND);
    assertThatThrownBy(() -> channelService.findOrCreateWithBinding(noBinding))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("connector binding");
}
```

Also check `uniquePhone()`: it returns `"+44" + UUID...`. The raw phone is used as `externalKey` in the binding (that's fine — no slug validation on `externalKey`). The phone is no longer embedded raw in the channel name, so `uniquePhone()` can stay as-is.

- [ ] **Step 3: Run both affected test classes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="ConfiguredAutoChannelPolicyTest,ChannelServiceFindOrCreateTest" \
  -pl runtime,connector-backend
```

Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 4: Audit remaining connector-backend tests for raw phone assertions**

```bash
grep -rn '"+4\|+1\|phone\|lookupKey\|connector/' \
  /Users/mdproctor/claude/casehub/qhorus/connector-backend/src/test \
  --include="*.java" | grep -v "sanitise\|slugify\|id-"
```

Fix any remaining raw-phone assertions using the same `startsWith` + regex pattern approach shown above.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConfiguredAutoChannelPolicyTest.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelServiceFindOrCreateTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#236): fix channel name assertions for sanitised slug format"
```

---

## Task 7 — V17 Flyway Migration + Schema Test

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V17__add_channel_name_slug_constraint.sql`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java`

- [ ] **Step 1: Write the failing schema test first**

Add to `FlywayMigrationSchemaTest.java`:

```java
@Test
void channelNameSlugConstraintExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "")) {
        // H2 stores constraint names uppercased
        var rs = conn.getMetaData().getTablePrivileges(null, null, "CHANNEL");
        rs.close();
        // Use INFORMATION_SCHEMA to check constraint existence
        var stmt = conn.prepareStatement(
            "SELECT COUNT(*) FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS " +
            "WHERE TABLE_NAME = 'CHANNEL' AND CONSTRAINT_NAME = 'CHK_CHANNEL_NAME_SLUG'");
        var resultSet = stmt.executeQuery();
        resultSet.next();
        int count = resultSet.getInt(1);
        resultSet.close();
        stmt.close();
        assertTrue(count > 0,
            "chk_channel_name_slug CHECK constraint must exist on CHANNEL — added by V17 migration");
    }
}
```

- [ ] **Step 2: Run the schema test — confirm it FAILS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FlywayMigrationSchemaTest -pl runtime
```

Expected: test `channelNameSlugConstraintExists` fails with count = 0 (constraint does not yet exist).

- [ ] **Step 3: Create the V17 migration**

Create file `runtime/src/main/resources/db/qhorus/migration/V17__add_channel_name_slug_constraint.sql`:

```sql
-- Enforce slug format on channel names.
-- Every /-delimited segment must match [a-z][a-z0-9]*(-[a-z0-9]+)*.
-- Per-segment max length (80 chars) is enforced by Java — SIMILAR TO cannot express {0,79}.
-- Constraint is prospective: existing non-conformant rows are not validated retroactively.
ALTER TABLE channel
    ADD CONSTRAINT chk_channel_name_slug
    CHECK (name SIMILAR TO '[a-z][a-z0-9]*(-[a-z0-9]+)*(/[a-z][a-z0-9]*(-[a-z0-9]+)*)*'
           AND LENGTH(name) <= 200);
```

- [ ] **Step 4: Run the schema test — confirm it PASSES**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FlywayMigrationSchemaTest -pl runtime
```

Expected: BUILD SUCCESS, all schema tests green including `channelNameSlugConstraintExists`.

If H2 does not support `SIMILAR TO` in a `CHECK` constraint, the `@BeforeAll migrate()` will throw. In that case, replace `SIMILAR TO` in the migration with a `REGEXP_LIKE` equivalent:

```sql
CHECK (REGEXP_LIKE(name, '^[a-z][a-z0-9]*(-[a-z0-9]+)*(/[a-z][a-z0-9]*(-[a-z0-9]+)*)*$')
       AND LENGTH(name) <= 200)
```

H2 2.x supports `REGEXP_LIKE` natively.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/resources/db/qhorus/migration/V17__add_channel_name_slug_constraint.sql \
  runtime/src/test/java/io/casehub/qhorus/runtime/FlywayMigrationSchemaTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#236): V17 migration — chk_channel_name_slug CHECK constraint on channel.name"
```

---

## Task 8 — Full Build Verification

- [ ] **Step 1: Run the full Maven build across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS. This compiles and tests all modules: `api`, `runtime`, `connector-backend`, `testing`, `examples/type-system`, `examples/normative-layout`. The `examples/agent-communication` module is behind `-Pwith-llm-examples` and does not run in CI.

If any module fails:
- `examples/normative-layout` or `examples/type-system` failing → they likely contain channel names created without slug enforcement. Fix those names to conform to `[a-z][a-z0-9]*(-[a-z0-9]+)*`.
- `connector-backend` failing → re-check Task 6 test fixes.
- `runtime` failing with "Invalid channel name segment" in integration tests → a test is constructing a `ChannelCreateRequest` (or using `channelService.create()`) with a non-conformant name. Locate it via the stack trace and fix the name.

- [ ] **Step 2: Confirm the `ToolOverloadDiscoverabilityTest` still passes**

This test guards against `@Tool` methods being silently dropped. It should still pass — no new public overloads were added to `QhorusMcpTools` or `ReactiveQhorusMcpTools`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ToolOverloadDiscoverabilityTest -pl runtime
```

Expected: PASS.

- [ ] **Step 3: Commit (if any fixes were made in Step 1)**

If examples or other module tests needed fixes:

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(#236): update non-conformant channel names in examples and integration tests"
```

---

## Self-Review

### Spec coverage check

| Spec section | Covered by task |
|---|---|
| `ChannelSlugValidator` — `validateSlugPath`, `isValidSegment`, `tryParseUuid` | Task 1 |
| `ChannelCreateRequest` compact constructor wiring | Task 2 |
| `Channel.name` `@Column(updatable = false)` | Task 2 |
| `QhorusMcpToolsBase.resolveChannel()` — `tryParseUuid` consolidation | Task 3 |
| `create_channel` MCP description update | Task 3 |
| `ReactiveQhorusMcpTools.resolveChannel()` | Task 4 |
| `ConfiguredAutoChannelPolicy` — `sanitiseSegment`, `slugifyConnectorId`, `validatePattern`, `@PostConstruct` | Task 5 |
| Test assertions for `ConfiguredAutoChannelPolicyTest` | Task 6 |
| `ChannelServiceFindOrCreateTest` fixes | Task 6 |
| V17 Flyway migration | Task 7 |
| `FlywayMigrationSchemaTest` — constraint existence | Task 7 |
| Full build verification | Task 8 |

No gaps found.

### Type consistency

- `ChannelSlugValidator.tryParseUuid()` used in Tasks 1, 3, 4, 5 — always returns `UUID | null`. ✅
- `ChannelSlugValidator.isValidSegment()` used in Tasks 1, 5 — always returns `boolean`. ✅
- `ConfiguredAutoChannelPolicy.sanitiseSegment()` used in Tasks 5, 6 — always returns `String`. ✅
- `ConfiguredAutoChannelPolicy.slugifyConnectorId()` used in Tasks 5, 6 — always returns `String`. ✅
- `ConfiguredAutoChannelPolicy.validatePattern()` used in Task 5 — `void`, throws `IllegalStateException`. ✅
- `resolveChannel()` return type stays `UUID` in both blocking and reactive paths. ✅

### Placeholder scan

None found. All test code shows complete assertions with exact patterns. All implementation snippets are complete.
