---
layout: post
title: "Using async/await in Unity"
permalink: /posts/unity-async/
---

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

[unitask]: https://github.com/Cysharp/UniTask	"The UniTask package for Unity"

[^sub-frame-await]: In theory, it would be possible to resume a task within the same frame by having it resume at a later part of the update loop, e.g. `await` during the main update and then resume during `LateUpdate`. However, this is probably not currently supported by UniTask and would probably not be something that is generally useful.

