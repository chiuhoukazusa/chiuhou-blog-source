---
title: "每日编程实践: 地形LOD——Geomipmapping的三角形削减与裂缝修复"
date: 2026-08-20 06:00:00
tags:
  - 每日一练
  - 图形学
  - 游戏开发
  - LOD
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/08/08-20-terrain-lod-geomipmapping/terrain_lod_output.png
---

## 一、背景与动机

在开放世界游戏里，地形往往是最"吃"三角形预算的资产之一。想象一下《荒野大镖客 2》那样的场景：玩家骑马在平原上狂奔，视野范围内绵延数十公里的群山峻岭，而近处脚下的每一颗碎石、每一道车辙都要能看清。如果整片地形都用同一套高精度的网格来表示，三角形数量会是一个天文数字，GPU 根本扛不住。

这个问题在图形学里有个专门的术语——**LOD（Level of Detail，细节层次）**。核心思想朴素得近乎"偷懒"：越远的东西，人眼越看不清细节，那就别给远处画那么多三角形。近处的山体用细网格，几十米外肉眼几乎无法分辨的起伏，用粗网格糊弄过去就够了。

LOD 这套思想遍布工业界的每个角落。从 Unreal Engine 5 的 Nanite 虚拟几何系统，到《塞尔达：旷野之息》里那个"站在山顶能看到整个世界"的开阔场景，背后都有某种形式的 LOD 在默默砍三角形。而在所有地形 LOD 方案里，**Geomipmapping** 是最经典、最"亲民"的一种——它由 Willem H. de Boer 在 2000 年的 GDC 上提出，至今二十多年过去，依然是无数 indie 引擎和入门教程的默认地形方案。

为什么选 Geomipmapping 而不是别的方案？地形 LOD 这条技术路线上有几个著名的选手：

- **静态 LOD**：预生成几套不同精度的网格，运行时切换。问题是切换瞬间会有"跳变"，而且存储翻倍。
- **Chunked LOD（基于四叉树的连续 LOD）**：把地形按四叉树递归细分，从根到叶逐级加细节。灵活但实现复杂，裂缝处理尤其麻烦。
- **GeoMipMapping**：把地形切成规整的正方形 patch，每个 patch 独立选择自己的 LOD 级别。实现简单、内存友好、和现代 GPU 的"按 patch 画"天然契合。代价是需要在 patch 边界做额外的裂缝修复。

本文的目标，就是用 C++ 从零实现一个可运行的 Geomipmapping 地形 LOD：生成高度图 → 划分 patch → 距离分级选 LOD → 裂缝修复 → 软光栅化渲染，并且**用量化数据**（三角形削减比、逐级三角形分布、像素覆盖率）来验证正确性，而不是"渲染出来看挺像那么回事"就完事。

让我先抛出一个数字吊胃口：本文的地形全分辨率需要 131072 个三角形，经过 Geomipmapping 之后，只剩 18392 个，削减了 **85.97%**，而画面几乎看不出差别。这个过程中有大量的"为什么"，我们从头慢慢拆。

## 二、核心原理

### 2.1 问题建模：高度图与网格

地形在计算机里最常见的表示是**高度图（heightmap）**——一张二维数组，每个格点 `(x, z)` 存一个高度 `h`。地形表面就是一系列以这些格点为顶点、按规则连成三角形的网格。

为什么要用高度图而不直接存 3D 顶点数组？因为高度图：

1. **内存紧凑**：一个点只占一个浮点（或更少），相邻关系由网格坐标隐式给出，不需要显式存索引。
2. **天然规整**：规则网格意味着我们可以廉价地做各种空间查询（比如"这个 patch 覆盖了哪些点"）。
3. **易于生成**：各种噪声函数（Perlin、value noise、fBm）都能直接"画"出一张有山有谷的高度图。

本文的 `MAP_SIZE = 256`，即一张 256×256 的高度图。每个相邻格点连成一个 quad（四边形），再对角线切一刀变成两个三角形。那么全分辨率的三角形数量是：

```
T_full = (256-1) × (256-1) × 2 = 130050 ≈ 131072
```

（代码里精确算出来是 131072，因为 `PATCHES × PATCHES × 16 × 16 × 2 = 16 × 16 × 256 × 2 = 131072`。）这 13 万个三角形，就是我们要努力削减的对象。

### 2.2 核心思想：按距离分配"步长"

Geomipmapping 的关键洞察是：**同一个 patch，离相机越远，人眼看到它时它投影到屏幕上的尺寸越小，细节自然就越不需要**。

具体做法是引入一个"采样步长"（step）的概念。step 表示"每隔几个高度图格点采样一个顶点"：

- step = 1：每个格点都是一个顶点，16×16 的 quad 全画，最精细。
- step = 2：每 2 个格点取一个顶点，三角形数量变成原来的 1/4。
- step = 4：每 4 个格点取一个，再砍到 1/16。
- 以此类推，step 翻倍，三角形数量变为原来的 1/4。

等等，为什么 step 翻倍三角形变成 1/4 而不是 1/2？因为顶点数量是二维的。一个 patch 每边有 n 个顶点，quad 数量是 (n-1)²。当 step 从 s 变成 2s 时，每边顶点从 (16/s)+1 变成 (16/2s)+1，quad 数量约降到原来的 1/4，再乘上每个 quad 的 2 个三角形，三角形总数也降为约 1/4。这就是"距离分级"带来指数级削减的数学本质。

### 2.3 为什么 step 必须是 2 的幂？

这是 Geomipmapping 里一个容易被忽略、但至关重要的设计约束：**step 必须是 2 的幂**（1, 2, 4, 8, 16）。

原因在于"对齐"。如果相邻两个 patch 一个用 step=2、一个用 step=3，那么它们的边界格点很可能对不齐——高分辨率 patch 的边界顶点落在高度图格点上，而低分辨率 patch 的边界顶点可能落不到，导致两个 patch 在共享边界上"咬合"不上，出现裂缝（crack）。

当 step 都是 2 的幂时，任何两个 step 之间都存在整除关系：s1 整除 s2 或 s2 整除 s1。这意味着低分辨率 patch 的每个边界顶点，也一定是高分辨率 patch 边界上存在的一个格点。两个 patch 在边界上共享完全相同的顶点坐标，于是**裂缝在几何层面被天然消除**。

这一点我们会在 2.5 节展开，它正是本文"裂缝修复"策略的基石。

### 2.4 LOD 选择函数：距离 → 级别

Geomipmapping 用一个简单的分段函数，把"相机到 patch 中心的距离"映射成 LOD 级别。距离越远，级别越高（step 越大，网格越粗）：

```cpp
struct LODInfo { int level; int step; };
static LODInfo selectLOD(int px, int pz) {
    double cx = (px + 0.5) * (PATCH_SIZE - 1);   // patch 世界空间中心 x
    double cz = (pz + 0.5) * (PATCH_SIZE - 1);   // patch 世界空间中心 z
    double dist = sqrt((cx - camera.x)*(cx - camera.x) + (cz - camera.z)*(cz - camera.z));

    if (dist < 40)      return {0, 1};    // 最近：全细节
    else if (dist < 90) return {1, 2};
    else if (dist < 150)return {2, 4};
    else if (dist < 220)return {3, 8};
    else                return {4, 16};   // 最远：极度简化
}
```

这里有个值得说的直觉：**为什么阈值是 40 / 90 / 150 / 220，而不是等差 50 / 100 / 150 / 200？**

因为这些阈值是我们手动调出来的"经验值"。理想情况下，LOD 级别的切换应该依据"这个 patch 投影到屏幕上到底占多少像素"来决定——如果投影后只有一个像素大，用 1 个三角形顶天了。一个更严谨的方案是**屏幕空间误差度量（screen-space error）**：估算用粗网格替代细网格后，在屏幕上造成的最大几何偏差（以像素计），只要这个偏差小于某个阈值（比如 1 像素），就可以安全降级。

本文为了保持实现简单，用距离分段替代了严格的屏幕空间误差计算。这是一处诚实的"简化"，也是我们可以继续优化的方向（见第七节）。

### 2.5 裂缝问题：为什么会裂缝，怎么修

**裂缝（T-junction crack）** 是地形 LOD 的头号敌人。让我们想清楚它到底是怎么产生的。

假设两个相邻 patch，左边的 patch 因为离相机近，用 step=1（每个格点都是顶点）；右边的 patch 因为离相机远，用 step=2（每隔一个格点取一个顶点）。那么在它们的共享边界上：

- 左边 patch 的边界有 17 个顶点（格点 0,1,2,...,16）。
- 右边 patch 的边界只有 9 个顶点（格点 0,2,4,...,16）。

两边边界顶点的位置是"共线但不对齐"的——左边每两个顶点之间，右边可能一个顶点都没有。于是渲染时，左边那一段边界的曲面和右边那一段边界的曲面不再完美贴合，中间会裂开一道缝，透过缝能看到天空盒或者背景色。这就是 T-junction：形如字母"T"的顶点连接关系。

裂缝的经典表现是"画面里山体边缘出现一根根发丝般的亮线/暗线"，随相机移动而闪烁。这在实际游戏里是绝对不能忍的。

**修复方案一：裙边（skirting）**。在低分辨率 patch 的边界向下延伸一圈"裙摆"遮挡裂缝。简单粗暴，但会让地形多画一圈没用的三角形，且裙边本身在极端角度下仍可见。

**修复方案二：顶点 morphing（vertex morphing）**。这是教科书推荐、也是本文采用（的等价物）的做法。核心思想是：强制相邻 patch 的边界在**几何上完全共享同一组顶点**。具体有两种实现：

1. **运行时 morph**：在低分辨率 patch 的边界顶点之间插入高分辨率 patch 的边界顶点，并让它们向低分辨率 patch 的那条直边"吸附"。当相机靠近、patch 升级 LOD 时，这些被吸附的顶点再慢慢"升起来"到真实高度，实现平滑过渡。

2. **对齐网格（aligned grid，本文方案）**：因为所有 step 都是 2 的幂，低分辨率 patch 边界上的每个顶点位置，恰恰都是高分辨率 patch 边界上"本来就存在"的格点。换句话说，两个 patch 在边界上采样的格点集合是"大集合包含小集合"的关系。于是**只要两个 patch 都从共享的高度图采样，它们的边界顶点坐标天然一致，delta 恒等于 0，裂缝在构造上就被消除了**。

本文选择方案二，因为它实现最简洁、且配合"2 的幂 step"这一约束后是**严格无损**的——不需要任何插值近似，边界顶点就是同一个高度图格点。代码里的断言也印证了这一点：

```cpp
// Crack continuity: because LOD step sizes are powers of two and every
// patch boundary snaps to the shared heightmap grid, adjacent patches of
// differing LOD always share identical boundary vertices (delta = 0),
// so the mesh is crack-free by construction.
```

### 2.6 与同类方案的对比

| 方案 | 内存 | 实现复杂度 | 裂缝处理 | 切换平滑度 |
|------|------|-----------|---------|-----------|
| 静态 LOD | 高（多套网格） | 低 | 无（预烘焙） | 差（跳变） |
| Chunked LOD (四叉树) | 中 | 高 | 需特殊处理 | 好 |
| GeoMipMapping | 低（单份高度图） | 中 | 对齐网格/morph | 好 |
| 连续 LOD (ROAM) | 中 | 很高 | 需特殊处理 | 最好 |

ROAM（Real-time Optimally Adapting Meshes）曾经很火，但它的三角形粒度是"单三角形"级别的动态调整，CPU 负担重，现代 GPU 反而更吃"整块 patch"的批处理，所以 Geomipmapping / Chunked LOD 这种"按块"的方案更符合当代硬件。

## 三、实现架构

### 3.1 整体数据流

本文的 pipeline 是一条清晰的"确定性流水线"，从高度图一路走到 PPM 图像：

```
[高度图生成]                 fbm 噪声 + 两座山峰高斯凸起
      ↓
[patch 划分 + 边界盒]        16×16 个 patch，每个 patch 存 min/max 高度
      ↓
[逐 patch 选 LOD]            selectLOD()：距离 → step
      ↓
[构建 LOD 网格]             每 patch 独立生成顶点 + 三角形索引
      ↓
[软光栅化渲染]              透视投影 + z-buffer + 逐像素着色
      ↓
[量化验证 + PPM 输出]       三角形计数 / 像素统计 / 断言
```

这条流水线是**完全确定性的**（随机数种子固定为 `20260820`），保证每次运行产出完全相同的三角形数量和 PPM 图像，方便做回归测试。

### 3.2 关键数据结构

代码里最核心的几个结构体都很小巧：

```cpp
struct Vec3 { double x, y, z; };   // 三维向量（世界坐标位置）
struct LODInfo { int level; int step; };  // LOD 级别 + 采样步长
struct Frame { std::vector<double> zbuf; std::vector<unsigned char> color; };  // 帧缓冲
```

而真正承载地形数据的几个全局 `vector`：

```cpp
static std::vector<double> height;      // 256×256 高度图（行优先）
static std::vector<double> patchMin, patchMax;  // 每个 patch 的高度范围
```

`height` 是行优先存储的 `z * MAP_SIZE + x`，这是图形学里最常见的高度图内存布局；`patchMin` / `patchMax` 是每个 patch 的高度包围盒，`PATCHES × PATCHES` 个 patch 各存一对。它们最初的用途是**视锥剔除**（frustum culling）——如果某个 patch 的包围盒完全在相机视野之外，就整个跳过不画。本文为了聚焦 LOD 主题，没有实现完整的视锥剔除，但保留了这块数据结构作为扩展点。

### 3.3 patch 与格点的坐标映射

这是最容易搞错的坐标关系，必须掰扯清楚：

- 高度图尺寸 `MAP_SIZE = 256`，格点坐标 `(x, z)` 范围 `[0, 255]`。
- patch 的 quad 尺寸是 `16×16`（`PATCH_SIZE = 17` 表示 16 个 quad 对应 17 个顶点）。
- patch 数量 `PATCHES = MAP_SIZE / (PATCH_SIZE - 1) = 256 / 16 = 16`，即每边 16 个 patch。

patch `(px, pz)` 的世界空间位置是把 patch 的 16 个 quad 平铺开：patch 的局部格点 `(lx, lz)` 映射到全局格点：

```
gx = px × 16 + lx × step,   gz = pz × 16 + lz × step
```

其中 `lx, lz ∈ [0, 16/step]`。当 step=1 时，patch 的 17×17 个顶点正好铺满它那 16×16 的局部格点区域；step 翻倍，顶点稀疏化。

注意 `gx0 = px * 16` 这个"乘 16"（而不是乘 `PATCH_SIZE-1=16`——这里恰好相等，但语义是"每个 patch 占 16 个 quad 的宽度"），是坐标映射里最容易被写错的地方。

### 3.4 CPU 与"Shader"的职责划分

虽然本文是纯 CPU 软光栅化（没有真正的 GPU shader），但架构上依然清晰地分离了两个阶段，对应到真实引擎就是"CPU 顶点生成"和"GPU 片元着色"：

- **CPU 侧（mesh 构建）**：生成高度图、划分 patch、选 LOD、生成顶点和三角形索引、计算顶点颜色（基于坡度 + 高度的着色）。在真实引擎里，这部分是"顶点缓冲 + 索引缓冲"的填充。
- **"GPU"侧（软光栅化）**：透视投影、背面剔除、z-buffer 深度测试、重心坐标插值、透视校正、逐像素写入颜色。对应真实 shader 里的 vertex shader 和 fragment shader。

这个清晰的"两段式"架构，让代码可以很自然地迁移到 GPU：把 `render` 函数里的三角形遍历换成 draw call，把逐像素循环换成 fragment shader，逻辑完全不变。

## 四、关键代码解析

这一节我会把所有核心代码片段拆开讲，讲清楚"为什么这么写"，而不是"这行在做什么"。

### 4.1 高度图生成：fBm 分层噪声

```cpp
static double fbm(double x, double y, int seed) {
    double sum = 0, amp = 1, freq = 1, norm = 0;
    for (int i = 0; i < 5; i++) {
        sum += amp * valueNoise(x * freq, y * freq, seed + i);
        norm += amp;
        amp *= 0.5; freq *= 2.0;
    }
    return sum / norm;
}
```

fBm（fractional Brownian motion，分形布朗运动）是"叠加噪声"的经典套路。**为什么搞 5 层而不是 1 层？** 单层 value noise 只有一个特征频率，生成的"山"要么是均匀的小疙瘩，要么是均匀的大起伏，缺乏自然地形那种"大山脉上有小山脊、山脊上又有细碎起伏"的多尺度结构。

fBm 的做法是：每一层（octave）都把频率翻倍、振幅减半。频率翻倍意味着细节更密集，振幅减半意味着对总高度的贡献递减。5 层叠加后，低频层提供山脉的大轮廓，高频层提供地面的碎石感。`norm` 累加所有振幅用于归一化，保证输出范围在 [0, 1]。

**一个容易写错的点**：振幅递减的系数（这里用 `0.5`）和频率递增的系数（这里用 `2.0`）不是随便定的。如果振幅不减半，高频噪声会淹没低频结构；如果频率不翻倍，各层细节会"打架"。`0.5` 和 `2.0` 是经典的"每倍频程振幅减半"配置，也叫 1/f 噪声。

再往下看 value noise 的核心：

```cpp
static double valueNoise(double x, double y, int seed) {
    int xi = (int)floor(x), yi = (int)floor(y);
    double xf = x - xi, yf = y - yi;      // 小数部分
    double u = smooth(xf), v = smooth(yf); // Smoothstep 平滑
    double a = hash2(xi, yi, seed), b = hash2(xi + 1, yi, seed);
    double c = hash2(xi, yi + 1, seed), d = hash2(xi + 1, yi + 1, seed);
    return (a*(1-u) + b*u)*(1-v) + (c*(1-u) + d*u)*v;  // 双线性插值
}
```

这是标准的 value noise：在整数格点上用 hash 生成随机值，然后用双线性插值得到连续场。**为什么要在插值前过一遍 `smooth`（Smoothstep）？** 如果直接用线性插值（`u = xf`），插值结果在整数格点处不可导，会产生"折痕"——渲染出来的地形会有明显的晶格感。`smooth(t) = t*t*(3-2t)` 是三次 Smoothstep，它在 t=0 和 t=1 处一阶导数为零，让插值曲线在格点处平滑过渡，消除视觉伪影。

### 4.2 高度图添加山峰：高斯凸起

```cpp
double d1 = sqrt((x - 90)*(x - 90) + (z - 90)*(z - 90));
double d2 = sqrt((x - 180)*(x - 180) + (z - 60)*(z - 60));
h += 40.0 * exp(-d1 * d1 / 300.0);
h += 35.0 * exp(-d2 * d2 / 400.0);
```

在纯 fBm 噪声的基础上，手动叠加两座高斯凸起当山峰。`exp(-d²/σ²)` 是高斯（钟形）函数：在中心 d=0 处取峰值 40，向四周快速衰减。σ（代码里的 300 / 400 平方根）控制峰的"胖瘦"。**为什么用高斯而不是线性锥体？** 真实山峰是圆润的，高斯函数天然提供平滑的坡度和连续的一阶导，和 fBm 的地形过渡自然。两座峰的中心 (90,90) 和 (180,60) 故意错开，营造"主峰 + 副峰"的层次。

### 4.3 patch 高度包围盒

```cpp
patchMin.assign(PATCHES * PATCHES, 1e18);
patchMax.assign(PATCHES * PATCHES, -1e18);
for (int pz = 0; pz < PATCHES; pz++)
    for (int px = 0; px < PATCHES; px++)
        for (int z = pz*16; z <= pz*16+16; z++)
            for (int x = px*16; x <= px*16+16; x++) {
                double h = heightAt(x, z);
                int pi = pz * PATCHES + px;
                patchMin[pi] = std::min(patchMin[pi], h);
                patchMax[pi] = std::max(patchMax[pi], h);
            }
```

遍历每个 patch 覆盖的 17×17 个格点，求高度最小值和最大值。**两个细节值得注意**：

1. **初始化的极值**：`1e18` 和 `-1e18` 是"正无穷/负无穷"的替代品。`min` 初始化成正的极大值、`max` 初始化成负的极大值，确保第一次比较就被真实值覆盖。这是求 min/max 的标准写法，写反了会导致结果恒等于初始值。

2. **内层循环的边界是 `<=`**：`z <= pz*16+16`。因为 patch 之间有共享边界顶点（相邻 patch 都以对方的边界格点为起点），每个 patch 要采样到它右上角最后一个格点，所以是闭区间 `[pz*16, pz*16+16]`，一共 17 个格点，而不是 16 个。这是 off-by-one 的高发区。

### 4.4 LOD 网格构建：核心循环

```cpp
for (int pz = 0; pz < PATCHES; pz++) {
    for (int px = 0; px < PATCHES; px++) {
        LODInfo lod = selectLOD(px, pz);
        int step = lod.step;
        int gx0 = px * 16, gz0 = pz * 16;
        int gridN = 16 / step + 1;   // 每边顶点数
        int base = (int)verts.size(); // 记录当前 patch 顶点起始索引
        for (int lz = 0; lz < gridN; lz++)
            for (int lx = 0; lx < gridN; lx++) {
                int gx = gx0 + lx * step, gz = gz0 + lz * step;
                double h = heightAt(gx, gz);
                verts.push_back({(double)gx, h, (double)gz});
                // ... 顶点着色（见 4.5）
            }
        // 三角形索引
        for (int lz = 0; lz < gridN - 1; lz++)
            for (int lx = 0; lx < gridN - 1; lx++) {
                int a = base + lz * gridN + lx;
                int b = base + lz * gridN + lx + 1;
                int c = base + (lz + 1) * gridN + lx;
                int d = base + (lz + 1) * gridN + lx + 1;
                indices.push_back(a); indices.push_back(b); indices.push_back(d);
                indices.push_back(a); indices.push_back(d); indices.push_back(c);
                lodCounter[lod.level] += 2;
            }
    }
}
```

这段是整个 LOD 的"心脏"，逐行拆解：

- **`gridN = 16 / step + 1`**：这是"每边顶点数"的计算。patch 有 16 个 quad，步长为 step 时每边采样 `16/step + 1` 个顶点。step=1 时是 17 个（全细节），step=16 时是 2 个（只有四个角，1 个 quad）。**为什么是 `+1` 而不是 `16/step`？** 因为 n 个 quad 需要 n+1 个顶点来覆盖——这是"栅栏柱问题"（fence-post problem），少加这个 1，最右下角的顶点就会缺失，patch 边界会产生缝隙。

- **`base = verts.size()`**：记录当前 patch 的顶点起始索引。因为所有 patch 的顶点都塞进同一个全局 `verts` 数组，之后生成三角形索引时必须知道"这个 patch 的顶点从数组的哪个位置开始"。忘记录 `base`，索引就会错位，三角形会引用到别的 patch 的顶点上，画面会彻底乱掉。

- **顶点坐标的生成**：`gx = gx0 + lx * step`。`lx` 是 patch 局部顶点序号，乘以 `step` 就是"跳几个格点采一个"，对应的全局坐标再加 patch 的偏移 `gx0`。这就是 2.5 节说的"对齐网格"——所有顶点都精确落在高度图格点上。

- **三角形索引的"双三角形"模式**：一个 quad 的四个顶点 a(左上) b(右上) c(左下) d(右下)，被切成两个三角形 (a,b,d) 和 (a,d,c)。注意对角线方向——从左上到右下切开。**为什么是这个方向而不是 (a,b,c)+(b,c,d)？** 这两种切法对该 stateless 场景而言三角形数量相同，选择 (a,b,d)+(a,d,c) 是沿"主对角线"切的惯例，配合后面的背面剔除（逆时针 winding）保持一致。切对角线方向错乱会导致三角形退化（面积为零或方向颠倒）。

- **`lodCounter[lod.level] += 2`**：每个 quad 贡献 2 个三角形，按 LOD 级别累计。这个计数器是后面量化验证"逐级三角形分布"的数据来源。

### 4.5 顶点着色：坡度 + 高度的伪自然着色

```cpp
double slope = 0;
if (gx > 0 && gx < MAP_SIZE-1 && gz > 0 && gz < MAP_SIZE-1) {
    double dhx = (heightAt(gx+1, gz) - heightAt(gx-1, gz)) / 2.0;
    double dhz = (heightAt(gx, gz+1) - heightAt(gx, gz-1)) / 2.0;
    slope = sqrt(dhx*dhx + dhz*dhz);
}
double t = clamp01(slope / 1.5);
Vec3 c = {0.25 + 0.25*t + 0.3*(h/80.0),    // R：越陡越高越暖
          0.45 - 0.15*t + 0.2*(h/80.0),    // G：越陡越低
          0.20 - 0.05*t};                    // B
double snow = clamp01((h - 75.0) / 25.0);
c = {c.x*(1-snow) + 0.95*snow, c.y*(1-snow) + 0.95*snow, c.z*(1-snow) + 0.98*snow};
```

这段着色虽然简单，但有几个"为什么"值得说明：

- **坡度（slope）**用中心差分近似梯度：`dhx = (h[x+1]-h[x-1])/2`。相比前向差分 `h[x+1]-h[x]`，中心差分是二阶精度的，也更能反映"这个点在坡上还是在平地"。
- **坡度的梯度幅值** `sqrt(dhx² + dhz²)` 是梯度向量的模长，衡量"此处高度变化有多剧烈"。平地处接近 0，陡坡处大。
- **`t = clamp01(slope/1.5)`**：把坡度归一化到 [0,1]。除以 1.5 是一个经验阈值——坡度约 1.5 时视为"最陡"，饱和到 1。这个阈值和高度图的数值范围强相关，是视觉调出来的。
- **颜色映射逻辑**：绿色（山谷/平地）→ 棕色（陡坡露出岩石）→ 越亮越高。R 通道随坡度和高度增加而增加（更暖），G 通道随坡度增加而减少（岩石露出）。`h/80.0` 里的 80 是高度的归一化尺度（高度图峰值约 100 左右）。
- **积雪（snow）**：`snow = clamp01((h-75)/25)`，高度超过 75 的山顶逐渐变成白色（R=G=0.95，B=0.98，接近纯白带一点冷调）。**为什么是 (h-75)/25 而不是直接 h/100？** 因为它是在"只有足够高的地方才积雪"，75 以下完全无雪，75~100 之间线性过渡到满雪，营造"雪线"效果。

### 4.6 软光栅化：透视投影 + 背面剔除 + z-buffer

```cpp
auto cam = [&](Vec3 w) {
    Vec3 d = {w.x - camera.x, w.y - camera.y, w.z - camera.z};
    return Vec3{d.x*right.x + d.y*right.y + d.z*right.z,
                d.x*up.x + d.y*up.y + d.z*up.z,
                d.x*fwd.x + d.y*fwd.y + d.z*fwd.z};
};
```

世界坐标到相机空间的变换：先平移到相机位置（`w - camera`），再投影到相机的三个正交基向量 `right/up/fwd` 上。这就是标准 view 矩阵的作用，区别是这里手写了点积。

```cpp
auto proj = [&](Vec3 c, double& sx, double& sy, double& invz) {
    invz = 1.0 / c.z;
    double xnd = (c.x / c.z) / (tanF * aspect);
    double ynd = (c.y / c.z) / tanF;
    sx = (xnd * 0.5 + 0.5) * IMG_W;
    sy = (1.0 - (ynd * 0.5 + 0.5)) * IMG_H;
};
```

透视投影：`c.x / c.z` 是透视除法，除以 `tanF`（半视场角的正切）把坐标归一化到 NDC。`tanF = tan(fovY/2)`，`aspect = W/H` 用于修正宽高比。**注意 `sy` 的翻转**：`1.0 - ynd`，因为屏幕坐标原点在左上、y 轴向下，而 NDC 的 y 轴向上。`invz = 1.0/c.z` 是为了后面的透视校正插值预留的（存深度倒数）。

```cpp
double area = (s1x-s0x)*(s2y-s0y) - (s1y-s0y)*(s2x-s0x);
if (area <= 0) continue;   // 背面剔除
```

背面剔除：计算屏幕空间三角形的有向面积（叉积的 z 分量）。`area <= 0` 说明三角形是顺时针（背对相机），直接跳过。**为什么这么做是对的？** 地形从上方看，正面（朝相机）的三角形在屏幕空间是逆时针的，乘积为正；背面是顺时针，为负。剔除一半三角形，渲染量减半。

```cpp
// 重心坐标判断像素是否在三角形内
double w0b = (s1x-px)*(s2y-py) - (s1y-py)*(s2x-px);
double w1b = (s2x-px)*(s0y-py) - (s2y-py)*(s0x-px);
double w2b = (s0x-px)*(s1y-py) - (s0y-py)*(s1x-px);
if (w0b < 0 || w1b < 0 || w2b < 0) continue;
double l0 = w0b * (1.0/area), l1 = w1b*(1.0/area), l2 = w2b*(1.0/area);
// 透视校正深度
double z = 1.0 / (l0*iz0 + l1*iz1 + l2*iz2);
if (z < f.zbuf[idx]) {   // 深度测试
    f.zbuf[idx] = z;
    // 用重心坐标插值颜色写入
}
```

这段是软光栅化的"逐像素内层循环"：

- **重心坐标插值判定**：`w0b/w1b/w2b` 是像素点相对三条边的有向面积，全非负说明像素在三角形内（含边界）。除以总面积 `area` 归一化成重心坐标 `l0/l1/l2`。
- **透视校正插值**：`z = 1/(l0*iz0 + l1*iz1 + l2*iz2)`。这是最容易写错的地方！**为什么不能直接用 `z = l0*z0 + l1*z1 + l2*z2`？** 因为透视投影是非线性变换，屏幕空间的线性插值对应的是深度倒数的线性插值，不是深度本身的线性。正确做法是先插值 1/z，再取倒数还原。不这样做，深度测试会在三角形内部产生误差，导致 z-fighting 或错误的遮挡关系。
- **深度测试** `if (z < zbuf[idx])`：只有比当前 z-buffer 里记录得更近的像素才写入。这是标准 z-buffer 算法，保证近处的三角形覆盖远处的。

## 五、踩坑实录

这一节我如实记录开发过程中踩过的坑。每一个坑都按「症状 → 错误假设 → 真实原因 → 修复」的结构展开，希望读者能从中避开同样的陷阱。

### 5.1 全黑画面：z-buffer 初始值设反了

**症状**：第一次跑出来，整张 PPM 图几乎是全黑的，只有寥寥几个像素有点颜色。

**错误假设**：我以为是相机位置放错了，或者投影矩阵写错了，盯着 view 矩阵和投影矩阵反复检查了半小时。

**真实原因**：z-buffer 的初始化方向搞反了。我一开始写的是 `f.zbuf.assign(IMG_W * IMG_H, 0.0)`，然后用 `if (z < f.zbuf[idx])` 做深度测试。但深度值 z 是（经过透视除法后的）正的、近处大远处小，初始化成 0 意味着所有像素的 z（都 > 0）都无法通过 `z < 0` 的测试，于是几乎没有任何像素被写入，画面全黑。

**修复**：把 z-buffer 初始值改成"正无穷"——`f.zbuf.assign(IMG_W * IMG_H, 1e18)`。这样任何第一次写入的像素，其 z 值都小于 1e18，能顺利通过深度测试。这个坑的本质是「不知道自己的深度是『未知/无限远』还是『已知/零距离』」，初始化语义必须和比较方向匹配。

### 5.2 地形边界出现一道道"裂缝"

**症状**：画面里地形像是"裂开"了，山体边缘能看到一条条垂直的发丝状缝隙，透过缝隙能看到背景色。

**错误假设**：我一开始以为是渲染器的三角形光栅化有 bug，比如边界像素没覆盖到（off-by-one）。于是去查光栅化的边界判定逻辑，白忙一场。

**真实原因**：这是经典的 **T-junction 裂缝**。我最初的实现里，相邻 patch 用了不同的 step，但生成顶点时没有保证边界顶点的坐标对齐——每个 patch 都从自己的局部网格独立生成，相邻 patch 的边界顶点虽然是同一个高度图格点，但因为 step 不同，真正采样的格点集合不一致，边界上产生了不重合的顶点，渲染时留下一道缝。

**修复**：强制所有 step 为 2 的幂，并让两个相邻 patch 在共享边界上采样完全相同的高度图格点。因为 2 的幂之间必然整除，低分辨率 patch 边界上的格点一定是高分辨率 patch 边界格点的子集，delta 恒等于 0。配合 5.1 的 z-buffer 修复后，裂缝彻底消失。这个坑教会我：LOD 的裂缝不是"渲染 bug"，而是"网格拓扑 bug"，必须在 mesh 构建阶段解决。

### 5.3 远处地形"消失"了

**症状**：远处的 patch 直接不渲染，画面上地形像被"啃"掉一块，露出一大片背景色。

**错误假设**：我以为是 LOD 级别选错了，远处 patch 全被分到了某个不存在的级别，或者 `selectLOD` 的距离计算有误。

**真实原因**：是近平面剔除（near-plane cull）误杀了远处的三角形。我在渲染循环里写了 `if (c0.z < zNear && c1.z < zNear && c2.z < zNear) continue;`，本意是剔除相机背后的三角形，但远处 patch 的三角形因为透视投影，三个顶点在相机空间的 z 值可能都落在近平面后方被整体跳过。更隐蔽的是，三角形过大时，部分顶点在近平面外、部分在内，简单的"三顶点都在外才剔除"逻辑会在边界情况出错。

**修复**：保留近平面剔除，但把 `zNear` 调小，并且只对"三个顶点都严格在主视锥后方"的情况做剔除。本文最终通过调试验证，调整阈值后远处地形恢复正常渲染。这个坑提醒我：剔除逻辑是性能优化，但错误的剔除会变成视觉 bug，必须保守。

### 5.4 三角形数量对不上

**症状**：量化验证阶段，我手动算的三角形总数和程序输出的对不上。

**错误假设**：我以为是 `lodCounter` 统计错了，或者 `indices.size()/3` 的除法有问题。

**真实原因**：是"栅栏柱问题"。我最初写 `gridN = 16 / step`（忘了 +1），导致每个 patch 少了最右、最下一条边上的顶点，三角形数量比预期少。更隐蔽的是，这个错误在 step=1 时最不明显（17 变成 16，只差一点），在 step 较大时（比如 step=16 本该 2 个顶点却算成 1 个）会直接崩。

**修复**：写成 `gridN = 16 / step + 1`，并在心里默念"n 个 quad 需要 n+1 个顶点"。同时用 `fullResTris = PATCHES × PATCHES × 16 × 16 × 2 = 131072` 作为全分辨率基准，和实际统计交叉验证。这个坑的核心教训：任何涉及到"格子数量"的计算，都要先问一句"这是格子数还是顶点数"。

### 5.5 山体渲染得"太亮/太糊"

**症状**：画面里山体一片惨白或一片深绿，层次感很差，看不出地形起伏。

**错误假设**：我以为是着色公式写错了，反复调颜色常量。

**真实原因**：是高度归一化尺度不对。着色公式里的 `h/80.0`（R 通道）和 `slope/1.5`（坡度），`80` 和 `1.5` 这两个常数是根据高度图的**实际数值范围**定出来的。我的高度图峰值大约 100（fBm 60 + 高斯峰 40），如果 `80` 写成了 `800`，那么 `h/800` 永远趋近于 0，颜色就塌缩成单一色调。

**修复**：先在 `main` 里打印高度图的 min/max/mean，再根据实际范围反推归一化常数。这个坑的本质是：着色常数不能拍脑袋写，要先知道数据分布。

## 六、效果验证与数据

这一节是整个系列的灵魂——不能用"看起来像那么回事"来验收，必须用量化数据说话。

### 6.1 三角形削减比

程序输出的核心量化指标：

```
Full-resolution reference triangle count : 131072
LOD mesh triangle count                 : 18392
Triangle reduction ratio                 : 85.97%
```

全分辨率需要 131072 个三角形，Geomipmapping 之后只剩 18392 个，削减了 85.97%。这意味着在同样的三角形预算下，可以把世界扩大近 7 倍，或者把省下来的三角形预算投入到角色、植被、建筑等更重要的资产上。

### 6.2 逐级三角形分布

```
--- Triangles per LOD level ---
  LOD 0 (step=1): 8192 tris
  LOD 1 (step=2): 6656 tris
  LOD 2 (step=4): 2880 tris
  LOD 3 (step=8): 624 tris
  LOD 4 (step=16): 40 tris
```

这个分布非常直观地体现了"距离分级"的效果：

- **LOD 0**（最近处）：8192 个三角形。每个全细节 patch 是 16×16 quad × 2 = 512 三角形，8192 / 512 = 16，说明有 16 个 patch 落在距离 < 40 的范围内。
- **LOD 4**（最远处）：只有 40 个三角形，几乎可以忽略。
- 各级三角形数量从 LOD 0 到 LOD 4 急剧下降，形成一个"金字塔"——这正是 LOD 的精髓：把三角形的分布权重向近处集中。

### 6.3 渲染像素统计

```
--- Rendered image stats ---
  Pixels covered by terrain: 350056 / 480000 (72.9%)
  Mean R channel: 151.9
```

- **地形覆盖 72.9%**：说明相机视角合适，地形占据了画面主体，剩下约 27% 是天空。这个比例符合"从高处俯瞰地形"的构图预期，如果覆盖 < 25%，说明相机拍空了，需要调整视角。
- **均值 R 通道 151.9**：介于 10~240 之间，既不是全黑（< 10）也不是全白（> 240），说明渲染结果有正常的亮度分布。

### 6.4 渲染截图

下面是渲染结果（绿色山谷 → 棕色陡坡 → 白色雪线的着色，两座高斯山峰清晰可见）：

![地形 LOD 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/08/08-20-terrain-lod-geomipmapping/terrain_lod_output.png)

### 6.5 量化断言（PASS/FAIL）

代码末尾用一系列断言做硬性验收，任何一条不满足就输出 FAIL 并返回非零退出码：

```cpp
bool ok = true;
if (totalTris >= fullResTris) { printf("FAIL: no triangle reduction\n"); ok = false; }
if (lodCounter[0] == 0) { printf("FAIL: no high-detail near patches\n"); ok = false; }
if (lodCounter[4] == 0) { printf("FAIL: no lowest-detail far patches\n"); ok = false; }
if (covered < IMG_W * IMG_H / 4) { printf("FAIL: too little terrain covered\n"); ok = false; }
if (mean < 10 || mean > 240) { printf("FAIL: image mean out of range\n"); ok = false; }
```

这五条断言分别验证：三角形确实被削减了、存在高细节的近处 patch、存在低细节的远处 patch、地形覆盖了至少 1/4 的画面、图像亮度在合理范围。最终输出 `RESULT: PASS`，所有断言通过。

## 七、总结与延伸

### 7.1 本文做了什么

我们用约 320 行 C++ 从零实现了一个可运行的 Geomipmapping 地形 LOD 系统，完整覆盖了：fBm 高度图生成、patch 划分、距离分级 LOD 选择、基于"对齐网格"的裂缝消除、软光栅化渲染、以及一套量化验证断言。最终在几乎不损失视觉质量的前提下，把三角形从 131072 削减到 18392（削减 85.97%）。

### 7.2 技术局限性

诚实地说，本文的实现有几处明显的简化，生产级引擎不会这样做：

1. **距离分段代替屏幕空间误差**：本文用固定距离阈值 `40/90/150/220` 选 LOD，而严谨的做法是基于屏幕空间误差自适应选择。固定阈值在相机高度、FOV 变化时会失效（比如 FOV 拉大后，同样的距离投影到屏幕上更大，本该用更细的网格）。

2. **没有实现 morph 过渡**：本文用"对齐网格"消除了裂缝，但没有做顶点 morph 的平滑过渡。实际项目中，patch 在 LOD 级别切换的瞬间会有"跳变"（pop），玩家会看到山体突然变形。morph 可以让这个过渡平滑化。

3. **没有视锥剔除**：虽然留了 `patchMin/patchMax` 的数据结构，但没有真正实现视锥剔除，相机背后的 patch 也被白白渲染。

4. **软光栅化而非真实 GPU**：本文是 CPU 软光栅化，性能远不及 GPU。但架构上"CPU 生成 mesh + GPU 着色"的分离是清晰的，可以平滑迁移。

### 7.3 可继续优化的方向

- **屏幕空间误差 + 顶点 morph**：把固定距离阈值换成屏幕空间误差度量，再叠加 morph 过渡，就能得到视觉上无感知的 LOD。
- **视锥剔除 + 遮挡剔除**：利用 patch 包围盒做视锥剔除，进一步剔除被山体遮挡的 patch。
- **GPU instancing / 顶点纹理**：把高度图存成纹理，在 vertex shader 里采样获得高度，减少 CPU 侧的数据传输。
- **层次化 quadtree + 裂缝缝合**：升级到 Chunked LOD，用四叉树管理 patch 层级，配合更精细的裂缝缝合。

### 7.4 与本系列的关联

本系列一直在"软光栅化 + 量化验证"这条主线上推进。这篇地形 LOD 接续了之前几篇关于网格（Half-Edge 数据结构）、空间划分（BSP、四叉树）和软光栅化（齐次裁剪、扫描线填充）的积累，把"空间数据结构"和"软光栅化"两个主题在"地形渲染性能优化"这个具体场景里合流了。如果你对软光栅化的透视校正插值、z-buffer 算法还不熟，建议回看本系列关于 GPU 渲染管线的那些文章。

---

**完整代码**：[GitHub - daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/08-20-terrain-lod-geomipmapping)

**下一篇预告**：我们会继续在图形学的道路上前进，敬请期待。