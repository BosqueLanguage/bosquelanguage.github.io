---
title: "Fun Bosque Feature -- Strings"
date: 2024-07-16
layout: post
---

# Strings!!!
As the first installment in a (hopefully ongoing and entertaining) series of posts on _fun_ Bosque features, I thought I would start with strings. Strings are a fundamental data type in most programming languages and Bosque is no exception. Despite their ubiquitous and seemingly simple nature, strings can often turn out to be the source of subtle complications, bugs, and security vulnerabilities!

There are a number of reasons for this, some of which we defer to later, but at the core is the tension between the simplicity of ASCII strings and the reality that we build software for the world -- and the world speaks many languages -- making Unicode and all the complexity it entails a requirement. So how should a programming language handle strings?

Bosque takes the view that different uses really do require different tradeoffs here!

## Unicode Strings -- \rocket cactus sparkles emoji and more
For the default `String` type Bosque defaults to `utf8` encoded strings. These cover the full unicode character set and are the defacto standard for intercommunication and storage. 


## CStrings -- [ -~\t\n]* and thats it!
However, for many use cases this is overkill. Uses of strings for (mostly internal) data manipulation and processing can be well served by a simple ASCII string model. So, Bosque provides a simple `CString` type that consists of the _printable_ subset of `ascii` characters -- no embedded nulls, backspaces, bells, etc. 

## String APIs with Soft-Edges