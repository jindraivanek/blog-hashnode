---
title: "F# tips weekly #15: Recursive memoize"
datePublished: Thu May 09 2024 08:54:39 GMT+0000 (Coordinated Universal Time)
cuid: clvz0ijae000009lbhow02m3f
slug: f-tips-weekly-15-recursive-memoize
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715244690238/37d81def-382e-4061-9dc7-fdb7eb137a7d.jpeg
tags: tips, memoization, f, fsharp

---

Memoization of a recursive function can be often useful, but has some pitfalls compared to memoization of a simple function. Let's take a detailed look.

## Standard Memoization

Let's use the classical example of computing the *n*\-th item in the Fibonacci sequence. Utilizing the memoize function from [Tip #14](https://jindraivanek.hashnode.dev/f-tips-weekly-14-memoize) works, but we encounter a compiler warning:

```fsharp
let rec fib = memoize <| fun n ->
    if n < 2 then
        1
    else
        fib (n - 1) + fib (n - 2)
```

The warning reads:

> This and other recursive references to the object(s) being defined will be checked for initialization-soundness at runtime through the use of a delayed reference. This is because you are defining one or more recursive objects, rather than recursive functions. This warning may be suppressed by using '#nowarn "40"' or '--nowarn:40'.

The issue here is that `fib` actually represents a constant object representing the closure of our memoized function. While calling this closure recursively is possible, it can lead to this warning and associated runtime checks.

## Recursive Call Placeholder Trick

We can eliminate the warning by employing the following trick. Instead of using the `rec` definition, we add a *placeholder* parameter within the function passed to `memoizeRec`:

```fsharp
let fib = memoizeRec <| fun recF n ->
    if n < 2 then
        1
    else
        recF (n - 1) + recF (n - 2)
```

`memoizeRec` is defined as:

```fsharp
let memoizeRec f =
    let cache = System.Collections.Concurrent.ConcurrentDictionary()
    let rec recF x =
        cache.GetOrAdd(x, lazy f recF x).Value
    recF
```

Notice that the recursive function is defined *inside* the `memoizeRec` function. This approach allows us to use the `rec` keyword only there and not in memoized function, and avoids recursive calls on the object.

## Y-Combinator

When we apply the same idea outside the context of memoization, we got a way to define a recursive function without explicitly define it as recursive:

```fsharp
let mkRec f =
    let rec recF x = f recF x
    recF
```

We can apply this technique to create recursive functions in the same manner as with `memoizeRec`:

```fsharp
let fib =
    mkRec <| fun recF n ->
        if n < 2 then
            1
        else
            recF (n - 1) + recF (n - 2)
```

`mkRec` is essentially the same thing as the fixed-point combinator or Y-combinator, nicely explained for example [here](https://fssnip.net/7W9/title/Stepbystep-explanation-of-the-Y-combinator).

## Infinite Recursion

When working with recursive functions, we should always be aware of the possibility of infinite recursion. Typically, this leads to a `Stack overflow` error. However, in our case, we encounter a different error:

> System.InvalidOperationException: ValueFactory attempted to access the Value property of this instance.

This error occurs because we are using a `Lazy` object to compute the result, and there is a check to prevent accessing the same instance of `Lazy` within its evaluation, indicating a circular reference. While this incurs some performance cost, it comes with the advantage that we catch the problem of infinite recursion earlier, before a stack overflow occurs.