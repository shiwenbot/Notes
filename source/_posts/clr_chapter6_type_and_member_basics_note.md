---
title: "【课堂笔记】CLR Chapter 6 - Type and Member Basics"
date: 2026-03-31
categories:
  - 书籍
tags:
  - CLR via C#
description: "> 日期: 2026-03-31"
---

> 日期: 2026-03-31
> 来源: CLR via C# (4th Edition), Pages 151-169

---

## 10 种类型成员

- **Constants**：编译时常量，隐式 `static`
- **Fields**：对象状态数据
- **Instance Constructors**：创建实例时初始化
- **Type Constructors**（静态构造器）：类型首次使用时执行
- **Methods**：执行操作的函数
- **Operator Overloads**：重载运算符（如 `+`, `==`）
- **Conversion Operators**：隐式/显式类型转换
- **Properties**：封装字段访问逻辑的"智能字段"
- **Events**：建立发布-订阅模型的回调机制
- **Nested Types**：定义在其他类型内部的类型

---

## 关键细节：const

- `const` 成员**隐式为 `static`**
- 不能显式给 `const` 添加 `static` 修饰符，否则编译报错
- `const` 的值在**编译时直接嵌入到调用方 IL** 中

---

## 关键细节：Events

- C# 的 `event` 关键字是语法糖
- 编译后生成三部分：
  1. 一个**私有委托字段**
  2. 一个 `add_事件名` 方法
  3. 一个 `remove_事件名` 方法
- 不直接暴露 delegate 字段的原因：
  - 防止外部用 `=` 覆盖整个订阅列表
  - 防止外部直接 `null` 清空订阅者
  - 保证线程安全的修改（通过访问器同步）

---

## 类型可见性

| 位置 | 默认可见性 |
|------|-----------|
| namespace 下的类型 | `internal` |
| 嵌套类型 | `private` |

- 设计原则：**默认保守，公开谨慎**（信息隐藏，降低 breaking change 风险）

---

## Friend Assembly

- 使用 `[assembly: InternalsVisibleTo("AssemblyName")]`
- 允许指定 Assembly 访问 `internal` 成员
- 最常用场景：**单元测试项目**访问被测代码的内部逻辑

---

## 6 种成员访问级别

1. `public`
2. `private`
3. `protected`
4. `internal`
5. `protected internal` — **或** 关系（子类 **或** 同项目）
6. `private protected` — **且** 关系（子类 **且** 同项目）

记忆口诀：`protected internal` 更开放（二选一），`private protected` 更严格（同时满足）

---

## Static Class

- 不能被实例化（不能使用 `new`）
- 只能包含 `static` 成员
- 编译器在 IL 中将其标记为 `abstract sealed`
  - `abstract` → 阻止 new
  - `sealed` → 阻止继承

---

## Partial Class

- 纯**编译器语法糖**
- 多个 `partial` 文件在编译时合并为一个完整类
- 最终 DLL 中不存在 `partial` 概念

---

## call vs callvirt

| 指令 | 特点 | 用途 |
|------|------|------|
| `call` | 编译时静态绑定，**不检查 `this` 是否为 null** | 静态方法、构造器、值类型方法、基类方法调用 |
| `callvirt` | 运行时查虚方法表，**调前检查 null** | 引用类型的实例方法（默认） |

### 重要反直觉点

- **引用类型的非虚实例方法**，C# 编译器也默认使用 `callvirt`
- 原因：统一在调用点进行 null 检查，保证行为一致、可预测
- 如果非虚方法用 `call`，调用 `null` 引用时，如果方法体不访问实例字段，居然能**正常执行完**！这会导致极度诡异的调试体验

---

## Component Versioning：virtual / new / override / sealed

### 核心区别

| 关键字 | 作用 | 是否走虚方法表 |
|--------|------|---------------|
| `new` | 隐藏继承的成员 | ❌ 不走 |
| `override` | 重写虚方法 | ✅ 走虚方法表 |
| `sealed` | 阻止进一步 override | — |

### Phone / BetterPhone 经典示例

```csharp
public class Phone
{
    public void Dial() { Console.WriteLine("Phone.Dial"); }
}

public class BetterPhone : Phone
{
    public new void Dial() { Console.WriteLine("BetterPhone.Dial"); base.Dial(); }
}

Phone p = new BetterPhone();
p.Dial();  // 输出：Phone.Dial
```

- `new` 只是"另开一张同名方法的桌子"，不走虚方法表
- 编译器/运行时根据变量声明类型（`Phone`）查方法表，找不到 `override`，就调用 `Phone.Dial`

### 如果基类后来改成 `virtual`

```csharp
public class Phone
{
    public virtual void Dial() { ... }
}

public class BetterPhone : Phone
{
    public new void Dial() { ... }  // 仍然是 new！
}

Phone p = new BetterPhone();
p.Dial();  // 仍然是 Phone.Dial！
```

- `new` 和 `override` 是**两条永远不相交的高速公路**
- 基类后来加 `virtual`，已有的 `new` 不会自动变成 `override`

### 设计教训

- 基类设计时就要想好方法是否支持多态
- 一开始没加 `virtual`，以后再加可能让大量已有子类"期望落空"

---

## sealed override 的使用场景

- 允许继承此类，但**禁止再 override 此方法**
- 常用于中间层子类做了最终优化，希望后续继承者不要再改动核心逻辑

```csharp
public class BetterPhone : Phone
{
    public sealed override void Dial() { ... }
}

public class SuperPhone : BetterPhone
{
    // ❌ 报错：不能 override 被 sealed 的方法
    public override void Dial() { ... }
}
```

---

## 面试高频考点总结

1. **10 种类型成员是哪些？** — 常量/字段/实例构造器/类型构造器/方法/操作符重载/转换操作符/属性/事件/嵌套类型
2. **const 能不能加 static？** — 不能，const 隐式 static
3. **事件为什么不暴露 delegate 字段？** — 编译为 add/remove 访问器，封装防止覆盖
4. **namespace 下类默认什么可见性？** — `internal`
5. **Friend Assembly 干嘛用的？** — `InternalsVisibleTo`，测试常用
6. **`protected internal` vs `private protected` 哪个更开放？** — `protected internal`（或关系）
7. **为什么非虚实例方法用 `callvirt`？** — 统一 null 检查
8. **`new` 和 `override` 的区别？** — `new` 隐藏不走虚方法表，`override` 重写支持多态
9. **基类后来改成 `virtual`，子类用 `new` 会怎么样？** — 仍然调基类方法（两条不相交的路）

---

**关联章节**:
- 前置：[Chapter 5 - Primitive, Reference, and Value Types](2026-03-31_clr_chapter5-2_boxing_note.md)
- 后续：Chapter 7 - Constants and Fields