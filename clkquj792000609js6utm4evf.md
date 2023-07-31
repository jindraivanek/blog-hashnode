---
title: "Curious case of List.contains performance"
datePublished: Mon Jul 31 2023 12:28:17 GMT+0000 (Coordinated Universal Time)
cuid: clkquj792000609js6utm4evf
slug: curious-case-of-listcontains-performance
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690790755399/48fcdf86-7253-4891-8503-06ea6651d9ce.png
tags: performance, compiler, fsharp

---

Some time ago, colleagues stumbled upon a case where `List.contains` was significantly slower than identical use of `List.exists`.

[`List.contains`](https://fsharp.github.io/fsharp-core-docs/reference/fsharp-collections-listmodule.html#contains) and [`List.exists`](https://fsharp.github.io/fsharp-core-docs/reference/fsharp-collections-listmodule.html#exists) are functions from F# standard library, where

* `List.contains x xs` check if the list `xs` contains element `x`
    
* `List.exists predicate xs` check if there is an element in the list `xs` that satisfy `predicate` condition.
    

`List.contains` being slower than `List.exists` is weird because it is a specific case of `List.exists`, and can be rephrased as `List.contains x xs = List.exists (fun y -> x = y)` xs.

Let's look at this problem in more detail.

## Benchmarking

Comparing `exists` and `contains` function on lists of `int`, `string` and `record` with .NET Benchmark library we get:

| Method | Mean | Error | StdDev | Gen0 | Allocated |
| --- | --- | --- | --- | --- | --- |
| 'int - List.exists' | 1.482 ms | 0.0490 ms | 0.1413 ms | 1.9531 | 48042 B |
| 'int - List.contains' | 20.230 ms | 0.4809 ms | 1.4105 ms | 1906.2500 | 24024059 B |
| 'string - List.exists' | 2.230 ms | 0.0290 ms | 0.0271 ms | \- | 48042 B |
| 'string - List.contains' | 4.269 ms | 0.1183 ms | 0.3413 ms | \- | 45 B |
| 'record - List.exists' | 1.208 ms | 0.0237 ms | 0.0308 ms | 1.9531 | 48041 B |
| 'record - List.contains' | 6.388 ms | 0.1276 ms | 0.1949 ms | \- | 45 B |

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Benchmark run function for each item of list of size 1000. The link for code for all benchmarks is at the bottom of the article.</div>
</div>

We see that `exists` is faster than `contains` for all types. For `string` it is around 2 times faster, for `record` it is around 5 times faster, and for `int` it is around 14 times faster.

## Decompiled code

One way to understand why `exists` is faster than `contains` is to look at the compiled code. To do that we can use a great tool called [SharpLab](https://sharplab.io/). This tool compiles given code in F# or C# to [CIL](https://en.wikipedia.org/wiki/Common_Intermediate_Language), and allows us to look at the decompiled version of CIL in C#. Let's look at a simple example for `exists` and `contains` for `int`:

### `List.contains` function:

```fsharp
[1 .. 1000] |> List.contains 1
```

decompiled CIL code is as follows

```csharp
using System.Diagnostics;
using System.Reflection;
using System.Runtime.CompilerServices;
using <StartupCode$_>;
using Microsoft.FSharp.Collections;
using Microsoft.FSharp.Core;

[assembly: FSharpInterfaceDataVersion(2, 0, 0)]
[assembly: AssemblyVersion("0.0.0.0")]

[CompilationMapping(SourceConstructFlags.Module)]
public static class @_
{
    [CompilationMapping(SourceConstructFlags.Value)]
    internal static FSharpList<int> arg@1
    {
        get
        {
            return $_.arg@1;
        }
    }

    [CompilationMapping(SourceConstructFlags.Value)]
    internal static FSharpList<int> source@1
    {
        get
        {
            return $_.source@1;
        }
    }

    [CompilerGenerated]
    internal static bool contains@1<a>(a e, FSharpList<a> xs1)
    {
        while (true)
        {
            if (xs1.TailOrNull == null)
            {
                return false;
            }
            FSharpList<a> fSharpList = xs1;
            FSharpList<a> tailOrNull = fSharpList.TailOrNull;
            a headOrDefault = fSharpList.HeadOrDefault;
            if (LanguagePrimitives.HashCompare.GenericEqualityIntrinsic(e, headOrDefault))
            {
                break;
            }
            a val = e;
            xs1 = tailOrNull;
            e = val;
        }
        return true;
    }
}

namespace <StartupCode$_>
{
    internal static class $_
    {
        [DebuggerBrowsable(DebuggerBrowsableState.Never)]
        internal static readonly FSharpList<int> arg@1;

        [DebuggerBrowsable(DebuggerBrowsableState.Never)]
        internal static readonly FSharpList<int> source@1;

        [DebuggerBrowsable(DebuggerBrowsableState.Never)]
        [CompilerGenerated]
        [DebuggerNonUserCode]
        internal static int init@;

        static $_()
        {
            arg@1 = SeqModule.ToList(Operators.CreateSequence(Operators.OperatorIntrinsics.RangeInt32(1, 1, 1000)));
            source@1 = @_.arg@1;
            @_.contains@1(1, @_.source@1);
        }
    }
}
```

The important thing here is the call of `LanguagePrimitives.HashCompare.GenericEqualityIntrinsic` function. That's a function for comparing two values, regardless of their type. Because that function is generic, it contains runtime type checks and it is slower than `=` operator applied on two `int`s.

Implementation of `GenericEqualityIntrinsic` can be found here: [https://github.com/dotnet/fsharp/blob/b4c26d84bc58a8d3abb2e7648ed7cc1e1cd7e25c/src/FSharp.Core/prim-types.fs#L1558](https://github.com/dotnet/fsharp/blob/b4c26d84bc58a8d3abb2e7648ed7cc1e1cd7e25c/src/FSharp.Core/prim-types.fs#L1558)

which just call `GenericEqualityObj` [https://github.com/dotnet/fsharp/blob/b4c26d84bc58a8d3abb2e7648ed7cc1e1cd7e25c/src/FSharp.Core/prim-types.fs#L1436](https://github.com/dotnet/fsharp/blob/b4c26d84bc58a8d3abb2e7648ed7cc1e1cd7e25c/src/FSharp.Core/prim-types.fs#L1436)

### `List.exists` function:

```fsharp
[1 .. 1000] |> List.exists (fun x -> x = 1)
```

decompiled CIL code is as follows

```csharp
using System;
using System.Diagnostics;
using System.Reflection;
using System.Runtime.CompilerServices;
using Microsoft.FSharp.Collections;
using Microsoft.FSharp.Core;

[assembly: FSharpInterfaceDataVersion(2, 0, 0)]
[assembly: AssemblyVersion("0.0.0.0")]

[CompilationMapping(SourceConstructFlags.Module)]
public static class @_
{
    [Serializable]
    internal sealed class clo@1 : FSharpFunc<int, bool>
    {
        internal static readonly clo@1 @_instance = new clo@1();

        [CompilerGenerated]
        [DebuggerNonUserCode]
        internal clo@1()
        {
        }

        public override bool Invoke(int x)
        {
            return x == 1;
        }
    }
}

namespace <StartupCode$_>
{
    internal static class $_
    {
        [DebuggerBrowsable(DebuggerBrowsableState.Never)]
        [CompilerGenerated]
        [DebuggerNonUserCode]
        internal static int init@;

        static $_()
        {
            ListModule.Exists(@_.clo@1.@_instance, SeqModule.ToList(Operators.CreateSequence(Operators.OperatorIntrinsics.RangeInt32(1, 1, 1000))));
        }
    }
}
```

We see that `ListModule.Exists` function uses `==` to compare two `int`s. That's because `=` operator is inside lambda function (here compiled as `@_.clo@1.@_instance`), and the compiler knows it's used on `int`s.

## Better List.contains implementation

We saw that `List.exists` works better than `List.contains`. How about we just redefine `List.contains` to use `List.exists`?

```fsharp
module List =
    let containsByExists value source =
        List.exists (fun x -> x = value) source
```

When we run benchmarks with this implementation we get:

| Method | Mean | Error | StdDev | Gen0 | Allocated |
| --- | --- | --- | --- | --- | --- |
| 'int - List.exists' | 1.482 ms | 0.0490 ms | 0.1413 ms | 1.9531 | 48042 B |
| 'int - List.contains' | 20.230 ms | 0.4809 ms | 1.4105 ms | 1906.2500 | 24024059 B |
| 'int - List.containsByExists' | 19.888 ms | 0.4198 ms | 1.1977 ms | 1906.2500 | 24048059 B |
| 'string - List.exists' | 2.230 ms | 0.0290 ms | 0.0271 ms | \- | 48042 B |
| 'string - List.contains' | 4.269 ms | 0.1183 ms | 0.3413 ms | \- | 45 B |
| 'string - List.containsByExists' | 5.130 ms | 0.1024 ms | 0.2837 ms | \- | 24045 B |
| 'record - List.exists' | 1.208 ms | 0.0237 ms | 0.0308 ms | 1.9531 | 48041 B |
| 'record - List.contains' | 6.388 ms | 0.1276 ms | 0.1949 ms | \- | 45 B |
| 'record - List.containsByExists' | 6.400 ms | 0.1240 ms | 0.1099 ms | \- | 48045 B |

Not much better.

If we return to SharpLab, we can see that `LanguagePrimitives.HashCompare.GenericEqualityIntrinsic` is still used:

```csharp
using System;
using System.Diagnostics;
using System.Reflection;
using System.Runtime.CompilerServices;
using Microsoft.FSharp.Collections;
using Microsoft.FSharp.Core;

[assembly: FSharpInterfaceDataVersion(2, 0, 0)]
[assembly: AssemblyVersion("0.0.0.0")]

[CompilationMapping(SourceConstructFlags.Module)]
public static class @_
{
    [CompilationMapping(SourceConstructFlags.Module)]
    public static class List
    {
        [Serializable]
        internal sealed class contains@3<a> : FSharpFunc<a, bool>
        {
            public a value;

            [CompilerGenerated]
            [DebuggerNonUserCode]
            internal contains@3(a value)
            {
                this.value = value;
            }

            public override bool Invoke(a x)
            {
                return LanguagePrimitives.HashCompare.GenericEqualityIntrinsic(x, value);
            }
        }

        [CompilationArgumentCounts(new int[] { 1, 1 })]
        public static bool contains<a>(a value, FSharpList<a> source)
        {
            return ListModule.Exists(new contains@3<a>(value), source);
        }
    }
}

namespace <StartupCode$_>
{
    internal static class $_
    {
        [DebuggerBrowsable(DebuggerBrowsableState.Never)]
        [CompilerGenerated]
        [DebuggerNonUserCode]
        internal static int init@;

        static $_()
        {
            @_.List.contains(1, SeqModule.ToList(Operators.CreateSequence(Operators.OperatorIntrinsics.RangeInt32(1, 1, 1000))));
        }
    }
}
```

Why? The reason is that the type of `=` operator in `containsByExists` is not known at the time of compilation. That's different from `exists` where the type of `=` operator is used outside of a function, in lambda parameter.

But there is a solution, we can use inlined function:

```fsharp
module List =
    let inline containsByExistsInline value source =
        List.exists (fun x -> x = value) source
```

When we run benchmarks with this implementation we get:

| Method | Mean | Error | StdDev | Gen0 | Allocated |
| --- | --- | --- | --- | --- | --- |
| 'int - List.exists' | 1.482 ms | 0.0490 ms | 0.1413 ms | 1.9531 | 48042 B |
| 'int - List.contains' | 20.230 ms | 0.4809 ms | 1.4105 ms | 1906.2500 | 24024059 B |
| 'int - List.containsByExists' | 19.888 ms | 0.4198 ms | 1.1977 ms | 1906.2500 | 24048059 B |
| 'int - List.containsByExistsInline' | 1.164 ms | 0.0230 ms | 0.0519 ms | \- | 24041 B |
| 'string - List.exists' | 2.230 ms | 0.0290 ms | 0.0271 ms | \- | 48042 B |
| 'string - List.contains' | 4.269 ms | 0.1183 ms | 0.3413 ms | \- | 45 B |
| 'string - List.containsByExists' | 5.130 ms | 0.1024 ms | 0.2837 ms | \- | 24045 B |
| 'string - List.containsByExistsInline' | 2.089 ms | 0.0373 ms | 0.0602 ms | \- | 24042 B |
| 'record - List.exists' | 1.208 ms | 0.0237 ms | 0.0308 ms | 1.9531 | 48041 B |
| 'record - List.contains' | 6.388 ms | 0.1276 ms | 0.1949 ms | \- | 45 B |
| 'record - List.containsByExists' | 6.400 ms | 0.1240 ms | 0.1099 ms | \- | 48045 B |
| 'record - List.containsByExistsInline' | 1.573 ms | 0.0312 ms | 0.0630 ms | \- | 24041 B |

Yay! Much better. Now `List.containsByExistsInline` have comparable performance to `List.exists`.

## Other collection types

Interestingly, this problem is not present in other collection types. It's probably because the implementation of `exists` and `contains` functions is different.

### Array

| Method | Mean | Error | StdDev | Gen0 | Allocated |
| --- | --- | --- | --- | --- | --- |
| 'int - Array.exists' | 680.7 us | 9.28 us | 8.22 us | \- | 33 B |
| 'int - Array.contains' | 678.8 us | 4.71 us | 4.18 us | \- | 33 B |
| 'int - Array.containsByExists' | 25,871.9 us | 516.35 us | 671.40 us | 1906.2500 | 24024051 B |
| 'int - Array.containsByExistsInline' | 674.6 us | 9.77 us | 9.14 us | \- | 33 B |
| 'string - Array.exists' | 2,601.8 us | 48.39 us | 69.41 us | \- | 34 B |
| 'string - Array.contains' | 2,531.9 us | 50.01 us | 53.51 us | \- | 34 B |
| 'string - Array.containsByExists' | 6,809.1 us | 96.07 us | 85.16 us | \- | 37 B |
| 'string - Array.containsByExistsInline' | 2,508.0 us | 47.43 us | 44.37 us | \- | 34 B |
| 'record - Array.exists' | 788.1 us | 12.45 us | 11.03 us | \- | 33 B |
| 'record - Array.contains' | 806.3 us | 15.28 us | 16.34 us | \- | 33 B |
| 'record - Array.containsByExists' | 10,248.9 us | 196.80 us | 210.57 us | \- | 24041 B |
| 'record - Array.containsByExistsInline' | 802.7 us | 15.17 us | 14.19 us | \- | 33 B |

### Seq

| Method | Mean | Error | StdDev | Gen0 | Allocated |
| --- | --- | --- | --- | --- | --- |
| 'int - Seq.exists' | 3.937 ms | 0.0760 ms | 0.0961 ms | 7.8125 | 101.62 KB |
| 'int - Seq.contains' | 2.885 ms | 0.0559 ms | 0.0574 ms | 3.9063 | 54.74 KB |
| 'int - Seq.containsByExists' | 29.364 ms | 0.4812 ms | 0.4501 ms | 1906.2500 | 23539.14 KB |
| 'int - Seq.containsByExistsInline' | 3.682 ms | 0.0726 ms | 0.0864 ms | \- | 78.18 KB |
| 'string - Seq.exists' | 20.745 ms | 0.2531 ms | 0.2114 ms | 1250.0000 | 15540.03 KB |
| 'string - Seq.contains' | 21.457 ms | 0.3560 ms | 0.3330 ms | 1250.0000 | 15493.15 KB |
| 'string - Seq.containsByExists' | 25.581 ms | 0.4793 ms | 0.4484 ms | 1250.0000 | 15516.59 KB |
| 'string - Seq.containsByExistsInline' | 21.232 ms | 0.3437 ms | 0.3678 ms | 1250.0000 | 15516.59 KB |
| 'record - Seq.exists' | 23.734 ms | 0.2825 ms | 0.2643 ms | 2531.2500 | 31211.9 KB |
| 'record - Seq.contains' | 22.695 ms | 0.2858 ms | 0.2534 ms | 2531.2500 | 31165.03 KB |
| 'record - Seq.containsByExists' | 33.592 ms | 0.6649 ms | 0.6530 ms | 2500.0000 | 31211.92 KB |
| 'record - Seq.containsByExistsInline' | 23.483 ms | 0.4104 ms | 0.3638 ms | 2531.2500 | 31188.46 KB |

## Impementation

Implementation of `Array.contains` in F# compiler source code:

```fsharp
let inline contains value (array: 'T[]) =
    checkNonNull "array" array
    let mutable state = false
    let mutable i = 0

    while not state && i < array.Length do
        state <- value = array.[i]
        i <- i + 1

    state
```

[https://github.com/dotnet/fsharp/blob/df3919d6940ca3ab63fa8f8fe7b1a511cf4cc96d/src/FSharp.Core/array.fs#L485](https://github.com/dotnet/fsharp/blob/df3919d6940ca3ab63fa8f8fe7b1a511cf4cc96d/src/FSharp.Core/array.fs#L485)

Implementation of `Seq.contains` in F# compiler source code:

```fsharp
let inline contains value (source: seq<'T>) =
    checkNonNull "source" source
    use e = source.GetEnumerator()
    let mutable state = false

    while (not state && e.MoveNext()) do
        state <- value = e.Current

    state
```

[https://github.com/dotnet/fsharp/blob/97c2d3784e907862bf4266cb0ffd8d2788e0a54b/src/FSharp.Core/seq.fs#L681](https://github.com/dotnet/fsharp/blob/97c2d3784e907862bf4266cb0ffd8d2788e0a54b/src/FSharp.Core/seq.fs#L681)

Both implementations use explicit mutation and `while` cycle. Inlining of `=` operator works well here. Problem with `List.contains` is probably that it uses recursion, and `=` operator is not inlined.

## Better solution?

Can we use a similar implementation for `List.contains`? Let's try:

```fsharp
let inline containsMutable value (source: list<_>) =
    let mutable xs = source
    let mutable state = false

    while (not state && not xs.IsEmpty) do
        state <- value = xs.Head
        xs <- xs.Tail

    state
```

Benchmark results:

| Method | Mean | Error | StdDev | Gen0 | Allocated |
| --- | --- | --- | --- | --- | --- |
| 'int - List.exists' | 1.865 ms | 0.0275 ms | 0.0257 ms | \- | 48042 B |
| 'int - List.contains' | 25.762 ms | 0.1914 ms | 0.1696 ms | 1906.2500 | 24024059 B |
| 'int - List.containsByExists' | 25.623 ms | 0.5022 ms | 0.5157 ms | 1906.2500 | 24048059 B |
| 'int - List.containsByExistsInline' | 1.562 ms | 0.0229 ms | 0.0214 ms | \- | 24041 B |
| 'int - List.containsMutable' | 2.461 ms | 0.0371 ms | 0.0310 ms | \- | 42 B |
| 'string - List.exists' | 3.330 ms | 0.0293 ms | 0.0274 ms | \- | 48042 B |
| 'string - List.contains' | 6.705 ms | 0.0284 ms | 0.0266 ms | \- | 45 B |
| 'string - List.containsByExists' | 8.022 ms | 0.0956 ms | 0.0894 ms | \- | 24049 B |
| 'string - List.containsByExistsInline' | 3.360 ms | 0.0516 ms | 0.0482 ms | \- | 24042 B |
| 'string - List.containsMutable' | 4.337 ms | 0.0760 ms | 0.0711 ms | \- | 45 B |
| 'record - List.exists' | 1.856 ms | 0.0191 ms | 0.0179 ms | 1.9531 | 48041 B |
| 'record - List.contains' | 9.894 ms | 0.1830 ms | 0.1712 ms | \- | 49 B |
| 'record - List.containsByExists' | 11.263 ms | 0.1205 ms | 0.1127 ms | \- | 48049 B |
| 'record - List.containsByExistsInline' | 2.369 ms | 0.0341 ms | 0.0319 ms | \- | 24042 B |
| 'record - List.containsMutable' | 2.878 ms | 0.0249 ms | 0.0233 ms | \- | 42 B |

It's better, but still not as good as `List.containsByExistsInline`.

Let's try something little different - use of `List.tryFind` function:

```fsharp
    let inline containsTryFind value (source: list<_>) =
        source |> List.tryFind (fun x -> value = x) |> Option.isSome
```

| Method | Mean | Error | StdDev | Gen0 | Allocated |
| --- | --- | --- | --- | --- | --- |
| 'int - List.exists' | 1.844 ms | 0.0235 ms | 0.0220 ms | 1.9531 | 48041 B |
| 'int - List.contains' | 25.150 ms | 0.4391 ms | 0.4108 ms | 1906.2500 | 24024059 B |
| 'int - List.containsByExists' | 27.099 ms | 0.3463 ms | 0.3239 ms | 1906.2500 | 24048059 B |
| 'int - List.containsByExistsInline' | 1.553 ms | 0.0171 ms | 0.0160 ms | \- | 24041 B |
| 'int - List.containsMutable' | 2.494 ms | 0.0471 ms | 0.0595 ms | \- | 42 B |
| 'int - List.containsTryFind' | 1.247 ms | 0.0245 ms | 0.0240 ms | 1.9531 | 48041 B |
| 'string - List.exists' | 3.396 ms | 0.0351 ms | 0.0311 ms | \- | 48042 B |
| 'string - List.contains' | 6.803 ms | 0.1287 ms | 0.1204 ms | \- | 45 B |
| 'string - List.containsByExists' | 8.119 ms | 0.1230 ms | 0.1151 ms | \- | 24049 B |
| 'string - List.containsByExistsInline' | 3.393 ms | 0.0451 ms | 0.0422 ms | \- | 24042 B |
| 'string - List.containsMutable' | 4.448 ms | 0.0770 ms | 0.0721 ms | \- | 45 B |
| 'string - List.containsTryFind' | 3.046 ms | 0.0602 ms | 0.0670 ms | \- | 48042 B |
| 'record - List.exists' | 1.928 ms | 0.0372 ms | 0.0382 ms | \- | 48042 B |
| 'record - List.contains' | 10.270 ms | 0.1622 ms | 0.1593 ms | \- | 49 B |
| 'record - List.containsByExists' | 11.283 ms | 0.1212 ms | 0.1133 ms | \- | 48049 B |
| 'record - List.containsByExistsInline' | 2.344 ms | 0.0235 ms | 0.0208 ms | \- | 24042 B |
| 'record - List.containsMutable' | 2.888 ms | 0.0455 ms | 0.0487 ms | \- | 42 B |
| 'record - List.containsTryFind' | 1.837 ms | 0.0360 ms | 0.0337 ms | 1.9531 | 48041 B |

This looks good!

`List.containsTryFind` is even slightly better than `List.containsByExistsInline.`

## Conclusion

We show that `List.contains` is much slower than `List.exists`. We analyzed the reason behind this, the usage of `GenericEqualityIntrinsic`, which is slower than inlined `=` operator for specific types. We present 2 possible faster implementations of `List.contains`, `List.containsByExistsInline` and `List.containsTryFind`, which is as fast as `List.exists` or better.

## Resources

Full benchmark code can be found in the [Benchmarks-list-contains repo](https://github.com/jindraivanek/Benchmarks-list-contains/blob/master/Benchmarks/Program.fs).