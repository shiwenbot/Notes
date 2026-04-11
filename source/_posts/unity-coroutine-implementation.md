---
title: "Unity协程背后的实现原理"
date: 2026-04-10
categories:
  - 代码库
tags:
  - Unity
  - C#
description: "IEnumerator接口、yield状态机编译原理、Unity协程调度机制、UniTask优化思路"
---

# Unity协程背后的实现原理

> 来源：[聊一聊Unity协程背后的实现原理 - 博客园](https://www.cnblogs.com/iwiniwin/p/14878498.html)

## 核心概念速览

- **IEnumerator接口**：Current + MoveNext + Reset，定义集合迭代方式
- **yield关键字**：C#编译器语法糖，自动生成状态机类
- **状态机原理**：局部变量提升为成员变量，goto实现状态跳转
- **Unity调度机制**：根据Current类型决定何时调用MoveNext

---

## IEnumerator的本质

IEnumerator是所有非泛型枚举器的基接口，定义了遍历集合的标准方式：

```csharp
public interface IEnumerator
{
    object Current { get; }  // 获取当前元素
    bool MoveNext();         // 推进到下一个位置，到末尾返回false
    void Reset();            // 重置到初始位置
}
```

`foreach`循环本质上是IEnumerator的语法糖，编译器会将其转换为：

```csharp
var enumerator = collection.GetEnumerator();
while (enumerator.MoveNext())
{
    var item = enumerator.Current;
    // 使用item
}
```

**类比**：IEnumerator就像一个"游标"，Current指向当前位置，MoveNext往前挪一步。

---

## yield的编译器魔法

`yield`是C#关键字，是快速定义迭代器的语法糖。编译器会将包含yield的方法自动编译成一个状态机类。

```csharp
IEnumerator Test()
{
    yield return 1;
    Debug.Log("Surprise");
    yield return 3;
}
```

编译器生成的状态机类大致如下：

```csharp
public class TestStateMachine
{
    int <>1__state;      // 状态值：0=初始，1=yield return后，-1=结束
    int <>2__current;    // 对应Current属性

    bool MoveNext()
    {
        switch (<>1__state)
        {
            case 0: goto label1;    // 第一次调用
            case 1: goto label2;    // 第二次调用
        }

    label1:
        <>1__state = 1;
        <>2__current = 1;
        return true;

    label2:
        Debug.Log("Surprise");  // 第二次MoveNext才执行
        <>1__state = -1;
        return false;
    }
}
```

**关键原理**：
- **局部变量提升为成员变量**：原方法中的局部变量全部变成状态机类的字段，保证跨MoveNext调用的值持久化
- **状态值控制跳转**：`<>1__state`记录"停在哪里"，每次MoveNext通过状态值跳到正确位置
- **每个协程一个实例**：是实例成员变量（不是static），多个协程互不干扰

**类比**：就像玩游戏存档，状态值就是"关卡号"，成员变量就是"存档数据"。

---

## Unity协程的调度机制

Unity协程的本质：**IEnumerator + 状态机 + 调度器**

Unity每帧在游戏循环中检查所有协程的IEnumerator，根据`Current`属性的类型决定何时调用`MoveNext`：

```
1. StartCoroutine(IEnumerator)
   ↓
2. Unity调用MoveNext()，获取Current
   ↓
3. 根据Current类型决定何时再次调用MoveNext：
   - null → 下一帧调用（等待一帧）
   - WaitForSeconds → 每帧检查时间到了没
   - AsyncOperation → 每帧检查isDone
   ↓
4. 条件满足 → 调用MoveNext() → 继续执行协程
```

**两种等待模式**：
- **被动等待**：`WaitForSeconds`、`null` — 只是数据对象，Unity自己每帧检查
- **异步通知**：`AsyncOperation`（如`UnityWebRequestAsyncOperation`） — 外部系统完成后将`isDone`设为true

---

## UniTask的优化思路

Unity协程的状态机是`class`，每次`StartCoroutine`都会产生GC分配。UniTask的核心优化是将状态机改为`struct`，在栈上分配，消除了GC压力。

---

## 待深入研究

- UniTask的完整实现机制
- ValueTask和IAsyncEnumerable的关系
- async/await的底层原理（和协程有什么区别？）
