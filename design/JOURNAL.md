# Design Journal — issue-261-slack-channel-backend

### 2026-06-18 · §Component Structure

`casehub-qhorus-slack-channel` establishes the pattern for optional qhorus modules that ship JPA entities: entities live in a package outside `io.casehub.qhorus.runtime`, so consumers must explicitly register the package in `quarkus.hibernate-orm.qhorus.packages`. This is intentional — optional modules should not silently expand the PU scan scope; consumers opt in. The module activates by classpath presence, consistent with `connector-backend`.

`SlackChannelBackend` uses `SlackBotClient` directly (Tier 1.5 credential pattern: workspaceId in DB, token resolved via MicroProfile Config). This bypasses `ConnectorService` to enable thread-aware delivery — the generic connector path has no concept of Slack thread continuity. The composite thread cache (in-memory + DB-backed by `SlackThreadCache`) is required for restart survival: without DB persistence, a server restart breaks all in-flight commitment threads into new top-level Slack messages with no error.

The write-before-dispatch ordering in `onInboundMessage` (cache entry written before `receiveHumanMessage`) prevents a race where a fast agent response arrives before the correlation cache is populated. This is a design constraint, not an optimisation, and should not be simplified.
