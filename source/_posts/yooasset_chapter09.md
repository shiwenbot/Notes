---
title: "【教材】第9章：扩展开发指南"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 9 章：扩展开发指南

## 概述

YooAsset 的强大之处不仅在于其完善的资源管理功能，更在于其高度可扩展的架构设计。框架通过接口优先的设计哲学，让开发者能够在不修改核心代码的前提下，灵活适配各种业务需求。从文件系统到构建管线，从加密服务到异步操作，YooAsset 提供了丰富的扩展点，让框架真正为业务服务，而不是让业务适应框架。

## 扩展点全景图

YooAsset 的扩展体系主要围绕四个维度展开：运行时服务接口、编辑器收集规则、构建系统扩展以及自定义异步操作。运行时服务接口包括 IFileSystem、IRemoteServices、IDecryptionServices 等 7 个核心接口，覆盖了从文件存储到资源加载的完整链路。编辑器扩展则通过 IPackRule 接口实现自定义收集规则，满足复杂的打包策略需求。构建系统提供 IBuildTask 接口，允许开发者向构建流程插入自定义任务。此外，GameAsyncOperation 基类为自定义异步操作提供了统一的基础设施。这种多层次、全方位的扩展设计，确保了框架能够适应各种特殊场景和平台需求。

![YooAsset 扩展点全景图](../diagrams/ch09-extension-map.png)

*图 9-1：YooAsset 扩展点全景图*

## 自定义文件系统

IFileSystem 接口是 YooAsset 中最复杂的扩展点，包含 15 个方法，涵盖了文件系统的完整生命周期。这些方法可以分为四大类：生命周期管理（InitializeFileSystemAsync、OnCreate、OnDestroy）、文件查询（Exists、Belong、NeedDownload 等）、文件操作（DownloadFileAsync、LoadBundleFile、ReadBundleFileData）以及参数注入（SetParameter）。实现自定义文件系统时，最关键的是正确处理平台差异，特别是在微信小游戏等特殊平台上。

注册自定义文件系统需要通过 FileSystemParameters 完成。以微信小游戏为例，其 WechatFileSystem 实现展示了三个关键技巧：FileRoot 返回空字符串以适配微信文件系统 API、内嵌 WebRemoteServices 作为默认 CDN 服务、使用微信平台的 WXFileSystemManager 替代 System.IO。这种设计既保持了接口的统一性，又充分利用了平台特性。

```csharp:Mini Game/Runtime/WechatFileSystem/WechatFileSystem.cs [32:62]
internal class WechatFileSystem : IFileSystem
{
    private class WebRemoteServices : IRemoteServices
    {
        private readonly string _webPackageRoot;
        protected readonly Dictionary<string, string> _mapping = new Dictionary<string, string>(10000);

        public WebRemoteServices(string buildinPackRoot)
        {
            _webPackageRoot = buildinPackRoot;
        }
        string IRemoteServices.GetRemoteMainURL(string fileName)
        {
            return GetFileLoadURL(fileName);
        }
        string IRemoteServices.GetRemoteFallbackURL(string fileName)
        {
            return GetFileLoadURL(fileName);
        }

        private string GetFileLoadURL(string fileName)
        {
            if (_mapping.TryGetValue(fileName, out string url) == false)
            {
                string filePath = PathUtility.Combine(_webPackageRoot, fileName);
                url = DownloadSystemHelper.ConvertToWWWPath(filePath);
                _mapping.Add(fileName, url);
            }
            return url;
        }
    }
```

WechatFileSystem 的另一个亮点是其参数注入机制。通过 SetParameter 方法，文件系统可以接收 IRemoteServices、IWebDecryptionServices 和 IManifestRestoreServices 等依赖，这种设计既保持了灵活性，又避免了硬编码依赖。在 OnCreate 方法中，如果未提供 RemoteServices，会自动创建基于 StreamingAssets 的 WebRemoteServices 作为后备方案，这种默认值处理大大降低了使用门槛。

## Runtime Services 接口实战

Runtime Services 是 YooAsset 运行时扩展的核心，包含 7 个关键接口。IRemoteServices 接口相对简单，只负责构建 CDN 下载地址。其典型实现是将文件名拼接上服务器根路径，同时支持主备 URL 切换以应对网络故障。在多区域部署场景中，甚至可以根据 IP 地理位置返回最近的 CDN 节点地址，这种灵活的 URL 构建机制为全球化资源分发提供了基础。

```csharp:Runtime/Services/IRemoteServices.cs [4:17]
public interface IRemoteServices
{
    /// <summary>
    /// 获取主资源站的资源地址
    /// </summary>
    /// <param name="fileName">请求的文件名称</param>
    string GetRemoteMainURL(string fileName);

    /// <summary>
    /// 获取备用资源站的资源地址
    /// </summary>
    /// <param name="fileName">请求的文件名称</param>
    string GetRemoteFallbackURL(string fileName);
}
```

IDecryptionServices 接口则承担了资源解密的核心职责。它提供了三种加载方式：同步的 LoadAssetBundle、异步的 LoadAssetBundleAsync，以及作为保底机制的 LoadAssetBundleFallback。这种多层次设计充分考虑了不同场景的需求——同步加载用于关键初始化阶段，异步加载保证流畅体验，而 Fallback 机制则在特殊加密算法失败时提供通过 LoadFromMemory 的后备方案。接口还提供了 ReadFileData 和 ReadFileText 方法，用于读取原始文件内容，这对于解析配置文件或二进制数据非常有用。

```csharp:Runtime/Services/IDecryptionServices.cs [42:71]
public interface IDecryptionServices
{
    /// <summary>
    /// 同步方式获取解密的资源包
    /// </summary>
    DecryptResult LoadAssetBundle(DecryptFileInfo fileInfo);

    /// <summary>
    /// 异步方式获取解密的资源包
    /// </summary>
    DecryptResult LoadAssetBundleAsync(DecryptFileInfo fileInfo);

    /// <summary>
    /// 后备方式获取解密的资源包
    /// 注意：当正常解密方法失败后，会触发后备加载！
    /// 说明：建议通过LoadFromMemory()方法加载资源包作为保底机制。
    /// issues : https://github.com/tuyoogame/YooAsset/issues/562
    /// </summary>
    DecryptResult LoadAssetBundleFallback(DecryptFileInfo fileInfo);

    /// <summary>
    /// 获取解密的字节数据
    /// </summary>
    byte[] ReadFileData(DecryptFileInfo fileInfo);

    /// <summary>
    /// 获取解密的文本数据
    /// </summary>
    string ReadFileText(DecryptFileInfo fileInfo);
}
```

在构建阶段，IEncryptionServices 与 IManifestProcessServices/IManifestRestoreServices 形成了对称的加密处理对。IEncryptionServices 在构建时对资源文件进行加密，IManifestProcessServices 对清单文件进行压缩或加密，而 IManifestRestoreServices 则在运行时执行逆向操作。这种对称设计确保了构建和运行时的加密解密流程完全匹配，避免了常见的数据不一致问题。一个完整的加密接入示例包括：在 Editor 端实现 IEncryptionServices 对 AssetBundle 数据进行 AES 加密，实现 IManifestProcessServices 对清单文件进行 GZIP 压缩；在 Runtime 端实现对应的 IDecryptionServices 和 IManifestRestoreServices 执行解密和解压操作。

```csharp:Runtime/Services/IManifestProcessServices.cs [4:13]
/// <summary>
/// 资源清单文件处理服务接口
/// </summary>
public interface IManifestProcessServices
{
    /// <summary>
    /// 处理资源清单（压缩或加密）
    /// </summary>
    byte[] ProcessManifest(byte[] fileData);
}
```

## 自定义收集规则

IPackRule 接口是编辑器端扩展的核心，用于控制资源收集和打包策略。实现自定义收集规则时，只需实现 GetPackRuleResult 方法，返回 PackRuleResult 对象指定 Bundle 名称和文件扩展名。一个典型场景是特效纹理的智能分组：根据文件名首字母将纹理分配到不同 Bundle，既避免单个 Bundle 过大，又保持合理的加载粒度。

```csharp:EditorExtension/CustomCollectRules/CustomPackRule.cs [8:24]
[DisplayName("打包特效纹理（自定义）")]
public class PackEffectTexture : IPackRule
{
    private const string PackDirectory = "Assets/Effect/Textures/";

    PackRuleResult IPackRule.GetPackRuleResult(PackRuleData data)
    {
        string assetPath = data.AssetPath;
        if (assetPath.StartsWith(PackDirectory) == false)
            throw new Exception($"Only support folder : {PackDirectory}");

        string assetName = Path.GetFileName(assetPath).ToLower();
        string firstChar = assetName.Substring(0, 1);
        string bundleName = $"{PackDirectory}effect_texture_{firstChar}";
        var packRuleResult = new PackRuleResult(bundleName, DefaultPackRule.AssetBundleFileExtension);
        return packRuleResult;
    }
}
```

何时需要自定义收集规则？当默认的收集规则无法满足业务需求时，就应该考虑自定义。常见场景包括：需要按业务模块而非资源类型分组、需要将关联资源打包到同一 Bundle 以减少依赖、需要为特定资源设置特殊的文件扩展名（如视频文件的原始扩展名）、或者需要实现动态的打包策略（如根据资源配置表决定打包方式）。通过 DisplayNameAttribute 可以为自定义规则指定友好的显示名称，这在 YooAsset 编辑器工具中选择收集规则时会显示出来，大大提升了用户体验。

## 自定义构建管线

YooAsset 的构建系统基于任务管道（Pipeline）模式，每个构建步骤都是一个 IBuildTask 实现。IBuildTask 接口非常简洁，只包含一个 Run 方法，接收 BuildContext 作为参数。BuildContext 在任务间传递，携带构建过程中的所有共享数据，包括收集结果、构建参数、清单信息等。这种设计使得各个任务可以解耦开发，同时又能够共享上下文信息。

```csharp:Editor/AssetBundleBuilder/BuildSystem/IBuildTask.cs [4:7]
public interface IBuildTask
{
    void Run(BuildContext context);
}
```

自定义构建管线有两种方式：向 BuiltinBuildPipeline 插入自定义任务，或者完全自定义构建流程。插入方式适合在现有流程基础上增加特定功能，如构建后自动上传 CDN、生成版本报告等。而完全自定义流程则适合需要完全控制构建过程的场景，如实现增量构建、多平台并行构建等。无论哪种方式，都需要通过 BuildPipelineAttribute 特性注册，这样才能在 YooAsset 的构建界面显示出来。

```csharp:Editor/AssetBundleBuilder/VisualViewers/BuildPipelineAttribute.cs [5:13]
public class BuildPipelineAttribute : Attribute
{
    public string PipelineName;

    public BuildPipelineAttribute(string name)
    {
        this.PipelineName = name;
    }
}
```

一个完整的自定义 Pipeline 骨架包括：定义实现 IBuildTask 的多个任务类、创建任务编排逻辑、通过 BuildPipelineAttribute 注册。在任务编排时，需要特别注意任务依赖关系——例如，必须先执行收集任务，再执行构建任务，最后执行清单生成任务。BuildContext 提供了丰富的上下文信息，包括构建参数、收集结果、版本信息等，合理利用这些信息可以让任务实现更加简洁高效。

## GameAsyncOperation：自定义异步操作

GameAsyncOperation 是 YooAsset 中所有异步操作的基类，提供了统一的异步编程模型。自定义异步操作时，需要继承 GameAsyncOperation 并实现核心方法：OnStart 执行初始化逻辑、OnUpdate 处理每帧更新、OnAbort 处理取消操作。这种设计模式将异步逻辑封装在操作对象内部，调用者只需等待操作完成即可，大大简化了异步代码的复杂度。

以 LoadGameObjectOperation 为例，它展示了自定义异步操作的完整实现。该操作内部封装了资源加载和实例化两个步骤，对外暴露了 Go 属性用于访问加载的游戏对象。实现的关键在于状态机设计——通过 ESteps 枚举管理操作状态，在 OnUpdate 中根据当前状态执行相应逻辑。Progress 属性会自动转发到内部的 AssetHandle，确保进度条正确显示。这种组合式设计允许开发者将复杂的异步流程封装成单一操作对象，提升了代码的可维护性。

```csharp:RuntimeExtension/ExtensionOperation/LoadGameObjectOperation.cs [17:81]
public class LoadGameObjectOperation : GameAsyncOperation
{
    private enum ESteps
    {
        None,
        LoadAsset,
        Done,
    }

    private readonly string _location;
    private readonly Vector3 _positon;
    private readonly Quaternion _rotation;
    private readonly Transform _parent;
    private readonly bool _destroyGoOnRelease;
    private AssetHandle _handle;
    private ESteps _steps = ESteps.None;

    /// <summary>
    /// 加载的游戏对象
    /// </summary>
    public GameObject Go { private set; get; }


    public LoadGameObjectOperation(string location, Vector3 position, Quaternion rotation, Transform parent, bool destroyGoOnRelease = false)
    {
        _location = location;
        _positon = position;
        _rotation = rotation;
        _parent = parent;
        _destroyGoOnRelease = destroyGoOnRelease;
    }
    protected override void OnStart()
    {
        _steps = ESteps.LoadAsset;
    }
    protected override void OnUpdate()
    {
        if (_steps == ESteps.None || _steps == ESteps.Done)
            return;

        if (_steps == ESteps.LoadAsset)
        {
            if (_handle == null)
            {
                _handle = YooAssets.LoadAssetAsync<GameObject>(_location);
            }

            Progress = _handle.Progress;
            if (_handle.IsDone == false)
                return;

            if (_handle.Status != EOperationStatus.Succeed)
            {
                Error = _handle.LastError;
                Status = EOperationStatus.Failed;
                _steps = ESteps.Done;
            }
            else
            {
                Go = _handle.InstantiateSync(_positon, _rotation, _parent);
                Status = EOperationStatus.Succeed;
                _steps = ESteps.Done;
            }
        }
    }
}
```

自定义异步操作的使用非常简单，通过 YooAssets.StartOperation 方法启动即可。这个方法会自动管理操作的生命周期，确保每帧调用其 OnUpdate 方法。对于需要返回结果的操作，可以在操作类中添加公开属性（如 LoadGameObjectOperation 的 Go 属性），调用者在操作完成后即可访问结果。这种设计模式既保持了接口的统一性，又提供了足够的灵活性，是 YooAsset 异步编程模型的核心支柱。

## 设计思考

YooAsset 的扩展系统采用了反射注入而非依赖注入容器，这是一个经过深思熟虑的设计选择。反射注入的优点是简单直接、零额外依赖、性能开销可控，通过字符串类名注册的方式在 Editor 和 Runtime 都能良好工作。而依赖注入容器虽然功能强大，但在 Unity 环境下会引入额外的复杂性和学习成本，对于资源管理这种相对稳定的场景来说未必是最佳选择。

字符串类名注册的取舍同样体现了实用主义原则。其缺点是缺乏编译时检查，容易出现拼写错误；但优点是支持配置文件驱动、支持运行时动态替换、降低了代码耦合度。在 YooAsset 的实际应用中，这种灵活性带来了显著收益——例如可以通过配置文件在不同环境切换文件系统实现，而无需重新编译代码。当然，为了缓解字符串硬编码的风险，YooAsset 在内部使用了常量定义（如 FileSystemParametersDefine.REMOTE_SERVICES），并在错误提示中提供了详细的调试信息。

## 小结

YooAsset 的扩展开发体系展现了框架设计的艺术——通过接口定义边界、通过反射注入解耦、通过默认值降低门槛。从 IFileSystem 的 15 个方法到 GameAsyncOperation 的状态机模式，每个扩展点都经过精心设计，既保证了功能的完整性，又保持了使用的简洁性。理解这些扩展点的内在逻辑，不仅能够帮助我们更好地使用 YooAsset，更能启发我们在自己的项目中设计优雅的扩展系统。真正优秀的框架不是做加法，而是做减法——通过合理的抽象和扩展机制，让复杂的系统变得简单，让简单的需求变得可能。

