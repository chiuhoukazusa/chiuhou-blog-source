---
title: "每日编程实践: Atmospheric Scattering Sky Renderer（基于物理的大气散射天空渲染）"
date: 2026-05-19 05:30:00
tags:
  - 每日一练
  - 图形学
  - 大气散射
  - C++
  - 光线追踪
  - 物理渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-19-atmospheric-scattering/atmospheric_scattering_output.png
---

# Atmospheric Scattering Sky Renderer

今天实现了一个基于物理的大气散射天空渲染器，使用 Rayleigh 散射和 Mie 散射模型，从零生成逼真的天空颜色——包括正午蓝天、日落橙红、黎明晨曦等多种时段效果。

<!-- more -->

## 一、背景与动机

### 为什么要渲染大气散射？

在早期游戏和实时渲染中，天空通常用一张静态的 cubemap 贴图来表示。这样做简单，但缺陷也很明显：

- **不能动态响应太阳位置变化**：想要展现日出、日落？必须预烘焙多张贴图然后插值，工作量巨大。
- **不支持大气参数定制**：星球大气密度不同（火星大气比地球稀薄），地面高度不同（高海拔山顶），这些场景无法用一张固定 cubemap 覆盖。
- **没有物理意义**：无法正确模拟雨后或沙尘天气时天空颜色变化（Mie 散射系数增大）。

真正的大气散射渲染能解决上述所有问题，而且有非常清晰的物理依据——大气中的分子和气溶胶颗粒会对光线进行散射，不同角度和波长的散射强度不同，最终形成我们看到的天空颜色。

### 工业界和游戏引擎的使用情况

大气散射在现代渲染管线中无处不在：

- **Unreal Engine 的 SkyAtmosphere**：UE4.26 之后引入了完整的大气散射系统，支持单次和多次散射，预计算查找表加速实时渲染。
- **Unity 的 HD Render Pipeline**：HDRP 包含基于 Bruneton 预计算模型的大气散射。
- **Guerrilla Games 的 Horizon: Zero Dawn**：采用 Frostbite 大气模型，天空色彩被誉为当代游戏中的标杆。
- **No Man's Sky**：每颗星球都有独立的大气参数，完全程序化。
- **航空航天模拟器（如 X-Plane）**：天空颜色精度要求极高，使用数值积分而非近似。

今天我们实现最经典的**单次散射（Single Scattering）**模型——在所有现实渲染器中，单次散射都是多次散射的起点和基础。

---

## 二、核心原理

### 2.1 光是如何在大气中传播的

想象太阳光从宇宙射向地面的过程：

1. 光线进入大气层
2. 在路径上的每一点，部分光子被大气分子散射（改变方向）
3. 朝观察者方向散射的那部分光，最终到达我们的眼睛（或相机）
4. 同时，原来方向上的光也因为散射而衰减（Beer-Lambert 定律）

所以我们看到的天空颜色 = 太阳光被"散"向我们方向的那部分。

### 2.2 Rayleigh 散射

Rayleigh 散射描述的是光与**比光波长小得多的粒子**（气体分子，如 N₂、O₂）的相互作用。

散射强度与波长的四次方成反比：

```
β_R(λ) ∝ 1/λ⁴
```

**直觉解释**：波长越短（蓝光，λ ≈ 440nm）散射越强，波长越长（红光，λ ≈ 680nm）散射越弱。这就是为什么天空是蓝色的！

具体数值（对应 R/G/B 三个通道）：

```
β_R = (5.8e-6, 13.5e-6, 33.1e-6)  m^-1
```

注意蓝色通道是红色通道的约 5.7 倍——这就是蓝天的物理来源。

Rayleigh 相位函数描述散射的角度分布：

```
P_R(θ) = (3 / 16π) × (1 + cos²θ)
```

其中 θ 是散射角（视线方向与太阳方向的夹角）。

**直觉解释**：θ=0 和 θ=π（向着太阳看和背着太阳看）散射最强（(1+1)=2），θ=π/2（垂直于太阳方向）最弱（(1+0)=1）。这解释了为什么头顶天空（通常与太阳夹角接近 90°）比太阳周围稍暗一些。

### 2.3 Mie 散射

Mie 散射描述的是光与**与波长相当或更大的粒子**（气溶胶、灰尘、水汽）的相互作用。

与 Rayleigh 不同，Mie 散射**几乎与波长无关**，所以散射出的光是白色的——这就是为什么太阳周围有白色光晕，以及为什么雾霾天空是灰白色的。

Mie 散射系数：

```
β_M ≈ 21e-6  m^-1  （与波长无关）
```

Mie 相位函数使用 Henyey-Greenstein 近似：

```
P_M(θ, g) = (1 - g²) / (4π × (1 + g² - 2g·cosθ)^(3/2))
```

其中 g 是不对称参数（asymmetry parameter）：
- g = 0：各向同性散射（随机方向）
- g > 0（通常 0.7~0.9）：**强前向散射**（光倾向于继续沿原方向传播）

我们使用 g = 0.76，这对应的是大气中典型的前向散射强度。**直觉解释**：太阳周围的光晕是因为 g 值大，光线倾向于朝太阳方向散射，所以在太阳附近看起来最亮。

### 2.4 大气密度随高度的变化

真实大气的密度并非均匀，而是随高度指数衰减（类似气压随高度的变化）：

```
ρ_R(h) = exp(-h / H_R)   // Rayleigh 密度剖面，H_R = 7994m
ρ_M(h) = exp(-h / H_M)   // Mie 密度剖面，    H_M = 1200m
```

这里 H_R 和 H_M 是"标高"（scale height）——大气密度降低到地面值 1/e ≈ 36.8% 时对应的高度。

Rayleigh 散射的标高 7994m ≈ 8km，意味着大气的散射效果主要来自低层大气。Mie 散射的标高更小（1.2km），气溶胶集中在对流层低层——这就是为什么雾霾常见于城市平原而不是高原。

### 2.5 透射率与光学深度

当光线穿过一段大气，会被散射而衰减。这种衰减由 **Beer-Lambert 定律** 描述：

```
T = exp(-τ)
```

其中 τ（光学深度）是沿光路的积分：

```
τ = ∫ (β_R × ρ_R(h) + β_M × ρ_M(h)) ds
```

**直觉解释**：光路越长（比如日落时太阳斜射穿越更多大气），衰减越大；且蓝光衰减更多（因为 β_R 蓝色通道最大），剩下的主要是红橙光——这就是日落的橙红色。

### 2.6 单次散射积分

对于一条从相机出发的视线，大气散射颜色由以下积分给出：

```
L(x, v) = ∫₀ᵀ T(x, p) × [P_R(θ) × β_R × ρ_R(h) + P_M(θ) × β_M × ρ_M(h)] × T(sun_p) × L_sun ds
```

其中：
- `T(x, p)`：从相机到采样点 p 的透射率
- `T(sun_p)`：从采样点 p 到太阳（大气层顶）的透射率
- `L_sun`：太阳辐照度
- `P_R, P_M`：Rayleigh 和 Mie 相位函数

这是一个双重积分——外层沿视线积分，内层沿"太阳光路"积分。数值实现中，我们使用黎曼求和近似：
- 视线方向：16 步
- 太阳光方向：8 步

共 16 × 8 = 128 次散射系数评估，对于 1600×800 = 128 万像素来说运算量约 1.6 亿次——这是离线渲染，1.4 秒完成也在预期内。

---

## 三、实现架构

### 3.1 整体渲染管线

```
相机视线方向
    ↓
与大气层顶求交（射线-球体相交）
    ↓
检测是否撞击地球（截断视线）
    ↓
沿视线均匀采样（16步）
    ↓
每个采样点：
  - 计算当前高度 h
  - 查找 Rayleigh/Mie 密度
  - 向太阳方向积分光学深度（8步）
  - 检测是否被地球遮挡（阴影）
  - 累加散射贡献
    ↓
应用相位函数（Rayleigh + Mie）
    ↓
ACES Filmic 色调映射（HDR→LDR）
    ↓
Gamma 校正（2.2）
    ↓
输出像素颜色
```

### 3.2 坐标系设计

使用**右手坐标系，Y 轴朝上**：

```
地球球心在原点 (0, 0, 0)
相机位于 (0, Re + 1, 0) — 地面上方 1 米
太阳方向向量由仰角（elevation）和方位角（azimuth）定义：
    sunDir.x = cos(elev) × cos(azim)
    sunDir.y = sin(elev)         ← 高度分量
    sunDir.z = cos(elev) × sin(azim)
```

天空图像采用**等矩形投影（Equirectangular）**——水平方向对应方位角 [0, 2π]，垂直方向对应仰角 [+90°, -90°]，简单直观。

### 3.3 主要数据结构

```cpp
// 大气物理常数
const float Re  = 6360e3f;           // 地球半径
const float Ra  = 6420e3f;           // 大气层顶半径
const float Hr  = 7994.f;            // Rayleigh 标高
const float Hm  = 1200.f;            // Mie 标高
const Vec3  BetaR(5.8e-6f, 13.5e-6f, 33.1e-6f);  // Rayleigh 散射系数 RGB
const float BetaM = 21e-6f;          // Mie 散射系数
const float MieG  = 0.76f;           // HG 相位函数 g 参数
const float SunIntensity = 20.0f;    // 太阳辐照度
```

这些都是真实物理值——直接用物理单位（米），不做任何缩放。

### 3.4 多时段 2×2 网格输出

最终输出是一张 1600×800 的图像，包含 4 个 800×400 的天空面板：

| 位置 | 时段 | 太阳仰角 |
|------|------|---------|
| 左上 | 正午（Noon） | 60° |
| 右上 | 日落（Sunset） | 5° |
| 左下 | 黎明（Dawn） | 2° |
| 右下 | 早晨（Morning） | 30° |

---

## 四、关键代码解析

### 4.1 射线-球体相交

大气层和地球都是球体，射线与球体相交是整个算法的基础操作：

```cpp
bool raySphereIntersect(const Vec3& origin, const Vec3& dir, float R, float& t0, float& t1) {
    // ||origin + t*dir||² = R²
    // 展开: t²(dir·dir) + 2t(origin·dir) + (origin·origin - R²) = 0
    float a = dir.dot(dir);         // 如果 dir 是单位向量则 a=1
    float b = 2.f * origin.dot(dir);
    float c = origin.dot(origin) - R * R;
    float disc = b*b - 4*a*c;
    if (disc < 0) return false;     // 无交点（射线不穿过球体）
    float sqrtDisc = std::sqrt(disc);
    t0 = (-b - sqrtDisc) / (2*a);  // 较小的根（近交点）
    t1 = (-b + sqrtDisc) / (2*a);  // 较大的根（远交点）
    return true;
}
```

**为什么这样用**：相机在地球内部（大气层内），所以 t0 可能为负（射线的"反方向"与大气层有交点）。我们用 `max(0, t0)` 作为积分起点，t1 作为终点——但要先检查是否打到地面（截断 tEnd）。

### 4.2 主散射积分

这是最核心的函数，逐步解析：

```cpp
Vec3 computeSkyColor(const Vec3& origin, const Vec3& dir, const Vec3& sunDir) {
    // Step 1: 找到视线与大气层顶的交点范围
    float t0, t1;
    if (!raySphereIntersect(origin, dir, Ra, t0, t1)) {
        return Vec3(0, 0, 0);  // 理论上不会发生，相机在大气内
    }
    float tStart = std::max(0.f, t0);
    float tEnd   = t1;

    // Step 2: 检查视线是否打到地面（截断积分）
    float tGround0, tGround1;
    if (raySphereIntersect(origin, dir, Re, tGround0, tGround1) && tGround1 > 0) {
        if (tGround0 > 0) tEnd = tGround0;  // 只积分到地面
    }

    // Step 3: 外层积分——沿视线分 16 步
    const int numSamples = 16;
    float segLen = (tEnd - tStart) / numSamples;

    float optDepthR = 0.f;    // 累积 Rayleigh 光学深度
    float optDepthM = 0.f;    // 累积 Mie 光学深度
    Vec3 sumR(0,0,0), sumM(0,0,0);

    for (int i = 0; i < numSamples; i++) {
        float tMid = tStart + segLen * (i + 0.5f);  // 取每段中点
        Vec3 pos = origin + dir * tMid;              // 采样点世界坐标

        // 计算采样点高度（相对地球表面）
        float height = pos.length() - Re;
        if (height < 0) height = 0;

        // 密度剖面——随高度指数衰减
        float hr = std::exp(-height / Hr);  // Rayleigh 密度
        float hm = std::exp(-height / Hm);  // Mie 密度

        // 累加光学深度（用于计算透射率）
        float dOptDepthR = hr * segLen;
        float dOptDepthM = hm * segLen;
        optDepthR += dOptDepthR;
        optDepthM += dOptDepthM;

        // Step 4: 内层积分——从采样点向太阳积分
        float t0s, t1s;
        if (!raySphereIntersect(pos, sunDir, Ra, t0s, t1s)) continue;

        // 检查采样点是否在地球阴影中（太阳被地球遮挡）
        float tg0, tg1;
        if (raySphereIntersect(pos, sunDir, Re, tg0, tg1) && tg1 > 0 && tg0 > 0) {
            continue;  // 太阳被遮挡，跳过此点
        }

        // 向太阳方向积分 8 步
        const int numSamplesSun = 8;
        float segLenSun = t1s / numSamplesSun;
        float optDepthSunR = 0.f, optDepthSunM = 0.f;

        for (int j = 0; j < numSamplesSun; j++) {
            Vec3 posSun = pos + sunDir * (segLenSun * (j + 0.5f));
            float hSun = std::max(0.f, posSun.length() - Re);
            optDepthSunR += std::exp(-hSun / Hr) * segLenSun;
            optDepthSunM += std::exp(-hSun / Hm) * segLenSun;
        }

        // Step 5: 计算总透射率 T = exp(-optical_depth)
        // 注意：Mie 消光 = 1.1 × Mie 散射（0.1 为吸收部分）
        Vec3 tau = (BetaR * (optDepthR + optDepthSunR) +
                    Vec3(BetaM, BetaM, BetaM) * 1.1f * (optDepthM + optDepthSunM));
        Vec3 attn(std::exp(-tau.x), std::exp(-tau.y), std::exp(-tau.z));

        // Step 6: 累加散射贡献
        sumR += attn * hr * dOptDepthR;  // Rayleigh 散射积分
        sumM += attn * hm * dOptDepthM;  // Mie 散射积分
    }

    // Step 7: 应用相位函数，乘以太阳强度
    float cosTheta = dir.dot(sunDir);
    float phR = phaseRayleigh(cosTheta);
    float phM  = phaseMie(cosTheta, MieG);

    return (sumR * BetaR * phR + sumM * Vec3(BetaM,BetaM,BetaM) * phM) * SunIntensity;
}
```

**关键设计说明**：

1. **为什么中点采样（i + 0.5f）而不是端点采样？** 数值积分中，中点法（midpoint rule）比端点法（rectangle rule）精度高一倍，能更好地近似光滑的密度剖面曲线。

2. **为什么 Mie 消光乘以 1.1？** Mie 散射系数只是消光的散射部分，真实的 Mie 消光 = 散射 + 吸收。1.1 的系数对应约 10% 的吸收项，这是大气气溶胶的典型值。

3. **光学深度的累加顺序很重要**：外层积分中，optDepthR/M 是从相机到当前点的累积，加上内层的太阳侧光学深度，合在一起计算从太阳到相机的全程衰减。

### 4.3 Rayleigh 相位函数

```cpp
float phaseRayleigh(float cosTheta) {
    // 标准化 Rayleigh 相位函数
    // 积分全立体角 = 1（能量守恒）
    return 3.f / (16.f * M_PI) * (1.f + cosTheta * cosTheta);
}
```

**直觉**：cosTheta = 1（向着太阳）和 cosTheta = -1（背着太阳）都返回最大值 3/(8π)；cosTheta = 0（垂直）返回最小值 3/(16π)。这个函数是对称的——日出和日落时天空颜色对称（只要大气参数相同）。

### 4.4 Henyey-Greenstein Mie 相位函数

```cpp
float phaseMie(float cosTheta, float g) {
    float g2 = g * g;
    // HG 函数：当 g=0 退化为各向同性；g→1 时强烈前向散射
    float denom = 1.f + g2 - 2.f * g * cosTheta;
    return (1.f - g2) / (4.f * M_PI * denom * std::sqrt(denom) + 1e-10f);
}
```

`1e-10f` 防止 denom=0 时的除零（理论上 g<1 时 denom≥(1-g)²>0，但浮点精度保险起见）。

当 g=0.76，cosTheta=1（正对太阳）时，denom = 1 + 0.5776 - 1.52 = 0.0576，分母变得很小——这就是为什么太阳圆盘附近极亮（太阳光晕效应）。

### 4.5 ACES Filmic 色调映射

```cpp
Vec3 acesFilmic(Vec3 x) {
    float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    // 对 RGB 三个通道分别应用同一个多项式映射
    x.x = (x.x*(a*x.x+b)) / (x.x*(c*x.x+d)+e);
    x.y = (x.y*(a*x.y+b)) / (x.y*(c*x.y+d)+e);
    x.z = (x.z*(a*x.z+b)) / (x.z*(c*x.z+d)+e);
    x.x = clamp01(x.x);
    x.y = clamp01(x.y);
    x.z = clamp01(x.z);
    return x;
}
```

**为什么需要色调映射？** 大气散射的输出是 HDR 值（最高可达几十甚至上百），但显示器只能显示 [0, 1] 的 LDR 值。简单 clamp 会损失很多细节；ACES Filmic 的特点是：
- 暗部保留细节（不会把暗处压黑）
- 高光柔和过渡（不会突兀地白出来）
- 整体色彩偏暖（符合电影调色风格）

---

## 五、踩坑实录

### 坑 1：stb_image_write.h 的 warnings 导致编译不干净

**症状**：
```
warning: missing initializer for member 'stbi__write_context::context'
```
出现约 12 条，导致无法达到"0 warnings"要求。

**错误假设**：以为是自己的代码有问题，修改了一遍 Vec3 结构。

**真实原因**：这是 stb_image_write.h 第三方库内部的 warning，与我的代码无关。该 struct 的初始化 `{ 0 }` 只初始化了第一个成员，其余未显式初始化（其实 C++ 中 `= {0}` 会零初始化所有成员，但 GCC 还是给出 warning）。

**修复方式**：用 GCC pragma 包裹 include，精准屏蔽第三方代码的警告：
```cpp
#define STB_IMAGE_WRITE_IMPLEMENTATION
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#include "../stb_image_write.h"
#pragma GCC diagnostic pop
```

**教训**：第三方库的 warning 要用 pragma 屏蔽，而不是修改自己的代码或关闭全局 warning。

### 坑 2：图像上下颠倒的潜在风险

在等矩形投影中，图像坐标行 row=0 对应顶部（天顶）还是底部（地平线）？

**我的选择**：
```cpp
float theta = (0.5f - v) * M_PI;  // v=0 → theta=+π/2（天顶），v=1 → theta=-π/2（地下）
```

这里 v = row / IMG_H，row=0 时 v=0，theta=π/2，对应天顶（y=1 方向）——图像顶部是天顶，底部是地面，符合视觉习惯。

**若写成 `(v - 0.5f) * π`** 则会上下颠倒——太阳会出现在图像底部而不是顶部。历史上这类坐标系 bug 很难发现（2026-02-20 曾经吃过这个亏），所以要特别注意。

**验证方法**：检查正午时段（太阳仰角 60°）的像素——图像上半部分应该明显亮于下半部分（天空比地面亮）。

### 坑 3：太阳光被地球遮挡的判断

在地球背阳面（相机看向地球夜晚一侧），某些采样点会有遮挡——从该点向太阳方向的射线打到了地球。

**症状**：若不处理，背阳面会出现错误的散射光，呈现奇怪的橙红色。

**修复**：
```cpp
float tg0, tg1;
if (raySphereIntersect(pos, sunDir, Re, tg0, tg1) && tg1 > 0 && tg0 > 0) {
    continue;  // 此点在地球阴影内，跳过
}
```

注意判断条件：`tg1 > 0 && tg0 > 0` 确保交点在射线前方（不是后方）。

### 坑 4：Mie 散射消光 vs 散射系数

Mie 消光包含散射和吸收两部分：

```
β_ext_M = β_scatter_M + β_absorb_M
       ≈ 1.1 × β_scatter_M  （1.1 是经验倍数）
```

若误将散射系数直接用作消光系数（不乘 1.1），在低太阳角时消光偏小，日落颜色不够红，看起来偏白。

**修复**：
```cpp
Vec3(BetaM, BetaM, BetaM) * 1.1f * (optDepthM + optDepthSunM)  // 注意 1.1f
```

---

## 六、效果验证与数据

### 6.1 输出图像

![大气散射多时段天空](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-19-atmospheric-scattering/atmospheric_scattering_output.png)

2×2 网格，从左到右、从上到下分别是：**正午**、**日落**、**黎明**、**早晨**。

### 6.2 量化像素统计

```
图像尺寸: 1600×800  文件大小: 329KB
总体均值: 85.8      标准差: 37.3

各时段亮度（均值）:
  正午  (Noon,    elev=60°): 103.8  ← 最亮，蓝天高光
  早晨  (Morning, elev=30°):  97.8  ← 次亮，暖色调
  日落  (Sunset,  elev=5°):   76.7  ← 中等，橙红天际
  黎明  (Dawn,    elev=2°):   64.9  ← 最暗，接近夜晚
```

各时段亮度差达到 38.9，远超"无差异"阈值（5），验证多时段渲染工作正常。

### 6.3 性能数据

```
渲染分辨率: 1600×800 = 1,280,000 像素
积分步数: 视线 16 步 × 太阳 8 步 = 128 次/像素
总积分评估: 1,280,000 × 128 ≈ 1.6 亿次
渲染耗时: 1.408 秒
吞吐量: ~1.14 亿次积分/秒
编译器优化: -O2（未使用 SIMD）
```

与实时渲染的对比：
- 实时大气散射（UE4 SkyAtmosphere）：预计算 LUT + 实时查表，帧耗时 < 0.5ms
- 我们的离线版本：每帧 1.4 秒，主要用于教学展示

### 6.4 编译验证

```bash
$ g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
(无输出)
$ echo $?
0
```

✅ 0 errors, 0 warnings

---

## 七、总结与延伸

### 7.1 局限性

**单次散射的限制**：真实大气中，光子会被多次散射。单次散射模型无法渲染：
- 天空背景光（离太阳较远区域应该比单次散射更亮）
- 地面反射光对天空的影响
- 彩虹（需要考虑水滴内的折射）

因此在太阳仰角极低时（< 2°），单次散射会导致天空偏暗，颜色不够丰富。

**性能限制**：128 步积分的离线渲染无法实时。

**大气模型简化**：我们假设球形地球、均匀大气成分。真实大气中，臭氧层（O₃）的吸收会让天空颜色略带青色，云层的存在会完全改变散射特性。

### 7.2 可优化方向

1. **预计算查找表（Bruneton 方法）**：将透射率预计算为关于"高度 + cos(天顶角)"的 2D LUT，实时渲染时直接查表，可以做到实时。这是 UE4/HDRP 的核心方法。

2. **多次散射近似**：Hillaire（2020）提出了一种高效的多次散射近似，只需多加一张 2D LUT 就能获得物理正确的多次散射效果。

3. **SIMD 优化**：用 SSE/AVX 并行计算 4/8 个像素的散射积分，理论上可获得 4-8× 加速。

4. **重要性采样**：当前均匀步进在太阳附近精度不足（光学深度变化剧烈）。可以用 exponential distribution 采样，把更多样本集中在大气密集处。

5. **Ozone 吸收层**：添加 O₃ 吸收项（σ_O₃ × ρ_O₃(h)），能让正午天空呈现更准确的青蓝色，而非纯蓝。

### 7.3 与本系列其他项目的关联

- **[05-16] Volumetric Cloud**：体积云同样使用 Ray Marching 积分，但密度场来自 FBM 噪声而非指数高度剖面——积分思路完全一致。
- **[05-01] ReSTIR**：ReSTIR 处理的是多光源直接光照重要性采样，大气散射可以看作是"连续分布的散射光源"的高效积分，两者思路互补。
- **[03-05] Volume Rendering**：体积渲染与大气散射的数学框架完全相同（辐射传输方程），大气散射是体积渲染在开放空间的特例。
- **[03-29] HDR Tone Mapping**：ACES Filmic 在那次项目中已经实现过，今天再次使用——HDR 管线是现代渲染的标配基础设施。

---

## 参考文献

1. T. Nishita, T. Sirai, K. Tadamura, E. Nakamae, "Display of the Earth Taking into Account Atmospheric Scattering", SIGGRAPH 1993
2. A. J. Preetham, P. Shirley, B. Smits, "A Practical Analytic Model for Daylight", SIGGRAPH 1999
3. S. Hillaire, "A Scalable and Production Ready Sky and Atmosphere Rendering Technique", EGSR 2020
4. Scratchapixel: "Simulating the Colors of the Sky" (https://www.scratchapixel.com/lessons/procedural-generation-virtual-worlds/simulating-sky/)
5. Inigo Quilez: "Atmosphere" (https://iquilezles.org/articles/fog/)

---

**代码仓库**: [daily-coding-practice / 2026-05-19-atmospheric-scattering](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-05-19-atmospheric-scattering)
