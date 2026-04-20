---
title: "【教材】第8章：Editor 构建管线"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 8 章：Editor 构建管线

## 8.1 概述

在 YooAsset 的资源管理体系中，构建管线扮演着"承上启下"的关键角色：上游承接收集系统整理好的资源集合，下游输出可部署的 AssetBundle 包体。如果把收集系统比作"食材准备车间"，那构建管线就是"中央厨房"，将分散的食材按配方加工成标准化的预制菜。

YooAsset 的构建管线设计采用了经典的 **Pipeline-Filter 架构模式**。这种模式的核心思想是将复杂流程拆解为独立的处理阶段（Pipeline），每个阶段再细分为多个可组合的过滤任务（Filter/Task）。在 YooAsset 中，这一思想体现为 `IBuildTask` 原子接口和 `BuildRunner` 执行器，通过组合不同的 Task 链，实现了四条特性各异的构建管线，既复用了核心逻辑，又保持了各管线的独特性。

这种架构带来的好处是显而易见的：新增构建规则只需实现新的 Task，无需修改现有代码；不同管线间的差异通过 Task 组合自然体现，而非硬编码的 if-else 分支。这符合单一职责原则和开闭原则，为系统的长期演进奠定了基础。

## 8.2 BuildSystem 三件套

### 8.2.1 IBuildTask：原子任务接口

构建系统的基本单元是 `IBuildTask` 接口，它定义了所有构建任务必须遵守的契约：

```csharp:Editor/AssetBundleBuilder/BuildSystem/IBuildTask.cs [1:8]
namespace YooAsset.Editor
{
    public interface IBuildTask
    {
        void Run(BuildContext context);
    }
}
```

这个接口的设计极其简洁：每个 Task 只需实现一个 `Run` 方法，接收 `BuildContext` 作为参数。这种"单方法接口"设计有深刻的考量：它强制每个 Task 只关注自身的逻辑，将上下文传递、异常处理、流程控制等横切关注点交给上层框架处理。

在实现层面，每个 Task 都是无状态的纯逻辑单元，通过名称约定来体现所属管线。例如 `TaskBuilding_BBP` 属于 BuiltinBuildPipeline，`TaskUpdateBundleInfo_ESBP` 属于 EditorSimulateBuildPipeline。这种命名约定让开发者一眼就能识别 Task 的归属，便于代码组织和维护。

### 8.2.2 BuildContext：类型安全的字典容器

`BuildContext` 是 Task 间数据传递的核心载体，本质上是一个以 `Type` 为 Key 的字典：

```csharp:Editor/AssetBundleBuilder/BuildSystem/BuildContext.cs [8:48]
public class BuildContext
{
    private readonly Dictionary<System.Type, IContextObject> _contextObjects = new Dictionary<System.Type, IContextObject>();

    public void SetContextObject(IContextObject contextObject)
    {
        if (contextObject == null)
            throw new ArgumentNullException("contextObject");

        var type = contextObject.GetType();
        if (_contextObjects.ContainsKey(type))
            throw new Exception($"Context object {type} is already existed.");

        _contextObjects.Add(type, contextObject);
    }

    public T GetContextObject<T>() where T : IContextObject
    {
        var type = typeof(T);
        if (_contextObjects.TryGetValue(type, out IContextObject contextObject))
        {
            return (T)contextObject;
        }
        else
        {
            throw new Exception($"Not found context object : {type}");
        }
    }

    public T TryGetContextObject<T>() where T : IContextObject
    {
        var type = typeof(T);
        if (_contextObjects.TryGetValue(type, out IContextObject contextObject))
        {
            return (T)contextObject;
        }
        else
        {
            return default;
        }
    }
}
```

**为什么用 Type 做 Key 而非 string？** 这是类型安全与性能的双重考量。使用 Type 作为 Key，编译器能在编译期检查类型匹配，避免拼写错误；泛型方法 `GetContextObject<T>()` 提供了简洁的调用方式，无需手动转换类型。此外，Type 的哈希计算是 CLR 内部优化的，比字符串比较更高效。

`BuildContext` 的生命周期贯穿整个构建流程，早期的 Task 将收集结果、构建参数等存入 Context，后续 Task 从中读取数据并写入新的结果。这种"接力棒"式的数据流动，避免了 Task 间的直接依赖，实现了低耦合。

### 8.2.3 BuildRunner：线性执行器

`BuildRunner` 是构建流程的总指挥，负责按顺序执行 Task 链并提供异常处理、性能统计等横切服务：

```csharp:Editor/AssetBundleBuilder/BuildSystem/BuildRunner.cs [23:63]
public static BuildResult Run(List<IBuildTask> pipeline, BuildContext context)
{
    if (pipeline == null)
        throw new ArgumentNullException("pipeline");
    if (context == null)
        throw new ArgumentNullException("context");

    BuildResult buildResult = new BuildResult();
    buildResult.Success = true;
    TotalSeconds = 0;
    for (int i = 0; i < pipeline.Count; i++)
    {
        IBuildTask task = pipeline[i];
        try
        {
            _buildWatch = Stopwatch.StartNew();
            string taskName = task.GetType().Name.Split('_')[0];
            BuildLogger.Log($"--------------------------------------------->{taskName}<--------------------------------------------");
            task.Run(context);
            _buildWatch.Stop();

            // 统计耗时
            int seconds = GetBuildSeconds();
            TotalSeconds += seconds;
            BuildLogger.Log($"{taskName} It takes {seconds} seconds in total");
        }
        catch (Exception e)
        {
            EditorTools.ClearProgressBar();
            buildResult.FailedTask = task.GetType().Name;
            buildResult.ErrorInfo = e.ToString();
            buildResult.ErrorStack = e.StackTrace;
            buildResult.Success = false;
            break;
        }
    }

    // 返回运行结果
    BuildLogger.Log($"Total build process time: {TotalSeconds} seconds");
    return buildResult;
}
```

`BuildRunner` 的设计体现了"异常熔断"模式：一旦某个 Task 抛出异常，立即终止流程并记录失败位置。这种快速失败策略避免了级联错误，让开发者能快速定位问题。同时，每个 Task 的执行时间都被独立统计，帮助开发者识别性能瓶颈。

这种线性执行模型虽然简单，但在构建场景下是合理的：资源打包本质上是串行操作（依赖关系决定了顺序），并行化带来的复杂性不值得收益。当然，对于独立的 Task（如多个 Bundle 的加密），未来可以探索并行化优化。

## 8.3 四条管线横向对比

YooAsset 提供了四条构建管线，每条管线都针对特定的使用场景。理解它们的差异，是选择合适管线的前提。

### 8.3.1 对比表格

| 管线名称 | 核心打包 API | 任务数 | 特有 Task | 适用场景 |
|---------|------------|-------|----------|---------|
| **BuiltinBuildPipeline (BBP)** | `BuildPipeline.BuildAssetBundles` | 11 | `TaskBuilding_BBP`, `TaskVerifyBuildResult_BBP` | 标准资源打包，生产环境 |
| **ScriptableBuildPipeline (SBP)** | `ContentPipeline.BuildAssetBundles` | 11 | `TaskBuilding_SBP`, `TaskLinkDepend_SBP` | 高级打包需求，精细控制 |
| **RawFileBuildPipeline (RFBP)** | `BuildPipeline.BuildAssetBundles` | 11 | `TaskBuilding_RFBP` | 无压缩原始文件打包 |
| **EditorSimulateBuildPipeline (ESBP)** | 无（模拟构建） | 4 | `TaskUpdateBundleInfo_ESBP` | 编辑器快速模拟，开发调试 |

从任务数量可以看出，BBP、SBP、RFBP 都是完整构建管线（11 个 Task），而 ESBP 是精简版（4 个 Task）。这反映了 ESBP 的定位：跳过实际的 AssetBundle 构建，只生成清单文件，用于编辑器内快速迭代。

### 8.3.2 管线任务链

以 **BuiltinBuildPipeline** 为例，其完整任务链展示了标准构建流程的全貌：

```csharp:Editor/AssetBundleBuilder/BuildPipeline/BuiltinBuildPipeline/BuiltinBuildPipeline.cs [25:42]
private List<IBuildTask> GetDefaultBuildPipeline()
{
    List<IBuildTask> pipeline = new List<IBuildTask>
        {
            new TaskPrepare_BBP(),                  // 1. 准备构建参数
            new TaskGetBuildMap_BBP(),              // 2. 生成构建映射
            new TaskBuilding_BBP(),                 // 3. 执行 Unity 构建
            new TaskVerifyBuildResult_BBP(),        // 4. 验证构建结果
            new TaskEncryption_BBP(),               // 5. 加密资源包
            new TaskUpdateBundleInfo_BBP(),         // 6. 更新包信息
            new TaskCreateManifest_BBP(),           // 7. 创建清单文件
            new TaskCreateReport_BBP(),             // 8. 生成构建报告
            new TaskCreatePackage_BBP(),            // 9. 创建包裹文件
            new TaskCopyBuildinFiles_BBP(),         // 10. 拷贝内置文件
            new TaskCreateCatalog_BBP()             // 11. 创建目录文件
        };
    return pipeline;
}
```

相比之下，**EditorSimulateBuildPipeline** 则大幅简化：

```csharp:Editor/AssetBundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/EditorSimulateBuildPipeline.cs [25:35]
private List<IBuildTask> GetDefaultBuildPipeline()
{
    List<IBuildTask> pipeline = new List<IBuildTask>
        {
            new TaskPrepare_ESBP(),                 // 1. 准备构建参数
            new TaskGetBuildMap_ESBP(),             // 2. 生成构建映射
            new TaskUpdateBundleInfo_ESBP(),        // 3. 更新包信息（模拟）
            new TaskCreateManifest_ESBP()           // 4. 创建清单文件
        };
    return pipeline;
}
```

![四条构建管线对比](../diagrams/ch08-pipeline-compare.png)
*图 8-1：四条构建管线任务对比*

从图中可以看出，ESBP 跳过了实际的构建、验证、加密等耗时操作，只保留了核心的映射和清单生成逻辑。这使得它在编辑器模式下能快速完成"伪构建"，让开发者无需等待完整打包就能测试资源流程。

## 8.4 从收集到构建：TaskGetBuildMap

`TaskGetBuildMap` 是连接收集系统和构建系统的桥梁，它将收集结果转换为构建系统可理解的 `BuildMapContext`。

### 8.4.1 调用收集器

Task 的第一步是调用收集器获取所有需要打包的资源：

```csharp:Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskGetBuildMap.cs [15:26]
public BuildMapContext CreateBuildMap(bool simulateBuild, BuildParameters buildParameters)
{
    BuildMapContext context = new BuildMapContext();
    var packageName = buildParameters.PackageName;

    Dictionary<string, BuildAssetInfo> allBuildAssetInfos = new Dictionary<string, BuildAssetInfo>(1000);

    // 1. 获取所有收集器收集的资源
    bool useAssetDependencyDB = buildParameters.UseAssetDependencyDB;
    var collectResult = AssetBundleCollectorSettingData.Setting.BeginCollect(packageName, simulateBuild, useAssetDependencyDB);
    List<CollectAssetInfo> allCollectAssets = collectResult.CollectAssets;
```

这里的 `useAssetDependencyDB` 参数是性能优化的关键：Unity 的依赖分析非常耗时，YooAsset 通过缓存依赖关系数据库，将后续构建的依赖分析从分钟级降到秒级。

### 8.4.2 共享资源打包规则

一个常见场景是：A 和 B 两个 Bundle 都依赖资源 C，但 C 本身没有设置 BundleName。按照传统规则，C 会被同时打入 A 和 B，导致冗余。YooAsset 的共享资源打包规则通过 `ProcessingPackShareBundle` 方法解决这个问题：

```csharp:Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskGetBuildMap.cs [209:225]
protected virtual void ProcessingPackShareBundle(BuildParameters buildParameters, CollectCommand command, BuildAssetInfo buildAssetInfo)
{
    PackRuleResult packRuleResult = GetShareBundleName(buildAssetInfo);
    if (packRuleResult.IsValid() == false)
        return;

    // 处理单个引用的共享资源
    if (buildAssetInfo.GetReferenceBundleCount() <= 1)
    {
        if (buildParameters.SingleReferencedPackAlone == false)
            return;
    }

    // 设置共享资源包名
    string shareBundleName = packRuleResult.GetShareBundleName(command.PackageName, command.UniqueBundleName);
    buildAssetInfo.SetBundleName(shareBundleName);
}
```

这段逻辑的核心是：当一个资源被多个 Bundle 引用时，将其打包到独立的共享包中，避免重复。`SingleReferencedPackAlone` 参数则控制单独引用的资源是否也要独立打包，这取决于项目对包体大小与加载性能的权衡。

### 8.4.3 BuildMapContext 数据结构

经过处理后的数据被封装到 `BuildMapContext` 中，它包含：
- `AssetFileCount`：资源文件总数
- `Command`：收集命令（包含打包策略等配置）
- `Collection`：打包 Bundle 的集合（按 BundleName 分组的所有资源）

这个 Context 被存入 `BuildContext`，供后续 Task 使用。通过将复杂数据结构封装在 Context 对象中，避免了参数传递的复杂化，符合"数据封装"的设计原则。

## 8.5 构建产物的生成链

构建管线的最终输出是可部署的资源包和清单文件，这一过程由多个 Task 协作完成。

### 8.5.1 Hash/CRC/Size 的计算差异

不同管线对 Bundle 信息的计算方式不同。以 **EditorSimulateBuildPipeline** 为例，它跳过了实际构建，因此需要"模拟"这些信息：

```csharp:Editor/AssetBundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskUpdateBundleInfo_ESBP.cs [14:34]
protected override string GetUnityHash(BuildBundleInfo bundleInfo, BuildContext context)
{
    return "00000000000000000000000000000000"; //32位
}
protected override uint GetUnityCRC(BuildBundleInfo bundleInfo, BuildContext context)
{
    return 0;
}
protected override string GetBundleFileHash(BuildBundleInfo bundleInfo, BuildParametersContext buildParametersContext)
{
    string filePath = bundleInfo.PackageSourceFilePath;
    return GetFilePathTempHash(filePath);
}
protected override uint GetBundleFileCRC(BuildBundleInfo bundleInfo, BuildParametersContext buildParametersContext)
{
    return 0;
}
protected override long GetBundleFileSize(BuildBundleInfo bundleInfo, BuildParametersContext buildParametersContext)
{
    return GetBundleTempSize(bundleInfo);
}
```

可以看到，ESBP 使用文件路径的 MD5 作为临时 Hash，文件大小通过累加所有源文件大小来估算。这些"假数据"足以支持编辑器模拟，但绝对不能用于生产环境。

相比之下，**BuiltinBuildPipeline** 会从 Unity 构建结果中读取真实的 Hash、CRC 和 Size，确保清单文件的准确性。

### 8.5.2 加密服务接入

YooAsset 通过 `IEncryptionServices` 接口支持资源加密，开发者可以实现自己的加密算法：

```csharp
public interface IEncryptionServices
{
    ulong EncryptOffset(ulong fileOffset);
    byte[] EncryptData(byte[] fileData);
}
```

`TaskEncryption` 会遍历所有 Bundle，调用这个接口对文件进行加密处理。加密是可选的，如果不提供服务，Bundle 会以明文形式存储。

### 8.5.3 清单文件的四种格式

`TaskCreateManifest` 负责生成清单文件，它创建四种产物：

```csharp:Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskCreateManifest.cs [70:103]
// 创建资源清单文本文件
{
    string fileName = YooAssetSettingsData.GetManifestJsonFileName(buildParameters.PackageName, buildParameters.PackageVersion);
    string filePath = $"{packageOutputDirectory}/{fileName}";
    ManifestTools.SerializeToJson(filePath, manifest);
    BuildLogger.Log($"Create package manifest file: {filePath}");
}

// 创建资源清单二进制文件
string packageHash;
string packagePath;
{
    string fileName = YooAssetSettingsData.GetManifestBinaryFileName(buildParameters.PackageName, buildParameters.PackageVersion);
    packagePath = $"{packageOutputDirectory}/{fileName}";
    ManifestTools.SerializeToBinary(packagePath, manifest, buildParameters.ManifestProcessServices);
    packageHash = HashUtility.FileCRC32(packagePath);
    BuildLogger.Log($"Create package manifest file: {packagePath}");
}

// 创建资源清单哈希文件
{
    string fileName = YooAssetSettingsData.GetPackageHashFileName(buildParameters.PackageName, buildParameters.PackageVersion);
    string filePath = $"{packageOutputDirectory}/{fileName}";
    FileUtility.WriteAllText(filePath, packageHash);
    BuildLogger.Log($"Create package manifest hash file: {filePath}");
}

// 创建资源清单版本文件
{
    string fileName = YooAssetSettingsData.GetPackageVersionFileName(buildParameters.PackageName);
    string filePath = $"{packageOutputDirectory}/{fileName}";
    FileUtility.WriteAllText(filePath, buildParameters.PackageVersion);
    BuildLogger.Log($"Create package manifest version file: {filePath}");
}
```

四种文件各有用途：
- **JSON 清单**：人类可读，用于调试和版本控制
- **二进制清单**：运行时加载的高效格式，体积更小
- **哈希文件**：用于快速验证清单完整性
- **版本文件**：记录当前版本号，方便版本切换

## 8.6 产物目录布局

构建管线的输出遵循清晰的目录层次，理解这一结构对于部署和更新至关重要。

### 8.6.1 三级目录结构

构建产物的目录结构分为三级：

```
BuildOutputRoot/
├── {BuildTarget}/              # 第一级：目标平台
│   ├── {PackageName}/          # 第二级：包裹名称
│   │   ├── {PackageVersion}/   # 第三级：版本目录
│   │   │   ├── manifest.json   # 清单文件
│   │   │   ├── bundle_files/   # 资源包
```

每一级都有明确的职责：平台级隔离确保跨平台构建不冲突，包裹级隔离支持多包管理，版本级隔离支持热更新回滚。

### 8.6.2 内置资源的五种拷贝策略

`EBuildinFileCopyOption` 枚举定义了内置资源（首包必需资源）的拷贝策略：

```csharp:Editor/AssetBundleBuilder/BuildParameters.cs [89:89]
public EBuildinFileCopyOption BuildinFileCopyOption = EBuildinFileCopyOption.None;
```

五种策略分别是：
- **None**：不拷贝，适用于纯 CDN 分发
- **ClearAndCopy**：清空后拷贝，适用于完全覆盖更新
- **OnlyCopy**：仅拷贝不删除，适用于增量更新
- **ClearWhenFileVersionChange**：版本变更时清空
- **AlwaysClear**：始终清空，适用于开发调试

`TaskCopyBuildinFiles` 根据这个策略，将指定的 Bundle 从构建产物拷贝到 `StreamingAssets` 目录，确保首包资源随 App 一起发布。

![构建产物目录布局](../diagrams/ch08-build-output.png)
*图 8-2：构建产物目录布局*

从图中可以看出，构建产物既包含用于远程更新的版本包目录，也包含用于首包部署的 StreamingAssets 目录。开发者可以通过 `BuildinFileCopyOption` 精确控制哪些资源需要内置，从而在首包大小和更新灵活性之间找到平衡。

## 8.7 设计思考

### 8.7.1 Pipeline-Filter 模式的优势

YooAsset 的构建系统是 Pipeline-Filter 模式的教科书级实现。这种模式的核心价值在于**可组合性**：每个 Task 都是独立的"过滤器"，可以自由组合成不同的"管道"。当我们需要支持新的构建需求时，只需实现新的 Task 并插入到管道中，无需修改现有代码。

这种模式特别适合构建这种流程化场景：构建过程本质上是数据转换的链式操作，每个 Task 负责一个特定的转换步骤。Pipeline-Filter 模式天然地表达了这种数据流动，比硬编码的流程更直观、更易维护。

### 8.7.2 BuildContext 字典模式的优劣

使用 Type 为 Key 的字典来传递上下文，是一种"弱类型动态容器"模式。它的优势在于：
- **解耦**：Task 间无需直接依赖，通过 Context 间接通信
- **扩展**：新增上下文类型只需实现 `IContextObject`，无需修改接口
- **类型安全**：泛型方法保证编译期类型检查

但也有一些劣势：
- **隐式依赖**：Task 的依赖关系不明确，需要阅读代码才能知道需要哪些上下文
- **运行时错误**：如果某个 Task 忘记存入必需的上下文，错误只能在运行时暴露
- **调试困难**：Context 的动态特性使得静态分析工具难以追踪数据流

这种模式适合中等复杂度的系统，对于超大规模系统，可能需要更显式的依赖注入框架。但在 YooAsset 的场景下，这种简单有效的设计已经足够。

## 8.8 小结

Editor 构建管线是 YooAsset 资源管理体系的"出口"，它将精心设计的资源规划转化为可部署的数字资产。通过 Pipeline-Filter 架构，YooAsset 实现了灵活的构建流程，四条管线各司其职：BBP 担纲主力生产，SBP 提供高级特性，RFBP 支持特殊格式，ESBP 加速开发迭代。

构建系统的核心是三件套：`IBuildTask` 定义任务边界，`BuildContext` 传递上下文数据，`BuildRunner` 协调流程执行。这种设计既保证了代码的简洁性，又提供了足够的扩展性。从 `TaskGetBuildMap` 的映射生成，到 `TaskCreateManifest` 的清单输出，每个 Task 都专注于自己的职责，共同完成了从资源集合到部署包体的华丽转身。

理解构建管线，不仅能帮助你定制打包流程，更能让你深入体会 YooAsset "可组合、可扩展、可维护"的设计哲学。这正是优秀框架的标志：它不仅解决了当下的问题，还为未来的演进预留了空间。
