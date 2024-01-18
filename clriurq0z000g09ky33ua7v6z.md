---
title: "F# tips weekly #2: Single case pattern match"
datePublished: Thu Jan 18 2024 06:50:42 GMT+0000 (Coordinated Universal Time)
cuid: clriurq0z000g09ky33ua7v6z
slug: f-tips-weekly-2-single-case-pattern-match
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1705161388502/9701d094-bf52-4ce5-a200-816e39bab871.png
tags: tips, fsharp, pattern-matching

---

Every F# programmer is likely familiar with [pattern matching](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/pattern-matching) using the <kbd>match</kbd> keyword, as illustrated below:

```fsharp
match x with
| Some 0 -> "Zero"
| Some n when n > 0 -> "Positive"
| Some n when n < 0 -> "Negative"
| None -> "Unknown"
```

However, a less well-known fact is that pattern matching can be employed in various other contexts. It can be utilized in:

Full pattern matches with multiple cases

* Using the <kbd>match</kbd> keyword
    
* Using the <kbd>function</kbd> keyword
    

Single-case patterns

* As parameters in function definitions
    
* In lambda function definitions, where parameters come after the <kbd>fun</kbd> keyword
    
* On the left side of <kbd>let</kbd> bindings
    

Even seemingly simple constructs, such as:

```fsharp
let a, b = pair
```

or

```fsharp
let foo (a, b) = ...
```

utilize single-case pattern matching to destructure tuples.

As demonstrated in [Tip #1](https://jindraivanek.hashnode.dev/f-tips-weekly-1-single-case-du), this technique can also be applied to single-case Discriminated Unions (DUs). However, attempting to use an incomplete pattern match results in a warning. For instance:

```fsharp
let [ x; y ] = someList
```

produces a warning stating:

```fsharp
Incomplete pattern matches on this expression. For example, the value '[;;_]' may indicate a case not covered by the pattern(s).
```

Converting this into a complete pattern match is possible only by rewriting it using <kbd>match</kbd> or <kbd>function</kbd>. Nevertheless, in quick scripts where warnings and the possibility of exceptions are acceptable, this method provides a convenient way to destructure lists or arrays.

A powerful combination involves using single-case patterns with single-case [active patterns](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/active-patterns) to achieve elegant conversions without the need for intermediate bindings. In a C# interop scenario, for instance, an active pattern can be defined for <kbd>Option.ofObj</kbd> to safely handle values that could be <kbd>null</kbd>:

```fsharp
let (|FromNull|) = Option.ofObj

let workWithNull (FromNull maybeValue) = ...
```

In this example, <kbd>maybeValue</kbd> is of type <kbd>option</kbd>, and the auto-documented parameter indicates that we expect a value that could be <kbd>null</kbd>. This approach can be extended to various scenarios such as handling <kbd>Nullable</kbd> types, auto-parsing of strings, and validation.

<div data-node-type="callout">
<div data-node-type="callout-emoji">📆</div>
<div data-node-type="callout-text">We will delve into active patterns in more detail in future posts.</div>
</div>