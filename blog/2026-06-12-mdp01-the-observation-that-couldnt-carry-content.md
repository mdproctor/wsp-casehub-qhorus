---
layout: post
title: "The Observation That Couldn't Carry Content"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [qhorus, connector-bridge, message-types, informatory]
---

The issue spec said to post an EVENT. That was the starting point, and it was wrong in two directions simultaneously.

The first direction: `MessageDispatch.Builder.build()` throws if you set content on an EVENT. The observe channel — the natural target — was configured EVENT-only. So the spec had us posting a content-free signal to a channel that rejects everything else. Technically legal. Also useless. An audit notification that carries no information about what was actually delivered serves nobody.

The second direction was more interesting. Challenging "use EVENT" forced the question: what makes something an observation in the first place? I'd been thinking about it as a message type property. Claude flagged the tension early — we can't use EVENT (content-free) and we can't use STATUS on an EVENT-only channel — and the only honest resolution was to go back to first principles.

The answer turned out to be simpler than the type taxonomy suggested. A message is informatory when it carries information without opening an expectation of reply. That's it. EVENT is informatory because nobody replies to tool-call telemetry. STATUS is also informatory when it's a standalone observation with no correlating COMMAND behind it. The observe channel in PLATFORM.md already says EVENT and STATUS are both valid — the EVENT-only example in the framework docs was just an overzealous restriction written for a pure telemetry use case and applied too broadly.

Once that was clear, the whole thing simplified. STATUS for content-bearing informatory messages. The earlier "STATUS is wrong because it's for obligation chains" objection dissolved: a STATUS with no correlationId and no inReplyTo is just a state report, not a commitment resolution. Perfectly valid. Perfectly informatory.

The implementation was straightforward from there — `@ApplicationScoped`, constructor injection, synchronous context capture before the `ManagedExecutor` fire-and-forget, `ConcurrentHashMap` keyed by tenancyId for the channel ID cache. The one thing the code review caught that I hadn't thought through: the destination field. Webhook URLs are credentials. Phone numbers are PII. The audit ledger is immutable. That combination has exactly one correct answer: exclude the destination entirely from content. The connectorId is enough for categorisation.

One other thing surfaced during work-end that had nothing to do with the bridge. The project main had two uncommitted files — `QhorusLedgerEntryRepository.java` and `ReactiveLedgerEntryJpaRepository.java` — with changes to a method signature that had been updated in the ledger library. `mvn test -pl connector-backend` had been passing the whole time because the installed runtime jar pre-dated the API change. Four call sites, mechanical update, clean compile once I ran `mvn clean install` from the root. The stale jar had been hiding the mismatch for at least two sessions. That one goes in the garden.

The larger question this opens: the informatory role concept probably has wider application than the observe channel. If the defining property is "no reply expected here," then the `allowedTypes` enforcement on channels is solving the wrong problem — it's trying to use message type as a proxy for intent, and intent doesn't actually reduce to type. #271 tracks the advisory-versus-enforced question.
