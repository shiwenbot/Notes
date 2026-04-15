---
title: "【教材】MOBA Chapter 4 - 锁步帧同步（LSF）"
date: 2026-04-12
categories:
  - 代码库
tags:
  - ET MOBA
description: "锁步帧同步（LSF）：命令体系、Ticker系统、客户端预测、服务端权威、一致性检查与回滚、逻辑-渲染分离"
---

# 第4章 - 锁步帧同步（LSF）

**日期**：2026-04-12
**课程**：NKGMobaBasedOnET项目 - Chapter 4 锁步帧同步
**时长**：约60分钟

---

## 1. LSF_Component - 帧同步的大脑

### 1.1 核心数据结构

```csharp
public class LSF_Component : Entity
{
    // 录像回放 + 断线重连（8192帧约4.5小时）
    public Dictionary<uint, Queue<ALSF_Cmd>> WholeCmds = new Dictionary<uint, Queue<ALSF_Cmd>>(8192);
    
    // 待处理命令队列，SortedDictionary保证按帧号顺序执行
    public SortedDictionary<uint, Queue<ALSF_Cmd>> FrameCmdsToHandle = new SortedDictionary<uint, Queue<ALSF_Cmd>>();
    
    // 本帧即将发出的命令
    public Dictionary<uint, Queue<ALSF_Cmd>> FrameCmdsToSend = new Dictionary<uint, Queue<ALSF_Cmd>>(64);
    
    public uint CurrentFrame;           // 帧同步的时间锚点
    public uint BufferFrame = 1;        // 服务端缓冲窗口（约33ms）
}
```

**关键点**：
- `WholeCmds`容量8192帧，用于录像回放和断线重连
- `FrameCmdsToHandle`用`SortedDictionary`保证帧顺序（确定性要求）
- `BufferFrame = 1`帧，容纳网络抖动

### 1.2 客户端独有字段

```csharp
#if !SERVER
public class LSF_Component : Entity
{
    // 缓存从上次服务端确认帧到当前帧的所有本地输入
    public Dictionary<uint, Queue<ALSF_Cmd>> PlayerInputCmdsBuffer = new Dictionary<uint, Queue<ALSF_Cmd>>();
    
    public uint ServerCurrentFrame;     // 服务端当前帧号
    public uint CurrentArrivedFrame;    // 客户端当前帧号
    public const int AheadOfFrameMax = 10;  // 最多超前10帧（约330ms）
    public long HalfRTT;                // 半程往返时间
    public bool ShouldTickInternal;     // 是否允许Tick（断线检测用）
    public bool IsInChaseFrameState;    // 是否处于追帧状态
}
#endif
```

**断线检测机制**：

```csharp
if (self.CurrentFrame > self.ServerCurrentFrame)
{
    self.CurrentAheadOfFrame = (int)(self.CurrentFrame - self.ServerCurrentFrame);
    
    if (self.CurrentAheadOfFrame > LSF_Component.AheadOfFrameMax)
    {
        self.ShouldTickInternal = false;
        Log.Error("长时间未收到服务端回包，开始断线重连，停止模拟");
```

**关键点**：
- 客户端超前服务端超过10帧 → 停止Tick → 进入断线重连
- 10帧是个经验值，平衡"容忍网络抖动"和"快速检测断线"

### 1.3 为什么在Update里手动调FixedUpdate.Tick()

```csharp
public class LockStepStateFrameSyncComponentUpdateSystem : UpdateSystem<LSF_Component>
{
    public override void Update(LSF_Component self)
    {
        if (!self.StartSync) return;
        self.FixedUpdate.Tick();  // ← 手动调用，而非Unity的FixedUpdate
#if !SERVER
        self.ClientHandleExceptionNet().Coroutine();
        self.LSF_TickBattleView(TimeAndFrameConverter.MS_Float2Long(Time.deltaTime));
#endif
    }
}
```

**原因**：
- 需要精确控制Tick频率（追帧/降速）
- **追帧**：回滚后从第N帧快速模拟到第N+10帧，调高Tick频率
- **降速**：客户端超前太多，等待服务端追上来，调低Tick频率
- 如果用Unity的`FixedUpdate`，无法灵活控制

---

## 2. ALSF_Cmd 命令体系

### 2.1 命令基类设计

```csharp
[ProtoContract]
[ProtobufBaseTypeRegister]
public abstract class ALSF_Cmd: IReference
{
    [ProtoMember(1)] public uint Frame;                              // 命令所属的帧号
    [ProtoMember(2)] public uint LockStepStateFrameSyncDataType;     // 命令类型ID
    [ProtoMember(3)] public long UnitId;                             // 目标实体ID
    public bool PassingConsistencyCheck;                            // 一致性检查结果（桥梁！）
    
    public abstract ALSF_Cmd Init(long unitId);
    public virtual bool CheckConsistency(ALSF_Cmd alsfCmd) { return true; }  // 默认不检查
    public virtual void Clear() { /* 重置所有字段 */ }
}
```

**关键设计**：
- **抽象操作而非状态**：传"玩家按了什么"，不传"角色在哪"
- `PassingConsistencyCheck`：连接**命令**和**Ticker**的桥梁
- 基类默认返回`true`，只有需要检查的Cmd才重写

### 2.2 命令类型常量（分段编号）

```csharp
public class LSF_CmdType
{
    //----------通用模块，1~100
    public const uint Move = 1;
    public const uint CreateSpiling = 3;
    public const uint CommonAttack = 4;
    public const uint SyncFSMState = 5;
    public const uint SyncAttribute = 6;
    public const uint CreateCollider = 7;
    public const uint SyncBuff = 8;
    
    //----------行为树模块，101 - 10000
    public const uint ChangeBlackBoardValue = 101;
    
    //----------Slate模块，10001 - 20000
    public const uint ChangeMainKey = 10001;
    
    //----------其他模块，20000~30000
    public const uint PlayerSkillInput = 20001;
}
```

**分段编号的好处**：
- 一眼看出命令属于哪个子系统
- 避免未来新增命令时ID冲突

### 2.3 具体命令示例

**移动命令**（携带信息最丰富）：

```csharp
[ProtoContract]
public class LSF_MoveCmd : ALSF_Cmd
{
    public const uint CmdType = LSF_CmdType.Move;
    [ProtoMember(1)] public bool IsMoveStartCmd;   // 区分"新寻路"vs"位置快照"
    [ProtoMember(2)] public float PosX, PosY, PosZ;
    [ProtoMember(5)] public float RotA, RotB, RotC, RotW;
    [ProtoMember(9)] public float Speed;
    [ProtoMember(10)] public bool IsStopped;
    [ProtoMember(11)] public float TargetPosX, TargetPosY, TargetPosZ;
}
```

**技能输入命令**（轻量级）：

```csharp
[ProtoContract]
public class LSF_PlaySkillInputCmd : ALSF_Cmd
{
    public const uint CmdType = LSF_CmdType.PlayerSkillInput;
    [ProtoMember(1)] public string InputTag;      // 技能标签
    [ProtoMember(2)] public string InputKey;      // 技能按键
    [ProtoMember(3)] public float Angle;          // 瞄准角度
    [ProtoMember(4)] public float TargetPosX, TargetPosY, TargetPosZ;
    [ProtoMember(7)] public long TargetUnitId;    // 目标单位ID
}
```

**关键点**：
- `IsMoveStartCmd`：区分"发起操作"和"上报状态"，减少命令类型数量
- 技能命令不直接描述效果，只记录"玩家按了什么"，具体行为由行为树决策

---

## 3. 命令分发与Handler

### 3.1 分发器自动注册机制

```csharp
public class LSF_CmdDispatcherComponent : Entity
{
    public Dictionary<uint, List<ILockStepStateFrameSyncMessageHandler>> Handlers = 
        new Dictionary<uint, List<ILockStepStateFrameSyncMessageHandler>>();
}

// 初始化时扫描所有带[LSF_MessageHandler]特性的类
public static void Load(this LSF_CmdDispatcherComponent self)
{
    self.Handlers.Clear();
    HashSet<Type> types = Game.EventSystem.GetTypes(typeof(LSF_MessageHandlerAttribute));
    foreach (Type type in types)
    {
        var handler = Activator.CreateInstance(type) as ILockStepStateFrameSyncMessageHandler;
        var attr = Game.EventSystem.GetAttribute<LSF_MessageHandlerAttribute>(type);
        self.RegisterHandler(attr.LSF_CmdHandlerType, handler);
    }
}
```

**为什么用`List<Handler>`？**

一个命令可能触发多个系统响应：
- 移动命令 → MoveComponent更新位置 + AnimationComponent播放移动动画 + BuffComponent检查"移动时触发"的Buff

### 3.2 Handler实现示例

**移动命令Handler**：

```csharp
[LSF_MessageHandler(LSF_CmdHandlerType = LSF_MoveCmd.CmdType)]
public class LSF_MoveCmdHandler : ALockStepStateFrameSyncMessageHandler<LSF_MoveCmd>
{
    protected override async ETVoid Run(Unit unit, LSF_MoveCmd cmd)
    {
#if !SERVER
        // 客户端：直接设置位置（同步服务端权威位置）
        Vector3 pos = new Vector3(cmd.PosX, cmd.PosY, cmd.PosZ);
        Quaternion rotation = new Quaternion(cmd.RotA, cmd.RotB, cmd.RotC, cmd.RotW);
        unit.Position = pos;
        unit.Rotation = rotation;
#endif
        MoveComponent moveComponent = unit.GetComponent<MoveComponent>();
        if (cmd.IsMoveStartCmd)
        {
            Vector3 target = new Vector3(cmd.TargetPosX, cmd.TargetPosY, cmd.TargetPosZ);
            unit.NavigateTodoSomething(target, 0, idleState).Coroutine();
        }
        if (cmd.IsStopped)
        {
            moveComponent.Stop(true);
        }
        await ETTask.CompletedTask;
    }
}
```

**关键点**：
- `#if !SERVER`的位置同步代码：
  - **远程玩家**：直接设置位置（无本地预测）
  - **本地玩家**：通过一致性检查后，用服务端权威位置覆盖（纠正偏差）

**技能输入Handler**：

```csharp
[LSF_MessageHandler(LSF_CmdHandlerType = LSF_PlaySkillInputCmd.CmdType)]
public class LSF_PlayerSkillInputHandler : ALockStepStateFrameSyncMessageHandler<LSF_PlaySkillInputCmd>
{
    protected override async ETVoid Run(Unit unit, LSF_PlaySkillInputCmd cmd)
    {
        // 写入行为树黑板，让行为树决策
        foreach (var skillTree in unit.GetComponent<NP_RuntimeTreeManager>().RuntimeTrees)
        {
            skillTree.Value.GetBlackboard().Set(cmd.InputTag, cmd.InputKey);
            skillTree.Value.GetBlackboard().Set("SkillTargetAngle", cmd.Angle);
        }
#if SERVER
        // 服务端：广播给其他客户端
        LSF_Component lsfComponent = unit.BelongToRoom.GetComponent<LSF_Component>();
        lsfComponent.AddCmdToSendQueue(cmd);
#endif
        await ETTask.CompletedTask;
    }
}
```

**"Handler写黑板 + 行为树决策"的解耦方式**：
- 技能逻辑可以纯数据驱动地配置
- 行为树独立于帧同步进行迭代

---

## 4. Ticker系统

### 4.1 四阶段执行时序

```
单个 LSF_Tick 的执行时序：

  TickStart          Tick              TickEnd          ViewTick
  ┌────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │ 预处理  │──▶│  核心逻辑    │──▶│  快照收集    │──▶│  渲染表现    │
  │ ·状态   │   │ ·移动计算    │   │ ·记录历史状态 │   │ (仅客户端)   │
  │  重置   │   │ ·技能处理    │   │ ·服务端判脏  │   │ ·插值渲染    │
  │ ·输入   │   │ ·碰撞检测    │   │ ·属性同步    │   │ ·动画驱动    │
  │  采集   │   │ ·Buff 更新   │   │             │   │             │
  └────────┘   └─────────────┘   └─────────────┘   └─────────────┘
```

**最关键的是TickEnd**：
- 保存历史快照（一致性检查的依据）
- 服务端判脏（优化带宽）

### 4.2 Ticker自动注册

```csharp
[LSF_Tickable(EntityType = typeof(MoveComponent))]
public class MoveComponentTicker : ALSF_TickHandler<MoveComponent>
{
    public override void OnLSF_Tick(MoveComponent entity, uint currentFrame, long deltaTime)
    {
        if (entity.ShouldMove)
        {
            entity.MoveForward(deltaTime, false);
        }
    }
    
    public override void OnLSF_TickEnd(MoveComponent entity, uint frame, long deltaTime)
    {
        // 打包当前状态为命令
        LSF_MoveCmd lsfMoveCmd = ReferencePool.Acquire<LSF_MoveCmd>().Init(unit.Id) as LSF_MoveCmd;
        lsfMoveCmd.Speed = entity.Speed;
        lsfMoveCmd.PosX = unit.Position.x;
        lsfMoveCmd.PosY = unit.Position.y;
        lsfMoveCmd.PosZ = unit.Position.z;
        lsfMoveCmd.Frame = lsfComponent.CurrentFrame;
        
        // 保存到历史快照
        entity.HistroyMoveStates[lsfComponent.CurrentFrame] = lsfMoveCmd;
        
#if SERVER
        // 服务端判脏：对比上一帧，只有变化了才广播
        if (!this.OnLSF_CheckConsistency(entity, lsfComponent.CurrentFrame - 1, lsfMoveCmd))
        {
            lsfComponent.AddCmdToSendQueue(lsfMoveCmd);  // 广播
        }
        else
        {
            lsfComponent.AddCmdsToWholeCmdsBuffer(ref lsfMoveCmd);  // 只记录不广播
        }
#endif
    }
    
    public override void OnLSF_ViewTick(MoveComponent entity, long deltaTime)
    {
        Unit unit = entity.GetParent<Unit>();
        unit.ViewPosition = Vector3.Lerp(unit.ViewPosition, unit.Position, 0.5f);
        unit.ViewRotation = Quaternion.Slerp(unit.ViewRotation, unit.Rotation, 0.5f);
    }
}
```

**服务端判脏优化**：

| 场景 | 无判脏 | 有判脏 |
|------|--------|--------|
| 10个英雄，5个站着不动 | 60个单位×30帧=1800条命令/秒 | 55个单位×30帧=1650条命令/秒 |
| 带宽消耗 | 100% | **约92%（省8%）** |

---

## 5. 逻辑-渲染分离

### 5.1 为什么分离？

```
假设：
- 逻辑位置Position每帧跳变（第100帧在Pos(10,0,10)，第101帧在Pos(11,0,11)）
- 渲染直接用Position

结果：角色会一卡一卡的，像PPT一样！
```

### 5.2 分离实现

```csharp
public override void OnLSF_ViewTick(MoveComponent entity, long deltaTime)
{
    Unit unit = entity.GetParent<Unit>();
    // ViewPosition从当前位置平滑过渡到逻辑位置
    unit.ViewPosition = Vector3.Lerp(unit.ViewPosition, unit.Position, 0.5f);
    unit.ViewRotation = Quaternion.Slerp(unit.ViewRotation, unit.Rotation, 0.5f);
}
```

**插值系数权衡**：

| 插值系数 | 效果 | 缺点 |
|----------|------|------|
| 0.1f | 非常平滑 | 延迟高（角色总在逻辑位置后面） |
| 0.5f | 平衡 | **最佳选择** |
| 0.9f | 延迟低 | 可能不平滑（偶尔会顿一下） |

---

## 6. 服务端 vs 客户端

### 6.1 核心差异

| 维度 | 服务端 | 客户端 |
|------|--------|--------|
| **命令来源** | 收集所有玩家的命令 | 仅发送自己的命令 |
| **帧执行** | 权威计算，确定真实状态 | 本地预测，等待服务端确认 |
| **历史状态** | 保存完整状态用于断线重连 | 保存完整状态用于回滚 |
| **一致性检查** | 不需要（自己就是权威） | 每帧对比本地预测 vs 服务端回包 |
| **回滚** | 不需要 | 不一致时回滚到服务端状态 |
| **渲染** | 不涉及 | ViewTick插值渲染 |
| **TickStart/TickEnd** | 遍历所有玩家的所有组件 | 只遍历本地玩家的组件 |
| **命令发送** | 广播给房间内所有客户端 | 发送到服务端 |

### 6.2 服务端权威的本质

```
如果每个客户端都是"权威"：
- 玩家A的客户端：我躲掉了技能！
- 玩家B的客户端：我打中你了！
- 服务端：？？？

结果：每个人看到的战斗都不一样！
```

**信任链**：
1. 服务端 = 权威，唯一的真实
2. 客户端 = 预测，大胆猜，但随时准备被纠正
3. 客户端大胆预测（让玩家觉得操作即时）
4. 收到服务端回包，做一致性检查
5. 不一致就回滚，用服务端权威状态覆盖
6. 继续预测...

---

## 7. 一致性检查与回滚

### 7.1 一致性检查的实现

```csharp
public override bool CheckConsistency(ALSF_Cmd alsfCmd)
{
    LSF_MoveCmd lsfMoveCmd = alsfCmd as LSF_MoveCmd;
    
    // 先检查是否都是"开始寻路"命令（布尔精确对比）
    if (lsfMoveCmd.IsMoveStartCmd == this.IsMoveStartCmd)
    {
        // 对比目标位置（浮点容差0.001）
        if (Mathf.Abs(lsfMoveCmd.TargetPosX - this.TargetPosX) > 0.001f) return false;
        if (Mathf.Abs(lsfMoveCmd.TargetPosZ - this.TargetPosZ) > 0.001f) return false;
    }
    else
    {
        return false;  // 一个是开始寻路，一个不是，直接不一致
    }
    
    // 对比当前位置、速度（浮点容差）
    if (Mathf.Abs(lsfMoveCmd.PosX - this.PosX) > 0.001f) return false;
    if (Mathf.Abs(lsfMoveCmd.PosZ - this.PosZ) > 0.001f) return false;
    if (Mathf.Abs(lsfMoveCmd.Speed - this.Speed) > 0.001f) return false;
    
    // 对比停止标记（布尔精确）
    if (lsfMoveCmd.IsStopped != this.IsStopped) return false;
    
    return true;  // 全部通过，一致！
}
```

**浮点数 vs 布尔值**：

| 数据类型 | 对比方式 | 原因 |
|----------|----------|------|
| **浮点数**（位置、速度） | 容差0.001 | 浮点运算有精度问题（CPU、编译器优化、运算顺序） |
| **布尔值**（IsStopped） | 精确对比 | 只有true/false，没有"中间值" |

### 7.2 回滚的执行

```csharp
if (shouldRollback)
{
    self.IsInChaseFrameState = true;
    self.CurrentFrame = targetFrame;  // 回滚到服务端确认帧
    
    // 恢复服务端状态
    foreach (var frameCmd in frameCmdsQueue)
    {
        if (frameCmd.UnitId == playerUnit.Id)
        {
            self.RollBack(self.CurrentFrame, frameCmd);  // 恢复状态
            if (!frameCmd.PassingConsistencyCheck)
            {
                LSF_CmdDispatcherComponent.Instance.Handle(self.GetParent<Room>(), frameCmd);
            }
            frameCmd.PassingConsistencyCheck = true;
        }
    }
    
    self.CurrentFrame++;
    int count = (int)self.CurrentArrivedFrame - 1 - (int)self.CurrentFrame;
    
    // 追帧：从确认帧的下一帧重新模拟到当前帧
    while (count-- >= 0)
    {
        self.LSF_TickManually();  // 手动执行一帧
        self.CurrentFrame++;
    }
    
    self.IsInChaseFrameState = false;
}
```

### 7.3 完整时序图

```
第 N 帧：

客户端：
1. 本地预测执行 Tick N
2. TickEnd 保存本地状态 State_N

服务端：
3. 权威计算 Tick N
4. TickEnd 保存权威状态 State_N
5. 广播 Tick N 权威状态给客户端

客户端收到回包：
6. OnLSF_CheckConsistency（对比本地 State_N vs 服务端 State_N）
   
   如果一致 → 继续正常执行
   
   如果不一致：
   7. OnLSF_RollBackTick（回滚到服务端权威状态）
   8. 从 N+1 帧重新模拟到当前帧
   9. 使用 PlayerInputCmdsBuffer 里缓存的输入重新预测
```

---

## 8. 设计权衡

### 8.1 为什么选帧同步而非状态同步？

| 游戏类型 | 适合方案 | 原因 |
|----------|----------|------|
| **MOBA**（LOL、Dota2） | **帧同步** | 单位多（小兵一波几十个）、操作频率低 |
| **RTS**（星际争霸） | **帧同步** | 同上，而且需要完美同步 |
| **FPS**（CS:GO） | **状态同步** | 玩家操作频率高（射击+移动）、实体相对少 |
| **MMO**（魔兽世界） | **状态同步** | 玩家分散在不同区域，不需要全局同步 |

**核心权衡**：
- **操作频率 vs 实体数量**
- MOBA/RTS：少操作、多单位 → 帧同步赚翻了
- FPS：高频操作、少单位 → 状态同步更合适

### 8.2 帧同步的代价

1. **服务端计算开销大**：必须跑完整的战斗逻辑
2. **客户端复杂度高**：每个组件都要实现`CheckConsistency`和`RollBackTick`
3. **追帧开销**：同一帧逻辑可能执行多次
4. **浮点一致性**：是个永恒的坑，只能用容差规避

### 8.3 务实的平衡

**只对本地玩家做预测和回滚，远程玩家直接用服务端命令驱动**：

- ✅ 大幅减少回滚的触发概率
- ✅ 减少追帧的计算量
- ❌ 代价：远程玩家的操作反馈多一个RTT的延迟

**在MOBA场景下是合理的**：
- 玩家对自身角色的操作手感要求极高
- 对其他角色的微秒级延迟并不敏感

---

## 面试要点总结

1. **帧同步 vs 状态同步的本质区别**
   - 帧同步：传操作指令（O(玩家数)）
   - 状态同步：传完整状态（O(实体数)）

2. **为什么MOBA选帧同步**
   - 单位多、操作频率低
   - 带宽省3倍以上

3. **客户端预测的作用**
   - 让玩家觉得操作即时（隐藏网络延迟）
   - 但随时准备被服务端纠正

4. **一致性检查的必要性**
   - 没有它，客户端永远不知道"我预测错了"
   - 会导致客户端和服务端看到完全不同的战斗

5. **浮点数为什么用容差对比**
   - 浮点运算有精度问题（CPU、编译器、运算顺序）
   - 客户端和服务端可能产生微小差异

6. **服务端判脏优化能省多少带宽**
   - 约8%（团战场景）
   - 角色站着不动就不发移动命令

7. **逻辑-渲染分离的原因**
   - 逻辑位置每帧跳变（30fps）
   - 直接渲染会一卡一卡的
   - 用Lerp平滑过渡

8. **为什么在Update里手动调FixedUpdate.Tick()**
   - 需要精确控制Tick频率
   - 追帧：调高频率
   - 降速：调低频率

---

## 课后思考

1. **如果不用`SortedDictionary`，而是用普通的`Dictionary`，会怎样？**
   - 提示：客户端可能先收到第10帧，后收到第9帧...

2. **`AheadOfFrameMax = 10`这个值是如何确定的？**
   - 提示：30帧/秒，10帧 = 330ms。考虑网络抖动、断线检测灵敏度...

3. **回滚追帧时，如果`PlayerInputCmdsBuffer`里的输入不完整，会怎样？**
   - 提示：玩家在第105帧按了Q键，但回滚到100帧重新模拟时，这个输入会丢失吗？

4. **为什么技能命令不直接描述技能效果，而是只记录"玩家按了什么"？**
   - 提示：行为树、数据驱动、技能配置...

---

**下节课预告**: Chapter 5 - 技能系统（行为树驱动）

**笔记创建**: 2026-04-12
**笔记作者**: 三月七老师 & 学生
