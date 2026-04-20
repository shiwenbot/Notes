---
title: "【教材】第06章：资源加载核心"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# Chapter 06: 资源加载核心 — Spec

## 章节目标

帮助读者深入理解 YooAsset 资源加载全链路的设计与实现：从业务代码调用 `ResourcePackage.LoadAssetAsync()` 到 `AssetHandle` 返回给调用方，中间经历 `ResourceManager` 的调度、`ProviderOperation` 的创建与复用、`LoadBundleFileOperation` 的主 Bundle + 依赖 Bundle 加载，以及 Handle 持有期间引用计数的完整生命周期。

## 内容边界

### 包含
- `ResourceManager.cs` 的整体职责：Provider 字典管理、加载入口分发、全局卸载调度
- `ProviderOperation`（抽象基类）的状态机：状态枚举、RefCount 递增/递减逻辑、Provider 挂起机制（v2.3.16+）
- 五种 Provider 实现的差异对比：`AssetProvider` / `SubAssetsProvider` / `AllAssetsProvider` / `SceneProvider` / `RawFileProvider`
- `CompletedProvider`：已完成提前返回的短路优化
- `LoadBundleFileOperation`：主 Bundle 与依赖 Bundle 的并行加载协调
- `HandleBase`：`IEnumerator` + `IDisposable` 双接口设计、`Release()`/`Dispose()` 等价性
- `AssetHandle`、`SceneHandle`、`SubAssetsHandle`、`AllAssetsHandle`、`RawFileHandle` 各 Handle 的特有 API
- `InstantiateOperation`：实例化操作对 `AssetHandle` 的依托
- 卸载操作三件套：`UnloadSceneOperation`、`UnloadAllAssetsOperation`、`UnloadUnusedAssetsOperation`
- 引用计数（RefCount）的全生命周期

### 不包含
- FileSystem 内部如何加载 AssetBundle 文件（第 3 章）
- OperationSystem 调度引擎（第 2 章）
- ResourcePackage 的初始化与清单更新流程（第 5 章）
- 下载系统与断点续传（第 4 章）
- Editor 侧的构建流程（第 7、8 章）
- `YOOASSET_EXPERIMENTAL` 宏控制的 `UseWeakReferenceHandle` 实验性功能

## 核心概念清单

| 概念 | 说明 | 重要程度 |
|------|------|----------|
| `ResourceManager` | 持有全局 Provider 字典，作为加载请求的调度中枢，负责 Provider 的创建、复用与销毁 | 高 |
| `ProviderOperation` | 抽象基类，继承 `AsyncOperationBase`，代表一次资源加载任务；维护 RefCount 与关联 Handle 列表 | 高 |
| `RefCount`（引用计数） | ProviderOperation 内部计数器；每创建 Handle +1，Handle Release -1；归零时触发 Provider 挂起或回收 | 高 |
| Provider 复用 | 同一 AssetPath 的加载请求共享同一 ProviderOperation 实例，避免重复加载 Bundle | 高 |
| Provider 挂起（v2.3.16+） | 引用计数为零时不立即销毁，而是挂起等待复用，减少频繁创建/销毁开销 | 中 |
| `AssetProvider` | 加载单个 `UnityEngine.Object`，调用 `AssetBundle.LoadAssetAsync` | 高 |
| `SubAssetsProvider` | 加载资源的所有子资产（如 Sprite Atlas 内的多个 Sprite） | 中 |
| `AllAssetsProvider` | 加载 Bundle 内所有资产，调用 `AssetBundle.LoadAllAssetsAsync` | 中 |
| `SceneProvider` | 加载场景，调用 `SceneManager.LoadSceneAsync`；支持主场景与附加场景 | 高 |
| `RawFileProvider` | 加载原始二进制/文本文件，绕过 AssetBundle，直接读取文件系统 | 中 |
| `CompletedProvider` | 预先完成状态的 Provider，用于同步加载或编辑器模拟模式的短路优化 | 中 |
| `LoadBundleFileOperation` | 协调主 Bundle 与所有依赖 Bundle 的加载，通过 `IFileSystem` 委托实际 IO | 高 |
| `HandleBase` | 所有 Handle 的抽象基类，实现 `IEnumerator`（协程等待）和 `IDisposable`（using 释放） | 高 |
| `AssetHandle` | 单资产句柄；暴露 `AssetObject`、`GetAssetObject<T>()`、`InstantiateAsync()` | 高 |
| `SceneHandle` | 场景句柄；暴露 `SceneObject`；卸载时调用 `UnloadAsync()` | 高 |
| `SubAssetsHandle` | 子资产句柄；暴露 `GetSubAssetObject<T>(name)` | 中 |
| `AllAssetsHandle` | 全资产句柄；暴露 `AllAssetObjects` 数组 | 中 |
| `RawFileHandle` | 原始文件句柄；暴露 `GetRawFileText()` / `GetRawFileData()` / `GetRawFilePath()` | 中 |
| `InstantiateOperation` | 基于 `AssetHandle` 的实例化异步操作，返回 `GameObject` | 中 |

## 源码文件清单

| 文件路径 | 作用 | 引用方式 |
|----------|------|----------|
| `Runtime/ResourceManager/ResourceManager.cs` | 调度中枢：Provider 字典、加载入口、卸载调度 | 完整解读 |
| `Runtime/ResourceManager/Provider/ProviderOperation.cs` | Provider 抽象基类：状态机、RefCount、挂起逻辑 | 完整解读 |
| `Runtime/ResourceManager/Provider/AssetProvider.cs` | 单资产加载 | 片段引用 |
| `Runtime/ResourceManager/Provider/SceneProvider.cs` | 场景加载：`SceneManager.LoadSceneAsync` 调用 | 完整解读 |
| `Runtime/ResourceManager/Provider/RawFileProvider.cs` | 原始文件加载 | 片段引用 |
| `Runtime/ResourceManager/Provider/CompletedProvider.cs` | 短路优化：直接构造已完成状态的 Provider | 片段引用 |
| `Runtime/ResourceManager/Operation/Internal/LoadBundleFileOperation.cs` | 主 Bundle + 依赖 Bundle 加载协调器 | 完整解读 |
| `Runtime/ResourceManager/Handle/HandleBase.cs` | Handle 基类：双接口实现、`IsValid`/`IsDone`/`Progress` | 完整解读 |
| `Runtime/ResourceManager/Handle/AssetHandle.cs` | 单资产句柄 | 完整解读 |
| `Runtime/ResourceManager/Handle/SceneHandle.cs` | 场景句柄 | 完整解读 |
| `Runtime/ResourceManager/Handle/SubAssetsHandle.cs` | 子资产句柄 | 片段引用 |
| `Runtime/ResourceManager/Handle/AllAssetsHandle.cs` | 全资产句柄 | 片段引用 |
| `Runtime/ResourceManager/Handle/RawFileHandle.cs` | 原始文件句柄 | 片段引用 |
| `Runtime/ResourceManager/Operation/InstantiateOperation.cs` | 实例化操作 | 片段引用 |
| `Runtime/ResourceManager/Operation/UnloadSceneOperation.cs` | 场景卸载操作 | 片段引用 |

## 验收标准

- [ ] 读者能描述从 `ResourcePackage.LoadAssetAsync("path")` 到 `AssetHandle` 返回的完整调用链
- [ ] 章节须包含 RefCount 完整生命周期图示（Handle 创建 → Release → 归零 → 挂起/回收）
- [ ] 章节须提供五种 Provider 类型的对比表格（类型/加载 API/典型场景/返回 Handle 类型）
- [ ] 章节须演示 Handle 三种消费模式的代码示例：`await handle` / `yield return handle` / `handle.Completed += callback`
- [ ] 章节须解释 `Release()` 与 `Dispose()` 在 v2.3.10+ 的等价性
- [ ] 章节须说明 `CompletedProvider` 的触发条件及其短路效果
- [ ] 章节须交代 v2.3.16 引入的"引用计数为零时 Provider 自动挂起"机制
- [ ] 对 `SceneProvider` 须单独说明主场景与附加场景的卸载差异
