---
layout: post
title: "Three Out of Four Were Fine"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [cdi, quarkus, spi, audit, defaultbean]
---

An automated audit flagged four `@ApplicationScoped` classes in qhorus for having "unnecessary CDI" — zero or one `@Inject` fields each. The heuristic is reasonable: if a class has no dependencies, why does it need CDI? Remove the annotation, construct it inline, and the codebase gets simpler.

Except the heuristic misses why Quarkus extension projects use CDI in the first place. Injection count is one reason. SPI displacement is another. And in an extension project, displacement is often the *primary* reason a class is CDI-managed.

Three of the four flagged classes turned out to be doing exactly this. `StoredMessageTypePolicy` implements `MessageTypePolicy` — consumers inject the interface and can override with `@Alternative @Priority`. `DefaultInboundNormaliser` is `@DefaultBean` implementing `InboundNormaliser` — the textbook displacement pattern, where the default exists specifically to be replaced. `QhorusEntityMapper` injects `ObjectMapper`, getting Quarkus's CDI-managed Jackson mapper with all the platform serialisation config. None of these are stateless utilities that happen to be CDI beans. They're CDI beans because the architecture requires it.

The fourth — `AllowedWritersPolicy` — was the genuine article. Zero injections, no interface, no `@DefaultBean`, no displacement contract. A pure validation function wearing an `@ApplicationScoped` annotation it didn't need. That one we removed, wiring it as a `final` field initialiser in `MessageService`.

The interesting question is why the audit got three wrong. The tool counted `@Inject` fields per class and flagged anything with zero or one. That works for application-layer beans where CDI is about dependency wiring. It fails for extension-layer beans where CDI is about architectural contracts — SPIs, default implementations, consumer overrides. The audit checked the plumbing and missed the blueprint.

The fix for future audits is a three-point check before removing `@ApplicationScoped`: does the class implement an interface that other beans inject? Is it `@DefaultBean`? Does it `@Inject` CDI-managed beans? Only when all three are negative is CDI genuinely unnecessary. Counting injections alone will systematically produce false positives in any project that uses CDI for SPI displacement.
