---
layout: post
title: "Handling Variant Data: A Journey in Three-And-A-Half Parts"
permalink: /posts/variant-data/
---

What you are about to read is an overly-long (and unnecessarily indulgent) exploration of handling variant data in a number of different programming languages. While I'm not good at writing introductions that help ease readers into the topic at had, I'll at least start with some extra context to help others figure out if this article is relevant to their interests:

* This article specifically focusses on [C#](https://docs.microsoft.com/en-us/dotnet/csharp/) and [PHP](https://www.php.net/), since those are the main languages in use at [Synapse Games](http://synapsegames.com/).
* For discussing how to represent data in an interchange format, I'll be focusing primarily on [JSON](https://www.json.org/).
* :crab: [Rust](https://www.rust-lang.org/) :crab: ​is also discussed for comparison purposes. I promise I'll try to keep it brief.
* Architecturally I'll be focusing on working within a client/server application, however most of the points discussed should apply to non-networked applications.
* I'll be discussing the topic within the context of game development, but the concept is generally applicable for most application domains.

Overall, the specifics in this article are tailored to be relevant to my current work, but I'll do my best to keep the discussion general.

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

This setup isn't a completely realistic example of how you'd want to setup this kind of game, however it presents the following challenges that make it a good example to look at:

* There are three different "shapes" for reward items: Just a quantity (⭐ and 💎), just an ID (🧙), or an ID plus an integer (🛡️).
* Two of the possible rewards look the same (⭐ and 💎), so we need to make sure we can differentiate between the two at runtime!
* Two of the possible reward types share a common field (🧙 and 🛡️ both have an ID field), but do not otherwise have the same shape. In practice, that means that this field may need to be interpreted differently depending on the actual type of the reward (i.e. a hero ID is used to look up the stats for the hero, and the equipment ID is used to look up the stats for the equipment, and the two can't be interchanged).

When a player completes a quest, the server for our hypothetical game needs to be able to do the following:

* Build a runtime representation of the list of rewards.
* Encode that list in JSON so that it can be sent to the client.

In turn, the client must be able to:

* Decode the rewards JSON into an appropriate runtime representation that it can use.
* Display the list of rewards to the player.

This leaves us with two main questions:

* How do we best represent our list of rewards at runtime in our language(s) of choice?
* How do we best represent our rewards in our data format of choice so that we can communicate between our client and server code?

For the first question, we'll be looking at C# and PHP for the client and server respectively. For the second, we'll use JSON as our data interchange format.

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

* ⭐ Stars and 💎 Gems are exactly the same in the JSON! That means we can't reliably distinguish between the two, and so the client may display 💎 when the player was in fact awarded ⭐.
* In order to distinguish between a 🧙 Hero and an 🛡️ Equipment we need to check for the presence of the `durability` field. While this is simple to do now, it will quickly become tedious and error-prone if we end up adding more item types in the future.

The best way to disambiguate the rewards is to **tag each reward with its type**. For example:

```json
[
  { "type": "stars", "quantity": 100 },
  { "type": "gems", "quantity": 5 }
]
```

Doing this allows us to quickly identify what item is being awarded for each item in the rewards list. For JSON specifically, there are at least three different ways we can tag our data:

* **Internal tagging**, where the tag is done as a field within the object:

  ```json
  {
    "type": "stars",
    "quantity": 100
  }
  ```

  This format is the cleanest in terms of readability (as long as you can guarantee that the tag field will always be listed first), but has the drawback that the data itself cannot contain a field with the same name used for the tag field. It's also worth noting that this approach won't work if your data wasn't already represented as a JSON object (e.g. if the data was a string then there's nowhere to add the tag field).

* **Adjacent tagging**, where the tag and data are adjacent fields within a containing object:

  ```json
  {
    "tag": "stars",
    "data": {
      "quantity": 100
    }
  }
  ```

  This approach avoids the issue of conflicting field names and will work for any kind of data (not just objects, as with internal tagging) at the cost of greater verbosity.

* **External tagging**, where the tag is the key a container object:

  ```json
  {
    "stars": {
      "quantity": 100
    }
  }
  ```

  The main advantage of this approach is that it can enable more efficient deserialization logic as the deserialization code can always determine the expected "type" of the data before reading any of the data itself, however it is arguably the most awkward syntax from a human-readability perspective.

All of these approaches are valid and will solve the issue of ambiguity in your data. In practice which approach you choose will come down to how important human-readability is to you and what approaches are best supported by the serialization systems used in your applications.

# Interlude: Goals and Criteria

Before we start looking at how to handle this data in our target programming languages, I want to lay out some criteria for what a "good" system for handling variant data looks like. As we'll see, there's many different ways of representing such data at runtime, so we'll need some way of comparing them against each other.

* **Robustness** - Is the deserialization logic able to reliably handle or reject unexpected data? While this depends on the specifics of your application, it's generally best to not 
* **Correctness** - When working with variant data in code, does the system catch invalid usage of variant data (e.g. trying to get the `durability` field of a ⭐ Stars reward)?
* **Performance** - How much overhead is needed to represent variant data? Variant data almost always has some addition costs as compared to non-variant data, but different approaches will have different performance characteristics.

# Part II: C#

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

The fact that this approach makes it easy to accidentally access invalid fields is problematic, as any field that's not valid for the current reward type is effectively uninitialized. This makes it a big source for potential bugs that will be difficult to track down later.

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

When working with an `IReward` object, you can take advantage of the pattern matching feature added in C# 7 to handle the reward based on its concrete type:

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

* You can't access any of the fields for any of the rewards variants without first checking the reward type (by downcasting to a concrete type). On the other hand, if there's data that's guaranteed to be shared by all reward types (e.g. if there's always a `quantity` field so that more than one of a given item can be given at once), that can be added to the `IReward` interface so that its accessible without downcasting.
* If you accidentally downcast an `IReward` object to the wrong type, you'll either get `null` or an exception (depending on what type of casting you did) but you'll never get an invalid object.
* Any given reward type only needs to have the fields that are relevant to it, making the reward data much easier to work with when writing code.
* Performance-wise there's a bit of extra overhead that comes with downcasting, but it also saves a bit of heap space by reducing the size of each allocated reward object.

The main drawback to this approach that I've seen is that it complicates deserialization. When deserializing a collection of variant data, the deserialization code needs to be able to determine which concrete class to instantiate based on the data. While the data is unambiguous and contains all the information we need in order to do this thanks the handy-dandy tag we added, the serialization logic may not know how to find the correct class based on the tag. For example, [this article discusses how to get Json.NET to handle this kind of data](https://skrift.io/articles/archive/bulletproof-interface-deserialization-in-jsonnet/). I won't explore how to handle deserialization in this article since it depends heavily on the serialization system you're using, but it's something you'll need to be aware of when implementing this pattern in your project.

# Part III: PHP

??????? ??? ? ??????  ?????????? ??? ?? ??????????? ??? ? ????????? ? ??  ???????? ?? ? ? ?? ????? ??? ????? ???????? ?? ???? ??? ??? ??? ?????? ??????? ? ?? ????? ????? ????? ?????? ????? ? ?? ??????? ????? ??????? ???? ? ?? ? ? ? ? ?????? ???????? ?????? ?????????? ?????????? ?? ?? ?? ??? ? ???? ????? ?? ? ????? ???? ?? ?? ?? ? ?????? ??? ?? ????? ????? ?? ??? ?? ????? ???? ???? ?? ???? ??? ????? ???? ? ???? ???? ? ???? ? ?? ???? ????? ????? ????? ???? ? ??? ??? ????? ??????? ???? ?? ???? ???? ?? ???? ??? ????? ????? ???? ????? ??????? ???? ??????? ??? ??? ?????? ???? ???? ????? ??????? ???????? ?????????? ???? ???? ???? ?? ?? ???? ??? ???? ????? ??? ????? ???? ???? ?????? ?? ????? ??????? ?? ??? ????? ??? ??? ???? ??? ??? ???? ?? ??? ?? ???? ???? ???? ????? ???? ?? ???? ???? ???? ?? ??? ????? ?????? ????? ????? ?????? ?? ??? ????? ????? ??? ???? ?? ??? ????? ???? ???? ???? ??? ????? ???? ???? ??? ???? ??? ??? ??? ???? ??? ???? ?????

# Prologue: Rust

Before we start looking at how to handle this in any of the programming languages that I actually use professionally, I'm going to take a (somewhat) brief look at how this would be handled in the [Rust programming language](https://www.rust-lang.org/). I'm specifically looking at Rust here because this happens to be a case that Rust has very strong support for this sort of case, providing strong guarantees about how variant data is handled that we will try to emulate as much is possible in our other target languages.

First, a brief introduction to [Rust's enums](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html). Like many languages, Rust supports creating user-defined enumeration types that can be one of several user-defined values:

```rust
pub enum MyEnum {
    Foo,
    Bar,
}

// You can create a value of `MyEnum::Foo` or `MyEnum::Bar`,
// any other value is a compiler error.
let my_enum = MyEnum::Foo;
```

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
    Reward::Gems { quantity } => println!("Awarding {} gems", quantity),
    Reward::Hero { id } => println!("Unlocking hero {}", id),
    Reward::Equipment { id, durability } =>
        println!("Awarding equipment {} with durability {}", id, durability),
}
```

This setup, in my opinion, is the ideal way of handling variant data in a programming language. It makes it very easy to always correctly handle reward data:

* You're statically prevented from accessing the data in the reward until you've checked which type of reward it is, and you can only ever access the fields of the correct variant.
* If you forget to handle any of the possible cases, you get a compiler error! This also means that if you later add a new reward type, the compiler will makes sure you go back and update all the places in the code base where you're already checking the type of a reward.

It's also very efficient: There's no allocation involved in creating an instance of `Reward`, and the size of `Reward` is equal to the size of its largest variant plus the size of the discriminant (which will rarely need to be larger than a single byte).



