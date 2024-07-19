---
title: "Fun Bosque Feature -- Strings"
date: 2024-07-17
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
One of the advantages of splitting these two string types is that it allows for a clear separation of the APIs that are needed for each. Consider one of the major pain points of unicode strings -- character indexing. In a `uft8` encoded string the number of bytes in a character can vary from 1 to 4. This means that indexing into a string by character is a awkward operation and, even in apis that support it (like JavaScript stings) have subtle bugs -- consider the `charCodeAt` vs `codePointAt` methods or indexing halfway into a non-utf16 postion ☹️ Even if everything works correctly we still need to worry about normalization, combining marks, glyphs, and never forget z̵̨̞̑̍͋a̸̪̒̐l̷͎̩̫̿g̷̗̾o̴͖͆ text!

By splitting the string types, and their uses, Bosque can provide easy to understand integer index based APIs for `CStrings`, where characters are always 1 byte and always have a single representation. For the unicode `String` type we provide a more complex regex/position based API where operations are defined based on matching pattern positions and slicing instead of raw indexing.

For example, consider a substring api. In a CString we can just use integer indices:
```
let s: CString = 'Hello, World!';
let sub: CString = s.extractFront(5); //'Hello'
```

In a String we need to use a regex based approach:
```
let s: String = "Hello, World!";
let sub: String = s.extractFront([^,]); //'Hello'
```

Initially this seems like a small difference but notice that it eliminates the need to worry about slicing in the middle of a character and makes explicit which chars are being matched, in this case we (correctly) include any combining marks. 

We are working on a flexible flavor of slicing as well, allowing for open/closed ranges and regex/constant (or integer for CString) based end points ([Issue #95](https://github.com/BosqueLanguage/BosqueCore/issues/95)). This allows for a wide range of operations to be expressed in a simple and compact manner. For example:
```
let s: String = "Hello, World!";
let what: String = s(" " : /[.?!]/]; //World!
let say: String = s[ : ","); //Hello

let cs: CString = 'Hello, World!';
let what: CString = cs[-6 : ]; //World!
let say: CString = cs[ : /',' | ' '/c); //Hello
```

Surprisingly, the use of regex expressions (or string literals) initially seems to be more complex than raw integer indexing but, in practice, it actually seems to do a better job of expressing the underlying intent of the operation and is more robust to varied (or corner case inputs). Interestingly, this seems to mirror the experience of [FlashFill in Excel](https://support.microsoft.com/en-us/office/using-flash-fill-in-excel-3f9bcf1e-db93-4890-94a0-1578341f73f7), which allows a user to provide examples of a string operations on a column of data and learns a program to perform the operation -- for example taking email addresses and extracting the domains. In this system the underlying program is actually in a simplified [regex matching and string extraction/concatination language](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/popl11-synthesis.pdf) as it turnd out that the regex based approach was much more robust and generalized better than a more traditional string manipulation API.


## Regex support, Templates, and ByteBuffers
There is of course much more to strings than just literals and slicing. In the same way we sought to simplify and improve string types for the multiple roles they play in programs, Bosque also has specialized (improved) Unicode and CString regex support, templates for building string in a safe and checkable manner, and newtype-able byte buffers for situations like (opaque) OS of FileSystem specific path manipulation. We will dive into some of these in future posts!