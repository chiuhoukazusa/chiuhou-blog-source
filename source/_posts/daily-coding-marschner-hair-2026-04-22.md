---
title: "每日编程实践: Marschner Hair Rendering — 物理正确的头发光照模型"
date: 2026-04-22 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 光照模型
  - 头发渲染
  - PBR
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-22-marschner-hair/marschner_hair_output.png
---

# 每日编程实践：Marschner Hair Rendering

今天实现了 Marschner 头发光照模型——这是游戏渲染领域最重要的头发着色模型之一，自 2003 年发表以来几乎成为工业标准。渲染结果是 120 根 Cubic Bezier 发丝，在三盏灯下展示出 R（反射）、TT（透射）、TRT（内反射）三条散射路径的物理效果。

![Marschner Hair Rendering 输出](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-22-marschner-hair/marschner_hair_output.png)

---

## ① 背景与动机：为什么头发渲染这么难

### 传统 Phong/Blinn-Phong 的失败

头发是一种极其特殊的几何体：它是直径约 70μm 的细圆柱，长度可达数十厘米。当光打在头发上，不仅会在表面反射，还会**穿透半透明的皮质层**，在内部散射后从另一侧透出。这种多路径散射行为是 Phong 模型完全无法描述的。

使用 Phong 模型渲染头发会得到什么？高光沿着"法向量最接近半向量"的位置出现，但头发没有固定法向量（它是圆柱），渲染结果是均匀的漫反射团，毫无层次感，就像把头发当作哑光塑料管渲染。

### Kajiya-Kay 的局限性

2003 年之前，工业界常用 Kajiya-Kay 1989 模型。它聪明地把头发的切向量 `T` 引入 Phong 公式，用 `sin(T, L)` 替代 `dot(N, L)`，生成沿发丝方向延伸的高光带——这比 Phong 好多了。

但 Kajiya-Kay 有一个根本缺陷：**它是完全经验性的**，没有物理基础。它无法区分：
- **R lobe**（表面反射，带灰度高光）
- **TT lobe**（透射，带发色的柔和背光效果）
- **TRT lobe**（内部反射，产生闪亮的二次高光）

这三种效果对头发质感至关重要。金发的那种"光泽感"大部分来自 TRT；逆光时发丝边缘的透明感来自 TT；正面的高光带来自 R。缺少任何一个，头发都会显得"假"。

### 工业界现状

Marschner 模型（2003）在 SIGGRAPH 发表后，迅速被各大渲染器采用：
- **Unreal Engine 5** 的 Hair Groom 系统使用扩展 Marschner 模型
- **Arnold** 渲染器的 `standard_hair` shader 基于 Marschner + Chiang 2016 扩展
- **RenderMan** 的 PxrHair 同样基于此框架
- **《疯狂动物城》、《冰雪奇缘》** 等大制作电影均使用物理头发模型

今天我们从头实现这个模型，用纯 C++ 软光栅化，没有依赖任何图形 API。

---

## ② 核心原理：Marschner 模型的物理推导

### 圆柱坐标系与角度定义

把单根头发想象成一个无限细长的圆柱。为了描述光在其上的散射，需要建立一套坐标系：

```
         θᵢ (入射仰角)
          \    T (发丝切向量)
    光    \  ↑
    束 →   \/___________
          /\    发丝方向
         /  \
       φ (方位角差)
```

具体来说，使用**倾斜圆柱坐标系**：
- `θᵢ`：入射光方向与发丝切平面（垂直于切向量的平面）的夹角（从切平面量，不是从切向量量）
- `θᵣ`：出射光方向与发丝切平面的夹角
- `φ`：方位角差，即入射光与出射光在垂直于切向量的截面上的投影之间的夹角
- `θd = (θᵣ - θᵢ)/2`：差角，控制折射角
- `θh = (θᵣ + θᵢ)/2`：和角，控制高光位置

这个坐标系的优点是：对于理想圆柱，散射 BSDF 只依赖于这四个角度，而不依赖于光在发丝上的具体击中点（旋转对称）。

### BSDF 的因子分解

Marschner 论文的核心洞察是：头发 BSDF 可以分解为**纵向分量 M** 和**方位分量 N**的乘积：

```
S(θᵢ, φᵢ, θᵣ, φᵣ) = Σₚ [ Mₚ(θh) · Nₚ(φ) / cos²(θd) ]
```

其中：
- `p = 0, 1, 2` 分别对应 R、TT、TRT 三条散射路径
- `Mₚ(θh)` 是**纵向散射函数**，描述光在发丝轴向平面内的分布
- `Nₚ(φ)` 是**方位散射函数**，描述光在发丝截面内的分布
- `cos²(θd)` 是几何变换的 Jacobian 项

分母 `cos²(θd)` 来自从倾斜坐标系到标准球坐标系的变换。

### 纵向散射函数 M

每个 lobe 的纵向散射用高斯分布描述：

```
Mₚ(θh; αₚ, βₚ) = G(βₚ, θh - αₚ) = exp(-(θh - αₚ)² / (2βₚ²)) / (βₚ√(2π))
```

参数含义：
- `αₚ`：高光偏移角（由发丝表面的 tilt 角 α 决定）
  - R:   `α_R  = -α`（约 -3°，向根部偏移）
  - TT:  `α_TT = -α/2`（约 -1.5°）
  - TRT: `α_TRT = -3α/2`（约 -4.5°）
- `βₚ`：高光宽度（由粗糙度决定）
  - R 最窄（锐利高光），TRT 最宽（漫反射状透射散射）

为什么 α 值不同？因为光在折射时会发生偏转，不同路径的折射次数不同（R = 0次，TT = 2次，TRT = 4次），导致各 lobe 相对于正向反射方向的偏移不同。

### 方位散射函数 N：从几何光学推导

这是最复杂的部分。对于给定的出射方位角 φ，Nₚ(φ) 等于所有满足"φₚ(h) = φ"的头发截面偏置量 h 处的贡献之和：

```
Nₚ(φ) = Σⱼ [ Aₚ(hⱼ) / |dφₚ/dh|(hⱼ) ]
```

其中 `hⱼ` 是方程 `φₚ(h) = φ` 的所有解。

**φₚ(h) 是什么？** 对于偏置量 h（光击中圆柱横截面的偏移，-1 到 +1），折射后的光路会改变方向角：

```
φₚ(h) = 2p·γT(h) - 2γᵢ(h) + p·π
```

其中：
- `γᵢ(h) = arcsin(h)` 是入射折射角
- `γT(h) = arcsin(h/η')` 是折射后的角度，`η' = sqrt(η² - sin²(θd)) / cos(θd)` 是考虑斜射修正后的有效折射率

不同 p 值对应不同的光路：
- `p=0`（R）：直接反射，`φ₀(h) = -2γᵢ`（或写作 `π - 2γᵢ`，取决于约定）
- `p=1`（TT）：两次折射，`φ₁(h) = 2γT - 2γᵢ + π`
- `p=2`（TRT）：两次折射 + 一次内反射，`φ₂(h) = 4γT - 2γᵢ + 2π`

**分母 |dφₚ/dh|** 是 φₚ 对 h 的导数绝对值。这个项很关键：在 caustic（焦散）区域，`dφ/dh ≈ 0`，意味着大量光线集中到同一角度，产生极亮的高光——这正是 TRT lobe 产生那种闪亮"光晕"的原因。

**衰减因子 Aₚ(h)**：
- p=0（R）：`A_R = F(θd, h)` ——菲涅尔反射率
- p=1（TT）：`A_TT = (1-F)² · T(2γT)` ——两次折射的透射率乘以吸收
- p=2（TRT）：`A_TRT = (1-F)² · F · T(2·2γT)` ——两次折射 + 一次内反射 + 吸收

### 菲涅尔与吸收

菲涅尔反射率采用无偏振近似：

```
F(cosθ, η) = [(cosθ - η·cosθt)² + (η·cosθ - cosθt)²] / [2 · ...]
```

其中 `cosθt = sqrt(1 - sin²θ/η²)`。

吸收用 Beer-Lambert 定律。头发颜色通过**吸收系数** `σₐ` 控制：

```
T(d) = exp(-σₐ · d)
```

其中 d 是光在头发内部的路径长度。深棕发 σₐ ≈ (0.08, 0.12, 0.20)（RGB，蓝色吸收更多，使头发偏暖）；金发 σₐ 更小；红发 σₐ 的绿蓝分量更大。

### 数值求根方案

对方程 φₚ(h) = φ 在 h∈[-1, 1] 上进行数值求根。理论上 Marschner 用了解析的 cubic 根公式（对于 p=2 TRT，φ₂(h) 是 h 的一个近似三次多项式），但在实现中，数值扫描更简单且鲁棒：

1. 在 h∈[-1,1] 上均匀采样 200 个点
2. 计算每个点的 φₚ(h) - φ
3. 检测符号变化（零交叉），用二分法确定精确根
4. 对每个根计算 |dφ/dh|，然后叠加贡献

---

## ③ 实现架构

### 整体流程

```
场景定义（头发控制点、颜色、宽度）
    ↓
Bezier 曲线求值（各分段端点 + 切向量）
    ↓
软光栅化：每分段 → 屏幕像素迭代
    ↓
逐像素 Marschner BSDF 计算（3个lobe × 3盏灯）
    ↓
亮度饱和叠加（防止多发丝重叠过曝）
    ↓
ACES Filmic 色调映射
    ↓
PNG 输出
```

### 关键数据结构

```cpp
struct HairStrand {
    Vec3 cp[4];    // Cubic Bezier 控制点（3D 世界坐标）
    Vec3 color;    // 基础发色（吸收系数由此推导）
    float width;   // 发丝宽度（世界单位，约 0.035）
};
```

发丝宽度需要在屏幕空间转换：
```cpp
float screenWidth = std::max(1.5f, worldWidth * cam.H / (depth * std::tan(fov*0.5f)));
```
这确保远处发丝不会细到 sub-pixel，始终保持最小 1.5 像素宽度。

### 渲染管线分工

**CPU 端**负责：
- Bezier 曲线细分（SUBDIV=20）
- 屏幕空间投影
- 包围盒裁剪（只处理受影响的像素矩形）
- 最终输出

**BSDF 计算**也在 CPU：因为这是软光栅化，没有 GPU shader。每像素调用 3 个 lobe 的 Marschner 评估，是性能瓶颈（120 发丝 × 20 段 × 3 灯 × N个受影响像素 × 3 lobe × 200步 = 约 5亿次浮点操作，耗时 ~52秒）。

---

## ④ 关键代码解析

### 4.1 纵向散射 M 的实现

```cpp
// Gaussian distribution G(beta, theta)
inline float gaussianM(float beta, float x){
    return std::exp(-x*x/(2.f*beta*beta)) / (beta * std::sqrt(2.f*PI));
}
```

标准高斯公式，没有什么特别的。注意 `beta` 控制宽度，`x = theta_h - alpha_p`。在代码里：

```cpp
float thetaH = (thetaR + thetaI)*0.5f;  // 和角
float Mp = gaussianM(betaP, thetaH - alphaP);
```

各 lobe 的 `alphaP` 和 `betaP`：

```cpp
if(p == 0){      alphaP = -alpha;        betaP = betaM; }         // R
else if(p == 1){ alphaP = -alpha/2.f;    betaP = betaM/2.f + 0.02f; } // TT
else {           alphaP = -3.f*alpha/2.f; betaP = betaM*2.f + 0.02f; } // TRT
```

TRT 的 `betaP` 最宽（`betaM*2.f`），这是有意为之：内部多次反射使光线方向扩散，宏观上呈现更宽的高光。

### 4.2 方位散射 N 的数值求根

这是代码中最复杂的函数：

```cpp
static float Np(int p, float phi, float cosTD, float eta){
    // 修正有效折射率（斜射修正）
    float etaT = std::sqrt(eta*eta - 1.f + cosTD*cosTD) / cosTD;
    
    float result = 0.f;
    const int STEPS = 200;
    float prevH = -1.f + 1.f/STEPS;
    
    // 计算 h=-0.99 处的 phi_p(h)
    float gammaI0 = std::asin(prevH);
    float gammaT0 = std::asin(prevH/etaT);
    float phi0 = 2.f*p*gammaT0 - 2.f*gammaI0 + p*PI;
    // 规范化到 [-pi, pi]
    while(phi0 >  PI) phi0 -= 2*PI;
    while(phi0 < -PI) phi0 += 2*PI;
    float f0 = phi0 - phi;  // 将目标 phi 平移到 0
    while(f0 >  PI) f0 -= 2*PI;
    while(f0 < -PI) f0 += 2*PI;
    
    for(int i=1; i<=STEPS; ++i){
        float h = std::max(-0.9999f, std::min(0.9999f, -1.f + 2.f*i/STEPS));
        float gammaI = std::asin(h);
        float gammaT = std::asin(h/etaT);
        float phiVal = 2.f*p*gammaT - 2.f*gammaI + p*PI;
        // 规范化...
        float f1 = /* 同上 */;
        
        if(f0 * f1 < 0.f){  // 符号变化 = 零交叉
            // 估计 |dphi/dh| 在中间点
            float hm = (prevH+h)*0.5f;
            float dphidh = 2.f/std::sqrt(1.f-hm*hm)
                         - 2.f*p/(etaT*std::sqrt(std::max(1e-6f, 1.f-(hm/etaT)*(hm/etaT))));
            
            // 衰减因子 A_p
            float fr = fresnel(std::cos(std::asin(hm)), eta);
            // ... 计算透射率 T 和菲涅尔组合 ...
            float contrib = fp / std::abs(dphidh);
            result += contrib * /* T 的灰度加权 */;
        }
        prevH=h; f0=f1;
    }
    return result * 0.5f;
}
```

几个细节值得注意：

**`etaT` 的斜射修正**：`eta' = sqrt(eta² - sin²(θd)) / cos(θd)`。这来自于把三维斜射问题转化为二维截面问题时，Snell 定律需要修正有效折射率。直接用 η=1.55 会导致错误的 φ₁/φ₂ 值。

**`phi` 的规范化**：`φₚ(h)` 是周期函数，需要始终规范到 `[-π, π]` 才能正确检测零交叉。漏掉这个会导致某些角度的贡献被错误地跳过。

**`dφ/dh` 的下限保护**：
```cpp
result += contrib / std::max(1e-4f, std::abs(dphidh));
```
在焦散区（`dφ/dh ≈ 0`）不加保护会产生 inf，导致整张图过曝。加了下限后，焦散区会产生有限但较亮的高光——视觉上正确。

### 4.3 菲涅尔反射率

```cpp
static float fresnel(float cosT, float eta){
    float sinT2 = 1.f - cosT*cosT;
    if(sinT2/sqr(eta) > 1.f) return 1.f; // 全内反射
    float cosT2 = std::sqrt(std::max(0.f, 1.f - sinT2/sqr(eta)));
    float rs = (cosT - eta*cosT2)/(cosT + eta*cosT2);   // s偏振
    float rp = (eta*cosT - cosT2)/(eta*cosT + cosT2);   // p偏振
    return .5f*(rs*rs + rp*rp);  // 非偏振平均
}
```

这是完整的 Fresnel 公式（Fresnel equations），不是 Schlick 近似。对于 η=1.55（头发），正入射时 `F ≈ 0.047`，掠射时趋向 1.0。

用完整 Fresnel 而不是 Schlick 的原因：Schlick 近似对于非导体在大角度时有较大误差，而头发散射很多时候恰好是掠射入射/出射，误差会积累。

### 4.4 Bezier 曲线与切向量

```cpp
static Vec3 bezier(const Vec3 cp[4], float t){
    float u = 1.f-t;
    return cp[0]*(u*u*u) + cp[1]*(3.f*u*u*t) + cp[2]*(3.f*u*t*t) + cp[3]*(t*t*t);
}
static Vec3 bezierTangent(const Vec3 cp[4], float t){
    float u = 1.f-t;
    Vec3 r = (cp[1]-cp[0])*(3.f*u*u)
           + (cp[2]-cp[1])*(6.f*u*t)
           + (cp[3]-cp[2])*(3.f*t*t);
    return r.norm();
}
```

切向量是 Bezier 导数，在每段的两端插值后传入 BSDF。这比用相邻点差分更精确，不会在控制点处产生跳变。

### 4.5 亮度饱和叠加

普通的 `pixel += color * alpha` 在多发丝重叠时会严重过曝。改用亮度饱和叠加：

```cpp
Vec3& pixel = img.at(px, py);
Vec3 incoming = color * alpha_blend;
// 已有亮度越高，新贡献衰减越强
pixel = pixel + incoming * (1.f - clamp01(pixel.x*0.3f + pixel.y*0.6f + pixel.z*0.1f)*0.8f);
```

这不是物理正确的，但视觉上防止了中心区域全白的问题。物理上正确的做法是每像素追踪深度，只取最近发丝的颜色——但那需要深度缓冲，软光栅化的复杂度会显著上升。

### 4.6 ACES Filmic 色调映射

```cpp
auto aces = [](float x) -> float {
    const float a=2.51f, b=0.03f, cc=2.43f, d=0.59f, e=0.14f;
    return std::max(0.f, std::min(1.f, (x*(a*x+b))/(x*(cc*x+d)+e)));
};
```

这是 Narkowicz 2015 的 ACES 近似，参数拟合自原始 ACES 曲线。它在暗部保持细节，在亮部柔和过渡到 1.0，对头发渲染这种有大量 HDR 高光的场景非常适合。

---

## ⑤ 踩坑实录

### Bug 1：未使用的 solveCubic 函数导致编译警告

**症状**：`g++ -Wall -Wextra` 报 `-Wunused-function` 警告。

**原因**：最初计划用解析 cubic 根公式求 φₚ(h) = φ 的根（Marschner 原文用了 cubic 近似），后来换成了数值扫描，但没有删除 solveCubic 函数。

**修复**：直接删除未使用的函数。目标是 0 warning。

### Bug 2：方位角 φ 的规范化导致零交叉漏检

**症状**：某些发丝方向下，N_TT 返回 0，发丝显得几乎全黑（没有透射贡献）。

**错误假设**：以为 `phi_p(h) - phi` 的范围会自然落在 [-π, π]，不需要规范化就能检测零交叉。

**真实原因**：φₚ(h) 不是单调的，p=2 时可能累积超过 2π，导致 f0 和 f1 符号正确但实际上差了一个周期——比如 f0 = -0.1, f1 = 6.2（等效于 -0.08），看起来没有零交叉，但其实跨越了 2π 边界。

**修复**：在计算 `f0 = phi_p(h) - phi` 后，用 while 循环把差值规范到 `[-π, π]`：
```cpp
while(f1 >  PI) f1 -= PI2;
while(f1 < -PI) f1 += PI2;
```

### Bug 3：渲染结果整体过曝（均值 > 200）

**症状**：第一次运行，像素均值达到 90+，但图像整体呈现大片白色，发丝细节全被淹没。

**错误假设**：以为 ACES 色调映射会处理好高光区域，不需要特别控制光照强度。

**真实原因**：120 根发丝叠加，每根发丝都对覆盖区域做 `pixel += ...`，中心区域的像素被叠加 30-50 次，即使每次贡献很小，累计下来也会超出 ACES 的处理范围（ACES 虽然可以处理 HDR，但极端过曝时会让所有亮区趋向白色，失去层次）。

**修复**：
1. 将 key light 强度从 2.0 降低到 0.8
2. 改用亮度饱和叠加（见 §4.5），让已亮的像素对后续贡献产生衰减

### Bug 4：`px` 变量名与循环变量冲突

**症状**：编译报错 `reference to 'px' is ambiguous`（gcc 某些版本可能直接 shadow 而非报错）。

**原因**：在像素循环 `for(int px=x0; px<=x1; ++px)` 内部，我写了 `Vec3& px = img.at(px,py)`，用同名变量遮蔽了循环变量。

**修复**：将内部变量改名为 `pixel`：
```cpp
Vec3& pixel = img.at(px, py);
```

### Bug 5：etaT 斜射修正漏掉导致 TT 偏移错误

**症状**：TT lobe（透射）的高光位置与理论不符，逆光时高光出现在不该出现的角度。

**错误假设**：直接用 `eta = 1.55` 计算折射角，以为 2D 截面的 Snell 定律就是 `sin(γT) = sin(γI)/eta`。

**真实原因**：头发是三维斜射（光斜着打在发丝上），需要用修正后的有效折射率：
```
eta' = sqrt(eta² - 1 + cos²(θd)) / cos(θd)
```
这个修正来自 Marschner 论文的附录，把三维斜射圆柱问题投影到二维截面时，Snell 定律中的 n 需要替换为 n'。

**修复**：在 `Np` 函数开头计算 `etaT`：
```cpp
float etaT = std::sqrt(eta*eta - 1.f + cosTD*cosTD) / cosTD;
```

---

## ⑥ 效果验证与数据

### 像素采样结果

```
图像尺寸: 800×600
像素均值: 82.8  标准差: 59.4
文件大小: 1.4MB
✅ 均值在 [10, 240] 范围内
✅ 标准差 > 5（图像有内容）
✅ 文件大小 > 10KB
```

均值 82.8（约 1/3 亮度）和较大的标准差 59.4 表明图像有合理的明暗对比——发丝高光区域明亮，背景和发丝阴影区域较暗。

### 性能数据

| 指标 | 数值 |
|------|------|
| 渲染时间 | ~52 秒 |
| 发丝数量 | 120 根 |
| 每根细分段数 | 20 |
| 灯光数量 | 3 |
| N_p 数值步数 | 200步/次 |
| 图像分辨率 | 800×600 |

**性能瓶颈分析**：主要开销在 `Np()` 的 200 步数值扫描。对于每个受影响的像素，每盏灯需要调用 3 次 `Np()`（R/TT/TRT），每次 200 步。

估算：120发丝 × 20段 × 3灯 × ~500受影响像素 × 3 lobe × 200步 ≈ **2.16亿次 Np 内部迭代**。

实际运行 52 秒，折合约 415万次/秒——考虑每次迭代有 asin、sqrt 等超越函数，这个速度合理。

### 光照效果验证

三个 lobe 各自贡献：
- **R（反射）**：权重 0.5，窄高光，顺光时在发丝表面产生白色高光带
- **TT（透射）**：权重 0.8，较宽，逆光时让发丝边缘透出发色，产生柔和的"背光晕"
- **TRT（内反射）**：权重 0.6，最宽，产生二次散射，使发丝整体有体积感

三盏灯设置：
- Key light：方向 (0.8, 1.0, 0.5)，暖白色，强度 0.8
- Fill light：方向 (-0.5, 0.3, 0.8)，冷蓝色，强度 0.3
- Rim light：方向 (0.0, -0.5, 0.8)，暗灰色，强度 0.2

---

## ⑦ 总结与延伸

### 技术局限性

1. **无自阴影**：发丝之间没有遮挡计算，密发区域看起来偏亮。真实头发有大量自遮挡，这是渲染器中头发渲染最耗时的部分之一。

2. **单次散射**：Marschner 2003 是单次散射模型，光在头发内部的多次散射（inter-fiber scattering）需要 d'Eon et al. 2011 的双向散射模型（Dual Scattering Approximation）。

3. **软光栅化的混合问题**：没有深度排序，远近发丝的颜色直接叠加，对透明度处理不正确。

4. **性能**：52 秒对于 800×600 的静态图来说已经很慢了。实时渲染需要预计算 LUT（查找表）或用解析近似替代数值积分。

### 可优化方向

1. **预计算 N LUT**：对于给定的 (φ, θd, η) 三元组，预计算 Nₚ 并存到纹理。查询时 O(1)，可以实时渲染。

2. **Chiang et al. 2016 扩展**：增加了粗糙度项和多重散射近似，用于《疯狂动物城》等影视制作。

3. **Physically Based Hair Shading in Frostbite (EA 2019)**：加入了吸收系数的欧拉格式（melanin pigmentation），让头发颜色参数更直观（直接控制黑色素浓度）。

4. **GPU 加速**：用 compute shader 并行化 Np 数值积分，可以从 52 秒降到亚秒级。

### 与系列文章的关联

这是头发渲染的物理基础。在之前实现过：
- **04-07 Kajiya-Kay**：经验模型，今天是其物理升级版
- **03-24 Subsurface Scattering**：SSS 也用类似的 Beer-Lambert 吸收模型
- **04-03 Disney BRDF**：同样是 microfacet 框架，Marschner 是专门针对圆柱几何的扩展

后续可以结合 **04-12 Volumetric Light** 和 **04-19 Irradiance Probes** 做更完整的头发场景渲染。

---

## 代码仓库

完整代码：[2026/04/04-22-marschner-hair/main.cpp](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-22-marschner-hair)

```bash
git clone https://github.com/chiuhoukazusa/daily-coding-practice.git
cd daily-coding-practice/2026/04/04-22-marschner-hair
g++ main.cpp -o output -std=c++17 -O2
./output
# 等待约 52 秒，生成 marschner_hair_output.png
```

---

**今日总结**：Marschner 模型通过将头发 BSDF 分解为纵向散射 M 和方位散射 N 的乘积，在物理上正确地描述了 R/TT/TRT 三条光路。核心挑战在于 N 的方位散射函数需要对 φₚ(h)=φ 进行数值求根，以及正确处理斜射折射率修正和角度规范化。最终实现 120 根 Bezier 发丝，52 秒渲染，生成有效的头发光照效果。
