# Design Journal — issue-328-conversation-model-enrichments

## §Session-1: Topic + Reactions (2026-07-10)

### Key Design Decisions

**Topic: Hybrid entity + denormalized string on Message.** Three approaches evaluated: (1) string-on-message only (Zulip's current impl — known regret, open since 2014), (2) Topic entity with FK (JOIN cost on hot polling path), (3) hybrid. Selected (3) — Topic entity for efficient listing and metadata + denormalized `message.topic` string for fast reads. No FK between message and topic.

**Ledger relationship: organizational, not normative.** Topic recorded on `MessageLedgerEntry.topic` at dispatch time (immutable). Message table stores current topic (mutable via rename). Rename emits a system EVENT in the ledger documenting the administrative action. Pattern follows Matrix event DAG / CQRS: ledger = write model (immutable), message table = read model (mutable).

**Topic naming: free-form text, not slug-restricted.** Deliberate divergence from channel slug convention. Topics are human-readable labels ("Authentication Investigation"), not machine identifiers. Case-insensitive matching, case-preserving storage. "general" reserved as default.

**Reactions: outside the normative layer.** Reactions bypass `MessageService.dispatch()` entirely — no enforcement gate, no ledger entry, no MessageObserver notification. Own CDI event (`ReactionChangedEvent`) for live UI notification. Correct boundary: reactions are lightweight acknowledgment without formal obligation weight.

**Deferred: moveTopic (#335), mergeTopics (#336).** Move changes `message.channelId` breaking `Commitment.channelId` references (normative integrity). Merge mixes correlation chains. Rename is safe (no normative field touched).

### Cross-Repo Integration

Filed integration test issues: engine#701 (choreography-over-topics), connectors#80 (chat UI), blocks#49 (end-to-end). Each repo tests its own contribution; blocks wires the full stack.

### Normative System Complementarity

Topics complement the normative layer without coupling to it: obligation context (ledger records original topic), topic resolution independent of obligation state (deliberate decoupling), task-based structure within normative channel layout (orthogonal and composable).
