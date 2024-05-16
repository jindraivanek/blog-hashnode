---
title: "Problem with NaN equality"
datePublished: Thu May 16 2024 10:12:00 GMT+0000 (Coordinated Universal Time)
cuid: clw93czbp000909me3t9j6zsh
slug: problem-with-nan-equality
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715853965966/c93f9cf7-f101-4804-8030-b00288707810.jpeg
tags: bug, net, nan, fsharp

---

I recently encountered a bug that was caused by a special equality definition on a *NaN* value. *NaN* means not-a-number, and it's special floating-point number value, representing result of impossible operation. This issue shows how *NaN* can introduce hard-to-find bugs.

## The problem

One of our external services started to return *NaN* values. After some investigation, it turns out that the responsible part is the following part of the code:

```fsharp
let missingToOption missing value =
    if value = missing then None
    else Some value
```

This seems perfectly valid. We take a value, and if it is equal to some specific `missing` value, it returns `None`. We can think about it as the inverse to `Option.defaultValue`.

But, there is a catch when applied to `NaN`. When `missing` and `value` are *NaN*s, this function returns `Some(nan)`. (`nan` here is an alias to `System.Double.NaN`, defined in the F# standard library). It's because equality for *NaN* is defined as `nan = nan` equals `false` and `nan <> nan` equals `true`. Actually, the `<>` operator is the only comparison operator that returns `true` when one of its operands is *NaN*. For details, check the wiki page on *NaN*.

In our case, the *NaN* value appeared from an external service, where some configuration for the missing value representation had a value of *NaN*. This is easy to miss because the *NaN* value wasn't explicitly mentioned to start with.

When we know that *NaN* is the problem, the solution is easy:

```fsharp
let missingToOption missing value =
    if value = missing then None
    elif Double.IsNaN missing && Double.IsNaN value then None
    else Some value
```

## NaN origin

Why is there this strange *NaN* value anyway? *NaN* means not-a-number, and it's defined as the result of undefined mathematical operations, like dividing by zero, taking the square root of negative numbers, and some others. Practice shows that raising an exception or another way of signaling an invalid operation would be better, but that's quite expensive not only because raising an exception is expensive, but maybe more importantly, it would mean adding a check to every affected operation. By introducing one special value, the division operator can remain fast, and still, we have a signal that some calculation was invalid.

*NaN* behavior is defined in the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754#NaNs) standard.

*NaN* was designed to be viral - every operation returning a `float` type returns `nan` if one or both of its operands are `nan`. This way *NaN* "bubbles up" through almost every calculation. That's quite reasonable, but what about comparisons?

Probably the main reason why `nan <> nan` was that *NaN* should be incomparable with every number as a comparison doesn't make sense. But it has the unfortunate effect of breaking the rules of comparisons, for example, that exactly one of `x < y` or `x >= y` must be `true`.

## Conclusion

Whatever the real reasons were, this behavior of *NaN* is present in probably every language. When working with floats, it's important to keep in mind that special values - *NaN* or *infinity* - can appear, and their specifics.