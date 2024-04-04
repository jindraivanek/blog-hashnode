---
title: "F# tips weekly #12: Recursive active patterns - parser example"
datePublished: Thu Apr 04 2024 07:15:31 GMT+0000 (Coordinated Universal Time)
cuid: clukwk8v8000608l30tc05txw
slug: f-tips-weekly-12-recursive-active-patterns-parser-example
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712151393538/d666eed2-357b-4669-8cde-7bd8290ca8aa.jpeg
tags: tips, f, parsing, fsharp

---

This week, this article will be a little different. I want to show you that we can use recursive Active Pattern to create an expression parser. While I wouldn't recommend using this instead of a library such as *FParsec* for parsing, I think it's very cool that something like that is possible with quite a nice syntax, and also that it's a great showcase of the power of Active Patterns.

## Goal

Our goal will be to parse a simple math expression that would consist of:

* non-negative whole numbers
    
* basic arithmetic operators: `+`, '-', '\*', '/'
    
* parentheses
    

The output will be defined by this type:

```fsharp
type Op =
    | Plus
    | Minus
    | Multiply
    | Divide

[<RequireQualifiedAccess>]
type Ast =
    | Number of int
    | Op of Op * Ast * Ast
```

For example, `12/(1+2)` will be parsed as `Op (Divide, Number 12, Op (Plus, Number 1, Number 2))`.

### Tokenizer

To make parsing with active patterns easier, we will first transform the input to a list of tokens, representing one item in our language (math expression), and then throw away any whitespaces. This way, we can avoid the need for handling details like multiple spaces. In our case, only multiple characters that create tokens are numbers; any other non-whitespace character creates a new token. To do that, we define a helper function `chunkBy`:

```fsharp
// split into chunks, where a chunk is where function f returns true for adjacent elements
let chunkBy f xs =
    let rec loop prev acc xs =
        match xs, acc with
        | [], _ -> List.rev acc
        | x :: rest, g :: accRest ->
            if f prev x then loop x ((x :: g) :: accRest) rest
            else loop x ([x] :: List.rev g :: accRest) rest
        | _, [] -> loop prev [[]] xs
    
    match Seq.toList xs with
    | [] -> []
    | [x] -> [[x]]
    | x :: xs ->
        loop x [[x]] xs

let tokens (s: string) =
    s |> chunkBy (fun x y -> System.Char.IsDigit x && System.Char.IsDigit y) 
    |> List.map (Seq.filter (System.Char.IsWhiteSpace >> not) >> Seq.map string >> String.concat "")
    |> List.filter (not << System.String.IsNullOrEmpty)
```

The `chunkBy` function allows us to define tokens by a condition on successive characters. I won't go into the details of the `chunkBy` function, as it is out of the scope of today's article. We use it in the `tokens` function, and then delete all whitespaces from chunks.

### Number

For a number, we can use the `Int` pattern from [tip #10](https://jindraivanek.hashnode.dev/f-tips-weekly-10-active-patterns-1?source=more_articles_bottom_blogs#heading-returning-values):

```fsharp
    let (|Int|_|) str =
        match System.Int32.TryParse(str: string) with
        | (true, x) -> 
                Some(x)
        | _ -> 
                None
```

### Operators

To parse a (sub)expression like `1+2`, we can make a `Split` active pattern:

```fsharp
    let (|TakeUntil|_|) x xs =
        match List.takeWhile ((<>) x) xs with
        | y when y = xs -> None
        | y -> Some(y, List.skipWhile ((<>) x) xs)

    let (|Split|_|) split xs =
        match xs with
        | TakeUntil split (x1, (_ :: x2)) -> Some(x1, x2)
        | _ -> None
```

`TakeUntil` is a variant of `TakeN` from [tip #11](https://jindraivanek.hashnode.dev/f-tips-weekly-11-active-patterns-2#heading-list-patterns). `Split` finds the first occurrence of the given item and returns the list before that and after.

We can use `Split` like this:

```fsharp
match tokens "41+1" with
| Split "+" (s1, s2) -> printfn "%A" (s1, s2)
```

### Parentheses

A pattern that detects a list that begins and ends with a given item is easy as:

```fsharp
let (|Surround|_|) before after xs =
    match xs with
    | b :: Split after (l, []) when b = before -> Some l
    | _ -> None
```

But for supporting inner parentheses (e.g., expressions like `1 + (4 / (1 + 1))`), we need to count `before` and `after` items:

```fsharp
let (|Eq|_|) x y =
    if x = y then Some()
    else None

// detect a list surrounded by before and after items, allow inner surrounds, count pairs
let (|Surround|_|) before after xs =
    let rec f d acc xs =
        match xs with
        | _ when d < 0 -> None // wrong pairing
        | Eq after :: [] when d = 1 -> List.rev acc |> Some // closing pair on end -> success
        | Eq before :: rest -> f (d + 1) (before :: acc) rest // inner "before" item -> increase depth
        | Eq after :: rest -> f (d - 1) (after :: acc) rest // inner "after" item -> decrease depth
        | x :: rest -> f d (x :: acc) rest // other item copy to output
        | _ -> None
    match xs with
    | Eq before :: rest -> f 1 [] rest
    | _ -> None
```

`Eq` here is only replacement for `... when x = y` to make things clearer.

### Operators again

There is a problem with the use of `Split` for operators - it splits on the first occurrence, and if there are multiple of them, we don't consider the correct split. For example, for the expression `(1 + 2) + 3`, we only consider splitting on the first `+`, leading to incorrect subexpressions `(1` and `2) + 3`, which are rejected by the `Surround` pattern. To solve this, we can make a pattern that tries all possibilities until we find the correct split.

```fsharp
let (|SplitsPick|_|) split f xs =
    let rec loop prev xs =
        seq {
            match xs with
            | TakeUntil split (x1, (_ :: x2)) ->
                yield (prev @ x1, x2)
                yield! loop (prev @ x1 @ [ split ]) x2
            | _ -> ()
        }
    loop [] xs |> Seq.tryPick f
```

### Expr pattern

Putting it all together:

```fsharp
let rec (|BinaryOp|_|) s =
    let twoExprs = function | (Expr e1, Expr e2) -> Some (e1, e2) | _ -> None
    match s with
    | SplitsPick "+" twoExprs (e1, e2) -> Ast.Op (Plus, e1, e2) |> Some
    | SplitsPick "-" twoExprs (e1, e2) -> Ast.Op (Minus, e1, e2) |> Some
    | SplitsPick "*" twoExprs (e1, e2) -> Ast.Op (Multiply, e1, e2) |> Some
    | SplitsPick "/" twoExprs (e1, e2) -> Ast.Op (Divide, e1, e2) |> Some
    | _ -> None

and (|Expr|_|) s =
    match s with
    | [ Int x ] -> (Ast.Number x) |> Some
    | Surround "(" ")" (Expr e) -> Some e
    | BinaryOp e -> Some e
    | e -> 
        None
```

Here we are finally using recursive active patterns. Note that `BinaryOp` uses `Expr` pattern - that way `SplitsPick` selects only the correct splitting to two valid expressions.

Now we have a working solution:

```fsharp
let parse s =
    match tokens s with
    | Expr e -> printfn "%A" e
    | xs -> failwithf "cannot parse: %A" xs

parse "(10 * 4) / ((30/10) + 1)"
```

This will give us:

```plaintext
Op
  (Divide, Op (Multiply, Number 10, Number 4),
   Op (Plus, Op (Divide, Number 30, Number 10), Number 1))
```

### Performance note

The way how `SplitsPick` works is not good performance-wise because we're creating sub-lists for every possibility and running the `Expr` pattern recursively on them. There are ways to fix it, for example, making `SplitsPick` aware of `Surround` logic and going for the top-level operator.

## Conclusion

With some list patterns utils, we can get powerful tools for writing a readable parser with active patterns. Also, this example shows how we can transform a sequence into recursive data types and apply patterns recursively.

Full code is available as [Gist](https://gist.github.com/jindraivanek/10b50f709ced2b39c5e7c0cc09b3a12b).