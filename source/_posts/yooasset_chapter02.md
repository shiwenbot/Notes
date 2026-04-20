---
title: "【教材】第2章：异步操作系统"
date: 2026-04-20
categories:
  - 代码库
tags:
  - YooAsset
---

# 第 2 章：异步操作系统

## 概述

YooAsset 里几乎所有的异步流程都跑在 OperationSystem 上。从资源初始化到下载文件，再到加载 AssetBundle，每一个异步操作都被抽象成 AsyncOperationBase 的子类，由 OperationSystem 统一调度。

理解它的调度模型，是读懂后续章节的基础。如果你跳过这一章，后面看到 `StartOperation`、`UpdateOperation` 这些方法时，就会一头雾水。

OperationSystem 的核心职责很简单：管理所有异步操作的生命周期，每帧更新它们，并提供优先级调度和时间片控制。它像是一个精简的"操作系统内核"，负责任务的创建、调度和回收。

## AsyncOperationBase 状态机

每个异步操作都有四种状态：None、Processing、Succeed、Failed。

```csharp:Runtime/OperationSystem/EOperationStatus.cs [1:10]
namespace YooAsset
{
    public enum EOperationStatus
    {
        None,
        Processing,
        Succeed,
        Failed
    }
}
```

状态流转是单向的：None → Processing → Succeed/Failed。一旦进入终态（Succeed 或 Failed），就无法再回到 Processing。这是一个典型的状态机设计。

AsyncOperationBase 用 `Status` 字段记录当前状态，并提供了两个重要属性：`IsDone` 和 `IsFinish`。

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [38:70]
/// <summary>
/// 任务状态
/// </summary>
public EOperationStatus Status { get; protected set; } = EOperationStatus.None;

/// <summary>
/// 是否已经完成
/// </summary>
public bool IsDone
{
    get
    {
        return Status == EOperationStatus.Failed || Status == EOperationStatus.Succeed;
    }
}
```

`IsDone` 判断任务是否已进入终态（Succeed 或 Failed）。而 `IsFinish` 则是一个内部标志，表示是否已经执行过完成回调。它们的语义差异很重要：`IsDone` 表示"任务执行完毕"，`IsFinish` 表示"任务后处理完毕"。

当任务从 Processing 变为 Succeed/Failed 时，会触发完成回调，并将 `Progress` 设置为 1.0。这个过程在 `UpdateOperation` 方法中实现。

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [170:196]
internal void UpdateOperation()
{
    if (IsDone == false)
    {
        // 更新记录
        DebugUpdateRecording();

        // 更新任务
        InternalUpdate();
    }

    if (IsDone && IsFinish == false)
    {
        IsFinish = true;

        // 进度百分百完成
        Progress = 1f;

        // 结束记录
        DebugEndRecording();

        //注意：如果完成回调内发生异常，会导致Task无限期等待
        _callback?.Invoke(this);

        if (_taskCompletionSource != null)
            _taskCompletionSource.TrySetResult(null);
    }
}
```

![AsyncOperationBase 状态机](../diagrams/ch02-state-machine.png)
*图 2-1：AsyncOperationBase 状态机流转*

## 双队列与帧循环

OperationSystem 使用双队列设计：`_operations` 主队列存储正在执行的操作，`_newList` 暂存队列存储新添加的操作。

```csharp:Runtime/OperationSystem/OperationSystem.cs [18:19]
private static readonly List<AsyncOperationBase> _operations = new List<AsyncOperationBase>(1000);
private static readonly List<AsyncOperationBase> _newList = new List<AsyncOperationBase>(1000);
```

为什么要双队列？因为 Update 在遍历 `_operations` 时，如果有新操作启动，不能直接修改正在遍历的集合，否则会抛异常。所以新操作先放到 `_newList`，等遍历结束后再合并。

Update 方法分为三个阶段：

1. **清理阶段**：移除上一帧已完成的操作
2. **合并阶段**：将 `_newList` 合并到主队列，并根据优先级排序
3. **执行阶段**：遍历主队列，调用每个操作的 `UpdateOperation`

```csharp:Runtime/OperationSystem/OperationSystem.cs [59:110]
public static void Update()
{
    // 移除已经完成的异步操作
    // 注意：移除上一帧完成的异步操作，方便调试器接收到完整的信息！
    for (int i = _operations.Count - 1; i >= 0; i--)
    {
        var operation = _operations[i];
        if (operation.IsFinish)
        {
            _operations.RemoveAt(i);

            if (_finishCallback != null)
                _finishCallback.Invoke(operation.PackageName, operation);
        }
    }

    // 添加新增的异步操作
    if (_newList.Count > 0)
    {
        bool sorting = false;
        foreach (var operation in _newList)
        {
            if (operation.Priority > 0)
            {
                sorting = true;
                break;
            }
        }

        _operations.AddRange(_newList);
        _newList.Clear();

        // 重新排序优先级
        if (sorting)
            _operations.Sort();
    }

    // 更新进行中的异步操作
    bool checkBusy = MaxTimeSlice < long.MaxValue;
    _frameTime = _watch.ElapsedMilliseconds;
    for (int i = 0; i < _operations.Count; i++)
    {
        if (checkBusy && IsBusy)
            break;

        var operation = _operations[i];
        if (operation.IsFinish)
            continue;

        operation.UpdateOperation();
    }
}
```

清理阶段从后往前遍历，这样 `RemoveAt` 不会影响索引。注意这里用的是 `IsFinish` 而非 `IsDone`，原因是：如果刚移除 `IsDone` 的操作，调试器可能还来不及查看它的状态信息。延迟一帧再移除，方便调试。

合并阶段有个细节：只有当 `_newList` 中存在 `Priority > 0` 的操作时，才会触发排序。这是一个性能优化——排序是 O(n log n) 操作，能省则省。

执行阶段会检查 `IsBusy`，如果时间片用完就立即停止更新。这个机制后面会详细讲。

![OperationSystem 帧循环](../diagrams/ch02-update-cycle.png)
*图 2-2：OperationSystem 三阶段帧循环*

## 优先级与时间片

每个异步操作都有一个 `Priority` 字段，默认值为 0。Priority 越大，越优先执行。

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [33:33]
public uint Priority { set; get; } = 0;
```

排序逻辑在 `AsyncOperationBase.CompareTo` 方法中实现：

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [323:326]
public int CompareTo(AsyncOperationBase other)
{
    return other.Priority.CompareTo(this.Priority);
}
```

注意这里是降序排列（`other.Priority.CompareTo(this.Priority)`），Priority 大的排在前面。

只有 Priority > 0 的操作才会参与排序。这个设计很有意思：默认情况下（Priority=0），操作按照"先来先服务"的顺序执行；只有显式设置了优先级的操作，才会插队到前面。这样既保证了公平性，又支持了优先级调度。

时间片是另一个重要的调度机制。OperationSystem 用 `MaxTimeSlice` 限制每帧执行异步操作的总时长：

```csharp:Runtime/OperationSystem/OperationSystem.cs [24:30]
// 计时器相关
private static Stopwatch _watch;
private static long _frameTime;

/// <summary>
/// 异步操作的最小时间片段
/// </summary>
public static long MaxTimeSlice { set; get; } = long.MaxValue;
```

`IsBusy` 属性检查当前帧是否超时：

```csharp:Runtime/OperationSystem/OperationSystem.cs [35:45]
public static bool IsBusy
{
    get
    {
        if (_watch == null)
            return false;

        // NOTE : 单次调用开销约1微秒
        return _watch.ElapsedMilliseconds - _frameTime >= MaxTimeSlice;
    }
}
```

每帧开始时记录 `_frameTime`，然后在遍历操作时检查 `IsBusy`。如果超时，立即停止更新剩余操作。这个机制防止某帧的异步操作过多，导致卡顿。

## 三种等待方式

AsyncOperationBase 支持三种等待方式：回调、协程、Task。

### 回调方式

最简单的是 Completed 事件：

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [75:88]
public event Action<AsyncOperationBase> Completed
{
    add
    {
        if (IsDone)
            value.Invoke(this);
        else
            _callback += value;
    }
    remove
    {
        _callback -= value;
    }
}
```

如果操作已经完成（`IsDone` 为 true），订阅时立即触发回调。否则，将回调保存到 `_callback` 委托链中，等操作完成时统一触发。

注意代码中的注释："如果完成回调内发生异常，会导致 Task 无限期等待"。这是因为回调抛异常时，`_taskCompletionSource.TrySetResult` 可能不会被执行，导致 Task 永远不完成。

### 协程方式

AsyncOperationBase 实现了 `IEnumerator` 接口：

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [330:337]
bool IEnumerator.MoveNext()
{
    return !IsDone;
}
void IEnumerator.Reset()
{
}
object IEnumerator.Current => null;
```

`MoveNext` 返回 `!IsDone`，所以可以在协程中直接 yield：

```csharp
yield return operation; // 操作完成后再继续执行
```

### Task 方式

Task 属性采用懒初始化策略：

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [93:105]
public Task Task
{
    get
    {
        if (_taskCompletionSource == null)
        {
            _taskCompletionSource = new TaskCompletionSource<object>();
            if (IsDone)
                _taskCompletionSource.SetResult(null);
        }
        return _taskCompletionSource.Task;
    }
}
```

只有访问 Task 属性时，才创建 `TaskCompletionSource`。如果操作已经完成，立即设置结果。否则，等到操作完成时在 `UpdateOperation` 中设置。

这种懒初始化的好处是：如果用户不使用 Task，就不会创建 `TaskCompletionSource`，避免不必要的内存分配。

## GameAsyncOperation 扩展点

GameAsyncOperation 是 AsyncOperationBase 的子类，它定义了四个扩展方法：

```csharp:Runtime/OperationSystem/GameAsyncOperation.cs [5:41]
internal override void InternalStart()
{
    OnStart();
}
internal override void InternalUpdate()
{
    OnUpdate();
}
internal override void InternalAbort()
{
    OnAbort();
}
internal override void InternalWaitForAsyncComplete()
{
    OnWaitForAsyncComplete();
}

/// <summary>
/// 异步操作开始
/// </summary>
protected abstract void OnStart();

/// <summary>
/// 异步操作更新
/// </summary>
protected abstract void OnUpdate();

/// <summary>
/// 异步操作终止
/// </summary>
protected abstract void OnAbort();

/// <summary>
/// 异步等待完成
/// </summary>
protected virtual void OnWaitForAsyncComplete() { }
```

子类只需重写这四个方法，就能实现自定义的异步操作逻辑。这是一个典型的模板方法模式。

`WaitForAsyncComplete` 方法提供了一个"同步强制完成"的路径：

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [251:268]
public void WaitForAsyncComplete()
{
    if (IsDone)
        return;

    //TODO 防止异步操作被挂起陷入无限死循环！
    // 例如：文件解压任务或者文件导入任务！
    if (Status == EOperationStatus.None)
    {
        StartOperation();
    }

    if (IsWaitForAsyncComplete == false)
    {
        IsWaitForAsyncComplete = true;
        InternalWaitForAsyncComplete();
    }
}
```

这个方法会阻塞当前线程，直到操作完成。它主要用于编辑器工具或测试代码，在运行时应该避免使用。

为了防止死循环，`ExecuteWhileDone` 方法提供了熔断机制：

```csharp:Runtime/OperationSystem/AsyncOperationBase.cs [221:238]
protected bool ExecuteWhileDone()
{
    if (IsDone == false)
    {
        // 执行更新逻辑
        InternalUpdate();

        // 当执行次数用完时
        _whileFrame--;
        if (_whileFrame <= 0)
        {
            Status = EOperationStatus.Failed;
            Error = $"Operation {this.GetType().Name} failed to wait for async complete !";
            YooLogger.Error(Error);
        }
    }
    return IsDone;
}
```

`_whileFrame` 默认值为 1000，超过这个次数还没完成，就强制标记为失败。这是最后的防线，避免无限卡死。

## 设计思考

为什么 YooAsset 要自研调度器，而不用 UniTask 或协程？

原因有三：

第一，UniTask 虽然强大，但引入了额外的依赖和学习成本。YooAsset 的异步操作相对简单，不需要那么复杂的调度逻辑。

第二，协程的调度由 Unity 控制，无法精确控制时间片。OperationSystem 可以通过 `MaxTimeSlice` 限制每帧的执行时长，避免卡顿。

第三，OperationSystem 提供了统一的调试接口和性能统计。所有异步操作都在一个系统中管理，方便监控和调优。

双队列的设计权衡也很值得思考。单队列加锁虽然也能解决遍历时修改的问题，但锁会带来性能开销。双队列虽然多了一次合并操作，但避免了锁，在高并发场景下性能更好。

## 小结

OperationSystem 是 YooAsset 异步操作的"心脏"，它用简单的双队列 + 帧循环模型，实现了高效的任务调度。

核心要点：
- AsyncOperationBase 的状态机：None → Processing → Succeed/Failed
- IsDone 和 IsFinish 的语义差异
- 双队列设计避免遍历时修改集合
- 优先级调度和时间片熔断
- 三种等待方式：回调、协程、Task
- GameAsyncOperation 提供的扩展点

理解了这些，再看 YooAsset 的其他部分，就会清晰很多。
