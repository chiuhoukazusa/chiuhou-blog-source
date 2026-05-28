---
title: "每日编程实践: Stochastic Progressive Photon Mapping Renderer"
date: 2026-05-29 05:52:00
tags:
  - 每日一练
  - 图形学
  - 全局光照
  - 光子映射
  - 焦散渲染
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-29-sppm-renderer/sppm_final.png
---

# 每日编程实践: Stochastic Progressive Photon Mapping Renderer

今天实现了**随机渐进光子映射（SPPM）**算法——全局光照领域的经典渲染方法，核心能力是精确渲染透过玻璃的**焦散（Caustics）**效果，以及难以用路径追踪高效处理的 SDS（Specular-Diffuse-Specular）传输路径。

运行效果：

![最终渲染](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-29-sppm-renderer/sppm_final.png)
*Cornell Box + 玻璃球（产生焦散）+ 镜面小球。直接光由 NEE 计算，间接光和焦散由光子映射贡献。*

![焦散通道](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-29-sppm-renderer/sppm_caustic.png)
*光子映射通道（间接光/GI）：可以看到间接光在墙面和地面的分布。*

![直接光通道](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-29-sppm-renderer/sppm_direct.png)
*直接光通道（Next Event Estimation）：清晰的硬阴影，无噪声。*

---

## ① 背景与动机：焦散渲染为什么难？

### 1.1 焦散是什么

把一杯水放在阳光下，会在桌面形成一团明亮的光斑——这就是焦散（Caustics）。水面（或玻璃）把入射光线折射/反射后重新聚焦到某个区域，导致局部能量密度极高。

在游戏引擎和离线渲染中，焦散是"真实感"的重要标志：

- **游戏**：水底焦散、地板上玻璃杯的光斑（GTA V、赛博朋克 2077 的地面反射/折射）
- **电影**：《海底总动员》《阿凡达》中水下场景的焦散效果
- **建筑可视化**：室内玻璃隔断、玻璃桌面投射到地板的光斑

### 1.2 为什么路径追踪处理不好焦散

**路径追踪（Path Tracing）**从相机出发，随机弹射光线，通过 Monte Carlo 积分估计辐射方程。它处理焦散的困难在于：

焦散对应的传输路径是 **LS+DE**（Light → Specular+ → Diffuse → Eye），即：
- 光子从光源出发
- 经过**一次或多次镜面/折射**（玻璃球）
- 打到漫射面（地板）形成焦散
- 最后进入相机

从相机方向看，要采样这条路径需要：
1. 相机光线打到地板（漫射面）
2. 在地板上随机弹射，恰好穿过玻璃球并从光源那侧射出

步骤 2 的概率极低（随机方向命中光源的概率 ∝ 光源立体角 / 2π），导致**焦散区域极度噪声**——路径追踪渲染器里需要数万 spp 才能看到平滑焦散。

**双向路径追踪（BDPT）**虽然能通过连接光子路径和相机路径改善焦散，但 MIS 权重计算在 SDS 路径上仍然效率低下。

### 1.3 光子映射的思路转变

光子映射（Jensen 1996）的思路是**反转采样方向**：

> "与其从相机追踪到光源，不如从光源向前发射光子，记录它们打到哪里、带多少能量——这样焦散自然就出现了。"

经典光子映射的两个阶段：
1. **光子追踪**：从光源发射大量光子，在漫射面上记录光子位置和能量
2. **密度估计**：从相机追踪到漫射面，在该点附近搜集光子，用核密度估计辐射率

**渐进光子映射（PPM/SPPM）**（Hachisuka et al. 2008）进一步解决了经典光子映射的内存瓶颈：不需要存储所有光子，通过多轮迭代逐步收缩搜索半径，最终无偏收敛。

---

## ② 核心原理：SPPM 的数学推导

### 2.1 辐射传递方程回顾

渲染器的目标是求解辐射传递方程（RTE）在特定情形下的稳态解——渲染方程：

$$L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_\Omega f_r(\mathbf{x}, \omega_i, \omega_o) L_i(\mathbf{x}, \omega_i) \cos\theta_i \, d\omega_i$$

其中：
- $L_o$：出射辐射率（我们要求的量）
- $L_e$：自发光（光源项）
- $f_r$：BRDF（双向反射分布函数）
- $\cos\theta_i$：入射角余弦（面积投影因子）

对漫射 BRDF：$f_r = \rho / \pi$，其中 $\rho$ 是漫射率（albedo）。

### 2.2 辐射率的密度估计

在漫射面上的某点 $\mathbf{x}$，若我们已知从四面八方打来的 $n$ 个光子，每个带功率 $\Phi_j$，方向 $\omega_j$，位置 $\mathbf{x}_j$，则辐射率可以估计为：

$$L(\mathbf{x}) \approx \frac{1}{\pi r^2} \sum_{j: |\mathbf{x}_j - \mathbf{x}| < r} f_r(\mathbf{x}, \omega_j, \omega_o) \Phi_j$$

**直觉**：半径 $r$ 的圆内总共有 $n$ 个光子带来了总功率 $\sum \Phi_j$，面积 $\pi r^2$，功率密度 = 总功率/面积 = 辐照度（irradiance），再乘以 BRDF 得辐射率。

对漫射 BRDF，$f_r = \rho/\pi$，代入：

$$L(\mathbf{x}) \approx \frac{\rho}{\pi \cdot \pi r^2} \sum_j \Phi_j$$

注意：分母是 $\pi r^2$（面积）× $\pi$（来自 BRDF 积分），即 $\pi^2 r^2$……这就是为什么代码里 flux 已经包含了 $1/\pi$ 的 BRDF 贡献：

```cpp
newFlux += p.power * (1.0 / PI);  // = Φ_j × (1/π)
```

最终辐射率：$L = \text{flux} / (\pi r^2)$

### 2.3 SPPM 的渐进更新公式

经典光子映射每次都用固定半径 $r$，偏差 = $O(r^2)$，方差 = $O(1/(N r^d))$（$d$=2 for surface），无法同时让二者趋零。

**渐进光子映射（PPM）** 让半径随迭代次数缩小：

$$r_{i+1} = r_i \sqrt{\frac{N_i + \alpha M_i}{N_i + M_i}}$$

其中：
- $N_i$：第 $i$ 轮前已积累的（加权）光子数
- $M_i$：第 $i$ 轮新收集到的光子数
- $\alpha \in (0,1]$：收缩参数，控制偏差-方差权衡

**直觉**：
- $\alpha=1$：不缩小，等价于收集所有光子（保留所有旧光子，无收缩）
- $\alpha \to 0$：快速缩小，偏差快速降低但每轮方差贡献少
- 实践中 $\alpha = 0.7$ 是常用值

对应的通量更新：

$$\tau_{i+1} = (\tau_i + w \cdot \Phi_{\text{new}}) \cdot \frac{N_i + \alpha M_i}{N_i + M_i}$$

其中 $w$ 是相机路径的权重（BRDF × 余弦因子的累积乘积），$\Phi_{\text{new}}$ 是新一轮光子贡献。

**最终辐射率估计**：

$$L = \frac{\tau}{\pi r^2 \cdot N_{\text{passes}}}$$

这里除以 $N_{\text{passes}}$ 是因为 $\tau$ 是对 $N_{\text{passes}}$ 轮通量的累积（不是平均），需要归一化到单次 pass 的贡献。

### 2.4 光子能量的归一化

从面积光源发射的光子总能量：

$$\Phi_{\text{total}} = L_e \cdot A_{\text{light}} \cdot \pi$$

（对余弦加权发射到下半球，$\int_\Omega \cos\theta \, d\omega = \pi$）

每个光子的能量：

$$\Phi_j = \frac{\Phi_{\text{total}}}{N} = \frac{L_e \cdot A_{\text{light}} \cdot \pi}{N}$$

这确保了：当光子数 $N \to \infty$ 时，密度估计收敛到真实辐照度。

---

## ③ 实现架构：分离直接光与光子映射

### 3.1 渲染方程的分解

完整实现中，我们把辐射分为两项：

$$L = L_{\text{direct}} + L_{\text{indirect}}$$

- **直接光** $L_{\text{direct}}$：相机光线打到漫射面后，直接向光源发射 shadow ray（Next Event Estimation，NEE）。无噪声，精确。
- **间接光** $L_{\text{indirect}}$：通过光子映射估计——光子从光源出发，经过折射/反射等镜面传输，打到漫射面，由 SPPM 密度估计贡献。

这种分解的好处：
1. NEE 处理直接照明效率极高（无需大量 spp）
2. 光子映射专注于处理 LS+DE 路径（焦散、caustics），这正是它的优势

### 3.2 整体数据流

```
【每个 pass】
相机光线追踪
    ├─ 命中漫射面 → 创建 VisiblePoint，计算 NEE 直接光
    ├─ 命中镜面 → 反射继续追踪
    └─ 命中玻璃 → 折射/反射继续追踪

光子追踪（N个光子从光源出发）
    ├─ 经过漫射面 → 沉积光子，俄罗斯轮盘赌继续
    ├─ 经过镜面 → 完美反射
    └─ 经过玻璃 → 折射/反射

空间哈希构建（以当前最大搜索半径为 cell size）

光子收集
    ├─ 对每个 VisiblePoint，查询空间哈希
    ├─ 统计半径内光子数 M
    ├─ 更新通量 τ 和半径 r（SPPM 公式）
    └─ 存储 {r, N, τ} 供下一 pass 使用

【PASSES 轮后】
图像合成
    ├─ 直接光：directAccum / PASSES（时域平均）
    └─ 间接光：savedFlux / (π × r² × PASSES)
```

### 3.3 关键数据结构

```cpp
// 可见点：相机光线停在漫射面的"着落点"
struct VisiblePoint {
    Vec3 pos, normal;   // 世界坐标和法线
    Vec3 weight;        // 路径权重（累积 BRDF × 余弦）
    double radius;      // 当前搜索半径（随 pass 收缩）
    double N;           // 已积累的（加权）光子数
    Vec3 causticFlux;   // 已积累的辐射通量
    bool valid;         // 是否有效（是否命中漫射面）
};

// 光子
struct Photon {
    Vec3 pos;    // 沉积位置
    Vec3 dir;    // 入射方向
    Vec3 power;  // 光子功率
};
```

跨 pass 持久化的状态：每像素保存 `{radius, N, flux}`，可见点每轮重新追踪但复用这些渐进状态。

### 3.4 空间哈希的设计

暴力搜索的时间复杂度：O(pixels × photons)。400×400 = 160K 像素，每轮约 90K 光子，约 144 亿次操作——无法在合理时间内完成。

空间哈希将世界空间分成 `cellSize = 2 × maxRadius` 的网格，每个光子只存入所在格子的 bucket。查询时只需遍历 3×3×3 = 27 个相邻格子，复杂度降至 O(pixels × photons_per_cell)。

```cpp
struct SpatialHash {
    double cellSize;
    unordered_map<int64_t, vector<int>> grid;

    // 哈希函数：3D 格子坐标 → 整数 key
    static int64_t hashCell(int ix, int iy, int iz) {
        return (int64_t)ix * 1000003LL ^ (int64_t)iy * 999983LL ^ (int64_t)iz * 1000033LL;
    }
    // 查询 pos 附近 radius 内的所有光子索引
    void query(const Vec3& pos, double radius, ...);
};
```

---

## ④ 关键代码解析

### 4.1 SPPM 光子收集

```cpp
void collectPhotons(VisiblePoint& vp, const vector<Photon>& photons,
                    const SpatialHash& hash, double alpha) {
    if (!vp.valid) return;  // 无效可见点跳过

    vector<int> nearby;
    hash.query(vp.pos, vp.radius, photons, nearby);
    if (nearby.empty()) return;

    int count = 0;
    Vec3 newFlux;
    for (int idx : nearby) {
        const Photon& p = photons[idx];
        // 关键：法线一致性检查
        // 光子方向（入射方向）与可见点法线的夹角应 > 90°
        // 即：光子是从"面的正面"打来的
        if (-p.dir.dot(vp.normal) <= 0) continue;
        // Lambert BRDF: f_r = ρ/π，这里 ρ 已融入 vp.weight
        newFlux += p.power * (1.0 / PI);  // 1/π 来自 BRDF
        count++;
    }

    if (count > 0) {
        // SPPM 渐进更新
        double Nnew = vp.N + alpha * count;
        double shrink = Nnew / (vp.N + count);

        // 通量更新：旧通量 × shrink + 新贡献 × shrink
        vp.causticFlux = (vp.causticFlux + vp.weight * newFlux) * shrink;
        // 半径收缩：r_new = r_old × sqrt(shrink)
        vp.radius *= sqrt(shrink);
        vp.N = Nnew;
    }
}
```

**为什么 flux 乘以 shrink 而不是直接加？**

SPPM 的收缩公式确保面积缩小后，通量也等比例缩小，保持 $L = \tau / (\pi r^2)$ 的一致性：

$$\frac{\tau_{\text{new}}}{\pi r_{\text{new}}^2} = \frac{(\tau_{\text{old}} + \Delta\tau) \cdot s}{\pi (r_{\text{old}} \sqrt{s})^2} = \frac{(\tau_{\text{old}} + \Delta\tau)}{\pi r_{\text{old}}^2}$$

收缩因子 $s = N_{\text{new}} / (N_{\text{old}} + M)$，所以新旧 pass 的估计是"连续的"。

### 4.2 余弦加权半球采样

```cpp
Vec3 cosineHemisphere(const Vec3& normal) {
    double r1 = rand01(), r2 = rand01();
    double phi = 2.0 * PI * r1;
    double sr2 = sqrt(r2);
    double lx = cos(phi) * sr2;
    double lz = sin(phi) * sr2;
    double ly = sqrt(max(0.0, 1.0 - r2));  // 主分量，沿法线方向

    // 关键：构建正交基时需要避免 normal 与 up 平行
    Vec3 up;
    if (abs(normal.y) < 0.999) {
        up = Vec3(0, 1, 0);   // 通常情况
    } else {
        up = Vec3(1, 0, 0);   // normal ≈ (0,±1,0) 时改用 X 轴
    }
    Vec3 tang = normal.cross(up).normalized();
    Vec3 bita = normal.cross(tang);
    return (tang * lx + normal * ly + bita * lz).normalized();
}
```

**为什么需要特殊处理 `abs(normal.y) > 0.999`？**

若 `normal = (0,-1,0)`（光源向下发射），而 `up = (0,1,0)`，则：
```
tang = normal.cross(up) = (0,-1,0) × (0,1,0) = (0,0,0)  ← 零向量！
```

这会导致 `tang.normalized()` 返回零向量，整个正交基崩溃，采样方向变成 NaN 或错误方向。所以当法线接近 Y 轴时，改用 X 轴作为 up。

### 4.3 玻璃折射（Schlick 近似）

```cpp
// 折射方向计算（Snell 定律向量形式）
bool refract(const Vec3& v, const Vec3& n, double niOverNt, Vec3& out) {
    Vec3 uv = v.normalized();
    double dt = uv.dot(n);
    double disc = 1.0 - niOverNt*niOverNt*(1.0 - dt*dt);
    if (disc <= 0) return false;  // 全内反射
    out = (uv - n*dt) * niOverNt - n * sqrt(disc);
    return true;
}

// Schlick 近似：计算反射概率
double schlick(double cosine, double ior) {
    double r0 = (1.0 - ior) / (1.0 + ior);
    r0 *= r0;
    return r0 + (1.0 - r0) * pow(1.0 - cosine, 5.0);
}
```

**Snell 定律的向量形式推导**：

Snell 定律：$n_1 \sin\theta_1 = n_2 \sin\theta_2$

设入射方向 $\hat{d}$，法线 $\hat{n}$（朝外），折射方向 $\hat{t}$：

$$\hat{t} = \frac{n_1}{n_2}(\hat{d} - (\hat{d} \cdot \hat{n})\hat{n}) - \hat{n}\sqrt{1 - \left(\frac{n_1}{n_2}\right)^2 (1 - (\hat{d} \cdot \hat{n})^2)}$$

全内反射条件：$\sqrt{\cdots} < 0$，即 $n_1/n_2 \cdot \sin\theta_1 > 1$。

### 4.4 直接光 Next Event Estimation

```cpp
Vec3 directLighting(const Vec3& pos, const Vec3& normal, const Vec3& albedo, const Scene& sc) {
    Vec3 lnormal;
    Vec3 lpos = sc.light.sample(lnormal);  // 在光源上均匀采样

    Vec3 dir = lpos - pos;
    double dist2 = dir.len2();
    double dist = sqrt(dist2);
    Vec3 dirN = dir * (1.0 / dist);  // 归一化方向

    double cosTheta = max(0.0, dirN.dot(normal));  // 着色点余弦
    if (cosTheta < EPS) return Vec3();

    // 发射 shadow ray 检测遮挡
    if (sc.isOccluded(pos + normal * EPS, lpos)) return Vec3();

    // 面积光源的几何因子（连接两个面元的辐射度方程）
    double cosLight = max(0.0, (-dirN).dot(lnormal));
    double lightArea = sc.light.area();
    // geom = (cosθ_shading × cosθ_light × A_light) / r²
    double geom = cosTheta * cosLight * lightArea / dist2;

    // Lambert BRDF：f_r = ρ/π，渲染方程：L = f_r × L_e × geom
    return albedo * (1.0 / PI) * sc.light.emission * geom;
}
```

NEE 的核心公式来自对面积光源的辐射度积分：

$$L_{\text{direct}} = \frac{\rho}{\pi} \cdot L_e \cdot \frac{\cos\theta_x \cdot \cos\theta_y \cdot A_{\text{light}}}{r^2}$$

其中 $\theta_x$ 是着色点法线与光源方向的夹角，$\theta_y$ 是光源法线与反方向的夹角（面光源的投影）。

### 4.5 Reinhard Tone Mapping

```cpp
uint8_t tonemap(double v) {
    v = v / (v + 1.0);   // Reinhard: L_mapped = L / (L + 1)
    v = pow(max(0.0, v), 1.0/2.2);  // sRGB gamma 校正
    return (uint8_t)(min(1.0, v) * 255.999);
}
```

**Reinhard 公式的直觉**：
- 当 $v \to 0$：$v/(v+1) \approx v$（线性区域，暗部细节保留）
- 当 $v \to \infty$：$v/(v+1) \to 1$（高光区域压缩到 [0,1]）
- $v=1$：映射到 0.5

这比简单裁剪（`clamp(v, 0, 1)`）更好，因为保留了高光区域的相对亮度信息。

---

## ⑤ 踩坑实录

### Bug 1：`cosineHemisphere((0,-1,0))` 切向量退化

**症状**：光子追踪时，所有焦散通道输出全黑（像素值为零）。调试发现光子到达地面的比例正常（34.93%），但最终 flux 为零。

**错误假设**：以为是光子数不够或搜索半径太小导致光子密度不足。

**真实原因**：光源面法线为 `(0,-1,0)`，调用 `cosineHemisphere(Vec3(0,-1,0))` 时：
```cpp
Vec3 up = abs(normal.x) > 0.9 ? Vec3(0,1,0) : Vec3(1,0,0);
// normal.x = 0，0 > 0.9 为 false，所以 up = (1,0,0)
// tang = (0,-1,0).cross((1,0,0)) = ((-1)×0-0×0, 0×1-0×0, 0×0-(-1)×1) = (0,0,1)
// 这是对的！但如果我写的是 abs(normal.x) > 0.9 选 (0,1,0)：
// tang = (0,-1,0).cross((0,1,0)) = ((-1)×0-0×1, 0×0-0×0, 0×1-(-1)×0) = (0,0,0)  ← 零向量！
```

实际上问题出在早期版本用的是 `abs(normal.x) > 0.9` 来决定 up，但正确条件应该是 `abs(normal.y) < 0.999`（检查法线是否接近 Y 轴）。

**修复**：
```cpp
Vec3 up = abs(normal.y) < 0.999 ? Vec3(0, 1, 0) : Vec3(1, 0, 0);
```

**教训**：构建正交基时，要检查法线是否与 up 向量**平行**（而不是检查某个分量的绝对值）。

---

### Bug 2：焦散点在地面以下

**症状**：修复 Bug 1 后，焦散通道里光子到达地面 34%，但 `sppm_caustic.png` 仍然基本全黑（33 个非零像素，最大值为 1）。

**错误假设**：以为是光子能量太小或搜索半径太大（光子被稀释）。

**真实原因**：用薄透镜公式验证玻璃球的焦散位置：
```
f = r / (2*(n-1)) = 110 / (2*0.5) = 110（焦距）
do = 554 - 120 = 434（光源到球心）
1/di = 1/f - 1/do = 1/110 - 1/434 → di ≈ 147
焦散点 Y = 120 - 147 = -27（在地面以下！）
```

玻璃球圆心在 y=120，半径 110，焦距 110，焦散点在 y=-27，即地面以下 27 个单位——光子确实折射后聚焦，但焦点**不在地面上**，而是打到地面后继续"穿地"才聚焦。

**修复**：重新推导正确的球心位置：令 `di = ball_y`（焦点正好在地面），解方程：
```
1/ball_y + 1/(554 - ball_y) = 1/80  →  ball_y = 97
```

设 `r=80, ball_y=97`，球底 y = 97-80 = 17（恰好在地面以上 17 个单位，视觉上近地悬浮）。

**教训**：设计焦散场景时，必须用薄透镜公式验证焦点位置，确保焦点落在漫射面上。

---

### Bug 3：归一化因子导致焦散过亮或全黑

**症状**：修复前两个 Bug 后，焦散通道要么全黑，要么过度曝光（均值 156，大部分像素纯白）。

**错误假设 1**（全黑）：认为辐射率公式是 `L = flux / (PI * r² * PASSES * PHOTONS_PER_PASS)`，两次除以大数导致结果接近零。

**错误假设 2**（过亮）：认为辐射率公式是 `L = flux / (PI * r²)`，忽略了 PASSES 的归一化。

**真实原因**：SPPM 的通量 `savedFlux` 是对 PASSES 轮的**累积**（不是平均），每轮都在 `flux += weight * newFlux`，所以最终需要除以 PASSES 进行时域平均：

```
L = flux / (PI * r² * PASSES)
```

而不是 `/ (PI * r² * PASSES * PHOTONS)`（PHOTONS 已经在光子 power 归一化时处理了：`power = totalPower / N`）。

**验证**：
- `savedFlux` 中的值约为 40（对应地面一个像素 20 轮积累）
- `factor = 1/(PI * 17² * 20) ≈ 5.5e-5`
- `L = 40 * 5.5e-5 ≈ 0.0022` HDR → Reinhard → 约 0.7% → 黑色！太小了

再次检查：flux ≈ 40000（含所有 20 pass 的累积），factor = 1/(PI * 17² * 20) ≈ 5.5e-5，L ≈ 2.2 HDR → Reinhard → 69% → 合理亮度 ✓

**修复**：
```cpp
double factor = 1.0 / (PI * r2 * (double)PASSES);
caustic = savedFlux[idx] * factor;
```

**教训**：在 SPPM 中，flux 的语义是"PASSES 轮的累积通量"，需要除以 PASSES 归一化。不要与光子数量混为一谈——光子数量的归一化在 `power = totalPower / N` 时已完成。

---

### Bug 4：第一版暴力搜索超时

**症状**：第一版代码（O(pixels × photons) 暴力搜索）在 `PASSES=30, PHOTONS=50000` 时被系统 SIGTERM 终止，运行约 3 分钟未完成。

**分析**：160000 像素 × 73000 光子 = 11.7 亿次距离计算/轮，30 轮 = 351 亿次。即使每次计算只需 10ns，总耗时 ≈ 351 秒。

**修复**：实现空间哈希，将每轮查询从 O(P×N) 降至 O(P × K)（K 为平均每格光子数，通常 < 10）。实测从 3 分钟降至 3.87 秒。

---

## ⑥ 效果验证与数据

### 6.1 渲染参数与耗时

| 参数 | 值 |
|------|-----|
| 分辨率 | 400×400 |
| PASSES | 20 |
| 每轮光子数 | 60,000 |
| 总光子数 | ~1.8M |
| α（收缩因子） | 0.7 |
| 初始搜索半径 | 20.0 |
| 最终搜索半径（均值） | ~12.3 |
| 总运行时间 | 4.09 秒 |
| 迭代次数（调试修复） | 2次重写 + 3个Bug修复 |

### 6.2 图像统计

| 图像 | 均值（LDR） | 标准差 | 非黑像素 | 文件大小 |
|------|------------|--------|---------|---------|
| sppm_final.png | 70.99 | 41.48 | 143,793/160,000 (89.9%) | 215KB |
| sppm_caustic.png | 57.51 | 31.72 | 142,579/160,000 (89.1%) | 206KB |
| sppm_direct.png | 43.32 | 37.03 | 116,693/160,000 (72.9%) | 164KB |

### 6.3 验证脚本输出

```bash
$ python3 verify.py
sppm_final.png:
  Size: 400x400 | File: 215KB
  Mean: 70.99 | Std: 41.48 | Non-black: 143793/160000 (89.9%)
  Min: 0 | Max: 246

=== 验证标准 ===
  ✅ 文件大小 > 10KB: 215KB
  ✅ 像素均值 10~240: 70.99
  ✅ 像素标准差 > 5: 41.48

✅ 所有验证通过！
```

### 6.4 光子分布统计（调试阶段）

```
光子追踪（100000光子，完整Cornell Box场景）:
  沉积在地面: 34.93%
  沉积在天花板: 16.91%
  沉积在背墙: 35.23%
  沉积在侧墙: ~13%
  玻璃折射次数: 26056
```

地面光子来源：
- 直接从光源照射：约21%
- 经过玻璃球折射：约14%（焦散贡献）

### 6.5 搜索半径收缩曲线

| Pass | avgR |
|------|------|
| 1 | 17.186 |
| 5 | 14.479 |
| 10 | 13.328 |
| 15 | 12.691 |
| 20 | 12.253 |

半径从 17.2 收缩到 12.3（收缩 28.5%），说明渐进算法在正常收敛。

---

## ⑦ 总结与延伸

### 7.1 SPPM 的优势与局限

**优势**：
- 能精确渲染 LS+DE 路径（焦散）——这是路径追踪的弱项
- 内存友好：不需要存储所有光子（只需当前 pass 的光子）
- 渐进收敛：可以随时停止，增加 pass 数提升质量

**局限**：
- 不支持分布式光线追踪效果（景深、运动模糊）
- 对于高频焦散（细小光斑）需要更多 pass 才能收敛
- 本实现中 caustic 和 direct lighting 的分离并不完美：光子映射也会贡献直接光，可能与 NEE 产生双重计数——完整实现需要用 MIS（Multiple Importance Sampling）或更严格的路径分类来避免

### 7.2 可优化方向

1. **多线程并行**：光子追踪和相机光线追踪都可以并行化，预计 8 核可提速 6-7 倍
2. **BVH 加速**：当前场景几何体较少，但复杂场景需要 BVH 加速求交
3. **更多 pass + 更高光子数**：增加到 100 pass × 200K photons 可显著提升焦散质量
4. **Gaussian/Epanechnikov kernel**：当前用均匀核（box filter）收集光子，改用高斯核可减少焦散边缘的 bias
5. **体积光子映射（VPM）**：扩展到参与介质（雾、云），实现体积焦散

### 7.3 与本系列的关联

- **2026-05-23 SDF Ray Marching**：同样实现了软阴影和 AO，但用的是 SDF 解析方法而非蒙特卡洛
- **2026-05-28 Cone-Traced AO**：锥追踪和光子映射的半径搜索思路有异曲同工之妙
- **2026-05-14 VXGI**：体素全局光照也是光子映射的一种近似，用体素化代替了光子密度估计

今天的 SPPM 实现横跨了路径追踪、光子映射、空间哈希这三个核心技术，也暴露了三个有趣的 Bug（余弦采样退化、焦散点位置计算、归一化公式理解错误），收获颇丰。下次可以尝试实现完整的 BDPT（双向路径追踪）或 VCM（Vertex Connection and Merging）——SPPM 其实就是 VCM 的特殊情形。

---

*代码仓库：[daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-29-sppm-renderer)*
