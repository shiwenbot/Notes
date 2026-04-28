---
title: "【课堂笔记】CLR via C# Chapter 2-2 - Building & Packaging"
date: 2026-03-30
categories:
  - 书籍
tags:
  - CLR via C#
description: "**Assembly Linker、版本管理、文化与私有部署**"
---

**Assembly Linker、版本管理、文化与私有部署**

**日期**: 2026-03-30
**章节**: CLR via C# - Chapter 2 (后半部分)
**时长**: 约45分钟

---

## 核心概念速览

- **AL.exe**: 当项目包含多种编程语言编写的模块时，用于组合这些模块的工具
- **AssemblyVersion**: CLR加载时使用的版本号，存储在Manifest中
- **AssemblyFileVersion**: Windows文件属性中显示的版本号，给人看的
- **AssemblyInformationalVersion**: 产品整体版本，与具体Assembly版本无关
- **Satellite Assembly**: 只包含资源（无代码）的程序集，用于本地化
- **XCopy部署**: 直接复制文件夹即可部署，无需注册表
- **Probing**: CLR搜索Assembly文件的规则

---

## 恍然大悟时刻（重点！）

### 1. 为什么需要 AL.exe？

**我的疑问**: 不同语言写的模块怎么组合在一起？

**探索过程**:
- 三月七问：csc.exe 能直接处理 VB.NET 的源代码吗？
- 我意识到：每个编译器只懂自己的语言
- 那怎么办？先用各自编译器编译成 IL，再用 AL.exe 组合

**最终解释**:
- **AL.exe 是项目经理**
- C# 编译器 → 把 C# 变成 IL
- VB 编译器 → 把 VB 变成 IL
- AL.exe → 把这些 IL 模块打包成一个 Assembly

**类比**: 小明说中文，小红说法语，项目经理（AL.exe）把他们的工作成果整合成一个项目交付物。

---

### 2. 三个版本号的区别

**我的疑问**: 为什么要有三个版本号？一个不够吗？

**最终解释**:

| 版本号 | 给谁看 | 存在哪里 | 用途 |
|--------|--------|----------|------|
| **AssemblyVersion** | CLR | Manifest | 🔴 加载时匹配，**最重要** |
| **AssemblyFileVersion** | 用户/运维 | Win32资源 | 文件属性显示 |
| **AssemblyInformationalVersion** | 产品经理 | Win32资源 | 产品整体版本 |

**记忆点**:
- CLR 只看 **AssemblyVersion**
- 人看 **FileVersion** 和 **InformationalVersion**

---

### 3. 卫星程序集的设计

**我的疑问**: 多语言支持怎么实现？每个语言都复制一份代码？

**最终解释**:

```
Game.exe              ← 中性程序集：代码 + 默认资源
├── zh-CN\
│   └── Game.resources.dll  ← 简体中文卫星程序集
├── en-US\
│   └── Game.resources.dll  ← 英文卫星程序集
└── ja-JP\
    └── Game.resources.dll  ← 日文卫星程序集
```

**设计优点**:
- 代码只存一份（中性程序集）
- 每个语言只下载自己需要的资源包
- 切换语言时只需加载对应的卫星程序集

---

### 4. XCopy部署的革命性

**我的疑问**: .NET 如何解决 DLL Hell？

**最终解释**:

**传统 DLL Hell**:
- 所有程序共享 C:\Windows\System32 里的同一个 DLL
- 安装新程序可能覆盖旧版本 → 旧程序崩溃

**.NET 解决方案**:
- **私有部署**：每个程序带自己的 Assembly 版本
- **Manifest 自描述**：Assembly 自己记录依赖信息
- **无需注册表**：直接复制文件夹就能运行

**类比**: 从"大家共用厨房（DLL Hell）"变成"每家有独立厨房（私有部署）"

---

### 5. Probing 规则

**我的疑问**: CLR 怎么找到引用的 Assembly？

**最终解释**:

CLR 按以下顺序搜索（以 AsmName 为例）：
1. `AppDir\AsmName.dll`
2. `AppDir\AsmName\AsmName.dll`
3. `AppDir\privatePath\AsmName.dll`
4. `AppDir\privatePath\AsmName\AsmName.dll`
5. 没找到？换 `.exe` 后缀再试一遍

**配置文件**:
```xml
<probing privatePath="Plugins;Modules" />
```
告诉 CLR 额外去哪些子目录找。

---

## 待深入研究

- [ ] 多文件 Assembly 的实际项目案例
- [ ] Satellite Assembly 在 .NET Core 中的变化（资源管理器的改进）
- [ ] Probing 性能优化（大量子目录时的加载速度）

---

## 三月七的课后留言

> 今天表现不错！能够主动把握学习节奏，知道什么时候需要深入、什么时候可以略过。
>
> Chapter 2 全部结束了，你对 .NET 程序的"构建-打包-部署"流程有了完整认知。
>
> 下节课 Chapter 3 是面试重点——强命名、GAC、版本绑定策略，准备好迎接挑战吧！
>
> ——三月七 💫

---

**笔记创建**: 2026-03-30
**关联文件**: [progress.md](../progress.md) | [diary.md](../diary.md)