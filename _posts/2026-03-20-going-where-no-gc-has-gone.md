---
title: "Going Where No GC Has Gone Before!"
date: 2026-03-20
layout: post
---

Garbage Collectors (GCs) are a critical component of a modern application stack. Long pauses, large memory consumptions, and high CPU usage can unexpectedly occur with certain workloads or series of events. These behaviors can make systems unresponsive, make it impossible to run them in resource-constrained environments, and are often very difficult to debug and fix -- as their appearance may be intermittent and the issues are often intrinsic to GC systems themselves. In fact [recent work](https://dl.acm.org/doi/10.1145/3720430) has shown what, for most languages, these issues are unavoidable and, regardless of the GC design or implementation, there will always be workloads that cause them to occur!

Intriguingly, the pathological and scenarios that are needed to cause these issues depend on a specific set of language features -- mutability and cyclic data structures. In previous posts we have discussed how these features impact [mechanized analysis](https://bosquelanguage.github.io/2025/09/18/small-models.html) and how they are common sources of [developer (human and AI!) errors](https://dl.acm.org/doi/10.1145/3622758.3622895). But, as it turns out, they are also the source of GC pathologies and, in fact, by designing a language (Bosque) that is easy to analyze we have also created a language that provides strong invariants that 1) the GC can rely on to optimize its behavior and 2) that, when combined with some careful implementation, allow us to prove that our new GC design for Bosque is free of all pathological behaviors! 

## Theoretical Guarantees
The first issue is to more precisely outline what behaviors we want to guarantee for the collector (and memory management system more generally).

1) **Bounded Collector Pauses:** The collector only requires the application to pause for a (small) bounded period that is proportional to the size of the nursery.
2) **Starvation Freedom:** The collector can never be outrun by the application allocation rate and will always satisfy allocation requests (until true exhaustion).
3) **Fixed Work Per Allocation:** The work done by the allocator and GC for each allocation is constant – regardless of object lifetimes or application behavior.
4) **Application Code Independence:** The application code does not pay any cost, e.g. read/write barriers, remembered sets, etc., for the GC implementation.
5) **Constant Memory Overhead:** The reserve memory required by the allocator/collector is bounded by a (small) constant overhead

The first two properties are the most critical and are the ones that are often violated by pathological workloads. They are foundational to user perceived performance -- long pauses to responses are annoying and application crashes due to out-of-memory conditions are obviously undesirable. As pointed out, these properties are also previously impossible for GC systems to simultaneously satisfy.

The next two properties are also important and are often violated in GC designs. The need to repeatedly visit objects (e.g. marked dirty and re-scanned after modification) is a parasitic cost that can lead to high CPU utilization and increased cache pressure (misses) that are difficult to predict and optimize for. Similarly, the cost for synchronizing the application and GC state (e.g. read/write barriers) is a smaller (5% direct) cost but, over the course of an application adds both the direct costs and indirect costs on every field write (read) operation. 

The final property is often overlooked, as memory is often considered cheap on modern servers, but given the over-provisioning of memory required by many GC designs (e.g. 3-4x the live data), this can be a significant cost. This overhead can stack-up when running multiple applications on the same machine and is a challenge when moving to edge/embedded environments where memory is more constrained -- and can require substantial work to port code to a more memory-efficient language (Rust/C++) or rework the application to target a specialized embedded target.

Our goal is to design a GC for Bosque that satisfies all of these properties.

## Key Language Features and Invariants

## Pando GC Design

## Experimental Results


####

<!-- TODO: very cool paper title == Going Where No GC Has Gone Before -- A Pathology-Free GC for Bosque -->

####


