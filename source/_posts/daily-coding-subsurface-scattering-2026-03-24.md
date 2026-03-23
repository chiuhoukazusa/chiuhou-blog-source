---
title: "每日编程实践: Subsurface Scattering Renderer（次表面散射渲染器）"
date: 2026-03-24 05:30:00
tags:
  - 每日一练
  - 图形学
  - 光照模型
  - C++
  - 渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-24-subsurface-scattering/sss_output.png
---

# 每日编程实践 Day 24/03: Subsurface Scattering Renderer

今天实现了 **次表面散射（Subsurface Scattering, SSS）** 渲染器。SSS 是让皮肤、玉石、蜡烛等半透明材质看起来真实的关键技术。在此之前，我们用 Lambertian 漫反射模型渲染皮肤，结果往往显得"塑料感十足"——因为真实皮肤的光不是在表面反射的，而是钻进去、在内部散射一番再从另一个地方射出来。这正是 SSS 的精髓。

![四材质渲染图：皮肤/大理石/蜡烛/玉石](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-24-subsurface-scattering/sss_output.png)

---

## 一、背景与动机——为什么普通漫反射不够用

### 1.1 漫反射模型的假设

标准的 Lambert/Phong/PBR 光照模型有一个隐含假设：**光在表面某一点入射，就在那一点反射**。用数学语言说，BRDF（双向反射分布函数）描述的是一个纯粹的表面现象：

```
Lo(xo, ωo) = ∫ f_r(xo, ωi, ωo) · Li(xo, ωi) · cosθi dωi
```

注意，入射点和出射点**是同一个点 xo**。这个假设对金属、塑料、光滑漆面非常准确，但对皮肤、大理石、玉石、蜡烛、牛奶却完全错了。

### 1.2 光在皮肤里发生了什么

皮肤是一种多层介质：
- **表皮层（Epidermis）**：厚约 0.1mm，含黑色素，强散射
- **真皮层（Dermis）**：厚约 1-4mm，含血红素，强吸收红色以外的光
- **皮下脂肪层**：高散射低吸收

光打到皮肤时，一部分在表面发生镜面反射（这就是皮肤的高光），另一部分**折射进入皮肤内部**，经历数百次到数千次的散射和吸收，最终从**入射点周围不同位置**射出来。这个过程可能涉及几毫米到数十毫米的横向扩散。

这就是为什么手指放在手电筒旁边透射看起来是红色的——皮肤强烈吸收绿色和蓝色，只有红色能穿透传播很远；也是为什么耳朵被灯光从背面照时会透出红色光晕。

### 1.3 工业界的使用现状

SSS 渲染技术在以下场景中是**标配**：
- **游戏角色皮肤**：Unreal Engine 5 的角色渲染管线专门有 Subsurface Profile 材质
- **影视级渲染**：《阿凡达》、《指环王》等 CG 角色都依赖 SSS 实现真实感皮肤
- **医学可视化**：模拟皮肤、组织对激光或光的响应
- **食品包装设计**：模拟牛奶、奶酪、水果的外观
- **珠宝/玉石渲染**：翡翠、白玉的半透明质感

---

## 二、核心原理——从 BRDF 到 BSSRDF

### 2.1 BSSRDF 的定义

为了正确描述次表面散射，Jensen 等人在 2001 年提出了 **BSSRDF（Bidirectional Subsurface Scattering Reflectance Distribution Function）** 的框架。与 BRDF 不同，BSSRDF 允许入射点和出射点不同：

```
Lo(xo, ωo) = ∫_A ∫_{2π} S(xi, ωi, xo, ωo) · Li(xi, ωi) · cosθi dωi dA
```

其中：
- `xi`：光线入射点（在表面 A 上积分）
- `xo`：观察点（我们关心的渲染点）
- `S(xi, ωi, xo, ωo)`：BSSRDF，描述从 xi 方向 ωi 入射的光在 xo 方向 ωo 射出的概率
- 积分遍历**整个表面** A，这是与 BRDF 的根本区别

这个式子的直觉是：xo 处的亮度，是来自**整个表面上所有可能入射点**的贡献之和。

### 2.2 Jensen 偶极子模型（Dipole Approximation）

BSSRDF 的完整计算极其昂贵，Jensen 提出了一个基于**扩散方程**的近似模型。

核心思路：把介质中的散射过程用**热传导方程（扩散方程）**近似。光在散射介质中的传播满足：

```
∇²φ(r) - σ_tr² · φ(r) = 0
```

其中 `φ` 是辐射通量密度，`σ_tr` 是有效传输系数。

对这个方程的格林函数（点源响应）求解，就得到了著名的**漫射剖面（Diffusion Profile）** `R_d(r)`，描述了距入射点距离 r 处的出射辐射密度。

#### 2.2.1 关键参数

首先定义几个材质参数：
- `σ_a`：吸收系数（材质吸收光的能力，单位 mm⁻¹）
- `σ_s`：散射系数（散射光的能力）
- `σ_sp = (1-g)·σ_s`：折减散射系数（考虑各向异性后的等效各向同性散射系数，g 是平均余弦/相函数参数）
- `σ_t = σ_a + σ_sp`：全消光系数
- `α' = σ_sp / σ_t`：折减单次散射反照率

有效传输系数反映光在介质中能传播多远：
```
σ_tr = sqrt(3 · σ_a · σ_t)
```
`1/σ_tr` 就是光的有效传播深度。蜡烛的 `σ_tr` 小，光能传很远；吸收性强的材质 `σ_tr` 大，光被快速吸收。

#### 2.2.2 偶极子公式

偶极子模型的核心：用一个**真实光源**和一个**虚拟镜像光源**来满足表面边界条件。

真实光源深度（从表面向内）：
```
z_r = 1 / σ_t
```

虚拟光源深度（从表面向外的镜像位置）：
```
z_v = z_r + 4·A·D   其中 D = 1/(3·σ_t) 为扩散系数
```

`A` 是边界修正系数，与折射率 η 相关：
```
F_dr = (Fresnel内反射系数的漫射近似)
A = (1 + F_dr) / (1 - F_dr)
```

最终的漫射剖面公式：

```
R_d(r) = (α'/(4π)) · [
    z_r · (σ_tr + 1/d_r) · exp(-σ_tr · d_r) / d_r²  +
    z_v · (σ_tr + 1/d_v) · exp(-σ_tr · d_v) / d_v²
]
```

其中：
```
d_r = sqrt(r² + z_r²)   // 到真实光源的距离
d_v = sqrt(r² + z_v²)   // 到虚拟光源的距离
```

**直觉解释**：
- 第一项是真实光源的贡献：光从表面进入，在深度 z_r 处扩散，然后向上传播距离 d_r 射出
- 第二项是虚拟光源（负值）修正边界条件：防止光从表面"逃逸"到外部介质
- `exp(-σ_tr · d)` 表示光在传播过程中因吸收而衰减
- `z·(σ_tr + 1/d)/d²` 是扩散方程的格林函数导数形式

这个函数描述的是一个**钟形曲线**：在 r=0 处最大（对应入射点正上方），随 r 增大指数衰减。散射强、吸收弱的材质（如蜡烛）衰减慢，散射效果明显；吸收强的材质衰减快。

![漫射剖面 R_d(r) 曲线：不同材质的对比](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-24-subsurface-scattering/sss_diffusion_profile.png)

### 2.3 Fresnel 折射率修正

`F_dr`（Fresnel 漫射反射率）用来处理光在介质表面的折射。当 η≥1 时，使用 Jensen 的多项式近似：
```
F_dr = -1.44/η² + 0.71/η + 0.668 + 0.0636·η
```
当 η<1 时：
```
F_dr = -0.4399 + 0.7099/η - 0.3319/η² + 0.0636/η³
```
这个近似在折射率 η ∈ [1.0, 2.0] 范围内误差小于 1%，足够实用。

### 2.4 Monte Carlo 表面积分

漫射剖面 `R_d(r)` 给了我们每个入射点的贡献权重，实际计算还需要在整个表面上积分：

```
L_sss(xo) = ∫_A R_d(|xi - xo|) · Li(xi) · dA(xi)
```

我们用 Monte Carlo 方法在球面上均匀采样 N 个入射点：
```
L_sss ≈ (1/N) · Σ R_d(|xi - xo|) · Li(xi) · (面积 / 采样密度)
       = Area · (1/N) · Σ R_d(|xi - xo|) · Li(xi)
```

---

## 三、实现架构

### 3.1 整体数据流

```
[相机光线] → [场景求交] → [命中SSS球]
                              ↓
                    [计算直接光照 (Blinn-Phong)]
                              +
                    [Monte Carlo 表面积分 SSS]
                              ↓
                    [混合直接光照 + SSS 贡献]
                              ↓
                    [ACES 色调映射 + Gamma 校正]
                              ↓
                         [写入 PPM]
```

### 3.2 关键数据结构

**SSSMaterial**：封装所有 SSS 相关参数及推导方法

```cpp
struct SSSMaterial {
    Vec3 sigma_a;    // 吸收系数（RGB 三通道独立）
    Vec3 sigma_sp;   // 折减散射系数
    double eta;      // 折射率

    // 导出参数（按需计算）
    Vec3 sigma_t()   const { return sigma_a + sigma_sp; }
    Vec3 sigma_tr()  const { /* sqrt(3*σ_a*σ_t) 逐通道 */ }
    double Fdr()     const { /* Fresnel 多项式近似 */ }
    double A()       const { return (1 + Fdr()) / (1 - Fdr()); }

    // 核心：漫射剖面
    Vec3 Rd(double r) const { /* 偶极子公式 */ }
};
```

**设计理由**：三通道独立处理是因为皮肤在红绿蓝波段的散射/吸收特性差异很大。比如皮肤 σ_a 在蓝色通道远大于红色通道，导致散射出来的光偏红色。

### 3.3 渲染管线的职责划分

| 阶段 | CPU 侧 | 备注 |
|------|--------|------|
| 场景求交 | Sphere::intersect | 简单球体析解法 |
| 直接光照 | directLighting() | Blinn-Phong，含阴影测试 |
| SSS 积分 | computeSSS() | Monte Carlo，球面均匀采样 |
| 混合 | renderPixel() | `result = direct * (1-w) + sss * w` |
| 后处理 | Image::savePPM() | ACES Tonemapping + Gamma 2.2 |

---

## 四、关键代码解析

### 4.1 偶极子漫射剖面实现

这是整个系统的核心，每个通道独立计算：

```cpp
Vec3 Rd(double r) const {
    Vec3 st = sigma_t();
    Vec3 sp = sigma_sp;

    // alpha' = sigma_sp / sigma_t（折减单次散射反照率）
    // 这个比值控制材质的"散射强度"
    Vec3 alphap = Vec3(sp.x/st.x, sp.y/st.y, sp.z/st.z);

    Vec3 sigtr = sigma_tr();  // 有效传输系数
    double a = A();           // 边界修正系数

    // 偶极子深度（真实/虚拟光源到表面的距离）
    // z_r = 1/sigma_t 是散射平均自由程
    Vec3 zr = Vec3(1.0/st.x, 1.0/st.y, 1.0/st.z);
    // z_v 通过扩散长度 D = 1/(3*sigma_t) 修正
    Vec3 D = Vec3(1.0/(3.0*st.x), 1.0/(3.0*st.y), 1.0/(3.0*st.z));
    Vec3 zv = Vec3(zr.x + 4.0*a*D.x, zr.y + 4.0*a*D.y, zr.z + 4.0*a*D.z);

    Vec3 result;
    for (int c = 0; c < 3; c++) {
        double sc  = /* sigma_tr for channel c */;
        double zcr = /* zr for channel c */;
        double zcv = /* zv for channel c */;
        double alc = /* alphap for channel c */;

        // 到真实/虚拟光源的欧式距离
        double d_r = std::sqrt(r*r + zcr*zcr);
        double d_v = std::sqrt(r*r + zcv*zcv);

        // 偶极子公式：真实项 - 虚拟项（确保边界 R_d(0) 最大）
        double term_r = zcr * (sc + 1.0/d_r) * std::exp(-sc * d_r) / (d_r * d_r);
        double term_v = zcv * (sc + 1.0/d_v) * std::exp(-sc * d_v) / (d_v * d_v);

        // 最终贡献：α'/(4π) * (term_r + term_v)
        result[c] = (alc / (4.0 * M_PI)) * (term_r + term_v);
    }
    return result;
}
```

**逐步解析**：
1. `z_r = 1/σ_t`：光的散射平均自由程，代表光在被散射之前平均能走多远
2. `d_r = sqrt(r² + z_r²)`：出射点到真实光源的三维距离（x-y 平面距离 r，深度 z_r）
3. `exp(-σ_tr · d_r)`：Beer-Lambert 定律，光在传播 d_r 距离后的衰减
4. `(σ_tr + 1/d_r) / d_r²`：扩散方程格林函数的梯度项，描述辐射通量的距离依赖性
5. 虚拟项 term_v 与真实项符号相同（都是正数），但因为 `d_v > d_r` 所以 term_v < term_r，净效果是**减小靠近表面的贡献**，满足表面边界条件

### 4.2 Monte Carlo SSS 积分

```cpp
Vec3 computeSSS(
    const Scene& scene,
    const Sphere& sphere,    // 被渲染的球体
    const Vec3& xo,          // 观察点（出射点）
    const Vec3& no,          // 出射点法线
    const SSSMaterial& mat,
    int sampleCount
) {
    Vec3 result;
    double r = sphere.radius;
    // 积分权重 = 球面面积（均匀采样的 PDF = 1/Area，蒙特卡洛权重 = Area）
    double area = 4.0 * M_PI * r * r;

    for (int i = 0; i < sampleCount; i++) {
        // Step 1: 在球面均匀采样入射点 xi
        Vec3 localDir = uniformSampleSphere();
        Vec3 xi = sphere.center + localDir * r;
        Vec3 ni = localDir;  // 球面上法线 = 归一化位置向量

        // Step 2: 计算 xi 处的直接光照（入射辐射 Li）
        Vec3 Li;
        for (const auto& light : scene.lights) {
            if (!scene.inShadow(xi + ni * 1e-3, light.pos)) {
                Vec3 ld = (light.pos - xi).norm();
                double ndotl = std::max(0.0, ni.dot(ld));
                double dist2 = (light.pos - xi).length2();
                // 点光源辐照度：I * cosθ / r²
                Li += light.color * (light.intensity * ndotl / dist2);
            }
        }

        // Step 3: 计算 xi 到 xo 的距离
        double dist = (xi - xo).length();

        // Step 4: 漫射剖面权重 R_d(dist)
        Vec3 Rd = mat.Rd(dist);

        // Step 5: 入射点到出射点的方向一致性权重
        // 只有 xi 和 xo 在同侧（法线方向相近）时才有贡献
        Vec3 contribution = Rd * Li * (area / sampleCount);
        result += contribution * std::max(0.0, no.dot(ni) * 0.5 + 0.5);
    }

    // 除以 π 是 Lambert 漫反射的归一化因子
    return result * (1.0 / M_PI);
}
```

**为什么要乘以 `no.dot(ni) * 0.5 + 0.5`？**

这是一个软性的方向一致性权重。如果 xi 在球的正后方（ni 与 no 反向），贡献为 0；如果 xi 在同侧（ni ≈ no），贡献最大为 1。`*0.5 + 0.5` 把范围从 [-1,1] 映射到 [0,1]，避免完全截断而产生硬接缝。

### 4.3 材质参数定义

真实材质参数来自 Jensen 2001 年论文 Table 2（单位 mm⁻¹）：

```cpp
SSSMaterial skin() {
    return {
        {0.032, 0.17,  0.48 },  // sigma_a: 红通道吸收最低（所以皮肤透出红光）
        {0.74,  0.88,  1.01 },  // sigma_sp: 高散射系数（皮肤是强散射介质）
        1.4,                     // eta: 皮肤折射率
        {0.9, 0.7, 0.6}         // 漫反射颜色
    };
}

SSSMaterial marble() {
    return {
        {0.0021, 0.0041, 0.0071},  // sigma_a: 非常小（大理石几乎不吸收）
        {2.19, 2.62, 3.00},        // sigma_sp: 非常大（大理石是强散射介质）
        1.5,
        {0.9, 0.9, 0.85}
    };
}
```

**参数对散射效果的影响分析**：

| 材质 | σ_a | σ_sp | 效果描述 |
|------|-----|------|---------|
| 皮肤 | 中等（蓝>红） | 高 | 红色偏移，柔和散射边缘 |
| 大理石 | 极低 | 极高 | 大范围散射，近乎半透明 |
| 蜡烛 | 低 | 中等 | 暖黄色调，蜡质半透明感 |
| 玉石 | 低（红>绿） | 中等 | 绿色偏移，玉质感 |

### 4.4 ACES 色调映射实现

```cpp
// 渲染结果是 HDR（High Dynamic Range），需要映射到 [0,1] 范围
auto tonemapACES = [](double v) {
    // ACES Filmic 曲线的近似（Hill 2016）
    // 公式：(v*(a*v+b)) / (v*(c*v+d)+e)
    double a = 2.51, b = 0.03, c = 2.43, d = 0.59, e = 0.14;
    return (v*(a*v+b)) / (v*(c*v+d)+e);
};
```

**为什么不用简单的 clamp(v, 0, 1)？**

SSS 积分的结果可能在高光区域超过 1.0（HDR 值）。简单 clamp 会导致高光区域完全饱和、失去细节。ACES 曲线在暗部接近线性，在亮部平滑压缩，保留了高光层次感，同时避免了过爆区域的完全饱和。

---

## 五、踩坑实录

### 坑 1：偶极子公式符号错误导致全黑

**症状**：渲染结果四个球全黑，仅边缘有微弱高光。

**错误假设**：以为虚拟项应该用减法（负号），即 `term_r - term_v`，因为虚拟光源是"负光源"用来满足边界条件。

**真实原因**：在 Jensen 的推导中，虚拟光源与真实光源都位于表面同侧（虚拟源在表面外侧），两项都是**正数**相加。是真实源的 `z_r` 与虚拟源的 `z_v` 的位置关系（`z_v > z_r`）导致了虚拟项贡献较小，而不是通过负号来"抵消"。

**修复方式**：将 `term_r - term_v` 改为 `term_r + term_v`，立即得到了正确的散射晕光效果。

**教训**：实现物理公式时，要仔细分辨"虚拟"的含义。偶极子的"虚拟光源"在电磁学中确实是负号的，但在光子扩散方程框架下，边界条件的满足方式不同，两项均为正。

### 坑 2：SSS 权重与直接光照混合比例失调

**症状**：渲染结果中，皮肤球看起来和普通漫反射没有区别，完全看不出 SSS 效果。

**错误假设**：默认将 SSS 权重设为 0.1，认为大部分光照来自表面反射。

**真实原因**：SSS 的表面积分（64个采样点）覆盖了整个球面，所以总 SSS 贡献量其实非常大，但如果权重 0.1 太小就完全被直接光照淹没了。反之，如果 SSS 权重过高（>0.8），场景会显得过于"发光"，失去高光细节。

**修复方式**：通过多次试验，将 SSS 权重设定为 0.6（直接光照 0.4），这在保留表面高光的同时让 SSS 透射效果明显可见。

**教训**：SSS 权重的调节类似 PBR 中金属度/粗糙度的调节，没有唯一"正确"值，需要根据材质和光照环境调整。实际生产中通常通过艺术家调参+物理测量数据结合的方式确定。

### 坑 3：均匀球面采样方法错误

**症状**：采样点大量聚集在球的两极，赤道附近采样不足，SSS 效果在顶底强、两侧弱，出现明显的垂直条纹。

**错误假设**：用 `θ = randF()*π, φ = randF()*2π` 直接采样球面坐标。

**真实原因**：球坐标系中面积元是 `dA = sinθ · dθ · dφ`，直接均匀采样 `(θ, φ)` 会导致极点附近面积被过采样。在极点附近，小角度变化对应的表面面积很小，但采样数量与赤道相同。

**修复方式**：使用正确的均匀球面采样公式：
```cpp
Vec3 uniformSampleSphere() {
    double u = randF(), v = randF();
    double theta = 2.0 * M_PI * u;
    // cosφ 均匀分布等价于 φ 的正确面积采样
    double phi = std::acos(1.0 - 2.0 * v);
    return {sin(phi)*cos(theta), sin(phi)*sin(theta), cos(phi)};
}
```
关键是用 `cos(φ) = 1 - 2v`（即 `φ = acos(1-2v)`）而不是 `φ = v*π`。

### 坑 4：缺少 -lm 链接导致链接错误

**症状**：`g++ main.cpp -o output -std=c++17 -O2` 链接失败，提示找不到 `sqrt`、`exp` 等数学函数符号。

**原因**：在 Linux 上，`<cmath>` 中的数学函数实现在 libm（数学库）中，需要显式用 `-lm` 链接。macOS 和 Windows 上通常不需要，但 Linux 严格要求。

**修复**：在编译命令末尾加上 `-lm` 即可。

**教训**：Linux 下编写图形学代码，编译命令模板应该是：
```bash
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra -lm
```
这个 `-lm` 要变成肌肉记忆。

---

## 六、效果验证与数据

### 6.1 渲染图分析

**主渲染图（1024×512，4xAA，64 SSS 采样/像素）**：

![四材质渲染图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-24-subsurface-scattering/sss_output.png)

从左到右依次是：皮肤（Skin）、大理石（Marble）、蜡烛（Wax）、玉石（Jade）。

可以观察到明显的材质差异：
- **皮肤**：边缘有明显的红色散射晕（因为 σ_a 红通道最小，红光传播最远）
- **大理石**：整体几乎半透明，散射范围最大（σ_sp 高达 2.19-3.00 mm⁻¹）
- **蜡烛**：暖黄色调，散射范围中等
- **玉石**：偏绿色调，SSS 赋予了玉质感

**皮肤特写（512×512）**：

![皮肤球特写](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-24-subsurface-scattering/sss_skin_closeup.png)

### 6.2 量化验证数据

像素统计验证（验证脚本输出）：

| 图像 | 分辨率 | 均值 | 标准差 | 结论 |
|------|--------|------|--------|------|
| sss_output.png | 1024×512 | 143.2 | 58.5 | ✅ 正常 |
| sss_skin_closeup.png | 512×512 | 143.6 | 59.1 | ✅ 正常 |
| sss_diffusion_profile.png | 800×400 | 70.2 | 17.1 | ✅ 正常 |

验证标准：均值 10-240（非全黑/全白），标准差 > 5（有内容变化）。三张图均通过。

### 6.3 漫射剖面曲线分析

![漫射剖面 R_d(r) 四材质对比](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-24-subsurface-scattering/sss_diffusion_profile.png)

曲线显示：
- **大理石**（蓝紫色）：衰减最慢，散射半径最大，这与大理石 σ_a 极低的参数一致
- **皮肤**（橙红色）：中等衰减速率
- **蜡烛/玉石**：类似的衰减曲线，主要差别体现在颜色通道比例上

### 6.4 性能数据

| 渲染配置 | 分辨率 | 耗时 |
|----------|--------|------|
| AA=4, SSS=64 | 1024×512 | ~95s |
| AA=4, SSS=64（特写）| 512×512 | ~25s |

SSS 每像素需要 4（AA）× 64（SSS 采样）= 256 次球面积分点，每次还需要做光照计算和阴影测试，所以速度较慢。实际引擎中会用预积分纹理（Pre-integrated SSS）或屏幕空间技术来加速。

---

## 七、总结与延伸

### 7.1 技术局限性

1. **表面积分效率低**：当前实现是对整个球面均匀积分，时间复杂度 O(N·M)（N 是像素数，M 是 SSS 采样数）。实际中 `R_d(r)` 随距离快速衰减，大多数采样点的贡献接近 0，是严重的浪费。

2. **仅适用于凸几何体**：偶极子模型假设介质是半无限平面，对球体等凸体近似尚可，但对凹形物体（如耳孔、鼻腔内壁）会失真——真实情况下光不应该穿越空气间隙，但模型会允许。

3. **不支持各向异性散射**：真实皮肤中的散射有一定方向偏好（前向散射），我们用折减散射系数 σ_sp 折算成了等效各向同性，这是一个近似。

4. **单层介质模型**：真实皮肤是多层的（表皮/真皮/脂肪），偶极子模型只处理单层均匀介质。多层 SSS 需要用"多极子"（Multipole）扩展。

### 7.2 可优化的方向

1. **预积分 SSS 纹理（Pre-integrated SSS）**：游戏引擎常用方法——预计算一张 2D 纹理，索引为 `(dot(N,L), 曲率)`，把 SSS 积分结果存在纹理里，渲染时直接查表，性能提升 100x 以上。

2. **屏幕空间 SSS（SSSSS）**：在屏幕空间对像素做高斯模糊（对应不同颜色通道用不同半径），能以 O(W×H) 的开销近似 SSS，是 AAA 游戏的常用方案。

3. **重要性采样**：不均匀采样整个表面，而是用 `R_d(r)` 作为概率密度函数进行重要性采样，将大量采样集中在对结果贡献显著的区域（入射点周围），可以大幅减少所需采样数。

4. **GPU 加速**：SSS 积分天然适合并行，每个像素独立计算，可以用 CUDA/Vulkan Compute Shader 加速。

### 7.3 与本系列的关联

- 前天（03-22）实现的 **BDPT** 从全局光照角度处理多次弹射，与 SSS 的多次内部散射在概念上同源，都是描述光的多次交互
- 昨天（03-23）的 **Voxel Cone Tracing** 使用体素化近似全局光照，与 SSS 同样属于"把光的传输简化为可计算的近似"的思路
- 后续可以尝试把 SSS 与 **延迟渲染（03-18）** 结合，实现 Screen-Space Subsurface Scattering

---

## 参考资料

1. Jensen et al. "A Practical Model for Subsurface Light Transport." SIGGRAPH 2001.
2. d'Eon & Luebke. "Advanced Techniques for Realistic Real-Time Skin Rendering." GPU Gems 3.
3. Jimenez et al. "Practical Real-Time Strategies for Accurate Indirect Occlusion." SIGGRAPH 2014.
4. 本项目代码：[GitHub - daily-coding-practice/2026/03/03-24-subsurface-scattering](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-24-subsurface-scattering)
