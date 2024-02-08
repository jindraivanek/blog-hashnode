---
title: "F# tips weekly #5: Discriminated Union Types"
datePublished: Thu Feb 08 2024 07:00:38 GMT+0000 (Coordinated Universal Time)
cuid: clscvddx6000109jw63nc5cyz
slug: f-tips-weekly-5-discriminated-union-types
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1707257670552/fd22fdcc-8a42-45c4-9c81-a8fb0e439c04.jpeg
tags: fsharp, discriminated-union-types

---

This week, we'll look at some interesting details about Discriminated Union types.

### Tuple vs. Multiple Fields

There is a subtle difference between these two cases of Discriminated Union (DU):

```fsharp
type DU = 
    | A of int * string
    | B of (int * string)
```

In the first case, there is a single case with two fields, while in the second case, there is a single case with a single field that is a tuple of two elements. In most cases, the difference is not apparent, but it can be surprising at times.

```fsharp
match a with
| A (x, _) -> x
| B t -> fst t
```

In pattern matching, for the `B` case, we can extract the value as a tuple and then use it. This is not possible with the `A` case, as the compiler enforces the deconstruction of all fields separately. This can be a good thing but not always.

### Named Fields

We can assign names to the arguments of DU cases, which can enhance readability.

```fsharp
type Shape =
    | Rectangle of width: float * length: float
    | Circle of radius: float
```

When creating a value, we can use named arguments:

```fsharp
let r = Rectangle(length=1.0, width=2.0)
```

Named arguments allow us to change the order of arguments, or we can use them for documentation purposes. Named arguments can only be used for the non-tupled form of DU cases.

### Case Constructor Shadowing

When we define a DU, a case constructor function is created for all cases. With the Shape example above, we have also created `Rectangle` and `Circle` functions. We can take advantage of F#'s ability to allow shadowing and redefine the function:

```fsharp
let Rectangle x = Rectangle(1.0, x)
```

Note that this only changes the constructor function, not the case and its usage in pattern matching.

<div data-node-type="callout">
<div data-node-type="callout-emoji">📆</div>
<div data-node-type="callout-text">We will cover shadowing in more detail in the future.</div>
</div>

### RequireQualifiedAccess Attribute

If we want to disallow creating case constructor functions in the same module as the type, we can use the `[<RequireQualifiedAccess>]` attribute. This will place case constructors and case patterns in the module with the DU name. For example,

```fsharp
[<RequireQualifiedAccess>]
type Shape =
    | Rectangle of width: float * length: float
    | Circle of radius: float
```

We can't use `Rectangle` and `Circle` as functions, but we must use `Shape.Rectangle` and `Shape.Circle`. Additionally, we can't use `Rectangle` and `Circle` in pattern matching; instead, we must use `Shape.Rectangle` and `Shape.Circle`. This is useful when we want to avoid name clashes with cases having the same name.

Interestingly, we can still use the shadowing trick; we just need to use a module:

```fsharp
module Shape =
    let Rectangle x = Shape.Rectangle(1.0, x)
```