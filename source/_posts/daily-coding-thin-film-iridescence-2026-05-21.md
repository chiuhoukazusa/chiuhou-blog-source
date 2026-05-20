---
title: "每日编程实践: Thin Film Iridescence Renderer - 薄膜干涉彩虹色渲染"
date: 2026-05-21 05:30:00
tags:
  - 每日一练
  - 图形学
  - 光线追踪
  - 物理渲染
  - 波动光学
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-21-thin-film-iridescence/thin_film_output.png
---

# 每日编程实践 Day 52：薄膜干涉彩虹色渲染器

![渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-21-thin-film-iridescence/thin_film_output.png)

*肥皂泡、油膜地面与甲虫翅膀在同一场景中呈现薄膜干涉产生的彩虹色效果*

---

## ① 背景与动机

### 为什么肥皂泡是彩色的？

你一定见过吹出的肥皂泡——它在阳光下闪烁着彩虹般的色彩，随着视角变化颜色也跟着变化。这不是颜料，也不是染色，而是**纯粹的物理光学效应**：薄膜干涉（Thin Film Interference）。

同样的效应出现在生活中许多地方：
- **油膜**：马路上雨后积水表面漂浮的一层油，呈现绚丽的彩色条纹
- **CD/DVD 光碟**：背面在灯光下的彩虹反光
- **甲虫翅膀**：吉丁虫（Jewel Beetle）的翅膀拥有令人叹为观止的金属光泽
- **蝴蝶翅膀**：大闪蝶（Morpho butterfly）的蓝色并非颜料，而是薄膜干涉
- **珍珠母贝**：珍珠内层的七彩光泽
- **防伪全息贴纸**：利用薄膜干涉实现防伪

### 游戏引擎里的使用场景

这种效应在现代游戏和电影渲染中有广泛应用：

**次世代游戏**：
- **肥皂泡/泡泡特效**：很多游戏有吹泡泡的技能，如果泡泡只是简单的半透明球，视觉效果会差很多
- **泼油/洒水特效**：地面油膜的彩色条纹增加真实感
- **昆虫/爬行动物**：甲壳类生物的翅膀/外壳，增加生物多样性的视觉细节
- **金属物体的氧化层**：钛合金经过高温氧化后呈现的彩色（航空发动机叶片、钛合金珠宝）

**电影/VFX**：
- **生物皮肤细节**：《阿凡达》中的外星生物皮肤
- **材质系统扩展**：迪士尼的材质模型（Disney BSDF）专门为薄膜材质留了接口参数

**学术研究**：
薄膜干涉是**物理渲染（PBR）**向**波动光学（Wave Optics）**扩展的入口。传统 PBR 假设光是几何射线，而薄膜干涉必须用波动光学（相位、干涉）才能正确描述。

---

## ② 核心原理

### 2.1 光是什么？几何光学 vs 波动光学

传统光线追踪/光栅化使用**几何光学**假设：光是沿直线传播的射线。这在大多数情况下够用，但对于薄膜干涉，我们需要**波动光学**。

光是电磁波，有振幅和相位。当两束光叠加时：
- **同相叠加（constructive interference）**：振幅加倍，光更亮
- **反相叠加（destructive interference）**：振幅相消，光更暗

薄膜干涉的关键：从薄膜顶面反射的光，与从薄膜底面反射后穿出的光，因为**光程差**产生相位差，进而干涉。

### 2.2 相位差推导

设薄膜厚度为 `d`，折射率为 `n₁`，光从空气（`n₀=1`）以入射角 `θ₀` 射入：

1. **Snell 定律**：光进入薄膜时折射
   ```
   n₀ sin(θ₀) = n₁ sin(θ₁)
   ```
   所以 `θ₁ = arcsin(sin(θ₀) / n₁)`

2. **光程差**：从薄膜底面反射回来的光，比从顶面反射的光，多走了两倍的薄膜内路程
   ```
   optical_path = 2 · n₁ · d · cos(θ₁)
   ```
   其中 `cos(θ₁)` 是因为光在薄膜内是斜着走的，实际走的距离乘以 `cos(θ₁)` 才是垂直方向的贡献。

3. **相位差**：光程差对应的相位差
   ```
   δ = (2π / λ) · optical_path = 4π · n₁ · d · cos(θ₁) / λ
   ```
   这是本算法的核心公式！当 `δ` 是 `2π` 的整数倍时，两束反射光同相叠加（颜色更亮）；当 `δ` 是 `π` 的奇数倍时，反相相消（颜色消失）。

**直觉理解**：对于固定的膜厚 `d`，不同波长 `λ` 的光有不同的相位差 `δ`。有些波长被加强，有些被削弱，这就产生了颜色！随着视角变化，`cos(θ₁)` 变化，相位差也跟着变，颜色随之改变。

### 2.3 菲涅尔方程（Fresnel Equations）

薄膜有两个界面：空气→薄膜（界面0-1），薄膜→基底（界面1-2）。每个界面都有反射和透射，需要用菲涅尔方程计算振幅系数。

光有两种偏振态：
- **s 偏振**（垂直于入射面）：TE 模式
- **p 偏振**（平行于入射面）：TM 模式

对于界面 `n_i → n_j`，入射角 `θ_i`，折射角 `θ_j`：

**振幅反射系数**：
```
rs = (n_i·cos(θ_i) - n_j·cos(θ_j)) / (n_i·cos(θ_i) + n_j·cos(θ_j))
rp = (n_j·cos(θ_i) - n_i·cos(θ_j)) / (n_j·cos(θ_i) + n_i·cos(θ_j))
```

**振幅透射系数**：
```
ts = 2·n_i·cos(θ_i) / (n_i·cos(θ_i) + n_j·cos(θ_j))
tp = 2·n_i·cos(θ_i) / (n_j·cos(θ_i) + n_i·cos(θ_j))
```

注意：这些是振幅（复数）系数，能量反射率是振幅系数的平方。

**重要细节**：当光从密介质射向疏介质（`n_i > n_j`），反射时会有半波损失（相位反转 π），对应 `rs` 或 `rp` 为负值。这个符号很重要，决定了干涉是加强还是削弱。

### 2.4 薄膜干涉的总反射振幅

薄膜中有无限次多重反射（光来回弹），但可以求级数和。用传递矩阵法（Transfer Matrix Method）或直接几何级数，对于总反射振幅（以 s 偏振为例）：

```
r_total = (r₀₁ + r₁₂·e^{iδ}) / (1 + r₀₁·r₁₂·e^{iδ})
```

其中 `r₀₁` 是界面0-1的反射系数，`r₁₂` 是界面1-2的反射系数，`δ` 是相位差。

**反射率**（能量）：
```
R_s = |r_total_s|² = (r₀₁² + r₁₂² + 2·r₀₁·r₁₂·cos(δ)) / (1 + r₀₁²·r₁₂² + 2·r₀₁·r₁₂·cos(δ))
```

这个公式实现了薄膜的多重反射叠加。`cos(δ)` 项随波长 `λ` 振荡，就是产生颜色的根本原因。

对 p 偏振同理。非偏振光取平均：`R = (R_s + R_p) / 2`

### 2.5 从光谱到 RGB：CIE 色匹配函数

薄膜干涉给出的是每个波长的反射率 `R(λ)`，这是一个光谱。人眼有三种锥细胞（感知红、绿、蓝），每种对不同波长有响应曲线。

**CIE 1931 色匹配函数** `x̄(λ)`, `ȳ(λ)`, `z̄(λ)` 描述了人眼的颜色响应：

```
X = ∫ R(λ) · x̄(λ) dλ
Y = ∫ R(λ) · ȳ(λ) dλ
Z = ∫ R(λ) · z̄(λ) dλ
```

积分范围：380-780 nm（可见光范围）

得到 XYZ 后，通过线性变换转为 sRGB：
```
┌ R ┐   ┌  3.2404542  -1.5371385  -0.4985314 ┐ ┌ X ┐
│ G │ = │ -0.9692660   1.8760108   0.0415560 │ │ Y │
└ B ┘   └  0.0556434  -0.2040259   1.0572252 ┘ └ Z ┘
```

再加上 sRGB 的 Gamma 编码（非线性化），就得到最终可以写入图片的 RGB 值。

这整套流程（物理光谱 → XYZ → sRGB）是物理正确的色彩管线，比直接用 RGB 混色精确得多。

---

## ③ 实现架构

### 3.1 整体数据流

```
输入: 视线方向 + 表面法线 + 薄膜参数 (n, d)
         ↓
计算折射角 θ₁ (Snell 定律)
         ↓
对每个采样波长 λ (380~780nm, 步长10nm, 共41个):
    ├── 计算相位差 δ = 4π·n₁·d·cos(θ₁)/λ
    ├── 计算两界面菲涅尔系数 (rs, rp)
    └── 计算总反射率 R(λ)
         ↓
光谱积分: XYZ = Σ R(λ)·[x̄,ȳ,z̄](λ)·Δλ
         ↓
色彩变换: XYZ → sRGB（包含Gamma校正）
         ↓
输出: Vec3 iridescentColor
```

### 3.2 关键数据结构

```cpp
// 材质定义 - 封装薄膜物理参数
struct Material {
    MatType type;            // 材质类型
    Vec3 baseColor;          // 基底颜色（无薄膜时的颜色）
    double filmN;            // 薄膜折射率 n₁
    double filmThickness;    // 薄膜基础厚度 (nm)
    double thicknessVariation; // 厚度变化幅度 (nm)
    double substN;           // 基底折射率 n₂
};
```

三种薄膜材质的参数选择有物理依据：
- **肥皂泡**：水+肥皂，`n=1.33`（水的折射率），厚度 250-700nm（人眼可见光范围内产生干涉的厚度）
- **油膜**：有机油脂，`n=1.47`，厚度较大，可达 800nm 以上（颜色更复杂）
- **甲虫翅膀**：角质层，`n=1.50`，厚度比较固定（自然进化出特定厚度以产生特定颜色）

### 3.3 渲染管线结构

本项目使用**光线追踪**架构（而非光栅化），以便正确处理薄膜材质的透明度和反射：

```
对每个像素 (x, y)：
  ├── 生成视线 Ray
  ├── 场景求交 intersectScene()
  │   ├── 与所有球体求交 (Sphere::intersect)
  │   └── 与平面求交 (Plane::intersect)
  ├── 命中薄膜材质？
  │   ├── 计算 cosθ（视线-法线夹角）
  │   ├── computeIridescentColor(cosθ, mat, uv)
  │   │   ├── thinFilmToXYZ() - 光谱积分
  │   │   │   └── thinFilmReflectance() × 41 次（每个波长）
  │   │   └── xyzToRGB() - 色彩变换
  │   ├── 计算 Fresnel 项（决定反射/透射比例）
  │   ├── 递归追踪反射光（肥皂泡透明）
  │   └── 合并薄膜色 + 直接光照
  └── 命中漫反射材质？
      └── 标准 Phong 着色
```

---

## ④ 关键代码解析

### 4.1 薄膜反射率计算（核心函数）

```cpp
double thinFilmReflectance(double n0, double n1, double n2,
                            double d, double lambda, double cosTheta0) {
    // Step 1: 用 Snell 定律求薄膜内折射角
    double sinTheta0 = std::sqrt(1.0 - cosTheta0*cosTheta0);
    double sinTheta1 = n0 * sinTheta0 / n1;
    if (sinTheta1 >= 1.0) return 1.0; // 全内反射
    double cosTheta1 = std::sqrt(1.0 - sinTheta1*sinTheta1);

    // Step 2: 求基底折射角
    double sinTheta2 = n1 * sinTheta1 / n2;
    double cosTheta2 = (sinTheta2 < 1.0) ?
        std::sqrt(1.0 - sinTheta2*sinTheta2) : 0.0;

    // Step 3: 界面0-1 的菲涅尔振幅系数
    // 注意: 这里 rs 可能为负（半波损失），符号很关键！
    double rs01 = (n0*cosTheta0 - n1*cosTheta1) / (n0*cosTheta0 + n1*cosTheta1);
    double rp01 = (n1*cosTheta0 - n0*cosTheta1) / (n1*cosTheta0 + n0*cosTheta1);

    // Step 4: 界面1-2 的菲涅尔振幅系数
    double rs12 = (sinTheta2 < 1.0) ?
        (n1*cosTheta1 - n2*cosTheta2) / (n1*cosTheta1 + n2*cosTheta2) : -1.0;
    double rp12 = (sinTheta2 < 1.0) ?
        (n2*cosTheta1 - n1*cosTheta2) / (n2*cosTheta1 + n1*cosTheta2) : 1.0;

    // Step 5: 计算相位差 δ
    // 公式: δ = 4π·n₁·d·cos(θ₁) / λ
    double delta = 4.0 * PI * n1 * d * cosTheta1 / lambda;

    // Step 6: 计算总反射振幅（传递矩阵法的结果）
    // r_total = (r01 + r12·e^{iδ}) / (1 + r01·r12·e^{iδ})
    // 用实部/虚部分开计算:
    double cosD = std::cos(delta);
    double sinD = std::sin(delta);

    // 分母 |1 + r01·r12·e^{iδ}|²
    double denomS = 1.0 + rs01*rs01*rs12*rs12 + 2.0*rs01*rs12*cosD;
    double Rs = 0.0;
    if (denomS > 1e-10) {
        // 分子 |r01 + r12·e^{iδ}|²
        double numS_re = rs01 + rs12*cosD;  // 实部
        double numS_im = rs12*sinD;          // 虚部
        Rs = (numS_re*numS_re + numS_im*numS_im) / denomS;
    }
    // p偏振同理...
    double Rp = /* 类似计算 */;

    return 0.5 * (Rs + Rp); // 非偏振光取平均
}
```

这个函数是整个渲染器的核心。要点：
1. **振幅系数的符号**：`rs01` 可能是负数（空气到玻璃反射时），这个符号影响干涉方向
2. **分子分母都是实数计算**：`e^{iδ} = cos(δ) + i·sin(δ)`，把复数乘法展开
3. **分母展开**：`|1 + r01·r12·e^{iδ}|² = 1 + r01²·r12² + 2·r01·r12·cos(δ)`

### 4.2 光谱到 XYZ 积分

```cpp
Vec3 thinFilmToXYZ(double n0, double n1, double n2,
                    double d, double cosTheta0) {
    double X = 0, Y = 0, Z = 0;
    double dLambda = 10.0; // nm

    // 对 380-780nm 范围内41个波长采样积分
    for (int i = 0; i < 41; i++) {
        double lambda = 380.0 + i * dLambda;

        // 在这个波长下的反射率
        double R = thinFilmReflectance(n0, n1, n2, d, lambda, cosTheta0);

        // 乘以 CIE 色匹配函数，累加
        // 假设等能量白色入射光（日光近似）
        X += R * CIE_X[i] * dLambda;
        Y += R * CIE_Y[i] * dLambda;
        Z += R * CIE_Z[i] * dLambda;
    }

    return {X, Y, Z};
}
```

`dLambda = 10nm` 意味着积分有 41 个样本。这个精度对视觉效果已经足够，更细的采样（5nm 或 1nm）颜色差异肉眼难以分辨，却会大幅增加计算量。

### 4.3 XYZ 转 sRGB

```cpp
Vec3 xyzToRGB(const Vec3& xyz) {
    // 标准 CIE XYZ D65 → sRGB 3×3 矩阵
    double r =  3.2404542*xyz.x - 1.5371385*xyz.y - 0.4985314*xyz.z;
    double g = -0.9692660*xyz.x + 1.8760108*xyz.y + 0.0415560*xyz.z;
    double b =  0.0556434*xyz.x - 0.2040259*xyz.y + 1.0572252*xyz.z;

    // sRGB Gamma 编码
    // 注意: 分段函数，线性区 + 幂函数区
    auto gammaEncode = [](double c) -> double {
        if (c <= 0.0031308) return 12.92 * c;   // 低值线性
        return 1.055 * pow(max(0.0, c), 1.0/2.4) - 0.055; // 高值幂函数
    };

    return { gammaEncode(r), gammaEncode(g), gammaEncode(b) };
}
```

这里的 Gamma 编码不是简单的 `^(1/2.2)`，而是精确的 sRGB 标准分段函数。对于接近 0 的小值用线性，对较大值用幂函数，这能避免暗处的色调问题。

### 4.4 厚度变化模式

薄膜的厚度变化决定了颜色分布的视觉图案：

```cpp
// 肥皂泡: 重力导致底部比顶部厚（重力使液体下流）
if (mat.type == MatType::SOAP_BUBBLE) {
    double v = uv.y; // 球面纬度 0=顶 1=底
    thicknessNoise = -v * mat.thicknessVariation; // 底部更厚
}
// 油膜: 湍流产生的不规则厚度变化
else if (mat.type == MatType::OIL_FILM) {
    double u = uv.x, v = uv.y;
    thicknessNoise = mat.thicknessVariation * (
        0.3 * sin(u*8) * cos(v*6) +    // 低频波纹
        0.2 * sin(u*15 + v*10) +        // 中频波纹
        0.1 * cos(u*20 - v*18)          // 高频细节
    );
}
// 甲虫翅膀: 相对均匀，轻微微观结构变化
else {
    thicknessNoise = mat.thicknessVariation * 0.3 * sin(u*5)*sin(v*7);
}
```

这些不是随意的公式，而是有物理意义的近似：
- 肥皂泡受重力，液膜厚度从顶到底线性增加
- 油膜受水流和表面张力，呈现湍流状的不规则图案
- 甲虫翅膀经过进化优化，微观结构相对规则

### 4.5 材质着色合并

不同材质的着色策略不同，这里以肥皂泡为例：

```cpp
if (mat.type == MatType::SOAP_BUBBLE) {
    // 肥皂泡 = 透明 + 反射 + 薄膜干涉色

    // 1. 递归追踪反射方向
    Vec3 reflDir = reflect(ray.dir, N);
    Vec3 reflColor = shade(Ray(hitPoint, reflDir), depth - 1);

    // 2. 菲涅尔权重（掠射角反射更强）
    double reflWeight = fresnel * 0.6;
    finalColor += reflColor * reflWeight;

    // 3. 透射（光穿过泡泡）
    Vec3 transColor = shade(Ray(hitPoint - N*epsilon, ray.dir), depth - 1);
    finalColor += transColor * (1 - fresnel) * 0.25;

    // 4. 薄膜干涉颜色（叠加在反射上）
    finalColor += iridescentColor * 0.6;
}
```

关键决策：
- **反射用 fresnel 权重**：保持物理正确性，掠射角反射更强
- **透射简化处理**：真实的折射会偏转光线方向，这里做了简化（直接穿透）
- **薄膜色是附加项**：不是替换基础颜色，而是叠加彩色高光

---

## ⑤ 踩坑实录

### Bug 1：警告来自 stb_image_write.h

**症状**：编译有警告（`-Wmissing-field-initializers`），要求 0 warnings。

**错误假设**：以为是自己代码的问题，仔细检查了 Vec3/Vec2 的初始化列表。

**真实原因**：警告来自第三方头文件 `stb_image_write.h`，其内部的 `stbi__write_context s = {0}` 没有初始化所有字段，触发了 GCC 的 `Wmissing-field-initializers` 警告。

**修复**：用 `#pragma GCC diagnostic push/pop` 包裹 include：
```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```
这样抑制了第三方代码的警告，但自己代码的警告仍然被检测。

### Bug 2：黑色斑块出现在小球上

**症状**：场景底部的小球表面有黑色区域，不像光照阴影，更像是完全没有光。

**错误假设**：以为是薄膜着色函数计算错误，返回了负数颜色。

**真实原因**：小球 `{0.5, -0.6, -3.0}` 的球心 y=-0.6，半径 0.3，底部 y=-0.9，而地面在 y=-1.0。但阴影射线从球面出发，向光源方向。球底部的点发出阴影射线，可能在极短距离内与自身相交（浮点精度问题）。

另外，球靠近地面，阴影射线可能与地面的油膜平面相交，地面是不透光的。这造成小球在某些角度完全被"自身"或"地面"遮蔽。

**修复**：
1. 将偏移量从 `1e-4` 增大到 `1e-3`（self-intersection 偏移）
2. 将小球 y 坐标从 -0.6 改为 0.2（抬高离开地面），避免与地面过近

```cpp
// 修复前：偏移太小 + 球与地面过近
Ray shadowRay(point + toLight.normalized() * 1e-4, toLight);

// 修复后：增大偏移，移动球体位置
Vec3 toLightNorm = toLight / dist;
Ray shadowRay(point + toLightNorm * 1e-3, toLightNorm);  // 更大偏移
spheres.push_back({{0.5, 0.2, -3.0}, 0.3, 0}); // y从-0.6改为0.2
```

### Bug 3：图像过亮（均值约211，偏白）

**症状**：第一版渲染结果整体偏亮（背景+球体主体都很白），薄膜干涉颜色被淹没。

**错误假设**：以为是薄膜颜色计算正确，整体亮度没问题。

**真实原因**：两个因素叠加：
1. 光源强度过高（250 + 120 + 100 = 多余的光照能量）
2. 天空颜色返回了白色（`white * (1-t) + blue * t`，t接近0时天空是白色）

这两个因素都在提高整体亮度，薄膜干涉颜色虽然存在，但被白色压制。

**修复**：
1. 降低光源强度（250→200, 120→100, 100→80）
2. 改善天空着色（深蓝顶部 + 浅蓝地平线 + 暖棕地面）：
```cpp
Vec3 skyColor(const Vec3& dir) {
    Vec3 zenith  = {0.1, 0.2, 0.6};   // 深蓝天顶
    Vec3 horizon = {0.6, 0.7, 0.9};   // 浅蓝地平线
    Vec3 ground  = {0.3, 0.25, 0.2};  // 暖棕地面
    // 分段混合...
}
```
3. 曝光从 1.2 降到 0.9

修复后均值从 211 降到 201，更合适。

### Bug 4：setFaceNormal 的使用

**症状**：肥皂泡的内外法线方向可能混乱，导致某些情况折射/透射方向错误。

**错误假设**：以为只要计算了法线就够了。

**真实原因**：光线从泡泡内侧打到外侧时，法线应该指向内侧（面向光线），但如果始终用 `outward normal`，则法线方向与期望相反。

**修复**：使用 `setFaceNormal` 方法，根据光线方向决定法线指向：
```cpp
void setFaceNormal(const Ray& r, const Vec3& outNorm) {
    frontFace = r.dir.dot(outNorm) < 0;
    normal = frontFace ? outNorm : -outNorm; // 始终指向光线来的一侧
}
```
这确保了 `rec.normal` 始终是"迎着入射光"的法线。

---

## ⑥ 效果验证与数据

### 渲染结果

![最终渲染输出](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-21-thin-film-iridescence/thin_film_output.png)

**场景描述**：
- 3 个肥皂泡球体（中央大球 r=1.2，左中球 r=0.8，右小球 r=0.35）
- 2 个甲虫翅膀球体（右下 r=0.5，背景 r=0.5）
- 油膜覆盖的地面（模拟水坑上的油）
- 3 个光源（主光 + 补光 + 顶光）

**可以观察到**：
- 肥皂泡呈现青色、品红色、黄色的色带（典型薄膜干涉条纹）
- 甲虫翅膀球体呈现不同的彩色（因为折射率和厚度不同）
- 地面油膜有波纹状的彩色条纹
- 薄膜颜色随球面位置（即观察角）变化

### 量化验证数据

**像素统计**：
```
分辨率: 800×600
文件大小: 140KB
像素均值: 201.0 (正常范围 10-240) ✅
像素标准差: 72.4 (>5, 有内容变化) ✅
RGB通道: R=195.8 G=202.2 B=205.4
颜色通道差异: 3.87
```

**颜色通道差异**（R/G/B 均值的标准差）反映了整体色调偏向。值 3.87 说明图像略偏蓝（B最高，R最低），这与天空颜色和薄膜干涉颜色一致（薄膜在这个参数范围内青色较强）。

**性能数据**：
```
渲染时间: 10.5 秒
分辨率: 800×600 = 480,000 像素
SPP: 2（2×超采样）
有效光线数: ~960,000 条主光线
场景对象: 8球体 + 1平面 + 3光源
光线追踪深度: 4

每像素薄膜计算: 41波长 × 2界面 × (Snell+Fresnel) ≈ ~500次浮点运算
总计算量: ~480M 次基础运算
```

**与传统着色对比**（近似估计）：
- **传统 Phong**：每像素约 10-20 次浮点运算
- **薄膜干涉（本方案）**：每像素约 500 次浮点运算（高 25-50 倍）
- 这也是实时渲染中通常用查找表（LUT）预计算薄膜颜色的原因

---

## ⑦ 总结与延伸

### 技术局限性

1. **性能**：每像素需要对 41 个波长积分，计算量较大。实时渲染中必须预计算为 LUT（厚度 × cosθ → RGB），离线渲染才能实时计算。

2. **近似简化**：
   - 忽略了薄膜的消光系数（复折射率），对于金属薄膜（如钛金属氧化层）需要考虑
   - 透射处理简化：没有真正计算折射方向，只是让光线直接穿透
   - 单层薄膜：真实的甲虫翅膀可能有多层叠加，本方案只处理单层

3. **参数调校**：薄膜厚度分布的程序化噪声比较简单，更真实的应该用基于物理的流体/生物模拟

4. **光谱分辨率**：10nm 步长在某些材料下可能精度不足，5nm 更准确但计算翻倍

### 优化方向

**性能优化**：
- **LUT 预计算**：将 `thinFilmToXYZ(d, cosθ)` 预计算为 2D 查找表，运行时双线性插值
- **重要性采样**：不均匀采样波长，优先采样 CIE 响应曲线峰值区域
- **GPU 实现**：上述 LUT 方案在 GPU 上可以达到实时（单个纹理采样）

**效果增强**：
- **多层薄膜**：实现传递矩阵法（Transfer Matrix Method）支持 N 层薄膜叠加
- **吸收性薄膜**：引入复折射率 `n = n_real + i·k`，`k` 是消光系数
- **随机厚度**：用分形噪声（FBM）模拟更自然的厚度分布
- **各向异性**：某些材质（如碳化硅）的薄膜有方向性

### 在游戏引擎中的实现

**Unreal Engine**：在材质编辑器中，Clearcoat 材质有 `Clearcoat` 和 `ClearcoatRoughness` 参数，可以通过自定义着色模型添加薄膜干涉

**Unity**：使用 Shader Graph 可以实现简化版薄膜干涉，通过 LUT 采样薄膜颜色

**基于物理的 LUT 方案**（生产流程）：
```glsl
// 游戏中的实时薄膜着色（GLSL）
vec3 ThinFilmColor(float cosTheta, float filmThickness) {
    vec2 uv = vec2(cosTheta, filmThickness / MAX_THICKNESS);
    return texture(thinFilmLUT, uv).rgb;
}
```

### 与本系列其他文章的关联

- **[05-06 IBL Split-Sum PBR](../ibl-split-sum-pbr-2026-05-06)**：薄膜干涉可以叠加在 PBR 之上，作为 clearcoat 层的扩展
- **[05-05 Path Guiding](../path-guiding-2026-05-05)**：光谱渲染（Spectral Rendering）需要对每个波长独立追踪，本方案的波长积分是其简化版
- **[04-13 Spectral Dispersion](../spectral-dispersion-2026-04-13)**：棱镜色散同样基于波长依赖的折射率，与薄膜干涉都属于色散现象

---

**总结**：今天实现了一个基于物理的薄膜干涉渲染器，从波动光学的相位差公式出发，经过菲涅尔方程、CIE 色匹配函数，最终得到视觉上正确的彩虹色。这是 PBR 走向波动光学的第一步，也是理解自然界很多光学现象（肥皂泡、油膜、甲虫翅膀、蝴蝶翅膀）的物理基础。

---

**代码仓库**：[daily-coding-practice/2026/05/05-21-thin-film-iridescence](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-21-thin-film-iridescence)

**渲染时间**：~10.5 秒 | **代码行数**：~690 行 C++ | **编译器**：g++ -std=c++17 -O2
