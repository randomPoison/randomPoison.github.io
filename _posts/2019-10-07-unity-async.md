---
layout: post
title: "Using async/await in Unity"
permalink: /posts/unity-async/
---

I've been investigating the usage of [C#'s async/await functionality](https://docs.microsoft.com/en-us/dotnet/csharp/async) in Unity projects, and in doing so I've done a number of experiments to determine the nuances of how it works in specific cases[^coroutines-are-bad]. This post attempts to list out and demonstrate these details so that others can better determine if using async/await makes sense for their Unity project.

All of these tests were done with Unity 2019.2.4f1. I can't guarantee that everything will behave the same on other versions of Unity, and the async/await support was known to be buggy in the 2017/2018 release cycles. You can view the various test scripts I wrote [on GitHub](https://github.com/randomPoison/unity-tests-and-examples/tree/async-await-examples/Assets/Scripts/AsyncTests).

A major caveat here: Some of the details highlighted are general details about task-based async code in C#, but some details are specific to how [UniTask][unitask] implements task support for Unity. If you choose to use a different implementation of tasks (or not use a custom task implementation at all), some of these details may be wrong.

## No Minimum Delay

One nice advantage of `await` is that it doesn't impose a 1 frame minimum delay the way that `yield return` does. This is something that I've run into when caching assets in-memory. For example:

```c#
public IEnumerator LoadPrefab(Action<GameObject> callback)
{
    if (!assetIsLoaded)
    {
        yield return LoadPrefabIntoCache();
    }
    
    var prefab = GetPrefabFromCache();
    callback(prefab);
}

// Some code that wants to use the coroutine;
GameObject prefab = null;
yield return LoadPrefab(result => { prefab = result; });
```

The first time coroutine runs you'll have to wait for the prefab to be loaded, however on subsequent calls it would be ideal if the code calling `LoadPrefab()` would resume on the same frame. Unfortunately, if you `yield return` on a coroutine, the calling code cannot resume until the next frame at the earliest, even if the invoked coroutine completes synchronously. If you need to avoid the 1 frame delay, you have to manually check if the work would complete synchronously:

```c#
GameObject prefab = null;
if (IsPrefabCached)
{
    prefab = GetPrefabFromCache();
}
else
{
    yield return LoadPrefab(result => { prefab = result; });
}
```

Fortunately, `await` doesn't have this restriction; If you `await` a task that completes synchronously, the calling task will also resume synchronously. It's worth noting, though, that if you `await` a task that doesn't complete synchronously, your task won't resume until the next frame even if the child task completes within the same frame[^sub-frame-await].

## Execution Order Stays The Same

When starting a coroutine, the body of the coroutine will synchronously execute up to the first `yield` statement:

```c#
public IEnumerator MyCoroutine()
{
    Debug.Log("Beginning of MyCoroutine()");
    yield return null;
    Debug.Log("End of MyCoroutine()");
}

// The following test:
Debug.Log("Before coroutine");
StartCoroutine(MyCoroutine());
Debug.Log("After coroutine");

// Will print the following:
//
// Before coroutine
// Beginning of MyCoroutine()
// After coroutine
// End of MyCoroutine()
```

Tasks work the same way, synchronously executing up to the first `await` before returning to the calling code:

```c#
private async void VoidTask()
{
    Debug.Log("Doing a void task!");
    await UniTask.Delay(1000);
    Debug.Log("Void task resumed!");
}

// The following test:
Debug.Log("About to call VoidTask()");
VoidTask();
Debug.Log("Returned from VoidTask()");

// Will print the following:
//
// About to call VoidTask()
// Doing a void task!
// Returned from VoidTask()
// Void task resumed!
```

## Starting and Stopping Tasks

There are two major differences between coroutines and tasks that you'll need to be aware of:

* Tasks start automatically as soon as you call an `async` function, there's no `StartCoroutine()` equivalent for tasks.
* Tasks are not tied to a `GameObject` the way that coroutines are, and will continue to run even if the the object that spawned them is destroyed.

For starting tasks, this change is convenient and removes some of the confusion that made coroutines difficult to work with (e.g. I've seen many people not call `StartCoroutine()` and then be confused why their coroutine wasn't running).

By default, a task will end automatically once it runs to completion. However, if you want to be able to cancel a task before it completes you'll need to [use a `CancellationToken`](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-cancel-a-task-and-its-children):

```c#
cancellation = new CancellationTokenSource();
try
{
    await UniTask.Delay(500, cancellationToken: cancellation.Token);
}
finally
{
    cancellation.Dispose();
    cancellation = null;
}

// Somewhere else, in reaction to an event that
// should cancel the pending task:
cancellation.Cancel();
```

This approach requires a bit more setup than is needed with coroutines (mainly to handle disposing of the `CancellationTokenSource` when done), but makes the default behavior more intuitive and gives you better control over when your tasks run.

### Cancel Task When Game Object is Destroyed

For example, imagine a scenario where you want to load a sprite from an asset bundle and assign it to a sprite renderer on a game object. If the game object is destroyed while waiting for the asset to load, the subsequent attempt to update the sprite renderer will throw an exception:

```c#
var sprite = await assetBundle.LoadAssetAsync<Sprite>();

// This will throw an exception of the game object was
// destroyed while waiting for the sprite to load.
spriteRenderer.sprite = sprite;
```

To cancel the task in the case that the game object is destroyed, you can use a `CancellationTokenSource` and the `RegisterRaiseCancelOnDestroy()` extension method:

```c#
var cancellation = new CancellationTokenSource();
cancellation.RegisterRaiseCancelOnDestroy(this);

// This await will never resume if the game object
// is destroyed.
var sprite = await assetBundle
    .LoadAssetAsync<Sprite>()
    .ConfigureAwait(cancellationToken: cancellation.Token);
spriteRenderer.sprite = sprite;
```

### Perform Cleanup When Task is Cancelled

One of the problems with coroutines is that there's no way to detect if a coroutine has been cancelled, making it effectively impossible to perform cleanup or consistently maintain invariants in the face of arbitrary coroutine cancellation. Fortunately, with async/await you can use `try` blocks to perform any final cleanup logic the same way you would anywhere else:

```c#
try
{
    await SomeAsyncOperation();
}
finally
{
    // This logic will be run, even if the task
    // is cancelled.
}
```

This works with task cancellation as well, since cancellation is done by throwing an [`OperationCancelledException`](https://docs.microsoft.com/en-us/dotnet/api/system.operationcanceledexception). This also means that it's possible to run logic only in the case that the task was cancelled while letting exceptions propagate as normal:

```
try
{
    await SomeAsyncOperation();
}
catch (OperationCancelledException)
{
    // This logic only runs if the task was
    // cancelled. Be sure to re-throw the
    // exception so that any parent tasks are
    // cancelled as well.
    throw;
}
```

## Exceptions and Callstacks

With coroutines, each one behaves as its own top-level "stack", meaning that callstacks from within a coroutine don't show where the coroutine was spawned from. This also applies to exceptions, which don't unwind through a hierarchy of coroutines, making exceptions thrown in coroutines non-intuitive. This makes debugging errors in coroutines often very difficult, as you lose all context for logging and exceptions from within a coroutine. Tasks, on the other hand, are a first-class part of the language and so handle these cases much better.

Exceptions behave the same as with synchronous code, unwinding the stack through tasks and providing a trace of the path it followed. For example:

```c#
private async Task ThrowRecursive(int depth)
{
    if (depth == 0)
    {
        throw new Exception("Exception thrown from deep in the call stack");
    }
    else
    {
        await ThrowRecursive(depth - 1);
    }
}
```

Calling this as `await ThrowRecursive(3)` shows the following in the console:

![Exception stack trace showing up in the Unity editor logs.](../../assets/unity-async-exception-log.png)

You can see the full stack of calls, from the top-level function that called `ThrowRecursive()` down through the multiple `await` statements. This also applies to any call stacks generated, including the ones included in debug logging. This makes debugging with async/await far easier than with coroutines.

## Compatibility with Coroutines

UniTask provides functionality for awaiting a coroutine within tasks and for yielding on tasks within coroutines. Using `await` with an `IEnumerator` an `AsyncOperation` or a `YieldInstruction` will work without issue. If you want to include a cancellation token or otherwise configure how the task will wait for the coroutine, you can use the `ConfigureAwait()` helper method:

```c#
await Resources.Load("MyAsset").ConfigureAwait(cancellationToken: token);
```

To `yield return` a `Task` or `UniTask` you must use the `ToCoroutine()` extension method:

```c#
yield return MyTask().ToCoroutine();
```

## Debugging

When debugging with Visual Studio, you can step over `await` statements in the debugger and it will step to the next line![^step-over-await-crash] This is because tasks are a first-class part of the language, so the debugger is able to track them directly and better determine when to break in the debugger. Coroutines, on the other hand, aren't really a part of the language and are a hacky misuse of [C# iterators](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/yield), and so the debugger can't "see through" them the way it can with tasks.

[unitask]: https://github.com/Cysharp/UniTask	"The UniTask package for Unity"

[^coroutines-are-bad]: Years of experience with coroutines and their *nuances* have made me very wary of all the ways that Unity can surprise you when it comes to async code.
[^sub-frame-await]: In theory, it would be possible to resume a task within the same frame by having it resume at a later part of the update loop, e.g. `await` during the main update and then resume during `LateUpdate`. However, this is probably not currently supported by UniTask and would probably not be something that is generally useful.
[^step-over-await-crash]: There's [currently a bug](https://github.com/Cysharp/UniTask/issues/41) with the Unity editor and `UniTask` that is causing the editor to crash when stepping over `await` in an `async UniTask` function. This shouldn't be an issue if you're using `async Task`, though.

