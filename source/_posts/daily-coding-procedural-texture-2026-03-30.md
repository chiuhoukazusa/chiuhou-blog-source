---
title: "每日编程实践: Procedural Texture Synthesis — 程序化纹理合成"
date: 2026-03-30 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 程序化生成
  - 噪声
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-30-procedural-texture/procedural_texture_output.png
---

> **今日主题**：程序化纹理合成——用数学公式而非美术资产生成无限变化的有机纹理。  
> 核心技术：Worley Noise（细胞/Voronoi 噪声）+ Fractal Brownian Motion（分形布朗运动），生成大理石、木纹、熔岩、有机细胞六种纹理。

---

## ① 背景与动机

### 为什么需要程序化纹理？

传统美术流程中，纹理贴图是美术师手工绘制的图像文件——一张 2048×2048 的漫反射贴图占用 16MB（未压缩）。大型开放世界游戏中，仅地形纹理就可能高达几十 GB。这带来三个核心痛点：

**1. 存储瓶颈**  
《巫师 3》的纹理资产超过 30GB。玩家在下载时要忍受漫长等待，主机平台更是受限于蓝光盘容量。而程序化纹理只需存储几十行数学参数，运行时实时生成任意分辨率的纹理。

**2. 分辨率无关性**  
手绘纹理存在物理分辨率上限——放大 4 倍就会看到像素化。程序化纹理本质是连续函数，理论上可以无限放大，细节永远清晰。《No Man's Sky》的整个星球纹理系统就建立在这一特性上。

**3. 参数化变化**  
手绘大理石纹理要变色，需要美术师重新画一张。程序化纹理只需修改一个颜色参数，立刻生成新变体。这使得实时换装系统、动态环境变化成为可能。

### 工业界实际使用场景

- **Substance Designer**：Adobe 的程序化纹理创作工具，整个游戏行业的标准工作流。其底层节点正是 Worley、Perlin 等噪声函数的可视化编程。
- **虚幻引擎材质图**：UE 的 Material Editor 内置了 VoronoiNoise、MakeMaterialAttributes 等节点，直接对应本文实现的算法。
- **地形系统**：Unity Terrain 和 UE Landscape 使用 FBM（分形布朗运动）生成高度图，驱动地形起伏。
- **体积云/体积烟雾**：现代游戏的体积云（如《Horizon Zero Dawn》）将 Worley Noise 用于云朵内部的涡流细节，提供远近不同 LOD 的细节变化。

### 今天的目标

从零实现一个 CPU 程序化纹理合成器，核心算法：
- **Worley Noise（F1、F2-F1）**：生成细胞/Voronoi 图案
- **Perlin Noise + FBM**：生成平滑噪声，叠加多倍频
- 组合应用：大理石、木纹、熔岩、有机细胞纹理

输出：6 张 512×512 纹理 + 1 张 1568×1048 对比图。

---

## ② 核心原理

### 2.1 Worley Noise（细胞噪声/Voronoi 噪声）

#### 直觉来源

设想往水面扔了一把石子，每颗石子激起同心圆波纹，波纹交汇处形成 Voronoi 图——这就是 Worley Noise 的视觉直觉。1996 年 Steven Worley 在 SIGGRAPH 上提出此算法，专门用于模拟细胞状、鳞片状的有机纹理。

#### 算法定义

将 2D 空间划分成单位网格，在每个网格 `(i, j)` 内随机放置一个**特征点**（Feature Point）：

```
p(i,j) = (i + h₁(i,j), j + h₂(i,j))
```

其中 `h₁, h₂` 是映射到 [0,1) 的确定性哈希函数（伪随机）。

对于查询点 `x = (px, py)`，定义：

```
F₁(x) = min_{i,j} ‖x - p(i,j)‖₂    (到最近特征点的距离)
F₂(x) = second_min_{i,j} ‖x - p(i,j)‖₂    (到第二近特征点的距离)
```

**为什么只需检查周围 5×5 的格子？**  
在缩放系数 scale=6 下，特征点最远在格子角落，到下一个格子的最大距离约为 `√2 ≈ 1.414` 个格子单位。因此搜索半径 2 格（`dx,dy ∈ [-2,2]`）保证不会遗漏最近点。若只搜索 3×3 则在某些边角情况下会错误，导致纹理出现直线状裂缝。

#### F1 纹理：细胞内部渐变

```
texture = f(F₁)    // F₁ 越小，离特征点越近，越亮
```

反转后得到中心亮、边缘暗的细胞感。施加幂次 `F₁^0.5` 控制渐变曲线。

#### F2-F1 纹理：细胞边界

数学证明：在两个特征点的**等距中线**（Voronoi 边界）上，F₁ 和 F₂ 相等，因此 `F₂ - F₁ = 0`。在远离边界的细胞中心，`F₂ - F₁` 较大。

结论：`F₂ - F₁` 在**细胞边界处最小（接近0）**，在细胞中心最大。  
反转映射：`t = (F₂ - F₁) / 0.5` → 边界高亮，细胞内部暗。

这正是模拟骨骼、网状结构的关键。

#### 哈希函数设计

本实现使用 LCG（线性同余生成器）变体：

```cpp
uint32_t hash2(int32_t x, int32_t y) {
    uint32_t h = (uint32_t)(x * 1664525u + 1013904223u) ^ (uint32_t)(y * 22695477u + 1u);
    h ^= h >> 16;
    h *= 0x45d9f3b;
    h ^= h >> 16;
    return h;
}
```

**为什么用整数哈希而不是随机数表（permutation table）？**  
经典 Perlin Noise 使用 256 个元素的置换表。优点是历史兼容；缺点是会在坐标 `i mod 256 = 0` 处产生重复，纹理在 256 个单位处形成明显的 Tile 边界。整数哈希对任意整数坐标直接计算，完全无 Tile 限制，适合大规模地形生成。

异或混淆（`h ^= h >> 16; h *= 0x45d9f3b`）等价于 Murmur Hash 的最终化步骤，目的是打散低位 pattern，使输出的各位相互独立（通过雪崩效应测试）。

### 2.2 Perlin Noise

#### 梯度噪声的直觉

Pure value noise 在网格点处线性插值，会产生明显的"格块感"。Perlin Noise 的改进：在每个网格顶点不存储一个随机值，而是一个随机**梯度向量**，通过点积转换为标量，再插值。

对于整数格点 `(i,j)`，梯度向量 `g(i,j)` 从8个方向中选取：
```
{(1,0), (-1,0), (0,1), (0,-1), (0.707,0.707), (-0.707,0.707), (0.707,-0.707), (-0.707,-0.707)}
```

查询点 `(px, py)` 到格点 `(i,j)` 的贡献：
```
v = g(i,j) · (px-i, py-j)
```

这个点积有直觉意义：当查询点沿梯度方向移动时，贡献值增大；反方向则减小。最终在四个格点之间双线性插值，得到平滑连续的值。

#### Smoothstep 插值（Ken Perlin 2002 改进）

普通线性插值在格点处的一阶导数不连续（折角），导致纹理呈现格块感。改进方案用五次多项式代替：

```
f(t) = 6t⁵ - 15t⁴ + 10t³
```

**数学推导**：此多项式满足 `f(0)=0, f(1)=1, f'(0)=0, f'(1)=0, f''(0)=0, f''(1)=0`，即在端点处一阶和二阶导数均为零——二阶连续，消除了格块状伪影。

#### FBM：分形布朗运动

单层 Perlin Noise 只有一种频率的细节，视觉上过于平滑。FBM 通过**倍频叠加**添加多层细节：

```
fbm(x, y) = Σ_{i=0}^{N-1} amplitude_i × perlin(x × freq_i, y × freq_i)
```

其中：
- `freq_i = 2^i`（每层频率翻倍，即细节更细）
- `amplitude_i = (1/2)^i`（每层振幅减半，即细节更弱）
- Hurst 指数 H=0.5，对应随机游走模型

**直觉**：山脉的整体起伏是低频项，山坡的岩石纹路是中频项，细微划痕是高频项。FBM 的分形特性使其在任意放大倍率下都保持视觉丰富性，这正是真实自然界的标度不变性（scale invariance）。

本实现使用 6 个 octave，频率 `1→2→4→8→16→32`，振幅 `0.5→0.25→0.125→0.0625→0.03125→0.015625`，总振幅约 `1.0`（几何级数求和）。

### 2.3 大理石纹理：正弦扰动

大理石的核心特征是**流动的层状纹脉**，数学上对应扰动的正弦条纹：

```
marble(u,v) = sin((u × freq + fbm(u×4, v×4) × distort) × π)
```

**直觉**：`u × freq` 创建竖直条纹，`fbm × distort` 对条纹进行非线性扭曲——就像流体动力学中对流线的变形。扰动强度 `distort` 越大，纹脉越曲折，越像真实大理石中的矿物流动痕迹。

幂次映射 `marble^0.7` 压缩暗部，扩展亮部，使纹脉线条更细、更清晰（像真实大理石的细纹而非色块）。

### 2.4 木纹纹理：同心圆 + 噪声扰动

木纹本质是树木的年轮——以树干截面中心为原点的同心圆：

```
ring = (dist_from_center + fbm(u×3, v×3)×0.3) × 15
wood = sin(ring × π)
```

`fbm × 0.3` 是关键的不规则性扰动：树木生长时受到不均匀日照、降水影响，年轮并非完美同心圆。扰动幅度 0.3（相对于半径）模拟轻度变形，视觉上真实但仍保持年轮的整体感。

乘以 15 控制年轮密度——数值越大，同等大小纹理中年轮越多，模拟生长缓慢的老木（密纹）；数值小则模拟快生木材（宽纹）。

---

## ③ 实现架构

### 整体数据流

```
UV坐标 (u,v) ∈ [0,1]²
        ↓
  纹理函数 texture(u, v)
   ├─ Worley F1  → worley_f1.png
   ├─ Worley F2-F1  → worley_edge.png
   ├─ Marble (Perlin FBM + sin)  → marble.png
   ├─ Wood (FBM + concentric rings)  → wood.png
   ├─ Lava (Worley + FBM)  → lava.png
   └─ Organic (Worley + FBM)  → organic.png
        ↓
  Image::setPixel → std::vector<uint8_t> RGB buffer
        ↓
  stbi_write_png → .png 文件
        ↓
  网格拼接 → procedural_texture_output.png
```

### 关键数据结构

**Image 结构**：最简单的线性缓冲区，row-major 存储。避免使用动态分配的 2D 数组（二次间接寻址影响 cache 效率）。

```cpp
struct Image {
    int width, height;
    std::vector<uint8_t> data;  // RGB, row-major
    
    Image(int w, int h) : width(w), height(h), data(w * h * 3, 0) {}
    
    void setPixel(int x, int y, Vec3 col) {
        // 边界检查防止越界写
        if (x < 0 || x >= width || y < 0 || y >= height) return;
        int idx = (y * width + x) * 3;
        data[idx+0] = (uint8_t)std::clamp((int)(col.r * 255.0f), 0, 255);
        // ...
    }
};
```

**TextureFunc 函数指针**：将六种纹理统一为 `Vec3(*)(float, float)` 签名，可以存入 `vector<TextureInfo>` 统一循环处理，避免为每种纹理写重复的渲染循环。

**WorleyResult 结构体**：同时返回 F1 和 F2，避免两次调用 worley 函数（F1 和 F2 是一次计算自然得到的副产品）：

```cpp
struct WorleyResult {
    float f1, f2;
};
```

使用 C++17 结构化绑定简化调用：`auto [f1, f2] = worley(u, v, 6.0f);`

### Y 轴约定

渲染循环中显式进行 Y 轴翻转：
```cpp
float v = 1.0f - (float)y / (float)h;  // 翻转 Y 轴
```

图像坐标系（像素）：Y 轴向下。纹理 UV 空间惯例：V=0 在下方，V=1 在上方。不翻转会导致木纹的年轮在屏幕上倒置（中心偏向左下而非中间），大理石纹脉方向反转。

---

## ④ 关键代码解析

### 4.1 Worley 噪声核心实现

```cpp
WorleyResult worley(float px, float py, float scale = 4.0f) {
    float sx = px * scale;  // 缩放到噪声空间
    float sy = py * scale;
    
    int ix = (int)std::floor(sx);  // 当前格子坐标（注意：不能用 (int)sx，负数会向零取整）
    int iy = (int)std::floor(sy);

    float f1 = 1e9f, f2 = 1e9f;  // 初始化为极大值

    for (int dy = -2; dy <= 2; ++dy) {
        for (int dx = -2; dx <= 2; ++dx) {
            int cx = ix + dx;
            int cy = iy + dy;
            Vec2 cp = cellPoint(cx, cy);  // 格子内的随机偏移（[0,1)²）
            float ox = (float)cx + cp.x;  // 特征点在噪声空间的绝对坐标
            float oy = (float)cy + cp.y;
            float ddx = ox - sx;
            float ddy = oy - sy;
            float dist = std::sqrt(ddx*ddx + ddy*ddy);
            // 维护有序的最近两个距离
            if (dist < f1) { f2 = f1; f1 = dist; }
            else if (dist < f2) { f2 = dist; }
        }
    }
    // 归一化到约 [0,1]
    return { std::min(f1 / 0.7f, 1.0f), std::min(f2 / 1.0f, 1.0f) };
}
```

**关键细节——为什么用 `std::floor` 而不是强制转换 `(int)`？**  
C++ 的 `(int)(-0.3)` 结果是 `0`（向零取整），而我们需要 `-1`（向下取整）。如果 UV 坐标超出 [0,1] 范围（Tile 化或边界外采样），格子坐标会错误，导致纹理在负坐标区域产生不连续跳变。`std::floor` 保证向下取整，数学上正确。

**归一化系数选择**：`f1 / 0.7f` 中的 0.7 是经验值。在 scale=6 的设置下，F₁ 的期望值（平均到最近特征点的距离）约为 0.35，最大值约为 0.7（特征点在格子角落时）。除以 0.7 使 F₁ 映射到约 [0,1]，超出部分 `clamp` 到 1。

### 4.2 确定性哈希函数

```cpp
inline uint32_t hash2(int32_t x, int32_t y) {
    // 两个独立的 LCG 混合，再做雪崩处理
    uint32_t h = (uint32_t)(x * 1664525u + 1013904223u) ^ (uint32_t)(y * 22695477u + 1u);
    h ^= h >> 16;
    h *= 0x45d9f3b;
    h ^= h >> 16;
    return h;
}

inline Vec2 cellPoint(int32_t cx, int32_t cy) {
    // 使用不同的参数顺序确保 x 和 y 方向的随机性独立
    return {rand2f(cx, cy), rand2f(cy, cx + 7919)};  // 7919 是质数，避免 x==y 时退化
}
```

**为什么 `rand2f(cy, cx + 7919)` 而不是 `rand2f(cx+1, cy)`？**  
如果只是简单移位，在 `cx=cy` 的对角线上，两个随机数可能产生相关性（相同的哈希输入路径）。加上质数 7919 打破了这种对称性，使 x 和 y 方向的偏移量在统计上独立，避免纹理沿对角线产生对称伪影。

### 4.3 Perlin Noise 实现

```cpp
float perlin(float px, float py) {
    int ix = (int)std::floor(px);
    int iy = (int)std::floor(py);
    float fx = px - ix;  // 局部小数坐标 [0,1)
    float fy = py - iy;
    float ux = smoothstep(fx);  // 六次平滑插值权重
    float uy = smoothstep(fy);

    // 四个格点的梯度贡献
    auto dot_grad = [&](int cx, int cy, float dx, float dy) {
        Vec2 g = gradient(cx, cy);
        return g.x * dx + g.y * dy;
    };

    float v00 = dot_grad(ix,   iy,   fx,   fy  );   // 左下角
    float v10 = dot_grad(ix+1, iy,   fx-1, fy  );   // 右下角（注意 dx = fx-1，距离为负）
    float v01 = dot_grad(ix,   iy+1, fx,   fy-1);   // 左上角
    float v11 = dot_grad(ix+1, iy+1, fx-1, fy-1);   // 右上角

    // 双线性插值（使用 smoothstep 权重）
    float a = v00 + ux * (v10 - v00);  // 下边插值
    float b = v01 + ux * (v11 - v01);  // 上边插值
    return a + uy * (b - a);           // 上下插值
}
```

**为什么 `dot_grad(ix+1, iy, fx-1, fy)` 的第三个参数是 `fx-1` 而不是 `fx`？**  
点积 `g · (query - gridpoint)`。对于右下格点 `(ix+1, iy)`，查询点到该格点的偏移向量是 `(px - (ix+1), py - iy) = (fx-1, fy)`。`fx-1` 是负数（因为 `fx ∈ [0,1)`），表示查询点在格点左侧，梯度向左时贡献为正。这是 Perlin Noise 产生平滑过渡而非锯齿的数学关键。

### 4.4 大理石纹理着色

```cpp
Vec3 textureMarble(float u, float v) {
    float noise = fbm(u * 4.0f, v * 4.0f, 6);  // [-1, 1] 范围的 FBM 扰动
    
    // 正弦条纹 + 噪声扰动
    float marble = std::sin((u * 6.0f + noise * 4.0f) * 3.14159f);
    marble = marble * 0.5f + 0.5f;  // [-1,1] → [0,1]
    marble = std::pow(marble, 0.7f); // 亮化：γ < 1 压暗反而扩展中高调，使亮纹更宽

    // 三色混合：白色基底 + 灰色矿脉 + 深色纹线
    Vec3 white(0.95f, 0.93f, 0.9f);
    Vec3 vein(0.3f, 0.25f, 0.2f);
    Vec3 darkVein(0.1f, 0.08f, 0.06f);

    float t1 = marble;
    float t2 = std::pow(marble, 3.0f);  // 三次方使深色更集中（细线感）
    Vec3 col = Vec3::lerp(white, vein, t1 * 0.6f);
    col = Vec3::lerp(col, darkVein, t2 * 0.4f);
    return col;
}
```

**双层着色的视觉效果**：第一层 `lerp(white, vein, t1*0.6)` 产生白色底色中的灰色大区域；第二层 `lerp(col, darkVein, t2*0.4)` 在灰色中叠加深色细线。`t2 = t1^3` 使深色只出现在 `marble` 值高的地方（即正弦峰值处），形成细而集中的黑色纹脉——这是真实卡拉拉大理石的视觉特征。

### 4.5 熔岩纹理的双梯度着色

```cpp
Vec3 textureLava(float u, float v) {
    auto [f1, f2] = worley(u, v, 5.0f);
    float t = std::pow(f1, 1.5f);  // 幂次拉伸：让热点更集中（中心更亮）
    float detail = fbm(u * 8.0f, v * 8.0f, 4) * 0.15f + 0.15f;  // 高频细节 [0, 0.3]
    t = std::clamp(t + detail, 0.0f, 1.0f);

    Vec3 hot(1.0f, 0.9f, 0.2f);   // 亮黄（高温核心）
    Vec3 warm(0.9f, 0.3f, 0.05f); // 橙红（中温区域）
    Vec3 cool(0.1f, 0.02f, 0.0f); // 暗红（冷却边缘）
    
    // 分段线性着色：0-0.5 热→温，0.5-1.0 温→冷
    if (t < 0.5f) return Vec3::lerp(hot, warm, t * 2.0f);
    else          return Vec3::lerp(warm, cool, (t - 0.5f) * 2.0f);
}
```

**为什么用 `f1` 而不是反转 `f1`？**  
直觉：距离特征点越远（`f1` 越大），温度越低（颜色越暗红）。特征点本身（`f1≈0`）是熔岩最热的气泡中心，映射到亮黄色。加入 FBM 细节使每个气泡的热点分布略有不规则，避免过于均匀的机械感。

### 4.6 网格对比图拼接

```cpp
// 直接在目标 Image 上按行逐像素复制
for (int i = 0; i < (int)rendered.size(); ++i) {
    int col = i % COLS;
    int row = i / COLS;
    int ox = MARGIN + col * (W + MARGIN);  // 目标起始像素坐标
    int oy = MARGIN + row * (H + MARGIN);
    
    const Image& src = rendered[i];
    for (int y = 0; y < H; ++y) {
        for (int x = 0; x < W; ++x) {
            int si = (y * src.width + x) * 3;  // 源像素 RGB 偏移
            Vec3 c(src.data[si]/255.0f, src.data[si+1]/255.0f, src.data[si+2]/255.0f);
            grid.setPixel(ox + x, oy + y, c);
        }
    }
}
```

背景初始化为深灰（`for (auto& px : grid.data) px = 32;`），使间距区域成为 `rgb(32,32,32)`，与纹理形成视觉分割，而不是生硬的白色或黑色边框。

---

## ⑤ 踩坑实录

### Bug 1：负坐标下格子索引错误（向零取整 vs 向下取整）

**症状**：当 UV 坐标扩展到 [0,1] 范围外（测试 Tile 化时），靠近 u=0 左侧区域出现一条垂直的不连续线，细胞在此处出现明显跳变。

**错误假设**：以为 `(int)px` 等价于 `floor(px)`。

**真实原因**：C++ 中 `(int)(-0.1) = 0`（向零取整），而 `floor(-0.1) = -1`（向下取整）。格子坐标 0 而非 -1，导致在负数格子区域搜索范围覆盖错误，遗漏了真正的最近特征点。

**修复**：所有格子坐标计算一律使用 `(int)std::floor(px)`。

```cpp
// 错误：int ix = (int)sx;        // 负数时向零取整
// 正确：int ix = (int)std::floor(sx);  // 始终向下取整
```

### Bug 2：F2 不总是比 F1 大（初始化逻辑错误）

**症状**：某些区域 Worley Edge 纹理（F2-F1）出现负值，渲染出奇怪的黑色斑块。

**错误假设**：认为循环总会先找到 F1 再找到 F2，因此 `f2 > f1` 始终成立。

**真实原因**：循环遍历顺序是任意的。如果第一个遇到的点成为 F2，而第二个成为 F1，逻辑是正确的；但如果只遇到一个点（极端情况），F2 保持初始值 1e9，计算 `F2-F1` 会得到极大正值，渲染为亮点而非错误。真正的 Bug 是：条件 `else if (dist < f2)` 中若 dist == f1（极小概率），f2 不会更新，但实际上这种情况在浮点运算中不可能精确相等，所以此路径安全。

**实际发现的真正问题**：`f2 = f1; f1 = dist;` 这行的顺序是关键——必须**先保存旧 f1 到 f2**，再更新 f1，否则 f2 会被覆盖为与 f1 相同的值。当时写成 `f1 = dist; f2 = f1;` 导致 f2 总等于 f1，F2-F1 恒为 0，全黑。

**修复**：
```cpp
// 错误：f1 = dist; f2 = f1;   // f2 会等于新的 f1！
// 正确：f2 = f1; f1 = dist;   // 先保存，再更新
if (dist < f1) { f2 = f1; f1 = dist; }
else if (dist < f2) { f2 = dist; }
```

### Bug 3：木纹中心偏移导致年轮不在中央

**症状**：木纹的同心圆圆心在图像左下角，不是中央。

**错误假设**：UV 坐标 (0,0) 在图像中央。

**真实原因**：UV (0,0) 实际上在图像角落（左下）。代码中 `dist = sqrt(u² + v²)` 以 (0,0) 为圆心，自然偏角。

**修复**：以中心为圆心，偏移 UV 坐标：
```cpp
// 错误：float dist = std::sqrt(u*u + v*v);
// 正确：
float cx = u - 0.5f, cy = v - 0.5f;
float dist = std::sqrt(cx*cx + cy*cy);
```

### Bug 4：大理石纹理过暗（gamma 方向混淆）

**症状**：大理石纹理整体偏暗，纹脉几乎看不见。

**错误假设**：`pow(marble, 2.0)` 会让纹理更亮（认为幂次是亮化操作）。

**真实原因**：对于 `t ∈ [0,1]`，`t^n (n>1)` 是**暗化**操作（0→0, 1→1 但中间值被压低）。要亮化需要 `t^(1/n)` 即 `n<1`。

**修复**：将 `pow(marble, 2.0)` 改为 `pow(marble, 0.7)`，使中间调亮化，让白色纹脉区域更宽、更明显。

---

## ⑥ 效果验证与数据

### 生成文件

```
procedural_texture_output.png  1568×1048   1.8MB（对比图）
marble.png                      512×512    376KB
wood.png                        512×512    290KB
organic.png                     512×512    283KB
lava.png                        512×512    259KB
worley_f1.png                   512×512    260KB
worley_edge.png                 512×512    194KB
```

### 像素统计验证

运行 Python 像素采样脚本（参见 SKILL.md 中的输出验证步骤）：

```
✅ lava.png:     512×512, mean=78.3,  std=81.7   [正常：明暗对比强烈]
✅ marble.png:   512×512, mean=159.4, std=51.8   [正常：白色为主，偏亮]
✅ organic.png:  512×512, mean=92.7,  std=53.6   [正常：绿色中调]
✅ wood.png:     512×512, mean=122.1, std=51.6   [正常：棕色中调]
✅ worley_f1.png: 512×512, mean=54.5, std=38.7  [正常：深色为主（细胞中心较暗）]
✅ worley_edge.png: 512×512, mean=74.6, std=77.7 [正常：边缘亮、内部暗，标准差大]
```

所有纹理满足：
- ✅ 均值在 10~240 之间（非全黑/全白）
- ✅ 标准差 > 5（图像有内容变化）
- ✅ 文件大小 > 10KB

### 视觉对比

![程序化纹理合成对比图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-30-procedural-texture/procedural_texture_output.png)

*从左到右：Worley F1 细胞 / Worley 边缘 / 大理石 | 木纹 / 熔岩气泡 / 有机细胞*

各纹理的视觉特征验证：
- **Worley F1**：清晰的蓝紫色 Voronoi 细胞，中心亮、边缘暗 ✅
- **Worley Edge**：白色网状边界，细胞内部深绿色 ✅
- **大理石**：白色底色上的弯曲灰色/深色纹脉 ✅
- **木纹**：以中心为圆心的同心圆年轮，棕色调 ✅
- **熔岩**：橙黄色气泡点，向外渐变为暗红，有 FBM 不规则细节 ✅
- **有机细胞**：绿色细胞，亮色核心，深色细胞壁 ✅

### 性能数据

在单线程 CPU 上（x86-64，无 SIMD 优化）：

- 512×512 单张纹理渲染：约 120ms（Worley 类，含 5×5 邻域搜索）
- 512×512 FBM 纹理（6 octave）：约 80ms（Perlin 类）
- 6 张 + 1 张对比图总计：约 1.2s

性能瓶颈分析：Worley 的 `sqrt` 调用是主要消耗。优化方向：
1. 用平方距离比较（避免 sqrt），只在返回时开方
2. SIMD：AVX2 一次处理 8 个格子
3. 多线程：按行划分，6 张纹理并行生成

---

## ⑦ 总结与延伸

### 技术局限性

**1. 纯 CPU 实现，无法实时**  
生成一张 512×512 需要 100ms+ 在 CPU 上，4K 纹理则需要数秒。这对于游戏实时渲染是不可接受的。真实引擎（UE/Unity）在 GPU Shader 中计算，利用并行计算实现实时程序化纹理。

**2. 无 Tiling 控制**  
目前的 FBM 实现在大范围 UV（超出 [0,1]）时自然 Tile，但视觉上没有跳变（因为哈希函数是确定性的）。然而对于某些场景，需要精确的无缝 Tile（toroidal topology），需要特殊处理边界的梯度或使用域翘曲（domain warping）技术。

**3. 2D 噪声的局限**  
3D 体积纹理（如云彩、烟雾、岩石内部）需要 3D Worley/Perlin，计算量是 2D 的若干倍。

### 可优化/延伸方向

**1. 域翘曲（Domain Warping）**——Inigo Quilez 技术：
```glsl
vec2 q = vec2(fbm(uv), fbm(uv + 0.5));
vec2 r = vec2(fbm(uv + 4.0*q + 1.7), fbm(uv + 4.0*q + 9.2));
float f = fbm(uv + 4.0*r);
```
对 FBM 的输入坐标进行 FBM 扰动，产生极度有机的卷曲流体感，被用于《星际争霸 2》的星云材质。

**2. 3D Worley Noise → 体积云**  
将今天的 2D Worley 扩展到 3D，用于体积云渲染（Guerrilla Games 的《Horizon》团队分享了具体实现）。F1 控制云朵内部的密度场，F2-F1 控制云层边缘的蓬松感。

**3. Tileable 程序化纹理**  
使用 stochastic tiling（随机 Tile）算法，在保持视觉随机性的同时消除 Tile 接缝——NVIDIA 2019 年发表的技术，现已进入 UE 5.x 材质节点。

**4. 结合今日系列**  
- 与 03-24「次表面散射」结合：用 Worley 噪声生成皮肤散射半径的空间变化
- 与 03-28「景深」结合：用 FBM 噪声作为焦散贴图，模拟水面焦散
- 与 03-27「运动模糊」结合：用程序化纹理替代硬编码颜色，使场景视觉更丰富

### 与本系列其他文章的关联

| 日期 | 技术 | 关联 |
|------|------|------|
| 02-10 | Perlin Noise | 今日 FBM 的基础 |
| 03-21 | SPPM | 焦散渲染可用 Worley 噪声强化细节 |
| 03-24 | SSS | 皮肤纹理可用今日有机细胞纹理 |
| 03-29 | HDR 色调映射 | 程序化纹理需要正确的 Gamma 处理 |

---

### 代码仓库

完整源码：[GitHub - Procedural Texture Synthesis](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-30-procedural-texture)

```bash
git clone https://github.com/chiuhoukazusa/daily-coding-practice.git
cd 2026/03/03-30-procedural-texture
g++ main.cpp -o output -std=c++17 -O2
./output
# 生成 6 张纹理 + 1 张对比图
```

---

*每日编程实践第 N 天 · 图形学系列 · 2026-03-30*
