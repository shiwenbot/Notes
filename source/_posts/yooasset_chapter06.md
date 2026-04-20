---
title: "【教材】第6章：资源加载核心"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 6 章：资源加载核心

## 概述

当我们在游戏代码中调用 `LoadAssetAsync` 时，从 API 请求到最终拿到 Unity 的 Asset 对象，这中间经历了一条精心设计的加载链路。这条链路的设计既要保证资源的正确加载，又要兼顾内存效率和并发控制。

本章我们将深入这条链路的核心部分：ResourceManager 如何作为调度中枢管理所有加载请求，ProviderOperation 如何通过状态机和引用计数控制加载生命周期，以及不同类型的 Provider 如何处理各类资源。最后，我们还会探讨 Handle 体系的设计，以及 YooAsset 在引用计数和垃圾回收之间的权衡思考。

## ResourceManager：调度中枢

ResourceManager 是资源加载系统的调度中枢，它维护着两个核心字典：`ProviderDic` 和 `LoaderDic`。前者存储所有资源提供者，后者存储所有 Bundle 文件加载器。通过这两个字典的复用机制，系统能够避免重复加载相同的资源和 Bundle 文件。

### Provider 字典复用机制

当我们调用 `LoadAssetAsync` 加载一个资源时，ResourceManager 会首先检查该资源是否已经有对应的 Provider 在运行：

```csharp:Runtime/ResourceManager/ResourceManager.cs [160:190]
public AssetHandle LoadAssetAsync(AssetInfo assetInfo, uint priority)
{
    if (LockLoadOperation)
    {
        string error = $"The load operation locked !";
        YooLogger.Error(error);
        CompletedProvider completedProvider = new CompletedProvider(this, assetInfo);
        completedProvider.SetCompletedWithError(error);
        return completedProvider.CreateHandle<AssetHandle>();
    }

    if (assetInfo.IsInvalid)
    {
        YooLogger.Error($"Failed to load asset ! {assetInfo.Error}");
        CompletedProvider completedProvider = new CompletedProvider(this, assetInfo);
        completedProvider.SetCompletedWithError(assetInfo.Error);
        return completedProvider.CreateHandle<AssetHandle>();
    }

    string providerGUID = nameof(LoadAssetAsync) + assetInfo.GUID;
    ProviderOperation provider = TryGetAssetProvider(providerGUID);
    if (provider == null)
    {
        provider = new AssetProvider(this, providerGUID, assetInfo);
        provider.InitProviderDebugInfo();
        ProviderDic.Add(providerGUID, provider);
        OperationSystem.StartOperation(PackageName, provider);
    }

    provider.Priority = priority;
    return provider.CreateHandle<AssetHandle>();
}
```

这段代码展示了三个关键点：

第一，错误处理前置。在创建 Provider 之前，系统会检查 `LockLoadOperation` 标志和 `AssetInfo` 的有效性，如果出现错误会立即返回一个 `CompletedProvider`，该 Provider 已经处于失败状态。这种设计避免了后续流程中的异常判断，使错误处理更加集中。

第二，Provider 的唯一标识。系统使用 `LoadAssetAsync` 加载方法名加上资源的 GUID 作为 Provider 的唯一标识。这意味着同一个资源的不同加载方式（比如 `LoadAssetAsync` 和 `LoadAllAssetsAsync`）会创建不同的 Provider，因为它们的加载行为和返回结果都不同。

第三，Provider 复用。如果字典中已经存在相同 GUID 的 Provider，系统会直接复用它，而不是创建新的。这种复用机制在多个地方同时请求同一资源时特别有用，避免了重复的 Bundle 加载和资源反序列化。

### 五类 Load 方法的入口分发

ResourceManager 提供了五类公开的加载方法，每类方法都有对应的 Provider 实现：

1. **LoadAssetAsync** → AssetProvider：加载单个资源对象
2. **LoadSubAssetsAsync** → SubAssetsProvider：加载子资源对象（例如 SpriteAtlas 中的多张图片）
3. **LoadAllAssetsAsync** → AllAssetsProvider：加载 Bundle 中的所有资源
4. **LoadRawFileAsync** → RawFileProvider：加载原生文件（返回 TextAsset）
5. **LoadSceneAsync** → SceneProvider：加载场景

除了场景加载外，其他四类方法的实现逻辑高度相似。它们都会先生成 Provider 的唯一标识，然后尝试从字典中获取已存在的 Provider。如果不存在，就创建新的 Provider 并启动它。

场景加载的特殊之处在于，每次加载场景都会创建新的 Provider，即使加载的是同一个场景文件：

```csharp:Runtime/ResourceManager/ResourceManager.cs [140:154]
string providerGUID = $"{assetInfo.GUID}-{++_sceneCreateIndex}";
ProviderOperation provider;
{
    provider = new SceneProvider(this, providerGUID, assetInfo, loadSceneParams, suspendLoad);
    provider.InitProviderDebugInfo();
    ProviderDic.Add(providerGUID, provider);
    OperationSystem.StartOperation(PackageName, provider);
}

provider.Priority = priority;
var handle = provider.CreateHandle<SceneHandle>();
handle.PackageName = PackageName;
SceneHandles.Add(handle);
return handle;
```

这是因为同一个场景可能被多次加载（例如多个玩家同时进入不同的副本场景），每次加载都需要独立的 Provider 和 Handle。系统通过自增的 `_sceneCreateIndex` 来确保每个场景加载请求都有唯一的标识。

### ProviderOperation 的创建时机

从上面的代码可以看到，ProviderOperation 的创建时机有两个关键点：在 `LoadAssetAsync` 等方法被调用时立即创建，以及在 `OperationSystem.StartOperation` 被调用时开始执行。

```csharp:Runtime/ResourceManager/ResourceManager.cs [184:186]
provider = new AssetProvider(this, providerGUID, assetInfo);
provider.InitProviderDebugInfo();
ProviderDic.Add(providerGUID, provider);
OperationSystem.StartOperation(PackageName, provider);
```

这种设计的优势在于，Provider 的创建和启动是分离的。创建时只是构建对象并加入字典，而启动时才会触发真正的异步加载流程。这为后续的优先级调整、批量控制等操作留下了空间。

![资源加载调用链](../diagrams/ch06-load-chain.png)

*图 6-1：资源加载调用链时序图*

从图中可以看到，从用户调用 API 到最终获取资源对象，中间经历了 ResourceManager 的调度、Provider 的执行、Bundle 文件的加载等多个环节。每个环节都有自己的职责边界，形成了一个清晰的分层架构。

## ProviderOperation：状态机与引用计数

ProviderOperation 是所有资源提供者的基类，它定义了加载流程的核心状态机，并通过引用计数机制管理资源的生命周期。理解这两个机制，是掌握 YooAsset 加载系统的关键。

### 状态机流转

ProviderOperation 定义了一个五状态的状态机：

```csharp:Runtime/ResourceManager/Provider/ProviderOperation.cs [11:18]
protected enum ESteps
{
    None = 0,
    StartBundleLoader,
    WaitBundleLoader,
    ProcessBundleResult,
    Done,
}
```

这五个状态的流转逻辑在 `InternalUpdate` 方法中实现：

```csharp:Runtime/ResourceManager/Provider/ProviderOperation.cs [117:184]
internal override void InternalUpdate()
{
    if (_steps == ESteps.None || _steps == ESteps.Done)
        return;

    // 注意：未在加载中的任务可以挂起！
    if (IsLoading == false)
    {
        if (RefCount <= 0)
            return;
    }

    if (_steps == ESteps.StartBundleLoader)
    {
        foreach (var bundleLoader in _bundleLoaders)
        {
            bundleLoader.StartOperation();
            AddChildOperation(bundleLoader);
        }
        _steps = ESteps.WaitBundleLoader;
    }

    if (_steps == ESteps.WaitBundleLoader)
    {
        if (IsWaitForAsyncComplete)
        {
            foreach (var bundleLoader in _bundleLoaders)
            {
                bundleLoader.WaitForAsyncComplete();
            }
        }

        // 更新资源包加载器
        foreach (var bundleLoader in _bundleLoaders)
        {
            bundleLoader.UpdateOperation();
        }

        // 检测加载是否完成
        foreach (var bundleLoader in _bundleLoaders)
        {
            if (bundleLoader.IsDone == false)
                return;

            if (bundleLoader.Status != EOperationStatus.Succeed)
            {
                InvokeCompletion(bundleLoader.Error, EOperationStatus.Failed);
                return;
            }
        }

        // 检测加载结果
        BundleResultObject = _mainBundleLoader.Result;
        if (BundleResultObject == null)
        {
            string error = $"Loaded bundle result is null !";
            InvokeCompletion(error, EOperationStatus.Failed);
            return;
        }

        _steps = ESteps.ProcessBundleResult;
    }

    if (_steps == ESteps.ProcessBundleResult)
    {
        ProcessBundleResult();
    }
}
```

状态机的流转有几个关键点需要注意：

第一，挂起机制。当任务不在加载中（`IsLoading == false`）且引用计数为零时，任务会暂停执行。这是 YooAsset v2.3.16 引入的优化机制，允许尚未开始的任务在没有引用时暂停，而不是立即销毁。

第二，Bundle 加载的并行处理。在 `StartBundleLoader` 状态，系统会同时启动主 Bundle 和所有依赖 Bundle 的加载操作。这些加载操作会并行执行，充分利用网络带宽和本地 IO 能力。

第三，统一的错误处理。无论哪个 Bundle 加载失败，系统都会调用 `InvokeCompletion` 将状态设置为失败，并记录错误信息。这种统一的错误处理机制避免了复杂的嵌套判断，使代码更加清晰。

### RefCount 完整生命周期

引用计数是 ProviderOperation 生命周期管理的核心。每次创建 Handle 时，Provider 的引用计数会加一；每次释放 Handle 时，引用计数会减一。

```csharp:Runtime/ResourceManager/Provider/ProviderOperation.cs [244:260]
public T CreateHandle<T>() where T : HandleBase
{
    // 引用计数增加
    RefCount++;

    HandleBase handle = HandleFactory.CreateHandle(this, typeof(T));
    if (_resManager.UseWeakReferenceHandle)
    {
        var weakRef = new WeakReference<HandleBase>(handle);
        _weakReferences.AddLast(weakRef);
    }
    else
    {
        _handles.Add(handle);
    }
    return handle as T;
}
```

```csharp:Runtime/ResourceManager/Provider/ProviderOperation.cs [265:283]
public void ReleaseHandle(HandleBase handle)
{
    if (RefCount <= 0)
        throw new System.Exception("Should never get here !");

    if (_resManager.UseWeakReferenceHandle)
    {
        if (RemoveWeakReference(handle) == false)
            throw new System.Exception("Should never get here !");
    }
    else
    {
        if (_handles.Remove(handle) == false)
            throw new System.Exception("Should never get here !");
    }

    // 引用计数减少
    RefCount--;
}
```

RefCount 的生命周期分为四个阶段：

**创建阶段**：当 Provider 被创建时，RefCount 初始化为 0。此时 Provider 还没有实际的引用，处于"空闲"状态。

**引用阶段**：每次调用 `CreateHandle` 创建新的 Handle 时，RefCount 会加一。这意味着可能有多个 Handle 同时引用同一个 Provider，这种情况下 Provider 会保持活跃状态。

**释放阶段**：当用户调用 `Handle.Release()` 或 `Handle.Dispose()` 时，Provider 的 RefCount 会减一。如果减到零，Provider 就可以被销毁了。

**销毁阶段**：ResourceManager 会定期检查所有 Provider 的状态，当 RefCount 为零且不在加载中时，会调用 `DestroyProvider` 销毁 Provider。

![Handle 引用计数](../diagrams/ch06-refcount.png)

*图 6-2：Handle 引用计数生命周期*

从图中可以看到，一个 Provider 可以被多个 Handle 引用，每个 Handle 的创建和释放都会影响 RefCount。只有当所有 Handle 都被释放后，Provider 才能被销毁。

### v2.3.16 挂起机制 vs 旧版立即销毁

YooAsset 在 v2.3.16 版本引入了挂起机制，这是对旧版立即销毁机制的重要优化。

在旧版本中，当 RefCount 减到零时，Provider 会立即被销毁。这导致一个问题：如果用户在加载过程中提前释放了 Handle（例如在异步加载过程中取消了操作），Provider 会被立即销毁，即使加载任务还在进行中。

新版本通过挂起机制解决了这个问题。当 RefCount 为零时，Provider 不会立即销毁，而是进入挂起状态。只有当 Provider 不在加载中且 RefCount 为零时，才会真正销毁：

```csharp:Runtime/ResourceManager/Provider/ProviderOperation.cs [227:239]
public bool CanDestroyProvider()
{
    // 注意：正在加载中的任务不可以销毁
    if (IsLoading)
        return false;

    if (_resManager.UseWeakReferenceHandle)
    {
        TryCleanupWeakReference();
    }

    return RefCount <= 0;
}
```

这种设计有几个好处：

第一，避免了加载过程中的突然中断。即使 Handle 被提前释放，加载任务也能正常完成，只是结果不会被使用。

第二，支持弱引用模式。当启用弱引用时，系统会定期清理失效的弱引用，并自动调整 RefCount。这为用户提供了更灵活的内存管理方式。

第三，提高了系统的鲁棒性。即使在异常情况下（例如用户忘记释放 Handle），系统也能通过定期清理来避免内存泄漏。

## 五种 Provider 类型对比

YooAsset 定义了五种具体的 Provider 类型，每种类型对应不同的加载场景和返回结果。理解它们的区别，有助于我们在实际开发中选择正确的加载 API。

### 对比表格

| Provider 类型 | 加载 API | 典型场景 | 返回 Handle | 特殊行为 |
|--------------|---------|---------|------------|---------|
| AssetProvider | LoadAssetAsync | 加载单个 Prefab、Texture、AudioClip | AssetHandle | 返回单个 UnityEngine.Object |
| SubAssetsProvider | LoadSubAssetsAsync | 加载 SpriteAtlas 中的多张图片、多资源 AssetBundle | SubAssetsHandle | 返回 UnityEngine.Object[] |
| AllAssetsProvider | LoadAllAssetsAsync | 加载整个 Bundle 的所有资源 | AllAssetsHandle | 返回 Bundle 中所有资源 |
| RawFileProvider | LoadRawFileAsync | 加载 JSON、XML、二进制文件 | RawFileHandle | 返回 TextAsset，不进行反序列化 |
| SceneProvider | LoadSceneAsync | 加载游戏场景 | SceneHandle | 支持挂起加载，需手动 Unload |

这个表格展示了五种 Provider 的核心区别。在实际使用中，我们需要根据资源类型和加载需求选择合适的 API。

### CompletedProvider 的短路优化

除了上述五种 Provider 外，YooAsset 还定义了一个特殊的 `CompletedProvider`。它不继承自 `ProviderOperation`，而是直接实现了已经完成的操作状态：

```csharp:Runtime/ResourceManager/ResourceManager.cs [127:129]
CompletedProvider completedProvider = new CompletedProvider(this, assetInfo);
completedProvider.SetCompletedWithError(error);
return completedProvider.CreateHandle<AssetHandle>();
```

CompletedProvider 的作用是提供短路优化。当加载操作在创建 Provider 之前就已经确定失败（例如参数无效、操作被锁定），系统会直接返回一个处于失败状态的 CompletedProvider，而不是创建真正的 Provider 并启动加载流程。

这种设计避免了不必要的对象创建和异步操作开销，提高了系统的响应速度。

### AssetProvider 的加载流程

AssetProvider 是最常用的 Provider 类型，它的加载流程相对简单直接：

```csharp:Runtime/ResourceManager/Provider/AssetProvider.cs [11:42]
protected override void ProcessBundleResult()
{
    if (_loadAssetOp == null)
    {
        _loadAssetOp = BundleResultObject.LoadAssetAsync(MainAssetInfo);
        _loadAssetOp.StartOperation();
        AddChildOperation(_loadAssetOp);

#if UNITY_WEBGL
        if (_resManager.WebGLForceSyncLoadAsset)
            _loadAssetOp.WaitForAsyncComplete();
#endif
    }

    if (IsWaitForAsyncComplete)
        _loadAssetOp.WaitForAsyncComplete();

    _loadAssetOp.UpdateOperation();
    Progress = _loadAssetOp.Progress;
    if (_loadAssetOp.IsDone == false)
        return;

    if (_loadAssetOp.Status != EOperationStatus.Succeed)
    {
        InvokeCompletion(_loadAssetOp.Error, EOperationStatus.Failed);
    }
    else
    {
        AssetObject = _loadAssetOp.Result;
        InvokeCompletion(string.Empty, EOperationStatus.Succeed);
    }
}
```

这段代码展示了 AssetProvider 的核心逻辑：在 Bundle 加载完成后，通过 `BundleResultObject.LoadAssetAsync` 加载具体的资源对象，并等待加载完成。对于 WebGL 平台，系统提供了强制同步加载的选项，以解决 WebGL 平台的异步加载限制。

其他类型的 Provider 流程类似，区别在于 `ProcessBundleResult` 的具体实现不同。例如，SceneProvider 会加载场景，SubAssetsProvider 会加载所有子资源。

## LoadBundleFileOperation：依赖加载协调

资源加载的另一个重要环节是 Bundle 文件的加载。LoadBundleFileOperation 负责协调主 Bundle 和依赖 Bundle 的加载，确保所有依赖都满足后再进行后续操作。

### 主 Bundle + 依赖 Bundle 并行加载

当 ProviderOperation 被创建时，它会同时创建主 Bundle 和所有依赖 Bundle 的加载器：

```csharp:Runtime/ResourceManager/Provider/ProviderOperation.cs [88:111]
public ProviderOperation(ResourceManager manager, string providerGUID, AssetInfo assetInfo)
{
    _resManager = manager;
    ProviderGUID = providerGUID;
    MainAssetInfo = assetInfo;

    if (string.IsNullOrEmpty(providerGUID) == false)
    {
        // 主资源包加载器
        _mainBundleLoader = manager.CreateMainBundleFileLoader(assetInfo);
        _mainBundleLoader.AddProvider(this);
        _bundleLoaders.Add(_mainBundleLoader);

        // 依赖资源包加载器集合
        var dependLoaders = manager.CreateDependBundleFileLoaders(assetInfo);
        if (dependLoaders.Count > 0)
            _bundleLoaders.AddRange(dependLoaders);

        // 增加引用计数
        foreach (var bundleLoader in _bundleLoaders)
        {
            bundleLoader.Reference();
        }
    }
}
```

这段代码的关键点在于，主 Bundle 和依赖 Bundle 的加载是并行进行的。系统不会等待主 Bundle 加载完成后再加载依赖 Bundle，而是同时启动所有加载操作。

这种并行加载的设计充分利用了现代系统的并发能力。在多核 CPU 和高速网络环境下，并行加载可以显著缩短总加载时间。

### IBundleQuery.GetDependBundleInfos

依赖信息的获取是通过 `IBundleQuery` 接口完成的：

```csharp:Runtime/ResourceManager/ResourceManager.cs [306:316]
internal List<LoadBundleFileOperation> CreateDependBundleFileLoaders(AssetInfo assetInfo)
{
    List<BundleInfo> bundleInfos = _bundleQuery.GetDependBundleInfos(assetInfo);
    List<LoadBundleFileOperation> result = new List<LoadBundleFileOperation>(bundleInfos.Count);
    foreach (var bundleInfo in bundleInfos)
    {
        var bundleLoader = CreateBundleFileLoaderInternal(bundleInfo);
        result.Add(bundleLoader);
    }
    return result;
}
```

`IBundleQuery.GetDependBundleInfos` 返回资源所依赖的所有 Bundle 信息。这些依赖信息是在构建阶段生成的，存储在资源的清单文件中。系统通过读取清单文件来获取依赖关系，而不是在运行时分析。

这种设计有几个优势：

第一，运行时性能高。依赖关系在构建时已经确定，运行时只需要查询即可，不需要复杂的分析计算。

第二，支持复杂的依赖链。一个资源可能依赖多个 Bundle，这些 Bundle 又可能依赖其他 Bundle。通过预生成的依赖信息，系统能够正确处理任意复杂的依赖关系。

第三，支持依赖复用。多个资源可能依赖同一个 Bundle，系统通过 LoaderDic 字典来复用这些 Bundle 加载器，避免重复加载。

### 并发控制

LoadBundleFileOperation 在加载开始前会检查系统的并发状态：

```csharp:Runtime/ResourceManager/Operation/Internal/LoadBundleFileOperation.cs [68:80]
if (_steps == ESteps.CheckConcurrency)
{
    if (IsWaitForAsyncComplete)
    {
        _steps = ESteps.LoadBundleFile;
    }
    else
    {
        if (_resManager.BundleLoadingIsBusy())
            return;
        _steps = ESteps.LoadBundleFile;
    }
}
```

`BundleLoadingIsBusy` 方法会检查当前正在加载的 Bundle 数量是否超过设定的最大并发数：

```csharp:Runtime/ResourceManager/ResourceManager.cs [344:347]
internal bool BundleLoadingIsBusy()
{
    return BundleLoadingCounter >= _bundleLoadingMaxConcurrency;
}
```

这种并发控制机制有几个目的：

第一，避免系统过载。同时加载过多 Bundle 会导致内存占用过高和 IO 争用，反而降低整体性能。通过限制并发数，系统能够在资源利用率和性能之间找到平衡点。

第二，支持用户自定义。最大并发数可以通过初始化参数配置，不同的项目可以根据自己的硬件特性和性能需求调整这个值。

第三，区分同步和异步路径。对于强制同步等待的场景（`IsWaitForAsyncComplete`），系统会跳过并发检查，因为同步等待本身就会阻塞其他操作。

## Handle 体系

Handle 是用户与资源加载系统交互的唯一接口，它封装了 Provider 的复杂状态，提供了简洁的 API 和多种消费模式。理解 Handle 的设计，有助于我们正确使用 YooAsset 的加载 API。

### HandleBase 双接口设计

HandleBase 实现了两个接口：`IEnumerator` 和 `IDisposable`：

```csharp:Runtime/ResourceManager/Handle/HandleBase.cs [6:6]
public abstract class HandleBase : IEnumerator, IDisposable
```

`IEnumerator` 接口使 Handle 可以用于 Unity 的协程系统：

```csharp:Runtime/ResourceManager/Handle/HandleBase.cs [163:173]
bool IEnumerator.MoveNext()
{
    return !IsDone;
}
void IEnumerator.Reset()
{
}
object IEnumerator.Current
{
    get { return Provider; }
}
```

当我们在协程中 `yield return handle` 时，Unity 会不断调用 `MoveNext`，直到 `IsDone` 返回 true。这意味着协程会等待加载完成后再继续执行。

`IDisposable` 接口使 Handle 可以使用 `using` 语句进行自动资源管理：

```csharp:Runtime/ResourceManager/Handle/HandleBase.cs [37:40]
public void Dispose()
{
    this.Release();
}
```

当 `using` 块结束时，系统会自动调用 `Dispose`，进而调用 `Release` 释放资源。这种设计避免了忘记释放资源导致的内存泄漏。

### 三种消费模式代码示例对比

YooAsset 的 Handle 支持三种消费模式：await、yield return、callback。它们各有适用场景，我们可以根据具体需求选择。

**Await 模式**（推荐）：

```csharp
var handle = YooAssets.LoadAssetAsync<GameObject>("Assets/Prefabs/Enemy.prefab");
await handle;
var enemy = handle.InstantiateSync();
handle.Release();
```

**Yield Return 模式**：

```csharp
IEnumerator LoadEnemy()
{
    var handle = YooAssets.LoadAssetAsync<GameObject>("Assets/Prefabs/Enemy.prefab");
    yield return handle;
    var enemy = handle.InstantiateSync();
    handle.Release();
}
```

**Callback 模式**：

```csharp
var handle = YooAssets.LoadAssetAsync<GameObject>("Assets/Prefabs/Enemy.prefab");
handle.Completed += (handle) =>
{
    var enemy = handle.InstantiateSync();
    handle.Release();
};
```

三种模式的使用场景有所不同：

Await 模式是最现代的写法，代码简洁清晰，适合在异步方法中使用。它是 YooAsset 推荐的使用方式。

Yield Return 模式适合在传统的 Unity 协程中使用，特别是与旧代码集成时。

Callback 模式适合需要在加载完成后执行特定逻辑的场景，但要注意避免回调地狱。

### Release 与 Dispose 等价性

在 YooAsset 的早期版本中，`Release` 和 `Dispose` 是两个不同的方法。v2.3.10 版本统一了这两个方法，使 `Dispose` 成为 `Release` 的别名：

```csharp:Runtime/ResourceManager/Handle/HandleBase.cs [21:32]
public void Release()
{
    if (IsValidWithWarning == false)
        return;
    Provider.ReleaseHandle(this);

    // 主动卸载零引用的资源包
    if (Provider.RefCount == 0)
        Provider.TryUnloadBundle();

    Provider = null;
}

public void Dispose()
{
    this.Release();
}
```

这种统一的设计有几个好处：

第一，符合 .NET 的惯用法。.NET 开发者习惯使用 `Dispose` 来释放资源，统一的接口降低了学习成本。

第二，支持 `using` 语句。实现了 `IDisposable` 的类型可以使用 `using` 语句，这是 C# 推荐的资源管理模式。

第三，保持向后兼容。旧代码中无论使用 `Release` 还是 `Dispose`，都能正确工作，不需要大规模重构。

### AssetHandle 和 SceneHandle 的特殊方法

除了基础的方法外，不同类型的 Handle 还提供了特定类型的方法。

AssetHandle 提供了资源对象访问和实例化方法：

```csharp:Runtime/ResourceManager/Handle/AssetHandle.cs [53:72]
public UnityEngine.Object AssetObject
{
    get
    {
        if (IsValidWithWarning == false)
            return null;
        return Provider.AssetObject;
    }
}

public TAsset GetAssetObject<TAsset>() where TAsset : UnityEngine.Object
{
    if (IsValidWithWarning == false)
        return null;
    return Provider.AssetObject as TAsset;
}
```

SceneHandle 提供了场景操作方法：

```csharp:Runtime/ResourceManager/Handle/SceneHandle.cs [79:93]
public bool ActivateScene()
{
    if (IsValidWithWarning == false)
        return false;

    if (SceneObject.IsValid() && SceneObject.isLoaded)
    {
        return SceneManager.SetActiveScene(SceneObject);
    }
    else
    {
        YooLogger.Warning($"Scene is invalid or not loaded : {SceneObject.name}");
        return false;
    }
}
```

这些特定类型的方法为不同类型的资源提供了针对性的操作接口，使 API 更加语义化和易用。

## 设计思考

在深入了解了资源加载系统的各个组件后，我们来思考一下 YooAsset 的设计选择背后的原因。

### 为什么用 Dictionary 复用 Provider

使用 Dictionary 复用 Provider 而不是每次都创建新的，这是一个重要的设计决策。这个设计有几个关键好处：

第一，避免重复加载。当多个地方同时请求同一资源时，复用 Provider 可以避免重复的 Bundle 加载和资源反序列化，显著提高性能。

第二，节省内存。多个 Handle 共享同一个 Provider，只需要维护一份加载状态和资源对象，而不是每个 Handle 都创建独立的副本。

第三，保证一致性。所有 Handle 访问的是同一个资源对象，避免了多个副本之间的状态不一致问题。

当然，这种设计也有一定的复杂度。系统需要管理 Provider 的生命周期，确保在没有引用时正确销毁。YooAsset 通过引用计数机制解决了这个问题。

### 引用计数 vs GC 的权衡

在内存管理方面，YooAsset 选择了引用计数而不是完全依赖 GC。这个选择有几个考虑：

第一，精确控制。引用计数允许系统在资源不再被使用时立即释放，而不需要等待 GC 的不确定时间。对于内存敏感的游戏应用，这种精确控制很重要。

第二，避免 GC 峰值。游戏运行过程中，大量的 Handle 创建和销毁会导致 GC 的频繁触发。通过引用计数，系统能够更平稳地管理内存，减少 GC 对游戏流畅度的影响。

第三，支持弱引用。YooAsset 提供了弱引用模式，允许用户选择让 GC 来管理 Handle 的生命周期。这种灵活性为不同的使用场景提供了选择。

当然，引用计数也有它的缺点。最主要的问题是循环引用导致的内存泄漏。不过，在 YooAsset 的设计中，Handle 和 Provider 之间的关系是单向的（Handle 引用 Provider，Provider 不引用 Handle），因此不会出现循环引用的问题。

另一个缺点是引用计数的维护成本。每次创建和释放 Handle 都需要更新计数，这增加了一定的性能开销。不过，YooAsset 通过优化设计（例如使用 `HashSet` 而不是 `List` 来存储 Handle）将这个开销降到了最低。

## 小结

本章我们深入探讨了 YooAsset 资源加载的核心机制。从 ResourceManager 的调度逻辑，到 ProviderOperation 的状态机和引用计数，再到 Handle 的多种消费模式，我们看到了一个精心设计的加载系统。

这个系统的设计体现了几个核心原则：

第一，分层清晰。ResourceManager 负责调度，Provider 负责执行，Handle 负责交互，各层职责明确，便于理解和维护。

第二，性能优先。通过 Provider 复用、并行加载、并发控制等机制，系统在保证正确性的前提下最大化了性能。

第三，灵活可扩展。多种 Provider 类型、多种消费模式、强弱引用支持，这些设计使系统能够适应不同的使用场景。

在下一章中，我们将探讨 YooAsset 的另一个重要特性：下载系统的设计与实现。我们将看到，YooAsset 如何在资源下载和缓存之间找到平衡，以及如何处理网络中断、磁盘空间不足等各种异常情况。
