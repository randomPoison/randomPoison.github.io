---
layout: post
title: "The State of the Unity Package Ecosystem"
permalink: /posts/the-state-of-unity-packages/
---

> TODO: Do a good introduction here.

# The Bad Old Days

I started using Unity right at the tail end of the Unity 4 cycle, just before Unity 5 came out. At that time, Unity's only tool for sharing assets between Unity projects was [asset packages](https://docs.unity3d.com/Manual/AssetPackages.html). Asset packages are glorified zip archives containing a set of game assets (including code files!) and a pre-defined directory tree. When you import an asset package Unity merges the package's directory tree into your project's root `Assets` folder, giving you an option to preview which files were going to be imported ahead of time.

This system works well enough as a way to do one-off asset imports, but as you might imagine it's not a terribly good system for managing more complex dependencies, especially code dependencies. If a new version of the package is released and files or folders were moved in the new version, the import process doesn't handle removing the old versions of the assets leaving you with duplicates. Ideally a package will keep all of its assets under a single top-level directory so that you only need to upgrade a single folder before importing the new version, but in practice many packages fail to follow this convention. There's also nothing preventing you from making modifications to code/assets pulled in from these packages, and in practice such modifications are common in Unity projects. This makes upgrading packages doubly difficult because you now have to manage merging your changes with incoming changes.

There were also limited means of distributing such packages. The main place for hosting them was the [Unity Asset Store](https://assetstore.unity.com/), which also allowed for selling premade assets. It was also not uncommon to find open source projects on GitHub that provided pre-built asset packages for import. However it was also just as common to find projects on GitHub that *didn't* provide a pre-built package, where the recommended way of grabbing the code was to manually copy the contents into your project. There's also the [Unify Community Wiki](http://wiki.unity3d.com/index.php/Main_Page), which provides loads of code snippets for you to just, you know, copy-paste into your project.

So while there was undoubtedly useful utilities out there, and loads of developers trying to do their best with the tools available, I wouldn't say that Unity really had an *ecosystem* per se. The tooling available simply made it impossible to share common code dependencies in a way that would allow for large, reusable libraries to be built. Instead, everyone working on a Unity project had their own local copy of the same handful of common dependencies (usually with a couple of bespoke modifications).

Take, for example, JSON parsing. For a long time Unity didn't have built-in support for JSON serialization, and even now the support it has is very limited and not usable for many games. So instead most projects using JSON in some form end up having to pull in a separate JSON serialization library. The most common solution for a long time was SimpleJSON, which was posted on the [Unify wiki](http://wiki.unity3d.com/index.php/SimpleJSON). The far more robust [Json.NET](https://www.newtonsoft.com/json) was also [made into a Unity package](https://assetstore.unity.com/packages/tools/input-management/json-net-for-unity-11347) for the low, low price of $20 (the actual distribution of Json.NET has always been free). Of course, if you're making a Unity package that itself needs JSON parsing support, you can't assume that everyone using your package will already have SimpleJSON or Json.NET in their project (or which one they'll be using), so you need to provide your own copy of the JSON parsing library you're using. I've worked on a project that had no fewer than 6 JSON parsing implementations in various places, 3 of them copies of SimpleJSON!

# A New Hope

In the 2017.2 release Unity started [adding a new package manager](https://blogs.unity3d.com/2017/10/12/unity-2017-2-is-now-available/#wakkawakka), which [became available to users in the 2018.1 release](https://blogs.unity3d.com/2018/05/04/project-management-is-evolving-unity-package-manager-overview/). In its initial form, there wasn't official support for making custom packages (it was only being used to distribute Unity's own packages). However at least one clever person was able to [reverse engineer the package format](https://gist.github.com/LotteMakesStuff/6e02e0ea303030517a071a1c81eb016e), making it possible to start experimenting with the package manager early.

With the [2018.3 release](https://unity3d.com/unity/whats-new/unity-2018.3.0), Unity added official support for custom packages, as well as experimental support for distributing packages via Git and custom NPM servers. At this point the functionality was still largely undocumented, but was working well enough to start using in actual projects. Synapse has at least one project on Unity 2018.4 that relies on this functionality and has found that it worked well in practice.

Starting in 2019.1 Unity provided [official documentation for setting up custom packages](https://docs.unity3d.com/2019.1/Documentation/Manual/CustomPackages.html), and they've continued to flesh out the docs and improve on UPMs functionality throughout the 2019 release cycle. At this point, UPM provides enough functionality that building out an ecosystem of Unity packages is actual a viable prospect.

Well, at least in theory. In practice there's still one major hiccup that needs to be addressed...

# Hosting Packages

At the time of writing Unity *still* doesn't have an official way to host custom Unity packages. The officially-supported ways for pulling in package dependencies are:

* The local file system, either by dropping the package directly into your project's `Packages` folder or by manually specifying the path to the package on your local filesystem.
* Via Git, by specifying the URL of Git repository. This makes posting a package up on GitHub a pretty common way of sharing Unity packages.
* Via NPM (of all things). Unity's docs specifically [suggest hosting your own NPM package registry](https://docs.unity3d.com/Manual/cus-share.html).

The last option is, *in theory*, the best option since it doesn't force a dependency on Git (since not all projects are already using Git) and doesn't require users to vendor local copies of the package. However, hosting your own package registry is a hurdle for most developers who just want to share some useful utility code. Some intrepid folks have actually started hosting their packages [on NPM proper](https://www.npmjs.com/search?q=unity), which... is something.

Fortunately, it was only a matter of time before someone stepped in to provide a common package registry for Unity developers. Enter [OpenUPM](https://openupm.com/): An open source package registry with a built-in build pipeline for automatically deploying packages to the registry. Any UPM package hosted on GitHub can be added to the registry, and OpenUPM will build the package and host it for redistribution. It also provides a [nifty command line tool](https://openupm.com/docs/#scope-registry-and-command-line-tool) for adding and updating packages, since the "scoped registry" system for adding external packages can be tedious to update by hand.

OpenUPM is... a bit of a weird project. As far as package registries go, it's pretty odd to be able to publish other people's packages. The built-in build pipeline is also somewhat unusual