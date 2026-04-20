---
title: "【教材】第09章：扩展开发指南"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# Chapter 09: 扩展开发指南 — Spec

## 章节目标

帮助读者系统理解 YooAsset 所有对外暴露的扩展点，掌握自定义文件系统、运行模式、资源收集规则、加密服务、构建流水线等机制的实现方式，使读者能够将 YooAsset 适配到微信小游戏、抖音小游戏、Google Play、自有 CDN 等非标准运行环境中，并能实现加密、自定义打包策略等个性化需求。

## 内容边界

### 包含
- `IFileSystem` 接口的全部方法职责，以及 `FileSystemParameters` / `FileSystemParametersDefine` 传参机制
- `CustomPlayModeParameters` 与 `EPlayMode.CustomPlayMode` 的使用方式
- Runtime Services 全部 7 个可扩展接口：`IRemoteServices`、`IDecryptionServices`、`IEncryptionServices`、`ICopyLocalFileServices`、`IManifestProcessServices`、`IManifestRestoreServices`、`IWebDecryptionServices`
- Editor 侧收集规则 5 个接口：`IFilterRule`、`IPackRule`、`IAddressRule`、`IActiveRule`、`IIgnoreRule`
- `IBuildPipeline`、`IBuildTask`、`IContextObject` 构建系统三件套及 UI 注册机制
- `GameAsyncOperation` 自定义异步操作扩展（`LoadGameObjectOperation` 示例）
- 官方 Mini Game 自定义文件系统参考实现（WechatFileSystem、TiktokFileSystem 等）
- `BundleResult` 抽象类与三种内置实现的关系

### 不包含
- YooAsset 内部实现细节（`PlayModeImpl`、`DefaultCacheFileSystem` 内部逻辑等 `internal` 类）
- 编辑器 UI / AssetBundleBuilderWindow 的界面布局细节
- AssetBundleReporter / AssetArtScanner 等只读工具
- `ShaderVariantCollector` 的具体使用流程

## 核心概念清单

| 概念 | 说明 | 重要程度 |
|---|---|---|
| `IFileSystem` | 自定义文件系统的唯一入口接口，共 15 个方法；通过 `FileSystemParameters` 以类型全名字符串动态反射实例化 | 高 |
| `FileSystemParameters` | 封装文件系统类名和 KV 参数字典，运行时通过 `Type.GetType` + `Activator.CreateInstance` 创建实例 | 高 |
| `CustomPlayModeParameters` | `EPlayMode.CustomPlayMode` 专用初始化参数，持有 `List<FileSystemParameters>`，列表最后一个元素为主文件系统 | 高 |
| `IRemoteServices` | 提供主 URL / Fallback URL 映射，注入到文件系统 | 高 |
| `IDecryptionServices` | 运行时 Bundle 解密，含同步/异步/Fallback 三种加载方式 | 高 |
| `IEncryptionServices` | 构建时对 Bundle 文件加密，返回 `EncryptResult` | 高 |
| `IManifestProcessServices` / `IManifestRestoreServices` | 对称的清单压缩/加密与还原接口，构建时处理、运行时还原 | 中 |
| `ICopyLocalFileServices` | 内置文件拷贝与沙盒文件导入的自定义拷贝逻辑 | 中 |
| `IWebDecryptionServices` | WebGL 平台专用 Bundle 解密，输入字节数组、输出 `AssetBundle` 对象 | 中 |
| `IPackRule` / `PackRuleResult` | 决定资源打入哪个 Bundle 及后缀名 | 高 |
| `IAddressRule` / `AddressRuleData` | 决定资源的寻址地址字符串 | 高 |
| `IFilterRule` | 控制收集器收集哪些资源类型 | 高 |
| `IBuildPipeline` | 自定义构建流水线的顶层接口 | 中 |
| `IBuildTask` | 构建流水线中的单个步骤接口 | 中 |
| `GameAsyncOperation` | 用户自定义异步操作的基类，实现 `OnStart`/`OnUpdate`/`OnAbort`，通过 `YooAssets.StartOperation` 注入 | 中 |
| `BundleResult` | 自定义文件系统的 Bundle 加载结果基类 | 中 |
| `WechatFileSystem` | 微信小游戏自定义文件系统完整实现，最完整的参考案例 | 高 |
| `DisplayName` Attribute | 用于在 Editor 下拉框中显示自定义规则的可读名称 | 中 |

## 源码文件清单

| 文件路径 | 作用 | 引用方式 |
|---|---|---|
| `Runtime/FileSystem/Interface/IFileSystem.cs` | 自定义文件系统必须实现的接口 | 完整解读 |
| `Runtime/FileSystem/FileSystemParameters.cs` | 文件系统参数封装类，含 5 个内置工厂方法及反射逻辑 | 完整解读 |
| `Runtime/InitializeParameters.cs` | `CustomPlayModeParameters` 定义 | 片段引用 |
| `Runtime/Services/IRemoteServices.cs` | CDN URL 查询接口 | 片段引用 |
| `Runtime/Services/IDecryptionServices.cs` | 运行时解密接口，含 `DecryptFileInfo` / `DecryptResult` 结构 | 片段引用 |
| `Runtime/Services/IEncryptionServices.cs` | 构建时加密接口，含 `EncryptFileInfo` / `EncryptResult` 结构 | 片段引用 |
| `Runtime/Services/IManifestProcessServices.cs` | 清单处理接口（构建端） | 片段引用 |
| `Runtime/Services/IManifestRestoreServices.cs` | 清单还原接口（运行端） | 片段引用 |
| `Runtime/OperationSystem/GameAsyncOperation.cs` | 用户自定义异步操作基类 | 完整解读 |
| `Runtime/FileSystem/BundleResult/BundleResult.cs` | Bundle 加载结果抽象基类 | 片段引用 |
| `Editor/AssetBundleCollector/CollectRules/IPackRule.cs` | 打包规则接口，含 PackRuleData/PackRuleResult | 片段引用 |
| `Editor/AssetBundleCollector/CollectRules/IAddressRule.cs` | 寻址规则接口 | 片段引用 |
| `Editor/AssetBundleCollector/DefaultRules/DefaultPackRule.cs` | 7 种内置打包规则参考实现 | 片段引用 |
| `Editor/AssetBundleBuilder/IBuildPipeline.cs` | 构建流水线顶层接口 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildSystem/IBuildTask.cs` | 单个构建步骤接口 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildSystem/BuildContext.cs` | 构建上下文容器 | 片段引用 |
| `Editor/AssetBundleBuilder/VisualViewers/BuildPipelineAttribute.cs` | 注册自定义流水线到 Editor 窗口的 Attribute | 片段引用 |
| `EditorExtension/CustomCollectRules/CustomAdressRule.cs` | 自定义寻址规则示例 | 完整解读 |
| `EditorExtension/CustomCollectRules/CustomPackRule.cs` | 自定义打包规则示例 | 完整解读 |
| `RuntimeExtension/ExtensionOperation/LoadGameObjectOperation.cs` | `GameAsyncOperation` 的完整使用示例 | 完整解读 |
| `Mini Game/Runtime/WechatFileSystem/WechatFileSystem.cs` | 微信小游戏自定义文件系统完整实现 | 完整解读 |
| `Mini Game/Runtime/TiktokFileSystem/TiktokFileSystem.cs` | 抖音小游戏自定义文件系统参考 | 片段引用 |
| `Mini Game/Runtime/GooglePlayFileSystem/GooglePlayFileSystem.cs` | Google Play Asset Delivery 文件系统参考 | 片段引用 |

## 验收标准

- [ ] 读者能描述 `IFileSystem` 的 15 个方法职责，并说明哪些方法返回抽象 Operation 子类
- [ ] 读者能写出注册自定义文件系统的最小代码：构造 FileSystemParameters、注入到 CustomPlayModeParameters、完成 InitializeAsync
- [ ] 读者能区分构建时加密（IEncryptionServices + BuildParameters）与运行时解密（IDecryptionServices + 文件系统参数）的注入位置
- [ ] 读者能说明 `IManifestProcessServices` 和 `IManifestRestoreServices` 为何成对设计，并写出透传的最小实现
- [ ] 读者能在 Editor 的收集器下拉框中注册自定义 `IPackRule`
- [ ] 读者能向 `BuiltinBuildPipeline` 的默认任务列表中插入一个自定义步骤，并用 `BuildContext.GetContextObject<T>` 读取前序步骤数据
- [ ] 读者能以 `WechatFileSystem` 为模板，说明小游戏平台自定义文件系统的关键适配点
- [ ] 读者能继承 `GameAsyncOperation` 实现一个自定义异步加载操作
- [ ] 章节中每个扩展点均有"接口定义 → 默认实现（或 Mini Game 实现）→ 注入方式"的完整链路说明
