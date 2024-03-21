---
title: "F# tips weekly #10: Active patterns (1)"
datePublished: Thu Mar 21 2024 06:00:26 GMT+0000 (Coordinated Universal Time)
cuid: clu0tpr1c000y08js9xy2cy93
slug: f-tips-weekly-10-active-patterns-1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1710778853820/c80aa6de-74ce-4156-b0ea-ed568bbfa93f.jpeg
tags: tips, f, fsharp

---

Active patterns are a great and unique feature of F#, allowing us to extend pattern matching with custom cases.

## Partial Active Patterns

The most basic usage is to use active patterns as switches based on input value:

```fsharp
let (|PositiveInt|_|) x =
    if x > 0 then Some ()
    else None

match 1 with
| PositiveInt -> printfn "Positive"
| _ -> printfn "Not positive"
```

## Complete Active Patterns

It's also possible to create multiple case active patterns that enforce the use of all cases, similar to discriminated union cases.

```fsharp
let (|PositiveInt|NegativeInt|ZeroInt|) x =
    if x > 0 then PositiveInt
    elif x < 0 then NegativeInt
    else ZeroInt

match 0 with
| PositiveInt -> printfn "Positive"
| NegativeInt -> printfn "Negative"
| ZeroInt -> printfn "Zero"
```

If we omit one of the variants, we receive an `Incomplete pattern matches` warning.

## Combining Patterns

These simple examples don't look much different from an `if` expression, but the great power of Active patterns comes from their ability to combine them. For example, we can check the values inside a list:

```fsharp
match [-1 .. 5] with
| NegativeInt :: ZeroInt :: PositiveInt :: rest -> printfn "Negative, Zero, Positive, %A" rest
| _ -> printfn "Other"
```

## Returning Values

Typically, active patterns are used to also return values. For example, we can attempt to parse an integer from a string:

```fsharp
let (|Int|_|) str =
   match System.Int32.TryParse(str: string) with
   | (true, x) -> Some(x)
   | _ -> None
```

It's useful to define this as an active pattern instead of as a classic function because we can then use it inside other patterns and easily combine them. Here we use parsing active patterns inside list pattern:

```fsharp
let (|Int|_|) str =
   match System.Int32.TryParse(str: string) with
   | (true, x) -> Some(x)
   | _ -> None

let (|Float|_|) str =
   match System.Double.TryParse(str: string) with
   | (true, x) -> Some(x)
   | _ -> None

match "123 3.1415".Split(" ") |> Array.toList with
| [Int i; Float f] -> printfn "Int: %d, Float: %f" i f
| _ -> printfn "Wrong input"
```

## Parameters

Active patterns can also take parameters outside of the main one from the match expression. My favorite example is the Regex match pattern:

```fsharp
let (|Match|_|) (pat: string) (inp: string) =
    let m = Regex.Match(inp, pat) in

    if m.Success then
        Some(List.tail [ for g in m.Groups -> g.Value ])
    else
        None
```

The Regex pattern is the first parameter of the active pattern. The main parameter (the one from the `match` expression) is always the last one.

We can use it like this:

```fsharp
match "Id=123, Value=3.1415" with
| Match """Id=(\d+), Value=(\d+\.\d+)""" [id; value] -> printfn "Id: %s, Value: %s" id value
| _ -> printfn "No match"
```

Note that the first parameter in the `Match` pattern goes directly to the active pattern function call, but the second one is the active pattern output, which allows us to name it or apply more pattern matching on it. Here we use all regex match groups as output in a list. But we can also combine it with parsing active patterns:

```fsharp
match "Id=123, Value=3.1415" with
| Match """Id=(\d+), Value=(\d+\.\d+)""" [Int id; Float value] -> printfn "Id: %i, Value: %f" id value
| _ -> printfn "No match"
```

## Other Resources

[Microsoft Documentation on Active Patterns](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/active-patterns)

[Active Patterns in F#: Convenience](https://fsharpforfunandprofit.com/posts/convenience-active-patterns/)