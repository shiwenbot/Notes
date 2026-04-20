---
title: "【教材】第4章：下载系统"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 4 章：下载系统

## 概述

热更新的网络环境充满挑战。玩家可能在地铁中突然断网，在信号弱的环境中下载超时，或者下载的文件因为网络错误而损坏。YooAsset 的下载系统必须解决这些核心问题：如何在弱网环境下保证下载成功率？如何处理大文件的断点续传？如何控制并发下载避免阻塞游戏主线程？

更重要的是，下载系统不仅仅是"从服务器获取文件"那么简单。它需要处理文件校验（CRC）、并发控制（避免同时下载过多文件导致卡顿）、超时重试机制、以及针对不同平台（特别是 WebGL）的特殊策略。每一个环节都关系到玩家的下载体验和游戏的稳定性。

YooAsset 的下载系统采用了分层设计：底层基于 UnityWebRequest 封装了多种操作类型，中间层通过 DownloadCenter 实现并发调度，上层通过 DownloadPackageBundleOperation 提供重试和错误恢复机制。这种设计既保证了代码的可维护性，又提供了足够的灵活性来应对各种复杂的网络场景。

## UnityWebRequest 操作类型体系

YooAsset 将网络请求抽象为六个专门的操作类，每个类负责特定的下载场景。这种分工明确的架构让代码职责清晰，便于维护和扩展。

| 操作类 | 职责 | 使用场景 |
|--------|------|---------|
| `UnityWebRequestOperation` | 基类，提供状态机和错误检测 | 所有网络请求的底层抽象 |
| `UnityDownloadFileOperation` | 文件下载基类 | 继承并实现具体的文件下载逻辑 |
| `UnityDownloadNormalFileOperation` | 普通文件下载 | 小于断点续传阈值的文件 |
| `UnityDownloadResumeFileOperation` | 断点续传下载 | 大文件（可配置阈值，默认 1MB） |
| `UnityDownloadLocalFileOperation` | 本地文件复制 | 从本地文件系统（StreamingAssets）加载 |
| `UnityWebRequestListOperation` | 文件列表获取 | 获取远程服务器的资源清单 |

所有操作类都继承自 `UnityWebRequestOperation` 基类，该基类封装了 UnityWebRequest 的生命周期管理和错误检测逻辑。

```csharp:Runtime/DownloadSystem/Operation/Internal/UnityWebRequestOperation.cs [7:96]
internal abstract class UnityWebRequestOperation : AsyncOperationBase
{
    protected UnityWebRequest _webRequest;
    protected readonly string _requestURL;
    private bool _isAbort = false;

    /// <summary>
    /// HTTP返回码
    /// </summary>
    public long HttpCode { private set; get; }

    /// <summary>
    /// 当前下载的字节数
    /// </summary>
    public long DownloadedBytes { protected set; get; }

    /// <summary>
    /// 当前下载进度（0f - 1f）
    /// </summary>
    public float DownloadProgress { protected set; get; }

    /// <summary>
    /// 请求的URL地址
    /// </summary>
    public string URL
    {
        get { return _requestURL; }
    }

    internal UnityWebRequestOperation(string url)
    {
        _requestURL = url;
    }
    internal override void InternalAbort()
    {
        //TODO
        // 1. 编辑器下停止运行游戏的时候主动终止下载任务
        // 2. 真机上销毁包裹的时候主动终止下载任务
        if (_isAbort == false)
        {
            if (_webRequest != null)
            {
                _webRequest.Abort();
                _isAbort = true;
            }
        }
    }

    /// <summary>
    /// 释放下载器
    /// </summary>
    protected void DisposeRequest()
    {
        if (_webRequest != null)
        {
            //注意：引擎底层会自动调用Abort方法
            _webRequest.Dispose();
            _webRequest = null;
        }
    }

    /// <summary>
    /// 检测请求结果
    /// </summary>
    protected bool CheckRequestResult()
    {
        HttpCode = _webRequest.responseCode;

#if UNITY_2020_3_OR_NEWER
        if (_webRequest.result != UnityWebRequest.Result.Success)
        {
            Error = $"URL : {_requestURL} Error : {_webRequest.error}";
            return false;
        }
        else
        {
            return true;
        }
#else
        if (_webRequest.isNetworkError || _webRequest.isHttpError)
        {
            Error = $"URL : {_requestURL} Error : {_webRequest.error}";
            return false;
        }
        else
        {
            return true;
        }
#endif
    }
}
```

基类提供了三个核心属性：`HttpCode` 用于获取 HTTP 状态码，`DownloadedBytes` 记录已下载的字节数，`DownloadProgress` 提供 0 到 1 的下载进度。`InternalAbort` 方法处理下载任务的主动终止，而 `CheckRequestResult` 则统一检测网络请求的成功或失败。这种设计让子类只需关注业务逻辑，底层的网络管理由基类统一处理。

## 断点续传：TempFile 机制

断点续传是大文件下载的必备功能。想象一下，玩家下载 100MB 的资源包，下载到 80MB 时突然断网。如果没有断点续传，玩家需要重新下载整个文件；而有了断点续传，只需下载剩余的 20MB。这不仅节省流量，更重要的是提升玩家体验。

YooAsset 的断点续传通过 TempFile（临时文件）机制实现。下载时，系统会创建一个临时文件（带 `.tmp` 后缀），数据直接写入这个临时文件。下载完成后，系统会校验文件的完整性和正确性，校验通过后才将临时文件转为正式文件。

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/Elements/TempFileElement.cs [1:22]
internal class TempFileElement
{
    public string TempFilePath { private set; get; }
    public uint TempFileCRC { private set; get; }
    public long TempFileSize { private set; get; }

    /// <summary>
    /// 注意：原子操作对象
    /// </summary>
    public volatile int Result = 0;

    public TempFileElement(string filePath, uint fileCRC, long fileSize)
    {
        TempFilePath = filePath;
        TempFileCRC = fileCRC;
        TempFileSize = fileSize;
    }
}
```

`TempFileElement` 是临时文件的数据结构，存储了临时文件路径、目标 CRC 值和目标文件大小。`Result` 字段使用 `volatile` 修饰符，确保多线程环境下的可见性。这个字段会在文件校验线程中被写入，主线程中读取，用于同步校验结果。

断点续传的核心流程在 `UnityDownloadResumeFileOperation` 中实现。下载开始前，系统会先检查临时文件是否存在。如果存在且大小小于目标文件大小，则从上次中断的位置继续下载；如果临时文件已损坏或大小异常，则删除并重新下载。

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/UnityDownloadResumeFileOperation.cs [146:156]
private void CreateWebRequest(long fileBeginLength)
{
    var handler = new DownloadHandlerFile(_tempFilePath, true);
    handler.removeFileOnAbort = false;
    _webRequest = DownloadSystemHelper.NewUnityWebRequestGet(_requestURL);
    _webRequest.downloadHandler = handler;
    _webRequest.disposeDownloadHandlerOnDispose = true;
    if (fileBeginLength > 0)
        _webRequest.SetRequestHeader("Range", $"bytes={fileBeginLength}-");
    _webRequest.SendWebRequest();
}
```

关键在于 `Range` 请求头的设置。`Range: bytes=N-` 告诉服务器："我只需要从第 N 个字节开始的数据"。服务器收到这个请求后，会返回 HTTP 206（Partial Content）状态码，并在响应体中只包含从 N 开始的数据。`DownloadHandlerFile` 的第二个参数 `append` 设置为 `true`，确保下载的数据追加到临时文件末尾而不是覆盖。

断点续传有一个常见的边界问题：如果临时文件的大小已经等于或超过目标文件大小，但文件内容不完整或损坏怎么办？YooAsset 的做法是直接删除临时文件并重新下载。这是因为在 `CreateRequest` 阶段，系统会检查临时文件大小：

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/UnityDownloadResumeFileOperation.cs [34:47]
_fileOriginLength = 0;
long fileBeginLength = -1;
if (File.Exists(_tempFilePath))
{
    FileInfo fileInfo = new FileInfo(_tempFilePath);
    if (fileInfo.Length >= _bundle.FileSize)
    {
        File.Delete(_tempFilePath);
    }
    else
    {
        fileBeginLength = fileInfo.Length;
        _fileOriginLength = fileBeginLength;
        DownloadedBytes = _fileOriginLength;
    }
}
```

如果临时文件大小大于等于目标大小，说明文件可能已损坏（因为完整文件校验会在下载完成后进行），系统会删除它并从头开始下载。这种保守的策略确保了文件的完整性，避免使用损坏的临时文件。

下载完成后，系统会通过 `VerifyTempFileOperation` 对临时文件进行校验。校验在独立线程中进行，避免阻塞主线程：

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/VerifyTempFileOperation.cs [84:89]
private void VerifyInThread(object obj)
{
    TempFileElement element = (TempFileElement)obj;
    int result = (int)FileVerifyHelper.FileVerify(element.TempFilePath, element.TempFileSize, element.TempFileCRC, EFileVerifyLevel.High);
    element.Result = result;
}
```

校验包括文件大小检查和 CRC 校验。只有当两者都通过时，临时文件才会被转为正式的缓存文件。这种"先下载到临时文件，校验通过后再转正"的策略，确保了缓存目录中的文件永远是完整且正确的。

## DownloadCenter 调度器

DownloadCenter 是下载系统的调度核心，负责管理所有下载任务的并发执行。它的核心功能有两个：控制并发数量和复用下载任务。

并发控制通过两个参数实现：`DownloadMaxConcurrency`（最大并发数，默认 10）和 `DownloadMaxRequestPerFrame`（每帧最大请求数，默认 3）。前者限制了同时进行的下载任务总数，避免过多并发导致网络拥塞；后者限制了每帧新启动的下载任务数，避免在一帧内创建大量网络请求导致主线程卡顿。

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/DownloadCenterOperation.cs [52:74]
// 最大并发数检测
int processCount = GetProcessingOperationCount();
if (processCount != _downloaders.Count)
{
    if (processCount < _fileSystem.DownloadMaxConcurrency)
    {
        int startCount = _fileSystem.DownloadMaxConcurrency - processCount;
        if (startCount > _fileSystem.DownloadMaxRequestPerFrame)
            startCount = _fileSystem.DownloadMaxRequestPerFrame;

        foreach (var operationPair in _downloaders)
        {
            var operation = operationPair.Value;
            if (operation.Status == EOperationStatus.None)
            {
                operation.StartOperation();
                startCount--;
                if (startCount <= 0)
                    break;
            }
        }
    }
}
```

这段代码体现了双重控制逻辑：首先计算当前正在进行的任务数（`processCount`），如果小于最大并发数，则计算可启动的任务数（`startCount`）。但 `startCount` 还要受到每帧最大请求数的限制，取两者的最小值。这样既保证了总并发数不超限，又避免了单帧启动过多任务。

下载任务复用是 DownloadCenter 的另一个重要特性。当多个系统请求下载同一个文件时，DownloadCenter 会返回同一个下载任务，并通过引用计数管理任务的生命周期。

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/DownloadCenterOperation.cs [80:116]
public UnityDownloadFileOperation DownloadFileAsync(PackageBundle bundle, string url)
{
    // 查询旧的下载器
    if (_downloaders.TryGetValue(bundle.BundleGUID, out var oldDownloader))
    {
        oldDownloader.Reference();
        return oldDownloader;
    }

    // 创建新的下载器
    UnityDownloadFileOperation newDownloader;
    bool isRequestLocalFile = DownloadSystemHelper.IsRequestLocalFile(url);
    if (isRequestLocalFile)
    {
        newDownloader = new UnityDownloadLocalFileOperation(_fileSystem, bundle, url);
        AddChildOperation(newDownloader);
        _downloaders.Add(bundle.BundleGUID, newDownloader);
    }
    else
    {
        if (bundle.FileSize >= _fileSystem.ResumeDownloadMinimumSize)
        {
            newDownloader = new UnityDownloadResumeFileOperation(_fileSystem, bundle, url);
            AddChildOperation(newDownloader);
            _downloaders.Add(bundle.BundleGUID, newDownloader);
        }
        else
        {
            newDownloader = new UnityDownloadNormalFileOperation(_fileSystem, bundle, url);
            AddChildOperation(newDownloader);
            _downloaders.Add(bundle.BundleGUID, newDownloader);
        }
    }

    newDownloader.Reference();
    return newDownloader;
}
```

这段代码还展示了下载策略的三分支逻辑：本地文件使用 `UnityDownloadLocalFileOperation`，大文件使用断点续传，小文件使用普通下载。`ResumeDownloadMinimumSize` 默认为 1MB，这个阈值可以根据项目需求调整。太小会导致频繁创建临时文件，太大会失去断点续传的优势。

![Bundle 下载流程](../diagrams/ch04-download-flow.png)

*图 4-1：Bundle 下载完整流程*

上图展示了从上层 API 到底层网络请求的完整流程。`DownloadPackageBundleOperation` 是对外的下载接口，它首先检查文件是否已存在于缓存中。如果不存在，则通过 `DownloadCenter` 获取或创建下载任务。`DownloadCenter` 根据 URL 类型和文件大小选择合适的下载操作类，并管理任务的并发执行。

## WatchDog 超时机制

网络请求的超时控制是下载系统的重要功能。YooAsset 实现了"无数据接收超时"机制，与传统的"总时限超时"有本质区别。

传统超时机制是：从请求开始计时，如果在指定时间内（如 30 秒）没有完成，就判定为超时。这种机制的问题是：在弱网环境下，可能请求一直在以极低速度传输数据，虽然超过了总时限，但并没有真正"卡死"，只是速度慢而已。此时判定为超时并重试，反而可能导致更慢的下载速度。

YooAsset 的 WatchDog 机制则是：监测"最后一次收到数据的时间"。如果超过指定时间（如 10 秒）没有收到任何新数据，才判定为超时。这种机制能更准确地检测"真正的网络卡死"，而不是"网络慢"。

```csharp:Runtime/FileSystem/DefaultCacheFileSystem/Operation/internal/UnityDownloadResumeFileOperation.cs [60:61]
UpdateWatchDog();
if (_webRequest.isDone == false)
    return;
```

在下载更新循环中，每次都会调用 `UpdateWatchDog()` 更新超时检测器。如果长时间没有新数据到达，WatchDog 会主动终止下载任务，触发重试机制。这种"智能超时"在弱网环境下表现更好：它能容忍低速下载，但能快速发现真正的网络中断。

## WebGL 平台特殊策略

WebGL 平台由于浏览器的安全限制，无法像其他平台那样自由访问本地文件系统。这导致 YooAsset 在 WebGL 上无法实现断点续传功能。

更复杂的问题是同源策略。WebGL 应用从 `https://game.example.com` 加载时，如果资源服务器在 `https://cdn.example.com`，浏览器会阻止跨域请求。解决方法有两种：配置 CDN 服务器的 CORS 头允许跨域，或者在游戏域名下部署反向代理服务器。

YooAsset 通过 `DownloadSystemHelper.IsRequestLocalFile()` 判断是否为本地请求：

```csharp:Runtime/DownloadSystem/DownloadSystemHelper.cs [94:103]
public static bool IsRequestLocalFile(string url)
{
    //TODO UNITY_STANDALONE_OSX平台目前无法确定
    if (url.StartsWith("file:"))
        return true;
    if (url.StartsWith("jar:file:"))
        return true;

    return false;
}
```

WebGL 平台下，本地文件路径不会以 `file://` 开头，而是相对路径（如 `StreamingAssets/bundle.bundle`）。因此，WebGL 的"本地文件"会走普通网络下载流程，而不是 `UnityDownloadLocalFileOperation`。此外，WebGL 无法使用 `DownloadHandlerFile` 的追加模式（浏览器不支持），所以断点续传在 WebGL 上会退化为普通下载。

## 设计思考

YooAsset 下载系统的设计体现了良好的分层架构。底层 `UnityWebRequestOperation` 负责网络通信，中间层 `DownloadCenter` 负责调度，上层 `DownloadPackageBundleOperation` 负责业务逻辑（重试、URL 轮换）。这种分层让每层都专注于自己的职责，便于测试和维护。

引用计数管理是另一个设计亮点。同一个文件的多次下载请求会共享同一个下载任务，通过引用计数管理任务生命周期。当所有请求都释放引用后，任务才会被真正销毁。这种设计避免了重复下载，节省了带宽和时间。

断点续传的临时文件机制也很巧妙。通过 `.tmp` 后缀区分临时文件和正式文件，下载完成后校验再转正，确保缓存目录的文件永远完整。这种"先写入临时文件，校验通过后原子性重命名"的模式，是文件系统操作的最佳实践。

## 小结

YooAsset 的下载系统通过分层架构、断点续传、并发控制和智能超时等机制，构建了一个健壮的网络下载解决方案。它不仅能应对各种复杂的网络环境，还能通过任务复用和引用计数优化性能。对于热更游戏来说，下载系统的稳定性直接关系到玩家的第一印象，而 YooAsset 在这方面做了很多细节优化，值得深入学习和借鉴。
