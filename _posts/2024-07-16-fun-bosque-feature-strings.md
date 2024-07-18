---
title: "Fun Bosque Feature -- Strings"
date: 2024-07-16
layout: post
---

# Strings!!!
As the first installment in a (hopefully ongoing and entertaining) series of posts on _fun_ Bosque features, I thought I would start with strings. Strings are a fundamental data type in most programming languages and Bosque is no exception. Despite their ubiquitous and seemingly simple nature, strings can often turn out to be the source of subtle complications, bugs, and security vulnerabilities!

There are a number of reasons for this, some of which we defer to later, but at the core is the tension between the simplicity of ASCII strings and the reality that we build software for the world -- and the world speaks many languages -- making Unicode and all the complexity it entails a requirement. So how should a programming language handle strings?

Bosque takes the view that different uses really do require different tradeoffs here!

## Unicode Strings -- 🌵🚀✨ and more
For the default `String` type Bosque defaults to `utf8` encoded strings. These cover the full unicode character set and are the defacto standard for intercommunication and storage. This allows for the full range of emoji, international characters, and more to be used in strings. 

In Bosque (unicode) string literals are denoted with the `"..."` marks. Special characters, or hex encoded char values, are escaped using a HTML inspired syntax of `%...;`. The value in the escape can be a hex value, say `%x1F335;` for the cactus emoji 🌵, a name of a special char like `%n;` or `%slash;` for the newline and / respectively, or the special `%%;` and `%;` for the % and " respectively. Examples of strings include:
```
"Hello, World!"       -> Hello World!
"🌵%x1F335;"          -> 🌵🌵
"a %; %dollar; %x59;" -> a " $ Y
```

In many situations we need multi-line literals and often times, for legibility, it is nice to be able to indent them to match the surrounding code. Bosque supports both of these, allowing literal newlines in any string literal and using a `\` as the first character of a line to indicate indentation only spaces in a newline. For example all of the following are equivalent and represent the same string value:
```
"Hello,%n; World!"

"Hello,
 World!"

        "Hello, 
       \ World!"
```

As we will hear many times in this blog -- correctness and reliability are key goals of Bosque. So, the string type provides aggressive validation for string literal (and other string inputs). Any string value is checked for valid `uft8` encoding and any truncated multi-byte values are compile errors (for literals) or input errors (for runtime values). Literals are also checked for unterminated escape sequences, bad escape names, and invalid numeric escapes. This is a simple but powerful tool to catch many typos or other small mistakes in literal strings in the code.
```
"Hello, %x1F335"         -> error missing ;
"Hello, %newline;"       -> invalid name (should be %n;)
"Hello, %x100000000000;" -> invalid hex character
"Hello, %x1G335;"        -> error G is not a valid hex character
```

## CStrings -- `[ -~\t\n]*` and thats it!
For many use cases the full Unicode character space is not needed and worrying about multi-byte characters, normalization, combining marks, visually similar glyphs, etc. is a burden and a source of bugs. These uses of strings for (mostly internal) data manipulation and processing can be well served by a simple ASCII string model. Bosque provides a simple `CString` type that consists of the _printable_ subset of `ascii` characters -- no embedded nulls, backspaces, bells, etc. -- which, with the disappearance of the teletype, now serve mainly as a a source of trivia and bugs!

These CString literals are denoted with the single `'...'` marks. Special characters and hex encoded char values are escaped using the `%...;` syntax. In a CString the special `%;` escape corresponds to a ' character (as opposed to a " for the unicode strings). Examples of strings include:
```
'Hello, World!'       -> Hello World!
"Y%x59;"              -> YY
"a %; %dollar; %x59;" -> a ' $ Y
```

As with the regular String type the CString type provides aggressive validation for string literal (and other string inputs). Any string value is checked for valid char ranges (e.g. no control or unicode values). Literals are also checked for unterminated escape sequences, bad escape names, and invalid numeric escapes.
```
"Hello, %x59"         -> error missing ;
"Hello, %newline;"    -> invalid name (should be %n;)
"Hello, %x0;"         -> invalid CString character
```

And of course CStrings also support multi-line literals and indentation using the same syntax as the unicode strings. 

## String APIs with Soft-Edges
In addition 

## Regex support, Templates, and ByteBuffers
...