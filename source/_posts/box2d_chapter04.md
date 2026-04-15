---
title: "【教材】第4章：形状系统——给刚体穿上碰撞外衣"
date: 2026-04-14
categories:
  - 代码库
tags:
  - Box2D
description: "Shape碰撞边界、质量分布、表面属性与ShapeDef配置"
---

## 概述

如果说刚体（Body）是物理对象的骨架，那么形状（Shape）就是它的外衣。形状负责定义刚体的碰撞边界、质量分布和表面属性。一个没有形状的刚体，在物理世界里就像个透明人——不会碰撞、不受力、对其他物体也没有任何影响。

在 Box2DSharp 中，形状通过 `ShapeDef` 与具体的几何数据配合使用。创建形状后，引擎会自动根据密度计算质量、生成 AABB 用于粗检测，并将形状注册到刚体的双向链表中。值得一提的是，一个刚体可以挂载多个形状，这在模拟桌子、飞船等复杂物体时非常实用。

## 五种形状详解

Box2DSharp 提供了五种基础形状类型，通过 `ShapeType` 枚举来区分：

```csharp:src/Collision/Geometry/ShapeType.cs [6:25]
public enum ShapeType
{
    CircleShape,
    CapsuleShape,
    SegmentShape,
    PolygonShape,
    ChainSegmentShape,
    ShapeTypeCount
}
```

这五种形状各有特点，下面我们来一一认识它们。

### Circle（圆形）

圆形是最简单的形状，只有两个字段：中心和半径。它在内存中只有 12 字节，碰撞检测的计算成本也最低。

```csharp:src/Collision/Geometry/Circle.cs [6:28]
public struct Circle
{
    public Vec2 Center;
    public float Radius;

    public Circle(Vec2 center, float radius)
    {
        Center = center;
        Radius = radius;
    }

    public static implicit operator Circle((Vec2 Center, float Radius) tuple)
    {
        return new Circle(tuple.Center, tuple.Radius);
    }
}
```

创建圆形的 API 很直接：

```csharp:src/Collision/Geometry/Shape.cs [225:229]
public static ShapeId CreateCircleShape(BodyId bodyId, ShapeDef def, Circle circle)
{
    return CreateShape(bodyId, def, circle, ShapeType.CircleShape);
}
```

圆形适合模拟球、轮子、炮弹等规则且需要旋转对称的物体。由于它各向同性，滚动起来也是最自然的。

### Polygon（凸多边形）

多边形是应用最广泛的形状。它是一个**凸多边形**，最多支持 8 个顶点，包含顶点数组、法线数组、质心和半径。

```csharp:src/Collision/Geometry/Polygon.cs [11:37]
public struct Polygon
{
    public FixedArray8<Vec2> Vertices;
    public FixedArray8<Vec2> Normals;
    public Vec2 Centroid;
    public float Radius;
    public int Count;
}
```

注意源码里的警告注释：不要手动填充这个结构体，应该使用 `Geometry.MakePolygon` 或 `Geometry.MakeBox` 等辅助函数。这些函数会自动计算法线、质心和凸包，确保多边形的正确性。

多边形占用 144 字节，是圆形的好几倍，但它能精确描述箱子、平台、角色躯干等各种外形。在实际项目中，大多数静态环境和动态物体都会用它来构建。

### Capsule（胶囊体）

胶囊体可以看作是一个"拉伸的圆"——两个半圆中间用矩形连接。它由两个圆心点和半径定义：

```csharp:src/Collision/Geometry/Capsule.cs [10:44]
public struct Capsule
{
    public Vec2 Center1;
    public Vec2 Center2;
    public float Radius;

    public Span<Vec2> Points
    {
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        get => MemoryMarshal.CreateSpan(ref Center1, 2);
    }

    public Capsule(Vec2 center1, Vec2 center2, float radius)
    {
        Center1 = center1;
        Center2 = center2;
        Radius = radius;
    }
}
```

胶囊体占 20 字节，碰撞检测比多边形便宜得多。它特别适合模拟人物角色的碰撞体——站立时细长、躺下时扁平，而且不会卡在墙角。如果两个圆心点太近（小于线性容差），`CreateCapsuleShape` 会自动降级为圆形：

```csharp:src/Collision/Geometry/Shape.cs [231:241]
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
```

### Segment（线段）

线段由两个端点构成，支持双面碰撞。

```csharp:src/Collision/Geometry/Segment.cs [10:38]
public struct Segment
{
    public Vec2 Point1;
    public Vec2 Point2;

    public Span<Vec2> Points
    {
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        get => MemoryMarshal.CreateSpan(ref Point1, 2);
    }

    public Segment(Vec2 point1, Vec2 point2)
    {
        Point1 = point1;
        Point2 = point2;
    }
}
```

线段占 16 字节，常用作地面、墙壁或平台的简化碰撞体。但它有个特点：**没有体积**，所以不会参与质量计算。这意味着一片纯线段组成的地面是"零质量"的，通常放在静态刚体上使用。

### ChainShape（链条形状）

链条形状不是单一形状，而是由多个 `ChainSegmentShape` 组成的序列。它专为消除"幽灵碰撞"（Ghost Collision）而设计，常用于构建地形边界。

```csharp:src/Collision/Geometry/ChainDef.cs [18:55]
public struct ChainDef
{
    public object UserData;
    public Vec2[] Points;
    public int Count;
    public float Friction;
    public float Restitution;
    public Filter Filter;
    public bool IsLoop;
    public int InternalValue;

    public static ChainDef DefaultChainDef()
    {
        return new ChainDef
        {
            Friction = 0.6f,
            Filter = Filter.DefaultFilter(),
            InternalValue = Core.SecretCookie
        };
    }
}
```

使用 `ChainDef` 初始化时，需要提供至少 4 个点。你可以通过 `IsLoop` 来指定是否首尾相连形成闭环。引擎内部会将这些点切分成多个 `ChainSegment`，并存储在一个 `ChainShape` 中统一管理：

```csharp:src/Collision/Geometry/ChainShape.cs [3:16]
public struct ChainShape
{
    public int Id;
    public int BodyId;
    public int NextChainId;
    public int[] ShapeIndices;
    public int Count;
    public ushort Revision;
}
```

链条形状有几个重要限制需要记住：
- 只能用于静态刚体
- 没有质量
- 是单向碰撞（默认逆时针环绕，内部在左侧）
- 开放式链条的首尾两段**没有碰撞**

### 五种形状对比

| 形状 | 内存大小 | 核心数据 | 有质量 | 典型场景 |
|------|---------|---------|--------|---------|
| Circle | 12 字节 | 中心 + 半径 | 是 | 球、轮子、炮弹 |
| Polygon | 144 字节 | 最多 8 顶点 + 法线 | 是 | 箱子、平台、角色躯干 |
| Capsule | 20 字节 | 两个圆心 + 半径 | 是 | 角色碰撞体、胶囊障碍物 |
| Segment | 16 字节 | 两个端点 | 否 | 简化地面、墙壁、平台边缘 |
| ChainShape | 动态 | 多个线段序列 | 否 | 地形边界、管道、跑道 |

## ShapeDef 详解

在 Box2DSharp 里，`ShapeDef` 是创建任何形状时都必须携带的"配置包"。它把密度、摩擦、弹性、过滤器、事件开关等所有属性打包在一起，既方便复用，也能保证初始化安全。

```csharp:src/Collision/Geometry/ShapeDef.cs [8:70]
public struct ShapeDef
{
    public object UserData;

    public float Friction;

    public float Restitution;

    public float Density;

    public Filter Filter;

    public uint CustomColor;

    public bool IsSensor;

    public bool EnableSensorEvents;

    public bool EnableContactEvents;

    public bool EnableHitEvents;

    public bool EnablePreSolveEvents;

    public bool ForceContactCreation;

    public int InternalValue;

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
}
```

文档注释里已经说得很清楚——`ShapeDef` 是一个临时对象，你可以用同一个定义去创建多个形状。但**千万别忘了**先用 `DefaultShapeDef()` 初始化，因为它会在 `InternalValue` 里写入一个"暗号"（`Core.SecretCookie`）。后续调用 `CreateShape` 时，引擎会执行 `def.CheckDef()` 校验，如果暗号不对就会直接断言失败。

这与 C API 里的"忘记初始化导致未定义行为"形成了鲜明对比：Box2DSharp 用一句断言把这类问题扼杀在调试期。

`Density` 的单位是 kg/m²（二维面密度），`Friction` 和 `Restitution` 通常都在 [0, 1] 范围内。`IsSensor` 比较特殊，打开后该形状只参与重叠检测、不产生任何碰撞响应，也不会和别的传感器碰撞。如果你要做触发区域、拾取范围之类的功能，`IsSensor` 就是你的首选。

事件开关家族一共有四个：`EnableSensorEvents`、`EnableContactEvents`、`EnableHitEvents`、`EnablePreSolveEvents`。默认前两个是开启的。需要注意的是，`EnablePreSolveEvents` 开销较大，而且只在动态刚体上生效，慎用。

`Filter` 字段掌管碰撞过滤，默认值来自 `Filter.DefaultFilter()`。它的内部结构我们会在后面"碰撞过滤系统"小节里展开讲，这里只要记住：**如果不手动改 Filter，所有形状默认都会互相碰撞**。

```csharp:src/Collision/Geometry/Filter.cs [6:53]
public struct Filter
{
    public ulong CategoryBits;

    public ulong MaskBits;

    public int GroupIndex;

    public Filter(ulong categoryBits, ulong maskBits, int groupIndex)
    {
        CategoryBits = categoryBits;
        MaskBits = maskBits;
        GroupIndex = groupIndex;
    }

    public static Filter DefaultFilter()
    {
        Filter filter = new(0x0001UL, ulong.MaxValue, 0);
        return filter;
    }
}
```

默认值里 `CategoryBits = 0x0001`、`MaskBits = ulong.MaxValue`，意思就是这个形状把自己归类为第 0 类，并且接受与任何类别的碰撞。`GroupIndex` 为 0 表示不使用分组覆盖逻辑。

## 复合形状 CompoundShapes

现实世界的物体很少是单一几何体。一辆坦克可能是履带加炮塔，一张桌子可能是桌面加四条腿。Box2DSharp 原生支持在一个刚体上挂载多个形状，这就是所谓的**复合形状**。

复合形状有两个核心好处：第一，每个子形状可以有自己的碰撞属性和过滤规则；第二，引擎会自动把所有子形状的质量累加到一起，计算出整个刚体的质心和转动惯量。

来看一个经典示例——用三个矩形拼成一张桌子：

```csharp:test/Testbed.Samples/Shapes/CompoundShapes.cs [26:41]
{
    BodyDef bodyDef = BodyDef.DefaultBodyDef();
    bodyDef.Type = BodyType.DynamicBody;
    bodyDef.Position = (-15.0f, 1.0f);
    Table1Id = Body.CreateBody(WorldId, bodyDef);

    ShapeDef shapeDef = ShapeDef.DefaultShapeDef();
    Polygon top = Geometry.MakeOffsetBox(3.0f, 0.5f, (0.0f, 3.5f), Rot.Identity);
    Polygon leftLeg = Geometry.MakeOffsetBox(0.5f, 1.5f, (-2.5f, 1.5f), Rot.Identity);
    Polygon rightLeg = Geometry.MakeOffsetBox(0.5f, 1.5f, (2.5f, 1.5f), Rot.Identity);

    Shape.CreatePolygonShape(Table1Id, shapeDef, top);
    Shape.CreatePolygonShape(Table1Id, shapeDef, leftLeg);
    Shape.CreatePolygonShape(Table1Id, shapeDef, rightLeg);
}
```

这里调用了三次 `CreatePolygonShape`，但目标 `BodyId` 都是 `Table1Id`。引擎内部会把这三个形状串成一个双向链表挂在刚体上。碰撞检测时，它们各自独立与外部物体产生接触；运动整合时，它们又共享同一个刚体坐标系。

```csharp:src/Collision/Geometry/Shape.cs [182:193]
// Add to shape doubly linked list
if (body.HeadShapeId != Core.NullIndex)
{
    world.ShapeArray.CheckId(body.HeadShapeId);
    var headShape = world.ShapeArray[body.HeadShapeId];
    headShape.PrevShapeId = shapeId;
}

shape.PrevShapeId = Core.NullIndex;
shape.NextShapeId = body.HeadShapeId;
body.HeadShapeId = shapeId;
body.ShapeCount += 1;
```

有时候复合形状还会用到凸包拆分。比如下面的"飞船"由左右两个三角形多边形组成，因为单个四边形可能不够贴合，或者你需要在碰撞时区分左翼和右翼：

```csharp:test/Testbed.Samples/Shapes/CompoundShapes.cs [60:84]
// Spaceship 1
{
    BodyDef bodyDef = BodyDef.DefaultBodyDef();
    bodyDef.Type = BodyType.DynamicBody;
    bodyDef.Position = (5.0f, 1.0f);
    Ship1Id = Body.CreateBody(WorldId, bodyDef);

    ShapeDef shapeDef = ShapeDef.DefaultShapeDef();
    Vec2[] vertices = new Vec2[3];

    vertices[0] = (-2.0f, 0.0f);
    vertices[1] = (0.0f, 4.0f / 3.0f);
    vertices[2] = (0.0f, 4.0f);
    Hull hull = HullFunc.ComputeHull(vertices, 3);
    Polygon left = Geometry.MakePolygon(hull, 0.0f);

    vertices[0] = (2.0f, 0.0f);
    vertices[1] = (0.0f, 4.0f / 3.0f);
    vertices[2] = (0.0f, 4.0f);
    hull = HullFunc.ComputeHull(vertices, 3);
    Polygon right = Geometry.MakePolygon(hull, 0.0f);

    Shape.CreatePolygonShape(Ship1Id, shapeDef, left);
    Shape.CreatePolygonShape(Ship1Id, shapeDef, right);
}
```

注意，每个子形状在创建的时候都会单独进入宽相（BroadPhase），所以形状越多，空间查询开销也会越大。不过对于一般游戏对象来说，三五个形状的复合体完全在性能可接受范围内。

## 运行时修改形状

游戏玩得多了你就知道，角色的体型会变大变小，地面的材质会从冰变成沙子。Box2DSharp 允许你在运行时修改形状属性甚至直接替换几何体。

密度、摩擦和弹性的修改最简单：

```csharp:src/Collision/Geometry/Shape.cs [879:937]
public static void SetDensity(ShapeId shapeId, float density)
{
    Debug.Assert(B2Math.IsValid(density) && density >= 0.0f);

    var world = World.GetWorldLocked(shapeId.World0);

    Shape shape = world.GetShape(shapeId);
    if (density == shape.Density)
    {
        // early return to avoid expensive function
        return;
    }

    shape.Density = density;
}

public static float GetDensity(ShapeId shapeId)
{
    World world = World.GetWorld(shapeId.World0);
    Shape shape = world.GetShape(shapeId);
    return shape.Density;
}

public static void SetFriction(ShapeId shapeId, float friction)
{
    Debug.Assert(B2Math.IsValid(friction) && friction >= 0.0f);

    World world = World.GetWorld(shapeId.World0);
    Debug.Assert(world.Locked == false);
    if (world.Locked)
    {
        return;
    }

    Shape shape = world.GetShape(shapeId);
    shape.Friction = friction;
}

public static void SetRestitution(ShapeId shapeId, float restitution)
{
    Debug.Assert(B2Math.IsValid(restitution) && restitution >= 0.0f);

    World world = World.GetWorld(shapeId.World0);
    Debug.Assert(world.Locked == false);
    if (world.Locked)
    {
        return;
    }

    Shape shape = world.GetShape(shapeId);
    shape.Restitution = restitution;
}
```

但是这里有一个**大坑**：`SetDensity` 只是改了形状对象上的数值，它并不会自动重新计算刚体的质量！对比一下 `CreateShape` 的源码：

```csharp:src/Collision/Geometry/Shape.cs [214:217]
if (body.AutomaticMass == true)
{
    Body.UpdateBodyMassData(world, body);
}
```

只有在**创建**形状时才会触发自动质量更新。运行时修改密度后，如果你希望刚体质心、质量和转动惯量跟着变，必须手动调用 `Body.ApplyMassFromShapes(bodyId)`。否则就会出现"密度变了，但刚体还是轻飘飘"的诡异现象。

Box2DSharp 甚至提供了一套 API 让你在运行时把形状从圆形改成多边形、从胶囊改成线段：

```csharp:src/Collision/Geometry/Shape.cs [1133:1187]
public static void SetCircle(ShapeId shapeId, Circle circle)
{
    var world = World.GetWorldLocked(shapeId.World0);

    Shape shape = world.GetShape(shapeId);
    shape.Union.Circle = circle;
    shape.Type = ShapeType.CircleShape;

    // need to wake bodies so they can react to the shape change
    bool wakeBodies = true;
    bool destroyProxy = true;
    ResetProxy(world, shape, wakeBodies, destroyProxy);
}

public static void SetCapsule(ShapeId shapeId, Capsule capsule)
{
    var world = World.GetWorldLocked(shapeId.World0);

    Shape shape = world.GetShape(shapeId);
    shape.Union.Capsule = capsule;
    shape.Type = ShapeType.CapsuleShape;

    // need to wake bodies so they can react to the shape change
    bool wakeBodies = true;
    bool destroyProxy = true;
    ResetProxy(world, shape, wakeBodies, destroyProxy);
}

public static void SetSegment(ShapeId shapeId, Segment segment)
{
    var world = World.GetWorldLocked(shapeId.World0);

    Shape shape = world.GetShape(shapeId);
    shape.Union.Segment = segment;
    shape.Type = ShapeType.SegmentShape;

    // need to wake bodies so they can react to the shape change
    bool wakeBodies = true;
    bool destroyProxy = true;
    ResetProxy(world, shape, wakeBodies, destroyProxy);
}

public static void SetPolygon(ShapeId shapeId, Polygon polygon)
{
    var world = World.GetWorldLocked(shapeId.World0);

    Shape shape = world.GetShape(shapeId);
    shape.Union.Polygon = polygon;
    shape.Type = ShapeType.PolygonShape;

    // need to wake bodies so they can react to the shape change
    bool wakeBodies = true;
    bool destroyProxy = true;
    ResetProxy(world, shape, wakeBodies, destroyProxy);
}
```

注意，这四个方法都会调用 `ResetProxy`，它会先销毁该形状相关的所有接触（Contact），再重建宽相代理。也就是说，**几何修改会立刻生效**，但也会带来一定的性能开销。不要在 `Update` 里每帧无意义地调用它们。

下面这个示例来自 `ModifyGeometry`，展示了如何根据用户选择实时切换形状，并在切换后重新计算质量：

```csharp:test/Testbed.Samples/Shapes/ModifyGeometry.cs [50:81]
protected void UpdateShape()
{
    switch (ShapeType)
    {
    case ShapeType.CircleShape:
        Circle = ((0.0f, 0.0f), 0.5f * Scale);
        Shape.SetCircle(ShapeId, Circle);
        break;

    case ShapeType.CapsuleShape:
        Capsule = ((-0.5f * Scale, 0.0f), (0.0f, 0.5f * Scale), 0.5f * Scale);
        Shape.SetCapsule(ShapeId, Capsule);
        break;

    case ShapeType.SegmentShape:
        Segment = ((-0.5f * Scale, 0.0f), (0.75f * Scale, 0.0f));
        Shape.SetSegment(ShapeId, Segment);
        break;

    case ShapeType.PolygonShape:
        Polygon = Geometry.MakeBox(0.5f * Scale, 0.75f * Scale);
        Shape.SetPolygon(ShapeId, Polygon);
        break;

    default:
        Debug.Assert(false);
        break;
    }

    BodyId bodyId = Shape.GetBody(ShapeId);
    Body.ApplyMassFromShapes(bodyId);
}
```

切换完成后，第 80 行的 `Body.ApplyMassFromShapes(bodyId)` 保证了刚体质量与新几何保持同步。如果你忘了这一步，动态刚体就会继续沿用旧的质量数据，导致受力行为异常。

```csharp:src/Body/Body.cs [1284:1290]
public static void ApplyMassFromShapes(BodyId bodyId)
{
    var world = World.GetWorldLocked(bodyId.World0);

    Body body = GetBodyFullId(world, bodyId);
    UpdateBodyMassData(world, body);
}
```

最后提醒一点：`SetFriction` 和 `SetRestitution` 在内部都会检查 `world.Locked`，如果当前世界正处于物理步的锁定状态（比如在碰撞回调里），这两个调用会被静默忽略。所以尽量不要在接触事件回调中直接修改这些属性，而是把请求缓存到下一帧再处理。

## 设计思考

Box2DSharp 的 Polygon 结构体上有一行注释写得很直接："DO NOT fill this out manually"——别手动填。这其实透露了一个重要的设计前提：引擎内部对凸多边形有严格的假设，用户必须通过 `b2MakePolygon` 或 `b2MakeBox` 这类工厂函数来创建。为什么做这么强的限制？答案藏在碰撞检测的效率和稳定性里。

凸多边形有一个特别好的性质：任意两点连线都在形状内部。这意味着 Separating Axis Theorem（SAT）可以毫无例外地适用——你只需要测试所有 edge normal 方向上的投影，如果存在任何一个方向上的间隙，两个形状就不碰撞。这个算法的复杂度是 O(m+n)，而且实现非常规整，没有凹多边形那种"内部空洞导致投影重叠但仍不碰撞"的 corner case。Box2DSharp 把时间步预算看得很紧，它不愿意在碰撞检测里处理分支复杂、数据不规整的通用多边形。所以不是不能做凹多边形，而是做了之后，性能的确定性和代码的可维护性都会大打折扣。如果你确实需要凹的外形，复合形状（一个 Body 挂多个 Polygon/Circle）就是官方推荐的替代方案。

接下来说说 ChainShape。它的存在感比 Circle、Polygon 低很多，但却是地形系统的"秘密武器"。`ChainDef` 的注释开门见山："designed to eliminate ghost collisions with some limitations"——专门用来消除幽灵碰撞，但有很多限制。什么是幽灵碰撞？想象一个由很多小线段拼成的地面，一个圆形角色沿着它滚动。滚到两个线段的连接处时，圆的重心可能刚好跨过相邻线段的边界，导致引擎错误地认为圆"撞"到了内侧的角，产生一个向上的跳动感。ChainShape 的解法很聪明：每个内部线段不是孤立存在的，它被包装成一个 `ChainSegment`，带有 `Ghost1` 和 `Ghost2` 两个"幽灵顶点"。这两个幽灵顶点不直接参与碰撞，但会为相邻线段提供平滑的过渡参考，让碰撞法线在连续线段之间自然插值，从而避免角色被接缝处的尖角弹飞。

不过 ChainShape 的限制也很鲜明：它默认是 one-sided 的（只有右侧会碰撞），没有质量（只能放在 static body 上），首尾两条边在 open chain 情况下甚至不参与碰撞。这些限制不是缺陷，而是刻意的 trade-off——ChainShape 的服务目标非常明确：高效、平滑、无幽灵碰撞的静态地形边界。如果你把它当成通用多边形来用，就会踩到这些限制设置的边界。

在内存层面，`Shape` 类用一个 `UnionShape` 联合体把 Circle、Capsule、Polygon、Segment、ChainSegment 叠在同一地址上。这种 C 风格的 union 让每种形状的内存开销都等于最大成员（Polygon，144 字节），换取的是 Shape 数组的完全同质化和缓存友好性。ShapeArray、BroadPhase tree、Contact graph 里流转的 Shape 指针/索引都是统一大小，不需要虚函数分发，也不需要运行时类型判断的额外开销。对于追求稳定帧率的物理引擎来说，这种"用空间换一致性"的选择是非常典型的工程取舍。

## 小结

- Box2DSharp 的 Polygon 严格限定为凸多边形，核心原因是 SAT 碰撞检测在凸形状上具有 O(m+n) 的确定性复杂度，而凹多边形会引入大量 corner case 和性能歧义。
- `HullFunc.ComputeHull` 和 `ValidateHull` 提供了自动化的凸包计算与验证，开发者不需要手动保证顶点顺序和凸性；需要凹外形时，应使用一个 Body 挂载多个形状的复合形状方案。
- `ChainShape` 的设计目标是消除静态地形中由多个线段拼接带来的幽灵碰撞，通过 `ChainSegment` 的 Ghost 顶点实现相邻线段间碰撞法线的平滑过渡。
- ChainShape 被明确限制为 one-sided、无质量、通常挂载在 static body 上；它不是通用形状，而是地形边界的专用工具，open chain 的首尾边甚至默认不参与碰撞。
- `UnionShape` 使用内存联合体将所有几何变体重叠存储，保证了 Shape 数组的同质布局，消除了虚函数和运行时类型分发的开销，是缓存友好型物理引擎的典型设计。
