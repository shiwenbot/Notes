---
title: "【课堂笔记】CLR via C# Chapter 3 - Shared Assemblies"
date: 2026-03-30
categories:
  - 书籍
tags:
  - CLR via C#
description: "---"
---

---
book: CLR via C# (4th Edition)
chapter: "Chapter 3 - Shared Assemblies and Strongly Named Assemblies"
date: 2026-03-30
duration: 45分钟
---

# CLR via C# Chapter 3 学习笔记

## 核心概念

### 两种 Assembly 类型

| 类型 | 私钥签名 | 可进 GAC | 部署方式 |
|------|---------|---------|---------|
| 弱命名 (Weakly Named) | ❌ 无 | ❌ 不可 | 仅私有部署 |
| 强命名 (Strongly Named) | ✅ 有 | ✅ 可以 | 私有或全局 |

**关键结论**：结构完全相同，区别只在是否用公私钥对签名

### 强命名的四个组成部分

1. **文件名** (Name) - 如 "MyLibrary"
2. **版本号** (Version) - 如 "1.0.0.0"
3. **文化** (Culture) - 如 "neutral" 或 "zh-CN"
4. **公钥** (Public Key) - 发布者的公钥，或简化的 Public Key Token

**Assembly Identity 示例**：
```
"MyLibrary, Version=1.0.8123.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"
```

### Public Key Token

- 公钥的 64-bit 哈希值
- 用于节省存储空间（完整公钥很大）
- ILDasm 中看到的是 Token，不是完整公钥

## GAC（全局程序集缓存）

### 路径
```
%SystemRoot%\Microsoft.NET\Assembly
```

### 优缺点

**优点**：
- 多程序共享，节省物理内存
- 支持 Side-by-Side 版本共存
- 可通过 Publisher Policy 统一升级

**缺点**：
- ❌ 破坏简单部署（XCopy 不再适用）
- 安装/卸载需要管理员权限和专用工具
- 卸载复杂（多个程序共享时不能随意删除）

### 黄金法则
> 只有当 Assembly 需要被很多程序共享时，才放进 GAC。否则一律私有部署！

## 防篡改机制

### 签名过程
1. 编译时计算 Assembly 内容的 hash
2. 用私钥对 hash 签名
3. 签名和公钥嵌入 PE 文件

### 验证过程
1. 运行时重新计算文件 hash
2. 用公钥解密签名得到原始 hash
3. 对比两个 hash，不一致则抛出异常

## 延迟签名 (Delayed Signing)

**使用场景**：
- 私钥保密（锁在保险库/硬件设备）
- 需要对 Assembly 做后处理（如混淆）

**流程**：
1. 开发：`csc /keyfile:MyCompany.PublicKey /delaysign MyAssembly.cs`
2. 注册：`SN -Vr MyAssembly.dll`（跳过验证）
3. 发布：`SN -R MyAssembly.dll MyCompany.snk`（正式签名）

## CLR 类型解析流程

### 搜索顺序（强命名 Assembly）
```
1. GAC
2. 应用程序基目录 / codeBase 指定路径
3. Probing 路径 (privatePath)
4. 未找到 → FileNotFoundException
```

### 搜索顺序（弱命名 Assembly）
```
1. 应用程序基目录
2. Probing 路径
3. 未找到 → FileNotFoundException
```

**注意**：弱命名 Assembly **不会**搜索 GAC

## 版本绑定策略

### Binding Redirect

**使用场景**：程序引用旧版本，但用户机器只有新版本

**配置示例**：
```xml
<configuration>
  <runtime>
    <assemblyBinding>
      <dependentAssembly>
        <assemblyIdentity name="Newtonsoft.Json"
                          publicKeyToken="30ad4fe6b2a6aeed"
                          culture="neutral"/>
        <bindingRedirect oldVersion="12.0.0.0" newVersion="13.0.0.0"/>
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
</configuration>
```

**面试考点**：
- `oldVersion` 可以是范围，如 `1.0.0.0-3.5.0.0`
- 配置优先级：app.config → publisher policy → machine.config

### Publisher Policy

- 发布者打包的版本策略 Assembly
- 名称格式：`Policy.1.0.AssemblyName.dll`
- 安装到 GAC 后对所有引用该 Assembly 的程序生效
- 管理员可用 `<publisherPolicy apply="no"/>` 禁用

## 面试高频问题

### Q1: 强命名和弱命名的区别？
> 结构相同，区别在强命名有发布者的公私钥签名，能唯一标识发布者，可进 GAC。

### Q2: GAC 的优缺点？
> 优点：多程序共享，节省内存。
> 缺点：破坏简单部署，安装/卸载/备份麻烦。

### Q3: CLR 如何验证 Assembly 未被篡改？
> 运行时计算文件 hash，用公钥解密签名对比。hash 不一致则抛出异常。

### Q4: Binding Redirect 的作用？
> 让 CLR 加载新版本代替旧版本，解决版本不匹配导致的 FileLoadException。

### Q5: 为什么编译器不直接从 GAC 找引用？
> 1. GAC 目录结构是 undocumented
> 2. .NET 解决方案：编译器目录和 GAC 各存一份

## 个人理解

1. **从问题出发理解设计**：先理解 DLL Hell 的痛，才能体会强命名+GAC 的价值
2. **抓重点**：工具命令简单带过，核心概念深入理解
3. **面试导向**：Binding Redirect、Publisher Policy 等考点要特别注意
4. **Part I 完成**：三章学完，对 .NET 程序完整生命周期有了系统认知

---

**下一步**：Chapter 4 - Type Fundamentals，进入 Part II（类型设计）