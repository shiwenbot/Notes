---
title: "【教材】第7章：Editor 资源收集"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 7 章：Editor 资源收集

## 7.1 概述

在 YooAsset 的构建流程中，资源收集是整个打包链路的起点，它决定了"哪些资源打进哪个 Bundle"。这个决策过程发生在编辑器阶段，通过一套配置化的规则系统，将项目中的原始资源映射到最终的 AssetBundle 结构。

与 Addressables 通过 Label 和 Group 的扁平配置不同，YooAsset 采用了四层层级配置结构。这种设计让大型项目的资源管理更加清晰，不同模块、不同类型的资源可以有独立的收集策略。更重要的是，资源收集的结果会直接影响到运行时的加载性能和更新策略——一个合理的收集策略能够最小化包体体积、优化加载速度、降低更新带宽消耗。

资源收集系统的核心职责可以概括为三个方面：资源发现（遍历指定路径下的资源）、规则应用（根据打包规则、寻址规则、过滤规则确定资源属性）、依赖处理（解析并收集依赖资源）。这三个环节环环相扣，任何一个环节的失误都可能导致最终的构建结果不符合预期。

## 7.2 四层配置结构

YooAsset 的资源收集采用四层配置架构，从全局到局部依次为：Setting（设置）、Package（包裹）、Group（分组）、Collector（收集器）。这种层级结构既保证了全局配置的统一性，又允许局部配置的灵活性。

```csharp:Editor/AssetBundleCollector/AssetBundleCollectorSetting.cs [9:30]
[CreateAssetMenu(fileName = "AssetBundleCollectorSetting", menuName = "YooAsset/Create AssetBundle Collector Settings")]
public class AssetBundleCollectorSetting : ScriptableObject
{
    /// <summary>
    /// 显示包裹列表视图
    /// </summary>
    public bool ShowPackageView = false;

    /// <summary>
    /// 是否显示编辑器别名
    /// </summary>
    public bool ShowEditorAlias = false;

    /// <summary>
    /// 资源包名唯一化
    /// </summary>
    public bool UniqueBundleName = false;

    /// <summary>
    /// 包裹列表
    /// </summary>
    public List<AssetBundleCollectorPackage> Packages = new List<AssetBundleCollectorPackage>();
}
```

最顶层是 Setting，它是一个 ScriptableObject 资源文件，作为整个收集配置的入口。每个项目可以有多个 Setting 实例，但通常情况下一个 Setting 就足够了。Setting 维护了一个 Package 列表，每个 Package 代表一个独立的资源包裹，例如可以按"默认包"、"DLC包"、"补丁包"等维度划分。

Package 之下是 Group 层级，Group 是对资源的逻辑分组，比如可以按"UI资源"、"角色资源"、"场景资源"等划分。每个 Group 可以设置自己的资源标签（AssetTags）和激活规则（ActiveRule）。Group 的设计让同一 Package 下的资源可以有更细粒度的管理策略。

最底层是 Collector，这是实际执行资源收集的工作单元。每个 Collector 指定一个收集路径（CollectPath），可以是一个文件夹或单个资源文件，并配置了收集器类型（CollectorType）、打包规则（PackRule）、寻址规则（AddressRule）、过滤规则（FilterRule）等参数。

![收集器四层配置结构](../diagrams/ch07-collector-hierarchy.png)

*图 7-1：收集器四层配置结构*

CollectorType 是收集器的核心属性，它决定了资源的收集语义。YooAsset 定义了三种收集器类型：

```csharp:Editor/AssetBundleCollector/ECollectorType.cs [6:23]
public enum ECollectorType
{
    /// <summary>
    /// 收集参与打包的主资源对象，并写入到资源清单的资源列表里（可以通过代码加载）。
    /// </summary>
    MainAssetCollector,

    /// <summary>
    /// 收集参与打包的主资源对象，但不写入到资源清单的资源列表里（无法通过代码加载）。
    /// </summary>
    StaticAssetCollector,

    /// <summary>
    /// 收集参与打包的依赖资源对象，但不写入到资源清单的资源列表里（无法通过代码加载）。
    /// 注意：如果依赖资源对象没有被主资源对象引用，则不参与打包构建。
    /// </summary>
    DependAssetCollector,
}
```

MainAssetCollector 是最常用的收集器类型，它收集的资源会写入资源清单，运行时可以通过地址直接加载。StaticAssetCollector 收集的资源也会被打包，但不会写入清单，主要用于那些不需要直接加载但需要被其他资源引用的场景。DependAssetCollector 则专门用于收集依赖资源，它只有被主资源引用时才会参与打包。

配置数据采用了双轨存储机制：.asset 文件用于 Unity 编辑器内的可视化配置，.xml 文件用于版本控制和团队协作。这种设计兼顾了开发体验和协作效率，开发者在编辑器中修改配置后，可以导出为 XML 文件提交到版本控制系统。

## 7.3 五种规则接口解析

YooAsset 的规则系统由五个核心接口构成，每个接口负责资源收集流程中的一个特定环节。这些接口的设计遵循单一职责原则，通过组合不同的规则实现，可以覆盖绝大多数打包场景。

### IPackRule：打包规则接口

打包规则决定了资源最终被打入哪个 Bundle，这是资源收集系统中最核心的决策。IPackRule 接口定义非常简洁：

```csharp:Editor/AssetBundleCollector/CollectRules/IPackRule.cs [68:77]
/// <summary>
/// 资源打包规则接口
/// </summary>
public interface IPackRule
{
    /// <summary>
    /// 获取打包规则结果
    /// </summary>
    PackRuleResult GetPackRuleResult(PackRuleData data);
}
```

输入参数 PackRuleData 包含了资源的上下文信息：AssetPath（资源路径）、CollectPath（收集器路径）、GroupName（分组名称）、UserData（用户自定义数据）。输出参数 PackRuleResult 包含两个核心属性：BundleName（资源包名）和 BundleExtension（资源包扩展名）。

YooAsset 内置了多种常用的打包规则实现：

| 规则类名 | BundleName 生成逻辑 | 适用场景 |
|---------|-------------------|---------|
| PackSeparately | 去除扩展名的资源路径 | 需要独立加载的稀有资源 |
| PackDirectory | 资源所在父目录路径 | 同一目录下资源经常一起加载 |
| PackTopDirectory | 收集器路径下的顶级文件夹 | 大型模块的整体打包 |
| PackCollector | 收集器路径本身 | 将收集器内所有资源打包为一个 Bundle |
| PackGroup | 分组名称 | 按逻辑分组而非物理路径打包 |
| PackRawFile | 原始资源路径 | 原生文件（如 .jpg、.txt）不处理 |
| PackVideoFile | 原始资源路径，扩展名为原扩展名 | 视频文件的流式加载 |

### IAddressRule：寻址规则接口

寻址规则决定了资源在运行时的访问地址，只有 MainAssetCollector 收集的资源才需要设置地址。接口定义同样简洁：

```csharp:Editor/AssetBundleCollector/CollectRules/IAddressRule.cs [20:26]
/// <summary>
/// 寻址规则接口
/// </summary>
public interface IAddressRule
{
    string GetAssetAddress(AddressRuleData data);
}
```

内置的 AddressByFileName 规则使用文件名（不含扩展名）作为地址，AddressByFolderPath 规则使用相对收集器的路径作为地址。对于需要精确控制地址的场景，可以实现自定义规则，比如将资源路径中的特定前缀或后缀映射为简化地址。

### IFilterRule：过滤规则接口

过滤规则决定了哪些资源会被收集器收集。它由两个属性组成：

```csharp:Editor/AssetBundleCollector/CollectRules/IFilterRule.cs [20:36]
/// <summary>
/// 资源过滤规则接口
/// </summary>
public interface IFilterRule
{
    /// <summary>
    /// 搜寻的资源类型
    /// 说明：使用引擎方法搜索获取所有资源列表
    /// </summary>
    string FindAssetType { get; }

    /// <summary>
    /// 验证搜寻的资源是否为收集资源
    /// </summary>
    /// <returns>如果收集该资源返回TRUE</returns>
    bool IsCollectAsset(FilterRuleData data);
}
```

FindAssetType 属性返回 AssetDatabase.FindAssets() 方法支持的资源类型标识符，如 "t:Prefab"、"t:Texture2D" 等。IsCollectAsset 方法在遍历到每个资源时被调用，返回 true 则收集，返回 false 则跳过。

内置的 CollectAll 规则会收集所有类型的资源，CollectShader 只收集 Shader 资源。通过自定义过滤规则，可以实现复杂的筛选逻辑，比如只收集特定命名模式的资源，或根据资源的元数据决定是否收集。

### IIgnoreRule：忽略规则接口

忽略规则与过滤规则不同，它在全局层面起作用，用于排除某些特定资源参与打包。典型的应用场景是排除 Editor 目录下的资源、测试资源、或临时资源。忽略规则在 Package 级别配置，对所有 Collector 生效。

### IActiveRule：激活规则接口

激活规则决定了资源是否写入最终的资源清单。这对于 DLC 内容或可选内容非常有用——资源被打包但不会在初始清单中，只有在玩家购买或解锁相应内容后，才会通过补丁方式激活。激活规则在 Group 级别配置，可以基于标签、玩家等级、或其他业务逻辑决定资源的可见性。

这五个规则接口的职责边界清晰：FilterRule 决定"哪些资源进入收集流程"，IgnoreRule 决定"哪些资源被全局排除"，PackRule 决定"资源打包到哪个 Bundle"，AddressRule 决定"资源的访问地址"，ActiveRule 决定"资源是否写入清单"。通过合理组合这些规则，可以实现极其灵活的打包策略。

## 7.4 自定义规则注册机制

YooAsset 通过反射机制自动发现和注册自定义规则，开发者无需手动编写注册代码。这个过程依赖于两个核心组件：DisplayNameAttribute 特性和 RuleDisplayName 反射扫描。

DisplayNameAttribute 用于标记自定义规则的显示名称，这个名称会出现在编辑器的下拉菜单中：

```csharp:Editor/AssetBundleCollector/DisplayNameAttribute.cs [5:16]
/// <summary>
/// 编辑器显示名字
/// </summary>
public class DisplayNameAttribute : Attribute
{
    public string DisplayName;

    public DisplayNameAttribute(string name)
    {
        this.DisplayName = name;
    }
}
```

下面通过一个完整的 IPackRule 自定义示例来演示整个流程：

```csharp:Editor/AssetBundleCollector/DefaultRules/DefaultPackRule.cs [47:56]
/// <summary>
/// 以文件路径作为资源包名
/// 注意：每个文件独自打资源包
/// 例如："Assets/UIPanel/Shop/Image/backgroud.png" --> "assets_uipanel_shop_image_backgroud.bundle"
/// 例如："Assets/UIPanel/Shop/View/main.prefab" --> "assets_uipanel_shop_view_main.bundle"
/// </summary>
[DisplayName("资源包名: 文件路径")]
public class PackSeparately : IPackRule
{
    PackRuleResult IPackRule.GetPackRuleResult(PackRuleData data)
    {
        string bundleName = PathUtility.RemoveExtension(data.AssetPath);
        PackRuleResult result = new PackRuleResult(bundleName, DefaultPackRule.AssetBundleFileExtension);
        return result;
    }
}
```

自定义规则的完整步骤如下：

1. **创建规则类**：实现相应的规则接口（如 IPackRule）
2. **添加 DisplayName 特性**：使用 `[DisplayName("显示名称")]` 标记类
3. **实现接口方法**：根据输入参数计算并返回结果
4. **放置位置**：将规则类放在 YooAsset 能够扫描到的程序集中（通常是同一程序集或通过 Assembly Definition 引用的程序集）

YooAsset 在编辑器启动时会通过反射扫描所有程序集中的规则类型，并建立显示名称与类型之间的映射关系。这个过程只在编辑器初始化时执行一次，后续的规则实例化通过 Activator.CreateInstance 完成，性能开销可以忽略不计。

自定义规则可以访问 PackRuleData 中的所有上下文信息，包括 UserData 字段。这为项目特定的打包逻辑提供了扩展点——比如可以在 UserData 中存储前缀、后缀、或其他配置信息，然后在规则中解析并应用。

## 7.5 收集数据流：从配置到 CollectResult

资源收集的实际执行流程始于 AssetBundleCollectorSetting 的 BeginCollect 方法。这个方法创建了一个 CollectCommand 对象，封装了收集任务的所有配置参数，然后传递给 Package 的 GetCollectAssets 方法。

```csharp:Editor/AssetBundleCollector/AssetBundleCollectorSetting.cs [95:123]
/// <summary>
/// 收集指定包裹的资源文件
/// </summary>
public CollectResult BeginCollect(string packageName, bool simulateBuild, bool useAssetDependencyDB)
{
    if (string.IsNullOrEmpty(packageName))
        throw new Exception("Build package name is null or empty !");

    // 检测配置合法性
    var package = GetPackage(packageName);
    package.CheckConfigError();

    // 创建资源收集命令
    IIgnoreRule ignoreRule = AssetBundleCollectorSettingData.GetIgnoreRuleInstance(package.IgnoreRuleName);
    var command = new CollectCommand(packageName, ignoreRule);
    command.SimulateBuild = simulateBuild;
    command.UniqueBundleName = UniqueBundleName;
    command.UseAssetDependencyDB = useAssetDependencyDB;
    command.EnableAddressable = package.EnableAddressable;
    command.SupportExtensionless = package.SupportExtensionless;
    command.LocationToLower = package.LocationToLower;
    command.IncludeAssetGUID = package.IncludeAssetGUID;
    command.AutoCollectShaders = package.AutoCollectShaders;

    // 开始收集工作
    var collectAssets = package.GetCollectAssets(command);
    var collectResult = new CollectResult(command, collectAssets);
    return collectResult;
}
```

CollectCommand 是一个值类型，它在整个收集过程中传递，避免了参数的逐个传递。它包含了配置参数（如 PackageName、UniqueBundleName）、功能开关（如 EnableAddressable、AutoCollectShaders）、以及工具对象（如 IgnoreRule、AssetDependency）。

Package.GetCollectAssets 方法会遍历其下的所有 Group，每个 Group 再遍历其下的所有 Collector。对于每个 Collector，调用 GetAllCollectAssets 方法执行实际的资源收集：

```csharp:Editor/AssetBundleCollector/AssetBundleCollector.cs [137:158]
/// <summary>
/// 获取打包收集的资源文件
/// </summary>
public List<CollectAssetInfo> GetAllCollectAssets(CollectCommand command, AssetBundleCollectorGroup group)
{
    bool ignoreStaticCollector = command.IsFlagSet(ECollectFlags.IgnoreStaticCollector);
    if (ignoreStaticCollector)
    {
        if (CollectorType == ECollectorType.StaticAssetCollector)
            return new List<CollectAssetInfo>();
    }

    bool ignoreDependCollector = command.IsFlagSet(ECollectFlags.IgnoreDependCollector);
    if (ignoreDependCollector)
    {
        if (CollectorType == ECollectorType.DependAssetCollector)
            return new List<CollectAssetInfo>();
    }

    Dictionary<string, CollectAssetInfo> result = new Dictionary<string, CollectAssetInfo>(1000);

    // 收集打包资源路径
    List<string> findAssets = new List<string>();
    if (AssetDatabase.IsValidFolder(CollectPath))
    {
        IFilterRule filterRuleInstance = AssetBundleCollectorSettingData.GetFilterRuleInstance(FilterRuleName);
        string findAssetType = filterRuleInstance.FindAssetType;
        string searchFolder = CollectPath;
        string[] findResult = EditorTools.FindAssets(findAssetType, searchFolder);
        findAssets.AddRange(findResult);
    }
    else
    {
        string assetPath = CollectPath;
        findAssets.Add(assetPath);
    }
    // ... 后续处理逻辑
}
```

收集过程分为三个阶段：资源发现、规则应用、依赖解析。资源发现阶段通过 AssetDatabase.FindAssets() 遍历收集路径下的所有资源；规则应用阶段为每个资源应用过滤规则、打包规则、寻址规则；依赖解析阶段通过 AssetDependencyDatabase 获取资源的所有依赖。

每个资源最终会被封装为一个 CollectAssetInfo 对象，包含以下核心信息：

- **AssetInfo**：资源的基本信息（路径、类型、GUID 等）
- **CollectorType**：收集器类型
- **BundleName**：最终打包的 Bundle 名称
- **Address**：运行时访问地址
- **AssetTags**：资源的标签列表
- **DependAssets**：依赖资源列表

所有 CollectAssetInfo 对象会被汇总到一个 List 中，然后封装到 CollectResult 对象返回：

```csharp:Editor/AssetBundleCollector/CollectResult.cs [6:23]
public class CollectResult
{
    /// <summary>
    /// 收集命令
    /// </summary>
    public CollectCommand Command { private set; get; }

    /// <summary>
    /// 收集的资源信息列表
    /// </summary>
    public List<CollectAssetInfo> CollectAssets { private set; get; }

    public CollectResult(CollectCommand command, List<CollectAssetInfo> collectAssets)
    {
        Command = command;
        CollectAssets = collectAssets;
    }
}
```

CollectResult 是资源收集阶段的最终产出，它会传递给后续的构建流程（TaskGetBuildMap），用于生成最终的 BuildMap 和资源清单。整个收集过程是幂等的——相同的配置总是产生相同的 CollectResult，这对于增量构建和缓存机制至关重要。

## 7.6 AssetDependencyDatabase：依赖缓存

资源依赖关系的解析是资源收集过程中性能开销最大的环节。Unity 的 AssetDatabase.GetDependencies() 方法虽然功能强大，但在处理大量资源时会成为性能瓶颈。假设项目中有 1000 个资源，每次收集都需要调用 1000 次 GetDependencies，如果每次调用耗时 30ms，总耗时将达到 30 秒。

AssetDependencyDatabase 通过缓存机制解决了这个问题。它在首次收集时构建一个完整的依赖关系数据库，后续收集直接从数据库读取，将耗时降低到 1 秒以内。

```csharp:Editor/AssetBundleCollector/AssetDependencyDatabase.cs [102:120]
// 查找新增或变动资源
var allAssetPaths = AssetDatabase.GetAllAssetPaths();
foreach (var assetPath in allAssetPaths)
{
    if (_database.TryGetValue(assetPath, out DependencyInfo cacheInfo))
    {
        var dependHash = AssetDatabase.GetAssetDependencyHash(assetPath);
        if (dependHash.ToString() != cacheInfo.DependHash)
        {
            _database[assetPath] = CreateDependencyInfo(assetPath);
        }
    }
    else
    {
        var newCacheInfo = CreateDependencyInfo(assetPath);
        _database.Add(assetPath, newCacheInfo);
    }
}
```

依赖缓存的核心是一个 Dictionary<string, DependencyInfo>，键是资源路径，值是 DependencyInfo 对象。DependencyInfo 包含两个核心属性：DependHash（依赖哈希）和 DependGUIDs（依赖资源的 GUID 列表）。

DependHash 是 Unity 提供的哈希值，它聚合了源资源路径、源资源、元文件、目标平台以及导入器版本的信息。当这个哈希值发生变化时，说明资源的依赖关系可能发生了变化，需要重新解析。

缓存的构建过程分为三个步骤：

1. **加载缓存文件**：如果存在缓存文件，尝试从磁盘加载已有的缓存数据
2. **验证并清理**：检查缓存中的资源是否仍然存在，移除已删除资源的缓存条目
3. **增量更新**：遍历项目中的所有资源，对于新增或 DependHash 变化的资源，重新解析其依赖关系

缓存文件采用二进制格式存储，包含文件版本、条目数量、每个条目的资源路径、依赖哈希和依赖 GUID 列表。这种格式紧凑且读取快速，适合存储大量数据。

依赖缓存的失效时机有三个：资源被删除（从数据库中移除）、资源被修改（DependHash 变化）、缓存文件不存在（首次构建）。这种增量更新机制确保了缓存的一致性，同时最小化了重新解析的开销。

获取依赖时，AssetDependencyDatabase 提供了递归和非递归两种模式：

```csharp:Editor/AssetBundleCollector/AssetDependencyDatabase.cs [175:196]
/// <summary>
///  获取资源的依赖列表
/// </summary>
public string[] GetDependencies(string assetPath, bool recursive)
{
    // 注意：机制上不允许存在未收录的资源
    if (_database.ContainsKey(assetPath) == false)
    {
        throw new Exception($"Fatal : can not found cache info : {assetPath}");
    }

    var result = new HashSet<string>();

    // 注意：递归收集依赖时，依赖列表中包含主资源
    if (recursive)
        result.Add(assetPath);

    // 收集依赖
    CollectDependencies(assetPath, assetPath, result, recursive);

    return result.ToArray();
}
```

递归模式会收集所有层级的依赖资源，非递归模式只收集直接依赖。对于大多数打包场景，递归模式是更常用的选择，因为它能够确保所有需要的资源都被包含进来。

依赖缓存的性能优化效果显著：在一个包含 1000 个资源的中型项目中，未启用缓存时的收集耗时约 30 秒，启用缓存后降至 1 秒以内。对于包含数万资源的大型项目，这个优化差异更加明显。

## 7.7 设计思考

### 层级 vs 扁平配置的权衡

YooAsset 采用四层配置结构，而 Addressables 采用扁平的 Group+Label 结构，这两种设计各有优劣。层级结构的优势在于模块化和可维护性——不同功能模块的资源配置相互隔离，修改一个模块的配置不会影响其他模块。这对于大型多人协作项目尤为重要，不同团队可以负责不同的 Package 或 Group，避免配置冲突。

扁平结构的优势在于简洁性和灵活性——所有配置在一个视图中可见，拖拽操作更加直观。但对于资源量巨大的项目，单一的配置视图会变得臃肿不堪，查找和修改配置变得困难。

YooAsset 通过 Package-Group-C ollector 的层级划分，在模块化和简洁性之间找到了平衡点。Package 对应独立的资源包裹，Group 对应逻辑分组，Collector 对应具体的收集路径。这种结构既保证了配置的清晰性，又提供了足够的灵活性。

### 规则接口的组合爆炸问题

五个规则接口（IFilterRule、IPackRule、IAddressRule、IIgnoreRule、IActiveRule）的组合提供了极大的灵活性，但也带来了配置复杂度的问题。理论上，每个 Collector 可以配置不同的规则组合，这会导致配置空间的爆炸。

实际项目中，大多数场景只需要有限的几种规则组合。YooAsset 通过内置规则覆盖了 80% 的常见场景，剩下的 20% 通过自定义规则实现。为了降低配置复杂度，建议项目层面制定规则使用规范——比如所有 UI 资源使用 PackDirectory，所有角色资源使用 PackSeparately，所有原生文件使用 PackRawFile。

规则接口的设计遵循"接口隔离原则"——每个接口只负责一个维度，而不是设计一个包含所有方法的大接口。这种设计使得规则的组合更加灵活，但也要求开发者清楚每个接口的职责边界。

### 依赖缓存的持久化策略

AssetDependencyDatabase 将缓存持久化到磁盘文件，这是在"构建速度"和"磁盘空间"之间做的权衡。缓存文件的大小与项目中的资源数量成正比，对于一个中型项目，缓存文件可能达到数十 MB。

另一种选择是不持久化缓存，每次构建时重新构建。这在小型项目中可行，但在大型项目中会显著增加构建时间。YooAsset 采用了折中方案：持久化缓存，但提供失效检测机制，确保缓存的一致性。

缓存文件的存储位置也是一个考虑因素。YooAsset 将其放在 Library 目录下，这是 Unity 的临时文件目录，不会被版本控制。这避免了缓存文件冲突的问题，但也意味着每个开发者的机器上都会有独立的缓存副本。

## 7.8 小结

资源收集是 YooAsset 构建流程的第一步，也是最重要的一步。它通过四层配置结构（Setting-Package-Group-C ollector）和五种规则接口（IFilterRule、IPackRule、IAddressRule、IIgnoreRule、IActiveRule），将项目中的原始资源映射到最终的 AssetBundle 结构。

四层配置结构提供了良好的模块化和可维护性，不同层级负责不同的配置维度。五种规则接口各司其职，通过组合实现灵活的打包策略。自定义规则机制通过反射和 DisplayNameAttribute，让开发者无需修改框架代码即可扩展功能。

AssetDependencyDatabase 通过缓存机制大幅提升了依赖解析的性能，将大型项目的收集时间从数十秒降低到一秒以内。增量更新机制确保了缓存的一致性，同时最小化了重新解析的开销。

资源收集系统的设计体现了"约定优于配置"的理念——内置规则覆盖常见场景，自定义规则处理特殊需求。这种设计让 YooAsset 既开箱即用，又具备足够的扩展性，适应各种规模和类型的项目需求。
