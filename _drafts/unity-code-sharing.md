---
layout: post
title: "Sharing C# Code with Unity"
permalink: /posts/sharing-with-unity/
---

I've been doing some investigation into building client/server game architectures with Unity on the front end and a C# server on the back end. One of the major advantages of using the same programming language for both client and server would be the ability to share common game logic between the two, making it easier to have the client predict the outcome of events without having to duplicate logic.

Unfortunately, what I have found in doing so is that the Unity ecosystem is so divorced from the main C# ecosystem that sharing code in any meaningful way is difficult, if not entirely impossible.

I have broken this post up into two parts: A direct description of the technical issues that come with sharing code, and then a more subjective evaluation of how this impacts any projects that want to take this approach.

You can have a look at the [DotNetGamePrototype](https://github.com/randomPoison/DotNetGamePrototype) repository to see the example project used for testing.

> TODO: Clarify that this is talking about using a Unity front end with a non-Unity back end.

# Part One: Technical Issues

At the most basic level, most C# code will be source-compatible between a Unity project and a standalone .NET C# project. That is, if you copy-and-paste the source code from one to the other, it will almost certainly compile and run as expected. There are a few caveats to this, though:

* Any code outside of the [.NET Standard](https://github.com/dotnet/standard) must be present in both environments, i.e. if you reference class `Foo`, `Foo` must be defined in both projects.
* [Not all of the .NET Standard is available to Unity](https://docs.microsoft.com/en-us/dotnet/standard/net-standard#net-implementation-support). At the time of writing, Unity supports .NET Standard 2.0 but not 2.1, with [no ETA on when support will arrive](https://forum.unity.com/threads/net-standard-2-1.757007/#post-5047175).
* For certain platforms, Unity uses a special C# scripting backend called [IL2CPP](https://docs.unity3d.com/Manual/IL2CPP.html). This backend has additional restrictions on what you can do, and only supports [a subset of the .NET Standard](https://docs.unity3d.com/Manual/ScriptingRestrictions.html). If building for platforms that require IL2CPP (such as iOS and most consoles), any shared code must limit itself to the supported subset of functionality.

Meeting these requirements only enables a bare minimum of source compatibility, though. Once you have some C# code that you want to share between projects, you'll need some method of making that code available in both contexts. There are two main ways to share code between the projects:

* **Define the shared code as both a UPM package and a standalone C# project** (i.e. add a `package.json` and a `.csproj` file to the directory) and then add the shared code as a direct dependency for both projects.
* **Only define the shared code as a standalone C# project**. Add it as a direct dependency to the server project, and add the built DLLs to Unity.

## Sharing Code Directly

The easiest way to share code is to setup your shared code with the necessary configuration for both a standalone .NET project (i.e. a `.csproj` file) and a Unity package (i.e. a [`package.json`](https://docs.unity3d.com/Manual/upm-manifestPkg.html) file and an [Assembly Definition file](https://docs.unity3d.com/Manual/ScriptCompilationAssemblyDefinitionFiles.html)). If you keep your client and server in the same repository, you can reference the shared project from both via relative paths. For example, here's how I did it in the [example Unity client](https://github.com/randomPoison/DotNetGamePrototype/blob/fb29ae47501e927044a5afe4f068e438c0bcaed5/DotNetGameClient/Packages/manifest.json#L12) and [example server project](https://github.com/randomPoison/DotNetGamePrototype/blob/fb29ae47501e927044a5afe4f068e438c0bcaed5/DotNetGameServer/DotNetGame.csproj#L17).

The advantage of this approach over the others is simplicity: It requires no extra tooling, no build steps to copy build results or publish packages. You can modify the shared code from both your server project and your Unity project and the changes will immediately show up in both. If keeping your client and server in the same repository is a viable solution for you, then this is probably the best approach to take.

The drawback of this approach is that you end up polluting your shared code project with Unity-specific details. In addition to the `package.json` and assembly definition file, Unity requires that there be a `.meta` for every file in the package. Unity generates these meta files for you, but that will require you to open Unity every time you add a new file in order to ensure the meta file is generated correctly. These meta files also clutter the files list in your editor (though, with some extra configuration, you can usually configure it to ignore them).

This drawback is relatively minor, and more one of project cleanliness more than a technical issue. Still, it highlights the fact that this solution is a hack, rather than a well-supported use case for Unity.

## Exporting a DLL

If you want to avoid polluting your shared codebase with Unity-specific configuration and files, you can instead build your shared library as a DLL and import that DLL into your Unity project. So long as you are meeting the constraints listed above, the DLL generated from your project can be loaded into a Unity project without issue, including ones that target IL2CPP platforms.

> TODO: Further investigate how well this works in practice. In particular, how well are Nuget dependencies supported in practice?
>
> * [Copy all dependencie into the build output](https://stackoverflow.com/a/43841481).
> * [MSBuild targets](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-targets?view=vs-2019) (make sure you only target `netstandard2.0`).
> * [Use `dotnet publish` to spit out final DLLs](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish?tabs=netcore21).
> * Need to figure out out to setup the output paths correctly.
> * Is it possible to split out the generated documentation XML?

## Incompatible Dependency Management

The above approaches work well when your shared code only depends on the .NET Standard, but things quickly begin to break down when you introduce additional dependencies. This could mean adding a Nuget package as a dependency, adding a dependency on a UPM package, or even just adding another internal shared package that is also referenced by the first shared package.

The key issue is that Unity uses a different dependency management system than the rest of the C# ecosystem. While the .NET ecosystem at large uses [NuGet](https://docs.microsoft.com/en-us/nuget/what-is-nuget), Unity has historically had no proper package management system, opting to manually copy source code (or sometimes pre-built DLLs) directly into each project. Starting with the 2018.3 release Unity has introduced their own [Unity Package Manager](https://docs.unity3d.com/Manual/Packages.html) as a more robust solution for dependency management.

While UPM is a massive improvement over the previous (non-)solution, it doesn't support loading NuGet packages, making interop between the two ecosystems difficult. Unless you're willing to forego using any code outside of the .NET Standard, you'll want to be able to add dependencies to your shared package via NuGet. Once you do, though, you'll have a hard time getting your code to still work in Unity.

If you're taking the shared package approach described above, Unity won't pull down your NuGet dependencies and your code won't compile. There are a couple of community-made tools for pulling down NuGet packages (such as [UnityNuGet](https://github.com/xoofx/UnityNuGet) and [NuGetForUnity](https://github.com/GlitchEnzo/NuGetForUnity)), but it is unclear if any solution is robust enough to be a reliable solution for projects looking to pull in NuGet packages.

On the other hand, building your code as a DLL to import into Unity also has problems. The way dependencies are handled with building a .NET library (in C# or otherwise) is to *not* include dependencies in the built DLL. Instead, you're expected to have as part of your deployment process a way of providing those dependencies to the runtime when running your final executable. For most .NET applications, this is just handled by C# and the .NET development toolchain. The best solution I've found for this so far is to use the `<CopyLocalLockFileAssemblies>` element in your `.csproj`, making the DLLs for all dependencies immediately available alongside the DLL for your project:

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

From there, it's relatively easy to have your built DLL and all of its dependencies added to your Unity project:

```
dotnet publish -c Release -o ..\UnityProject\Packages\my-package
```

While this seems to work well if you have a single locally-maintained C# package that you're importing into Unity project, it doesn't scale up to a more complex project setup. For example, if you were two have to different projects both depending on `SomePackage`, each one would pull a copy of `SomePackage.dll` into your Unity project and your project will fail to build due to the duplicate DLLs. There are ways to work around this if there are only a few conflicts, but it's not clear if any such solution would scale well with a large tree of dependencies.

## Incompatible Software Ecosystems

While the above solution works to get dependencies exported into a Unity project, there are deeper incompatibilities to contend with once you've gotten to that point. As noted previously, Unity only supports a subset of valid C#/.NET code on all platforms. At the most basic, you'll only be able to use NuGet packages that support .NET Standard 2.0, which not all packages do. Fortunately, it should be possible to avoid including such packages in the first place by specifying `netstandard2.0` as the target framework in your shared package's `.csproj`.

Things get more tricky when dealing with the restrictions imposed by IL2CPP and the other platform-specific restrictions that Unity projects need to deal with. According to [Unity's documentation](https://docs.unity3d.com/Manual/ScriptingRestrictions.html), there are a number of things things that are perfectly valid in regular .NET development that will fail in Unity projects built with IL2CPP:

* The contents of `System.Reflection.Emit` are explicitly not supported on platforms that do not support just-in-time compilation. iOS is the prime example of this, though as I understand it some consoles also have this restriction.
* The compiler will aggressively remove any code that is never referenced (i.e. a class that is never instantiated). This interacts badly with reflection-based serialization, where a given class may only ever be instantiated via reflection. In this case you can [manually tell the compiler to not strip a class](https://docs.unity3d.com/Manual/IL2CPP-BytecodeStripping.html), but that can be difficult to do if the missing class is hidden in the internals of a pre-compiled DLL.
* Generic virtual methods also interact badly with ahead-of-time compilation. There are [hacky workarounds](https://docs.unity3d.com/Manual/ScriptingRestrictions.html) for dealing with this when you know all of the concrete instantiations, but this can again be difficult when the specifics are hidden in a pre-compiled DLL that you're pulling in from a dependency.

Additionally, there are platform-specific restrictions unique to the set of platforms supported by Unity that don't get taken into account by most (or any) packages published to NuGet. Especially, when publishing to the web you'll run into various restrictions that no other .NET environment has to deal with:

* Not all platforms support threads, so any code that relies on threads will fail at runtime.
* System resources don't behave the same on all platforms. On the web, browser sandboxing means that very few system resources are accessible at all. In some cases Unity can fake these for you (as is the case with how Unity fakes the existence of a file system), in other cases those APIs will simply fail at runtime. On mobile and console platforms, you have only limited access to the file system, so a library that attempts to create files in the background (e.g. as a data cache) may fail unexpectedly.

In my limited experimentation, I have already run into a couple of cases where these limitations come up: The [Json.NET](https://www.newtonsoft.com/json) library and WebSocket handling.

Json.NET is [by far the most widely used NuGet package](https://www.nuget.org/stats), and is the de facto standard for JSON serialization in C# and the .NET ecosystem. It also [doesn't work with Unity](https://github.com/JamesNK/Newtonsoft.Json/issues/1440). There are [multiple](https://github.com/jilleJr/Newtonsoft.Json-for-Unity) [ports](https://assetstore.unity.com/packages/tools/input-management/json-net-for-unity-11347) [out there](https://github.com/SaladLab/Json.Net.Unity3D) in various states of abandonment or disrepair, but none of them are available via NuGet so they can't be shared with a non-Unity C# library. In order to use Json.NET in your shared code, you have to setup a system where it is pulled in via NuGet when used in your server code, and then pulled in by a different method in your Unity project. This is doable, but if you use the setup described above to pull NuGet dependencies into your Unity project automatically you're going to run into conflicts fast. This could be possibly be fixed if the upstream library were setup to better support Unity, but there's been no indication that the maintainer is interested in taking on that work. This solution also only works because Json.NET is popular enough to have community-maintained forks that work with Unity, you likely won't have the same luck with smaller libraries.

In the case of WebSockets, it's actually *impossible* to provide a NuGet package that supports Unity.

# Part Two: Assessment

> Coming ~~soon~~ eventually!