---
title: "【教材】第6章：接触与事件——碰撞发生之后"
date: 2026-04-19
order: 006
categories:
  - 代码库
tags:
  - Box2D
description: "碰撞响应与碰撞通知，接触事件处理机制"
---

## 概述

碰撞检测告诉你两个形状"叠上了"，但游戏真正关心的是碰撞之后怎么办。玩家踩到地面要切换成"着地"状态，子弹命中敌人要扣血并播放特效，两个碎片轻轻擦过和猛烈撞击应该发出不同的声音——这些统统属于**碰撞响应**和**碰撞通知**的范畴。

Box2DSharp 把这部分逻辑封装进了 Contact 系统和事件机制里。引擎不会一有碰撞就打断你的代码，而是先把所有事件收集到缓冲区中，等整个物理步（Time Step）走完再一次性交给你处理。本章就来拆解这套机制是怎么运转的。

## Contact 的生命周期

Box2DSharp 里，两个形状的接触关系用两层对象管理：**Contact** 和 **ContactSim**。

Contact 是"冷数据"（Cold Data），负责维护接触的持久句柄和岛屿连通性。你可以把它想象成一张档案卡，记录着这个接触的基本信息：

```csharp:src/Contact/Contact.cs [9:53]
public class Contact
{
    public int SetIndex = Core.NullIndex;
    public int ColorIndex = Core.NullIndex;
    public int LocalIndex = Core.NullIndex;

    public ContactEdge[] Edges = new ContactEdge[2];

    public int ShapeIdA;
    public int ShapeIdB;

    public int IslandPrev;
    public int IslandNext;
    public int IslandId;
    public int ContactId = Core.NullIndex;

    public ContactFlags Flags;
    public bool IsMarked;
}
```

ContactSim 则是"热数据"（Hot Data），负责物理模拟中的实际计算——流形、摩擦力、恢复系数等：

```csharp:src/Contact/ContactSim.cs [8:50]
public class ContactSim
{
    public int ContactId = Core.NullIndex;
    public int BodyIdA = Core.NullIndex;
    public int BodyIdB = Core.NullIndex;
    public int ShapeIdA = Core.NullIndex;
    public int ShapeIdB = Core.NullIndex;

    public float InvMassA;
    public float InvIA;
    public float InvMassB;
    public float InvIB;

    public Manifold Manifold;
    public float Friction;
    public float Restitution;
    public float TangentSpeed;

    public ContactSimFlags SimFlags;
    public DistanceCache Cache;
}
```

为什么要分两层？因为引擎在不同场景下访问的数据粒度不一样。岛屿求解、接触图遍历这些"管理型"操作只需要 Contact；而约束求解、冲量计算这些"计算型"操作需要 ContactSim。分离之后，内存布局更高效，缓存命中率也更高。

Contact 的一生可以分成三个阶段：**创建、更新、销毁**。

**创建阶段**：当粗检测（BroadPhase）发现两个形状的 AABB 重叠，且碰撞过滤允许碰撞时，`CreateContact` 就会被调用。新建的接触初始状态是"未触碰"的，意味着它还没有实际的接触点。注意，摩擦力和恢复系数在创建时就通过混合规则算好了：

```csharp:src/Contact/Contact.cs [54:64]
public static float MixFriction(float friction1, float friction2)
{
    return (float)Math.Sqrt(friction1 * friction2);
}

public static float MixRestitution(float restitution1, float restitution2)
{
    return restitution1 > restitution2 ? restitution1 : restitution2;
}
```

摩擦力取几何平均，所以只要其中一方是冰面（摩擦力接近 0），混合后几乎就是 0——任何东西都能在冰上滑。恢复系数取最大值，所以只要一方是高弹材料，碰撞就会有弹性。

创建时，Contact 还会通过 `ContactEdge` 双向插入到两个刚体的接触链表中，构成一张接触图：

```csharp:src/Contact/ContactEdge.cs [11:27]
public struct ContactEdge
{
    public int BodyId;
    public int PrevKey;
    public int NextKey;
}
```

**更新阶段**：创建不等于触碰。每一步物理更新时，`UpdateContact` 会重新计算流形，来判断它们是否真的碰上了：

```csharp:src/Contact/Contact.cs [532:628]
public static bool UpdateContact(World world, ContactSim contactSim, Shape shapeA, in Transform transformA, Vec2 centerOffsetA, Shape shapeB, in Transform transformB, Vec2 centerOffsetB)
{
    bool touching;

    if (shapeA.IsSensor || shapeB.IsSensor)
    {
        touching = TestShapeOverlap(ref shapeA, transformA, ref shapeB, transformB, ref contactSim.Cache);
    }
    else
    {
        Manifold oldManifold = contactSim.Manifold;
        ManifoldFcn fcn = ContactRegister.Registers[(int)shapeA.Type, (int)shapeB.Type].Fcn;
        contactSim.Manifold = fcn(shapeA, transformA, shapeB, transformB, ref contactSim.Cache);

        int pointCount = contactSim.Manifold.PointCount;
        touching = pointCount > 0;

        if (touching && world.PreSolveFcn != null && (contactSim.SimFlags & ContactSimFlags.EnablePreSolveEvents) != 0)
        {
            ShapeId shapeIdA = new(shapeA.Id + 1, world.WorldId, shapeA.Revision);
            ShapeId shapeIdB = new(shapeB.Id + 1, world.WorldId, shapeB.Revision);
            touching = world.PreSolveFcn(shapeIdA, shapeIdB, ref contactSim.Manifold, world.PreSolveContext);
            if (touching == false)
            {
                contactSim.Manifold.PointCount = 0;
            }
        }

        if (touching && (shapeA.EnableHitEvents || shapeB.EnableHitEvents))
        {
            contactSim.SimFlags |= ContactSimFlags.EnableHitEvent;
        }
        else
        {
            contactSim.SimFlags &= ~ContactSimFlags.EnableHitEvent;
        }

        for (int i = 0; i < pointCount; ++i)
        {
            ref ManifoldPoint mp2 = ref contactSim.Manifold.Points[i];
            mp2.AnchorA = B2Math.Sub(mp2.AnchorA, centerOffsetA);
            mp2.AnchorB = B2Math.Sub(mp2.AnchorB, centerOffsetB);
            mp2.NormalImpulse = 0.0f;
            mp2.TangentImpulse = 0.0f;
            mp2.MaxNormalImpulse = 0.0f;
            mp2.NormalVelocity = 0.0f;
            mp2.Persisted = false;

            var id2 = mp2.Id;
            for (int j = 0; j < oldManifold.PointCount; ++j)
            {
                ref ManifoldPoint mp1 = ref oldManifold.Points[j];
                if (mp1.Id == id2)
                {
                    mp2.NormalImpulse = mp1.NormalImpulse;
                    mp2.TangentImpulse = mp1.TangentImpulse;
                    mp2.Persisted = true;
                    break;
                }
            }
        }
    }

    if (touching)
    {
        contactSim.SimFlags |= ContactSimFlags.TouchingFlag;
    }
    else
    {
        contactSim.SimFlags &= ~ContactSimFlags.TouchingFlag;
    }

    return touching;
}
```

这段更新逻辑做了三件重要的事：重新计算流形、调用 PreSolveFcn 回调（如果有的话）、把上一帧的冲量复制到新接触点做热启动。热启动让求解器收敛更快，静止接触的物体不会出现不必要的抖动。

**销毁阶段**：接触在多种情况下会被销毁。源码注释已经列得很清楚：

```csharp:src/Contact/Contact.cs [370:377]
// A contact is destroyed when:
// - broad-phase proxies stop overlapping
// - a body is destroyed
// - a body is disabled
// - a body changes type from dynamic to kinematic or static
// - a shape is destroyed
// - contact filtering is modified
// - a shape becomes a sensor (check this!!!)
```

`DestroyContact` 的核心任务是从接触图断开连接、从岛屿或约束图中移除、回收 Id：

```csharp:src/Contact/Contact.cs [378:482]
public static void DestroyContact(World world, Contact contact, bool wakeBodies)
{
    var pairKey = Core.ShapePairKey(contact.ShapeIdA, contact.ShapeIdB);
    world.BroadPhase.PairSet.RemoveKey(pairKey);

    ref ContactEdge edgeA = ref contact.Edges[0];
    ref ContactEdge edgeB = ref contact.Edges[1];

    int bodyIdA = edgeA.BodyId;
    int bodyIdB = edgeB.BodyId;
    Body bodyA = Body.GetBody(world, bodyIdA);
    Body bodyB = Body.GetBody(world, bodyIdB);

    if (edgeA.PrevKey != Core.NullIndex)
    {
        ref readonly Contact prevContact = ref world.ContactArray[(edgeA.PrevKey >> 1)];
        ref ContactEdge prevEdge = ref prevContact.Edges[(edgeA.PrevKey & 1)];
        prevEdge.NextKey = edgeA.NextKey;
    }

    if (edgeA.NextKey != Core.NullIndex)
    {
        ref readonly Contact nextContact = ref world.ContactArray[(edgeA.NextKey >> 1)];
        ref ContactEdge nextEdge = ref nextContact.Edges[(edgeA.NextKey & 1)];
        nextEdge.PrevKey = edgeA.PrevKey;
    }

    int contactId = contact.ContactId;
    int edgeKeyA = (contactId << 1) | 0;
    if (bodyA.HeadContactKey == edgeKeyA)
    {
        bodyA.HeadContactKey = edgeA.NextKey;
    }
    bodyA.ContactCount -= 1;

    if (edgeB.PrevKey != Core.NullIndex)
    {
        ref readonly Contact prevContact = ref world.ContactArray[(edgeB.PrevKey >> 1)];
        ref ContactEdge prevEdge = ref prevContact.Edges[(edgeB.PrevKey & 1)];
        prevEdge.NextKey = edgeB.NextKey;
    }

    if (edgeB.NextKey != Core.NullIndex)
    {
        ref readonly Contact nextContact = ref world.ContactArray[(edgeB.NextKey >> 1)];
        ref ContactEdge nextEdge = ref nextContact.Edges[(edgeB.NextKey & 1)];
        nextEdge.PrevKey = edgeB.PrevKey;
    }

    int edgeKeyB = (contactId << 1) | 1;
    if (bodyB.HeadContactKey == edgeKeyB)
    {
        bodyB.HeadContactKey = edgeB.NextKey;
    }
    bodyB.ContactCount -= 1;

    if (contact.IslandId != Core.NullIndex)
    {
        Island.UnlinkContact(world, contact);
    }

    if (contact.ColorIndex != Core.NullIndex)
    {
        Debug.Assert(contact.SetIndex == SolverSetType.AwakeSet);
        ConstraintGraph.RemoveContactFromGraph(world, bodyIdA, bodyIdB, contact.ColorIndex, contact.LocalIndex);
    }
    else
    {
        Debug.Assert(contact.SetIndex != SolverSetType.AwakeSet || (contact.Flags & ContactFlags.ContactTouchingFlag) == 0 || (contact.Flags & ContactFlags.ContactSensorFlag) != 0);
        var set = world.SolverSetArray[contact.SetIndex];
        var movedIndex = set.Contacts.RemoveContact(contact.LocalIndex);
        if (movedIndex != Core.NullIndex)
        {
            var movedContact = set.Contacts.Data[contact.LocalIndex];
            world.ContactArray[movedContact.ContactId].LocalIndex = contact.LocalIndex;
        }
    }

    contact.ContactId = Core.NullIndex;
    contact.SetIndex = Core.NullIndex;
    contact.ColorIndex = Core.NullIndex;
    contact.LocalIndex = Core.NullIndex;

    world.ContactIdPool.FreeId(contactId);

    if (wakeBodies)
    {
        Body.WakeBody(world, bodyA);
        Body.WakeBody(world, bodyB);
    }
}
```

整个生命周期就是一条清晰的线：粗检测发现重叠 -> `CreateContact` -> 初始"未触碰" -> `UpdateContact` 计算流形 -> 有接触点则标记"触碰" -> 持续更新 -> 不再重叠或其他条件触发 -> `DestroyContact`。

## 三类碰撞事件详解

Box2DSharp 提供了三种碰撞事件，它们都以结构体定义，在时间步结束后通过事件缓冲区一次性返回。

**ContactBeginTouchEvent：开始触碰**

当两个形状从"未触碰"变为"触碰"时触发。结构非常简单，只告诉你谁和谁碰上了：

```csharp:src/Contact/ContactBeginTouchEvent.cs [6:19]
public struct ContactBeginTouchEvent
{
    public ShapeId ShapeIdA;
    public ShapeId ShapeIdB;

    public ContactBeginTouchEvent(ShapeId shapeIdA, ShapeId shapeIdB)
    {
        ShapeIdA = shapeIdA;
        ShapeIdB = shapeIdB;
    }
}
```

典型用途包括：角色碰到拾取物、子弹命中敌人、角色判定"着地"。事件里只有 ShapeId，如果你想知道碰的是哪个刚体，可以用 `Shape.GetBody(shapeId)` 去查。

**ContactEndTouchEvent：停止触碰**

和 BeginTouch 对称，当两个形状从"触碰"变为"未触碰"时触发：

```csharp:src/Contact/ContactEndTouchEvent.cs [6:19]
public struct ContactEndTouchEvent
{
    public ShapeId ShapeIdA;
    public ShapeId ShapeIdB;

    public ContactEndTouchEvent(ShapeId shapeIdA, ShapeId shapeIdB)
    {
        ShapeIdA = shapeIdA;
        ShapeIdB = shapeIdB;
    }
}
```

处理 EndTouch 时要格外小心一个细节：收到事件时，对应的 ShapeId 可能已经失效了——如果刚体或形状在上一帧被销毁的话。所以访问前最好先验证一下有效性。

**ContactHitEvent：碰撞力度**

这是三者中信息量最大的。它只在碰撞的接近速度超过阈值时触发，告诉你碰撞点、法线和速度：

```csharp:src/Contact/ContactHitEvent.cs [7:32]
public struct ContactHitEvent
{
    public ShapeId ShapeIdA;
    public ShapeId ShapeIdB;
    public Vec2 Point;
    public Vec2 Normal;
    public float ApproachSpeed;

    public ContactHitEvent(ShapeId shapeIdA, ShapeId shapeIdB, Vec2 point, Vec2 normal, float approachSpeed)
    {
        ShapeIdA = shapeIdA;
        ShapeIdB = shapeIdB;
        Point = point;
        Normal = normal;
        ApproachSpeed = approachSpeed;
    }
}
```

`ApproachSpeed` 总是正值，单位通常是米每秒。你可以用它来决定播放多大的碰撞音效、产生多少粒子、扣多少血。

三类事件统一封装在 `ContactEvents` 结构中：

```csharp:src/Contact/ContactEvents.cs [8:37]
public struct ContactEvents
{
    public Memory<ContactBeginTouchEvent> BeginEvents;
    public Memory<ContactEndTouchEvent> EndEvents;
    public Memory<ContactHitEvent> HitEvents;

    public int BeginCount;
    public int EndCount;
    public int HitCount;

    public ContactEvents(Memory<ContactBeginTouchEvent> beginEvents, Memory<ContactEndTouchEvent> endEvents, Memory<ContactHitEvent> hitEvents, int beginCount, int endCount, int hitCount)
    {
        BeginEvents = beginEvents;
        EndEvents = endEvents;
        HitEvents = hitEvents;
        BeginCount = beginCount;
        EndCount = endCount;
        HitCount = hitCount;
    }
}
```

注意这里用了 `Memory<T>` 而不是原始数组，这是为了减少每帧的内存分配。

**事件不是默认开启的**，你需要在 `ShapeDef` 中显式启用：

```csharp:src/Collision/Geometry/ShapeDef.cs [36:44]
public struct ShapeDef
{
    public bool EnableSensorEvents;
    public bool EnableContactEvents;
    public bool EnableHitEvents;
    public bool EnablePreSolveEvents;
}
```

这三个开关有几个共同规则：
- 只对**运动学**和**动力学**刚体有效，静态刚体不会触发。
- **传感器**形状不会触发接触事件（传感器有自己的事件通道）。
- 只要参与接触的**任一方形状**开启了对应开关，该接触就会产生事件。

PreSolveEvents 是最贵的，因为它每帧对每个接触点都会调用。ContactEvents 和 HitEvents 的开销相对小得多。

**怎么获取事件？**通过 `World.GetContactEvents`：

```csharp:src/World.cs [1617:1633]
public static ContactEvents GetContactEvents(WorldId worldId)
{
    World world = GetWorldFromId(worldId);
    Debug.Assert(world.Locked == false);
    if (world.Locked)
    {
        return new ContactEvents();
    }

    int beginCount = (world.ContactBeginArray).Count;
    int endCount = (world.ContactEndArray).Count;
    int hitCount = (world.ContactHitArray).Count;

    ContactEvents events = new(world.ContactBeginArray, world.ContactEndArray, world.ContactHitArray, beginCount, endCount, hitCount);

    return events;
}
```

关键点：**必须在 `World.Step` 之后调用**，不能在 Step 内部调用。`world.Locked` 会阻止你在错误时机获取。

下面来看一个实战案例。Box2DSharp 自带的 `ContactEvent` 示例展示了标准的事件处理流程：

```csharp:test/Testbed.Samples/Events/ContactEvent.cs [120:228]
public override void Step()
{
    Vec2 position = Body.GetPosition(m_playerId);

    if (Input.IsKeyDown(KeyCodes.A))
    {
        Body.ApplyForce(m_playerId, (-m_force, 0.0f), position, true);
    }

    if (Input.IsKeyDown(KeyCodes.D))
    {
        Body.ApplyForce(m_playerId, (m_force, 0.0f), position, true);
    }

    if (Input.IsKeyDown(KeyCodes.W))
    {
        Body.ApplyForce(m_playerId, (0.0f, m_force), position, true);
    }

    if (Input.IsKeyDown(KeyCodes.S))
    {
        Body.ApplyForce(m_playerId, (0.0f, -m_force), position, true);
    }

    base.Step();

    int[] debrisToAttach = new int[e_count];
    ShapeId[] shapesToDestroy = new ShapeId[e_count];
    int attachCount = 0;
    int destroyCount = 0;

    ContactEvents contactEvents = World.GetContactEvents(WorldId);
    var beginEvents = contactEvents.BeginEvents.Span;
    for (int i = 0; i < contactEvents.BeginCount; ++i)
    {
        ref readonly ContactBeginTouchEvent evt = ref beginEvents[i];
        BodyId bodyIdA = Shape.GetBody(evt.ShapeIdA);
        BodyId bodyIdB = Shape.GetBody(evt.ShapeIdB);

        if (bodyIdA == m_playerId)
        {
            var userDataB = (BodyUserData?)Body.GetUserData(bodyIdB);
            if (userDataB == null)
            {
                if (evt.ShapeIdA != m_coreShapeId && destroyCount < e_count)
                {
                    bool found = false;
                    for (int j = 0; j < destroyCount; ++j)
                    {
                        if (evt.ShapeIdA == shapesToDestroy[j])
                        {
                            found = true;
                            break;
                        }
                    }

                    if (found == false)
                    {
                        shapesToDestroy[destroyCount] = evt.ShapeIdA;
                        destroyCount += 1;
                    }
                }
            }
            else if (attachCount < e_count)
            {
                debrisToAttach[attachCount] = userDataB.Index;
                attachCount += 1;
            }
        }
        else
        {
            Debug.Assert(bodyIdB == m_playerId);
            var userDataA = (BodyUserData?)Body.GetUserData(bodyIdA);
            if (userDataA == null)
            {
                if (evt.ShapeIdB != m_coreShapeId && destroyCount < e_count)
                {
                    bool found = false;
                    for (int j = 0; j < destroyCount; ++j)
                    {
                        if (evt.ShapeIdB == shapesToDestroy[j])
                        {
                            found = true;
                            break;
                        }
                    }

                    if (found == false)
                    {
                        shapesToDestroy[destroyCount] = evt.ShapeIdB;
                        destroyCount += 1;
                    }
                }
            }
            else if (attachCount < e_count)
            {
                debrisToAttach[attachCount] = userDataA.Index;
                attachCount += 1;
            }
        }
    }

    for (int i = 0; i < attachCount; ++i)
    {
        int index = debrisToAttach[i];
        BodyId debrisId = m_debrisIds[index];
        if (debrisId.IsNotNull)
        {
            continue;
        }

        Transform playerTransform = Body.GetTransform(m_playerId);
        Transform debrisTransform = Body.GetTransform(debrisId);
        Transform relativeTransform = B2Math.InvMulTransforms(playerTransform, debrisTransform);

        int shapeCount = Body.GetShapeCount(debrisId);
        if (shapeCount == 0)
        {
            continue;
        }

        var shapeId = Body.GetShape(debrisId);
        ShapeType type = Shape.GetType(shapeId);
        ShapeDef shapeDef = ShapeDef.DefaultShapeDef();
        shapeDef.EnableContactEvents = true;

        switch (type)
        {
            case ShapeType.CircleShape:
            {
                Circle circle = Shape.GetCircle(shapeId);
                circle.Center = B2Math.TransformPoint(relativeTransform, circle.Center);
                Shape.CreateCircleShape(m_playerId, shapeDef, circle);
            }
                break;

            case ShapeType.CapsuleShape:
            {
                Capsule capsule = Shape.GetCapsule(shapeId);
                capsule.Center1 = B2Math.TransformPoint(relativeTransform, capsule.Center1);
                capsule.Center2 = B2Math.TransformPoint(relativeTransform, capsule.Center2);
                Shape.CreateCapsuleShape(m_playerId, shapeDef, capsule);
            }
                break;

            case ShapeType.PolygonShape:
            {
                Polygon originalPolygon = Shape.GetPolygon(shapeId);
                Polygon polygon = Geometry.TransformPolygon(relativeTransform, originalPolygon);
                Shape.CreatePolygonShape(m_playerId, shapeDef, polygon);
            }
                break;

            default:
                Debug.Assert(false);
                break;
        }

        Body.DestroyBody(debrisId);
        m_debrisIds[index] = BodyId.NullId;
    }

    for (int i = 0; i < destroyCount; ++i)
    {
        Shape.DestroyShape(shapesToDestroy[i]);
    }
}
```

这个示例展示了一个极其重要的模式：**延迟处理**。遍历事件时，代码只是把需要销毁的 ShapeId 记录到 `shapesToDestroy` 数组里，并没有立即调用 `Shape.DestroyShape`。真正的销毁发生在所有事件遍历完毕之后。如果你在事件遍历过程中直接销毁形状，刚体的接触链表结构会被破坏，导致未定义行为。

另外，示例中的玩家形状开启了 `EnableContactEvents = true`，而碎片形状设为 `false`，这样碎片之间的碰撞就不会产生事件，只有玩家和碎片的碰撞才会触发。这种精细控制对减少不必要的开销很有用。

## 事件处理实战模式：获取事件 → 遍历处理 → 延迟销毁

游戏里的碰撞响应可不能随心所欲。Box2DSharp 在每一步结束后，会把所有事件整整齐齐地装进一个 `ContactEvents` 结构里。你要做的就是在 `Step()` 之后调用 `World.GetContactEvents()`，然后遍历处理。


`ContactEvents` 里面有三类事件的数组和计数：`BeginEvents`、`EndEvents`、`HitEvents`，以及对应的 `BeginCount`、`EndCount`、`HitCount`。注意这里用的是 `Memory<T>`，你可以直接用 `.Span` 来遍历，完全没有额外的内存分配。


箱子的入口很简单，但真正的学问在于怎么处理这些事件。看下面这个 `ContactEvent` 示例，它展示了最标准的实战模式：先收集，再统一处理，最后执行延迟销毁。


这段代码里有几个非常值得学习的细节。首先，它用 `beginEvents` 数组遍历所有 BeginTouch 事件，而不是在事件产生时立刻回调。这就是 Box2DSharp 的缓冲区设计——稳定、可预测，而且线程安全。

其次，**延迟销毁**非常关键。代码里创建了两个数组：`debrisToAttach` 和 `shapesToDestroy`。在遍历事件的过程中，它只是把需要处理的对象索引记录下来，真正执行 `Shape.DestroyShape()` 和 `Body.DestroyBody()` 的操作，是在遍历结束之后进行的。如果你在事件循环里直接销毁刚体或形状，世界内部的接触图和约束图可能会立刻失效，轻则崩溃，重则产生难以复现的隐藏 bug。

最后再提醒一点：示例中对 `shapesToDestroy` 做了去重检查（`found == false`）。这是因为同一个形状可能在同一帧里触发多次碰撞事件，重复销毁会导致断言失败。这个小细节在实际项目里能帮你省下不少调试时间。

## 预求解回调 PreSolveFcn：自定义碰撞响应

有时候，仅仅知道"两个形状碰上了"是不够的。你还想在一个时间步开始求解之前，临时决定"这次碰撞到底要不要生效"。这就是 `PreSolveFcn` 存在的意义。


这个委托的签名很直白：给你两个 `ShapeId`、一个 `Manifold` 引用，以及一个自定义的 `context`，你返回 `true` 表示这次碰撞正常参与物理求解，返回 `false` 就表示"假装没碰到"。

最典型的游戏场景就是**单向平台**。玩家从平台上方跳下来时应该能站上去，但从下方往上跳时则应该直接穿过去。这个逻辑用 `PreSolveFcn` 实现非常自然。来看 `Platformer` 示例：


角色形状的 `ShapeDef` 开启了 `EnablePreSolveEvents`，并且把 `PreSolveStatic` 注册给了世界。回调的实现如下：


核心逻辑很清楚：先判断这次碰撞是不是跟平台有关。如果不是，直接返回 `true` 放行。如果角色碰到了平台，就检查碰撞法线的方向。`sign * normal.Y > 0.95f` 意味着角色是从上方落到平台上的，这时候返回 `true` 允许碰撞。如果法线方向朝下，说明角色想从平台底部跳上去，就返回 `false` 禁用本次碰撞。

还有一个很贴心的细节：代码检查了 `separation > 0.1f * m_radius`。如果角色和平台只是轻微重叠，也允许碰撞继续，避免在平台边缘反复"闪烁"穿模。

不过要注意注释里的警告：**这个回调必须线程安全**。因为它可能在多线程并行求解时被同时调用，所以你不能在回调里修改世界的任何状态，只能基于传入的参数做只读判断。

## 碰撞过滤进阶：CategoryBits × MaskBits 真值表 + GroupIndex 特殊规则

如果你做过团队竞技类游戏，一定遇到过"同队友军之间不应该互相碰撞"这样的需求。Box2DSharp 的 `Filter` 结构就是专门解决这类问题的利器。


`Filter` 有三个字段：`CategoryBits`（我是谁）、`MaskBits`（我愿意和谁碰撞），以及 `GroupIndex`（特殊分组规则）。默认过滤器 `DefaultFilter()` 的 `CategoryBits` 是 `0x0001`，`MaskBits` 是 `ulong.MaxValue`，意思是"我是默认类别，我跟谁都撞"。

### CategoryBits 与 MaskBits 的匹配规则

两个形状到底能不能碰撞，核心逻辑在 `Contact.ShouldShapesCollide` 里：


看明白了吗？碰撞成立的前提是**双向同意**：
- `filterA.MaskBits & filterB.CategoryBits != 0`（A 愿意接受 B 的类别）
- `filterA.CategoryBits & filterB.MaskBits != 0`（B 愿意接受 A 的类别）

只有这两个条件同时满足，碰撞才会发生。下面我用真值表帮你梳理几个常见场景：

| 场景 | 形状 A | 形状 B | 碰撞结果 | 说明 |
|---|---|---|---|---|
| 默认值互相碰撞 | Cat=0x1, Mask=Max | Cat=0x1, Mask=Max | 是 | 默认全开 |
| 玩家只撞地面 | Cat=Player, Mask=Ground | Cat=Ground, Mask=All | 是 | 双向匹配 |
| 玩家只撞地面 | Cat=Player, Mask=Ground | Cat=Enemy, Mask=All | 否 | A 不接受 Enemy |
| 子弹只撞敌人 | Cat=Bullet, Mask=Enemy | Cat=Enemy, Mask=All | 是 | 单向也能成立 |
| 两个友军单位 | Cat=Team1, Mask=Ground\|Enemy | Cat=Team1, Mask=Ground\|Enemy | 否 | Mask 里没有 Team1，互相拒绝 |

### ShapeFilter 示例

`Testbed` 里的 `ShapeFilter` 示例把上面的逻辑做得非常直观。它创建了三个玩家，分别属于 `Team1`、`Team2`、`Team3`，然后在 UI 里动态勾选"想和谁碰撞"：


这个示例的精妙之处在于，它通过位运算 `|=` 和 `&= ~` 来动态修改 `MaskBits`，立刻就能看到不同队伍之间的碰撞关系变化。注意这里用的是 `uint` 常量，但 `Filter` 的字段类型是 `ulong`，所以实际项目里如果你的位标志超过 32 个，记得用 `ulong` 常量。

### GroupIndex 的特殊规则

`GroupIndex` 是碰撞过滤里的"大招"。它的规则凌驾于 `CategoryBits` 和 `MaskBits` 之上：

- `GroupIndex == 0`：完全不起作用，走正常的位掩码逻辑。
- `GroupIndex > 0` 且两个形状的 `GroupIndex` 相同：强制碰撞。
- `GroupIndex < 0` 且两个形状的 `GroupIndex` 相同：强制不碰撞。

用真值表总结：

| 条件 | 结果 | 游戏场景 |
|---|---|---|
| `groupA == groupB > 0` | 强制碰撞 | 同一联盟必须互相碰撞 |
| `groupA == groupB < 0` | 强制不碰撞 | 同一布娃娃的身体部件避免自碰撞 |
| `groupA != groupB` 或任一为零 | 走 Mask 逻辑 | 正常情况 |

`Filter` 源码注释里提到的一个经典例子就是**布娃娃（ragdoll）**：你可能希望角色的四肢和身体之间有碰撞，但不希望左臂和右臂这些同一个布娃娃的内部部件互相穿模卡住。这时候就可以给同一个布娃娃的所有形状分配一个相同的负数 `GroupIndex`，它们之间就永远不会碰撞了。而不同布娃娃之间的角色，依然可以正常用 `CategoryBits` 和 `MaskBits` 控制碰撞关系。

## ContactFlags / ContactSimFlags 标志解读

Box2DSharp 里有两套标志系统，分别服务于不同的层次。`ContactFlags` 存在 `Contact` 对象上，属于持久化的"冷数据"；`ContactSimFlags` 存在 `ContactSim` 上，用于模拟步内的实时状态跟踪。


`ContactFlags` 各标志的含义如下：

`ContactTouchingFlag`：两个**实体形状**正在接触。注意这里不包括传感器，仅用于有物理响应的实体接触。

`ContactHitEventFlag`：这个接触点满足产生碰撞力度事件的条件。

`ContactSensorFlag`：两个形状中至少有一个是传感器。

`ContactSensorTouchingFlag`：传感器正在检测到重叠（目前源码注释说尚未使用）。

`ContactEnableSensorEvents` / `ContactEnableContactEvents`：这个接触点是否想要接收传感器事件或碰撞事件。这些标志由 `ShapeDef` 里的 `EnableSensorEvents` 和 `EnableContactEvents` 设置后传递过来。


`ContactSimFlags` 各标志的含义：

`TouchingFlag`：形状正在接触，**包括传感器**。和 `ContactFlags.ContactTouchingFlag` 的区别在于后者只关心实体。

`Disjoint`：这个接触的 AABB 已经没有重叠了，等待清理。

`StartedTouching`：这一帧刚刚开始接触，用于触发 Begin 事件和岛屿链接。

`StoppedTouching`：这一帧刚刚停止接触，用于触发 End 事件和岛屿解链。

`EnableHitEvent`：开启碰撞力度事件。

`EnablePreSolveEvents`：开启预求解回调。

这两套标志为什么要分开？因为 `Contact` 是长期存在的持久对象，而 `ContactSim` 可能在每一步里在不同的解算集合之间移动。把实时状态放在 `ContactSim` 上，既保证了并行求解时的高效访问，又避免了频繁修改持久 `Contact` 对象带来的缓存不友好。你在写游戏逻辑时，通常只需要通过 `World.GetContactEvents()` 拿到已经整理好的事件数组，不需要直接操作这些底层标志。但理解它们的存在，能让你在遇到奇怪的碰撞行为时，更快定位到问题根因。

## 设计思考：为什么事件是缓冲区模式而非即时回调？

Box2DSharp 没有让你在 physic step 中间收到 `OnCollisionEnter`，而是在 `World.Step()` 结束后统一给出一整个 `ContactEvents` 数组。这个设计乍看有点绕，其实藏着三个很实际的考量。

第一，内部并行求解的安全。物理引擎的接触更新是在多线程里并行跑的，如果允许即时回调，你的游戏逻辑就会在中途乱入物理线程，数据竞争和死锁几乎不可避免。缓冲区相当于给物理线程和游戏主线程之间加了一道"闸口"，所有事件先暂存，等步进安全结束后再一次性交接。

第二，防止在回调里破坏世界状态。即时回调最大的陷阱是用户可能在碰撞发生的瞬间就销毁刚体、创建关节，甚至修改 world 结构。而物理引擎此时可能还在遍历约束图，中间删节点会直接崩溃。缓冲区让你只能在 step 之外处理事件，天然避开了这个雷区。

第三，批量处理更高效。你的游戏帧里可能发生几十次碰撞，缓冲区模式让你用一次遍历就能全部消化，不需要在每个接触点更新时都回调一次 C# 边界。这也是一种"数据局部性优先"的设计思路——物理引擎先把事情做完，再集中输出结果。

## 小结

这一章我们梳理了 Contact 的完整生命周期：从 `CreateContact` 建立冷数据与模拟层，到 `UpdateContact` 更新流形和触碰状态，再到 `DestroyContact` 回收资源。三类事件——`BeginTouch`、`EndTouch` 和 `HitEvent`——都通过 `ContactEvents` 缓冲区在步进结束后统一交付，避免了物理中间状态被外部逻辑干扰。PreSolveFcn 则让你在求解前有机会按帧禁用接触或调整法线，实现单向平台这类高级效果。最后，碰撞过滤的 `CategoryBits` 与 `MaskBits` 提供了灵活的位掩码配对规则，而 `GroupIndex` 用正负号覆盖了这层规则，是同组强制碰撞或强制忽略的最简手段。
