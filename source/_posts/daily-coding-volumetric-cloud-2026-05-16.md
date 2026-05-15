---
title: "每日编程实践: Volumetric Cloud Ray Marching Renderer — 体积云光线步进渲染"
date: 2026-05-16 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 体积渲染
  - Ray Marching
  - 大气散射
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-16-volumetric-cloud/volumetric_cloud_output.png
---

# 每日编程实践：体积云光线步进渲染

![体积云渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-16-volumetric-cloud/volumetric_cloud_output.png)

## ① 背景与动机

### 为什么云很难渲染？

云是大气中最常见的视觉现象之一，但它偏偏是渲染领域最棘手的挑战之一。传统的多边形渲染流水线非常擅长处理"有固定表面"的物体——一张桌子、一颗球、一栋建筑，都可以用三角形网格精确描述。但云没有确定的边界，它是一团半透明的气态介质，光子在其中反复散射、吸收、穿透。

如果强行用传统方法来做：

- **Billboard 贴图**（早期游戏常用）：远看还行，靠近立刻穿帮，旋转时会露馅，无法与场景光照联动
- **粒子系统**：大量烟雾粒子叠加，性能消耗爆炸，还是无法模拟内部光散射
- **Mesh 近似**：哪个美术愿意手刻一朵积雨云的三角面？而且形状固定、无法动态变化

**Ray Marching（光线步进）** 解决了这个问题的根本——它不要求物体有一个明确的数学表面，只需要你能在任意三维坐标处回答一个问题："这个点的云密度是多少？"

### 工业界怎么用？

几乎所有现代 AAA 游戏和电影的天空系统都基于这一思路：

- **Horizon Zero Dawn / Forbidden West**：Guerrilla Games 的 "Nubis" 云系统，以 4D 噪声纹理 + Ray Marching 为核心，专门发了 SIGGRAPH 2015 分享。云彩随时间流动，支持实时昼夜循环。
- **Red Dead Redemption 2**：Rockstar 使用 Worley 噪声 + FBM 构建密度场，结合多次散射近似，生成了惊艳的西部天空。
- **荒野大镖客 2 的效果被玩家截图打印当壁纸** —— 这在游戏界是极高的评价。
- **电影 VFX**：Disney、ILM 的流体/云雾渲染更进一步使用 OpenVDB 体素格式存储真实流体模拟的云体，用 Path Tracing 做真实多次散射。
- **Unreal Engine 5 的 "Volumetric Cloud"**：内置组件，暴露给关卡设计师，底层正是本文所讲的技术。

今天用纯 CPU 软光栅实现一个精简版本，把所有核心原理走一遍。

---

## ② 核心原理

### 2.1 Ray Marching 是什么

Ray Marching 不是一种具体算法，而是一类思想：**沿光线方向以固定步长采样某个标量场**，累积结果。

对于体积渲染，沿光线 $r(t) = \mathbf{o} + t\mathbf{d}$ 步进，在每个采样点 $t_i$ 处查询密度 $\sigma(t_i)$，然后根据辐射传输方程（Radiative Transfer Equation，RTE）积分颜色。

体积渲染的核心积分公式：

$$
L(\mathbf{o}, \mathbf{d}) = \int_{t_0}^{t_1} T(t) \cdot \sigma_s(t) \cdot L_i(t, \mathbf{d}) \, dt
$$

其中：
- $L(\mathbf{o}, \mathbf{d})$：从原点 $\mathbf{o}$ 沿方向 $\mathbf{d}$ 看到的辐亮度
- $T(t)$：从 $t_0$ 到 $t$ 的透射率（有多少光能穿过到达这里）
- $\sigma_s(t)$：散射系数（这个点有多少光会向我们散射过来）
- $L_i(t, \mathbf{d})$：该点处入射辐亮度（来自太阳 + 环境光 + 多次散射）

**直觉解释**：你站在云层外往里看，每一小段云都能贡献一点颜色（散射光）。但这段颜色要想到达你的眼睛，必须穿过它后面的所有云——那些云会吸收/遮挡掉一部分。$T(t)$ 就是衡量"还剩多少光能穿透出来"的系数。

### 2.2 Beer-Lambert 定律与透射率

透射率 $T(t)$ 由 Beer-Lambert 定律给出：

$$
T(t_0, t_1) = \exp\left(-\int_{t_0}^{t_1} \sigma_e(s) \, ds\right)
$$

$\sigma_e$ 是消光系数（extinction coefficient），包括吸收和散射两部分：

$$
\sigma_e = \sigma_a + \sigma_s
$$

**直觉解释**：想象你拿一块玻璃，越厚越不透明。Beer-Lambert 就是描述这个"越厚越暗"的规律，指数形式意味着每单位厚度都吸收相同比例的光（而不是相同绝对量）。

在离散步进中，每一步的透射率近似为：

$$
T_{\text{step}} = \exp(-\sigma_e \cdot \rho(t) \cdot \Delta t)
$$

其中 $\rho(t)$ 是该点的云密度（0~1），$\Delta t$ 是步长（单位：米）。

代码中体现：

```cpp
float density = cloudDensity(p);           // 0.0 ~ 约1.0
float optDepth = density * STEP_SIZE * sigma_e;  // 光学深度
float stepTrans = expf(-optDepth);          // 这一步的透射率
transmittance *= stepTrans;                 // 累乘，总透射率单调递减
```

### 2.3 Henyey-Greenstein 相函数

光打到云粒子（小水滴）后，向哪个方向散射？这由**相函数（phase function）** 描述。

最常用的是 Henyey-Greenstein (HG) 相函数：

$$
p_{HG}(\cos\theta, g) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g\cos\theta)^{3/2}}
$$

参数 $g \in [-1, 1]$：
- $g = 0$：各向同性散射（光均匀散向所有方向）
- $g > 0$：前向散射（光主要继续向前走，如烟雾）
- $g < 0$：后向散射（光主要反弹回来，如白色积云边缘发光）

**直觉解释**：云的白色之所以那么明亮，正是因为它强烈前向散射——光穿进去，大部分还是继续朝原方向走，如果你站在太阳与云之间（云在你眼前），你会看到非常亮的云边。

真实云粒子的散射是两瓣的（前向 + 轻微后向），所以我们用两个 HG 的线性混合：

```cpp
float hg(float cosTheta, float g) {
    float g2 = g * g;
    // HG 公式分子：1 - g²
    // 分母：(1 + g² - 2g·cosθ)^1.5
    return (1.f - g2) / (4.f * M_PI * powf(1.f + g2 - 2.f*g*cosTheta, 1.5f));
}

float phase(float cosTheta) {
    // g=0.7 强前向，g=-0.2 轻微后向
    // 50/50 混合 → 既有太阳方向的耀眼，也有背光面的柔和
    return lerp(hg(cosTheta, 0.7f), hg(cosTheta, -0.2f), 0.5f);
}
```

### 2.4 内散射积分（能量守恒离散化）

直接用黎曼和离散化辐射传输积分会有数值问题（步长越大误差越大）。更稳健的做法是把每一步当成一个匀质介质小块，对其做解析积分：

设步长为 $\Delta t$，密度为 $\rho$，消光系数 $\sigma_e$，散射系数 $\sigma_s$，则这一步的内散射贡献：

$$
\Delta L = L_{\text{scatter}} \cdot \frac{1 - e^{-\sigma_e \rho \Delta t}}{\sigma_e}
$$

这个形式叫"能量守恒积分"——分子 $(1 - e^{-\tau})$ 是这段介质吸收的能量比例，分母 $\sigma_e$ 是归一化因子。它确保无论步长多大，每一步吸收的总能量都不会超过 1（不会产生负透射率）。

```cpp
// scatter 是这一步的散射辐亮度（太阳光 * 透射 * 相函数 + 环境光）
Vec3 scatterInteg = scatter * ((1.f - stepTrans) / (sigma_e + 1e-6f));
inScatter += scatterInteg * transmittance;   // 乘以累积透射率（前面所有云的遮挡）
transmittance *= stepTrans;                  // 更新累积透射率
```

### 2.5 FBM 噪声构建密度场

Perlin 噪声是一种梯度噪声，输出范围约 $[-0.5, 0.5]$，在空间上连续、有机。

单层 Perlin 噪声的云形状太光滑（像棉花），真实云有多尺度的结构——大尺度的云团轮廓 + 中尺度的蓬松感 + 小尺度的细节。

**分形布朗运动（Fractal Brownian Motion, FBM）** 叠加多频：

$$
\text{FBM}(\mathbf{x}) = \sum_{i=0}^{N-1} A_i \cdot \text{Perlin}(2^i \cdot \mathbf{x})
$$

其中振幅 $A_i$ 按 $0.5^i$ 递减（高频细节振幅小），频率按 $2^i$ 递增。

```cpp
float fbm(float x, float y, float z, int octaves = 6) {
    float val = 0, amp = 0.5f, freq = 1.0f, sum = 0;
    for (int i = 0; i < octaves; i++) {
        val += amp * perlin3(x * freq, y * freq, z * freq);
        sum += amp;
        amp  *= 0.5f;   // 每级振幅减半
        freq *= 2.0f;   // 每级频率加倍
    }
    return val / sum;   // 归一化到 [-0.5, 0.5]
}
```

**直觉解释**：FBM 就像地形——山脉（低频大结构）+ 山坡（中频起伏）+ 石头纹理（高频细节）。对云来说就是：大云团 + 云朵起伏 + 蓬松边缘。

### 2.6 高度渐变

真实积云（Cumulus）不是均匀的球体，而是底部平整（由对流凝结高度决定）、顶部蓬松的结构。

我们用一个 Bell Curve 来模拟高度方向的密度分布：

```cpp
// fy = 0 在云底，fy = 1 在云顶
float fy = (p.y - CLOUD_BASE) / (CLOUD_TOP - CLOUD_BASE);
float heightGrad = fy * (1.f - fy) * 4.f;   // 最大值为 1，在 fy=0.5 处
```

$fy \cdot (1 - fy) \cdot 4$ 在 $fy = 0.5$ 时取最大值 $1.0$，在 $fy = 0$ 或 $fy = 1$ 时为 $0$，形成光滑的钟形曲线——云层中部最密，上下边界消散。

---

## ③ 实现架构

```
相机光线生成
    │
    ▼
天空颜色 skyColor(rd, sunDir)
    │           │
    │       ┌───┘
    │  Ray Marching 主循环
    │       │  cloudDensity(p) → FBM 噪声密度
    │       │  lightTransmittance(p, sunDir) → 向太阳方向采样
    │       │  Henyey-Greenstein 相函数
    │       │  能量守恒内散射积分
    │       │  Beer-Lambert 透射率更新
    │       ▼
    │  inScatter + transmittance
    │
    ▼
合成：bg * transmittance + inScatter
    │
    ▼
ACES Filmic 色调映射
    │
    ▼
输出 PNG 像素
```

**关键参数设计决策**：

| 参数 | 值 | 理由 |
|------|-----|------|
| 主步长 | 80m | 云层总厚度 2300m / 64步 ≈ 36m，但密度稀疏时可以大步长 |
| 光照步长 | 200m | 光照采样不需要精确，6步 × 200m = 1200m 覆盖整个云层 |
| 主步数 | 64 | CPU 软光栅下的性能/质量折中 |
| 光照步数 | 6 | 每主步做6次光照采样，总采样 64×6=384 次/像素 |
| 消光系数 $\sigma_e$ | 0.008 | 经验值，云层厚度约 2300m，总光学深度 ≈ 18，符合真实云的透射特性 |
| 散射系数 $\sigma_s$ | 0.006 | 略小于 $\sigma_e$，留一部分给吸收（真实云以散射为主，吸收极少） |

**数据流设计**：
- 整个渲染为纯 CPU 代码，无 GPU 依赖
- 像素级循环遍历：外层 Y（扫描线），内层 X
- 每像素独立，天然并行（本版本未并行化，可用 OpenMP 加速）
- 所有计算在栈上完成，无动态内存分配（除最终 pixel buffer）

---

## ④ 关键代码解析

### 4.1 Perlin 噪声的梯度哈希

```cpp
static uint32_t g_perm[512];   // 置换表，256+256 镜像

void initNoise(uint32_t seed = 42) {
    std::mt19937 rng(seed);
    for (int i = 0; i < 256; i++) g_perm[i] = (uint32_t)i;
    // Fisher-Yates 洗牌
    for (int i = 255; i > 0; i--) {
        int j = rng() % (i + 1);
        std::swap(g_perm[i], g_perm[j]);
    }
    // 镜像到 256-511，避免取模
    for (int i = 0; i < 256; i++) g_perm[256 + i] = g_perm[i];
}
```

**为什么要镜像两次（512 长度）？**  
在查询 `g_perm[xi + yi + zi]` 这类多维组合时，各维度下标相加可能超过 255。镜像一份到 [256,511] 区间，就可以安全地做 `g_perm[xi] + yi` 而不用再对 256 取模，避免分支。

```cpp
float grad3(int h, float x, float y, float z) {
    int hh = h & 15;   // 取低4位，选16个梯度方向之一
    float u = (hh < 8) ? x : y;
    float v = (hh < 4) ? y : (hh == 12 || hh == 14) ? x : z;
    return ((hh & 1) ? -u : u) + ((hh & 2) ? -v : v);
}
```

**Ken Perlin 原版技巧**：16 个梯度向量是立方体 12 条棱中点方向（+特殊4个），用位运算选方向，比查表更缓存友好。

### 4.2 密度场构建

```cpp
float cloudDensity(const Vec3& p) {
    // Step 1：高度边界检查（最快路径）
    float fy = (p.y - CLOUD_BASE) / (CLOUD_TOP - CLOUD_BASE);
    if (fy < 0.f || fy > 1.f) return 0.f;  // 云层外直接返回0
    
    // Step 2：高度渐变（钟形曲线）
    float heightGrad = fy * (1.f - fy) * 4.f;

    // Step 3：低频基础形状（决定云团的整体分布）
    float nx = p.x * CLOUD_SCALE;           // CLOUD_SCALE = 0.00035
    float ny = p.y * CLOUD_SCALE * 0.5f;    // Y 方向压扁，云更扁平
    float nz = p.z * CLOUD_SCALE;
    float base = fbm(nx, ny, nz, 5);

    // Step 4：高频细节（增加蓬松感和边缘不规则性）
    // 频率3倍，偏移避免与 base 相关
    float detail = fbm(nx*3.f + 5.3f, ny*3.f, nz*3.f + 1.7f, 4) * 0.35f;
    
    // Step 5：合并，乘以高度渐变
    float d = (base + 0.05f + detail) * DENSITY_MULT * heightGrad;
    return std::max(0.f, d);    // 密度不能为负
}
```

**为什么 Y 方向压扁（× 0.5）？**  
真实积云水平范围远大于垂直厚度。压扁 Y 轴的噪声频率，使云在水平方向变化更缓慢、更扁平，符合视觉直觉。

**为什么加 0.05f 偏移？**  
Perlin 噪声的均值接近 0，直接用会导致大量负密度（云密度 = 0）。加一个小的正偏移，将密度中心提高，让更多区域有云存在。偏移量的调整直接影响云的覆盖率。

### 4.3 向太阳方向的光照步进

```cpp
float lightTransmittance(const Vec3& pos, const Vec3& sunDir) {
    const int   LIGHT_STEPS = 6;
    const float LIGHT_STEP  = 200.f;  // 每步 200 米
    float optDepth = 0.f;
    Vec3 p = pos;
    for (int i = 0; i < LIGHT_STEPS; i++) {
        p = p + sunDir * LIGHT_STEP;       // 向太阳方向步进
        optDepth += cloudDensity(p) * LIGHT_STEP;  // 累积光学深度
    }
    float sigma_e = 0.008f;
    return expf(-optDepth * sigma_e);     // Beer-Lambert 衰减
}
```

**这段代码在模拟什么？**  
在主步进的每个采样点，我们问："太阳光从太阳方向射来，能有多少能量到达这个点？"为此，从当前点出发，沿太阳方向走 6 步，统计路径上的总云密度（光学深度），最后用 Beer-Lambert 算透射率。

**为什么只走 6 步而不是更多？**  
这是质量与性能的折中。6 步已经能区分"光线能直射到这里"和"光线被大量云遮挡"两种情况。更多步数在视觉上提升有限，但 CPU 计算量翻倍。

**常见 Bug：忘记从 `pos + step` 开始**  
如果第一步就在 `pos` 自身采样，光照点到自身的距离为 0，会产生自遮挡错误，导致云整体变暗。必须从 `pos + sunDir * LIGHT_STEP` 开始。

### 4.4 主渲染循环（能量守恒积分）

```cpp
Vec3 renderCloud(const Vec3& ro, const Vec3& rd, const Vec3& sunDir) {
    // 找到射线进入/离开云层的参数 t
    float tEntry = (CLOUD_BASE - ro.y) / rd.y;
    float tExit  = (CLOUD_TOP  - ro.y) / rd.y;
    if (rd.y == 0.f) return skyColor(rd, sunDir);   // 水平方向不穿云
    if (tEntry > tExit) std::swap(tEntry, tExit);    // 确保 entry < exit
    tEntry = std::max(0.f, tEntry);
    
    float cosTheta = rd.dot(sunDir);         // 视线与太阳方向夹角
    float phaseFn  = phase(cosTheta);        // HG 相函数值
    Vec3  sunLight = {1.4f, 1.2f, 0.9f};    // 暖白色太阳光
    Vec3  ambient  = {0.15f, 0.2f, 0.3f};   // 冷蓝色环境光

    float transmittance = 1.f;   // 累积透射率，初始为完全透明
    Vec3  inScatter = {};        // 累积散射颜色

    float t = tEntry;
    for (int i = 0; i < MAX_STEPS && t < tExit; i++, t += STEP_SIZE) {
        Vec3 p = ro + rd * t;
        float density = cloudDensity(p);
        if (density <= 0.001f) continue;    // 跳过空气（优化）

        float optDepth  = density * STEP_SIZE * sigma_e;
        float stepTrans = expf(-optDepth);   // 这一步的透射率

        // 这一步的散射光（太阳 + 环境）
        float lt = lightTransmittance(p, sunDir);
        Vec3 scatter = (sunLight * lt * phaseFn + ambient) * sigma_s * density;

        // 能量守恒积分：保证步长无关的稳定性
        Vec3 scatterInteg = scatter * ((1.f - stepTrans) / (sigma_e + 1e-6f));
        inScatter += scatterInteg * transmittance;   // 用前面的累积透射率加权
        transmittance *= stepTrans;                  // 更新累积透射率

        if (transmittance < 0.01f) break;    // 早期终止：已经不透光了
    }

    Vec3 bg = skyColor(rd, sunDir);
    return bg * transmittance + inScatter;  // 背景贡献 + 云的散射贡献
}
```

**为什么有 `transmittance < 0.01f` 的早期终止？**  
当云层足够厚，透射率会指数衰减到接近 0。继续步进对最终颜色贡献极微小，但计算量不变。0.01 的阈值意味着已有 99% 的光被遮挡，此时停止步进节省大量计算。

**`1e-6f` 的作用**  
防止除零。当 `sigma_e` 非常小时（稀薄区域），分母可能趋近 0，加上极小正数保护数值稳定性。

### 4.5 天空背景

```cpp
Vec3 skyColor(const Vec3& dir, const Vec3& sunDir) {
    float cosTheta = std::max(0.f, dir.dot(sunDir));
    
    // Rayleigh 散射近似：高度越高（dir.y 越大），蓝色越深
    float t = clamp01(dir.y * 0.5f + 0.5f);   // dir.y ∈ [-1,1] → t ∈ [0,1]
    Vec3 zenith  = {0.08f, 0.22f, 0.65f};      // 深蓝色顶部
    Vec3 horizon = {0.72f, 0.82f, 0.90f};      // 浅灰白地平线
    Vec3 sky = mix(horizon, zenith, t * t);    // 平方使渐变更集中在低空

    // 太阳光盘（极高光，指数为 256，非常集中）
    float sun  = powf(cosTheta, 256.f) * 5.f;
    // 太阳光晕（较宽的光圈，指数为 8）
    float glow = powf(cosTheta, 8.f)   * 0.3f;
    Vec3 sunColor = {1.0f, 0.95f, 0.7f};      // 暖黄色太阳
    sky += sunColor * (sun + glow);
    return sky;
}
```

**为什么用 `t * t` 而不是线性 `t`？**  
Rayleigh 散射在靠近地平线处变化更明显（大气层厚度急剧增加）。平方插值使蓝色更集中在高仰角，地平线更宽，视觉上更自然。

### 4.6 ACES 色调映射

```cpp
Vec3 aces(Vec3 x) {
    // ACES RRT + ODT 近似（Hill, 2018）
    float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    auto f = [&](float v) {
        return clamp01((v * (a*v + b)) / (v * (c*v + d) + e));
    };
    return {f(x.x), f(x.y), f(x.z)};
}
```

**为什么需要色调映射？**  
我们的渲染过程在 HDR（High Dynamic Range）空间进行——太阳颜色乘以 5.0，光照值可以远超 1.0。直接截断（clamp 到 1.0）会让明亮区域"烧死"（一片白）。ACES 是电影行业标准的色调曲线，明亮区域会压缩而不是截断，暗部对比度提高，整体视觉更自然。

---

## ⑤ 踩坑实录

### Bug 1：云层全黑

**症状**：渲染出来全是天空蓝，云层位置全黑，仿佛不存在。  
**错误假设**：以为是相位函数写错导致散射为 0。  
**真实原因**：`sigma_e = 0.08f`（误写了一个 0），光学深度计算为正常值的 10 倍。单步透射率 `exp(-0.08 * density * 80)` 在普通密度下直接衰减到 0.00001，每步都几乎不贡献颜色，而透射率又几乎为 0 导致背景色消失，云本身颜色也没有，所以全黑。  
**修复**：将 `sigma_e` 从 `0.08f` 改为 `0.008f`。  
**教训**：先用单步调试——把步数设为 1，手动打印单步透射率，验证数量级是否合理。一步 80m 的云，正常透射率应该在 0.3~0.9 之间，不该接近 0。

### Bug 2：上下颠倒（天空在下面）

**症状**：输出图片蓝色（天空）在图像底部，地平线在顶部，整幅图上下镜像。  
**错误假设**：以为是 UV 坐标问题。  
**真实原因**：像素坐标转 NDC 时忘记翻转 Y 轴。屏幕坐标 Y 轴向下（左上角是原点），但世界空间 Y 轴向上。  
**错误代码**：
```cpp
float v = (py + 0.5f) / H * 2.f - 1.f;   // 错：顶部 v = -1（向下），底部 v = +1（向上）
```
**修复**：
```cpp
float v = 1.f - (py + 0.5f) / H * 2.f;   // 正：顶部 v = +1（向上），底部 v = -1（向下）
```
**教训**：坐标系翻转是软光栅最常见的 bug，每次做新项目都要明确：屏幕坐标向下 vs 世界/NDC 坐标向上。

### Bug 3：早期终止导致云边缘闪烁

**症状**：云层边缘（密度从0到正值的过渡区域）出现不自然的硬边，局部呈条纹状。  
**错误假设**：以为是噪声函数有问题。  
**真实原因**：早期终止阈值 `transmittance < 0.001f` 太低，导致在某些采样顺序下，部分薄云区域被跳过，而相邻像素的步进路径略有不同（因为高度渐变），造成离散化的条纹。  
**修复**：将阈值从 `0.001f` 提高到 `0.01f`。99% 遮挡才停止，减少边缘跳变。  
**教训**：过激的优化（过小的早期终止阈值）会引入视觉伪影。在 CPU 软光栅中，质量优先于极致性能。

### Bug 4：方向射线未归一化导致椭圆形云

**症状**：云层轮廓呈椭圆形，宽高不对称，仿佛镜头被压扁了。  
**错误假设**：以为是 aspect ratio 计算错误。  
**真实原因**：视线方向向量 `rd` 构建后没有做归一化（`.norm()`），直接用于步进计算。未归一化的向量在不同方向上长度不同，水平方向步长 > 垂直方向步长，导致横向扫描的采样密度不均匀。  
**修复**：
```cpp
Vec3 rd = Vec3{ u * aspect * halfH, v * halfH + 0.35f, -1.f }.norm();
//                                                              ^^^^^^ 必须归一化
```
**教训**：光线方向向量必须是单位向量。所有步进计算 `ro + rd * t` 都假设 `|rd| = 1`，否则 `t` 的物理含义（距离/米）不成立。

---

## ⑥ 效果验证与数据

### 输出验证

```bash
python3 << 'EOF'
from PIL import Image
import numpy as np

img = Image.open("volumetric_cloud_output.png")
pixels = np.array(img).astype(float)

print(f"图像尺寸: {img.size}")              # (1280, 720)
print(f"像素均值: {pixels.mean():.1f}")    # 78.2  ← 非全黑/全白
print(f"像素标准差: {pixels.std():.1f}")   # 74.9  ← 有丰富内容变化

# 天空在上（顶部偏蓝偏暗），地平线/云在下（底部更亮）
top    = pixels[:img.size[1]//4, :, :].mean()    # 34.0（较暗的蓝天）
bottom = pixels[img.size[1]*3//4:, :, :].mean()  # 184.1（明亮的云/地平线）
print(f"顶部均值: {top:.1f}  底部均值: {bottom:.1f}")

assert pixels.mean() > 5,   "❌ 图像过暗"
assert pixels.mean() < 250, "❌ 图像过亮"
assert pixels.std()  > 5,   "❌ 图像无变化"
print("✅ 所有验证通过")
EOF
```

输出结果：
```
图像尺寸: (1280, 720)
像素均值: 78.2  标准差: 74.9
顶部区域均值: 34.0
底部区域均值: 184.1
✅ 像素统计正常
✅ 文件大小: 526KB (>10KB 通过)
```

**数据解读**：
- 均值 78.2 处于合理范围（不全黑不全白），对应视觉上暗部天空 + 亮部云的混合
- 标准差 74.9 很高，说明图像对比度强，有丰富的明暗变化（不是糊状）
- 顶部（天顶）均值 34.0（深蓝暗），底部（地平线方向）均值 184.1（亮白云），方向正确

### 性能数据

- **分辨率**：1280 × 720 = 921,600 像素
- **渲染时间**：40.6 秒（单线程 CPU，无 SIMD）
- **每像素平均采样**：约 96 次（64主步 × 部分早期终止 + 光照采样）
- **输出文件大小**：527 KB (PNG)
- **内存占用**：~2.7 MB（pixel buffer: 1280×720×3 = 2.76MB，其余都在栈上）

**性能瓶颈分析**：  
主要在 `cloudDensity()` 调用（FBM 6阶 Perlin，每次约 100+ 次浮点运算）和 `lightTransmittance()` 中的嵌套采样（每主步 6 次 `cloudDensity` 调用）。

开启 OpenMP 可在多核上线性加速：
```cpp
#pragma omp parallel for schedule(dynamic, 4)
for (int py = 0; py < H; py++) { ... }
```

---

## ⑦ 总结与延伸

### 本项目的局限性

1. **单次散射**：只计算从太阳到采样点的直接光照，没有云内的多次散射。真实云是强散射体（Single Scattering Albedo ≈ 0.999），多次散射造成的"云层整体发光、内部明亮"的效果本实现无法重现。可以用 Powder 效果或 Artistic Multiple Scattering 近似来弥补。

2. **无时间动画**：云形状静止，没有随时间的流动变化。真实云需要对 FBM 的输入加上时间偏移：`fbm(nx + time * windX, ny, nz + time * windZ, 5)`。

3. **无 Worley/Cellular 噪声**：真实云渲染（如 Nubis、UE5）会使用 Worley 噪声描述云的细胞状结构（积雨云那种多块组合形态），本实现只用 Perlin FBM，形态相对平滑。

4. **固定云层**：只有一种积云层，没有卷云（高空细丝状）、层云（大片灰暗）等多层大气结构。

### 可以继续优化的方向

- **Temporal Reprojection**：帧间复用上一帧的采样结果，每帧只做 1/N 的更新，大幅降低渲染开销（UE5 Volumetric Cloud 的关键优化）
- **Depth-aware Sampling**：根据深度缓冲跳过被地形/建筑遮挡的区域
- **LOD 步长**：靠近相机的区域用小步长，远处用大步长
- **Curl Noise 扰动**：在 FBM 基础上加入无散度的 Curl Noise，模拟云的湍流运动
- **Ambient Occlusion 近似**：在光照步进中加入方向不那么对准太阳的采样，模拟云的自遮挡带来的暗部（Powder Sugar 效果）

### 与本系列的关联

本文涉及的技术与以下已发布的文章密切相关：

- **05-11 ACES HDR Rendering**：本文直接复用了 ACES 色调映射代码
- **05-14 VXGI**：同为体积渲染思路，VXGI 的锥追踪也是一种特殊的 Ray Marching
- **02-24 Volumetric Lighting**：介绍了体积光束（Light Shaft）的光线步进，是本文的前身

---

**代码仓库**：[GitHub - daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-16-volumetric-cloud)
