---
title: "第1章：快速入门——从零开始的物理世界"
date: 2026-04-14
categories:
  - 代码库
tags:
  - Box2D
description: "Box2DSharp 四步工作流、Def模式、ID系统、刚体类型、Step函数与ShapeDef属性"
---

## 概述

Box2DSharp 是 Erin Catto 经典物理引擎 Box2D v3 的 C# 移植版。它完整保留了原版的高性能架构——包括基于 Island 的睡眠机制、约束图着色多线程求解，以及连续碰撞检测——同时全部使用托管代码实现，方便你在 .NET 或 Unity 环境中直接引用。

如果你之前用过其他物理引擎，Box2DSharp 的设计哲学非常简洁：先创建一个 `World`，往里面丢 `Body`，给 Body 挂上 `Shape`，然后每帧调用 `Step` 让时间前进。没有隐式的全局状态，也没有复杂的对象树，四个步骤就能跑起一个物理模拟。接下来我们就按这个顺序，从一个最简单的"方块落在地面上"的例子开始。

## 核心概念

### 核心工作流：CreateWorld / CreateBody / CreateShape / Step

一切 Box2DSharp 程序都遵循同一个四步模板。理解了它，你就理解了这座大厦的骨架。

**第一步：创建世界**

```csharp:src/WorldDef.cs [63:83]
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
```

框架强烈建议你从 `DefaultWorldDef()` 起步。它默认给出了地表重力 `-10 m/s²`、合理的接触刚度与休眠策略，再加一个工作线程。拿到这个配置后，调用 `World.CreateWorld` 即可得到一个 `WorldId`。

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

            // pools
            world.BodyIdPool = new();
            world.BodyArray = new();
            world.SolverSetArray = new();

            // add empty static, active, and disabled body sets
            world.SolverSetIdPool = new();

            // static set
            SolverSet set = new();
            set.SetIndex = world.SolverSetIdPool.AllocId();
            world.SolverSetArray.Push(set);
            Debug.Assert(world.SolverSetArray[SolverSetType.StaticSet].SetIndex == SolverSetType.StaticSet);

            // disabled set
            set = new();
            set.SetIndex = world.SolverSetIdPool.AllocId();
            world.SolverSetArray.Push(set);
            Debug.Assert(world.SolverSetArray[SolverSetType.DisabledSet].SetIndex == SolverSetType.DisabledSet);

            // awake set
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

            // add one to worldId so that 0 represents a null b2WorldId
            return new WorldId((ushort)(worldId + 1), world.Revision);
        }
```

`CreateWorld` 会在内部帮你建好静态集、活跃集、禁用集等解算集，分配好各种对象池。你不需要关心这些细节，只要记住它返回的 `WorldId` 是你后续所有操作的大门钥匙。

**第二步：创建刚体**

```csharp:src/Body/BodyDef.cs [80:94]
        public static BodyDef DefaultBodyDef()
        {
            return new BodyDef
            {
                Type = BodyType.StaticBody,
                Rotation = Rot.Identity,
                SleepThreshold = 0.05f * Core.LengthUnitsPerMeter,
                GravityScale = 1.0f,
                EnableSleep = true,
                IsAwake = true,
                IsEnabled = true,
                AutomaticMass = true,
                InternalValue = Core.SecretCookie
            };
        }
```

与 `WorldDef` 一样，也要从 `DefaultBodyDef()` 开始。它默认把刚体设为 `StaticBody`，想象成地面或墙壁。如果你要让它受力下落，只需把 `Type` 改成 `DynamicBody`，再指定一个初始位置，然后调用 `Body.CreateBody`。

```csharp:src/Body/Body.cs [229:248]
        public static BodyId CreateBody(WorldId worldId, in BodyDef def)
        {
            def.CheckDef();
            Debug.Assert(B2Math.Vec2_IsValid(def.Position));
            Debug.Assert(B2Math.Rot_IsValid(def.Rotation));
            Debug.Assert(B2Math.Vec2_IsValid(def.LinearVelocity));
            Debug.Assert(B2Math.IsValid(def.AngularVelocity));
            Debug.Assert(B2Math.IsValid(def.LinearDamping) && def.LinearDamping >= 0.0f);
            Debug.Assert(B2Math.IsValid(def.AngularDamping) && def.AngularDamping >= 0.0f);
            Debug.Assert(B2Math.IsValid(def.SleepThreshold) && def.SleepThreshold >= 0.0f);
            Debug.Assert(B2Math.IsValid(def.GravityScale));

            World world = World.GetWorldFromId(worldId);
            Debug.Assert(world.Locked == false);

            if (world.Locked)
            {
                return BodyId.NullId;
            }
```

`CreateBody` 会在世界中创建刚体记录，并根据类型把它放进正确的解算集。它返回的 `BodyId` 同样是一个不透明句柄，不要直接拿它去数组下标索引。

**第三步：创建形状**

```csharp:src/Collision/Geometry/ShapeDef.cs [57:68]
        public static ShapeDef DefaultShapeDef()
        {
            return new ShapeDef
            {
                Friction = 0.6f,
                Density = 1.0f,
                Filter = Filter.DefaultFilter(),
                EnableSensorEvents = true,
                EnableContactEvents = true,
                InternalValue = Core.SecretCookie
            };
        }
```

形状定义决定了一个物体的"外表"和"材质"：密度决定质量，摩擦决定滑动阻力，弹性（Restitution）决定碰撞后的反弹程度。默认值通常已经够用。创建时，只需要把几何体和 `ShapeDef` 一起交给形状创建函数。

```csharp:src/Collision/Geometry/Shape.cs [225:248]
        public static ShapeId CreateCircleShape(BodyId bodyId, ShapeDef def, Circle circle)

        {
            return CreateShape(bodyId, def, circle, ShapeType.CircleShape);
        }

        public static ShapeId CreateCapsuleShape(BodyId bodyId, ShapeDef def, Capsule capsule)
        {
            var lengthSqr = B2Math.DistanceSquared(capsule.Center1, capsule.Center2);
            if (lengthSqr <= Core.LinearSlop * Core.LinearSlop)
            {
                Circle circle = new(B2Math.Lerp(capsule.Center1, capsule.Center2, 0.5f), (float)capsule.Radius);
                return CreateShape(bodyId, def, circle, ShapeType.CircleShape);
            }

            return CreateShape(bodyId, def, capsule, ShapeType.CapsuleShape);
        }

        public static ShapeId CreatePolygonShape(BodyId bodyId, ShapeDef def, Polygon polygon)

        {
            Debug.Assert(B2Math.IsValid(polygon.Radius) && polygon.Radius >= 0.0f);
            return CreateShape(bodyId, def, polygon, ShapeType.PolygonShape);
        }
```

Box2DSharp 提供四组常用几何辅助函数：`MakeBox`、`MakePolygon`、`MakeRoundedBox`、`MakeOffsetBox`，以及直接使用 `Circle`、`Capsule`、`Segment` 结构。对于矩形，最常用的是 `Geometry.MakeBox(hx, hy)`，传入半边长即可。

**第四步：步进模拟**

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

`Step` 接收两个参数：单帧物理时间 `timeStep`（通常填 `1/60f`）和子步数 `subStepCount`。子步数越大，碰撞求解越稳定，但计算量也越大。典型游戏项目中，`subStepCount` 取 4 是比较稳妥的起点。

### WorldId / BodyId / ShapeId：只做句柄，不暴露内部

Box2DSharp 内部使用密集的数组池来存储数据，但对外只暴露轻量的 ID 结构。

```csharp:src/Common/WorldId.cs [8:24]
    public struct WorldId : IEquatable<WorldId>
    {
        public ushort Index1;

        public ushort Revision;

        public static WorldId NullId = new();
```

```csharp:src/Common/BodyId.cs [11:23]
    [StructLayout(LayoutKind.Explicit)]
    public struct BodyId : IEquatable<BodyId>
    {
        [FieldOffset(0)]
        public readonly ulong Value;

        [FieldOffset(0)]
        public readonly int Index1;

        [FieldOffset(4)]
        public readonly ushort World0;

        [FieldOffset(6)]
        public readonly ushort Revision;
```

```csharp:src/Common/ShapeId.cs [8:20]
    public struct ShapeId : IEquatable<ShapeId>
    {
        public int Index1;

        public ushort World0;

        public ushort Revision;
```

这三个 ID 的共同点是：都包含一个 `Index1`（从 1 开始计数，0 表示空）、一个 `Revision`（版本号，用于检测过期句柄），以及对于 `BodyId` 和 `ShapeId` 还有一个 `World0`（所属世界的索引）。用句柄而不是直接指针的好处是安全——即使底层对象被销毁并复用了内存，旧句柄也会因为 `Revision` 不匹配而失效，避免野指针崩溃。

### BodyType：三种刚体姿态

```csharp:src/Body/BodyType.cs [6:19]
    public enum BodyType
    {
        /// zero mass, zero velocity, may be manually moved
        StaticBody = 0,

        /// zero mass, velocity set by user, moved by solver
        KinematicBody = 1,

        /// positive mass, velocity determined by forces, moved by solver
        DynamicBody = 2,

        /// number of body types
        BodyTypeCount,
    }
```

- **StaticBody**：静态刚体，质量为零，不会移动，也不会受力影响。地面、墙壁、固定平台都用它。
- **KinematicBody**：运动学刚体，质量为零，但它可以有速度，并且会被求解器移动。适合电梯、移动平台这类"按程序驱动但又能推开动态物体"的对象。
- **DynamicBody**：动态刚体，有正质量，受力、重力、碰撞都会影响它。你的主角、箱子、球体都应该是它。

习惯上我们先创建 `StaticBody` 作为地面，再创建一个 `DynamicBody` 作为下落物体。如果忘记把类型改成 `DynamicBody`，方块就会悬在半空，一动不动。

## SingleBox 示例完整解读

SingleBox 是框架自带的最简示例，一共不到 30 行代码，完整展现了"创建世界 → 创建地面 → 创建动态方块 → 挂载形状"的整个流程。我们先看完整代码：


来逐段拆解。构造函数里先用 `BodyDef.DefaultBodyDef()` 创建地面，地面的 Type 默认就是 StaticBody，所以不需要额外设置。然后用一条 `Segment` 做水平线当成地面，长度是 `66.0f * extent`，摩擦系数设为 0.5。

接着把 `bodyDef.Type` 改成 `DynamicBody`，用 `Geometry.MakeBox(1.0f, 1.0f)` 生成了一个 2x2 的矩形（注意 `MakeBox` 的参数是半边长），并把刚体放在 (0, 4) 的高度。最后 `CreatePolygonShape` 把形状挂上去。


运行这个示例，你会看到一个 2x2 的白色方块从空中落下，砸在绿色地面上，然后静止。这就是 Box2DSharp 的"Hello World"。

## CircleStack 示例：从单物体到多物体

SingleBox 只有一个动态物体，CircleStack 则展示了如何用循环批量创建多个动态刚体。这是完整代码：


这个示例做了几件有趣的事。首先，它在创建场景时调整了摄像机和重力：把重力加大到 -20 m/s²，让下落感更强；同时调用 `World.SetContactTuning` 把接触刚度调低，让圆球堆叠时更有"弹性"。

然后它用 `Circle` 结构定义了一个半径 0.5 的圆，在 for 循环里连续创建了 8 个 `DynamicBody`，每个间隔 1.0 米垂直堆叠。


注意这里的技巧：循环外只创建了一次 `BodyDef` 和 `ShapeDef`，循环内只修改 `bodyDef.Position.Y` 就重复利用。因为定义对象是轻量的临时结构，这样可以减少分配。运行后你会看到 8 个粉色圆球依次落下，在最底层挤压、滚动，最终形成一个不稳定的堆叠塔——这也是测试物理引擎求解器稳定性的经典场景。

## 设计思考

为什么 Box2DSharp 不直接给你返回一个 `Body` 对象，而是塞给你一个 `BodyId`？

我一开始也觉得有点别扭——明明别的物理引擎都是 `world.CreateBody()` 返回一个刚体引用，我拿着它想干啥就干啥。但 Box2D 从 v3 开始彻底拥抱了 Id 系统，这里面的设计考量其实很深。

首先，**Id 系统是一种"安全把手"**。当你看到 `BodyId` 的结构时，会发现它不仅存储了索引，还有一个 `Revision` 版本号。这意味着当一个刚体被销毁后，它的内存槽位可能被回收、分配给另一个刚体，但你手里旧的 `BodyId` 因为版本号对不上，引擎可以立刻识别出这是个"孤儿句柄"。如果你拿的是裸指针或对象引用，程序可能悄无声息地访问到错误数据，甚至直接崩溃。Id 系统把这种"悬空引用"问题从灾难级降级成了可检测、可防御的运行时错误。

其次，**Id 是对引擎内部数据布局的解耦**。Box2DSharp 内部把刚体数据拆成了两部分：`BodyArray` 里放的是组织信息（连接关系、事件标记、Island 链表），而真正的物理仿真数据 `BodySim` 和 `BodyState` 则存放在 `SolverSet` 的连续数组里。引擎可以在你毫不知情的情况下，把一个刚体从"活跃集"搬到"休眠岛"，甚至重新排列内存以获得更好的缓存局部性。如果你直接持有对象引用，这些内部优化就会变得寸步难行。Id 就像一把钥匙——你告诉引擎"我要操作这个刚体"，引擎自己决定这把钥匙最终开到哪个内存地址。

再说说这个 **CreateWorld → CreateBody → CreateShape → Step** 的四步工作流。

这个顺序不是拍脑袋定的，它反映了物理引擎对"实体"的严格分层理解。**World 是时空容器**，负责管理全局规则和多线程任务；**Body 是运动学实体**，定义了质量和位置，但它本身没有"外形"；**Shape 是碰撞体**，给 Body 赋予可触摸的边界。把这三层拆开，而不是让你一句 `world.AddBox()` 搞定一切，带来的好处是极大的组合自由度。同一个 Body 可以挂多个 Shape，形状可以各自拥有不同的密度、摩擦系数、碰撞过滤规则。如果你玩过 Unity，会发现这和"Rigidbody + Collider"的分离如出一辙——Box2D 的设计甚至更早。

最后，`Step` 被单独拎出来作为显式调用，而不是在后台默默运行。这给了你完全的控制权：你可以决定物理帧率、子步数量、甚至把物理更新和渲染更新解耦。对游戏开发来说，这种"显式优于隐式"的哲学意味着更少的不确定性，也更容易调试。

## 小结

- Box2DSharp 的核心工作流是 **CreateWorld → CreateBody → CreateShape → Step**，掌握这四步就能跑通最小物理程序
- 引擎使用 **Id 系统（WorldId / BodyId / ShapeId）** 作为对象句柄，配合版本号机制避免悬空引用，同时允许内部自由优化数据布局
- **Def 模式（WorldDef / BodyDef / ShapeDef）** 是 Box2DSharp 的标志性用法：先取默认值，再覆盖你关心的字段
- 刚体类型 `StaticBody` 代表不动的基础设施，`DynamicBody` 代表受力和运动影响的物体，`KinematicBody` 则可以按程序驱动
- 物理世界不会自己动，你必须在循环里调用 `World.Step()`，并传入时间步长和子步数
