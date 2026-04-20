---
title: "【教材】第3章：文件系统抽象"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 3 章：文件系统抽象

## 概述

在移动平台、WebGL 和编辑器环境下，资源文件的存储来源各不相同：移动端可能从 APK 内置、沙盒缓存或远程服务器加载；WebGL 依赖 UnityWebRequest；编辑器则直接访问 StreamingAssets 或 AssetDatabase。如果让业务代码直接处理这些差异，会产生大量平台分支和重复逻辑。YooAsset 通过文件系统抽象层（IFileSystem 接口）统一了不同存储来源的访问方式，让上层操作流程无需关心底层实现细节。

文件系统抽象层本质上是策略模式的典型应用：每个 IFileSystem 实现类封装了一种特定的文件存储策略（内置、缓存、远程等），运行时根据配置动态创建合适的实例。这种设计使得 YooAsset 能够在同一个框架内支持离线包、热更新、CDN 分发、编辑器模拟等多种部署场景，同时保持代码的可测试性和可扩展性。

## IFileSystem 接口解析

IFileSystem 是文件系统抽象的核心契约，定义了 28 个成员方法，可以归纳为三大职责族：生命周期管理、查询谓词和数据读取。理解这些方法的语义差异，是掌握文件系统协作机制的关键。

### 生命周期方法族

生命周期方法负责文件系统的初始化和销毁：

```csharp:Runtime/FileSystem/Interface/IFileSystem.cs [22:40]
/// <summary>
/// 初始化文件系统
/// </summary>
FSInitializeFileSystemOperation InitializeFileSystemAsync();

/// <summary>
/// 加载包裹清单
/// </summary>
FSLoadPackageManifestOperation LoadPackageManifestAsync(string packageVersion, int timeout);

/// <summary>
/// 查询包裹版本
/// </summary>
FSRequestPackageVersionOperation RequestPackageVersionAsync(bool appendTimeTicks, int timeout);

/// <summary>
/// 清理缓存文件
/// </summary>
FSClearCacheFilesOperation ClearCacheFilesAsync(PackageManifest manifest, ClearCacheFilesOptions options);

/// <summary>
/// 下载Bundle文件
/// </summary>
FSDownloadFileOperation DownloadFileAsync(PackageBundle bundle, DownloadFileOptions options);

/// <summary>
/// 加载Bundle文件
/// </summary>
FSLoadBundleOperation LoadBundleFile(PackageBundle bundle);
```

这些方法都返回异步操作对象，遵循 YooAsset 的异步操作模式。`InitializeFileSystemAsync` 执行文件系统级别的初始化，比如缓存文件系统会扫描沙盒目录建立文件索引；`LoadPackageManifestAsync` 负责从存储介质加载清单文件；`RequestPackageVersionAsync` 用于远程文件系统查询最新版本号；`ClearCacheFilesAsync` 提供缓存清理能力；`DownloadFileAsync` 处理文件下载或导入（从内置文件拷贝到缓存）；`LoadBundleFile` 则是实际加载 AssetBundle 或原始文件的核心入口。

### 查询谓词方法族

查询谓词用于判断文件是否归属于当前文件系统、是否存在以及需要何种操作：

```csharp:Runtime/FileSystem/Interface/IFileSystem.cs [53:92]
/// <summary>
/// 查询文件归属
/// </summary>
bool Belong(PackageBundle bundle);

/// <summary>
/// 查询文件是否存在
/// </summary>
bool Exists(PackageBundle bundle);

/// <summary>
/// 是否需要下载
/// </summary>
bool NeedDownload(PackageBundle bundle);

/// <summary>
/// 是否需要解压
/// </summary>
bool NeedUnpack(PackageBundle bundle);

/// <summary>
/// 是否需要导入
/// </summary>
bool NeedImport(PackageBundle bundle);
```

这五个谓词构成了多文件系统协作的决策基础。`Belong` 判断 Bundle 是否属于当前文件系统的管理范围——比如缓存文件系统的 Belong 固定返回 true，作为保底加载机制；`Exists` 检查文件是否物理存在于当前存储位置；`NeedDownload`、`NeedUnpack`、`NeedImport` 则分别指示需要下载、从 APK 解压、或从内置文件导入。这些返回值的组合差异，正是不同文件系统特性的体现。

### 数据读取方法族

数据读取方法提供实际的文件访问能力：

```csharp:Runtime/FileSystem/Interface/IFileSystem.cs [95:108]
/// <summary>
/// 获取Bundle文件路径
/// </summary>
string GetBundleFilePath(PackageBundle bundle);

/// <summary>
/// 读取Bundle文件的二进制数据
/// </summary>
byte[] ReadBundleFileData(PackageBundle bundle);

/// <summary>
/// 读取Bundle文件的文本数据
/// </summary>
string ReadBundleFileText(PackageBundle bundle);
```

`GetBundleFilePath` 返回文件的物理路径，不同文件系统返回不同格式的路径：内置文件系统返回 StreamingAssets 路径，缓存文件系统返回沙盒路径，Web 文件系统则可能抛出异常。`ReadBundleFileData` 和 `ReadBundleFileText` 用于直接读取文件内容，主要服务于清单文件等文本资源的加载，以及加密文件的解密读取。

## FileSystemParameters：工厂与注入

FileSystemParameters 是文件系统的工厂类，负责创建 IFileSystem 实例并注入配置参数。它的核心机制是反射实例化：

```csharp:Runtime/FileSystem/FileSystemParameters.cs [40:67]
/// <summary>
/// 创建文件系统
/// </summary>
internal IFileSystem CreateFileSystem(string packageName)
{
    YooLogger.Log($"The package {packageName} create file system : {FileSystemClass}");

    Type classType = Type.GetType(FileSystemClass);
    if (classType == null)
    {
        YooLogger.Error($"Can not found file system class type {FileSystemClass}");
        return null;
    }

    var instance = (IFileSystem)System.Activator.CreateInstance(classType, true);
    if (instance == null)
    {
        YooLogger.Error($"Failed to create file system instance {FileSystemClass}");
        return null;
    }

    foreach (var param in CreateParameters)
    {
        instance.SetParameter(param.Key, param.Value);
    }
    instance.OnCreate(packageName, PackageRoot);
    return instance;
}
```

这里使用 `Type.GetType` 和 `Activator.CreateInstance` 而非直接 `new`，有几个原因：首先，用户可能实现自定义文件系统类，框架无法预知类型；其次，通过字符串指定类型，可以实现配置文件驱动的文件系统切换；最后，这种模式允许在运行时动态选择文件系统实现。

FileSystemParameters 提供了六个静态工厂方法，用于创建常见的文件系统配置：

```csharp:Runtime/FileSystem/FileSystemParameters.cs [69:121]
public static FileSystemParameters CreateDefaultEditorFileSystemParameters(string packageRoot)
public static FileSystemParameters CreateDefaultBuildinFileSystemParameters(IDecryptionServices decryptionServices = null, string packageRoot = null)
public static FileSystemParameters CreateDefaultCacheFileSystemParameters(IRemoteServices remoteServices, IDecryptionServices decryptionServices = null, string packageRoot = null)
public static FileSystemParameters CreateDefaultWebServerFileSystemParameters(IWebDecryptionServices decryptionServices = null, bool disableUnityWebCache = false)
public static FileSystemParameters CreateDefaultWebRemoteFileSystemParameters(IRemoteServices remoteServices, IWebDecryptionServices decryptionServices = null, bool disableUnityWebCache = false)
```

每个工厂方法都封装了对应文件系统必需的参数：Editor 文件系统只需 packageRoot；内置文件系统需要可选的解密服务；缓存文件系统需要远程服务接口（用于生成下载 URL）；Web 文件系统需要 Web 专用的解密服务和缓存控制参数。这些参数通过 `SetParameter` 方法注入到文件系统实例中，FileSystemParametersDefine 类定义了所有参数名称的字符串常量，避免硬编码魔法字符串。

## 六种实现横向对比

YooAsset 内置了六种文件系统实现，每种实现针对特定的部署场景和平台特性。理解它们在查询谓词上的返回值差异，是掌握多文件系统协作机制的关键。

### 内置文件系统

DefaultBuildinFileSystem 负责访问打包时嵌入应用分发的资源文件，在移动端对应 StreamingAssets 目录，在编辑器对应实际文件路径：

```csharp:Runtime/FileSystem/DefaultBuildinFileSystem/DefaultBuildinFileSystem.cs [235:265]
public virtual bool Belong(PackageBundle bundle)
{
    if (DisableCatalogFile)
        return true;
    return _wrappers.ContainsKey(bundle.BundleGUID);
}
public virtual bool Exists(PackageBundle bundle)
{
    if (DisableCatalogFile)
        return true;
    return _wrappers.ContainsKey(bundle.BundleGUID);
}
public virtual bool NeedDownload(PackageBundle bundle)
{
    return false;
}
public virtual bool NeedUnpack(PackageBundle bundle)
{
    if (IsUnpackBundleFile(bundle))
    {
        return _unpackFileSystem.Exists(bundle) == false;
    }
    else
    {
        return false;
    }
}
public virtual bool NeedImport(PackageBundle bundle)
{
    return false;
}
```

内置文件系统的 Belong 和 Exists 通过检查 `_wrappers` 字典判断文件是否在内置目录中。`_wrappers` 在初始化时从 Catalog 文件加载，Catalog 是一个内置文件的索引文件，记录了所有内置资源的 GUID 到文件名的映射。`DisableCatalogFile` 参数可以禁用目录查询，强制返回 true，这在某些特殊场景下有用。

`NeedDownload` 固定返回 false，因为内置文件不需要网络下载。`NeedUnpack` 则有一个特殊分支：Android 平台的加密文件和 Raw 文件需要从 APK 解压到沙盒才能访问，这是因为 Unity 的 AssetBundle.LoadFromFile 无法直接读取 APK 压缩包内的加密文件。`IsUnpackBundleFile` 方法封装了这个平台相关的判断逻辑：

```csharp:Runtime/FileSystem/DefaultBuildinFileSystem/DefaultBuildinFileSystem.cs [349:368]
protected virtual bool IsUnpackBundleFile(PackageBundle bundle)
{
    if (Belong(bundle) == false)
        return false;

#if UNITY_ANDROID || UNITY_OPENHARMONY
    if (bundle.Encrypted)
        return true;

    if (bundle.BundleType == (int)EBuildBundleType.RawBundle)
        return true;

    return false;
#else
    return false;
#endif
}
```

这个方法体现了平台差异的集中处理：在 Android 和 OpenHarmony 平台上，加密文件和 Raw 文件必须解压；其他平台则直接从内置路径加载。

### 缓存文件系统

DefaultCacheFileSystem 管理沙盒目录中的缓存文件，是热更新的核心存储层：

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/DefaultCacheFileSystem.cs [312:338]
public virtual bool Belong(PackageBundle bundle)
{
    // 注意：缓存文件系统保底加载！
    return true;
}
public virtual bool Exists(PackageBundle bundle)
{
    return _records.ContainsKey(bundle.BundleGUID);
}
public virtual bool NeedDownload(PackageBundle bundle)
{
    if (Belong(bundle) == false)
        return false;

    return Exists(bundle) == false;
}
public virtual bool NeedUnpack(PackageBundle bundle)
{
    return false;
}
public virtual bool NeedImport(PackageBundle bundle)
{
    if (Belong(bundle) == false)
        return false;

    return Exists(bundle) == false;
}
```

缓存文件系统的 Belong 固定返回 true，这是一个关键设计决策——缓存文件系统作为保底加载机制，任何文件都可以尝试从缓存加载。这种"兜底"语义确保了即使内置文件和远程文件系统都无法提供文件，缓存中已有的版本仍可使用。

Exists 通过 `_records` 字典检查文件是否已缓存，`_records` 在初始化时扫描沙盒目录建立。`NeedDownload` 和 `NeedImport` 的逻辑相同：文件属于当前系统且不存在时，需要下载或导入。这里的"导入"指的是从内置文件拷贝到缓存，而不是从网络下载。

### Web 远程文件系统

DefaultWebRemoteFileSystem 是 WebGL 平台的专用实现，它的特点是所有谓词都返回固定值：

```csharp:Runtime/FileSystem/DefaultWebRemoteFileSystem/DefaultWebRemoteFileSystem.cs [136:155]
public virtual bool Belong(PackageBundle bundle)
{
    return true;
}
public virtual bool Exists(PackageBundle bundle)
{
    return true;
}
public virtual bool NeedDownload(PackageBundle bundle)
{
    return false;
}
public virtual bool NeedUnpack(PackageBundle bundle)
{
    return false;
}
public virtual bool NeedImport(PackageBundle bundle)
{
    return false;
}
```

Web 远程文件系统不需要判断文件归属和存在性，因为 UnityWebRequest 可以直接请求任意 URL，成功与否由网络请求结果决定。所有谓词返回 true 或 false 的固定值，简化了 WebGL 平台的文件系统协作逻辑——它总是直接尝试加载，由加载操作本身处理失败情况。

![文件系统选型矩阵](/img/yooasset/ch03-filesystem-matrix.png)
*图 3-1：六种文件系统选型矩阵*

这个矩阵对比了六种文件系统在关键谓词上的返回值差异，以及它们的典型使用场景。可以看到，不同文件系统的设计哲学差异明显：内置文件系统强调"精确归属"，缓存文件系统强调"保底兜底"，Web 文件系统强调"简单直接"。

## BundleResult 类型层次

BundleResult 是文件系统加载 Bundle 后的返回值抽象，它封装了不同类型 Bundle 的加载结果和操作接口。理解这个类型层次，是掌握资源加载流程的关键。

### 抽象基类 BundleResult

BundleResult 定义了所有 Bundle 结果的通用接口：

```csharp:Runtime/FileSystem/BundleResult/BundleResult.cs [5:47]
internal abstract class BundleResult
{
    /// <summary>
    /// 卸载资源包文件
    /// </summary>
    public abstract void UnloadBundleFile();

    /// <summary>
    /// 获取资源包文件的路径
    /// </summary>
    public abstract string GetBundleFilePath();

    /// <summary>
    /// 读取资源包文件的二进制数据
    /// </summary>
    public abstract byte[] ReadBundleFileData();

    /// <summary>
    /// 读取资源包文件的文本数据
    /// </summary>
    public abstract string ReadBundleFileText();

    /// <summary>
    /// 加载资源包内的资源对象
    /// </summary>
    public abstract FSLoadAssetOperation LoadAssetAsync(AssetInfo assetInfo);

    /// <summary>
    /// 加载资源包内的所有资源对象
    /// </summary>
    public abstract FSLoadAllAssetsOperation LoadAllAssetsAsync(AssetInfo assetInfo);

    /// <summary>
    /// 加载资源包内的资源对象及所有子资源对象
    /// </summary>
    public abstract FSLoadSubAssetsOperation LoadSubAssetsAsync(AssetInfo assetInfo);

    /// <summary>
    /// 加载资源包内的场景对象
    /// </summary>
    public abstract FSLoadSceneOperation LoadSceneOperation(AssetInfo assetInfo, LoadSceneParameters loadParams, bool suspendLoad);
}
```

这些方法分为三类：卸载和查询（`UnloadBundleFile`、`GetBundleFilePath`、`ReadBundleFileData`、`ReadBundleFileText`）、资源加载（`LoadAssetAsync`、`LoadAllAssetsAsync`、`LoadSubAssetsAsync`）和场景加载（`LoadSceneOperation`）。所有方法都是抽象的，由子类根据 Bundle 类型提供具体实现。

### AssetBundleResult 实现

AssetBundleResult 是最常用的 BundleResult 子类，它封装了 AssetBundle 对象和可选的托管流：

```csharp:Runtime/FileSystem/BundleResult/AssetBundleResult/AssetBundleResult.cs [7:20]
private readonly IFileSystem _fileSystem;
private readonly PackageBundle _packageBundle;
private readonly AssetBundle _assetBundle;
private readonly Stream _managedStream;

public AssetBundleResult(IFileSystem fileSystem, PackageBundle packageBundle, AssetBundle assetBundle, Stream managedStream)
{
    _fileSystem = fileSystem;
    _packageBundle = packageBundle;
    _assetBundle = assetBundle;
    _managedStream = managedStream;
}
```

`_managedStream` 是一个关键字段：当从文件路径加载 AssetBundle 时，Unity 内部会打开文件流；当从内存加载（比如加密文件解密后）时，需要手动创建 MemoryStream。这个流的生命周期由 AssetBundleResult 管理，在 `UnloadBundleFile` 中释放：

```csharp:Runtime/FileSystem/BundleResult/AssetBundleResult/AssetBundleResult.cs [22:34]
public override void UnloadBundleFile()
{
    if (_assetBundle != null)
    {
        _assetBundle.Unload(true);
    }

    if (_managedStream != null)
    {
        _managedStream.Close();
        _managedStream.Dispose();
    }
}
```

注意这里调用 `AssetBundle.Unload(true)`，会卸载所有从该 Bundle 加载的资源，包括已在内存中的对象。这是 YooAsset 的默认卸载策略，确保资源释放的彻底性。`_managedStream` 的释放顺序也很重要：先关闭流，再释放资源，避免访问已释放的流。

资源加载方法则委托给对应的操作类：

```csharp:Runtime/FileSystem/BundleResult/AssetBundleResult/AssetBundleResult.cs [48:67]
public override FSLoadAssetOperation LoadAssetAsync(AssetInfo assetInfo)
{
    var operation = new AssetBundleLoadAssetOperation(_packageBundle, _assetBundle, assetInfo);
    return operation;
}
public override FSLoadAllAssetsOperation LoadAllAssetsAsync(AssetInfo assetInfo)
{
    var operation = new AssetBundleLoadAllAssetsOperation(_packageBundle, _assetBundle, assetInfo);
    return operation;
}
public override FSLoadSubAssetsOperation LoadSubAssetsAsync(AssetInfo assetInfo)
{
    var operation = new AssetBundleLoadSubAssetsOperation(_packageBundle, _assetBundle, assetInfo);
    return operation;
}
public override FSLoadSceneOperation LoadSceneOperation(AssetInfo assetInfo, LoadSceneParameters loadParams, bool suspendLoad)
{
    var operation = new AssetBundleLoadSceneOperation(assetInfo, loadParams, suspendLoad);
    return operation;
}
```

这些操作类封装了 Unity 原生的 AssetBundle.LoadAssetAsync 等 API，提供统一的异步操作接口。不同子类的关键差异在于卸载行为：RawBundleResult 和 VirtualBundleResult 不需要卸载 AssetBundle，因为它们根本不持有 AssetBundle 对象。

![BundleResult 类型层次](/img/yooasset/ch03-bundle-result.png)
*图 3-2：BundleResult 类型层次结构*

这个层次结构展示了三种 BundleResult 的继承关系和职责分工。AssetBundleResult 处理真正的 AssetBundle，RawBundleResult 处理原始文件（如 JSON、二进制配置），VirtualBundleResult 则用于编辑器模式的虚拟加载（直接从 AssetDatabase 加载，不经过 AssetBundle）。

## 设计思考

文件系统抽象层的设计体现了两个经典设计模式的组合：策略模式和工厂方法。策略模式体现在 IFileSystem 接口定义了文件访问的策略契约，不同实现类封装了不同的存储策略；工厂方法体现在 FileSystemParameters 通过反射创建具体实现类，并注入配置参数。

反射实例化而非直接 new 的选择，是一个值得权衡的设计点。反射带来了两个优势：一是配置文件驱动，可以通过修改配置切换文件系统而无需重新编译；二是用户扩展性，可以实现自定义文件系统类而无需修改框架代码。但反射也有代价：类型错误只能在运行时发现，性能开销略高于直接实例化。在 YooAsset 的场景中，这些代价是可接受的——文件系统创建发生在初始化阶段，执行频率低；类型错误可以通过配置校验提前发现。

另一个设计思考是字符串反射注册 vs 接口注入。YooAsset 选择了前者，通过 `FileSystemClass` 字符串指定类型名。这种方式的优势是配置简单，JSON 或 YAML 文件可以直接存储类型名；劣势是缺乏编译时检查，容易拼写错误。接口注入方式则要求用户代码显式注册文件系统类型，类型安全但增加配置复杂度。在 YooAsset 的使用场景中，字符串反射注册提供了更好的易用性。

## 小结

文件系统抽象层是 YooAsset 架构中的关键基础设施，它通过 IFileSystem 接口统一了不同存储来源的访问方式，让上层流程无需关心平台差异和存储细节。六种内置实现覆盖了编辑器、移动端、WebGL 等主流场景，通过查询谓词的差异返回值实现了多文件系统的协作机制。BundleResult 类型层次则封装了不同 Bundle 类型的加载结果，提供了统一的卸载和资源访问接口。

理解文件系统抽象层的设计，有助于掌握 YooAsset 的可扩展性机制——如果需要支持新的存储介质（比如自定义加密方案、云存储集成），只需实现 IFileSystem 接口并通过 FileSystemParameters 注册即可。这种设计让 YooAsset 既能开箱即用于常见场景，又能灵活适应特殊需求，体现了框架设计的开放封闭原则。
