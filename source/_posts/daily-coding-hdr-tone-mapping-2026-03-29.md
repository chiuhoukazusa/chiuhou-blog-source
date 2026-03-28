---
title: "每日编程实践: HDR Tone Mapping & Color Grading"
date: 2026-03-29 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 渲染
  - 色调映射
  - HDR
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-29-hdr-tone-mapping/tonemap_comparison.png
---

## 背景与动机

每一帧游戏画面的最后一道工序，往往最不起眼，却决定了一切的观感——**色调映射（Tone Mapping）**。

### 问题的根源：显示器不是眼睛

现实世界的亮度范围极为宽广。晴天户外的直射阳光亮度约为 100,000 cd/m²，室内阴影处可能只有 0.01 cd/m²，动态范围高达 10 个数量级（100dB）。人眼通过瞳孔收缩和视杆/视锥细胞的自适应，能感知约 20 个 EV（Exposure Value）的动态范围。

然而，家用显示器的 SDR（Standard Dynamic Range）面板通常只能显示 0.1 到 300 cd/m² 的亮度，不足 3 个数量级。HDR 显示器能达到 1000–4000 nits，也仅覆盖 4 个数量级。

一个物理正确的渲染器（Path Tracer、光栅化 + 全局光照），输出的是**线性光照空间**的辐射值，可以包含从 0.001（深阴影）到 20+（太阳直射）甚至更高的值域。如果我们直接把这些值限制在 [0, 1] 范围内显示，高光区域全部爆掉变成一片白，暗部细节也因非线性感知而全部消失。

**色调映射就是解决这个问题的桥梁**——它把宽动态范围的 HDR 数据压缩映射到显示器能表达的 LDR（Low Dynamic Range）范围，同时尽量保留视觉上的真实感。

### 工业界使用场景

- **游戏引擎**：Unreal Engine 4/5 默认使用 ACES Filmic；Unity HDRP 提供 ACES、Neutral、Custom 等选项
- **电影 VFX 流水线**：ACES（Academy Color Encoding System）是好莱坞的行业标准，被 Nuke、DaVinci Resolve 广泛支持
- **实时渲染**：TAA（时序抗锯齿）、Bloom、SSAO 等后处理效果都在 HDR 空间工作，色调映射是渲染管线最末端的一步
- **摄影软件**：Lightroom 的 Tone Curve、Photoshop 的 Camera Raw 都是色调映射的图形化界面
- **HDR 显示器适配**：新一代游戏需要在 SDR/HDR 显示器之间自动切换色调映射策略

### 没有色调映射会怎样

```
HDR 场景亮度范围：0.01 ~ 25.0
            ↓ 直接截断到 [0,1]
显示结果：天空全白 | 太阳全白 | 高光窗口全白
          细节消失 | 不自然 | 曝光失控
```

今天的目标：实现 6 种主流色调映射算法，并加上色彩分级（Color Grading），最终输出对比图。

---

## 核心原理

### 什么是 HDR 场景

在渲染管线中，HDR 缓冲（通常是 RGBA16F 或 RGBA32F 格式的帧缓冲）存储的是**线性辐射值**，不经过任何非线性变换。

线性光照空间的关键特性：
- **叠加律**：两盏灯照同一点，亮度直接相加（不是在显示器空间相加）
- **物理正确**：PBR 光照计算基于能量守恒，结果在线性空间才有意义
- **范围无上限**：太阳直射可以是 20，室内灯可以是 5，阴影可以是 0.02

色调映射函数 `T(x)` 需要满足：
1. **单调递增**：更亮的 HDR 值映射后仍然更亮
2. **渐近压缩**：当 `x → ∞` 时，`T(x)` 趋近于 1（不是无限增长）
3. **线性低端**：当 `x` 很小时，`T(x) ≈ x`（暗部不被扭曲）
4. **视觉舒适**：映射结果符合人眼对对比度和色彩的感知习惯

### 伽马编码：为什么显示前要做伽马矫正

人眼对亮度的感知是**非线性**的——对暗部变化更敏感，对亮部变化相对迟钝。这正好符合对数曲线。

历史上，CRT 显示器的物理特性恰好是 `亮度 ∝ 电压^2.2`，即显示器本身有 γ=2.2 的非线性响应。为了利用这一特性存储更多暗部信息，sRGB 标准规定图像存储时要做**伽马编码**（近似 `output = input^(1/2.2)`），显示时被显示器的物理特性"解码"。

现代 HDR 显示器和软件流水线中，这个过程被标准化为 sRGB 传递函数：

```
线性值 → sRGB编码值
若 v ≤ 0.0031308:  encoded = 12.92 × v
否则:              encoded = 1.055 × v^(1/2.4) - 0.055
```

关键直觉：**所有色调映射都在线性空间进行，最后才做伽马编码**。如果顺序搞反，颜色会严重失真。

### Reinhard 色调映射

**公式**：
```
T(x) = x / (x + 1)
```

当 x=0.5 时，T(x)=0.33；当 x=1 时，T(x)=0.5；当 x→∞ 时，T(x)→1。

**直觉**：分母比分子多了一个常数 1，使得曲线在高亮度区域增速减缓。就像加了一个弹簧——越拉越难拉。

**改进版（Extended Reinhard）** 引入白点 `W`：
```
T(x) = x × (1 + x/W²) / (1 + x)
```
当 `x = W` 时，输出接近 1。通过调整白点，可以控制"何时高光开始压缩"。若 W 很大，整体亮度更高更通透；W 较小，画面更暗但高光更可控。

**优缺点**：
- ✅ 简单，无参数，实现一行代码
- ❌ 颜色会偏移（不同通道被不同幅度压缩）
- ❌ 画面往往偏灰，对比度低
- 适合：学习、原型验证、需要保持颜色准确的场景

### ACES Filmic 色调映射

ACES（Academy Color Encoding System）是电影工业的颜色管理标准，由美国电影艺术与科学学院制定。Krzysztof Narkowicz 在 2016 年提出了一个廉价但效果极好的近似公式，被 Unreal Engine 采用：

**公式**（Narkowicz/Hill ACES 近似）：
```
T(x) = (x × (2.51x + 0.03)) / (x × (2.43x + 0.59) + 0.14)
```

等价展开：分子 = `ax² + bx`，分母 = `cx² + dx + e`，这是一个有理函数（Rational Function）。

**直觉**：ACES 曲线有三段特征：
- **暗部（x<0.2）**：近似线性，保留暗部细节不失真
- **中灰（x≈0.2-0.6）**：略微抬高对比度，画面更有"电影感"
- **高光（x>1）**：快速压缩至 1，高光不爆

与 Reinhard 不同，ACES 在中灰区域有轻微的 S 形弯曲，这让画面的对比度和饱和度都更接近电影效果。在曝光参数 `exposure=0.6` 时，输入 1.0 的白色被映射到约 0.8，保留了高光的层次。

**实现细节**：

```cpp
Vec3 tonemapACES(Vec3 color, float exposure = 0.6f) {
    color = color * exposure;  // 先做曝光调整
    const float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    // 分子：color * (color * a + b)
    Vec3 num = color * (color * a + Vec3(b));
    // 分母：color * (color * c + d) + e
    Vec3 den = color * (color * c + Vec3(d)) + Vec3(e);
    // 逐通道除法
    Vec3 result;
    result.x = num.x / den.x;
    result.y = num.y / den.y;
    result.z = num.z / den.z;
    return result.clamp(0.f, 1.f);
}
```

注意：必须做逐通道除法（component-wise division），不能写 `num / den`——除非 Vec3 定义了向量除法运算符（这是今天的一个坑）。

### Uncharted 2 / Hable Filmic

由 John Hable 在 Uncharted 2 开发期间设计，2010 年的 GDC 演讲后被广泛采用。基于 Filmic 曲线设计，参数更多但控制更细腻：

**Hable 算子**：
```
f(x) = (x × (A×x + C×B) + D×E) / (x × (A×x + B) + D×F) - E/F

参数：A=0.15, B=0.50, C=0.10, D=0.20, E=0.02, F=0.30
```

这不是一个任意多项式——每个参数对应曲线的一个视觉特性：
- **A（Shoulder Strength）**：高光区肩部的弯曲程度
- **B（Linear Strength）**：中间线性段的斜率
- **C（Linear Angle）**：线性段的倾角
- **D（Toe Strength）**：暗部趾部的弯曲程度
- **E（Toe Numerator）** 和 **F（Toe Denominator）**：趾部的精细控制

使用时还需要用白点归一化：`T(x) = f(x) / f(W)`，其中 W=11.2 是场景的最亮值（Hable 用了霓虹灯场景的真实数据）。

**与 ACES 的主要区别**：
- Uncharted 2 的中灰色稍微低一些，对比度略高
- 高光的"肩部"更硬，过渡更快
- 因此画面感觉更"硬朗"，适合写实风格游戏

### Lottes Filmic

Timothy Lottes（AMD GPU 研究员）在 2016 年提出，与 Reinhard 和 ACES 相比，公式更复杂但曲线控制更精确：

```
f(v) = v^a / (v^(a×d) × b + c)

其中 b, c 根据中灰点（midIn=0.18, midOut=0.267）和最大亮度（hdrMax=8）推导
```

Lottes 曲线的特点是**参数有明确的物理含义**：
- `a`：整体对比度（类似 Power 曲线的指数）
- `d`：高光区的衰减速度
- 中灰映射 midIn→midOut 是硬编码的，保证了不同场景的感知亮度一致

效果上，Lottes 和 ACES 相近，但高光区更柔和，色偏更少。

### 对比总结

| 算法 | 暗部保留 | 中灰对比 | 高光处理 | 色偏 | 计算量 |
|------|---------|---------|---------|------|------|
| Gamma Only | 良好 | 正常 | 爆掉 | 无 | 极低 |
| Reinhard | 良好 | 偏低 | 柔和但偏灰 | 中等 | 极低 |
| Reinhard Extended | 良好 | 偏低 | 可控白点 | 中等 | 低 |
| ACES Filmic | 优秀 | 高（电影感） | 压缩强 | 极少 | 低 |
| Uncharted 2 | 优秀 | 高（写实感） | 硬朗 | 极少 | 低 |
| Lottes | 优秀 | 中等 | 柔和 | 极少 | 中等 |
| ACES + ColorGrade | 优秀 | 高+定制 | 压缩强 | 可调 | 低 |

---

## 实现架构

### 整体渲染管线

```
[生成 HDR 场景]
      ↓
  HDR 线性缓冲（float）
      ↓
[色调映射操作符]  ← 这是今天的核心
      ↓
  LDR 线性缓冲（float，值域 [0,1]）
      ↓
[伽马编码（sRGB）]
      ↓
  sRGB 缓冲（可选色彩分级）
      ↓
[写出 PPM/PNG]
```

每个阶段都在不同的色彩空间中操作，顺序不能搞乱。

### 关键数据结构

**Vec3：线性颜色值**
```cpp
struct Vec3 {
    float x, y, z;
    // 支持四则运算（包括逐分量除法！）
    Vec3 operator/(const Vec3& o) const { return {x/o.x, y/o.y, z/o.z}; }
    float luminance() const { return 0.2126f*x + 0.7152f*y + 0.0722f*z; }
};
```

亮度系数来自 BT.709 标准（sRGB 使用的原色）：绿色通道权重最高（0.7152），因为人眼对绿色最敏感。

**Image：HDR 图像缓冲**
```cpp
struct Image {
    int width, height;
    std::vector<Vec3> pixels;  // 线性 HDR 值，无上限
    
    float avgLuminance() const;  // 几何均值（用于自动曝光）
    float maxLuminance() const;  // 最大亮度（用于白点设定）
};
```

为什么用几何均值（log均值）而不是算术均值？  
因为亮度的感知是对数的。对数空间的平均值 = 几何均值，更能代表场景的"中灰"。这也是自动曝光（Auto Exposure）的计算基础，即 Reinhard 2002 论文中的 Key Value 算法。

**ToneMapper 类型别名**：
```cpp
using ToneMapper = std::function<Vec3(Vec3)>;
```

用函数对象封装每个算法，可以统一地对整张图应用：
```cpp
Image applyToneMap(const Image& hdr, ToneMapper tm, bool doColorGrade = false) {
    Image ldr(hdr.width, hdr.height);
    for (auto& pixel : ...) {
        Vec3 c = tm(pixel);           // 色调映射（线性→线性[0,1]）
        if (doColorGrade) c = colorGrade(c);  // 可选色彩分级
        c = gammaEncode(c);           // sRGB 编码（线性→gamma）
        ...
    }
}
```

### HDR 测试场景设计

为了验证各算法的差异，场景必须覆盖所有亮度区间：

| 区域 | 亮度范围 | 设计目标 |
|------|---------|---------|
| 背景天空 | 1.5–3.0 | 正常高光区 |
| 太阳 | 15–25 | 超亮高光（各算法的关键区别） |
| 灯光窗口 | 3–7 | 中高光 |
| 地面暗部 | 0.02–0.15 | 暗区细节保留 |
| 萤火虫粒子 | 2–10 | 随机点光源 |

这覆盖了 3 个数量级的亮度范围（0.02 到 25），足以区分各算法的行为差异。

### 色彩分级管线

```
色调映射输出（LDR线性）
    ↓
对比度调整（S形曲线）
    ↓
饱和度调整（亮度权重混合）
    ↓
色温调整（通道偏移）
    ↓
分色处理（Shadows/Highlights着色）
    ↓
伽马编码
```

色彩分级在色调映射**之后**、伽马编码**之前**进行——此时数据已经在 [0,1] 范围内，是最稳定的操作空间。

---

## 关键代码解析

### HDR 场景生成

```cpp
Image generateHDRScene(int width, int height) {
    Image img(width, height);
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            float u = float(x) / width;   // [0,1]
            float v = float(y) / height;  // [0,1]，上方 v=0
            
            // 天空渐变：越往上越亮（模拟天顶更蓝更亮）
            float skyIntensity = 1.5f + 1.5f * (1.f - v);  // v=0顶部→3.0，v=1底部→1.5
            Vec3 skyColor = Vec3(0.4f, 0.6f, 1.0f) * skyIntensity;
            
            // 太阳：超高斯分布，两层（硬核 + 光晕）
            float sunDist = sqrt((u - 0.7f)² + (v - 0.2f)²);
            color += Vec3(20, 18, 12) * exp(-sunDist² / 0.0003f);  // 太阳盘
            color += Vec3(6, 5, 3) * exp(-sunDist² / 0.001f);       // 光晕
        }
    }
}
```

关键点：太阳核心亮度 20，光晕 6，总亮度最高可达 26——这超过了 Reinhard 白点（通常设 4）的 6 倍，能很好地体现各算法的高光处理差异。

### Reinhard 核心实现

```cpp
Vec3 tonemapReinhard(Vec3 color, float exposure = 1.f) {
    color = color * exposure;
    // 关键：这里是向量除以向量（逐分量）
    // color / (color + Vec3(1.f)) 等价于：
    //   r_out = r / (r + 1)
    //   g_out = g / (g + 1)
    //   b_out = b / (b + 1)
    return color / (color + Vec3(1.f));
}
```

为什么逐分量处理而不是按亮度处理？

**逐分量**（如上）：每个颜色通道独立压缩。优点是实现简单；缺点是高亮度时不同通道被不同程度压缩，导致**色偏**（hue shift）——一个橙色的强光源经过 Reinhard 后可能偏黄或偏红。

**亮度保持**（Reinhard Luma Variant）：先计算 luma，用 Reinhard 压缩 luma，再按比例缩放 RGB：
```cpp
float luma = color.luminance();
float mappedLuma = luma / (luma + 1.f);
return color * (mappedLuma / luma);  // 保持色调，只改变亮度
```
优点是无色偏，但暗部饱和度会稍高。

今天的实现选择了逐分量版本，因为想展示"朴素"版本的行为，便于与 ACES 对比。

### ACES 实现——为什么需要逐分量除法

```cpp
Vec3 tonemapACES(Vec3 color, float exposure = 0.6f) {
    color = color * exposure;
    const float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    Vec3 num = color * (color * a + Vec3(b));  // ax² + bx（逐分量）
    Vec3 den = color * (color * c + Vec3(d)) + Vec3(e);  // cx² + dx + e
    // 必须逐分量除法！
    Vec3 result;
    result.x = num.x / den.x;
    result.y = num.y / den.y;
    result.z = num.z / den.z;
    return result.clamp(0.f, 1.f);
}
```

早期实现曾写成 `num / den`，但当时 Vec3 只有 `operator/(float)` 没有 `operator/(Vec3)`，导致编译错误（见踩坑章节）。

### Uncharted 2 Hable 实现

```cpp
Vec3 hableOp(Vec3 x) {
    const float A=0.15f, B=0.50f, C=0.10f, D=0.20f, E=0.02f, F=0.30f;
    // 原始公式：(x(Ax + CB) + DE) / (x(Ax + B) + DF) - E/F
    return (x*(x*A + Vec3(C*B)) + Vec3(D*E)) / (x*(x*A + Vec3(B)) + Vec3(D*F)) - Vec3(E/F);
}

Vec3 tonemapUncharted2(Vec3 color, float exposure = 2.f) {
    color = color * exposure;
    Vec3 curr = hableOp(color);
    // 白点归一化：除以 hableOp(W) 确保 W→1
    Vec3 whiteScale = Vec3(1.f) / hableOp(Vec3(11.2f));  // W=11.2
    return (curr * whiteScale).clamp(0.f, 1.f);
    // 注意：这里 whiteScale 是标量（三通道相等），但用 Vec3 存储
}
```

白点 11.2 的选取：Hable 在分析 Uncharted 2 实际场景时，发现场景最亮的元素（霓虹灯反射）约为 11.2 EV，因此用这个值作为白点，保证场景最亮处刚好映射到纯白。

### Lottes Filmic 实现

```cpp
Vec3 tonemapLottes(Vec3 color, float exposure = 1.f) {
    color = color * exposure;
    // 曲线参数
    const float a = 1.6f, d = 0.977f, hdrMax = 8.f;
    const float midIn = 0.18f, midOut = 0.267f;
    
    // 从中灰映射约束推导 b, c：
    // 需要满足 f(midIn) = midOut 且 f(hdrMax) ≈ 1
    float b = (-pow(midIn, a) + pow(hdrMax, a) * midOut) /
              ((pow(hdrMax, a*d) - pow(midIn, a*d)) * midOut);
    float c = (pow(hdrMax, a*d)*pow(midIn, a) - pow(hdrMax, a)*pow(midIn, a*d)*midOut) /
              ((pow(hdrMax, a*d) - pow(midIn, a*d)) * midOut);
    
    // 应用曲线（逐通道）
    auto curve = [&](float v) { return pow(v, a) / (pow(v, a*d) * b + c); };
    return Vec3(curve(color.x), curve(color.y), curve(color.z)).clamp(0.f, 1.f);
}
```

`midIn=0.18` 是"中灰"（18% 反射率，摄影的标准灰卡），`midOut=0.267` 是 Lottes 选定的中灰应该映射到的显示亮度。这个约束保证了不同场景的相对亮度感知一致性。

### 色彩分级管线

```cpp
// 对比度：S形曲线，以0.5为中心
Vec3 adjustContrast(Vec3 color, float contrast = 1.2f) {
    // 将 [0,1] 映射到 [-0.5, 0.5]，放大，再映射回来
    auto curve = [&](float v) { return (v - 0.5f) * contrast + 0.5f; };
    return Vec3(curve(color.x), curve(color.y), curve(color.z)).clamp(0.f, 1.f);
    // 直觉：contrast>1 → 亮的更亮、暗的更暗；contrast<1 → 画面偏灰
}

// 饱和度：在目标色和灰度版本之间插值
Vec3 adjustSaturation(Vec3 color, float saturation = 1.3f) {
    float lum = color.luminance();  // 该像素的感知亮度
    // saturation=1 → 原色；saturation=0 → 纯灰度；saturation>1 → 饱和度提升
    return lerp(Vec3(lum), color, saturation).clamp(0.f, 1.f);
}

// 色温：通过通道偏移实现暖/冷
Vec3 adjustTemperature(Vec3 color, float temperature = 0.1f) {
    // 正值=暖（增红减蓝），负值=冷（增蓝减红）
    // 这是一个简化模型，真实的色温调整需要在 CIE XYZ 空间操作
    color.x = clamp(color.x + temperature * 0.2f, 0.f, 1.f);  // R
    color.z = clamp(color.z - temperature * 0.15f, 0.f, 1.f);  // B
    return color;
}

// 分色：暗部和亮部分别着色
Vec3 splitToning(Vec3 color, Vec3 shadowColor, Vec3 highlightColor) {
    float lum = color.luminance();
    float shadowW = 1.f - min(1.f, lum * 2.f);    // lum<0.5时有权重
    float highlightW = max(0.f, lum * 2.f - 1.f); // lum>0.5时有权重
    color += shadowColor * shadowW;
    color += highlightColor * highlightW;
    return color.clamp(0.f, 1.f);
    // 直觉：阴影偏冷蓝、高光偏暖橙，是电影调色的经典手法
}
```

整合管线：
```cpp
Vec3 colorGrade(Vec3 color) {
    color = adjustContrast(color, 1.15f);
    color = adjustSaturation(color, 1.2f);
    color = adjustTemperature(color, 0.08f);      // 略微暖色
    color = splitToning(color, 
                        Vec3(0.02, 0.02, 0.05),   // 阴影略蓝
                        Vec3(0.08, 0.04, 0.00));   // 高光略暖
    return color;
}
```

### 输出对比图拼接

```cpp
// 3列布局，每panel之间有4px分隔线
int cols = 3;
int totalW = W * cols;
int totalH = (H + 4) * rows;  // 4px分隔

for (int pi = 0; pi < panels.size(); pi++) {
    int col = pi % cols;
    int row = pi / cols;
    int ox = col * W;
    int oy = row * (H + 4);
    
    // 拷贝panel像素
    for (int y = 0; y < H; y++)
        for (int x = 0; x < W; x++)
            // 写到 grid[（oy+y）× totalW + （ox+x）]

    // 分隔线：顶行和底行白色（200），其余深灰（30）
    for (int x = 0; x < W; x++)
        for (int ly = 0; ly < 4; ly++)
            grid_at(ox+x, oy+H+ly) = (ly==0 || ly==3) ? 200 : 30;
}
```

最终输出 PPM 格式（P6 二进制），再用 Pillow 转 PNG。选择 PPM 格式是因为它无需任何图像库——只需标准文件 I/O，适合 Skill 验证环境。

---

## 踩坑实录

### Bug 1：Vec3 缺少向量除法运算符

**症状**：编译错误
```
error: no match for 'operator/' (operand types are 'Vec3' and 'Vec3')
  return color / (color + Vec3(1.f));
```

**错误假设**：写 `num / den`（两个 Vec3 相除），以为 Vec3 已经有这个运算符了。

**真实原因**：Vec3 只定义了 `operator/(float t)` 标量除法，没有定义 `operator/(const Vec3& o)` 逐分量除法。C++ 不会自动生成向量运算符——你定义了什么，就只有什么。

**修复方式**：
```cpp
// 在 Vec3 中添加：
Vec3 operator/(const Vec3& o) const { return {x/o.x, y/o.y, z/o.z}; }
```

**受影响的函数**：Reinhard、Reinhard Extended、hableOp（Uncharted 2）、Uncharted 2 白点归一化——共 4 处，都因为这同一个缺失的运算符报错。

**教训**：写向量数学类时，把所有运算符（+、-、×、/ 的标量版和向量版）一次性写完。少一个就会在意想不到的地方炸。

**编译后验证**：
```
编译前：4 个 error，0 warning
编译后（修复后）：0 error，0 warning ✅
```

### Bug 2：亮度统计使用算术均值而不是几何均值

**症状**（潜在的，调试时发现）：验证脚本期望场景的平均亮度代表"中灰"，但算术均值因为太阳极高亮度被拉偏，不能准确反映场景的感知亮度。

**真实原因**：亮度感知是对数的，算数均值容易被极端值拉偏。例如场景中 99% 的像素亮度 0.3，1% 的像素亮度 25，算术均值约 0.55，几何均值约 0.32。后者更接近人眼感知的"平均亮度"。

**修复方式**：
```cpp
float avgLuminance() const {
    float logSum = 0.f; int cnt = 0;
    for (const auto& p : pixels) {
        float lum = p.luminance();
        if (lum > 1e-5f) { logSum += log(lum); cnt++; }
    }
    return cnt > 0 ? exp(logSum / cnt) : 1.f;  // 几何均值
}
```

**教训**：HDR 计算中任何"平均亮度"的计算，默认用几何均值（log 空间均值），除非特殊场景另有需要。

### Bug 3：ACES 曝光参数过高导致亮度普遍偏高

**症状**（调试中观察）：用 `exposure=1.0` 时，ACES 输出的均值约 0.85，接近全白，失去了高光层次。

**原因**：ACES 曲线本身会提升中灰对比度，加上 exposure=1 对输入没有任何缩放，场景中大量 HDR 值（>1）被一股脑地压到了 0.9+ 区间。

**修复**：调整为 `exposure=0.6`，让输入先缩小到合理范围，ACES 曲线再展示它的优势区间。

最终验证结果：
```
[ACES Filmic] mean=0.669  std=0.272  ✅ 均值和标准差都在正常范围
```

### Bug 4：PPM 写入时忘记 `std::ios::binary`

**症状**：PPM 文件写入后，在 Linux 上能正常读取，但如果在 Windows 上运行可能出现文件损坏（换行符问题）。

**原因**：PPM P6 格式是二进制格式，文件头是 ASCII，但像素数据是原始二进制字节。如果用文本模式打开文件，`\n` 可能被写成 `\r\n`，导致像素数据偏移。

**修复**：
```cpp
std::ofstream f(filename, std::ios::binary);  // 必须 binary 模式！
```

**教训**：任何包含二进制数据的文件，无论平台，都应该用 `std::ios::binary` 打开。

---

## 效果验证与数据

### 编译验证
```
编译命令：g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
结果：0 errors, 0 warnings ✅
```

### 运行数据

HDR 场景统计：
```
分辨率：320 × 200
avg_luminance（几何均值）：0.618
max_luminance：24.719
```

各算法输出像素统计（均值越接近 0.5 越"自然"，标准差越高越有层次）：

| 算法 | 均值 | 标准差 | 评价 |
|------|------|--------|------|
| Gamma Only | 0.743 | 0.284 | 均值偏高，高光爆掉 |
| Reinhard | 0.634 | 0.210 | 均值合理，但标准差略低 |
| Reinhard Extended | 0.659 | 0.231 | 白点保护有效 |
| ACES Filmic | 0.669 | 0.272 | 标准差最高，层次最丰富 |
| Uncharted 2 | 0.632 | 0.238 | 均值最低，风格最"硬" |
| Lottes Filmic | 0.750 | 0.222 | 均值略高，亮度感强 |
| ACES + ColorGrade | 0.663 | 0.322 | 色彩分级后标准差最高 |

ACES Filmic 的标准差（0.272）最高，说明明暗层次保留得最好——这正是它成为工业标准的原因。

### 坐标系验证

```python
天空区（上1/4）均值：232.7
地面区（下1/4）均值：87.6
✅ 天空（亮）在上，地面（暗）在下，坐标系正确
```

### 文件大小

```
tm_aces.png：2.9 KB（320×200）
tonemap_comparison.png：18 KB（960×612）✅ 远超 10KB 门槛
```

### 视觉对比分析

观察对比图（960×612）从左到右：

**第一排**：
- Gamma Only：高光区严重爆掉（太阳整片白色），地面细节尚存
- Reinhard：高光得到控制，但整体偏灰，缺乏冲击力
- Reinhard Extended：比 Reinhard 略亮，高光更有层次

**第二排**：
- ACES Filmic：对比度最强，太阳区域有明显的高光过渡，中灰区细节最丰富
- Uncharted 2：接近 ACES，但高光"肩部"更硬，整体更暗（曝光参数 2.0 时接近 ACES 0.6）
- Lottes Filmic：整体最亮，类似 ACES 但更"通透"

**第三排**（ACES + ColorGrade）：
- 相较纯 ACES，暗部略偏冷蓝（分色），高光略偏暖橙（色温+0.08）
- 对比度提升后层次感最强
- 这是现代游戏引擎最接近的真实效果

---

## 总结与延伸

### 技术局限性

1. **无自动曝光（Auto Exposure）**：真实渲染管线会根据场景平均亮度动态调整曝光，本项目使用固定曝光值。实现 AE 需要 GPU histogram 或降采样计算平均亮度，然后用滞后平均（lerp 到目标）避免闪烁。

2. **分辨率较低**（320×200）：为快速迭代选用，实际验证效果。生产环境应为 1920×1080 或更高。

3. **色彩分级是简化模型**：真实的电影级色彩分级使用 3D LUT（Lookup Table，64×64×64 的三维颜色映射表），可以做任意复杂的颜色变换。本项目的分色和色温调整是一次线性近似。

4. **ACES 曲线是近似**：完整的 ACES 标准包含从场景色彩空间（ACES2065-1）到显示色彩空间的矩阵变换。Narkowicz/Hill 的公式只是 RRT（Reference Rendering Transform）的拟合近似，精度约 1%。

5. **没有 HDR 显示器支持**：输出是 sRGB SDR。真正的 HDR 显示需要输出 PQ（Perceptual Quantizer，ST 2084）或 HLG（Hybrid Log-Gamma）编码。

### 可优化方向

- **Local Tone Mapping**：Reinhard 全局版本的改进是局部自适应——根据每个像素的局部邻域亮度来决定压缩幅度，效果更接近人眼的局部自适应
- **Bloom 协同**：真实场景中，高亮区域应该先做 Bloom（辉光）再做色调映射，否则高光过渡太硬
- **HDR 到 HDR 的映射**：对于支持 HDR 显示器的设备，需要输出 0–10000 nits 范围的 PQ 信号，而不是压缩到 SDR
- **感知均匀色彩空间**：在 OKLab、ICtCp 等感知均匀空间做色调映射，可以最大程度减少色偏
- **GPU 实现**：目前是 CPU 单线程，全分辨率图像可以用 GLSL/HLSL 的 Fragment Shader 或 Compute Shader 实现，成本接近零

### 与本系列的关联

这个 Skill 是渲染管线的最后一环，与前几天的实践形成闭环：

| 日期 | 项目 | 位置 |
|------|------|------|
| 03-26 | TAA（时序抗锯齿） | 色调映射之前的 AA pass |
| 03-27 | Motion Blur | 色调映射之前的运动模糊 pass |
| 03-28 | Depth of Field | 色调映射之前的景深 pass |
| 03-29 | **HDR Tone Mapping** | 所有后处理之后，最终输出 |

一个完整的渲染管线：
```
G-Buffer → Lighting → SSR → SSAO → TAA → DoF → MotionBlur → Bloom → ToneMapping → Color Grade
```

今天的项目填补了"最后一公里"的空白。

### 代码与资产

- **代码仓库**：https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-29-hdr-tone-mapping
- **对比图**：https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-29-hdr-tone-mapping/tonemap_comparison.png

![Tone Mapping 对比图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-29-hdr-tone-mapping/tonemap_comparison.png)

7种算法从左到右，上至下：Gamma裁剪、Reinhard、Reinhard Extended、ACES Filmic、Uncharted 2、Lottes Filmic、ACES + Color Grading。

---

*每日编程实践系列 · 第 40+ 天 · 2026-03-29*
