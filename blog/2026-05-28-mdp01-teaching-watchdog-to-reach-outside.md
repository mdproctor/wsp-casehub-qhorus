---
layout: post
title: "Teaching the watchdog to reach outside"
date: 2026-05-28
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [cdi, sealed-types, watchdog, connectors]
---

The watchdog module has always had a gap: conditions fire, alerts go into a Qhorus channel, and nothing reaches a human unless an agent is watching. For pure agent-to-agent systems, that's fine. For anything with human oversight, it's a silent failure mode.

We closed it this week. `WatchdogAlertEvent` fires from `WatchdogEvaluationService` whenever a condition triggers. `ConnectorAlertBridge` observes it and routes to Slack, Teams, SMS, or email via an opt-in bridge module.

## The routing question

The hard part wasn't the CDI event. It was routing — how does the bridge know where to send the alert? `ConnectorMessage.destination` is per-message (a webhook URL or phone number), so the event payload alone can't answer it.

Three options: a global config list, a per-watchdog schema extension, or a `WatchdogAlertRouter` SPI with a config-backed `@DefaultBean`.

The schema option I dismissed quickly. Deployment config — webhook URLs, phone numbers — belongs in `application.properties`, not the `watchdog` table. It would also force a Flyway migration on every qhorus deployment, including ones that never touch the bridge. That's the wrong layer.

The config list is fine out of the box but becomes a problem the moment you have watchdogs with different criticalities. Every alert goes to every endpoint. In practice that's a policy wrapped in YAML, without the logic to express it.

I went with the SPI. `CommitmentAttestationPolicy` and `InstanceActorIdProvider` use the same pattern already. The `@DefaultBean` provides global config fan-out by default; sophisticated consumers in Claudony or devtown provide their own router for per-watchdog logic, without touching the bridge.

## The payload: sealed over open

`WatchdogAlertEvent` needs to carry condition-specific data — which barrier contributors are missing, how many approvals are pending, which agents are stale. The first draft used `Map<String, String>` with documented key contracts per condition type.

I asked a separate Claude to audit the spec before implementation. It flagged the map immediately, and correctly. The condition set is closed: five types. A sealed interface with five typed records is the right Java 21 answer. Misspelled keys don't compile. The `buildBody` switch in `ConnectorAlertBridge` is exhaustive — adding a sixth condition type without a corresponding record fails at compile time, not at 2am.

Six new types in `casehub-qhorus-api`. That's not a cost; that's the design working.

## fireAsync goes first

`fireAlert()` does two things: fires the CDI event and dispatches a STATUS message to the internal Qhorus channel. The order isn't arbitrary.

If the channel dispatch comes first and throws — rate-limited, paused, policy-rejected — the CDI event never fires. External delivery is silently killed by an internal failure. That's wrong. The two paths need to be independent.

`fireAsync()` goes first. The accepted trade-off: if the outer `@Transactional` rolls back after the event fires, the observer runs for a state change that never committed. False-positive alert. For watchdog alerting, that's the right call — a false-positive notification is better than a missed one. It's documented in the comment, not buried.

## Where the bridge lives

`casehub-qhorus-connectors` is an optional submodule in the qhorus repo, listed before `runtime` in the root `pom.xml`. The runtime module takes it as a test-scope dependency for the E2E test; that dependency requires the bridge to be installed before the runtime's tests compile. The qhorus runtime itself has no dependency on the bridge — connectors on the classpath activates it.

The pattern follows `casehub-engine-ledger` and `casehub-engine-work-adapter`. Bridge modules for cross-foundation wiring live in the event-source repo.
