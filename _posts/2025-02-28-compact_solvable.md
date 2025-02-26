---
title: "Compactly Solvable Languages"
date: 2025-02-28
layout: post
---

One of the key goals of the Bosque project is to create a language, and development stack, where formal methods can be widely and effectively applied at scale. This goal is to be agnostic about the form of underlying reasoning, from simple rule based linting, through abstract-interpretation based tools, to full logical reasoning systems, all of them should be easier to build and more effective/practical in the Bosque ecosystem.

First-order logic based reasoning is one of the oldest, and theoretically most powerful approaches to formal analysis and verification (or validation) of software systems. In this context, building a first-order logic based framework for reasoning about Bosque code is a key component of the project. A practical limit to this type of system is the fundamental undecidability of the problem in full generality! It may be interesting in the future to explore a fully-verified software system story for Bosque, using proof techniques similar to [dafny]() or [F*](). However, for most scenarios and developers, an incomplete but simple and fully automated formalism is more useful. 

The most well-know and (sometimes) used in practice approaches to incomplete symbolic reasoning about program behavior are based on symbolic model checking. .... Also mention interpolants 

