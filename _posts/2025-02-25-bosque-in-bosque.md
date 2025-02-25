---
title: "Bosque-in-Bosque!"
date: 2025-02-25
layout: post
---

Happy new year to everyone! Got off the blogging track a but, hopefully, things will be back on track now. I just merged [PR #185](https://github.com/BosqueLanguage/BosqueCore/commit/ce81effe6a80de0973ced8840a7b2ad702d55128) and it seems like a good one to highlight a bit.

This specific PR marks the first PR where the major parts of the feature code are written in Bosque itself! This code also is built and runs as part of the CI test suite. This Bosque-in-Bosque (or self hosting) is a priority for the project as it will help rapidly smoke out bugs, get us stabilizing the standard library, and build up a large corpus of Bosque code for later testing/evaluation tasks. Happily, I can report that the process of writing this code was quite pleasant. Two features that really helped were the ability to use EADT (extended algebraic data-types) to model the AST and the explicit flow typing system to handle type-test and extract logic. 

## Extended Algebraic Data Types (EADTs)

As and example of EADTs consider the following code snippet from the PR:

```
concept Expression {
    field line: Nat;
}

datatype BinOpExpression extends Expression using {
    field left: Expression;
    field right: Expression;
}
of
AddExpression { }
| SubExpression { }
| MulExpression { }
| DivExpression { field checkDivByZero: Bool; }
;
```

In this bit of code we see (part) the the definition for the Bosque AST for binary operations. This is defined in an Object-Oriented style with a base type `Expression` and a set of derived classes for each specific operation. If all we had was classic inheritance we would need to define a separate class for each of the derived `AddExpression` ... `DivExpression` classes. Instead we use the EADT system to define the sub-parts of the AST that have simple/correlated relations like the `BinOpExpression` case here.

In the EADT system the `datatype` keyword is used to define the base class and the derived classes. The `extends` clause allows for general inheritance from previously defined concepts and the `using` clause allows us to declare more fields. Each of the `of` clauses defines a specific derived entity type that singly inherits from the enclosing 
datatype. The equivalent code in a straight OO design would be:

```
concept BinOpExpression extends Expression {
    Expression left;
    Expression right;
}
entity AddExpression extends BinOpExpression {
}
entity SubExpression extends BinOpExpression {
}
entity MulExpression extends BinOpExpression {
}
entity DivExpression extends BinOpExpression {
    field checkDivByZero: Bool;
}
```

As can be seen in this example, the code without EADTs, is more verbose, requiring more boilerplate code to be written and maintained. As an added bonus, as the inhertiance set of the `datatype` is closed at the declaration, the typechecker can also do completeness checks on any pattern matching code that is written against the EADT!

## Explicit Flow Typing

Explicit flow typing in Bosque is a variant of [anaphoric-macros](https://docs.racket-lang.org/anaphoric/index.html) in Racket (Lisp) or 
[conditional pattern matching](https://docs.oracle.com/en/java/javase/22/language/pattern-matching-switch-expressions-and-statements.html#GUID-E69EEA63-E204-41B4-AA7F-D58B26A3B232) from Java. This feature allows for the definition of a pattern match that also binds the matched value, 
or for special matches a projected value, to a variable. This is particularly useful in the context of type tests and extracts as it allows for the value 
to be directly used in the subsequent code. As an example from the PR:

```
method transformStdTypeToSMT(tsig: BSQAssembly::TypeSignature): SMTAssembly::TypeKey {
        if(tsig)@!<BSQAssembly::NominalTypeSignature> {
            return TransformNameManager::convertTypeKey(tsig.tkeystr);
        }
        else {
            if(this.assembly.isNominalTypeConcrete($tsig.tkeystr)) {
                return SMTAssembly::TypeKey::from($tsig.tkeystr);
            }
            else {
                return SMTAssembly::TypeKey::from('@BTerm');
            }
        }
    }
```

This code contains the test `if(tsig)@!<BSQAssembly::NominalTypeSignature>` which tests if the `tsig` value is _NOT_ a `NominalTypeSignature` and binds 
the value to the variable `$tsig`. This allows the subsequent code to directly use the value as `NominalTypeSignature` in the `else` block. This also 
works with option types and allows explicit naming of the bound value (Thanks to [@kwangure](https://github.com/kwangure) for the naming syntax ideas). 
For example:

```
let x: Option<Int> = ...;
if($y=x)@some {
    return $y + 1i;
}
else {
    %% $y is none in this branch
    return 0;
}
```
In this example we check if `x` is a `Some<Int>` and bind the value to `$y` in the `if` block. This allows the value to be directly used in the `return` statement. Without the explicit flow typing the value would need to be manually extracted from the `Some` and then used in the `return` statement.
```
if(x?Some<Int>) {
    return x@<Some>.value + 1i;
}
else {
    return 0;
}
```

This binding also works in if-then-else expressions and in match statments. We may extend the syntax to allow for chained flow type checks or checks on multiple values in a single condition. However, I was quite happy with how the current form gave 80% of the power of a flow-sensitive type checker (like in TypeScript) while being both explicit in what was being constrained and not requiring me to remember what the type-checker could/could not implicitly infer.
