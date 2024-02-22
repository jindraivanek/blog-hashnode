---
title: "# F# tips weekly #7: Custom equality and comparison (1)"
datePublished: Thu Feb 22 2024 06:00:56 GMT+0000 (Coordinated Universal Time)
cuid: clswtejn1000a09l47kfchbhb
slug: f-tips-weekly-7-custom-equality-and-comparison-1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1708358831754/6bd0ec95-4189-4851-b9f2-0a13871df9a4.jpeg
tags: tips, fsharp

---

[Last week](https://jindraivanek.hashnode.dev/f-tips-weekly-6-structural-equality-and-comparison), we explored structural equality and comparison. This week, we'll look how structural equality and comparison is implemented, and how we can alter default behavior, or define custom comparisons and equality for our types.

### `Equals`, `GetHashCode`, and `CompareTo`

Every .NET object has a method `Equals` and a method `GetHashCode`. The `Equals` method compares two objects for equality, while the `GetHashCode` method generates a hash code for the object, crucial for hash tables and dictionaries.

When we create a new F# type, the `Equals` and `GetHashCode` methods implementing structural equality are automatically generated for us. These methods are defined for every .NET object, and what is generated overrides these methods. Similarly, the implementation of the `IComparable` interface through the `CompareTo` method is generated for structural comparison. However, we can opt-out from this behavior in various ways.

### No Comparison

By adding the `NoComparison` attribute to a type, we can opt-out from generating the `CompareTo` method. Attempting to use comparison then results in a compile-time error. This can be useful when we want to prevent the comparison of types, where comparison is actually a code smell.

```fsharp
[<NoComparison>]
type UserId = UserId of int

UserId 1 < UserId 2 // error FS0001: The type 'UserId' does not support the 'comparison' constraint
```

### No Equality

Additionally, we can also declare that our type doesn't support equality at all:

```fsharp
[<NoEquality>]
type UserId = UserId of int

UserId 1 = UserId 2 // error FS0001: The type 'UserId' does not support the 'equality' constraint because it has the 'NoEquality' attribute
```

Note that `NoEquality` also implies `NoComparison`.

### Reference Equality

We can opt-out from structural equality and comparison, and use reference equality instead. This can be useful when structural equality or comparison impacts performance, and reference equality is sufficient. However, we should be careful with this, as it can lead to subtle bugs.

```fsharp
[<ReferenceEquality>]
type SomeRecord = { BigList: int list }
```

One good use case for `ReferenceEquality` is when we have a static set of records and need quick lookup. Then we can use a C# `Dictionary` for these records, and because we have static set, reference equality is good enough.

### Custom Equality

By providing the `[<CustomEquality>]` attribute, we can define custom equality for our types. We need to provide our implementation of the `Equals` and `GetHashCode` methods. This way, we can provide a quicker implementation of equality, or we can ignore some fields.

```fsharp
[<CustomEquality; NoComparison>]
type Record = { Id: int; Name: string; ActivityLog: string list }
with    
    override x.Equals y =
        match y with
        | :? Record as y -> {| Id = x.Id; Name = x.Name |} = {| Id = y.Id; Name = y.Name |}
        | _ -> false
    override x.GetHashCode() =
        hash {| Id = x.Id; Name = x.Name |}
```

A few notes:

* When using `CustomEquality`, we also need to choose whether to provide custom comparison (`CustomComparison`) or not support comparison (`NoComparison`).
    
* While `GetHashCode` is not enforced, omitting it yields a warning: `The struct, record, or union type 'Record' has an explicit implementation of 'Object.Equals'. Consider implementing a matching override for 'Object.GetHashCode()'`. Implementing `GetHashCode` is crucial for C# Dictionary and similar collections to function correctly. Learn more about hash code on [wiki](https://en.wikipedia.org/wiki/Hash_function).
    
* The general `hash` function can be used to easily implement `GetHashCode`.
    

### Custom Comparison

In addition to providing custom equality, we can also implement custom comparison. We need to do this if we want to use our type in immutable collections like `Set` or `Map`.

```fsharp
[<CustomEquality; CustomComparison>]
type Record = { Id: int; Name: string; ActivityLog: string list }
with    
    member private x.Compare y =
        compare {| Id = x.Id; Name = x.Name |} {| Id = y.Id; Name = y.Name |}
    override x.Equals y =
        match y with
        | :? Record as y -> x.Compare y = 0
        | _ -> false
    override x.GetHashCode() =
         hash {| Id = x.Id; Name = x.Name |}
    
    interface System.IComparable with
        member x.CompareTo y =
            match y with
            | :? Record as y -> x.Compare y
            | _ -> -1
```

Here, we reuse one custom `Compare` method. Similarly, as with `hash`, we can use the general `compare` function to simplify things.

<div data-node-type="callout">
<div data-node-type="callout-emoji">📅</div>
<div data-node-type="callout-text">Next week, we will explore writing custom comparisons in more detail and examine various scenarios.</div>
</div>