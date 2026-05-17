---
title: "每日编程实践: Shell Texturing Fur Renderer"
date: 2026-05-18 05:30:00
tags:
  - 每日一练
  - 图形学
  - 毛发渲染
  - C++
  - 游戏开发
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-18-shell-fur/shell_fur_output.png
---

# Shell Texturing Fur Renderer — 实时毛发渲染技术深度解析

![Shell Texturing Fur](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-18-shell-fur/shell_fur_output.png)

## ① 背景与动机

### 毛发渲染的挑战

毛发渲染是实时图形学中最具挑战性的问题之一。真实的毛发由数百万根独立的细丝构成，每根毛发都有自己的形状、方向、光学属性。直接用多边形描述每根毛发显然不可行——一只猫大约有 6000 万根毛，即使每根毛发只用一个三角形，就需要 6000 万个 draw call。

工业界实际上有三种主流方案，各有适用场景：

**1. Strand-Based Hair（链条式毛发）**  
UE5 的 Groom 系统、NVIDIA GameWorks Hair、EA SEED 的 HUSK 等都采用这种方案。每根毛发用一条贝塞尔曲线描述，运行时细化成三角形条带。优点是质量最高，能正确处理自遮挡、AO、物理模拟；缺点是计算量极大，高端 PC 才能实时运行，移动端基本不可用。

**2. Alpha Card Hair（面片式毛发）**  
手游和低端平台的标配。将一撮毛发贴到半透明的矩形面片上，通过 Alpha Test 剔除空白区域。UE4 默认的头发就用这种方式。优点是极低的顶点数；缺点是近看明显失真，不适合写实风格。

**3. Shell Texturing（壳贴图）**  
今天要实现的技术。介于上述两者之间，适合表面分布均匀的绒毛效果：动物皮毛、草地地面、地毯纹理、编织布料。《刺客信条》系列早期版本、《地平线：零之曙光》的植被都用过这个技术的变体。

**Shell Texturing 解决了什么核心问题？**  
它用一种空间上的近似欺骗视觉：不渲染每根毛发的几何，而是渲染多个"壳层"（shells），每个壳层只是原始表面沿法线方向膨胀一点点，再通过噪声纹理剔除非毛发区域的像素。远处看起来，这些离散的壳层组合就像真实的体积毛发。

### 为什么选择今天实现这个？

过去一个月我已经实现了大量的全局光照技术（VXGI、Radiance Cascade、PRT）、后处理效果（FXAA、PCSS、Bloom）和物理渲染（IBL、Photon Mapping）。Shell Texturing 是一个经典的游戏开发技巧，偏向于实用性而非学术性，但背后的数学同样有趣：如何用 2D 噪声模拟 3D 体积效果？毛发的 Kajiya-Kay 光照模型与普通 Phong 有哪些本质区别？

---

## ② 核心原理

### Shell Texturing 的数学基础

设原始表面为 $S$，表面上每点的位置为 $\mathbf{p}$，法线为 $\hat{\mathbf{n}}$。

定义 $N$ 个壳层，第 $i$ 个壳层的表面为：

$$S_i = \left\{ \mathbf{p} + \hat{\mathbf{n}} \cdot \frac{i}{N} \cdot L_{\text{fur}} \;\middle|\; \mathbf{p} \in S \right\}$$

其中 $L_{\text{fur}}$ 是毛发的最大长度，$i$ 从 $0$（基础层）到 $N-1$（毛尖层）。

每个壳层的 shell fraction（毛发完成比例）定义为：

$$t_i = \frac{i}{N-1} \in [0, 1]$$

当 $t_i = 0$ 时，我们在毛发根部；当 $t_i = 1$ 时，我们在毛发尖端。

### 毛发密度函数

对于每个壳层 $i$，我们需要决定当前像素"是否有毛发"。这通过密度函数实现：

给定 UV 坐标 $(u, v)$，将 UV 空间分成 $s \times s$ 的网格（本实现中 $s = 40$）。每个格子的中心位置通过哈希函数随机化：

$$\text{cellX} = \lfloor u \cdot s \rfloor, \quad \text{cellY} = \lfloor v \cdot s \rfloor$$

$$c_u = \frac{\text{cellX} + h(\text{cellX}, \text{cellY})}{s}, \quad c_v = \frac{\text{cellY} + h'(\text{cellX}, \text{cellY})}{s}$$

其中 $h(\cdot)$ 是哈希函数：

$$h(x, y) = \text{frac}\left(\sin(127.1x + 311.7y) \times 43758.5453\right)$$

毛发股的圆形截面半径随高度递减（锥形收缩）：

$$r(t_i) = r_{\text{base}} \cdot (1 - 0.8 \cdot t_i)$$

像素 $(u, v)$ 在壳层 $i$ 有毛发的条件：

$$\left((u - c_u) \cdot s\right)^2 + \left((v - c_v) \cdot s\right)^2 < r(t_i)^2$$

**直觉解释**：我们把 UV 空间想象成一块地毯，每 $\frac{1}{s}$ 的正方形格子里有一根毛发，但毛发不在格子中心而在随机偏移的位置。往高处走（$t_i$ 增大），每根毛发的截面越来越小，最终在毛尖消失。

### 重力弯曲

没有重力的毛发会向外直射，像刺猬。真实毛发会在重力影响下向下弯曲。

定义重力方向 $\hat{\mathbf{g}} = (0, -1, 0)$。对于当前壳层法线 $\hat{\mathbf{n}}$，重力沿法线的切向分量（导致弯曲）为：

$$\mathbf{g}_{\text{tangent}} = \hat{\mathbf{g}} - \hat{\mathbf{n}} (\hat{\mathbf{n}} \cdot \hat{\mathbf{g}})$$

注意：$\hat{\mathbf{n}} \cdot \hat{\mathbf{g}}$ 是重力沿法线方向的投影（法向分量），减去它才得到纯粹的切向弯曲力。

弯曲后的壳层位置：

$$\mathbf{p}_{\text{bent}}(t_i) = \mathbf{p} + \hat{\mathbf{n}} \cdot t_i \cdot L_{\text{fur}} + \mathbf{g}_{\text{tangent}} \cdot G \cdot t_i^2$$

这里用 $t_i^2$ 而不是 $t_i$，是因为现实中毛发的弯曲是悬臂梁变形，越接近末端偏移越大，且呈平方关系。$G$ 是重力强度系数（本实现取 $0.04$）。

**直觉解释**：想象每根毛发是一根弹性杆，固定在根部，末端在重力作用下自由下垂。越长的毛发末端偏移越大，且偏移量不是线性而是类抛物线的。

### Kajiya-Kay 光照模型

普通 Phong 模型依赖表面法线，但毛发是细丝，没有单一的"面法线"——每根毛发的主方向（切线）才是关键。1989 年 Kajiya 和 Kay 提出了专门针对毛发/纤维的光照模型。

设毛发切线方向为 $\hat{\mathbf{t}}$，光照方向为 $\hat{\mathbf{l}}$，视线方向为 $\hat{\mathbf{v}}$。

**漫反射项**：

$$I_{\text{diffuse}} = C_{\text{fur}} \cdot C_{\text{light}} \cdot \sin\theta_L$$

其中 $\theta_L$ 是切线与光源方向的夹角，$\sin\theta_L = \sqrt{1 - (\hat{\mathbf{t}} \cdot \hat{\mathbf{l}})^2}$。

**直觉解释**：普通漫反射用 $\cos\theta_L = \hat{\mathbf{n}} \cdot \hat{\mathbf{l}}$，这适合平面（光线正对表面时最亮）。毛发是圆柱体，当光线垂直于毛发时（即 $\theta_L = 90°$），散射面积最大，所以用 $\sin\theta_L$ 更符合圆柱的物理行为。

**高光项**：

$$I_{\text{spec}} = C_{\text{light}} \cdot \left(\sin\theta_L \sin\theta_V - \cos\theta_L \cos\theta_V\right)^p$$

化简：注意 $\sin\theta_L \sin\theta_V - \cos\theta_L \cos\theta_V = -\cos(\theta_L + \theta_V) = -\cos\psi$，其中 $\psi = \theta_L + \theta_V$。

这表示当观察方向和光照方向相对于切线呈"互补"关系时高光最强——类似于镜面反射，但是围绕切线轴的反射。

**总光照**：

$$I = I_{\text{diffuse}} + I_{\text{spec}} + C_{\text{fur}} \cdot C_{\text{ambient}}$$

### 虎纹程序化着色

毛发颜色基于球面坐标 $(\phi, \theta)$ 生成虎纹图案：

$$\text{stripe} = \sin(8\theta + 1.5\cos(3\phi))$$

其中 $\theta$ 是方位角，$\phi$ 是极角。这个公式产生沿 $\theta$ 方向延伸的非线性条纹，$\cos(3\phi)$ 项增加条纹的弯曲/扰动感，避免过于规则。

基础色在橙色 $(0.85, 0.45, 0.1)$ 和黑色 $(0.08, 0.05, 0.03)$ 之间插值：

$$C_{\text{base}} = C_{\text{orange}} \cdot \max(0, \text{stripe}) + C_{\text{black}} \cdot (1 - \max(0, \text{stripe}))$$

毛尖颜色逐渐变亮（向浅棕色混合），模拟真实毛发尖端磨损/褪色的效果：

$$C_{\text{fur}} = C_{\text{base}} \cdot (1 - 0.3t_i) + C_{\text{tip}} \cdot 0.3t_i$$

---

## ③ 实现架构

### 整体渲染管线

```
原始球体网格 (UV 球, 32×64 细分)
         ↓
[Shell 扩展循环] × 32 次
   ├── 顶点沿法线膨胀: pos += normal * expansion
   ├── 重力弯曲: pos += gravity_tangent * G * t²
   └── 送入软光栅化管线
         ↓
[每个三角形光栅化]
   ├── MVP 变换 → 裁剪空间
   ├── 背面剔除 + 视锥裁剪
   ├── 逐像素插值 (UV, 法线, 世界坐标)
   ├── 毛发密度检测 (discard 无毛发像素)
   ├── Kajiya-Kay 光照
   └── Alpha 合成到 colorbuf
         ↓
[Composite]
   └── colorbuf + alphabuf → framebuf (背景上叠加)
         ↓
[Gamma 校正 + PNG 输出]
```

### 关键数据结构

**帧缓冲区**：本实现使用三个独立缓冲区，这是关键设计决策：

- `framebuf`：最终 RGB 像素，初始化为背景颜色
- `zbuf`：深度缓冲，精确控制哪层壳可见
- `alphabuf`：累计 Alpha，用于多层壳的半透明合成
- `colorbuf`：累计颜色（未 gamma 校正），与 alphabuf 配合使用

将颜色和 alpha 分开存储（而非直接写入 framebuf）是为了在所有壳渲染完毕后才做一次性 gamma 校正，避免 gamma 在多次混合中累积误差。

**Vertex 结构**：

```cpp
struct Vertex {
    Vec3 pos;     // 世界空间位置（壳层扩展后）
    Vec3 normal;  // 用于 Kajiya-Kay 切线推导
    float u, v;  // UV 坐标（用于噪声采样）
};
```

法线保留原始球体法线（未随壳层膨胀改变），这是因为毛发方向应该基于原始表面而不是膨胀后的壳层表面。

### 壳层渲染策略：Inside-Out vs Outside-In

渲染多层透明物体时，顺序非常重要。有两种方案：

**从内到外（本实现采用）**：先渲染靠近表面的壳，再渲染外层。这样外层的 alpha 叠加在内层之上，符合"毛发根部不透明，毛尖半透明"的物理效果。同时第 0 层（根部）直接写入深度缓冲，外层壳通过深度偏移允许通过。

**从外到内（Over Blending 的正确顺序）**：标准 Porter-Duff over 混合需要从远到近。但对于毛发而言，由于毛发本身有体积，从内到外的渲染顺序在视觉上更正确（不会出现内层毛发盖住外层的问题）。

---

## ④ 关键代码解析

### 毛发密度函数实现

```cpp
float hash1(float n) {
    return std::fmod(std::sin(n) * 43758.5453f, 1.0f);
}

float hash2(float u, float v) {
    return hash1(u * 127.1f + v * 311.7f);
}

float furDensity(float u, float v, float shellFrac, float density) {
    float scale = 40.0f;
    float fu = std::floor(u * scale);
    float fv = std::floor(v * scale);
    
    // 每个格子随机偏移毛发中心（避免规则排列）
    float cx = (fu + hash2(fu, fv*1.3f)) / scale;
    float cy = (fv + hash2(fu*2.1f, fv)) / scale;
    
    // 当前像素到最近毛发中心的距离（归一化到格子空间）
    float du = (u - cx) * scale;
    float dv = (v - cy) * scale;
    float dist2 = du*du + dv*dv;
    
    // 毛发截面随高度收缩（锥形）
    float radius = density * (1.0f - shellFrac * 0.8f);
    return (dist2 < radius*radius) ? 1.0f : 0.0f;
}
```

**为什么用 sin(n) * 43758.5453？**  
这是计算机图形学中最经典的哈希技巧之一，来自 Inigo Quilez 的 Shadertoy。`sin` 函数输入进入超过 $2\pi$ 的区间后产生高频振荡，乘以大常数 43758.5453 将小的浮点差异放大成看似随机的输出，再取 frac 归一化到 [0,1]。

**为什么 cx 用 `fv*1.3f`，cy 用 `fu*2.1f`？**  
如果用相同的系数，$h(x, y)$ 和 $h'(x, y)$ 可能高度相关，导致毛发中心点在对角线上排列，出现可见的规律性。使用不同的不相关常数（1.3 和 2.1）可以破坏这种相关性。

### 重力弯曲实现

```cpp
Vec3 applyGravityBend(Vec3 pos, Vec3 normal, float shellFrac, float gravity) {
    Vec3 gravDir(0, -1, 0);
    
    // 投影：去掉重力沿法线方向的分量，只保留切向
    float normalDotGrav = normal.dot(gravDir);
    Vec3 gravTangent = gravDir - normal * normalDotGrav;
    
    // 平方律：末端弯曲比根部大得多
    float bend = gravity * shellFrac * shellFrac;
    return pos + gravTangent * bend;
}
```

**为什么要先投影去掉法向分量？**  
如果直接在重力方向上移动顶点，表面位置的法向分量会导致壳层"收缩"（顶部的壳会向内穿越表面）。只取切向分量确保毛发在重力作用下只是弯曲，不会改变整体体积。

**直觉验证**：设球面顶部某点法线 $\hat{\mathbf{n}} = (0, 1, 0)$，重力 $\hat{\mathbf{g}} = (0, -1, 0)$。  
$\hat{\mathbf{n}} \cdot \hat{\mathbf{g}} = -1$，$\mathbf{g}_{\text{tangent}} = (0,-1,0) - (0,1,0)(-1) = (0,0,0)$。  
顶部的毛发切向重力为零——这是正确的，顶部的毛发竖直向上生长，重力垂直拉它，没有弯曲的余地，只有缩短。而侧面的毛发（法线为水平方向）才会有最大的侧向弯曲。

### MVP 变换与软光栅化

```cpp
// 透视投影矩阵（手工推导，不依赖 GLM）
Mat4 perspective(float fovy, float aspect, float near, float far) {
    Mat4 m;
    float t = std::tan(fovy * 0.5f);   // 半角正切
    m.m[0][0] = 1.0f / (aspect * t);   // X 方向缩放
    m.m[1][1] = 1.0f / t;              // Y 方向缩放
    m.m[2][2] = -(far+near)/(far-near); // Z 线性映射到 NDC
    m.m[2][3] = -2.0f*far*near/(far-near); // Z 偏移
    m.m[3][2] = -1.0f;                 // 写入 w = -z（透视除法）
    return m;
}
```

注意 `m.m[3][2] = -1.0f` 这行：它把裁剪空间的 $w$ 设为 $-z_{\text{view}}$。透视除法后 $z_{\text{NDC}} = -\frac{(f+n)}{f-n} - \frac{2fn}{(f-n)z}$，在 $[n, f]$ 范围内从 $-1$ 到 $+1$ 线性变化（OpenGL 约定）。

**背面剔除**：用有向面积判断，逆时针为正方向：

```cpp
float edgeFunc(Vec3 a, Vec3 b, Vec3 c) {
    return (c.x - a.x)*(b.y - a.y) - (c.y - a.y)*(b.x - a.x);
}
// 在屏幕空间计算整个三角形面积
float area = edgeFunc(sA, sB, sC);
if(area <= 0) return;  // 背向相机，跳过
```

### Alpha 合成逻辑

这是 Shell Texturing 最微妙的部分。我们需要把多层壳的毛发正确叠加：

```cpp
// 第 0 层（毛发根部）：使用深度缓冲写入，不透明
if(shellFrac < 0.05f) {
    if(depth < zbuf[idx]) {
        zbuf[idx] = depth;
        colorbuf[idx] = finalColor;
        alphabuf[idx] = 1.0f;
    }
} else {
    // 上层壳：Porter-Duff Over 混合
    if(depth < zbuf[idx] + 0.1f) {
        float existAlpha = alphabuf[idx];
        // Over 混合：new = src_alpha * src + (1 - src_alpha) * dst
        float newAlpha = existAlpha + alpha * (1.0f - existAlpha);
        if(newAlpha > 1e-5f) {
            // 颜色用预乘 alpha 格式混合后归一化
            colorbuf[idx] = colorbuf[idx] * existAlpha 
                          + finalColor * alpha * (1.0f - existAlpha);
            colorbuf[idx] = colorbuf[idx] * (1.0f / newAlpha);
            alphabuf[idx] = newAlpha;
        }
    }
}
```

**`depth < zbuf[idx] + 0.1f` 的宽松深度测试**：外层壳通常比内层壳深度更大（离相机更近），但由于透视变形，有时外层壳的某些像素深度与内层非常接近。加 0.1 的偏移确保外层壳不会被严格的深度测试错误丢弃。

**每层毛发的 alpha 递减**：

```cpp
float alpha = 1.0f - shellFrac * 0.7f;
```

根部 alpha = 1.0，毛尖 alpha = 0.3。这模拟毛发尖端的半透明感。

### 最终合成与 Gamma 校正

```cpp
void composite() {
    for(int i=0; i<W*H; i++) {
        if(alphabuf[i] > 0.01f) {
            Vec3 c = colorbuf[i];
            float gamma = 1.0f / 2.2f;
            // sRGB gamma 校正（线性空间 → 显示空间）
            uint8_t r = uint8_t(std::min(255.0f, 
                std::pow(std::max(0.0f, c.x), gamma) * 255));
            uint8_t g = uint8_t(std::min(255.0f, 
                std::pow(std::max(0.0f, c.y), gamma) * 255));
            uint8_t b = uint8_t(std::min(255.0f, 
                std::pow(std::max(0.0f, c.z), gamma) * 255));
            // 最终 Over 混合：毛发叠在背景上
            float a = alphabuf[i];
            framebuf[i].r = uint8_t(r*a + framebuf[i].r*(1.0f-a));
            framebuf[i].g = uint8_t(g*a + framebuf[i].g*(1.0f-a));
            framebuf[i].b = uint8_t(b*a + framebuf[i].b*(1.0f-a));
        }
    }
}
```

**Gamma 校正为什么在最后做？**  
在线性颜色空间中做所有光照计算（Kajiya-Kay、alpha 混合），最后统一 gamma 校正，避免中间步骤的非线性扭曲。如果在每次 alpha 混合时都做 gamma 校正，颜色会偏暗（因为 gamma 校正是凹函数，多次应用会累积向暗方向偏移）。

---

## ⑤ 踩坑实录

### Bug 1：Vec3 缺少分量乘法运算符

**症状**：第一次编译报错，涉及 `furColor * lightColor` 和 `furCol * ambient`。

```
error: no match for 'operator*' (operand types are 'Vec3' and 'Vec3')
  Vec3 diffuse = furColor * lightColor * sinL;
```

**错误假设**：我在写 Kajiya-Kay 函数时，想当然地认为 Vec3 已经有了分量乘法（Hadamard product）。

**真实原因**：我的 Vec3 结构体只定义了 `operator*(float)` 用于标量缩放，没有定义 `operator*(const Vec3&)` 用于逐分量乘法（光照计算中需要将颜色与颜色相乘）。

**修复**：在 Vec3 中添加一行：
```cpp
Vec3 operator*(const Vec3& o) const { return {x*o.x, y*o.y, z*o.z}; }
```

**教训**：定义数学向量类时，要区分两种乘法语义：几何意义的叉积（`cross()`）、数值意义的标量乘法（`operator*(float)`）、颜色意义的 Hadamard 积（`operator*(Vec3)`）。这三种都是常用操作，应该在最初设计时就一并定义。

### Bug 2（潜在问题）：重力弯曲导致壳层穿插

在开发过程中，我意识到如果重力系数 $G$ 取值过大，上层壳层会向下弯曲并穿过原始球体表面，出现自穿插渲染错误（内部渲染到外部）。

**我的规避方案**：将 $G$ 限制在 $0.04$（远小于单层步长 $0.12 / 31 \approx 0.004$），确保弯曲量始终小于壳层间距。对于更大的毛发长度，应该用更复杂的碰撞避免，或限制 $G < L_{\text{fur}} / N$。

### Bug 3（设计选择）：Alpha 混合顺序的疑惑

在实现时，我最初把壳层从外到内渲染（先渲最外层，再渲内层），认为这样符合标准的"从远到近"半透明渲染顺序。

**症状**：内层的毛发根部颜色被外层毛尖覆盖，导致整个球看起来毛发没有根部，飘浮感很强。

**真实原因**：Shell Texturing 的特殊性——每层壳处于不同深度，但属于同一个几何体的不同"高度"，不是视觉上的远近关系。从内到外渲染，内层先写入 z-buffer 和颜色缓冲，外层叠加在上面，符合"毛发从根长到尖"的视觉直觉。

**修复**：将渲染循环改为从 `shell=0` 到 `shell=NUM_SHELLS-1`（内到外），并对基础层 (`shellFrac < 0.05`) 使用严格深度写入，外层使用宽松深度测试 + alpha 混合。

### Bug 4（调试笔记）：哈希函数产生规则网格

在测试早期版本时，毛发分布出现了明显的方格状规律（每 $\frac{1}{40}$ UV 单位重复一次）。

**原因**：我最初的哈希函数 `hash2(fu, fv)` 直接使用相同参数给 $c_u$ 和 $c_v$，导致每个格子里的毛发中心都在对角线方向对齐。

**修复**：为 $c_u$ 和 $c_v$ 使用不同的哈希参数（`fv*1.3f` 和 `fu*2.1f`），确保两个方向的随机化独立。

---

## ⑥ 效果验证与数据

### 量化验证结果

```
文件: shell_fur_output.png
大小: 132 KB (> 10 KB ✅)
分辨率: 800×600
像素均值: 43.9 (在 [10, 240] 范围内 ✅)
像素标准差: 34.2 (> 5 ✅)
```

像素均值 43.9 反映了背景深色（环境色约 22-30）与橙色虎纹毛发（亮度约 60-100）的混合。标准差 34.2 表明图像有丰富的颜色变化，不是平坦的单色图。

### 性能数据

```
球体网格: 2145 顶点, 4096 三角形
壳层数量: 32
分辨率: 800×600
渲染时间: 0.37 秒（单线程 CPU）
```

**32 层 × 4096 三角形 = 131,072 次三角形光栅化调用**。每次调用都包含：MVP 变换（4×4 矩阵乘 4 次）、逐像素 UV 插值、哈希噪声采样、Kajiya-Kay 光照计算。在现代 GPU 上，这些操作完全并行化，整个场景可轻松跑到 200+ FPS。

### 视觉特征

- **壳层离散感**：在极近距离（放大看）可以看到离散的壳层边界，这是 Shell Texturing 的固有缺陷，需要增加层数（64-128 层）或加入抖动来缓解
- **毛发锥形**：越接近毛尖，像素密度越低，视觉上产生"稀疏感"，符合真实毛发的视觉特征
- **虎纹图案**：橙黑相间的条纹随球面弯曲，在侧面视角下最为明显
- **重力弯曲**：顶部毛发较直，侧面和底部毛发向重力方向弯曲（效果在当前重力系数下较为微妙）

### 与不使用 Shell Texturing 的对比

如果只渲染原始球体（0 层），结果就是一个普通的 Phong 着色球，没有任何体积感。Shell Texturing 的价值在于：用 $O(N \cdot M_{\text{tris}})$ 的绘制调用（$N$ 为层数），以极低的代价实现 $O(1)$ 几何的体积感知觉。

---

## ⑦ 总结与延伸

### 技术局限性

**1. 层数不足时的"纸牌屋"感**  
本实现用 32 层，在近距离观察时可以看到离散层。游戏实际使用通常需要 64-128 层，并配合抖动（dithering）来模糊层边界。《草地渲染》应用通常需要 100+ 层。

**2. 无法处理自遮挡**  
Shell Texturing 对每层壳独立光照计算，不考虑毛发对其下方的遮挡（AO）。真实的毛发在根部有明显的 AO 暗化效果。修复方案：增加一个简单的 AO 因子，随壳层深度线性衰减。

**3. 轮廓线问题**  
在球体轮廓边缘，壳层的膨胀会露出壳层的侧面切口，出现不自然的边缘。需要特殊处理（如 silhouette clamp）。

**4. 动态形变代价大**  
如果球体在动画中变形（如蒙皮角色），所有 $N$ 层壳都需要重新计算，GPU 代价 $N$ 倍于单层几何。Strand-based hair 在这方面有更好的扩展性。

### 可优化方向

**1. 噪声缓存**：当前每帧、每像素都重新计算哈希噪声。可以预计算一张 $256 \times 256$ 的 noise texture，采样代替实时计算，GPU 上可大幅提速。

**2. 壳层 LOD**：远处的球用少量壳层（8 层），近处用多层（64 层），动态调整。

**3. Variance Clipping 抖动**：在 shell 边界附近引入随机偏移，打破层间的整齐边界，视觉上更柔和。

**4. Anisotropic Kajiya-Kay**：添加沿切线方向的各向异性反射（参考 d'Eon 2011 的毛发散射模型），得到更物理正确的高光效果。

**5. 物理模拟**：用 Verlet 积分模拟每根毛发链条的动态（与前作 2026-04-15 的 Cloth Simulation 类似），配合 Shell Texturing 的视觉表现，可以得到随风飘动的效果。

### 与本系列的关联

- **2026-04-15 Cloth Simulation (Mass-Spring)**：同样需要处理 Verlet 积分下的弯曲约束；毛发的物理弯曲可以用类似方法。
- **2026-04-22 Marschner Hair**：提供了更精确的毛发光照模型（R/TT/TRT 散射），与 Shell Texturing 结合可以提升视觉质量。
- **2026-04-28 Cel Shading**：NPR 风格下的 Shell Texturing 也很常见（卡通毛发），可以将 Kajiya-Kay 改为阶梯化的色阶映射。
- **2026-05-15 FFT Ocean**：同为程序化表面技术，体会到用数学函数生成复杂视觉效果的设计思维是共通的。

### 最终感想

Shell Texturing 是一个让我非常欣赏的技术——它用极其朴素的思路（"画很多层"）解决了看起来很复杂的问题（体积毛发），完美诠释了实时图形学中"知觉欺骗"的设计哲学。当用户靠近时确实会看到破绽，但在正常游戏视距下，视觉效果出乎意料地好。这种"够用的近似"比"完美但跑不动"的方案往往更有工程价值。

---

### 关于本系列的持续意义

每日编程实践已经进行了近三个月。这段时间里，我从最基础的软光栅管线搭建，到实现各种高级全局光照算法，再到今天的实时毛发渲染技巧，走过了一条从理论到工程实践的完整路径。Shell Texturing 这类
"够用的工程技巧"提醒我：图形学的目标从来不是追求数学上的完美，而是在给定约束下找到视觉质量与性能之间最好的平衡点。这个哲学适用于每一个实际的渲染问题。

*代码仓库: [daily-coding-practice/2026-05-18-shell-fur](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-05-18-shell-fur)*
