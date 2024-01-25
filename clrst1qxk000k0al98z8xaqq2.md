---
title: "F# tips weekly #3: piped lambda function"
datePublished: Thu Jan 25 2024 06:00:12 GMT+0000 (Coordinated Universal Time)
cuid: clrst1qxk000k0al98z8xaqq2
slug: f-tips-weekly-3-piped-lambda-function
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1705675724584/d2794489-f331-41f7-9cbe-ca8fd7c33c4a.jpeg
tags: tips, pipe, fsharp, anonymous-function

---

Usage of [lambda functions](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/lambda-expressions-the-fun-keyword) as parameters for another function is well known. Less commonly used is the application of lambda functions in other contexts. We can leverage the fact that a lambda function is a value and use it wherever we can use a normal function. Let's explore the combination of lambda functions with the pipe `|>` operator.

One great application is to call a function with a parameter other than the last, which can be quite common with `Map` or `Set`:

```fsharp
let updateWithDefault key def (m: Map<_, _>) =
    m
    |> Map.tryFind key
    |> Option.defaultValue def
    |> fun x -> Map.add key x m
```

When we have a value that we want to add to a Map, we can't use `|> Map.add` directly. However, with a lambda function, we can pass the value from the pipe as any parameter.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Note that we don't need to use parentheses around <code>fun x -&gt; Map.add key x m</code> when used together with the <code>|&gt;</code> operator. This is because the <code>|&gt;</code> operator has lower precedence than function application.</div>
</div>

Another application is inlining single-case pattern matching as we seen in [previous week](https://jindraivanek.hashnode.dev/f-tips-weekly-2-single-case-pattern-match):

```fsharp
createTriple
|> fun (a, b, c) -> a + b + c
```

Also, we can use an active pattern:

```fsharp
value
|> someCSharpMethod
|> function FromNull o -> o
|> Option.defaultValue ...
```

Here, we use the `function` keyword, which is a shortcut for `fun x -> match x with ...`. This can also be used, of course, to handle multiple cases:

```fsharp
maybeValue
|> function
    | Some v -> v
    | None -> defaultValue
```

Another related technique is to inline an `if` condition in the pipe. We can use the `id` function to do nothing and pass the value through:

```fsharp
[1..10]
|> if filterEnabled then List.filter (fun x -> x % 2 = 0) else id
```

A nice advantage of this technique is that `List.filter` is not called for the `else` case, in contrast to:

```fsharp
[1..10]
|> List.filter (fun x -> if filterEnabled then x % 2 = 0 else true)
```

Overall, I believe using lambda functions with the pipe is a useful tool that can make code more readable and concise, especially when there is no need to create a new binding. When using pipes, don't forget that the value in the pipe is only one `|> fun x ->` away!