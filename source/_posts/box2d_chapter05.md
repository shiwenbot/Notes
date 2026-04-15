---
title: "【教材】第5章：碰撞检测——BroadPhase 与空间查询"
date: 2026-04-18
order: 005
categories:
  - 代码库
tags:
  - Box2D
description: "两阶段碰撞检测架构：BroadPhase快速筛查与精细碰撞检测"
---

## 概述

Box2DSharp 的碰撞检测采用的是经典的两阶段架构。你可以把它想象成安检流程：先过一道快速筛查门，再通过人工精密检查。

第一阶段是 BroadPhase（宽阶段）。它的任务非常单纯——用尽可能快的速度，把"完全不可能发生碰撞"的对象剔除掉。这里不关心形状到底是圆还是多边形，只看它们的 AABB 是否重叠。

第二阶段是 NarrowPhase（窄阶段）。只有通过了宽阶段筛选的候选配对，才会进入这里。窄阶段调用真正的几何算法，比如 GJK 距离算法，来计算两个形状是否确实相交，并生成碰撞流形（Manifold）。这样设计的好处显而易见：大部分物体在大部分时间里都不会毗邻，宽阶段替精确计算挡住了海量无效工作，是整个物理引擎效率的基石。

## AABB 轴对齐包围盒

AABB（Axis-Aligned Bounding Box）是宽阶段的灵魂。它就是一个与坐标轴对齐的矩形框，用最小点和最大点两个二维向量就能完全描述。

```csharp:src/Collision/AABB.cs [9:14]
    public struct AABB
    {
        public Vec2 LowerBound;

        public Vec2 UpperBound;
```

在碰撞检测中，AABB 承担着三重角色。第一，它是每个形状在 BroadPhase 里的代理身份证。DynamicTree 的叶子节点不直接存储形状，而是存储这个形状的 AABB。第二，它是快速剔除的判据。两个 AABB 的重叠检测只需要四次比较，效率极高：

```csharp:src/Collision/AABB.cs [229:237]
        public static bool Overlaps(in AABB a, in AABB b)
        {
            if (b.LowerBound.X - a.UpperBound.X > 0.0f || b.LowerBound.Y - a.UpperBound.Y > 0.0f || a.LowerBound.X - b.UpperBound.X > 0 || a.LowerBound.Y - b.UpperBound.Y > 0)
            {
                return false;
            }

            return true;
        }
```

第三，AABB 还支撑了所有空间查询。`AABB.RayCast` 实现了 slab 方法，能判断一条射线是否与包围盒相交：

```csharp:src/Collision/AABB.cs [68:90]
        public CastOutput RayCast(Vec2 p1, Vec2 p2)
        {
            // Radius not handled
            CastOutput output = new();

            var tmin = -float.MaxValue;
            var tmax = float.MaxValue;

            Vec2 p = p1;
            Vec2 d = B2Math.Sub(p2, p1);
            Vec2 absD = B2Math.Abs(d);

            Vec2 normal = Vec2.Zero;

            // x-coordinate
            if (absD.X < float.Epsilon)
            {
                // parallel
                if (p.X < LowerBound.X || UpperBound.X < p.X)
                {
                    return output;
                }
            }
```

为了让树结构更稳定，Box2DSharp 还会给 AABB 附加一点"脂肪边距"。形状一旦移动超出这个边距，就会触发重新插入，避免每帧都动树。`EnlargeAABB` 和 `Perimeter` 这些工具方法，也都为 DynamicTree 的 SAH 代价计算提供了基础。

## DynamicTree（BVH）

DynamicTree 是 Box2DSharp 里实现 BVH（Bounding Volume Hierarchy）的核心类。简单理解，它是一棵不断自我调整的二叉树，叶子节点对应物体，内部节点对应越来越大的包围盒。树根的 AABB 最终会覆盖整棵树的所有内容。

```csharp:src/Collision/DynamicTree.cs [12:48]
    public class DynamicTree
    {
        /// <summary>
        /// The tree nodes
        /// </summary>
        public TreeNode[] Nodes;

        /// <summary>
        /// The root index
        /// </summary>
        public int Root;

        /// <summary>
        /// The number of nodes
        /// </summary>
        public int NodeCount;

        /// <summary>
        /// The allocated node space
        /// </summary>
        public int NodeCapacity;

        /// <summary>
        /// Node free list
        /// </summary>
        public int FreeList;
```

树的每个节点由 `TreeNode` 结构体表示，它紧凑地放在了一块连续的内存里。叶子节点的 `Height` 为 0，内部节点存两个子节点索引，空闲节点则用 `Next` 串成链表，实现了一个轻量级的对象池：

```csharp:src/Collision/DynamicTree.cs [1828:1905]
        [StructLayout(LayoutKind.Explicit, Size = 48)]
        public struct TreeNode
        {
            /// <summary>
            /// The node bounding box
            /// </summary>
            [FieldOffset(0)]
            public AABB AABB; // 16

            /// <summary>
            /// Category bits for collision filtering
            /// </summary>
            [FieldOffset(16)]
            public ulong CategoryBits; // 8

            /// <summary>
            /// The node parent index
            /// </summary>
            [FieldOffset(24)]
            public int Parent; // union with 'next' field

            /// <summary>
            /// The node freelist next index
            /// </summary>
            [FieldOffset(24)]
            public int Next;
```

为什么要用 BVH？因为游戏里的大部分物体是静态或缓慢移动的。如果用均匀网格，要么格子太大导致查询遍历很多物体，要么格子太小导致内存爆掉。BVH 不受坐标范围限制，查询复杂度是 O(log n)，而且在局部运动的情况下只需要修改很小一部分树结构，非常优雅。

当你往 `BroadPhase` 里添加一个形状时，最终会调用 `DynamicTree.CreateProxy`。这个方法的实质是先在节点池里分配一个叶子节点，然后把这片叶子插进树里：

```csharp:src/Collision/DynamicTree.cs [784:805]
        public int CreateProxy(AABB aabb, ulong categoryBits, int userData)
        {
            Debug.Assert(-Core.Huge < aabb.LowerBound.X && aabb.LowerBound.X < Core.Huge);
            Debug.Assert(-Core.Huge < aabb.LowerBound.Y && aabb.LowerBound.Y < Core.Huge);
            Debug.Assert(-Core.Huge < aabb.UpperBound.X && aabb.UpperBound.X < Core.Huge);
            Debug.Assert(-Core.Huge < aabb.UpperBound.Y && aabb.UpperBound.Y < Core.Huge);

            int proxyId = AllocateNode();
            ref var node = ref Nodes[proxyId];

            node.AABB = aabb;
            node.UserData = userData;
            node.CategoryBits = categoryBits;
            node.Height = 0;

            bool shouldRotate = true;
            InsertLeaf(proxyId, shouldRotate);

            ProxyCount += 1;

            return proxyId;
        }
```

插入新叶子的核心流程在 `InsertLeaf` 里，分成三步。第一步是沿着树从上到下，找一个"最佳兄弟"。这个选择可不是随机的，而是用 SAH（Surface Area Heuristic）贪心计算：每往下一层，都估算把新叶子挂在这棵子树里的代价增量，选总表面积增加最少的那条路，以此保证树尽可能紧凑：

```csharp:src/Collision/DynamicTree.cs [640:714]
        public void InsertLeaf(int leaf, bool shouldRotate)
        {
            if (Root == Core.NullIndex)
            {
                Root = leaf;
                Nodes[Root].Parent = Core.NullIndex;
                return;
            }

            // Stage 1: find the best sibling for this node
            AABB leafAABB = Nodes[leaf].AABB;
            int sibling = FindBestSibling(leafAABB);

            // Stage 2: create a new parent for the leaf and sibling
            int oldParent = Nodes[sibling].Parent;
            int newParent = AllocateNode();

            // warning: node pointer can change after allocation
            Span<TreeNode> nodeSpan = Nodes;
            nodeSpan[newParent].Parent = oldParent;
            nodeSpan[newParent].UserData = -1;
            nodeSpan[newParent].AABB = B2Math.AABB_Union(leafAABB, nodeSpan[sibling].AABB);
            nodeSpan[newParent].CategoryBits = nodeSpan[leaf].CategoryBits | nodeSpan[sibling].CategoryBits;
            nodeSpan[newParent].Height = (short)(nodeSpan[sibling].Height + 1);
```

找到兄弟后，第二步是为叶子和兄弟创建一个新的父节点，把它们收编到同一个屋檐下。第三步是从新父节点往上回溯，一路上重新计算每个祖先的 AABB 和高度。如果发现树不平衡，就触发局部旋转（`RotateNodes`），把子树的位置微调一下，让整棵树的代价重新回到较优状态。正是这种"小修小补"的增量维护策略，让 DynamicTree 能胜任游戏中物体频繁创建、删除和移动的场景。

## SAH 启发式与树旋转优化

在 `FindBestSibling` 中，Box2DSharp 用贪心策略寻找新叶节点的最佳兄弟。核心思想是：从根节点一路向下，计算把叶节点 D 放到当前节点的成本。如果当前子树是内部节点，还要估算继续下潜的"最低成本下界"。


当成本 ties 时，代码回退到包围盒中心距离（`LengthSquared`），保证确定性。相比全局最优，这条单一路径的搜索复杂度是 O(log N)，非常适合动态场景。

插入后，`InsertLeaf` 会不断向上回溯父节点，并在每一层调用 `RotateNodes`。


`RotateNodes` 只处理当前节点 A 的两棵子树 B 和 C。根据 B、C 是否为叶节点，分三种情况计算旋转收益：如果交换兄弟节点能降低总包围盒面积，就执行左旋或右旋。


注意这一行 `bool shouldRotate = true;` 只在 `CreateProxy` 时启用旋转，而 `MoveProxy` 里设为 `false`——因为移动代理是高频操作，避免每次插入都旋转反而更稳。树的质量主要靠周期性的 `Rebuild` 来保证。

## BroadPhase 宽阶段：多树管理、移动缓冲与配对查询

Box2DSharp 的 `BroadPhase` 同时维护了三棵 `DynamicTree`，分别对应 `StaticBody`、`KinematicBody` 和 `DynamicBody`。


为什么要分开？静态物体几乎不移动，重建频率可以极低；而动态和运动刚体每帧都在更新。把它们隔离在三棵独立树中，既减少了动态树的节点数量，也让查询策略更精确。

代理的创建和移动都会被记录到 `MoveSet` / `MoveArray` 中：


`BufferMove` 把 proxyKey 写入哈希集合（`B2HashSet`），同时追加到数组，保证后续遍历的确定性顺序。静态物体默认不加入移动缓冲，除非显式强制。

配对查询的核心在 `FindPairsTask`。对于每一个移动的代理，先用它的 fatAABB 在三棵树中做重叠查询，回调是 `_pairQueryCallback`。


回调里做了大量过滤：自身排除、同体排除、碰撞过滤器（`ShouldShapesCollide`）、传感器互斥、关节覆盖、以及用户自定义过滤。最终通过所有检查的配对会以 `MovePair` 链表形式挂到 `MoveResult` 下。

`UpdateBroadPhasePairs` 把 `FindPairsTask` 放到 `Parallel.ForEach` 里并行执行，然后在主线程串行遍历结果、创建 `Contact`。


这种做法兼得并行效率和确定性：并行阶段只生成候选配对，串行阶段按 `MoveArray` 的固定顺序真正创建接触点，确保跨平台结果一致。

## 碰撞流形（Manifold）的物理意义

当 BroadPhase 说"你们可能撞了"，NarrowPhase 就要算出具体怎么撞。结果是 `Manifold`——也叫接触流形。


在 2D 物理引擎中，两个凸形状最多产生两个接触点，所以 `Manifold` 直接内嵌了 `Point1` 和 `Point2`，并用 `MemoryMarshal.CreateSpan` 提供 `Points` 访问。`Normal` 的方向特别重要：它始终从 shapeA 指向 bodyB。这个约定在求解器里计算分离和冲量时不能搞反。

每个接触点的细节在 `ManifoldPoint` 中：


这里有几个关键量：

- `Point`：世界坐标下的接触位置。注释提醒它在大坐标下会有精度损失，建议只用于调试绘制。
- `AnchorA` / `AnchorB`：相对于两个刚体原点的局部锚点。求解器内部其实用的是质心相对坐标。
- `Separation`：分离距离。正值表示还没碰上（speculative contact），负值表示穿透深度。调试时你会看到穿透点被标成不同颜色。
- `Normal`：单位法向量，决定冲量该往哪个方向推。
- `NormalImpulse` / `TangentImpulse`：法向和切向冲量，也就是求解器算出来的"推"和"摩擦"大小。
- `Persisted`：这个点在上一步是否存在。用于热启动（warm starting）和调试绘制里的颜色区分（新增绿色、保持蓝色）。

在 `World.Draw` 的调试绘制代码里，你可以看到这些数据被直接可视化：


`Separation > linearSlop` 的点是 speculative contact，用灰色；刚出现的新点是绿色； persistent 点是蓝色。法向量、冲量、摩擦力也都可以用线段画出来。

## 空间查询实战

Box2DSharp 在 `World` 类里提供了一套完整的空间查询 API：`CastRay`、`CastRayClosest`、`CastCircle`、`CastCapsule`、`CastPolygon`、`OverlapAABB`、`OverlapCircle`、`OverlapCapsule`、`OverlapPolygon`。它们的实现模式非常统一：先算一个查询用的 AABB，再用 `DynamicTree.Query` / `RayCast` / `ShapeCast` 遍历三棵树，最后在回调里做精确检测。

### 射线投射

`CastRay` 的入口很简单：原点、方向向量、过滤器和回调。


内部会维护一个 `WorldRayCastContext`，其中 `Fraction` 会随着每次命中不断缩小。这意味着回调返回的 fraction 越小，射线就被裁剪得越短，后续查询也就越快。

如果你只关心最近的命中，`CastRayClosest` 直接返回 `RayResult`：


`RayResult` 的结构如下：


### 形状投射

`CastCircle`、`CastCapsule`、`CastPolygon` 都是把形状转换成 `ShapeCastInput`，然后调用 `DynamicTree.ShapeCast`。


`ShapeCastInput` 用"点云 + 半径"统一描述任意凸形状：


### 重叠检测

`OverlapAABB` 会遍历全部三棵树，用 `WorldQueryContext` 包装用户回调：


`OverlapCircle` / `OverlapCapsule` / `OverlapPolygon` 则先用 AABB 粗筛，再用 `DistanceFunc.ShapeDistance` 做精确判定：


### 完整示例：从 RayCastWorld 样品看实战用法

在 `RayCastWorld` 样品中，同时演示了 Closest、Any、Multiple、Sorted 四种回调模式，以及射线、圆形、胶囊体、多边形四种投射类型：


这是 `Closest` 回调的实现：


返回 `-1.0f` 表示忽略这个形状；返回 `fraction` 表示裁剪射线并继续；返回 `0.0f` 表示立即终止。这给了你完全的控制权。

`OverlapWorld` 样品则展示了重叠查询的用法。在 `PostStep` 中，根据当前选择的查询形状调用不同的 `OverlapXxx`：


回调里把重叠的 shapeId 收集到 `DoomIds` 数组，退出循环后再统一销毁，避免在树遍历过程中修改树结构。

## GJK 距离算法简介

GJK（Gilbert-Johnson-Keerthi）是 Box2D 里计算两个凸形状之间最短距离的核心算法。它不需要显式处理每种形状组合，而是把所有形状都抽象成 `DistanceProxy`——一个带半径的点云。


圆形是一个点加半径；胶囊体是两个点加半径；多边形是 N 个点、半径通常为 0。

GJK 的核心是一个叫 Simplex 的结构：


算法迭代地在 Minkowski 差里寻找一个最接近原点的单纯形（1 个点、2 个点的线段、或 3 个点的三角形）。每次迭代都检查当前单纯形里离原点最近的区域，并沿着那个方向在 support 函数里取新的点加入单纯形。如果新点的投影不再有进展，就说明已经收敛到了最小距离。

在 Box2DSharp 碰撞检测的流程中，GJK 主要用在两处：一是 `ShapeDistance`，计算两个形状是否穿透以及穿透深度；二是 `ShapeCast`，用于形状投射时求首次碰撞时间。对于日常游戏开发，你不需要手动调用 GJK，但理解它的作用有助于你排查"为什么这两个形状还没碰上就被认为碰撞了"或者"射线/形状投射结果异常"之类的问题。

## 设计思考：为什么为不同 BodyType 维护独立的树？

Box2DSharp 的 `BroadPhase` 里有一个 `DynamicTree[] Trees`，按 `DynamicBody`、`KinematicBody`、`StaticBody` 各存一棵。这不是随手写的数组，而是一笔精打细算的性能账。

第一，三类刚体的"活跃度"天差地别。Dynamic 物体每时每刻都在动，Kinematic 偶尔变位，Static 几乎永远不动。如果把它们硬塞进同一棵动态树，Static 的叶子会被 Dynamic 反复带动重排，BVH 的质量会迅速劣化。分树之后，Static 树几乎不重构，Dynamic 树只管自己折腾。

第二，配对查询可以大幅剪枝。`FindPairsTask` 里的逻辑写得很清楚：Dynamic 刚体只需要去查 Kinematic 树和 Static 树，再查一遍自己的 Dynamic 树；而 Kinematic 和 Static 根本不需要互相查。如果三者在同一棵树里，这些规则就不好快速跳过了。

第三，空间查询（如 `CastRay`、`OverlapAABB`）虽然要遍历三棵树，但每棵树都更紧凑、更平衡，整体遍历次数反而更少。而且射线投射可以利用 `MaxFraction` 在三棵树之间顺序裁剪，不会因为混在一起而多走冤枉路。

一句话总结：用三棵小树的维护成本，换来了配对阶段的大幅剪枝和动态树更高的整体质量。这笔买卖非常划算。

## 小结

本章我们从两阶段碰撞检测的架构出发，理解了 BroadPhase 如何用 AABB 快速"筛掉"不可能相交的组合，把精算留给 NarrowPhase；也深入看了 DynamicTree 这棵 BVH 树——它靠 SAH 启发式选兄弟节点、靠旋转保持平衡，让插入、删除和查询都能在对数时间内完成；最后动手写了射线投射和重叠检测的代码，体会到 World 提供的 `CastRay`、`CastRayClosest`、`OverlapAABB` 等 API 是如何在底层三棵树上工作的。

下一章，我们将继续往前走，看看 NarrowPhase 怎么算精确相交、碰撞流形 `Manifold` 又是怎么诞生的。
