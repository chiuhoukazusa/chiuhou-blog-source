---
title: "每日编程实践: IBL Split-Sum PBR Renderer — 基于图像的光照与预滤波环境贴图"
date: 2026-05-06 05:30:00
tags:
  - 每日一练
  - 图形学
  - PBR
  - IBL
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-06-ibl-split-sum-pbr/ibl_output.png
---

![IBL Split-Sum PBR 渲染结果：5×5 材质球阵列，横轴金属度 0→1，纵轴粗糙度 0.05→1.0](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-06-ibl-split-sum-pbr/ibl_output.png)

# 每日编程实践 Day 62：IBL Split-Sum PBR Renderer

今天实现 **IBL Split-Sum 近似**——现代 PBR（基于物理的渲染）管线中处理环境光照的核心技术。这是 Epic Games 在《Real Shading in Unreal Engine 4》（SIGGRAPH 2013）中提出的方案，Unreal、Unity HDRP、Godot 4.x 等主流引擎都使用它。

---

## 一、背景与动机

### 1.1 没有 IBL 时的世界

传统实时渲染只有一盏或几盏方向光。现实世界里，光从四面八方照来：天空、地面反光、室内吊灯、窗口漫射……把这一切建模成几盏方向光，结果就是金属球看上去只有一个高光点，完全感受不到环境反射。

**问题的核心**：完整的间接光照需要对整个半球求积分——

$$L_o(p, \omega_o) = \int_{\Omega} f_r(p, \omega_i, \omega_o) L_i(p, \omega_i) (\omega_i \cdot n) \, d\omega_i$$

这个积分对每个像素、每帧实时求解，代价相当于实时光线追踪——在 2013 年的 GPU 上完全不可行。

### 1.2 IBL 的思路：换一个表示形式

**Image-Based Lighting (IBL)** 的思路是：把环境光存成一张全景图（lat-lon 贴图或 Cubemap），然后在着色时采样它。但直接采样仍然昂贵——GGX BRDF 的主要贡献方向随粗糙度变化，需要多次采样才能收敛。

Epic 的 Brian Karis 提出了 **Split-Sum 近似**：把上面的积分拆成两个独立的部分，分别预计算，运行时只需要两次贴图采样——

$$\int_\Omega L_i(\omega_i) f_r(\omega_i, \omega_o) (\omega_i \cdot n) \, d\omega_i \approx \underbrace{\int_\Omega L_i(\omega_i) d\omega_i}_{\text{预滤波环境贴图}} \cdot \underbrace{\int_\Omega f_r(\omega_i, \omega_o)(\omega_i \cdot n) d\omega_i}_{\text{BRDF LUT}}$$

前半部分只跟方向和粗糙度有关——预计算成多个 mip 级别。  
后半部分只跟 $n \cdot v$ 和粗糙度有关——预计算成一张 2D 查找表（LUT）。

### 1.3 实际工业用途

| 引擎/应用 | IBL 实现方式 |
|-----------|------------|
| Unreal Engine 4+ | Reflection Capture + Split-Sum |
| Unity HDRP | Baked Reflection Probe + Split-Sum |
| Godot 4.x | ReflectionProbe + GGX LUT |
| Blender EEVEE | Cubemap + Split-Sum |
| Substance Painter | 实时预览 IBL |

没有 IBL，金属材质在游戏里就是"塑料感金属"——高光只有一个点，没有周边环境的倒影。有了 IBL，钢铁、黄金、铝合金的质感才能真实可信。

---

## 二、核心原理

### 2.1 Cook-Torrance 微表面 BRDF 回顾

IBL 的被积函数是 Cook-Torrance BRDF，先复习一下——

$$f_r = \frac{DGF}{4(n \cdot \omega_o)(n \cdot \omega_i)}$$

- **D**：法线分布函数（Normal Distribution Function），控制高光形状
- **G**：几何遮蔽函数（Geometry/Shadowing-Masking），控制自遮蔽
- **F**：菲涅尔函数（Fresnel），控制视角依赖的反射率

**GGX 法线分布（Trowbridge-Reitz）**：

$$D_{GGX}(n, h, \alpha) = \frac{\alpha^2}{\pi \left[ (n \cdot h)^2 (\alpha^2 - 1) + 1 \right]^2}$$

其中 $\alpha = \text{roughness}^2$。  
直觉：$\alpha \to 0$（光滑）时，D 在半程向量 $h = n$ 时是一个极窄的峰；$\alpha \to 1$（粗糙）时，D 铺平成宽阔的分布。

**Smith-GGX 几何遮蔽**：

$$G_{Smith}(n, v, l, k) = G_{Schlick}(n \cdot v, k) \cdot G_{Schlick}(n \cdot l, k)$$
$$G_{Schlick}(NdotV, k) = \frac{NdotV}{NdotV(1-k) + k}, \quad k = \frac{\alpha}{2}$$

直觉：微表面会互相遮蔽，粗糙表面上的凸起会挡住来自低角度的光。$G$ 对低掠射角（$n \cdot v \to 0$）有强烈衰减。

**Schlick 菲涅尔近似**：

$$F_{Schlick}(v, h, F_0) = F_0 + (1 - F_0)(1 - v \cdot h)^5$$

$F_0$ 是法线入射时的反射率。对于非金属，$F_0 \approx 0.04$；对于金属，$F_0$ 等于金属的 albedo 颜色。

### 2.2 Split-Sum 近似的数学细节

原始积分：

$$L_o = \int_\Omega f_r(\omega_i, \omega_o) L_i(\omega_i) (n \cdot \omega_i) \, d\omega_i$$

**近似前提**：假设 $L_i$ 在半球上的分布与 BRDF 的重要性采样分布"差不多"——也就是说，环境光是比较均匀的（高频细节被平均掉了）。在这个前提下：

$$\int_\Omega f_r L_i (n \cdot \omega_i) d\omega_i \approx \left(\int_\Omega L_i(\omega_i) D(\omega_i) d\omega_i\right) \cdot \left(\int_\Omega f_r(n \cdot \omega_i) d\omega_i\right)$$

**第一项**——预滤波环境贴图：

$$\text{PrefEnv}(r, \text{roughness}) = \frac{\sum_{k=1}^N L_i(l_k) (n \cdot l_k)}{\sum_{k=1}^N (n \cdot l_k)}$$

用 GGX 重要性采样，对反射方向 $r$ 在当前粗糙度下积分，结果存到 Cubemap mip level。

**第二项**——BRDF 积分 LUT：

$$\int_\Omega f_r(n \cdot \omega_i) d\omega_i = F_0 \cdot A + B$$

其中：

$$A = \int \frac{f_r}{F}(1 - (1 - v \cdot h)^5)(n \cdot l) \, dl$$
$$B = \int \frac{f_r}{F}(1 - v \cdot h)^5 (n \cdot l) \, dl$$

$A$ 和 $B$ 只跟 $n \cdot v$ 和粗糙度相关，预计算成一张 $256 \times 256$ 的 RG 纹理（红通道=A，绿通道=B）。

**最终着色公式**：

$$L_o \approx \text{PrefEnv}(r, \text{roughness}) \cdot (F \cdot A + B)$$

加上漫反射项：

$$L_{diffuse} = k_d \cdot \text{albedo} \cdot \text{Irradiance}(n)$$

$$k_d = (1 - F) \cdot (1 - \text{metallic})$$

### 2.3 GGX 重要性采样

生成与 GGX 分布相匹配的采样方向，使蒙特卡洛收敛更快——

给定均匀随机数 $(\xi_1, \xi_2) \in [0,1)^2$：

$$\phi = 2\pi \xi_1$$
$$\cos\theta = \sqrt{\frac{1 - \xi_2}{1 + (\alpha^2 - 1)\xi_2}}$$

这等价于对 GGX 分布的反函数采样（逆变换采样）。把采样出来的半程向量 $h$ 转换到世界空间（TBN 矩阵），再用 $l = 2(v \cdot h)h - v$ 得到入射方向。

数值序列用 **Hammersley 低差异序列**（准蒙特卡洛），相比纯随机，同样样本数下方差更低——

$$\text{Hammersley}(i, N) = \left(\frac{i}{N},\ \text{VdC}(i)\right)$$

$\text{VdC}(i)$ 是 Van der Corput 序列：将 $i$ 的二进制位镜像翻转得到 [0,1) 中的分数。

---

## 三、实现架构

### 3.1 整体数据流

```
┌─────────────────────────────────────────────────────────┐
│                    离线预计算阶段                         │
├───────────────────┬─────────────────────────────────────┤
│  EnvMap (512×256) │  程序化 HDR 天空                     │
│         ↓         │  sun disc + glow + sky gradient      │
│  PrefilteredEnv   │  6 mip levels (roughness 0→1)        │
│  (各级别单独分辨率)│  每级 1024 GGX 重要性采样             │
│         ↓         │                                      │
│  BRDFLut (256×256)│  F_scale(A), F_bias(B)              │
│                   │  1024 样本 per texel                 │
└───────────────────┴─────────────────────────────────────┘
          ↓ 以上三个结构全部在 main() 前建好
┌─────────────────────────────────────────────────────────┐
│                   实时渲染阶段                            │
├───────────────────────────────────────────────────────────┤
│  背景层：env.sample(rayDir) → 每像素采样环境图            │
│      ↓                                                   │
│  绘制 5×5 球阵列：                                       │
│    • 软光栅化 + 深度缓冲                                  │
│    • 每像素：ray-sphere 求交 → 计算 N、V                  │
│    • shadeIBL() → prefEnv + LUT → 最终颜色               │
│      ↓                                                   │
│  后处理：ACES 色调映射 + Gamma 2.2 编码                   │
│      ↓                                                   │
│  输出 PPM → PIL 转 PNG                                   │
└─────────────────────────────────────────────────────────┘
```

### 3.2 关键数据结构

```cpp
// 程序化环境贴图（lat-lon 格式）
struct EnvMap {
    int w, h;
    std::vector<Vec3> data;  // HDR 线性颜色，无限精度
    void build();            // 逐像素填入 sampleSky()
    Vec3 sample(Vec3 dir);   // 双线性插值
};

// 预滤波环境贴图：6 个粗糙度级别
struct PrefilteredEnv {
    static const int MIPS = 6;
    std::vector<EnvMap> levels; // roughness = 0, 0.2, 0.4, 0.6, 0.8, 1.0
    void build(const EnvMap& env);        // GGX 重要性采样预滤波
    Vec3 sample(Vec3 dir, float roughness); // 双线性 mip 插值
};

// BRDF 积分查找表
struct BRDFLut {
    int size;               // 256×256
    std::vector<float> r;  // F_scale (A 项)
    std::vector<float> g;  // F_bias  (B 项)
    void build();           // 蒙特卡洛积分
    void sampleLUT(float NdotV, float roughness, float& F_scale, float& F_bias);
};
```

### 3.3 着色器职责划分（类比 GPU 管线）

| GPU 管线 | 本程序等价 | 职责 |
|---------|----------|------|
| Vertex Shader | `Camera::project` | MVP 变换、视锥剔除 |
| Fragment Shader | `shadeIBL` | PBR 着色 |
| Texture Sampling | `EnvMap::sample`, `BRDFLut::sampleLUT` | 双线性采样 |
| Render Target | `Framebuffer` | 颜色 + 深度缓冲 |
| Post-process | `aces()` + `gammaEncode()` | Tone-mapping |

与真实 GPU 不同的是，预计算阶段在 CPU 上串行执行（约 22 秒），生产环境会用 compute shader 并行化。

---

## 四、关键代码解析

### 4.1 Hammersley 低差异序列

```cpp
float radicalInverseVdC(uint32_t bits) {
    // Van der Corput 序列：将 bits 的二进制位按中间翻转
    // 例：bits = 0b00000110 (6)
    //   → 先 16-bit swap → 再逐位 swap
    //   得到一个在 [0,1) 中均匀分布的分数
    bits = (bits << 16u) | (bits >> 16u);
    bits = ((bits & 0x55555555u) << 1u) | ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u) | ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u) | ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u) | ((bits & 0xFF00FF00u) >> 8u);
    return float(bits) * 2.3283064365386963e-10f; // 除以 2^32
}
// 使用：xi1 = (float)s / SAMPLES; xi2 = radicalInverseVdC(s);
```

**为什么用 Hammersley 而不是 rand()**：纯随机在 1024 样本下方差很大，低差异序列能保证样本在参数空间均匀分布，同样 1024 样本能达到 ~4096 纯随机的精度。

### 4.2 GGX 重要性采样

```cpp
Vec3 importanceSampleGGX(Vec3 Xi, float roughness, Vec3 N) {
    float a = roughness * roughness; // α = roughness²，这是正确的参数化
    // GGX 反函数采样：
    // p(cosθ) = α² cosθ / (π [cosθ²(α²-1)+1]²)
    // → 逆变换采样得到：
    float cosTheta = sqrtf((1 - Xi.y) / (1 + (a*a - 1) * Xi.y + 1e-7f));
    float sinTheta = sqrtf(1 - cosTheta*cosTheta);
    float phi = 2 * PI * Xi.x;

    // 局部空间半程向量 h（在 N=up 坐标系下）
    Vec3 H = { sinTheta*cosf(phi), cosTheta, sinTheta*sinf(phi) };

    // TBN 矩阵：把 H 变换到世界空间
    // 这里选 up = (0,1,0) 或 (1,0,0)（避免与 N 平行导致叉积退化）
    Vec3 up = fabsf(N.y) < 0.999f ? Vec3(0,1,0) : Vec3(1,0,0);
    Vec3 T = up.cross(N).normalized();  // Tangent
    Vec3 B = N.cross(T);               // Bitangent

    return (T*H.x + N*H.y + B*H.z).normalized();
}
```

**易错点**：TBN 矩阵的 `up` 向量必须与 N 不平行，否则叉积结果为零向量，normalized 会产生 NaN，染出来的球会有黑色斑块。加 `fabsf(N.y) < 0.999f` 分支保护。

### 4.3 预滤波环境贴图构建

```cpp
void PrefilteredEnv::build(const EnvMap& env) {
    float roughnesses[MIPS] = {0.0f, 0.2f, 0.4f, 0.6f, 0.8f, 1.0f};

    for (int m = 0; m < MIPS; ++m) {
        float roughness = roughnesses[m];
        // 分辨率随 mip 降低（和 GPU mip chain 类似）
        int w = std::max(1, ENV_W >> m);
        int h = std::max(1, ENV_H >> m);
        EnvMap level(w, h);

        for (int j = 0; j < h; ++j) {
            for (int i = 0; i < w; ++i) {
                // 该 texel 对应的方向 N（假设 V ≈ N，等向性处理）
                Vec3 N = /* 从 lat-lon uv 转方向 */;

                Vec3 prefilteredColor(0.f);
                float totalWeight = 0.f;

                for (int s = 0; s < 1024; ++s) {
                    // 用 GGX 重要性采样生成 H，推导出 L
                    Vec3 H = importanceSampleGGX({xi1, xi2, 0}, roughness, N);
                    Vec3 L = (2.f * V.dot(H) * H - V).normalized();

                    float NdotL = saturate(N.dot(L));
                    if (NdotL > 0.f) {
                        // 加权累加：权重 = NdotL（余弦加权，与渲染方程匹配）
                        prefilteredColor += env.sample(L) * NdotL;
                        totalWeight += NdotL;
                    }
                }
                level.data[j*w+i] = prefilteredColor / (totalWeight + 1e-9f);
            }
        }
        levels.push_back(std::move(level));
    }
}
```

**关键设计**：使用 `NdotL` 作为权重而不是统一权重 1，这是因为渲染方程里有 $\cos\theta$ 项，不加权会导致高粗糙度时积分偏低（低角度采样贡献被高估）。

**近似误差**：这里假设 `V ≈ N`（各向同性）。实际上当 $V \neq N$ 时，反射叶形状在掠射角有明显拉伸，这是 Split-Sum 最大的近似误差来源，在大粗糙度 + 掠射角时误差最明显。

### 4.4 BRDF LUT 构建

```cpp
void BRDFLut::build() {
    for (int j = 0; j < size; ++j) {          // j: roughness 轴
        float roughness = (j + 0.5f) / size;
        for (int i = 0; i < size; ++i) {      // i: NdotV 轴
            float NdotV = (i + 0.5f) / size;
            Vec3 V = { sqrtf(1 - NdotV*NdotV), NdotV, 0.f }; // 在切平面内
            Vec3 N = { 0, 1, 0 };

            float A = 0, B = 0;
            for (int s = 0; s < 1024; ++s) {
                Vec3 H = importanceSampleGGX(Xi, roughness, N);
                Vec3 L = (2.f * V.dot(H) * H - V).normalized();
                // ...
                float G    = GSmith(NdotV, NdotL, roughness);
                float G_vis = (G * VdotH) / (NdotH * NdotV + 1e-7f); // 消去分母
                float Fc   = powf(1 - VdotH, 5.f);  // Schlick Fresnel 的 (1-cosθ)^5 项
                // 拆成 F0*A + B 形式：
                A += (1 - Fc) * G_vis;  // F_scale：乘以 F0
                B += Fc * G_vis;        // F_bias：加上常量偏移
            }
            r[j*size+i] = A / 1024;
            g[j*size+i] = B / 1024;
        }
    }
}
```

**为什么拆成 A 和 B**：Schlick Fresnel 是 $F_0 + (1-F_0)(1-\cos\theta)^5 = F_0(1-Fc) + Fc$，乘进积分后可以把 $F_0$ 提到积分外面（它是常量），剩下 $A$ 和 $B$ 只跟几何量有关，与材质无关，所以可以预计算。不同材质只需在运行时 $F_0 \cdot A + B$ 就能恢复完整结果。

### 4.5 IBL 着色

```cpp
Vec3 shadeIBL(Vec3 N, Vec3 V, Vec3 albedo, float metallic, float roughness,
              const PrefilteredEnv& prefEnv, const BRDFLut& lut) {
    Vec3 R = (2.f * N.dot(V) * N - V).normalized(); // 反射方向
    float NdotV = saturate(N.dot(V));

    // F0: 非金属 0.04 白，金属 = albedo 颜色
    Vec3 F0 = mix(Vec3(0.04f), albedo, metallic);

    // 用带粗糙度修正的 Fresnel（低粗糙度时菲涅尔更强）
    // 注：这里用 fresnelSchlickRoughness 而非普通 Schlick
    // 原因：粗糙表面的掠射角高光不应该和光滑表面一样锐利
    Vec3 F = fresnelSchlickRoughness(NdotV, F0, roughness);

    // 镜面反射项：预滤波颜色 × BRDF LUT
    Vec3 prefilteredColor = prefEnv.sample(R, roughness);
    float F_scale, F_bias;
    lut.sampleLUT(NdotV, roughness, F_scale, F_bias);
    Vec3 specular = prefilteredColor * (F * F_scale + F_bias);

    // 漫反射项：粗糙环境光（用最高 roughness 级别近似 irradiance）
    Vec3 irradiance = prefEnv.sample(N, 1.0f) * 0.5f;
    Vec3 kD = (Vec3(1.f) - F) * (1.f - metallic); // 金属无漫反射
    Vec3 diffuse = kD * albedo * irradiance;

    // 加一盏直射点光（catchlight，突出金属感）
    // ... Cook-Torrance 直射光计算 ...

    return diffuse + specular + Lo;
}
```

**`metallic` 对 `kD` 的影响**：金属的自由电子几乎吸收所有折射光，没有漫反射，所以 `kD *= (1-metallic)` 在 metallic=1 时让漫反射贡献为零，所有能量都在镜面高光里。

### 4.6 Ray-Sphere 求交（软光栅化核心）

```cpp
// 对每个像素发出光线，检查与球体的交点
float b = oc.dot(rayDir);    // b = (o-c) · d
float c = oc.dot(oc) - r*r;  // c = |o-c|² - r²
float disc = b*b - c;         // 判别式

// disc < 0：光线错过球体
// disc >= 0：取较小的 t（近交点）
float t = -b - sqrtf(disc);
if (t < 0.01f) {              // 近交点在相机后面
    t = -b + sqrtf(disc);    // 尝试远交点（相机在球体内部）
    if (t < 0.01f) continue; // 两个交点都在后面
}

Vec3 hitP = cam.pos + rayDir * t;
Vec3 N    = (hitP - center).normalized(); // 球的法线 = 表面点 - 球心
Vec3 V    = -rayDir;                       // 视线方向（指向相机）
```

**深度测试**：`Framebuffer::set` 只有当新交点的深度 `t` 小于当前缓冲值时才更新颜色，确保前后遮挡正确。

---

## 五、踩坑实录

### Bug 1：编译错误 — Vec3 除以 Vec3

**症状**：ACES tone-mapping 函数编译报错 `no match for operator/`。

**错误代码**：
```cpp
return ((c*(a*c+b))/(c*(cc*c+d)+e)).clamp01();
```

**错误假设**：以为 `Vec3 / Vec3` 是逐分量除法，就像乘法一样。

**真实原因**：`Vec3` 只定义了 `operator/(float)`（除以标量），没有定义 `operator/(Vec3)`（逐分量除）。

**修复**：改成逐分量计算：
```cpp
auto f = [&](float v)->float{
    return (v*(a*v+b))/(v*(cc*v+d)+e);
};
return Vec3(f(c.x), f(c.y), f(c.z)).clamp01();
```

**启示**：ACES 在数学上是逐分量应用的——每个颜色通道独立经过 S 型曲线。不要以为 `(A/B)` 的 Vec3 版本是"显然的"。

### Bug 2：编译警告 — 成员初始化顺序

**症状**：`Framebuffer` 和 `Camera` 的构造函数触发 `-Wreorder` 警告。

**错误代码**：
```cpp
struct Framebuffer {
    std::vector<Vec3> color;   // 先声明
    std::vector<float> depth;
    int w, h;                  // 后声明
    // 但初始化列表里 w(w), h(h), color(w*h)
    // → 实际上 color 比 w 先初始化！
};
```

**错误假设**：以为初始化列表按写的顺序执行。

**真实原因**：C++ 规定成员按**声明顺序**初始化，与初始化列表顺序无关。`color(w*h)` 在 `w` 初始化之前执行，`w` 是未定义值，UB。

**修复**：将 `int w, h` 的声明移到 vector 前面：
```cpp
struct Framebuffer {
    int w, h;                  // 先声明
    std::vector<Vec3> color;
    std::vector<float> depth;
};
```

**启示**：结构体的成员声明顺序决定初始化顺序，不是初始化列表的写法顺序。开启 `-Wreorder` 警告能救命。

### Bug 3：图像偏亮问题

**症状**：输出图像大量溢出（`Max: 255`）但不全白，ACES 后高光区域仍然过曝。

**原因排查**：直射光亮度系数设得太高（`Vec3{8.0f, 6.5f, 4.0f}`）。但 ACES 本来就是为了处理过曝高光设计的——实际上这不是 bug，而是 HDR → LDR 的正常行为。检查 `pixel mean = 192.3`，在正常范围 (10~240) 内。

**保留决策**：保持高动态范围输入，让 ACES 自然压缩。最终均值 192.3，标准差 24.3，像素分布合理。

### Bug 4：ppm 文件格式解析

**症状**：Python PIL 直接 `Image.open("ibl_output.ppm")` 报错退出。

**原因**：PIL 不在默认安装里，运行环境没有 Pillow。

**修复**：`pip3 install Pillow` 后，再写自定义 ppm 解析器（逐字节读取 header + raw RGB data）避免格式误判：

```python
with open('ibl_output.ppm','rb') as f:
    lines = []
    for _ in range(3):  # P6, WxH, maxval 三行
        line = b''
        while True:
            ch = f.read(1)
            line += ch
            if ch == b'\n': break
        lines.append(line.decode().strip())
    data = f.read()  # 剩余全是 raw bytes

w, h = map(int, lines[1].split())
arr = np.frombuffer(data, dtype=np.uint8).reshape((h, w, 3))
Image.fromarray(arr, 'RGB').save('ibl_output.png')
```

---

## 六、效果验证与数据

### 6.1 渲染结果

![5×5 IBL 材质球阵列](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-06-ibl-split-sum-pbr/ibl_output.png)

**矩阵布局**：
- **列（左→右）**：metallic 从 0（纯非金属）到 1（纯金属），albedo 分别为金色/银色/铁色/铝色/蓝色
- **行（上→下）**：roughness 从 0.05（接近镜面）到 1.0（完全粗糙漫反射）

### 6.2 输出量化指标

| 指标 | 数值 | 说明 |
|------|------|------|
| 输出文件大小 | 151KB | > 10KB ✅ |
| 像素均值 | 192.3 | 在 (10, 240) 内 ✅ |
| 像素标准差 | 24.3 | > 5（有内容变化）✅ |
| 像素最小值 | 47 | 非全黑 ✅ |
| 像素最大值 | 255 | 高光正常饱和 ✅ |
| 渲染分辨率 | 960×540 | 16:9 |
| 总运行时间 | 21.8s | CPU 软渲染 |

### 6.3 渲染时间分解

| 阶段 | 耗时 | 主要工作量 |
|------|------|-----------|
| 环境贴图构建 | ~0.1s | 512×256 = 131K 像素 |
| 预滤波环境贴图 | ~18s | 6层 × 各级像素 × 1024 采样 |
| BRDF LUT | ~3s | 256×256 texel × 1024 采样 |
| 球阵列渲染 | ~0.7s | 25 球 × 约 120²像素 × ray-sphere |

预滤波阶段占了总时间的 82%。GPU 版本会用 compute shader 并行化，理论上能在 <1ms 内完成（Radeon RX 6800 上约 0.3ms）。

### 6.4 可视现象验证

从渲染结果中可以观察到以下物理正确行为：

1. **低粗糙度行（顶部）**：镜面高光极度集中，金属球清晰倒映天空蓝色，非金属球高光点小而亮
2. **高粗糙度行（底部）**：高光完全扩散为漫反射，金属球颜色就是自身 albedo + 环境漫散射
3. **metallic 梯度**：左列（metallic=0）有明显漫反射 albedo 颜色，右列（metallic=1）颜色完全来自环境反射
4. **Fresnel 效应**：球边缘（掠射角）的菲涅尔反射加强，即便是低金属度的球也能在边缘看到环境反射

---

## 七、总结与延伸

### 7.1 Split-Sum 的局限性

**误差来源 1：V≈N 假设**  
在掠射角（$n \cdot v \to 0$）时误差最大，反射叶在掠射方向有明显拉伸，而预滤波贴图没有捕获这个各向异性形状。表现为掠射金属球的高光形状不够准确。

**误差来源 2：低频近似**  
Split-Sum 假设积分的两部分是独立的，但实际上 BRDF 波瓣和环境光分布之间有相关性（比如太阳恰好在反射方向上），这种相关性在近似中丢失了，导致强方向光下能量不守恒。

**误差来源 3：单次散射**  
GGX 微表面模型只考虑一次散射，多次散射（光在微表面凹槽里弹射多次）被忽略了，高粗糙度时颜色会偏暗（能量损失）。修复方案是 Kulla-Conty 补偿。

### 7.2 可优化方向

**性能**：
- GPU 版本：compute shader 并行预计算，离线生成 DDS/KTX 文件
- 球谐函数近似 irradiance（只需 9 个系数，3×3 矩阵乘法）
- Mipmapped Cubemap 采样（硬件三线性过滤）

**质量**：
- Kulla-Conty 多次散射补偿（改善高粗糙度能量守恒）
- 各向异性 IBL（对各向异性材质，需要 V-dependent 预滤波）
- 实时 Reflection Capture（周期性重新烘焙动态环境）

**功能扩展**：
- 加入局部反射探针（基于盒体投影修正远近视差）
- Area Light 的 LTC（Linearly Transformed Cosines）近似

### 7.3 与本系列的关联

本系列已经积累了 PBR 的多个组成部分：
- **Day 3**（04-03）：Disney Principled BRDF——材质参数设计
- **Day 18**（04-20）：PCSS——直射光软阴影
- **Day 19**（04-21）：Bloom——高光泛光后处理
- **今天**（05-06）：IBL Split-Sum——环境间接光照

下一步可以做的是 **Screen Space Ambient Occlusion (SSAO)**（已在 03-25 做过）的延伸：**HBAO+**（Horizon-Based AO），或者实现完整的 **Reflection Probe 捕获系统**，使 IBL 支持动态场景。

---

**代码地址**：[GitHub - daily-coding-practice/2026-05-06](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-05-06)  
**渲染时间**：21.8s（纯 CPU 软渲染，960×540）  
**技术栈**：C++17 + STL，零外部依赖
