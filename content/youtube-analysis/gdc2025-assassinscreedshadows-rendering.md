---
title: "GDC2025: Assassin's Creed Shadows 渲染技术 (v1)"
date: 2024-01-01T00:00:00+08:00
draft: false
description: "> **元信息**：GDC 2025 | 时长 ~62min | 演讲者：Nicolas Lopez (Ubisoft Anvil 引擎团队)"
categories: ["youtube-analysis"]
tags: []
---

> **元信息**：GDC 2025 | 时长 ~62min | 演讲者：Nicolas Lopez (Ubisoft Anvil 引擎团队)
> **YouTube**: https://www.youtube.com/watch?v=yj5pYktC3X8
> **生成日期**：2026-04-08
> **难度**：⭐⭐⭐⭐ (高级渲染技术)

## 📋 内容概览

本演讲系统性地介绍了育碧 Anvil 引擎在《刺客信条：影》中的大规模升级。核心主题围绕三大技术支柱展开：(1) GPU Driven Rendering Pipeline 的全面重构——包括新增的 Micro Polygon 虚拟几何管线；(2) 面向 16×16km 开放世界的全局光照解决方案——从 2TB 理论数据压缩至 9GB 的烘焙 GI，并叠加光线追踪 GI；(3) 系统化的天气与季节系统——结合流体模拟、延迟渲染雨雪和多状态实体。演讲同时详细阐述了统一光线追踪抽象层 Fusion 的设计，以及面向多平台、多模式的可扩展性架构。

---

## 🏗️ Anvil 引擎架构

### 引擎历史与品牌覆盖

Anvil 引擎诞生于 2007 年（开发始于 2004 年），最初为《刺客信条 1》设计，核心定位是**大规模系统化开放世界**（large systemic open worlds）、**远距离渲染**和**系统化玩法**。除刺客信条系列外，Anvil 还支撑了《幽灵行动》、《荣耀战魂》和《彩虹六号：围攻》等不同类型的 3A 作品。

### Fork 问题与统一化决策

随着引擎发展，育碧内部产生了大量引擎分叉（fork）：

| 引擎代号 | 主要用途 |
|---------|---------|
| Cimitar（Anvil 原始版本） | 刺客信条系列 |
| Blacksmiths | 彩虹六号：围攻 |
| Silex | 幽灵行动开放世界 |

育碧内部同时存在 Anvil、Snowdrop、Dunia、Voyager、UB Art 等多个引擎，且每个引擎还有多个 fork（仅 Cimitar 和 Dunia 各有 3 个以上 fork）。

**统一化的核心观察**：
1. **开发努力被严重稀释**——同一功能在多个引擎中重复开发
2. **大型系统生命周期极长**——从开发到可部署需要数年，在代码库中可存活十年，重写机会极少
3. **竞争力下降**——多引擎并行维护使育碧在技术迭代上失去优势

**最终决策**：将 Anvil 作为**跨品牌、跨类型、跨游戏**的共享引擎，采用 **mono repo**（单一代码库）模式，全球技术团队与制作团队协同开发。

### 从 Valhalla 到 Shadows 的技术跃迁

Valhalla 作为跨世代（cross-gen）标题，可扩展性非常有限：质量模式和性能模式的帧时间预算几乎相同。在 Xbox Series X 上以 4K 原生分辨率运行时，GPU 在 30FPS 下仅消耗略高于 20ms（16.67ms 预算），利用率不足。

Shadows 定位为**真正的次世代独占**（gen five only），目标是彻底拉开不同模式的性能差距：

| 平台 | 性能模式 | 平衡模式 | 质量模式 |
|------|---------|---------|---------|
| PS5 | 1440p | 1620p | 2160p |
| Xbox Series X | 1440p | 1620p | 2160p |
| PS5 Pro | — | — | 2160p (RT 开启) |

**关键设计决策**：光线追踪仅在质量模式中启用，给予玩家明确的选择权。这种设计考虑到光线追踪的成本高昂，且不想为了在不支持 RT 的平台上以 60FPS 出货而牺牲其他视觉品质。

---

## 🌍 大世界渲染管线

### 流式加载系统

Anvil 从底层设计就面向远距离渲染。世界被划分为**单元格（cells）**，引擎原生支持**多人协作编辑**。数据以 **Streaming Grid Layers**（流式网格层）组织，网格定义完全**数据驱动（data-driven）**。

#### LOD Grid 层次结构

Shadows 中的网格组织如下（其他 Anvil 游戏可完全不同配置）：

```
┌─────────────────────────────────────────────┐
│           Terrain Vista (>8km)              │  烘焙地形的远景细节
├─────────────────────────────────────────────┤
│        Point Cloud Imposters (≤8km)         │  大规模 quad imposter
├─────────────────────────────────────────────┤
│   Fake Entity Grid: Decimated/Aggregated    │  简化聚合体 (≤4km)
├─────────────────────────────────────────────┤
│  Long Range Grid    │  Main Grid  │ Short  │  实际 LOD 网格
│  (大物体/POI)       │  (大部分资产)│ Range  │
│                     │             │(近处小物)│
└─────────────────────────────────────────────┘
```

- **Short Range Grid**：容纳在近距离就会消失的小型道具，对渲染和玩法影响不大
- **Main Grid**：大部分资产所在的层级
- **Long Range Grid**：大尺度物体和兴趣点（Point of Interest），保持更远的可视距离
- **Fake Entity Grid**：包含降面（decimated）和聚合（aggregated）资产，可达 4km
- **Point Cloud**：大规模 quad imposter 渲染系统，覆盖至 8km
- **Terrain Vista**：超过 8km 仅显示烘焙了丰富细节的地形远景

**设计约束**：流式网格在实体级别指定，绘制距离由 **LOD Selectors** 驱动，且受流式网格约束——两者必须匹配，否则会出现物体异常消失的问题。

### Point Cloud Imposter 系统

Point Cloud 渲染器是一个**大规模 quad renderer**，专门用于渲染远距离树木（最远 8km）。它本质上是一个极快速的**四元误差度量（Quadrant Error Metrics）**系统。

在启用/禁用的对比中可以看到，Point Cloud 覆盖了画面中所有方向的大量树木区域。系统带有淡入淡出（fading）机制，确保与近处 LOD 的无缝过渡。

### GPU Driven Pipeline 演进

#### 第一代：Batch Renderer（AC Unity, 2014）

Batch Renderer 是一个基于 **DirectX 11 class API** 的 GPU 驱动管线，核心目标：减少昂贵的 DX11 draw call 开销。

特点与局限：
- 依赖 **MultiDraw Indirect** 进行 cluster 级别的剔除和渲染
- 引入了实例（instance）、cluster 和三角面级别的精细剔除
- **仍有大量逻辑在 CPU 上执行**
- **不支持 bindless textures**——限制了 batching 的程度，只能做"裸材质"级别的 batching
- **不支持异步计算（Async Compute）**——导致材质渲染被延迟

#### 第二代：GPU Instance Renderer / GPUIR（Shadows）

GPUIR 面向 **DirectX 12 class API** 设计，完全在 GPU 上执行批处理。

关键改进：
| 特性 | Batch Renderer (AC Unity) | GPUIR (Shadows) |
|------|--------------------------|-----------------|
| API | DX11 class | DX12 class |
| 批处理位置 | 部分 CPU | **完全 GPU** |
| 批处理粒度 | Per shader | **Per pass** |
| Bindless Textures | ❌ | ✅ |
| Async Compute | ❌ | ✅ |
| 虚拟几何 | ❌ | ✅ |
| 实例剔除规模 | — | **百万级/帧** |

#### Database 系统

Database 是 GPUIR 的核心数据基础设施——一个可在 CPU 和 GPU 之间共享的数据结构容器。

**设计理念**：
- 遵循**数据导向设计（Data-Oriented Design）**，自带引用机制
- 本质上是一个**超级结构化缓冲区（super structured buffer）**
- 通过 Anvil 内部的 **CIG**（Shader Input Groups）编译器生成访问函数和绑定代码

**Table（表）声明**：CIG 语言被扩展以声明表，同一声明可同时生成 C++ 和 HLSL 的访问代码（getter、setter 全部自动生成）。

**关系系统（Relations）**：
- **One-to-One Relation**：等价于指针
- **One-to-Many Relation**：指针 + 大小的扩展概念，可指向多个子节点

这使得引擎能够用类 C++ 的方式完整描述 mesh 关系和属性——在 CPU 和 GPU 上有各自的表实例，通过数据复制策略同步。

**数据复制模式**：

| 模式 | 适用场景 | 特点 |
|------|---------|------|
| Copy | 小型表 | 整表从 CPU 复制到 GPU 的 byte address buffer |
| Dirty Row / Dirty Pages | 中型表 | 维护脏标记（行级或页级），仅复制变更部分 |
| Dirty Row Only (CPU) | 超大表 | CPU 端仅存储脏行，flush 到 GPU 后丢弃，避免 CPU 内存饱和 |

**内存布局控制**：支持通过注解声明表是 **Structure of Arrays (SoA)** 还是 **Array of Structures (AoS)**，甚至可以混合使用。

**场景描述**：GPU 上的场景描述与 CPU 端高度相似（但不完全相同，GPU 端做了优化）。一个 Mesh 最多有 5 个 LOD，每个 LOD 有两个 LOD 距离（主视图和阴影），几何与 PSO 的关联在 sub-mesh 级别通过 **Batch Hash** 建立，确保每个 PSO 只有一个 batch（零重复）。

#### 完整的 GPU 驱动渲染管线

```
CPU                    GPU
│                       │
│ Coarse Culling        │
│ (剔除 Leaf Nodes/     │
│  实例组)              │
│                       │
│                       │ Frame Culling
│                       │ (对所有视图/通道的
│                       │  实例剔除 + LOD 选择/混合)
│                       │
│                       │ Per-Pass Culling
│                       │ (通道级视锥体 + 遮挡剔除)
│                       │ (如阴影：per cascade 逆视锥体剔除)
│                       │
│                       │ Instance Data Preparation
│                       │ (准备顶点/像素着色器所需的
│                       │  几何/材质/常量数据寻址信息)
│                       │
│                       │ Optional: Cluster Culling
│                       │ Optional: Triangle Culling
│                       │
│                       │ Execute (Indirect Draw)
```

**Leaf Node 概念**：空间上聚集的实例组。将空间上接近的实例放在同一个 Leaf Node 中，以最小化其包围盒体积，提高 CPU 粗剔除效率和 GPU 核心扩展性。

---

## 🔺 虚拟几何管线 (Micro Polygon)

### 概述

Micro Polygon 管线基于 **Brian Karis 在 GDC 2021 提出的 Nanite** 技术路线，是**离散 Cluster Mesh 管线的演进**。

核心特征：
- Mesh 由 **cluster 层次结构**构成，带有**连续 LOD**
- Cluster 根据**可见性独立流式加载**
- 三角形根据其大小选择**硬件光栅化或软件光栅化**
- 用于 **G-Buffer** 和 **阴影**的光栅化

与 GPUIR 的对比：

| 特性 | GPUIR | Micro Polygon |
|------|-------|---------------|
| 适用对象 | Alpha-tested 植被 | 静态不透明几何体 |
| Cluster 大小 | — | **128 三角形**（Nanite/AC Unity 为 64） |
| 内存占用 | 基准 | **约为 GPUIR 的 50%** |
| 简化工具 | — | Simpligon（生成 cluster 组简化） |
| 自定义着色器 | ✅ | ✅（手动 API 接入 shader graph） |
| 执行方式 | 完全 GPU | 完全 GPU |
| Cluster 划分 | — | Metis 库（与 Nanite 相同） |
| Z-Buffer 碰撞 | — | 两遍层级 Z-Buffer 系统 |
| 材质渲染 | — | 从 VZT buffer 渲染到 G-Buffer |

### 与其他技术的异同

与 Nanite 类似之处：
- 使用 Metis 库进行 cluster 分区
- 根据三角面大小选择 mesh shader 或软件光栅化
- 两遍层级 Z-Buffer 碰撞系统

Shadows 的独特选择：
- 仍依赖 **Simpligon** 进行 mesh 简化（生成 cluster group 的 LOD chain）
- 支持用户自定义着色器代码，通过手动 API 接入 shader graph
- 管线完全在 GPU 上执行
- Cluster 大小为 **128 三角形**（而非常见的 64）

### 软件光栅化优化

**XY 坐标交换优化**：软件光栅化使用扫描线（scanline）算法光栅化三角形，传统实现通常沿水平方向扫描。但对于**纵向三角形**（vertical triangles），算法效率会下降——分支不相干导致 workload 不均衡。

解决方案：根据三角形的朝向（水平/纵向），**交换 X 和 Y 坐标**。这样即使是纵向三角形也能以"水平"方式高效扫描，从而**改善分支相干性和负载均衡**。

### 实际场景数据

#### 京都城市场景

| 管线 | 实例数 | 渲染三角面数 | 备注 |
|------|--------|-------------|------|
| Micro Polygon | ~28,000 | ~34,000,000 | 90% 三角面为软件光栅化 |
| GPUIR | ~9,000 | ~2,000,000 | 主要为树木（跨所有通道） |

90% 的三角面通过软件光栅化——但这个比例取决于具体几何体、LOD 细节级别和渲染分辨率（分辨率影响每个多边形的像素覆盖率）。

#### 森林场景

| 管线 | 实例数（剔除前） | 剔除后实例数 | 渲染三角面数 |
|------|----------------|-------------|-------------|
| Micro Polygon | — | 较少（建筑/岩石） | — |
| GPUIR | ~3,000,000 | ~30,000 | ~7,000,000 |

剔除前场景包含 **15 亿三角面**，最终仅渲染 **700 万**。这解释了为什么 Micro Polygon 暂不支持 alpha-test——引擎对不透明几何体没有内存压力，可以专注于把不透明管线做好。

---

## 💡 全局光照系统

全局光照是刺客信条系列的"一等公民"。Shadows 的 GI 管线是一个**频谱**（spectrum），从完全烘焙的漫反射/镜面反射到光线追踪的漫反射/镜面反射。

```
完全烘焙 ←————————————————→ 光线追踪
Diffuse  ◆━━━━━━━━━━━━━━━━━◆
Specular ◆━━━━━━━━━━━━━━━━━◆
         烘焙GI    RTGI
```

### 烘焙 GI 演进

#### 第一代：AC Unity 的体积烘焙 GI

- 在 **GPU** 上使用 **Ray Bundles** 进行烘焙
- 世界被均匀的 **irradiance volume** 覆盖
- Mipmap 作为远距离的 LOD 机制
- **不支持动态时间**：通过 4 个固定的静态环境光模拟
- 参考：Josh Obson 在 GDC 上关于《战神》间接光照的演讲（基于 AC Unity 的工作）

#### 演进：Syndicate 的关键帧混合

Syndicate 加入了动态时间支持，方法是在**两个固定 GI 关键帧之间混合**。

#### 数据规模危机

如果简单地从 Unity/Syndicate 的方案外推到 16×16km 的开放世界：

| 维度 | 数据量估算 |
|------|----------|
| 16×16km，AC Unity 密度（50cm 体素） | ~500 GB |
| + 4 个季节 | ~2 TB |
| 烘焙时间 | > 600 天 |

即使拥有无限计算资源（Google Cloud、Azure 等），**Blu-ray 光盘的空间**也是硬约束。

#### 解决方案：稀疏化 + 压缩

**Step 1：从 GPU 迁移到 CPU 烘焙**

原因：
- GPU 烘焙需要大量 VRAM，当时内存悬崖（memory cliff）是真实问题（尤其在 Origins 时代）
- CPU 烘焙减少 build farm 上驱动和 GPU 的差异导致的变化
- 输入确定性——精确知道要烘焙什么、不烘焙什么
- 更快的任务派发（更少工作在本地机器执行）

**Step 2：NCT Map + 稀疏探针放置**

- **NCT Map**：由技术美术绘制 GI 分辨率密度图，确保细节集中在重要的地方
- AC Unity 在全世界均匀分布 50cm 体素；Shadows 仅在蓝色区域（城市核心区）使用此分辨率，但这些区域仍大于前代游戏的总面积
- 探针以**稀疏八叉树（sparse octree）** 分布——节点仅在有 surfel 的位置才细分
- 丢弃位于几何体内部的探针
- 平均仅需**约 10%** 的均匀分布探针量

**Step 3：时间与季节编码**

**时间支持**：存储 **11 个关键帧**（1 个用于环境光 + 10 个用于不同太阳位置），完全数据驱动（可自定义数量）。存储格式为 **YCoCG**：
- **Y（亮度）**：有方向的，存储为**球谐函数（Spherical Harmonics）**
- **Co/Cg（色度）**：标量值

**季节支持**——两个"明知有误但实际可行"的假设：

1. **春、夏两季的间接光照相同**——实际上并非精确（albedo 几乎一致），季节变化更多来自资产切换、雾效和天气
2. **亮度（Y）在所有季节保持一致，仅存储一次**；色度（Co/Cg）按季节独立存储

这些假设在实践中效果良好：
- 可能被打破的情况（几何体跨季节变化或不存在）可通过标记跳过烘焙来解决
- 受季节影响大的植被可由技术美术排除

**Step 4：运行时加载**

- 稀疏体素插入**级联 3D 体积（cascaded 3D volumes）**
- 级联为均匀网格，在运行时采样
- 流程：稀疏数据 → 解压缩（使之非稀疏化）→ 插入级联 → 跨级联混合 → GI block LOD 混合（block 加载时原位混合，避免 popping）

对比：
- **AC Unity**：离散 GI grid（类似距离剔除的 LOD 流式加载）
- **Shadows**：稀疏 GI blocks 插入级联 GI volumes，创建更连续的 LOD 过渡

**最终成果**：从理论上 2TB 压缩至 **约 9GB**。

### Probe 系统

Shadows 使用基于 **DDGI**（Dynamic Diffuse Global Illumination， McGuire 提出）的级联探针系统：

- **5 个级联**，共 **10,000 个探针**
- 每帧更新约 **1,000 个探针**
- 探针被卷积为**辐照度缓存（irradiance cache）**，兼作 fallback

**室内体积（Indoor Volumes）**：
- 由平面列表构成
- **共享数据**——GI、光线追踪、延迟天气、音效等系统共用同一套室内体积定义
- 用于探针分类：采样时找到 8 个周围探针，仅对**相同分类**的探针进行加权采样

**信号稳定化策略**：追踪较短的光线，fallback 到时间上稳定的辐照度缓存。这是在像素级精确度与信号稳定性之间的权衡。

**半透明表面处理**：
- 在半透明表面简单翻转光线
- 对于 "障子门"（soji door，日本和纸门）等部分半透明表面：在表面两侧各采样探针，根据透射因子混合

### Specular 反射层级

镜面反射采用三级降级策略：

```
优先级：高 → 低
─────────────────────────────────────────
1. SSR (Screen Space Reflections)
      ↓ 未命中时
2. Baked Q Maps (GB4Q Maps)
      → 提供反照率、法线、深度
      → 16,000 个 Q Map（不含变体）
      → 动态烘焙，运行时释放
      → 阴影烘焙：8 个时间关键帧存入 8-bit 纹理（每 bit 一个关键帧）
      → 运行时取最近两个关键帧混合
      → 季节变体：每个 Q Map 3 个变体（春/夏合一 + 秋 + 冬）
      ↓ 未覆盖区域
3. Dynamic Q Map（玩家位置的区域性 fallback）
      → 渲染 Fake Entity（简化聚合 LOD）
      → 使用低分辨率阴影
      → 设计目标：极其廉价的渲染
```

**Q Map 的时间关键帧技巧**：将 8 个阴影关键帧编码到一张 8-bit 纹理中（每 bit 代表一个关键帧），运行时选取最接近的两个关键帧进行混合。由于 Q Map 分辨率本身不高，即使没有过滤也看不出来。

---

## 🌦️ 天气与季节系统

### Atmos 流体模拟

Atmos 是一个**大气流体模拟系统**，本身值得一个完整的演讲。它模拟和传播以下大气因子：

- 水汽（vapor）
- 温度（temperature）
- 湿度（humidity）
- 风场碰撞（wind collision，使用玩家周围的低分辨率体素数据）

模拟量被传递给多个下游系统：
- **体积云**：驱动云的形成和消散
- **风与雨**：影响粒子行为和方向

Atmos 中的模拟量随海拔高度变化，提供不同高度层的大气特性。

### Ambience Graph（氛围图系统）

取代了前代基于**曲线管理器**（curve-based manager）的简单方案：
- 旧系统：主要用于驱动时间光照和后期效果，逻辑有限
- 新系统：基于 Anvil 的 **non-graph 系统**，完全**数据驱动**
- 由技术美术控制——Ambience Graph 消费来自引擎和 Atmos 的输入，驱动整个天气和季节系统栈的逻辑

### 延迟雨雪 (Deferred Rain/Snow)

基于 Sebastian Lagarde 的 **Darkening Albedo** 方法：

**延迟雨**：根据湿度级别暗化反照率并降低粗糙度。湿润度遮罩存储在 G-Buffer 中（单 float），角色等没有动态角色层系统的实体被排除（显示为红色）。

效果序列：干燥场景 → 潮湿场景 → 水坑出现。

**积雪系统**包含三个层次：

1. **Deep Snow（深雪）**：数据驱动的"印章器（stomper）"，可压印胶囊体和纹理，用于地形或任何贴图的可视变形
2. **Dynamic Snow Accumulation（动态积雪）**：类似延迟雨，修改材质输入
   - 反照率向雪的白色混合
   - 粗糙度根据当前积雪量调整
3. **冷/暖区域**：由技术美术绘制的**静态动态雪遮罩（static dynamic snow mask）** 驱动

**动态积雪的完整生命周期**（由 Ambience Graph 驱动）：

```
暴风雪 → 动态积雪增加 → 暴风雪消散 → 积雪缓慢减少
→ 转变为湿润/水坑 → 最终融化干燥
```

**室内排除**：
- 复用室内体积（Indoor Volumes）数据
- 在室内深度图中渲染——用于确定天气碰撞因子
- 雨粒子：在带雨方向的深度图中渲染周围环境 → 用于遮挡雨粒子（雨可以穿透窗户进入室内）

### 多状态实体 (Multi-state Entities)

Multi-state Entities 允许在实体级别进行**数据驱动的游戏特定逻辑**，主要用于季节系统。

实现方式——以 Q Map 为例：
- 存储每个 Q Map 的 3 个季节变体
- 春/夏视为同一变体
- 16,000 个 Q Map 中大多数都有变体（部分室内空间可以跳过冬季变体）

---

## ☀️ 光线追踪

### 统一 RT 抽象层 (Fusion)

光线追踪是 Shadows 的一大技术投入，深受 Snowdrop 团队工作的启发。

**设计动机**：
- 初期缺乏数据支撑——不确定 inline ray tracing 是否可行
- 需要保持选项开放，能在不同方案间快速切换

**Fusion** 是育碧跨引擎共享的**硬件抽象层（HAL）**，通过 **insourcing 模型**开发（在内部 GitLab 上，多个项目可贡献和改进）。

**抽象的完整范围**：

| 维度 | 覆盖范围 |
|------|---------|
| 平台 API | 统一不同平台的 RT API |
| 硬件 vs 软件 | 硬件 RT / 软件 RT（compute shader）/ CPU RT |
| Inline vs Non-inline | 内联 vs 非内联光线追踪 |

**Callback 接口——inline/non-inline 的统一抽象**：

```
┌─────────────────────────────────────────┐
│            RT Loop (Fusion)             │
│                                         │
│  for each ray:                          │
│    ┌─────────────────────────────────┐  │
│    │     Callback Interface          │  │
│    │  ┌────────┐ ┌────────┐ ┌─────┐ │  │
│    │  │  Hit   │ │Any Hit │ │Miss │ │  │
│    │  └────────┘ └────────┘ └─────┘ │  │
│    └─────────────────────────────────┘  │
└─────────────────────────────────────────┘

Inline RT:   在 ray loop 中直接调用对应 callback
Non-inline:  在 hit/miss shader 中手动调用同一 callback
```

切换成本：
- **软件 ↔ 硬件**：仅需更改一个 **enum** 值
- **Inline ↔ Non-inline**：主要差异在于是否需要设置 shader table
- 可以轻松在两种模式间切换来验证假设、测试新功能

**为什么 inline 和 non-inline 之间的切换如此重要？**
- 验证设计假设——确认 inline 方案的实际表现
- 测试新功能——某些功能只在其中一种模式下容易实现
- 找到最适合游戏的最佳方案

### 加速结构

- 支持二进制纹理（binary textures）或平均颜色（用于低端平台的金属纹理等）
- 根据 size 或 type 对 mesh 进行分类
- 使用全局 LOD offset 决定 BVH 内容
- Micro Polygon 基于 **LOD error metric** 输出代理 mesh（proxy mesh）——演讲前一天 NVIDIA 的演讲也提到了这是当前没有 mega geometry 时的最佳方案
- 所有参数均可按资产或材质级别覆盖

**典型城市场景（Xbox Series X）**：约 2,000 个 mesh，30,000 个实例，BVH 数据总计 **320 MB**。PC 上更大（因为有更多抽象层级）。

### Alpha Test 处理

Shadows 尝试了多种 alpha test 方案：

| 方案 | 结果 |
|------|------|
| Blue noise + stochastic alpha | — |
| Bindless alpha texture（几乎零开销，GPUIR 基础设施已就绪） | — |
| Any-hit（精确但昂贵） | 在植被密集场景中，大量重叠导致 any-hit 开销极大——**deal-breaker** |
| 限制 hit 数量 | 过多遮挡（over-occlusion）|
| 仅对植被补偿 | 效果不满意——"more of a hack" |

**最终方案：三角形缩放（Triangle Scaling）**

根据平均不透明度（average opacity）缩放三角形大小：

- 比任何 any-hit 方案 **快约 30%**
- 漫反射 GI 结果与参考非常接近
- 实际效果好的原因：**Closest Hit 通过屏幕空间解析，精度极高**——结合远处的缩放三角形，得到"两全其美"的效果
- 作为额外好处，对**镜面反射光线追踪**也是很好的近似

### RTGI 管线

RTGI 分为两个主要部分：

1. **Per-Pixel Ray Tracing**（逐像素光线追踪）
   - 屏幕空间光线（Screen Space Rays）——零近似，命中时直接使用
   - 世界空间光线（World Space Rays）——屏幕空间未命中时的 fallback
   - 大部分平台在**四分之一分辨率**下追踪屏幕空间光线以提升性能

2. **DDGI Probe Cascade**（探针级联）
   - 5 个级联，共 10,000 个探针
   - 每帧更新约 1,000 个

**完整管线流程**：

```
① Screen Space Rays (1/4 res)
   ├─ 命中 → 存入 Hit Buffer
   └─ 未命中 → 携带 T 值传递到下一步

② World Space Rays (Hardware RT / BVH)
   ├─ 从 T 值继续步进
   └─ 命中 → 存入 Hit Buffer
   └─ 未命中 → Miss Event

③ Hit Lighting Pass
   → 对 Hit Buffer 中的命中重光照
   → 未命中：采样 Probe Cache（DDGI）

④ 获取后续反弹
   → 从 DDGI cascade 采样

⑤ 降噪

⑥ 可选：追加 RTAO（艺术驱动）+ Specular RT

⑦ 最终图像
```

**为什么 RTAO 在最后添加？**

理论上 AO 应该是光照积分的一部分，但放在最后有几个合理原因：

1. **频率不匹配**——per-pixel rays 与 ray-traced probes 是两个不同频率的系统，探针可能衰减 AO 细节
2. **四分之一分辨率追踪**——屏幕空间光线在 1/4 分辨率下运行，高频 AO 细节丢失
3. **光栅世界与 RT 世界的差异**——RT 世界中没有草地（出于性能考虑），导致大型物体（房屋、树木）与地面的接触不够自然

解决方案：
- 追加 RTAO 项使大物体"接地"
- 保留一个微妙的 SSAO 项来补偿 1/4 分辨率追踪丢失的厘米级细节（树冠、树叶等）

**材质处理**：
- Inline RT 使用 **Uber Shader** 方案
- 材质存储在 **Material Table** 中，hit 时通过 geometry ID 访问
- 季节通过材质表中的**材质变体（Material Variation）**处理——如落叶场景减少遮挡、叶色查找表
- 延迟天气（deferred weather）对**镜面反射 RT 完全评估**，对**漫反射不评估**（多次评估太昂贵）——静态雪烘焙在 terrain vista 中，漫反射仍可访问
- BVH 中的所有内容均可覆盖：LOD mesh、材质属性、着色器等

### 反射光线追踪 (Specular RT)

Specular RT 是在游戏延期后、接近发布前的"最后一刻"加入的。目标平台：PS5 Pro 和 PC。

**设计**：遵循与 RTGI Diffuse 大致相同的架构，大部分基础设施可复用。

**主要挑战**：
- **BVH 质量问题**——低频漫反射 GI 中不明显的问题在镜面反射中变得突出
- **降噪**——镜面信号的降噪比漫反射困难得多

Specular RT 在 PS5 Pro 上以**半分辨率（半宽、全高）**出货。原因：
- 半分辨率视觉效果显著优于更低分辨率
- PS5 Pro 上漫反射 RT 成本大幅降低，为镜面反射腾出预算
- 成功维持了目标分辨率

### 本地光源

本地光源插入**全向 Cluster Lighting 结构（Omnidirectional Cluster Lighting）**：

- 基于常规 Cluster Lighting 结构，但有独特设计
- Cluster volume 映射到**相机周围的均匀网格**
- **两级层级**：High Clusters（高层簇）→ Clusters（簇）
  - 每个 High Cluster 包含 16×16×16 = 4,096 个 Cluster
  - 总计 **260K clusters**
- 先进行**粗粒度 High Cluster 剔除**，再进行**细粒度 Cluster 剔除**

### 降噪 (Denoising)

引擎中有三款降噪器：

| 降噪器 | 类型 | 来源 | Shadows 中是否使用 |
|--------|------|------|-------------------|
| Atrous | SVGF 家族 | 内部开发 | ❌ |
| NRD ReBLUR | 循环模糊 | NVIDIA | ❌ |
| **MSD** | **循环模糊** | **Snowdrop** | **✅** |

**MSD（Snowdrop 降噪器）的特征**：
- 基于循环模糊（recurrent blur）方法，与 ReBLUR 类似
- 支持材质遮罩——可对角色、植被、动态物体分别处理
- 支持**空间谐波降噪（Spatial Harmonics Denoising）**——信号存储在 **YCoCG** 格式中，Y 分量为空间谐波（spatial harmonics）

空间谐波降噪的效果非常显著：原始 RTGI 输出有明显噪点，经过 MSD 降噪后图像干净平滑，开启空间谐波后进一步提升了细节保留能力。

### 特殊场景：Hideout（隐匿之所）

Hideout 是一个"类 SimCity 的自由动态建造"系统。由于布局完全由玩家决定，无法烘焙任何 GI——**必须依赖光线追踪**，否则完全没有全局光照。

---

## 📊 性能与可扩展性

### Platform Manager

由于性能模式和质量模式使用完全不同的 GI 系统，两种模式的**帧结构（frame layout）差异极大**。Platform Manager 是 Anvil 中的一等公民，用于管理这种复杂性。

**核心功能**：

| 层级 | 触发条件 | 行为 |
|------|---------|------|
| **Profile** | 玩家选择（性能/平衡/质量） | 需要重新加载世界 |
| **Profile Boot** | 启动时 | 需要重启可执行文件 |
| **Context** | 玩法触发（进入菜单/照片模式/回到游戏） | 运行时切换，无需重启或重载 |
| **Modifier** | 数据触发（画笔/3D 体积） | 解决局部性能问题或特殊需求 |

**特性**：
- 主要用于引擎和图形系统（但不仅限于）
- UI 自动生成，所有设置**实时可编辑（live-editable）**
- 可在编辑器中模拟主机设置（虽然有限，但可提供参考）
- 非常适合与硬件厂商协作迭代

### RT 性能开销

| 组件 | PS5 | PS5 Pro | Xbox Series X |
|------|-----|---------|---------------|
| RT Probes（DDGI） | ~1.0 ms | <1.0 ms | ~1.0 ms |
| Per-Pixel RT（Diffuse） | ~4-5 ms | 更低 | ~4-5 ms |
| **Diffuse RT 总计** | **~5-6 ms** | — | **~5-6 ms** |
| Specular RT | — | 半分辨率（半宽全高） | — |

Diffuse RT 开销分布在**图形队列（Graphics Queue）和异步队列（Async Queue）**之间。

### 内存追踪

两套内部工具：
1. **高层视图**：追踪资源生命周期，定位帧中内存峰值
2. **底层工具**：检查分配器，调试内存浪费或不良的内存别名模式

### 遥测系统

大量性能计数器存入数据库，用于追踪性能回归：

| 追踪内容 | 示例 |
|---------|------|
| 动态分辨率因子 | 世界中每个可能位置的值 |
| 平台间对比 | PS5 各 build 间的 GPU 性能回归 |

### 帧时间调度挑战

Nicolas 在 Q&A 中坦诚了帧时间调度的核心困难：

> "主要问题是**没有 RTGI 的帧**与**有 RTGI 的帧**是两种完全不同的帧。性能模式和质量模式是两个不同的帧结构。我们试图让 GPU 调度尽可能接近，以避免每种模式中出现特定的 bug。说实话，这很头疼。除非我们把这些帧拉近，否则可能无法避免按模式定制调度。"

在 mono repo 环境下的额外挑战：同一代码库中有不同的游戏，需要更多的定制化。

引擎已开始探索**数据驱动的 frame graph 调度**，但在本作中尚未大胆推进。

---

## 🔑 关键数据汇总

### 世界与场景

| 参数 | 数值 |
|------|------|
| 世界大小 | **16 × 16 km** |
| Point Cloud 渲染距离 | **≤ 8 km** |
| Fake Entity 渲染距离 | **≤ 4 km** |
| 烘焙 GI 体素分辨率（核心区） | **50 cm** |
| 烘焙 GI 最终大小 | **~9 GB**（理论上 2TB） |
| Baked Q Map 数量 | **16,000** 个（不含变体） |
| Q Map 季节变体 | 每个最多 **3 个** |
| Q Map 阴影关键帧 | **8 个**（编码入 8-bit 纹理） |

### 几何与渲染

| 参数 | 数值 |
|------|------|
| Micro Polygon Cluster 大小 | **128 三角形** |
| Micro Polygon 内存（vs GPUIR） | **约 50%** |
| 京都：Micro Polygon 实例 | **~28,000** |
| 京都：Micro Polygon 三角面 | **~34M**（90% 软件光栅化） |
| 京都：GPUIR 实例 | **~9,000** |
| 京都：GPUIR 三角面 | **~2M** |
| 森林：GPUIR 实例（剔除前） | **~3,000,000** |
| 森林：GPUIR 实例（渲染） | **~30,000** |
| 森林：渲染三角面 | **~7,000,000** |
| 森林：场景总三角面（剔除前） | **~1.5B** |

### 光线追踪

| 参数 | 数值 |
|------|------|
| DDGI 级联数 | **5** |
| DDGI 探针总数 | **10,000** |
| 每帧更新探针 | **~1,000** |
| GI 时间关键帧 | **11**（1 环境光 + 10 太阳位置） |
| Alpha Test 三角形缩放 vs any-hit | **快 ~30%** |
| Screen Space RT 分辨率 | **1/4 分辨率** |
| Specular RT 分辨率（PS5 Pro） | **半宽 × 全高** |
| 城市 BVH：mesh 数 | **~2,000** |
| 城市 BVH：实例数 | **~30,000** |
| 城市 BVH：数据大小 | **~320 MB** |

### 性能

| 参数 | 数值 |
|------|------|
| RT Probes（DDGI）| **~1 ms**（主机） |
| Per-Pixel RT（Diffuse） | **~4-5 ms**（主机） |
| Diffuse RT 总计 | **~5-6 ms**（主机） |
| Omnidirectional Cluster 数 | **260,000** |
| High Cluster 粒度 | **16³ = 4,096** Clusters/High Cluster |

### 平台分辨率

| 平台 | 性能模式 | 平衡模式 | 质量模式 |
|------|---------|---------|---------|
| PS5 | 1440p | 1620p | 2160p |
| Xbox Series X | 1440p | 1620p | 2160p |
| PS5 Pro | — | — | 2160p |

---

## 💡 核心收获与启发

### 1. "明知有误但实用"的近似胜过完美

Shadows 的 GI 季节系统建立在两个"技术上错误"的假设上（春夏合一、亮度共享）。在实际项目中，**足够好的近似 + 艺术可控性**远比理论正确性重要。关键是要有逃生舱——标记异常几何体、跳过烘焙。

### 2. 统一抽象层是技术探索的倍增器

Fusion 的 inline/non-inline、hardware/software 统一抽象不是"过度工程"——它允许团队在项目中期快速验证设计假设。当不确定技术路线时，**低成本切换能力**比任何单一方案都更有价值。这比花时间选错方案后回滚便宜得多。

### 3. 三角形缩放是 alpha test 在 RT 中的"务实之选"

在屏幕空间解析 closest hit + 远处用缩放三角形近似透射，是工程权衡的典范。与其追求理论上正确的 any-hit（在植被场景中因大量重叠而性能爆炸），不如利用屏幕空间已有信息的优势。**最好的 RT 方案不一定是理论上最精确的——而是利用管线其他部分的信息做出整体最优的选择。**

### 4. 大世界数据压缩的核心是稀疏化，不是有损压缩

从 2TB 到 9GB 的压缩不是靠降低质量，而是靠**只存储需要的数据**。NCT Map 精确控制密度 + 稀疏八叉树 + 丢弃几何体内探针 = 平均仅需 10% 的均匀分布探针。**在数据密集型系统中，"不存储"永远比"压缩存储"更高效。**

### 5. 性能模式与质量模式使用不同的 GI 系统是一种勇气

让两种模式走完全不同的渲染路径（烘焙 vs RTGI）增加了巨大的工程复杂度——帧结构、调度、bug 面都是两倍的。但这种选择给了玩家真正的选择权，而不是"同样的渲染，不同分辨率"。**如果你的 RT 方案足够成熟，不要为了兼容性让非 RT 用户承担妥协。**

---

## 📚 延伸阅读

演讲中直接提到或强烈参考的工作：

| 工作 | 关联 |
|------|------|
| **Josh Obson — Indirect Lighting in God of War (GDC)** | AC Unity 的 Ray Bundle 烘焙 GI 管线的详细描述 |
| **Brian Karis — Nanite (GDC 2021)** | Micro Polygon 管线的理论基础 |
| **McGuire et al. — DDGI (Dynamic Diffuse Global Illumination)** | 探针系统的理论基础 |
| **Sebastian Lagarde — Darkening Albedo (Deferred Rain)** | 延迟雨效果的方法论 |
| **Snowdrop Engine (Ubisoft) — MSD Denoiser, Software RT, RT Architecture** | 降噪器、软件光线追踪和 RT 抽象层的直接灵感来源 |
| **NVIDIA NRD — ReBLUR** | 被评估但最终未采用的降噪方案 |
| **G3D (Gamasutra / GPU Zen 3) — GPU-Driven Rendering Articles** | GPUIR 管线的详细技术文档 |
| **Metis Library** | Cluster 分区算法（与 Nanite 相同选择） |
| **Simpligon** | Shadows 使用的 mesh 简化工具 |
| **NVIDIA GDC 2025 — Virtual Geometry Proxy Mesh** | Micro Polygon 的 LOD error metric / proxy mesh 方法 |