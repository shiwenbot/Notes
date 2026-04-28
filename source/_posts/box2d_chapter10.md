---
title: "【教材】第10章：性能优化——让物理跑得更快"
date: 2026-04-05
order: 010
categories:
  - 代码库
tags:
  - Box2D
description: "内存池、并行求解器、SIMD向量化、岛屿管理与睡眠机制"
---

## 概述

Box2DSharp 在性能上做了全栈优化。从减少 GC 压力的内存池，到利用多核的并行求解器，再到 SIMD 向量化的约束计算，每一层都在追求更低的延迟。你还会看到岛屿管理和睡眠机制如何用"不计算"来换取最大性能，以及约束图着色如何让多线程安全地并行工作。

## 内存管理：B2Array / Pool / FixedArray 的设计思想

物理引擎每帧都要创建和销毁大量临时数据结构。Box2DSharp 没有简单依赖 .NET 的 GC，而是自己实现了一套轻量级的内存管理设施：B2Array、B2ArrayPool、B2ObjectPool 和 FixedArray。

### B2Array：带池化的动态数组

B2Array 是最常用的容器，它比普通 List 更适合物理引擎的工作负载。




它内部通过 `B2ArrayPool<T>.Shared.Rent` 获取数组内存，扩容时走 `Grow()` 按 50% 增长。需要特别注意的是 `RemoveSwap`：它不是像 List 那样移动后续元素，而是直接用末尾元素覆盖被删位置，保证 O(1) 的删除开销。当你处理成千上万个接触点或刚体时，这个细节能省下大量 CPU 时间。

### B2ArrayPool 与 B2ObjectPool：减少 GC 的两大池子

B2ArrayPool 是对 `System.Buffers.ArrayPool<T>` 的薄封装，加了自旋锁来保证线程安全。



`Rent` 借数组，`Return` 还数组，`Resize` 则负责在池内完成扩容和拷贝。物理步进中那些大大小小的数组——比如 `ContactConstraintSIMD[]`、`SolverBlock[]`——几乎都是走这个池子，步进结束后统一归还。

如果说 B2ArrayPool 管的是"数组内存"，那 B2ObjectPool 管的就是"对象实例"。



这个池子同样用自旋锁保护，`_objects` 数组存被回收的对象实例。`Get` 时优先从池中复用，空了才 `new T()`；`Return` 时把对象塞回去，容量不够就翻倍。这样像 `ContinuousContext` 这类求解过程中频繁创建的对象，就不会再给 GC 添乱了。

### FixedArray：把 small array 放到栈或结构体里

物理引擎里很多数据天然就是固定长度的，比如多边形的顶点通常是 3~8 个。如果每次都用堆分配数组，既浪费又分散缓存。


`FixedArray3<T>` 直接把三个字段 `V0`、`V1`、`V2` 内联在结构体里，通过 `MemoryMarshal.CreateSpan` 提供 Span 视图。这意味着它完全 inline，没有堆分配，也不需要 GC 追踪，CPU 缓存友好得很。类似的还有 `FixedArray8` 等，专门服务于那些长度可预期的小批量数据。

总的来说，Box2DSharp 的内存策略非常清晰：B2Array 负责可变长度的批量数据，ArrayPool 和 ObjectPool 负责回收复用，FixedArray 负责消除小尺寸的堆分配。三层配合下来，GC 压力被压到很低，缓存命中率也更高。

## 并行计算：WorkerCount / EnqueueTask / FinishTask

现代 CPU 核心数越来越多，物理引擎能不能把计算拆开并行做，直接决定了它能不能跑满多核。Box2DSharp 没有绑死某个线程库，而是定义了两个回调，让用户把并行的"开关"握在自己手里。

### 配置工作线程数

一切从 `WorldDef` 开始。默认情况下 `WorkerCount = 1`，也就是单线程。



你可以把它设成你期望的线程数（通常不超过性能核心数），Box2DSharp 会在创建 World 时自动截断到 `Core.MaxWorkers`。


### 两个回调：EnqueueTask 与 FinishTask

真正让引擎"会用"多线程的，是这两个委托：



`EnqueueTaskCallback` 接收一个 `TaskCallback`（实际工作任务）、`itemCount`（总工作项数）、`minRange`（建议每个线程的最小任务量）以及上下文参数。你的任务系统需要把这个工作拆成若干份，丢给不同的线程执行。如果工作很简单、直接在回调里同步跑完了，你可以返回 `null`，这样引擎就知道不需要再调 `FinishTask` 了。

`FinishTaskCallback` 则是用来等待任务完成的。引擎在 `EnqueueTask` 之后，会紧接着调用 `FinishTask` 来确保本轮并行阶段全部结束。

Box2DSharp 内部也提供了默认的单线程实现：


### 并行求解的实际调度

在 `Solver.Solve()` 里，Box2DSharp 会把整个物理步进拆成多个 `SolverStage`，比如准备关节、积分速度、WarmStart、Solve、积分位置等等。每个阶段内部再切成多个 `SolverBlock`，然后分发给所有 Worker。


这段代码就是为每个 worker 发起 `SolverTask`。Worker 0 作为主线程，不仅负责同步，也会参与实际计算；其他 worker 则通过原子变量 `AtomicSyncBits` 来感知当前阶段， steal 或执行分配给自己的 block。

除了约束求解，FinalizeBodies、FastBodyTask、BulletBodyTask 这些阶段也通过同样的 `EnqueueTask` / `FinishTask` 模式并行化。比如：


总结一下：Box2DSharp 的并行设计非常"开放"——它不自带线程池，只定义了两个标准接口。你可以接入 `Parallel.For`、`enkiTS`，或者你自己写的 job system，只要正确实现 EnqueueTask 和 FinishTask，就能让整个物理世界跑在多核上。

## SIMD 约束求解：ContactConstraintSIMD 原理简介

Box2DSharp 的求解器在处理接触约束时，大量使用了 SIMD（单指令多数据）向量化运算。核心思路是把 8 个独立的 `ContactConstraint` 打包成一个 `ContactConstraintSIMD` 实例，然后一次性完成准备、Warm Start、Solve 和 Store 四个阶段，把 CPU 的吞吐量吃满。


你也不必手动编写 SIMD 指令。Box2DSharp 在 `ContactSolver` 里封装了 `GatherBodies` 和 `ScatterBodies` 两个函数，负责把 8 个 `BodyState` 从内存里批量装载到 `SimdBody`，求解完后再批量写回。如果平台支持 AVX，它会走 `Avx.LoadVector256` 与转置优化；不支持的 CPU 则优雅回退到 For 循环版本。


你不需要做任何额外配置，默认的 `SolveContactsTask` 已经会按 SIMD 宽度分组并自动调度。唯一需要注意的是：如果场景里的接触数量不是 8 的整数倍，末尾剩余的几个约束会进入 Overflow 通道，用普通标量路径处理。这对性能影响微乎其微。

## 岛屿管理与睡眠：Island 合并/分裂、睡眠/唤醒优化

Box2DSharp 把整个物理世界划分成若干 **Island**。每个 Island 是由刚体、接触和关节连接而成的连通分量。Island 的意义在于：一个静止的小岛，可以整片进入睡眠，完全跳过积分和碰撞检测，这是性价比最高的性能优化。

当两个原本分离的 Island 因为新接触而连通时，就需要合并。


这里用了一个带路径压缩的 **Union-Find** 数据结构：先找到两个 Island 的根，如果不同，就把其中一个挂到另一个下面。为了避免每帧都执行大量链表操作，真正的合并被延迟到 `MergeAwakeIslands` 里统一处理。


合并之后，Island 的规模会变大；相反，当接触或关节被移除时，原来的 Island 可能不再连通，这时候就要触发 **Split**。Box2DSharp 只会在 `ConstraintRemoveCount > 0` 的情况下执行一次 DFS 分裂。代码用了一个临时栈和 `B2ArrayPool<int>` 租借数组，避免在热路径上分配堆内存。

更重要的是睡眠与唤醒的联动：在 `LinkContact` 中可以看到，只要 island A 是 awake 的，而 island B 正在睡觉，`SolverSet.WakeSolverSet` 就会被触发。这意味着你朝一堆沉睡的箱子扔一块石头，这块石头会沿着约束链把所有相关刚体全部唤醒，不需要你手动干预。

## 约束图着色：ConstraintGraph / GraphColor 并行化

光有 SIMD 还不够，Box2DSharp 还要让多个工作线程同时求解不同的约束。问题是：如果两个接触共享同一个刚体，求解顺序会互相影响——一个线程在改速度，另一个线程也在改同一个速度，就会产生数据竞争。

解决的办法是 **图着色**。Box2DSharp 把约束图拆成若干颜色，规则很简单：同一个颜色里的约束，不能共享任何动态刚体。这样每个颜色就可以被多个线程安全地并行求解。


`ConstraintGraph` 内部维护了一个 `GraphColor[] Colors`，默认 12 个槽位。其中前 11 个是正常颜色，最后一个 `OverflowIndex` 专门收容那些因为某个刚体连接太多约束、没法找到一个干净颜色的"漏网之鱼"。


添加接触时，`AddContactToGraph` 会从前到后扫描颜色，找到一个两个 bodyId 的 bit 都还没被置位的颜色。如果两个都是动态刚体（最一般的情况），它会从 color 0 开始找；如果有一个是静态的，则从 color 1 开始，因为静态刚体不会被写入速度，冲突风险更低。着色的开销是 O(GraphColorCount)，非常快。

运行时，`SolveContactsTask` 会对每个颜色分别调用，不同颜色之间天然无需加锁。唯一的一点损失是：颜色越多，约束内存越分散，cache miss 可能略有上升。但换来的并行收益通常远远盖过这点开销。

## 实用优化建议：场景规模 / 子步数 / 形状复杂度 / 睡眠启用

聊了这么多内部机制，最后落到开发者手里能拧的旋钮并不多，但每个都很关键。

### 1. 睡眠开关不要关

`WorldDef.EnableSleep` 默认就是 `true`。如果你把它关掉，所有 Island 每帧都要积分和求解，哪怕是一动不动堆在地上的箱子。对于大地图或堆叠场景，这就是零成本和金矿的区别。


### 2. WorkerCount 跟着物理核走

`WorkerCount` 决定并行任务系统的线程数。Box2DSharp 官方建议只绑定性能核（P-Core），避开超线程和能效核（E-Core）。因为物理引擎对 L2/L3 cache 非常敏感，超线程的两个逻辑核共享资源反而可能互相挤占。实际配置时，先试 `物理核数 / 2`，再根据 Profile 微调。

### 3. 子步数不是越多越好

Box2DSharp 默认使用 sub-stepping，子步数直接决定单位时间里求解器跑几轮。更多子步意味着更稳定的堆叠和更精确的高速碰撞，但 CPU 消耗线性增长。日常游戏推荐 `1~4`；需要精确模拟（如竞技平台跳跃、弹球）可以开到 `8+`，再往上边际效益会迅速衰减。

### 4. 控制形状复杂度

多边形（Polygon）的顶点数直接影响碰撞检测和 Manifold 生成速度。Box2D 的凸多边形上限是 8 个顶点，但你的设计没必要顶满。圆、胶囊（Capsule）的 BroadPhase 和 NarrowPhase 都更快，而且滚动行为更稳定。如果场景里有大量地形，优先用 Edge/Chain Shape 而不是一排凸多边形。

### 5. 注意场景规模与 Island 大小

当一个 Island 包含成百上千个刚体（比如一堵砖墙），它的求解和分裂成本会显著上升。可以通过在设计层面打断约束链：给堆叠的箱子之间留一点缝隙、减少不必要的关节、或者把静态地形拆成独立的 piece。岛越小，睡眠和并行的效率就越高。

## 设计思考：精度与性能的权衡

做物理模拟时，最纠结的问题往往是：精度要多少才够？性能要牺牲多少才值？

Box2DSharp 给了你一把很好的尺子——`subStepCount`。子步数越高，碰撞穿透越少，堆叠越稳，高速物体也越不容易穿墙。但代价也很直观：每多一子步，CPU 就要多跑一遍碰撞检测和约束求解。日常游戏里 1 到 4 子步足够流畅；如果你在做多米诺骨牌或者精密机关，8 子步以上才能让玩家"看得舒服"。

另一个容易忽略的平衡点是**连续碰撞检测（CCD）**。`EnableContinuous` 能防止子弹穿墙，但它比离散检测贵不少。如果场景中高速物体不多，关掉它就是立竿见影的省钱操作。同理，静态摩擦和恢复系数的阈值也可以放宽一点——不是每个小碰撞都值得精打细算。

我的建议是：**先保睡眠、再调子步、最后才开 CCD**。睡眠机制几乎是零成本却能砍掉大半计算量；子步数根据玩法需求量力而行；CCD 和复杂的形状细节，只留给真正需要的场景。

## 小结

这一章我们梳理了 Box2DSharp 提速的几条主线。

内存层面，它用 `B2Array`、`B2ArrayPool` 和 `FixedArray` 把 GC 压力压得极低；并行层面，`WorkerCount` 配合 `EnqueueTask`/`FinishTask` 能把求解任务拆到多核上跑；SIMD 则在约束求解时批量处理数据，进一步榨干 CPU 潜力。岛屿管理和图着色是更"聪明"的优化——通过 `Island` 的合并与分裂剔除无关计算，再用 `ConstraintGraph` 的图着色把可并行的约束分组，实现无锁求解。

作为使用者，最实用的建议就三条：**让能睡的物体睡过去**、**子步数别盲目拉高**、**形状尽量简单**。先把这三条做到位，再考虑进阶调优。
