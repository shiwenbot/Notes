---
title: "第11章：综合实战——从示例中学习"
date: 2026-04-14
categories:
  - 代码库
tags:
  - Box2D
description: "从简单示例到复杂系统的递进实战"
---

## 概述：从简单示例到复杂系统的递进

前面十章我们把 Box2DSharp 的核心概念逐个拆解了一遍。但真实项目里，这些概念是组合在一起使用的。从一颗单独下落的盒子，到一辆能在崎岖地形上奔驰的汽车，再到由十几个关节联动的布娃娃——复杂度每上一层，考验的都是你对基础 API 的熟悉程度和对系统架构的把控力。

本章我们走进 Box2DSharp 自带的示例代码，用"效果描述 → 架构分析 → 关键代码 → 参数调优"的思路，带你把散落的知识点串成线。你会看到 WheelJoint 如何撑起一辆车的悬挂，RevoluteJoint 怎样约束布娃娃的四肢，以及软体、桥梁、连续碰撞检测等高级话题在实战中是怎么落地的。

## 车辆系统：底盘 + 两轮 + WheelJoint + MotorJoint

车辆的物理建模是 Box2D 示例里最经典、也最贴近游戏的案例之一。它的核心思路非常清晰：用一个多边形刚体当底盘，两个圆形刚体当车轮，然后用 WheelJoint 把车轮"挂"在底盘上。WheelJoint 同时承担了悬挂弹簧和驱动马达两个职责，所以一辆车只需要三个刚体、两个关节就能跑起来。

### 整体架构

Car 类的结构非常干净。它用五个字段分别记录底盘、前后轮的 BodyId，以及前后轴的 JointId：

```csharp:test/ConsoleTest/Base/Car.cs [6:18]
public class Car
{
    public BodyId ChassisId;

    public BodyId RearWheelId;

    public BodyId FrontWheelId;

    public JointId RearAxleId;

    public JointId FrontAxleId;

    public bool IsSpawned;
...
```

这种显式暴露 ID 的设计，方便外部系统（比如游戏逻辑层）直接查询车辆状态，而不必绕过 Car 实例。在一套需要复用物理实体的框架里，这种写法很常见。

### 底盘与车轮的创建

Spawn 方法负责一次性把车组装好。底盘是一个经过缩放的六边形 Polygon，车轮则是两个 Circle：

```csharp:test/ConsoleTest/Base/Car.cs [38:72]
        Vec2[] vertices = [(-1.5f, -0.5f), (1.5f, -0.5f), (1.5f, 0.0f), (0.0f, 0.9f), (-1.15f, 0.9f), (-1.5f, 0.2f),];

        for (int i = 0; i < 6; ++i)
        {
            vertices[i].X *= 0.85f * scale;
            vertices[i].Y *= 0.85f * scale;
        }

        Hull hull = HullFunc.ComputeHull(vertices, 6);
        Polygon chassis = Geometry.MakePolygon(hull, 0.15f * scale);

        ShapeDef shapeDef = ShapeDef.DefaultShapeDef();
        shapeDef.Density = 1.0f / scale;
        shapeDef.Friction = 0.2f;

        Circle circle = ((0.0f, 0.0f), 0.4f * scale);

        BodyDef bodyDef = BodyDef.DefaultBodyDef();
        bodyDef.Type = BodyType.DynamicBody;
        bodyDef.Position = (0.0f, 1.0f * scale) + position;
        ChassisId = Body.CreateBody(worldId, bodyDef);
        Shape.CreatePolygonShape(ChassisId, shapeDef, chassis);

        shapeDef.Density = 2.0f / scale;
        shapeDef.Friction = 1.5f;

        bodyDef.Position = ((-1.0f * scale, 0.35f * scale) + position);
        bodyDef.AllowFastRotation = true;
        RearWheelId = Body.CreateBody(worldId, bodyDef);
        Shape.CreateCircleShape(RearWheelId, shapeDef, circle);

        bodyDef.Position = ((1.0f * scale, 0.4f * scale) + position);
        bodyDef.AllowFastRotation = true;
        FrontWheelId = Body.CreateBody(worldId, bodyDef);
        Shape.CreateCircleShape(FrontWheelId, shapeDef, circle);
```

注意几个细节：
- `AllowFastRotation = true` 给车轮开启快速旋转优化，减少高速旋转时的数值误差。
- 车轮密度（2.0f/scale）比底盘（1.0f/scale）更高，让车的重心更低、行驶更稳。
- 车轮摩擦系数拉到 1.5f，底盘只有 0.2f，这样抓地力主要来自轮胎，而不是车身拖地。

### WheelJoint：悬挂 + 驱动的双重身份

WheelJoint 的关键在于它允许 bodyB（车轮）沿着一条本地坐标轴相对 bodyA（底盘）平移，同时又能绕锚点旋转。换句话说，它既约束了车轮不能左右乱飞，又允许上下弹跳——这正好就是悬挂的物理抽象。

前后轮共用一套 WheelJointDef 模板，只是锚点不同：

```csharp:test/ConsoleTest/Base/Car.cs [83:114]
        WheelJointDef jointDef = WheelJointDef.DefaultWheelJointDef();

        jointDef.BodyIdA = ChassisId;
        jointDef.BodyIdB = RearWheelId;
        jointDef.LocalAxisA = Body.GetLocalVector(jointDef.BodyIdA, axis);
        jointDef.LocalAnchorA = Body.GetLocalPoint(jointDef.BodyIdA, pivot);
        jointDef.LocalAnchorB = Body.GetLocalPoint(jointDef.BodyIdB, pivot);
        jointDef.MotorSpeed = 0.0f;
        jointDef.MaxMotorTorque = torque;
        jointDef.EnableMotor = true;
        jointDef.Hertz = hertz;
        jointDef.DampingRatio = dampingRatio;
        jointDef.LowerTranslation = -0.25f * scale;
        jointDef.UpperTranslation = 0.25f * scale;
        jointDef.EnableLimit = true;
        RearAxleId = Joint.CreateWheelJoint(worldId, jointDef);

        pivot = Body.GetPosition(FrontWheelId);
        jointDef.BodyIdA = ChassisId;
        jointDef.BodyIdB = FrontWheelId;
        ...
        FrontAxleId = Joint.CreateWheelJoint(worldId, jointDef);
```

`LocalAxisA` 被设为竖直方向 `(0.0f, 1.0f)`，意味着车轮只能相对底盘上下运动。`LowerTranslation` 和 `UpperTranslation` 限制了悬挂行程（±0.25 * scale），避免车身压到地板上。`EnableMotor` 则让 WheelJoint 同时成为驱动源，后续只要改 `MotorSpeed` 就能控制车速。

### 运行时参数调节接口

Car 类还提供了一组可以在游戏循环里随时调用的方法：

```csharp:test/ConsoleTest/Base/Car.cs [130:153]
    public void SetSpeed(float speed)
    {
        WheelJointFunc.SetMotorSpeed(RearAxleId, speed);
        WheelJointFunc.SetMotorSpeed(FrontAxleId, speed);
        Joint.WakeBodies(RearAxleId);
    }

    public void SetTorque(float torque)
    {
        WheelJointFunc.SetMaxMotorTorque(RearAxleId, torque);
        WheelJointFunc.SetMaxMotorTorque(FrontAxleId, torque);
    }

    public void SetHertz(float hertz)
    {
        WheelJointFunc.SetSpringHertz(RearAxleId, hertz);
        WheelJointFunc.SetSpringHertz(FrontAxleId, hertz);
    }
```

`SetSpeed` 同时设置前后轮马达速度，并调用 `Joint.WakeBodies` 防止车在静止时踩油门没反应。`SetHertz` 和 `SetDampingRadio` 则让你在运行时动态调试悬挂手感——这对寻找"软硬适中"的驾驶体验非常有用。

### WheelJoint 参数调优建议

根据 Testbed 里的 `WheelJointSample`，WheelJoint 的常用参数范围如下：

- **Hertz**：悬挂弹簧频率，一般取 1.0f ~ 5.0f。值越大悬挂越硬，车身越容易弹跳。
- **DampingRatio**：阻尼比，0.5f ~ 1.0f 是舒适区。低于 0.3f 会明显振荡，高于 1.5f 会显得过硬。
- **MaxMotorTorque**：最大扭矩，决定爬坡能力和加速响应。小车 2.5f 左右，越野大脚车可以拉到 10f 以上。
- **MotorSpeed**：目标转速，正负号控制前进/后退。给马达开关时别忘了 `EnableMotor = true`。

## 布娃娃系统：Human 类的 11 段骨骼

布娃娃（Ragdoll）是动作游戏里最常见的物理死亡效果。Box2DSharp 的 `Human` 类把整个身体拆成 11 段 Capsule 刚体，通过 RevoluteJoint 连接成树状层级，并且每段骨骼都限制了关节活动角度，防止肢体做出反人类的扭曲。

### 骨骼层级设计

Human 类内部用了一个 `Bone` 数组来管理 11 段身体，顺序由 `BoneType` 常量定义：

```csharp:test/ConsoleTest/Base/Human.cs [8:42]
public class Human
{
    public class Bone
    {
        public BodyId BodyId;

        public JointId JointId;

        public float FrictionScale;

        public int ParentIndex;
    }

    public struct BoneType
    {
        public const int Hip = 0;
        public const int Torso = 1;
        public const int Head = 2;
        public const int UpperLeftLeg = 3;
        public const int LowerLeftLeg = 4;
        public const int UpperRightLeg = 5;
        public const int LowerRightLeg = 6;
        public const int UpperLeftArm = 7;
        public const int LowerLeftArm = 8;
        public const int UpperRightArm = 9;
        public const int LowerRightArm = 10;
    }

    public const int BoneCount = 11;
```

`Bone` 里的 `ParentIndex` 非常关键，它把骨盆（Hip）设为根节点， torso 挂在 hip 上，head 挂在 torso 上，四肢依次向下级联。这种树状层级符合真实人体的骨骼连接方式，也方便后续做正向/反向运动学扩展。

### 骨盆与躯干：根节点的建立

Spawn 方法从骨盆开始创建。骨盆没有 ParentIndex（值为 -1），是整个布娃娃的物理根：

```csharp:test/ConsoleTest/Base/Human.cs [104:121]
        // hip
        {
            Bone bone = _bones[BoneType.Hip];
            bone.ParentIndex = -1;

            bodyDef.Position = (0.0f, 0.95f * s) + position;

            bodyDef.LinearDamping = 0.0f;
            bone.BodyId = Body.CreateBody(worldId, bodyDef);

            if (colorize)
            {
                shapeDef.CustomColor = (uint)pantColor;
            }

            Capsule capsule = ((0.0f, -0.02f * s), (0.0f, 0.02f * s), 0.095f * s);
            Shape.CreateCapsuleShape(bone.BodyId, shapeDef, capsule);
        }
```

躯干（Torso）则通过 RevoluteJoint 连接到骨盆。注意这里 `EnableLimit = true`，并且设定了 `LowerAngle` 和 `UpperAngle`，把躯干相对骨盆的前倾/后仰限制在一个合理范围内：

```csharp:test/ConsoleTest/Base/Human.cs [123:161]
        // torso
        {
            Bone bone = _bones[BoneType.Torso];
            bone.ParentIndex = BoneType.Hip;

            bodyDef.Position = (0.0f, 1.2f * s) + position;
            bodyDef.LinearDamping = 0.0f;

            bone.BodyId = Body.CreateBody(worldId, bodyDef);
            bone.FrictionScale = 0.5f;
            bodyDef.Type = BodyType.DynamicBody;

            if (colorize)
            {
                shapeDef.CustomColor = (uint)shirtColor;
            }

            Capsule capsule = ((0.0f, -0.135f * s), (0.0f, 0.135f * s), 0.09f * s);
            Shape.CreateCapsuleShape(bone.BodyId, shapeDef, capsule);

            Vec2 pivot = (0.0f, 1.0f * s) + position;
            RevoluteJointDef jointDef = RevoluteJointDef.DefaultRevoluteJointDef();
            jointDef.BodyIdA = _bones[bone.ParentIndex].BodyId;
            jointDef.BodyIdB = bone.BodyId;
            jointDef.LocalAnchorA = Body.GetLocalPoint(jointDef.BodyIdA, pivot);
            jointDef.LocalAnchorB = Body.GetLocalPoint(jointDef.BodyIdB, pivot);
            jointDef.EnableLimit = enableLimit;
            jointDef.LowerAngle = -0.25f * B2Math.Pi;
            jointDef.UpperAngle = 0.0f;
            jointDef.EnableMotor = enableMotor;
            jointDef.MaxMotorTorque = bone.FrictionScale * maxTorque;
            jointDef.EnableSpring = hertz > 0.0f;
            jointDef.Hertz = hertz;
            jointDef.DampingRatio = dampingRatio;
            jointDef.DrawSize = drawSize;

            bone.JointId = Joint.CreateRevoluteJoint(worldId, jointDef);
        }
```

骨盆到躯干的关节限制在 -0.25π ~ 0.0f 之间，意味着躯干可以前倾（向后仰被锁住），这很符合人体弯腰的自然姿态。RevoluteJoint 的锚点放在 `(0.0f, 1.0f * s)` 处，也就是骨盆和躯干的交界处，保证了旋转中心正确。

### 头部与颈部连接

头部（Head）挂在躯干上，ParentIndex 设为 `BoneType.Torso`。为了模拟颈部视觉上的过渡，头部刚体除了本身的球形胶囊外，还额外加了一段短胶囊作为脖子：

```csharp:test/ConsoleTest/Base/Human.cs [163:203]
        // head
        {
            Bone bone = _bones[BoneType.Head];
            bone.ParentIndex = BoneType.Torso;

            bodyDef.Position = (0.0f * s, 1.5f * s) + position;
            bodyDef.LinearDamping = 0.1f;

            bone.BodyId = Body.CreateBody(worldId, bodyDef);
            bone.FrictionScale = 0.25f;

            if (colorize)
            {
                shapeDef.CustomColor = (uint)skinColor;
            }

            Capsule capsule = ((0.0f, -0.0325f * s), (0.0f, 0.0325f * s), 0.08f * s);
            Shape.CreateCapsuleShape(bone.BodyId, shapeDef, capsule);

            // neck
            capsule = ((0.0f, -0.12f * s), (0.0f, -0.08f * s), 0.05f * s);
            Shape.CreateCapsuleShape(bone.BodyId, shapeDef, capsule);

            Vec2 pivot = (0.0f, 1.4f * s) + position;
            RevoluteJointDef jointDef = RevoluteJointDef.DefaultRevoluteJointDef();
            jointDef.BodyIdA = _bones[bone.ParentIndex].BodyId;
            jointDef.BodyIdB = bone.BodyId;
            jointDef.LocalAnchorA = Body.GetLocalPoint(jointDef.BodyIdA, pivot);
            jointDef.LocalAnchorB = Body.GetLocalPoint(jointDef.BodyIdB, pivot);
            jointDef.EnableLimit = enableLimit;
            jointDef.LowerAngle = -0.3f * B2Math.Pi;
            jointDef.UpperAngle = 0.1f * B2Math.Pi;
            jointDef.EnableMotor = enableMotor;
            jointDef.MaxMotorTorque = bone.FrictionScale * maxTorque;
            jointDef.EnableSpring = hertz > 0.0f;
            jointDef.Hertz = hertz;
            jointDef.DampingRatio = dampingRatio;
            jointDef.DrawSize = drawSize;

            bone.JointId = Joint.CreateRevoluteJoint(worldId, jointDef);
        }
```

颈部角度限制在 -0.3π ~ 0.1π，低头幅度大于抬头，这也是符合人体工学的。每一个关节都统一启用了马达（`EnableMotor = true`），但 `MaxMotorTorque` 并不高，实际效果是让肢体在碰撞后保持微阻尼的回正，而不是硬邦邦地卡住。如果你把 `hertz` 设成 0，`EnableSpring` 就会关闭，整具布娃娃会变得软塌塌、更像纯粹的 ragdoll。

## 布娃娃系统

前面我们用 `Human.cs` 搭出了 11 段骨骼，但真正的细节在于关节怎么约束、刚体怎么归类，以及怎么让布娃娃从"睡着"变成"摔醒"。

角度限制是布娃娃不生硬扭曲的核心。以下肢为例，大腿相对髋部的屈伸范围被限制在 `-0.05π` 到 `0.4π`，小腿相对大腿则是 `-0.5π` 到 `-0.02π`。这里用到了 `EnableLimit` 和 `LowerAngle/UpperAngle`。


同时每个关节都启用了马达 friction，并给了一个按比例缩小的 `MaxMotorTorque`，用来模拟肌肉阻尼：


碰撞过滤方面，`Human` 将所有骨头的 `Filter.GroupIndex` 设为同一个负数，这样同一具身体的部位之间不会发生内部碰撞。脚部还单独设置了低摩擦的 `footShapeDef`。Ragdoll 示例则在创建后调用 `ApplyRandomAngularImpulse`，给躯干一个随机角冲量，把整套骨骼从休眠中"摔醒"：


如果你在做游戏，建议把 `SleepThreshold` 设得稍高一些（比如 0.1），并在布娃娃倒地后统一设置关节摩擦为零，让尸体慢慢静止。

## 桥梁与链条

桥梁和链条是 RevoluteJoint 链最典型的应用场景：把一段段刚体串起来，两端锚定到地面。

### Bridge：160 块木板的吊桥

Bridge 示例创建了 160 个长方形 `Polygon` 作为桥面，每块之间用 RevoluteJoint 链接。注意关节的 pivot 点都落在相邻两块木板的交界处，这样才能让桥面像真实链条一样弯曲：


为了让桥梁不过度晃荡，中间每个关节都开启了马达摩擦，`MaxMotorTorque = 200.0f`。首尾两块则分别固定到地面上，形成"两端接地、中间悬空"的结构。示例最后还丢了几个三角形和圆球上去，用来测试桥面承载重物时的表现。

### BallAndChain：流星锤式的球链

BallAndChain 演示了另一种链式结构：30 段 Capsule 刚体连成链条，末端挂一个巨大的圆球。它的关节位置和 Bridge 类似，但每段刚体用的是 Capsule：


有意思的点是末端的"重锤"关节额外开启了马达：


这让整个链条在摆动时带有一定的阻尼感，不会无限震荡。如果你想做鞭子、绳索或者吊灯，这个模式可以直接套用：链节用轻刚体，末端挂重物，关节给少量摩擦。

## 软体物理

SoftBody 的代码本身非常简洁，因为它把核心逻辑委托给了 `Donut` 类。Donut 展示了一种最直观的软体思路：用一圈刚体围成环形，再用 WeldJoint 把它们"缝合"起来，形成一个可变形但保持拓扑的整体。

### Donut 的环形结构

Donut 由 7 个 Capsule 组成，每个 Capsule 沿圆周均匀分布，并且通过自身的 `Rotation` 对准圆心：


关节部分使用的是 WeldJoint，而不是 RevoluteJoint。WeldJoint 会同时约束位置和旋转，把相邻两段骨架"焊接"在一起。为了允许适度的弹性弯曲，Donut 给 WeldJoint 设置了 `AngularHertz = 5.0f`，但 `AngularDampingRatio = 0.0f`：


注意锚点设计：`LocalAnchorA = (0, 0.5 * length)` 和 `LocalAnchorB = (0, -0.5 * length)`，刚好对应每段 Capsule 的两端，这样 7 段就能头尾无缝相接。

### SoftBody 示例的调用

SoftBody 本身只是创建了地面和一个 Donut：


这种"小单元 + 弹性焊接"的思路，就是 2D 软体通用的入门级方案。你完全可以把圆环改成三角形网格，用 WeldJoint 或 DistanceJoint 连接顶点，就能拼出果冻、布料等效果。

## 高级关节

接下来看三个把关节用到极致的复杂机械系统：剪式升降机、悬臂梁和驾驶地形。

### ScissorLift：剪式升降机

ScissorLift 的本质是一组交叉杆件组成的菱形机构，通过层层叠加实现升降。每一层有两根 Capsule，以相反的小角度（±0.15 弧度）交叉放置：


三根关键关节把每层固定起来：左右两端的铰链（RevoluteJoint）加上中间交叉点的铰链。最底下右侧用了 WheelJoint，允许水平滑动：


升降动力来自顶部的 DistanceJoint，它连接地面和中间某根杆件，带有弹簧和马达：


为了让升降动作更平滑，这个示例特意把 `SubStepCount` 提升到 8。如果你的机械结构也有大量重叠约束，增加子步数是首选优化。

### Cantilever：悬臂梁

Cantilever 用 WeldJoint 把 8 段 Capsule 从左到右焊接成一根悬臂，左端固定在地面上：


WeldJoint 在这里不仅仅是"焊死"，还被赋予了弹簧属性：`LinearHertz = 15.0f`、`AngularHertz = 5.0f`。这让长梁在受力时会出现明显的弯曲和抖动，而不是像一根绝对刚性的铁杆。运行示例时，顶端 Y 坐标会实时打印出来，你可以直观看到悬臂的挠度变化。

### Driving：复杂地形驾驶

Driving 是一个完整的可交互场景，结合了车辆、桥梁、跷跷板和坡度地形。它的重点不是某种单一关节，而是如何把多种物理元素拼接成一条"赛道"。

车辆本身调用 `Car.Spawn`，而我们已经在本章前半部分讲过它的原理。地形部分使用 `ChainDef` 拼接出起伏路面：


桥面和跷跷板依然是 RevoluteJoint 链：


键盘控制逻辑把 `A/S/D` 映射为前进、刹车、倒车，并实时调用 `m_car.SetSpeed`：


这个示例教会我们一件事：复杂的物理关卡不需要"神奇的新 API"，只要把 Chain、RevoluteJoint、WheelJoint 和 MotorJoint 组合好，就能做出非常有手感的交互场景。

## 大型世界：场景管理的性能数据与优化思路

`LargeWorld` 是 Box2DSharp 中规模最大的综合测试场景。它在一条超长的正弦地形上同时布置了地面网格、堆叠方块、Human 布娃娃、Donut 软体和一辆可操控的汽车，周期数达到 600 个，实体总量轻松破万。

这个场景最关心的不是"能不能跑"，而是"怎么跑得稳"。第一个优化点藏在静态地面的创建逻辑里：`ShapeDef.ForceContactCreation` 被显式设为 `false`，这样可以显著降低创建大量静态形状时的开销。


第二个优化点体现在 body 的切分策略上。代码每 10 个地面格子就新建一个 ground body，而不是把所有地形塞到同一个 body 里。原因是 Box2D 虽然大多在局部坐标运算，但接触点最终要相对于 body 原点计算；body 离原点越远，浮点精度损失越明显。


通过这种"分桶"策略，远处地形既不会浪费模拟资源，也不会出现奇怪的抖动。对于你的游戏来说，这意味着开放大地图完全可以按区块拆分 body，而不是一股脑扔进一个"世界中心"。

## 连续碰撞检测实战：BounceHouse 与 Pinball

高速小物体穿透厚墙体，是离散时间步模拟的经典噩梦。Box2D 用连续碰撞检测（CCD）来解决，而 `BounceHouse` 和 `Pinball` 就是两套最直观的实战教材。

`BounceHouse` 的核心是一个在密闭房间里高速弹射的刚体。它会不断改变形状（圆形、胶囊、长条盒）来测试 CCD 的鲁棒性。圆形由于对称，可以开得再快也不怕"转穿墙"，所以代码里对圆 shape 专门开启了 `AllowFastRotation`：


同时 `shapeDef.EnableHitEvents = true` 让你在渲染阶段就能读取碰撞切入速度和碰撞点，非常适合做命中反馈或者音效触发。

如果说 `BounceHouse` 是"测试CCD极限"，那 `Pinball` 就是"把CCD用在游戏里"的典范。弹球速度很快，如果不用 CCD，小球很容易直接穿过 bumper 或者直接从 flippers 中间漏下去。代码里对球体 body 设置了 `IsBullet = true`，强制引擎对它做连续碰撞检测：


弹板部分则用两个带马达的 `RevoluteJoint` 实现。按下空格时马达速度翻转，弹板猛地扬起；松开时自动回弹。整个逻辑不到 30 行，却完整呈现了街机弹珠台的核心物理交互。


## 设计思考：复杂物理系统的架构设计

当你开始组装车辆、布娃娃、桥梁、软体这些复杂机制时，"堆砌刚体和关节"很快就会变成维护灾难。真正可维护的做法是：把每一个游戏机制封装成一个独立的"物理装置"类，对外只暴露 `Spawn`、`Update` 和 `Destroy` 三个生命周期接口。

比如 `Car`、`Human`、`Donut` 都是这种思路的产物。它们内部自己管理 body 列表、joint 引用和 shape 配置，调用方只需要知道"在某某位置生成一辆车"就行。这样上层游戏逻辑完全不需要关心底盘和轮子的具体连接方式。

传感器（sensor shape）是另一个常被低估的架构利器。它不参与物理碰撞响应，但能持续产生接触事件，非常适合做"触发区域"。你可以用 sensor 检测车辆是否离地、角色是否站在平台上、或者弹球是否经过得分轨道，而不必在碰撞回调里写一堆 `if-else` 过滤逻辑。

最后，建议把所有"纯视觉"参数（颜色、缩放、特效）与物理实体分离。物理层只负责模拟正确的运动，渲染层根据 body 的 transform 去绘制。这样即使以后更换渲染方案，物理代码也完全不用动。

## 小结

本章我们沿着 Box2DSharp 的示例代码，把前 10 章的知识点串成了一条完整的实战链条。从 `Car` 的 `WheelJoint` 和 `MotorJoint`，到 `Human` 的 `RevoluteJoint` 骨骼约束；从 `Bridge` 的链式结构，到 `Donut` 的弹簧网络；再到 `LargeWorld` 的性能切分和 `Pinball` 的 CCD 应用——每一个示例都不是孤立的技术点，而是多个基础概念的有机组合。

如果你能从这些示例里提炼出两条最重要的经验，那应该是：第一，复杂机制要封装成高内聚的"装置"类；第二，性能问题要在设计阶段就通过 body 拆分、关闭不必要的接触创建和合理使用睡眠机制来预防。掌握了这两条，你就可以开始在自己的项目里搭建真正的物理驱动游戏世界了。
