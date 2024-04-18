---
title: "F# tips weekly #13: Operators"
datePublished: Thu Apr 18 2024 05:00:35 GMT+0000 (Coordinated Universal Time)
cuid: clv4rwn2n000g08jvf39f8t9h
slug: f-tips-weekly-13-operators
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1713367247086/b651e10d-1d73-495f-a3bf-8b51a00eff7a.jpeg
tags: tips, f, fsharp

---

Using operators like `=`, `+`, `&&` or `|>` is daily bread of writing F#. Let's look at how they work under the hood.

## Operator is a function

Every operator can be used as standard function by enclosing it in parentheses, for example

```fsharp
(+) 1 2
```

is the same as

```fsharp
1 + 2
```

This can be used with currying in place of lambda function. For example, we can write **product** function like this:

```fsharp
let product xs = xs |> Seq.reduce ((*))
```

## Operator methods

During compilation, operator is replaced with their name - `(+)` is changed to `op_Addition`. In fact, using named form of operator is valid F# code:

```fsharp
op_Addition 1 2
```

Full list of operator names can be found in [F# specification](https://fsharp.org/specs/language-spec/4.1/FSharpSpec-4.1-latest.pdf) on page 34.

## Unary operators

Alongside the classic binary operators, that works on two arguments, there are also unary operators. These operators are prefix (are used before its argument). One example is `~~~`, operator for bitwise negation.

The most common usage of unary operator is unary variant of `-`. When `-` is used before number without space, it is considered unary operator and gets translated into `(~-)`, so compiler can distinguish between unary and binary variant.

That's the reason why missed space after `-` leads to compiler error:

```fsharp
1 -2
```

`-` there is applied as unary operator to `2`.

This behavior is specific only to few specific operators, namely `+`, `-`, `%`, `%%`, `&`, `&&`.

## Custom operators

We can define new operators using `let (op) = ...` syntax. For example

```fsharp
let (=~) x y = abs (x - y) < 1e-6
```

defines `=~` operator for comparing *floats* with ignoring rounding errors.

Note that this operator is defined on floats only. Defining generic operators that work on multiple types is possible, but it is complicated and discouraged, because it can lead to puzzling compiler errors.

New operators are also translated into named form, new name is based on character used, so in our case we get `op_EqualsTwiddle`.

## Redefine operators

We can redefine existing operator by defining a new operator. For example:

```fsharp
let (+) a b = (a + b) % 7
```

Note that this way we redefined `+` to work only on *int*s, `+` on *float*s ceases to work.

## Operators as static members

Another way is to (re)define operator on custom type as `static member`:

```fsharp
type Vector2D =
    {
        X: float
        Y: float
    }

    static member (+) (a: Vector2D, b: Vector2D) = { X = a.X + b.X; Y = a.Y + b.Y }
```

In this case it is quite easy to write generic operator:

```fsharp
type Vector2D =
    {
        X: float
        Y: float
    }

    static member (+) (a: Vector2D, b: Vector2D) = { X = a.X + b.X; Y = a.Y + b.Y }
    static member (*) (a: Vector2D, b: Vector2D) = { X = a.X * b.X; Y = a.Y * b.Y }
    static member (*) (a: Vector2D, b: float) = { X = a.X * b; Y = a.Y * b }
```

`static member` operator methods actually works as extension of operators.