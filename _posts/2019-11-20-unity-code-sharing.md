---
layout: post
title: "Sharing C# Code with Unity"
permalink: /posts/sharing-with-unity/
---

I've been doing some investigation into building client/server game architectures with Unity on the front end and a C# server on the back end, specifically pairing an [ASP.NET](https://dotnet.microsoft.com/apps/aspnet) server in a traditional .NET environment with a Unity front-end. For games with no realtime gameplay and a desire for high scalability, as is often the case with online mobile games (my industry, for better or worse), it makes sense to build your server with something other than Unity. Traditional .NET development is attractive in this case because it allows you to use the same programming language, C#, to build both your client and server.

In particular, one of the major advantages of using the same programming language for both client and server would be the ability to share common game logic between the two. Doing so has major advantages over having the two codebases be completely isolated:

* Shared definitions for data objects and serialization logic greatly simplifies communication between client and server, reducing errors while making it easier to evolve your API contract.
* Sharing common game logic means that you can do client-side prediction and keep your game experience smooth in the face of a slow network connection. This is useful even for non-realtime games!
* Generally reduced development time, since logic written for the server can be reused in the client (and vice versa).

You can have a look at the [DotNetGamePrototype](https://github.com/randomPoison/DotNetGamePrototype) repository to see the example project I've been building as part of this investigation.

I have broken this post up into two parts: A direct description of the technical issues that come with sharing code, along with potential ways to deal with these issues, and then a more subjective evaluation of how this impacts any projects that want to take this approach. If you care primarily about my final conclusions, skip to the second part below.

# Part One: Technical Issues

At the most basic level, most C# code will be source-compatible between a Unity project and a standalone .NET C# project. That is, if you copy-and-paste the source code from one to the other, it will almost certainly compile and run as expected. There are a few caveats to this, though:

* Any code outside of the [.NET Standard](https://github.com/dotnet/standard) must be present in both environments, i.e. if you reference class `Foo`, `Foo` must be defined in both projects.
* [Not all of the .NET Standard is available to Unity](https://docs.microsoft.com/en-us/dotnet/standard/net-standard#net-implementation-support). At the time of writing, Unity supports .NET Standard 2.0 but not 2.1, with [no ETA on when support will arrive](https://forum.unity.com/threads/net-standard-2-1.757007/#post-5047175).
* For certain platforms, Unity uses a special C# scripting backend called [IL2CPP](https://docs.unity3d.com/Manual/IL2CPP.html). This backend has additional restrictions on what you can do, and only supports [a subset of the .NET Standard](https://docs.unity3d.com/Manual/ScriptingRestrictions.html). If building for platforms that require IL2CPP (such as iOS and most consoles), any shared code must limit itself to the supported subset of functionality.

Meeting these requirements only enables a bare minimum of source compatibility, though. Once you have some C# code that you want to share between projects, you'll need some method of making that code available in both contexts. There are two main ways to share code between the projects:

* **Define the shared code as both a UPM package and a standalone C# project** (i.e. add a `package.json` and a `.csproj` file to the directory) and then add the shared code as a direct dependency for both projects.[^text-sharing]
* **Only define the shared code as a standalone C# project**. Add it as a direct dependency to the server project, and add the built DLLs to Unity.

## Sharing Code Directly

The easiest way to share code is to set it up with the necessary configuration for both a standalone .NET project (i.e. a `.csproj` file) and a Unity package (i.e. a [`package.json`](https://docs.unity3d.com/Manual/upm-manifestPkg.html) file and an [Assembly Definition file](https://docs.unity3d.com/Manual/ScriptCompilationAssemblyDefinitionFiles.html)). If you keep your client and server in the same repository, you can reference the shared project from both via relative paths. This was how I did it in the [example Unity client](https://github.com/randomPoison/DotNetGamePrototype/blob/fb29ae47501e927044a5afe4f068e438c0bcaed5/DotNetGameClient/Packages/manifest.json#L12) and [example server project](https://github.com/randomPoison/DotNetGamePrototype/blob/fb29ae47501e927044a5afe4f068e438c0bcaed5/DotNetGameServer/DotNetGame.csproj#L17), and it took minimal effort to setup.

It's worth noting that setting up your code as a UPM package isn't strictly necessary. For example, you could directly embed your shared code directly in your Unity project and then reference it from your server code. The key part of this approach is that it shares the source files directly, rather than pre-building a DLL to import into Unity.

The advantage of this approach over most others is simplicity: It requires no extra tooling, no build steps to copy build results or publish packages. You can modify the shared code from both your server project and your Unity project and the changes will immediately show up in both. If keeping your client and server in the same repository is a viable solution for you, then this is probably the easiest approach.

The drawback of this approach is that you end up polluting your shared code project with Unity-specific details. In addition to the `package.json` and assembly definition file, Unity requires that there be a `.meta` for every file in the package. Unity generates these meta files for you, but that will require you to open Unity every time you add a new file in order to ensure the meta file is generated correctly. These meta files also clutter the files list in your editor (though, with some extra configuration, you can usually configure it to ignore them).

This drawback is relatively minor, and more one of project cleanliness more than a technical issue. Still, it highlights the fact that this solution is a hack, rather than a well-supported use case for Unity.

## Exporting a DLL

If you want to avoid polluting your shared codebase with Unity-specific configuration and files, you can instead build your shared library as a DLL and import that DLL into your Unity project. So long as you are meeting the constraints listed above, the DLL generated from your project can be loaded into a Unity project without issue, including ones that target IL2CPP platforms.

Exporting your project as a DLL to Unity is fairly simple:

```
dotnet publish -c Release -o ../UnityProject/Assets/Plugins
```

You'll have to remember to do this any time you update the shared project, or else setup some kind of automation to do so automatically. Such a solution is theoretically possible (at least, I see no technical blockers), but none exists already as far as I can see.

While this approach involves more built-time work, it has the advantage of enabling you to easily pull NuGet dependencies into your Unity project. You can add a `<CopyLocalLockFileAssemblies>` element in your `.csproj`, making the DLLs for all dependencies immediately available alongside the DLL for your project:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <CopyLocalLockFileAssemblies>true</CopyLocalLockFileAssemblies>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="SomePackage" Version="1.2.3" />
  </ItemGroup>
</Project>
```

When you run `dotnet publish`, it will also copy any NuGet packages that you're using in your shared project into the Unity project. Unfortunately, using NuGet packages within Unity has its own problems which we'll discuss below.

## Incompatible Dependency Management

The above approaches work well when your shared code only depends on the .NET Standard, but things quickly begin to break down when you introduce additional dependencies. This could mean adding a Nuget package as a dependency, adding a dependency on a UPM package, or even just adding another internal shared package that is also referenced by the first shared package.

The key issue is that Unity uses a different dependency management system than the rest of the C# ecosystem. While the .NET ecosystem at large uses [NuGet](https://docs.microsoft.com/en-us/nuget/what-is-nuget), Unity has historically had no proper package management system, opting to manually copy source code (or sometimes pre-built DLLs) directly into each project. Starting with the 2018.3 release Unity has introduced their own [Unity Package Manager](https://docs.unity3d.com/Manual/Packages.html) as a more robust solution for dependency management.

While UPM is a massive improvement over the previous (non-)solution, it doesn't support loading NuGet packages, making interop between the two ecosystems difficult. Unless you're willing to forego using any code outside of the .NET Standard, you'll want to be able to add dependencies to your shared package via NuGet. Once you do, though, you'll have a hard time getting your code to still work in Unity.

If you're taking the shared package approach described above, Unity won't pull down your NuGet dependencies and your code won't compile. There are a couple of community-made tools for pulling down NuGet packages (such as [UnityNuGet](https://github.com/xoofx/UnityNuGet) and [NuGetForUnity](https://github.com/GlitchEnzo/NuGetForUnity)), but it is unclear if any solution is robust enough to be a reliable solution for projects looking to pull in NuGet packages.

On the other hand, the approach described above for automatically copying NuGet dependencies into the Unity project seems to work well if you have a single locally-maintained C# package, but it doesn't scale up to a more complex project setup. For example, if you were two have two different projects both depending on `SomePackage`, each one would pull a copy of `SomePackage.dll` into your Unity project and your project will fail to build due to the duplicate DLLs. There are ways to work around this if there are only a few conflicts, but it's not clear if any such solution would scale well with a large tree of dependencies.

## Incompatible Software Ecosystems

Even if you find some solution for sharing NuGet packages with Unity, there are deeper incompatibilities to contend with. As noted previously, Unity only supports a subset of valid C#/.NET code on all platforms. At the most basic, you'll only be able to use NuGet packages that support .NET Standard 2.0, which not all packages do. Fortunately, it should be possible to avoid including such packages in the first place by specifying `netstandard2.0` as the target framework in your shared package's `.csproj`.

Things get more tricky when dealing with the restrictions imposed by IL2CPP and the other platform-specific restrictions that Unity projects need to deal with. According to [Unity's documentation](https://docs.unity3d.com/Manual/ScriptingRestrictions.html), there are a number of things things that are perfectly valid in regular .NET development that will fail in Unity projects built with IL2CPP:

* The contents of `System.Reflection.Emit` are explicitly not supported on platforms that do not support just-in-time compilation. iOS is the prime example of this, though as I understand it some consoles also have this restriction.
* The compiler will aggressively remove any code that is never referenced (i.e. a class that is never instantiated). This interacts badly with reflection-based serialization, where a given class may only ever be instantiated via reflection. In this case you can [manually tell the compiler to not strip a class](https://docs.unity3d.com/Manual/IL2CPP-BytecodeStripping.html), but that can be difficult to do if the missing class is hidden in the internals of a pre-compiled DLL.
* Generic virtual methods also interact badly with ahead-of-time compilation. There are [hacky workarounds](https://docs.unity3d.com/Manual/ScriptingRestrictions.html) for dealing with this when you know all of the concrete instantiations, but this can again be difficult when the specifics are hidden in a pre-compiled DLL that you're pulling in from a dependency.

Additionally, there are platform-specific restrictions unique to the set of platforms supported by Unity that don't get taken into account by most (or any) packages published to NuGet. Especially, when publishing to the web you'll run into various restrictions that no other .NET environment has to deal with:

* Not all platforms support threads, so any code that relies on threads will fail at runtime.
* System resources don't behave the same on all platforms. On the web, browser sandboxing means that very few system resources are accessible at all. In some cases Unity can fake these for you (as is the case with how Unity fakes the existence of a file system), in other cases those APIs will simply fail at runtime. On mobile and console platforms, you have only limited access to the file system, so a library that attempts to create files in the background (e.g. as a data cache) may fail unexpectedly.

In my limited experimentation, I have already run into a couple of cases where these limitations come up: The [Json.NET](https://www.newtonsoft.com/json) library and WebSocket handling.

Json.NET is [by far the most widely used NuGet package](https://www.nuget.org/stats), and is the de facto standard for JSON serialization in C# and the .NET ecosystem. It also [doesn't work with Unity](https://github.com/JamesNK/Newtonsoft.Json/issues/1440). There are [multiple](https://github.com/jilleJr/Newtonsoft.Json-for-Unity) [ports](https://assetstore.unity.com/packages/tools/input-management/json-net-for-unity-11347) [out there](https://github.com/SaladLab/Json.Net.Unity3D) in various states of abandonment or disrepair, but none of them are available via NuGet so they can't be shared with a non-Unity C# library. In order to use Json.NET in your shared code, you have to setup a system where it is pulled in via NuGet when used in your server code, and then pulled in by a different method in your Unity project. This is doable, but if you use the setup described above to pull NuGet dependencies into your Unity project automatically you're going to run into conflicts fast. This could be possibly be fixed if the upstream library were setup to better support Unity, but there's been no indication that the maintainer is interested in taking on that work. This solution also only works because Json.NET is popular enough to have community-maintained forks that work with Unity, you likely won't have the same luck with smaller libraries.

In the case of WebSockets, it's actually *impossible* to provide a NuGet package that supports Unity on all platforms. When running in a browser, you can't open a socket directly. Instead, you have to use the [browser's WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API), creating C# bindings to the JavaScript API. And, of course, this setup only works in the browser so you'll need [to abstract over both the browser API and a native implementation](https://github.com/randomPoison/DotNetGamePrototype/tree/master/DotNetGameClient/Packages/com.synapse-games.websockets) for other platforms. You'll find plenty of WebSocket implementations on NuGet, but none that can be shared meaningfully between Unity and a .NET project. It's also worth highlighting that this issue isn't specific to WebSockets; Many system resources have similar caveats that will make NuGet packages incompatible with Unity. Generally you'll deal with this in the Unity project by abstracting over two or more platform-specific implementations, but this in turn presents complications for dependency management, in that you may only need a given dependency on certain platforms.

# Part Two: Assessment

Overall, my assessment of the situation is that code sharing between Unity and .NET works just well enough to be tempting to use, while being broken enough to present major obstacles to large scale development.

The fundamental issue is that, while the two are similar on the surface, Unity is not a true .NET environment. On some platforms Unity uses Mono to run your code, which at least means that your code will behave like regular C# at runtime, but IL2CPP is introduces huge problems for compatibility. Combine that with the fact that Unity has a completely bespoke solution for dealing with dependency management and has to handle certain platform-specific issues that other .NET runtimes don't (most notably ahead-of-time compilation and running in web browsers), and the picture we end up with is one where Unity is not just another .NET runtime, but its own separate thing that happens to be largely (but not completely!) source-compatible with .NET.

Source compatibility is the notable thing here: Being able to copy a piece of code between a Unity project and a .NET project sure *feels* like compatibility, so it's awfully tempting to say that code sharing is possible. And, as noted above, it is possible (easy, even!) at a small scale. What worries me is that nothing about this setup seems scalable.

It's easy enough to start writing some shared code and use one of the basic methods described above to integrate it into both projects. You can even get pretty far while using some small, simple packages off of NuGet. But eventually you'll get to a point where you want to use Json.NET or WebSockets or some other thing that needs a fundamentally different solution between .NET and Unity and you'll hit a wall. You'll have some piece of core functionality that should be shared between your client and server yet _can't be_. And at that point you'll be close to your ship date and too heavily invested in your current architecture to make major changes, so you'll hack around the problem and do what you can to get things working.

In some ways, this is a worse situation than being outright incompatible, because it provides the opportunity to start doing something now that is all-but-guaranteed to fail somewhere down the line. It feels like something that should *just work*; I'm writing the same code in my client and server projects, it seems silly to not be able to share code between the two. But on digging deeper into the situation, it becomes clear that the two have very different code environments: Different compilation processes, different runtime environment, different idiomatic solutions to common problems. The shared language gives the veneer of commonality, where in reality there is an ocean of difference.

It's worth noting, though, that this assessment is more a gut feeling than a complete analysis. In practice there are no hard blockers to sharing code, just a hundred little things that make it difficult. My experience as a software developer tells me that won't work out, that you'll spend more hacking around the problems than is worth it for the convenience of sharing code, that you'll not be able to use helpful libraries in your server code because they're not compatible with Unity. But, the only way to really know how bad things are would be to build out a real, large-scale, production-ready project and see what problems you run into.

That all being said, there are potentially ways in which better automation and tooling could improve the situation, maybe even enough to make the effort worthwhile:

* **Build out better support for resolving NuGet dependencies** in a Unity project. It's already possible to export a packages dependencies as a JSON file and include it in Unity, so a plugin that can fully resolve dependency trees, detect conflict, and import the dependencies into Unity would make interop much smoother. Ideally this would be built directly into UPM, but if it's not something Unity is willing to do, then it may be possible to make this work as a custom package.
* **Automatically detect incompatibilities with Unity**. You can use reflection and disassembly to inspect the contents of a .NET DLL. In theory, you could automate the process of detecting if the library uses any language features that don't work with IL2CPP. I have no idea if this is actually possible to do in a robust way, but it would make a huge difference to be able to detect these issues ahead of time.

I'm certainly not the only person interested in making this work, so hopefully someone with more time (and more expertise with the wider .NET world) can make some progress here. Until then, I'll be investigating other avenues for client/server code sharing and see if I can't come up with a more satisfying solution.

