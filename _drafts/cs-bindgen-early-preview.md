---
layout: post
title: "An early preview of cs-bindgen"
permalink: /posts/cs-bindgen-early-preview/
---

For the last few months I've been toiling away in relative secrecy on a project that's been in the back of my mind for a long time: Embedding Rust code in a C# project, specifically for the purpose of being able to share application logic between a Rust server application and a C# client application. I had initially assumed that, despite my personal love for the Rust programming language, this wasn't worth pursuing because it would ultimately be more practical to build your server in C# and share code through the common language. But when I actually tried doing this, I found that sharing C# code [wasn't possible for my use case](https://randompoison.github.io/posts/sharing-with-unity/). This led me to finally start investigating embedding Rust into a C# application.

It took about five months of investigation and prototyping, but I finally have something that's at least solid enough to share with the world (and my coworkers). This post is an early demonstration of [cs-bindgen](https://github.com/randomPoison/cs-bindgen/), a new tool for building high-level C# bindings to Rust code. It's still very rough around the edges and is nowhere near ready for production use, but I've built out enough of the core functionality to feel confident that this project could grow to be production-ready with more work.

# Lightning Tour

Let's start with a quick example. For the following Rust code:

```rust
#[cs_bindgen]
pub struct Person {
    pub name: String,
    pub age: u32,
}

#[cs_bindgen]
impl Person {
    pub fn new(name: String, age: u32) -> Self {
        Person { name, age }
    }
}

#[cs_bindgen]
pub fn greet_person(person: &Person) -> String {
    format!(
        "Hello, {}! You are {} years old.",
        person.name,
        person.age,
    )
}
```

You can access the `Person` type (including its fields) from C# and can call the `greet_person` function as if they were written in C#:

```csharp
var person = new Person("Randall", 87);
Console.WriteLine(person.Name);
Console.WriteLine(person.Age);

string greeting = RustCode.GreetPerson(person);
Console.WriteLine(greeting);
```

Here's the lightning tour of what's going on here:

* We can annotate items in Rust with `#[cs_bindgen]` in order to export them to C#. Currently structs, enums, impl blocks, and free functions are supported.
* `Person::new` is converted into a constructor for the generated `Person` struct in C#, allowing you to create instances of `Person` from C#.
* Exported functions can take exported types as parameters and can return them.
* The generated C# items use C# naming conventions, rather than having the exact naming of the original Rust items.

cs-bindgen has a very specific goal: Generate high-level, idiomatic C# wrappers for calling into Rust code. It specifically targets the use case of embedding Rust within a C# library or application (so it doesn't support calling arbitrary C# functions, for example). I want calling into Rust code to feel no different than calling into any other C# code.

## Enums

As much as possible, I try to preserve Rust's semantics while still exposing them in a way that makes sense for C#. For example, if you export a simple C-like enum:

```rust
#[cs_bindgen]
pub enum SimpleEnum {
    Foo,
    Bar,
    Baz,
}
```

You'll get the obvious equivalent in C#:

```csharp
public enum SimpleEnum
{
    Foo,
    Bar,
    Baz,
}
```

However, you can also export data-carrying enums:

```rust
pub enum DataEnum {
    NoData,
    Toggle(bool),
    KeyValue {
        key: String,
        value: u32,
    }
}
```

This is represented in C# as an `IDataEnum` interface and a static `DataEnum` class containing a struct type for each variant of the enum:

```csharp
public interface IDataEnum { }

public static class DataEnum
{
    public struct NoData : IDataEnum { }
    
    public struct Toggle : IDataEnum
    {
        public bool Item0;
    }
    
    public struct KeyValue : IDataEnum
    {
        public string Key,
        public uint Value,
    }
}
```

Any Rust exported Rust function that takes a `DataEnum` as a parameter or returns one will take/return `IData` enum on the C# side. This approach gives us a good approximation of the enum defined in Rust, while still leveraging C#'s type system to in a way that feels reasonably idiomatic. In particular, it takes advantage of the pattern matching functionality added in C# 7 to make handling the variants type-safe. The end result is that using `DataEnum` from C# feels wonderfully similar to using it in Rust while still sticking to common C# idioms:

```rust
let value = DataEnum::KeyValue {
    key: "foo".into(),
    value: 7,
};

match value {
    DataEnum::NoData => println!("No data"),
    DataEnum::Toggle(toggle) => println!("Toggle: {}", toggle),
    DataEnum::KeyValue { key, value } => println!("{} => {}", key, value),
}
```

```csharp
let value = (IDataEnum)new DataEnum.KeyValue("foo", 7);
switch (value)
{
    case DataEnum.NoData _:
        Console.WriteLine("No data");
        break;

    case DataEnum.Toggle toggle:
        Console.WriteLine($"Toggle: {toggle.Item0}");
        break;

    case DataEnum.KeyValue kv:
        Console.WriteLine($"{kv.Key} => {kv.Value}");
        break;
}
```