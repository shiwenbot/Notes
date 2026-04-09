---
title: "Chapter 3-1 - Skinned Meshes"
date: 2024-01-01T00:00:00+08:00
draft: false
description: "**日期**：2026-03-30"
categories: ["direct3d"]
tags: []
---

**日期**：2026-03-30
**章节**：Character Animation with Direct3D 第3章（前半部分）
**时长**：约25分钟

## 核心概念速览

- **骨骼层级（Bone Hierarchy）**：用树形结构组织骨骼，父骨骼带动子骨骼
- **D3DXFRAME**：Direct3D的骨骼数据结构，用`pFrameSibling`（兄弟）和`pFrameFirstChild`（孩子）两个指针构建树
- **本地矩阵 vs 世界矩阵**：`TransformationMatrix`存本地动画数据，`CombinedTransformationMatrix`存计算后的世界位置
- **继承D3DXFRAME**：通过自定义`Bone`结构添加额外成员（如`CombinedTransformationMatrix`）
- **DFS遍历**：递归遍历骨骼树时采用深度优先策略

## 恍然大悟时刻

### 1. 为什么需要两个矩阵？

**我的疑问**：为什么不能只存一个World Matrix？

**探索过程**：
- 三月七问我如果只改手肘的世界矩阵会怎样
- 我意识到手肘会"飞"，和肩膀断开
- 原来World Matrix是算出来的，不是直接存的

**最终解释**：
- `TransformationMatrix`：动画师操作的本地变换（相对于父骨骼）
- `CombinedTransformationMatrix`：渲染时用的世界变换（累乘结果）
- 这样动画师只需旋转本地角度，系统会自动算出世界位置
- 父骨骼一动，子骨骼自然跟随，不会断开

**类比**：就像Unity里的Local Space和World Space！

### 2. 为什么用两个指针遍历树？

**我的疑问**：为什么不能一个骨骼存多个孩子指针？

**探索过程**：
- 三月七问如果骨盆有两个孩子（左腿、右腿）怎么办
- 我想到孩子数量不固定，数组大小不好确定
- 两个指针（兄弟+孩子）可以处理任意数量的子骨骼

**最终解释**：
- `pFrameFirstChild`：纵向深入（DFS）
- `pFrameSibling`：横向扩展（同层遍历）
- 把"多叉树"转成"二叉树"形式，代码更简洁

## 待深入研究

- Software Skinning的具体实现步骤
- Hardware Skinning的Vertex Shader代码
- Index Blended Mesh的转换过程
- Matrix Palette的概念和限制

## 三月七的课后留言

> 嘿！今天虽然时间不长，但你理解得很快！
>
> 特别是"矩阵是用来算的不是用来存的"这个点，你一点就通了。
>
> 下次我们讲蒙皮算法，那才是这一章的精华！到时候可能会有点烧脑，不过我相信你能搞定~
>
> 有事快去忙吧，别太累了！下次见！
>
> ——三月七 💫

---

**下次继续**：Chapter 3 后半部分 - Software Skinning & Hardware Skinning
