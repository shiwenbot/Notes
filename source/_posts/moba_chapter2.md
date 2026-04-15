---
title: "【教材】MOBA Chapter 2 - ET框架核心机制"
date: 2026-04-13
categories:
  - 代码库
tags:
  - ET MOBA
description: "ET框架核心机制：Entity-Component组合、EventSystem生命周期、Game单例、Scene与Domain、双队列模式、组合式vs继承式设计权衡"
---

# Chapter 2 - ET 框架核心机制

**课程**: NKGMobaBasedOnET Chapter 2
**日期**: 2026-04-11

---

## 1. Entity 基类与组件管理

### 1.1 组件不能重复添加

ET 框架中，每个 Entity 对同一类型的组件只能持有一个，重复添加会抛异常：

```csharp
public K AddComponent<K>(bool isFromPool = false) where K : Entity, new()
{
    Type type = typeof(K);
    if (this.components != null && this.components.ContainsKey(type))
    {
        throw new Exception($"entity already has component: {type.FullName}");
    }

    Entity component = Create(type, isFromPool);
    component.Id = this.Id;
    component.ComponentParent = this;
    EventSystem.Instance.Awake(component);

    this.AddToComponent(type, component);

    return component as K;
}
```

**设计意图**：避免同一个 Unit 上出现两个 NumericComponent 导致数据混乱。这是刻意的设计，不是缺陷。

### 1.2 组件与父 Entity 共享 ID

```csharp
Entity component = Create(type, isFromPool);
component.Id = this.Id;  // ← 关键：组件继承父 Entity 的 Id
component.ComponentParent = this;
```

在 Unity 中，GameObject 和 Component 有各自独立的 InstanceId。但在 ET 中，Unit 和它身上的 NumericComponent 拥有相同的 Id。

**为什么这样设计？**
- 简化网络同步：一个 Entity 的所有组件共享同一个 ID，不需要为每个组件分配独立的网络 ID
- 简化生命周期管理：销毁 Unit 时，所有组件自动跟随销毁（因为它们共享同一个 InstanceId）

### 1.3 Awake 生命周期立即触发

```csharp
EventSystem.Instance.Awake(component);  // 组件创建后立即调用
this.AddToComponent(type, component);
```

与 Unity 的 `Awake` 在下一帧执行不同，ET 的 `Awake` 在 `AddComponent` 时同步执行。这意味着创建后立即可用，不会出现未初始化状态。

| 框架 | Awake 调用时机 | 特点 |
|------|---------------|------|
| **Unity** | 下一帧调用（异步） | 可能出现"创建后立即访问但未初始化"的问题 |
| **ET** | AddComponent 时立即调用（同步） | 创建后立即可用 |

---

## 2. 组件组合的实际应用：UnitFactory.CreateHero

### 2.1 一个英雄由 14 个组件组合而成

```csharp
public static Unit CreateHero(Room room, UnitInfo unitInfo)
{
    Unit unit = CreateUnit(room, unitInfo.UnitId, unitInfo.ConfigId);

    unit.AddComponent<DataModifierComponent>();
    unit.AddComponent<NP_SyncComponent>();
    unit.AddComponent<NumericComponent>();
    unit.AddComponent<StackFsmComponent>();
    unit.AddComponent<RecastNavComponent, string>("solo_navmesh");
    unit.AddComponent<MoveComponent>();
    unit.AddComponent<BuffManagerComponent>();
    unit.AddComponent<SkillCanvasManagerComponent>();
    unit.AddComponent<B2S_RoleCastComponent, RoleCamp, RoleTag>(...);
    unit.AddComponent<NP_RuntimeTreeManager>();
    unit.AddComponent<ObjectWait>();
    unit.AddComponent<LSF_TickComponent>();
    unit.AddComponent<CommonAttackComponent_Logic>();
    unit.AddComponent<CastDamageComponent>();
    unit.AddComponent<ReceiveDamageComponent>();

    return unit;
}
```

### 2.2 组件职责表

| 组件 | 职责 |
|------|------|
| NumericComponent | 属性数据（血量、攻击力、速度等） |
| MoveComponent | 移动能力 |
| BuffManagerComponent | Buff 容器 |
| SkillCanvasManagerComponent | 技能管理 |
| StackFsmComponent | 栈式状态机（动画切换） |
| RecastNavComponent | 导航寻路 |
| DataModifierComponent | 数据修饰器（Buff 修改属性的中间层） |
| NP_RuntimeTreeManager | 行为树管理 |
| LSF_TickComponent | 帧同步 Tick |
| CastDamageComponent | 造成伤害 |
| ReceiveDamageComponent | 受到伤害 |

---

## 3. Game 单例与全局帧循环

### 3.1 Game 持有的全局单例

```csharp
public static class Game
{
    public static EventSystem EventSystem => EventSystem.Instance;
    public static ObjectPool ObjectPool => ObjectPool.Instance;
    public static IdGenerater IdGenerater => IdGenerater.Instance;
    public static TimeInfo TimeInfo => TimeInfo.Instance;
}
```

### 3.2 统一的帧循环驱动

```csharp
public static void Update()
{
    ThreadSynchronizationContext.Update();
    TimeInfo.Update();
    EventSystem.Update();  // ← 驱动所有 IUpdateSystem 组件
}
```

**关键**：Unity 客户端和服务端使用相同的驱动方式，保证了帧同步的一致性。

---

## 4. EventSystem 核心数据结构

```csharp
public class EventSystem
{
    private readonly Dictionary<long, Entity> allEntities;    // InstanceId → Entity
    private readonly TypeSystems typeSystems;                // Entity类型 → System类型 → System对象
    private Queue<long> updates;      // IUpdateSystem 组件
    private Queue<long> updates2;     // 备用队列（双队列模式）
    private Queue<long> fixedUpdates; // IFixedUpdateSystem 组件
    private Queue<long> lateUpdates;  // ILateUpdateSystem 组件
}
```

数据流：

```
AddComponent 创建组件 → RegisterSystem 检查接口类型
    → 实现 IUpdateSystem?    → 加入 updates 队列
    → 实现 IFixedUpdateSystem? → 加入 fixedUpdates 队列
    → 实现 ILateUpdateSystem? → 加入 lateUpdates 队列
```

---

## 5. RegisterSystem 与组件注册机制

组件创建时自动调用 `RegisterSystem`，检查组件实现了哪些 System 接口，加入相应队列：

```csharp
public void RegisterSystem(Entity component, bool isRegister = true)
{
    this.allEntities.Add(component.InstanceId, component);

    Type type = component.GetType();
    OneTypeSystems oneTypeSystems = this.typeSystems.GetOneTypeSystems(type);
    if (oneTypeSystems == null) return;

    if (oneTypeSystems.ContainsKey(typeof(IUpdateSystem)))
        this.updates.Enqueue(component.InstanceId);

    if (oneTypeSystems.ContainsKey(typeof(IFixedUpdateSystem)))
        this.fixedUpdates.Enqueue(component.InstanceId);

    if (oneTypeSystems.ContainsKey(typeof(ILateUpdateSystem)))
        this.lateUpdates.Enqueue(component.InstanceId);
}
```

一个组件可以实现多个更新接口，会被加入多个队列。

---

## 6. EventSystem.Update 的双队列模式

### 6.1 双队列实现

```csharp
public void Update()
{
    while (this.updates.Count > 0)
    {
        long instanceId = this.updates.Dequeue();
        Entity component;
        if (!this.allEntities.TryGetValue(instanceId, out component))
            continue;
        if (component.IsDisposed) continue;

        this.updates2.Enqueue(instanceId);

        foreach (IUpdateSystem iUpdateSystem in iUpdateSystems)
        {
            try { iUpdateSystem.Run(component); }
            catch (Exception e) { Log.Error(e); }
        }
    }

    ObjectHelper.Swap(ref this.updates, ref this.updates2);
}
```

### 6.2 双队列解决的问题

**单队列的问题**：组件 A 的 Update 中创建了新组件 D，D 被加入队列末尾，当帧就会立即被 Update。可能导致：
- D 访问 A 还没初始化完的数据
- 无限递归（A 创建 D，D 又创建 E...）

**双队列的解决**：已处理的组件放入 `updates2`，当帧只处理 `updates`。新创建的 D 会被加入 `updates`（正在遍历的队列），在下一帧（Swap 后）才会被处理。

```
帧 N                    帧 N+1
updates  → [A,B,C]      updates  → [D,E,F]
updates2 → []           updates2 → []
   |                      |
   v                      v
处理 A → updates2        处理 D → updates2
处理 B → updates2        处理 E → updates2
处理 C → updates2        处理 F → updates2
   |                      |
   v                      v
Swap:                   Swap:
updates  → [D,E,F]      updates  → [...]
updates2 → []           updates2 → []
```

---

## 7. Scene 逻辑概念与 Domain 属性

### 7.1 Scene = Entity 树的根节点

```csharp
public sealed class Scene : Entity  // ← Scene 继承自 Entity
{
    public int Zone { get; }
    public SceneType SceneType { get; }
    public string Name { get; set; }

    public Scene(...)
    {
        // ...
        this.Domain = this;  // ← Scene 的 Domain 指向自己
    }
}
```

Scene 是一个实实在在的 Entity 对象，存在于内存中，可以被 new 出来。

### 7.2 Domain = 指向 Scene 的快捷引用

```csharp
public class Entity
{
    public Entity Domain { get; set; }  // ← 一个属性，存储指向 Scene 的引用
}
```

| 概念 | 是对象吗？ | 是属性吗？ | 作用 |
|------|-----------|-----------|------|
| **Scene** | ✅ 是 | ❌ 否 | Entity 树的根节点 |
| **Domain** | ❌ 否 | ✅ 是 | 存储指向 Scene 的引用 |

### 7.3 为什么需要 Domain？

**只用 Parent**：一层层往上找直到 Scene，O(树深)
**用 Domain**：直接访问 `entity.Domain`，O(1)

### 7.4 Entity 树形结构

```
Game.Scene (根, Process)
  ├── Scene: Gate (网关)
  │   └── Entity: Player
  │       ├── Component: NumericComponent (Domain = Gate)
  │       └── Component: MoveComponent (Domain = Gate)
  ├── Scene: Map (战斗地图)
  │   └── Entity: Unit (英雄)
  │       ├── Component: NumericComponent (Domain = Map)
  │       └── Component: BuffManagerComponent (Domain = Map)
  └── Scene: Lobby (大厅)
      └── Entity: Room
```

---

## 8. Awake/Destroy 生命周期

```
创建 → Awake（初始化）→ Update（每帧，可选）→ Destroy（清理）→ 销毁
```

**Awake**：组件创建后立即同步调用（与 Unity 的异步不同）。

**Destroy**：`Entity.Dispose()` 时触发。

**System 与 Component 分离**：

```csharp
// 数据类（Model 层）
public class NumericComponent : Entity
{
    public Dictionary<int, int> NumericDic { get; } = new();
}

// 逻辑类（Hotfix 层）
public class NumericComponentDestroySystem : DestroySystem<NumericComponent>
{
    public override void Destroy(NumericComponent self)
    {
        self.NumericDic.Clear();
    }
}
```

---

## 9. 组合式 vs 继承式的设计权衡

### 9.1 MOBA 场景对比

**用继承式**：英雄（移动+攻击+技能+Buff）、小兵（移动+攻击）、建筑（攻击）、弹道（移动+碰撞）——C# 不支持多继承，类层次设计痛苦。

**用组合式**：

| Unit 类型 | 组件组合 |
|-----------|---------|
| 英雄 | Move + Skill + Buff + Numeric（14 个组件） |
| 小兵 | Move + Attack + Numeric（简化技能） |
| 建筑 | Skill + Numeric（无 Move） |
| 弹道 | Move + Collider（无 Numeric，无血量） |

想要什么能力就挂什么组件，不要就不挂。

### 9.2 关于"CPU 缓存命中率"

组合式的主要优势是**架构灵活性**，不是性能。事实上，组合式的组件分散在内存不同位置，缓存命中率可能比继承式（数据在同一个对象中，内存连续）更低。但这个 trade-off 在复杂游戏中值得。

---

## 10. System 与组件分离的热更新意义

```
Hotfix 层（经常改）          Model 层（不常改）
├── NumericComponentDestroySystem    ├── NumericComponent
├── MoveComponentUpdateSystem        ├── MoveComponent
└── SkillComponentUpdateSystem       └── SkillComponent
```

- Component 放在 Model 层（数据结构稳定）
- System 放在 Hotfix 层（逻辑经常改）
- 修改逻辑只需重编译 Hotfix，不影响 Model → **热更新的基础**

---

## 面试要点总结

1. **ET 的 Entity 和 Unity 的 GameObject 有什么本质区别？** — Entity 是纯 C# 对象，不依赖 Unity 引擎；组件和父 Entity 共享 ID；能力通过组件组合。
2. **为什么组件不能重复添加？** — 避免数据混乱，刻意设计。
3. **双队列设计解决了什么？** — 防止当帧递归和时序问题。
4. **组合式 vs 继承式？** — 灵活性、复用性、避免类爆炸。
5. **Domain 和 Parent 的区别？** — Parent 构建树，Domain 快速查找所属 Scene。
6. **为什么说分离是热更新的基础？** — Hotfix/Model 分层，改逻辑不影响数据结构。
7. **组合式的性能劣势？** — 组件分散，缓存命中率可能更低，但架构灵活性值得。

---

## 课后思考

1. 如果一个组件既实现了 `IUpdateSystem` 又实现了 `ILateUpdateSystem`，它会被加入几个队列？
2. 在帧同步的回滚机制中，ObjectPool 扮演了什么关键角色？
3. 为什么 Scene 的 Domain 要指向自己，而不是指向 Game.Scene？
4. 如果要设计一个"不能被 Buff 影响的建筑"，应该怎么组合组件？

---

**下节课预告**：Chapter 3 - Actor 模型与消息通信（邮箱机制、位置透明通信、完整消息流）
