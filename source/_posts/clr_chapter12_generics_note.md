---
title: "【课堂笔记】CLR via C# Chapter 12 - Generics"
date: 2026-04-29
categories:
  - 书籍
tags:
  - CLR via C#
description: "泛型完整笔记：泛型基础、开放/封闭类型、JIT优化、协变逆变、泛型约束"
---

# 【课堂笔记】CLR via C# Chapter 12 - Generics

**日期**：2026-04-29
**课程**：CLR via C# Chapter 12 - Generics
**时长**：约95分钟（第一课45分钟 + 第二课50分钟）

---

## 第一课：泛型基础与底层机制

### 1. 泛型的核心概念

#### 1.1 什么是泛型？

**泛型** = **算法复用**机制（不同于OO的"代码复用"）

- **OO的代码复用**：继承基类，复用基类的代码
- **泛型的算法复用**：定义算法时不指定数据类型，使用时再指定

#### 1.2 类型参数 vs 类型参数

- **类型参数（Type Parameter）**：定义时的占位符，如 `T`、`TKey`、`TValue`
- **类型参数（Type Argument）**：使用时的具体类型，如 `int`、`string`、`DateTime`

```csharp
// 定义：T 是类型参数
public class List<T> { }

// 使用：int 是类型参数
List<int> list = new List<int>();
```

#### 1.3 泛型的4大优势

1. **源码保护**：不需要暴露算法的源码（vs C++模板）
2. **类型安全**：编译时检查类型，避免运行时错误
3. **代码更简洁**：不需要类型转换
4. **性能更好**：避免装箱拆箱

---

### 2. 泛型的性能优势（⚠️ 面试核心）

#### 2.1 性能测试数据（Jeffrey Richter 测试）

**值类型场景**：1亿次操作

```
List<int>:       1.6秒，GC 6次
ArrayList<int>:  10.9秒，GC 390次
```

**性能差异：7倍！**  
**GC次数差异：65倍！**

#### 2.2 为什么性能差异这么大？

**ArrayList（非泛型）**：
```csharp
ArrayList list = new ArrayList();
list.Add(100);  // Boxing：值类型转引用类型
int x = (int)list[0];  // Unboxing：引用类型转值类型
```

**List<int>（泛型）**：
```csharp
List<int> list = new List<int>();
list.Add(100);  // 直接传值类型，不需要装箱
int x = list[0];  // 直接返回值类型，不需要拆箱
```

#### 2.3 Boxing 的代价

**一次 Boxing = 堆上分配20字节**（64位系统）：
- 对象头（Object Header）：8字节
- 类型指针（Type Pointer）：8字节
- 字段（如 int 字段）：4字节

**频繁 Boxing → 堆快速增长 → 频繁 GC → 性能下降**

---

### 3. 开放类型 vs 封闭类型

#### 3.1 核心概念

**开放类型（Open Type）**：
- 还有**泛型参数未填**的类型
- **不能创建实例**
- 例如：`List<>`、`Dictionary<,>`（arity = 2，有2个类型参数）

**封闭类型（Closed Type）**：
- 所有泛型参数**都已指定**的类型
- **可以创建实例**
- 例如：`List<int>`、`Dictionary<string, int>`

#### 3.2 示例代码

```csharp
// 开放类型：不能实例化
Type t1 = typeof(Dictionary<,>);
Activator.CreateInstance(t1);  // ❌ 抛异常

// 封闭类型：可以实例化
Type t2 = typeof(Dictionary<string, int>);
Activator.CreateInstance(t2);  // ✅ 成功
```

---

### 4. 泛型的静态字段

#### 4.1 核心原理

**每个封闭类型有自己独立的静态字段！**

```csharp
public class List<T> {
    public static int Count = 0;
}

// 这三个 Count 是不同的字段！
List<int>.Count = 10;
List<string>.Count = 20;
List<DateTime>.Count = 30;

Console.WriteLine(List<int>.Count);      // 输出：10
Console.WriteLine(List<string>.Count);   // 输出：20
Console.WriteLine(List<DateTime>.Count); // 输出：30
```

#### 4.2 为什么不共享？

**静态字段存在 type object 里**！

```
List<int>        的 type object → 有自己的 Count 字段
List<string>     的 type object → 有自己的 Count 字段
List<DateTime>   的 type object → 有自己的 Count 字段
```

**三个不同的 type object → 三个不同的 Count 字段 → 不共享！**

---

### 5. 泛型类型继承

#### 5.1 核心原理

**指定类型参数不影响继承层次**！

```
List<T>         继承自 Object
  ↓
List<string>    继承自 Object（不是继承自 List<T>）
List<int>       继承自 Object（不是继承自 List<T>）
```

#### 5.2 泛型的不变性

**List<string> 不能转换成 List<object>**（虽然 string 继承自 object）

```csharp
List<string> strings = new List<string>();
List<object> objects = strings;  // ❌ 编译错误！
```

**为什么不能转换？**

假设**允许**这样转换：

```csharp
List<string> strings = new List<string>();
List<object> objects = strings;  // 假设这行编译通过

objects.Add(123);  // 💥 问题来了！

// 现在 strings 里面是什么？
// strings[0] = 123 ？？？
```

**破坏了类型安全！**

---

### 6. 代码膨胀与 JIT 优化

#### 6.1 JIT 编译策略

**问题**：JIT 会为 `List<T>.Add(T)` 生成几份原生代码？

**答案**：
- **值类型**：每种值类型单独编译（`List<int>.Add`、`List<DateTime>.Add` 是不同的代码）
- **引用类型**：所有引用类型共享一份代码（`List<string>.Add`、`List<Form>.Add` 共享）

#### 6.2 为什么引用类型可以共享？

**所有引用类型的指针大小都是8字节**（64位系统）！

- `List<T>.Add(T)` 的逻辑：在内部数组里存一个指针
- 不管 `T` 是 `string`、`object`、`Form`，存指针的方式都一样
- **所以所有引用类型可以共享同一份原生代码！**

#### 6.3 性能权衡

**优点**：减少代码量（工作集），节省内存

**代价**：第一次使用泛型时，JIT要编译（增加启动时间）

---

## 第二课：泛型高级特性

### 7. 泛型接口

#### 7.1 常用泛型接口

- `IEnumerable<T>`：可枚举接口
- `IEnumerator<T>`：枚举器接口
- `IList<T>`：列表接口
- `ICollection<T>`：集合接口

#### 7.2 泛型接口与非泛型接口的共存

```csharp
// List<T> 同时实现了泛型接口和非泛型接口
public class List<T> : IEnumerable<T>, IEnumerable, ...
{
    // 泛型接口
    public IEnumerator<T> GetEnumerator();  // 来自 IEnumerable<T>
    
    // 非泛型接口（显式实现）
    IEnumerator IEnumerable.GetEnumerator();  // 来自 IEnumerable
}
```

**为什么要共存？**  
**向后兼容**：旧的代码（.NET 1.0）用 `IEnumerable`，新的代码（.NET 2.0+）用 `IEnumerable<T>`

---

### 8. 泛型委托

#### 8.1 Action<T>：无返回值

```csharp
public delegate void Action();
public delegate void Action<in T>(T obj);
public delegate void Action<in T1, in T2>(T1 arg1, T2 arg2);
// ... 最多 16 个参数
```

**使用场景**：执行操作，不需要返回值

```csharp
// 使用场景1：List.ForEach
List<int> numbers = new List<int> { 1, 2, 3 };
numbers.ForEach(x => Console.WriteLine(x));  // Action<int>

// 使用场景2：作为回调
void DoWork(Action<int> callback) {
    callback(100);  // 完成后回调
}

DoWork(x => Console.WriteLine($"完成！{x}"));
```

#### 8.2 Func<TResult>：有返回值

```csharp
public delegate TResult Func<out TResult>();
public delegate TResult Func<in T, out TResult>(T arg);
public delegate TResult Func<in T1, in T2, out TResult>(T1 arg1, T2 arg2);
// ... 最多 16 个参数 + 1 个返回值
```

**关键点**：**最后一个类型参数是返回值类型**！

**使用场景**：计算值，返回结果

```csharp
// 使用场景1：Lambda 表达式
Func<int, int, int> add = (a, b) => a + b;
int sum = add(1, 2);  // 3

// 使用场景2：LINQ 查询
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
var evenNumbers = numbers.Where(x => x % 2 == 0);  // Func<int, bool>

// 使用场景3：策略模式
Func<double, double, double> operation = (a, b) => a + b;
double result = operation(1.5, 2.5);  // 4.0
```

#### 8.3 Predicate<T>：返回 bool

```csharp
public delegate bool Predicate<in T>(T obj);
```

**使用场景**：判断某个条件是否满足

```csharp
// 使用场景：List.Find
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
int firstEven = numbers.Find(x => x % 2 == 0);  // Predicate<int>
```

#### 8.4 三种委托的对比

| 委托 | 返回值 | 典型用途 | 示例 |
|------|--------|----------|------|
| **Action<T>** | void | 执行操作 | `ForEach`、回调 |
| **Func<TResult>** | TResult | 计算值 | `Where`、`Select`、映射 |
| **Predicate<T>** | bool | 判断条件 | `Find`、`Exists` |

---

### 9. 协变和逆变（⚠️ 超难点）

#### 9.1 核心问题

```csharp
// 情况1：不能转换
List<string> strings = new List<string>();
List<object> objects = strings;  // ❌ 编译错误！

// 情况2：可以转换
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;  // ✅ 编译通过！
```

**为什么情况2能编译通过？**

因为 `IEnumerable<T>` 的定义有 `out` 关键字！

```csharp
public interface IEnumerable<out T>  // ← 注意这个 out！
{
    IEnumerator<T> GetEnumerator();
}
```

#### 9.2 协变（Covariance）：`out` 关键字

**定义**：泛型参数用 `out` 修饰，只能用作**返回值**。

```csharp
public interface IEnumerable<out T>  // T 只能返回，不能作为参数
{
    T GetElement();        // ✅ T 作为返回值
    void Add(T element);  // ❌ T 作为参数（编译错误）
}
```

**效果**：`IEnumerable<派生类>` 可以转换成 `IEnumerable<基类>`

```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;  // ✅ 协变允许转换
```

#### 9.3 逆变（Contravariance）：`in` 关键字

**定义**：泛型参数用 `in` 修饰，只能用作**参数**。

```csharp
public interface IComparer<in T>  // T 只能作为参数，不能返回
{
    int Compare(T x, T y);  // ✅ T 作为参数
    T GetElement();         // ❌ T 作为返回值（编译错误）
}
```

**效果**：`IComparer<基类>` 可以转换成 `IComparer<派生类>`

```csharp
IComparer<object> objComparer = ...;
IComparer<string> strComparer = objComparer;  // ✅ 逆变允许转换
```

#### 9.4 生活例子：水果篮子

**协变（out）**：只能往外拿水果的篮子

```csharp
interface IFruitBasket<out T> {
    T GetFruit();  // 往外拿
}

// 苹果篮子可以当水果篮子用
IFruitBasket<Apple> appleBasket = ...;
IFruitBasket<Fruit> fruitBasket = appleBasket;  // ✅
```

**逆变**：只能往里塞水果的处理器

```csharp
interface IFruitProcessor<in T> {
    void Process(T fruit);  // 往里塞
}

// 水果处理器可以当苹果处理器用
IFruitProcessor<Fruit> fruitProcessor = ...;
IFruitProcessor<Apple> appleProcessor = fruitProcessor;  // ✅
```

#### 9.5 游戏开发中的应用场景

**协变：返回敌人/道具列表**

```csharp
public IEnumerable<Enemy> GetAllEnemies() {
    List<Zombie> zombies = GetZombies();
    return zombies;  // ✅ List<Zombie> → IEnumerable<Enemy>
}
```

**逆变：伤害回调/比较器**

```csharp
public void ApplyDamageToUnit(Action<Unit> damageCallback) {
    Action<Soldier> soldierCallback = damageCallback;  // ✅ 协变
    soldierCallback(new Soldier());
}
```

---

### 10. 泛型方法

#### 10.1 在非泛型类中定义泛型方法

```csharp
// 这个类不是泛型的
public class Helper {
    // 但这个方法是泛型的！
    public static void Swap<T>(ref T a, ref T b) {
        T temp = a;
        a = b;
        b = temp;
    }
}
```

#### 10.2 泛型方法 vs 泛型类

**泛型类**：整个类都是泛型的

```csharp
public class MyClass<T> {
    public void Method(T arg) { }
}
// 使用：MyClass<int> obj = new MyClass<int>();
```

**泛型方法**：只有这个方法是泛型的

```csharp
public class MyClass {
    public void Method<T>(T arg) { }
}
// 使用：MyClass obj = new MyClass(); obj.Method<int>(123);
```

#### 10.3 什么时候用泛型方法？

**只有这个方法需要泛型，其他方法不需要**

```csharp
public class MathHelper {
    // 这个方法需要泛型
    public static T Max<T>(T a, T b) where T : IComparable<T> {
        return a.CompareTo(b) > 0 ? a : b;
    }
    
    // 其他方法不需要泛型
    public static int Add(int a, int b) => a + b;
}
```

---

### 11. 类型推断

#### 11.1 编译器自动推断类型参数

```csharp
public static void Swap<T>(ref T a, ref T b) { }

// 有实参：可以推断
int x = 1, y = 2;
Swap(ref x, ref y);  // 编译器推断 T = int
```

#### 11.2 什么时候必须显式指定？

**无实参时**：必须显式指定类型参数

```csharp
public static T Create<T>() where T : new() {
    return new T();
}

Create<int>();  // ✅ 必须显式指定
```

#### 11.3 类型推断的规则

| 场景 | 能否推断 | 示例 |
|------|---------|------|
| **有实参** | ✅ 可以 | `Swap(ref x, ref y)` → T = int |
| **无实参** | ❌ 不能 | `Create<int>()` 必须显式指定 |
| **泛型类** | ❌ 不能 | `List<int>` 必须显式指定 |

**类型推断只适用于泛型方法，不适用于泛型类！**

---

### 12. 泛型约束（⚠️ 面试核心）

#### 12.1 为什么需要约束？

**问题**：T 在编译时是"未知"的

```csharp
public void Method<T>(T obj) {
    int result = obj.CompareTo(null);  // ❌ 编译错误！
}
```

**编译器说**："我不知道 T 有没有 CompareTo 方法！"

**解决方案**：约束 = 限制 T 可以是什么类型

```csharp
public void Method<T>(T obj)
    where T : IComparable<T>  // ← 约束：T 必须实现 IComparable 接口
{
    int result = obj.CompareTo(null);  // ✅ 现在可以调用了！
}
```

#### 12.2 5种泛型约束

**1. 主约束：class 或 struct**

```csharp
where T : class    // T 必须是引用类型
where T : struct   // T 必须是值类型
```

**2. 次要约束：基类或接口**

```csharp
where T : BaseEntity        // T 必须是 BaseEntity 或其派生类
where T : IComparable, IDisposable  // T 必须实现这两个接口
```

**3. 构造器约束：new()**

```csharp
where T : new()  // T 必须有无参构造器
```

#### 12.3 约束的完整示例

```csharp
public T Create<T>()
    where T : class, IComparable, IDisposable, new()
{
    T obj = new T();           // ✅ 可以创建（new()）
    obj.CompareTo(null);       // ✅ 可以调用（IComparable）
    obj.Dispose();             // ✅ 可以调用（IDisposable）
    return obj;
}
```

#### 12.4 约束的对比

| 约束类型 | 关键字 | 作用 | 示例 |
|---------|--------|------|------|
| **主约束** | class、struct | 限制是引用类型还是值类型 | `where T : class` |
| **次要约束** | 基类、接口 | 限制继承关系或接口实现 | `where T : BaseEntity` |
| **构造器约束** | new() | 要求有无参构造器 | `where T : new()` |

---

## 面试要点总结

### 1. 为什么需要泛型？
- 算法复用（不同于OO的代码复用）
- 类型安全、避免装箱、代码更简洁、性能更好

### 2. 泛型 vs C++模板的区别？
- C++模板是编译期文本替换（需要源码）
- C#泛型是运行时机制（IL支持，只需DLL）

### 3. List<Int32> vs ArrayList<Int32> 性能差异？
- List无装箱，ArrayList每次Add装箱
- 性能差异7倍，GC次数差异巨大（65倍）

### 4. 什么是开放类型和封闭类型？
- 开放类型：带有泛型参数，不能实例化（如 `List<>`）
- 封闭类型：所有类型参数已指定，可以实例化（如 `List<int>`）

### 5. List<int> 和 List<string> 共享静态字段吗？
- 不共享！每个封闭类型有独立的静态字段

### 6. List<String> 能转换成 List<Object> 吗？
- 不能！泛型类不支持协变/逆变

### 7. IEnumerable<String> 能转换成 IEnumerable<Object> 吗？
- 能！因为 `IEnumerable<out T>` 支持协变

### 8. 什么是协变和逆变？
- 协变（out）：只能用作返回值，`IEnumerable<out T>`，派生类→基类
- 逆变：只能用作参数，`IComparer<in T>`，基类→派生类

### 9. 泛型约束有哪些？
- 主约束：class、struct
- 次要约束：基类、接口
- 构造器约束：new()

---

## 课后思考

1. **设计题**：设计一个泛型方法 `Max<T>`，返回两个值中的较大者。需要添加什么约束？

2. **实战题**：在Unity中实现一个敌人管理系统，使用泛型接口返回 `IEnumerable<Enemy>`，实际存储的是 `List<Zombie>` 和 `List<Skeleton>`。

3. **深入思考**：为什么泛型类不支持协变/逆变？如果支持会有什么问题？

---

**下节课预告**：Chapter 13 - Character, Equals, and GetHashCode
- Object 的 Equals 方法
- Object 的 GetHashCode 方法
- 操作符重载
- 动态绑定

---

**笔记创建**：2026-04-28
**笔记作者**：三月七老师 & 学生
