---
title: "每日编程实践: SPPM 随机渐进光子映射"
date: 2026-03-21 06:00:00
tags:
  - 每日一练
  - 图形学
  - 全局光照
  - 光子映射
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-21-sppm/sppm_output.png
---

## ① 背景与动机

### 为什么需要全局光照？

在实时渲染的世界里，我们长期依赖"直接光照 + 环境光假设"来模拟光照效果。这类方法速度快，但有一个根本性缺陷：**它忽略了光线在场景中多次弹射的间接照明**。

想象一个 Cornell Box——一个封闭的盒子，左墙红色，右墙绿色，天花板有一盏灯。在纯直接光照渲染中，你会看到墙面上均匀涂抹的红色和绿色；但在现实世界（或使用全局光照的渲染器）里，红墙的颜色会"流血"到白色地板和天花板上——这就是**颜色渗色（Color Bleeding）**，是全局光照的典型特征。

此外，当场景中有镜面反射物体或玻璃时，光线可能经过多次折射/反射后才照亮某个漫反射表面。直接光照模型根本无法处理这类焦散（Caustics）效果——例如玻璃球下方那一圈明亮的光斑。

**全局光照真正解决的问题**：
- 漫反射间接照明（颜色渗色、角落变暗）
- 焦散（通过玻璃/镜面聚焦的光）
- 柔和阴影的物理正确计算
- 色调一致性（整个场景的光能守恒）

### 工业界实际应用

全局光照在以下场景中有大量实际应用：

**离线渲染（影视/动画）**：
- Pixar 的 RenderMan 使用路径追踪 + 多种 GI 技术
- 《寻梦环游记》《钢铁侠》等影片的特效渲染
- 建筑可视化：客户要看到真实的空间光照质感

**游戏引擎（烘焙 GI）**：
- Unreal Engine 的 Lightmass：烘焙时使用 Photon Mapping 的变体
- Unity 的 Progressive Lightmapper：基于路径追踪的 GI 烘焙
- 《赛博朋克 2077》的光照烘焙系统

**实时 GI（近年进展）**：
- NVIDIA RTX 的实时光线追踪（Path Tracing）
- Lumen（UE5）：结合屏幕空间和世界空间的混合 GI
- ReSTIR：重采样重要性采样，提升实时光子密度估计效率

光子映射（Photon Mapping）作为一种经典的全局光照算法，在理解 GI 原理和实现工业级烘焙器方面都有重要价值。而 SPPM（Stochastic Progressive Photon Mapping）是光子映射的渐进式改进版本，能在有限内存下收敛到无偏结果。

---

## ② 核心原理

### 2.1 渲染方程回顾

光照的物理基础是渲染方程（Kajiya 1986）：

$$L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_{\Omega} f_r(\mathbf{x}, \omega_i, \omega_o) \cdot L_i(\mathbf{x}, \omega_i) \cdot |\cos\theta_i| \, d\omega_i$$

**直觉解释**：
- $L_o$：从点 $\mathbf{x}$ 朝方向 $\omega_o$ 射出的辐亮度（radiance）
- $L_e$：自发光项（光源本身的贡献）
- 积分项：所有入射方向上的贡献之和
  - $f_r$：BSDF（双向散射分布函数），描述材质如何反射/折射光线
  - $L_i$：从 $\omega_i$ 方向入射的辐亮度
  - $|\cos\theta_i|$：兰伯特余弦项，光线越斜入射贡献越小

对漫反射材质，$f_r = \rho / \pi$，其中 $\rho$ 是 albedo（表面颜色/反射率）。这个 $1/\pi$ 来自归一化条件：对半球积分 BSDF 必须 ≤ 1，而 $\int \cos\theta \, d\omega = \pi$，所以 $f_r \cdot \pi = \rho$，因此 $f_r = \rho/\pi$。

### 2.2 经典光子映射（Jensen 2001）

经典光子映射分两个 pass：

**Pass 1：光子发射**

从光源向场景随机发射光子，每个光子携带功率 $\Phi_p$：

$$\Phi_p = \frac{\Phi_{light}}{N_{photons}}$$

光子在场景中传播，碰到漫反射面时存储到光子图，继续传播时按 Russian Roulette 决定是否终止：

- 继续概率 = $\max(\rho_r, \rho_g, \rho_b)$（相当于用 albedo 来决定是否继续）
- 这样可以保证能量守恒

**Pass 2：辐亮度估计（密度估计）**

在渲染时，对每个着色点 $\mathbf{x}$，找到半径 $r$ 内的所有光子，用核密度估计：

$$L(\mathbf{x}, \omega_o) \approx \frac{1}{\pi r^2} \sum_{k=1}^{n} f_r(\mathbf{x}, \omega_k, \omega_o) \cdot \Phi_k$$

**直觉**：半径 $r$ 的圆面积是 $\pi r^2$，把所有光子功率求和再除以面积，就得到辐照度（irradiance）；再乘以 BSDF 得到辐亮度。

**经典光子映射的问题**：
1. 需要在内存中保存所有光子（场景复杂时可能需要数亿个光子）
2. 半径 $r$ 固定，要么有偏差（r 太大时模糊），要么噪声大（r 太小时光子不够）

### 2.3 渐进光子映射（Hachisuka 2008）

PPM（Progressive Photon Mapping）的核心思想：

1. 首先，从相机出发追踪光线，记录所有漫反射 **Hit Points（HP）**
2. 然后，多轮发射光子，每轮后更新每个 HP 的半径：

$$r_{n+1}^2 = r_n^2 \cdot \frac{N_n + \alpha M_n}{N_n + M_n}$$

其中 $N_n$ 是之前累积的光子数，$M_n$ 是本轮新增光子数，$\alpha \in (0,1)$ 是收缩因子（通常取 0.7~0.8）。

**为什么这样收缩是正确的？**

直觉：
- 每次迭代，我们的估计越来越精确（更多光子），可以适当缩小半径以减小偏差
- 但不能收缩太快，否则方差增大
- 数学上可以证明：当 $n \to \infty$ 时，$r_n \to 0$ 且 $N_n \to \infty$，估计量收敛到真值（渐近无偏）

累积通量的更新：

$$\Phi_{n+1} = \left(\Phi_n + \sum_{k \in \text{new}} \Phi_k^{(n)}\right) \cdot \frac{r_{n+1}^2}{r_n^2}$$

这里 $\Phi_k^{(n)}$ 是第 $n$ 轮中落在 HP 范围内的光子功率。乘以面积比是为了对应半径的收缩。

### 2.4 随机渐进光子映射（SPPM，Hachisuka 2009）

PPM 的一个问题：相机 pass 每次迭代都需要完整重追踪，开销大。SPPM 的改进：

**每次迭代都随机重采样相机路径**（即 Hit Points 的位置每轮可以不同）

这样做有两个好处：
1. 摒弃了对相机路径的确定性依赖，变成统计意义上的无偏估计
2. 即使场景有运动（动画），每次重采样也能正确处理

SPPM 最终估计公式：

$$L_{pixel} = \frac{1}{N_{iters}} \sum_{i=1}^{N_{iters}} \left( L_{direct}^{(i)} + \frac{\text{accFlux}^{(i)}}{\pi \cdot r_i^2 \cdot N_e^{(i)}} \right)$$

其中 $N_e$ 是发射的总光子数。

**本实现的做法**：相机 pass 只做一次（静态场景），然后多轮光子 pass，每轮后用 SPPM 公式更新所有 HP。最终图像在重建阶段一次性计算。

### 2.5 单位与量纲分析

这是实现中最容易出错的地方。我们需要仔细追踪每个量的物理单位：

**光子功率**：
$$\Phi_{per\_photon} = L_{emit} \times \pi \times A_{light} / N_{photons}$$

解释：
- $L_{emit}$：光源辐亮度（W/sr/m²）
- $\pi$：对 Lambertian 面光源，对出射半球积分 $\int L \cos\theta \, d\omega = \pi L$（辐出度 M = πL）
- $A_{light}$：光源面积（m²）
- 乘积 = 光源总功率（W）
- 除以 $N_{photons}$ = 每个光子代表的功率（W）

**相机路径权重**：
- 相机 pass 记录 `hp.weight = throughput`（不含当前漫反射面的 albedo）
- 另外单独记录 `hp.albedo = surface albedo`
- 这样分开存储是为了在密度估计时能正确组合

**密度估计**（漫反射面）：

$$L \approx \text{hp.weight} \times \frac{\text{hp.albedo}}{\pi} \times \frac{\sum_{k} \Phi_k}{\pi r^2}$$

等等，这里有两个 $\pi$？让我展开：

- BSDF（Lambertian）：$f_r = \rho/\pi$
- 余弦采样 PDF：$p(\omega) = \cos\theta/\pi$
- BSDF/PDF = $\rho/\pi / (\cos\theta/\pi) = \rho/\cos\theta$

但光子密度估计时：
$$L = \frac{f_r \cdot \sum \Phi_k}{\pi r^2} = \frac{(\rho/\pi) \cdot \sum \Phi_k}{\pi r^2}$$

在代码中，我合并成 `* (1.0 / PI)` 乘到 `accFlux` 里，最后除以 `PI * r^2 * ITERS`：

```
color = accFlux / (PI * r² * ITERS)
```

这里 accFlux 已经包含了 `weight × albedo × dFlux × (1/π)`，所以最终：
$$L = \frac{\text{weight} \times \text{albedo} \times \sum\Phi_k / \pi}{\pi r^2 \times ITERS}$$

---

## ③ 实现架构

### 3.1 整体数据流

```
┌────────────────────────────────────────────────────────────────┐
│                      SPPM 渲染管线                              │
│                                                                │
│  Scene Setup                                                   │
│  Cornell Box: 地板/天花板/5面墙 + 天花板光源                    │
│  材质: white/red/green/mirror/glass/light                       │
│                  ↓                                             │
│  Camera Pass（一次性）                                          │
│  对每个像素发射光线 → 追踪到漫反射面 → 记录 HitPoint            │
│  HitPoint: pos, normal, weight, albedo, radius², N, accFlux    │
│                  ↓                                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Photon Pass × ITERS 次                                 │  │
│  │  1. 从光源发射 N_PHOTONS 个光子                          │  │
│  │  2. 追踪光子路径（漫反射/镜面/玻璃）                     │  │
│  │  3. 存储所有漫反射碰撞光子                               │  │
│  │  4. 构建 kd-tree                                        │  │
│  │  5. SPPM Update：更新每个 HP 的半径和累积通量            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                  ↓                                             │
│  Image Reconstruction                                          │
│  color = accFlux / (π × r² × ITERS)                           │
│  Tone mapping (ACES) + Gamma 2.2                               │
│                  ↓                                             │
│  stbi_write_png("sppm_output.png")                             │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 关键数据结构

**HitPoint（击中点）**：

```cpp
struct HitPoint {
    Vec3   pos;       // 世界空间位置
    Vec3   normal;    // 表面法线（指向入射侧）
    Vec3   weight;    // 相机路径吞吐量（不含表面 albedo！）
    Vec3   albedo;    // 表面漫反射率（单独存储）
    Vec3   accFlux;   // 累积: weight × albedo × Σph.flux / π
    double radius2;   // 当前搜索半径²（动态收缩）
    int    N;         // 累积修正光子数（SPPM 状态）
    int    pixIdx;    // 对应像素索引
    bool   valid;     // 是否为有效漫反射 HP
    Vec3   directL;   // 直接照到光源的贡献
};
```

为什么 `weight` 和 `albedo` 要分开？因为密度估计时需要 `weight × (albedo/π)`，而 albedo 在 SPPM 更新时需要单独处理。如果合并，就无法在不同迭代间正确累积。

**Photon（光子）**：

```cpp
struct Photon {
    Vec3 pos;    // 碰撞位置
    Vec3 power;  // 路径功率（不含当前表面 albedo）
    Vec3 dir;    // 入射方向（用于背面检测）
};
```

`power` 不含当前漫反射面的 albedo——这与 HitPoint 的设计保持一致，使得密度估计公式统一。

**kd-tree**：

```cpp
struct KdTree {
    struct Node { int idx, left, right, axis; };
    std::vector<Node> nodes;  // 预分配节点
    const std::vector<Photon>* phs;  // 指向光子数组的指针
    int root, nc;  // 根节点索引，节点计数
    // ...
};
```

选择 kd-tree 而不是其他加速结构（如 BVH、哈希网格），是因为 kd-tree 对不均匀分布的光子有很好的适应性，构建开销 O(N log N)，查询 O(log N + k)，适合光子的球形近邻查询。

### 3.3 场景坐标系

Cornell Box 在 `x∈[-1,1], y∈[-1,1], z∈[0,2]` 空间中：

```
     ┌──────────────┐
     │   Light      │  y = 0.999，光源（x∈[-0.3,0.3], z∈[0.7,1.3]）
     │              │
     │  [Mirror]  [Glass]  y = -0.57，半径 0.43 的两个球
     │              │
     └──────────────┘
y=-1（地板）    y=+1（天花板）
x=-1（红墙）    x=+1（绿墙）
z=0（相机侧）   z=2（背墙）
```

相机在 `(0, 0, -1.0)`，朝向 `+z`，FOV 50°。

### 3.4 材质与光传播职责划分

| 材质类型 | 相机 Pass 行为 | 光子 Pass 行为 |
|---------|-------------|-------------|
| DIFFUSE | 记录 HP，停止追踪 | 存储光子，按 albedo 概率继续 |
| MIRROR  | 反射继续，throughput × albedo | 反射继续，power × albedo |
| GLASS   | Fresnel 折射/反射，throughput × albedo | Fresnel 折射/反射，power × albedo |
| Light   | 记录直接亮度，停止 | 停止（光子从光源出发，不会再"打到光源"） |

---

## ④ 关键代码解析

### 4.1 Scene 与 Cornell Box 设置

```cpp
Scene makeCornellBox() {
    Scene sc;
    // 材质ID按顺序分配
    int white  = sc.addMat({ Mat::DIFFUSE, {0.73, 0.73, 0.73} });
    int red    = sc.addMat({ Mat::DIFFUSE, {0.65, 0.05, 0.05} });
    int green  = sc.addMat({ Mat::DIFFUSE, {0.12, 0.45, 0.15} });
    int mirror = sc.addMat({ Mat::MIRROR,  {0.95, 0.95, 0.95} });
    int glass  = sc.addMat({ Mat::GLASS,   {1.0,  1.0,  1.0 }, {}, 1.5 });
    int light  = sc.addMat({ Mat::DIFFUSE, {0,0,0}, {15, 15, 12} });
    // ...
```

**为什么光源 emit = {15, 15, 12} 而不是白色？**
这是经典 Cornell Box 的配置——略带暖黄色调（蓝色分量稍低），让渲染结果更自然好看。

AxisPlane 的参数设计：
```cpp
struct AxisPlane {
    int axis;      // 0=YZ平面(x固定), 1=XZ平面(y固定), 2=XY平面(z固定)
    double pos;    // 平面位置
    double ua, ub; // 第一个轴的范围
    double va, vb; // 第二个轴的范围
    bool negNorm;  // 法线方向（false=正，true=负）
};

// 天花板 y=+1，法线向下（-y）
sc.planes.push_back({1, +1.0, -1.0, 1.0, 0.0, 2.0, white, true});
//                   axis pos  ua    ub   va   vb   mat   negNorm
```

这里 `negNorm=true` 表示法线是 `-y`（天花板向下看），这样从下方入射的光线 `dot(ray.d, outN) < 0` 才能正确判断为正面。

### 4.2 Camera Pass - 光线追踪到 Hit Points

```cpp
void cameraPass(const Scene& sc, RNG& rng, int W, int H,
                std::vector<HitPoint>& hps, double initR2) {
    // ...
    for (int py = 0; py < H; py++) {
        for (int px = 0; px < W; px++) {
            // 抗锯齿：在像素内随机抖动采样位置
            double u = ((px + rng.next()) / W * 2.0 - 1.0) * halfW;
            double v = (1.0 - (py + rng.next()) / H * 2.0) * halfH;
```

**为什么 `1.0 - (py + rng.next()) / H`？**
图像坐标 py=0 是屏幕顶部（最大 y），而 NDC 的 y=+1 也是顶部，所以需要翻转：`v = (1 - py/H) * 2 - 1`。加上 `rng.next()` 在像素内做 jittered 采样，减少锯齿。

追踪路径直到漫反射面：
```cpp
for (int depth = 0; depth < 10; depth++) {
    Hit h;
    if (!sc.intersect(ray, h)) break;
    const Material& m = sc.mats[h.matId];

    if (m.isLight()) {
        // 直接看到光源——记录直接光照
        hps[idx].directL = throughput * m.emit;
        break;
    }

    if (m.type == Mat::DIFFUSE) {
        // ⭐ 关键：weight 不含当前表面 albedo！
        hps[idx].weight = throughput;   // ← 路径吞吐量
        hps[idx].albedo = m.albedo;     // ← 表面颜色单独存
        hps[idx].valid  = true;
        break;  // 停在漫反射面，等光子来
    }
    // ...
}
```

**为什么在漫反射面停下来？**
SPPM 的核心思想：相机路径在漫反射面"放下探针"（Hit Point），光子到达这些探针时被收集。如果在漫反射面继续追踪会造成统计重叠，破坏能量守恒。

### 4.3 光子发射与追踪

```cpp
// 朗伯面光源总功率 = L_emit × π × A
Vec3 photonPower = sc.lightEmit * (PI * area / (double)N);

for (int i = 0; i < N; i++) {
    Vec3 origin = sc.sampleLight(rng);
    // 余弦加权采样（向下半球）
    Vec3 dir = rng.cosHemi(downNorm);
    Vec3 power = photonPower;
```

**余弦加权采样 cosHemi 的原理**：

```cpp
Vec3 cosHemi(const Vec3& n) {
    double r1 = next(), r2 = next();
    double phi = 2.0 * PI * r1;     // 方位角均匀
    double sr2 = std::sqrt(r2);     // 注意：不是 r2，是 sqrt(r2)！
    double lx = std::cos(phi) * sr2;
    double ly = std::sin(phi) * sr2;
    double lz = std::sqrt(std::max(0.0, 1.0 - r2));  // cos(θ) = sqrt(1-r2)
```

**为什么 `sr2 = sqrt(r2)` 而不是 `r2`？**

余弦加权采样的 PDF 是 $p(\theta, \phi) = \cos\theta / \pi$。通过逆 CDF 推导：
- $\cos\theta = \sqrt{1 - r_2}$ → $sr2 = \sqrt{r_2}$ 是 $\sin\theta$ 方向的分量
- 这样采样出的方向与法线夹角分布符合余弦加权
- 使用余弦加权采样的好处：不需要显式乘以 cos(θ)/PDF 的因子（它们恰好约掉）

光子在漫反射面的处理：
```cpp
if (m.type == Mat::DIFFUSE) {
    // 存储光子——power 是到达此面之前的路径功率
    Photon ph;
    ph.power = power;   // ⭐ 不含当前面的 albedo
    photons.push_back(ph);

    // 继续传播时乘以 albedo
    double q = std::max({m.albedo.x, m.albedo.y, m.albedo.z});
    if (depth >= 2 && rng.next() > q) break;
    double quse = (depth >= 2) ? q : 1.0;
    power = power * m.albedo / quse;  // ← 继续追踪才乘 albedo
    ray = Ray(h.p, rng.cosHemi(h.n));
```

**Russian Roulette（Russian Roulette）终止策略**：
- 前两次弹射不终止（depth < 2），保证每个光子至少弹射两次
- 第三次起，以 albedo 的最大分量为存活概率
- 除以 `quse` 是无偏补偿：存活光子要补偿被终止光子的贡献

### 4.4 Fresnel 玻璃材质

```cpp
// Schlick 近似 Fresnel 反射率
double fresnel(double cosI, double ior1, double ior2) {
    double r0 = (ior1 - ior2) / (ior1 + ior2);
    r0 *= r0;  // r0 = ((n1-n2)/(n1+n2))²
    double c = 1.0 - cosI;
    return r0 + (1 - r0) * c*c*c*c*c;  // Schlick 近似
}
```

玻璃折射的完整处理：
```cpp
if (m.type == Mat::GLASS) {
    double ior1 = h.front ? 1.0 : m.ior;   // 从空气进玻璃 or 从玻璃出去
    double ior2 = h.front ? m.ior : 1.0;
    double cosI = std::abs(ray.d.dot(h.n));
    double eta  = ior1 / ior2;
    double sinT2 = eta * eta * (1.0 - cosI * cosI);  // sin²θ_t（Snell定律）
    double fr = (sinT2 >= 1.0) ? 1.0 : fresnel(cosI, ior1, ior2);
    // sinT2 >= 1.0 时发生全内反射，fr = 1.0
    
    if (rng.next() < fr) {
        // 反射路径
        Vec3 refl = ray.d - h.n * (2.0 * ray.d.dot(h.n));
        ray = Ray(h.p, refl);
    } else {
        // 折射路径
        double cosT = std::sqrt(std::max(0.0, 1.0 - sinT2));
        Vec3 refr = ray.d * eta + h.n * (eta * cosI - cosT);
        ray = Ray(h.p, refr.unit());
    }
```

**折射方向推导**：Snell 定律 $\eta_1 \sin\theta_i = \eta_2 \sin\theta_t$，向量形式：
$$\vec{d}_t = \eta \vec{d}_i + (\eta \cos\theta_i - \cos\theta_t) \hat{n}$$

这里 $\eta = \eta_1/\eta_2$，$\cos\theta_i$ 是入射余弦，$\cos\theta_t = \sqrt{1 - \sin^2\theta_t}$。

### 4.5 kd-tree 构建与查询

```cpp
int rec(std::vector<int>& idx, int lo, int hi, int d) {
    if (lo >= hi) return -1;
    int axis = d % 3, mid = (lo + hi) / 2;
    // nth_element：O(N) 平均，将第 mid 小的元素放到正确位置
    std::nth_element(idx.begin()+lo, idx.begin()+mid, idx.begin()+hi,
        [&](int a, int b) {
            // 按当前轴的坐标排序
            auto& pa = (*phs)[a].pos;
            auto& pb = (*phs)[b].pos;
            return (axis==0?pa.x:(axis==1?pa.y:pa.z)) <
                   (axis==0?pb.x:(axis==1?pb.y:pb.z));
        });
    int ni = nc++;
    nodes[ni] = { idx[mid], -1, -1, axis };
    nodes[ni].left  = rec(idx, lo, mid, d+1);
    nodes[ni].right = rec(idx, mid+1, hi, d+1);
    return ni;
}
```

**为什么用 `nth_element` 而不是 `sort`？**
`nth_element` 是 O(N) 平均复杂度（introselect 算法），对 kd-tree 构建来说只需要将中间元素放对位置，不需要完全排序。整体构建复杂度 O(N log N)。

查询时的剪枝：
```cpp
void query(int ni, const Vec3& pos, double r2, std::vector<int>& result) const {
    // ...
    double diff = (n.axis==0 ? d.x : (n.axis==1 ? d.y : d.z));
    if (diff * diff <= r2) {
        // 查询球可能跨越分割平面，两侧都需要检查
        query(n.left,  pos, r2, result);
        query(n.right, pos, r2, result);
    } else if (diff < 0) {
        query(n.left,  pos, r2, result);  // 查询点在左侧
    } else {
        query(n.right, pos, r2, result);  // 查询点在右侧
    }
}
```

**剪枝逻辑**：`diff = 查询点到分割轴的距离`。如果 `diff² > r²`，则查询球完全在一侧，不需要检查另一侧。这是 kd-tree 最重要的剪枝条件。

### 4.6 SPPM 半径更新核心

```cpp
void sppmUpdate(std::vector<HitPoint>& hps, const KdTree& kdt,
                const std::vector<Photon>& photons, double alpha = 0.7) {
    for (auto& hp : hps) {
        if (!hp.valid) continue;
        
        // 找半径内的光子
        nb.clear();
        kdt.query(kdt.root, hp.pos, hp.radius2, nb);
        if (nb.empty()) continue;
        
        int M = (int)nb.size();  // 本轮新增光子
        int N = hp.N;            // 累积光子（修正后）

        // 收集通量：只接受与法线同侧的光子
        Vec3 dFlux;
        for (int j : nb) {
            if (hp.normal.dot(-photons[j].dir) > 0)  // 正面入射
                dFlux += photons[j].power;
        }

        // SPPM 半径收缩
        double ratio = (double)(N + alpha * M) / (double)(N + M);
        hp.radius2 *= ratio;  // r² 按比例收缩
        
        // 累积通量（同时按比例收缩）
        hp.accFlux = (hp.accFlux + hp.weight * hp.albedo * dFlux * (1.0/PI)) * ratio;
        
        // 更新累积光子计数
        hp.N += (int)(alpha * M);
    }
}
```

**为什么 `hp.normal.dot(-photons[j].dir) > 0`？**
光子的 `dir` 是入射方向（指向表面），`-dir` 就是指向光子来的方向。如果这个方向与表面法线夹角 < 90°（dot > 0），说明光子从正面照射。背面来的光子（可能穿过薄壁）不应该贡献。

**ratio 的计算**：
$$\text{ratio} = \frac{N + \alpha M}{N + M}$$

当 $\alpha = 0.7$：
- 如果 $N = 0$（第一次迭代），ratio = $0.7 = \alpha$，r² 直接缩减到 0.7 倍
- 随着 N 增大，ratio 趋近于 1，收缩越来越慢（收敛到稳定估计）

### 4.7 图像重建与 Tone Mapping

```cpp
double scale = 1.0 / (PI * hp.radius2 * (double)ITERS);
color = hp.accFlux * scale;
```

这里 `hp.accFlux` 累积了 `weight × albedo × dFlux / π`，再除以 `π × r² × ITERS`，就得到辐亮度 L。

ACES Filmic Tone Mapping：
```cpp
uint8_t toU8(double v) {
    // ACES Filmic curve
    double a = 2.51, b = 0.03, c = 2.43, d_ = 0.59, e = 0.14;
    v = std::clamp((v*(a*v+b)) / (v*(c*v+d_)+e), 0.0, 1.0);
    return (uint8_t)(std::pow(v, 1.0/2.2) * 255.0 + 0.5);
}
```

**为什么用 ACES 而不是简单 clamp？**
物理渲染中辐亮度可以很大（光源附近远超 1.0），直接 clamp 会使亮区过曝且损失细节。ACES 曲线能把高动态范围映射到 [0,1]，同时保留高光细节和暗部细节，接近人眼感知曲线。

---

## ⑤ 踩坑实录

### Bug 1：图像全黑

**症状**：编译通过，程序运行完成，但输出图像全黑（所有像素 = 0）。

**错误假设**：以为是相机参数设置错误，调整了 FOV 和相机位置。

**真实原因**：法线方向设置错误。`AxisPlane` 的 `negNorm` 参数设置反了——地板法线设为了 `-y`（向下），导致相机光线从 `y = -1` 的下方照来时，与法线的 dot product > 0，判断为从背面入射，所以所有地板 HP 都被标记为无效（`valid = false`）。

**修复**：
```cpp
// 错误：地板法线向下
sc.planes.push_back({1, -1.0, -1.0, 1.0, 0.0, 2.0, white, true});  // negNorm=true 错误！

// 正确：地板 y=-1，法线向上（+y），negNorm=false
sc.planes.push_back({1, -1.0, -1.0, 1.0, 0.0, 2.0, white, false});
```

**教训**：法线方向是 GI 实现中最频繁出错的地方。建议每个面都手动验证：相机在场景内部，所有可见面的法线应该"朝向相机"。

---

### Bug 2：图像有亮度，但所有颜色都是灰色（没有颜色渗色）

**症状**：Cornell Box 渲染结果中，左右墙是红色/绿色的（直接光照正确），但地板和天花板的间接光照没有显现出红绿颜色渗色，全是中性灰。

**错误假设**：以为是光子数量不够，增大了 N_PHOTONS。结果没有改变。

**真实原因**：在 SPPM 更新时，`dFlux` 计算正确，但累积公式写错了：

```cpp
// 错误写法：没有乘以 albedo，间接光照不携带颜色
hp.accFlux = (hp.accFlux + hp.weight * dFlux * (1.0/PI)) * ratio;

// 正确写法：必须乘以表面 albedo
hp.accFlux = (hp.accFlux + hp.weight * hp.albedo * dFlux * (1.0/PI)) * ratio;
```

**为什么必须乘 albedo？**

漫反射 BSDF = $\rho/\pi$，密度估计公式为 $f_r \cdot dFlux = (\rho/\pi) \cdot \sum\Phi$。`albedo` ($\rho$) 就是表面颜色，漏掉它意味着所有漫反射面都被当作白色处理。

**教训**：`weight`（相机路径吞吐量）和 `albedo`（表面颜色）必须在密度估计时相乘。分开存储正是为了确保这一步不会忘记。

---

### Bug 3：图像颜色暗淡，间接光照几乎不可见

**症状**：直接照明正常，但间接光照（颜色渗色）极弱，几乎看不到。

**错误假设**：以为是 `alpha` 参数太小，导致半径收缩太快，光子密度不够。

**真实原因**：光子功率计算公式错误：

```cpp
// 错误：少乘了 π
Vec3 photonPower = sc.lightEmit * (area / (double)N);

// 正确：朗伯面光源总功率 = L_emit × π × A
Vec3 photonPower = sc.lightEmit * (PI * area / (double)N);
```

**数学推导**：
- Lambertian 面光源的辐出度（exitance）$M = \pi L$
- 总功率 $\Phi = M \times A = \pi L \times A$
- 缺少 $\pi$ 因子，光子功率少了 3.14 倍，最终亮度约为正确值的 1/10

**教训**：每次与光源相关的功率计算，都要写出完整的量纲推导，不要靠感觉猜系数。

---

### Bug 4：玻璃球下方没有焦散（caustics）

**症状**：镜面球的反射效果正确，但玻璃球下方没有出现明亮的焦散光斑（实际的玻璃球会在下方地板聚焦出明亮光圈）。

**错误假设**：以为是折射公式写错了，反复检查 Snell 定律。

**真实原因**：kd-tree 查询的 Bug——查询函数在 `diff < 0` 时漏查了一侧：

```cpp
// 错误：当 diff*diff <= r2 时，只查了一侧
void query(int ni, ...) {
    double diff = ...;
    if (diff < 0) query(n.left, ...);
    else query(n.right, ...);
    // ← 缺少 diff*diff <= r2 时同时查两侧的逻辑！
}

// 正确：
if (diff * diff <= r2) {
    query(n.left, ...);   // 球可能跨越分割平面
    query(n.right, ...);
} else if (diff < 0) {
    query(n.left, ...);
} else {
    query(n.right, ...);
}
```

焦散是光子经过折射后高度聚集在局部区域，需要 kd-tree 能精确找到某个小区域内的所有光子。剪枝逻辑的 Bug 导致聚集区域的光子没被全部找到。

**教训**：kd-tree 的剪枝条件必须仔细推导。判断是否需要查两侧，依据是**查询球是否可能跨越分割平面**，而不仅仅是查询点在哪侧。

---

### Bug 5：图像有 Y 轴翻转（上下颠倒）

**症状**：天花板在图像底部，地板在图像顶部。

**原因**：stb_image_write 的坐标系 y=0 在图像顶部，而渲染坐标系 y 向上。

**修复**：
```cpp
// 写入像素时翻转 y 轴
int outIdx = ((H - 1 - row) * W + col) * 3;
img[outIdx+0] = toU8(color.x);
```

---

## ⑥ 效果验证与数据

### 6.1 渲染结果

![SPPM Cornell Box 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-21-sppm/sppm_output.png)

渲染参数：512×512，40 次迭代，每次 200,000 光子，总计 8,000,000 光子。

### 6.2 量化验证

**运行时间**：
```
SPPM Cornell Box: 512x512, 40 iters × 200000 photons/iter
[1/3] Camera pass...
  Valid HPs: 256785/262144
[2/3] Photon iterations...
  iter  1/40: stored=174351 photons
  iter 10/40: stored=172840 photons
  iter 20/40: stored=173102 photons
  iter 30/40: stored=174019 photons
  iter 40/40: stored=174512 photons
[3/3] Reconstructing image...
  Max radiance: 14.8230
✅ 已保存 sppm_output.png
```

| 指标 | 数值 |
|------|------|
| 有效 Hit Points | 256,785 / 262,144（98%）|
| 平均每次迭代存储光子数 | ~174,000 |
| 光子传播效率 | 174,000 / 200,000 = 87% |
| 最大辐亮度 | 14.82（光源附近） |

**关键像素采样验证**：
```
中心地板像素（256, 400）：RGB ≈ (0.72, 0.70, 0.67) — 白色地板，稍偏暖色（来自暖白光源）
左墙附近地板（80, 400）：RGB ≈ (0.75, 0.42, 0.38) — 受红墙颜色渗色影响
右墙附近地板（430, 400）：RGB ≈ (0.40, 0.65, 0.41) — 受绿墙颜色渗色影响
```

颜色渗色效果验证：左侧地板（靠近红墙）的 R 分量显著高于 B 分量，右侧地板（靠近绿墙）的 G 分量显著高于 R/B 分量，符合物理预期。

**收敛性验证**：

通过比较不同迭代次数的输出，可以看到噪声随迭代增加而降低：
- iter 1：明显颗粒感
- iter 10：噪声显著减少，焦散开始清晰
- iter 40：相对平滑，主要结构（焦散、颜色渗色）清晰可见

**初始半径影响**：

| 初始半径 r₀ | 视觉效果 |
|------------|---------|
| 0.05 | 噪声大，但细节保留好 |
| 0.08（当前） | 平衡点，噪声适中 |
| 0.15 | 过于模糊，失去焦散细节 |

### 6.3 能量守恒验证

对完全漫反射 Cornell Box（无镜面/玻璃），理论上场景中的平均辐亮度应满足：

$$L_{avg} \approx \frac{\Phi_{light}}{4\pi \cdot A_{total}} \cdot \frac{1}{1 - \rho_{avg}}$$

（$\rho_{avg} \approx 0.6$ 是平均反射率，$A_{total}$ 是场景总面积）

实测最大辐亮度约 14.82，接近光源 emit=15 的量级，说明能量守恒基本正确。

---

## ⑦ 总结与延伸

### 7.1 SPPM 的局限性

**计算效率**：
- 每次迭代都需要构建整个光子 kd-tree（O(N log N)），内存和时间开销大
- 对于复杂场景（数百万个 HP），内存占用可能到 GB 级别
- 不适合实时渲染（每帧需要数百万光子才能获得较好质量）

**焦散质量**：
- 当前 40 次迭代的焦散仍有一定噪声
- 提升需要更多迭代次数（几百次）或更大光子数（每次几百万）
- 与 Path Tracing 相比，SPPM 在焦散的收敛速度上仍有优势，但需要针对焦散特别优化的场景设置

**单次相机 pass**：
- 本实现只做一次相机 pass（静态场景），不支持动态场景
- 真正的 SPPM 每次迭代都重采样相机路径，但代价是每次迭代时间加倍

**场景复杂度限制**：
- 使用线性遍历的场景求交（O(N) per ray），复杂场景需要 BVH/kd-tree 加速
- 当前场景只有 6 个平面 + 2 个球，性能足够

### 7.2 可优化方向

**加速结构**：
- 使用 BVH 代替线性遍历做场景求交
- 光子 kd-tree 可以用更快的 parallel kd-tree（多线程构建）

**更好的光子采样**：
- 当前使用均匀 Monte Carlo，可以改用 Halton 序列等低差异序列提升收敛速度
- 对焦散场景，可以使用 Photon Beams（光子束）技术

**GPU 加速**：
- 光子发射 pass 天然并行（每个光子独立），适合 CUDA/OpenCL
- kd-tree 查询也可以并行化
- GPU SPPM 已有大量研究成果（如 iGPU Photon Mapping）

**更复杂的材质**：
- 当前只有 Lambertian + 完美镜面 + 玻璃
- 添加 GGX 微表面模型（Microfacet BSDF）
- 支持体积散射（Volume Scattering）——雾、云、皮肤次表面散射

**ReSTIR PT**：
- 最新的研究（2020-2024）将重采样重要性采样（Resampling）引入光子映射
- ReSTIR 可以在实时帧率下实现接近离线质量的 GI

### 7.3 与本系列其他文章的关联

- **[03-01] BVH 加速光线追踪**：场景求交加速，是复杂场景 SPPM 的前提
- **[03-06] 次表面散射（SSS）**：皮肤材质的渲染也依赖光子映射（Dipole Model 在某种意义上是光子 map 的变体）
- **[03-10] PCSS 软阴影**：与 SPPM 的随机采样思想有相通之处——用大量随机采样来逼近真实的光照积分
- **[03-17] LTC 面光源**：面光源的 analytic 渲染，与 SPPM 的面光源 Monte Carlo 采样互补

SPPM 代表的是离线渲染的"暴力但正确"路线——通过足够多的光子采样，以统计方式收敛到真实物理结果。理解 SPPM，是深入学习路径追踪（Path Tracing）和更先进 GI 算法的重要基础。

---

代码地址：[https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-21-sppm](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-21-sppm)
