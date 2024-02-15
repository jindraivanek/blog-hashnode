---
title: "F# tips weekly #6: Structural equality and comparison"
datePublished: Thu Feb 15 2024 06:00:26 GMT+0000 (Coordinated Universal Time)
cuid: clsmtaxsb000509jkbveueg9p
slug: f-tips-weekly-6-structural-equality-and-comparison
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1707900333353/873430de-1a54-4ff2-b7ec-d058e35e1e84.jpeg
tags: immutable, fsharp

---

Core F# types, such as records, discriminated unions, and tuples, are compared by their structure, not by reference. That allows us to work efficiently with immutable data structures without surprises. Let's look at some details.

### Eligible types

Structural equality and comparison are implemented for types that are composed of:

* primitive types (`int`, `string`, etc.)
    
* unit (`()` - F# "empty" type)
    
* tuples
    
* records
    
* discriminated unions
    

By definition, structural equality and comparison are implemented recursively. It's quite evident how it works for equality -- we simply check all fields of type for equality, until we reach primitive types. For comparison, order of fields is important. This feature can be used to our advantage.

### Field ordering

The simplest example is tuples. They are compared by their elements in order. For example:

```fsharp
let sorted = items |> List.sortBy (fun x -> x.LastName, x.FirstName)
```

This is a very useful pattern, and often tuple can be used for advanced sorting scenarios, instead of creating custom types.

---

Similarly, records are compared by their fields in order of definition.

Discriminated unions are compared by the order of cases, then by the value of the case. The first case is considered the smallest, and the last case is the largest.

This allows us to implement, for example poker hand comparison just by defining cases in order of strength.

```fsharp
type PokerHand =
    | HighCard of int
    | Pair of int
    | TwoPair of int * int
    | ThreeOfAKind of int
    | Straight of int
    | Flush of int
    | FullHouse of int * int
    | FourOfAKind of int
    | StraightFlush of int
    | RoyalFlush
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Although this behavior is useful, it can be quite surprising for someone who is not familiar with this feature. For that reason, I would add comment to every type, where we rely on ordering of fields / cases.</div>
</div>

### `[<StructuralEquality>]` and `[<StructuralComparison>]` attributes

Sometimes, we need to include objects (typically from some .NET library) that don't support structural equality and comparison in our structures. These objects are compared by reference, which can sometimes came as a surprise. The compiler doesn't warn us about this by default, but we can opt-in to a compiler error if some field is not structurally comparable.

For example:

```fsharp
[<StructuralEquality; StructuralComparison>]
type MyType = { A: int; B: System.Collections.Generic.List<int> }
```

results in error: `The struct, record or union type 'MyType' has the 'StructuralComparison' attribute but the component type 'System.Collections.Generic.List<int>' does not satisfy the 'comparison' constraint`

This can help us to find non-structural types that can be hidden deep in type structure.

### .NET type that supports structural comparison

There are some .NET types outside F# that support structural comparison (not exhaustive list):

* every struct type, for example `System.DataTime`,
    
* C# [record](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records) types,
    
* some types implement it explicitly, for example `System.Tuple`, `System.Array` or `System.Collections.Immutable.ImmutableArray`.
    

Interestingly, no other type from `System.Collections.Immutable` namespace supports structural comparison.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">We will cover implementation details of structural equality and comparison, and how to implement custom equality or comparison next week.</div>
</div>
