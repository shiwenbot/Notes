---
title: "第2章：物理世界详解——World 与 WorldDef"
date: 2026-04-14
categories:
  - 书籍
tags:
  - Box2D
description: "World容器与指挥中心、WorldDef配置、重力与时间步长"
---

## 概述

World 是整个 Box2DSharp 物理引擎的容器与指挥中心。你可以把它想象成一个独立的物理宇宙：所有的刚体、形状、关节、碰撞接触点都生活在里面，而 World 负责协调它们之间的相互作用。

每一个 World 都有一个唯一的 `WorldId` 作为标识。这个 ID 采用了索引加版本号的机制，既能防止越界访问，又能在世界被销毁重建后识别出已经失效的旧引用。World 本身并不暴露给用户直接使用，而是通过一组静态 API（比如 `CreateWorld`、`Step`、`DestroyWorld`）来操作，这样做的好处是避免了对象引用的生命周期问题，也让多世界共存变得更安全。

在接下来的小节里，我们会从 World 的"出生证明"——`WorldDef` 开始，逐字段拆解它的配置参数，然后深入 `Step` 函数，理解物理模拟是怎么一步一步推进的。

## 核心概念

### WorldDef 详解：物理世界的出生证明

创建物理世界之前，你必须先准备一个 `WorldDef` 结构体。它就像是世界的初始化配置单，每个参数都会深刻影响后续的物理表现。Box2DSharp 提供了一个工厂方法 `DefaultWorldDef()`，强烈建议你从它出发，再按需覆盖特定字段。

```csharp:src/WorldDef.cs [8:84]
namespace Box2DSharp
{
    /// <summary>
    /// World definition used to create a simulation world.
    /// Must be initialized using b2DefaultWorldDef().
    /// @ingroup world
    /// </summary>
    public struct WorldDef
    {
        /// Gravity vector. Box2D has no up-vector defined.
        public Vec2 Gravity;

        /// Restitution velocity threshold, usually in m/s. Collisions above this
        /// speed have restitution applied (will bounce).
        public float RestitutionThreshold;

        /// This parameter controls how fast overlap is resolved and has units of meters per second
        public float ContactPushoutVelocity;

        /// Threshold velocity for hit events. Usually meters per second.
        public float HitEventThreshold;

        /// Contact stiffness. Cycles per second.
        public float ContactHertz;

        /// Contact bounciness. Non-dimensional.
        public float ContactDampingRatio;

        /// Joint stiffness. Cycles per second.
        public float JointHertz;

        /// Joint bounciness. Non-dimensional.
        public float JointDampingRatio;

        /// Maximum linear velocity. Usually meters per second.
        public float MaximumLinearVelocity;

        /// Can bodies go to sleep to improve performance
        public bool EnableSleep;

        /// Enable continuous collision
        public bool EnableContinuous;

        /// Number of workers to use with the provided task system. Box2D performs best when using only
        ///	performance cores and accessing a single L2 cache. Efficiency cores and hyper-threading provide
        ///	little benefit and may even harm performance.
        public int WorkerCount;

        /// Function to spawn tasks
        public EnqueueTaskCallback? EnqueueTask;

        /// Function to finish a task
        public FinishTaskCallback? FinishTask;

        /// User context that is provided to enqueueTask and finishTask
        public object UserTaskContext;

        /// Used internally to detect a valid definition. DO NOT SET.
        public int InternalValue;

        /// Use this to initialize your world definition
        /// @ingroup world
        public static WorldDef DefaultWorldDef()
        {
            WorldDef def = new();
            def.Gravity.X = 0.0f;
            def.Gravity.Y = -10.0f;
            def.HitEventThreshold = 1.0f * Core.LengthUnitsPerMeter;
            def.RestitutionThreshold = 1.0f * Core.LengthUnitsPerMeter;
            def.ContactPushoutVelocity = 3.0f * Core.LengthUnitsPerMeter;
            def.ContactHertz = 30.0f;
            def.ContactDampingRatio = 10.0f;
            def.JointHertz = 60.0f;
            def.JointDampingRatio = 2.0f;

            // 400 meters per second, faster than the speed of sound
            def.MaximumLinearVelocity = 400.0f * Core.LengthUnitsPerMeter;
            def.EnableSleep = true;
            def.EnableContinuous = true;
            def.InternalValue = Core.SecretCookie;
            def.WorkerCount = 1;
            return def;
        }
    }
}
```

下面我们把这些字段整理成一张表格，方便你快速查阅：

| 字段名 | 类型 | 默认值 | 作用 | 调优建议 |
|--------|------|--------|------|----------|
| `Gravity` | `Vec2` | `(0, -10)` | 全局重力向量 | 横版卷轴可设为 `(0, 0)` 或反向；太空游戏可大幅减小 |
| `RestitutionThreshold` | `float` | `1 * LengthUnitsPerMeter` | 弹性生效的最小碰撞速度阈值 | 低于此速度碰撞不会产生反弹；堆叠物体多时适当降低可减少抖动 |
| `ContactPushoutVelocity` | `float` | `3 * LengthUnitsPerMeter` | 穿透重叠的修正速度 | 物体互相穿透严重时适度提高，但太高会导致震荡 |
| `HitEventThreshold` | `float` | `1 * LengthUnitsPerMeter` | 触发 `ContactHitEvent` 的最小相对速度 | 想忽略轻微摩擦碰撞时提高此值 |
| `ContactHertz` | `float` | `30` | 接触约束的刚度频率 | 如同弹簧的"硬度"；数值越高接触越硬，但过高会不稳定 |
| `ContactDampingRatio` | `float` | `10` | 接触约束的阻尼比 | 大于 1 为过阻尼，快速消除震荡；通常保持默认即可 |
| `JointHertz` | `float` | `60` | 关节约束的刚度频率 | 比接触约束更高，因为关节通常需要更稳定的绑定 |
| `JointDampingRatio` | `float` | `2` | 关节约束的阻尼比 | 默认欠阻尼，关节会有轻微弹性感；需要刚性时提高 |
| `MaximumLinearVelocity` | `float` | `400 * LengthUnitsPerMeter` | 刚体最大线速度上限 | 防止隧道效应；超高速场景（如弹丸）可用 `EnableBullet` 单独处理 |
| `EnableSleep` | `bool` | `true` | 是否允许静止刚体进入休眠 | 强烈建议开启，可大幅提升性能；调试时关闭便于观察 |
| `EnableContinuous` | `bool` | `true` | 是否启用连续碰撞检测（CCD） | 高速物体多时必须开启；纯静态场景可关闭节省性能 |
| `WorkerCount` | `int` | `1` | 并行解算的工人线程数 | 根据 CPU 性能核心数设置，超线程和能效核收益不大 |

`InternalValue` 是由 `DefaultWorldDef()` 自动填充的"魔法校验码"，用于在 `CreateWorld` 时检测你是否真的初始化了定义。不要手动修改它。

### Step 机制：物理模拟的心跳

物理引擎不是连续运行的，而是被离散的时间步"推着走"。在 Box2DSharp 中，这个入口就是 `Step` 方法。你通常会在游戏的每一帧调用它，比如：

```csharp:src/World.cs [892:988]
        public static void Step(WorldId worldId, float timeStep, int subStepCount)
        {
            World world = GetWorldFromId(worldId);
            Debug.Assert(world.Locked == false);
            if (world.Locked)
            {
                return;
            }

            // Prepare to capture events
            // Ensure user does not access stale data if there is an early return
            world.BodyMoveEventArray.Clear();
            world.SensorBeginEventArray.Clear();
            world.SensorEndEventArray.Clear();
            world.ContactBeginArray.Clear();
            world.ContactEndArray.Clear();
            world.ContactHitArray.Clear();

            world.Profile = new Profile();

            if (timeStep == 0.0f)
            {
                // todo would be useful to still process collision while paused
                return;
            }

            world.Locked = true;
            world.ActiveTaskCount = 0;
            world.TaskCount = 0;

            var stepTimer = Stopwatch.StartNew();

            // Update collision pairs and create contacts
            // 更新粗检测碰撞配对，创建接触点
            {
                var t1 = Stopwatch.GetTimestamp();
                BroadPhase.UpdateBroadPhasePairs(world);

                world.Profile.Pairs = (float)StopwatchHelper.GetElapsedTime(t1).TotalMilliseconds;
            }

            StepContext context = new();
            context.World = world;
            context.Dt = timeStep;
            context.SubStepCount = Math.Max(1, subStepCount);

            if (timeStep > 0.0f)
            {
                context.InvDt = 1.0f / timeStep;
                context.H = timeStep / context.SubStepCount;
                context.InvH = context.SubStepCount * context.InvDt;
            }
            else
            {
                context.InvDt = 0.0f;
                context.H = 0.0f;
                context.InvH = 0.0f;
            }

            world.InvH = context.InvH;

            // Hertz values get reduced for large time steps
            float contactHertz = Math.Min(world.ContactHertz, 0.25f * context.InvH);
            float jointHertz = Math.Min(world.JointHertz, 0.125f * context.InvH);

            context.ContactSoftness = Softness.MakeSoft(contactHertz, world.ContactDampingRatio, context.H);
            context.StaticSoftness = Softness.MakeSoft(2.0f * contactHertz, world.ContactDampingRatio, context.H);
            context.JointSoftness = Softness.MakeSoft(jointHertz, world.JointDampingRatio, context.H);

            context.RestitutionThreshold = world.RestitutionThreshold;
            context.MaxLinearVelocity = world.MaxLinearVelocity;
            context.EnableWarmStarting = world.EnableWarmStarting;

            // Update contacts
            // 近距离精细碰撞，更新接触点
            {
                var t1 = Stopwatch.GetTimestamp();
                Collide(context);
                world.Profile.Collide = (float)StopwatchHelper.GetElapsedTime(t1).TotalMilliseconds;
            }

            // Integrate velocities, solve velocity constraints, and integrate positions.
            // 速度积分，求解速度约束，然后位置积分
            if (context.Dt > 0.0f)
            {
                var t1 = Stopwatch.GetTimestamp();
                Solver.Solve(world, context);
                world.Profile.Solve = (float)StopwatchHelper.GetElapsedTime(t1).TotalMilliseconds;
            }

            world.Locked = false;

            world.Profile.Step = stepTimer.ElapsedMilliseconds;

            // Make sure all tasks that were started were also finished
            Debug.Assert(world.ActiveTaskCount == 0);
        }
```

`Step` 接收三个参数：`worldId`（哪个世界）、`timeStep`（这一步要走多少时间）、`subStepCount`（子步数量）。它的执行流程可以概括为四个阶段：

1. **事件清空**：把上一帧的刚体移动、传感器触碰、接触碰撞等事件全部清空，防止用户读到旧数据。
2. **BroadPhase 更新**：通过空间加速结构找出可能碰撞的物体对，并创建新的接触点。
3. **Collide 阶段**：对 BroadPhase 筛选出的候选对进行精细碰撞检测，更新接触状态，触发传感器和接触事件。
4. **Solve 阶段**：由求解器进行速度积分、约束求解、位置积分，这是真正的物理运算核心。

`Step` 执行期间会把 `world.Locked` 设为 `true`，这意味着你在 Step 过程中不能从外部增删刚体或形状。如果你尝试这么做，Box2DSharp 会直接忽略或抛出断言错误。

#### 时间步与子步：精度的秘密

这里有一个非常关键的理解点：`timeStep` 和 `subStepCount` 的关系。

`timeStep` 是你告诉引擎"这一帧物理世界应该前进多久"，比如 `1/60f` 秒。`subStepCount` 则是把这段总时间切成多少个小片。引擎实际进行约束求解时，每次只走 `h = timeStep / subStepCount` 这么长的时间。

子步越多，模拟精度越高，但 CPU 消耗也越大。Box2DSharp 根据 `h` 自动调整 `Hertz`（频率），确保在大时间步下依然稳定。如果你设置 `timeStep = 1/30f` 且 `subStepCount = 4`，那么内部实际以 `1/120f` 为粒度迭代了 4 次约束求解。

`StepContext` 就是承载这些运行时计算参数的上下文对象：

```csharp:src/Solver/StepContext.cs [8:60]
    public class StepContext
    {
        /// <summary>
        /// time step
        /// 单步经历的时间
        /// </summary>
        public float Dt;

        /// <summary>
        /// inverse time step (0 if dt == 0).
        /// t的倒数，如果Dt是0，InvDt也记为0
        /// </summary>
        public float InvDt;

        /// <summary>
        /// sub-step
        /// 子步时间分片数，等于单步时间/子步数 (Dt / SubStepCount)
        /// </summary>
        public float H;

        /// <summary>
        /// 子步时间分片数的倒数
        /// </summary>
        public float InvH;

        /// <summary>
        /// 子步数量
        /// </summary>
        public int SubStepCount;

        public Softness JointSoftness;

        public Softness ContactSoftness;

        public Softness StaticSoftness;

        /// <summary>
        /// 弹性阈值
        /// </summary>
        public float RestitutionThreshold;

        /// <summary>
        /// 最大速度
        /// </summary>
        public float MaxLinearVelocity;
```

注意 `InvH` 会同时被存到 `World` 实例上，因为后续计算受力（如爆炸、施加冲量）时需要用它反推冲量大小。`SubStepCount` 至少为 1，所以即使你传了 0，引擎也会自动修正。

#### 求解器内部一瞥

`Solver.Solve` 是 Step 的"重体力活"担当。它首先合并所有活跃的岛屿（Island），然后检测哪些刚体速度过快需要启用连续碰撞，最后通过图着色（Graph Coloring）将约束分组并并行求解。

```csharp:src/Solver/Solver.cs [1113:1140]
        public static void Solve(World world, StepContext stepContext)
        {
            var t1 = Stopwatch.GetTimestamp();

            world.StepIndex += 1;

            // 合并活跃岛屿
            Island.MergeAwakeIslands(world);

            var e = StopwatchHelper.GetElapsedTime(t1);
            world.Profile.BuildIslands = (float)e.TotalMilliseconds;

            SolverSet awakeSet = world.SolverSetArray[SolverSetType.AwakeSet];
            int awakeBodyCount = awakeSet.Sims.Count;
            if (awakeBodyCount == 0)
            {
                // Nothing to simulate, however I must still finish the broad-phase rebuild.
                // 没有需要物理模拟的活跃刚体，但仍然执行粗检测重建
                if (world.UserTreeTask != null)
                {
                    world.UserTreeTask.Wait();
                    world.UserTreeTask = null;
                    world.ActiveTaskCount -= 1;
                }

                world.BroadPhase.ValidateNoEnlarged();
                return;
            }
```

如果没有活跃刚体，`Solve` 会直接返回，但会等待 BroadPhase 的重建任务完成。这个设计保证了即使世界"睡着了"，空间查询树也不会处于不一致状态。

总结一下，使用 `Step` 的黄金法则是：**固定时间步长 + 适当子步数**。不要根据帧率波动随意改变 `timeStep`，否则会导致物理模拟的不确定性和不稳定。如果你的游戏帧率不固定，推荐使用固定时间步的循环（比如每帧累积时间，然后以固定步长多次调用 `Step`）。

### 世界的创建与销毁

CreateWorld 并不返回对象引用，而是返回一个 `WorldId`。内部有一个静态数组 `Worlds` 做池化管理。

```csharp:src/World.cs [303:426]
        public static WorldId CreateWorld(in WorldDef def)
        {
            Debug.Assert(Core.MaxWorlds < ushort.MaxValue, "b2_maxWorlds limit exceeded");
            def.CheckDef();

            var worldId = Core.NullIndex;
            for (var i = 0; i < Core.MaxWorlds; ++i)
            {
                var w = Worlds[i];
                if (w is { InUse: true })
                {
                    continue;
                }

                worldId = i;
                break;
            }

            if (worldId == Core.NullIndex)
            {
                return Box2DSharp.WorldId.NullId;
            }

            Contact.InitializeContactRegisters();

            ref var world = ref Worlds[worldId];
            var revision = world?.Revision ?? 0;

            world = new World();

            world.WorldId = (ushort)worldId;
            world.Revision = revision;
            world.InUse = true;

            world.BroadPhase = new();
            world.ConstraintGraph = new(16);

            world.BodyIdPool = new();
            world.BodyArray = new();
            world.SolverSetArray = new();
            world.SolverSetIdPool = new();

            SolverSet set = new();
            set.SetIndex = world.SolverSetIdPool.AllocId();
            world.SolverSetArray.Push(set);
            Debug.Assert(world.SolverSetArray[SolverSetType.StaticSet].SetIndex == SolverSetType.StaticSet);

            set = new();
            set.SetIndex = world.SolverSetIdPool.AllocId();
            world.SolverSetArray.Push(set);
            Debug.Assert(world.SolverSetArray[SolverSetType.DisabledSet].SetIndex == SolverSetType.DisabledSet);

            set = new();
            set.SetIndex = world.SolverSetIdPool.AllocId();
            world.SolverSetArray.Push(set);
            Debug.Assert(world.SolverSetArray[SolverSetType.AwakeSet].SetIndex == SolverSetType.AwakeSet);

            world.ShapeIdPool = new();
            world.ShapeArray = new(16);
            world.ChainIdPool = new();
            world.ChainArray = new(4);
            world.ContactIdPool = new();
            world.ContactArray = new(16);
            world.JointIdPool = new();
            world.JointArray = new(16);
            world.IslandIdPool = new();
            world.IslandArray = new(8);

            world.BodyMoveEventArray = new(4);
            world.SensorBeginEventArray = new(4);
            world.SensorEndEventArray = new(4);
            world.ContactBeginArray = new(4);
            world.ContactEndArray = new(4);
            world.ContactHitArray = new(4);

            world.StepIndex = 0;
            world.SplitIslandId = Core.NullIndex;
            world.ActiveTaskCount = 0;
            world.TaskCount = 0;
            world.Gravity = def.Gravity;
            world.HitEventThreshold = def.HitEventThreshold;
            world.RestitutionThreshold = def.RestitutionThreshold;
            world.MaxLinearVelocity = def.MaximumLinearVelocity;
            world.ContactPushOutVelocity = def.ContactPushoutVelocity;
            world.ContactHertz = def.ContactHertz;
            world.ContactDampingRatio = def.ContactDampingRatio;
            world.JointHertz = def.JointHertz;
            world.JointDampingRatio = def.JointDampingRatio;
            world.EnableSleep = def.EnableSleep;
            world.Locked = false;
            world.EnableWarmStarting = true;
            world.EnableContinuous = def.EnableContinuous;
            world.UserTreeTask = null;

            world.WorkerCount = Math.Min(def.WorkerCount, Core.MaxWorkers);
            world.EnqueueTaskFcn = def.EnqueueTask ?? DefaultAddTaskFcn;
            world.FinishTaskFcn = def.FinishTask ?? DefaultFinishTaskFcn;
            world.UserTaskContext = def.UserTaskContext;

            world.TaskContextArray = new(world.WorkerCount);
            for (int i = 0; i < world.WorkerCount; ++i)
            {
                world.TaskContextArray.Push(new TaskContext());
            }

            world.WorkerStack = new WorkerStack(def.WorkerCount);

            world.DebugBodySet = new(256);
            world.DebugJointSet = new(256);
            world.DebugContactSet = new(256);

            return new WorldId((ushort)(worldId + 1), world.Revision);
        }
```

注意 `SolverSetArray` 的前三个位置是预分配的：
- `[0] StaticSet`: 静止刚体
- `[1] DisabledSet`: 被禁用的刚体
- `[2] AwakeSet`: 活跃刚体

后续索引才是各个休眠岛屿。这种设计让刚体状态的切换变成数组移动，而不是频繁分配。

DestroyWorld 会清空所有内部数组，但保留 `Revision` 并自增，防止旧 `WorldId` 被误用。

```csharp:src/World.cs [428:504]
        public static void DestroyWorld(WorldId worldId)
        {
            var world = GetWorldFromId(worldId);

            world.DebugBodySet = null!;
            world.DebugJointSet = null!;
            world.DebugContactSet = null!;

            for (int i = 0; i < world.WorkerCount; ++i)
            {
                world.TaskContextArray[i].ContactStateBitSet = null!;
                world.TaskContextArray[i].EnlargedSimBitSet = null!;
                world.TaskContextArray[i].AwakeIslandBitSet = null!;
            }

            world.TaskContextArray.Dispose();
            world.BodyMoveEventArray.Dispose();
            world.SensorBeginEventArray.Dispose();
            world.SensorEndEventArray.Dispose();
            world.ContactBeginArray.Dispose();
            world.ContactEndArray.Dispose();
            world.ContactHitArray.Dispose();

            int chainCapacity = world.ChainArray.Count;
            for (var i = 0; i < chainCapacity; ++i)
            {
                ref var chain = ref world.ChainArray[i];
                if (chain.Id != Core.NullIndex)
                {
                    chain.ShapeIndices = null!;
                }
                else
                {
                    Debug.Assert(chain.ShapeIndices == null);
                }
            }

            world.BodyArray.Dispose();
            world.ShapeArray.Dispose();
            world.ChainArray.Dispose();
            world.ContactArray.Dispose();
            world.JointArray.Dispose();
            world.IslandArray.Dispose();

            var setCapacity = world.SolverSetArray.Count;
            for (var i = 0; i < setCapacity; ++i)
            {
                var set = world.SolverSetArray[i];
                if (set.SetIndex != Core.NullIndex)
                {
                    SolverSet.DestroySolverSet(world, i);
                }
            }

            world.SolverSetArray.Dispose();

            world.ConstraintGraph.DestroyGraph();
            world.BroadPhase.DestroyBroadPhase();

            world.BodyIdPool.DestroyIdPool();
            world.ShapeIdPool.DestroyIdPool();
            world.ChainIdPool.DestroyIdPool();
            world.ContactIdPool.DestroyIdPool();
            world.JointIdPool.DestroyIdPool();
            world.IslandIdPool.DestroyIdPool();
            world.SolverSetIdPool.DestroyIdPool();

            var revision = world.Revision;
            world = new();
            world.WorldId = unchecked((ushort)Core.NullIndex);
            world.Revision = (ushort)(revision + 1);
            Worlds[worldId.Index1 - 1] = world;
        }
```

## 世界查询 API（CastRay / OverlapAABB / CastCircle）

### QueryFilter：先筛后查

所有查询都接受 `QueryFilter` 参数，这让你能按碰撞类别过滤。

```csharp:src/Types/QueryFilter.cs [7:29]
    public struct QueryFilter
    {
        public ulong CategoryBits;
        public ulong MaskBits;

        public QueryFilter(ulong categoryBits, ulong MaskBits)
        {
            CategoryBits = categoryBits;
            MaskBits = maskBits;
        }

        public static QueryFilter DefaultQueryFilter()
        {
            QueryFilter filter = new(0x0001UL, ulong.MaxValue);
            return filter;
        }
    }
```

过滤规则遵循位运算：`(shapeFilter.CategoryBits & queryFilter.MaskBits) != 0 && (shapeFilter.MaskBits & queryFilter.CategoryBits) != 0` 才命中。`DefaultQueryFilter` 默认只设了 `0x0001` 类别位，但掩码全开，所以能搜到任何东西。

### 回调函数签名

射线/形状投射用 `CastResultFcn`，重叠查询用 `OverlapResultFcn`。

```csharp:src/Types/CastResultFcn.cs [1:18]
namespace Box2DSharp
{
    public delegate float CastResultFcn(ShapeId shapeId, Vec2 point, Vec2 normal, float fraction, object context);
}
```

```csharp:src/Types/OverlapResultFcn.cs [1:8]
namespace Box2DSharp
{
    public delegate bool OverlapResultFcn(ShapeId shapeId, object context);
}
```

**CastResultFcn 的返回值决定射线行为：**
- `-1`: 忽略这个结果，继续往前搜。
- `0`: 立刻终止整个射线检测。
- `fraction`: 把射线截断到这个比例，后续只搜更近的（常用于找最近点）。
- `1`: 不截断，继续搜更远的。

**OverlapResultFcn 的返回值：**
- `true`: 继续搜下一个。
- `false`: 终止查询。

### 射线检测 CastRay

```csharp:src/World.cs [2156:2189]
        public static void CastRay(
            WorldId worldId,
            Vec2 origin,
            Vec2 translation,
            QueryFilter filter,
            CastResultFcn fcn,
            object context)
        {
            World world = GetWorldFromId(worldId);
            Debug.Assert(world.Locked == false);
            if (world.Locked)
            {
                return;
            }

            Debug.Assert(B2Math.Vec2_IsValid(origin));
            Debug.Assert(B2Math.Vec2_IsValid(translation));

            RayCastInput input = new(origin, translation, (float)1.0f);

            WorldRayCastContext worldContext = new(world, fcn, filter, 1.0f, context);

            for (int i = 0; i < Core.BodyTypeCount; ++i)
            {
                world.BroadPhase.Trees[i].RayCast(ref input, filter.MaskBits, _rayCastCallback, worldContext);

                if (worldContext.Fraction == 0.0f)
                {
                    return;
                }

                input.MaxFraction = worldContext.Fraction;
            }
        }
```

引擎会依次遍历静态树、运动学树、动态树。如果回调返回了 `fraction < 1.0f`，下一次查询的 `MaxFraction` 就会被更新，从而自动实现"只找更近的"效果。如果回调返回 `0.0f`，就立即退出。

库里还贴心地提供了一个"找最近点"的快捷方法：

```csharp:src/World.cs [2203:2233]
        public static RayResult CastRayClosest(WorldId worldId, Vec2 origin, Vec2 translation, QueryFilter filter)
        {
            RayResult result = B2ObjectPool<RayResult>.Shared.Get();

            World world = GetWorldFromId(worldId);
            Debug.Assert(world.Locked == false);
            if (world.Locked)
            {
                return result;
            }

            Debug.Assert(B2Math.Vec2_IsValid(origin));
            Debug.Assert(B2Math.Vec2_IsValid(translation));

            RayCastInput input = new(origin, translation, (float)1.0f);
            WorldRayCastContext worldContext = new(world, _rayCastClosestFcn, filter, 1.0f, result);

            for (int i = 0; i < Core.BodyTypeCount; ++i)
            {
                world.BroadPhase.Trees[i].RayCast(ref input, filter.MaskBits, _rayCastCallback, worldContext);

                if (worldContext.Fraction == 0.0f)
                {
                    return result;
                }

                input.MaxFraction = worldContext.Fraction;
            }

            return result;
        }
```

`_rayCastClosestFcn` 内部就是返回 `fraction`，自动截断射线，最终 result 里存的就是最近命中点。

### AABB 重叠查询 OverlapAABB

这是最简单高效的查询，只检查 AABB 包围盒相交。

```csharp:src/World.cs [1957:1974]
        public static void OverlapAABB(WorldId worldId, in AABB aabb, in QueryFilter filter, OverlapResultFcn fcn, object context)
        {
            World world = GetWorldFromId(worldId);
            Debug.Assert(world.Locked == false);
            if (world.Locked)
            {
                return;
            }

            Debug.Assert(aabb.IsValid);

            WorldQueryContext worldContext = new(world, fcn, filter, context);

            for (int i = 0; i < Core.BodyTypeCount; ++i)
            {
                world.BroadPhase.Trees[i].Query(aabb, filter.MaskBits, _treeQueryCallback, worldContext);
            }
        }
```

`_treeQueryCallback` 会先过一遍 `QueryFilter` 的位掩码过滤，然后再调用你的 `fcn`。注意这里只查 AABB，**不保证形状真正几何相交**。如果你要精确判断，应该用 `OverlapCircle` 或 `OverlapPolygon`。

### 形状投射 CastCircle

CastCircle 不是"查询某个圆覆盖的区域"，而是"让圆沿着一条路径移动，检测它会先撞上什么"。常用于子弹、技能判定、角色移动预测。

```csharp:src/World.cs [2265:2298]
        public static void CastCircle(WorldId worldId, Circle circle, Transform originTransform, Vec2 translation, QueryFilter filter, CastResultFcn fcn, object context)
        {
            World world = GetWorldFromId(worldId);
            Debug.Assert(world.Locked == false);
            if (world.Locked)
            {
                return;
            }

            Debug.Assert(B2Math.Vec2_IsValid(originTransform.P));
            Debug.Assert(B2Math.Rot_IsValid(originTransform.Q));
            Debug.Assert(B2Math.Vec2_IsValid(translation));

            ShapeCastInput input = new();
            input.Points[0] = B2Math.TransformPoint(originTransform, circle.Center);
            input.Count = 1;
            input.Radius = circle.Radius;
            input.Translation = translation;
            input.MaxFraction = 1.0f;

            WorldRayCastContext worldContext = new(world, fcn, filter, 1.0f, context);

            for (int i = 0; i < Core.BodyTypeCount; ++i)
            {
                world.BroadPhase.Trees[i].ShapeCast(ref input, filter.MaskBits, ShapeCastCallback, worldContext);

                if (worldContext.Fraction == 0.0f)
                {
                    return;
                }

                input.MaxFraction = worldContext.Fraction;
            }
        }
```

注意 `ShapeCastInput` 的构造：
- `Count = 1` 表示这是一个圆（只有中心一个点），配合 `Radius` 就是圆。
- `Count = 2` 是胶囊，`Count >= 3` 是多边形。

`ShapeCastCallback` 内部的工作方式与射线类似：先过滤，再调用 `Shape.ShapeCastShape` 做精确的形状-形状投射计算，然后把结果和 fraction 传给你的回调。

### 查询 API 的使用流程总结

实际使用时，套路很固定：

1. 构造 `QueryFilter`（或直接用 `DefaultQueryFilter`）。
2. 写回调函数，决定是继续搜、终止还是截断。
3. 调用 `CastRay`、`OverlapAABB` 或 `CastCircle`。
4. 从回调的 `context` 里收集结果。

**一个小提示**：不要在 `Locked == true` 的时候调用查询，也就是不要在 `Step` 执行期间调用。如果在 `Step` 里需要查询，建议把查询逻辑放到 `Step` 之前或之后。

## 设计思考

Box2DSharp 的 API 风格第一眼看上去可能有点"反直觉"——你不能直接 `new World()`，而是调用 `World.CreateWorld(def)` 拿回一个 `WorldId`；之后所有操作几乎都是静态方法，比如 `World.Step(worldId, ...)`、`World.CastRay(worldId, ...)`。为什么要这样设计？在我看来，这背后有两个核心考量：**生命周期集中管控**和**数据导向的性能优化**。

先看 World 句柄这件事。`WorldId` 本质上是一个轻量级的索引+版本号（`Index1` + `Revision`），真正的 `World` 对象被存放在内部的静态数组 `World.Worlds[]` 中。这样做的好处是销毁世界时非常干净：`DestroyWorld` 只要把数组槽位重置，并让 `Revision` 自增，那么所有外头残留的 `WorldId` 立刻就能通过 `World_IsValid` 检测出来。如果你手里拿的是对象引用，一旦世界被释放，那个引用就会变成"野指针"，C# 里虽然不会直接崩溃，但逻辑上却可能悄悄访问到已经被回收或复用的状态。用 ID 做句柄，相当于给安全性加了一道闸。

再说 Def 模式。`WorldDef`、`BodyDef`、`ShapeDef`……几乎每创建一个东西都要先配一个 Def。这个模式源自原版 Box2D 的 C 语言设计，但到了 C# 里依然被保留，而且非常实用。Def 结构体把"配置"和"实例"彻底分开：你可以先把所有参数写在 Def 里，甚至作为资源文件序列化，等到真正需要时再一次性传给创建函数。更关键的是，Box2D 内部在创建实例时会按自己的规则安排内存布局（比如把 body 放进连续的 solver set、把 shape 挂到 broad-phase 树），Def 在这里充当了一个清晰的边界，让外部调用者不必关心这些内部细节。

还有一个容易被忽略的点：静态方法 + ID 参数的设计让 Box2DSharp 更容易做**多线程和确定性回滚**。因为 World 的内部状态全都放在可控的数组里，Step 时只需要锁定当前 worldId 对应的数据即可；如果你在做网络同步或者录制回放，序列化一组 ID 和数组状态也比序列化一堆对象引用图要简单得多。这种"数据扁平化"的思路，其实是现代物理引擎和 ECS 架构的共同趋势。

所以下次看到 `World.CreateWorld` 和满屏的 `*Def` 时，不必觉得啰嗦——它们是在用一点仪式性代码，换来内存安全、性能可控和跨平台一致性。对一个需要稳定跑在 60 FPS 甚至更高帧率上的物理引擎来说，这笔买卖非常划算。

## 小结

- `WorldId` 采用索引+版本号的句柄设计，让世界的创建、销毁和有效性校验都变得安全且可追踪，避免了直接持有对象引用的生命周期风险。
- `World.CreateWorld` 和 `World.DestroyWorld` 管理着全局的 `Worlds` 数组，配合 `Revision` 机制，能够有效检测悬空 ID。
- `WorldDef` 延续了 Box2D 的 Def 模式，将外部配置与内部实例化逻辑解耦，便于预配置、序列化和后续扩展。
- `Step` 作为静态方法接收 `WorldId`，既保证了 API 的一致性，也为底层的数据导向设计和多线程并行求解留出了空间。
- 掌握 `WorldDef` 的参数含义、`Step` 的时间步进机制，以及查询和事件 API，是熟练使用 Box2DSharp 的第一步。
