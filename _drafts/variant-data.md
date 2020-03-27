---
layout: post
title: "Handling Variant Data: A Journey in Three-And-A-Half Parts"
permalink: /posts/variant-data/
---

This post is an overly-long (and unnecessarily self-indulgent) exploration of handling variant data in a number of different programming languages. While I'm not good at writing introductions that help ease readers into the topic at had, I'll at least start with some extra context to help others figure out if this article is relevant to their interests:

* This article specifically discusses [C#](https://docs.microsoft.com/en-us/dotnet/csharp/) and JavaScript, though I'll try to extract conclusions that are applicable to a broader set of languages.
* For discussing how to represent data in an interchange format, I'll be focusing primarily on [JSON](https://www.json.org/).
* :crab: [Rust](https://www.rust-lang.org/) :crab: ​is also discussed for comparison purposes. I promise I'll try to keep it brief.
* Architecturally I'll be focusing on working within a client/server application, however most of the points discussed should apply to non-networked applications.
* I'll be discussing the topic within the context of game development, but the concept is generally applicable for most application domains.

Admittedly the specifics in this article are tailored to be relevant to my current work at [Synapse Games](http://synapsegames.com/), but I'll do my best to keep the discussion general.

# Prologue

The point of this article is not to discuss what variant data *is*, rather to discuss how to handle variant data in software development. However, it's going to be difficult to have that discussion without having a shared understand of what variant data is, so I suppose some introduction is in order.

At a high level, you have variant data whenever a given value can have multiple possible "types" or "shapes" and you need to be able to interpret the value differently based on the "type". Generally this comes up when dealing with lists (or other collections) of heterogeneous data, where you can't statically determine the type of each element in the collection.

As a practical example of this, we'll look at awarding a player items for completing a quest in a hypothetical mobile game. In our example game, the player can earn a number of different types of rewards from completing a quest:

* ⭐ Stars (i.e. soft currency)
* 💎 Gems (i.e. hard currency)
* 🧙 New heroes
* 🛡️ Equipment items

Each of these different items is defined slightly differently:

* ⭐ Stars and 💎 Gems both only specify a quantity.
* 🧙 Heroes specify a unique ID for the hero that has been unlocked. No quantity is specified, since you can only unlock a given hero once!
* 🛡️ Equipment specifies a unique ID for the item, plus positive integer value for the item's durability.

This setup isn't a completely realistic example of how you'd want to setup this kind of game, however it presents the following challenges that make it a useful example:

* There are three different "shapes" for reward items: Just a quantity (⭐ and 💎), just an ID (🧙), or an ID plus an integer (🛡️).
* Two of the possible rewards look the same (⭐ and 💎), so we need to make sure we can differentiate between the two at runtime!
* Two of the possible reward types share a common field (🧙 and 🛡️ both have an ID field), but do not otherwise have the same shape. In practice, that means that this field may need to be interpreted differently depending on the actual type of the reward (i.e. a hero ID is used to look up the stats for the hero, and the equipment ID is used to look up the stats for the equipment, and the two can't be interchanged).

When a player completes a quest, the server for our hypothetical game needs to be able to do the following:

* Load the list of rewards from a configuration file somewhere.
* Build a runtime representation of the list of rewards such that it can update the players account with the awarded items.
* Encode that list in JSON so that it can be sent to the client. This could be as simple as forwarding the same configuration data it originally loaded, or could involve re-serializing its own in-memory representation.

In turn, the client must be able to:

* Decode the rewards JSON into an appropriate runtime representation that it can use.
* Display the list of rewards to the player.

This leaves us with two main questions:

* How do we best represent our rewards in our data format of choice so that we can save it to a configuration file and communicate between our client and server code?
* How do we best represent our list of rewards at runtime in our language(s) of choice?

For the first question, we'll look at how we can robustly represent this kind of data in JSON since it's a very common data format and the principles we discuss will be broadly applicable for many other formats. For the latter, the answer depends heavily on what language you're using and what features it provides to help with this kind of data. As such we'll discuss two different options: We'll look at JavaScript to see how variant data can be handled in a highly dynamic language, and C# to see how we can use a stronger type system to enforce correctness when working with variant data.

# Part I: JSON

Before we dig into the nuances of how to represent this data in different programming languages, let's look at how we can represent a list of rewards in a language-independent way by encoding it in JSON. Both our client and server code ultimately needs to understand this JSON format, so it should be informative to the subsequent discussions.

The dead simplest approach would be to make a list of objects, with each object containing the fields needed for each item:

```json
[
  { "quantity": 100 },
  { "quantity": 5 },
  { "id": 123 },
  {
    "id": 111,
    "durability": 1000
  }
]
```

While this setup contains all of the necessary data for our rewards, we can quickly identify a couple of problems with this approach:

* ⭐ Stars and 💎 Gems are exactly the same in the JSON! That means our code is going to have a hard time distinguishing between the two, risking that we award 💎 when the player should have gotten ⭐ (or vice versa). You could get around this by using a different field names for the two (e.g. "quantity" for ⭐ Stars and "amount" for 💎 Gems), but that often leads to making the data (and the code relating to it) harder to understand.
* In order to distinguish between a 🧙 Hero and an 🛡️ Equipment we need to check for the presence of the `durability` field. While this is simple to do now, it will quickly become tedious and error-prone if we end up adding more item types in the future.

The best way to disambiguate the rewards is to **tag each reward with its type**. For example:

```json
[
  { "type": "stars", "quantity": 100 },
  { "type": "gems", "quantity": 5 }
]
```

Doing this means we only have to check a single field in order to reliably determine the type of each reward. For JSON specifically, there are at least three different ways we can tag our data:

* **Internal tagging**, where the tag is done as a field within the object:

  ```json
  {
    "type": "stars",
    "quantity": 100
  }
  ```

  This format is the cleanest in terms of readability (as long as you can guarantee that the tag field will always be listed first), but has the drawback that the data itself cannot contain a field with the same name used for the tag field. It's also worth noting that this approach won't work if your data wasn't already represented as an object, e.g. if the data is a string then there's nowhere to add the tag field.

* **Adjacent tagging**, where the tag and data are adjacent fields within a containing object:

  ```json
  {
    "tag": "stars",
    "data": {
      "quantity": 100
    }
  }
  ```

  This approach avoids the issue of conflicting field names and allows more flexibility in how you represent the data for each reward as compared to the internal tagging approach. For example, a ⭐ Stars reward could also be represented as:

  ```json
  {
    "tag": "stars",
    "data": 100,
  }
  ```

  Where `data` is a numeric value, rather than an object containing a numeric value. This depends somewhat on the capabilities of your programming language, though.

* **External tagging**, where the tag is the key a container object:

  ```json
  {
    "stars": {
      "quantity": 100
    }
  }
  ```

  The main advantage of this approach is that it can enable more efficient deserialization logic as the deserialization code can always determine the expected "type" of the data before reading any of the data itself, however it is arguably the most awkward syntax from a human-readability perspective. It also has the same flexibility in representation for the reward data that the adjacent tagging approach does.

All of these approaches are valid and will solve the issue of ambiguity in your data. In practice which approach you choose will come down to two factors:

* How important human-readability is for your purposes. It may be worth going with internal tagging if you expect to often be reading (or writing!) the JSON for your data.
* What format is best supported by the serialization system used for your language. Different serialization libraries will have different conventions for how they manage this kind of data, and it's often easiest to stick with the default conventions of the library you're using.

# Interlude: Goals and Criteria

Before we start looking at how to handle this data in our target programming languages, I want to lay out some criteria for what a "good" system for handling variant data looks like. As we'll see, there's many different ways of representing such data at runtime, so we'll need some way of comparing them against each other.

* **Robustness** - Is the deserialization logic able to reliably handle or reject unexpected data? While this depends on the specifics of your application, it's generally best to catch invalid input data as early as possible.
* **Correctness** - When working with variant data in code, does the system catch invalid usage of variant data (e.g. trying to get the `durability` field of a ⭐ Stars reward)? Does it ensure that you handle all the possible variants when? Does it make it easy to refactor existing code when you add/remove/change a variant?
* **Performance** - How much overhead is needed to represent variant data? Variant data almost always has some additional costs as compared to non-variant data, but different approaches will have different performance characteristics.

My personal goal is to find a solution that best enforces correctness/robustness while minimizing performance overhead, and the examples I bring up throughout this post will generally trend in the direction of finding more tools to enforce correctness. Where possible I try to bring up opportunities to make different trade offs, or at least point out where further pursuing correctness would have diminishing returns.

It's also worth noting up front that often times it's possible to side step needing to handle variant data at all. Looking at our rewards example, we could in theory not put all of our rewards in a single list. Instead, we could make each reward type its own field or list, such that each item in the list is always of a known type:

```json
{
  "stars": 100,
  "heroes": [
      { "id": 123 },
      { "id": 234 }
  ],
  "equipment": [
      { "id": 111, "durability": 70 },
      { "id": 707, "durability": 100 }
  ]
}
```

This completely sidesteps the need to differentiate between different types of reward, since each field only ever contains a reward of a single type!

This is absolutely a reasonable approach, and it may well be a better solution for your use case than dealing with variant data. An alternate solution we've used at Synapse is to use a generic system for defining items, such that all items are effectively the same "type". However, sometimes this simply isn't an option for what you're trying to do, sometimes you specifically need to have different types of data/object in a single collection. As such, it's still helpful to explore the available options for dealing with variant data, even if a non-variant solution is sometimes the better option.

# Part II: JavaScript (and dynamic languages in general)

The nice thing about implementing this in JavaScript is that we can represent the data in memory identically to how we represent it in JSON. The bad thing about implementing this in JavaScript is that that's the only nice thing.

Okay, let me try that again with less snark: For highly dynamic languages, we have both the gift and the curse of having no type system to worry about when dealing with variant data. This means that it's very easy for us to jam heterogenous data into a collection and start working with it immediately, but unfortunately means that you often don't have much support at the language-level for handling that data in a robust way. I'm going to be looking at JavaScript specifically because it's widely used (and it's the only dynamic language that I know fairly well), but a lot of these solutions will apply to other dynamic languages.

Since JSON is (deliberately) so similar to JS types, it's very easy to translate one of the tagging solutions described above directly. By `switch`ing on the `type` field, we can iterate over a list of rewards and handle each reward based on its type. For example, using the internal tagging example from before:

```js
let rewards = [
    { "type": "stars", "quantity": 100 },
    { "type": "gems", "quantity": 5 },
];

for (const reward of rewards) {
    switch (reward.type)
    {
        case "stars":
            console.log("Got some stars: ", reward.quantity);
            break;

        case "gems":
            console.log("Got some gems: ", reward.quantity);
            break;
    }
}
```

This solution is fairly straightforward and will work basically the same way for any of the tagging styles shown in the previous section. However, when we look at the criteria I laid out above it leaves a lot to be desired:

- There's nothing in the language to help you correctly handle all possible variants when `switch`ing on the variant tag. You have to remember to list them all, or your code will silently ignore some of your elements.
- There's nothing preventing you from accessing invalid fields on the variant (or fields from the wrong variant) even after you've checked the tag.
- If you're using `JSON.parse()`, there's nothing to prevent your code from loading invalid or malformed data. This is generally true with `JSON.parse()` (you'll need to use a separate JSON schema validator like [ajv](https://www.npmjs.com/package/ajv) to validate the data), but working with variant data exacerbates the issues that come with silently consuming invalid data: Unless you have a `default` case in every `switch` block where you check the variant tag, your code will always silently ignore invalid or unknown variants. This can lead to especially subtle bugs that can be difficult to diagnose.

That being said, those limitations are pretty general limitations of JavaScript and aren't specific to working with variant data: There's nothing stopping you from accessing invalid fields on non-variant data, either, for example. This means that our nice-and-simple approach is also probably about as good as it gets. There are plenty of ways that you could build more infrastructure around this in order to enforce correctness, but doing so adds a lot of overhead in the form of runtime checking. Doing so also goes against most idiomatic usage patterns for JavaScript, since JavaScript APIs often lean into the flexibility the language provides in order to be as permissive as possible, rather than trying to proactively reject invalid data.

Ultimately, there's not much you need to do when dealing with variant types in a dynamic language. Make sure you structure your data with an explicit tag and you'll have everything you need to disambiguate objects of different types.

# Part III: C#

In C#, the obvious way to represent variant data like this is with an enum:

```csharp
public enum RewardType
{
	Stars,
	Gems,
	HeroUnlock,
	Equipment,
}
```

However, this leaves us with the question of how to handle the data for each variant. A simple solution is to create a single class that contains the data for all the variants, plus a field for the tag so that you can determine which fields are valid:

```csharp
public class Reward
{
	public RewardType Type;

	public long Quantity;
	public HeroId Hero;
	public EquipmentId Equipment;
	public int Durability;
}
```

This approach wins out in simplicity, but has a number of drawbacks that make it less than ideal for actual use:

* There's nothing that requires you to check the `Type` field before accessing any of the fields of the reward.
* There's nothing preventing you from accessing the wrong fields for the current reward type.
* There's nothing obvious indicating which fields are valid for any given reward type. You'll either need to have comments in the code (and then make sure you check those comments when working with `Reward` data) or document those details somewhere else (and then hope you can remember where those docs are).
* Every reward uses as much memory as all reward types combined. This is a fairly minor point compared to the other two, but minimizing garbage allocation is often important for consistent performance in games, so it would be good to reduce the memory needed for `Reward` objects if possible.

The fact that this approach makes it easy to accidentally access invalid fields is problematic, as any field that's not valid for the current reward type is effectively uninitialized, making it a potential source of bugs.

The better way to represent variant data, in my opinion, is to use an interface (or base class) and downcasting:

```csharp
public interface IReward { }

public struct Stars : IReward
{
	public readonly long Quantity;
}

public struct Equipment : IReward
{
	public readonly EquipmentId Id;
    public readonly int Durability;
}

// And so on, with a different struct or class
// for each reward type.
```

When working with an `IReward` object, you can take advantage of the pattern matching feature added in C# 7.0 to handle the reward based on its concrete type:

```csharp
switch (reward)
{
    case Stars stars:
        Console.WriteLine($"Awarded {stars.Quantity} stars");
        break;

    case Equipment equipment:
        Console.WriteLine(
            $"Awarded equipment with ID {equipment.Id} " +
            $"and durability {equipment.Durability}");
        break;

    // And so on...
}
```

This approach has a number of advantages over using a single, combined class for all of the reward types:

* You can't access any of the fields for any of the rewards variants without first checking the reward type (by downcasting to a concrete type). On the other hand, if there's data that's guaranteed to be shared by all reward types (e.g. if there's always a `quantity` field so that more than one of a given item can be given at once), that can be added to the `IReward` interface so that it's accessible without downcasting.
* If you accidentally downcast an `IReward` object to the wrong type, you'll either get `null` or an exception (depending on what type of casting you did) but you'll never get an invalid object.
* Any given reward type only needs to have the fields that are relevant to it, making the reward data much easier to work with when writing code.
* Performance-wise there's a bit of extra overhead that comes with downcasting, but it also saves a bit of heap space by reducing the size of each allocated reward object.

While I like this solution a lot, there are a few things about it that I'm not quite satisfied with:

* The compiler won't remind you to handle all possible variants. If you don't have a case for all variants, your `switch` statement will silently do nothing. The [switch expression](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/switch-expression) added in C# 8.0 is a bit better in that it at least throws an exception if none of your cases are executed, but it has other restrictions that mean it can't always be used (i.e. that it must return a value). And unfortunately, C# doesn't emit warnings when you fail to handle possible cases [even when working with regular enums](https://stackoverflow.com/a/12531166/6649664).
* This approach also doesn't play well with deserialization conventions. Most serialization libraries for C# use reflection to handle loading data into instances of your classes, [Json.NET](https://www.newtonsoft.com/json) being perhaps the most widely used example. When you're deserializing into a `List<IReward>`, the serialization system can't necessarily tell what concrete type should be instantiated for each element in the list. In general this means you'll need to write some extra glue to tell it what the valid variants are and how to determine the type of each element. [This article discusses how to handle this kind of data in Json.NET](https://skrift.io/articles/archive/bulletproof-interface-deserialization-in-jsonnet/), for example.

That said, this approach is a pretty solid solution as far as C# goes. It should also apply nicely to most other languages that support classical inheritance. Even if your language of choice doesn't have pattern matching, most languages have some kind of speculative downcasting that will allow you to do something similar.

# Epilogue: Rust

If you're curious about how we could further pursue correctness in handling variant data, we can take a look at how this would be handled in the [Rust programming language](https://www.rust-lang.org/). If you're not familiar, Rust is a relatively new programming language that combines a very strong, expressive type system with the ability to write abstractions with very little performance overhead. This includes first-class support for the kind of variant types that we've been looking at!

First, a brief introduction to [Rust's enums](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html). Like many languages, Rust supports creating user-defined enumeration types that can be one of several user-defined values. So for the following enum definition:

```rust
pub enum MyEnum {
    Foo,
    Bar,
}
```

You can create a value of `MyEnum::Foo` or `MyEnum::Bar`, any other value for a variable of type `MyEnum` is a compiler error:

````rust
// Correct way to use `MyEnum`.
let my_enum = MyEnum::Foo;

// ERROR: Not a valid variant.
let my_enum = MyEnum::NotReal;

// ERROR: Arbitrary integers aren't valid values.
let my_enum = 5;

// ERROR: Integers can't be cast to the enum type.
let my_enum = 5 as MyEnum;
````

Rust then has `match` blocks which behave similarly to `switch` blocks in other languages:

```rust
match my_enum {
    MyEnum::Foo => println!("my_enum was Foo"),
    MyEnum::Bar => println!("my_enum was Bar"),
}
```

However, unlike most languages, Rust's enums can also contain data!

```rust
pub enum Reward {
    Stars { quantity: u32 },
    Gems { quantity: u32 },
    Hero { id: HeroId },
    Equipment {
        id: EquipmentId,
        durability: u32
    },
}
```

When working with a value of an enum, you can't directly access any of the fields declared in the variants:

```rust
let reward = Reward::Stars { quantity: 20 };

// ERROR: No field `quantity` on type `Reward`.
let quantity = reward.quantity.
```

Instead, you need to match on the value and handle all of the possible variants. Only within the relevant match arm can you access the fields of any given variant:

```rust
match reward {
    Reward::Stars { quantity } => println!("Awarding {} stars", quantity),
    Reward::Gems { quantity } => rintln!("Awarding {} gems", quantity),
    Reward::Hero { id } => println!("Unlocking hero {}", id),
    Reward::Equipment { id, durability } => {
        println!("Awarding equipment {} with durability {}", id, durability);
    }
}
```

This setup, in my opinion, is the ideal way of handling variant data at runtime. It makes it very easy to always correctly handle reward data:

* You're statically prevented from accessing the data in the reward until you've checked which type of reward it is, and you can only ever access the fields of the correct variant.
* If you forget to handle any of the possible cases, you get a compiler error! This also means that if you later add a new reward type, the compiler will makes sure you go back and update all the places in the code base where you're already checking the type of a reward.
* It's also very efficient: There's no allocation involved in creating an instance of `Reward`, and the size of `Reward` is equal to the size of its largest variant plus the size of the discriminant (which will rarely need to be larger than a single byte).
* Rust's de facto serialization library [Serde](https://serde.rs/) automatically validates incoming data and rejects any data that can't be correctly represented at runtime. And because validation happens as part of deserialization, there's little-to-no performance overhead in doing so!

While not many people are using Rust in production, most functional programming languages support something similar (referred to as ["sum types", "algebraic data types"](https://en.wikipedia.org/wiki/Algebraic_data_type), or ["tagged unions"](https://en.wikipedia.org/wiki/Tagged_union)). If you're using such a language, you'll likely get similar results to what you'd get from using an `enum` in Rust!

# Conclusion

Whew! That's a lot of words on a pretty minor data pattern. While I covered a lot of details across a number of different languages, I think the main takeaways to keep in mind are:

* **Tag your variant data!** Don't do ad hoc variant detection by checking for the presence of different fields, as that can still fail if you have ambiguous variants.
* **Take advantage of language features** to make your variant data safer to work with. If you have a type system, don't just jam all of your variants into a single type that has a bunch of uninitialized fields. If you're working with a more dynamic language, make sure to still use a tag at runtime!

