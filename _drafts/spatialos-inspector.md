---
layout: post
title: "The Brilliance of SpatialOS's Modular Inspector"
permalink: /posts/spatialos-inspector/
---

Improbable, the developers of SpatialOS, recently released an alpha preview of their new Modular Inspector, and boy is this thing :fire: HOT :fire:. When I first saw the demo videos, I was blown away by the sheer ingenuity of the design. In particular, I'm impressed by how well the new inspector is designed to give the user the power to wrangle complex worlds and see exactly what information they care about.

Since I'm part of the ongoing process to put together an editor for Amethyst, I want to go into some deeper detail about why this new inspector is so brilliant. ECS is starting to become a more widely-recognized paradigm in the gamedev community, but I think there's a lot of confusion around how you make ECS as comprehensible and easy to use as the existing Object Oriented paradigms. I think the SpatialOS Modular Inspector is the first graphical tool to really demonstrate how this is possible.

## Queries, Not Hierarchies

The biggest thing that the Modular Inspector does is do away with the traditional hierarchy view that's common in game editors and replace it with a tool for building dynamic queries over the contents of your world.

![The Query Editor module in the Modular Inspector](https://commondatastorage.googleapis.com/improbable-docs/docs2/reference/46ca90822ed1466a/assets/shared/operate/inspector/query-editor-module.png)

In most editors that come with game engines, the default scene view is a nested hierarchy based on parent/child relationships between objects. For example, this is the Hierarchy view in the PlayCanvas editor:

![The Hierarchy view in the PlayCanvas editor](https://developer.playcanvas.com/images/user-manual/editor/hierarchy.png)

This is also the provided scene view in Unity (in the Hierarchy view) and Unreal (in the World Outliner). While this view is often comfortable and familiar for game developers, it maps poorly to ECS, where the vast majority of your game logic is oriented around flat lists of entities, grouped by which components they have. As such, the Query Editor in the SpatialOS inspector is a brilliant paradigm shift because it means that the graphical inspector for you game operates on the same logic that your systems (or workers, in the case of SpatialOS do): It queries the world with a set of constraints (usually the presence or absence of certain component types) and gets back a flat list of all entities matching the specified criteria.

## Modular, Composable Tools

> Output of one module becomes the input of another. Takes inspiration from visual scripting tools to model the flow of data. Putting the emphasis on *data* (as opposed to high-level views of "objects") is more in-line with the ECS paradigm.

## Different Visualizations for Different Situations

> Sometimes you want a viewport showing things moving around in realtime, sometimes you want a list of data, often you want both or multiples of each.