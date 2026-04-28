---
title: "GDC2025: Assassin's Creed Shadows 渲染技术"
date: 2026-04-08
categories:
  - 视频
tags:
  - GDC
  - YouTube
description: "GDC 2025 Assassin's Creed Shadows 渲染技术分析"
---

> **元信息**：GDC 2025 | 时长 ~62 min | 演讲者：Nicolas Lopez (Ubisoft)
> **YouTube**: https://www.youtube.com/watch?v=yj5pYktC3X8
> **生成日期**：2026-04-08
> **难度**：⭐⭐⭐⭐（需要基础的图形学概念，但每个术语都做了通俗解释）

## 📋 内容概览

本演讲全面介绍了育碧 Anvil 引擎在《刺客信条：影》中的渲染技术。游戏是一个 16×16 公里的开放世界，同时具备体积化全局光照（GI）、光线追踪、虚拟几何管线、系统性天气与四季系统。演讲覆盖五个核心领域：（1）GPU 驱动的几何渲染管线，包括 Micro Polygon 虚拟几何和 Database 数据共享框架；（2）全局光照系统——从烘焙稀疏体积 GI 到光线追踪 GI 的完整光谱；（3）光线追踪架构——软硬件统一抽象、Alpha-test 三角形缩放、Uber shader 材质系统；（4）天气与四季的延迟渲染方案和系统化氛围图驱动逻辑；（5）跨平台可扩展性与性能管理模式。核心主线是：如何在 16×16 公里、四季变化、60fps 开放世界中将 GI 从理论上 2TB 压缩到实际 9GB。

---

## 🏗️ 一、Anvil 引擎：从分支到统一

### 1.1 引擎演进

Anvil 诞生于 2004 年开发期（2007 年随《刺客信条 1》问世），是为大尺度系统化开放世界和远距渲染设计的引擎。除刺客信条系列外，还驱动了《幽灵行动》、《荣耀战魂》、《彩虹六号：围攻》。

育碧曾经历严重的引擎碎片化问题——Anvil、Snowdrop、Dunia、Voyager、UB Art 等多个引擎并存，每个引擎又有多个分支（如 Cimitar → Blacksmith、Silex 三个分支）。最终选择统一到 Anvil 作为共享引擎，采用 **mono repo（单一代码库）** 模式，全球团队共同维护。

**为什么统一？** 大型系统开发耗时多年，生命周期可达十年。在多个引擎上重复开发相同功能直接削弱了竞争力。

### 1.2 与 Valhalla 的代际对比

| 维度 | AC Valhalla | AC Shadows |
|------|------------|------------|
| 平台 | 跨代（PS4/Xbox One + PS5/XSX） | 纯次世代（PS5/XSX） |
| 可扩展性 | 极有限，质量/性能模式帧结构几乎相同 | 三个模式：性能/平衡/质量 |
| 缩放驱动 | 主要靠渲染分辨率 | 渲染分辨率 + GI 系统切换 |
| GPU 利用率（XSX 4K@30fps） | ~20ms（大量余量浪费） | 更充分地压榨 GPU |
| RT（光线追踪） | 无 | 质量模式启用 |

**Valhalla 在 XSX 上 4K@30fps 只用了约 20ms GPU 时间**，说明 GPU 资源严重浪费。**这直接促使 Shadows 团队决定大幅提升可扩展性**——同时提供 30fps（质量，含 RT）和 60fps（性能，baked GI）两种完全不同的帧结构。

---

## 🔺 二、GPU 驱动几何管线

### 2.1 超远距离渲染分层架构

Anvil 的世界被划分为 **cells（格子）**，支持多用户协同编辑。数据以 **streaming grid layers（流式网格层）** 组织，定义完全数据驱动。

```
┌──────────────────────────────────────────────────────┐
│  >8 km: Terrain Vista（烘焙了远距细节的地形远景）      │
├──────────────────────────────────────────────────────┤
│  4-8 km: Point Clouds（大规模 Imposter 渲染器）        │
│           - mass quad renderer，用于远距树木           │
├──────────────────────────────────────────────────────┤
│  fake entity grid: 简化/聚合的 LOD 网格（至 4 km）    │
├──────────┬──────────────────┬────────────────────────┤
│ Short    │  Main Grid       │  Long Range Grid        │
│ Range    │  （大多数资产）   │  （大物件/兴趣点）       │
│ Grid     │                  │                          │
│（近距离）│                  │                          │
└──────────┴──────────────────┴────────────────────────┘
```

- **LOD grids** 分三种：short range（近距道具，可较早消失）、main grid（主力）、long range（大物件，远距保留）
- **draw distance** 由 **LOD selectors** 驱动，受 loading grids 约束——二者必须匹配，否则物体可能在错误距离消失
- **Point Cloud renderer**：mass quad renderer，将树木渲染为 billboard 四边形，可延伸至 8km
- **Fake entity grid**：包含 decimated（简化）和 aggregated（聚合）资产，到 4km
- **Terrain Vista**：8km 以外仅剩地形远景，大量细节已烘焙

### 2.2 GPU 驱动管线的三代演进

#### Batch Renderer（AC Unity, 2014）→ GPU Instance Renderer（AC Valhalla）→ Micro Polygon（AC Shadows）

**Batch Renderer**（第一代）：
- DirectX 11 class API 的 GPU 驱动管线
- 用 **MultiDraw Indirect** 做 cluster culling（簇剔除）
- 引入了 instance / cluster / triangle 三级精细剔除
- **局限**：仍有大量 CPU 参与；不支持 bindless textures（限制了 batching 深度）；不支持 async compute

**GPU Instance Renderer / GPUIR**（第二代）：
- DirectX 12 class API 设计
- **fully bindless**——batching 按 shader 而非 material 分组
- 设计目标：cull 数百万 instances per frame
- CPU→GPU 的数据共享通过 **Database** 实现（详见 2.3）

#### 关键术语解释

- **GPU driven pipeline（GPU 驱动管线）**——传统渲染中，CPU 告诉 GPU "画这个、画那个"，每次通信都有开销。GPU 驱动管线把"画什么"的决定权交给 GPU 自身，CPU 只准备数据，GPU 自己做剔除和排序，大幅减少 CPU-GPU 通信。
- **Draw Call（绘制调用）**——CPU 向 GPU 下达的一次"画一个物体"的指令。每个 Draw Call 有固定开销（状态切换、验证等），画 1000 个物体就要 1000 次。减少 Draw Call 是实时渲染的核心优化之一。
- **Indirect Draw（间接绘制）**——把"画多少个、从哪里取数据"的信息写在 GPU 缓冲区里，让 GPU 自己读。CPU 只发一次指令，GPU 就能批量画出数千个物体。
- **Bindless**（无绑定纹理访问）——传统方式下，GPU shader 要用一张纹理就必须先"绑定"它，切换纹理有开销。Bindless 允许 shader 直接通过索引访问任意纹理，省去切换开销，从而让大量物体可以合并成一次绘制。
- **Async Compute（异步计算）**——GPU 有图形队列和计算队列，通常排队执行。异步计算允许两者并行工作，比如计算队列在做剔除的同时，图形队列在渲染上一帧的内容。
- **Cluster（簇）**——将一个网格的三角形按空间连续性分成小团（每团几十到几百个三角形），是 GPU 驱动管线的基本剔除单位。

### 2.3 Database：CPU-GPU 数据共享框架

**Database**（与数据库无关）是 CPU-GPU 共享数据结构的容器，采用 data-oriented design（数据导向设计）。可以理解为"超级 Structured Buffer"。

核心特性：
- 通过 **CIG（Shader Input Groups）**——育碧内部的 shader binding 编译器——声明表结构
- 自动生成 C++ getter/setter 和 HLSL 访问代码
- 支持 **relations（关系）**：one-to-one（类似指针）、one-to-many（指针+尺寸），可完整描述 mesh 层级和属性
- 支持 **三种数据复制模式**：
  1. **Copy**：全量 CPU→GPU 拷贝到 Byte Address Buffer
  2. **Dirty Row / Dirty Page**：维护脏标记，仅复制变化的行或页
  3. **GPU-only dirty rows**：CPU 仅存脏行，刷出后释放——适用于超大表，避免 CPU 内存饱和

**为什么选 Database？** 在 GPU 驱动管线中，CPU 和 GPU 需要共享完整的 scene description（场景描述）。传统方式是手动同步两份数据，容易出错且缓存不友好。Database 提供了类型安全、自动生成的统一接口，同时保持最优缓存访问模式（可声明 SoA 或 AoS）。

### 2.4 GPU Instance Renderer 管线架构

```
┌────────────┐    ┌─────────────────┐    ┌────────────────┐    ┌──────────────┐
│  CPU:      │    │  GPU: Frame     │    │  GPU: Per-Pass │    │  Render      │
│  Coarse    │───>│  Culling        │───>│  Culling       │───>│  (optional   │
│  Culling   │    │  (all views,    │    │  (frustum +    │    │   cluster +  │
│  (leaf     │    │   all passes,   │    │   occlusion,   │    │   triangle   │
│   nodes)   │    │   LOD select)   │    │   cascade)     │    │   culling)   │
└────────────┘    └─────────────────┘    └────────────────┘    └──────────────┘
```

- **CPU Coarse Culling**：对 **leaf nodes**（一组空间相近的 mesh instances）做粗剔除。空间相近的 instance 被聚到同一 leaf node 以最小化包围盒，提升 CPU 核心并行效率
- **Frame Culling**（GPU）：对通过 CPU 剔除的 instance 做全视图全通道剔除，同时执行 LOD 选择和混合
- **Per-Pass Culling**（GPU）：通道特定剔除，如 shadow cascade 的 anti-frustum culling；同时准备 vertex/pixel shader 访问统一缓冲区所需的 form（描述符）
- **Render**：支持可选的 cluster + triangle 级剔除
- 场景用 **octree（八叉树）** 组织

**关键数据组织**：
- 每个 **mesh** 最多 5 个 LOD
- 每个 LOD 有**两个独立距离**：main view + shadow（允许阴影 LOD 与主视图不同）
- geometry 与 **PSO（Pipeline State Object，管线状态对象——定义一次绘制调用的完整渲染配置）** 的关联在 **sub-mesh 级别**通过 **batch hash** 实现
- 保证每个 PSO 只有一个 batch，**零重复**

### 2.5 Micro Polygon 虚拟几何管线

基于 Brian Karis 的 Nanite（2021），是离散 cluster mesh 管线的进化版。

**核心概念**：
- Mesh 由 cluster 层级组成，具有 **continuous LOD（连续细节层次）**
- Clusters 按可见性单独流式加载
- 三角形根据尺寸选择 **hardware rasterization（硬件光栅化）** 或 **software rasterization（软件光栅化）**
- 用于 G-buffer 和 shadow rasterization
- 几何内存约为 GPUIR 的 **一半**

**关键术语解释**：
- **Software Rasterization（软件光栅化）**——正常情况下 GPU 有专门的硬件电路把三角形变成屏幕上的像素。软件光栅化是放弃硬件，用 compute shader 通用代码手动做同样的事。听起来更慢，但对极小的三角形（在屏幕上只占几个像素甚至亚像素）反而更快，因为省去了硬件调度的固定开销。
- **Hierarchical Z-buffer（层次深度缓冲）**——类似 mipmaps 的深度缓冲：先检查低分辨率深度图判断某区域是否被遮挡，被遮挡就跳过，没遮挡再检查高分辨率。两遍扫描实现高效剔除。
- **VZT Buffer（Virtual Z-Texture Buffer）**——一种压缩格式，存储虚拟几何管线的深度和三角形 ID，用于后续材质渲染阶段从同一个 buffer 中提取不同材质通道写入 G-buffer。

**与 Nanite 的差异**：
- 网格简化仍依赖 **Simpligon**（而非 Nanite 自带的简化器）
- Cluster 大小为 **128 triangles**（Nanite/旧管线用 64）
- 使用相同 **Metis** 库做 cluster partitioning
- 支持用户自定义 shader（manual API）
- 全 bindless
- **软件光栅化优化**：根据三角形朝向（水平/垂直）交换 X/Y 坐标，改善 scan-line 算法的 branch coherency（分支一致性）和负载均衡

**分工**：
| 管线 | 用途 | 当前限制 |
|------|------|---------|
| Micro Polygon | 城市/岩石等静态不透明几何 | 不支持 alpha-test |
| GPU Instance Renderer | 大规模 alpha-tested 植被 | — |

**为什么暂时不支持 alpha-test？** 这是有意为之的优先级选择。森林场景有 15 亿三角形，剔除到仅渲染 700 万——200:1 的剔除比意味着引擎对不透明几何毫无内存压力。团队选择先完善不透明管线，alpha-test 是未来工作。

### 2.6 场景统计数据

| 场景 | 管线 | Instances | 渲染三角形 | 软件光栅化占比 |
|------|------|-----------|-----------|-------------|
| 京都（城市） | Micro Polygon | ~28,000 | 3400 万 | ~90% |
| 京都（城市） | GPU Instance Renderer | ~9,000 | ~200 万 | — |
| 森林 | Micro Polygon | 少量（建筑/岩石） | — | — |
| 森林 | GPU Instance Renderer | 300 万→3 万（剔除后） | 15 亿→700 万 | — |

**森林场景 200:1 的剔除比**（15 亿→700 万）意味着即使不使用虚拟几何，传统管线也能轻松应对植被密度。**这进一步解释了为什么 Micro Polygon 暂不需要支持 alpha-test**——GPU Instance Renderer 已经足够高效。

---

## 🌍 三、全局光照（GI）系统

### 3.1 GI 光谱：从完全烘焙到完全光线追踪

```
Baked Diffuse ──────→ RT Diffuse
     |                      |
Baked Specular ─────→ RT Specular
     
     ↕ 切换由 Performance / Quality 模式决定
     Performance: 全烘焙
     Quality: RT Diffuse（可选 + RT Specular on PS5 Pro/PC）
```

**特别约束**：Hideout（基地，类似 SimCity 的自由建造系统）必须用 RT，因为动态建造无法烘焙 GI。

### 3.2 烘焙体积 GI 的演进

#### AC Unity → Syndicate → Origins 时代

| 特性 | AC Unity | Syndicate |
|------|----------|-----------|
| 烘焙方式 | GPU ray bundles | GPU ray bundles |
| 时间支持 | 无（4 个固定氛围） | 两个关键帧插值 |
| 探针分布 | 均匀 50cm 体素 | 均匀 |

**问题**：如果用 Unity/Syndicate 的方法推到 16×16km，**仅 GI 数据就 500GB**。加入四季后接近 **2TB**，烘烤时间超 600 天。即使有无限云计算力，蓝光盘也装不下。

#### Shadows 的三大压缩策略

**策略一：CPU Path Tracer 烘焙**

从 GPU 切换到 CPU 烘焙，原因：
- GPU 烘焙需要大量 VRAM，存在内存悬崖（memory cliff）风险
- CPU 烘焙的输入确定性高（不受 GPU 驱动差异影响）
- less variations from driver/GPU differences on build farms
- 可快速分发到云端/农场

**策略二：NCT Map + 稀疏探针分布**

- **NCT Map**（由技术美术绘制）：指定世界各区域的 GI 分辨率。只有少数城市核心区使用 50cm（与 Unity 相同），其余区域使用更低分辨率
- **稀疏探针放置**：octree 节点仅在包含 surfel（表面元素）时才细分；丢弃在几何内部的探针
- **结果：平均仅需均匀分布约 10% 的探针**

**为什么密度不均匀可行？** GI 的细节质量取决于探针密度，但开放世界中大部分区域是地面、树林——不需要城市场景级别的高精度探针。NCT map 让美术精确控制"哪里需要高精度"，其余区域自动降级。

**策略三：时间与季节压缩存储**

- **时间**：存储 **11 个关键帧**（1 个月光 + 10 个太阳位置），数据驱动可调整
- **存储格式**：YCoCg 格式，Sun / Moonlight / Sky 分开存储
  - **Y（亮度）**：方向性信息，存为 **Spherical Harmonics（球谐函数——用一组数学基函数近似表示方向性光照分布，类似傅里叶级数但作用在球面上）**
  - **Co/Cg（色度）**：标量值

**季节存储的两个"明知有误"的假设**：
1. **春天 = 夏天**（间接光照）。近似合理，因为植被反照率几乎一致，季节变化更多来自资产替换和天气雾气
2. **Y（亮度）四季共用**，Co/Cg 每季节单独存储。可能导致 Y 和 Co/Cg 不匹配（如某几何体在冬季不存在），但实践中通过技术美术标记跳过有问题的物体来解决

### 3.3 运行时：级联 3D 体积 + 稀疏块插入

**关键术语解释**：
- **Cascade（级联）**——想象一摞从大到小的同心盒子套在摄像机周围。近处的盒子小而密，远处的大而疏。每个盒子是一个级联，负责一个距离范围的 GI 采样。近处需要高精度（盒子小），远处可以粗糙（盒子大）。

**从 AC Unity 的离散块到 Shadows 的级联体积**：
- AC Unity：离散 GI 块 + mipmaps，随视距流式加载——类似 2D 纹理的 mipmap streaming，但是在 3D 上
- Shadows：稀疏 GI 块插入到 **cascaded 3D volumes**（级联 3D 体积）中，产生更连续的 LOD 过渡

运行时流程：
1. 稀疏体素解压为均匀级联
2. 执行 **cross-cascade blending**（跨级联混合）
3. 执行 **in-place GI block LOD blending**（块加载时的原地 LOD 混合）避免 popping（跳变）

**压缩效果**：理论 2TB → 实际 **~9GB**（依赖 density map + 稀疏网格）。**9GB 相对于 2TB 的 200:1 压缩比，让 GI 数据从"装不下蓝光盘"变为完全可行**。

### 3.4 镜面反射：SSR + Cubemap 光谱

反射系统按优先级分层：

1. **SSR（Screen Space Reflections，屏幕空间反射）**——只在屏幕内可见的像素之间做反射。优点是精确，缺点是屏幕外信息完全丢失。
2. **Baked Cubemaps（烘焙立方体贴图）**——存储 albedo、normal、depth，动态烘焙。阴影通过 8 个关键帧烘焙到 8-bit 纹理中（每关键帧 1 bit），运行时选最近两个关键帧混合。因 cubemap 分辨率不高，无 filtering 的跳变几乎不可见。
3. **Dynamic Cubemap（动态立方体贴图）**——位于玩家位置，作为 regional fallback。极低成本：渲染 fake entities + 低分辨率阴影。

**季节处理**：通过 **variation states** 实现，每个 cubemap 最多 3 个变体（春+夏共用）。游戏共有 **16,000 个 cubemap**（不含变体），大部分有季节变体，少量室内可跳过冬季变体。

**为什么 SSR 优先？** SSR 是精确的逐像素反射，无需预计算。但它只能看到屏幕内的内容，因此必须用 cubemap 作为 fallback。"可靠"的烘焙 cubemap 已经过验证且成本低，动态 cubemap 仅作为最后的兜底。

---

## ☀️ 四、光线追踪（RT）架构

### 4.1 Fusion：软硬件统一抽象层

**关键设计决策**：由于不确定硬件 RT 性能是否足够，且需要覆盖低端平台，团队选择了**完全抽象**的方案。

**Fusion** 是育碧跨引擎共享的硬件抽象层（HAL），采用 insourcing 模式（内部 GitLab，各项目贡献）。

**抽象维度**：
- 平台特定 API（DXR、Vulkan RT 等）→ 统一 API
- **硬件 vs 软件 RT** → 切换仅需 enum
- **Inline vs Non-inline RT** → callback 接口统一

**关键术语解释**：
- **Inline Ray Tracing（内联光线追踪）**——在普通 compute shader 中直接调用光线追踪指令，光线就像函数调用一样嵌入你的代码。灵活但可能不够优化。
- **Non-inline Ray Tracing（非内联光线追踪）**——使用专门的 shader stage（ray generation、closest hit、any hit、miss），由硬件调度器管理。通过 shader table 配置，更接近传统图形管线。
- **Software Ray Tracing（软件光线追踪）**——不用 RTX 硬件的专用单元，完全用 compute shader 模拟 BVH 遍历和光线求交。性能更低但可在任意 GPU 上运行。

**Callback 接口**统一 inline 和 non-inline：
- Inline RT 中，遇到 hit/miss 时调用对应 callback 函数
- Non-inline RT 中，在对应 shader stage 中调用同一个 callback 函数
- **切换只需设置 shader table**，C++ 代码几乎不变
- 甚至支持 CPU ray tracing（同一条代码路径）

**为什么这么抽象？** 团队缺乏数据支撑来判断"硬件 RT 是否够用"，因此需要保留随时切换的能力。实际开发中，经常在 inline/non-inline 之间来回切换以验证假设、测试新功能。**这种"保持选项开放"的策略让团队能够在性能和质量之间灵活调整**。

### 4.2 RTGI 管线详解

**RTGI = Per-Pixel RT + DDGI Probe Cascade**

```
Screen Space Rays ──hit──> Hit Buffer
        │                              
        └──miss──> World Space RT (BVH) ──hit──> Hit Buffer
                       │
                       └──miss──> Sample Probe Cache
                                      │
                                      ▼
                        Hit Buffer ──> Re-lighting Pass (default lighting)
                                      │
                                      ▼
                              Denoiser
                                      │
                                      ▼
                              + Optional RTAO
                              + Optional SSAO (quarter-res补偿)
                              + Optional RT Specular
                                      │
                                      ▼
                                 Final Image
```

**DDGI Probe Cascade**：
- **DDGI（Dynamic Diffuse Global Illumination，动态漫反射全局光照）**——一种用 3D 探针网格近似间接光照的技术。每个探针存储来自各方向的入射光。与静态烘焙不同，DDGI 通过不断更新探针来适应动态场景变化。
- 5 个级联，共 **10,000 probes**
- 每帧更新约 **1,000 probes**

**两步法原理**：
1. **Per-pixel RT**：先用屏幕空间光线（免费）尝试解析，miss 后用世界空间硬件光线继续（沿同一 T 值继续步进）。所有 hit 存入 **RT G-buffer**（PBR 属性）
2. **Re-lighting**：用标准光照 pass 对 RT G-buffer 重新打光。若某像素无 hit，触发 miss 事件采样探针缓存

**多弹跳**：第一弹跳来自 per-pixel RT，后续弹跳从 DDGI cascade 采样获取。

**为什么屏幕空间先行？** 硬件光线追踪极其昂贵（每条光线需要遍历 BVH）。先在屏幕空间解析本质上是"免费的加速"——如果反射目标已经在屏幕上了，完全不需要发射硬件光线。**只有屏幕空间 miss 的光线才 fall back 到世界空间**，大幅减少硬件光线数量。

### 4.3 RTAO：延迟应用的必要性

**为什么不在光积分内部做 AO，而是延迟到最后再加？**

1. **频率不匹配**：per-pixel rays（高频率细节）+ RT probes（低频率信号）叠加时，探针会衰减 AO 细节
2. **四分之一分辨率**：屏幕空间光线在多数平台上以 quarter resolution 运行，丢失了高频细节，需要补偿
3. **raster 与 RT 世界不一致**：RT 世界中不含草地等 per-instance 噪声对象，导致大物体（房屋、树木）缺少接地感。重新施加 RTAO 帮助它们"扎根"到画面中

此外还保留了微妙的 **SSAO（Screen Space Ambient Occlusion，屏幕空间环境光遮蔽——通过深度缓冲估算像素周围被几何遮挡的程度，用于模拟缝隙和角落处的阴影）** 项，补充 quarter-res 丢失的厘米级细节。

### 4.4 BVH 加速结构

**关键术语解释**：
- **BVH（Bounding Volume Hierarchy，层次包围盒）**——可以想象成一棵决策树：先问"这个大房间里有东西吗？"，没有就跳过整个房间；有的话再问"这个角落里有东西吗？"，一层层缩小范围。光线追踪用它避免对每个三角形逐一检测，将复杂度从 O(n) 降到 O(log n)。

**构建策略**：
- 支持二值纹理（binary texture）或平均色（低端平台）
- Meshes 按 size 或 type 分类
- 全局 **LOD offset** 决定 BVH 中放什么
- Micro Polygon 使用 **LOD error metric** 输出 proxy meshes（因为不支持 Nanite-style mega geometry）
- 一切均可 per-asset / per-material 覆盖（LOD meshes、材质属性、shader 等）

**统计数据**（典型城市场景，Xbox Series X）：
- ~2,000 meshes / 30,000 instances
- BVH 数据总量 **~320 MB**（PC 更大，因更多抽象层）

### 4.5 Alpha-Test 材质的光线追踪方案

团队尝试了三种方案，经历了典型的"试错→选择"过程：

| 方案 | 做法 | 结果 |
|------|------|------|
| 蓝噪声随机采样 | 用蓝噪声近似透明度 | 精度不够 |
| Bindless Alpha Texture + Any Hit | 直接读取纹理 alpha 做 any hit | 遮挡过重（叶片太密）或 any hit 太贵 |
| **三角形缩放（选定）** | 按平均透明度缩放三角形面积 | ✅ 30% 比 any hit 快，结果接近参考 |

**三角形缩放的原理**：如果一个叶子的 alpha 是 30%，就把代表它的三角形缩小到 30% 面积。光线命中的概率自然降低，避免了 any hit 的重叠开销。

**为什么最终方案效果好？** 因为 closest hit（用于屏幕空间反射）已经在屏幕空间精确解析了，所以不透明层只是提供一层背景间接光照。**你得到了两全其美的效果**：屏幕空间像素完美 + 世界空间半透明近似层。作为额外好处，这个近似对 specular RT 也表现良好。

### 4.6 材质系统：Uber Shader + Material Table

因为使用 inline RT，采用 **Uber Shader（通用着色器——一个包含所有材质分支的大 shader，通过分支选择实际路径）** 方案。

**为什么能写 Uber Shader？** 育碧内部对 shader 种类管理非常严格，实际材质变体数量可控。演讲者提到看 Capcom 用 JSON 文件重映射 shader 时"很庆幸我们不需要这样做"。

- 材质存储在 **Material Table**（通过 hit geometry ID 在 hit 时访问）
- 季节处理：material table 中的 material variation（落叶时减少遮挡 + 叶色 lookup table）
- **deferred weather 在 specular RT 中完全计算，diffuse RT 中不计算**（多次求交时代价过高）
- 静态雪烘焙在 terrain vista 中，diffuse RT 可访问

### 4.7 软件 RT 栈

- 与 Snowdrop 引擎共享，属于 Fusion 低层抽象
- 面向低端平台（如 GTX 1070），未在主机上发布
- **Three-level BVH** + 空间分区做部分更新（仅当 cell 加载或物体移动时）
- 仍依赖 **indoor volumes**（平面列表）防止室内 GI 泄漏——同一数据被 Baked GI、RTGI、deferred weather、声音等多个系统复用

### 4.8 半透明表面处理

- 简单半透明：flip rays on translucent surfaces（反转光线方向）
- **Soji（障子）门**等特殊半透明：在表面两侧采样探针，按透射因子混合

### 4.9 局部光照：全向 Cluster Lighting

**关键术语解释**：
- **Cluster（簇光照）**——把摄像机周围的空间切成小格子（比如 16×16×16），每个格子记录里面有哪些光源。渲染时像素所在的格子直接告诉你需要处理哪些灯，避免遍历所有光源。

- 基于 regular cluster lighting，关键差异：cluster volume 映射在摄像机周围的**均匀网格**上
- **两级层次**：high cluster + cluster，每个 high cluster 包含 16×16×16 cluster，共 **260K clusters**
- 先 coarse cluster culling，再 fine cluster culling

### 4.10 RT Specular：最后一刻的冲刺

- 游戏延期后追加的功能，复用 RT Diffuse 大部分架构
- 主要面向 **PS5 Pro 和 PC**
- 主要挑战：BVH 质量问题（diffuse GI 低频下不明显，specular 高频下暴露）和去噪
- 以 **半分辨率**（宽度÷2，高度不变）发布，因为效果显著更好且有预算余量
- PS5 Pro 因整体 RT 成本降低，能维持目标分辨率

### 4.11 去噪器

引擎内集成三个去噪器：

| 去噪器 | 来源 | 类型 | 是否使用 |
|--------|------|------|---------|
| Atru | 内部 | SVGF 家族 | ❌ |
| NRD Reblur | NVIDIA | recurrent blur | ❌ |
| **MSD** | Snowdrop (Ubisoft) | spatial + temporal, recurrent blur | ✅ |

**MSD 的扩展**：
- **material masking**：为角色、植被、动态物体单独处理（避免拖影和伪影）
- **spatial harmonics denoising**：信号存储在 YCoCg 中，Y 通道使用 spatial harmonics（空间谐波——一种用低频基函数表示高频信号的技术，类似 JPEG 压缩中用 DCT 保留主要频率分量）

### 4.12 RT 性能数据

| 项目 | 成本 |
|------|------|
| RT Probes（10,000 probes） | **~1 ms**（大部分主机）；PS5 Pro 更低 |
| Per-pixel RT（diffuse） | **~4-5 ms**（大部分主机）；PS5 Pro 更低 |
| **RTGI 总计（diffuse）** | **5-6 ms**，分散在 graphics queue + async queue |
| RT Specular | 半分辨率，成本较高；PS5 Pro 可承受 |

**Probes 仅 1ms 的低成本是它作为屏幕空间 miss fallback 的关键原因**——足够便宜可以每帧运行，同时提供合理的多弹跳间接光照。

---

## 🌦️ 五、天气与四季系统

### 5.1 Atmos：大气流体模拟

**Atmos** 模拟并传播大气因子（vapor、temperature、humidity），用低分辨率体素数据（围绕玩家）做风碰撞。输出驱动多个系统：
- 体积云（formation + dissipation）
- 风
- 雨

**为什么需要流体模拟？** 天气不是简单的"下雨/不下雨"开关。湿度需要从水面蒸发、随风传播、遇冷凝结成云再降雨——这是一个物理过程链。Atmos 让天气变化更自然，避免突然切换。

### 5.2 Ambience Graph：技术美术驱动的系统化逻辑

取代前作的 manager 系统（基于曲线，逻辑有限），Shadows 引入基于 node graph 的 **Ambience Graph**：
- 消费来自引擎和 Atmos 的输入
- 驱动整个天气和季节栈的逻辑
- 完全数据驱动，技术美术可自由定义行为

### 5.3 Deferred Rain（延迟雨湿效果）

基于 Sebastian Lagarde 的方法：
- Albedo 变暗 + roughness 降低（湿表面更光滑更暗）
- **wetness mask** 存储在 G-buffer 中（single float）
- 角色不参与此系统（缺少 dynamic character layer）
- 演进序列：干燥 → 湿润 → 水坑

### 5.4 雪：三层系统

1. **Deep Snow（深雪）**：data-driven stomper 印戳系统，可压印胶囊和纹理来形变地形/地面
2. **Deferred Snow（延迟积雪）**：类似 Deferred Rain，修改材质输入（Albedo → 雪色，roughness → 雪粗糙度）。由 cold/warm zones 驱动（静态 mask）
3. **融化序列**：暴风雪消散 → 雪量缓慢减少 → 变为湿润/水坑 → 干燥。逻辑全部由 Ambience Graph 驱动

### 5.5 室内外隔离

复用 Baked GI / RTGI / 声音系统的 **indoor volumes**（平面列表）：
- 生成 **indoor depth map** 用于碰撞因子
- 雨粒子：沿雨方向的定向深度图做遮挡
- **已知限制**：雨粒子可以透过窗户进入室内

---

## 📊 六、可扩展性与性能管理

### 6.1 Platform Manager

由于性能模式（baked GI）和质量模式（RTGI）的帧结构完全不同，团队开发了 **Platform Manager** 管理所有平台和上下文的性能设置。

**层级结构**：
| 层级 | 作用 | 热重载 |
|------|------|--------|
| Profile | 性能/平衡/质量模式 | 需重载世界 |
| Profile Boot | 启动级设置 | 需重启 |
| Context | 游戏玩法触发（菜单/照片模式/游戏内） | 实时 |
| Modifier | 数据触发的局部调整（画笔/3D体积） | 实时 |

- UI 自动生成，所有设置实时可编辑
- 可在编辑器中模拟主机设置（有限但有效）

**为什么需要这么多层级？** 因为同一个游戏中，开放世界和洞穴的性能特征完全不同。照片模式可能需要更高的毛发渲染质量。Context 和 Modifier 层让团队在不改变全局 Profile 的情况下精确调控局部场景。

### 6.2 内存追踪

- **高层工具**：资源生命周期追踪 + 帧内内存峰值定位
- **低层工具**：分配器检查（内存浪费、aliasing 模式）
- **遥测系统**：所有计数器存入数据库，追踪性能回归

### 6.3 帧调度挑战

- 性能模式（无 RT）和质量模式（有 RT）是两种不同的帧结构
- 团队试图保持 GPU scheduling 尽量接近以避免模式特定 bug——"是个头疼事"
- 未来可能需要 **data-driven frame graph scheduling**
- mono repo 中多个游戏共享代码库也增加了复杂度

---

## 🔑 关键数据汇总

### 世界与场景

| 参数 | 值 |
|------|------|
| 世界尺寸 | 16 × 16 km |
| 远距渲染极限 | Point clouds 至 8 km，terrain vista 至更远 |
| Point Cloud | mass quad renderer（树木 imposter） |
| Fake Entity Grid | 至 4 km |
| cubemap 总数 | 16,000（不含季节变体） |

### 几何管线

| 参数 | 值 |
|------|------|
| Micro Polygon cluster 大小 | 128 triangles |
| 每个 mesh 最大 LOD 数 | 5 |
| 每个 LOD 独立距离数 | 2（main view + shadow） |
| Micro Polygon vs GPUIR 内存比 | ~50% |
| 网格简化工具 | Simpligon |
| Cluster partitioning | Metis 库 |

### 场景渲染统计

| 场景 | 管线 | Instances | 渲染三角形 | 软光栅占比 |
|------|------|-----------|-----------|-----------|
| 京都（城市） | Micro Polygon | ~28,000 | 3400 万 | ~90% |
| 京都（城市） | GPU Instance Renderer | ~9,000 | ~200 万 | — |
| 森林 | GPU Instance Renderer | 300万→3万（剔除后） | 15亿→700万 | — |

### 烘焙 GI

| 参数 | 值 |
|------|------|
| 理论体积（旧方法推算） | ~2 TB（四季），~500 GB（无四季） |
| 实际体积 | ~9 GB |
| 体素密度 | 密度图控制，城市核心 50cm |
| 探针压缩率 | 均匀分布的 ~10% |
| 时间关键帧 | 11（1 个月光 + 10 太阳位置） |
| GI 存储格式 | YCoCg（Y = SH 方向性，Co/Cg = 标量） |
| 季节假设 | 春=夏；Y 四季共用，Co/Cg 按季节存储 |
| Cubemap 阴影关键帧 | 8（8-bit 纹理，1 bit/帧） |
| Cubemap 季节变体 | 最多 3 个/cubemap（春+夏共用） |
| 烘焙方式 | CPU Path Tracer |

### 光线追踪

| 参数 | 值 |
|------|------|
| DDGI 级联数 | 5 |
| DDGI 探针总数 | ~10,000 |
| 每帧更新探针 | ~1,000 |
| RT Probes 成本 | ~1 ms（大部分主机） |
| Per-pixel RT（diffuse）成本 | ~4-5 ms（大部分主机） |
| RTGI diffuse 总成本 | 5-6 ms |
| RT Specular 分辨率 | 半分辨率（宽÷2，高不变） |
| BVH 数据（城市场景/XSX） | ~320 MB |
| BVH 场景（城市场景/XSX） | ~2,000 meshes / 30,000 instances |
| 屏幕空间光线分辨率 | Quarter（大部分平台） |
| Cluster 数量（全向 cluster） | 260K（两级层次） |
| Alpha-test 方案 | 三角形缩放（比 any hit 快 ~30%） |
| Software RT 目标平台 | 低端 PC（如 GTX 1070），非主机 |

### 平台与模式

| 参数 | 值 |
|------|------|
| 目标平台 | 纯次世代（PS5 / Xbox Series X|S） |
| 渲染模式 | 性能 / 平衡 / 质量 |
| RT 启用 | 质量模式（diffuse）；PS5 Pro + PC 额外支持 specular |
| Valhalla GPU 利用率（XSX 4K@30fps） | ~20 ms |
| Valhalla 可扩展性 | 极有限（主要靠分辨率缩放） |

---

## 💡 核心收获与启发

### 1. 压缩 ≠ 损失质量：稀疏 + 密度图的 200:1 压缩艺术

GI 数据从理论 2TB 到实际 9GB，不是靠暴力压缩算法，而是通过三个层次的空间/时间/语义稀疏化（密度图、稀疏探针、亮度四季共用）。关键是认识到"不是所有地方都需要相同精度"。**这个思想适用于任何大规模数据问题**：先分析哪些部分真正需要高精度，然后分层分配资源。

### 2. "明知有误但实用"的近似是工程智慧

两个季节假设（春=夏、Y 四季共用）在理论上都有瑕疵，但实践中几乎不可见。团队的应对方式也很务实：允许技术美术标记并排除有问题的物体。**完美的方案如果装不下磁盘，不如"95% 正确但能装下"的方案。**

### 3. 保持选项开放直到有数据支撑

Fusion 抽象层的设计动机不是"我们想支持所有平台"，而是"我们不知道硬件 RT 够不够用"。通过 inline/non-inline、硬件/软件的完全抽象，团队可以在开发过程中自由切换以验证假设。**在技术方向不确定时，投资于可逆性比投资于单一方案更安全。**

### 4. 分层 fallback 是大规模系统的生存策略

反射系统（SSR → Baked Cubemap → Dynamic Cubemap）、GI 系统（RT Diffuse → DDGI Probes → Baked GI）、剔除系统（CPU Coarse → GPU Frame → GPU Per-Pass）都采用分层 fallback。每一层都更快但更粗糙，确保即使在最差情况下系统也不会崩溃。**这种"优雅降级"模式适用于任何需要同时覆盖高端和低端硬件的系统。**

### 5. 专业化分工优于万能方案

Micro Polygon 专注不透明静态几何，GPU Instance Renderer 专注 alpha-test 植被——两个管线各司其职而非试图做一个万能管线。**森林 200:1 的剔除比证明 GPUIR 已经足够高效，没有理由把 alpha-test 强塞进 Micro Polygon。** 先把一件事做到极致，再考虑扩展边界。

---

## 📚 延伸阅读

| 引用 | 关系 |
|------|------|
| Brian Karis - "Nanite: Virtualized Geometry" (2021) | Micro Polygon 管线的基础设计来源 |
| Josh Obson - "Indirect Lighting in God of War" (GDC) | AC Unity 体积化 GI 烘焙管线的详细描述，基于 ACU 的工作 |
| McGuire et al. - "DDGI" | RT probe cascade 的 irradiance cache 设计参考 |
| Sebastian Lagarde - "Darkening Albedo / Wet Surfaces" | Deferred Rain 效果的方法论来源 |
| NVIDIA NRD Reblur | 引擎中集成但未使用的去噪器，MSD 的设计参考 |
| SVGF family | Atru 去噪器的算法族 |
| G3D GPU-Driven articles | GPUIR 管线详细架构参考 |

---

## 📋 审计报告

### 覆盖率

| 类别 | 覆盖 |
|------|------|
| 技术主题 | 14/14（Anvil 演进、Streaming 架构、Batch Renderer、GPUIR、Database、Micro Polygon、烘焙 GI、RTGI 管线、Fusion 抽象层、Alpha-test RT、天气/四季、Platform Manager、Denoiser、RT Specular） |
| 关键数字 | 42/42（世界尺寸、分辨率、三角形数、实例数、剔除比、内存大小、帧时间、探针数、成本、压缩比等） |
| 工具/论文引用 | 7/7（Nanite、God of War GI talk、DDGI、Lagarde wet surfaces、NRD Reblur、SVGF、G3D articles） |
| 因果连接 | 8/8（Valhalla 20ms→Shadows 可扩展性驱动；200:1 剔除→不急需 Micro Polygon alpha-test；2TB→9GB 压缩→蓝光盘可行；10% 探针→NCT map + sparse 三大策略；Probes 1ms→screen miss 的经济 fallback；Quarter-res→延迟 RTAO 补偿；Fusion 抽象→不确定硬件 RT 性能→保持选项开放；mono repo→帧调度复杂度增加） |
| 术语解释 | 18/18（GPU driven pipeline、Draw Call、Indirect Draw、Bindless、Async Compute、Cluster、Software Rasterization、Hierarchical Z-buffer、VZT Buffer、Cubemap、SSR、DDGI、BVH、Spherical Harmonics、Uber Shader、Inline/Non-inline RT、SSAO、Cascade） |

### 因果连接清单

1. ✅ Valhalla XSX 4K@30fps 仅用 ~20ms GPU → 证明 GPU 资源浪费 → Shadows 决定大幅提升可扩展性
2. ✅ 森林 15亿→700万三角面（200:1 剔除）→ GPUIR 对 alpha-test 植被足够高效 → Micro Polygon 暂不需支持 alpha-test
3. ✅ 2TB 理论 GI 数据 → 三大压缩策略 → 实际 9GB → 蓝光盘可行
4. ✅ NCT map + sparse probe → 仅需 10% 探针 → 进一步减小烘焙 GI 体积
5. ✅ RT Probes 仅 ~1ms → 经济到可以每帧运行 → 作为屏幕空间 miss 的可靠 fallback
6. ✅ 屏幕空间光线 quarter resolution → 丢失高频 AO 细节 → 需要延迟 RTAO 补偿
7. ✅ 不确定硬件 RT 性能 → Fusion 完全抽象 → 开发中可自由切换验证
8. ✅ Mono repo 多游戏共享 + RTGI/非 RTGI 帧结构差异 → 帧调度复杂度增加 → 未来需要 data-driven frame graph scheduling

### 未覆盖项（转录中存在但信息不足）

- 具体分辨率表格（演讲中有 spreadsheet 但转录中无法提取数值，只提到"三个模式"和"RT 在质量模式中"）
- Valhalla 的具体分辨率数字（仅提到 4K native）
- BVH 质量问题的具体案例（只提到"specular 暴露了 diffuse 看不出的问题"）