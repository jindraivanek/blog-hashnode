---
title: "F# tips weekly #1: Single case DU"
datePublished: Thu Jan 11 2024 06:50:21 GMT+0000 (Coordinated Universal Time)
cuid: clr8uob3y00060aie2ge3h0a6
slug: f-tips-weekly-1-single-case-du
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1704897105207/db401b1a-d6ed-4760-9dca-03514dbcb056.jpeg
tags: tips, dotnet, fsharp, discriminated-union-types

---

Single-case discriminated unions are a great way to avoid *primitive obsession*.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><strong>Primitive obsession</strong> is an anti-pattern where primitive types are used too much in places where a custom type would be a better fit. By using custom types or wrapper types, we can catch more potential bugs.</div>
</div>

We can create a new type that simply wraps the value of another (typically primitive) type. In F#, this is a simple one-liner:

```fsharp
type ItemId = ItemId of int
```

By using <kbd>ItemId</kbd> instead of <kbd>int</kbd>, we can ensure that it is used only in places where it is explicitly requested. Also, using this type is self-documenting.

For example, this function:

```fsharp
let getItem (x: ItemId) =
    ...
```

can't be called with a different type (let's say `CustomerId`) - preventing mistakes that could happen if we used just `int`.

---

When we need to access the value inside the type, we can take advantage of the fact that pattern matching can be used on the left-hand side of <kbd>let</kbd>:

```fsharp
let itemId = ItemId 42
let (ItemId value) = itemId
...
```

And in function parameters:

```fsharp
let getItem (ItemId value) =
	...
```

And also in lambda functions:

```fsharp
[ ItemId 1; ItemId 2; ItemId 3 ] |> List.map (fun (ItemId value) -> value)
```

But sometimes using pattern matching can be cumbersome because we must create a new binding. We can improve this by defining a helper member on the DU type:

```fsharp
type ItemId = ItemId of int
with member this.Value = let (ItemId value) = this in value
```

Then we can use <kbd>itemId.Value</kbd>.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">The <code>in</code> keyword is shorthand syntax for writing <code>let</code> binding and following expression in one line. Learn more on <a target="_blank" rel="noopener noreferrer nofollow" href="https://fsharpforfunandprofit.com/posts/let-use-do/#nested-let-bindings-as-expressions" style="pointer-events: none">fsharpforfunandprofit</a>.</div>
</div>

```plaintext
member this.Value = 
    let (ItemId value) = this
    value
```

---

Another advantage is that we can restrict the usage of our type. For example, we can disallow comparing values (because for IDs it doesn't make sense):

```fsharp
[<NoComparison>]
type ItemId = ItemId of int

(ItemId 1) < (ItemId 2)
```

This will generate an error: `The type 'ItemId' does not support the 'comparison' constraint because it has the 'NoComparison' attribute1`.