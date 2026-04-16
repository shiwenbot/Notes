---
title: "Unity协程背后的实现原理"
date: 2026-04-10
categories:
  - 代码库
tags:
  - Unity
  - C#
  - 游戏开发
  - 协程
description: "深入解析Unity协程的底层实现，包括IEnumerator接口、yield编译器魔法和状态机原理"
---

# 课程笔记：Unity协程背后的实现原理

**日期**：2026-04-10
**类型**：临时加课（基于博客文章）
**时长**：约25分钟

## 核心概念速览

- **IEnumerator接口**：Current + MoveNext + Reset，定义集合迭代方式
- **yield关键字**：C#编译器语法糖，自动生成状态机类
- **状态机原理**：局部变量提升为成员变量，goto实现状态跳转
- **Unity调度机制**：根据Current类型决定何时调用MoveNext

---

## 恍然大悟时刻

### 1. IEnumerator的本质

**我的疑问**：IEnumerator和协程有什么关系？为什么StartCoroutine接收IEnumerator参数？

**探索过程**：
- 三月七让我用while循环 + IEnumerator写出等价的foreach代码
- 我写出了：`GetEnumerator()` + `while(MoveNext())` + `Current`
- 理解了IEnumerator就是一个"迭代器接口"，定义了遍历集合的标准方式

**最终解释**：
```csharp
public interface IEnumerator
{
    object Current { get; }  // 获取当前元素
    bool MoveNext();         // 推进到下一个位置
    void Reset();            // 重置到初始位置
}
```

foreach循环的编译器转换：
```csharp
var enumerator = collection.GetEnumerator();
while (enumerator.MoveNext())
{
    var item = enumerator.Current;
    // 使用item
}
```

**类比/记忆点**：IEnumerator就像一个"游标"，Current指向当前位置，MoveNext往前挪一步。

### 2. yield的编译器魔法

**我的疑问**：yield return为什么能"停住"代码？下次还能从"停住"的地方继续执行？

**探索过程**：
- 我提到之前读过UniTask源码，见过状态值+switch case的处理方式
- 三月七纠正：Unity编译器用goto而不是switch case，但核心思想一样
- 三月七追问：局部变量的值怎么保存的？

**最终解释**：

编译器生成的状态机类：
```csharp
public class TestStateMachine
{
    int <>1__state;      // 状态值：0=初始，1=yield return后，-1=结束
    int <>2__current;    // 对应Current属性
    int count;           // 原方法的局部变量变成成员变量！

    bool MoveNext()
    {
        switch (<>1__state)
        {
            case 0: goto label1;
            case 1: goto label2;
        }

    label1:
        <>1__state = 1;
        <>2__current = 1;
        return true;

    label2:
        Debug.Log("Surprise");
        <>1__state = -1;
        return false;
    }
}
```

关键点：
- **局部变量提升为成员变量**（count不是栈变量，是成员变量）
- **状态值控制跳转**（state记住"停在哪里"）
- **每次MoveNext从上次停住的地方继续**

**类比/记忆点**：就像玩游戏存档，状态值就是"关卡号"，成员变量就是"存档数据"。

### 3. Unity协程的调度机制

**我的疑问**：yield return new WaitForSeconds(3)，Unity怎么知道要等3秒？

**探索过程**：
- 我一开始以为是"外部runner执行完通知Unity"
- 三月七纠正：WaitForSeconds只是数据对象，Unity自己查时间

**最终解释**：

Unity的调度流程：
```
1. StartCoroutine(IEnumerator)
   ↓
2. 调用MoveNext()，获取Current
   ↓
3. 根据Current类型决定何时再次调用MoveNext：
   - null → 下一帧调用
   - WaitForSeconds → 每帧检查时间到了没
   - AsyncOperation → 每帧检查isDone
   ↓
4. 条件满足 → 调用MoveNext() → 继续执行协程
```

**两种等待模式**：
- **被动等待**：WaitForSeconds、null（Unity自己检查）
- **异步通知**：AsyncOperation（外部系统标记isDone）

**类比/记忆点**：Unity就像一个调度员，手里拿着一堆协程的IEnumerator，根据它们的Current类型决定"什么时候再叫它干活"。

### 4. UniTask的优化思路

**我的回答**：用的struct，没gc呗

**三月七的评价**：直接答对了！🎉

**最终理解**：
- Unity协程：状态机是class，每次StartCoroutine产生GC分配
- UniTask：状态机是struct，在栈上分配，没有GC压力

---

## 三月七的课后留言

嘿！今天你学得超棒的！🎉

这节课虽然是临时加课，但你理解得非常透彻！从IEnumerator的本质，到yield的编译器魔法，再到Unity的调度机制，**整个链路你都打通了！**

最让我惊喜的是你对UniTask的理解——"用的struct，没gc呗"，一句话就抓住了核心！这说明你真的理解了Unity协程的实现原理，能立刻联想到优化方向。

这种**"从原理到实践"的思维方式**，就是成为高级工程师的关键！✨

下次我们可以聊聊async/await的底层原理，或者深入看看UniTask的源码。这种底层知识积累多了，你写代码的时候就能"知其然，也知其所以然"！

加油！你今天的状态超赞的！💪

---

*三月七于 2026-04-10*