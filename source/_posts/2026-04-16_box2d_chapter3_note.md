---
title: "【课堂笔记】Box2DSharp Chapter 3 - 刚体系统"
date: 2026-04-16
categories:
  - 代码库
tags:
  - Box2D
description: "BodyType 三种类型、BodyDef 核心参数、ApplyForce vs ApplyImpulse、睡眠机制性能优化"
---

# 【课堂笔记】Box2DSharp Chapter 3 - 刚体系统

**日期**：2026-04-16
**课程**：Box2DSharp物理引擎实战指南 Chapter 3 - 刚体系统
**时长**：约45分钟

---

## 1. BodyType：三种身份，三种命运

### 1.1 核心差异

Box2DSharp 只有三种刚体类型，没有第四种：

| 特性 | StaticBody | KinematicBody | DynamicBody |
|---|---|---|---|
| 质量 | 零 | 零 | 正质量 |
| 速度来源 | 无（手动改位置除外） | 用户代码设定 | 力和冲量驱动 |
| 受力响应 | 忽略 | 忽略 | 完全响应 |
| 适用场景 | 地面、墙壁 | 电梯、平台 | 箱子、角色 |

### 1.2 为什么 KinematicBody 能推 DynamicBody，但不被反推？

**核心原理**：InvMass = 0

- Static/Kinematic 的 InvMass = 0 → 质量无穷大 → 加速度 a = F × 0 = 0
- 无论多大的力，速度都不会改变

**物理推导**：
用牛顿第二定律 F = ma，如果用 InvMass（1/m）表示：
$$a = F \times \text{InvMass}$$

当 InvMass = 0 时，加速度 a = 0，速度纹丝不变。

---

## 2. BodyDef 核心参数调优

### 2.1 平台跳跃游戏主角的刚体配置

**实际案例**：主角应该设置的 BodyDef 参数

```csharp
BodyDef def = BodyDef.DefaultBodyDef();
def.BodyType = BodyType.DynamicBody;      // 必须是 Dynamic
def.FixedRotation = true;                 // 防止翻滚
def.GravityScale = 1.0f;                  // 受重力影响
def.EnableSleep = false;                  // 永不睡眠（玩家控制）
```

**为什么这样配置？**

- **DynamicBody**：需要受力影响，能跳跃、被碰撞推动
- **FixedRotation = true**：不能因为碰撞就翻跟头
- **GravityScale = 1.0**：需要重力，不能飘起来
- **EnableSleep = false**：必须永远保持活跃！如果睡着了，玩家按键会有延迟

---

### 2.2 其他常用参数

| 参数 | 用途 | 示例 |
|------|------|------|
| **Position** | 创建时就放在目标位置 | 避免先创建再移动的开销 |
| **GravityScale = 0** | 让刚体飘起来 | 零重力区域、飞行道具 |
| **IsBullet** | 高速物体开启 CCD | 子弹、快球（性能开销，勿滥用） |

---

## 3. BodySim vs BodyState：数据导向设计

### 3.1 为什么要拆分数据？

Box2DSharp 没有把所有数据塞进一个巨大的 Body 类里，而是拆成了两半：

- **BodySim**：存质量、力、阻尼、位置、质心这些"仿真数据"
- **BodyState**：存速度、delta 位移、状态标记，是一个**紧凑的 32 字节结构体**

### 3.2 设计原理

**物理引擎的两个阶段**：

1. **积分阶段**：根据力 → 计算加速度 → 更新速度 → 更新位置
   - 需要：质量、力、阻尼、当前位置（在 **BodySim** 里）

2. **约束求解阶段**：处理碰撞、关节约束 → 修正速度
   - 需要：速度、delta 位移、状态标记（在 **BodyState** 里）

**求解阶段是每帧最高频的循环**——它要反复读写速度、修正速度。

**数据导向优化**：
- 把求解阶段需要的高频数据，打包成一个**紧凑的 32 字节 BodyState**
- 把所有 BodyState 存放在**连续数组**里
- Solver 迭代时，只读这个连续数组，缓存命中率最大化

### 3.3 为什么是 32 字节？

**答案**：匹配 CPU 缓存行！

现代 CPU 的缓存行通常是 **64 字节**。32 字节的 BodyState 正好是缓存行的一半，意味着**一次内存加载可以拿到 2 个刚体的状态**，最大化缓存利用率。

---

## 4. ApplyForce vs ApplyImpulse：本质区别

### 4.1 源码对比

**ApplyForce**：
```csharp
public static void ApplyForce(BodyId bodyId, Vec2 force, Vec2 point, bool wake)
{
    // ...
    BodySim bodySim = GetBodySim(world, body);
    bodySim.Force = B2Math.Add(bodySim.Force, force);  // ← 累加到 Force
    bodySim.Torque += B2Math.Cross(B2Math.Sub(point, bodySim.Center), force);
}
```

**ApplyImpulse**：
```csharp
public static void ApplyLinearImpulse(BodyId bodyId, Vec2 impulse, Vec2 point, bool wake)
{
    // ...
    ref BodyState state = ref set.States[localIndex];
    BodySim bodySim = set.Sims[localIndex];
    state.LinearVelocity = B2Math.MulAdd(state.LinearVelocity, bodySim.InvMass, impulse);  // ← 直接改速度！
    state.AngularVelocity += bodySim.InvInertia * B2Math.Cross(B2Math.Sub(point, bodySim.Center), impulse);
}
```

### 4.2 本质区别

| API | 写到哪里 | 何时生效 | 适用场景 |
|-----|---------|---------|---------|
| **ApplyForce** | BodySim.Force | 下一步解算时 | 风、推力、磁力（持续） |
| **ApplyImpulse** | BodyState.LinearVelocity | 立即生效 | 爆炸、子弹、跳跃（瞬时） |

### 4.3 为什么跳跃要用 ApplyImpulse？

玩家按下跳跃键，希望角色**立刻弹起来**，而不是"慢慢加速往上飞"。

**力（Force）**：累加到 Force → 下一步解算时才用来计算加速度 → 再改变速度（延迟生效）

**冲量（Impulse）**：直接改 LinearVelocity → **立即生效**

---

## 5. 睡眠机制的性能优化原理

### 5.1 为什么能提升性能？

睡眠的刚体省掉了：
1. **积分阶段**：不用计算力 → 加速度 → 速度 → 位置
2. **碰撞检测**：不用做宽相位 + 窄相位检测
3. **约束求解**：不用处理碰撞约束

### 5.2 唤醒机制

**三种自动唤醒情况**：

1. **施加力/冲量时自动唤醒**（最常用）：
   ```csharp
   Body.ApplyForce(bodyId, force, point, wake: true);
   Body.ApplyImpulse(bodyId, impulse, point, wake: true);
   ```

2. **其他刚体碰撞时唤醒**：醒着的刚体撞到睡着的刚体，睡着的会被自动唤醒

3. **手动唤醒**：
   ```csharp
   Body.SetAwake(bodyId, awake: true);
   ```

### 5.3 睡眠配置

**Box2DSharp**：
```csharp
BodyDef def = BodyDef.DefaultBodyDef();
def.EnableSleep = true;   // 允许睡眠
def.IsAwake = false;      // 初始就是睡着的！
```

**Unity 2D 对比**：
- `SleepingMode.NeverSleep`：永不睡眠（类似 `EnableSleep = false`）
- `SleepingMode.StartAwake`：初始清醒，可以入睡（默认）
- `SleepingMode.StartAsleep`：初始睡眠，直到被碰撞唤醒

### 5.4 什么场景禁用睡眠？

**EnableSleep = false** 的场景：
- 玩家控制的角色（需要即时响应输入）
- 敌人 AI、巡逻的 NPC（需要随时改变行为）
- 由代码驱动的运动体（比如移动平台）

---

## 6. 中优先级知识点

### 6.1 BodyDef 详细字段

| 字段 | 用途 |
|------|------|
| **LinearDamping** / **AngularDamping** | 速度衰减（模拟空气阻力） |
| **AllowFastRotation** | 允许快速旋转（车轮之类） |
| **AutomaticMass** | 根据形状自动计算质量 |

### 6.2 速度直接控制 API

```csharp
// 直接设定速度（平台、传送门）
Body.SetLinearVelocity(bodyId, new Vec2(5, 0));
Body.SetAngularVelocity(bodyId, 1.5f);
```

设置非零速度会自动唤醒刚体。

### 6.3 ApplyTorque 与角冲量

| API | 效果 |
|-----|------|
| **ApplyTorque** | 持续力矩（累加到 BodySim.Torque） |
| **ApplyAngularImpulse** | 瞬时角冲量（直接改 AngularVelocity） |

### 6.4 刚体生命周期管理

| API | 作用 |
|-----|------|
| **CreateBody** | 放入正确的解算集（StaticSet/AwakeSet/睡眠岛） |
| **DestroyBody** | 级联销毁碰撞、形状、关节 |
| **Enable/Disable** | 临时脱离物理模拟（不移动也不碰撞） |

---

## 7. 面试要点

**Q: 为什么 Box2D 要把刚体分成 Static、Kinematic、Dynamic 三种？**

A: 核心是 **InvMass 的差异**：
- Static/Kinematic 的 InvMass = 0 → 质量无穷大 → 加速度 a = 0 → 不会被碰撞改变速度
- Dynamic 的 InvMass > 0 → 会受力影响

**Q: ApplyForce 和 ApplyImpulse 的本质区别是什么？**

A: **写的位置不一样**：
- ApplyForce 写到 BodySim.Force（延迟生效）
- ApplyImpulse 直接改 BodyState.LinearVelocity（立即生效）

**Q: 睡眠机制为什么能提升性能？**

A: 睡眠的刚体跳过了积分、碰撞检测、约束求解三大阶段。

---

## 8. 学生反馈记录

**Q: 什么样的刚体应该禁用睡眠？**

A: 频繁移动的物体：
- 玩家控制的角色（需要即时响应输入）
- 敌人 AI、巡逻的 NPC
- 由代码驱动的运动体（比如移动平台）

**Q: 睡眠的物体会被唤醒吗？**

A: 三种唤醒方式：
1. 施加力/冲量时自动唤醒（wake: true）
2. 其他刚体碰撞时自动唤醒
3. 手动唤醒 Body.SetAwake(bodyId, awake: true)

**Q: Unity 2D 有睡眠机制吗？**

A: 有！Rigidbody2D.SleepingMode 有三个选项：
- NeverSleep：永不睡眠
- StartAwake：初始清醒，可以入睡（默认）
- StartAsleep：初始睡眠，直到被碰撞唤醒

---

## 9. 教学反馈

**学生反馈**：中优先级的内容也要详细讲解，不能快速过。

**教师调整**：以后的教学规则：
- 🔴 **高优先级**：详细讲解 + 深入提问
- 🟡 **中优先级**：也要详细讲解！
- 🟢 **低优先级**：可以快速过

---

## 10. 下节课预告

**Chapter 4 - 碰撞检测系统**

- Shape 形状系统（圆形、多边形、胶囊）
- Fixture 夹具系统
- 宽相位与窄相位
- 碰撞回调与筛选
