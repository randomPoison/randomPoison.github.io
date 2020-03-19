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

# Interlude: Rust

# Part II: C#

# Part III: PHP



