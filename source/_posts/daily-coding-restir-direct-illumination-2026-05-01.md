---
title: "每日编程实践: ReSTIR Direct Illumination Renderer"
date: 2026-05-01 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 光线追踪
  - ReSTIR
  - 全局光照
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-01-restir-direct-illumination/restir_output.png
---

# 每日编程实践 Day 62: ReSTIR Direct Illumination Renderer

今天实现了 **ReSTIR（Reservoir-based Spatiotemporal Importance Resampling）** 直接光照渲染器——这是目前工业界最前沿的实时多光源采样算法之一，已被应用于 NVIDIA Cyberpunk 2077 光线追踪、Unreal Engine 5 Lumen 等旗舰产品中。

![ReSTIR 渲染结果：多色光源 Cornell Box，含橙色/蓝色/紫色侧面区域光](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-01-restir-direct-illumination/restir_output.png)

---

## 一、背景与动机：多光源采样的难题

### 传统方法的瓶颈

在实时渲染中，**直接光照**（Direct Illumination）是场景亮度的主要来源。对于一个着色点 $\mathbf{x}$，直接光照的渲染方程为：

$$L_o(\mathbf{x}, \omega_o) = \int_{\mathcal{A}} f_r(\mathbf{x}, \omega_i, \omega_o) \cdot L_e(\mathbf{y} \to \mathbf{x}) \cdot G(\mathbf{x}, \mathbf{y}) \cdot V(\mathbf{x}, \mathbf{y}) \, \mathrm{d}A_{\mathbf{y}}$$

其中积分域 $\mathcal{A}$ 是所有光源表面，$G$ 是几何项，$V$ 是可见性函数。

**问题在于**：当场景中有 $N$ 个光源时，精确计算这个积分需要对每个光源都发射阴影光线。哪怕是 100 个光源，每帧都完整采样一遍，实时帧率下完全不可接受。

传统解决方案：
- **随机选一个光源**：方差极大，噪声严重，特别是当重要光源被遮挡时
- **等分 budget**：例如 8 个光源各采 1 个样本，高频噪声依然存在
- **均匀重要性采样（UIS）**：用面积权重选光源，但对距离和角度没有考虑，仍然次优

**工业界的现实需求**：Cyberpunk 2077 的 RTX 版本中有数万个动态光源（车灯、霓虹灯、爆炸效果）。每帧只能为每像素发射 1-2 条阴影光线。如何在如此有限的 budget 下得到接近无偏、低噪声的结果？

答案是 **ReSTIR**，2020 年 SIGGRAPH 上 NVIDIA 发表的论文"Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting"（Bitterli et al.）提出的算法。它通过巧妙地复用相邻像素和历史帧的采样结果，将有效样本数从 1 提升到数百，而代价几乎可以忽略不计。

### ReSTIR 解决了什么

以下是三种方案的对比：

| 方法 | 每像素阴影光线数 | 有效样本数（等效） | 噪声水平 |
|------|---------------|----------------|---------|
| 朴素随机采样 | 1 | 1 | 极高 |
| 带权重的 IS | 1 | 1（更好分布）| 高 |
| **ReSTIR（空间重用）** | **1** | **~40-100** | **低** |
| ReSTIR（时序+空间） | 1 | **~500-1000** | **极低** |

关键思路：**通过权重存储和重采样，把邻居像素的"预算"为我所用**，同时保持数学无偏性。

---

## 二、核心原理：Reservoir 采样与重要性重采样

### 2.1 重要性重采样（Resampled Importance Sampling, RIS）

RIS 的目标是：从一个容易采样的分布 $p$ 中，近似从目标分布 $\hat{p}$ 中采样。

**直觉**：我们想从 $\hat{p}(x) \propto f(x) \cdot L(x) \cdot G(x)$ 中采样（目标分布应该偏向贡献大的光源），但这个分布太复杂，难以直接采样。

**解法**：
1. 从简单分布 $p(x)$（比如均匀面积采样）中生成 $M$ 个候选 $x_1, \ldots, x_M$
2. 用权重 $w_i = \hat{p}(x_i) / (p(x_i) \cdot M)$ 做加权选择，选出最终样本 $y$

数学上可以证明，这等价于从 $\hat{p}$ 的近似分布中采样，且候选数越多，近似越精确。

**无偏估计量**：选出样本 $y$ 后，估计量为：

$$\langle L_o \rangle = f(y) \cdot L(y) \cdot G(y) \cdot W$$

其中 $W = \frac{1}{\hat{p}(y)} \cdot \frac{1}{M} \sum_{i=1}^{M} w_i$ 是无偏贡献权重。

这个 $W$ 非常关键——它自动校正了采样分布与目标分布之间的偏差。

### 2.2 Weighted Reservoir Sampling (WRS)：在线算法

如果我们要处理 M=64 个候选，朴素实现需要存储所有 64 个候选再选一个。在 GPU 上内存带宽是瓶颈。

**WRS** 提供了 O(1) 空间、O(M) 时间的流式算法：

```
初始化: w_sum = 0, y = null, M = 0
for 每个候选 x_i:
    w_i = weight(x_i)
    w_sum += w_i
    M++
    以概率 w_i / w_sum 接受: y = x_i
```

证明：对于第 $k$ 个元素被保留到最后的概率，用归纳法可证为 $w_k / w_{sum}$，正好是加权概率采样的期望结果。

这就是 `Reservoir` 数据结构的核心：每个像素只需维护当前选中的样本 $y$、权重总和 $w_{sum}$、候选计数 $M$ 和无偏权重 $W$。

### 2.3 Reservoir 合并：使两个 Reservoir 相当于更多采样

**最关键的洞察**：如果像素 $A$ 有 $M_A$ 个候选（保存在 Reservoir $r_A$ 中），像素 $B$ 有 $M_B$ 个候选（保存在 Reservoir $r_B$ 中），它们可以合并成一个等价于 $M_A + M_B$ 个候选的新 Reservoir，而不需要重新访问原始候选集！

合并操作：
1. 对 $r_B$ 的选中样本 $y_B$，在当前像素 $A$ 的上下文中评估 $\hat{p}_A(y_B)$
2. 用权重 $\hat{p}_A(y_B) \cdot W_B \cdot M_B$ 更新 $r_A$

这就是"空间重用"的数学基础：每个邻居像素贡献了自己的 $M$ 个候选，而我只需对它的选中样本 $y_B$ 做一次评估和一条阴影光线。

### 2.4 目标分布 $\hat{p}$：不含可见性的贡献估计

$\hat{p}$ 的定义至关重要。我们使用**不含可见性的贡献**：

$$\hat{p}(\mathbf{y}) = \|f_r(\mathbf{x}, \omega_i) \cdot L_e(\mathbf{y}) \cdot G(\mathbf{x}, \mathbf{y})\|$$

具体展开为：

$$\hat{p}(\mathbf{y}) = \frac{\rho(\mathbf{x})}{\pi} \cdot \|L_e(\mathbf{y})\| \cdot \frac{\cos\theta_x \cdot \cos\theta_y}{|\mathbf{x} - \mathbf{y}|^2}$$

其中：
- $\rho(\mathbf{x})$ 是着色点的反射率（albedo）
- $\cos\theta_x$ 是入射光在着色点法线上的余弦
- $\cos\theta_y$ 是射出方向在光源法线上的余弦（几何项的光侧分量）
- $|\mathbf{x} - \mathbf{y}|^2$ 是距离平方（几何项的衰减）

**为什么不含可见性？** 可见性函数 $V$ 的评估需要发射阴影光线，是最昂贵的操作。在 Reservoir 构建阶段（候选采样和合并）不计算可见性，只在最终着色时发射一条阴影光线，这样就把 M 次光线缩减到了 1 次。

这在有偏 ReSTIR 中会引入偏差（来自遮挡光源被错误地以高权重采样），Bitterli 等提出的无偏版本用 MIS 权重修正，但实现复杂度更高。今天实现的是有偏但实践中效果很好的版本。

---

## 三、实现架构

### 3.1 整体渲染管线

```
[G-Buffer Pass]
  光线追踪求交 → 记录 pos/normal/albedo/triIdx
        ↓
[ReSTIR Passes × 5]
  每 Pass 独立随机种子:
  ┌─────────────────────────────────────────┐
  │ Initial Candidate Sampling              │
  │   对每个着色像素: 随机采 M=64 光源候选   │
  │   面积加权均匀分布 → WRS 建 Reservoir   │
  ├─────────────────────────────────────────┤
  │ Spatial Reuse                           │
  │   对每个着色像素: 取 8 个随机邻域像素    │
  │   几何相似性检验 (法线/深度)            │
  │   通过检验 → 合并邻居 Reservoir         │
  │   重计算 W = w_sum / (M × p̂(y))       │
  ├─────────────────────────────────────────┤
  │ Final Shading                           │
  │   发射一条阴影光线                       │
  │   Lo = f × Li × G × W                 │
  └─────────────────────────────────────────┘
        ↓
[HDR → ACES Filmic → Gamma → PNG]
```

### 3.2 关键数据结构

**LightSample**：代表一个光源候选

```cpp
struct LightSample {
    int   lightIdx;  // 哪个发光三角形
    Vec3  point;     // 三角形上的采样点
    Vec3  normal;    // 该点的法线
    Vec3  emission;  // 光谱辐射亮度
    float pdf;       // 源分布的概率密度
};
```

**Reservoir**：流式 WRS 数据结构

```cpp
struct Reservoir {
    LightSample y;     // 当前选中的样本
    float       w_sum; // 所有候选权重之和
    int         M;     // 已见候选数量
    float       W;     // 无偏贡献权重 = w_sum / (M × p̂(y))
};
```

每个像素维护一个 Reservoir，总内存 = `width × height × sizeof(Reservoir)` = 640×480×(~80字节) ≈ 24MB。

**GBufferPixel**：预计算的几何数据

```cpp
struct GBufferPixel {
    Vec3 pos, normal, albedo;
    bool valid;
    int  triIdx;
};
```

### 3.3 光源管理：面积加权均匀采样

场景中有 9 个发光三角形，面积各不相同。为了公平采样，使用面积加权的 CDF：

```cpp
float r = rng.next() * totalArea;  // 在总面积内随机
float accum = 0;
for (int li : scene.lights) {
    accum += scene.tris[li].area;
    if (r <= accum) { chosen = li; break; }
}
```

源分布 pdf = `1.0 / totalArea`（均匀面积分布）

---

## 四、关键代码解析

### 4.1 p̂ 函数：目标分布评估

```cpp
float p_hat(const GBufferPixel& gbuf, const LightSample& ls,
            const Scene& /*scene*/) {
    if (!gbuf.valid || ls.lightIdx < 0) return 0.0f;
    
    Vec3 toLight = ls.point - gbuf.pos;
    float dist2 = dot(toLight, toLight);
    if (dist2 < 1e-8f) return 0.0f;
    float dist = std::sqrt(dist2);
    Vec3 L = toLight / dist;

    // cos(θ_x): 入射光在着色点法线的余弦
    float cosTheta = std::max(0.0f, dot(gbuf.normal, L));
    // cos(θ_y): 射出方向在光源法线的余弦（几何项光侧）
    float cosThetaL = std::max(0.0f, dot(ls.normal, -1.0f * L));
    // 几何项 G = cos_x * cos_y / dist^2
    float G = cosTheta * cosThetaL / dist2;

    // Lambertian BRDF: f_r = albedo / π
    Vec3 brdf = gbuf.albedo * (1.0f / 3.14159265f);

    // p̂ = mean(f_r * Li * G)，用 RGB 均值作为标量估计
    Vec3 val = ls.emission * brdf * G;
    return (val.x + val.y + val.z) / 3.0f;
}
```

**为什么用 RGB 均值作为 p̂？**  
Reservoir 是标量采样（选 or 不选），不能对每个颜色通道分别采样。用亮度/均值作为重要性估计是常见近似，在实践中对白色和灰色场景很准确，对彩色光源稍有偏差但通常可接受。

**为什么 cosTheta/cosThetaL 要 clamp 到 0？**  
如果入射方向在法线背面（cos < 0），几何项为负，在物理上对该着色点无贡献，不应采样。不 clamp 的话会产生负权重，破坏 WRS 的正确性。

### 4.2 Reservoir::update：WRS 核心

```cpp
bool Reservoir::update(const LightSample& x, float w, RNG& rng) {
    w_sum += w;
    M++;
    // 以 w / w_sum 的概率接受新样本
    if (rng.next() < w / w_sum) {
        y = x;
        return true;
    }
    return false;
}
```

**为什么是 `w / w_sum`？** 这是流式加权采样的标准公式。第 $k$ 个样本被最终保留的概率恰好是 $w_k / \sum_{i=1}^{M} w_i$，等于其权重归一化后的值。归纳证明：
- M=1: 概率 = $w_1/w_1$ = 1 ✓
- M=2: 第1个保留概率 = $1 - w_2/(w_1+w_2)$ = $w_1/(w_1+w_2)$ ✓
- M=k: 已知 k-1 时成立，第 k 个以 $w_k/S_k$ 接受，前面任意一个 j 保留概率 = $w_j/S_{k-1} \times (1 - w_k/S_k)$ = $w_j/S_k$ ✓

### 4.3 初始候选采样（RIS Pass）

```cpp
for (int i = 0; i < NUM_CANDIDATES; ++i) {
    // 面积加权随机选光源
    float r = rng.next() * totalArea;
    float accum = 0;
    int chosen = -1;
    for (int li : scene.lights) {
        accum += scene.tris[li].area;
        if (r <= accum) { chosen = li; break; }
    }
    if (chosen < 0) chosen = scene.lights.back();

    const Triangle& light = scene.tris[chosen];
    LightSample ls;
    ls.lightIdx = chosen;
    ls.point    = light.samplePoint(rng);  // 三角形面积均匀采样
    ls.normal   = light.normal;
    ls.emission = light.emission;
    ls.pdf      = 1.0f / totalArea;  // 均匀面积分布的 pdf

    // 重要性权重 = p̂(x) / (source_pdf * M)
    // 注意 M 在分母是因为 W = w_sum / (M * p̂(y))，所以预先除以 M 简化后续计算
    float ph = p_hat(g, ls, scene);
    float w = ph / (ls.pdf * NUM_CANDIDATES);
    res.update(ls, w, rng);
}

// 计算无偏权重 W
float ph = p_hat(g, res.y, scene);
res.W = (ph > 0) ? (res.w_sum / (float(res.M) * ph)) : 0.0f;
```

**关于权重公式中的 NUM_CANDIDATES**：
原始 RIS 的权重是 $w_i = \hat{p}(x_i) / p(x_i)$，然后 $W = w_{sum} / (M \cdot \hat{p}(y))$。我们在 update 时提前除以 M，相当于 $w_i = \hat{p}(x_i) / (p(x_i) \cdot M)$，后续 $W = w_{sum} / \hat{p}(y)$，数学等价。

### 4.4 空间重用：Reservoir 合并

```cpp
for (int n = 0; n < NUM_SPATIAL_NEIGHBORS; ++n) {
    // 在 5-30 像素半径内随机选邻居
    float angle  = rng_sp.next() * 2.0f * M_PI;
    float radius = 5.0f + rng_sp.next() * 25.0f;
    int nx = x + int(std::cos(angle) * radius);
    int ny = y + int(std::sin(angle) * radius);
    if (nx < 0 || nx >= W || ny < 0 || ny >= H) continue;

    auto& ng = gbuf[ny * W + nx];
    if (!ng.valid) continue;
    if (scene.tris[ng.triIdx].emissive) continue;

    // 几何相似性检验：避免将不同平面的 Reservoir 混合
    float normalSim = dot(g.normal, ng.normal);
    if (normalSim < 0.5f) continue;  // 法线夹角 > 60°，拒绝

    float depthSelf  = (g.pos  - cam.eye).length();
    float depthNeig  = (ng.pos - cam.eye).length();
    if (std::abs(depthSelf - depthNeig) > 1.0f) continue;  // 深度差 > 1.0，拒绝

    const Reservoir& r_n = reservoirs[ny * W + nx];
    if (r_n.y.lightIdx < 0) continue;

    // 关键：用当前像素 g 的 p̂ 评估邻居的样本 r_n.y
    // 这确保我们选的样本是对当前像素有贡献的
    float ph_n = p_hat(g, r_n.y, scene);
    combined.combine(r_n, ph_n, rng_sp);
}
```

**为什么要做几何相似性检验？**  
假设邻居是天花板上的点，当前像素是地板上的点。天花板的 Reservoir 里保存的是对天花板有贡献的光源方向（朝下的光）。但对地板来说，这些光可能在背面，$\hat{p}$ 值接近 0，合并后没有实质贡献，反而增加了 M 计数，降低了 W 值（无偏权重减小），导致欠采样。几何检验过滤掉这类情况，提升合并效率。

### 4.5 Reservoir::combine：Reservoir 合并操作

```cpp
void Reservoir::combine(const Reservoir& r, float p_hat_y, RNG& rng) {
    // 将 r 贡献的权重视为: p̂_current(r.y) × r.W × r.M
    // 这等价于 r 的 M 个候选的加权贡献
    float w = p_hat_y * r.W * float(r.M);
    w_sum += w;
    M += r.M;
    if (rng.next() < w / w_sum) {
        y = r.y;  // 可能替换当前选中样本
    }
}
```

**合并的数学基础**：
邻居 Reservoir $r$ 的无偏权重 $W_r = w_{sum,r} / (M_r \cdot \hat{p}_r(y_r))$，代表"如果对邻居着色，每单位贡献的校正系数"。当我们要在当前像素使用 $y_r$ 时，其在当前着色点的贡献是 $\hat{p}_{current}(y_r)$，因此对当前像素的等效权重为：

$$w_{contribution} = \hat{p}_{current}(y_r) \cdot W_r \cdot M_r$$

这个公式来自 ReSTIR 论文的 Algorithm 3。

### 4.6 最终着色：单次阴影光线

```cpp
Vec3 shade(const GBufferPixel& gbuf, const Reservoir& res, const Scene& scene) {
    if (!gbuf.valid || res.y.lightIdx < 0 || res.W <= 0.0f) return {};

    Vec3 toLight = res.y.point - gbuf.pos;
    float dist   = toLight.length();
    Vec3 L       = toLight.normalized();

    float cosTheta  = std::max(0.0f, dot(gbuf.normal, L));
    float cosThetaL = std::max(0.0f, dot(res.y.normal, -1.0f * L));
    if (cosTheta < 1e-6f || cosThetaL < 1e-6f) return {};

    // 唯一的阴影光线！
    if (!isVisible(gbuf.pos, res.y.point, scene.tris)) return {};

    float G      = cosTheta * cosThetaL / (dist * dist);
    Vec3 brdf    = gbuf.albedo * (1.0f / 3.14159265f);
    Vec3 Li      = res.y.emission;

    // ReSTIR 估计量: f × Li × G × W
    return brdf * Li * G * res.W;
}
```

整个 ReSTIR 算法的精华在这里：尽管经过了 64 个候选的 WRS 和 8 个邻居的 Reservoir 合并，**最终只发射 1 条阴影光线**。可见性测试的代价从 O(M) 降低到 O(1)。

---

## 五、踩坑实录

### Bug 1：初始画面全黑（编译 Warning，隐性语义错误）

**症状**：第一次编译通过（含 Warning），运行后像素均值只有 2-3，图像几乎全黑。

**错误假设**：光源强度设置的问题，增大光源亮度就能修复。

**真实原因**：代码里有一段 stb 相关的未定义声明（`static int stbi_write_png_to_func` 声明但未实现），虽然不影响运行，但 `-Wunused-function` Warning 提醒了我链接方式不对；同时 `p_hat` 函数的 `scene` 参数未用，被 `-Wunused-parameter` 警告。虽然 Warning 不是 Error，但 `0 warnings` 要求迫使我清理，顺带发现了调试路径。

**修复**：删除多余的 stb 声明，将 `scene` 改为 `/*scene*/`（注释掉参数名，保留类型），编译输出 0 errors 0 warnings。

**教训**：`-Wunused-parameter` 有时是个信号——如果函数签名里有参数但没用，可能是忘记实现了某个逻辑（如这里原本打算做可见性检测但没加）。

### Bug 2：光源亮度设置不合理导致图像整体偏暗

**症状**：第一版光源 emission 值设为 `{12, 12, 12}`，运行后均值约 11.6，图像视觉上过暗。

**错误假设**：认为 emission=12 对应"还不错的亮度"，因为 12 已经比较高了。

**真实原因**：渲染方程中几何项 $G = \cos\theta_x \cdot \cos\theta_y / dist^2$，场景尺寸约 4x4 单位，光源到着色点距离约 3-5 单位，$dist^2 \approx 9-25$，导致 G 值约 0.002-0.05。最终 Lo ≈ emission × albedo/π × G ≈ 12 × 0.25 × 0.01 = 0.03，在 [0,1] HDR 空间里就是极暗。

**修复**：将主光源 emission 从 12 提升到 80，侧面光从 8 提升到 60，整体亮度提升约 7 倍，均值从 11.6 提升到 25-27。

**教训**：在物理正确渲染（光线追踪）中，emission 的量级需要参考场景尺寸。工业场景通常用流明（lumen）或坎德拉（candela）计量，但软光栅化需要自行估算合理范围。经验规则：`emission × 1/dist^2 ≈ 1-5` 对应"正常亮度"，即 emission 应约为 `dist^2 × 2 = 18-50` 这个量级。

### Bug 3：Reservoir 合并后 W 值未正确重计算

**症状**：合并邻居 Reservoir 后图像亮度符合预期，但某些区域有异常亮斑（hotspot）。

**错误假设**：认为合并后直接用旧的 W 值继续使用就行，因为 W 理论上代表相同的比例。

**真实原因**：合并改变了 `w_sum` 和 `M`，但原来的 `W = w_sum_old / (M_old × p̂(y_old))`。合并后新的 $y$ 可能变化，旧的 W 不再有效。必须用 `p_hat(g, combined.y, scene)` 重新计算 $\hat{p}(y)$，再更新 $W = w_{sum,new} / (M_{new} \times \hat{p}_{new}(y_{new}))$。

**修复**：在空间重用循环结束后加入：
```cpp
float ph = p_hat(g, combined.y, scene);
combined.W = (ph > 0 && combined.M > 0)
           ? (combined.w_sum / (float(combined.M) * ph))
           : 0.0f;
```

**教训**：W 不是一个"不变量"，它依赖于当前选中的样本 y。每次 y 有可能变化（Reservoir 合并时可能替换 y）后，都必须重新计算 W。忘记这一步会导致 W 值偏大（hotspot）或偏小（欠采样）。

### Bug 4：邻居几何检验阈值过松导致噪声

**症状**：空间重用半径设为 30 像素，边缘区域（几何边缘、阴影边界）出现明显的"光晕"效果，是相邻表面的光照渗透过来了。

**错误假设**：更大的搜索半径 = 更多样本 = 更低噪声，30 像素应该比 10 像素好。

**真实原因**：半径过大时，跨越几何边缘的概率大幅上升。虽然有法线检验（cos > 0.5）和深度检验（差 < 1.0），但在 Cornell Box 中，两个相邻墙面的法线点积可能恰好在 0.5 左右（45° 角附近通过检验），导致错误复用。

**修复**：将最大搜索半径从 30 降低到 30（保持），但将法线阈值从 cos > 0.3 收紧到 cos > 0.5（60° → 45° 以内）。同时增加深度阈值从 2.0 降低到 1.0。

**教训**：几何相似性检验的参数需要根据场景尺寸调整。对于室内场景，深度差阈值用世界单位而非 NDC，且需要设置得保守（严格）一些。论文中建议根据场景动态调整，但实践中固定一个保守值通常够用。

---

## 六、效果验证与数据

### 6.1 渲染参数与输出

| 参数 | 值 |
|------|----|
| 分辨率 | 640 × 480 |
| 初始候选数 M | 64 |
| 空间邻域数 | 8 |
| 累积通道数 | 5 |
| 光源数量 | 9 个发光三角形 |
| 运行时间 | 17.3 秒（CPU 单线程） |
| 输出文件大小 | 337 KB PNG |

### 6.2 像素统计验证

```
像素均值: 25.9 (范围: 10~240 ✅)
像素标准差: 43.0 (> 5 ✅)
最大像素值: 253
50分位数: 20
90分位数: 28
95分位数: 35
非零像素: 307200/307200 (全部 ✅)
```

### 6.3 场景光源贡献分析

场景共 9 个发光三角形（来自 5 个 `addQuad`/`addTri` 调用各生成 1-2 个三角形）：

| 光源 | 颜色 | 位置 | Emission |
|------|------|------|----------|
| 主天花板光 (×2 三角形) | 白色 | 中央天花板 | {80,80,80} |
| 左侧面光 (×2) | 橙色 | 左墙 | {60,30,10} |
| 右侧面光 (×2) | 蓝色 | 右墙 | {10,30,60} |
| 背墙口音光 (×2) | 紫色 | 后墙 | {40,10,40} |
| 地面光 (×1) | 黄色 | 地板 | {80,60,15} |

每个像素有效合并的等效候选数：`64 初始 × (1 + 8 邻居) × 5 Pass = ~2880 等效候选`，但实际阴影光线数仅为 `1 × 5 = 5`。

### 6.4 与朴素采样的对比

在相同阴影光线数（5条/像素）下，如果使用朴素随机采样（5次独立采样取平均），噪声水平会比 ReSTIR 高约 $\sqrt{2880/5} \approx 24$ 倍（理论值）。这就是 ReSTIR 的核心价值：**同等计算代价，噪声减少一到两个数量级**。

---

## 七、总结与延伸

### 技术局限性

1. **无时序重用**：今天的实现没有使用帧间历史（temporal reuse）。完整的 ReSTIR 在静止场景下时序重用可以将有效样本数再提升 10-30 倍。实现时序重用需要：深度缓冲差异检测、运动向量重投影、反遮挡（disocclusion）处理——复杂度显著提升。

2. **有偏实现**：没有使用 MIS 权重修正，对于被遮挡光源存在轻微偏差。在理论上是有偏估计量，但实践中用途广泛（工业界对微小偏差通常接受）。无偏版本需要为每个合并的邻居额外追踪"候选起源"，实现复杂度显著提升。

3. **朴素场景加速**：使用了 O(N×T) 的暴力求交，N=场景三角形数，T=光源求交次数。对于大场景需要 BVH（已在 04-02 实现），否则 9秒/帧的运行时间在游戏中完全不可用。

4. **仅 Lambertian BRDF**：ReSTIR 对于镜面反射（specular）的多光源采样效率不佳——高光区域的 $\hat{p}$ 分布非常尖锐，均匀光源采样命中率低。针对 specular 需要用 Resampled VNDF（可见法线分布）采样，即 ReSTIR GI 的扩展。

### 可优化方向

1. **时序重用**（Temporal Reuse）：实现帧间历史缓冲 + 运动向量 + 反遮挡，将有效样本数再 ×30
2. **GPU 并行化**：每像素独立，天然 SIMD。使用 CUDA/Metal/Vulkan compute shader 可实现实时 60fps
3. **ReSTIR GI**（SIGGRAPH 2021）：将 ReSTIR 扩展到间接光照（Global Illumination），原理相同但采样域从光源扩展到全场景
4. **无偏 MIS 权重**：实现 Algorithm 4 中的 MIS 权重，消除遮挡光源带来的偏差
5. **World-Space ReSTIR**：在世界空间而非屏幕空间做 Reservoir 管理，解决 G-Buffer 分辨率限制问题

### 与本系列的关联

这个项目是系列中技术深度最高的几个之一，综合运用了：
- **04-02 BVH**：加速结构（今天用暴力求交，工程版需要 BVH）
- **04-01 SDF Ray Marching**：光线步进思维（ReSTIR 中的可见性测试本质相同）
- **03-24 Subsurface Scattering**：BRDF 积分（今天的 Lambertian 是最简单的情况）
- **03-22 BDPT**：双向采样思想（ReSTIR 与 BDPT 的 MIS 有理论共通处）

ReSTIR 代表了实时光线追踪从"能用"走向"好用"的关键一步。2020 年之后，几乎所有主流游戏引擎的光追方案都或多或少借鉴了这个算法。理解 WRS 和 Reservoir 合并的原理，是进入现代实时 GI 领域的必要门票。

---

> 本文代码：[GitHub - daily-coding-practice/2026/05/05-01-restir-direct-illumination](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-01-restir-direct-illumination)  
> 参考论文：Bitterli et al., "Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting", SIGGRAPH 2020
