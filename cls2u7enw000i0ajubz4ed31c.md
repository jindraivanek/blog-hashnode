---
title: "F# tips weekly #4: Record types"
datePublished: Thu Feb 01 2024 06:30:17 GMT+0000 (Coordinated Universal Time)
cuid: cls2u7enw000i0ajubz4ed31c
slug: f-tips-weekly-4-record-types
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1706439667048/1f39d2b3-1047-46e8-aa61-02014bc6a70f.jpeg
tags: fsharp, record

---

The record type is a core feature in F#, but some of its details are not well-known. Let's delve into them.

### Type Inference

Type inference in F# may sometimes deduce a record type differently than expected. For instance, in the following code, `r` is inferred as the `B` type, even though it match the `A` type, resulting in compiler errors:

```fsharp
type A = { X: int; Y: string }
type B = { Y: int; X: string }

let r = { X = 1; Y = "a" }
```

The compiler checks only the field names in the record type and selects the *last* record definition that matches.

To resolve these errors, we can use explicit type annotations:

```fsharp
let b: A = { X = 1; Y = "a" }
```

or

```fsharp
let b = { X = 1; Y = "a" } : A
```

Another option is to use the type in the field name:

```fsharp
let b = { A.X = 1; Y = "a" }
```

### Pattern Matching

Pattern matching allows us to deconstruct records:

```fsharp
type User = { Name: string; Age: int }
let user = { Name = "John"; Age = 20 }
let { Name = name; Age = age } = user
```

We can deconstruct only specific fields and use other patterns inside, especially in combination with the `as` keyword:

```fsharp
type User = { Name: string; Age: int option; Email: string option; Address: string option }
let user = { Name = "John"; Age = None; Email = Some "someEmail"; Address = None }
match user with
| { Email = Some email } as u -> printfn "Sending email to %s: %s" u.Name email
| { Address = Some address } as u -> printfn "Sending a postcard to %s: %s" u.Name address
| u -> printfn "No contact info for %s" u.Name
```

This way, we can handle cases based on certain fields while still having access to the entire record.

Patterns can also be nested:

```fsharp
type Address = { Street: string; City: string }
type User = { Name: string; Age: int option; Email: string option; Address: Address option }

let { Address = { City = city } } = user
```

### Default Values

Often, we need to create a record specifying only some fields and use default values for the rest. Although F# don't have direct support for this, it is common to use the `Default` static member:

```fsharp
type Config = {
    ServerAddress: string
    Port: int
    UseSSL: bool
    Timeout: System.TimeSpan option
} with
    static member Default = { ServerAddress = "localhost"; Port = 80; UseSSL = false; Timeout = None }

let config = { Config.Default with ServerAddress = "my-great-site.com" }
```

### Updating Nested Records

Starting from F# 8, we can use shorthand syntax for updating nested records:

```fsharp
type PersonalInfo = { Age: int; Email: string }
type User = { Name: string; Info: PersonalInfo }

let user = { Name = "John"; Info = { Age = 20; Email = "john@email" }}
let user2 = { user with Info.Age = user.Info.Age + 1 }
```