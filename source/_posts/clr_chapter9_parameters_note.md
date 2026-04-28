---
title: "Chapter 9 - Parameters（参数传递机制详解）"
date: 2026-04-28
categories:
  - 书籍
tags:
  - CLR via C#
  - 三月七
description: "可选参数版本陷阱、命名参数求值顺序、var本质、ref/out的IL层面解析、params性能陷阱、API设计原则"
---

# 课堂笔记：Chapter 9 - Parameters

**日期**: 2026-04-28
**课程**: CLR via C# - Chapter 9 (Parameters)
**时长**: 约90分钟

---

## 1. 可选参数和命名参数

### 1.1 默认参数的版本陷阱

**核心机制**：
```csharp
// 你写的代码
private static String MakePath(String filename = "Untitled") {
    return String.Format(@"C:\{0}.txt", filename);
}

// 调用方代码
MakePath();  // 编译后等价于 MakePath("Untitled")
```

**编译器行为**：
- 默认值在**编译期嵌入调用方 IL**
- 调用方的 IL：`call string MakePath(string "Untitled")`
- 默认值是硬编码的

**版本陷阱场景**：
```
程序集 A：MakePath(String filename = "Untitled")     编译 → DLL
程序集 B：调用 MakePath()                             编译 → 引用 A

修改 A：MakePath(String filename = "Default")        重新编译 A
不编译 B：
  → B 的 IL 还是 "Untitled"
  → 拿到的是旧值！
```

**解决方案**：用 null 作为哨兵值
```csharp
// ✅ 推荐：用 null sentinel
private static String MakePath(String filename = null) {
    return String.Format(@"C:\{0}.txt", filename ?? "Untitled");
}
```

**对比 const 的版本陷阱**：
- const 的值嵌入调用方 IL
- 可选参数的默认值也嵌入调用方 IL
- **解决方式相同**：用 null/0 作为哨兵，方法内部决定默认行为

---

### 1.2 命名参数的求值顺序

**常见误解**：命名参数可以乱序传，求值也是乱序？

**真相**：求值顺序永远是从左到右，命名参数只做"值到参数的映射"。

```csharp
private static Int32 s_n = 0;

// 方式4：位置参数
M(s_n++, s_n++.ToString());
// 输出：x=0, s=1

// 方式5：命名参数（看起来乱序）
M(s: (s_n++).ToString(), x: s_n++);
// 输出：x=3, s="2"（不是 x=2, s="3"！）
```

**关键**：
- **书写顺序** = `s: ..., x: ...`（s 在左，x 在右）
- **求值顺序** = 从左到右（先求左边的 s，再求右边的 x）
- **传参顺序** = 按命名参数映射（3 给 x，"2" 给 s）

---

## 2. var 的本质

### 2.1 编译期类型推断

```csharp
var name = "Jeff";
// 编译器推断为 String
// IL 中生成：String name = "Jeff"
```

**var 在编译后不存在**，只是"编译器语法糖"。

### 2.2 var vs dynamic

| 特性 | var | dynamic |
|------|-----|---------|
| 类型确定时机 | **编译时** | **运行时** |
| IL 中的实际类型 | 推断出的具体类型 | System.Object |
| 能否用于字段 | ❌ 不可以 | ✅ 可以 |
| 能否用于参数 | ❌ 不可以 | ✅ 可以 |
| 能否不初始化 | ❌ 必须初始化 | ✅ 可以不初始化 |
| IntelliSense | ✅ 有 | ❌ 无 |

**为什么 var 不能用于字段？**
1. 字段可以被多个方法访问，**类型契约应该显式声明**
2. 防止匿名类型"泄漏"到单个方法之外

### 2.3 var 的重构优势

**场景**：开发过程中可能改变方法返回类型。

```csharp
// 初版
public DataTable GetUser(int id) { ... }
var user = GetUser(123);

// 重构后：改成返回 List<User>
public List<User> GetUser(int id) { ... }
var user = GetUser(123);  // ✅ 不用改这行，编译器自动推断
```

---

## 3. ref 和 out

### 3.1 CLR 层面的本质

**从 CLR 视角，ref 和 out 完全相同**：
- 传递的都是变量的**地址**（指针）
- IL 代码相同
- Metadata 相同（只有 1 bit 差异）

### 3.2 C# 编译器的语义约束

| 特性 | ref | out |
|------|-----|-----|
| 调用方是否需要初始化 | ✅ 必须 | ❌ 不需要 |
| 方法内能否读取值 | ✅ 可以 | ❌ 不能（未定义） |
| 方法内是否必须写入 | 可选 | ✅ 必须 |

### 3.3 为什么必须显式写 ref/out？

**为了让调用方显式声明意图**。

```csharp
Int32 x = 5;
AddVal(ref x);  // ✅ 一眼看出"x 会被修改"
```

### 3.4 ref/out 与重载规则

**允许的重载**：
```csharp
static void Add(Point p) { }         // ✅ 可以
static void Add(ref Point p) { }     // ✅ 可以共存
```

**不允许的重载**：
```csharp
static void Add(ref Point p) { }     // ❌ 不能共存
static void Add(out Point p) { }     // 元数据签名相同
```

**原因**：元数据签名相同（只有 1 bit 差异，但签名不记录）。

### 3.5 ref/out + 引用类型

**真正的用途**：**修改引用本身**，让它指向新对象。

```csharp
FileStream fs;  // 未初始化

StartProcessingFiles(out fs);  // 返回第一个文件

for (; fs != null; ContinueProcessingFiles(ref fs)) {
    fs.Read(...);
}

private static void ContinueProcessingFiles(ref FileStream fs) {
    fs.Close();
    if (noMoreFilesToProcess)
        fs = null;  // 修改引用本身
    else
        fs = new FileStream(...);  // 指向新对象
}
```

### 3.6 ref/out + 引用类型的类型安全

**问题**：为什么 `String` 不能传给 `ref Object`？

```csharp
String s1 = "Jeffrey";
Swap(ref s1, ref s2);  // ❌ 编译错误
```

**原因**：防止方法内部把"应该指向 String 的引用"替换成"指向其他类型的引用"。

**反例**（假设能编译）：
```csharp
SomeType st;
GetAnObject(out st);  // 假设能编译
Console.WriteLine(st.m_val);  // 💥 运行时崩溃！

private static void GetAnObject(out Object o) {
    o = new String('X', 100);  // st 现在指向 String，不是 SomeType
}
```

**解决方案**：用泛型
```csharp
public static void Swap<T>(ref T a, ref T b) { ... }
```

---

## 4. params 可变参数

### 4.1 底层机制

**params 的本质**：编译器自动构造数组。

```csharp
Add(1, 2, 3, 4, 5);
// 编译后等价于
Add(new Int32[] { 1, 2, 3, 4, 5 });
```

### 4.2 性能陷阱

**每次调用 params 方法**：
1. 在堆上分配数组对象
2. 初始化数组元素
3. 用完后等待 GC 回收

**高频调用时，这个开销会很可观**。

### 4.3 BCL 的优化策略

**示例**：`String.Concat` 的重载设计

```csharp
// 1-4 个参数的显式重载（避免数组分配）
public static string Concat(string str0, string str1);
public static string Concat(string str0, string str1, string str2);
public static string Concat(string str0, string str1, string str2, string str3);

// 任意数量参数的 params 重载
public static string Concat(params string[] values);
```

**设计思路**：
- **常见情况**（1-4个参数）→ 显式重载，**不需要数组分配**
- **特殊情况**（5个以上）→ params 重载，接受性能开销

---

## 5. 参数和返回类型的设计原则

### 5.1 参数用最弱类型

**原则**：参数应该用**最弱的类型**（接口 > 基类）。

```csharp
// ✅ 推荐：参数用接口
public void ManipulateItems<T>(IEnumerable<T> collection) { ... }

// ❌ 不推荐：参数用具体类
public void ManipulateItems<T>(List<T> collection) { ... }
```

### 5.2 返回用最强类型

**原则**：返回类型应该用**最强的类型**（具体类 > 接口）。

```csharp
// ✅ 推荐：返回具体类
public FileStream OpenFile() { ... }

// ❌ 不推荐：返回接口
public Stream OpenFile() { ... }
```

**例外**：需要保留内部实现变化的空间时，返回 `IList<String>`。

---

## 6. 默认参数的使用规则

### 6.1 可以用默认参数的地方

- 方法、构造函数、索引器、委托定义

### 6.2 位置规则

**有默认值的参数必须在右边**：
```csharp
// ✅ 正确
void M(int x, int y = 1, int z = 2) { }

// ❌ 错误
void M(int x, int y = 1, int z) { }
```

**例外**：params 必须在最后，且不能有默认值。

### 6.3 默认值必须是编译时常量

```csharp
// ✅ 可以
void M(int x = 1, String s = null) { }

// ❌ 不可以：运行时计算的值
void M(DateTime dt = DateTime.Now) { }
```

### 6.4 ref/out 不能有默认值

**原因**：ref/out 传递的是"变量的地址"，而"地址"没有"默认值"的概念。

---

## 7. 跨语言支持

**C# 的可选参数如何跨语言互操作？**

编译器自动添加：
1. `OptionalAttribute` —— 标记"可以省略"
2. `DefaultParameterValueAttribute` —— 记录默认值

其他语言的编译器读取这些 attribute，实现跨语言调用。

---

## 8. CLR 为什么不支持 C++ 式的 const-ness

### 8.1 C++ 的 const 是"假安全"

- 可以通过 `const_cast` 绕过
- 可以直接操作内存地址

### 8.2 为什么 CLR 不实现真 const？

**性能代价**：每次写内存都要检查"目标对象是不是 const"。

**作为替代**：C# 用**设计层面的不可变性**（如 `String` 类）。

---

## 9. 面试要点总结

### 可选参数和命名参数
1. **为什么可选参数有版本陷阱？** → 默认值嵌入调用方 IL → 用 null sentinel 解决
2. **命名参数的求值顺序是什么？** → 永远从左到右

### var
1. **var 和 dynamic 的区别？** → var：编译期推断 / dynamic：运行期绑定

### ref/out
1. **ref 和 out 在 CLR 层面有什么区别？** → 没有区别，IL 代码相同
2. **为什么 String 不能传给 ref Object？** → 类型安全保护

### params
1. **params 的底层机制是什么？** → 编译器自动构造数组
2. **params 的性能陷阱是什么？** → 每次调用都堆分配数组 → 提供 1-4 个参数的显式重载

### API 设计
1. **参数用最弱类型，返回用最强类型**

---

## 10. 课后思考

1. 你在日常开发中遇到过哪些"默认参数版本陷阱"的情况？
2. 如果让你设计一个高频调用的 API，你会如何设计参数重载？
3. 阅读 `System.String.Concat` 的源码。

---

**下节课预告**: Chapter 10 - Properties
