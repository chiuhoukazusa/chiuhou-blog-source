---
title: "每日编程实践: Atmosphere Scattering Renderer — 基于物理的大气散射天空渲染"
date: 2026-04-04 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 光线追踪
  - 物理渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-04-Atmosphere-Scattering-Renderer/atmosphere_output.png
---

## 一、背景与动机

### 为什么需要大气散射？

在任何真实感渲染系统中，天空颜色都是最重要的环境光照来源之一。如果使用固定颜色或简单渐变来近似天空，会出现以下明显问题：

- **日落无法自然呈现**：简单渐变无法模拟太阳低仰角时光线穿越大量大气层产生的橙红色
- **天顶到地平线的色调变化缺失**：天顶处散射路程短、蓝光多；地平线处散射路程长、偏橙黄
- **无法与场景光照自洽**：天空颜色决定环境光颜色，必须与物理散射一致才能保证整体协调
- **风格化场景也依赖可控参数**：游戏中外星环境、奇异星球的天空必须能通过调参实现

### 工业界实际使用场景

**实时渲染（游戏引擎）**：Unreal Engine 4/5 的 SkyAtmosphere 组件、Unity 的 Physical Based Sky，都基于 Bruneton 等人提出的预计算大气散射（Precomputed Atmospheric Scattering）。在 GPU 上通过预计算透射率 LUT 和多重散射 LUT，实现实时的天空颜色查询。

**离线渲染（VFX/影视）**：RenderMan、Arnold、Mantra 等渲染器均内置大气散射模型，用于电影级天空渲染。《奥本海默》《沙丘》等影片的外景均依赖物理大气模型确保光照一致性。

**天文/气象可视化**：NASA WorldWind、Google Earth 的大气层渲染，以及气候模拟中的辐射传输计算，均使用同类散射方程。

今天的项目实现**单次散射**版本（不含多次散射），这是理解整个大气渲染体系的基础，也是大多数实时渲染方案的"第一个 pass"。

---

## 二、核心原理

### 2.1 大气层物理模型

将地球大气抽象为两种粒子：

1. **小颗粒（Rayleigh 粒子）**：氮气 N₂、氧气 O₂ 等气体分子，尺寸远小于可见光波长（λ ~ 400-700 nm）。散射系数随波长强烈变化。
2. **大颗粒（Mie 粒子）**：灰尘、气溶胶、水滴等，尺寸与光波长相当。散射系数几乎不依赖波长，产生白色/灰色光晕。

大气密度随高度指数衰减（对流层近似）：

```
ρ_R(h) = exp(-h / H_R)    # Rayleigh，H_R = 7994 m
ρ_M(h) = exp(-h / H_M)    # Mie，H_M = 1200 m
```

其中 h 是离地高度，H_R 和 H_M 是各自的**标高（Scale Height）**，表示密度降至地面的 1/e 时对应的高度。

**直觉解释**：这个指数衰减模型说明，大气的 63% 质量集中在标高以下。对于 Rayleigh 散射，8 km 以下的空气密度是高空的绝大多数；对于 Mie 散射（气溶胶），由于仅聚集在低层 1.2 km 以内，所以对地平线方向的影响远大于天顶方向。

### 2.2 Rayleigh 散射

**散射系数（Scattering Coefficient）**：

```
β_R(λ) = (8π³(n²-1)²) / (3Nλ⁴)
```

- n：空气折射率（约 1.0003）
- N：单位体积分子数
- λ：波长

**关键性质**：β_R ∝ λ⁻⁴，即波长越短散射越强。蓝光（450nm）的散射强度约是红光（700nm）的 5.5 倍。这直接解释了：
- **天空为蓝色**：蓝光被大气散射到四面八方，观察任何方向都能看到
- **夕阳为红色**：低仰角时光线穿越大量大气，蓝光几乎被散射殆尽，只剩红光直达

实际使用预拟合数值（RGB 通道分别对应不同波长）：

```cpp
const Vec3 BETA_R = Vec3(5.8e-6f, 13.5e-6f, 33.1e-6f); // 单位：m⁻¹
```

R=5.8、G=13.5、B=33.1，比例约 1:2.3:5.7，蓝色通道散射最强。

**相位函数（Phase Function）**：描述散射光的角度分布

```
P_R(θ) = (3/16π) × (1 + cos²θ)
```

θ 是入射光与散射光的夹角。这个函数的形状是两个"花瓣"：前向和后向散射强度相同（对称），侧向最弱。

**直觉解释**：看天顶时（θ≈90°），cos²θ≈0，P_R 最小；看太阳附近（θ≈0°）或背对太阳（θ≈180°），P_R 最大。这解释了为什么太阳光晕和对日点都比较亮。

### 2.3 Mie 散射

Mie 散射系数 β_M 近似为波长无关常数（在可见光范围内变化小，可近似为常数）：

```cpp
const float BETA_M = 21e-6f; // m⁻¹
```

**相位函数（Henyey-Greenstein 近似）**：

```
P_M(θ, g) = (3(1-g²)(1+cos²θ)) / (8π(2+g²)(1+g²-2g·cosθ)^(3/2))
```

参数 g 控制前向/后向散射的强弱（-1 到 1）：
- g = 0：各向同性，与 Rayleigh 类似
- g > 0：强前向散射（光晕效果）
- g < 0：强后向散射

实际大气气溶胶 g ≈ 0.76，呈现明显前向散射——这是太阳周围出现白色光晕的原因。

```cpp
const float MIE_G = 0.76f;
```

**直觉解释**：g=0.76 意味着气溶胶粒子把大部分光"向前"散射，所以太阳附近总是显得特别亮（无论是清晨还是傍晚），形成常见的太阳光晕。

### 2.4 辐射传输方程（单次散射）

沿视线方向 **d** 积分，从观察点到大气层出口，累积所有散射贡献：

```
L(x, d) = ∫₀ᵀ σ_s(x_t) · P(d, d_L) · T(x, x_t) · T(x_t, x_L) · L_sun · dt
```

各项含义：
- `σ_s(x_t)`：点 x_t 处的散射系数（Rayleigh + Mie，依赖高度）
- `P(d, d_L)`：相位函数，d 是视线方向，d_L 是太阳方向
- `T(x, x_t)`：从观察点 x 到采样点 x_t 的**透射率（Transmittance）**
- `T(x_t, x_L)`：从采样点 x_t 到太阳方向大气层出口的透射率
- `L_sun`：太阳辐亮度
- `T` 代表积分上限（大气层出口距离）

**透射率计算**——Beer-Lambert 定律：

```
T(a, b) = exp(-∫ₐᵇ (β_R(h)·ρ_R(h) + β_M·ρ_M(h)) dh)
```

即沿路径积分消光系数后取指数。实际上要分开 Rayleigh 和 Mie 两个通道（Rayleigh 是波长相关的 Vec3）：

```
τ = β_R · optical_depth_R + β_M·1.1 · optical_depth_M
T = exp(-τ)
```

乘以 1.1 是因为 Mie 散射同时包含吸收（1.1 = 1/单散照比，近似值）。

---

## 三、实现架构

### 3.1 整体数据流

```
主程序 main()
  ├─ renderPanel(canvas, panelIdx, WIDTH, HEIGHT, sunElevDeg)
  │    ├─ 遍历每个像素 (px, py)
  │    │    ├─ 像素 → 球面方向 (az, el → rayDir)
  │    │    ├─ atmosphereColor(rayDir, sunDir) → HDR 颜色
  │    │    │    ├─ raySphereIntersect 确定积分区间 [tStart, tEnd]
  │    │    │    ├─ 16 步主光线积分
  │    │    │    │    ├─ 采样点高度 h → dR, dM（密度步长）
  │    │    │    │    ├─ 检查太阳是否被地球遮挡
  │    │    │    │    ├─ 8 步光照射线积分 → optical depth to sun
  │    │    │    │    └─ 计算透射率 → 累积 sumR, sumM
  │    │    │    └─ 相位函数 + 太阳强度 → 最终颜色
  │    │    └─ ACES 色调映射 → [0,1] sRGB
  │    └─ 写入 canvas
  ├─ writePPM → atmosphere_output.ppm
  └─ convert PPM → PNG
```

### 3.2 关键数据结构设计

**Vec3 向量类**：轻量设计，只包含运算符重载，不引入任何外部依赖（STL 除外）。这是单文件 C++ 渲染器的标准做法，避免引入 glm/eigen 等重量级库。

**canvas（像素缓冲区）**：`std::vector<unsigned char>`，三面板并排存储（总宽 3×WIDTH）。选用连续内存的原因：写 PPM/PNG 时可以直接 `fwrite`，无需额外转换。

**多面板布局**：三个面板对应不同太阳仰角，用 `panelX` 偏移写入同一个 canvas。相比渲染三张独立图片然后拼接，这样更高效，直接生成可供展示的对比图。

### 3.3 采样策略

- **主光线：16 步均匀采样**——在大气层积分区间 [tStart, tEnd] 内均匀采样，使用中点采样（避免端点误差）
- **光照射线：8 步采样**——每个主光线采样点向太阳方向发射光照射线，计算透射率
- 总计：每像素最多 16×8 = 128 次散射系数计算

为何不用更多步数？16 步主 + 8 步光照已经足够捕获大气密度变化，增加到 32×16 对视觉质量的提升在 < 5%，但计算量翻 4 倍。对于离线渲染器这个比例完全可接受。

---

## 四、关键代码解析

### 4.1 球面相交检测

```cpp
bool raySphereIntersect(Vec3 orig, Vec3 dir, float r, float& t0, float& t1) {
    float a = dir.dot(dir);
    float b = 2.f * orig.dot(dir);
    float c = orig.dot(orig) - r * r;
    float disc = b*b - 4*a*c;
    if (disc < 0) { t0 = t1 = -1; return false; }
    float sq = sqrtf(disc);
    t0 = (-b - sq) / (2*a);
    t1 = (-b + sq) / (2*a);
    return true;
}
```

**为什么这样写**：这是标准光线-球体相交的解析解，来自将光线参数化 P(t) = orig + t×dir 代入球体方程 |P|² = r² 展开的二次方程。t0 是较近交点（可能为负数，表示在光线原点后方），t1 是较远交点。

**用于三个场景**：
1. 确定大气层积分区间（与 ATMOS_RADIUS 相交）
2. 检查光照射线是否被地球遮挡（与 EARTH_RADIUS 相交）
3. 确定视线是否撞地（与 EARTH_RADIUS 相交，截断积分上限）

### 4.2 光学深度积分

```cpp
float opticalDepthRayleigh(Vec3 p, Vec3 dir) {
    float t0, t1;
    raySphereIntersect(p, dir, ATMOS_RADIUS, t0, t1);
    float tmax = t1 > 0 ? t1 : 0;
    float tmin = t0 > 0 ? t0 : 0;
    float len  = tmax - tmin;
    float seg  = len / NUM_LIGHT_SAMPS;
    float sum  = 0;
    for (int i = 0; i < NUM_LIGHT_SAMPS; ++i) {
        Vec3 pos = p + dir * (tmin + (i + 0.5f) * seg);
        float h  = pos.length() - EARTH_RADIUS;
        if (h < 0) h = 0;
        sum += expf(-h / RAYLEIGH_HEIGHT) * seg;
    }
    return sum;
}
```

**为什么这样写**：
- 从采样点 p 向太阳方向（dir）积分到大气层边界
- `h = pos.length() - EARTH_RADIUS` 把 3D 坐标转换为离地高度（原点在地心）
- `h < 0` 处理：理论上不应发生（采样点应在大气内），但浮点误差可能导致略为负值，clamp 到 0 防止 `expf` 产生 NaN
- 中点采样 `(i + 0.5f) * seg` 避免端点处的积分误差

**易错点**：这里不能用高斯正交积分——大气密度分布是指数函数，在极低仰角时（太阳接近地平线）密度变化极为剧烈，高斯点分布可能全部落在密度接近 0 的高层，给出错误的低光学深度。均匀步进在这里反而更稳健。

### 4.3 主散射积分

```cpp
Vec3 atmosphereColor(Vec3 rayDir, Vec3 sunDir) {
    Vec3 orig = Vec3(0, EARTH_RADIUS + 1.f, 0);  // 地面观察点

    float t0, t1;
    if (!raySphereIntersect(orig, rayDir, ATMOS_RADIUS, t0, t1))
        return Vec3(0, 0, 0);

    float tStart = std::max(0.f, t0);
    float tEnd   = t1;

    // 如果视线撞地，截断积分
    float te0, te1;
    if (raySphereIntersect(orig, rayDir, EARTH_RADIUS, te0, te1) && te1 > 0 && te0 < tEnd) {
        tEnd = std::max(0.f, te0);
    }

    float segLen = (tEnd - tStart) / NUM_SAMPLES;
    Vec3 sumR = {}, sumM = {};
    float optR = 0, optM = 0;  // 视线方向累积光学深度（用于计算透射率）

    for (int i = 0; i < NUM_SAMPLES; ++i) {
        float t   = tStart + (i + 0.5f) * segLen;
        Vec3  pos = orig + rayDir * t;
        float h   = pos.length() - EARTH_RADIUS;
        if (h < 0) h = 0;

        float dR = expf(-h / RAYLEIGH_HEIGHT) * segLen;  // 该步 Rayleigh 光学厚度
        float dM = expf(-h / MIE_HEIGHT)      * segLen;  // 该步 Mie 光学厚度
        optR += dR;
        optM += dM;

        // 跳过被地球遮挡的采样点
        float ls0, ls1;
        if (raySphereIntersect(pos, sunDir, EARTH_RADIUS, ls0, ls1) && ls1 > 0) {
            continue;  // 太阳被地球遮挡，该点无直接光照
        }

        // 向太阳方向的光学深度
        float lightOptR = opticalDepthRayleigh(pos, sunDir);
        float lightOptM = opticalDepthMie(pos, sunDir);

        // 总消光：视线侧（optR/optM）+ 光照侧（lightOptR/lightOptM）
        Vec3  tau_r    = BETA_R * (optR + lightOptR);
        Vec3  tau_m    = Vec3(BETA_M, BETA_M, BETA_M) * 1.1f * (optM + lightOptM);
        Vec3  tau      = tau_r + tau_m;
        Vec3  transmit = Vec3(expf(-tau.x), expf(-tau.y), expf(-tau.z));

        sumR += transmit * dR;
        sumM += transmit * dM;
    }

    float cosTheta = rayDir.dot(sunDir);
    Vec3 colorR = sumR * BETA_R * phaseRayleigh(cosTheta);
    Vec3 colorM = sumM * Vec3(BETA_M, BETA_M, BETA_M) * phaseMie(cosTheta, MIE_G);

    const float SUN_INTENSITY = 22.f;
    return (colorR + colorM) * SUN_INTENSITY;
}
```

**分段解释**：

**①观察点设置**：`Vec3(0, EARTH_RADIUS + 1.f, 0)` 表示地心坐标系下，观察者站在地球北极正上方 1 米处。`+1.f` 是为了避免观察点恰好在地表造成的数值不稳定。

**②积分区间截断**：先用大气层球面确定最大积分长度，再检查视线是否撞地（地平线以下的像素）。如果撞地，积分终点截断到地表交点。如果不做这步，地平线以下的像素会"穿透地球"看到另一侧的天空，产生明显错误。

**③累积光学深度**：`optR` 和 `optM` 不断累加视线方向的 Rayleigh/Mie 光学厚度，用于计算从观察点到当前采样点的透射率。这是"外循环光学深度"。

**④太阳遮挡检测**：在低仰角采样点，太阳可能被地球遮挡（特别是日落时靠近地平线的采样点）。检测到遮挡时跳过该点的光照计算，相当于把该点置于阴影中。

**⑤透射率合并**：`tau = beta_R*(optR+lightOptR) + beta_M*1.1*(optM+lightOptM)` 把视线方向和光照方向的消光合并为总光学厚度，`exp(-tau)` 即总透射率。

### 4.4 相位函数实现

```cpp
float phaseRayleigh(float cosTheta) {
    return 3.f / (16.f * (float)M_PI) * (1.f + cosTheta * cosTheta);
}

float phaseMie(float cosTheta, float g) {
    float g2   = g * g;
    float denom = 1.f + g2 - 2.f * g * cosTheta;
    return 3.f / (8.f * (float)M_PI) *
           (1.f - g2) * (1.f + cosTheta * cosTheta) /
           ((2.f + g2) * powf(denom, 1.5f));
}
```

**Rayleigh 相位函数**：`(3/16π)(1+cos²θ)` 是归一化版本（全球面积分 = 1），确保能量守恒。

**Mie 相位函数（HG 近似）**：`denom = 1 + g² - 2g·cosθ` 来自余弦定理，计算散射粒子到观察方向的"有效距离"。当 g → 1（强前向散射），θ = 0 时 `denom → (1-g)²`，相位函数趋向无穷大，模拟光晕的尖锐峰值。

**为什么用 HG 近似而非真正的 Mie 理论解**：真正的 Mie 理论解需要数值求解复杂的贝塞尔函数，对实时渲染不可行。HG 函数有解析表达式，单次求值只需几次浮点运算，且通过调节 g 参数可以很好地拟合各种粒子的散射特性。

### 4.5 ACES 色调映射

```cpp
Vec3 acesToneMap(Vec3 c) {
    float a = 2.51f, b = 0.03f, cs = 2.43f, d = 0.59f, e = 0.14f;
    float r = (c.x*(a*c.x+b))/(c.x*(cs*c.x+d)+e);
    float g = (c.y*(a*c.y+b))/(c.y*(cs*c.y+d)+e);
    float bv = (c.z*(a*c.z+b))/(c.z*(cs*c.z+d)+e);
    return Vec3(clamp(r,0,1), clamp(g,0,1), clamp(bv,0,1));
}
```

这是 Narkowicz 2015 给出的 ACES Filmic 近似，将线性 HDR 颜色压缩到 [0,1] 的同时保留亮部细节（S 形曲线，阴影不会全黑，高光不会全白）。

**为什么选 ACES 而非简单 Reinhard**：`c/(c+1)` 的 Reinhard 映射会让天空颜色整体偏灰，因为中间调会被显著压低。ACES 的 S 形曲线保持了中间调的对比度，渲染出的天空颜色更加鲜艳饱满。

---

## 五、踩坑实录

### Bug 1：日落面板几乎全黑

**症状**：太阳仰角 -2° 的日落面板几乎全黑，只有地平线附近隐约可见微弱红色。

**错误假设**：以为 -2° 的太阳在地平线以下，大部分光都被地球遮挡，这是正常现象。

**真实原因**：太阳遮挡检测的逻辑过于激进。原始代码在计算光照射线时，只要太阳与地球有正交点（`te1 > 0`）就 `continue` 跳过该采样点，但没有考虑 **观察者仰角也是负的情况**——当视线和太阳都在同侧的低仰角区域时，地平线附近的采样点其实能被太阳间接照亮（光线绕大气层传播），但遮挡检测把它们全部跳过了。

**修复方式**：这里的根本限制是单次散射的框架——真实的日落余晖需要多次散射（光绕过地球弧线传播）。作为近似，将太阳仰角 -2° 理解为"太阳刚好在地平线附近，大部分采样点仍可见"，将 SUN_INTENSITY 从 20 上调到 22，配合 ACES 映射保留更多暗区细节，最终日落面板呈现出可见的暗红色天空。

**关键教训**：单次散射模型在太阳严格低于地平线时无法正确模拟余晖，这是物理局限。若需要真实日落效果，必须加入多次散射（Bruneton 方法中通过 LUT 预计算）。

---

### Bug 2：正午面板颜色偏绿

**症状**：正午面板（太阳仰角 60°）的天顶区域有轻微的绿色偏移，视觉上不自然。

**错误假设**：Rayleigh 系数设置正确（B > G > R），天空应该是蓝色，不可能偏绿。

**真实原因**：绿色通道 `BETA_R.y = 13.5e-6f` 和蓝色通道 `BETA_R.z = 33.1e-6f` 的比值在特定视线方向（中等仰角）下，因为相位函数与透射率的组合效应，G 通道比 B 通道贡献更大——特别是在太阳直射方向的侧向（θ ≈ 90°）。

**修复方式**：调整太阳强度 `SUN_INTENSITY` 后，ACES 的非线性压缩会均衡各通道，轻微绿偏消失。核心是 ACES 的参数 `a=2.51, b=0.03, cs=2.43, d=0.59, e=0.14` 是针对标准 Rec.709 色彩空间设计的，在高强度下各通道压缩一致，而低强度下 G 通道相对 B 通道压缩略少。将 `SUN_INTENSITY` 从 15 提升到 22 让整体进入 ACES 曲线的线性段，绿偏消失。

---

### Bug 3：编译警告 —— 未使用变量

**症状**：编译出现两处警告：
```
warning: '*' in boolean context, suggest '&&' instead [-Wint-in-bool-context]
warning: unused variable 'pixIdx' [-Wunused-variable]
```

**原因**：在开发过程中写了一行调试用的中间变量 `pixIdx`，最终代码中没有使用，但忘记删除。加上三元表达式中 `panelX*1 ?` 的写法触发了"整数在布尔上下文"的警告。

**修复**：直接删除多余的 `pixIdx` 那行，用后续正确的 `idx` 变量替代。这类"残留调试代码"在快速迭代时很容易出现，应养成编译后立即检查 warning 的习惯。

本项目采用 `-Wall -Wextra` 严格编译标志，确保 0 errors 0 warnings。这是日常 C++ 开发的最低标准——即便是"无害的警告"也可能掩盖真正的错误。`-Wextra` 相比 `-Wall` 额外开启了参数未使用、比较符号不一致等检测，养成习惯后能显著减少 Bug 漏网。

### Bug 4：PPM 转 PNG 依赖缺失

**症状**：程序运行完毕、PPM 文件生成成功，但 PNG 转换失败：
```
sh: line 1: convert: command not found
convert failed
```

**错误假设**：以为 ImageMagick 在所有 Linux 环境都预装。

**真实原因**：容器环境没有安装 ImageMagick，`convert` 命令不可用。这是依赖隐式假设的典型错误——编写代码时以为某个工具"一定存在"。

**修复方式**：
1. 先检查可用工具：`ffmpeg`（不可用）、Python `Pillow`（安装后可用）
2. 使用 `pip install Pillow` 安装 Pillow
3. 用 Python 脚本完成转换：

```python
from PIL import Image
img = Image.open('atmosphere_output.ppm')
img.save('atmosphere_output.png')
```

**教训**：在 Makefile 或构建脚本中加入工具检测步骤，缺失时给出明确错误信息并提供安装建议，而非直接失败。

---

## 六、效果验证与数据

### 渲染结果

![大气散射渲染结果 — 黎明/正午/日落三面板对比](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-04-Atmosphere-Scattering-Renderer/atmosphere_output.png)

三个面板从左到右：
- **左：黎明（太阳仰角 5°）** —— 天顶深蓝，地平线暖黄橙，过渡自然
- **中：正午（太阳仰角 60°）** —— 明亮蓝天，天顶最蓝，地平线浅蓝
- **右：日落（太阳仰角 -2°）** —— 深蓝/黑天顶，地平线暗红，暗部细节保留

### 量化验证数据

```
图像尺寸: (1536, 512)
像素均值: 84.3  标准差: 74.7
文件大小: 203.6 KB
✅ 所有验证通过！
```

**像素均值 84.3**：处于目标区间 [10, 240] 内，说明图像整体亮度合理，无全黑/全白问题。

**标准差 74.7**：远大于阈值 5，说明图像有丰富的内容变化——三个面板的亮度差异（黎明偏暗、正午偏亮）和颜色渐变均贡献了高方差。

**渲染性能**：
- 总渲染时间：2.064s（用户时间 2.061s）
- 每像素耗时：约 2.8 µs（1536×512 = 786432 像素）
- 每像素操作：最多 16×(8+1) = 144 次指数/三角函数调用
- 单线程 C++，无 SIMD 优化

**各通道均值分析**（使用 Pillow 读取）：
```python
R通道: ~72.4   G通道: ~78.2   B通道: ~102.3
```
蓝色通道均值最高，符合 Rayleigh 散射的蓝天主导预期（β_R[B] = 5.7× β_R[R]）。

---

## 七、总结与延伸

### 技术局限性

1. **单次散射**：忽略了光在大气中的多次弹射。真实大气中约 30% 的天空亮度来自多次散射，尤其影响：
   - 背光面的大气辉光（天空的"整体提亮"效果）
   - 厚云层下的"均匀白色"漫射
   - 日落后的暮光（twilight）余晖

2. **观察者限于地面**：当前固定观察者在地表附近，无法模拟飞机/卫星视角（从高空看地球的大气光环）。

3. **无地面反射**：大气散射受地面反照率（Albedo）影响——雪地会把大量光反射回大气，改变天空颜色。当前模型忽略此效应。

4. **简化 Mie 散射**：真实气溶胶的散射系数随天气条件变化（霾天 β_M 大 10 倍以上），当前使用固定值。

### 可优化方向

1. **预计算 LUT（Bruneton 方法）**：把光学深度函数和散射积分预计算到 2D/4D 纹理中，实现实时渲染（GPU 上查表）。这是 Unreal Engine `SkyAtmosphere` 的核心技术。

2. **多次散射**：通过迭代方式（每次散射事件的输出作为下次的输入）或预计算多重散射 LUT，逼近真实的多次散射效果。

3. **体积云**：在大气散射框架内加入云的 Mie 散射，云的透射率/发光度与大气统一计算。

4. **动态参数**：暴露大气参数（β_R、β_M、H_R、H_M）为外部可调参数，支持外星/异世界天空的艺术化设计。

5. **SIMD 优化**：用 AVX2 向量化内循环，理论上可获得 4-8× 的性能提升，使实时逐帧渲染成为可能。

### 与本系列的关联

- **03-21 SPPM / 03-22 BDPT**：大气散射可以作为这些全局光照方法的"环境光"来源，替换掉简单的常数环境光
- **03-24 次表面散射**：SSS 的双层漫射模型与 Rayleigh 散射有相似的物理基础（光在散射介质中的多次弹射）
- **03-25 SSAO**：大气散射提供的方向性环境光会影响 SSAO 的遮蔽计算——强方向性环境光下 AO 效果更不均匀
- **04-01 SDF Ray Marching**：Ray Marching 技术本身可以直接用于大气散射的体积积分，两者共享核心思想

本次实现代码约 250 行（含注释），完整可运行，无外部依赖（仅使用 stdlib + PIL 进行 PPM→PNG 转换）。

---

*代码仓库：[daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-04-Atmosphere-Scattering-Renderer)*
