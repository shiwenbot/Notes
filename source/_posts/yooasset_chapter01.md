---
title: "【教材】第1章：整体架构概览"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 1 章：整体架构概览

## 1.1 概述

打开 YooAsset 的源码目录，你会看到 443 个 CS 文件。面对如此庞大的代码库，从哪里入手？这一章帮你在脑中建立一张清晰的地图。我们不讲细节，只讲骨架——理解了骨架，后续章节的深入分析就会顺理成章。

YooAsset 的核心设计思想是**分层与隔离**。通过三层架构将职责清晰划分：对外暴露简洁 API，对内封装复杂的资源加载逻辑。每个层级只关注自己的职责，边界清晰，耦合度低。

这种设计带来的好处是明显的：你可以轻松地替换底层实现，或者扩展新的功能模块，而不影响上层代码。当你理解了这个架构，就掌握了理解整个 YooAsset 的钥匙。

## 1.2 三层架构划分

YooAsset 的架构分为三层，从外到内依次是：

**1. Facade 层（外观层）**
- 提供 `YooAssets` 静态类作为统一入口
- 封装初始化、销毁、包管理等全局 API
- 隐藏内部实现细节，对外暴露简洁接口

```csharp:Runtime/YooAssets.cs [10:32]
public static partial class YooAssets
{
    private static bool _isInitialize = false;
    private static GameObject _driver = null;
    private static readonly List<ResourcePackage> _packages = new List<ResourcePackage>();

    /// <summary>
    /// 是否已经初始化
    /// </summary>
    public static bool Initialized
    {
        get { return _isInitialize; }
    }
```

Facade 层的 `Initialize` 方法负责创建驱动器并启动整个系统：

```csharp:Runtime/YooAssets.cs [38:64]
public static void Initialize(ILogger logger = null)
{
    if (_isInitialize)
    {
        UnityEngine.Debug.LogWarning($"{nameof(YooAssets)} is initialized !");
        return;
    }

    if (_isInitialize == false)
    {
        YooLogger.Logger = logger;

        // 创建驱动器
        _isInitialize = true;
        _driver = new UnityEngine.GameObject($"[{nameof(YooAssets)}]");
        _driver.AddComponent<YooAssetsDriver>();
        UnityEngine.Object.DontDestroyOnLoad(_driver);
        YooLogger.Log($"{nameof(YooAssets)} initialize !");

#if DEBUG
        // 添加远程调试脚本
        _driver.AddComponent<RemoteDebuggerInRuntime>();
#endif

        OperationSystem.Initialize();
    }
}
```

**2. 业务层（Business Layer）**
- `ResourcePackage`：资源包管理器，负责单个包的初始化和资源加载
- `ResourceManager`：资源管理器，处理具体的加载操作和缓存
- 实现 PlayMode 接口，支持不同的运行模式

**3. 基础设施层（Infrastructure Layer）**
- `OperationSystem`：异步操作系统，负责时间片调度
- `FileSystem`：文件系统抽象，支持多种存储方式
- `DownloadSystem`：下载系统，处理网络下载
- `Services`：各种辅助服务（日志、加密等）

![YooAsset 整体架构](../diagrams/ch01-architecture.png)
*图 1-1：YooAsset 三层架构概览（Facade → 业务层 → 基础设施层）*

## 1.3 多 Package 设计

YooAsset 没有采用单一全局管理器的设计，而是引入了 `ResourcePackage` 概念。每个 Package 是独立的资源包，有自己的生命周期和资源缓存。

```csharp:Runtime/YooAssets.cs [22:24]
private static bool _isInitialize = false;
private static GameObject _driver = null;
private static readonly List<ResourcePackage> _packages = new List<ResourcePackage>();
```

这种设计的优势在于**灵活性和隔离性**：

**场景 1：DLC 独立管理**
游戏本体和 DLC 可以是不同的 Package。玩家卸载 DLC 时，只需要销毁对应的 Package，不会影响主游戏的资源缓存。

**场景 2：热更包隔离**
热更新的资源可以放在独立的 Package 中。当热更失败需要回滚时，只需卸载热更包，重新加载旧版本即可。

**场景 3：多项目共享**
同一个游戏可以有多个 Package 分别管理不同类型的资源（UI、场景、角色），方便团队协作和独立更新。

获取 Package 的 API 设计也很简洁：

```csharp:Runtime/YooAssets.cs [145:149]
/// <summary>
/// 尝试获取资源包裹
/// </summary>
/// <param name="packageName">包裹名称</param>
public static ResourcePackage TryGetPackage(string packageName)
{
    CheckException(packageName);
    return GetPackageInternal(packageName);
}
```

## 1.4 五种运行模式速览

YooAsset 支持五种运行模式（`EPlayMode` 枚举），每种模式对应不同的文件系统组合和使用场景：

| 模式 | 文件系统组合 | 适用场景 |
|------|-------------|---------|
| **EditorSimulateMode** | 无文件系统 | 编辑器下快速开发，无需打包 |
| **OfflinePlayMode** | 只读文件系统 | 单机游戏，无热更需求 |
| **HostPlayMode** | 只读文件系统 | 网络游戏，资源在服务器 |
| **WebPlayMode** | 缓存文件系统 + 只读文件系统 | WebGL 平台，支持本地缓存 |
| **EmbeddedPlayMode** | 内嵌文件系统 + 缓存文件系统 | 移动平台，内置资源 + 热更 |

![五种 PlayMode](../diagrams/ch01-playmode.png)
*图 1-2：YooAsset 五种 PlayMode 对比*

每种模式的核心区别在于**文件系统的组合方式**：

- **EditorSimulateMode**：直接从 `Assets` 目录加载资源，适合快速迭代
- **OfflinePlayMode**：所有资源打包到本地，无需网络下载
- **HostPlayMode**：资源从 CDN 下载，适合纯网络游戏
- **WebPlayMode**：结合 IndexedDB 缓存和 CDN 下载，优化 WebGL 加载速度
- **EmbeddedPlayMode**：内置资源 + 热更资源，最常见的移动端方案

## 1.5 YooAssetsDriver：帧驱动的秘密

YooAsset 的核心是一个隐藏的 MonoBehaviour——`YooAssetsDriver`。它在 `Initialize` 时被创建，并且标记为 `DontDestroyOnLoad`，确保在场景切换时不被销毁。

```csharp:Runtime/YooAssetsDriver.cs [6:22]
internal class YooAssetsDriver : MonoBehaviour
{
    void Update()
    {
        DebugCheckDuplicateDriver();
        YooAssets.Update();
    }
}
```

这个驱动器的作用是**在每帧调用 YooAssets.Update()**，从而驱动整个异步操作系统。

为什么不用 `[RuntimeInitializeOnMethod]` 或纯静态方法？

**答案在于帧控制和生命周期管理**：
- MonoBehaviour 的 `Update` 方法与 Unity 的帧循环完美同步
- 可以方便地在 `OnApplicationQuit` 中清理资源（编辑器下）
- 通过 `DontDestroyOnLoad` 确保系统跨场景持久化

此外，`YooAssetsDriver` 还包含一个调试机制，检测场景中是否存在多个驱动器实例：

```csharp:Runtime/YooAssetsDriver.cs [32:42]
[Conditional("DEBUG")]
private void DebugCheckDuplicateDriver()
{
    if (LastestUpdateFrame > 0)
    {
        if (LastestUpdateFrame == Time.frameCount)
            YooLogger.Warning($"There are two {nameof(YooAssetsDriver)} in the scene. Please ensure there is always exactly one driver in the scene.");
    }

    LastestUpdateFrame = Time.frameCount;
}
```

## 1.6 OperationSystem 与时间片调度

`OperationSystem` 是 YooAsset 的异步调度引擎，负责管理所有异步操作的执行。它的核心设计是**时间片调度**——每帧只执行有限时间的任务，避免阻塞主线程。

```csharp:Runtime/OperationSystem/OperationSystem.cs [28:30]
/// <summary>
/// 异步操作的最小时间片段
/// </summary>
public static long MaxTimeSlice { set; get; } = long.MaxValue;
```

默认值是 `long.MaxValue`，意味着**不限制执行时间**。这是为什么呢？

因为 YooAsset 的异步操作大多是**IO 密集型**（下载、解压、加载 Bundle），而非 CPU 密集型。真正的阻塞操作已经通过 `async/await` 或协程交给了后台线程，主线程的调度开销相对较小。

当你需要优化帧率时，可以通过 `SetOperationSystemMaxTimeSlice` 限制每帧的执行时间：

```csharp:Runtime/YooAssets.cs [250:260]
/// <summary>
/// 设置异步系统参数，每帧执行消耗的最大时间切片（单位：毫秒）
/// </summary>
public static void SetOperationSystemMaxTimeSlice(long milliseconds)
{
    if (milliseconds < 10)
    {
        milliseconds = 10;
        YooLogger.Warning($"MaxTimeSlice minimum value is 10 milliseconds.");
    }
    OperationSystem.MaxTimeSlice = milliseconds;
}
```

`OperationSystem.Update()` 的执行流程如下：

1. **移除已完成的操作**：从列表中清理上一帧完成的任务
2. **添加新操作**：将新启动的操作加入列表，并按优先级排序
3. **执行进行中的操作**：遍历操作列表，调用每个操作的 `UpdateOperation()` 方法
4. **检查时间片**：如果超过 `MaxTimeSlice`，暂停执行，留到下一帧

这种设计确保了：
- **不会阻塞主线程**：即使有大量任务，也能保持流畅的帧率
- **优先级调度**：高优先级任务（如场景加载）优先执行
- **任务可中断**：每帧都会检查时间限制，超时立即暂停

## 1.7 设计思考

### 为什么采用三层架构？

三层架构的核心价值在于**职责分离**：

- **Facade 层**：专注于对外 API 的简洁性和易用性
- **业务层**：关注资源管理的业务逻辑
- **基础设施层**：提供通用的技术支撑

这种分层使得每个层级可以独立演进：
- 你可以替换 `OperationSystem` 的实现，而不影响 `ResourceManager`
- 你可以添加新的 `FileSystem`，而不需要修改 `ResourcePackage`

### `internal` 关键字的边界控制

YooAsset 大量使用 `internal` 关键字来控制访问权限：

- `YooAssets` 是 `public static`，对外暴露
- `ResourcePackage` 是 `public`，但内部方法很多是 `internal`
- `ResourceManager`、`OperationSystem` 是 `internal`，完全隐藏

这种设计体现了**信息隐藏原则**：
- 用户只需要知道 `YooAssets` 的 API
- 扩展开发者可以通过 `ResourcePackage` 进行定制
- 核心实现细节被严格隔离，防止误用

### 为什么不用事件驱动？

YooAsset 没有采用事件驱动模型，而是选择**帧驱动 + 状态机**：

- **帧驱动**：每帧检查并更新所有操作的状态
- **状态机**：每个操作内部维护自己的状态流转

这种设计的原因是：
1. **Unity 环境的特性**：MonoBehaviour 的 `Update` 本身就是帧驱动的
2. **状态可控**：每个操作的状态转换是确定性的，便于调试
3. **避免回调地狱**：相比于事件驱动，状态机的代码更线性

## 1.8 小结

这一章我们建立了 YooAsset 的整体认知：

- **三层架构**：Facade → 业务层 → 基础设施层，职责清晰
- **多 Package 设计**：支持独立管理多个资源包，灵活隔离
- **五种运行模式**：覆盖从开发到上线的全场景
- **帧驱动模型**：通过 `YooAssetsDriver` 驱动 `OperationSystem` 进行时间片调度

在接下来的章节中，我们会深入每个子系统，剖析它们的实现细节。记住这个整体架构，你会发现所有的细节都是为了支撑这个清晰的设计理念。
