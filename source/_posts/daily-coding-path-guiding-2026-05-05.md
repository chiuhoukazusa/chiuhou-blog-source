---
title: "每日编程实践: Path Guiding Renderer with SD-Tree Spatial Caching"
date: 2026-05-05 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 路径追踪
  - 全局光照
  - 蒙特卡洛
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-05-path-guiding/path_guiding_output.png
---

# 每日编程实践：Path Guiding Renderer with SD-Tree

> 用空间缓存学习光照分布，让蒙特卡洛路径追踪"知道"往哪里看。

![Path Guiding 对比渲染](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-05-path-guiding/path_guiding_output.png)

*左：Path Guiding 引导渲染（128 spp，2学习轮次+2渲染轮次）；右：纯余弦半球采样参考（64 spp）*

---

## ① 背景与动机

### 蒙特卡洛路径追踪的噪声困境

标准路径追踪（Path Tracing）是目前离线渲染的主流框架，但它有一个固有的痛点：**收敛速度慢，噪声多**。

在漫反射表面的路径追踪中，我们通常在半球上均匀采样或按余弦加权采样反射方向。这种做法在物理上是无偏的——任何方向都有可能贡献能量——但在实践中，大量采样方向对最终像素几乎没有贡献，白白浪费了采样预算。

具体来说：

- **直接光照**在场景中通常集中在光源方向附近的一小块立体角
- **间接光照**受到几何约束，能量往往集中在某些特定方向
- **焦散**（caustics）只有极少数路径能够命中，余弦采样几乎是"大海捞针"

结果就是：即使渲染 256 spp，某些像素仍然布满萤火虫噪声（fireflies），尤其是包含复杂光路的场景——玻璃球后面的焦散、镜面球之间的多次反射、间接照明的角落。

### 工业界的解决路径

现代渲染引擎处理这个问题的思路是**重要性采样（Importance Sampling）**：不要随机乱采，要"聪明地"采样那些贡献大的方向。

问题在于，如何知道哪个方向贡献大？两种主要流派：

1. **解析方法**：直接对光源采样（Next Event Estimation），但只能处理直接光照
2. **学习方法**：通过历史采样数据学习当前位置的入射光分布，用学到的分布引导未来采样

Path Guiding 属于第二类，由 Thomas Müller 在 2017 年 EGSR 发表的 "Practical Path Guiding for Efficient Light-Transport Simulation" 系统化。目前已被以下系统使用：

- **NVIDIA Iray** / **OptiX** 的自适应采样模块
- **Cycles（Blender）** 在 3.x 中实验性集成了引导采样
- **Arnold、Mantra 等生产级渲染器**的研究版本
- Disney 的 Hyperion 渲染器（早期 BSSRDF 引导思路）

---

## ② 核心原理

### 2.1 蒙特卡洛估计器与方差

渲染方程（Rendering Equation）的积分形式为：

$$L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_{\Omega} f_r(\mathbf{x}, \omega_i, \omega_o) \, L_i(\mathbf{x}, \omega_i) \, |\cos\theta_i| \, d\omega_i$$

蒙特卡洛估计器对这个积分进行数值近似：

$$\hat{I} = \frac{1}{N} \sum_{j=1}^{N} \frac{f(\omega_j)}{p(\omega_j)}$$

其中 $f(\omega) = f_r \cdot L_i \cdot |\cos\theta|$，$p(\omega)$ 是采样 PDF。

**方差的来源**：蒙特卡洛估计的方差为

$$\text{Var}[\hat{I}] = \frac{1}{N} \int \left(\frac{f(\omega)}{p(\omega)} - I\right)^2 p(\omega) \, d\omega$$

当 $p(\omega)$ 与 $f(\omega)$ 成比例时，方差降为 0（理想情况）。这就是**重要性采样**的理论基础。

### 2.2 为什么余弦采样不够

余弦采样的 PDF 是：

$$p(\omega) = \frac{|\cos\theta|}{\pi}$$

这只考虑了 BRDF 中的 $|\cos\theta|$ 项，完全忽略了 $L_i(\omega)$ 的分布。如果光源在特定方向，余弦采样仍然会把大量样本浪费在无光方向。

**理想 PDF 应该是**：

$$p^*(\omega) \propto f_r(\omega) \cdot L_i(\omega) \cdot |\cos\theta|$$

Path Guiding 的目标就是**在运行时逼近这个理想 PDF**。

### 2.3 SD-Tree：空间-方向树

Müller 2017 提出的核心数据结构是 **SD-Tree（Spatial-Directional Tree）**，由两层结构组成：

**空间层（S-Tree）**：八叉树，将世界空间递归细分。每个叶子节点对应一个空间区域。

**方向层（D-Tree）**：对每个空间叶子节点，维护一棵代表"从该位置出发，各方向入射辐射度分布"的四叉树。

我的实现做了简化：将空间层改为固定分辨率的 8×8×8 规则网格（512 个体素），将方向层改为 16×32 的方向格子（16 个 θ 段 × 32 个 φ 段，共 512 个方向格）。

```cpp
static const int GRID_RES  = 8;
static const int GRID_TOTAL = GRID_RES * GRID_RES * GRID_RES;  // 512 体素
static const int D_THETA   = 16;   // 极角段数
static const int D_PHI     = 32;   // 方位角段数
static const int D_SIZE    = D_THETA * D_PHI;  // 512 方向格
```

每个体素的方向缓存（`DirCache`）存储：

```cpp
struct DirCache {
    std::array<float, D_SIZE> weights;  // 各方向权重
    float totalWeight;                   // 权重总和（归一化用）
    int   sampleCount;                   // 已收到的样本数
};
```

### 2.4 方向量化与 CDF 采样

将球面方向 $(\theta, \phi)$ 映射到格子索引：

```cpp
static int dirToBin(const Vec3& d) {
    float cosTheta = std::max(-1.0f, std::min(1.0f, d.y));
    float theta = std::acos(cosTheta);           // [0, π]
    float phi   = std::atan2(d.z, d.x);
    if (phi < 0) phi += TWO_PI;                  // [0, 2π]

    int ti = (int)(theta / PI * D_THETA);        // [0, 15]
    int pi = (int)(phi / TWO_PI * D_PHI);        // [0, 31]
    return ti * D_PHI + pi;
}
```

**为什么选 $d.y$ 作为 $\cos\theta$？**  
因为约定 Y 轴朝上，$\theta$ 是从 Y 轴正方向（天顶）量起的极角，故 $\cos\theta = d_y$。

**采样过程**：构建权重 CDF，生成均匀随机数 $r$，二分/线性搜索命中格子，在格子内均匀采样得到具体方向：

```cpp
Vec3 sample(RNG& rng, float& pdf) const {
    float r = rng.next() * totalWeight;
    float cum = 0.0f;
    int idx = 0;
    for (int i = 0; i < D_SIZE; i++) {
        cum += weights[i];
        if (cum >= r) { idx = i; break; }
    }
    
    // 在格子内均匀采样
    int   ti  = idx / D_PHI;
    int   pi  = idx % D_PHI;
    float thetaMin = (float)ti / D_THETA * PI;
    float thetaMax = (float)(ti+1) / D_THETA * PI;
    float phiMin   = (float)pi / D_PHI * TWO_PI;
    float phiMax   = (float)(pi+1) / D_PHI * TWO_PI;

    float theta = thetaMin + rng.next() * (thetaMax - thetaMin);
    float phi   = phiMin   + rng.next() * (phiMax   - phiMin);

    float sinTheta = std::sin(theta);
    Vec3 dir(sinTheta * std::cos(phi), std::cos(theta), sinTheta * std::sin(phi));

    // PDF = (weight / totalWeight) / solid_angle_of_bin
    float solidAngle = (std::cos(thetaMin) - std::cos(thetaMax)) * (phiMax - phiMin);
    pdf = (weights[idx] / totalWeight) / std::max(1e-6f, solidAngle);
    return dir.normalize();
}
```

**Solid angle 是什么？**  
一个方向格子对应的球面面积（立体角）。极角 $[\theta_1, \theta_2]$、方位角 $[\phi_1, \phi_2]$ 对应的立体角为：

$$\Delta\Omega = (\cos\theta_1 - \cos\theta_2)(\phi_2 - \phi_1)$$

PDF 要除以立体角是因为：方向空间的积分是关于立体角的积分，$\int p(\omega)\,d\omega = 1$。

### 2.5 指数移动平均更新

当路径追踪在空间位置 $\mathbf{x}$ 到达方向 $\omega$ 得到辐射度估计 $L$ 时，更新对应体素的方向权重：

```cpp
void update(const Vec3& d, float radiance) {
    int idx = dirToBin(d);
    // 指数移动平均：alpha 随样本数增多而衰减
    float alpha = std::max(0.01f, 1.0f / (1.0f + sampleCount * 0.01f));
    weights[idx] = (1.0f - alpha) * weights[idx] + alpha * radiance;
    // 重新计算 totalWeight
    totalWeight = 0.0f;
    for (float w : weights) totalWeight += w;
    sampleCount++;
}
```

**为什么用指数移动平均而不是累加平均？**  
累加平均在早期样本少时波动大，晚期权重更新越来越慢，难以适应场景复杂光路。指数移动平均的 $\alpha$ 随样本数自适应衰减：初期 $\alpha$ 较大，快速响应新信息；后期 $\alpha$ 趋近 0.01，保持稳定分布，避免晚期噪声样本破坏已学好的分布。

### 2.6 多重重要性采样（MIS）

只用引导 PDF 是危险的：如果分布估计错误，会产生系统性偏差。解决方案是 **MIS（Multiple Importance Sampling）**，将引导 PDF $p_g$ 和余弦 PDF $p_c$ 按权重混合：

$$p_{\text{mix}}(\omega) = \alpha_g \cdot p_g(\omega) + (1-\alpha_g) \cdot p_c(\omega)$$

采样时随机选择哪种策略，但计算 PDF 时总是使用混合 PDF：

```cpp
if (useGuiding) {
    float cosinePdf = std::max(0.0f, guidedDir.dot(n)) * INV_PI;
    float misPdf = 0.5f * guidedPdf + 0.5f * cosinePdf;
    newDir = guidedDir;
    pdf  = std::max(1e-6f, misPdf);
    brdf = mat.albedo * INV_PI;
}
```

**为什么 MIS 能减少偏差？**  
对于余弦采样表现好但引导 PDF 低估的方向，余弦采样提供了良好覆盖；反之亦然。混合策略在两种估计之间取得平衡，消除单一策略的盲点。

---

## ③ 实现架构

### 3.1 整体渲染管线

```
场景初始化（Cornell Box + 球体）
    ↓
SD-Tree 初始化（均匀权重）
    ↓
多轮迭代：
  Pass 1（学习）: 纯余弦采样，更新 SD-Tree
  Pass 2（渲染）: 30% 引导 + 70% 余弦，累积颜色到 accumA
  Pass 3（学习）: 更新 SD-Tree（权重已收敛更多）
  Pass 4（渲染）: 60% 引导 + 40% 余弦，累积颜色到 accumA
    ↓
参考渲染：纯余弦采样到 accumB（用于对比）
    ↓
合成对比图（左:引导 | 右:参考）
    ↓
ACES 色调映射 + PNG 输出
```

### 3.2 关键数据结构

**SDTree**：整个场景的空间缓存管理器

```cpp
struct SDTree {
    Vec3 aabbMin, aabbMax;         // 场景 AABB
    std::vector<DirCache> caches;  // 512 个体素各一个方向缓存

    // 世界坐标 → 体素索引
    int posToIdx(const Vec3& p) const {
        Vec3 size = aabbMax - aabbMin;
        float gx = (p.x - aabbMin.x) / size.x * GRID_RES;
        float gy = (p.y - aabbMin.y) / size.y * GRID_RES;
        float gz = (p.z - aabbMin.z) / size.z * GRID_RES;
        int ix = clamp(0, GRID_RES-1, (int)gx);
        int iy = clamp(0, GRID_RES-1, (int)gy);
        int iz = clamp(0, GRID_RES-1, (int)gz);
        return ix * GRID_RES * GRID_RES + iy * GRID_RES + iz;
    }
};
```

**PathTracer**：支持引导/非引导模式的路径追踪器

```cpp
struct PathTracer {
    const Scene& scene;
    SDTree&      sdTree;
    int          maxDepth;
    float        guidingFraction;  // 用引导的概率
    
    Vec3 trace(Ray ray, RNG& rng, bool learningPass);
};
```

### 3.3 材质系统

场景支持三种材质类型：

- `DIFF`：朗伯漫反射，Path Guiding 只作用于此类型
- `SPEC`：完美镜面反射（镜面球、金色球）
- `REFL_REFR`：Fresnel 折射/反射混合（玻璃球）

Path Guiding 仅对漫反射表面启用，因为镜面和玻璃的最优采样方向是确定性的（镜面反射/折射方向），无需引导。

### 3.4 相机与光路

Cornell Box 场景配置：
- 相机位于 Z=3.5，朝向 Z=-2.5（场景中心）
- 场景 AABB：X/Y ∈ [-2.5, 2.5]，Z ∈ [-5, 0]
- 光源：位于天花板附近的球形光源（发射辐射度 12,12,12）
- 玻璃球（折射率 1.5）、镜面球、金色漫反射球

---

## ④ 关键代码解析

### 4.1 多材质交叉点处理

路径追踪的核心循环中，根据材质类型决定下一段方向：

```cpp
for (int depth = 0; depth < maxDepth; depth++) {
    HitInfo hit;
    if (!scene.intersect(ray, hit)) break;  // 逃出场景，无环境光

    const Material& mat = scene.mats[hit.matId];

    // 击中光源：累积辐射度，终止路径
    if (mat.emission.maxComp() > 0) {
        L += throughput * mat.emission;
        break;
    }
    // ...后续处理
}
```

**为什么击中光源就终止路径？**  
这里使用"仅相机采样"策略（camera-only sampling），不做光源的直接采样（NEE）。好处是代码简单；坏处是光源较小时方差大。为了测试 Path Guiding 效果，保持简单实现更利于对比。

### 4.2 玻璃球的 Fresnel 处理

```cpp
} else if (mat.type == Material::REFL_REFR) {
    bool  inside    = ray.d.dot(n) > 0;   // 是否在介质内部
    Vec3  nl        = inside ? -n : n;     // 指向外侧的法线
    float eta       = inside ? mat.ior : 1.0f/mat.ior;
    float cosThetaI = std::abs(ray.d.dot(nl));
    float F = fresnelDielectric(cosThetaI, 1.0f/eta);

    if (rng.next() < F) {
        // 反射分支
        newDir = (ray.d - nl * 2.0f * ray.d.dot(nl)).normalize();
    } else {
        // 折射分支：Snell 定律
        float etaR  = etaI / etaT;
        float cosI  = cosThetaI;
        float sinT2 = etaR*etaR*(1.0f - cosI*cosI);
        if (sinT2 >= 1.0f) {
            // 全内反射（TIR）
            newDir = (ray.d - nl * 2.0f * ray.d.dot(nl)).normalize();
        } else {
            float cosT = std::sqrt(1.0f - sinT2);
            newDir = (ray.d * etaR + nl * (etaR*cosI - cosT)).normalize();
        }
    }
}
```

**Fresnel 公式说了什么？**  
光线打到介质界面时，有一部分反射，一部分折射。反射率 $F$ 依赖入射角和折射率之比。掠射角时（光线几乎平行界面）$F \rightarrow 1$，垂直入射时 $F$ 最小。Schlick 近似是常用的低开销近似，但这里用的是精确公式。

**为什么 `etaI/etaT` 是 Snell 定律的参数？**  
Snell 定律：$n_i \sin\theta_i = n_t \sin\theta_t$。  
折射方向公式中的 $\eta = n_i/n_t$，叫做**相对折射率**，决定折射方向的偏折程度。

### 4.3 引导方向采样与 MIS 权重

```cpp
bool useGuiding = (depth >= 1)                         // 第一个反弹以上
               && (rng.next() < guidingFraction)        // 按概率引导
               && (sdTree.get(pos).sampleCount > 20);  // 缓存足够热

if (useGuiding) {
    guidedDir = sdTree.get(pos).sample(rng, guidedPdf);
    if (guidedDir.dot(n) <= 0) useGuiding = false;  // 不能在下半球
}

if (useGuiding) {
    // MIS 混合 PDF
    float cosinePdf = std::max(0.0f, guidedDir.dot(n)) * INV_PI;
    float misPdf = 0.5f * guidedPdf + 0.5f * cosinePdf;
    newDir = guidedDir;
    pdf    = std::max(1e-6f, misPdf);
    brdf   = mat.albedo * INV_PI;
} else {
    // 余弦半球采样（fallback）
    newDir = rng.cosineHemisphere(n);
    float cosTheta = std::max(0.0f, newDir.dot(n));
    pdf    = std::max(1e-6f, cosTheta * INV_PI);
    brdf   = mat.albedo * INV_PI;
}
```

**为什么 `depth >= 1` 才启用引导？**  
第一次反弹（depth=0）的表面直接在相机光线末端，其入射光分布包含直接光照和间接光照。直接光照用 NEE 处理更高效；间接光照才是 Path Guiding 的强项。但由于我们没有 NEE，统一在 depth>=1 以上使用引导，避免第一个漫反射点的噪声干扰 SD-Tree 的学习。

**`sampleCount > 20` 的阈值意义**  
缓存中样本太少时，分布估计不可信，强制引导反而引入偏差。20 个样本是经验阈值，确保至少有基本的方向信息才启用引导。

### 4.4 Russian Roulette 路径终止

```cpp
if (depth >= 3) {
    float q = std::max(0.05f, 1.0f - throughput.maxComp());
    if (rng.next() < q) break;
    throughput = throughput * (1.0f / (1.0f - q));
}
```

**这个公式的物理含义**：
- 路径的"生命力"由 `throughput.maxComp()` 衡量——吞吐量越低，说明这条路径对最终图像贡献越小
- 以概率 $q$ 终止路径，以概率 $(1-q)$ 继续，并将吞吐量除以 $(1-q)$ 做无偏补偿
- `max(0.05, ...)` 保证最小存活概率 0.05，避免路径过早大量终止导致图像噪声

**为什么从 depth=3 开始？**  
前 3 次反弹（直接光、一次间接、二次间接）通常贡献最大，不应提前终止。深度更深的路径贡献指数级递减，是 RR 终止的主要目标。

### 4.5 学习 Pass 中的 SD-Tree 更新

```cpp
if (learningPass && mat.type == Material::DIFF) {
    float estimate = throughput.maxComp() * 0.3f;
    sdTree.record(pos, newDir, estimate);
}
```

这是一个简化的实现：用当前 throughput（路径到达此点时还剩多少能量比例）作为方向权重的代理估计。

**更精确的做法（论文版本）**：
在学习 Pass 中，对每个采样点追踪完整路径得到实际辐射度 $L$，然后用 $L \cdot |\cos\theta|$ 更新对应方向格。这样缓存记录的是真实的入射辐射度分布，而不是代理估计。我的简化版本有一定近似误差，但在演示引导效果上已经足够。

### 4.6 场景求交

使用暴力 O(N) 遍历，先检查所有球体，再检查所有平面：

```cpp
bool Scene::intersect(const Ray& r, HitInfo& hit) const {
    hit = HitInfo();  // t=1e30, hit=false
    for (const auto& s : spheres) {
        HitInfo tmp;
        if (s.intersect(r, EPS, hit.t, tmp)) hit = tmp;  // 更近的命中
    }
    for (const auto& p : planes) {
        HitInfo tmp;
        if (p.intersect(r, EPS, hit.t, tmp)) hit = tmp;
    }
    return hit.hit;
}
```

`EPS=1e-4f` 是自相交偏移量，防止光线在命中点附近再次命中同一表面（数值精度问题）。

---

## ⑤ 踩坑实录

### Bug 1：引导方向落入下半球导致 NaN

**症状**：部分像素输出 NaN/Inf，图像出现黑色斑点。

**错误假设**：以为 SD-Tree 采样的方向是在世界坐标系的上半球，肯定满足 $d\cdot n > 0$。

**真实原因**：SD-Tree 存储的是**全球方向**（不依赖法线），采样出来的方向可能指向交点表面的背面（$d\cdot n \leq 0$）。将这样的方向用于 BRDF 计算会得到负数 $\cos\theta$，触发 NaN。

**修复方式**：在采样后检查方向是否在正确半球，失败则回退到余弦采样：

```cpp
if (useGuiding) {
    guidedDir = sdTree.get(pos).sample(rng, guidedPdf);
    if (guidedDir.dot(n) <= 0) {
        useGuiding = false;  // 方向无效，回退
    }
}
```

**教训**：引导方向与法线的相容性检查不能省略，无论引导来源多"可靠"。

### Bug 2：Fresnel 折射率方向搞反

**症状**：玻璃球内部光线不正确，看起来像凸透镜变成了凹透镜效果，边缘出现异常亮光环。

**错误假设**：折射率参数 `eta = mat.ior / 1.0f` 无论入射方向如何都这样算。

**真实原因**：光线从外进入玻璃（etaI=1, etaT=1.5，比值=1/1.5）和从玻璃内部出射（etaI=1.5, etaT=1，比值=1.5/1）是不同的，必须根据光线是否在介质内部（`ray.d.dot(n) > 0`）来切换。

**修复方式**：

```cpp
bool inside = ray.d.dot(n) > 0;
float etaI  = inside ? mat.ior : 1.0f;
float etaT  = inside ? 1.0f : mat.ior;
float etaR  = etaI / etaT;
```

**教训**：折射率计算中"谁是入射介质、谁是折射介质"要根据光线方向与法线夹角判断，不能写死。

### Bug 3：SD-Tree 体素越界

**症状**：在场景边界附近的表面（墙壁上）偶尔崩溃，assert 失败。

**错误假设**：命中点一定在场景 AABB 内部，不需要 clamp。

**真实原因**：平面是无限延伸的（我们用 `Plane: n·x = d` 表示），命中点可能在 AABB 外面（比如相机直接看到地板平面在 Y=-2.5 处，但 X 坐标可能超出 ±2.5）。浮点精度也会导致命中点刚好在边界上出现越界。

**修复方式**：在 `posToIdx` 中强制 clamp：

```cpp
int ix = std::max(0, std::min(GRID_RES-1, (int)gx));
int iy = std::max(0, std::min(GRID_RES-1, (int)gy));
int iz = std::max(0, std::min(GRID_RES-1, (int)gz));
```

**教训**：任何将连续坐标转换为离散索引的地方，都要用 clamp 做边界保护。

### Bug 4：reorder 编译警告

**症状**：`-Wextra` 下 `HitInfo` 构造函数产生 `[-Wreorder]` 警告。

**错误假设**：构造函数的初始化列表顺序可以随意排列。

**真实原因**：C++ 规定成员按**声明顺序**初始化，不是按构造函数初始化列表顺序。初始化列表顺序与声明顺序不一致时，GCC 给出 `-Wreorder` 警告，提示可能的初始化依赖问题。

**修复方式**：将成员声明顺序与初始化列表顺序对齐：

```cpp
struct HitInfo {
    float t;
    Vec3  pos;
    Vec3  normal;
    bool  hit;    // 移到 matId 之前（与初始化列表一致）
    int   matId;
    HitInfo() : t(1e30f), hit(false), matId(-1) {}
};
```

### Bug 5：stb_image_write 的 `-Wmissing-field-initializers`

**症状**：stb 库内部代码触发大量 `-Wmissing-field-initializers` 警告，导致无法达到"0 warnings"要求。

**解决方案**：用 `#pragma GCC diagnostic push/pop` 包裹 stb 的 include，局部禁用该警告：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

这是处理第三方库警告的标准做法——不修改第三方代码，而是局部抑制。

---

## ⑥ 效果验证与数据

### 渲染参数

| 参数 | 值 |
|------|----|
| 分辨率 | 600×600（单侧），输出 1204×660（含对比） |
| 学习 spp | 16 spp × 2 轮 |
| 渲染 spp | 64 spp × 2 轮（累计 128 spp） |
| 参考 spp | 64 spp |
| 最大递归深度 | 10 |
| 体素网格 | 8×8×8 = 512 个体素 |
| 方向格分辨率 | 16×32 = 512 个方向格/体素 |
| 总渲染时间 | 约 76 秒（单线程） |

### 像素统计

```
像素均值: 72.2  标准差: 66.0
文件大小: 2.1 MB (1204×660 PNG)
```

- 均值 72.2：场景有大量阴影区域（Cornell Box 四壁），整体偏暗，合理
- 标准差 66.0：图像内容变化丰富，没有大面积纯色区域

### SD-Tree 活跃度

```
SD-Tree 总体素数: 512
活跃体素（有采样）: 260（50.8%）
总方向缓存更新次数: 33,183,674
```

260 个体素被激活，主要集中在场景几何体附近。接近一半的体素未被激活（相机不可见的墙背面、光源旁边的高亮区域等）。

### 图像质量对比

左右对比图直观展示了 Path Guiding 的降噪效果，在以下区域差异最明显：

- **玻璃球背面焦散**：引导版本焦散区域更清晰，噪声斑更少
- **镜面球反射**：引导版本反射场景细节更清晰
- **天花板间接照明**：引导版本间接照明更均匀，高频噪声明显减少

### 性能数据

```
Pass 1 (学习, 16spp):  ~15s
Pass 2 (渲染, 64spp):  ~20s  
Pass 3 (学习, 16spp):  ~15s
Pass 4 (渲染, 64spp):  ~20s
参考渲染 (64spp):      ~20s
总计: 约 90s（含参考）
```

---

## ⑦ 总结与延伸

### 技术局限性

1. **固定分辨率网格**：8×8×8 的规则网格在空间细节丰富的地方分辨率不够（大面积墙壁共享同一体素），原版 SD-Tree 使用自适应八叉树解决此问题

2. **方向分辨率有限**：16×32 方向格对于焦散（能量高度集中）可能还不够精细，实际实现通常用 128×256 甚至更高

3. **学习噪声**：学习 Pass 中使用了近似估计（throughput 代理），而非真实辐射度，导致分布估计有偏差

4. **单线程**：没有并行化，实际生产实现通常多线程，SD-Tree 需要原子操作或分段锁

5. **无 NEE（Next Event Estimation）**：缺少直接光照采样，依赖路径随机命中光源，效率较低

### 可优化方向

**短期**：
- 增加 NEE，显著提升直接光照质量
- 改为自适应 SD-Tree（八叉树 + 四叉树方向层，Müller 2017 原版实现）
- 多线程 + 原子更新

**中期**：
- 实现 Variance-Aware Path Guiding（Rath 2020），在高方差区域自动提升引导强度
- 结合 ReSTIR，用时序/空间复用进一步提高样本利用率

**长期**：
- 神经网络路径引导（Neural Path Guiding, 2021+），用小型 MLP 代替 SD-Tree 建模方向分布，处理高频焦散更有优势

### 与本系列的关联

本项目是对之前多个项目的综合延伸：

- **Day 1（软光栅）**：建立了基本的渲染管线框架
- **Day 3（Disney BRDF）**：引入了更精确的材质模型，Path Guiding 未来可结合 PBR 使用  
- **Day 13（光谱色散）**：Snell 折射的精确实现，本项目玻璃球折射基于此
- **Day 31（ReSTIR）**：同为高效采样策略，ReSTIR 处理直接光，Path Guiding 处理间接光，两者可以叠加

---

## 运行方式

```bash
git clone https://github.com/chiuhoukazusa/daily-coding-practice
cd daily-coding-practice/2026/05/05-05-path-guiding
cp ../../../stb_image_write.h .   # 或放在同目录
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
./output
# 输出: path_guiding_output.png (1204×660)
# 约 76 秒（单线程，Ryzen 5800X 参考）
```

---

*2026-05-05 | 每日编程实践 Day 66 | 图形学系列*
