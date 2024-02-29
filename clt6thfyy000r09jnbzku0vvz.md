---
title: "F# tips weekly #8: Custom equality and comparison (2)"
datePublished: Thu Feb 29 2024 06:00:53 GMT+0000 (Coordinated Universal Time)
cuid: clt6thfyy000r09jnbzku0vvz
slug: f-tips-weekly-8-custom-equality-and-comparison-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1709141044076/9cc7fa01-0da5-4ee0-b053-d5a74e41afe8.jpeg
tags: tips, fsharp

---

This week, we continue with custom comparison. Let's deep dive into writing custom comparisons.

If we examine the last example of custom comparison:

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

We can simplify it a little more by creating a function for the anonymous record we are using for comparison:

```fsharp
[<CustomEquality; CustomComparison>]
type Record = { Id: int; Name: string; ActivityLog: string list }
with    
    member private x.CompareRepr = {| Id = x.Id; Name = x.Name |}
    member private x.Compare y =
        compare x.CompareRepr y.CompareRepr
    override x.Equals y =
        match y with
        | :? Record as y -> x.Compare y = 0
        | _ -> false
    override x.GetHashCode() =
         hash x.CompareRepr
    
    interface System.IComparable with
        member x.CompareTo y =
            match y with
            | :? Record as y -> x.Compare y
            | _ -> -1
```

This way, we can specify which fields we want to compare, easily allowing us to ignore some fields in both equality and comparison. We can reuse this template for any type where we want to define custom comparison. The only things we need to change are the `CompareRepr` method, the `Record` type annotation, and the two occurrences of `Record` type check.

🔎 The only reason why we need to do that `:? Record` type checking is that the compiler expects the `Equals` and `CompareTo` methods to accept parameters of type `obj`.

We can also use the same template to change what is compared for some fields - for example, easily implement structural comparison for some fields that don't support it:

```fsharp
[<CustomEquality; CustomComparison>]
type Record = { Id: int; Name: string; ActivityLog: System.Collections.Generic.List<string> }
with    
    member private x.CompareRepr = {| Id = x.Id; Name = x.Name; ActivityLog = x.ActivityLog |> Seq.toList |}
    member private x.Compare (y: Record) =
        compare x.CompareRepr y.CompareRepr
    override x.Equals y =
        match y with
        | :? Record as y -> x.Compare y = 0
        | _ -> false
    override x.GetHashCode() =
         hash x.CompareRepr
    
    interface System.IComparable with
        member x.CompareTo y =
            match y with
            | :? Record as y -> x.Compare y
            | _ -> -1
```

## Performance

This works fine but has a performance problem — we are creating new anonymous record objects for every comparison. That could be slow when sorting big collections of these objects. The best way to solve this is to write `Compare` and `GetHashCode` in a way that avoids new object allocations. To do this in a reusable way, we need to write some helper combinator functions.

### Compare function

`Comparison` in .NET works so that it returns `0` if items are equal, less than zero if the first item is smaller, and more than zero if the second item is smaller. The result of these functions cannot be easily combined out-of-the-box, but we can write a simple combinator function:

```fsharp
let inline compareBy f x y = compare (f x) (f y)
let inline compareComb g f x y =
    let r = f x y
    if r <> 0 then r else g x y
```

`compareBy` creates a compare function from a projection function, allowing us to create a compare function for one field. `compareComb` combines two compare functions together in a <abbr title="by switching order of f and g parameters">pipeline-friendly</abbr> way. By using `inline`, we avoid the performance cost of calling these functions.

We can rewrite `Compare` like this:

```fsharp
    member private x.Compare (y: Record) =
        let compareFun = 
            (compareBy <| fun x -> x.Id)
            |> compareComb (compareBy <| fun x -> x.Name) 
            |> compareComb (compareBy <| fun x -> Seq.toList x.ActivityLog)
        compareFun x y
```

### Hash function

Combining two hash codes is more complicated (as shown in [this](https://stackoverflow.com/a/27952689) StackOverflow answer), but thankfully .NET has a helper method [System.HashCode.Combine](https://learn.microsoft.com/en-us/dotnet/api/system.hashcode.combine?view=net-6.0#system-hashcode-combine-2(-0-1)):

```fsharp
let inline hashCombine y x = System.HashCode.Combine(hash x, hash y)
```

```fsharp
    override x.GetHashCode() =
        x.Id
        |> hashCombine x.Name
        |> hashCombine (x.ActivityLog |> Seq.toList)
```

### Conclusion

This way we can create efficient custom comparison by combining simple functions, without need to write hash or compare functions from scratch.