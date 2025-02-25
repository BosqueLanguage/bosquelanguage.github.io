---
title: "Bosque in Bosque!"
date: 2025-02-25
layout: post
---

Happy new year to everyone! Got off the blogging track a bit after the new year but, hopefully, will be back on track now. I just merged [PR #185](https://github.com/BosqueLanguage/BosqueCore/commit/ce81effe6a80de0973ced8840a7b2ad702d55128) and it seems like a good one to highlight a bit.

This specific PR marks two interesting events in the project. The first is that the majority of the code is now written 
in Bosque itself! This code also is built and runs as part of the CI test suite. This Bosque-in-Bosque (or self hosting) 
is a priority for the project as it will help rapidly smoke out bugs, get us stabilizing the standard library, and build 
up a large corpus of Bosque code for later testing/evaluation tasks. Happily, I can report that the process of writing 
this code was quite pleasant. Two features that really helped were the ability to use EADT (extended algebraic data
types) to model the AST and the explicit flow typing system to handle type-test and extract logic. 

```
concept Expression {
    line: Nat;
}

datatype BinOpExpression extends Expression using {
    left: Expression;
    right: Expression;
}
of
AddExpression { }
SubExpression { }
MulExpression { }
DivExpression { checkDivByZero: Bool; }
```

In the first bit of code ... 

```
xxxx
```

In the second bit of code ...
I'll talk more about these two features in future posts but, for now, I'll just say that they are both quite useful and 
I think will serve developers well.




_compactly solvable language_

community activity!!!

EADT and explicit flow typing
