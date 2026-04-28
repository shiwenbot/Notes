---
title: "【课堂笔记】CLR via C# Chapter 11 - Events"
date: 2026-04-28
categories:
  - 书籍
tags:
  - CLR via C#
description: "事件的核心概念、编译本质（委托字段+add/remove）、线程安全触发、显式实现事件与内存优化"
---

# 【课堂笔记】CLR via C# Chapter 11 - Events

**日期**：2026-04-28
**课程**：CLR via C# Chapter 11 - Events
**时长**：约60分钟

---

## 1. 事件的核心概念

事件是一种类型成员，允许对象通知其他对象"发生了某事"。三个核心能力：
- 注册订阅（register interest）
- 取消订阅（unregister interest）
- 通知已注册的方法（notify registered methods）

**观察者模式（Observer Pattern）**的实现：
- 发布者（Publisher）：定义事件的对象
- 订阅者（Subscriber）：注册事件处理方法的对象
- 解耦：发布者不需要知道订阅者的存在

---

## 2. 事件定义的 4 个步骤

### Step 1: 定义 EventArgs 派生类

```csharp
internal class NewMailEventArgs : EventArgs {
    private readonly String m_from, m_to, m_subject;
    
    public NewMailEventArgs(String from, String to, String subject) {
        m_from = from; m_to = to; m_subject = subject;
    }
    
    public String From { get { return m_from; } }
    public String To { get { return m_to; } }
    public String Subject { get { return m_subject; } }
}
```

**约定**：
- 派生自 `System.EventArgs`
- 类名以 `EventArgs` 结尾
- 字段是 `readonly`，属性只有 `get`（不可变）
- 如果不需要传递额外信息，使用 `EventArgs.Empty`

---

### Step 2: 定义事件成员

```csharp
internal class MailManager {
    public event EventHandler<NewMailEventArgs> NewMail;
}
```

**`EventHandler<T>` 的约定**：
```csharp
void MethodName(Object sender, TEventArgs e);
```
- 返回类型 `void`（无法获取多个回调的返回值）
- 第一个参数 `sender`：触发事件的对象（用 Object 是为了兼容继承和多类型复用）
- 第二个参数 `e`：事件数据

---

### Step 3: 定义 protected virtual 方法（触发事件）

```csharp
internal class MailManager {
    protected virtual void OnNewMail(NewMailEventArgs e) {
        // 线程安全：复制到临时变量
        EventHandler<NewMailEventArgs> temp = Volatile.Read(ref NewMail);
        
        if (temp != null) {
            temp(this, e);
        }
    }
}
```

**为什么要 protected virtual？**
- `protected`：子类可以重写，控制事件触发逻辑
- `virtual`：允许派生类扩展行为

---

### Step 4: 定义翻译方法（将输入转化为事件）

```csharp
internal class MailManager {
    public void SimulateNewMail(String from, String to, String subject) {
        NewMailEventArgs e = new NewMailEventArgs(from, to, subject);
        OnNewMail(e);
    }
}
```

---

## 3. 事件的编译本质（⚠️ 面试核心）

### 编译后的 3 个构造

当你写：
```csharp
public event EventHandler<NewMailEventArgs> NewMail;
```

**C# 编译器翻译成**：

```csharp
// 1. 私有委托字段
private EventHandler<NewMailEventArgs> NewMail = null;

// 2. add_NewMail 方法
public void add_NewMail(EventHandler<NewMailEventArgs> value) {
    // 线程安全的 Delegate.Combine
    EventHandler<NewMailEventArgs> prevHandler;
    EventHandler<NewMailEventArgs> newMail = this.NewMail;
    do {
        prevHandler = newMail;
        EventHandler<NewMailEventArgs> newHandler =
            (EventHandler<NewMailEventArgs>)Delegate.Combine(prevHandler, value);
        newMail = Interlocked.CompareExchange<EventHandler<NewMailEventArgs>>(
            ref this.NewMail, newHandler, prevHandler);
    } while (newMail != prevHandler);
}

// 3. remove_NewMail 方法
public void remove_NewMail(EventHandler<NewMailEventArgs> value) {
    // 线程安全的 Delegate.Remove
    // ... 类似的循环结构
}
```

### 为什么委托字段是 private？

**如果委托字段是 public，外部代码可以这样搞破坏**：
```csharp
mm.NewMail = null;  // 清空所有订阅！
mm.NewMail(someArgs);  // 直接触发事件，绕过所有检查！
```

**事件用 private 委托字段 + public add/remove 方法封装起来**，强制你只能用 `+=` 和 `-=` 操作订阅列表！

---

## 4. += 和 -= 的编译原理

### += 的编译

**当你写**：
```csharp
mm.NewMail += FaxMsg;
```

**编译器翻译成**：
```csharp
mm.add_NewMail(new EventHandler<NewMailEventArgs>(this.FaxMsg));
```

**步骤**：
1. 创建 `EventHandler<NewMailEventArgs>` 委托对象，包装 `FaxMsg` 方法
2. 调用 `MailManager.add_NewMail()` 方法
3. `add_NewMail` 内部调用 `Delegate.Combine()`，把新委托添加到委托链

---

### -= 的编译

**当你写**：
```csharp
mm.NewMail -= FaxMsg;
```

**编译器翻译成**：
```csharp
mm.remove_NewMail(new EventHandler<NewMailEventArgs>(this.FaxMsg));
```

**步骤**：
1. 创建委托对象（和 += 一样）
2. 调用 `remove_NewMail()` 方法
3. `remove_NewMail` 内部调用 `Delegate.Remove()`，从委托链中移除

---

## 5. 线程安全的事件触发（⚠️ 面试核心）

### Version 1（有问题）

```csharp
if (NewMail != null) NewMail(this, e);
```

**问题**：Race Condition
- 线程A检查：`NewMail != null` ✅ 通过
- **线程B调用 `mm.NewMail -= FaxHandler`，把最后一个订阅者移除了**
- 线程A继续执行：`NewMail(this, e)` 💥 **NullReferenceException！**

---

### Version 2（改进）

```csharp
EventHandler<NewMailEventArgs> temp = NewMail;
if (temp != null) temp(this, e);
```

**原理**：
- 委托是**不可变**的
- 复制到 `temp` 后，即使另一个线程修改 `NewMail`，`temp` 指向的旧委托链仍然有效
- `temp` 要么是 null，要么是完整的委托链，不会"突然变成 null"

---

### Version 3（最佳实践）

```csharp
EventHandler<NewMailEventArgs> temp = Volatile.Read(ref NewMail);
if (temp != null) temp(this, e);
```

**为什么用 `Volatile.Read`？**
- 理论上，编译器可能优化掉 Version 2 的 `temp` 变量
- `Volatile.Read` 强制在调用点读取，**禁止编译器优化**
- 这是**100%线程安全**的写法

**实际情况**：
- **Jeffrey Richter 说**：JIT 编译器知道 Version 2 这个模式，不会优化掉 `temp`
- 所以 Version 2 在**实践中是安全的**
- 但 Version 3 是**理论上正确的**方式（文档化保证）

---

## 6. EventHandler<T> 的 sender 参数设计

### 为什么 sender 是 Object 类型？

**原因1：继承场景**
```csharp
class MailManager {
    public event EventHandler<NewMailEventArgs> NewMail;
}

class SmtpMailManager : MailManager {
    // 继承了 NewMail 事件
    // 但触发者变成了 SmtpMailManager 类型！
}

// 订阅者写的方法：
void OnNewMail(Object sender, NewMailEventArgs e) {
    SmtpMailManager smtp = (SmtpMailManager)sender;
    // 如果 sender 的类型是 MailManager，这里就会有问题
}
```

---

**原因2：多类型复用**
```csharp
class PopMailManager {
    public event EventHandler<NewMailEventArgs> NewMail;
}

class ImapMailManager {
    public event EventHandler<NewMailEventArgs> NewMail;
}

// 同一个回调方法可以订阅两个不同类型的事件！
void OnNewMail(Object sender, NewMailEventArgs e) {
    if (sender is PopMailManager pop) {
        // 处理 POP 邮件
    } else if (sender is ImapMailManager imap) {
        // 处理 IMAP 邮件
    }
}
```

---

## 7. 取消订阅与 GC（⚠️ 重要）

```csharp
Fax fax = new Fax(mm);
// fax 订阅了 mm.NewMail 事件
// 现在 mm.NewMail 的委托链持有 fax 的引用！

// 即使你不再使用 fax：
fax = null;
// fax 对象仍然不会被 GC 回收！
// 因为 mm.NewMail 的委托链还在引用它
```

**解决方案**：
```csharp
fax.Unregister(mm);  // 先取消订阅
fax = null;          // 现在可以被 GC 回收了
```

**最佳实践**：
- 如果你的类实现了 `IDisposable`，在 `Dispose()` 方法中取消所有事件订阅
- 避免内存泄漏！

---

## 8. 显式实现事件（内存优化）

### 问题：大量事件的内存浪费

```csharp
class Button {
    public event EventHandler Click;
    public event EventHandler DoubleClick;
    public event EventHandler MouseMove;
    // ... 假设有 70 个事件
}
```

**编译后每个事件都会生成**：
```csharp
private EventHandler Click = null;        // 8字节（引用）
private EventHandler DoubleClick = null;  // 8字节
// ... 70 × 8 = 560字节
```

**问题**：
- 大多数对象只订阅 2-3 个事件
- 但所有 70 个委托字段都会分配内存
- **巨大浪费！**

---

### 解决方案：显式实现事件（稀疏存储）

**核心思想**：
- 用一个 `Dictionary<EventKey, Delegate>` 存储所有事件
- **只有被订阅的事件才会占用内存**
- 没被订阅的事件 = 字典中没有 key = 不占内存

---

### EventSet 实现（简化版）

```csharp
// 事件的"键"（用于字典查找）
public sealed class EventKey { }

// 事件集合：用字典存储 EventKey → 委托链
public sealed class EventSet {
    private readonly Dictionary<EventKey, Delegate> m_events =
        new Dictionary<EventKey, Delegate>();

    public void Add(EventKey eventKey, Delegate handler) {
        Monitor.Enter(m_events);
        Delegate d;
        m_events.TryGetValue(eventKey, out d);
        m_events[eventKey] = Delegate.Combine(d, handler);
        Monitor.Exit(m_events);
    }

    public void Remove(EventKey eventKey, Delegate handler) {
        Monitor.Enter(m_events);
        Delegate d;
        if (m_events.TryGetValue(eventKey, out d)) {
            d = Delegate.Remove(d, handler);
            if (d != null)
                m_events[eventKey] = d;
            else
                m_events.Remove(eventKey);
        }
        Monitor.Exit(m_events);
    }

    public void Raise(EventKey eventKey, Object sender, EventArgs e) {
        Delegate d;
        Monitor.Enter(m_events);
        m_events.TryGetValue(eventKey, out d);
        Monitor.Exit(m_events);

        if (d != null) {
            d.DynamicInvoke(new Object[] { sender, e });
        }
    }
}
```

---

### 使用 EventSet 的类型

```csharp
public class TypeWithLotsOfEvents {
    private readonly EventSet m_eventSet = new EventSet();

    protected static readonly EventKey s_fooEventKey = new EventKey();
    protected static readonly EventKey s_barEventKey = new EventKey();

    // 显式实现 Foo 事件
    public event EventHandler<FooEventArgs> Foo {
        add { m_eventSet.Add(s_fooEventKey, value); }
        remove { m_eventSet.Remove(s_fooEventKey, value); }
    }

    // 显式实现 Bar 事件
    public event EventHandler<BarEventArgs> Bar {
        add { m_eventSet.Add(s_barEventKey, value); }
        remove { m_eventSet.Remove(s_barEventKey, value); }
    }

    protected virtual void OnFoo(FooEventArgs e) {
        m_eventSet.Raise(s_fooEventKey, this, e);
    }
}
```

---

### 对比：隐式 vs 显式实现

| 特性 | 隐式实现 | 显式实现 |
|------|---------|---------|
| **内存占用** | 每个事件一个委托字段（固定） | 只存储被订阅的事件（动态） |
| **适用场景** | 少量事件（< 10个） | 大量事件（如 Control 有 70 个） |
| **代码复杂度** | 简单（编译器自动生成） | 复杂（需要手写 add/remove） |
| **性能** | 直接访问字段（快） | 字典查找（稍慢） |

**现实应用**：
- WPF 的 `System.Windows.UIElement`：几十个事件，用 `EventHandlersStore` 类似机制
- WinForms 的 `System.Windows.Forms.Control`：70+ 个事件，用显式实现

---

## 9. 事件 vs 委托字段（面试必考）

| 特性 | 事件 | 委托字段（public） |
|------|------|-------------------|
| **封装性** | ✅ 封装（private 委托字段 + add/remove） | ❌ 无封装（直接暴露） |
| **订阅方式** | 只能用 `+=` 和 `-=` | 可以用 `+=`、`-=`、`=` |
| **外部清空** | ❌ 不能直接清空（`mm.NewMail = null` 编译错误） | ✅ 可以直接清空 |
| **外部触发** | ❌ 不能直接触发（只能在定义类内部） | ✅ 可以直接触发 |
| **适用场景** | 发布-订阅模式（推荐） | 简单回调（不推荐） |

---

## 面试要点总结

### 1. 事件编译后是什么？
**答案**：私有委托字段 + add 方法 + remove 方法

### 2. 为什么要用事件而不是直接暴露委托字段？
**答案**：
- 封装性：防止外部代码直接清空或触发事件
- 只允许通过 add/remove 管理订阅列表

### 3. EventHandler<T> 的约定是什么？
**答案**：
- 返回类型 `void`
- 第一个参数 `Object sender`
- 第二个参数 `TEventArgs e`（派生自 EventArgs）

### 4. 如何线程安全地触发事件？
**答案**：
```csharp
var temp = Volatile.Read(ref NewMail);
if (temp != null) temp(this, e);
```

### 5. 显式实现事件的场景？
**答案**：
- 大量事件（如 Control 有 70 个）
- 大多数事件不会被订阅
- 使用 `Dictionary<EventKey, Delegate>` 优化内存

### 6. 为什么委托字段是 private？
**答案**：防止外部代码直接清空订阅列表或绕过 add/remove 方法

### 7. Delegate.Remove 移除不存在的委托会怎样？
**答案**：静默失败，不抛异常，什么都不做

### 8. 不取消订阅会导致什么问题？
**答案**：对象无法被 GC 回收（委托链持有引用），导致内存泄漏

---

## 课后思考

1. **设计题**：设计一个 `TemperatureSensor` 类，当温度超过阈值时触发 `TemperatureExceeded` 事件。需要定义 EventArgs、事件成员、On 方法。

2. **优化题**：你的类有 30 个事件，大多数情况下只订阅 2-3 个。如何优化内存占用？

3. **线程安全题**：下面的代码有线程安全问题吗？如果有，如何修复？
   ```csharp
   public void RaiseEvent() {
       if (MyEvent != null) MyEvent(this, EventArgs.Empty);
   }
   ```

---

**下节课预告**：Chapter 12 - Generics（泛型）
- 泛型的本质、为什么需要泛型
- 泛型约束、协变和逆变
- 泛型在 CLR 中的实现

---

**笔记创建**：2026-04-28
**笔记作者**：三月七老师 & 学生
