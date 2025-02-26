---
title: "Compactly Solvable Languages"
date: 2025-02-28
layout: post
---

One of the key goals of the Bosque project is to create a language, and development stack, where formal methods can be widely and effectively applied at scale. This goal is to be agnostic about the form of underlying reasoning, from simple rule based linting, through abstract-interpretation based tools, to full logical reasoning systems, all of them should be easier to build and more effective/practical in the Bosque ecosystem.

First-order logic based reasoning is one of the oldest, and theoretically most powerful approaches to formal analysis and verification (or validation) of software systems. In this context, building a first-order logic based framework for reasoning about Bosque code is a key component of the project. A practical limit to this type of system is the fundamental undecidability of the problem in full generality! It may be interesting in the future to explore a fully-verified software system story for Bosque, using proof techniques similar to [Dafny](https://dafny.org/) or [F*](https://fstar-lang.org/). However, for most scenarios and developers, an incomplete but simple and fully automated formalism is more useful. 

The most well-know and (sometimes) used in practice approaches to incomplete symbolic reasoning about program behavior are based on [symbolic model checking](https://en.wikipedia.org/wiki/Model_checking). Instead of proving that a given property holds for all possible executions of a program these techniques symbolically (or concretely) unroll and explore possible paths in the program until they find a violation of the property or exhaust the search resources -- [cbmc](https://www.cprover.org/cbmc/) and similar tools for Rust ([Kani](https://model-checking.github.io/kani/)) or Java ([JBMC](https://www.cprover.org/jbmc/)). A challenge in this approach is how to explore possible paths in programs -- particularly as in loops or recursive functions where the path is not syntactically bounded. In these cases the checker can either try to unroll the loop (expand the call) another time _or_ decide to exit and explore deeper into the code. In some cases only one option is possible but in general the both choices may be possible. 

As a result the formal guarantee that a model checker makes on the existence of an error is that it will find one if it exists _and_ it is reachable within a bounded number of unrollings (or exploration time). This is useful but, on some level, unsatisfying as there may be small/simple inputs that cause the program to enter a loop that is too deep to explore in the time budget and will be missed. 

This is an interesting issue and I am trying to formulate more precisely what/why this is a problem. So, let me share an experimental viewpoint on the topic. Conceptually, if we view the program input space as being a space that can be (partitioned into a finite cover)[https://www.cs.montana.edu/courses/se422/currentLectures/Ch4.pdf] then an exploration does not guarantee the coverage of any particular part of the input space. Instead each run covers some part of the execution trace partition but, as the unrollings are unbounded, the trace coverage will never be complete (in some cases the use of (interpolants)[https://rg1-teaching.mpi-inf.mpg.de/old-ag2/teachingpodelski/MCseminar/talk-ScottCotton.pdf] enable completion but this is not guaranteed). Conversely, if we believe the _input equivalence partition_ hypothesis then we can say that if we verify the absence of errors for a finite cover for each input partition then we can guarantee the absence of errors for all inputs -- and one particular input partition is widely hypothesized to be related to the vast majority of errors.

....


remember interpolants 

