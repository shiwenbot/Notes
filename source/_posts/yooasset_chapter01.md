---
title: "【教材】第01章：整体架构概览"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# Chapter 01: 整体架构概览 — Spec

## 章节目标

帮助读者在深入各模块之前，建立对 YooAsset 系统全貌的清晰认知——理解三层职责划分（Facade / Package 业务层 / 基础设施层）、多 Package 并存的设计意义，以及各层之间的协作协议。

## 内容边界

### 包含
- `YooAssets` 静态 Facade 的职责：生命周期（Initialize/Destroy）、Package 注册表管理、全局参数配置
- `YooAssetsDriver` MonoBehaviour 驱动机制（DontDestroyOnLoad + Update 帧驱动）
- `ResourcePackage` 作为业务入口的角色：InitializeAsync、资源加载/版本更新/清单更新的顶层接口
- `ResourceManager` 作为包内资源管理核心的职责：ProviderDic / LoaderDic / 五类 Load 方法
- `OperationSystem` 的调度模型：帧时间片（MaxTimeSlice / IsBusy）、优先级排序、跨 Package 隔离
- 五种 PlayMode（EditorSimulate / Offline / Host / Web / Custom）的枚举意义与选择场景
- FileSystem 层（六类内置实现）与 Services 接口层（IRemoteServices / IDecryptionServices 等）的分层定位
- `package.json` 声明的版本与依赖（Unity 2019.4+，com.unity.scriptablebuildpipeline，AssetBundle/WebRequest 模块）

### 不包含
- 各 FileSystem 的内部实现细节（留给专章）
- AssetBundle 加载流程的完整链路（Provider / BundleLoader 状态机）
- Manifest 结构与版本差量算法
- 下载系统（DownloadSystem）的具体实现
- Editor 构建管线（EditorSimulateModeHelper / BuildPipeline）
- Luban / HybridCLR 等宿主框架与 YooAsset 的集成方式

## 核心概念清单

| 概念 | 说明 | 重要程度 |
|------|------|----------|
| `YooAssets`（Facade） | 全局静态入口，维护 `_packages` 列表，代理 OperationSystem 的初始化与帧更新 | 高 |
| `YooAssetsDriver` | 隐藏的 MonoBehaviour，挂载于 DontDestroyOnLoad GameObject，负责每帧调用 `YooAssets.Update()` | 高 |
| `ResourcePackage` | 资源包裹，代表一个独立的资源分组，持有 ResourceManager 与 PlayModeImpl，是业务代码的直接交互对象 | 高 |
| 多 Package 设计 | 每个 Package 独立版本、独立 Manifest、独立 FileSystem，支持主包+DLC+热更包并存 | 高 |
| `ResourceManager` | 包内资源管理核心（internal），维护 ProviderDic / LoaderDic，提供 LoadAssetAsync / LoadSceneAsync 等五类操作 | 高 |
| `OperationSystem` | 全局异步操作调度器（internal），基于帧时间片的协作式调度，支持优先级排序与跨 Package 隔离取消 | 高 |
| `EPlayMode` | 五种运行模式（EditorSimulate / Offline / Host / Web / Custom），决定 FileSystem 的初始化路径 | 高 |
| `AsyncOperationBase` / `GameAsyncOperation` | 所有异步操作的基类；`GameAsyncOperation` 是游戏业务逻辑可继承扩展的公共操作基类 | 中 |
| `IBundleQuery` | PlayModeImpl 向 ResourceManager 暴露的查询接口，解耦 Manifest 查询与资源加载 | 中 |
| Services 接口层 | `IRemoteServices` / `IDecryptionServices` / `IEncryptionServices` 等，允许业务代码注入自定义行为 | 中 |
| FileSystem 层 | 六类内置实现（Buildin / Cache / Editor / Unpack / WebServer / WebRemote），对应不同存储来源 | 低 |
| `MaxTimeSlice` | OperationSystem 的帧时间预算（毫秒），防止异步操作占满单帧主线程，默认 `long.MaxValue`（不限制） | 中 |

## 源码文件清单

| 文件路径 | 作用 | 引用方式 |
|----------|------|----------|
| `Runtime/YooAssets.cs` | 全局静态 Facade；Package 注册表管理；全局参数；OperationSystem 初始化入口 | 完整解读 |
| `Runtime/YooAssetsDriver.cs` | MonoBehaviour 驱动桥接，每帧调用 `YooAssets.Update()` | 片段引用 |
| `Runtime/ResourcePackage/ResourcePackage.cs` | 业务层核心类；PlayMode 分支初始化；资源加载 / 版本请求 / Manifest 更新的公开 API | 完整解读 |
| `Runtime/ResourceManager/ResourceManager.cs` | 包内资源管理器（internal）；五类 Load 方法；ProviderDic / LoaderDic 结构 | 片段引用 |
| `Runtime/OperationSystem/OperationSystem.cs` | 帧时间片调度器；优先级排序；跨 Package 操作隔离取消 | 完整解读 |
| `Runtime/ResourcePackage/PlayMode/PlayModeImpl.cs` | PlayMode 策略实现，将 EPlayMode 映射到具体 FileSystem 初始化参数 | 仅提及 |
| `Runtime/FileSystem/`（目录） | 六类内置 FileSystem 实现 | 仅提及 |
| `Runtime/Services/`（目录） | 可插拔 Services 接口定义 | 仅提及 |
| `Runtime/OperationSystem/AsyncOperationBase.cs` | 所有异步操作的基类 | 仅提及 |
| `Runtime/OperationSystem/GameAsyncOperation.cs` | 业务侧可继承的公共异步操作基类 | 仅提及 |
| `package.json` | 版本（2.3.17）、最低 Unity（2019.4）、依赖声明 | 片段引用 |

## 验收标准

- [ ] 读者能够用一张分层图描述 YooAsset 的模块划分（Facade → Package 业务层 → ResourceManager → OperationSystem / FileSystem）
- [ ] 读者能够解释为何需要多 Package 而非单一全局管理器，并举出至少一个实际场景
- [ ] 读者能够说明 `YooAssetsDriver` 存在的原因，以及没有它会发生什么
- [ ] 读者能够区分五种 PlayMode 并正确匹配其适用平台/阶段
- [ ] 读者能够解释 `OperationSystem.MaxTimeSlice` 的作用，并说明默认值 `long.MaxValue` 的设计含义
- [ ] 读者理解 `ResourceManager` 是 `internal` 的，业务代码只能通过 `ResourcePackage` 的公开方法间接使用它
- [ ] 读者能够指出 Services 接口层在架构中所处的位置，并说明其可扩展的边界
