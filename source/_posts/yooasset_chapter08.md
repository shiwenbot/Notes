---
title: "【教材】第08章：Editor 构建管线"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# Chapter 08: Editor 构建管线 — Spec

## 章节目标

深入解析 YooAsset 的 Editor 构建管线架构，揭示 BuildSystem（任务调度骨架）与四条具体 Pipeline 的设计逻辑。帮助读者理解：IBuildTask / BuildContext / BuildRunner 三位一体的控制反转模式、四条 Pipeline 各自的任务序列及差异化点、`TaskGetBuildMap` 如何将收集体系的输出转化为构建输入、以及构建产物的生成流程与文件布局。

## 内容边界

### 包含
- `BuildSystem` 层四个核心类型：`IBuildTask`、`IContextObject`/`BuildContext`、`BuildRunner`、`BuildResult`
- `IBuildPipeline` 接口及 `AssetBundleBuilder.Run()` 的调度入口
- 四条 Pipeline 的完整任务列表及每条 Pipeline 的特有 Task 说明
- `BuildParameters` 抽象基类与四条 Pipeline 各自的 Parameters 子类
- 所有 BaseTasks 的职责与继承关系（TaskGetBuildMap / TaskUpdateBundleInfo / TaskEncryption / TaskCreateManifest / TaskCreateReport / TaskCopyBuildinFiles / TaskCreateCatalog）
- `BuildMapContext` 与 `BuildParametersContext` 作为 Context 对象的数据流转模型
- `BuildAssetInfo` / `BuildBundleInfo` 的构建期数据结构
- 产物文件布局：Pipeline 输出目录、Package 输出目录、StreamingAssets 内置目录
- `EBuildPipeline` 枚举与 `BuildPipelineAttribute` 的反射注册机制
- 加密扩展点 `IEncryptionServices` 与清单扩展点 `IManifestProcessServices` / `IManifestRestoreServices` 的接入方式
- `AssetBundleSimulateBuilder.SimulateBuild()` 在编辑器模拟模式下的调用路径
- `ErrorCode` 错误码体系的分层设计（按 Task 分段）

### 不包含
- 运行时资源加载流程（ResourcePackage、文件系统、下载系统）
- 第 7 章收集体系的内部实现（仅说明与 `TaskGetBuildMap` 的接口边界）
- AssetBundleReporterWindow、AssetArtScanner 等独立工具窗口
- CI/CD 集成与命令行构建脚本的具体写法

## 核心概念清单

| 概念 | 说明 | 重要程度 |
|---|---|---|
| `IBuildTask` | 单方法接口 `void Run(BuildContext)`，是整个管线的原子单元 | 高 |
| `BuildContext` | 以 `Type` 为 Key 的字典容器，任务间通过 `SetContextObject` / `GetContextObject<T>` 传递数据 | 高 |
| `BuildRunner.Run()` | 线性遍历 `List<IBuildTask>` 并用 `Stopwatch` 计时；任一 Task 抛异常即短路，返回 `BuildResult` | 高 |
| `IBuildPipeline` | 单方法接口 `BuildResult Run(BuildParameters, bool)`，四条管线的统一入口 | 高 |
| `BuildParameters`（抽象） | 所有构建参数的基类，含输出目录、包名/版本、加密服务、清单服务等 | 高 |
| `TaskGetBuildMap`（基类） | 调用 `AssetBundleCollectorSettingData.BeginCollect()` 拿到收集结果，生成 `BuildMapContext` | 高 |
| `BuildMapContext` | 核心数据结构：以 bundleName 为 Key 的 `BuildBundleInfo` 字典 | 高 |
| `BuildAssetInfo` | 单个资源的构建期描述：CollectorType、BundleName、Address、依赖链 | 中 |
| `BuildBundleInfo` | 单个 Bundle 的构建期描述：打包资源列表 + Hash/CRC/Size/路径（由 `TaskUpdateBundleInfo` 后填充） | 中 |
| `BuiltinBuildPipeline`（BBP） | 使用 Unity 传统 `BuildPipeline.BuildAssetBundles` API；11 步任务序列 | 高 |
| `ScriptableBuildPipeline`（SBP） | 使用 SBP 包 `ContentPipeline.BuildAssetBundles`；支持 Cache Server、link.xml | 高 |
| `RawFileBuildPipeline`（RFBP） | 无 Unity 引擎打包，直接 `CopyFile` 原始文件到输出目录；10 步任务序列 | 高 |
| `EditorSimulateBuildPipeline`（ESBP） | 仅 4 步（Prepare/GetBuildMap/UpdateBundleInfo/CreateManifest）；Hash 为全零 | 高 |
| `TaskUpdateBundleInfo`（抽象） | 4 步流程：校验文件名长度 → 写路径 → 计算 Hash/CRC/Size → 计算 PackageDestFilePath | 高 |
| `TaskCreateManifest`（抽象） | 生成 `PackageManifest`，写出 JSON/Binary/.hash/.version 共 4 个文件 | 高 |
| `TaskEncryption` | 遍历 BuildMapContext，对每个 Bundle 调用 `IEncryptionServices.Encrypt()` | 高 |
| `TaskCreateReport` | 生成 JSON 格式 `BuildReport`，包含 Summary/AssetInfos/BundleInfos/IndependAssets | 中 |
| `TaskCopyBuildinFiles` | 按 `EBuildinFileCopyOption` 策略将清单文件及 Bundle 拷贝到 StreamingAssets 内置目录 | 高 |
| `TaskVerifyBuildResult`（BBP/SBP 专属） | 用 `Except` 校验 Unity 实际构建产物与 BuildMapContext 规划内容是否一致 | 中 |
| 共享资源打包（SharePackRule） | `TaskGetBuildMap` 中，对无 BundleName 的依赖资源按目录路径生成共享包名 | 高 |
| `ErrorCode` | 分段枚举：100=Prepare，200=GetBuildMap，300=Building，400=UpdateBundleInfo，600=CreateManifest | 中 |

## 源码文件清单

| 文件路径 | 作用 | 引用方式 |
|---|---|---|
| `Editor/AssetBundleBuilder/BuildSystem/IBuildTask.cs` | 管线原子接口定义 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildSystem/BuildContext.cs` | Context 容器实现 | 完整解读 |
| `Editor/AssetBundleBuilder/BuildSystem/BuildRunner.cs` | 管线驱动器：线性执行 + 计时 + 异常熔断 | 完整解读 |
| `Editor/AssetBundleBuilder/IBuildPipeline.cs` | 四条管线的统一入口接口 | 片段引用 |
| `Editor/AssetBundleBuilder/AssetBundleBuilder.cs` | BuildParameters 注入 + BuildRunner 委托 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildParameters.cs` | 构建参数抽象基类 | 完整解读 |
| `Editor/AssetBundleBuilder/BuildMapContext.cs` | BuildMap 的 Context 数据结构 | 完整解读 |
| `Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskGetBuildMap.cs` | 核心衔接 Task：调用 Collector → 构建依赖图 → 生成 BuildMapContext | 完整解读 |
| `Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskUpdateBundleInfo.cs` | 抽象基类：填充 Bundle 产物信息 | 完整解读 |
| `Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskEncryption.cs` | 加密 Bundle 文件 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskCreateManifest.cs` | 生成 4 种产物文件 | 完整解读 |
| `Editor/AssetBundleBuilder/BuildPipeline/BaseTasks/TaskCopyBuildinFiles.cs` | 按策略拷贝至 StreamingAssets | 片段引用 |
| `Editor/AssetBundleBuilder/BuildPipeline/BuiltinBuildPipeline/BuiltinBuildPipeline.cs` | BBP 入口：11 步任务列表 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildPipeline/ScriptableBuildPipeline/ScriptableBuildPipeline.cs` | SBP 入口：11 步任务列表 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildPipeline/RawFileBuildPipeline/RawFileBuildPipeline.cs` | RFBP 入口：10 步任务列表 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/EditorSimulateBuildPipeline.cs` | ESBP 入口：4 步任务列表 | 片段引用 |
| `Editor/AssetBundleBuilder/BuildPipeline/BuiltinBuildPipeline/BuildTasks/TaskBuilding_BBP.cs` | 调用 `BuildPipeline.BuildAssetBundles` | 片段引用 |
| `Editor/AssetBundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskUpdateBundleInfo_ESBP.cs` | Hash=全零，Size=磁盘文件大小累加 | 片段引用 |
| `Editor/AssetBundleBuilder/AssetBundleSimulateBuilder.cs` | 运行时模拟构建的静态入口 | 片段引用 |

## 验收标准

- [ ] 读者能画出 `AssetBundleBuilder.Run()` → `BuildRunner.Run()` → `IBuildTask.Run(BuildContext)` 的调用链
- [ ] 读者能列出四条管线各自任务序列，并指出"打包"这一步的核心差异
- [ ] 读者能解释 `TaskGetBuildMap` 中共享资源打包逻辑的意义
- [ ] 读者能描述 `TaskUpdateBundleInfo` 的抽象设计：为何 BBP/SBP 从 Unity 内部 Manifest 读 Hash/CRC，而 ESBP 使用路径 MD5
- [ ] 读者能说明 `TaskCreateManifest` 生成的 4 种产物文件及其在运行时的用途
- [ ] 读者能解释 ESBP 的 `processBundleDepends=false` 意味着什么
- [ ] 读者能描述 `IEncryptionServices` 的接入方式及加密文件的产物路径
- [ ] 读者能说明 SBP 相比 BBP 多出的内置 Bundle 依赖修复逻辑
- [ ] 读者能说明 `EBuildinFileCopyOption` 的枚举值对应的 StreamingAssets 拷贝策略
