---
title: "【课堂笔记】CLR via C# Chapter 8-1 - Constructors"
date: 2026-04-01
categories:
  - 书籍
tags:
  - CLR via C#
description: "**日期**: 2026-04-01"
---

**日期**: 2026-04-01
**章节**: CLR via C# Chapter 8 - Methods（Instance Constructors + Type Constructors）
**时长**: 约50分钟

---

## 核心概念速览

- **引用类型构造器 .ctor**: metadata 中名为 `.ctor`，内存先清零再调用，不继承
- **Code Explosion**: 多构造器 + 内联初始化 = 初始化代码被复制多次
- **绕过构造器**: MemberwiseClone、反序列化（GetUninitializedObject）
- **值类型构造器限制**: CS0568（无参）、CS0573（内联初始化），必须显式调用
- **类型构造器 .cctor**: JIT 首次访问触发，互斥锁保证线程安全

---

## 恍然大悟时刻（重点！）

### 1. 构造器里调虚方法的危险

**我的疑问**: 为什么不能？最多就是基类的实现执行呗？

**探索过程**: 三月七让我看代码执行顺序
```csharp
class Base {
    public Base() { DoSomething(); }  // 构造器里调虚方法
    public virtual void DoSomething() { }
}
class Derived : Base {
    private string name = "三月七";  // 字段初始化
    public override void DoSomething() { Console.WriteLine(name.Length); }
}
```

**实际执行顺序**:
1. 分配内存，全部清零（name = null）
2. 执行 Base 构造器 → 调用 DoSomething()
3. **虚方法调用走到 Derived.DoSomething()**
4. **访问 name.Length → 💥 NullReferenceException！**
5. 执行 Derived 字段初始化（name = "三月七"）← 已经晚了！
6. 执行 Derived 构造器体

**最终解释**: 构造器里调虚方法，子类重写的方法会在**子类字段初始化前**执行，导致访问未初始化的字段！

**记忆点**: "基类在盖房子，还没轮到子类铺地板，你就冲进子类房间了！"

---

### 2. Code Explosion（代码膨胀）

**我的疑问**: 内联初始化不是很方便吗？有什么问题？

**探索过程**: 三月七让我数——3个构造器 × 3个内联初始化字段 = IL里出现几次？

**答案**: **9次！** 每个构造器开头都有这3行初始化代码！

```csharp
// 3个构造器 + 3个内联初始化字段
internal sealed class SomeType {
    private Int32 m_x = 5;
    private String m_s = "Hi there";
    private Double m_d = 3.14159;
    
    public SomeType() { }
    public SomeType(Int32 x) { ... }
    public SomeType(String s) { ...; m_d = 10; }
}

// 实际 IL（简化）:
.ctor() {
    m_x = 5; m_s = "Hi there"; m_d = 3.14159;  // ← 第1次
    base..ctor();
}
.ctor(int x) {
    m_x = 5; m_s = "Hi there"; m_d = 3.14159;  // ← 第2次！重复！
    base..ctor();
    // 用户代码...
}
.ctor(string s) {
    m_x = 5; m_s = "Hi there"; m_d = 3.14159;  // ← 第3次！重复！
    base..ctor();
    m_d = 10;  // 注意：先内联赋值，再覆盖，重复赋值！
}
```

**优化方案**: `: this()` 调用公共构造器

```csharp
public SomeType() {
    m_x = 5; m_s = "Hi there"; m_d = 3.14159;  // 只在这里初始化
}
public SomeType(Int32 x) : this() {  // 调用上面的
    m_x = x;
}
public SomeType(String s) : this() {
    m_s = s;
}
```

**最终解释**: 内联初始化是语法糖，会被复制到每个构造器。多个构造器时用 `: this()` 复用初始化代码。

---

### 3. 绕过构造器的风险

**我的疑问**: MemberwiseClone 和反序列化为啥要绕过构造器？有啥风险？

**三月七反问**: 如果构造器里有参数校验，比如年龄不能为负数呢？

**场景**:
```csharp
public class Person {
    public int Age { get; }
    public Person(int age) {
        if (age < 0) throw new ArgumentException("Age cannot be negative");
        Age = age;
    }
}

// 反序列化绕过构造器
Person p = (Person)formatter.Deserialize(stream);  // Age 可能为 -100！
```

**最终解释**: 反序列化时类定义可能已变，调用构造器可能有副作用（如记日志），所以直接分配内存填数据。但这会**绕过所有校验逻辑**，对象可能处于非法状态！

**我的发现**: "这就是安全风险！后续代码以为对象合法，其实不合法！"

---

### 4. 值类型 `this = new` 语法

**我的疑问**: struct 构造器里为什么可以写 `this = new SomeValType()`，class 里不行？

**探索过程**: 三月七反问——Chapter 5 学过值类型和引用类型的赋值语义差异吗？

**答案**:
| 类型 | 赋值行为 | `this` 代表 |
|------|---------|------------|
| 值类型 (struct) | **完整复制所有字段** | 栈上的整个结构体值 |
| 引用类型 (class) | **复制引用地址** | 指向堆对象的引用 |

**struct 可以**:
```csharp
public SomeValType(int x) {
    this = new SomeValType();  // ✅ 清零所有字段
    this.m_x = x;  // 只改想改的
}
```
这是值复制——把新创建的结构体的所有字节复制到当前位置。

**class 不行**:
```csharp
public SomeClass() {
    this = new SomeClass();  // ❌ this 是 readonly 引用，不能重新赋值
}
```

**最终解释**: 值类型赋值是**复制整个值**，`this = new` 就是清零；引用类型赋值是**改变引用指向**，而 this 在类中是 readonly，不能被重新赋值。

**记忆点**: "struct 的 this 是"这包东西"，可以整包替换；class 的 this 是"指向东西的箭头"，箭头固定不能改。"

---

### 5. 类型构造器 .cctor 的线程安全

**我的疑问**: 多个线程同时访问一个类，static 构造器会执行几次？

**答案**: **只执行一次**，CLR 用互斥锁保证！

**触发时机**（JIT 首次访问类型时）:
- 首次访问该类型的 **静态字段**
- 首次调用该类型的 **静态方法**
- 首次创建该类型的 **实例**（new）
- 首次访问该类型的 **静态属性/事件**

**线程安全机制**:
```
线程A → 要访问 SomeType → JIT 发现需要执行 .cctor
       → 获取互斥锁 → 执行 .cctor → 释放锁 → 继续
线程B → 同时要访问 SomeType → 阻塞等待锁
       → 线程A释放后 → 发现已初始化标记 → 跳过 .cctor
```

**最终解释**: JIT 编译器在首次访问时触发，CLR 用互斥锁保证每个 AppDomain 内只执行一次。

**陷阱警告**: 不要在值类型中定义类型构造器！
```csharp
struct SomeValType {
    public static int s_x = 5;
    static SomeValType() { /* 这个可能永远不会执行！ */ }
}

SomeValType[] arr = new SomeValType[100];  // 创建数组不触发 .cctor
Console.WriteLine(SomeValType.s_x);  // 可能读到 0 而不是 5！
```

---

## 待深入研究

- [ ] 类型构造器的循环引用场景（CLR 不保证执行顺序）
- [ ] 静态字段内联初始化与显式 static constructor 的执行顺序细节
- [ ] `FormatterServices.GetSafeUninitializedObject` 与 `GetUninitializedObject` 的区别

---

## 三月七的课后留言

"今天表现超棒的！特别是你主动提出反序列化绕过校验的安全风险——这种从机制层面思考问题的能力，是高级程序员的必备素质！

构造器的内容比想象的多吧？从 .ctor 到 .cctor，从 code explosion 到线程安全，每个细节背后都有设计考量。继续加油，Chapter 8-2 的操作符重载和扩展方法更精彩！"

——三月七 ✨