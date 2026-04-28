---
title: "【课堂笔记】CLR Chapter 10 - Properties"
date: 2026-04-28
categories:
  - 书籍
tags:
  - CLR via C#
description: "CLR via C# Chapter 10 学习笔记——属性的编译产物、自动实现属性、对象/集合初始化器、匿名类型、属性vs字段、JIT内联优化等"
---

# 课堂笔记：Chapter 10 - Properties

**日期**: 2026-04-28
**课程**: CLR via C# - Chapter 10 (Properties)

---

## 1. 无参属性（Parameterless Properties）

### 1.1 属性的编译产物

**核心机制**：

```csharp
// 你写的属性
public sealed class Employee {
    private String m_Name;
    
    public String Name {
        get { return m_Name; }
        set { m_Name = value; }
    }
}

// 编译器实际生成的是三个东西：
// 1. get_Name 方法
public String get_Name() {
    return m_Name;
}

// 2. set_Name 方法
public void set_Name(String value) {
    m_Name = value;
}

// 3. Property definition in metadata（属性定义的元数据）
// - 记录 "get_Name 和 set_Name 是一对"
// - 记录 "Name 属性的类型是 String"
```

**调用属性 = 调用方法**：

```csharp
Employee e = new Employee();
e.Name = "Jeffrey";  // 编译为：e.set_Name("Jeffrey");
String name = e.Name; // 编译为：name = e.get_Name();
```

---

### 1.2 metadata 的作用

**metadata 不用于运行时**：

> "For the CLR, properties are just meaningless metadata. The CLR just requires **methods**."

- **运行时**：CLR 只认方法，不认 metadata
- **编译时/工具时**：metadata 用来"打包"方法对

**metadata 的用途**：
1. **IDE 和工具**：IntelliSense、反射工具（Reflector、ILSpy）
2. **跨语言互操作**：不同语言的编译器都知道"调用 get_XXX/set_XXX"
3. **反射**：`typeof(Employee).GetProperty("Name")` 能找到属性

---

### 1.3 自动实现属性（Automatically Implemented Properties）

**语法**：

```csharp
public class Employee {
    public String Name { get; set; }  // 自动实现属性
}
```

**编译器生成**：

```csharp
private String <Name>k__BackingField;  // 私有字段（编译器生成）

public String Name {
    get { return <Name>k__BackingField; }
    set { <Name>k__BackingField = value; }
}
```

**限制**：

1. **无法访问 backing field**：字段名包含 `<` 和 `>`，不是合法的 C# 标识符
2. **无法做参数验证、线程安全、懒加载、逻辑字段**
3. **序列化问题**：字段名每次编译可能不同

**Jeffrey Richter 的态度**：

> "I strongly suggest that all of your fields be private. Then, to allow a user of your type to get or set state information, you expose methods for that specific purpose."

- **强烈推荐**：显式定义私有字段 + 手写 get/set
- **不推荐**：自动实现属性（和公有字段几乎一样，失去封装意义）
- **什么时候可以用**：简单的 POCO 类、数据传输对象（DTO）

---

### 1.4 对象初始化器（Object Initializers）

**语法**：

```csharp
Employee e = new Employee {
    Name = "Jeffrey",
    Age = 48
};
```

**编译器生成：临时变量 + 异常安全**

```csharp
// 你写的
Employee e = new Employee {
    Name = "Jeffrey",
    Age = -5  // 假设这里抛异常
};

// 编译器实际生成的
Employee <>g__initLocal0 = new Employee();  // 临时变量
<>g__initLocal0.Name = "Jeffrey";          // ✅ 成功
<>g__initLocal0.Age = -5;                  // ❌ 抛异常
e = <>g__initLocal0;                       // 永远不会执行
```

**异常安全机制**：

- **全部成功** → e 得到完整初始化的对象
- **中间失败** → e 保持原值（`null`），不会得到"部分初始化"的对象

> "if any of the property setters throws an exception, the newly allocated object **will not be accessible**."

---

### 1.5 集合初始化器（Collection Initializers）

**语法**：

```csharp
List<Int32> numbers = new List<Int32> { 1, 2, 3, 4, 5 };
```

**编译器生成：多次 Add 调用**

```csharp
List<Int32> numbers = new List<Int32>();
numbers.Add(1);
numbers.Add(2);
numbers.Add(3);
numbers.Add(4);
numbers.Add(5);
```

**条件**：类型必须实现 `IEnumerable` + 有 `Add` 方法。

**花括号 vs 方括号**：

```csharp
// 花括号 → 调用 Add
var dict = new Dictionary<String, Int32> {
    { "Jeff", 48 },    // dict.Add("Jeff", 48)
    { "Richter", 50 }
};

// 方括号 → 调用索引器
var dict2 = new Dictionary<String, Int32> {
    ["Jeff"] = 48,     // dict2["Jeff"] = 48
    ["Richter"] = 50
};
```

---

### 1.6 匿名类型（Anonymous Types）

**语法**：

```csharp
var person = new { Name = "Jeff", Age = 48 };
```

**编译器生成**：

```csharp
[CompilerGenerated]
internal sealed class <>__AnonType0<String, Int32> {
    private readonly <Name>i__Field;
    private readonly <Age>i__Field;
    
    public String Name { get { return <Name>i__Field; } }
    public Int32 Age { get { return <Age>i__Field; } }
    // 重写 Equals、GetHashCode、ToString
}
```

**类型复用规则**：属性名称、类型、顺序**都相同**才复用同一个类型。

```csharp
var p1 = new { Name = "Jeff", Age = 48 };
var p2 = new { Name = "Richter", Age = 50 };
p1 = p2;  // ✅ 同一个类型

var p3 = new { Age = 48, Name = "Jeff" };  // 顺序不同
p1 = p3;  // ❌ 编译错误
```

**使用场景**：LINQ 查询、临时数据传递（只能用 `var`）。

---

## 2. 属性 vs 字段的差异

### 属性能做到但字段做不到的

1. **只读或只写**：`public String Name { get; private set; }`
2. **抛异常**：set 中做参数验证
3. **不能作为 ref/out 参数**
4. **可能执行慢**（懒加载、计算属性）
5. **多次调用可能返回不同值**
6. **可能有副作用**
7. **可能返回内部状态的引用**

### 什么时候用属性，什么时候用方法

- **用属性**：简单数据访问（验证、懒加载、线程安全），看起来像字段
- **用方法**：耗时操作、有副作用、每次返回不同值、需要 ref/out

---

## 3. 属性访问器的不同可访问性

```csharp
// 外部只读
public String Name { get; private set; }

// 派生类可写
public String Name { get; protected set; }

// 同程序集可写
public String Name { get; internal set; }
```

**规则**：属性本身必须最宽松，get/set 只能更严格。

---

## 4. 属性不能引入泛型类型参数

```csharp
// ❌ 编译错误
public T MyProperty<T> { get; set; }

// ✅ 方法可以
public T MyMethod<T>() { ... }
```

**原因**：属性代表对象的固定状态，泛型意味着类型可变，破坏语义。

**解决方案**：用方法或索引器代替。

---

## 5. 属性访问器的 JIT 内联优化

**内联 = 把方法代码直接嵌入调用处**

```csharp
// 你写的
String name = e.Name;

// JIT 编译后（release）
String name = e.m_Name;  // 直接访问字段，无方法调用！
```

- **简单属性**：✅ 会被内联，无性能损失
- **复杂属性**（懒加载、计算）：❌ 不会内联，应该改成方法

---

## 6. System.Tuple 类型

| 特性 | Tuple | 匿名类型 |
|------|-------|---------|
| 属性名 | `Item1`, `Item2`... 无语义 | 可自定义，有语义 |
| 跨方法传递 | ✅ | ❌ |
| 作为返回值 | ✅ | ❌ |

- **跨方法传递** → Tuple
- **只在方法内用** → 匿名类型
- **需要语义化属性名** → 定义新类

---

## 7. 面试要点

1. **属性编译后变成什么？** → `get_XXX`/`set_XXX` 方法 + metadata
2. **CLR 运行时用 metadata 吗？** → 不用，只用方法
3. **对象初始化器异常安全？** → 编译器生成临时变量，全部成功才赋值
4. **集合初始化器本质？** → 多次 Add 调用
5. **匿名类型复用规则？** → 名称、类型、顺序都相同
6. **属性访问有性能损失吗？** → 简单属性 JIT 内联，无损失
7. **什么时候用属性 vs 方法？** → 像字段用属性，耗时/副作用用方法

---

## 8. 课后思考

1. 你在日常开发中，遇到过哪些"属性应该改成方法"的案例？
2. 如果让你设计一个 POCO 类，你会用自动实现属性还是手写属性？为什么？
3. 阅读 `System.String` 的源码，看看哪些是属性，哪些是方法，为什么这样设计？

---

**下节课预告**: Chapter 11 - Events
