---
title: "Going Where No GC Has Gone Before!"
date: 2026-02-19
layout: post
---

Garbage Collectors (GCs) are a critical component of a modern application stack. Long pauses, large memory consumptions, and high CPU usage can unexpectedly occur with certain workloads or series of events. These behaviors can make systems unresponsive, make it impossible to run them in resource-constrained environments, and are often very difficult to debug and fix -- as their appearance may be intermittent and the issues are often intrinsic to GC systems themselves. In fact [recent work](https://dl.acm.org/doi/10.1145/3720430) has shown what, for most languages, these issues are unavoidable and, regardless of the GC design or implementation, there will always be workloads that cause them to occur!

Intriguingly, the pathological and scenarios that are needed to cause these issues depend on a specific set of language features -- mutability and cyclic data structures. In previous posts we have discussed how these features impact [mechanized analysis](https://bosquelanguage.github.io/2025/09/18/small-models.html) and how they are common sources of [developer (human and AI!) errors](https://dl.acm.org/doi/10.1145/3622758.3622895). But, as it turns out, they are also the source of GC pathologies and, in fact, by designing a language (Bosque) that is easy to analyze we have also created a language that provides strong invariants that 1) the GC can rely on to optimize its behavior and 2) that, when combined with some careful implementation, allow us to prove that our new GC design for Bosque is free of all pathological behaviors! 

## Critical Language Features and Invariants

## Theoretical Guarantees

## Pando GC Design

## Experimental Results


####

<!-- TODO: very cool paper title == Going Where No GC Has Gone Before -- A Pathology-Free GC for Bosque -->

####


