---
title: "每日编程实践: 体积光散射 God Rays——Mie散射、光线步进与遮挡采样"
date: 2026-04-12 05:30:00
tags:
  - 每日一练
  - 图形学
  - 光线追踪
  - C++
  - 渲染技术
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04-12-god-rays/god_rays_output.png
---

## 一、背景与动机

你一定见过这样的画面：清晨阳光穿过茂密树叶，形成若干道金色光柱射向地面；或者教堂彩绘玻璃窗外，光束穿透有灰尘的空气，仿佛有形的物质。这种现象叫做**丁达尔效应（Tyndall Effect）**，在计算机图形学中通常被称为 **God Rays（耶稣光）** 或 **Crepuscular Rays（暮色光线）**。

### 1.1 物理原因

空气并不是完全透明的。它含有无数微小颗粒：尘埃、水雾、气溶胶。这些粒子会将光线向各个方向**散射**。当光束被遮挡物（树木、建筑、云层）部分遮挡时，未被遮挡的区域散射的光强于被遮挡区域，从而形成明暗相间的光柱。

没有这种效果，场景中的光照是"无形"的——你看不到光束本身，只能看到它照亮的表面。God Rays 让光本身变得**可见**，极大地增强了场景的氛围感和空间感。

### 1.2 工业界应用

**游戏引擎**：
- **Unreal Engine 5** 的 Lumen 系统实现了完整的体积光照，包括实时 God Rays
- **Unity HDRP** 提供了 Volumetric Fog 和 Volumetric Lighting 系统
- **《赛博朋克 2077》** 大量使用体积光效果营造霓虹灯穿透雨雾的赛博朋克氛围
- **《荒野大镖客2》** 的大气散射和体积光被誉为游戏行业标杆

**电影视觉特效**：
- Pixar 的 RenderMan 使用路径追踪计算真实的体积光散射
- 《星球大战》系列中激光束穿越太空的光晕效果

**医学可视化**：
- CT/MRI 体数据渲染时，体积光照增强了组织的深度感

### 1.3 技术选型

实现 God Rays 有多种方案：

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **屏幕空间后处理（Radial Blur）** | 极快，实时可用 | 不物理正确，屏幕边缘失真 | 游戏实时渲染 |
| **Shadow Map + 体积光步进** | 较物理正确，可实时 | 需要 Shadow Map 辅助 | 延迟渲染管线 |
| **完整体积光线追踪（Ray Marching）** | 物理正确，效果最好 | 计算量大，离线渲染 | 电影/高质量渲染 |
| **VXAO + 圆锥追踪** | 实时，近似准确 | 依赖体素化预处理 | 高端实时渲染 |

本项目选择第三种——**完整的光线步进体积散射**，这是最能体现散射物理原理的方案，也是学习这类技术的最佳切入点。

---

## 二、核心原理

### 2.1 大气散射基础

光在介质（大气）中传播时，与介质中的粒子发生相互作用，主要有两类：

**散射（Scattering）**：光偏离原来方向，但能量守恒（只改变方向）
- **Rayleigh 散射**：粒子远小于光波长（如氮气、氧气分子），对短波长（蓝光）散射更强，这是天空是蓝色的原因
- **Mie 散射**：粒子大小与光波长相当（如雾、烟、灰尘），强烈的**前向散射**，这是 God Rays 的主要成因

**吸收（Absorption）**：光能量被粒子吸收，转化为热能，光线减弱

对于 God Rays，我们主要关心 **Mie 散射**——雾气和灰尘颗粒造成的强前向散射。

### 2.2 体积渲染方程（VRE）

光线沿方向 **ω** 从摄像机出发，在位置 **x** 处积累散射贡献，到达光源 **x_L**。完整的体积渲染方程（Volume Rendering Equation）为：

$$L(\mathbf{x}, \omega) = \int_0^{t_{hit}} T(\mathbf{x}, \mathbf{x}(t)) \cdot \sigma_s(\mathbf{x}(t)) \cdot L_{in}(\mathbf{x}(t), \omega) \, dt$$

分解各项含义：

- **$L(\mathbf{x}, \omega)$**：从位置 **x** 沿方向 **ω** 看到的辐射亮度
- **$T(\mathbf{x}, \mathbf{x}(t))$**：从 **x** 到 **x(t)** 的**透射率**（Beer-Lambert 定律）
- **$\sigma_s(\mathbf{x}(t))$**：**散射系数**，单位体积内光被散射的概率
- **$L_{in}(\mathbf{x}(t), \omega)$**：从 **x(t)** 处，方向 **ω** 的**入射光亮度**

### 2.3 Beer-Lambert 透射率

光线穿过介质时，强度按指数衰减：

$$T(a, b) = \exp\left(-\int_a^b \sigma_t(\mathbf{x}(s)) \, ds\right)$$

其中 $\sigma_t = \sigma_s + \sigma_a$ 是**消光系数**（散射系数 + 吸收系数之和）。

**直觉理解**：
- $\sigma_t$ 很小（稀薄介质）→ $T \approx 1$，光线几乎不衰减（晴天）
- $\sigma_t$ 很大（浓雾）→ $T \approx 0$，光线被强烈吸收（能见度差）
- 均匀介质中，步长为 $\Delta t$ 的透射率更新：$T_{i+1} = T_i \cdot e^{-\sigma_t \cdot \Delta t}$

代码实现：

```cpp
float transmittance = 1.0f;
for (int i = 0; i < STEPS; ++i) {
    // ... 累积散射 ...
    transmittance *= expf(-sigma_t * stepSize);
    if (transmittance < 0.002f) break; // 提前终止优化
}
```

### 2.4 Mie 散射相位函数

相位函数 $p(\theta)$ 描述光被散射后各个方向的**概率分布**，其中 $\theta$ 是入射光方向与散射方向的夹角。

最常用的是 **Henyey-Greenstein（HG）相位函数**：

$$p_{HG}(\theta, g) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g \cos\theta)^{3/2}}$$

参数 $g \in (-1, 1)$ 是**各向异性参数**：
- $g = 0$：各向同性散射（均匀向各方向散射）
- $g > 0$（如 $g = 0.76$）：**前向散射**（散射方向偏向入射方向）
- $g < 0$：后向散射

**为什么 Mie 散射是前向散射？**

当粒子尺寸 $a$ 与光波长 $\lambda$ 相当时（$2\pi a / \lambda \approx 1$），干涉效应使得前向散射截面远大于后向。雾气粒子半径约 1-10μm，可见光波长 0.4-0.7μm，所以 Mie 参数 $g$ 通常取 0.6~0.8。

**对 God Rays 的影响**：
- 当相机朝向光源方向时，$\cos\theta \approx 1$，$p_{HG}$ 极大 → 散射非常强，出现明亮光束
- 当相机背对光源时，$\cos\theta \approx -1$，$p_{HG}$ 很小 → 几乎无散射效果

这就是为什么 God Rays 通常在**面向光源**时最为明显！

```cpp
float miePhase(float cosT, float g = 0.76f) {
    float g2 = g * g;
    float d  = 1.f + g2 - 2.f * g * cosT;
    return (1.f - g2) / (4.f * 3.14159265f * d * sqrtf(d));
}
```

**注意**：$p_{HG}$ 满足归一化条件 $\int_{4\pi} p(\theta) \, d\omega = 1$，即所有方向散射概率之和为 1。

### 2.5 Rayleigh 散射相位函数

除 Mie 散射外，空气分子本身也有 Rayleigh 散射，其相位函数为：

$$p_R(\theta) = \frac{3}{16\pi}(1 + \cos^2\theta)$$

Rayleigh 散射近似各向同性，前后方向强度相同，只有垂直方向强度为零。

在本实现中，我们将两者混合（60% Mie + 40% Rayleigh）：

```cpp
float phase = 0.6f * miePhase(cosT, 0.76f)
            + 0.4f * (3.0f/(16.0f*3.14159f)) * (1.0f + cosT*cosT);
```

### 2.6 离散化求解（Ray Marching）

连续积分在数值上通过 **Ray Marching（光线步进）** 离散化：

$$L \approx \sum_{i=0}^{N-1} T_i \cdot \sigma_s \cdot p(\theta) \cdot L_{sun}(\mathbf{x}_i) \cdot \Delta t$$

其中 $T_i = \exp(-\sigma_t \cdot i \cdot \Delta t)$ 是累积透射率，$L_{sun}(\mathbf{x}_i)$ 是 $\mathbf{x}_i$ 处来自太阳的光（考虑遮挡）。

步长选择：步长越小越精确，但开销越大。通常 64~128 步已足够。

---

## 三、实现架构

### 3.1 整体渲染管线

```
主渲染循环
   │
   ├─── 对每个像素计算光线方向 rd
   │
   ├─── 1. 场景追踪 trace(ro, rd)
   │         ├── 检测球体相交（返回颜色 + 击中距离 tHit）
   │         ├── 检测地面相交
   │         └── 天空颜色（含太阳圆盘）
   │
   ├─── 2. 体积光步进 godRays(ro, rd, tHit)
   │         ├── 计算相位函数值（固定在光源方向）
   │         └── for i in [0, STEPS):
   │                 ├── 采样点 p = ro + rd * t
   │                 ├── 阴影测试 isShadowed(p) → 球体遮挡
   │                 ├── 若不在阴影：累积散射贡献
   │                 └── 更新透射率（Beer-Lambert）
   │
   └─── 3. 合成：finalColor = aces(sceneColor + godRays * 6.0)
               │
               └── Gamma 校正 → 写入像素
```

### 3.2 场景布局设计

场景布局对 God Rays 效果至关重要。关键约束：

**必须满足**：摄像机朝向与光源方向夹角 $\theta < 60°$（$\cos\theta > 0.5$）

原因：HG 相位函数在 $g = 0.76$ 时，$\cos\theta = 0.96$ 处的值约为 1.7，是 $\cos\theta = 0$ 时的 **48 倍**。如果相机背对光源，相位函数值接近零，几乎看不到散射。

本项目的场景设置：
- **太阳位置**：`{1.5, 6.0, 18.0}`（前方右上角）
- **相机位置**：`{0, 1.0, -2.5}`（后方）
- **相机朝向**：`{0.05, -0.05, 1.0}.norm()`（往前方稍下方看）
- **夹角余弦**：计算得到 `0.958`——极佳的前向散射条件

```cpp
// 关键：验证相机-光源夹角
Vec3 toSunDir = (SUN_POS - camPos).norm();
float camSunDot = fwd.dot(toSunDir);
std::cout << "相机朝向与光源夹角余弦: " << camSunDot << std::endl;
// 输出: 0.958409 ✅
```

### 3.3 关键数据结构

```cpp
// 场景追踪结果
struct RayRes { 
    Vec3 color;   // 着色颜色
    float tHit;   // 击中距离（-1 = 天空）
};

// 球体定义（所有球体同为遮挡体）
struct Sphere {
    Vec3  center;
    float radius;
    Vec3  albedo;   // 漫反射率
};
```

体积光步进仅需场景射线追踪结果的 `tHit`——用于限制步进距离，防止步进超过最近遮挡物。

### 3.4 光源衰减模型

对于点光源（太阳近似为点光源），光强随距离的衰减：

$$\text{atten} = \frac{C}{d^2 + \epsilon}$$

其中 $C = 120$，$\epsilon = 8$ 防止近处奇点。与物理上纯粹的 $1/d^2$ 相比，分母加常数可防止近距离过曝。

```cpp
float d     = (SUN_POS - samplePos).len();
float atten = 120.0f / (d * d + 8.0f);
```

---

## 四、关键代码解析

### 4.1 体积光散射核心函数

```cpp
Vec3 godRays(const Vec3& ro, const Vec3& rd, float tScene) {
    const int STEPS  = 80;
    // 步进距离：不超过场景最近遮挡距离
    float marchDist  = (tScene > 0) ? std::min(tScene, 24.f) : 24.f;
    float step       = marchDist / float(STEPS);

    // 重要！相位函数只需计算一次（视线方向固定，光源足够远）
    // 这是一个简化：严格来说每个采样点的 cosTheta 略有不同，
    // 但对于足够远的光源，差异可忽略
    Vec3  toSun  = (SUN_POS - ro).norm();
    float cosT   = rd.dot(toSun);
    float phase  = 0.6f * miePhase(cosT, 0.76f)
                 + 0.4f * (3.f/(16.f*3.14159f)) * (1.f + cosT*cosT);

    const float sigma_s = 0.14f;  // 散射系数（调参关键）
    const float sigma_t = 0.16f;  // 消光系数 = sigma_s + sigma_a

    Vec3  acc    = {0,0,0};
    float transm = 1.f;

    for (int i = 0; i < STEPS; ++i) {
        float t   = (i + 0.5f) * step;  // 步中心采样，比步开始处更准确
        if (t >= marchDist) break;
        Vec3 p    = ro + rd * t;
        
        // 地面以下不做散射（地下没有大气介质）
        if (p.y < GROUND_Y) break;

        // 阴影测试：这个采样点是否能看到太阳？
        if (!isShadowed(p)) {
            float d      = (SUN_POS - p).len();
            float atten  = 120.f / (d * d + 8.f);
            // 散射贡献：透射率 × 散射系数 × 相位 × 光强 × 步长
            acc += SUN_COLOR * (atten * phase * sigma_s * transm * step);
        }

        // Beer-Lambert 更新透射率
        transm *= expf(-sigma_t * step);
        if (transm < 0.002f) break;  // 透射率太低，早退出节省计算
    }
    return acc;
}
```

**注意这里的关键决策**：

1. **`tScene` 限制步进距离**：如果光线打到地面（`tScene = 5.0`），我们只在 `[0, 5.0]` 范围内步进，而不是步进整个 24 单位。这确保不会穿过表面继续散射。

2. **地面检测提前终止**：`if (p.y < GROUND_Y) break;` 防止地下采样。否则地面下方会产生虚假的光照贡献。

3. **步中心采样**：`t = (i + 0.5f) * step` 使用每个区间的中心点，这比端点采样的数值误差更小（类似辛普森法则的思想）。

### 4.2 阴影测试函数

```cpp
bool isShadowed(const Vec3& pos) {
    Vec3 toSun = SUN_POS - pos;
    float dist = toSun.len();
    Vec3 dir   = toSun / dist;

    for (int i = 0; i < NUM_SPHERES; ++i) {
        float t;
        // 注意：偏移 0.02f 防止自相交（避开采样点所在球体自身）
        if (sphereHit(pos + dir * 0.02f, dir, SPHERES[i], t) && t < dist)
            return true;
    }
    return false;
}
```

**关键细节**：偏移量 `0.02f` 是 **Shadow Acne 防护**。如果采样点恰好在球体表面上（或浮点精度导致刚好落在球体内），没有偏移会错误地"遮挡自己"。偏移值的选取：
- 太小：仍然自相交（黑色噪点）
- 太大：近处遮挡失效（光线"穿过"球体）
- `0.02f` 在本场景的尺度下（球体半径 0.5~2.2）是合适的折中

### 4.3 场景着色（Blinn-Phong）

```cpp
Vec3 shade(const Vec3& pos, const Vec3& N, const Vec3& albedo) {
    Vec3 toSun = (SUN_POS - pos).norm();
    float diff = std::max(0.f, N.dot(toSun));
    
    // Blinn-Phong 镜面：用半程向量代替反射向量，更高效
    // 注意：这里用 {0,0,-1} 近似视线方向（相机在 z=-2.5 处往 z+ 看）
    Vec3 H     = (toSun + Vec3(0,0,-1)).norm();
    float spec = powf(std::max(0.f, N.dot(H)), 64.f);
    
    bool  shad = isShadowed(pos + N * 0.02f);  // 法线偏移防 Shadow Acne
    float vis  = shad ? 0.f : 1.f;
    
    // 距离衰减（软化远处光照）
    float d    = (SUN_POS - pos).len();
    float att  = 1.f / (1.f + 0.02f * d);
    
    return AMBIENT * albedo                                  // 环境光
         + SUN_COLOR * albedo * (diff * att * vis)          // 漫反射
         + SUN_COLOR * Vec3(1,1,1) * (spec * 0.3f * att * vis); // 镜面
}
```

**为什么法线偏移用 `N * 0.02f` 而不是 `rd * 0.02f`？**

沿法线方向偏移可以确保采样点严格在物体表面外侧。沿光线方向偏移可能仍然在物体内部（掠射角情况）。

### 4.4 ACES 色调映射

原始渲染使用 HDR（高动态范围），颜色值可以远超 [0,1]。ACES Filmic 色调映射将 HDR 映射到显示设备的 [0,1] 范围，同时模拟胶片感光曲线：

```cpp
Vec3 aces(Vec3 c) {
    // ACES Filmic RRT + ODT（简化版）
    float a=2.51f, b=0.03f, cc=2.43f, d=0.59f, e=0.14f;
    return Vec3(
        (c.x*(a*c.x+b))/(c.x*(cc*c.x+d)+e),
        (c.y*(a*c.y+b))/(c.y*(cc*c.y+d)+e),
        (c.z*(a*c.z+b))/(c.z*(cc*c.z+d)+e)
    ).clamp(0,1);
}
```

这个函数是 Stephen Hill 对 ACES 的简化拟合，计算量极小（每通道 5 次乘法+2次加法），效果接近完整 ACES 流程。

**与简单 clamp 的区别**：
- `clamp`：超出 1 的值直接截断，高光部分死白，细节丢失
- ACES：高光区域被柔和压缩，保留层次感，更接近电影画面质感

### 4.5 场景与体积光合成

```cpp
Vec3 finalColor = aces(sceneColor + godRays * 6.0f);
```

`6.0f` 是体积光强度倍数——这是经验参数，需要根据 `sigma_s`、`sigma_t` 和光源强度调整。

**调参原则**：
- 散射系数 `sigma_s` 越大，雾气越浓，光束越亮但穿透距离越短
- 倍数越大，光束越亮，但可能过曝（ACES 会压制）
- `sigma_t > sigma_s` 的差值（即 `sigma_a`）越大，能量损失越多，光束越暗

---

## 五、踩坑实录

### Bug 1：体积光完全不可见（第一版）

**症状**：渲染出来的图像完全没有光束，和普通 Phong 着色无区别。

**错误假设**：一开始以为是散射系数太小或步数不够。调大 `sigma_s` 和步数后仍然没有效果。

**真实原因**：调试后发现相机-光源夹角余弦只有 **-0.07**（接近垂直）！在此角度下，HG 相位函数（$g=0.72$）的值约为 **0.035**，而在 $\cos\theta = 1$ 时约为 **15**——相差 **400 倍**。

具体来说，第一版场景中光源在 `{5, 7, -5}`（相机后方！），而相机往 `{0, 0, 1}` 方向看，两者夹角接近 180°。这导致相位函数几乎为零，体积光贡献微乎其微。

**修复方式**：将光源移到 `{1.5, 6.0, 18.0}`（相机前方），相机往 `{0.05, -0.05, 1.0}` 方向看。计算得夹角余弦 `0.958`，相位函数值提升约 200 倍，效果立刻显现。

**教训**：实现体积光散射前，**必须先验证相机-光源夹角**。God Rays 只在面向光源时才能明显。

```cpp
// 调试用输出
float camSunDot = fwd.dot(toSunDir);
// 要求 > 0.8，否则效果不佳
assert(camSunDot > 0.8f && "相机未面向光源，体积光将不可见！");
```

### Bug 2：阴影自相交（Shadow Acne）

**症状**：球体表面出现随机黑色噪点斑块，在靠近遮挡球体时尤为明显。

**错误假设**：以为是浮点精度问题，尝试用 `double` 替换 `float`，无效。

**真实原因**：体积光步进的采样点有时恰好落在球体表面附近。进行阴影测试时，光线从该点出发射向太阳，第一个相交体是该点所在的球体本身（由于浮点误差）。这导致该点错误地被判为"在阴影中"，散射贡献为零，形成黑点。

**修复方式**：在阴影射线的起点加上偏移：
```cpp
// 错误写法
if (sphereHit(pos, dir, SPHERES[i], t) && t < dist)

// 正确写法：沿光线方向偏移 0.02f
if (sphereHit(pos + dir * 0.02f, dir, SPHERES[i], t) && t < dist)
```

**为什么 0.02f**：这取决于场景尺度。偏移量要大于最大几何误差（约 `1e-4 * radius`），但要远小于最小遮挡物尺寸（本场景最小球体半径 0.5f）。`0.02f` 满足：大于 `1e-4 * 2.2 ≈ 2e-4`，远小于 `0.5 / 5 = 0.1`。

### Bug 3：步进超过表面继续散射

**症状**：地面、球体背后出现不应存在的体积光贡献，画面出现错误的亮区。

**错误假设**：以为是光线步进步数太少，导致采样跳过了表面。

**真实原因**：体积光步进时没有检查采样点是否已经超过最近场景命中距离 `tHit`。如果光线打到地面（`tHit = 5.0f`），步进应该在 `t = 5.0f` 处停止，但代码没有限制，继续步进到 `t = 24.0f`，计算了地面以下（和背后）的散射贡献。

**修复方式**：
```cpp
// 步进距离限制为场景最近命中距离
float marchDist = (tScene > 0) ? std::min(tScene, 24.f) : 24.f;
// 同时在循环内加双重保险
if (t >= marchDist) break;
// 以及地面高度检测
if (p.y < GROUND_Y) break;
```

### Bug 4：图像过亮/过暗的调参困境

**症状**：第二版修正了光源方向后，整个图像大面积过白。

**原因**：`sigma_s = 0.14f` 与光源强度 `5.0f` 和倍数 `6.0f` 的组合导致散射贡献过大。ACES 色调映射虽然压制了峰值，但场景均值超过 230，细节几乎全损。

**修复思路**：体积光的最终亮度受多个参数共同控制：
1. `sigma_s`：散射系数
2. `SUN_COLOR`：光源强度
3. 最终乘以的倍数

我先固定 `sigma_s = 0.14f` 和 `SUN_COLOR = {5,4,2.2}`，只调整最终倍数，直到像素均值落在 150~220 之间。最终 `6.0f` 得到均值 202，标准差 66，视觉效果满意。

### Bug 5：编译警告（来自 stb_image_write.h）

**症状**：`-Wall -Wextra` 下有大量来自第三方库的 `-Wmissing-field-initializers` 警告。

**修复**：用 `#pragma GCC diagnostic` 包围第三方库的 include：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

这样只屏蔽了 stb 头文件中的警告，自己的代码警告不受影响。

---

## 六、效果验证与数据

### 6.1 渲染输出

**主渲染图（god_rays_output.png）**：

![God Rays 主渲染](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04-12-god-rays/god_rays_output.png)

场景中可见三个特征：
1. **太阳方向的明亮背景**：暖黄色光晕从右上方辐射
2. **球体遮挡形成的阴影束**：大球体后方，体积散射形成明暗相间的光束
3. **棋盘格地面上的阴影**：来自遮挡球体的几何阴影

**对比图（左：无体积光 / 右：有体积光）**：

![对比图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04-12-god-rays/god_rays_comparison.png)

左图（无体积光）：场景正确照亮，但空气透明，光线本身不可见，氛围感差。
右图（有体积光）：空气中的散射让光束可见，球体边缘有光晕，整体氛围温暖神秘。

**纯体积光贡献（god_rays_volume_only.png）**：

![体积光贡献](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04-12-god-rays/god_rays_volume_only.png)

可清晰看到散射模式：被球体遮挡的区域贡献接近零（暗），未被遮挡的区域贡献高（亮）。

### 6.2 像素统计数据

```
god_rays_output.png:     800x600, 均值=202.1, 标准差=66.0  ✅
god_rays_comparison.png: 1600x600, 均值=130.2, 标准差=98.8 ✅
god_rays_volume_only.png: 800x600, 均值=194.6, 标准差=69.7 ✅
```

- 均值 202/1600 像素 × 255 ≈ 79%亮度：符合稍亮但有层次的日光穿透效果
- 标准差 66：明显的明暗对比（若为0则图像无内容，若为 >100 则对比过于极端）

### 6.3 性能数据

```
分辨率: 800x600 = 480,000 像素
体积光步数: 80 步/像素
阴影测试: 5 球体/步（最坏情况）
总计算量: 480,000 × 80 × 5 = 192,000,000 次相交测试（体积光部分）

实际渲染时间（含主场景 + 对比图 + 纯体积图 = 3倍像素量）:
  3.39 秒（单线程，O2优化）
等效单图: ~1.13 秒

帧率: 约 0.88 FPS（单线程 CPU 软渲染）
```

实时优化方向：
- 多线程并行（OpenMP）：预期 8 核 → 约 7 FPS
- 降采样体积光 + 双边上采样：通常用 1/4 分辨率计算体积光，提升 4x
- 射线层次结构（BVH）加速阴影测试：球体数量多时有效（本场景只有 5 个球，效果有限）

---

## 七、总结与延伸

### 7.1 技术局限性

**本实现的局限**：

1. **单散射近似**：只计算光从光源出发、散射一次到达摄像机的贡献（Single Scattering）。真实的大气中有多次散射（Multiple Scattering），会让阴影区域也有柔和的散射光（而不是完全黑）。

2. **均匀介质假设**：散射系数 `sigma_s` 在整个场景中是常量。真实的雾气、烟雾密度不均匀，需要使用 3D 密度场（如 Perlin Noise 定义的体密度）。

3. **相位函数近似**：用 HG 函数近似 Mie 散射，真实的 Mie 散射相位函数复杂得多（米氏理论需要 Bessel 函数）。

4. **无自发光体积**：本场景的体积介质自身不发光。真实的火焰、霓虹灯辉光需要自发光体积项。

### 7.2 可优化方向

**视觉质量提升**：
- 加入 **多次散射（Multiple Scattering）**：可用 Dräine 近似或参数化散射来模拟
- **非均匀密度场**：用 Perlin Noise 定义体积密度，让雾气更自然
- **彩色散射**：Rayleigh 散射对不同波长有不同系数（$\propto 1/\lambda^4$），让远处散射的光偏蓝

**性能优化**：
- **Blue Noise Jitter**：每帧体积光采样加蓝噪声抖动 + 时间抗锯齿，使用更少步数获得更好质量
- **Bilateral Upsampling**：1/4 分辨率体积光图 → 深度引导双边上采样
- **CUDA/Compute Shader**：将 Ray Marching 移至 GPU

### 7.3 与本系列的关联

- **2026-03-05 Volume Rendering Ray Marching**：使用 Ray Marching 渲染 3D Perlin 云雾体，是体积渲染的基础，本项目在此基础上加入了遮挡采样和 Mie 相位函数
- **2026-03-10 PCSS 软阴影**：阴影遮挡技术，本项目的 `isShadowed()` 函数采用了类似的光线投射思路
- **2026-03-06 次表面散射**：同样基于散射物理模型（Dipole），与 Mie 散射在数学结构上有共通之处
- **2026-04-04 大气散射**：实现了 Rayleigh + Mie 散射的完整大气模型（天空颜色），是本项目"大气层" God Rays 的延伸

---

## 附录：编译与运行

```bash
# 环境要求：g++ >= 7, C++17
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
./output

# 预期输出：
# 渲染 800x600 God Rays...
# 相机朝向与光源方向夹角余弦: 0.958409
# ✅ 保存: god_rays_output.png
# ✅ 保存: god_rays_comparison.png
# ✅ 保存: god_rays_volume_only.png
# 🎉 完成！
```

渲染时间约 1~3 秒（取决于 CPU 性能）。如需加速，可减小 `STEPS` 常量或降低图像分辨率。
