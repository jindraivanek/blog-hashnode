---
title: "Infinity ♾ forecast bug"
datePublished: Thu Mar 14 2024 06:00:06 GMT+0000 (Coordinated Universal Time)
cuid: cltqtmcwc001409jufpvsfrqo
slug: infinity-forecast-bug
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1710164843629/178279da-d903-41d8-b657-2ca5f22a854e.jpeg
tags: csharp, f, dotnet, postmortem, fsharp

---

At one of our customer project at [Ciklum Western Europe](https://www.ciklum.com/we), we encountered an interesting bug after migrating to .NET8 from .NET6. The project is a weather forecast service written in F# that provides weather forecast data from various weather forecast providers. We expose an API that can return forecast data for a specific point and date.

Shortly after upgrading to .NET8, the client reported that part of the requested forecast data is missing. After some investigation, we noticed that some forecast values are infinity. 🤔

## Infinity value

We checked the forecast data from the provider for that specific point and didn't find anything problematic. After some more digging, we noticed some strange dates in our weather data that look like this: `2024-02-05T16:44:59.9999997Z`. That rings a bell; such a rounding error could lead to a [divide by zero](https://en.wikipedia.org/wiki/Floating-point_arithmetic#Dealing_with_exceptional_cases), resulting in infinity when working with floating-point numbers.

Our first suspicion was the interpolating function. To provide a forecast for the exact point and date, we do [linear interpolation](https://en.wikipedia.org/wiki/Linear_interpolation) of forecast values from the 2 nearest points in time and space. The interpolation function looks like this:

```fsharp
((xp - x1) * (v2 - v1) / (x2 - x1)) + v1
```

Where in our case `xp` is the point in time we want to forecast, `x1` and `x2` are the nearest points in time, and `v1` and `v2` are the forecast values for `x1` and `x2`.

Because of the rounding error, the service considered `2024-02-05T16:44:59.9999997Z` and `2024-02-05T16:45:00.0000000Z` as different time points, and when we were looking for the 2 nearest times, in some cases we got these 2 points. Using them in the interpolation function, `x2 - x1` evaluates to zero, resulting in infinity.

At this point we deployed a hotfix that in case of infinity result of interpolation returns the first value instead.

## .NET breaking change

But why did this bug appear only after the upgrade to .NET8? After some debugging, we were able to isolate the problem in parsing weather data from the provider. To represent time, the provider uses the number of days from some fixed date. That could easily create rounding errors when converting to .NET `DateTime` type. But it didn't explain why this bug appeared only after the upgrade to .NET8.

It turns out that in .NET7 there was a [change in behavior](https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/7.0/datetime-add-precision) in the `DateTime.AddDays` method. Until .NET6, the method was rounding the result to the nearest millisecond, but in .NET7 and later, no rounding is performed.

After learning this, the fix was simple. We just needed to round the `DateTime` value to the nearest millisecond.

### Note about DateTimeOffset

In discussion with colleagues, there was a question if the same problem could happen with `DateTimeOffset` type. Although `DateTimeOffset` methods are not mentioned in the [breaking change page](https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/7.0/datetime-add-precision), they share the same implementation as `DateTime` methods. So, the same problem will occur with `DateTimeOffset`.