---
title: "【教材】第5章：初始化与运行模式"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 5 章：初始化与运行模式

## 概述

ResourcePackage 是 YooAsset 资源系统的核心入口，而初始化则是整个资源系统启动的全链路入口。如果说第 3 章讲的文件系统是"基础设施"，那么本章讲的初始化流程就是"基础设施的搭建过程"。

初始化的核心任务有三个：第一，根据运行模式创建对应的文件系统实例；第二，初始化这些文件系统（如创建缓存目录、验证内置资源完整性）；第三，为联机模式准备版本更新和清单更新的能力。

整个初始化流程采用了严谨的状态机设计，确保每一步都可控、可追踪。当初始化失败时，系统还提供了重试机制，让联机模式能够应对网络波动等异常情况。

## 五种运行模式详解

YooAsset 通过 `EPlayMode` 枚举定义了五种运行模式，每种模式对应不同的文件系统组合和使用场景。理解这些模式的差异，是正确使用 YooAsset 的第一步。

### 运行模式对应表

```csharp:Runtime/InitializeParameters.cs [8:34]
public enum EPlayMode
{
    /// <summary>
    /// 编辑器下的模拟模式
    /// </summary>
    EditorSimulateMode,

    /// <summary>
    /// 离线运行模式
    /// </summary>
    OfflinePlayMode,

    /// <summary>
    /// 联机运行模式
    /// </summary>
    HostPlayMode,

    /// <summary>
    /// WebGL运行模式
    /// </summary>
    WebPlayMode,

    /// <summary>
    /// 自定义运行模式
    /// </summary>
    CustomPlayMode,
}
```

### 各模式的文件系统组合

每种运行模式通过不同的 `InitializeParameters` 子类来配置，这些参数类指定了需要初始化的文件系统列表：

- **EditorSimulateMode**：使用 `EditorFileSystemParameters`，创建编辑器模拟文件系统，直接从 Assets 目录加载资源，无需打包。
- **OfflinePlayMode**：使用 `OfflinePlayModeParameters`，仅包含 `BuildinFileSystemParameters`，对应内置文件系统（StreamingAssets）。
- **HostPlayMode**：使用 `HostPlayModeParameters`，包含 `BuildinFileSystemParameters` 和 `CacheFileSystemParameters`，支持从内置文件系统读取，并缓存到本地文件系统。
- **WebPlayMode**：使用 `WebPlayModeParameters`，包含 `WebServerFileSystemParameters` 和 `WebRemoteFileSystemParameters`，专为 WebGL 平台设计。
- **CustomPlayMode**：使用 `CustomPlayModeParameters`，通过 `FileSystemParameterList` 自由组合多个文件系统。

```csharp:Runtime/InitializeParameters.cs [66:110]
public class EditorSimulateModeParameters : InitializeParameters
{
    public FileSystemParameters EditorFileSystemParameters;
}

public class OfflinePlayModeParameters : InitializeParameters
{
    public FileSystemParameters BuildinFileSystemParameters;
}

public class HostPlayModeParameters : InitializeParameters
{
    public FileSystemParameters BuildinFileSystemParameters;
    public FileSystemParameters CacheFileSystemParameters;
}

public class WebPlayModeParameters : InitializeParameters
{
    public FileSystemParameters WebServerFileSystemParameters;
    public FileSystemParameters WebRemoteFileSystemParameters;
}

public class CustomPlayModeParameters : InitializeParameters
{
    /// <summary>
    /// 文件系统初始化参数列表
    /// 注意：列表最后一个元素作为主文件系统！
    /// </summary>
    public readonly List<FileSystemParameters> FileSystemParameterList = new List<FileSystemParameters>();
}
```

### 离线模式初始化示例

离线模式适用于无需热更新的单机游戏，初始化代码非常简洁：

```csharp
// 创建离线运行模式的初始化参数
var initParameters = new OfflinePlayModeParameters();
initParameters.BuildinFileSystemParameters = new FileSystemParameters();
initParameters.BuildinFileSystemParameters.FileSystemRoot = Application.streamingAssetsPath;
initParameters.BuildinFileSystemParameters.FileSystemQuery = typeof(BuildinFileSystemQuery);

// 初始化包裹
var package = YooAssets.GetPackage("DefaultPackage");
var operation = package.InitializeAsync(initParameters);
yield return operation;

if (operation.Status != EOperationStatus.Succeed)
{
    Debug.LogError($"初始化失败：{operation.Error}");
}
```

### 联机模式初始化示例

联机模式支持热更新，需要配置内置和缓存两个文件系统：

```csharp
// 创建联机运行模式的初始化参数
var initParameters = new HostPlayModeParameters();

// 配置内置文件系统（只读，从 StreamingAssets 加载）
initParameters.BuildinFileSystemParameters = new FileSystemParameters();
initParameters.BuildinFileSystemParameters.FileSystemRoot = Application.streamingAssetsPath;
initParameters.BuildinFileSystemParameters.FileSystemQuery = typeof(BuildinFileSystemQuery);

// 配置缓存文件系统（读写，从持久化目录加载）
initParameters.CacheFileSystemParameters = new FileSystemParameters();
initParameters.CacheFileSystemParameters.FileSystemRoot = Application.persistentDataPath;
initParameters.CacheFileSystemParameters.FileSystemQuery = typeof(CacheFileSystemQuery);

// 初始化包裹
var package = YooAssets.GetPackage("DefaultPackage");
var operation = package.InitializeAsync(initParameters);
yield return operation;

if (operation.Status != EOperationStatus.Succeed)
{
    Debug.LogError($"初始化失败：{operation.Error}");
}
else
{
    // 初始化成功后，请求最新版本
    var versionOp = package.RequestPackageVersionAsync();
    yield return versionOp;
    Debug.Log($"最新版本：{versionOp.PackageVersion}");
}
```

## InitializationOperation：顺序初始化状态机

`InitializationOperation` 是负责初始化流程的核心类，它采用五步状态机设计，确保文件系统的初始化过程严谨可靠。

### 五步状态机流程

```csharp:Runtime/ResourcePackage/Operation/InitializationOperation.cs [8:16]
private enum ESteps
{
    None,
    Prepare,
    ClearOldFileSystem,
    InitFileSystem,
    CheckInitResult,
    Done,
}
```

这五个状态的流转逻辑如下：

1. **Prepare**：验证参数列表的有效性，确保没有空对象，并克隆参数列表。
2. **ClearOldFileSystem**：清理旧的文件系统实例，防止初始化失败后残留资源。
3. **InitFileSystem**：依次创建文件系统实例并初始化，每次处理一个文件系统。
4. **CheckInitResult**：检查当前文件系统的初始化结果，如果成功则继续下一个，否则失败。
5. **Done**：所有文件系统初始化完成，标记操作结束。

![ResourcePackage 初始化流程](/img/yooasset/ch05-init-flow.png)

*图 5-1：ResourcePackage 初始化流程*

### 为什么串行不并行

你可能注意到，文件系统的初始化是串行执行的，而非并行。这是有意为之的设计决策：

- **依赖关系**：某些文件系统可能依赖其他文件系统的初始化结果（例如缓存文件系统需要验证内置文件系统的完整性）。
- **错误隔离**：串行执行可以精确定位哪个文件系统初始化失败，并行执行会让错误排查变得复杂。
- **资源竞争**：多个文件系统同时初始化可能竞争磁盘 I/O 和网络资源，反而降低性能。

### 清理旧文件系统的重要性

在 `ClearOldFileSystem` 步骤中，有一个关键的清理逻辑：

```csharp:Runtime/ResourcePackage/Operation/InitializationOperation.cs [63:73]
if (_steps == ESteps.ClearOldFileSystem)
{
    // 注意：初始化失败后可能会残存一些旧的文件系统！
    foreach (var fileSystem in _impl.FileSystems)
    {
        fileSystem.OnDestroy();
    }

    _impl.FileSystems.Clear();
    _steps = ESteps.InitFileSystem;
}
```

这段代码处理了一个重要的边界情况：当初始化失败后，用户可能调用重试逻辑，此时需要先清理掉部分初始化的文件系统，避免资源泄漏和状态不一致。

## PlayModeImpl：双接口适配器

`PlayModeImpl` 是运行模式的具体实现类，它同时实现了 `IPlayMode` 和 `IBundleQuery` 两个接口，分别承担不同的职责。

### IPlayMode 接口：生命周期管理

`IPlayMode` 接口负责资源包的生命周期管理，包括清单更新、资源下载、缓存清理等高级操作：

```csharp:Runtime/ResourcePackage/Interface/IPlayMode.cs [4:48]
internal interface IPlayMode
{
    /// <summary>
    /// 当前激活的清单
    /// </summary>
    PackageManifest ActiveManifest { set; get; }

    /// <summary>
    /// 销毁文件系统
    /// </summary>
    void DestroyFileSystem();

    /// <summary>
    /// 向网络端请求最新的资源版本
    /// </summary>
    RequestPackageVersionOperation RequestPackageVersionAsync(bool appendTimeTicks, int timeout);

    /// <summary>
    /// 向网络端请求并更新清单
    /// </summary>
    UpdatePackageManifestOperation UpdatePackageManifestAsync(string packageVersion, int timeout);

    /// <summary>
    /// 预下载指定版本的包裹内容
    /// </summary>
    PreDownloadContentOperation PreDownloadContentAsync(string packageVersion, int timeout);

    /// <summary>
    /// 清理缓存文件
    /// </summary>
    ClearCacheFilesOperation ClearCacheFilesAsync(ClearCacheFilesOptions options);

    // 下载相关
    ResourceDownloaderOperation CreateResourceDownloaderByAll(int downloadingMaxNumber, int failedTryAgain);
    ResourceDownloaderOperation CreateResourceDownloaderByTags(string[] tags, int downloadingMaxNumber, int failedTryAgain);
    ResourceDownloaderOperation CreateResourceDownloaderByPaths(AssetInfo[] assetInfos, bool recursiveDownload, int downloadingMaxNumber, int failedTryAgain);

    // 解压相关
    ResourceUnpackerOperation CreateResourceUnpackerByAll(int upackingMaxNumber, int failedTryAgain);
    ResourceUnpackerOperation CreateResourceUnpackerByTags(string[] tags, int upackingMaxNumber, int failedTryAgain);

    // 导入相关
    ResourceImporterOperation CreateResourceImporterByFilePaths(string[] filePaths, int importingMaxNumber, int failedTryAgain);
    ResourceImporterOperation CreateResourceImporterByFileInfos(ImportFileInfo[] fileInfos, int importingMaxNumber, int failedTryAgain);
}
```

### IBundleQuery 接口：资源查询

`IBundleQuery` 接口负责资源包的查询操作，包括获取主资源包信息和依赖资源包信息：

```csharp:Runtime/ResourcePackage/Interface/IBundleQuery.cs [6:22]
internal interface IBundleQuery
{
    /// <summary>
    /// 获取主资源包信息
    /// </summary>
    BundleInfo GetMainBundleInfo(AssetInfo assetInfo);

    /// <summary>
    /// 获取依赖的资源包信息集合
    /// </summary>
    List<BundleInfo> GetDependBundleInfos(AssetInfo assetPath);

    /// <summary>
    /// 获取主资源包名称
    /// </summary>
    string GetMainBundleName(int bundleID);
}
```

### FileSystems 列表与主文件系统约定

`PlayModeImpl` 维护了一个 `FileSystems` 列表，存储所有初始化的文件系统实例。有一个重要的约定：**列表中最后一个元素作为主文件系统**。

```csharp:Runtime/ResourcePackage/PlayMode/PlayModeImpl.cs [212:222]
/// <summary>
/// 获取主文件系统
///  说明：文件系统列表里，最后一个属于主文件系统
/// </summary>
public IFileSystem GetMainFileSystem()
{
    int count = FileSystems.Count;
    if (count == 0)
        return null;
    return FileSystems[count - 1];
}
```

这个约定确保了联机模式下，版本请求和清单更新操作总是从最新的文件系统（通常是缓存文件系统或远程文件系统）发起。

### GetBelongFileSystem 路由逻辑

当一个资源包需要加载时，系统需要确定它归属于哪个文件系统。`GetBelongFileSystem` 方法通过遍历文件系统列表，找到第一个声明"归属"该资源包的文件系统：

```csharp:Runtime/ResourcePackage/PlayMode/PlayModeImpl.cs [224:240]
/// <summary>
/// 获取资源包所属文件系统
/// </summary>
public IFileSystem GetBelongFileSystem(PackageBundle packageBundle)
{
    for (int i = 0; i < FileSystems.Count; i++)
    {
        IFileSystem fileSystem = FileSystems[i];
        if (fileSystem.Belong(packageBundle))
        {
            return fileSystem;
        }
    }

    YooLogger.Error($"Can not found belong file system : {packageBundle.BundleName}");
    return null;
}
```

这个路由逻辑实现了第 3 章讲的"文件系统优先级"：缓存文件系统优先于内置文件系统，因为缓存文件系统在列表中排在前面。

## 联机更新双步流程

联机模式的核心能力是热更新，YooAsset 将热更新拆分为两个独立的步骤：版本请求和清单更新。这种拆分设计提供了更大的灵活性和可控性。

### 第一步：请求版本

`RequestPackageVersionOperation` 负责向服务器请求最新的资源包版本号：

```csharp:Runtime/ResourcePackage/Operation/RequestPackageVersionOperation.cs [41:67]
if (_steps == ESteps.RequestPackageVersion)
{
    if (_requestPackageVersionOp == null)
    {
        var mainFileSystem = _impl.GetMainFileSystem();
        _requestPackageVersionOp = mainFileSystem.RequestPackageVersionAsync(_appendTimeTicks, _timeout);
        _requestPackageVersionOp.StartOperation();
        AddChildOperation(_requestPackageVersionOp);
    }

    _requestPackageVersionOp.UpdateOperation();
    if (_requestPackageVersionOp.IsDone == false)
        return;

    if (_requestPackageVersionOp.Status == EOperationStatus.Succeed)
    {
        _steps = ESteps.Done;
        PackageVersion = _requestPackageVersionOp.PackageVersion;
        Status = EOperationStatus.Succeed;
    }
    else
    {
        _steps = ESteps.Done;
        Status = EOperationStatus.Failed;
        Error = _requestPackageVersionOp.Error;
    }
}
```

这个操作非常轻量，只返回一个版本字符串，不涉及大量数据传输。

### 第二步：更新清单

`UpdatePackageManifestOperation` 负责下载并解析指定版本的清单文件：

```csharp:Runtime/ResourcePackage/Operation/UpdatePackageManifestOperation.cs [64:90]
if (_steps == ESteps.LoadPackageManifest)
{
    if (_loadPackageManifestOp == null)
    {
        var mainFileSystem = _impl.GetMainFileSystem();
        _loadPackageManifestOp = mainFileSystem.LoadPackageManifestAsync(_packageVersion, _timeout);
        _loadPackageManifestOp.StartOperation();
        AddChildOperation(_loadPackageManifestOp);
    }

    _loadPackageManifestOp.UpdateOperation();
    if (_loadPackageManifestOp.IsDone == false)
        return;

    if (_loadPackageManifestOp.Status == EOperationStatus.Succeed)
    {
        _steps = ESteps.Done;
        _impl.ActiveManifest = _loadPackageManifestOp.Manifest;
        Status = EOperationStatus.Succeed;
    }
    else
    {
        _steps = ESteps.Done;
        Status = EOperationStatus.Failed;
        Error = _loadPackageManifestOp.Error;
    }
}
```

### 版本缓存短路优化

在更新清单之前，系统会检查当前激活的清单版本是否已经是目标版本，如果是则直接返回成功，避免重复下载：

```csharp:Runtime/ResourcePackage/Operation/UpdatePackageManifestOperation.cs [50:62]
if (_steps == ESteps.CheckActiveManifest)
{
    // 检测当前激活的清单对象
    if (_impl.ActiveManifest != null && _impl.ActiveManifest.PackageVersion == _packageVersion)
    {
        _steps = ESteps.Done;
        Status = EOperationStatus.Succeed;
    }
    else
    {
        _steps = ESteps.LoadPackageManifest;
    }
}
```

这个优化在应用重启后特别有用，因为本地已经缓存了最新版本的清单。

### ActiveManifest 设置时机与 PackageValid 守卫

`ActiveManifest` 只在清单更新成功后设置，它是 `ResourcePackage.PackageValid` 属性的判断依据：

```csharp:Runtime/ResourcePackage/ResourcePackage.cs [34:45]
/// <summary>
/// 包裹是否有效
/// </summary>
public bool PackageValid
{
    get
    {
        if (_playModeImpl == null)
            return false;
        return _playModeImpl.ActiveManifest != null;
    }
}
```

这意味着在联机模式下，即使初始化成功，如果还没有更新清单，`PackageValid` 也会返回 `false`。这是一个重要的守卫机制，防止在清单未就绪时加载资源。

## 销毁与重试

### DestroyOperation 正确卸载顺序

当销毁资源包时，需要按照正确的顺序卸载组件，避免资源泄漏和访问冲突：

```csharp:Runtime/ResourcePackage/ResourcePackage.cs [56:78]
internal void DestroyPackage()
{
    if (_isInitialize)
    {
        _isInitialize = false;
        _initializeError = string.Empty;
        _initializeStatus = EOperationStatus.None;

        // 销毁资源管理器
        if (_resourceManager != null)
        {
            _resourceManager.Destroy();
            _resourceManager = null;
        }

        // 销毁文件系统
        if (_playModeImpl != null)
            _playModeImpl.DestroyFileSystem();

        _bundleQuery = null;
        _playModeImpl = null;
    }
}
```

卸载顺序是：先销毁资源管理器（释放所有加载的资源），再销毁文件系统（关闭文件句柄、释放缓存）。

### ResetInitializeAfterFailed 重试语义

当初始化失败后，系统提供了重试机制。在重新初始化前，会调用 `ResetInitializeAfterFailed` 清理失败状态：

```csharp:Runtime/ResourcePackage/ResourcePackage.cs [143:151]
private void ResetInitializeAfterFailed()
{
    if (_isInitialize && _initializeStatus == EOperationStatus.Failed)
    {
        _isInitialize = false;
        _initializeStatus = EOperationStatus.None;
        _initializeError = string.Empty;
    }
}
```

这个方法只在初始化失败且未销毁时才执行，确保用户可以在网络恢复后重新调用 `InitializeAsync`。

## 设计思考

### 为什么拆成"版本请求+清单更新"两步

YooAsset 的联机更新流程被拆分为两个独立的操作，而不是一个"更新到最新版本"的操作。这个设计有以下几个考虑：

1. **灵活性**：用户可以在请求版本后，决定是否更新。例如，可以显示更新日志，让用户选择是否更新。
2. **增量更新**：用户可以指定更新到某个特定版本，而不一定是最新版本。这对于测试和回滚非常有用。
3. **网络优化**：版本请求非常轻量，可以频繁调用（如每次启动时检查），而清单下载可能较大，只在需要时才下载。
4. **错误处理**：将两个操作分离，可以更精确地处理错误。例如，版本请求成功但清单下载失败时，可以提示用户网络问题，而不是版本问题。

### 主文件系统约定的深层含义

"列表最后一个元素作为主文件系统"这个约定看似简单，但蕴含了深层的设计智慧：

1. **向后兼容**：新版本的资源总是放在后面的文件系统（如缓存文件系统），确保优先使用新版本。
2. **写入目标**：下载的新资源需要写入到可写的文件系统，通常是最后一个（如缓存文件系统）。
3. **版本来源**：版本请求和清单更新总是从最新的文件系统发起，确保获取的是最新版本信息。

## 小结

本章深入分析了 YooAsset 的初始化与运行模式，主要内容包括：

1. **五种运行模式**：EditorSimulateMode、OfflinePlayMode、HostPlayMode、WebPlayMode、CustomPlayMode，每种模式对应不同的文件系统组合。
2. **InitializationOperation 状态机**：五步状态机（Prepare→ClearOldFileSystem→InitFileSystem→CheckInitResult→Done）确保初始化流程的可靠性。
3. **PlayModeImpl 双接口**：IPlayMode 负责生命周期管理，IBundleQuery 负责资源查询，职责分离清晰。
4. **联机更新双步流程**：版本请求和清单更新分离，提供更大的灵活性和可控性。
5. **版本缓存优化**：通过 ActiveManifest 版本检查，避免重复下载相同版本的清单。

初始化是资源系统的起点，理解了初始化流程，就理解了资源系统的整体架构。下一章我们将深入探讨资源加载的完整链路，看看一个资源请求是如何从应用层传递到文件系统，最终返回 Unity 对象的。
