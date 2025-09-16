---
title: "SMT-able Language and Small-Model Validation"
date: 2025-09-18
layout: post
---

Part of the work on Bosque over our 21kloc experiment involved the translation of Bosque code into SMTLib for [logical analysis](https://bosquelanguage.github.io/2025/03/20/compact_solvable.html) -- both formal validation and testing applications. So, key question is -- how well did this work out??

The short answer is -- interestingly. The translation to SMTLib is pretty straightforward for many constructs and names are not mangled so it is easy to integrate the solver into a variety of workflows and tasks. 
- The highlight is that we can run the Z3 on the resulting SMTLib code and prove correctness and/or find failing inputs for interesting programs (see below for it in action on an in memory database based on the [SpecJVM DB benchmark](https://www.spec.org/jvm98/jvm98/doc/benchmarks/index.html)) &#128077; 
- The low-light is that the solver is _much slower_ than we would like, and even slower than an earlier prototype using a different translation approach (3-valued IR vs. tree-based here) &#128078;
- The cool part is that, this is still enough to be useful for AI agentic tasks &#128640;

![gif running on db here....](path/to/gif)

So, how does all this work, what is the relation to agentic AI, and what are the next steps?

## Hit the paper

## Agentic AI and Scaling Up!
(refer to ACM Queue)
