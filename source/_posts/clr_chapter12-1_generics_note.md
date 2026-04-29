---
title: "【课堂笔记】CLR via C# Chapter 12-1 - Generics 泛型基础"
date: 2026-04-28 22:51:00
categories:
  - 书籍
tags:
  - CLR via C#
  - 三月七
description: "泛型核心概念、C++模板vs C#泛型、性能优势与装箱代价、开放类型vs封闭类型、静态字段独立性、泛型不变性、JIT代码膨胀策略"
---

# 课堂笔记：Chapter 12 - Generics (Part 1)

**日期**: 2026-04-28
**课程**: CLR via C# - Chapter 12-1 (Generics 泛型基础)
**时长**: 约90分钟

---

## 目录

1. [泛型的核心概念](#1-泛型的核心概念)
2. [泛型的性能优势](#2-泛型的性能优势面试核心)
3. [开放类型 vs 封闭类型](#3-开放类型-vs-封闭类型面试常考)
4. [泛型的静态字段](#4-泛型的静态字段面试陷阱)
5. [泛型类型继承（泛型的不变性）](#5-泛型类型继承泛型的不变性)
6. [代码膨胀（JIT编译策略）](#6-代码膨胀jit编译策略)
7. [面试要点总结](#7-面试要点总结)
8. [课后思考](#8-课后思考)

---

## 1. 泛型的核心概念

### 1.1 泛型解决的问题：算法复用，不暴露源码

**核心诉求**：一套算法，适用于多种数据类型。

```
排序算法、查找算法、链表操作……
逻辑一样，只是数据类型不同。
```

**C++ 模板 vs C# 泛型**：

| 特性 | C++ 模板 (Template) | C# 泛型 (Generics) |
|------|---------------------|---------------------|
| 机制 | 编译期文本替换 | IL 层面存在泛型类型 |
| 需要源码 | ✅ 需要 .h 头文件 | ❌ 只需编译后的 DLL |
| 实例化时机 | 编译期（每个类型生成一份代码） | 运行时 JIT 编译 |
| 类型安全 | 弱（文本替换，不做类型检查） | 强（IL 层验证） |
| 代码膨胀 | 严重（每种类型一份完整代码） | 部分缓解（引用类型共享） |

**关键区别**：

```cpp
// C++ 模板：编译器把 T 替换成 int，生成一份新代码
// 必须有源码！因为是文本替换
template<typename T>
class List { ... };

// 编译 List<int> → 生成完整的一份 List_int 类
// 编译 List<string> → 再生成完整的一份 List_string 类
```

```csharp
// C# 泛型：IL 中保留泛型参数 T
// 只需要 DLL，不需要源码！
public class List<T> { ... }

// IL 中：List`1<T>，T 是占位符
// 运行时 JIT 编译时才确定具体类型
```

### 1.2 日常使用回顾

大家每天都在用泛型，只是不深究底层：

```csharp
// 每天都在写的东西
List<int> numbers = new List<int>();
Dictionary<string, int> ages = new Dictionary<string, int>();
Task<string> task = Task.FromResult("hello");

// 知道表面好处：减少拆装箱
// 但为什么能减少？底层怎么实现的？——这就是今天要讲的
```

---

## 2. 泛型的性能优势（面试核心）

### 2.1 Boxing 的代价

**ArrayList.Add(object)**：

```csharp
ArrayList list = new ArrayList();
list.Add(5);  // int 5 → 装箱成 object
```

**装箱过程中发生了什么？**

```
栈上的 int（4字节）
    ↓ 装箱
堆上分配 20 字节对象：
┌──────────────────────┐
│  对象头 (Object Header) │  8 字节
│  类型指针 (Type Pointer) │  8 字节
│  值 (5)               │  4 字节
└──────────────────────┘
```

**代价清单**：
1. 在堆上分配 20 字节（4 字节值 → 20 字节对象，膨胀 5 倍）
2. 将 4 字节值复制到新分配的堆内存
3. 返回堆上的引用

### 2.2 List\<int\> 的零装箱

```csharp
List<int> list = new List<int>();
list.Add(5);  // 直接传 int，零装箱！
```

**内部实现**：

```csharp
// List<T> 的 Add 方法（简化）
public class List<T> {
    private T[] _items;
    
    public void Add(T item) {
        // T 在 JIT 编译时变成 int
        // _items 在 JIT 编译时变成 int[]
        // 直接赋值，没有装箱！
        _items[_size++] = item;
    }
}
```

### 2.3 性能测试数据

**Jeffrey Richter 的基准测试**：1 亿次 Add 操作

| 指标 | `List<int>` | `ArrayList` | 差异 |
|------|-------------|-------------|------|
| 执行时间 | 1.6 秒 | 10.9 秒 | **7 倍** |
| GC 次数 | 6 次 | 390 次 | **65 倍** |

**为什么差距这么大？**

- `ArrayList`：每次 Add 都装箱 → 1 亿次堆分配 → GC 疯狂回收
- `List<int>`：零装箱 → 内存连续、无 GC 压力

> 💡 **面试话术**：泛型的性能优势不仅仅是"减少装箱"，更关键的是大幅降低 GC 压力。在高频操作场景下，GC 次数从 390 次降到 6 次，这才是真正的杀手级优势。

### 2.4 代码对比

```csharp
// ❌ 不用泛型：ArrayList
ArrayList list = new ArrayList();
list.Add(5);           // boxing: int → object
list.Add("hello");     // boxing: string → object（引用类型也要转型）
int value = (int)list[0];  // unboxing + cast，可能抛异常

// ✅ 用泛型：List<int>
List<int> list = new List<int>();
list.Add(5);           // 零装箱
// list.Add("hello"); // ❌ 编译错误！类型安全
int value = list[0];   // 零拆箱，直接取值
```

---

## 3. 开放类型 vs 封闭类型（面试常考）

### 3.1 概念定义

**开放类型（Open Type）**：
- 还有泛型参数没有填充具体类型
- **不能创建实例**
- 类型参数作为"占位符"存在

**封闭类型（Closed Type）**：
- 所有泛型参数都填了具体类型
- **可以创建实例**
- CLR 会为每个封闭类型创建独立的 Type Object

```csharp
// 开放类型 —— 有类型参数未填充
typeof(List<>)           // 1个未填充参数
typeof(Dictionary<,>)    // 2个未填充参数

// 封闭类型 —— 所有参数都已填充
typeof(List<int>)        // T → int
typeof(List<string>)     // T → string
typeof(Dictionary<string, int>)  // TKey → string, TValue → int

// ⚠️ 注意
List<> openList = new List<>();  // ❌ 编译错误！不能实例化开放类型
List<int> closedList = new List<int>();  // ✅ 可以实例化封闭类型
```

### 3.2 为什么需要区分？

**开放类型的用途**：反射、代码生成、泛型约束检查。

```csharp
// 反射场景：获取泛型类型定义
Type openType = typeof(List<>);          // List`1[T]
Type closedType = typeof(List<int>);     // List`1[System.Int32]

// 从开放类型构造封闭类型
Type stringListType = openType.MakeGenericType(typeof(string));
// stringListType == typeof(List<string>)  → true
```

### 3.3 面试陷阱

```
Q: typeof(List<>) 和 typeof(List<int>) 是同一个类型吗？
A: 不是。List<> 是开放类型（类型定义），
   List<int> 是封闭类型（具体化后的类型）。
   它们的 IsGenericTypeDefinition 属性不同。
```

```csharp
typeof(List<>).IsGenericTypeDefinition     // true
typeof(List<int>).IsGenericTypeDefinition  // false
typeof(List<int>).IsGenericType            // true（封闭类型也是泛型类型）
```

---

## 4. 泛型的静态字段（面试陷阱）

### 4.1 核心规则：每个封闭类型有独立的 Type Object

```
CLR 运行时：
  List<int>    → 独立的 Type Object A → 独立的静态字段存储
  List<string> → 独立的 Type Object B → 独立的静态字段存储
  List<DateTime> → 独立的 Type Object C → 独立的静态字段存储
```

**比喻**：
> Type Object = 类的"家"
> 静态字段住在这个"家"里
> 不同的封闭类型 = 不同的"家"
> `List<int>.Count` 和 `List<string>.Count` 住在不同的"家"里，互不影响

### 4.2 代码示例

```csharp
public class TypeCounter<T> {
    // 每个封闭类型各自持有一份 InstanceCount
    public static int InstanceCount = 0;
    
    public TypeCounter() {
        InstanceCount++;
    }
}

// 使用
var counter1 = new TypeCounter<int>();    // TypeCounter<int>.InstanceCount = 1
var counter2 = new TypeCounter<int>();    // TypeCounter<int>.InstanceCount = 2
var counter3 = new TypeCounter<string>(); // TypeCounter<string>.InstanceCount = 1
var counter4 = new TypeCounter<int>();    // TypeCounter<int>.InstanceCount = 3

Console.WriteLine(TypeCounter<int>.InstanceCount);     // 3
Console.WriteLine(TypeCounter<string>.InstanceCount);  // 1
Console.WriteLine(TypeCounter<DateTime>.InstanceCount); // 0（从未创建过）
```

### 4.3 内存布局

```
┌─────────────────────────┐    ┌─────────────────────────┐
│   Type Object for        │    │   Type Object for        │
│   TypeCounter<int>       │    │   TypeCounter<string>    │
├─────────────────────────┤    ├─────────────────────────┤
│ InstanceCount = 3       │    │ InstanceCount = 1       │
│ (静态字段存储在这里)      │    │ (静态字段存储在这里)      │
└─────────────────────────┘    └─────────────────────────┘
         ↑                              ↑
    int 相关实例                    string 相关实例
    都指向这个"家"                  都指向这个"家"
```

### 4.4 面试陷阱

```
Q: 下面代码输出什么？

List<int> list1 = new List<int>();
List<int> list2 = new List<int>();
List<string> list3 = new List<string>();

如果 List<T> 有一个静态字段 Count：

A: List<int>.Count 和 List<string>.Count 是两个完全不同的字段！
   封闭类型不同 → Type Object 不同 → 静态字段存储位置不同。
```

---

## 5. 泛型类型继承（泛型的不变性）

### 5.1 核心规则：指定类型参数不影响继承层次

```
List<int>     继承自 Object  ✅
List<string>  继承自 Object  ✅

List<string> 继承自 List<int>  ❌ 它们之间没有继承关系！
List<int>    继承自 List<string> ❌ 也没有！
```

### 5.2 为什么 `List<string>` 不能转成 `List<object>`？

```csharp
List<string> strings = new List<string> { "hello", "world" };

// ✅ 可以转成 object（所有类都继承自 Object）
object obj = strings;

// ❌ 不能转成 List<object>
List<object> objects = strings;  // 编译错误！

// 如果允许……
// List<object> objects = strings;
// objects.Add(123);  // 💥 string 列表里混入了 int！
// string s = strings[2];  // 💥 运行时崩溃！
```

**这就是泛型的不变性（Invariance）**：
- `List<string>` ≠ `List<object>` 的子类
- 虽然在逻辑上 "string 列表" 是一种 "object 列表"
- 但编译器不允许，因为类型安全会被破坏

### 5.3 类型转换规则总结

```csharp
List<string> strings = new List<string>();

// ✅ 合法转换
object obj = strings;              // 任何类型 → object
IEnumerable<string> ie = strings;  // 子类 → 接口

// ❌ 非法转换
List<object> lo = strings;         // List<string> ≠ List<object>
IEnumerable<object> ieo = strings; // IEnumerable<string> ≠ IEnumerable<object>
                               // （默认情况下，接口也是不变的）
```

### 5.4 补充：协变和逆变（C# 4.0+）

```csharp
// 协变 (out) —— 只能作为返回值
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;  // ✅ string 是 object 的子类

// 逆变 (in) —— 只能作为参数
Action<object> actObj = o => { };
Action<string> actStr = actObj;  // ✅ object 参数能接受 string

// 但 List<T> 既不是协变也不是逆变（T 同时作为输入和输出）
List<string> list = new List<string>();
List<object> listObj = list;  // ❌ 仍然不行
```

| 变性 | 关键字 | T 的位置 | 示例 |
|------|--------|---------|------|
| 不变 (Invariant) | 无 | 输入+输出 | `List<T>`, `Dictionary<TKey,TValue>` |
| 协变 (Covariant) | `out T` | 只输出（返回值） | `IEnumerable<out T>` |
| 逆变 (Contravariant) | `in T` | 只输入（参数） | `Action<in T>`, `IComparer<in T>` |

---

## 6. 代码膨胀（JIT编译策略）

### 6.1 问题：泛型会不会导致代码膨胀？

如果 `List<int>`、`List<string>`、`List<DateTime>` 每种都编译一份完整代码……
理论上确实会膨胀。但 CLR 的 JIT 有优化策略。

### 6.2 JIT 的分类处理

**值类型（Value Types）—— 各自独立编译**

```
List<int>       → JIT 编译一份（元素 4 字节）
List<long>      → JIT 编译一份（元素 8 字节）
List<DateTime>  → JIT 编译一份（元素 8 字节）
List<Guid>      → JIT 编译一份（元素 16 字节）
List<decimal>   → JIT 编译一份（元素 16 字节）
```

> **为什么？** 值类型大小不同（int 4字节、DateTime 8字节、Guid 16字节），
> 对应的数组操作（索引计算、内存布局）也不同，必须分别编译。

**引用类型（Reference Types）—— 共享一份编译结果**

```
List<string>    ┐
List<object>    ├→ JIT 只编译一份（所有引用都是 8 字节指针）
List<FileStream>┘
List<MyClass>   ┘
```

> **为什么可以共享？** 所有引用类型在内存中都是 8 字节指针，
> 指针大小一样，索引计算一样，内存布局一样，代码逻辑完全相同。

### 6.3 图解 JIT 编译策略

```
                    JIT 编译器
                       │
        ┌──────────────┴──────────────┐
        │                             │
   值类型？                      引用类型？
   (大小不同)                   (指针大小相同)
        │                             │
   每种类型                      所有类型
   单独编译                      共享一份
        │                             │
   ┌────┴────┐                   ┌────┴────┐
   │ List<int>  │ ← 独立代码      │ List<string>   │──┐
   │ List<long> │ ← 独立代码      │ List<object>   │  │ 共享同一份
   │ List<Guid> │ ← 独立代码      │ List<MyClass>  │──┘ JIT 代码
   └──────────┘                   └───────────────┘
```

### 6.4 优缺点

| 方面 | 说明 |
|------|------|
| ✅ 优点 | 引用类型共享代码，大幅减少代码膨胀 |
| ✅ 优点 | 值类型各编译一份，保证最优性能（无额外间接寻址） |
| ❌ 代价 | 第一次使用某个泛型类型时，JIT 需要即时编译，增加启动时间 |
| ❌ 代价 | 值类型泛型在类型参数很多时仍有一定膨胀（但通常数量有限） |

### 6.5 代码验证

```csharp
// 值类型 —— 各自独立的类型
Console.WriteLine(typeof(List<int>) == typeof(List<long>));
// False！不同的封闭类型，不同的 Type Object

// 引用类型 —— 也各自独立（Type Object 层面）
Console.WriteLine(typeof(List<string>) == typeof(List<object>));
// False！Type Object 仍然是独立的
// 但 JIT 编译出的机器代码是共享的！
// Type Object 独立 ≠ 机器代码独立
```

---

## 7. 面试要点总结

### 泛型基础
1. **C++ 模板和 C# 泛型的核心区别？**
   - C++ 模板是编译期文本替换，需要源码
   - C# 泛型在 IL 层面存在，只需 DLL

2. **泛型的两大好处？**
   - 性能：减少/消除装箱，降低 GC 压力
   - 安全：编译期类型检查

### 性能
3. **ArrayList vs List\<int\> 的性能差距？**
   - 1 亿次操作：List\<int\> 1.6秒/GC 6次 vs ArrayList 10.9秒/GC 390次
   - 性能 7 倍，GC 65 倍

4. **装箱时 int 在堆上占多少字节？**
   - 20 字节：对象头 8 + 类型指针 8 + 值 4

### 类型系统
5. **开放类型和封闭类型的区别？**
   - 开放类型：有未填充的类型参数，不能实例化（如 `typeof(List<>)`）
   - 封闭类型：所有参数已填充，可以实例化（如 `typeof(List<int>)`）

6. **泛型的静态字段行为？**
   - 每个封闭类型有独立的 Type Object
   - 静态字段存储在各自的 Type Object 中
   - `List<int>.Count` 和 `List<string>.Count` 是不同的字段

7. **为什么 `List<string>` 不能转成 `List<object>`？**
   - 泛型的不变性
   - 如果允许，可以通过 `List<object>.Add(123)` 破坏类型安全

### JIT 编译
8. **CLR 如何处理泛型的代码膨胀？**
   - 值类型：每种单独编译（大小不同）
   - 引用类型：共享一份（指针大小相同）

---

## 8. 课后思考

1. `Dictionary<int, string>` 和 `Dictionary<string, int>` 是否共享 JIT 代码？为什么？
2. 如果你要实现一个泛型缓存（如 `Cache<T>`），静态字段的行为会有什么影响？如何设计？
3. 阅读 `System.Collections.Generic.List<T>` 的源码，特别关注其内部数组的增长策略。
4. 思考：为什么 C# 团队在设计 `List<T>` 时没有让它支持协变？如果支持了会有什么问题？
5. 实验题：写一个 `TypeCounter<T>` 类，验证不同封闭类型的静态字段确实独立。

---

**下节课预告**: Chapter 12 - Generics (Part 2) — 泛型约束、协变逆变、泛型接口深入

**笔记创建**: 2026-04-29
**笔记作者**: 三月七老师 & 学生
