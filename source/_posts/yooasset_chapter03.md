---
title: "【教材】第03章：文件系统抽象"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# Chapter 03: 文件系统抽象 — Spec

## 章节目标

深入剖析 `IFileSystem` 接口的完整契约定义，并横向对比六种内置实现（Editor/Buildin/Unpack/Cache/WebServer/WebRemote）的职责边界与差异点，使读者能够为任意部署场景选择正确的文件系统组合，并掌握自定义扩展的切入方式。

## 内容边界

### 包含
- `IFileSystem` 接口的三大方法族（生命周期、查询谓词、数据读取）的逐成员分析
- `FileSystemParameters` 的工厂方法模式与反射实例化机制
- `FileSystemParametersDefine` 所有参数键常量的语义分组
- 六种文件系统的类结构对比
- `DefaultUnpackFileSystem` 继承 `DefaultCacheFileSystem` 的设计意图
- `Buildin` 在 Android/OpenHarmony 平台的特殊路径（自动委托 `_unpackFileSystem`）
- `BundleResult` 抽象类及三个具体子类（AssetBundleResult / RawBundleResult / VirtualBundleResult）的职责
- `Belong` / `Exists` / `NeedDownload` / `NeedUnpack` / `NeedImport` 五个查询谓词在各实现中的返回值对比

### 不包含
- Operation 操作类的内部实现细节
- 下载中心 `DownloadCenterOperation` 的并发调度逻辑（属于第 4 章）
- 清单文件（`PackageManifest`）的序列化格式
- 解密服务 `IDecryptionServices` 的实现
- Editor 构建流水线相关代码

## 核心概念清单

| 概念 | 说明 | 重要程度 |
|------|------|----------|
| `IFileSystem` | 内部接口，定义文件系统完整契约（3 属性 + 6 异步操作 + 2 生命周期 + 5 查询谓词 + 3 读取方法） | 高 |
| `FileSystemParameters` | 公开配置载体，持有 `FileSystemClass` 和参数字典，通过 `CreateFileSystem()` 完成反射实例化 | 高 |
| `FileSystemParametersDefine` | 所有参数键的字符串常量集中定义，避免魔法字符串散落各处 | 中 |
| `DefaultEditorFileSystem` | 仅在编辑器中运行，直接读取工程内原始资产路径，支持帧数随机模拟 | 高 |
| `DefaultBuildinFileSystem` | 读取随包内置资源（StreamingAssets），通过 Catalog 二进制文件记录 FileWrapper | 高 |
| `DefaultUnpackFileSystem` | 继承 `DefaultCacheFileSystem`，仅重写三个目录路径常量 | 中 |
| `DefaultCacheFileSystem` | 网络下载的核心落盘文件系统，维护 `_records` 字典，持有 `DownloadCenter` 生命周期 | 高 |
| `DefaultWebServerFileSystem` | WebGL 本地 Web 服务器场景，通过 IWebDecryptionServices 解密，无磁盘缓存 | 中 |
| `DefaultWebRemoteFileSystem` | WebGL 跨域远程场景，`FileRoot` 固定返回空字符串 | 中 |
| `Belong()` 谓词 | 声明某 Bundle 是否归属此文件系统；Cache 实现固定返回 `true`（保底加载） | 高 |
| `NeedDownload/NeedUnpack/NeedImport` | 三个互斥语义的状态查询谓词，驱动资源包准备流程的路径选择 | 高 |
| `BundleResult` | 抽象基类，封装已加载资产包的生命周期与内容访问 | 高 |
| `AssetBundleResult` | 持有 `AssetBundle` 和可选 `Stream`（加密解密时的托管流），卸载时同步关闭流 | 高 |
| `VirtualBundleResult` | 编辑器虚拟包，`UnloadBundleFile()` 为空实现，读取回调全部转发至 `IFileSystem` | 中 |
| `RawBundleResult` | 原始文件包，不含 `LoadAsset` 语义 | 中 |

## 源码文件清单

| 文件路径 | 作用 | 引用方式 |
|----------|------|----------|
| `Runtime/FileSystem/Interface/IFileSystem.cs` | 接口完整定义，全部成员契约的唯一权威来源 | 完整解读 |
| `Runtime/FileSystem/FileSystemParameters.cs` | 工厂入口，反射实例化 + 参数注入逻辑；含六个静态工厂方法 | 完整解读 |
| `Runtime/FileSystem/FileSystemParametersDefine.cs` | 参数键常量表 | 片段引用 |
| `Runtime/FileSystem/DefaultEditorFileSystem/DefaultEditorFileSystem.cs` | 编辑器模拟文件系统，最简实现 | 完整解读 |
| `Runtime/FileSystem/DefaultBuildinFileSystem/DefaultBuildinFileSystem.cs` | 内置文件系统，含 Catalog/Unpack/Decrypt 三路分支 | 完整解读 |
| `Runtime/FileSystem/DefaultUnpackFileSystem/DefaultUnpackFileSystem.cs` | 解压文件系统，继承 Cache，仅覆盖目录路径 | 片段引用 |
| `Runtime/FileSystem/DefaultCacheFileSystem/DefaultCacheFileSystem.cs` | 缓存文件系统，最完整的下载/存储/校验实现 | 完整解读 |
| `Runtime/FileSystem/DefaultWebServerFileSystem/DefaultWebServerFileSystem.cs` | WebGL 本地服务器文件系统 | 片段引用 |
| `Runtime/FileSystem/DefaultWebRemoteFileSystem/DefaultWebRemoteFileSystem.cs` | WebGL 跨域远程文件系统 | 片段引用 |
| `Runtime/FileSystem/BundleResult/BundleResult.cs` | 抽象基类，定义已加载包的统一访问契约 | 完整解读 |
| `Runtime/FileSystem/BundleResult/AssetBundleResult/AssetBundleResult.cs` | AssetBundle 具体实现，处理托管流生命周期 | 完整解读 |
| `Runtime/FileSystem/BundleResult/VirtualBundleResult/VirtualBundleResult.cs` | 虚拟包实现，编辑器模式下的空壳 | 片段引用 |

## 验收标准

- [ ] 能够不查代码描述 `IFileSystem` 接口的主要方法族（生命周期/查询谓词/数据读取）
- [ ] 能够解释 `FileSystemParameters.CreateFileSystem()` 为何使用反射而非 `new`
- [ ] 能够以表格形式回答六种文件系统在 `Belong/NeedDownload/NeedUnpack` 三个谓词上的返回值规律
- [ ] 能够说明 `DefaultBuildinFileSystem` 在 Android 平台的特殊处理
- [ ] 能够描述 `DefaultUnpackFileSystem` 与 `DefaultCacheFileSystem` 的关系
- [ ] 能够说明 `BundleResult` 三个子类各自负责哪种 Bundle 类型
- [ ] 能够为"仅包内置资源"和"热更新 CDN"场景各自给出正确的 `FileSystemParameters` 工厂方法调用示例
