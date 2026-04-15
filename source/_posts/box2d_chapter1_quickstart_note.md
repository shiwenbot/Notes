---
title: "Box2DSharp Chapter 1 - 快速入门（从零开始的物理世界）"
date: 2026-04-15
categories:
  - 代码库
tags:
  - Box2D
  - 物理引擎
description: "Box2DSharp物理引擎第一章学习笔记：四步工作流、ID系统设计权衡、Def模式、刚体类型、Step函数、碰撞过滤、几何体类型"
---

# Box2DSharp Chapter 1 - 快速入门（从零开始的物理世界）

> 系统讲解了Box2DSharp物理引擎的核心工作流和设计哲学，通过苏格拉底式提问深入理解每个设计决策背后的权衡。

## 四步工作流

**CreateWorld → CreateBody → CreateShape → Step**

- **World**：时空容器，管理全局规则（重力、接触刚度、睡眠策略）和tick
- **Body**：运动学实体，定义质量、位置、速度、类型
- **Shape**：碰撞外形，定义几何形状、摩擦力、弹性、密度
- **Step**：推动时间前进，执行碰撞检测和物理求解

为什么分三层？组合自由度——一个Body可以挂多个Shape（不同密度、摩擦力），职责分离（World管规则，Body管运动，Shape管碰撞）。

## ID系统 vs 对象引用

Box2DSharp不返回Body对象引用，而是返回BodyId：

```csharp
public struct BodyId {
    int Index1;        // 索引（从1开始，0表示空）
    ushort Revision;   // 版本号
    ushort World0;     // 所属世界
}
```

| 设计 | 优点 | 缺点 |
|------|------|------|
| 对象引用 | 使用方便 | 野指针风险、阻止内部优化、API强耦合 |
| ID系统 | 安全（Revision检测）、允许内部优化、API稳定 | 需间接访问 |

Revision机制就像Git的commit hash——索引相同但版本号不同，就是不同的对象。销毁后索引复用但Revision递增，旧ID因版本号不匹配被拒绝。

## Def模式 vs C#默认参数

C#默认参数的值是**编译时嵌入调用方IL代码**的，DLL改默认值后exe不重新编译仍拿到旧值。Def模式把默认值移到运行时函数调用，解决版本陷阱。

| 特征 | Def模式 | Builder模式 |
|------|---------|------------|
| 适用 | 配置简单、有默认值 | 构建步骤复杂、需验证 |
| 例子 | 物理对象、材质配置 | 网络请求、技能效果、UI组件 |

## 三种刚体类型 + InvMass机制

源码验证：Static和Kinematic的InvMass都是0（质量无穷大），Dynamic的InvMass是1/Mass。

KinematicBody能推开DynamicBody但不能被推开——因为Kinematic的InvMass=0碰撞后速度不变，Dynamic的InvMass>0会被碰撞改变速度。

## Step函数设计

**为什么显式调用**：帧率分离、子弹时间（调整timeStep）、暂停物理（不调用Step）。

**subStepCount**的核心价值不是"调小时间步"而是"增加迭代次数"。1/60秒内求解器迭代4次 vs 1/240秒内迭代1次，前者约束更充分收敛（堆叠物体更稳定）。Unity类比：Solver Iteration Count。

## 密度在ShapeDef

密度是**材料的属性**，不是物体的属性。铁头木身人偶——头部Shape密度7.8，身体Shape密度0.6，最终Body质量=Σ(Shape密度×Shape面积)，重心自动偏向密度高的地方。

## 碰撞过滤

```csharp
public struct Filter {
    public ushort CategoryBits;  // 我是什么
    public ushort MaskBits;      // 我能和谁碰撞
    public int GroupIndex;       // 正数=总是碰撞，负数=永不碰撞
}
```

CategoryBits+MaskBits是位掩码机制（Unity类比：Layer Collision Matrix），16位可表达65536种组合。GroupIndex覆盖前两者，解决布娃娃关节互碰等特殊场景。

## 几何体类型

| 类型 | 碰撞检测 | 性能 | 用途 |
|------|---------|------|------|
| Circle | 距离≤半径 | O(1) | 子弹、轮子 |
| Capsule | 两圆+矩形 | O(1) | 角色碰撞体 |
| Polygon | SAT分离轴定理 | O(N) | 箱子、墙壁（最多8顶点） |
| Segment | 线段距离 | O(1) | 地面、斜坡 |

Capsule是角色碰撞体的"黄金形状"——圆可以"滑"过障碍物，不容易卡住。
