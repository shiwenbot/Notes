---
title: "【教材】第04章：下载系统"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# Chapter 04: 下载系统 — Spec

## 章节目标

深度解析 YooAsset 下载系统的分层架构：从底层 `UnityWebRequest` 操作封装，到断点续传 TempFile 机制，到 `WebRequestCounter` 重试计数，再到 `DownloadCenterOperation` 调度协同，帮助读者理解"一次文件下载"从发起到落盘的完整路径，并掌握并发控制与 WebGL 平台特殊策略的设计意图。

## 内容边界

### 包含
- `DownloadSystem/` 目录的完整结构：`DownloadDefine`、`DownloadSystemHelper`、`WebRequestCounter` 的职责
- `UnityWebRequest*Operation` 六种操作类型（File/Data/Text/AssetBundle/VirtualBundle/WebCache）的类型体系与分工
- 三种具体下载实现：`UnityDownloadNormalFileOperation`（普通下载）、`UnityDownloadResumeFileOperation`（断点续传）、`UnityDownloadLocalFileOperation`（本地拷贝）
- 断点续传机制：`TempFileElement`、`VerifyTempFileOperation`、`RecordFileElement` 的协作关系
- `DownloadCenterOperation` 与 `DownloadPackageBundleOperation` 的职责划分
- `WebRequestCounter` 的重试计数逻辑与失败策略
- 下载相关参数：`DOWNLOAD_MAX_CONCURRENCY`、`DOWNLOAD_MAX_REQUEST_PER_FRAME`、`DOWNLOAD_WATCH_DOG_TIME`
- WebGL 平台特殊策略：`DefaultWebServerFileSystem` 与 `DefaultWebRemoteFileSystem` 的差异
- `IRemoteServices` 在下载系统中的角色（URL 构建入口）
- WatchDog 超时机制替代旧 timeout 的设计演进

### 不包含
- `ResourceDownloaderOperation`（Facade 层下载器，属于第 5/6 章）
- 文件系统接口 `IFileSystem` 的整体协议（第 3 章已覆盖）
- `IDecryptionServices` / `IWebDecryptionServices` 解密逻辑（属于文件系统层）
- `DefaultBuildinFileSystem`、`DefaultEditorFileSystem` 的下载行为（非网络下载）
- Editor 构建系统（第 8 章）

## 核心概念清单

| 概念 | 说明 | 重要程度 |
|------|------|----------|
| `UnityWebRequestOperation`（基类） | 所有网络请求操作的基类，封装 `UnityWebRequest` 生命周期管理、超时检测与错误处理 | 高 |
| `UnityWebFileRequestOperation` | 将远端文件下载到本地磁盘；断点续传策略的直接承载者 | 高 |
| `UnityWebDataRequestOperation` | 下载二进制数据到内存，用于清单文件等小数据 | 中 |
| `UnityWebTextRequestOperation` | 下载文本内容，用于版本号、Hash 文件 | 中 |
| `UnityWebAssetBundleRequestOperation` | 使用 `DownloadHandlerAssetBundle` 直接将 AB 加载进内存，绕过磁盘写入 | 中 |
| `UnityWebCacheRequestOperation` | WebGL 平台专用，利用浏览器缓存机制请求 AB | 中 |
| `WebRequestCounter` | 单次下载任务的请求次数计数器，控制最大重试次数 | 高 |
| `DownloadSystemHelper` | 静态工具类，提供 URL 拼接、响应码判断、临时文件路径生成等帮助方法 | 中 |
| `TempFileElement` | 断点续传核心数据结构，代表一个下载中的临时文件（`.tmp`），记录已下载字节数 | 高 |
| `RecordFileElement` | 已完成下载并通过校验的缓存文件记录（Hash、CRC、Size），驱动"是否需要重新下载"判断 | 高 |
| `UnityDownloadNormalFileOperation` | 普通全量下载策略：从零开始下载，写入临时文件后原子移动到缓存目录 | 高 |
| `UnityDownloadResumeFileOperation` | 断点续传策略：检测本地 `.tmp` 已下载字节数，发送 HTTP `Range` 请求从中断点续传 | 高 |
| `UnityDownloadLocalFileOperation` | 本地拷贝策略（`ICopyLocalFileServices`），将本地已有文件直接拷贝到缓存目录 | 中 |
| `DownloadCenterOperation` | 下载中心调度器，管理并发槽位与每帧请求数，驱动队列消费 | 高 |
| `DownloadPackageBundleOperation` | 单个 Bundle 的下载编排 Operation，根据策略选择具体下载实现，持有重试状态 | 高 |
| `VerifyTempFileOperation` | 续传前对已有 `.tmp` 文件做完整性预校验，校验失败则删除 `.tmp` 降级为全量下载 | 中 |
| `DOWNLOAD_WATCH_DOG_TIME` | 看门狗：监控窗口内若无任何数据接收，则强制终止任务；替代旧版基于 socket 超时的参数 | 中 |
| HTTP 416 边界问题 | 当 `.tmp` 大小已等于目标大小时发送 `Range` 请求会被服务器返回 416，需在续传前做保护判断 | 中 |

## 源码文件清单

| 文件路径 | 作用 | 引用方式 |
|----------|------|----------|
| `Runtime/DownloadSystem/DownloadDefine.cs` | 下载系统常量、枚举定义 | 片段引用 |
| `Runtime/DownloadSystem/DownloadSystemHelper.cs` | URL 构建、临时文件路径等静态工具方法 | 片段引用 |
| `Runtime/DownloadSystem/WebRequestCounter.cs` | 重试计数器 | 片段引用 |
| `Runtime/DownloadSystem/Operation/Internal/UnityWebRequestOperation.cs` | 所有 WebRequest 操作的基类（超时、WatchDog、错误处理） | 完整解读 |
| `Runtime/DownloadSystem/Operation/Internal/UnityWebFileRequestOperation.cs` | 磁盘文件下载，`Range` 头写入，临时文件落盘 | 完整解读 |
| `Runtime/FileSystem/DefaultCacheFileSystem/Elements/TempFileElement.cs` | 临时文件数据结构 | 完整解读 |
| `Runtime/FileSystem/DefaultCacheFileSystem/Elements/RecordFileElement.cs` | 缓存文件记录 | 完整解读 |
| `Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/UnityDownloadNormalFileOperation.cs` | 普通全量下载策略 | 片段引用 |
| `Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/UnityDownloadResumeFileOperation.cs` | 断点续传策略 | 完整解读 |
| `Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/DownloadCenterOperation.cs` | 并发调度中心 | 完整解读 |
| `Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/DownloadPackageBundleOperation.cs` | 单 Bundle 下载编排，策略选择与重试驱动 | 完整解读 |
| `Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/VerifyTempFileOperation.cs` | 续传前 `.tmp` 文件完整性预校验 | 片段引用 |

## 验收标准

- [ ] 读者能说清六种 WebRequest 操作类各自适用的场景
- [ ] 读者能描述断点续传的完整路径：TempFileElement → VerifyTempFileOperation → UnityDownloadResumeFileOperation → 追加写入 → 原子移动
- [ ] 读者能解释 HTTP 416 边界问题的成因及保护措施
- [ ] 读者能说明 `WebRequestCounter` 如何与 `DownloadPackageBundleOperation` 配合实现重试上限控制
- [ ] 读者能区分两个并发参数（`DOWNLOAD_MAX_CONCURRENCY` vs `DOWNLOAD_MAX_REQUEST_PER_FRAME`）的不同控制维度
- [ ] 读者能说明 WatchDog 机制与旧版 `timeout` 参数的本质区别
- [ ] 读者能区分 WebGL 两种策略（同域缓存 vs 跨域内存流）的适用场景
- [ ] 读者能画出一次 Bundle 下载的操作链：DownloadCenterOperation → DownloadPackageBundleOperation → UnityDownload*FileOperation → UnityWebFileRequestOperation
