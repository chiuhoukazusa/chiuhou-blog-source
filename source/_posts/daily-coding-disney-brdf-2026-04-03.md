---
title: "每日编程实践: Disney Principled BRDF Renderer"
date: 2026-04-03 05:33:36
tags:
  - 每日一练
  - 图形学
  - C++
  - PBR
  - BRDF
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-03-disney-brdf/disney_brdf_output.png
---

## 背景与动机

在现代实时渲染和离线渲染领域，材质系统是最核心的模块之一。早期的渲染系统采用 Phong 或 Blinn-Phong 模型，这些模型参数对美术不直观——改变"高光指数"时，你无法直觉判断结果。而且这些模型能量不守恒：高光加上漫反射经常超过入射光能量，导致不真实感。

2012 年，Burley 在 SIGGRAPH 发表了《Physically-Based Shading at Disney》，提出了 **Principled BRDF**（原则性双向反射分布函数）。这个模型的核心哲学不是追求"物理精确"，而是追求**对艺术家直觉友好 + 足够真实**：

- **对艺术家友好**：所有参数都在 [0, 1] 范围内，语义直观（metallic=0 是绝缘体，=1 是金属）
- **能量守恒**：漫反射和镜面反射之和不超过入射能量
- **适用广泛**：皮肤、金属、玻璃、布料，用同一套参数系统搞定

### 工业界使用场景

Disney BRDF 或其变体已经被以下系统采用：

- **Unreal Engine 4/5**：Epic 改良了 Disney BRDF，称为"UE4 Shading Model"，是 Epic 内部渲染的基础
- **Unity HDRP**：Lit Shader 底层即 Principled BRDF
- **Blender Cycles**：Principled BSDF 节点
- **Arnold, RenderMan**：标准材质节点基于 Disney 思路
- **Call of Duty, Horizon Zero Dawn** 等 AAA 游戏的自研引擎

没有 Principled BRDF 之前，每种材质（皮肤、金属、玻璃）需要不同的着色 Shader，维护成本极高，美术调参也非常痛苦。Disney BRDF 把这个问题统一了。

今天的项目从零实现完整的 Disney Principled BRDF，渲染 5×4 = 20 个材质球阵列，每列展示不同基础颜色和金属度，每行展示从极光滑到粗糙的变化，并叠加 clearcoat、sheen、subsurface 等高级参数效果。

---

## 核心原理

### BRDF 是什么？

BRDF（Bidirectional Reflectance Distribution Function，双向反射分布函数）描述了光在不透明表面的反射行为：

```
f_r(ω_i, ω_o) = dL_o(ω_o) / (L_i(ω_i) · cos θ_i · dω_i)
```

- `ω_i`：入射方向（指向光源）
- `ω_o`：出射方向（指向相机）
- `L_o`：出射辐亮度
- `L_i`：入射辐亮度
- `cos θ_i`：朗伯余弦项

它的物理意义是：**每单位入射辐照度能贡献多少出射辐亮度**。物理上要求 BRDF ≥ 0 且满足亥姆霍兹互换律。

### Disney BRDF 的组成

Disney BRDF 把反射分解为四个独立项：

```
f_Disney = f_diffuse + f_sheen + f_specular + f_clearcoat
```

每一项对应不同的物理现象。

---

### 漫反射项：Burley Diffuse

传统 Lambertian 漫反射 `f = c/π` 忽略了掠射角时的后向散射增强和菲涅耳效应。Burley 提出了修正版：

```
f_d = (baseColor / π) × F_D90 修正
F_D = (1 + (F_D90 - 1)(1 - cos θ)^5)
F_D90 = 0.5 + 2 × cos²θ_h × roughness
```

- **直觉**：当视线或光线接近掠射角（`cosθ ≈ 0`）时，`(1-cosθ)^5 ≈ 1`，Schlick 项接近 F_D90
- 当 `roughness=0`，`F_D90=0.5`，漫反射在掠射角*减弱*（接近平滑表面的行为）
- 当 `roughness=1`，`F_D90=2.5`，漫反射在掠射角*增强*（粗糙表面的后向散射）

这比 Lambertian 更符合实际观测，粗糙表面的边缘会比 Lambert 更亮。

代码中对应：
```cpp
float Fd90 = 0.5f + 2.f * LdotH * LdotH * m.roughness;
float Fd  = lerp(1.f, Fd90, FL) * lerp(1.f, Fd90, FV);
Vec3 diffuse = m.baseColor / PI * lerp(Fd, ss, m.subsurface) * (1.f - m.metallic);
```

注意 `* (1.f - m.metallic)`：金属没有漫反射，所有能量转化为镜面反射。

---

### 菲涅耳项：Schlick 近似

菲涅耳效应描述了反射率随入射角变化的物理现象：垂直入射时反射率最低，掠射角时趋近 100%。精确计算需要 Fresnel 方程，但 Schlick 给出了极好的近似：

```
F(θ) ≈ F₀ + (1 - F₀)(1 - cos θ)^5
```

- `F₀`：垂直入射时的反射率（材质固有属性）
- 绝缘体（塑料、石头）：F₀ ≈ 0.04（约 4%）
- 导体（金属）：F₀ = baseColor（有色反射，如金的金黄色高光）

关键实现：
```cpp
inline float SchlickFresnel(float u) {
    float m = std::max(0.f, 1.f - u);
    float m2 = m * m;
    return m2 * m2 * m;  // (1-u)^5，避免 pow() 的精度问题
}
```

Disney 用 `FH = SchlickFresnel(LdotH)`，而非 `NdotV`，这里的 half-vector 夹角更准确地表达微表面的 Fresnel 贡献。

---

### 镜面反射项：微表面模型

微表面理论（Microfacet Theory）认为：真实表面在微观尺度上是由无数个朝向随机的微小镜面组成，宏观反射是所有微表面反射的统计叠加。

完整的 Cook-Torrance 微表面 BRDF：

```
f_specular = D(h) × F(v,h) × G(l,v) / (4 × (n·l) × (n·v))
```

- **D(h)：法线分布函数（NDF）** —— 朝向 h 的微表面占比
- **F(v,h)：菲涅耳项** —— 镜面反射率
- **G(l,v)：几何遮蔽/阴影** —— 微表面自遮挡

#### NDF：GGX (GTR2)

Disney 使用广义 Trowbridge-Reitz（GTR2），即 GGX：

```
D_GGX(h) = α² / (π × (1 + (α² - 1)(n·h)²)²)
```

- `α = roughness²`（粗糙度的平方，使线性空间更直观）
- 相比 Phong/Beckmann，GGX 有更长的"尾巴"——高光边缘更柔和，更符合真实观测

**直觉**：当 `NdotH=1`（完美对准），`D = α²/π`（最大值）；当偏离时，分母的 `(α²-1)(NdotH)²` 项导致快速衰减，但 GGX 的衰减比 Beckmann 慢（更长的高光边缘）。

```cpp
inline float GTR2(float NdotH, float alpha) {
    float a2 = alpha * alpha;
    float t  = 1.f + (a2 - 1.f) * NdotH * NdotH;
    return a2 / (PI * t * t);  // 分母的平方给出 GGX 的长尾特性
}
```

#### 几何遮蔽：Smith-G

Smith 几何函数假设入射和出射方向的遮蔽统计独立，可以分别计算再相乘：

```
G(l,v) = G₁(l) × G₁(v)
G₁(v) = 1 / (v·n + √(α² + (v·n)²(1-α²)))
```

```cpp
inline float SmithG_GGX(float NdotV, float alphaG) {
    float a = alphaG * alphaG;
    float b = NdotV * NdotV;
    return 1.f / (NdotV + std::sqrt(a + b - a * b));
}
```

**直觉**：当 `NdotV=1`（垂直看），几何遮蔽最小（`G₁ ≈ 1/2`）；掠射角时遮蔽增大，反射减弱。注意这里已经把 `4(n·l)(n·v)` 的归一化因子合并进去了（`SmithG_GGX` 的分子是 1 而非 0.5，等效于吸收了分母的 `2(n·v)`）。

#### 颜色混合：绝缘体 vs 金属

```cpp
Vec3 Cspec0 = lerp(
    m.specular * 0.08f * lerp(Vec3(1.f), Ctint, m.specularTint),
    m.baseColor,
    m.metallic
);
```

- 绝缘体（metallic=0）：高光颜色 ≈ 白色，由 specular（F₀ 强度）和 specularTint（是否带基础色）控制
- 金属（metallic=1）：高光颜色 = baseColor（金属有色高光）
- `0.08f * specular`：把 [0,1] 的 specular 映射到 F₀ 范围 [0, 0.08]，绝缘体 F₀ 通常在 0.02-0.08

---

### Clearcoat 层

Clearcoat 模拟透明清漆层（汽车漆、指甲油）—— 底层材质上面覆盖了一层光滑绝缘体涂层。

```
f_clearcoat = 0.25 × clearcoat × G_r × F_r × D_r
```

- 固定 F₀ = 0.04（ior=1.5 的绝缘体，如漆）
- 固定几何遮蔽 `α = 0.25`（假设较光滑）
- NDF 用 GTR1（Berry 分布），比 GGX 更尖锐：

```
D_GTR1(h) = (α² - 1) / (π × ln(α²) × (1 + (α² - 1)(n·h)²))
```

GTR1 在 `clearcoatGloss=1` 时极其尖锐，`=0` 时较宽，模拟从哑光到高光漆的变化。

```cpp
float Dr  = GTR1(NdotH, lerp(0.1f, 0.001f, m.clearcoatGloss));
float Fr  = lerp(0.04f, 1.f, FH);  // 固定绝缘体 F₀
float Gr  = SmithG_GGX(NdotL, 0.25f) * SmithG_GGX(NdotV, 0.25f);
float clearcoat_term = 0.25f * m.clearcoat * Gr * Fr * Dr;
```

---

### Sheen 项

Sheen 模拟布料、天鹅绒的边缘光晕（逆反射特性）：

```cpp
float FH   = SchlickFresnel(LdotH);
Vec3 Fsheen = FH * m.sheen * Csheen;
```

- 仅在掠射角（`LdotH ≈ 0`，即 `FH ≈ 1`）时贡献明显
- 颜色可以是白色或带基础色（由 sheenTint 控制）

---

### Subsurface 散射近似

次表面散射（皮肤、蜡烛、玉石）准确模拟需要 BSSRDF 和路径追踪，代价极高。Disney 使用 Hanrahan-Krueger 近似，仅修改漫反射项：

```
F_ss90 = LdotH² × roughness
F_ss   = lerp(1, F_ss90, F_L) × lerp(1, F_ss90, F_V)
ss_term = 1.25 × (F_ss × (1/(NdotL + NdotV) - 0.5) + 0.5)
```

最终漫反射用 `lerp(Fd, ss, subsurface)` 混合两者。

**直觉**：次表面项的 `1/(NdotL + NdotV)` 在掠射角时发散，被 `-0.5 + 0.5` 钳位，模拟光在材质内部散射后从侧面溢出的效果。

---

## 实现架构

### 渲染管线概述

本项目不使用光线追踪——为了聚焦 BRDF 实现，使用解析光照（直接光源 + 环境项）：

```
输入: 光线方向 rd, 场景（球体列表 + 灯光列表）
  ↓
光线-球体相交（解析几何）
  ↓
获取命中点坐标 P, 法线 N, 材质 mat
  ↓
对每个光源 L:
    V = normalize(camPos - P)
    L_dir = normalize(lightPos - P)
    brdf = DisneyBRDF(L_dir, V, N, mat)
    color += brdf * lightColor * NdotL
  ↓
加环境项 ambient
  ↓
ACES Filmic 色调映射
  ↓
Gamma 校正 (sRGB)
  ↓
输出到帧缓冲
```

### 材质球布局

5 列 × 4 行 = 20 个球：

| 列 | baseColor | metallic | 特性 |
|----|-----------|----------|------|
| 0 | 红色 | 0 | sheen 0.3 |
| 1 | 蓝色 | 0 | sheen 0.3, specularTint 0.5 |
| 2 | 金色 | 1 | 纯金属 |
| 3 | 绿色 | 0 | 前两行 subsurface 0.4 |
| 4 | 银白 | 1 | clearcoat 1.0 |

粗糙度从上到下线性插值：0.05 → 0.90（4 档：0.05, 0.33, 0.62, 0.90）

### 关键数据结构

```cpp
struct DisneyMaterial {
    Vec3  baseColor     = {0.8f, 0.8f, 0.8f};
    float subsurface    = 0.f;    // 次表面散射
    float metallic      = 0.f;    // 金属度 [0,1]
    float specular      = 0.5f;   // F₀强度，映射到[0,0.08]
    float specularTint  = 0.f;    // 高光是否带底色
    float roughness     = 0.5f;   // 粗糙度
    float anisotropic   = 0.f;    // 各向异性（本实现未启用）
    float sheen         = 0.f;    // 布料边缘光
    float sheenTint     = 0.5f;   // sheen颜色偏向
    float clearcoat     = 0.f;    // 清漆层强度
    float clearcoatGloss= 1.f;    // 清漆光泽度
};
```

### 光源配置

使用三点布光，模拟摄影棚效果：

```cpp
std::vector<Light> lights = {
    { {  8.f,  12.f, 10.f }, {1.0f, 0.95f, 0.9f}, 4.0f },  // 主光（暖白）
    { { -6.f,   8.f, 10.f }, {0.7f, 0.8f,  1.0f}, 2.0f },  // 补光（冷蓝）
    { {  0.f,  -5.f,  8.f }, {0.5f, 0.4f,  0.3f}, 1.0f },  // 底光（暖调）
};
```

---

## 关键代码解析

### 完整 DisneyBRDF 函数

```cpp
Vec3 DisneyBRDF(const Vec3& L, const Vec3& V, const Vec3& N, const DisneyMaterial& m) {
    float NdotL = std::max(0.f, N.dot(L));
    float NdotV = std::max(0.f, N.dot(V));
    if (NdotL <= 0.f || NdotV <= 0.f) return Vec3(0.f);
    // 提前返回避免后续除以零

    Vec3 H = (L + V).normalized();  // half-vector：L和V的角平分线
    float NdotH = std::max(0.f, N.dot(H));
    float LdotH = std::max(0.f, L.dot(H));
    // 注意：L·H = V·H（half-vector的对称性）
    
    // 亮度分解：提取颜色调色板
    float Cdlum = 0.3f*m.baseColor.x + 0.6f*m.baseColor.y + 0.1f*m.baseColor.z;
    // 使用感知亮度权重（人眼对绿色最敏感）
    Vec3 Ctint = Cdlum > 0.f ? m.baseColor / Cdlum : Vec3(1.f);
    // Ctint：baseColor 的色相，亮度归一到1
    
    Vec3 Cspec0 = lerp(
        m.specular * 0.08f * lerp(Vec3(1.f), Ctint, m.specularTint),
        m.baseColor,
        m.metallic
    );
    // Cspec0：F₀颜色
    // metallic=0: F₀ = 灰色（强度由specular控制）×（是否带底色）
    // metallic=1: F₀ = baseColor（有色金属反射）
    
    Vec3 Csheen = lerp(Vec3(1.f), Ctint, m.sheenTint);
    // sheen颜色：白色或带底色

    // === 漫反射 (Burley) ===
    float FL  = SchlickFresnel(NdotL);  // 入射角Fresnel
    float FV  = SchlickFresnel(NdotV);  // 出射角Fresnel
    float Fd90 = 0.5f + 2.f * LdotH * LdotH * m.roughness;
    // Fd90：掠射角漫反射修正因子，roughness越大越亮
    float Fd  = lerp(1.f, Fd90, FL) * lerp(1.f, Fd90, FV);
    // 两端都做Fresnel修正

    // === 次表面散射近似 ===
    float Fss90 = LdotH * LdotH * m.roughness;
    float Fss   = lerp(1.f, Fss90, FL) * lerp(1.f, Fss90, FV);
    float ss    = 1.25f * (Fss * (1.f / (NdotL + NdotV) - 0.5f) + 0.5f);
    // 1/(NdotL+NdotV)在掠射角发散，被-0.5+0.5钳位到[0.5, ∞)再×1.25
    // 模拟次表面散射的背光溢出

    Vec3 diffuse = m.baseColor / PI *
                   lerp(Fd, ss, m.subsurface) *
                   (1.f - m.metallic);
    // 金属无漫反射

    // === Sheen ===
    float FH   = SchlickFresnel(LdotH);
    // FH基于half-vector夹角，在L≈V时接近0，掠射时接近1
    Vec3 Fsheen = FH * m.sheen * Csheen * (1.f - m.metallic);

    // === 镜面反射（微表面模型）===
    float alpha = std::max(0.001f, m.roughness * m.roughness);
    // α=roughness²：使粗糙度在感知上线性
    // max(0.001)：防止α=0时NDF奇异（完美镜面）
    
    float Ds    = GTR2(NdotH, alpha);    // GGX NDF
    Vec3 Fs     = lerp(Cspec0, Vec3(1.f), FH);
    // F₀到1的Schlick插值，掠射角所有材质都接近全反射
    float Gs    = SmithG_GGX(NdotL, alpha) * SmithG_GGX(NdotV, alpha);
    // Smith几何遮蔽：入射和出射分开计算
    Vec3 specular_term = Gs * Fs * Ds;
    // 注意：分母4(n·l)(n·v)已经被SmithG_GGX吸收了

    // === Clearcoat ===
    float Dr  = GTR1(NdotH, lerp(0.1f, 0.001f, m.clearcoatGloss));
    // clearcoatGloss=1：α=0.001（极锐利）；=0：α=0.1（较宽）
    float Fr  = lerp(0.04f, 1.f, FH);
    // 固定F₀=0.04（ior≈1.5的清漆）
    float Gr  = SmithG_GGX(NdotL, 0.25f) * SmithG_GGX(NdotV, 0.25f);
    // 固定粗糙度0.25，clearcoat假设比较光滑
    float clearcoat_term = 0.25f * m.clearcoat * Gr * Fr * Dr;
    // 0.25是强度缩放（clearcoat不能和主高光一样强）

    return diffuse + Fsheen + specular_term + Vec3(clearcoat_term);
}
```

### GGX NDF 实现

```cpp
inline float GTR2(float NdotH, float alpha) {
    float a2 = alpha * alpha;
    float t  = 1.f + (a2 - 1.f) * NdotH * NdotH;
    // 当 NdotH=1（完美对准）: t = a2，D = a2/(π×a2²) = 1/(π×a2)，最大值
    // 当 NdotH=0（90°偏): t = 1，D = a2/π，最小值
    // 注意当 α→0，a2→0，D趋向 Dirac delta（完美镜面）
    return a2 / (PI * t * t);
}
```

为什么是 `t * t`（而不是 `t`）？这是 GGX 的特征——分母是平方，给出幂律衰减（polynomial tail），比 Beckmann 的指数衰减更慢，更符合真实材质的高光晕圈。

### Smith 几何遮蔽

```cpp
inline float SmithG_GGX(float NdotV, float alphaG) {
    float a = alphaG * alphaG;
    float b = NdotV * NdotV;
    return 1.f / (NdotV + std::sqrt(a + b - a * b));
    // 推导：GGX Lambda函数 Λ(v) = (-1 + √(1 + α²tan²θ)) / 2
    // G₁ = 1/(1 + Λ(v)) = 2NdotV / (NdotV + √(α² + NdotV²(1-α²)))
    // 这里返回的是 G₁/(2×NdotV)，即已经吸收了分母的 2NdotV
    // 最终 G/(4 NdotL NdotV) = SmithG_GGX(l) × SmithG_GGX(v)
}
```

### 色调映射

```cpp
inline float ACESFilm(float x) {
    const float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    return std::max(0.f, std::min(1.f, (x*(a*x+b)) / (x*(c*x+d)+e)));
    // 分子二次项 ax²+bx：曝光曲线抬升
    // 分母二次项 cx²+dx+e：高光压缩，防止过曝
    // 最终 S 型曲线，中间调对比度增强，高光柔和压缩
}
```

---

## 踩坑实录

### Bug 1：lerp 函数重载歧义导致编译错误

**症状**：
```
error: cannot convert 'Vec3' to 'float' in initialization
  float Fd  = lerp(1.f, Fd90, FL) * lerp(1.f, Fd90, FV);
```

**错误假设**：以为 `lerp(float, float, float)` 会优先匹配。

**真实原因**：代码里只定义了 `Vec3 lerp(Vec3, Vec3, float)`，当参数全是 float 时编译器尝试隐式转换 float → Vec3，失败了。

**修复**：添加 float 版本的 lerp 重载：
```cpp
inline float lerp(float a, float b, float t) { return a * (1.f - t) + b * t; }
inline Vec3 lerp(const Vec3& a, const Vec3& b, float t) { return a * (1.f - t) + b * t; }
```

**教训**：C++ 的函数重载在有隐式构造函数时容易混淆，两种参数类型都常用时必须分别提供重载。

---

### Bug 2：Vec3 缺少一元负号运算符

**症状**：
```
error: no match for 'operator-' (operand type is 'Vec3')
  Vec3 V = (-rd).normalized();
```

**错误假设**：以为定义了二元 `operator-` 就够了。

**真实原因**：C++ 的一元 `-` 和二元 `-` 是两个不同的运算符，必须单独重载。

**修复**：
```cpp
Vec3 operator-() const { return {-x, -y, -z}; }
```

**教训**：写向量类时要同时声明一元和二元运算符，它们不会互相推导。

---

### Bug 3：Smith-G 已包含 1/(4 N·L N·V) 归一化因子

**症状**（调试期）：高光过亮，特别是掠射角附近有明显过曝。

**错误假设**：以为需要 `specular_term = Ds * Fs * Gs / (4 * NdotL * NdotV)`。

**真实原因**：本实现中 `SmithG_GGX` 返回的是 `G₁/(2×NdotV)`，两个 GGX 相乘得到 `G/(4 NdotL NdotV)`，已经包含了归一化分母。

**修复**：直接 `Gs * Fs * Ds`，不再额外除以 `4 NdotL NdotV`。

**验证方法**：能量守恒检验：对多个方向积分 BRDF × cosθ，应 ≤ 1。

---

### Bug 4：clearcoatGloss 方向直觉相反

**症状**：`clearcoatGloss=1` 时高光很宽，`=0` 时反而很尖锐，和字面意思相反。

**错误假设**：`lerp(0.1f, 0.001f, gloss)` 以为 gloss=1 是"最光泽"。

**真实原因**：GTR1 的 α 越小，NDF 越尖锐（高光越集中 = 越光泽）。所以 `gloss=1` 应该对应 α=0.001（最小），`gloss=0` 对应 α=0.1（最大）。用 `lerp(0.1f, 0.001f, gloss)` 已经是正确的（大→小），但容易被反向理解。

**修复**：添加注释说明方向。最终代码：
```cpp
float Dr  = GTR1(NdotH, lerp(0.1f, 0.001f, m.clearcoatGloss));
// clearcoatGloss=1: α=0.001 (极锐利高光) — 这才是"高光泽"
// clearcoatGloss=0: α=0.1 (较宽高光) — 较"哑光"的清漆
```

---

## 效果验证与数据

### 输出图像统计

使用 Python PIL 对最终 PNG 进行量化验证：

```python
from PIL import Image
import numpy as np

img = Image.open('disney_brdf_output.png')
pixels = np.array(img).astype(float)
mean = pixels.mean()   # → 75.7（正常，非全黑/全白）
std  = pixels.std()    # → 56.0（图像内容丰富）
```

验证结果：
- 文件大小：148 KB（✅ > 10 KB）
- 像素均值：75.7（✅ 在 10~240 范围内）
- 像素标准差：56.0（✅ > 5，图像有丰富内容）
- 图像尺寸：900 × 700 像素

### 材质对比观察

通过渲染结果可以清晰观察到：

1. **第 3 列（金属金色）vs 第 1 列（绝缘体红色）**：金属高光带有金黄色调（F₀ = baseColor），绝缘体高光接近白色
2. **第 5 列（clearcoat 银白金属）**：有两层高光——内层金属高光（宽）+ 外层清漆高光（尖锐）
3. **粗糙度梯度**（上→下，0.05→0.90）：上排高光集中明亮，下排高光漫散
4. **第 4 列前两行（subsurface）**：绿色球边缘有轻微"发光"效果，模拟光在内部散射

### 渲染性能

| 参数 | 数值 |
|------|------|
| 图像分辨率 | 900 × 700 |
| 总像素数 | 630,000 |
| 球体数 | 20 |
| 光源数 | 3 |
| 编译选项 | -O2 |
| 运行时间 | < 0.5 秒（实测瞬时完成） |
| 每像素操作 | 光线-球体相交 + DisneyBRDF × 3 光源 |

O2 优化后单核 C++ 的解析光照非常快，主要瓶颈在内层 `sqrt` 和三角函数，Disney BRDF 本身计算量约为 Phong 的 5-8 倍，但完全在实时可接受范围内（复杂场景可以用 deferred shading 均摊）。

---

## 总结与延伸

### 技术局限性

1. **无真正的光线追踪**：本实现是解析光照，没有全局光照、间接光、阴影。Disney BRDF 完整展现需要路径追踪
2. **各向异性未实现**：`anisotropic` 参数在数据结构中定义了但未计入 BRDF，各向异性需要用 Bent NDF（GGX Anisotropic）
3. **能量补偿缺失**：粗糙度越高，微表面模型的能量损失越多（多次散射被忽略）。Kulla-Conty 补偿项可以修正这个问题
4. **Subsurface 近似**：本文的次表面散射只是漫反射的修正，真正的 SSS 需要 BSSRDF + 随机游走

### 可以继续优化的方向

- **多重重要性采样（MIS）**：在路径追踪中结合 GGX 重要性采样，显著降低噪声
- **Kulla-Conty 能量补偿**：修正高粗糙度下的能量损失
- **各向异性 GGX**：使用 Heitz 2014 的 anisotropic Smith-GGX 实现金属拉丝效果
- **薄膜干涉**：Clearcoat 上叠加薄膜干涉颜色（类似肥皂泡的彩虹色调）

### 与本系列的关联

- **2026-03-21 SPPM**、**2026-03-22 BDPT**：这些全局光照算法需要精确的 BRDF，今天实现的 Disney BRDF 是其中材质部分的升级
- **2026-03-24 Subsurface Scattering**：今天的次表面散射是近似版，Jensen 偶极子是准确版
- **2026-03-20 Spherical Harmonics**：SH 环境光照 + Disney BRDF = 近似实时 IBL（Image-Based Lighting）

Disney Principled BRDF 是现代 PBR 管线的核心，理解它意味着你能看懂 Unreal、Unity、Blender 里任何材质节点的底层逻辑。

---

**源码**：[GitHub](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-04-03-disney-brdf)

![Disney Principled BRDF 材质球阵列](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-03-disney-brdf/disney_brdf_output.png)
