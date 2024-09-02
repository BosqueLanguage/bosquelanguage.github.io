---
title: "Fun Bosque Feature -- TypeOf Strings"
date: 2024-09-02
layout: post
---

# Simple new `type` and Type-Of Strings!!!
In the last 2 features of the blog we have looked at the Bosque {% raw %}[string types]({{ site.base_url }}{% link _posts/2024-07-17-fun-bosque-feature-strings.md %}){% endraw %} and a {% raw %}[reworking of regular expressions]({{ site.base_url }}{% link _posts/2024-08-12-fun-bosque-feature-regex.md %}){% endraw %}. In this entry we are going to explore how these features, work the the new `type` operator in Bosque to provide a powerful and flexible way to work with strings containing implicitly structured data in a type safe manner.

## Bosque new `type` Operator
Type alias declarations are the feature that allow us to lift from primitive types built into the language into types that are specific to an application domain. A `type` alias is a new-type definition that takes a primitive type and creates a _distinct_ new named type. 

The ability to create distinct semantic/conceptual types using primitive data is a powerful tool for developers and can be easily used to prevent eliminate entire classes of subtle bugs. Consider the motivating example, from this [paper](https://dl.acm.org/doi/10.1145/3133928) on argument position defects of a function with two string parameters.
```
function getUser(companyId: String, userId: String): User {...}

function doSomething(companyId: String, userId: String): User {
    let user = getUser(userId, companyId); %%oops -- swapped the parameters
    ...
}
```

In this example the `companyId` and `userId` parameters are both of type `String` so the type-checker will miss the accidental swap of the parameters leading to an error or accidental disclosure of data! However, if we define a new `type` for each of these parameters, even if they are still using strings as the underlying type, we will catch this error with the type system.
```
type CompanyId = String;
type UserId = String;

function getUser(companyId: CompanyId, userId: UserId): User {...}

function doSomething(companyId: CompanyId, userId: UserId): User {
    let user = getUser(userId, companyId); %%now detected as type error!
    ...

    %%still some problems!
    let user2 = getUser(
        " Robert'); DROP TABLE Students;--"<CompanyId>, 
        ""<UserId>
    );
    ...
}
```

The use of type aliases will catch the parameter-confusion error but cannot catch issues like the potential SQL injection attack in the `user2` call as the underlying type of a `CompanyId` is still just a string and can have arbitrary contents. 

## Type-Of Strings
To support the, very common case, of having latently structured data passed around as strings Bosque provides specialized support for declaring type aliases of strings (and character strings). These string type aliases can be parameterized by 
a regular expression using the syntax:
```
type Zipcode = String of /[0-9]{5}("-"[0-9]{4})?/;

const azip: Zipcode = "10001";          %%error a String -- not Zipcode
const azip: Zipcode = "abc"<Zipcode>;   %%error not accepted by Zipcode regex
const azip: Zipcode = "10001"<Zipcode>; %%ok
```

In this example the specification declares a type alias `Zipcode` as a `String` that _must be in the language_ defined by the regex `/[0-9]{5}("-"[0-9]{4})?/`. The first two assignments are errors as the literal values are not type compatible and do not match the regex respectively. The third assignment is correct as the literal string value matches the `Zipcode` regex. 

Returning to the running example we can see how this allows us to catch the SQL injection attack at compile time, or runtime for dynamically generated strings, in the `user2` call. 
```
type CompanyId = CString of /[a-zA-Z0-9_]+/c; %%CString is better here as we don't care about unicode
type UserId = CString of /[a-zA-Z0-9_ -]+/c;
    
function doSomething(companyId: CompanyId, userId: UserId): User {
    ...
    %%Type error for both literals (at compile time)!
    let user2 = getUser(
        " Robert'); DROP TABLE Students;--"<CompanyId>, 
        ""<UserId>
    );
    ...
}
```

The `type` alias feature has been used to create a `CompanyId` type that is a `CString` and that is restricted to a specific (non-empty) set of SQL safe characters, which can be checked at compile time and/or automatically at runtime, and catch issues like the possible SQL injection attack in the `companyId` type. 

In this example the Type-Of strings were all created with the literal syntax `literal<T>` and so the errors were caught at compile time. However, these checks are also always run for dynamically generated values as well: 
```
let cid = CompanyId{'company'.prepend('_')};  %%ok regex accepts the string
let cid = CompanyId{'company'.prepend('--')}; %%runtime error "--" not in the regex language
```

## Stacking 
In addition to simply defining a new type from a primitive type, the `type` operator can also be used to stack new types on top of each other. This allows for the creation of more complex types that are composed of simpler types. For example, we can define a `type` that is a `Zipcode` specific to, say, Kentucky as:
```
type Zipcode = CString of /[0-9]{5}("-"[0-9]{4})?/c;
type KentuckyZipcode = Zipcode of /^"4"[0-2]/c; %%Zipcode and starting with "4" and [0-2]
```

## Other Values
We have focused on string in this post but new types can be created from any primitive type in Bosque. This includes the other primitive types like `Int`, `Float`, `Bool`, `BigInt`, `Char`, `UUID`, etc. For these values the compiler is aware of the underlying type, and where possible, lifts primitive operators to the new type as well. For example, we can define a new `type` based on `Int` and get the basic compare/arithmetic operations:
```
type Fahrenheit = Int;

let a = 100i_Fahrenheit;​
let b = 99i_Fahrenheit;​

assert a < b;​
```

## Take-away and Next Time
As this post shows the ability to declare new (distinct) `type` aliases for primitives, and strings in particular, combined with regex validation is a very powerful tool, and easy to use, for modeling data domains and avoiding entire classes of (serious!) bugs. Next time we will take a look at how we can make the new `type` operator even more powerful by allowing arbitrary user defined constraints on the values, for example a percentage type that is always in the range [0, 100], and associating operations and functions with the newly declared type! 🚀✨