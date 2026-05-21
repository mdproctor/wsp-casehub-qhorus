---
layout: post
title: "The obligor isn't who sent the command"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [commitment, speech-act, testing]
---

Small fix today, but one that required a small mental reset.

`delete_channel` already deletes messages before deleting the channel row — the `fk_message_channel` constraint made that necessary. What we hadn't done was the same for commitments. `fk_commitment_channel` also has no `ON DELETE CASCADE`, which means deleting a channel with open commitments fails at the DB level with a constraint violation.

The fix is one line in each of the two `delete_channel` implementations:

```java
commitmentStore.deleteAll(ch.id);
```

...placed before `channelService.delete()`. Then we implemented `deleteAll` on `CommitmentStore` and its in-memory and JPA implementations. Straightforward.

The less straightforward part was writing the test to confirm the fix was needed.

To verify that the fix actually does something, the test has to establish that a commitment exists before the deletion. The natural query:

```java
commitmentStore.findOpenByObligor("agent-a", channelId)
```

...returned empty. Even after sending a COMMAND from `"agent-a"` with a non-null correlationId. The assertion `assertFalse(list.isEmpty())` failed — list was empty when it shouldn't be.

The reason: in Qhorus's commitment model, a COMMAND creates a commitment where `sender → requester` and `target → obligor`. The obligor is the receiver of the obligation — the party who must fulfil it. The sender of the COMMAND is making the request, not taking it on. With no explicit target in the test, `obligor` is null.

`findOpenByObligor("agent-a", channelId)` was correct. "agent-a" wasn't obligated to anything.

The right query is `findByState(CommitmentState.OPEN, channelId)` — which asks whether the channel has any open commitments at all, without caring about roles.

The underlying semantics are from speech act theory: a COMMAND creates a deontic obligation on the receiver, not the sender. The API is faithful to the model. The test was asking the wrong question.
