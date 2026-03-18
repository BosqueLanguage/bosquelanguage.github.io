---
title: "Going Where No GC Has Gone Before!"
date: 2026-03-20
layout: post
---

Garbage Collectors (GCs) are a critical component of a modern application stack. Long pauses, large memory consumptions, and high CPU usage can unexpectedly occur with certain workloads or series of events. These behaviors can make systems unresponsive, make it impossible to run them in resource-constrained environments, and are often very difficult to debug and fix -- as their appearance may be intermittent and the issues are often intrinsic to GC systems themselves. In fact [recent work](https://dl.acm.org/doi/10.1145/3720430) has shown what, for most languages, these issues are unavoidable and, regardless of the GC design or implementation, there will always be workloads that cause them to occur!

Intriguingly, the pathological and scenarios that are needed to cause these issues depend on a specific set of language features -- mutability and cyclic data structures. In previous posts we have discussed how these features impact [mechanized analysis](https://bosquelanguage.github.io/2025/09/18/small-models.html) and how they are common sources of [developer (human and AI!) errors](https://dl.acm.org/doi/10.1145/3622758.3622895). But, as it turns out, they are also the source of GC pathologies and, in fact, by designing a language (Bosque) that is easy to analyze we have also created a language that provides strong invariants that 1) the GC can rely on to optimize its behavior and 2) that, when combined with some careful implementation, allow us to prove that our [new GC design for Bosque](https://arxiv.org/abs/2509.13429) is free of all pathological behaviors! 

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

## Key Language Features and Invariants

Looking at the structure of a garbage collector, and the pathological scenarios that cause it to violate one (or more) of the above properties, we can identify two common features. The first is if, at some point, the GC must have global knowledge of the heap connectivity graph in order to make progress. The second is if the mutator can take actions, outside of direct allocation, that increase the amount of work that the GC must do (e.g. by modifying objects that the GC has already visited).

In the cases of generational collectors like Java's G1 (or reference counting collectors with cycles), the GC must have global knowledge of the heap connectivity graph to identify which objects are live. For these systems, it is clear that eventually, the collector will need to trace the entire heap and, as the heap size is arbitrarily large, the resulting tracing will induce an arbitrarily large pause -- violating property 1 in addition to 4 and 5. 

As an alternative we might consider an incremental collector like in Go or Java's ZGC. These systems can avoid long pauses by breaking up the tracing work into smaller increments and interleaving it with application execution. However, these collectors are still vulnerable to workloads where the application allocates new objects and invalidates reachability information (faster) then the GC can trace and reclaim memory. The most common cases are when objects are modified repeatedly, which marks them as dirty, and thus must be re-scanned by the collector. In this case the collector will either need to fail on allocation (prematurely starve -- violating property 2) or it will need to fall-back to a fully stop-the-world collector or degenerate collection (again violating property 1).

As noted in the introduction, these issues are not simply engineering challenges that may one day be solved but they are [intrinsic](https://dl.acm.org/doi/10.1145/3720430) to _any_ GC design for languages with mutability and cyclic data structures. In contrast, the Bosque language has two key features that allow us to design a GC that is free of these pathologies: 

- **Immutability:** The entity declaration in Bosqe creates a new composite datatype and these are always immutable. A key implication of this fact is that once an entity value is created then fields (and thus pointer targets) will never change.
- **Cycle Freedom:** In combination with the immutability of entities, and careful definition of constructor semantics, the Bosqe language ensures that all data structures are acyclic. This invariant allows us to utilize reference-counting techniques without concern for backup cycle-collection or other special case logic.

## Pando GC Design
The absence of the above pathologies creates the potential for building a fully pathology free GC. However, this theoretical possibility does not give us a specific design. Thus, our challenge is to find a GC design (and implementation) that can leverage these language features and invariants to satisfy the above properties.

Looking at properties of GC designs, tracing vs reference counting (RC), a key item to note is that tracing is very good at quickly allocating and reclaiming memory while RC systems are excellent at short-pauses and only touching objects when lifetime events occur (e.g. deallocations). Thus, a natural design is to combine these two approaches -- using tracing for the nursery and RC for the older generation. We are not the first to notice this, in fact this idea was proposed by two groups in 2003 [[1](https://dl.acm.org/doi/10.1145/949343.949336), [2](https://dl.acm.org/doi/10.5555/1145763.1145764)]. However, as they were working with Java, which has both mutation and cycles, their ability to leverage key features of this design were limited.

In contrast, Bosque allows us to aggressively simplify the design and implementation of this combined tracing/RC collector! 

In the Bosque collector design heap values can be in one of two logical regions, the nursery or the reference-counted old space. 

![Logical Heap Layout -- Young Generation and RC Old Space](https://bosquelanguage.github.io/assets/imgs/logicalheap.jpg)

In this design the roots can point to values in the nursery, the RC-heap, or other stack allocated values. However, while there may be pointers from the nursery to the RC-heap, pointer _are not_ allowed to go from the RC-heap into the nursery. This hierarchical relation ensures that objects in the nursery can be collected without concern for references from older objects which, by construction, cannot exist.

The first step in the collection algorithm is to trace and evacuate the young objects in the nursery. For each marked object we encounter in the tracing walk are two possibilities, the first is that the object is in the nursery and only referenced by other young objects, and the second is that the object is already in the RC-heap. 

In the first case we can evacuate the object values to a compacting page, performing pointer forwarding as necessary, and converting the object to a reference counted representation (the objects in the dashed box). As each object is promoted to the RC-old space all fields are 
(precisely) scanned and reference counts for children are incremented (shown with black plus).

![Logical Heap Layout -- After Young Generation Evacuation](https://bosquelanguage.github.io/assets/imgs/afterdecs.jpg)

Next the algorithm sweeps the now mostly (or entirely) evacuated nursery and rebuilds the free-lists for the next round of allocation -- allowing the collector to re-use both completely empty and semi filled pages. As RC objects are not moved the ability to re-use partially filled pages is critical to avoid fragmentation. The sweep reclaims the unmarked dead objects and the space cleared by the evacuation. In our example, there is only one object that remains on the page, the root referenced object, while all other slots are evacuated or reclaimed.

The final step in the collection algorithm is to process deferred reference count decrements and root reference status. Our implementation compares the root set from the previous collection with the root set identified in the current collection to determine which root references are no longer live. For each of these references the collector checks if the reference count has dropped to zero, in which case the object can be reclaimed. This reclamation is then a standard release walk of the object graph, decrementing reference counts and reclaiming objects as necessary.

![Logical Heap Layout -- After Reclaim](https://bosquelanguage.github.io/assets/imgs/afterreclaim.jpg)

The most notable feature of this algorithm is not what it has but what doesn't. Despite the use of a generational collector and reference-counting, there are no remembered sets, no write barriers, no backup cycle-collector, and no need for runtime support in the application code. As a result of these features, the cost of a collection cycle is independent of any application code behavior and can always be performed in a bounded time that is proportional to the size of the nursery -- and the additional memory overhead is also a constant factor of the nursery size.

## Experimental Results




