---
title: "F# tips weekly #16: Asynchronous memoize"
datePublished: Thu May 30 2024 05:00:41 GMT+0000 (Coordinated Universal Time)
cuid: clwssejv4000b09m5eztv5dyf
slug: f-tips-weekly-16-asynchronous-memoize
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716994940861/fe713b31-6dbd-4d8f-a607-5977c63ccb2c.jpeg
tags: tips, memoization, f, fsharp

---

Caching the results of asynchronous functions is a common task. Can we use `memoize` for it? Let's find out!

## Memoized Task

Surprisingly, `task` works great with the basic `memoize` function from [Tip #14](https://jindraivanek.hashnode.dev/f-tips-weekly-14-memoize). If we use a function that returns `Task<_>`, we get a memoized version of that `task`. For example:

```fsharp
let t = memoize (fun x -> task { printfn "Run %i" x; Task.Delay 1000; return x })
[|1..10|] |> Array.map (fun x -> x % 2 |> t) |> System.Threading.Tasks.Task.WhenAll |> fun t -> t.Result
```

This code will output `Run 0` and `Run 1` just once.

This happens because `Task<_>` remembers its computed value. If the `Task<_>` value is queried again, it just returns the value without recomputation. So, our memoization is simply caching `Task<_>` values, but it works just fine. `ConcurrentDictionary` and `Lazy` take care of thread safety as in the non-task case.

This is quite useful for caching things like database queries. All memoize variants we showed earlier work with `task` out of the box.

## Memoized Async

With `async`, it's a different story. `async` doesn't store its result; it's more like a delayed function that can be executed asynchronously. We can't modify `memoize` in a simple way because we would need to compute async inside `cache.GetOrAdd`, which is not easily possible without breaking thread safety.

One way to solve this is to handle thread locking ourselves using a synchronization primitive:

```fsharp
[<Struct>]
type MemoizeAsyncState<'a> = | Running of sem:System.Threading.SemaphoreSlim | Completed of 'a

let memoizeAsync f =
    let cache = System.Collections.Concurrent.ConcurrentDictionary()
    fun x -> async {
        match cache.GetOrAdd(x, valueFactory = fun _ -> Running(new System.Threading.SemaphoreSlim 1)) with
        | Running sem ->
            if sem.Wait(0) then
                // Not in cache, computing
                try
                    let! y = f x
                    cache.AddOrUpdate(x, Completed y, (fun _ _ -> Completed y)) |> ignore
                    return y
                finally
                    sem.Release() |> ignore
            else
                // Another thread is computing
                sem.Wait()
                sem.Release() |> ignore
                match cache.TryGetValue x with
                | (_, Completed y) -> return y
                | _ -> return failwith "computation aborted"
        | Completed y -> return y
    }
```

We use a custom state to mark that a value is being computed. The [SemaphoreSlim](https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim?view=net-8.0) synchronization primitive is used to lock other threads when one thread is computing the value. By using `sem.Wait(0)`, we branch to either computing the value or waiting for another thread to compute the value. When the value is computed, we replace the value in the cache with the resulting value. We still use `ConcurrentDictionary`, but this time only to ensure that we create only one `SemaphoreSlim` for each key. The `try ... finally` block in the compute section ensures that we don't create a deadlock if the `async` operation throws an exception.

<div data-node-type="callout">
<div data-node-type="callout-emoji">🤔</div>
<div data-node-type="callout-text">I'm not 100% sure there is no simpler alternative to this. If you know a better solution, let me know in the comments!</div>
</div>

## Memoize with Timed Validity

Not directly related to asynchronous memoization, but a useful `memoize` variant, especially when combining memoization with async/task, is `memoize` with timed validity.

```fsharp
let rec memoizeTimed invalidateTime f =
    let cache = System.Collections.Concurrent.ConcurrentDictionary()
    let fWithTime x =
        let y = f x
        let time = System.DateTime.Now
        y, time
    fun x -> 
        let y, time = cache.GetOrAdd(x, lazy fWithTime x).Value
        if System.DateTime.Now - time > invalidateTime then
            cache.TryRemove(x) |> ignore
            memoizeTimed invalidateTime f x
        else y
```

This variant uses the cached value only if it is younger than `invalidateTime`; otherwise, it recomputes it. This is useful for scenarios like "query the database only once per minute".