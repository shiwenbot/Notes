---
title: "【课堂笔记】CLR Chapter 2-1 - Building & Packaging"
date: 2026-03-30
categories:
  - 书籍
tags:
  - CLR via C#
description: "**日期**: 2026-03-30"
---

**日期**: 2026-03-30
**章节**: CLR via C# Chapter 2 (前半部分)
**时长**: 约50分钟

---

## 核心概念速览

- **DLL Hell**: 多个程序共享系统目录的同一个DLL，版本冲突导致崩溃
- **Private Deployment**: 每个程序自带依赖，放在自己的目录里，互不干扰
- **Manifest**: Assembly的自我描述信息，包含版本、依赖、导出类型等
- **XCopy部署**: 直接复制文件夹即可运行，无需安装程序
- **Metadata**: 托管PE文件的核心，包含Definition/Reference/Manifest三类表
- **多文件Assembly**: 一个Assembly可包含多个Module，按需加载

---

## 恍然大悟时刻

### 1. DLL Hell 的根源

**我的疑问**: 为什么会发生 DLL Hell？问题的根源是什么？

**探索过程**:
- 三月七引导我用 Unity 第三方插件版本冲突做类比
- 发现 COM/DLL 时代，DLL 放在系统目录被所有程序共享
- 程序A要v1.0，程序B要v2.0，同一个文件无法满足两个版本需求

**最终解释**:
- **共享 vs 隔离** 的根本矛盾
- Windows 为了"节约磁盘空间"让 DLL 全局共享，但忽略了版本隔离需求
- .NET 用 **Private Deployment** 解决：每个程序有自己的依赖副本

**类比/记忆点**:
- Unity 项目里的 Plugins 文件夹——每个项目带自己的插件版本

---

### 2. Manifest 的作用

**我的疑问**: Assembly 怎么实现"自我描述"，不需要注册表？

**探索过程**:
- 三月七问：Assembly 里有什么东西记录了它的依赖？
- 联想到 Metadata，但具体是 Manifest 表
- 原来 Manifest 记录了 Assembly 包含的文件、依赖的其他 Assembly、版本信息等

**最终解释**:
- **Manifest** 是一组特殊的 Metadata 表
- 包含：AssemblyDef(自身信息)、FileDef(包含的文件)、AssemblyRef(引用的外部Assembly)、ExportedTypesDef(导出的公共类型)
- CLR 加载时读取 Manifest，就能找到所有依赖，**无需注册表**

**类比/记忆点**:
- 像快递单——上面写清楚了包裹里有什么、从哪来、到哪去

---

### 3. Metadata 的三大类表

**我的疑问**: `System.Console.WriteLine` 在哪种 Metadata 表里？

**探索过程**:
- 三月七问：这是你自己定义的还是外部引用的？
- 发现 "自己定义的" 和 "引用的外部类型" 应该分开记录
- Definition 表放自己定义的，Reference 表放引用的

**最终解释**:
| 类别 | 表名 | 用途 |
|------|------|------|
| Definition | TypeDef, MethodDef, FieldDef... | 记录自己定义的类型/方法/字段 |
| Reference | TypeRef, MemberRef, AssemblyRef... | 记录引用的外部类型/成员/Assembly |
| Manifest | AssemblyDef, FileDef... | 记录Assembly自身信息和组成 |

**类比/记忆点**:
- 像名片夹：Definition 是自己印的名片，Reference 是别人给的名片

---

### 4. 多文件 Assembly 的价值

**我的疑问**: 既然一个Assembly通常就是一个文件，为什么要支持多文件？

**探索过程**:
- 三月七提示：想想互联网下载、按需加载的场景
- 想到 UI 控件库的例子——用户可能只用一小部分控件
- 如果打包在一个大文件里，下载太浪费

**最终解释**:
- **按需下载**：把不常用的类型放到单独的 Module，用户不访问就不下载
- **按需加载**：运行时只有访问到某个 Module 里的类型，CLR 才会加载那个文件
- **多语言混合**：C#写的放A文件，VB写的放B文件，合并成一个 Assembly

**类比/记忆点**:
- 像游戏 DLC——本体是必装的，DLC 是可选的，按需下载

---

## 待深入研究

- [ ] Assembly Linker (AL.exe) 的具体使用场景
- [ ] Satellite Assembly 的实际项目配置
- [ ] CLR Probing 规则的完整细节和性能影响
- [ ] 用 ILDasm 实际查看一个 DLL 的 Metadata 表

---

## 三月七的课后留言

> 嘿嘿，今天的课很扎实呢！>
> 从 DLL Hell 讲到 XCopy 部署，再到 Metadata 和多文件 Assembly，你理解得超快的～特别是你用 Unity 第三方插件版本冲突来类比 DLL Hell，一下子就抓住核心了！
>
> 我感觉我们配合越来越默契了，你不仅回答问题，还会主动举出 Unity 开发里的对应场景，这种迁移能力对进大厂特别重要。
>
> 下节课继续 Chapter 2 后半部分，版本号那块面试常考的！加油～

---

**笔记创建**: 2026-03-30
**下次复习**: Chapter 2-2