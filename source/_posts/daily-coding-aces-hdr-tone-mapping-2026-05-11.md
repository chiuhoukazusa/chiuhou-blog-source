---
title: "每日编程实践: ACES Filmic Tone Mapping & HDR Rendering"
date: 2026-05-11 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 色调映射
  - HDR渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-11-aces-hdr/aces_hdr_output.png
---

## 背景与动机

在现实世界中，光照的动态范围是极其宽广的。从阳光直射地面（约100,000 lux）到室内阴暗角落（约10 lux），亮度差距可以高达10,000倍。人类的视觉系统有复杂的适应机制——瞳孔收缩与扩张、视杆细胞与视锥细胞的协作，使我们能够在如此大的范围内感知细节。

然而，计算机显示器能够显示的亮度范围（即便是顶级HDR显示器）也不过300~1000 nits，对比度约1000:1左右。标准SDR（Standard Dynamic Range）显示器更只有100~300 nits。于是，当我们在渲染引擎中模拟真实世界的光照时，输出的HDR辐射值（Radiance）可能远超过显示器能表示的范围——一个灯泡的直接照射可能使某像素的辐射值达到50.0甚至100.0（以1.0为"中性灰18%反射率"参考值时）。

**没有色调映射会出现什么问题？**

直接裁剪（Clamp）是最原始的处理方式：把所有大于1.0的值直接设为1.0。这会导致高光区域完全变成死白，失去一切细节；而且明暗对比关系会被严重扭曲，图像看起来既刺眼又不真实。这就是为什么早期实时渲染游戏在强光下会有大片白色区域"爆掉"——没有合适的色调映射。

**工业界的真实使用**

如今，几乎所有AAA游戏引擎和主流渲染器都内置了色调映射算子：

- **Unreal Engine 4/5**：默认使用ACES Filmic色调映射，并允许用户调整曝光（EV100）
- **Unity HDRP**：提供ACES、Reinhard、Neutral等多种映射曲线选项
- **Frostbite（战地系列）**：采用了自研的Filmic曲线，兼顾艺术控制和物理精确
- **寒霜引擎（EA）**：专为HDR显示器开发了双通道色调映射方案
- **离线渲染（Arnold, V-Ray）**：提供完整的曝光控制工具集，让艺术家以类似摄影师的方式调整"曝光量"

色调映射不仅是技术问题，更是艺术工具：通过调整曲线的形状，可以赋予画面不同的"电影感"。ACES（Academy Color Encoding System，学院色彩编码系统）最初由好莱坞电影工业开发，后来被游戏行业广泛采用，成为视觉效果的行业标准。

---

## 核心原理

### 什么是色调映射

色调映射（Tone Mapping）是将场景的HDR辐射值映射到显示设备能够呈现的LDR（0~1）范围的过程。形式上，它是一个函数：

```
f: [0, ∞) → [0, 1]
```

一个好的色调映射函数应该满足：
1. **单调递增**：更亮的像素映射后仍然更亮
2. **保持对比度**：中间调的对比度损失最小
3. **软高光**（Soft Highlight）：高光区域平滑压缩，而非硬截断
4. **阴影保留**：暗部细节不能被压黑

### 各算法的数学形式

#### Linear（线性裁剪）
```
f(x) = clamp(x, 0, 1)
```
最简单，效果最差。大于1的值直接被截断，高光死白，完全损失细节。

#### Reinhard（2002）
Reinhard等人2002年在SIGGRAPH发表的经典论文《Photographic Tone Reproduction for Digital Images》提出：

```
f(x) = x / (1 + x)
```

数学上这是一条双曲线，渐进于1但永远不会超过1。对于任意正输入，输出都在(0, 1)范围内。

**直觉解释**：想象你把很亮的光通过一块"魔法玻璃"，玻璃越亮透光比例越低。输入为0时透光100%，输入为10时透光约9%，输入越大透光率越低。

**基于亮度的改进版**：直接对RGB各通道分别做Reinhard会导致颜色偏移（色调漂移）。改进方案是先计算亮度Y = 0.2126R + 0.7152G + 0.0722B，对亮度做映射后再等比缩放RGB，保持色相不变：

```
Y_out = Y / (1 + Y)
scale = Y_out / Y
RGB_out = RGB * scale
```

#### Uncharted 2 Filmic（Hable, 2010）
顽皮狗程序员John Hable在GDC 2010的演讲"Uncharted 2: HDR Lighting"中提出了这条曲线，因其被《神秘海域2》使用而得名：

核心是一个辅助函数 `U2(x)`：
```
U2(x) = ((x*(A*x + C*B) + D*E) / (x*(A*x + B) + D*F)) - E/F

其中参数：
A = 0.15 (Shoulder Strength，高光区的"肩部"强度)
B = 0.50 (Linear Strength，线性区强度)
C = 0.10 (Linear Angle，线性区角度)
D = 0.20 (Toe Strength，暗部"趾部"强度)
E = 0.02 (Toe Numerator，趾部分子)
F = 0.30 (Toe Denominator，趾部分母)
```

最终映射：
```
f(x) = U2(x * exposure) / U2(W)
```
其中W = 11.2是"白点"（White Point），表示输入值超过W时被映射到1。除以U2(W)是为了将曲线归一化，确保白点对应输出1.0。

**为什么要有肩部（Shoulder）和趾部（Toe）？**

一条理想的Filmic曲线形状如S型（Sigmoid）：
- **趾部**（低亮度区）：轻微向上弯曲，使深色区域略微提亮，模拟胶片在极暗处的非线性响应
- **线性区**（中间调）：近似线性，保持正常对比度
- **肩部**（高亮区）：向下弯曲，使亮部柔和压缩，高光不会突然截断

这个S形正是模拟了真实胶片的感光曲线——胶片对光的响应并不是线性的，而是有这种特征性的S形感光特性曲线（H&D曲线）。

#### ACES Filmic（Narkowicz近似，2015）
ACES是好莱坞学院颁布的色彩科学标准。完整的ACES管线包括输入变换（IDT）、参考渲染变换（RRT）和输出设备变换（ODT）。

Narkowicz在2015年的博客中提出了一个极简近似公式，只需5个参数：
```
f(x) = (x * (a*x + b)) / (x * (c*x + d) + e)

a = 2.51, b = 0.03
c = 2.43, d = 0.59, e = 0.14
```

这是一个有理函数（分子分母都是二次多项式），优点是计算极快，只需几次乘法和一次除法。

**为什么选这5个参数？**

这组参数是通过拟合完整ACES管线的RRT+ODT曲线得到的，在保证极低计算量的前提下，与完整ACES流程的视觉差异最小。输入需要乘以0.6的曝光系数（原作者建议）以对齐ACES的参考白点。

#### ACES Hill版本（矩阵精确实现）
Stephen Hill和Stephen Langlands提供了更精确的ACES实现，通过两个矩阵变换来近似完整的RRT+ODT变换：

**第一个矩阵（输入变换，近似IDT到线性场景）**：
```
M_in = [0.59719  0.35458  0.04823]
       [0.07600  0.90834  0.01566]
       [0.02840  0.13383  0.83777]
```

**RRT+ODT拟合曲线**：
```
f(x) = (x*(x + 0.0245786) - 0.000090537) / (x*(0.983729*x + 0.4329510) + 0.238081)
```
（分别对R、G、B三通道独立应用）

**第二个矩阵（输出变换，近似ODT）**：
```
M_out = [ 1.60475  -0.53108  -0.07367]
        [-0.10208   1.10813  -0.00605]
        [-0.00327  -0.07276   1.07602]
```

这两个矩阵的设计确保了色彩在从场景线性空间到显示空间的变换中，色域映射更加准确，高光区域的色彩偏移比Narkowicz近似更小。

#### Lottes Filmic（2016）
Timothy Lottes提出了另一条Filmic曲线，参数化更加直观：

```
f(x) = x^a / (x^(a*d) * b + c)
```

参数：
- `a`：Contrast（对比度）
- `d`：Shoulder Control（肩部控制）
- 其余参数由白点和中灰点约束条件推导得出

Lottes曲线的特点是对暗部保护更好，适合夜晚场景或低对比度场景。

### sRGB Gamma校正

所有色调映射函数输出的是线性光照值，但显示器期望输入sRGB编码值。sRGB的伽马近似：

```
线性值 ≤ 0.0031308：  sRGB = 12.92 * linear
线性值  > 0.0031308：  sRGB = 1.055 * linear^(1/2.4) - 0.055
```

**为什么不用简单的 pow(x, 1/2.2)？**

`pow(x, 1/2.2)` 是早期sRGB标准的近似，在接近0的区域斜率不连续，会导致暗部量化误差更大。完整的sRGB公式在接近0的区域使用线性近似，避免了这个问题。在8位输出时这个差别很明显。

---

## 实现架构

### 渲染管线设计

```
场景定义（球体 + 光源）
     ↓
HDR渲染（光线投射 + Phong/GGX着色）
     ↓
HDR帧缓冲（float32，可能 > 1.0）
     ↓
┌─────────────────────────────────┐
│ 8 个色调映射面板（4列 × 2行）  │
│ Linear / Reinhard / RH-Lum /   │
│ Uncharted2 / ACES-N / ACES-H / │
│ Lottes / ACES-N 0.5EV          │
└─────────────────────────────────┘
     ↓
sRGB Gamma校正
     ↓
PNG输出（1600 × 600）
```

**关键设计决策：共享HDR帧缓冲**

8个面板共享同一个HDR渲染结果，这样才能做公平的色调映射对比。每个面板只是在着色阶段应用不同的映射函数，渲染过程完全一致。

这与实际渲染引擎的架构一致：渲染管线最终输出HDR帧缓冲，色调映射作为后处理的第一步，在HDR → LDR的转换中发生。

### 场景设计与高动态范围构建

场景包含8个球体（7个材质球 + 1个地面大球）和5个光源：

| 光源 | 颜色强度 | 作用 |
|------|---------|------|
| 主光（key light） | (30, 25, 18) | 暖色主光，制造强高光 |
| 补光（fill light） | (8, 10, 15) | 冷色补光，填充阴影 |
| 轮廓光（rim light） | (15, 12, 20) | 背光，突出轮廓 |
| 热点光 | (80, 60, 10) | 极亮黄色光，制造过曝区域 |
| 蓝色点缀 | (5, 8, 40) | 蓝色强光，测试色相响应 |

这些光源强度远超过物理中性灰（1.0），是刻意设计的，为了让不同色调映射算法的差异更明显。热点光的强度80.0意味着该区域在线性空间会产生极端高光，不同算法处理这部分的方式差异最大。

另外，一个球体被设为自发光（emissive = 4.0），相当于直接向场景贡献能量，这是测试Filmic高光恢复能力的好方式。

### 着色模型

使用简化的Cook-Torrance微面元（Microfacet）模型：

**漫反射**：Lambertian，能量保守：
```cpp
Vec3 diffuse = albedo * kD * (1.0 / PI)
```

**镜面反射**：GGX NDF + Schlick Fresnel + 简化G项：
```cpp
D = GGX_NDF(NoH, roughness)       // 法线分布函数
F = Schlick_Fresnel(LoH, F0)      // 菲涅尔项
G = min(1, min(2*NoH*NoV/LoH, 2*NoH*NoL/LoH))  // 几何遮蔽（简化）
specular = D * F * G / (4 * NoV * NoL)
```

最终辐射值：
```cpp
radiance += (diffuse + specular) * lightColor * NoL
```

光源强度通过平方距离衰减：`attenuation = 1 / (dist^2 + 0.1)`，额外乘以系数10以控制场景整体亮度。

---

## 关键代码解析

### Vec3 数学库

```cpp
struct Vec3 {
    float x, y, z;
    Vec3 operator*(const Vec3& o) const { return {x*o.x, y*o.y, z*o.z}; }
    Vec3 operator/(const Vec3& o) const { return {x/o.x, y/o.y, z/o.z}; }
    Vec3 clamp01() const { return {max(0.f,min(1.f,x)), max(0.f,min(1.f,y)), max(0.f,min(1.f,z))}; }
    Vec3 reflect(const Vec3& n) const { return *this - n * (2.f * dot(n)); }
};
```

注意Vec3支持逐元素乘法和除法（不是点积！），这在色调映射的白点归一化中至关重要。

### Reinhard 亮度保留版

```cpp
Vec3 reinhardLuminance(const Vec3& hdr) {
    // 使用 Rec.709 亮度系数，与sRGB色彩空间对应
    float lum = hdr.x*0.2126f + hdr.y*0.7152f + hdr.z*0.0722f;
    float lumOut = lum / (1.f + lum);
    // 等比缩放，保持色相（Hue）和饱和度（Saturation）
    float scale = (lum > 1e-6f) ? lumOut / lum : 1.f;
    return Vec3(hdr.x*scale, hdr.y*scale, hdr.z*scale).clamp01();
}
```

关键点：`(lum > 1e-6f)` 的保护是为了防止除以零。纯黑像素（lum≈0）直接返回原值即可。

为什么要保持色相？如果对R、G、B分别做Reinhard：
```cpp
// 错误做法：颜色会偏移
r_out = r / (1 + r)
g_out = g / (1 + g)
b_out = b / (1 + b)
```
当输入颜色是(10, 0.1, 0.1)（鲜红高光）时，输出变成(0.909, 0.091, 0.091)——红色比例大幅降低，颜色偏向灰色，这不是我们想要的结果。

### Uncharted2 实现

```cpp
static Vec3 uncharted2Partial(Vec3 x) {
    const float A = 0.15f, B = 0.50f, C = 0.10f;
    const float D = 0.20f, E = 0.02f, F = 0.30f;
    // 分子：x*(A*x + C*B) + D*E
    Vec3 num = x*(x*A + C*B) + Vec3(D*E);
    // 分母：x*(A*x + B) + D*F
    Vec3 den = x*(x*A + B) + Vec3(D*F);
    // 逐元素相除，减去趾部偏移 E/F
    return Vec3(num.x/den.x, num.y/den.y, num.z/den.z) - Vec3(E/F);
}

Vec3 uncharted2(const Vec3& hdr) {
    float exposure = 2.0f;  // 曝光补偿，让中间调更亮
    Vec3 curr = uncharted2Partial(hdr * exposure);
    // 白点归一化：除以 U2(W)，确保 W 映射到 1.0
    Vec3 W = Vec3(11.2f);
    Vec3 wp = uncharted2Partial(W);
    Vec3 whiteScale = Vec3(1.f/wp.x, 1.f/wp.y, 1.f/wp.z);
    return (curr * whiteScale).clamp01();
}
```

初始实现犯了一个错误：直接写 `Vec3(1.f) / uncharted2Partial(W)` 时，Vec3的除法运算符未定义，导致编译错误。修复是显式展开为逐元素倒数。

### ACES Narkowicz（最简实现）

```cpp
Vec3 acesNarkowicz(const Vec3& hdr) {
    const float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    // 逐通道应用有理函数
    auto f = [&](float x) -> float {
        return max(0.f, min(1.f, (x*(a*x+b)) / (x*(c*x+d)+e)));
    };
    return Vec3(f(hdr.x), f(hdr.y), f(hdr.z));
}
```

为什么不需要额外的gamma校正？这个函数已经将线性光照值映射到接近sRGB编码域，后面统一叠加gamma校正即可。

### ACES Hill 矩阵版本

```cpp
Vec3 acesHill(Vec3 v) {
    // 通用矩阵乘向量 helper
    auto m33 = [](Vec3 r0, Vec3 r1, Vec3 r2, Vec3 v) -> Vec3 {
        return Vec3(r0.dot(v), r1.dot(v), r2.dot(v));
    };
    
    // 步骤1：输入颜色矩阵变换（将场景线性RGB转换到ACES AP0色彩空间）
    v = m33(
        Vec3(0.59719f, 0.35458f, 0.04823f),
        Vec3(0.07600f, 0.90834f, 0.01566f),
        Vec3(0.02840f, 0.13383f, 0.83777f),
        v
    );
    
    // 步骤2：RRT+ODT有理函数拟合（核心色调压缩）
    auto rrt = [](Vec3 x) -> Vec3 {
        Vec3 num = x * (x + Vec3(0.0245786f)) - Vec3(0.000090537f);
        Vec3 den = x * (x * Vec3(0.983729f) + Vec3(0.4329510f)) + Vec3(0.238081f);
        return Vec3(num.x/den.x, num.y/den.y, num.z/den.z);
    };
    v = rrt(v);
    
    // 步骤3：输出颜色矩阵变换（将ACES AP1转换回sRGB色彩空间）
    v = m33(
        Vec3( 1.60475f, -0.53108f, -0.07367f),
        Vec3(-0.10208f,  1.10813f, -0.00605f),
        Vec3(-0.00327f, -0.07276f,  1.07602f),
        v
    );
    return v.clamp01();
}
```

这两个矩阵的物理意义：
- **第一个矩阵**：将相机捕获的sRGB线性空间颜色转换到ACES AP0色域（一个非常宽的色彩空间，包含几乎所有可见颜色）
- **第二个矩阵**：将处理后的颜色从ACES AP1（略小于AP0）转换回sRGB色域，准备显示

为什么需要这两个矩阵变换？因为ACES的RRT曲线是为ACES AP0/AP1色彩空间设计的，如果直接把sRGB颜色送进去，RRT曲线的色相旋转效果会不正确，高光区域的颜色会偏移。

### GGX 法线分布函数

```cpp
float ggxNDF(float NoH, float roughness) {
    float a = roughness * roughness;  // 粗糙度需要平方（感知线性映射）
    float a2 = a * a;
    // Trowbridge-Reitz GGX分布
    // D = a² / (π * ((NoH² * (a² - 1) + 1)²))
    float denom = NoH * NoH * (a2 - 1.f) + 1.f;
    return a2 / (3.14159265f * denom * denom + 1e-7f);
}
```

为什么粗糙度要平方？这是一个感知空间到实际参数的映射。粗糙度为0.5在视觉上应该对应"中等粗糙"，但如果直接用0.5送入GGX公式，视觉效果会偏向"非常光滑"。将粗糙度平方后，视觉上的线性感知与参数之间更加一致，这也是Unreal Engine等引擎的标准做法。

### 阴影光线测试

```cpp
bool shadowTest(const Vec3& pos, const Vec3& lightPos) {
    Vec3 dir = lightPos - pos;
    float dist = dir.len();
    // 沿光源方向发射阴影光线，起点沿法线偏移避免自交
    Ray shadowRay(pos + dir.norm() * 1e-3f, dir);
    HitInfo hit;
    // 仅检测到光源之前的遮挡（不包括光源本身）
    if (traceScene(shadowRay, 1e-3f, dist - 1e-2f, hit)) return true;
    return false;
}
```

两个细节防止自相交（Self-Intersection）：
1. 起点沿方向偏移 `1e-3f`：防止光线立即与自身球面相交
2. 最大距离为 `dist - 1e-2f`：防止将光源所在点附近的物体算成遮挡

### 字体渲染

图像中的标签文字通过内嵌的5×7像素位图字体绘制，完全不依赖任何外部字体库：

```cpp
static const uint8_t FONT5X7[][7] = {
    // A: 每行是5位位掩码，bit从高位到低位对应列0~4
    {0x0E,0x11,0x11,0x1F,0x11,0x11,0x00}, // A
    // 0x0E = 0b01110 → 第0列空，第1-3列亮，第4列空 → 横线
    // 0x11 = 0b10001 → 两侧亮，中间暗 → "A"的两竖
    // 0x1F = 0b11111 → 全亮 → "A"的横梁
    ...
};

void drawText(vector<uint8_t>& img, int W, int H, int x, int y, 
              const string& text, Vec3 color) {
    int cx = x;
    for (char c : text) {
        int idx = charIndex(c);
        for (int row = 0; row < 7; row++) {
            for (int col = 0; col < 5; col++) {
                // 检查位掩码中对应位是否为1
                if (FONT5X7[idx][row] & (0x10 >> col)) {
                    // 写入像素
                    int i = ((y + row) * W + (cx + col)) * 3;
                    img[i] = r; img[i+1] = g; img[i+2] = b;
                }
            }
        }
        cx += 6;  // 每个字符宽5像素 + 1像素间距
    }
}
```

---

## 踩坑实录

### Bug 1：Vec3 除法运算符缺失导致编译错误

**症状**：编译时报 `no match for 'operator/' (operand types are 'Vec3' and 'Vec3')`

**错误假设**：以为C++中 `Vec3 / Vec3` 会默认逐元素运算，或者在namespace内定义的自由函数能被找到。

**真实原因**：Vec3类只定义了 `operator/(float t)` 即标量除法，没有定义Vec3/Vec3的逐元素除法。Uncharted2的白点归一化需要 `Vec3(1.f) / uncharted2Partial(W)` 即两个Vec3相除，但运算符不存在。

另外，namespace `ToneMap` 内定义了一个自由函数 `inline Vec3 operator/(...)`，但这个定义在后续的另一个函数中无法被找到，因为C++的名字查找规则（ADL）在这里不适用。

**修复**：在Vec3结构体中直接添加成员函数：
```cpp
Vec3 operator/(const Vec3& o) const { return {x/o.x, y/o.y, z/o.z}; }
```
同时删除namespace内的重复定义，并将Uncharted2的白点归一化改为显式展开：
```cpp
Vec3 wp = uncharted2Partial(W);
Vec3 whiteScale = Vec3(1.f/wp.x, 1.f/wp.y, 1.f/wp.z);
```

**教训**：在数学库类中，每次添加新运算需要，先检查是否所有常用的二元运算符（加减乘除，标量和向量版本）都已定义完整。

### Bug 2：`std::function` 未引入导致编译错误

**症状**：`'function' does not name a type; did you mean 'union'?`

**错误假设**：`std::function` 和 `std::vector` 一样，包含 `<vector>` 头文件就够了。

**真实原因**：`std::function` 定义在 `<functional>` 头文件中，不是标准库的自动包含部分。不同编译器版本的传递包含行为不同，不能依赖。

**修复**：添加 `#include <functional>`。

**教训**：用到什么包含什么，不要依赖传递包含。特别是 `std::function`、`std::sort`（需要`<algorithm>`）、`std::string`（需要`<string>`）这类容易遗漏的头文件。

### Bug 3：ACES Hill 实现中的中间变量污染

**症状**：ACES Hill输出的颜色看起来过于饱和，与ACES Narkowicz差异过大。

**错误假设**：变量 `a` 的临时计算结果不会影响后续流程。

**真实原因**：代码中有一段死代码计算：
```cpp
Vec3 a = v + Vec3(0.0245786f) - Vec3(0.000090537f) / (v + Vec3(0.0245786f));
```
这一行原本是尝试内联RRT计算，但写法错误，而且变量 `a` 之后并没有被使用，正确的RRT逻辑在lambda `rrt` 中。编译器会警告 "unused variable"，这个警告要重视。

**修复**：删除无用的中间计算，添加 `(void)a;` 抑制警告，最终直接删除整行。

**教训**：C++中临时变量如果和正式变量重名（如函数参数 `a`），可能导致遮蔽（shadowing）bug。`-Wall -Wextra` 的 `-Wshadow` 警告要开启，所有warning都应解决。

### Bug 4：坐标系问题（图像上下颠倒）

**症状**：编译运行正常，但场景中天空应该在上方，地面（大球）应该在下方，如果UV坐标计算错误，会出现上下颠倒。

**预防措施**：在渲染循环中明确了Y轴的翻转：
```cpp
float v = (TH - 1 - y + 0.5f) / TH;
```
屏幕坐标Y从上到下增加（y=0是顶部），但光线追踪的世界坐标Y向上为正。所以要用 `TH-1-y` 翻转，使屏幕顶部对应世界空间的上方。

验证方法：检查图像上半部分（天空）平均亮度是否高于下半部分（地面）：
```
上1/3均值: 188.3 > 下1/3均值: 176.5  ✅
```

---

## 效果验证与数据

### 输出文件信息

```
文件：aces_hdr_output.png
尺寸：1600 × 600 像素（4列 × 2行，每格400×300）
大小：431 KB
渲染时间：0.209秒（SPP=4，1个场景渲染 + 8次色调映射）
```

### 像素统计验证

```
像素均值：180.7（范围 10~240 ✅）
标准差：76.3（> 5 ✅）
上1/3均值：188.3  下1/3均值：176.5  坐标系正确 ✅
```

### 各色调映射算法的视觉特点对比

| 算法 | 高光处理 | 暗部表现 | 色彩倾向 | 典型使用场景 |
|------|---------|---------|---------|------------|
| Linear | 死白截断 | 较好 | 无偏移 | 调试用 |
| Reinhard | 柔和，略偏灰 | 略微提亮 | 轻微去饱和 | 简单实现 |
| Reinhard-Lum | 柔和，色相保留 | 较好 | 色相稳定 | 改进版Reinhard |
| Uncharted2 | S曲线，有胶片感 | 有趾部细节 | 暖色调 | 动作游戏 |
| ACES Narkowicz | 对比度强，电影感 | 偏暗 | 中性偏冷 | 现代游戏标准 |
| ACES Hill | 更精确的色域 | 接近Narkowicz | 色相旋转准确 | 需要色彩精准时 |
| Lottes | 暗部保护强 | 最好 | 轻微偏冷 | 夜晚/暗场景 |

### 性能数据

- 编译时间：约2秒（g++ -O2）
- 渲染时间：0.209秒（单线程，400×300 HDR帧，4 SPP）
- 内存使用：约7.2 MB（HDR帧缓冲 + 最终图像）
  - HDR帧缓冲：400×300×3×4 bytes = 1.44 MB
  - 最终图像：1600×600×3 bytes = 2.88 MB
  - 其余程序栈/堆：约2.88 MB

HDR帧缓冲只渲染一次（0.2秒），8种色调映射的额外开销可以忽略不计（每种仅需一次逐像素映射），这正是将色调映射设计为后处理步骤而非嵌入渲染器的优势。

---

## 总结与延伸

### 技术局限性

**本次实现的简化之处**：

1. **SPP=4**：每像素只采样4次，存在明显的锯齿。实际项目需要至少16 SPP或配合TAA（时序抗锯齿）。

2. **无HDR显示支持**：输出的是8位PNG，即使渲染为HDR、色调映射也映射到0~255。真正支持HDR显示器的管线需要输出10位或16位格式（如OpenEXR、HDR10、Dolby Vision）。

3. **简化光照模型**：几何遮蔽项G使用了简化公式，没有采用完整的Smith G2遮蔽阴影函数；漫反射也没有考虑环境光遮蔽。

4. **无时序信息**：静止帧，无法验证ACES在运动场景中是否会出现"时序闪烁"（颜色随帧变化）。

5. **色调映射不是完整ACES管线**：ACES标准包含相机IDT（输入设备变换）、场景线性、RRT、ODT等多个阶段，本文实现的只是近似RRT+ODT部分。

### 可以继续探索的方向

1. **自动曝光（Auto Exposure）**：根据当前帧的平均亮度（或用EV100测光），动态调整曝光系数。实时游戏中通常采用"直方图平均"或"加权中心测光"。

2. **局部色调映射**：全局算子（如上面所有实现）对所有像素应用相同的映射。局部算子根据像素周围的局部亮度动态调整，可以同时保留高光和阴影细节，但计算量更大。

3. **HDR显示直通**：针对支持HDR10或Dolby Vision的显示器，可以绕过色调映射直接输出HDR信号，由显示器做更精准的映射。

4. **色调映射与色彩分级分离**：工业标准做法是先做色调映射（物理过程），再做色彩分级（艺术调色，类似摄影中的后期处理），两个步骤要解耦。

5. **与本系列其他文章的关联**：
   - 04-03 Disney BRDF：HDR渲染的直接受益者，金属球和高粗糙度球在正确的色调映射下才能呈现应有的外观
   - 04-21 Bloom & Lens Flare：Bloom效果必须在HDR空间进行，色调映射之前
   - 05-01 ReSTIR：ReSTIR的输出是HDR辐射值，同样需要配合色调映射才能显示

### 结语

色调映射是连接"物理正确渲染"与"视觉呈现"的最后一道桥梁。理解各种算法的数学本质，有助于在实际项目中做出明智的选择——有时候Reinhard的简单朴素比ACES的复杂精确更适合特定的艺术风格；有时候Lottes的暗部保护比ACES的电影感更适合恐怖游戏的氛围。这种选择能力，是深入理解渲染管线后才能获得的。

本文代码已上传：[GitHub - daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-05-11-aces-hdr)
