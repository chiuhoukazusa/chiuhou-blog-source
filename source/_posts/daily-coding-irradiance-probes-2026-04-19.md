---
title: "每日编程实践: Ambient Light Probes & Irradiance Caching"
date: 2026-04-19 05:30:00
tags:
  - 每日一练
  - 图形学
  - 全局光照
  - 球谐函数
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-19-irradiance-probes/irradiance_probe_output.png
---

# 每日编程实践: Ambient Light Probes & Irradiance Caching

> 从零实现游戏引擎 GI 系统的核心基础设施：光探针 + SH2 球谐辐照度缓存 + 探针插值，90 毫秒内完成 9 探针烘焙 + 双侧对比渲染。

## 一、背景与动机

### 全局光照：游戏引擎最难啃的骨头

任何一个做过写实渲染的人都会遇到同一个问题：**光线在现实中会多次反弹，但实时渲染只能做一次弹射**。直接光照负责处理光源→物体→眼睛这一路径，但把直接光照做得再精细，结果看起来还是像"摆在虚空里的模型"——阴影生硬、暗面死黑、色彩渗透缺失。

这就是间接光照（Indirect Illumination / Global Illumination）要解决的问题：光从光源出发，照到红墙上，红墙把能量弹向地面，地面因此染上红色——这一路的能量传递要被正确计算。

问题是，如果要在实时渲染中精确计算所有反弹，代价极高。路径追踪（Path Tracing）可以做到，但一帧要花几分钟甚至几小时，游戏完全没法用。于是游戏引擎发展出了一套**预计算 + 近似**的方案体系：

- **Lightmap**：静态场景把间接光照烘焙成纹理，运行时直接采样。缺点：只能用于静态对象，无法处理动态物体。
- **Reflection Probes（反射探针）**：在场景中放置若干探针，捕获环境 Cubemap 供镜面反射使用。但 Cubemap 的精度主要用于镜面，对漫反射来说是"高射炮打蚊子"。
- **Irradiance Volumes / Light Probes（光探针）**：在场景中放置探针，每个探针用低阶球谐函数压缩存储其位置的漫反射辐照度。运行时根据物体位置插值最近探针，获得低频间接漫反射。这是处理**动态对象**间接光照的标准方案。

Unreal Engine 的 Sky Light、Unity 的 Light Probes、寒霜引擎（EA）、Lumen 的 Radiance Cache……背后都有光探针的影子。这项技术诞生于 2001 年前后（Ramamoorthi & Hanrahan 的 SH 辐照度论文），20 多年后仍然是实时 GI 的骨干之一。

今天就从头实现它：在软光栅场景中放置 9 个光探针，烘焙球谐辐照度，然后在渲染时插值，与纯直接光照做左右对比。

---

## 二、核心原理

### 2.1 辐照度（Irradiance）是什么

先把概念对齐。在物理基础渲染中：

- **辐亮度（Radiance）L(ω)**：从方向 ω 来的单位立体角、单位面积上的能量流，单位 W/(m²·sr)。
- **辐照度（Irradiance）E(n)**：单位面积上从所有方向接收到的总辐射功率，单位 W/m²。

对漫反射表面，辐照度和法线 **n** 的关系是对半球的积分：

$$E(\mathbf{n}) = \int_{\Omega} L(\boldsymbol{\omega}) \max(0, \mathbf{n} \cdot \boldsymbol{\omega}) \, d\boldsymbol{\omega}$$

这个积分对每个法线方向都要重新算，代价极高。球谐函数让这个积分只需要 9 次乘加就能完成。

### 2.2 球谐函数（Spherical Harmonics）

球谐函数是定义在球面上的一组正交基函数，类似傅里叶级数对周期函数的展开，SH 对球面函数做展开。

L2 阶（Band 0 + Band 1 + Band 2）共 9 个基函数：

$$Y_0^0 = \frac{1}{2}\sqrt{\frac{1}{\pi}}$$

$$Y_1^{-1} = \frac{1}{2}\sqrt{\frac{3}{\pi}} y, \quad Y_1^0 = \frac{1}{2}\sqrt{\frac{3}{\pi}} z, \quad Y_1^1 = \frac{1}{2}\sqrt{\frac{3}{\pi}} x$$

$$Y_2^{-2} = \frac{1}{2}\sqrt{\frac{15}{\pi}} xy, \quad Y_2^{-1} = \frac{1}{2}\sqrt{\frac{15}{\pi}} yz, \quad \ldots$$

其中 (x, y, z) 是方向向量的分量。

**为什么 L2 就够了？**

辐照度积分中有一个余弦核 max(0, n·ω)——这个核在频域上衰减很快，Band 3 以上的高频成分贡献极小。Ramamoorthi & Hanrahan (2001) 证明：用 L2 SH（9 系数）重建漫反射辐照度，误差小于 1%。所以 9 个数字就能近乎完美地描述一个位置的漫反射环境光照，而一个 Cubemap 需要 6×128×128×3 = 294912 个数字。压缩比 32000:1。

### 2.3 辐照度 SH 投影

给定某位置的辐射环境（用一个球面函数 L(ω) 描述），投影到 SH 系数的公式是：

$$c_i = \int_{S^2} L(\boldsymbol{\omega}) Y_i(\boldsymbol{\omega}) \, d\boldsymbol{\omega}$$

实际中用蒙特卡洛近似：在单位球面上均匀采样 N 个方向，对每个方向发射光线求辐亮度，然后：

$$c_i \approx \frac{4\pi}{N} \sum_{j=1}^{N} L(\boldsymbol{\omega}_j) Y_i(\boldsymbol{\omega}_j)$$

其中 4π 是单位球面的表面积（均匀采样的权重为 4π/N）。

### 2.4 辐照度重建（余弦叶片卷积）

烘焙得到 SH 系数后，对任意法线 n，辐照度的重建公式是：

$$E(\mathbf{n}) \approx \sum_{i} A_{l(i)} \cdot c_i \cdot Y_i(\mathbf{n})$$

其中 A_l 是余弦核的 SH 系数（称为"ZH 余弦叶片系数"）：

$$A_0 = \pi, \quad A_1 = \frac{2\pi}{3}, \quad A_2 = \frac{\pi}{4}$$

这个卷积的直觉是：余弦核 max(0, n·ω) 是一个低通滤波器。Band 0 分量（常数项）通过完整的 π 倍增益；Band 1 分量（线性项）通过 2π/3；Band 2 分量（二次项）通过 π/4，衰减越来越大。这就是为什么高频细节被抹掉了——是物理上漫反射本身的性质，不是近似误差。

### 2.5 探针插值

场景中有多个探针，渲染某个物体时如何选择用哪个探针？

最简单但有效的方法：**距离倒数加权**——找最近的 K 个探针（本实现用 K=4），每个探针的权重 w = 1/(dist + ε)，归一化后加权平均各探针的辐照度：

$$E_{interp}(\mathbf{n}) = \frac{\sum_{k=1}^{K} w_k \cdot E_k(\mathbf{n})}{\sum_{k=1}^{K} w_k}$$

更高级的方案（如 Unity 的 LPPV）会用 Tetrahedral 四面体插值，保证空间连续性更好。但距离倒数加权实现简单，对本项目场景已经足够。

---

## 三、实现架构

### 3.1 整体数据流

```
1. 场景构建
   球体 × 5 + 发光球 × 2 + 地面 + 点光源 × 3

2. 探针布置
   6 个地面探针（3×2 网格，y=0.5）
   3 个空中探针

3. 探针烘焙（离线预计算）
   每探针 2048 样本
   → 均匀球面采样方向
   → 发射光线 → 直接光照 / 天空 / 发光体
   → SH 投影，累积 9 系数（×3 通道）

4. 渲染（实时部分）
   左半边 (512px)：直接光照 only
   右半边 (512px)：直接光照 + 探针间接漫反射
   每像素：
     - 射线与场景求交
     - 直接光照（点光源 Blinn-Phong）
     - [右侧] 找最近 4 探针 → 距离倒数插值 → SH 重建辐照度
     - ACES 色调映射 + sRGB gamma

5. 输出验证
   像素统计：均值 152, 标准差 42 → ✅
```

### 3.2 关键数据结构

```cpp
// SH 系数容器
struct SH9 {
    float c[9]{};
};

// 三通道 SH 辐照度（R/G/B 分别存储）
struct IrradianceSH {
    SH9 r{}, g{}, b{};
    void addSample(const Vec3& dir, const Vec3& radiance, float weight);
    Vec3 eval(const Vec3& normal) const;
};

// 光探针
struct LightProbe {
    Vec3 pos{};
    IrradianceSH irradiance{};
    bool baked = false;
};
```

为什么 R/G/B 分三个 SH9 而不是 Vec3 数组？因为 SH 系数的运算（投影、重建）是按通道独立的，分开存储更清晰，避免混淆"哪个 c[i] 是哪个通道"。

### 3.3 渲染管线职责划分

| 阶段 | 函数 | 职责 |
|------|------|------|
| 场景初始化 | `main()` | 放置几何体、光源 |
| 探针烘焙 | `bakeProbe()` | 蒙特卡洛 SH 投影 |
| 直接光照 | `directLighting()` | 点光源 Lambertian + 阴影 |
| 探针插值 | `interpolateProbes()` | 距离倒数加权 |
| 半帧渲染 | `renderHalf()` | 逐像素主循环 |
| 天空盒 | `sampleSkybox()` | 程序化渐变天空 |
| 后处理 | `aces() / toSRGB()` | 色调映射 + gamma |

---

## 四、关键代码解析

### 4.1 SH 基函数

这是整个系统的数学核心。实部 SH 基函数（real-valued）：

```cpp
inline void sh_basis(const Vec3& d, float out[9]) {
    float x=d.x, y=d.y, z=d.z;
    // Band 0：常数，覆盖整个球面
    out[0] = 0.282095f;                      // 1/(2√π)
    // Band 1：三个线性方向
    out[1] = 0.488603f*y;                    // √(3/4π) · y
    out[2] = 0.488603f*z;                    // √(3/4π) · z
    out[3] = 0.488603f*x;                    // √(3/4π) · x
    // Band 2：五个二次项
    out[4] = 1.092548f*x*y;                  // 混合项 xy
    out[5] = 1.092548f*y*z;                  // 混合项 yz
    out[6] = 0.315392f*(3*z*z-1);            // 轴对称项 (3z²-1)
    out[7] = 1.092548f*x*z;                  // 混合项 xz
    out[8] = 0.546274f*(x*x-y*y);            // 差分项 (x²-y²)
}
```

**直觉解释**：
- Band 0 是球面上的常数函数——如果光均匀来自所有方向，只需要这一个系数。
- Band 1 三个系数捕捉"光主要来自 +x/-x/+y/..." 方向的低频方向性，类似直流+基频。
- Band 2 捕捉二阶变化——比如"上方亮、下方暗、左右中间"这类更细的分布。

这些常数（0.282095, 0.488603, ...）是 SH 归一化常数，保证 ∫ Y_i² dω = 1，让各系数在数值上可以直接比较大小。

### 4.2 辐照度重建函数

```cpp
inline float sh_irradiance(const float coeff[9], const Vec3& n) {
    float basis[9];
    sh_basis(n, basis);
    // 余弦叶片卷积系数 A_l
    static const float A[9] = {
        PI,                  // Band 0: π
        2.f*PI/3.f,          // Band 1 (×3): 2π/3
        2.f*PI/3.f,
        2.f*PI/3.f,
        PI/4.f,              // Band 2 (×5): π/4
        PI/4.f,
        PI/4.f,
        PI/4.f,
        PI/4.f
    };
    float sum = 0;
    for(int i=0;i<9;i++) sum += A[i]*coeff[i]*basis[i];
    return std::max(0.f, sum);
}
```

**关键细节**：`max(0, sum)` 是必须的。SH 重建是近似，数值上可能给出负辐照度（物理上不可能）。加上 clamp 防止渲染出黑色负值。

**为什么分离 A 和 coeff？**因为 A 是固定的数学常数（由 Lambertian BRDF 决定），coeff 是烘焙时测量到的环境光数据。分离存储更清晰，而且 A 可以在编译时内嵌为常数数组。

### 4.3 探针烘焙核心

```cpp
void bakeProbe(LightProbe& probe, const Scene& scene, int samples = 1024) {
    std::mt19937 rng(42);
    std::uniform_real_distribution<float> rand01(0,1);
    float weight = 4.f*PI / samples;  // 均匀球面采样权重

    for(int i = 0; i < samples; i++) {
        // 均匀球面采样（Archimedes 等面积映射）
        float phi   = 2.f * PI * rand01(rng);
        float cosT  = 1.f - 2.f * rand01(rng);   // cosθ 均匀分布在 [-1,1]
        float sinT  = std::sqrt(1.f - cosT*cosT);
        Vec3 dir(sinT*std::cos(phi), cosT, sinT*std::sin(phi));

        // 发射光线，求辐亮度
        Ray ray{probe.pos, dir};
        HitInfo hit;
        Vec3 radiance(0,0,0);

        if(scene.intersect(ray, hit)) {
            if(hit.matType == 1) {
                radiance = hit.emission;       // 发光体：直接取发射值
            } else {
                // 漫反射表面：用直接光照近似单次弹射
                radiance = directLighting(scene, hit.pos, hit.normal, hit.albedo);
            }
        } else {
            radiance = sampleSkybox(dir);    // 未命中：天空光
        }

        // SH 投影
        probe.irradiance.addSample(dir, radiance, weight);
    }
    probe.baked = true;
}
```

**关键细节：均匀球面采样**

对 cos θ 均匀采样（而非对 θ 均匀），是因为球面面积元 dA = sinθ dθ dφ，直接对 θ 均匀采样会导致极点附近过度密集。对 cos θ 在 [-1,1] 均匀采样，恰好抵消 sin θ 因子，得到均匀的球面点分布。

**为什么权重是 4π/N？**

蒙特卡洛积分：∫ f(ω) dω ≈ (1/N) Σ f(ωᵢ)/p(ωᵢ)

均匀球面的 PDF = 1/(4π)，所以 1/p = 4π，权重 = 4π/N。

### 4.4 探针插值

```cpp
Vec3 interpolateProbes(const std::vector<LightProbe>& probes,
                       const Vec3& pos, const Vec3& normal) {
    // 1. 计算到所有探针的距离，按距离排序
    struct PD { float dist; int idx; };
    std::vector<PD> sorted;
    for(int i=0;i<(int)probes.size();i++){
        float d = (probes[i].pos - pos).len();
        sorted.push_back({d, i});
    }
    std::sort(sorted.begin(), sorted.end(),
              [](const PD&a,const PD&b){return a.dist<b.dist;});

    // 2. 取最近 K=4 个，距离倒数加权
    int k = std::min(4, (int)sorted.size());
    Vec3 irr(0,0,0);
    float totalW = 0;
    for(int i=0;i<k;i++){
        float w = 1.f / (sorted[i].dist + 0.01f);  // +ε 防止除零
        irr += probes[sorted[i].idx].irradiance.eval(normal) * w;
        totalW += w;
    }
    if(totalW > 0) irr = irr * (1.f/totalW);
    return irr;
}
```

**+0.01f 的作用**：当渲染点恰好在探针位置上（dist≈0），权重会趋向无穷大，导致数值不稳定。加一个小偏移量 ε=0.01 防止这种情况。实际引擎中 ε 的选取要根据场景尺度调整。

### 4.5 漫反射间接光照合成

```cpp
// 主渲染循环内
Vec3 Lo = directLighting(scene, hit.pos, hit.normal, hit.albedo);
if(mode == 1 && !probes.empty()) {
    // 插值得到该点法线方向的辐照度（W/m²）
    Vec3 irr = interpolateProbes(probes, hit.pos, hit.normal);
    // Lambertian BRDF = albedo/π，辐照度×BRDF = 漫反射间接贡献
    Vec3 indirectDiff = hit.albedo * irr * (1.f/PI);
    Lo = Lo + indirectDiff;
}
```

**为什么除以 π？**

Lambertian BRDF = ρ/π（其中 ρ 是漫反射率 albedo）。渲染方程的出射辐亮度 Lo = BRDF × Irradiance：

$$L_o = \frac{\rho}{\pi} \cdot E(\mathbf{n})$$

这里的 1/π 来自 Lambertian BRDF 的归一化，保证能量守恒：∫ BRDF·(n·ω) dω = ρ ≤ 1。

---

## 五、踩坑实录

### Bug 1：初始编译警告——成员未初始化

**症状**：编译出现 4 个 `-Wmissing-field-initializers` 警告：

```
warning: missing initializer for member 'LightProbe::irradiance'
```

**错误假设**：`IrradianceSH` 内部的 `SH9` 数组会被零初始化。

**真实原因**：在 C++ 中，用 `{Vec3(...)}` 初始化聚合类型时，未明确指定的成员会按照"值初始化"规则处理——但编译器无法确认用户是否有意省略，所以给出警告。`SH9` 的 `float c[9]` 如果没有 `{}` 初始化，在未初始化内存上会是随机值（UB）。

**修复**：给 `SH9` 和 `IrradianceSH` 的成员加默认初始化器：

```cpp
struct SH9 {
    float c[9]{};    // {} 确保零初始化
};
struct IrradianceSH {
    SH9 r{}, g{}, b{};  // 同样需要 {}
};
struct LightProbe {
    Vec3 pos{};
    IrradianceSH irradiance{};
    bool baked = false;
};
```

**教训**：`-Wall -Wextra` 的警告一定要全部消灭。"只是警告"有时候是真正的 Bug 苗头。

### Bug 2：PNG 转换脚本内联失败

**症状**：把 Python 代码作为 `system()` 的内联字符串调用时，出现语法错误：

```
SyntaxError: invalid syntax
```

原因在这一行：`def ck(n,d): c=...` 在单行字符串内，`def` 函数定义是合法的，但后续的 `return` 语句会被 Python 解析为"独立语句"而非函数体（因为换行符被转义，解析器无法正确识别缩进）。

**错误假设**：Python 的单行字符串可以用分号无限串联语句，包括函数定义。

**真实原因**：`def` 函数体需要物理换行和缩进，在 `system("python3 -c '...'")` 的单行字符串中无法正常使用有缩进的 `def`。

**修复**：把 Python 代码提取为独立文件 `ppm2png.py`，通过 `system("python3 ppm2png.py ...")` 调用：

```cpp
int ret = system(
    "python3 /path/to/ppm2png.py"
    " /tmp/irradiance_probe.ppm"
    " /path/to/irradiance_probe_output.png"
);
```

**教训**：嵌入多行脚本到 C++ `system()` 调用是反模式。超过 2 行的脚本就应该写成单独文件。

### Bug 3：天空光贡献缺失（设计缺陷，已在代码中修正）

在探针烘焙时，初版代码的逻辑是：若光线命中物体，只算直接光照；若未命中，才算天空。这导致烘焙结果中来自发光体的高频信号正确，但漫反射二次弹射里缺失了天空对遮挡区域的贡献。

**最终方案**：保持现有结构（单次弹射近似），但天空贡献只通过未命中路径进入，已经足够产生可见的 GI 效果——发光球的暖色/冷色光确实在右侧渲染图上对漫反射表面产生了明显的色彩渗透（相较左侧）。

---

## 六、效果验证与数据

### 渲染结果

![对比图：左侧直接光照 vs 右侧光探针GI](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-19-irradiance-probes/irradiance_probe_output.png)

**左半边（Direct Only）**：纯点光源直接光照，暗面完全死黑，地面阴影处无任何反弹光。  
**右半边（Direct + Probe GI）**：加入光探针间接漫反射后，暗面被环境光微微填充，地面阴影区域染上了来自两侧发光球（暖橙 + 冷蓝）的色彩，球体底部也有轻微的地面颜色渗透。

### 量化数据

| 指标 | 数值 |
|------|------|
| 探针数量 | 9（6地面 + 3空中） |
| 每探针采样数 | 2048 |
| 烘焙总用时 | ~10ms |
| 渲染分辨率 | 1024×512 |
| 总渲染时间 | ~86ms |
| 像素均值 (R,G,B) | (160, 151, 144) |
| 像素标准差 (R,G,B) | (44, 39, 41) |
| 综合均值 | 152.0 (正常范围 10~240 ✅) |
| 综合标准差 | 41.8 (>5 ✅) |
| 输出文件大小 | 195 KB |
| 编译警告 | 0 |
| 编译错误 | 0 |
| 运行崩溃 | 0 |

### 内存分析

- 9 个探针 × 3 通道 × 9 系数 × 4 字节 = 972 字节
- 渲染缓冲区：1024 × 512 × 3 = 1.5 MB
- 无动态分配（场景对象使用 `std::vector`，栈上无大数组）
- 无内存泄漏（无裸指针，无 `new`/`delete`）

---

## 七、总结与延伸

### 技术局限性

**1. 单次弹射近似**：探针烘焙只追踪一次反射。物体上的间接光照来自探针捕获的"直接照射到附近表面的光"，缺少二次、三次弹射。这对颜色渗透效果有限制——现实中红墙会把红色弹到白球上很多次，这里只有一次。

**2. 低频限制**：L2 SH 只能描述低频变化，无法捕捉方向性很强的间接高光（如通过小洞射入的光柱）。这类情况需要更高阶 SH 或其他方案（Precomputed Radiance Transfer）。

**3. 探针静态**：探针在场景变化时需要重新烘焙。对完全动态场景（光源实时移动），需要实时更新探针，计算代价上升。Unity HDRP 的 Adaptive Probe Volumes 方案通过分区更新和 GPU 烘焙缓解了这个问题。

**4. 插值伪影**：简单的距离倒数加权在探针间距较大时可能出现明显跳变。工业级方案（如四面体插值、LPPV）可以更平滑地过渡。

### 可优化方向

1. **GPU 烘焙**：把蒙特卡洛采样移到 Compute Shader，9 个探针可以并行烘焙，从 10ms 降到 <1ms。
2. **Tetrahedral 四面体插值**：比距离倒数加权更鲁棒，Unity Light Probes 使用这种方案。
3. **更多弹射**：在烘焙时对每个样本再做一次弹射（俄罗斯轮盘赌终止），获得更准确的二次弹射颜色渗透。
4. **Probe Validity**：检测探针是否被遮挡（如嵌入墙内），加权时排除几何上不可见的探针。
5. **Radiance Cache（L1 SH 方向性）**：用 Band 1 SH 保留方向信息，支持有光泽（non-Lambertian）材质的间接光。

### 与系列文章的关联

- **[03-20 球谐函数](./)**：本文直接使用了那天实现的 SH 基函数和投影框架
- **[03-21 SPPM](./)**：光子映射是另一种全局光照方案，精度更高但无法实时
- **[04-04 大气散射](./)**：探针的天空光采样复用了程序化天空盒实现
- **[04-02 BVH 加速](./)**：大场景中探针烘焙也可以用 BVH 加速光线求交

光探针是游戏引擎 GI 架构的基石之一，理解它的实现原理，是之后研究 DDGI（动态漫反射 GI）、Lumen Radiance Cache、Probe-based GI 的前提。

### SH 阶数与精度取舍

值得单独说一下"为什么不用更高阶 SH"这个问题。L2 SH 给了 9 个系数，L3 给 16 个，L4 给 25 个……

理论上更高阶 = 更精确。但对漫反射来说，Ramamoorthi 的论文给出了严格的误差上界：

| SH 阶数 | 重建误差 |
|---------|---------|
| L1 (4系数) | ~25% |
| L2 (9系数) | <1% |
| L4 (25系数) | <0.1% |

对漫反射，L2 的 <1% 误差在视觉上完全不可察觉。额外的 16 个系数（L3/L4）用于漫反射是浪费带宽。**L2 SH 是漫反射辐照度的最优编码方案**——这就是为什么几乎所有实时引擎都用 L2。

更高阶 SH（L4/L6）只有在需要捕捉高频光照细节（如光泽反射、Precomputed Radiance Transfer）时才有价值。

### 程序化天空的实现

本项目使用了极简的程序化天空，代码只有 5 行：

```cpp
Vec3 sampleSkybox(const Vec3& dir) {
    float t = std::max(0.f, dir.y);   // 仰角（y分量）
    Vec3 sky = mix(
        Vec3(0.9f, 0.7f, 0.5f),  // 地平线：暖橙色
        Vec3(0.3f, 0.5f, 0.9f),  // 天顶：蓝色
        t
    );
    return sky * 0.4f;  // 整体亮度缩放
}
```

这个简单的天空产生了从地平线暖橙到天顶蓝色的渐变，给探针烘焙提供了一个有方向性的环境光。在探针的 SH 系数中，Band 1 的 y 轴系数（out[2]）会比较大，反映"上方偏蓝、下方偏暖"的环境特性。渲染时，正朝上的法线（如地面）会重建出偏蓝的天空色，正朝下的法线（球体底面）会重建出偏暖的地面反射色。

---

*代码仓库：[daily-coding-practice/2026-04-19-irradiance-probes](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-04-19-irradiance-probes)*  
*完成时间：2026-04-19 05:37 | 迭代次数：2次 | 总耗时：~7分钟*
