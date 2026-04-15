---
title: "【教材】第9章：调试与可视化"
date: 2026-04-06
order: 009
categories:
  - 代码库
tags:
  - Box2D
description: "Box2DSharp调试渲染抽象层与可视化对接"
---

## 概述

调试与可视化是物理开发中不可或缺的环节。Box2DSharp 在设计上将调试渲染完全抽象出来，不依赖任何具体图形 API，让你可以自由对接 Unity 的 `GL` 接口、ImGui、甚至远程 Web 渲染器。

本章我们会从 `DebugDrawBase` 抽象基类出发，讲解每个渲染开关的含义与用途。同时我们也会介绍 `World.Draw` 的调用方式、Testbed 中的实际参考实现，以及 `Profile` 和 `Counters` 这两个性能监控工具。读完之后，你不仅能够为自己的项目接入一套完整的物理调试视图，还能准确找到性能瓶颈该从哪里下手。

## DebugDrawBase 接口

`DebugDrawBase` 是整个调试系统的核心接口，定义了所有绘图原语。你只需继承它，把这些原语映射到你自己的渲染后端即可。

先来看看它的抽象方法列表：

```csharp:src/Debug/DebugDrawBase.cs [8:79]
public abstract class DebugDrawBase
{
    public AABB DrawingBounds;
    public bool UseDrawingBounds;
    public bool DrawShapes;
    public bool DrawJoints;
    public bool DrawJointExtras;
    public bool DrawAABBs;
    public bool DrawMass;
    public bool DrawContacts;
    public bool DrawGraphColors;
    public bool DrawContactNormals;
    public bool DrawContactImpulses;
    public bool DrawFrictionImpulses;
    public object? Context;

    public abstract void DrawPolygon(ReadOnlySpan<Vec2> vertices, int vertexCount, B2HexColor color, object? context);
    public abstract void DrawSolidPolygon(Transform transform, ReadOnlySpan<Vec2> vertices, int vertexCount, float radius, B2HexColor color, object? context);
    public abstract void DrawCircle(Vec2 center, float radius, B2HexColor color, object? context);
    public abstract void DrawSolidCircle(Transform transform, float radius, B2HexColor color, object? context);
    public abstract void DrawCapsule(Vec2 p1, Vec2 p2, float radius, B2HexColor color, object? context);
    public abstract void DrawSolidCapsule(Vec2 p1, Vec2 p2, float radius, B2HexColor color, object? context);
    public abstract void DrawSegment(Vec2 p1, Vec2 p2, B2HexColor color, object? context);
    public abstract void DrawTransform(Transform transform, object? context);
    public abstract void DrawPoint(Vec2 p, float size, B2HexColor color, object? context);
    public abstract void DrawString(Vec2 p, string s, object? context);
    public abstract void DrawBackground();
}
```

从这个列表里你能看出几件事：

- **几何原语非常全**：线段、多边形（空心+实心）、圆、胶囊体，还有变换轴、点、文字和背景。
- **所有方法都接受 `B2HexColor`**：它是一个大枚举，里面有标准的 Web 颜色名，也有 Box2D 自己的品牌色（`Box2DRed`、`Box2DBlue` 等）。
- **`context` 参数就是你自己**：这个 `object?` 在 `World.Draw` 时会被原样传回，你可以用它存储相机参数、材质缓存、渲染批次等上下文。

Testbed 里已经有一个现成的参考实现，非常简洁。它把 `DebugDrawBase` 的方法一一转发给内部的 `IDraw` 接口：

```csharp:test/Testbed/Render/DebugDraw.cs [7:74]
public class DebugDraw(IDraw draw) : DebugDrawBase, IDisposable
{
    private IDraw _draw = draw;

    public void Flush()
    {
        _draw.Flush();
    }

    public void Dispose()
    {
        _draw = null!;
    }

    public override void DrawPolygon(ReadOnlySpan<Vec2> vertices, int vertexCount, B2HexColor color, object? context)
    {
        _draw.DrawPolygon(vertices, vertexCount, color);
    }

    public override void DrawSolidPolygon(Transform transform, ReadOnlySpan<Vec2> vertices, int vertexCount, float radius, B2HexColor color, object? context)
    {
        _draw.DrawSolidPolygon(transform, vertices, vertexCount, radius, color);
    }

    public override void DrawCircle(Vec2 center, float radius, B2HexColor color, object? context)
    {
        _draw.DrawCircle(center, radius, color);
    }

    public override void DrawSolidCircle(Transform transform, float radius, B2HexColor color, object? context)
    {
        _draw.DrawSolidCircle(transform, Vec2.Zero, radius, color);
    }

    public override void DrawCapsule(Vec2 p1, Vec2 p2, float radius, B2HexColor color, object? context)
    {
        _draw.DrawCapsule(p1, p2, radius, color);
    }

    public override void DrawSolidCapsule(Vec2 p1, Vec2 p2, float radius, B2HexColor color, object? context)
    {
        _draw.DrawSolidCapsule(p1, p2, radius, color);
    }

    public override void DrawSegment(Vec2 p1, Vec2 p2, B2HexColor color, object? context)
    {
        _draw.DrawSegment(p1, p2, color);
    }

    public override void DrawTransform(Transform transform, object? context)
    {
        _draw.DrawTransform(transform);
    }

    public override void DrawPoint(Vec2 p, float size, B2HexColor color, object? context)
    {
        _draw.DrawPoint(p, size, color);
    }

    public override void DrawString(Vec2 p, string s, object? context)
    {
        _draw.DrawString(p, s);
    }

    public override void DrawBackground()
    {
        _draw.DrawBackground();
    }
}
```

继承方式一句话总结：**写一个类继承 `DebugDrawBase`，实现这 12 个抽象方法，然后在每一帧调用 `World.Draw(worldId, yourDebugDraw)` 即可**。邯郸学步可以套用上面的 `DebugDraw`，直接把 Box2DSharp 的坐标和颜色转发到你的渲染管线。

## 渲染选项

`DebugDrawBase` 身兼两职：它既是渲染接口，也是配置面板。上半部分的全都是布尔开关，决定 `World.Draw` 到底画什么。我们来逐个过一遍：

| 开关 | 作用 |
|------|------|
| `DrawShapes` | 绘制所有形状的外轮廓和实心填充。这是最重要的开关，默认建议打开。 |
| `DrawAABBs` | 绘制每个形状的膨胀 AABB（Fat AABB），用于观察宽相检测的包围盒。 |
| `DrawJoints` | 绘制所有关节的锚点、连接线和极限。 |
| `DrawJointExtras` | 在关节上绘制额外的辅助信息。 |
| `DrawMass` | 绘制质心、质量数值和刚体的局部坐标轴。 |
| `DrawContacts` | 绘制接触点，不同状态用不同颜色区分。 |
| `DrawGraphColors` | 用不同颜色区分接触点所属的图着色（用于观察并行求解器分组）。 |
| `DrawContactNormals` | 在接触点处绘制法线方向。 |
| `DrawContactImpulses` | 在接触点处绘制法向冲量的大小。 |
| `DrawFrictionImpulses` | 在接触点处绘制摩擦冲量的大小。 |

`World.Draw` 内部就是按照这套开关依次执行的。先画形状，再画关节，再画 AABB，最后处理接触点和质量信息：

```csharp:src/World.cs [1333:1347]
public static void Draw(WorldId worldId, DebugDrawBase draw)
{
    World world = GetWorldFromId(worldId);
    Debug.Assert(world.Locked == false);
    if (world.Locked)
    {
        return;
    }

    if (draw.UseDrawingBounds)
    {
        DrawWithBounds(world, draw);
        return;
    }
```

如果 `DrawShapes` 为 `true`，引擎会遍历所有 `SolverSet` 中的刚体，根据刚体类型、休眠状态、传感器标记等自动选择一个标准色，最后调用 `DrawShape` 把你自定义的颜色和几何体发给你：

```csharp:src/World.cs [1349:1421]
if (draw.DrawShapes)
{
    int setCount = (world.SolverSetArray).Count;
    for (int setIndex = 0; setIndex < setCount; ++setIndex)
    {
        SolverSet set = world.SolverSetArray[setIndex];
        int bodyCount = set.Sims.Count;
        for (int bodyIndex = 0; bodyIndex < bodyCount; ++bodyIndex)
        {
            BodySim bodySim = set.Sims.Data[bodyIndex];
            world.BodyArray.CheckIndex(bodySim.BodyId);
            Body body = world.Array[bodySim.BodyId];
            Debug.Assert(body.SetIndex == setIndex);

            Transform xf = bodySim.Transform;
            int shapeId = body.HeadShapeId;
            while (shapeId != Core.NullIndex)
            {
                Shape shape = world.ShapeArray[shapeId];
                B2HexColor color;

                if (shape.CustomColor != 0)
                {
                    color = (B2HexColor)shape.CustomColor;
                }
                else if (body.Type == BodyType.DynamicBody && bodySim.Mass == 0.0f)
                {
                    color = B2HexColor.Red;
                }
                else if (body.SetIndex == SolverSetType.DisabledSet)
                {
                    color = B2HexColor.SlateGray;
                }
                else if (shape.IsSensor)
                {
                    color = B2HexColor.Wheat;
                }
                else if (bodySim.IsBullet && body.SetIndex == SolverSetType.AwakeSet)
                {
                    color = B2HexColor.Turquoise;
                }
                else if (body.IsSpeedCapped)
                {
                    color = B2HexColor.Yellow;
                }
                else if (bodySim.IsFast)
                {
                    color = B2HexColor.Salmon;
                }
                else if (body.Type == BodyType.StaticBody)
                {
                    color = B2HexColor.PaleGreen;
                }
                else if (body.Type == BodyType.KinematicBody)
                {
                    color = B2HexColor.RoyalBlue;
                }
                else if (body.SetIndex == SolverSetType.AwakeSet)
                {
                    color = B2HexColor.Pink;
                }
                else
                {
                    color = B2HexColor.Gray;
                }

                DrawShape(draw, shape, xf, color);
                shapeId = shape.NextShapeId;
            }
        }
    }
}
```

同时打开所有开关当然很爽，但性能开销也会叠加。建议日常开发只开 `DrawShapes` 和 `DrawJoints`；查碰撞问题的时候再打开 `DrawContacts` 和 `DrawAABBs`；到了性能调优阶段，再按需启用 `DrawGraphColors` 观察并行分组。

## World.Draw 调用流程

做好了自定义的 `DebugDrawBase` 子类，怎么让它生效呢？答案是调用 `World.Draw`。


`Draw` 方法会先判断世界是否处于锁定状态——如果正在执行 `Step`，世界会被锁定，此时调用 `Draw` 会直接返回。


如果开启了 `DrawShapes`，这段代码会遍历所有 `SolverSet`，再逐个遍历其中的 `BodySim`。它会根据刚体类型和状态自动挑颜色：自定义颜色优先；动态刚体质量为 0 时画红色；传感器画小麦色；子弹体画青绿色；超速刚体画黄色；静态体画淡绿色；运动体画皇家蓝；醒着的动态体画粉红色；睡着的画灰色。


真正的绘图委托由 `DrawShape` 完成，它按形状类别调用 `DrawSolidCapsule`、`DrawSolidCircle`、`DrawSolidPolygon` 或 `DrawSegment`。`World.Draw` 还负责绘制关节、AABB、质量和接触点，引擎已经把所有遍历逻辑封装好了，你只管实现绘制方法就行。

## Profile 性能分析器：各阶段耗时统计

除了"看到"物理世界长什么样，我们还需要"看到"物理计算花了多少时间。Box2DSharp 在每个 `World.Step` 里都会自动采集各阶段耗时，存到一个 `Profile` 对象里。


这个结构体里全是 `float` 字段，单位是毫秒。关键字段的含义如下：

- `Step`：整个 `Step` 的总耗时
- `Pairs`：粗检测（BroadPhase）配对更新的耗时
- `Collide`：精细碰撞（NarrowPhase）的耗时
- `Solve`：约束求解的总耗时
- `BuildIslands` / `SplitIslands` / `SleepIslands`：岛屿构建、拆分和休眠管理
- `SolveConstraints` / `PrepareConstraints`：约束求解的核心阶段
- `IntegrateVelocities` / `IntegratePositions`：速度和位置积分
- `WarmStart` / `SolveVelocities` / `RelaxVelocities`：速度求解器的子阶段
- `Broadphase` / `Continuous`：BroadPhase 更新和连续碰撞检测

数据采集发生在 `World.Step` 内部：


每次 `Step` 开始时，`world.Profile` 都会被重置。随后在关键阶段前后用 `Stopwatch` 打点，把耗时记录下来。

你可以通过 `World.GetProfile` 随时拿到这些数据：


拿到 `Profile` 后，你就可以在屏幕上显示关键指标。如果发现 `Collide` 或 `Solve` 经常飙高，就要考虑减少接触点数量或者简化形状了。

## Counters 计数器：运行时状态监控

`Profile` 告诉你"花了多少时间"，`Counters` 告诉你"有多少东西"。


这个类的字段很直观：

- `BodyCount`、`ShapeCount`、`ContactCount`、`JointCount`：基础实体数量
- `IslandCount`：当前岛屿数量
- `StaticTreeHeight`、`TreeHeight`：AABB 树的高度，反映空间查询效率
- `TaskCount`：上一步 `Step` 实际产生的任务数
- `ColorCounts`：约束图中各颜色层的约束数量

获取方式同样很简单：


这段代码组装了一个 `Counters` 实例，从各个池子和树结构里取出当前世界的规模数据。`TreeHeight` 取了动态树和运动树中较高的那个，`ColorCounts` 则统计了约束图里每个颜色层的接触点和关节总数。

建议你在调试面板里常驻显示 `BodyCount`、`ContactCount` 和 `TreeHeight`。如果 `ContactCount` 远大于 `BodyCount`，说明场景中有很多不必要的密集接触，可能需要优化形状布局或者减少堆叠。

## 实战：继承 DebugDrawBase 实现自定义渲染

Box2DSharp 的设计哲学很明确：引擎只负责"描述要画什么"，你负责"怎么画"。这意味着它不绑定任何图形 API，无论你用 OpenGL、SkiaSharp、Unity 还是 Godot，都能轻松接入。

`DebugDrawBase` 就是这座桥梁：


Testbed 项目里有一个完整的实现，它把绘制工作转发给了一个跨平台的 `IDraw` 接口：


这个类持有一个 `IDraw` 实例，把所有抽象方法都转发过去。这说明 Testbed 自己封装了底层的图形 API，而 Box2DSharp 对此一无所知——典型的解耦设计。

如果你在自己的 Unity 项目里做集成，可以写一个类似的子类。比如在 `DrawSegment` 里调用 `UnityEngine.Debug.DrawLine`，在 `DrawSolidPolygon` 里用 `GL` 画填充多边形，在 `DrawString` 里用 `GUI.Label` 显示文字。每帧只需要调用 `World.Draw(worldId, _debugDraw)` 即可。

关键步骤就三点：创建继承 `DebugDrawBase` 的类；在每个抽象方法里调用你熟悉的图形 API；每帧调用 `World.Draw` 并打开需要的开关。

唯一要注意的是，`World.Draw` 每帧可能调用上千次绘制方法。所以你的实现最好能做一点批处理，比如把线段先收集到一个数组里，再统一提交给 GPU，避免每根线都发一次 draw call。

## 设计思考

Box2DSharp 把调试渲染做成了一个抽象基类 `DebugDrawBase`，而不是直接内置任何真正的绘图代码。这背后有几个很务实的考量。

第一，跨平台是刚性需求。物理引擎本身跑在各种环境里——PC、手机、游戏主机，甚至纯服务端。内置一个 OpenGL 或 MonoGame 渲染器，等于强行绑定某个图形平台，对其他用户反倒成了包袱。接口化之后，无论你是用 Unity、Godot、DirectX 还是自定义软渲染，只要实现几个虚方法就能对接。

第二，零依赖让引擎更轻量。Box2DSharp 的核心代码不引用任何第三方图形库，编译产物干净小巧。调试绘制只是 world 的一个外部调用，不侵入物理模拟的内部数据结构和执行流程。

第三，灵活控制成为可能。`DebugDrawBase` 提供了一排布尔开关，你可以按需只画形状、只画 AABB、或者只监控接触点。除此之外，Testbed 里的 `DebugDraw` 实现也印证了这种桥接模式：它把引擎发来的几何数据再转发给 `IDraw` 层，物理代码与最终渲染彻底解耦。这给了你"调试用一套渲染器，发布时直接关闭"的自由。

## 小结

这一章咱们梳理了 Box2DSharp 的调试与可视化全家桶。`DebugDrawBase` 是整个调试渲染的入口，通过继承它并交给 `World.Draw`，你就能把物理世界的内部状态"看"出来。`Profile` 和 `Counters` 则是诊断性能的左右手：前者记录各个模拟阶段的毫秒耗时，后者统计刚体、形状、接触点、关节的运行时数量。自定义渲染的关键在于把基类的虚方法桥接到你项目现有的图形层，同时利用布尔开关精准控制画什么、不画什么，避免在正式发布里引入多余的绘制开销。
