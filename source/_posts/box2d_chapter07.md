---
title: "第7章：关节系统——约束刚体的相对运动"
date: 2026-04-14
categories:
  - 书籍
tags:
  - Box2D
description: "Joint关节约束、约束力与约束力矩、常见关节类型"
---

## 概述

在 Box2DSharp 中，**关节（Joint）** 的作用是约束两个刚体之间的相对运动。它像是隐形的绳索、铰链或轨道，让物体既能保持特定的空间关系，又不会完全失去物理互动。从物理本质上讲，关节通过施加约束力（或约束力矩）来修正位置偏差，使两个刚体的一部分自由度被"锁定"在期望的状态。

Box2DSharp 一共提供了七种关节类型：DistanceJoint、RevoluteJoint、PrismaticJoint、WheelJoint、WeldJoint、MotorJoint 和 MouseJoint。你可能用它们来做吊桥铰链、车轮悬挂、弹簧抓钩，甚至是类似抓娃娃机的鼠标拖拽效果。每种关节共享一些基础概念，比如锚点、碰撞连接和弹簧阻尼参数，但各自约束的自由度不同。接下来我们先搞清楚这些通用概念，再逐一拆解每种关节的用法。

## 关节的通用概念

### 锚点与局部坐标

每个关节都通过两个锚点连接刚体：`LocalAnchorA` 和 `LocalAnchorB`。它们分别定义在 `BodyA` 和 `BodyB` 的局部坐标系下。也就是说，无论刚体怎么旋转平移，锚点始终跟随刚体一起运动。这样设计有两个好处：一是你不需要在建模时就精确知道质心位置；二是游戏存档读档时，关节的初始配置可以允许一点点误差，不会因为浮点精度而立刻断裂。

```csharp:src/Joints/JointSim.cs [15:18]
        // Anchors relative to body origin
        public Vec2 LocalOriginAnchorA;

        public Vec2 LocalOriginAnchorB;
```

### BodyIdA 与 BodyIdB

所有关节定义结构（`DistanceJointDef`、`RevoluteJointDef` 等）都以 `BodyIdA` 和 `BodyIdB` 开头。这两个字段告诉世界："我要把这两具刚体拴在一起"。比如下面这段代码创建距离关节时，会先验证这两个 `BodyId` 是否有效，然后再去查表找到对应的 `Body` 实体。

```csharp:src/Joints/Joint.cs [332:352]
        public static JointId CreateDistanceJoint(WorldId worldId, in DistanceJointDef def)
        {
            def.CheckDef();
            World world = World.GetWorldFromId(worldId);

            Debug.Assert(world.Locked == false);

            if (world.Locked)
            {
                return new JointId();
            }

            Debug.Assert(def.BodyIdA.IsValid());
            Debug.Assert(def.BodyIdB.IsValid());
            Debug.Assert(B2Math.IsValid(def.Length) && def.Length > 0.0f);

            Body bodyA = Body.GetBodyFullId(world, def.BodyIdA);
            Body bodyB = Body.GetBodyFullId(world, def.BodyIdB);
```

### CollideConnected：允许或禁止碰撞

`CollideConnected` 是一个布尔值，默认通常是 `false`。如果它为 `false`，则被关节连接的两个刚体之间不会生成接触点，也就是说它们可以穿透彼此。这在吊桥、绳索、人体骨架等场景中很重要——你当然不希望两个被铰链连在一起的木块互相弹开。

反过来，如果你把 `CollideConnected` 设为 `true`，这两个刚体就可以发生碰撞。这个标志在运行时也能改，但改变它可能会需要触发布相检测或销毁已有的接触。

```csharp:src/Joints/JointEdge.cs [8:15]
    public struct JointEdge
    {
        public int BodyId;

        public int PrevKey;

        public int NextKey;
    }
```

关节的边缘（`JointEdge`）会被挂到两个刚体的双向链表里，所以物理引擎能通过一个刚体快速找到它身上所有的关节。

### 约束力与约束扭矩

每种关节内部都会累积"冲量"（Impulse），用来修正相对位置或速度偏差。如果你想在运行时读取关节对刚体施加了多大的力或扭矩，Box2DSharp 提供了两个全局 API：

- `Joint.GetConstraintForce(jointId)` —— 返回线性的约束力（`Vec2`）
- `Joint.GetConstraintTorque(jointId)` —— 返回角向的约束扭矩（`float`）

它们会内部查表，根据关节类型调用对应的 `*Func` 实现。注意 `DistanceJoint` 没有扭矩分量，因为它只约束距离。

```csharp:src/Joints/Joint.cs [896:929]
        public static Vec2 GetConstraintForce(JointId jointId)
        {
            World world = World.GetWorld(jointId.World0);
            Joint joint = GetJointFullId(world, jointId);
            JointSim sim = GetJointSim(world, joint);

            switch (joint.Type)
            {
            case JointType.DistanceJoint:
                return DistanceJointFunc.GetDistanceJointForce(world, sim);

            case JointType.MotorJoint:
                return MotorJointFunc.GetMotorJointForce(world, sim);

            case JointType.MouseJoint:
                return MouseJointFunc.GetMouseJointForce(world, sim);

            case JointType.PrismaticJoint:
                return PrismaticJointFunc.GetPrismaticJointForce(world, sim);

            case JointType.RevoluteJoint:
                return RevoluteJointFunc.GetRevoluteJointForce(world, sim);

            case JointType.WeldJoint:
                return WeldJointFunc.GetWeldJointForce(world, sim);

            case JointType.WheelJoint:
                return WheelJointFunc.GetWheelJointForce(world, sim);

            default:
                Debug.Assert(false);
                return Vec2.Zero;
            }
        }
```

## DistanceJoint：距离约束与弹性

### 核心概念

`DistanceJoint` 是最直观的关节之一：它约束两个锚点之间的距离始终等于（或接近）一个给定的 `Length`。如果你把 `EnableSpring` 设为 `false`，它就是一根不可伸缩的刚性杆；如果开启弹簧，它就变成了一根橡皮筋，而且你还可以给它加上马达，让它主动伸缩。

### 定义参数

`DistanceJointDef` 的结构非常清晰。除了共享的 `BodyIdA`、`BodyIdB`、`LocalAnchorA`、`LocalAnchorB` 和 `CollideConnected`，它还包含这些专属字段：

```csharp:src/Joints/DistanceJointDef.cs [24:56]
        /// The rest length of this joint. Clamped to a stable minimum value.
        public float Length;

        /// Enable the distance constraint to behave like a spring. If false
        ///	then the distance joint will be rigid, overriding the limit and motor.
        public bool EnableSpring;

        /// The spring linear stiffness Hertz, cycles per second
        public float Hertz;

        /// The spring linear damping ratio, non-dimensional
        public float DampingRatio;

        /// Enable/disable the joint limit
        public bool EnableLimit;

        /// Minimum length. Clamped to a stable minimum value.
        public float MinLength;

        /// Maximum length. Must be greater than or equal to the minimum length.
        public float MaxLength;

        /// Enable/disable the joint motor
        public bool EnableMotor;

        /// The maximum motor force, usually in newtons
        public float MaxMotorForce;

        /// The desired motor speed, usually in meters per second
        public float MotorSpeed;
```

### 运行时 API

创建之后，你可以随时通过 `DistanceJointFunc` 修改各种开关和参数。比如调整目标长度、弹簧参数、距离限制范围，以及马达状态：

- `SetLength` / `GetLength`
- `EnableSpring` / `SetSpringHertz` / `SetSpringDampingRatio`
- `EnableLimit` / `SetLengthRange` / `GetMinLength` / `GetMaxLength`
- `EnableMotor` / `SetMotorSpeed` / `SetMaxMotorForce`

下面代码展示了如何在运行时打开限制，并同步唤醒两端刚体：

```csharp:src/Joints/DistanceJointFunc.cs [26:51]
        public static void EnableLimit(JointId jointId, bool enableLimit)
        {
            JointSim baseSim = Joint.GetJointSimCheckType(jointId, JointType.DistanceJoint);
            DistanceJoint joint = baseSim.Joint.DistanceJoint;
            joint.EnableLimit = enableLimit;
        }

        public static void SetLengthRange(JointId jointId, float minLength, float maxLength)
        {
            JointSim baseSim = Joint.GetJointSimCheckType(jointId, JointType.DistanceJoint);
            DistanceJoint joint = baseSim.Joint.DistanceJoint;

            minLength = Math.Clamp(minLength, Core.LinearSlop, Core.Huge);
            maxLength = Math.Clamp(maxLength, Core.LinearSlop, Core.Huge);
            joint.MinLength = Math.Min(minLength, maxLength);
            joint.MaxLength = Math.Max(minLength, maxLength);
            joint.Impulse = 0.0f;
            joint.LowerImpulse = 0.0f;
            joint.UpperImpulse = 0.0f;
        }
```

修改长度或限制范围时，内部会把 `Impulse`、`LowerImpulse`、`UpperImpulse` 清零。这是因为旧的约束冲量可能与新配置不再兼容，清零可以避免求解器在下一帧产生剧烈的"回弹"。

### 内部求解逻辑

在 `SolveDistanceJoint` 中，Box2DSharp 分两条路走：如果开启了弹簧，或者长度限制不相等，就按"软约束"处理；否则走刚性约束的分支。

弹簧分支会先计算当前距离与目标长度的偏差，然后结合 `Hertz` 和 `DampingRatio` 产生一个修正冲量。接着如果开启了限制，还会分别检查 `MinLength` 和 `MaxLength`，用类似于接触碰撞的"推测（speculative）"逻辑来防止穿透。

```csharp:src/Joints/DistanceJointFunc.cs [337:360]
            if (joint.EnableSpring && (joint.MinLength < joint.MaxLength || joint.EnableLimit == false))
            {
                // spring
                if (joint.Hertz > 0.0f)
                {
                    // Cdot = dot(u, v + cross(w, r))
                    Vec2 vr = B2Math.Add(B2Math.Sub(vB, vA), B2Math.Sub(B2Math.CrossSV(wB, rB), B2Math.CrossSV(wA, rA)));
                    float Cdot = B2Math.Dot(axis, vr);
                    float C = length - joint.Length;
                    float bias = joint.DistanceSoftness.BiasRate * C;

                    float m = joint.DistanceSoftness.MassScale * joint.AxialMass;
                    float impulse = -m * (Cdot + bias) - joint.DistanceSoftness.ImpulseScale * joint.Impulse;
                    joint.Impulse += impulse;
```

### 示例：悬链桥的小球链

在 `DistanceJointSample` 里，开发者用一排行列式的小球和距离关节模拟了一条链条。每个关节的 `BodyIdA` 接上一个球，`BodyIdB` 接下一个球，然后用 ` pivotA` 和 `pivotB` 计算局部锚点。UI 里还能实时拖动 `Length`、`Hertz`、`Damping` 滑杆，观察弹性变化。

```csharp:test/Testbed.Samples/Joints/DistanceJointSample.cs [71:98]
        DistanceJointDef jointDef = DistanceJointDef.DefaultDistanceJointDef();
        jointDef.Hertz = m_hertz;
        jointDef.DampingRatio = m_dampingRatio;
        jointDef.Length = m_length;
        jointDef.MinLength = m_minLength;
        jointDef.MaxLength = m_maxLength;
        jointDef.EnableSpring = m_enableSpring;
        jointDef.EnableLimit = m_enableLimit;

        BodyId prevBodyId = m_groundId;
        for (int i = 0; i < m_count; ++i)
        {
            BodyDef bodyDef = BodyDef.DefaultBodyDef();
            bodyDef.Type = BodyType.DynamicBody;
            bodyDef.Position = (m_length * (i + 1.0f), yOffset);
            m_bodyIds[i] = Body.CreateBody(WorldId, bodyDef);
            Shape.CreateCircleShape(m_bodyIds[i], shapeDef, circle);

            Vec2 pivotA = (m_length * i, yOffset);
            Vec2 pivotB = (m_length * (i + 1.0f), yOffset);
            jointDef.BodyIdA = prevBodyId;
            jointDef.BodyIdB = m_bodyIds[i];
            jointDef.LocalAnchorA = Body.GetLocalPoint(jointDef.BodyIdA, pivotA);
            jointDef.LocalAnchorB = Body.GetLocalPoint(jointDef.BodyIdB, pivotB);
            m_jointIds[i] = Joint.CreateDistanceJoint(WorldId, jointDef);

            prevBodyId = m_bodyIds[i];
        }
```

### 实用技巧

- `EnableSpring` 开启后，`Length` 才变成"静止长度"，不再是硬约束。
- 马达的 `MotorSpeed` 表示目标伸缩速度（米/秒），配合 `MaxMotorForce` 可以模拟液压杆的效果。
- 如果不想小球之间互相挤开，就把 `CollideConnected` 设为 `false`。

## RevoluteJoint：旋转关节与角度限制

### 核心概念

`RevoluteJoint` 又叫铰链关节，它把两个刚体的一个点"钉"在一起，并允许它们绕着这个点自由旋转。如果把你的手指想象成一根轴，两个刚体就像粘在指尖上的纸片——它们可以随意转圈，但中心点不能分开。

除了基础的铰链功能，Box2DSharp 的旋转关节还支持：
- **角度限制**：只允许在 `LowerAngle` 和 `UpperAngle` 之间摆动
- **马达**：以恒定角速度主动旋转
- **弹簧**：让关节角度像弹簧一样回归 `ReferenceAngle`

### 定义参数

`RevoluteJointDef` 比 `DistanceJointDef` 多出了一个很重要的字段：`ReferenceAngle`。它表示在初始状态下，`BodyB` 的角度减去 `BodyA` 的角度，用来定义角度限制的"零点"。

```csharp:src/Joints/RevoluteJointDef.cs [29:65]
        /// The bodyB angle minus bodyA angle in the reference state (radians).
        /// This defines the zero angle for the joint limit.
        public float ReferenceAngle;

        /// Enable a rotational spring on the revolute hinge axis
        public bool EnableSpring;

        /// The spring stiffness Hertz, cycles per second
        public float Hertz;

        /// The spring damping ratio, non-dimensional
        public float DampingRatio;

        /// A flag to enable joint limits
        public bool EnableLimit;

        /// The lower angle for the joint limit in radians
        public float LowerAngle;

        /// The upper angle for the joint limit in radians
        public float UpperAngle;

        /// A flag to enable the joint motor
        public bool EnableMotor;

        /// The maximum motor torque, typically in newton-meters
        public float MaxMotorTorque;

        /// The desired motor speed in radians per second
        public float MotorSpeed;
```

另外还有个不常用的 `DrawSize`，它是调试绘制的圆圈半径，默认是 `0.25f`。

### 运行时 API

`RevoluteJointFunc` 提供了完整的控制接口。你可以开关弹簧、调整 Hertz 和阻尼比、设置角度限制、开关马达、读取当前角度和马达扭矩。

```csharp:src/Joints/RevoluteJointFunc.cs [48:68]
        public static float GetAngle(JointId jointId)
        {
            World world = World.GetWorld(jointId.World0);
            JointSim jointSim = Joint.GetJointSimCheckType(jointId, JointType.RevoluteJoint);
            Transform transformA = Body.GetBodyTransform(world, jointSim.BodyIdA);
            Transform transformB = Body.GetBodyTransform(world, jointSim.BodyIdB);

            float angle = B2Math.RelativeAngle(transformB.Q, transformA.Q) - jointSim.Joint.RevoluteJoint.ReferenceAngle;
            angle = B2Math.UnwindAngle(angle);
            return angle;
        }

        public static void EnableLimit(JointId jointId, bool enableLimit)
        {
            JointSim joint = Joint.GetJointSimCheckType(jointId, JointType.RevoluteJoint);
            if (enableLimit != joint.Joint.RevoluteJoint.EnableLimit)
            {
                joint.Joint.RevoluteJoint.EnableLimit = enableLimit;
                joint.Joint.RevoluteJoint.LowerImpulse = 0.0f;
                joint.Joint.RevoluteJoint.UpperImpulse = 0.0f;
            }
        }
```

`GetAngle` 会先用相对旋转减去 `ReferenceAngle`，然后用 `UnwindAngle` 把它规范到 `(-π, π]` 区间。这样你拿到的角度就是以"零点"为基准的。

### 内部求解逻辑

在 `SolveRevoluteJoint` 中，求解器按顺序处理弹簧、马达、角度限制和点对点约束。

如果开启了弹簧，偏差 `C` 就是当前相对角度加上 `DeltaAngle`，然后乘以 `SpringSoftness.BiasRate` 得到修正项。马达则用一个简单的 PD 逻辑来追踪 `MotorSpeed`，并把输出冲量限制在 `context.H * MaxMotorTorque` 范围内。

```csharp:src/Joints/RevoluteJointFunc.cs [315:343]
            // Solve spring.
            if (joint.EnableSpring && fixedRotation == false)
            {
                float C = B2Math.RelativeAngle(stateB.DeltaRotation, stateA.DeltaRotation) + joint.DeltaAngle;
                float bias = joint.SpringSoftness.BiasRate * C;
                float massScale = joint.SpringSoftness.MassScale;
                float impulseScale = joint.SpringSoftness.ImpulseScale;

                float Cdot = wB - wA;
                float impulse = -massScale * joint.AxialMass * (Cdot + bias) - impulseScale * joint.SpringImpulse;
                joint.SpringImpulse += impulse;

                wA -= iA * impulse;
                wB += iB * impulse;
            }

            // Solve motor constraint.
            if (joint.EnableMotor && fixedRotation == false)
            {
                float Cdot = wB - wA - joint.MotorSpeed;
                float impulse = -joint.AxialMass * Cdot;
                float oldImpulse = joint.MotorImpulse;
                float maxImpulse = context.H * joint.MaxMotorTorque;
                joint.MotorImpulse = Math.Clamp(joint.MotorImpulse + impulse, -maxImpulse, maxImpulse);
                impulse = joint.MotorImpulse - oldImpulse;
```

### 示例：旋转门与摆锤

`RevoluteJointSample` 里搭建了两个典型场景：
- 一个长条状的胶囊体被固定在地面，像一扇旋转门
- 一个长方块被钉在另一侧，用马达把它推回水平位置

在第一个关节的初始化代码中，`ReferenceAngle` 被设成了 `0.5f * B2Math.Pi`（90度），意味着零点已经偏离了世界坐标系的水平方向。`LowerAngle` 和 `UpperAngle` 则把这扇门的摆动范围限制在 `-90°` 到 `+135°` 之间。

```csharp:test/Testbed.Samples/Joints/RevoluteJointSample.cs [50:68]
            Vec2 pivot = (-10.0f, 20.5f);
            RevoluteJointDef jointDef = RevoluteJointDef.DefaultRevoluteJointDef();
            jointDef.BodyIdA = groundId;
            jointDef.BodyIdB = bodyId;
            jointDef.LocalAnchorA = Body.GetLocalPoint(jointDef.BodyIdA, pivot);
            jointDef.LocalAnchorB = Body.GetLocalPoint(jointDef.BodyIdB, pivot);
            jointDef.EnableSpring = m_enableSpring;
            jointDef.Hertz = m_hertz;
            jointDef.DampingRatio = m_dampingRatio;
            jointDef.MotorSpeed = m_motorSpeed;
            jointDef.MaxMotorTorque = m_motorTorque;
            jointDef.EnableMotor = m_enableMotor;
            jointDef.ReferenceAngle = 0.5f * B2Math.Pi;
            jointDef.LowerAngle = -0.5f * B2Math.Pi;
            jointDef.UpperAngle = 0.75f * B2Math.Pi;
            jointDef.EnableLimit = m_enableLimit;

            m_jointId1 = Joint.CreateRevoluteJoint(WorldId, jointDef);
```

### 实用技巧

- `ReferenceAngle` 很关键。如果你发现刚体一创建就疯狂转动，多半是它的初始相对角度和 `ReferenceAngle` 没对齐。
- 开启 `EnableMotor` 并给 `MotorSpeed` 一个非零值，可以让关节自己转，像风扇叶片或车轮转向机构。
- `GetMotorTorque` 返回的是实际扭矩值，你可以用它来判断马达是不是在吃力，从而决定要不要"烧掉"电机。
- 如果你只需要摆锤效果，可以只开角度限制，不开弹簧和马达，这样求解成本最低。

## PrismaticJoint：棱柱关节

棱柱关节也叫滑动关节，它把两个刚体的相对运动限制在一条直线上。你可以把它想象成抽屉的导轨——物体只能沿着指定的轴平移，不能旋转，也不能在垂直方向移动。


`PrismaticJointDef` 的参数很丰富。除了常规的 `LocalAnchorA`、`LocalAnchorB` 和 `LocalAxisA` 定义滑动轴外，它还支持弹簧、平移限制和马达。`ReferenceAngle` 记录初始角度差，`EnableLimit` 配合 `LowerTranslation` / `UpperTranslation` 设置平移范围，`EnableMotor` 配合 `MotorSpeed` 和 `MaxMotorForce` 驱动滑块。


创建时只需要填充定义并调用 `CreatePrismaticJoint`。参考 `PrismaticJointSample` 的做法，这里把地面和动态的箱子连接起来，让箱子沿 45 度角滑动：


运行时你可以通过 `PrismaticJointFunc` 开关弹簧、调整限制、或者改变马达速度和最大作用力。`GetMotorForce` 能实时返回马达当前输出的力，非常适合做力反馈或调试显示。

## WheelJoint：车轮关节

车轮关节是专门为车辆悬挂设计的。它允许刚体沿一条轴平移（悬挂的上下运动），同时又能绕锚点自由旋转（轮胎滚动）。内置的线性弹簧正好模拟减震器的效果。


注意 `WheelJointDef` 的默认值已经开启了弹簧，并且把 `LocalAxisA` 设成了 `Vec2.UnitY`，`Hertz = 1.0f`，`DampingRatio = 0.7f`。这意味着如果你什么都不调直接创建，它已经是一个带减震的悬挂了。`EnableMotor` 控制的是旋转马达，用 `MaxMotorTorque` 和 `MotorSpeed` 来驱动轮子转动。


`WheelJointSample` 里把胶囊体当作车轮挂在地面上，轴指向 `(1,1)` 归一化后的方向。马达开启后车轮会自己转，弹簧参数则控制悬挂的软硬：


运行时可以用 `WheelJointFunc.GetMotorTorque(m_jointId)` 读取当前马达扭矩。如果你想做一辆赛车，调大 `MaxMotorTorque` 就是更强的动力，调小 `DampingRatio` 就是更软的悬挂。

## WeldJoint：焊接关节

焊接关节想把两个刚体"焊"在一起，约束它们的相对位置和角度。理想情况下它应该完全刚性，但 Box2D 的迭代求解器对大量刚性约束会比较吃力，所以 WeldJoint 提供了线性弹簧和角弹簧来做软化。


`LinearHertz` / `LinearDampingRatio` 控制位置上的软弹簧，`AngularHertz` / `AngularDampingRatio` 控制角度上的软弹簧。如果把这些频率设成 0，弹簧会变成最大刚度，也就更接近真正的刚性焊接。


实际项目中， weld 关节很适合做碎裂物体的临时粘合，或者把武器挂载到角色身上。如果你发现焊接处会"抖动"，把 Hertz 适当调高、DampingRatio 设到 1 附近通常能稳定下来。

## MotorJoint：马达关节

马达关节不和任何物理几何绑定，它纯粹用力和扭矩去驱动 BodyB 追踪一个相对 BodyA 的目标位姿。最常见的用法是让地面（BodyA）去控制一个动态刚体（BodyB），实现动画或 AI 驱动的运动。


关键参数是 `LinearOffset`、`AngularOffset`、`MaxForce`、`MaxTorque` 和 `CorrectionFactor`。`CorrectionFactor` 在 `[0,1]` 之间，决定位置误差的修正激进程度。设为 0 时，关节像干摩擦——只提供阻力而不主动回位。


看看示例里的用法：每一帧都更新 `LinearOffset` 和 `AngularOffset` 为正弦波，马达关节就会"拉着"刚体跟着目标走。遇到碰撞时，如果阻力超过 `MaxForce`，刚体就会被挡住。


## MouseJoint：鼠标关节

鼠标关节用来实现拖拽效果。它本质上是一个点跟踪约束，带弹簧阻尼，并且限制了最大作用力，所以拖拽时不会瞬间产生无限大的力把物体"甩飞"。


`Target` 是鼠标在世界空间的目标点，`Hertz` 和 `DampingRatio` 决定"橡皮筋"的松紧，`MaxForce` 则是橡皮筋的最大拉力。默认值已经设得比较温和：`Hertz = 4.0f`、`DampingRatio = 1.0f`。


运行时调用 `MouseJointFunc.SetTarget(jointId, newPosition)` 即可跟随鼠标。因为它的作用力和距离成正比，太远时拉力变大，直到被 `MaxForce` 封顶。这种软约束手感自然，比直接把物体位置设到鼠标下要物理得多。

## 弹簧参数：Hertz 与 DampingRatio 调优指南

Box2DSharp 的弹簧参数用"频率-阻尼"模型而不是传统的劲度系数。`Hertz` 表示弹簧的固有频率（Hz），`DampingRatio` 是无量纲的阻尼比。理解这两个数的物理意义，调参就不会靠猜了。

- **Hertz 越大，弹簧越硬**。当 Hertz 趋近于 0 时，弹簧最软；Box2D 内部用 0 表示"最大刚度"的刚性弹簧。
- **DampingRatio = 1** 是临界阻尼，系统回到平衡位置最快且不超调。
- **DampingRatio < 1** 是欠阻尼，会振荡，适合做蹦床、软悬挂。
- **DampingRatio > 1** 是过阻尼，回位慢但稳，适合需要沉稳感的机械臂。

如果把刚度-阻尼的关系画成曲线，横轴是 Hertz，纵轴是弹簧表现出的"硬度"：

1. **低 Hertz（0.5-2）+ 低 DampingRatio（0.1-0.5）**：软而弹，像橡皮筋或松软悬挂。
2. **中等 Hertz（3-8）+ DampingRatio ≈ 1**：紧致而稳定，适合大多数车辆悬挂和鼠标拖拽。
3. **高 Hertz（>10）或 Hertz = 0（最大刚度）+ DampingRatio ≥ 1**：硬而不振，适合焊接关节或高精度机械限位。

一般建议从 `Hertz = 2~4`、`DampingRatio = 0.7` 开始测试，然后根据视觉反馈微调。如果发现关节"发抖"，优先增大 DampingRatio；如果发现回弹太慢，优先增大 Hertz。

## ConstraintGraph 与图着色并行求解

Box2DSharp 的约束求解器为了利用多核性能，会把所有关节和接触约束分配到不同的"颜色"（Color）中。同一个颜色里的约束不能共享同一个动态刚体，这样它们就可以安全地并行求解。


`ConstraintGraph` 维护了一个 `GraphColor` 数组，默认有 12 种颜色。`AddContactToGraph` 和 `CreateJointInGraph` 在插入约束时，会遍历颜色桶，找到第一个两个 BodyId 都不冲突的颜色。如果实在找不到，就扔进最后的 `OverflowIndex` 颜色里，串行求解。


`AssignJointColor` 的逻辑和接触约束类似：


静态刚体有特殊处理——涉及静态地面的约束会被优先分散到非零颜色中，减少颜色 0 的拥挤。这种图着色策略让 Box2D 在多核平台上能把大量约束的求解开销分摊到多个线程，是性能优化的核心之一。当你开启调试绘制看到关节显示出不同颜色的小点时，那就是 ConstraintGraph 在工作。

## 设计思考：关节求解的精度与性能取舍

关节求解本质上是一场精度与性能的拉锯战。Box2DSharp 的迭代求解器每帧只给每个关节有限的修正机会，关节越多、链条越长，误差就越容易累积。你会看到 WeldJoint 焊接的长桥在中段微微下垂，DistanceJoint 串成的绳链在受力时被拉长——这些都不是 bug，而是求解器在告诉你要么加迭代、要么减关节。

好消息是，ConstraintGraph 的图着色机制已经帮我们把大部分关节约束并行化，多核红利基本吃到头了。再想提速，就不能靠"加机器"，而得在场景设计上动脑筋。实战经验是：**把超长的关节链拆成几段独立的短链**，中间用没有关节约束的自由刚体过渡；**去掉那些玩家根本注意不到的隐藏关节**；对精度要求极高的机械结构，可以单独给它一个更高迭代次数的世界实例，而不是全局拉满。

还有一点容易忽略：弹簧关节虽然舒服，但软的 DistanceJoint 或 WheelJoint 悬挂其实比纯刚性约束更难收敛。如果你把 Hertz 拉得太高、DampingRatio 又设得太低，求解器会在每一子步里反复震荡，精度没提升，CPU 先烧起来了。最稳妥的做法是先在低迭代次数下调出满意的弹簧手感，再决定是否为了边角稳定性而加码。

说到底，没有放之四海的最佳参数。关节系统的调优，永远是在"看起来对"和"跑得动"之间找一个你场景能接受的最优解。

## 小结

本章我们聊了关节系统的全貌。无论哪种关节，都离不开四个通用概念：**锚点**把连接点绑定在刚体局部空间里，**BodyIdA / BodyIdB** 确定被约束的双方，**CollideConnected** 决定它们之间是否还能碰撞，而**约束力**则是你实现绳子断裂、铰链崩坏效果的关键数据来源。

七种关节各有千秋：DistanceJoint 做绳链和弹簧，RevoluteJoint 做门轴和摆臂，PrismaticJoint 做滑轨活塞，WheelJoint 做车辆悬挂，WeldJoint 做刚性拼接，MotorJoint 做目标驱动，MouseJoint 做拖拽交互。选对关节类型，往往比堆参数更有效。

最后别忘了两个弹簧参数的魔法：Hertz 控制刚度硬软，DampingRatio 控制震荡衰减。调它们的时候记住一个原则——先调出舒服的手感，再考虑是否追加迭代求稳定。把这些概念融会贯通，你的物理场景就会既生动又高效。
