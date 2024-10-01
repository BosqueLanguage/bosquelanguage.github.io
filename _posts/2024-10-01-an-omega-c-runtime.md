---
title: "An Effectively Ω(c) Language and Runtime"
date: 2024-10-01
layout: post
---

Taking a quick break from some (fun) bosque language features and their implications for correctness and reliability to think about performance and efficiency a bit. 

One of the key goals of the Bosque project is to provide a language and runtime that is both _reliable_ and _performant_. Often performance is thought of in terms of the min, or average, time/memory taken by the program but in many cases the _worst case_ (or tail-latency) performance is mire important. As long as the application responds in under, say 100ms, and is fast enough for a user to barely notice that is fine but if it takes noticeable amount of time longer, even a small percentage of the time, then users may start abandoning the task -- and leaving our application!

With this in mind Bosque is exploring a runtime design that is focused on minimizing variability (tail-latency) and providing a predictable and consistent performance profile even if it reduces max performance slightly. More concretely the design is focused on:
1) Maintaining an effectively constant time to execute operations -- eliminating common quadratic or exponential slow paths.
2) Using _only_ a constant fixed memory overhead beyond the application footprint -- no need to heavily over-provision memory for GC.
3) Limiting the garbage-collector to a (small) fixed pause for collection and guaranteeing a constant amount of work per allocation (regardless of GC or application behavior).

These goals, how features of the Bosque language enable them, and a roadmap for implementing the desired runtime are the focus of a short paper ([arxiv version here](https://arxiv.org/abs/2409.20494)) that is going to be presented at the upcoming [VMIL workshop](https://2024.splashcon.org/home/vmil-2024#program) in October! 

The paper outlines three key ideas for achieving these goals. 

The first is in the design of the standard libraries. The approach here is primarily to replace commonly used amortized cost data structures and algorithms, like re-sizing vectors or quick-sort, with slightly less _big-O_ performant but efficient _tight-Ω_ versions. Examples include eliminating worst-case _O(n)_ resizing vector implementations with tight _Ω(log n)_ [RRB structures](https://dl.acm.org/doi/10.1145/2858949.2784739) and replacing [exponentially backtracking](https://dl.acm.org/doi/pdf/10.1145/3236024.3236027) PCRE regex libraries with polynomial with variations like [BREX](https://github.com/BosqueLanguage/BREX).

The next component is to leverage the design of the Bosque language to maximize the ability of the compiler to statically resolve code. This includes using the type system and closed world assumptions to eliminate dynamic dispatch, using the immutability of values to statically evaluate and simplify abstractions, and using the structured nature of the language to enable aggressive tree-shaking, inlining, and specialization.

Finally, the garbage collector is a critical component of a runtime system and is often a [major source of variance](https://www.steveblackburn.org/pubs/papers/lbo-ispass-2022.pdf) in application performance behavior. Massive work has gone into various GC algorithms to reduce their, costs, pause times, and memory overheads. A particular focus in recent years has been on reducing pause times. However, in a language with mutation, cycles, and semantically observable object identity, there are limitations to what can be achieved – specifcally tradeoffs between latency and throughput and the increasing complexity of the memory management implementation.

Bosque presents a unique opportunity to re-think GC design and implementation. The language is designed with fully immutable values/objects, no cycles, and no way to observe identity (directly via addresses or indirectly via any language semantics). This allows for novel and aggressive design choices to be made in the GC and allocator implementation.

Using the basic design of a generational GC with a copying young-space and a [reference-counting old-space](https://www.cs.utexas.edu/~mckinley/papers/urc-oopsla-2003.pdf) these goals are achievable. By construction, and the fact that Bosque prevents cycles, this algorithm satisfies our first goal of a fixed and constant memory overhead. The size of the young generation defines the constant for the fixed overhead beyond the application live memory. As Bosque values are immutable, once an object is copied to the old generation it will never have any pointer updates. This eliminates the need for any write barriers as well as any work to re-process reference counts on changes.

In addition to having a low total-cost per allocation, the design of the Bosque ensures that the collector pauses are bounded too – that is we are not trading off throughput for large and/or unpredictable collector latency or risking the GC failing behind and stalling the application. In particular, the direct implementation is to perform a stop-the-world collection (STW) of the young generation and a constant amount of ref-count work in the old generation. This bounds the pause to the time to process the young generation + a constant time for the old generation while always reclaiming memory as fast as the application thread uses/recycles it!

A prototype collector design, using the immutability and cycle-freedom of Bosque, does not require any write/read barriers or remembered sets. It even works nicely with conservative collection, enabling the compiler to skip root maps, and easily support pointers into the stack and interior value pointers! This is a major simplification of the GC design and, of course, allows for more aggressive runtime design and compiler optimizations

I think this is an exciting direction for Bosque, that it opens up new possibilities for the design of runtimes/compilers, and leads to applications that are more stable, more predictable, and more performant than ever before! Looking forward for fun conversations at the workshop and, as always, feedback on X (@BosqueLanguage) or in the [Bosque repository](https://github.com/BosqueLanguage/BosqueCore) are welcome!