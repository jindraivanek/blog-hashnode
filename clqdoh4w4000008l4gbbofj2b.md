---
title: "Improved F# computation expression suggestions in Rider 2023.3"
datePublished: Wed Dec 20 2023 11:15:57 GMT+0000 (Coordinated Universal Time)
cuid: clqdoh4w4000008l4gbbofj2b
slug: improved-f-computation-expression-suggestions-in-rider-20233
tags: opensource, fsharp, contribution-to-open-source, rider, amplifying-fsharp

---

Recently, [Rider version 2023.3](https://www.jetbrains.com/rider/whatsnew/2023-3/) was released, including my contribution - improved suggestions for [computation expressions](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions). [Custom operation](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions#custom-operations) inside the computation expression is now on top of the auto-complete suggestion list.

The majority of work on this contribution was done live on [Amplifying F#](https://amplifying-fsharp.github.io) session: [Rider: CE custom operations completion list.](https://amplifying-fsharp.github.io/sessions/2023/09/08/)

You can watch it here:

%[https://youtu.be/U4fZ8nASokc] 

This improves developer experience when working with custom operation heavy computation expressions, like [Dapper.FSharp](https://github.com/Dzoukr/Dapper.FSharp), [Saturn](https://saturnframework.org), [Farmer](https://compositionalit.github.io/farmer/), and many more. See it in action:

#### **Dapper.FSharp**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1703069389660/6a0f5063-d6f4-4df0-bf9b-466a5aab265a.png align="center")

#### **Saturn**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1703069447724/401e8636-c27a-434b-a229-f39b39bdf243.png align="center")

#### **Farmer**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1703069418295/87aa4e75-b1ce-4555-8af6-e833dad200e3.png align="center")

PR adding this feature is [here](https://github.com/JetBrains/resharper-fsharp/pull/562).