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

# Interlude: Rust

# Part II: C#

# Part III: PHP



