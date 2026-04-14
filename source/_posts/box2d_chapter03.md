---
title: "第3章：刚体系统——Body 的三种类型"
date: 2026-04-14
categories:
  - 书籍
tags:
  - Box2D
description: "Static/Kinematic/Dynamic三种刚体的本质区别与应用场景"
---

## 概述

刚体（Body）是 Box2DSharp 物理世界的核心角色。它不像 Fixture 那样直接接触碰撞几何，也不像 Joint 那样连接两个实体——刚体是这一切的载体。

每个 Body 都有一个类型、一组动力学属性（质量、速度、阻尼），以及在世界中的位置和旋转。理解 Body，就等于理解了物理引擎怎么处理“运动”这件事。这一章我们从源头讲起：Body 有哪几种类型、怎么配置它、以及 Box2DSharp 内部怎么把数据拆分得干净利索。

## 核心概念

### BodyType：三种身份，三种命运

Box2DSharp 只认三种刚体，没有第四种。源码里说得非常直白。

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

这三行英文注释已经把核心差异讲透了。StaticBody 是这个世界的不动基石：质量为零，不受力影响，也不会自己动起来。地面、墙壁、固定平台，统统用它。KinematicBody 比较特殊，质量也是零，但速度由你（用户代码）说了算，物理解算器只负责按这个速度推进位置。电梯、传送带、玩家控制的移动平台，都是它的主场。DynamicBody 才是“真正的物理对象”：有正质量，受力决定速度，完全交给解算器去折腾。箱子、球、角色 ragdoll，都是 DynamicBody。

| 特性 | StaticBody | KinematicBody | DynamicBody |
|---|---|---|---|
| 质量 | 零 | 零 | 正质量 |
| 速度来源 | 无（手动改位置除外） | 用户代码设定 | 力和冲量驱动 |
| 受力响应 | 忽略 | 忽略 | 完全响应 |
| 适用场景 | 地面、墙壁 | 电梯、平台 | 箱子、角色 |

KinematicBody 和 DynamicBody 还有一个关键区别：Kinematic 不会因为碰撞而改变自己的速度。你设了 `v = (3, 0)`，它就一直往右走，撞到什么都不会减速。DynamicBody 则会被碰撞反弹、摩擦阻滞，完全符合牛顿定律。

### BodyDef：造一个 Body 的配置单

创建 Body 之前，你得先填一张表，叫 `BodyDef`。源码里它的字段不少，但逻辑很清晰。

```csharp:src/Body/BodyDef.cs [8:77]
public struct BodyDef
{
    // 刚体类型：Static / Kinematic / Dynamic
    public BodyType Type;

    // 初始位置和旋转
    public Vec2 Position;
    public Rot Rotation;

    // 初始线速度和角速度
    public Vec2 LinearVelocity;
    public float AngularVelocity;

    // 阻尼：越大速度衰减越快
    public float LinearDamping;
    public float AngularDamping;

    // 重力缩放，0 表示不受重力
    public float GravityScale;

    // 睡眠速度阈值，低于此值可能进入休眠
    public float SleepThreshold;

    // 应用自定义数据
    public object UserData;

    // 是否允许睡眠、初始是否唤醒
    public bool EnableSleep;
    public bool IsAwake;

    // 是否固定旋转（角色常用）
    public bool FixedRotation;

    // 子弹模式：开启连续碰撞检测
    public bool IsBullet;

    // 是否启用
    public bool IsEnabled;

    // 是否根据形状自动计算质量
    public bool AutomaticMass;

    // 允许快速旋转（车轮之类）
    public bool AllowFastRotation;

    // 内部校验值，不要动
    public int InternalValue;
}
```

初始化 BodyDef 千万别用 `new BodyDef()`，因为那样 `InternalValue` 不会被正确填充。一定要用 `DefaultBodyDef()`：

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

这里有几个参数特别值得留意：

- `Position`：注释里特别提醒，创建时就放在目标位置。如果你先创在原点，再手动挪过去，开销几乎翻倍，尤其是在已经附加了 Shape 之后。
- `GravityScale`：默认是 1，设为 0 可以让刚体“飘起来”。这在制作飞行道具或零重力区域时非常方便。
- `FixedRotation`：很多平台游戏的主角都会开这个，防止角色因为碰撞而莫名其妙地翻滚。
- `IsBullet`：高速小物体（比如子弹）建议开启，它会和 Dynamic/Kinematic 体做连续碰撞检测（CCD），避免“穿墙”。但别滥用，会影响性能，也可能干扰关节约束。

### BodySim 与 BodyState：为什么数据要拆两半？

如果你打开 `Body.cs` 的源码，会发现 Box2DSharp 并没有把所有数据塞进一个巨大的类里。和动力学相关的核心数据被拆成了两个结构：`BodySim` 和 `BodyState`。这种拆分不是面向对象庸癖，而是为了服务物理引擎的两个阶段：**积分**和**约束求解**。

`BodySim` 存放的是“仿真数据”——质量、力、阻尼、变换、质心位置，这些都需要在积分阶段被读取和修改。

```csharp:src/Body/BodySim.cs [8:50]
public class BodySim
{
    // 刚体原点的位置、角度
    public Transform Transform;

    // 世界坐标系中的质心坐标
    public Vec2 Center;

    // 上一次的角度和质心，用于 TOI 检查
    public Rot Rotation0;
    public Vec2 Center0;

    // 相对于刚体原点的质心本地坐标
    public Vec2 LocalCenter;

    // 受力和力矩
    public Vec2 Force;
    public float Torque;

    // 质量与转动惯量
    public float Mass;
    public float InvMass;
    public float Inertia;
    public float InvInertia;
    // ...
}
```

而 `BodyState` 是一个紧凑的 32 字节结构体，用 `StructLayout.Explicit` 做了精确内存布局。它放的是速度、微小位移增量，以及状态标记——这些正是约束求解器（Solver）在每一帧反复读写的数据。

```csharp:src/Body/BodyState.cs [5:25]
[StructLayout(LayoutKind.Explicit, Pack = 4, Size = 32)]
public struct BodyState
{
    [FieldOffset(0)]
    public Vec2 LinearVelocity; // 8

    [FieldOffset(8)]
    public float AngularVelocity; // 4

    [FieldOffset(12)]
    public int Flags; // 4

    // 使用 delta 位置减少远原点时的浮点误差
    [FieldOffset(16)]
    public Vec2 DeltaPosition; // 8

    [FieldOffset(24)]
    public Rot DeltaRotation; // 8
}
```

为什么要拆？因为 Solver 对内存布局和访问模式极其敏感。把求解阶段的高频数据打包成一个连续、等大的 `BodyState` 数组，能最大化缓存命中率，也方便并行计算。而 `BodySim` 里的数据（比如质量和阻尼）在求解阶段大多是只读的，不需要跟着 Solver 一起抖动。

更进一步看，`BodyState` 里用 `DeltaPosition` 和 `DeltaRotation` 而不是存绝对位置，也是一门学问。这样一来，StaticBody 在 Solver 里可以直接贡献零 delta，不用读它的 Transform，减少内存带宽；二来能缓解大坐标场景下的浮点精度问题。

说白了，`BodySim` 回答的是“这物体有多重、在哪、受什么力”；`BodyState` 回答的是“这一帧它速度变了多少、位置挪了多少”。两者分工明确，既让代码结构清晰，也为后续的性能优化（比如多线程求解）铺好了路。

## 运行时 API：力、速度与状态控制

刚体创建之后，真正的乐趣才开始。Box2DSharp 提供了一套非常直观的运行时 API，让你能直接操控刚体的运动状态。我们按「力与冲量」、「速度控制」、「睡眠唤醒」、「生命周期」四个维度展开。

### ApplyForce：持续施加力

`ApplyForce` 用于模拟**持续作用**的力，比如风、推力、磁力。它会在多个时间步内累积效果。如果你想让力作用在刚体的某个特定点上（会产生力矩），就用完整版；如果想直接作用于质心，用 `ApplyForceToCenter`。

来看源码里的实现：

```csharp:src/Body/Body.cs [822:855]
        public static void ApplyForce(BodyId bodyId, Vec2 force, Vec2 point, bool wake)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);

            if (wake && body.SetIndex >= SolverSetType.FirstSleepingSet)
            {
                WakeBody(world, body);
            }

            if (body.SetIndex == SolverSetType.AwakeSet)
            {
                BodySim bodySim = GetBodySim(world, body);
                bodySim.Force = B2Math.Add(bodySim.Force, force);
                bodySim.Torque += B2Math.Cross(B2Math.Sub(point, bodySim.Center), force);
            }
        }

        public static void ApplyForceToCenter(BodyId bodyId, Vec2 force, bool wake)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);

            if (wake && body.SetIndex >= SolverSetType.FirstSleepingSet)
            {
                WakeBody(world, body);
            }

            if (body.SetIndex == SolverSetType.AwakeSet)
            {
                BodySim bodySim = GetBodySim(world, body);
                bodySim.Force = B2Math.Add(bodySim.Force, force);
            }
        }
```

注意两个细节：

- 最后一个参数 `wake` 如果为 `true`，且刚体正在睡觉，引擎会先把它**唤醒**。这对碰撞触发的力非常重要。
- 力只会被累加到 `BodySim` 中；真正的速度变化发生在**下一步解算**时。所以你调用 `ApplyForce` 后不会立刻看到位移。

### ApplyImpulse：瞬时冲量

如果说 `ApplyForce` 是「推」，那 `ApplyImpulse` 就是「撞」——它直接改变速度，效果**立竿见影**。非常适合模拟爆炸、子弹撞击、跳跃等一次性事件。

```csharp:src/Body/Body.cs [874:915]
        public static void ApplyLinearImpulse(BodyId bodyId, Vec2 impulse, Vec2 point, bool wake)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);

            if (wake && body.SetIndex >= SolverSetType.FirstSleepingSet)
            {
                WakeBody(world, body);
            }

            if (body.SetIndex == SolverSetType.AwakeSet)
            {
                int localIndex = body.LocalIndex;
                SolverSet set = world.SolverSetArray[SolverSetType.AwakeSet];
                Debug.Assert(0 <= localIndex && localIndex < set.States.Count);
                ref BodyState state = ref set.States[localIndex];
                BodySim bodySim = set.Sims[localIndex];
                state.LinearVelocity = B2Math.MulAdd(state.LinearVelocity, bodySim.InvMass, impulse);
                state.AngularVelocity += bodySim.InvInertia * B2Math.Cross(B2Math.Sub(point, bodySim.Center), impulse);
            }
        }

        public static void ApplyLinearImpulseToCenter(BodyId bodyId, Vec2 impulse, bool wake)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);

            if (wake && body.SetIndex >= SolverSetType.FirstSleepingSet)
            {
                WakeBody(world, body);
            }

            if (body.SetIndex == SolverSetType.AwakeSet)
            {
                int localIndex = body.LocalIndex;
                SolverSet set = world.SolverSetArray[SolverSetType.AwakeSet];
                Debug.Assert(0 <= localIndex && localIndex < set.States.Count);
                ref BodyState state = ref set.States[localIndex];
                BodySim bodySim = set.Sims[localIndex];
                state.LinearVelocity = B2Math.MulAdd(state.LinearVelocity, bodySim.InvMass, impulse);
            }
        }
```

你会发现冲量直接修改了 `BodyState.LinearVelocity`，而不是像力那样写到 `BodySim.Force` 里。这就是「瞬时」和「持续」的本质区别。

### ApplyTorque 与角冲量

旋转控制同样分持续力矩和瞬时角冲量：

```csharp:src/Body/Body.cs [857:872]
        public static void ApplyTorque(BodyId bodyId, float torque, bool wake)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);

            if (wake && body.SetIndex >= SolverSetType.FirstSleepingSet)
            {
                WakeBody(world, body);
            }

            if (body.SetIndex == SolverSetType.AwakeSet)
            {
                BodySim bodySim = GetBodySim(world, body);
                bodySim.Torque += torque;
            }
        }
```

```csharp:src/Body/Body.cs [917:942]
        public static void ApplyAngularImpulse(BodyId bodyId, float impulse, bool wake)
        {
            Debug.Assert(bodyId.IsValid());
            World world = World.GetWorld(bodyId.World0);

            int id = bodyId.Index1 - 1;
            world.BodyArray.CheckIndex(id);
            Body body = world.BodyArray[id];
            Debug.Assert(body.Revision == bodyId.Revision);

            if (wake && body.SetIndex >= SolverSetType.FirstSleepingSet)
            {
                // this will not invalidate body pointer
                WakeBody(world, body);
            }

            if (body.SetIndex == SolverSetType.AwakeSet)
            {
                int localIndex = body.LocalIndex;
                SolverSet set = world.SolverSetArray[SolverSetType.AwakeSet];
                Debug.Assert(0 <= localIndex && localIndex < set.States.Count);
                ref BodyState state = ref set.States[localIndex];
                BodySim sim = set.Sims[localIndex];
                state.AngularVelocity += sim.InvInertia * impulse;
            }
        }
```

### 速度直接控制

有时候你不想通过力来间接调速，而是直接「设定」速度。比如平台角色、传送门、或者完全由脚本驱动的运动体。Box2DSharp 提供了 `SetLinearVelocity` 和 `SetAngularVelocity`：

```csharp:src/Body/Body.cs [788:820]
        public static void SetLinearVelocity(BodyId bodyId, Vec2 linearVelocity)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);

            if (B2Math.LengthSquared(linearVelocity) > 0.0f)
            {
                WakeBody(world, body);
            }

            ref var state = ref GetBodyState(world, body, out var success);
            if (success)
            {
                state.LinearVelocity = linearVelocity;
            }
        }

        public static void SetAngularVelocity(BodyId bodyId, float angularVelocity)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);

            if (angularVelocity != 0.0f)
            {
                WakeBody(world, body);
            }

            ref var state = ref GetBodyState(world, body, out var success);
            if (success)
            {
                state.AngularVelocity = angularVelocity;
            }
        }
```

注意引擎的一个贴心设计：一旦设置的速度非零，它会**自动唤醒**刚体。所以你不用手动调用 `SetAwake`。

### 睡眠与唤醒

睡眠机制是 Box2D 的性能杀手锏。当刚体运动极其微弱时，引擎会把它放入睡眠岛，跳过它的积分和碰撞检测，从而大幅降低 CPU 消耗。

你可以通过以下 API 控制睡眠：

```csharp:src/Body/Body.cs [1366:1395]
        public static bool IsAwake(BodyId bodyId)
        {
            World world = World.GetWorld(bodyId.World0);
            Body body = GetBodyFullId(world, bodyId);
            return body.SetIndex == SolverSetType.AwakeSet;
        }

        public static void SetAwake(BodyId bodyId, bool awake)
        {
            var world = World.GetWorldLocked(bodyId.World0);

            Body body = GetBodyFullId(world, bodyId);

            if (awake && body.SetIndex >= SolverSetType.FirstSleepingSet)
            {
                WakeBody(world, body);
            }
            else if (awake == false && body.SetIndex == SolverSetType.AwakeSet)
            {
                world.IslandArray.CheckIndex(body.IslandId);
                Island island = world.IslandArray[body.IslandId];
                if (island.ConstraintRemoveCount > 0)
               强制入睡，Box2DSharp 会先检查岛屿约束状态。如果岛屿最近有约束被移除（`ConstraintRemoveCount > 0`），需要先执行岛屿分裂，确保睡眠的数据结构一致性。
```

睡眠阈值 `SleepThreshold` 控制多容易入睡：

```csharp:src/Body/BodyDef.cs [43:53]
        /// Sleep velocity threshold, default is 0.05 meter per second
        public float SleepThreshold;

        /// Set this flag to false if this body should never fall asleep.
        public bool EnableSleep;

        /// Is this body initially awake or sleeping?
        public bool IsAwake;
```

默认值是 `0.05f * Core.LengthUnitsPerMeter`。如果你想做一个永远保持活跃的角色，可以把 `EnableSleep` 设为 `false`。测试场景里有一个长摆锤，演示了如何动态调整睡眠阈值：

```csharp:test/Testbed.TestSamples/Bodies/Sleep.cs [24:30]
        float sleepVelocity = Body.GetSleepThreshold(PendulumId);
        if (ImGui.SliderFloat("sleep velocity", ref sleepVelocity, 0.0f, 1.0f, "%.2f"))
        {
            Body.SetSleepThreshold(PendulumId, sleepVelocity);
            Body.SetAwake(PendulumId, true);
        }
```

### 生命周期：创建、销毁、启用与禁用

刚体的完整生命周期包含四个关键操作。

**创建**：`Body.CreateBody` 接收 `WorldId` 和 `BodyDef`，返回 `BodyId`。它负责把刚体放入正确的解算集（StaticSet、AwakeSet 或某个睡眠岛），并初始化所有质量、阻尼、速度等数据。

```csharp:src/Body/Body.cs [229:367]
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

            bool isAwake = (def.IsAwake || def.EnableSleep == false) && def.IsEnabled;

            // determine the solver set
            int setId;
            if (def.IsEnabled == false)
            {
                // any body type can be disabled
                setId = SolverSetType.DisabledSet;
            }
            else if (def.Type == BodyType.StaticBody)
            {
                setId = SolverSetType.StaticSet;
            }
            else if (isAwake == true)
            {
                setId = SolverSetType.AwakeSet;
            }
            else
            {
                // new set for a sleeping body in its own island
                setId = world.SolverSetIdPool.AllocId();
                if (setId == world.SolverSetArray.Count)
                {
                    world.SolverSetArray.Push(new SolverSet());
                }
                else
                {
                    Debug.Assert(world.SolverSetArray[setId].SetIndex == Core.NullIndex);
                }

                world.SolverSetArray[setId].SetIndex = setId;
            }

            // ... 初始化 BodySim 和 BodyState ...
```

**销毁**：`Body.DestroyBody` 会级联销毁所有关联的碰撞、形状、链和关节，并从解算集中移除数据。注意销毁时会主动唤醒与其接触的其他刚体，避免「悬浮」的物理对象。

```csharp:src/Body/Body.cs [385:472]
        public static void DestroyBody(BodyId bodyId)
        {
            var world = World.GetWorldLocked(bodyId.World0);

            Body body = GetBodyFullId(world, bodyId);

            // Wake bodies attached to this body, even if this body is static.
            bool wakeBodies = true;

            // Destroy the attached joints
            int edgeKey = body.HeadJointKey;
            while (edgeKey != Core.NullIndex)
            {
                int jointId = edgeKey >> 1;
                int edgeIndex = edgeKey & 1;

                Joint joint = world.JointArray[jointId];
                edgeKey = joint.Edges[edgeIndex].NextKey;

                // Careful because this modifies the list being traversed
                Joint.DestroyJointInternal(world, joint, wakeBodies);
            }

            // Destroy all contacts attached to this body.
            DestroyBodyContacts(world, body, wakeBodies);

            // Destroy the attached shapes and their broad-phase proxies.
            int shapeId = body.HeadShapeId;
            while (shapeId != Core.NullIndex)
            {
                Shape shape = world.ShapeArray[shapeId];

                Shape.DestroyShapeProxy(shape, world.BroadPhase);

                // Return shape to free list.
                world.ShapeIdPool.FreeId(shapeId);
                shape.Id = Core.NullIndex;

                shapeId = shape.NextShapeId;
            }

            // ... 移除 BodySim / BodyState，回收 ID ...
```

**启用 / 禁用**：`Body.Enable` 和 `Body.Disable` 可以让你临时让刚体脱离物理模拟，既不移动也不碰撞。禁用时会从宽相位中移除形状代理，并从岛屿中解离；启用时则反向恢复。

```csharp:src/Body/Body.cs [1439:1507]
        public static void Disable(BodyId bodyId)
        {
            var world = World.GetWorldLocked(bodyId.World0);

            Body body = GetBodyFullId(world, bodyId);
            if (body.SetIndex == SolverSetType.DisabledSet)
            {
                return;
            }

            // Destroy contacts and wake bodies touching this body. This avoid floating bodies.
            // This is necessary even for static bodies.
            bool wakeBodies = true;
            DestroyBodyContacts(world, body, wakeBodies);

            // Disabled bodies are not in an island.
            RemoveBodyFromIsland(world, body);

            // Remove shapes from broad-phase
            int shapeId = body.HeadShapeId;
            while (shapeId != Core.NullIndex)
            {
                Shape shape = world.ShapeArray[shapeId];
                shapeId = shape.NextShapeId;
                Shape.DestroyShapeProxy(shape, world.BroadPhase);
            }

            // Transfer simulation data to disabled set
            // ...
```

```csharp:src/Body/Body.cs [1508:1598]
        public static void Enable(BodyId bodyId)
        {
            var world = World.GetWorldLocked(bodyId.World0);

            Body body = GetBodyFullId(world, bodyId);
            if (body.SetIndex != SolverSetType.DisabledSet)
            {
                return;
            }

            SolverSet disabledSet = world.SolverSetArray[SolverSetType.DisabledSet];
            int setId = body.Type == BodyType.StaticBody ? SolverSetType.StaticSet : SolverSetType.AwakeSet;
            SolverSet targetSet = world.SolverSetArray[setId];

            SolverSet.TransferBody(world, targetSet, disabledSet, body);

            // Add shapes to broad-phase
            // ...
```

> 小提示：禁用和改变类型（`SetType`）的内部实现非常复杂，因为它们要处理关节转移、岛屿分裂/合并、图着色重建等一连串连锁反应。日常使用你只需调用高层 API 即可，底层数据一致性由引擎自动保证。

## 设计思考

在早期的代码里，一个 Body 对象往往"大包大揽"，把所有数据都塞在一起。Box2DSharp 却选择了一条更细粒度的路：把 Body 拆成了 `BodyArray`（管理索引与关系）、`BodySim`（仿真数据）和 `BodyState`（求解状态）三块。为什么要分得这么清？

核心原因是**"谁需要什么数据"并不一样**。

`Body` 类本身只保留"组织细节"——比如 `SetIndex`、`LocalIndex`、关联的 shape 和 joint 链表、岛链信息等。这些数据用来管理生命周期和图结构，但**解算器根本用不到**。解算器只关心位置和速度，所以 Erin Catto 干脆把注释写得很直白："Body organizational details that are not used in the solver"。

真正进解算器的是 `BodySim` 和 `BodyState`。`BodySim` 是"物理属性"：Transform、Center、Force、Torque、Mass、InvMass、Inertia 等。它会被整整齐齐地塞进 `SolverSet.Sims` 的连续数组里。而 `BodyState` 更"临时"——它只存在于活跃（Awake）的刚体上，保存 `LinearVelocity`、`AngularVelocity`、Flags 和本帧的 delta 值，大小被严格限定在 32 字节，用 `StructLayout(LayoutKind.Explicit)` 做了精确内存布局。


这种拆分带来了三个直接好处：

**第一，内存局部性。** 解算器迭代的是 `SolverSet.Sims` 和 `SolverSet.States`，它们都是连续存储的数组。解算器在循环里不会被无关字段"带偏"缓存行，命中率自然就上去了。静态体甚至有状态但没 `BodyState`，进一步节省了无效内存。

**第二，状态迁移方便。** 刚体从"活跃"变"睡眠"、从"启用"变"禁用"时，Box2DSharp 只需要把它在 `SolverSet` 之间搬来搬去，比如从活跃集搬到某个睡眠岛集。`Body` 对象本身稳坐在 `BodyArray` 里一动不动，只改 `SetIndex` 和 `LocalIndex` 两个整数就行。如果所有数据都绑在一个 Body 对象上，迁移的代价会高得多。

**第三，按需读写。** 游戏逻辑层（比如 GetBodyTransform）读 `BodySim` 就够了；约束求解器才需要读写 `BodyState`。`GetBodyState` 的代码甚至加了断言：只有 `AwakeSet` 里的刚体才能拿到真正的状态引用，其余情况返回 `BodyState.Null`。


所以，这不仅是"代码洁癖"，而是一种数据导向的设计（Data-Oriented Design）。它的目标很明确：让热路径上的代码更快、更省、更可预测。

## 小结

- Box2DSharp 把 Body 拆成 `Body`（管理/关系）、`BodySim`（物理属性）、`BodyState`（求解状态）三层，解耦了不同子系统的数据需求。
- `BodySim` 存储在 `SolverSet.Sims` 连续数组中，`BodyState` 只存在于活跃集的紧凑数组里，最大化缓存友好性。
- 刚体在活跃、睡眠、禁用之间切换时，只需在 `SolverSet` 间迁移 `BodySim`/`BodyState`，`Body` 本身固定不动，开销极低。
- `BodyState` 被精确布局为 32 字节结构体，专门服务于约束求解器的高速迭代。
- 写物理引擎时，考虑"谁会读这段数据、多频繁、需不需要一起访问"，往往比面向对象封装更能决定性能上限。
