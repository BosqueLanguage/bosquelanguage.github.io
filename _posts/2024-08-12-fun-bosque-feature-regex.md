---
title: "Fun Bosque Feature -- Regexes"
date: 2024-08-12
layout: post
---

# Regular Expressions!!!
Regular expressions are a powerful and ubiquitous feature of modern programming languages. However, they are also a frequent source of bugs, confusion, security issues, and can even be subject to denial-of-service attacks -- a great rundown of these issues can be found [here](https://davisjam.github.io//files/publications/MichaelDonohueDavisLeeServant-RegexesAreHard-ASE19.pdf) and [here](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS). In addition to these issues, the historical use and development of modern regexes has been driven by text search tasks but, in [Bosque](https://github.com/BosqueLanguage/BosqueCore), a critical use case is validation and parsing. 

Keeping with the theme of Bosque, instead of simply inheriting classic PCRE style regular expressions, we decided to do a ground-up design and implementation of a regular expression language for Bosque. The result is the **BRex** style regex language and matching engine which is designed to be a more maintainable, readable, and correct alternative to classic PCRE style regular expressions as well as support novel features of that are useful in the context of describe and validating data with a regular language.

## Goals
Specific goals for the BRex language include:
- **Readability**: BRex should be more comprehensible than classic PCRE style regular expressions. Specifically, BRex uses quoted literals, allows line breaks and comments, simplifies the context dependent use of special characters, and supports named regex patterns for composition and reuse.

- **Correctness**: BRex aims to reliably produce correct expressions. This means eliminating rarely used but complex features (back-references) in favor of alternative workflows,  providing a more structured and less error prone syntax, and simplifying greedy/lazy matching behavior. 

- **Performance**: BRex is designed with performance as a primary goal. This is both for efficient average case matching, with a NFA simulation based engine, and for efficient worst case matching, avoiding pathalogical ReDOS behavior.

- **Extensibility**: BRex should expand the domain where regular language tools can be applied. Specifically, BRex adds new features, such as conjunction and negation, that are useful in data specification/validation, and provides a full language for describing URI paths and path-globs with well founded semantics.

- **Universality** BRex is designed for everyone! Unicode (utf8) support is part of the design and implementation from the start and simple Char regexes are a distinct subset for use where the simplicity of the printable + whitespace ASCII subset is desired.

## Notable Features
BRex includes a number of distinct features that are not present in classic PCRE style regular expressions. These include:

- **Quoted Literals**: BRex supports quoted literals, `/"this is a literal"*/`. This allows for the use of special characters in literals without escaping, making the expression more readable, and provides a way to distingush `/"unicode literal 🌶"/` from `/'simple character literals %x59;'/c`.

- **Unified Escaping**: BRex eschews the classic, and frequently error prone, PCRE style of using `\` to escape special characters in favor of a unified escaping mechanism. All characters can be escaped using hex codes `/"%x7;%0;"/` or memonic names `%a;`, `%NUL;`, or `%;` (for the quote literal). Unicode can be hex escaped or inserted directly into the string (or char range `/[🌵-🌶]/`)

- **Comments and Line Breaks**: BRex supports comments and splitting as well as whitespace within an expression (outside of a literal or range whitespace is ignored). This allows for the expression to be structured for readability. 

- **Named Patterns**: BRex supports named patterns for composition and reuse allowing expressions to be built up in parts and common features to be shared across multiple expressions -- `/[+-]${Digit}+/`.

- **Negation**: BRex supports a syntactically limited from of negation (`!` operator) to express that a string does not match a regular expression -- `/!(".txt" | ".pdf")/`. This generalizes conjunction in ranges `[^a-z]` to the full language -- but only the top-level can be negated allowing us to preserve match efficiency.

- **Conjunction**: BRex supports the `&` operator to express, in a single regular expression, that a string must a member of multiple languages! This is a invaluable feature for data validation tasks where there are often multiple logically-distinct constrains that would otherwise be forced into a single expression -- check a zipcode is valid *and* for Kentucky `/"[0-9]{5}("-"[0-9]{3})? & ^"4"[0-2]/` or with names `/${Zipcode} & ^${PrefixKY}/`.

- **Anchors**: BRex supports anchors that allow guarded and conditional matching before and after fixed regex -- `/${UserName}"_"^<[a-zA-Z]+>$!(".tmp" | ".scratch")/` which matches the `[a-zA-Z]+` term but only when proceeded by the username/underscore and not followed by the given extensions.

## Example Expressions

### A simple regex -- letter _h_ followed by one or more vowels
In BRex this is expressed as and defaults to a Unicode regex:
```
/"h"[aeiou]+/
```

We can also specify that it matches ASCII printable and whitespace (using an c literal and the `c` flag):
```
/'h'[aeiou]+/c
```

Comments and line breaks are fine too (note whitespace is ignored outside of literals and ranges):
```
/
  "h"      %% start with h
  [aeiou]+ %% followed by one or more vowels
/
```

### Using unicode and escapes
We can use unicode directly in the regex:
```
/"🌶" %*unicode pepper*%/
```

Or we can use hex escapes:
```
/"%x1f335; %x59;" %*unicode 🌵 and Y*%/
```

Common escapes are also supported:
```
/"%NUL; %n; %%; %;" %* null, newline, literal %, and a " quote*%/
```

Also in ranges:
```
/[🌵🌶]?/
```

### The usual set of repeats and optional (but no greedy/lazy behaviors)
A simple number regex:
```
/[+-]? ("0" | [1-9][0-9]+)/
```

A Zipcode regex:
```
/[0-9]{5}("-"[0-9]{3})?/
```

A (simple) filename + short extension regex:
```
/[a-zA-Z0-9_]+ "." [a-zA-Z0-9]{1,3}/
```

### Named patterns
A simple number regex with named parts (defined previously):
```
/[+-]? ("0" | ${NonZeroDigit}${Digit}+)/
```

### Conjunction, Start/End Anchors, and Negation
A regex that matches a Zipcode **AND** that is a valid Kentucky prefix:
```
/${Zipcode} & ^"4"[0-2]/
```

A regex that matches a filename that ends with ".txt":
```
/${FileName} & ".txt"$/
```

A regex that matches a filename that does not end with ".tmp" or ".scratch":
```
/${Filename} & !(".tmp" | ".scratch")$/
```

### Matching Anchors
These allow is to find matches that are guarded by other expressions (which we don't want to include in the match).

For a file like mark_abc.txt we can match the abc part but make sure it is contained in the context of the username and not followed by a .tmp or .scratch file:
```
/"mark_"^<${FilenameFragment}>$!(".tmp" | ".scratch")/
```

## Implications
As the examples show the new regex language syntax eliminates many of the common sources of errors that occur when writing classic PCRE expressions, mainly around mistakes in escaping and greedy/lazy matching. The ability to add comments and compose regexes from named patterns is also a big win for readability, understandability, and maintainability when the inevitable need to update a regex comes! 

The new features of negation and conjunction are also powerful tools for data validation tasks and allow for a more natural expression of complex constraints that would otherwise need to be awkwardly combined into a single expression. Similarly the explicit separation of unicode vs. simpler character based regexes allows for the right level of complexity to be used for the task at hand and avoids the pitfalls of unicode regex matching when it is not needed or the subtle semantic issues of the behaviors when running an ASCII regex on a unicode string.

Finally, the regex matching semantic design has been carefully crafted to (1) be implemented via a NFA matching engine and (2) encodes directly into the semantics for the [theory of regular expressions in Z3](https://microsoft.github.io/z3guide/docs/theories/Regular%20Expressions). The ability to run regex matching via a NFA simulation ensures that worst case behavior is always polynomial, thus avoiding the worst of the pathological ReDOS behavior in backtracking based engines. The NFA algorithms also tend to require fewer heuristics and are more predictable in their performance. 

Matching the Z3 theory semantics ensures that we can consistently solve for Regex constraints and generate strings (or counter examples) for programs that depend on regex validated values!
