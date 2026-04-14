---
title: "第8章：传感器——无形之眼"
date: 2026-04-14
categories:
  - 书籍
tags:
  - Box2D
description: "Sensor传感器机制，无形碰撞检测与区域感知"
---

## 概述

传感器（Sensor）是物理世界里的一双"无形之眼"。

它和普通形状看起来一样，也能被碰撞检测系统发现，但有一个关键区别：传感器只检测重叠，不产生任何碰撞响应。也就是说，物体可以穿透它，不会反弹、不会被阻挡，也不会产生摩擦力。这在游戏中非常实用，比如触发陷阱、检测角色是否进入某个区域、或者实现传送门逻辑。

传感器和传感器之间也不会互相产生事件，它们只关心与普通形状的重叠情况。

## 创建传感器：IsSensor 标志

在 Box2DSharp 中，创建传感器只需要一步：把 `ShapeDef.IsSensor` 设为 `true`。

```csharp:src/Collision/Geometry/ShapeDef.cs [28:31]
        /// A sensor shape generates overlap events but never generates a collision response.
        ///	Sensors do not collide with other sensors and do not have continuous collision.
        ///	Instead use a ray or shape cast for those scenarios.
        public bool IsSensor;
```

注意注释里的三条重要规则：
- 传感器不会产生碰撞响应
- 传感器之间不会互相碰撞
- 传感器不支持连续碰撞检测

来看一个实际的创建例子。在传感器事件示例中，底部有一个检测区域就是这么创建的：

```csharp:test/Testbed.Samples/Events/SensorEvent.cs [135:140]
            {
                Polygon box = Geometry.MakeOffsetBox(4.0f, 1.0f, (0.0f, -30.5f), Rot.Identity);
                ShapeDef shapeDef = ShapeDef.DefaultShapeDef();
                shapeDef.IsSensor = true;
                Shape.CreatePolygonShape(groundId, shapeDef, box);
            }
```

这里用 `MakeOffsetBox` 创建了一个偏移的矩形，然后直接把 `IsSensor` 打开，再挂到地面刚体上。就这么简单，一个触发区域就做好了。

另外，`ShapeDef` 里还有一个 `EnableSensorEvents` 字段，默认值就是 `true`。这个标志控制形状是否参与传感器事件的生成，通常保持默认即可。

```csharp:src/Collision/Geometry/ShapeDef.cs [33:34]
        /// Enable sensor events for this shape. Only applies to kinematic and dynamic bodies. Ignored for sensors.
        public bool EnableSensorEvents;
```

## 传感器事件详解

传感器产生的事件不是通过回调函数实时通知的，而是由 `World` 在 `Step()` 结束后统一收集，供你在下一帧批量查询。这种设计避免了多线程模拟时的事件处理竞态问题。

### 开始触碰事件：SensorBeginTouchEvent

当普通形状进入一个传感器的范围时，引擎会生成一个 `SensorBeginTouchEvent`：

```csharp:src/Sensor/SensorBeginTouchEvent.cs [19:33]
    /// A begin touch event is generated when a shape starts to overlap a sensor shape.
    public struct SensorBeginTouchEvent
    {
        /// The id of the sensor shape
        public ShapeId SensorShapeId;

        /// The id of the dynamic shape that began touching the sensor shape
        public ShapeId VisitorShapeId;

        public SensorBeginTouchEvent(ShapeId sensorShapeId, ShapeId visitorShapeId)
        {
            SensorShapeId = sensorShapeId;
            VisitorShapeId = visitorShapeId;
        }
    }
```

这个结构体非常简洁，只包含两个 `ShapeId`：
- `SensorShapeId`：传感器本身的形状 ID
- `VisitorShapeId`：进入传感器范围的那个"访客"形状 ID

通过 `VisitorShapeId`，你可以进一步用 `Shape.GetBody()` 拿到对应的刚体 ID，再读取刚体的 UserData 来定位游戏对象。

### 结束触碰事件：SensorEndTouchEvent

当访客形状离开传感器时，引擎生成 `SensorEndTouchEvent`：

```csharp:src/Sensor/SensorEndTouchEvent.cs [3:17]
    /// An end touch event is generated when a shape stops overlapping a sensor shape.
    public struct SensorEndTouchEvent
    {
        /// The id of the sensor shape
        public ShapeId SensorShapeId;

        /// The id of the dynamic shape that stopped touching the sensor shape
        public ShapeId VisitorShapeId;

        public SensorEndTouchEvent(ShapeId sensorShapeId, ShapeId visitorShapeId)
        {
            SensorShapeId = sensorShapeId;
            VisitorShapeId = visitorShapeId;
        }
    }
```

结构和开始事件完全对称，只是语义不同。有些逻辑你可能只需要处理 `Begin`（比如触发一次伤害），有些则需要成对处理（比如进入时高亮、离开时取消高亮）。

### 事件集合：SensorEvents

`World` 并不会直接返回单个事件，而是把它们打包成 `SensorEvents`：

```csharp:src/Sensor/SensorEvents.cs [7:29]
    public struct SensorEvents
    {
        /// Array of sensor begin touch events
        public Memory<SensorBeginTouchEvent> BeginEvents;

        /// Array of sensor end touch events
        public Memory<SensorEndTouchEvent> EndEvents;

        /// The number of begin touch events
        public int BeginCount;

        /// The number of end touch events
        public int EndCount;

        public SensorEvents(Memory<SensorBeginTouchEvent> beginEvents, Memory<SensorEndEvent> endEvents, int beginCount, int endCount)
        {
            BeginEvents = beginEvents;
            EndEvents = endEvents;
            BeginCount = beginCount;
            EndCount = endCount;
        }
    }
```

这里用 `Memory<T>` 包装了事件数组，并附带 `BeginCount` 和 `EndCount`。你可以通过 `.Span` 获取实际的事件数据进行遍历。要注意，数组的底层容量可能大于实际事件数，所以务必使用 `BeginCount` / `EndCount` 作为遍历上限，不要直接用 `.Length`。

### 从 World 获取事件

事件的入口是 `World.GetSensorEvents`：

```csharp:src/World.cs [1601:1615]
        public static SensorEvents GetSensorEvents(WorldId worldId)
        {
            World world = GetWorldFromId(worldId);
            Debug.Assert(world.Locked == false);
            if (world.Locked)
            {
                return new SensorEvents();
            }

            int beginCount = (world.SensorBeginEventArray).Count;
            int endCount = (world.SensorEndEventArray).Count;

            SensorEvents events = new(world.SensorBeginEventArray, world.SensorEndEventArray, beginCount, endCount);
            return events;
        }
```

这个方法只能在 `World.Step()` 完成之后调用。如果世界还处于锁定状态（`Locked == true`），会返回空的事件集合。每次调用 `Step()` 之前，引擎会先清空这些事件数组，所以你拿到的事件总是当前步的，不会累积。

### 事件是怎么生成的？

事件生成的逻辑藏在 `World.Collide()` 的接触状态更新里。当检测到传感器接触开始触碰时：

```csharp:src/World.cs [783:803]
                        if ((flags & ContactFlags.ContactSensorFlag) != 0)
                        {
                            // Contact is a sensor
                            if ((flags & ContactFlags.ContactEnableSensorEvents) != 0)
                            {
                                if (shapeA.IsSensor)
                                {
                                    SensorBeginTouchEvent evt = new(shapeIdA, shapeIdB);
                                    world.SensorBeginEventArray.Push(evt);
                                }

                                if (shapeB.IsSensor)
                                {
                                    SensorBeginTouchEvent evt = new(shapeIdB, shapeIdA);
                                    world.SensorBeginEventArray.Push(evt);
                                }
                            }

                            contactSim.SimFlags &= ~ ContactSimFlags.StartedTouching;
                            contact.Flags |= ContactFlags.ContactSensorTouchingFlag;
                        }
```

注意这里有一个对称处理：如果 A 是传感器，就生成 `(A, B)` 的事件；如果 B 也是传感器，再生成 `(B, A)` 的事件。虽然通常只有一个形状是传感器，但引擎的设计允许双边都是传感器的情况被正确记录。

停止触碰的逻辑也是类似的：

```csharp:src/World.cs [840:859]
                            if (flags.IsSet(ContactFlags.ContactEnableSensorEvents))
                            {
                                if (shapeA.IsSensor)
                                {
                                    SensorEndTouchEvent evt = new(shapeIdA, shapeIdB);
                                    world.SensorEndEventArray.Push(evt);
                                }

                                if (shapeB.IsSensor)
                                {
                                    SensorEndTouchEvent evt = new(shapeIdB, shapeIdA)
                                        ;
                                    world.SensorEndEventArray.Push(evt);
                                }
                            }
```

## 事件处理实战模式：GetSensorEvents → 遍历 → 安全处理

每帧物理模拟结束后，传感器事件被集中缓冲在 `World` 中。你需要显式调用 `World.GetSensorEvents` 来获取这些事件。


该函数返回一个 `SensorEvents` 结构，里面包含两个数组：`BeginEvents` 和 `EndEvents`，分别记录"开始重叠"和"结束重叠"。


遍历事件时，最常见的操作是通过 `VisitorShapeId` 找到对应的刚体，再读取 `UserData` 来关联游戏对象。看下面这段标准写法：


注意这里的 **延迟销毁（Deferred Destruction）** 技巧。`SensorEvent` 示例里先将要销毁的对象标记到 `deferredDestructions` 数组里，等整个事件循环结束后再统一调用 `DestroyElement`。


为什么要延迟？因为如果你在遍历事件的过程中直接销毁形状或刚体，后续事件里的 `ShapeId` 可能会变成"孤儿 ID"（orphaned shape ids），导致断言失败或读取无效数据。这是传感器事件处理中最容易踩的坑，务必养成先标记、后处理的习惯。

## 典型应用场景：触发区域 / 角色检测 / 关卡逻辑

### 1. 触发区域（Trigger Zone）

传感器最典型的用法就是做一个"无形的区域"。比如 `SensorEvent` 示例底部的那条传感器：


这个矩形传感器固定在地面底部，没有任何碰撞响应。当甜甜圈（Donut）或火柴人（Human）掉进去时，就会触发 `SensorBeginTouchEvent`，示例随即把它们销毁并重新生成。这其实就是最简单的"掉落即死亡"机制。

### 2. 角色检测（Character Detection）

传感器非常适合做角色的"感知范围"。想象一个敌人头顶有一个圆形的传感器：

- 当玩家进入传感器范围 → `BeginEvent` 触发，敌人发现目标，进入追击状态。
- 当玩家离开传感器范围 → `EndEvent` 触发，敌人丢失目标，返回巡逻状态。

你不需要写任何距离计算，Box2D 的宽相检测会自动帮你完成。相比每帧遍历所有角色算距离，传感器的效率更高，而且代码更干净。

### 3. 平台跳跃与碰撞过滤

在 `Platformer` 示例中，虽然没有直接使用传感器，但它展示了和传感器精神相通的"重叠检测 + 条件过滤"思路：


这里通过 `PreSolve` 回调判断角色是否站在平台正上方，从而决定是否允许碰撞。如果你的需求是"站在平台上可以跳跃，但从下面跳上去不会被撞头"，那么传感器也能帮上忙：你可以在平台边缘放一个很薄的传感器条，角色触碰到传感器条就设置"允许跳跃"标志，而平台本身的碰撞体保持普通形状不变。

再看 `Platformer` 中检测角色是否能跳跃的逻辑：


它直接读取角色的接触数据并判断法线。如果你把脚下的地面换成一个传感器条，逻辑会更简单：当角色进入脚底传感器时设置 `_canJump = true`，离开时重置。这样连 `Manifold.Normal` 的判断都省了。

### 4. 关卡逻辑与事件驱动

传感器天然适合事件驱动的关卡设计：

- **传送门**：两个传感器区域，进入 A 区域触发传送，把角色瞬移到 B 区域。
- **拾取道具**：金币或血包用传感器形状包裹，角色走过去自动拾取，没有物理碰撞的弹开效果。
- **机关触发**：踩下压力板（一个传感器形状）后开门、放桥、启动陷阱。

所有这些场景的共同点是：**只关心"有没有东西进入/离开"，不关心"撞完之后怎么反弹"**。这正是传感器存在的意义。

## 性能注意事项：与第6章接触事件的区别、开销分析

传感器事件和普通接触事件（第6章）在获取方式上非常相似，都是每帧结束后通过 `World` 的 API 批量读取。但底层行为有本质区别：

| 特性 | 传感器（Sensor） | 普通接触（Contact） |
|---|---|---|
| 物理响应 | 无 | 有（产生冲量、分离） |
| 事件类型 | `SensorBeginTouchEvent` / `SensorEndTouchEvent` | `ContactBeginTouchEvent` / `ContactEndTouchEvent` 等 |
| 约束求解 | 不参与 | 参与 |
| 主要用途 | 检测重叠 | 碰撞响应 |

由于传感器不产生碰撞冲量，它**不参与约束求解**，这部分 CPU 开销是零。但要注意，传感器**仍然会被宽相检测和中相检测处理**。如果你把整个世界都铺满传感器，碰撞检测的 Broadway 开销依然会显著增加。

实际开发中的建议：

- 传感器数量控制在合理范围，不要把每一个道具、每一面墙都做成传感器。
- 对于大量小物品的拾取，优先考虑用射线检测或 AABB 查询，而不是给每个金币挂传感器。
- 传感器事件和接触事件一样，只在时间步结束后生效，不要在 `Step` 中间读取它们。

## 设计思考

普通碰撞和传感器检测，本质上是在回答两个不同的问题。普通碰撞问的是"物体该怎么弹开"，传感器检测问的则是"这里有没有东西"。如果把这两个问题硬塞进同一套机制，代码会迅速变得一团糟。

试想一下，如果你用普通碰撞来实现一个"玩家进入房间触发敌人"的功能，会发生什么？你需要给触发区域一个极大的质量，或者写复杂的碰撞过滤逻辑，来避免玩家被一扇"无形的门"弹开。更糟糕的是，物理求解器会认真计算这个触发区域的反弹和摩擦，白白浪费性能。传感器存在的意义，正是把"检测"这件事从物理响应中干净地剥离出来。

Box2D 把传感器设计为独立的通道，还有一个架构上的好处：事件可以被安全地缓冲和批量处理。`SensorBeginTouchEvent` 和 `SensorEndTouchEvent` 都是在时间步结束后才统一交付的，这避免了在物理模拟中途修改世界的风险。如果传感器和普通碰撞共用回调，开发者很容易在碰撞回调里销毁物体，导致内部状态崩溃。独立通道让"只读检测"和"写操作响应"有了清晰的边界。

所以，传感器不是普通碰撞的"弱化版"，而是一个专门解决"空间存在感"问题的工具。它让游戏逻辑和物理逻辑各司其职，互不干扰。

## 小结

传感器与普通碰撞的核心区别在于：传感器只检测重叠、不产生任何物理响应。它不会让物体弹开，也不参与摩擦和冲量计算。开启方式很简单，把 `ShapeDef.IsSensor` 设为 `true` 即可。

事件处理上，传感器通过 `World.GetSensorEvents()` 获取本帧所有 `SensorBeginTouchEvent` 和 `SensorEndTouchEvent`。由于事件数据可能在物体销毁后失效，安全的做法是先遍历事件做标记，再统一执行销毁或状态变更。

典型的应用场景包括触发区域（如进入房间刷怪）、角色脚下落地检测、收集品的拾取判定、以及关卡逻辑中的传送门和死亡区域等。用好传感器，能让你的游戏逻辑在物理世界里真正做到"无形而有力"。
