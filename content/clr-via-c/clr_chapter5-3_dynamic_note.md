---
title: "CLR Chapter 5-3 - Dynamic"
date: 2024-01-01T00:00:00+08:00
draft: false
description: "**日期**: 2026-03-31"
categories: ["clr-via-c"]
tags: []
---

**日期**: 2026-03-31
**课程**: CLR via C# - Chapter 5-3 (Part II 第一章完结)
**时长**: 约30分钟

---

## 1. Object Hash Codes

### 1.1 Dictionary 工作原理

```
┌─────────────────────────────────────────────────────┐
│  查找流程：                                          │
│  1. 计算 key.GetHashCode() → 确定 bucket 索引        │
│  2. 在该 bucket 内遍历，用 Equals 精确匹配           │
└─────────────────────────────────────────────────────┘
```

**黄金法则**：如果 `a.Equals(b) == true`，则 `a.GetHashCode() == b.GetHashCode()`

### 1.2 Hash Code 陷阱

**陷阱1：只重写 Equals 不重写 GetHashCode**
- 编译器警告：CS0659
- 后果：Dictionary 用 hash code 找 bucket，相等的对象可能进不同 bucket，永远找不到

**陷阱2：对象存入 Dictionary 后修改了参与哈希码计算的字段**
- 对象的哈希码变了
- 但 Dictionary 不会自动把它搬到新的 bucket
- 结果：想找的时候按新 hash code 找，找不到；旧 bucket 里还留着它，但没人能找到
- **解决方案**：参与哈希码计算的字段用 `readonly` 或不可变类型

### 1.3 好的 GetHashCode 实现

```csharp
internal sealed class Point {
    private readonly Int32 m_x, m_y;  // 不可变字段！
    public override Int32 GetHashCode() {
        return m_x ^ m_y;  // XOR 运算
    }
}
```

**原则**：
- 使用不可变字段计算
- 算法要快速
- 分布要均匀（减少冲突）
- 相等的对象必须返回相同的哈希码

---

## 2. dynamic 类型

### 2.1 本质

```csharp
// 这两行编译后完全一样
dynamic d = 123;
Object d = 123;  // 只是多了 DynamicAttribute
```

**编译器行为**：
- 遇到 `dynamic` → 生成 **payload code**（特殊 IL 代码）
- payload code 在运行时根据实际类型解析操作

### 2.2 Runtime Binder 机制

```csharp
dynamic value = 5;
value = value + value;  // 运行时发生什么？
```

**执行流程**：
1. Runtime Binder（Microsoft.CSharp.dll）检查 `value` 的实际类型 → Int32
2. 根据 C# 规则，两个 Int32 相加 → 整数加法
3. 生成并执行加法代码

**如果是字符串**：
```csharp
dynamic value = "A";
value = value + value;  // 结果："AA"（字符串拼接）
```

### 2.3 dynamic vs var

| 特性 | var | dynamic |
|------|-----|---------|
| 类型确定时机 | 编译时 | 运行时 |
| 实际类型 | 编译器推断的具体类型 | System.Object |
| IntelliSense | ✅ 有 | ❌ 无 |
| 错误检查 | 编译时报错 | 运行时抛异常 |
| 使用场景 | 简化代码，类型明确 | 与 COM/动态语言交互 |

### 2.4 性能代价

**使用 dynamic 的额外开销**：

1. **加载额外 Assembly**
   - Microsoft.CSharp.dll（C# Runtime Binder）
   - System.dll
   - System.Core.dll
   - System.Dynamic.dll（如果用 COM）

2. **生成动态代码**
   - 运行时生成 "Anonymously Hosted DynamicMethods Assembly"
   - 缓存常用调用以加速

3. **运行时解析开销**
   - 每次调用都要查实际类型、找方法、匹配参数

### 2.5 使用建议

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 大量 COM 交互（如 Excel 自动化） | dynamic | 代码简洁，频繁调用有缓存 |
| 与 Python/Ruby 交互 | dynamic | 语法直观 |
| 只有 1-2 处动态调用 | 反射 | 避免加载额外 DLL |
| 性能敏感代码 | 静态类型 | 零额外开销 |

**示例对比**：

```csharp
// 反射方式（繁琐）
Object target = "Jeffrey Richter";
Object arg = "ff";
Type[] argTypes = new Type[] { arg.GetType() };
MethodInfo method = target.GetType().GetMethod("Contains", argTypes);
Object[] arguments = new Object[] { arg };
Boolean result = Convert.ToBoolean(method.Invoke(target, arguments));

// dynamic 方式（简洁）
dynamic target = "Jeffrey Richter";
dynamic arg = "ff";
Boolean result = target.Contains(arg);
```

---

## 3. 面试要点总结

### Hash Code
1. **为什么重写 Equals 必须重写 GetHashCode？**
   - Dictionary/HashSet 先用 hash code 找 bucket，再用 Equals 精确匹配
   - 如果相等但 hash code 不同，会找错 bucket，永远找不到

2. **对象存入 Dictionary 后修改字段会怎样？**
   - 如果字段参与 hash code 计算 → hash code 变了
   - Dictionary 不会自动迁移 → 找不到对象，也无法删除 → 内存泄漏

### dynamic
1. **dynamic 和 object 的区别？**
   - 编译时：dynamic 生成 payload code，object 正常编译
   - 运行时：无区别，都是 Object 引用

2. **dynamic 的性能代价？**
   - 加载 Microsoft.CSharp.dll 等额外 Assembly
   - 运行时生成动态代码
   - 首次调用有解析开销

3. **什么时候用 dynamic，什么时候用反射？**
   - 频繁动态调用 → dynamic（有缓存优化）
   - 偶尔调用 → 反射（避免加载额外 DLL）

---

## 4. 课后思考

1. 设计一个自定义 struct，如何实现高效的 GetHashCode？
2. 如果要在 Dictionary 中存储可变对象，有什么策略可以避免 hash code 变化问题？
3. 阅读 Microsoft.CSharp.dll 的源码，了解 Runtime Binder 的具体实现

---

**下节课预告**: Chapter 6 - Type and Member Basics
- 类型的各种成员（字段、方法、属性、事件等）
- 类型可见性（public/internal）
- 成员访问控制（private/protected/internal/protected internal）
- Static Classes、Partial Classes

**笔记创建**: 2026-03-31
**笔记作者**: 三月七老师 & 学生
