---
layout: post
title: "Handling Variant Data: A Journey in Three-And-A-Half Parts"
permalink: /posts/variant-data/
---

What you are about to read is an overly-long (and unnecessarily indulgent) exploration of handling variant data in a number of different programming languages. While I'm not good at writing introductions that help ease readers into the topic at had, I'll at least start with some extra context to help others figure out if this article is relevant to their interests:

* This article specifically focusses on [C#](https://docs.microsoft.com/en-us/dotnet/csharp/) and [PHP](https://www.php.net/), since those are the main languages in use at [Synapse Games](http://synapsegames.com/).
* For discussing how to represent data in an interchange format, I'll be focussing primarily on [JSON](https://www.json.org/).
* :crab: [Rust](https://www.rust-lang.org/) :crab: ​is also discussed for comparison purposes. I promise I'll try to keep it brief.
* Architecturally I'll be focussing on working within a client/server application, however most of the points discussed should apply to non-networked applications.
* I'll be discussing the topic within the context of game development, but the concept is generally applicable for most application domains.

Overall, the specifics in this article are tailored to be relevant to my current work, but I'll do my best to keep the discussion general.

# Prologue

The point of this article is not to discuss what variant data *is*, rather to discuss how to handle variant data in software development. However, it's going to be difficult to have that discussion without having a shared understand of what variant data is, so I suppose some introduction is in order.

At a high level, you have variant data whenever a given value can have multiple possible "types" or "shapes" and you need to be able to interpret the value differently based on the "type". Generally this comes up when dealing with lists (or other collections) of hetergeneous data, where you can't statically determine the type of each element in the collection.

As a practical example of this, we're going to use awarding in-game items as an example. 

