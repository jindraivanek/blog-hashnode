---
title: "F# tips weekly #14: Memoize"
datePublished: Thu Apr 25 2024 05:00:09 GMT+0000 (Coordinated Universal Time)
cuid: clverz1oa00000al04qmvcwar
slug: f-tips-weekly-14-memoize
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1713966829912/d1653962-f278-4de0-bf06-730a5e4af79a.jpeg
tags: tips, f, fsharp

---

The memoize function is my favorite method for solving performance problems. Its significant advantage is that it requires only a slight change in code, which doesn't increase the complexity of the original code. Despite its short implementation, it combines several techniques that are worth a detailed explanation.

```fsharp
let memoize f =
    let cache = System.Collections.Concurrent.ConcurrentDictionary()
    fun x -> cache.GetOrAdd(x, lazy f x).Value
```

The `memoize` function takes another function `f` and returns a new function with the same type as `f`, but the results of this new function are cached. Repeated calls to the function with the same parameter will not call `f`, but only return the cached return value. We can use it like this:

```fsharp
let cachedFun = memoize (fun x -> expensiveFun x)
```

It's important that `cachedFun` doesn't take any arguments; otherwise, it will not work, as the cache gets recreated with every call.

Another way is to use `memoize` on top of the function body:

```fsharp
let cachedFun = memoize <| fun x ->
    // expensiveFun body
```

## Referential Transparency

Of course, memoize should be used only for functions that return the same result for the same input (and don't have any side effects). This property is called [referential transparency](https://en.wikipedia.org/wiki/Referential_transparency) and can also be rephrased as *"we can replace a function call with its result without changing the meaning of the program"*. All F# functions that don't use mutability outside their scope or side effects fulfill this requirement.

## Thread Safety

A more common variant of `memoize` uses `System.Collections.Dictionary`, but that is not thread-safe. By using `ConcurrentDictionary`, we can be sure that our cache is thread-safe. Also, by storing a `Lazy` value in the dictionary, we ensure that `f` will not be called twice in case of simultaneous calls. Here, we add a `Lazy` object into the dictionary, which when evaluated with `.Value`, actually executes the `f` function. Both the `GetOrAdd` method and the `Value` property on `Lazy` are thread-safe (more info: [GetOrAdd](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2.getoradd?view=net-8.0#system-collections-concurrent-concurrentdictionary-2-getoradd(-0-system-func((-0-1)))), [Lazy](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1?view=net-8.0#remarks)).

## Scope

The typical usage of `memoize` is to define it as a constant on the module level, which means the cache will be global and will live throughout the whole application lifetime. But of course, when we need to cache something only inside some algorithm, we can define the memoized function at a local level, and the cache will get garbage-collected when the memoized function goes out of scope.

## Cache Invalidation

When we use `memoize` on a global level, we need to be careful with cache size; it's possible that non-relevant objects will remain in the cache, wasting precious memory. However, it's quite easy to customize `memoize` with various invalidation techniques.

The simplest approach is probably limiting the number of items in the cache:

```fsharp
let memoizeLimited limit (f: 'a -> 'b) =
    let keysStack = System.Collections.Concurrent.ConcurrentQueue<'a>()
    let cache = System.Collections.Concurrent.ConcurrentDictionary<'a, Lazy<'b>>()
    let f' x =
        if keysStack.Count >= limit then
            let (_, y) = keysStack.TryDequeue()
            cache.TryRemove(y) |> ignore
        keysStack.Enqueue(x)
        f x
    fun x -> cache.GetOrAdd(x, lazy f' x).Value
```

Here, we use a `ConcurrentQueue` to store the order of keys inserted into the cache and remove older ones when we reach the limit. We put the invalidating logic into the computed function to take advantage of the thread safety already in place. This is just a basic sample and can be improved, for example, if we want to remove items that haven't been read for the longest time.

Another possibility is to invalidate entries based on the time of read/write.

## Specify Cache Key

To allow more flexibility, the key for caching can be customized.

```fsharp
let memoizeBy projection f =
    let cache = System.Collections.Concurrent.ConcurrentDictionary()
    fun x -> cache.GetOrAdd(projection x, lazy f x).Value
```

## More Parameters

Our `memoize` variant supports functions with only one argument; for multiple arguments, we need to use tuples. To avoid this, we can define helpers for more arguments:

```fsharp
let memoize2 f = memoize (fun (x, y) -> f x y) |> fun f' x y -> f' (x, y)
let memoize3 f = memoize (fun (x, y, z) -> f x y z) |> fun f' x y z -> f' (x, y, z)
...
```