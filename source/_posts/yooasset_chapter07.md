---
title: "【教材】第07章：Editor 资源收集"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# Chapter 07: Editor 资源收集 — Spec

## 章节目标

深入剖析 YooAsset Editor 侧的资源收集配置体系，让读者理解"三层数据模型（Setting → Package → Group → Collector）"的层级关系、五种规则接口的职责边界、依赖数据库的缓存机制，以及收集流程从配置到 `CollectResult` 的完整数据流转路径。读完本章后，读者应能独立自定义规则实现并正确配置多包体收集策略。

## 内容边界

### 包含
- `AssetBundleCollectorSetting`（ScriptableObject 根配置）的数据结构与持久化方式
- `AssetBundleCollectorSettingData` 静态门面类的职责（单例加载、脏标记、XML 导入导出）
- `AssetBundleCollectorConfig`（XML 序列化 DTO）与 `.asset` 的双轨存储设计
- 三层容器（Package → Group → Collector）的字段定义与聚合关系
- `ECollectorType` 枚举（MainAssetCollector / DependAssetCollector / StaticAssetCollector）的语义区别
- 五种规则接口的方法签名与职责：`IFilterRule`、`IPackRule`、`IAddressRule`、`IActiveRule`、`IIgnoreRule`
- `DefaultRules/` 中每个内置规则实现的策略说明
- `DisplayNameAttribute` 与 `RuleDisplayName` 在 Inspector 下拉菜单中的反射注册机制
- `CollectCommand` 收集入参的组成与触发时机
- `CollectAssetInfo` 收集结果单元与 `CollectResult` 聚合输出的数据结构
- `AssetDependencyDatabase` 与 `AssetDependencyCache` 的缓存策略
- 与构建管线（`TaskGetBuildMap`）的衔接点说明

### 不包含
- `AssetBundleBuilder` 构建管线的具体任务链（第 8 章）
- 运行时加载 API
- `AssetArtScanner` / `AssetArtReporter` 美术规范扫描子系统
- CI/CD 脚本与命令行构建调用
- UI Toolkit UXML/USS 的详细实现

## 核心概念清单

| 概念 | 说明 | 重要程度 |
|---|---|---|
| `AssetBundleCollectorSetting` | ScriptableObject 根节点，持有 `Packages` 列表，配置序列化的唯一真相来源 | 高 |
| `AssetBundleCollectorSettingData` | 静态门面，负责 `.asset` 懒加载、`EditorUtility.SetDirty` 脏标记及 XML 导入导出 | 高 |
| `AssetBundleCollectorConfig` | XML 可序列化的 DTO，与 `.asset` 互转，支持跨项目配置共享 | 中 |
| `AssetBundleCollectorPackage` | 第一层容器，对应一个资源包，持有 `Groups` 列表及包级全局规则覆盖 | 高 |
| `AssetBundleCollectorGroup` | 第二层容器，对应一组资源路径，持有 `Collectors` 列表及组级规则（可继承或覆盖包级规则） | 高 |
| `AssetBundleCollector` | 叶级配置单元，指向具体目录/文件路径，携带 ECollectorType 和各规则名称 | 高 |
| `ECollectorType` | 三值枚举：MainAssetCollector（生成 Address）、DependAssetCollector（自动打包）、StaticAssetCollector（不参与自动依赖） | 高 |
| `IFilterRule` | 过滤规则接口，`IsCollectAsset(FilterRuleData)→bool`，决定某个文件是否应被收集 | 高 |
| `IPackRule` | 打包规则接口，`GetBundleName(PackRuleData)→PackRuleResult`，决定资源归属哪个 Bundle | 高 |
| `IAddressRule` | 寻址规则接口，`GetAssetAddress(AddressRuleData)→string`，决定资源的运行时 Address 键 | 高 |
| `IActiveRule` | 激活规则接口，`IsActivePackage()→bool`，决定某个 Package 是否参与当前构建 | 中 |
| `IIgnoreRule` | 忽略规则接口，`IsIgnoreFile(IgnoreRuleData)→bool`，决定某文件是否在依赖遍历时被跳过 | 中 |
| `DefaultPackRule` | 内置打包实现集：PackDirectory、PackTopDirectory、PackCollector、PackSeparately 等 | 高 |
| `DisplayNameAttribute` | 标注规则实现类的可读显示名，供 Inspector 下拉框通过反射枚举 | 中 |
| `CollectAssetInfo` | 单个收集资源的信息载体：AssetPath、BundleName、Address、AssetType、依赖列表 | 高 |
| `CollectResult` | 一次收集的聚合输出，持有 MainAssets、AllAssets 列表，供构建管线的 `TaskGetBuildMap` 消费 | 高 |
| `AssetDependencyDatabase` | 依赖数据库，封装 `AssetDatabase.GetDependencies` 结果的缓存层，避免重复 IO | 高 |

## 源码文件清单

| 文件路径 | 作用 | 引用方式 |
|---|---|---|
| `Editor/AssetBundleCollector/AssetBundleCollectorSetting.cs` | ScriptableObject 根配置 | 完整解读 |
| `Editor/AssetBundleCollector/AssetBundleCollectorSettingData.cs` | 静态门面：加载/保存/脏标记/XML 导入导出 | 完整解读 |
| `Editor/AssetBundleCollector/AssetBundleCollectorPackage.cs` | Package 层容器，`GetCollectResult()` 聚合入口 | 完整解读 |
| `Editor/AssetBundleCollector/AssetBundleCollectorGroup.cs` | Group 层容器，规则继承与覆盖逻辑 | 完整解读 |
| `Editor/AssetBundleCollector/AssetBundleCollector.cs` | 叶级配置单元，`GetCollectAssets()` 执行实际收集 | 完整解读 |
| `Editor/AssetBundleCollector/ECollectorType.cs` | 枚举定义 | 片段引用 |
| `Editor/AssetBundleCollector/CollectRules/IFilterRule.cs` | 过滤规则接口 | 片段引用 |
| `Editor/AssetBundleCollector/CollectRules/IPackRule.cs` | 打包规则接口 | 片段引用 |
| `Editor/AssetBundleCollector/CollectRules/IAddressRule.cs` | 寻址规则接口 | 片段引用 |
| `Editor/AssetBundleCollector/CollectRules/IActiveRule.cs` | 激活规则接口 | 片段引用 |
| `Editor/AssetBundleCollector/CollectRules/IIgnoreRule.cs` | 忽略规则接口 | 片段引用 |
| `Editor/AssetBundleCollector/DefaultRules/DefaultPackRule.cs` | 内置打包规则实现集 | 片段引用 |
| `Editor/AssetBundleCollector/DefaultRules/DefaultAddressRule.cs` | 内置寻址规则实现集 | 片段引用 |
| `Editor/AssetBundleCollector/DisplayNameAttribute.cs` | 自定义 Attribute | 片段引用 |
| `Editor/AssetBundleCollector/CollectResult.cs` | 收集聚合输出 | 片段引用 |
| `Editor/AssetBundleCollector/AssetDependencyDatabase.cs` | 依赖数据库缓存层 | 完整解读 |
| `Editor/AssetBundleCollector/AssetDependencyCache.cs` | 单会话内存缓存 | 片段引用 |

## 验收标准

- [ ] 读者能用一句话解释 Setting / Package / Group / Collector 四层之间的组合关系
- [ ] 读者能区分三种 CollectorType 在构建时对 Address 生成和依赖追踪的不同行为
- [ ] 读者能实现一个自定义 `IPackRule`，并通过 `DisplayNameAttribute` 使其出现在 Editor 下拉菜单
- [ ] 读者能说明 `IFilterRule` 与 `IIgnoreRule` 的职责差异
- [ ] 读者能解释 `AssetDependencyDatabase` 的缓存意义及失效时机
- [ ] 读者能描述一次完整收集流程的调用链：CollectCommand → Package.GetCollectResult → Group → Collector.GetCollectAssets → CollectResult
- [ ] 读者能说明 `.asset` 与 `.xml` 双轨持久化的使用场景差异
- [ ] 读者能找到 `CollectResult` 被 `TaskGetBuildMap` 消费的位置
