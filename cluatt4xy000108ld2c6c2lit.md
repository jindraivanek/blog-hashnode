---
title: "F# tips weekly #11: Active patterns (2)"
datePublished: Thu Mar 28 2024 06:00:46 GMT+0000 (Coordinated Universal Time)
cuid: cluatt4xy000108ld2c6c2lit
slug: f-tips-weekly-11-active-patterns-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711363273484/e126e071-9b00-43a1-b03b-0d2c8bd6ef0e.jpeg
tags: tips, f, fsharp

---

Continuing from the previous week, let's delve into some active pattern implementation details and advanced use cases.

## Single-case active pattern

As mentioned in [Tip #2](https://jindraivanek.hashnode.dev/f-tips-weekly-2-single-case-pattern-match), we can use a single-case active pattern to directly transform a value:

```fsharp
let (|FromNull|) = Option.ofObj

let workWithNull (FromNull maybeValue) = ...
```

It's worth noting that the use case in the definition is optional, for example:

```fsharp
let (|FromNull|) x = FromNull (Option.ofObj x)
```

is equivalent.

---

Another interesting case for a single-case active pattern is a pattern that only performs some side effect:

```fsharp
let (|Debug|) x =
    printfn "%A" x // or other logging
    x
```

Then we can add debugging prints of function parameters by just adding one word. We can even improve it further by marking this pattern as obsolete to ensure we don't forget to remove it later.

```fsharp
[<System.Obsolete>]
let (|Debug|) x =
    printfn "%A" x // or other logging
    x
```

This will generate a warning for each use of the pattern.

## Call as a function

Active patterns not only look like standard functions, they actually are and can be called:

```fsharp
(|Match|_|) """Id=(\d+), Value=(\d+\.\d+)""" "Id=123, Value=3.1415"
```

This function has a type `string -> string -> option<list<string>>` as we would expect.

In the case of a complete active pattern, there is a little compiler magic involved. Consider this even/odd active pattern:

```fsharp
let (|Even|Odd|) x = if x % 2 = 0 then Even x else Odd x
```

We are using pattern cases `Even` and `Odd` in the definition as if they were cases of some discriminated union, but the function has the type `int -> Choice<int, int>`. [Choice](https://fsharp.github.io/fsharp-core-docs/reference/fsharp-core.html#category-3_4) is a generic type for multiple cases. We can actually use it in the active pattern directly, and the result will be the same:

```fsharp
let (|Even|Odd|) x = if x % 2 = 0 then Choice1Of2 x else Choice2Of2 x
```

We can call this complete active pattern like this:

```fsharp
let isEven x = (|Even|Odd|) x |> function | Choice1Of2 _ -> true | Choice2Of2 _ -> false
```

As you can see, in the case of a complete pattern, it doesn't make much sense to call it as a function, as we need to pattern match on the result anyway.

## List patterns

List patterns like `::` and `[]` are very useful, but sometimes we wish for more powerful patterns.

For example, we might want to match on the end of a list:

```fsharp
let (|Reversed|) xs = List.rev xs
```

A fun example is checking for a palindromic list:

```fsharp
let rec isPalindrome xs =
    match xs with
    | [] -> true
    | [x] -> true
    | x :: Reversed(y :: xs) when x = y -> isPalindrome xs
    | _ -> false
```

Another useful pattern can be a variant of `List.take`.

```fsharp
let (|TakeN|_|) n xs =
    let rec loop acc n xs =
        match xs with
        | x :: tl when n > 0 -> loop (x :: acc) (n-1) tl
        | [] when n > 0 -> None
        | xs -> Some (List.rev acc, xs)  
    loop [] n xs
```

Note that this pattern returns both the first *n* items and the rest of the list. This is common when writing active patterns - not discarding any of the input so we can combine with more patterns.

To get the sixth item of the list:

```fsharp
match [1..10] with
| TakeN 5 (_, x :: _) -> x
```

To get the third from the back:

```fsharp
match [1..10] with
| Reversed(TakeN 2 (_, x :: _)) -> x
```

Similarly, we can implement active pattern variants of other `List` functions, like `List.takeWhile`.

## Generic

Active patterns can also be generic:

```fsharp
let (|FromJson|) (x: string) = FromJson (System.Text.Json.JsonSerializer.Deserialize x)

match """{"Id": 1, "Name": "bob"}""" with
| FromJson ({ Id = a; Name = b } as r) -> printfn $"{r}"
```

## Active pattern function calls order

Evaluation of match expressions is lazy - only the patterns that are needed to find the correct pattern are executed. Let's alter our parsing active patterns with logging:

```fsharp
let (|Int|_|) str =
   match System.Int32.TryParse(str: string) with
   | (true, x) -> 
        printfn "parsed %s as int %d" str x
        Some(x)
   | _ -> 
        printfn "could not parse %s to int" str
        None

let (|Float|_|) str =
   match System.Double.TryParse(str: string) with
   | (true, x) -> 
        printfn "parsed %s as float %f" str x
        Some(x)
   | _ -> 
        printfn "could not parse %s to float" str
        None
```

Then this expression

```fsharp
match "3.14" with
| Float f -> printfn "Float: %f" f
| Int i -> printfn "Int: %d" i
```

prints out

```plaintext
parsed 3.14 as float 3.140000
Float: 3.140000
```

the `Int` active pattern is not executed. Evaluation of match expression patterns ends when we find a matching case.

But laziness goes deeper than that; it works also for combined patterns:

```fsharp
match ["3.14"; "42"] with
| [Int i; Int j] -> printfn "Case 1"
| [Float f; Int i] -> printfn "Case 2"
```

prints out

```plaintext
could not parse 3.14 to int
parsed 3.14 as float 3.140000
parsed 42 as int 42
Case 2
```

We can see that when the first pattern in the list failed, the second one wasn't attempted, and we moved to the second case.