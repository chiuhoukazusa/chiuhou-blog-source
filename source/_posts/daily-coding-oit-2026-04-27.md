---
title: "每日编程实践: 顺序无关透明度渲染 (OIT) — Weighted Blended OIT 完整实现"
date: 2026-04-27 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 透明渲染
  - 实时渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-27-oit/oit_output.png
---

## 一、背景与动机

### 透明渲染的历史痛点

在实时图形渲染中，透明物体的正确渲染是一个长期困扰开发者的难题。传统方法——**画家算法（Painter's Algorithm）**——要求将所有透明物体按照从远到近的顺序排序，然后逐层叠加。这个方法直觉上很自然，就像画家先画背景再画前景。

但问题很快就暴露出来：**当多个透明物体互相穿插时，不存在全局一致的排序**。

想象一下这个场景：一个红色半透明球体，穿过一块蓝色半透明玻璃板。在左侧，球体在板的前方；在右侧，球体在板的后方。这时无论你按什么顺序绘制，都必然在某些像素上出现错误的颜色混合顺序。

这就是著名的**透明度排序问题（Transparency Ordering Problem）**。

### 工业界的真实痛点

在游戏开发和实时渲染中，这个问题的影响是非常实际的：

1. **植被渲染**：树叶是半透明的，大量树叶互相交叉穿插，在复杂森林场景中逐像素排序开销极大。
2. **粒子系统**：烟雾、火焰、爆炸效果由大量半透明粒子组成，粒子之间频繁遮挡关系变化。
3. **玻璃和晶体材质**：多个玻璃物体叠放时，排序问题直接导致渲染错误。
4. **透明UI元素**：全屏特效层、HUD元素的混合。
5. **毛发和细小几何体**：头发丝、草叶等细微几何往往通过半透明纹理实现。

传统排序方案在上述场景中要么性能太差（每帧对所有三角形排序），要么出现视觉 artifact（固定排序）。

### Weighted Blended OIT 的诞生

2013年，Morgan McGuire 和 Louis Bavoil 在 SIGGRAPH 上发表了论文 *"Weighted Blended Order-Independent Transparency"*，提出了一种简单却有效的近似解法。

核心思想是：**不追求完美的物理正确性，而是给出一个视觉上可接受的近似结果，同时保证 GPU 友好、无需排序、支持多遍混合**。

这个方案被 Unity HDRP、Unreal Engine、Godot 4 等主流引擎采纳，成为实时渲染中透明物体处理的标准参考方案之一。

---

## 二、核心原理

### 2.1 标准 Alpha 混合回顾

在介绍 OIT 之前，先回顾标准 alpha 混合的数学：

对于一个不透明背景颜色 `C_bg` 和一个透明前景颜色 `C_src`（alpha 值为 `α`），混合结果为：

```
C_out = C_src * α + C_bg * (1 - α)
```

这个式子的直觉解释：最终颜色是前景颜色和背景颜色的线性插值，插值系数是 alpha 值。当 α=1 时完全遮挡背景，当 α=0 时完全透明。

对于多层透明物体，假设有 N 个透明层，从近到远分别是 `C_1, α_1`, `C_2, α_2`, ..., `C_N, α_N`，背景为 `C_bg`：

```
C_out = C_1*α_1 
      + C_2*α_2*(1-α_1) 
      + C_3*α_3*(1-α_1)*(1-α_2) 
      + ... 
      + C_bg*(1-α_1)*(1-α_2)*...*(1-α_N)
```

这就是**透射率乘积（Revealage）**的来源：每一层的贡献被所有更近层的不透明度所衰减。

**关键约束**：这个求和式要求 `C_i` 按照从近到远排序。这正是传统方法的瓶颈所在。

### 2.2 Weighted Blended OIT 的推导

WBOIT 的核心洞察是：**大多数情况下，我们并不需要严格的物理正确混合，只需要一个"看起来合理"的近似**。

论文提出将透明层的混合近似为加权平均：

**加权累积缓冲（Accumulation Buffer）**：

```
accum.rgb = Σ_i (C_i * α_i * w_i)
accum.a   = Σ_i (α_i * w_i)
```

**透射率（Revealage）**：

```
reveal = Π_i (1 - α_i)
```

**合成公式**：

```
C_final = (accum.rgb / accum.a) * (1 - reveal) + C_background * reveal
```

来拆解这个公式：

- `(accum.rgb / accum.a)`：这是所有透明层的**加权平均颜色**。通过权重 `w_i`，更重要的层（通常是更近、更不透明的层）对平均值贡献更多。
- `(1 - reveal)`：这是总体不透明度，表示所有透明层叠加起来遮住背景的程度。`reveal = Π(1-α_i)` 是透过所有透明层能看到背景的概率。
- `C_background * reveal`：背景颜色乘以能看穿所有层的概率。

直觉上：最终颜色 = 平均透明颜色 × 总遮挡度 + 背景颜色 × 总透明度。

### 2.3 权重函数 w(z, α) 的设计

权重函数是 WBOIT 质量的关键。它应该满足：
- **距离相机越近（z越小），权重越大**：前景层对视觉的影响通常更大
- **alpha越大（越不透明），权重越大**：不透明层应该更主导颜色
- **数值稳定性**：避免浮点溢出

论文中提供了几个版本，我们使用的版本是：

```cpp
float oitWeight(float z, float alpha) {
    // z 已归一化到 [0, 1]，0 为相机近平面，1 为远平面
    float dz = z;
    float w = alpha * max(1e-2, min(3e3, 
                  0.03 / (1e-5 + dz*dz*dz*dz)));
    return w;
}
```

公式解读：
- `dz*dz*dz*dz`：z 的四次方，使权重随深度快速衰减（距离越远权重越小）
- `0.03 / (1e-5 + ...)` 的形式：分母中加 `1e-5` 防止除以零，`0.03` 是缩放系数
- `min(3e3)` 和 `max(1e-2)`：防止权重过大或过小，提高数值稳定性
- 最后乘以 `alpha`：更不透明的片段权重更大

这个函数在 z=0（近处）时值很大，在 z=1（远处）时值很小，符合直觉——近处的透明物体对最终颜色影响更大。

### 2.4 WBOIT 的局限性

WBOIT 不是完美的物理正确方案，它有以下已知局限：

1. **高度重叠时颜色失真**：当许多高透明度（α接近1）的层叠加时，加权平均可能与真实混合差异较大。
2. **α接近0的细薄层处理较差**：权重函数在低α时可能不稳定。
3. **不支持自身折射**：纯粹是颜色混合，不模拟折射、散焦等效果。

尽管有这些局限，在大多数游戏场景（烟雾、玻璃、粒子等）中，视觉效果已经足够令人满意，而性能开销远低于精确排序算法。

---

## 三、实现架构

### 3.1 整体渲染管线

本项目实现了一个完整的软光栅化 OIT 渲染器，渲染管线分为三个 Pass：

```
输入: 不透明几何体列表 + 透明几何体列表 + 相机参数

Pass 1: 不透明渲染
  ├── 顶点变换 (Model → View → Projection → NDC → Screen)
  ├── 背面剔除 (基于屏幕空间三角形 winding 判断)
  ├── 三角形光栅化
  ├── Blinn-Phong 着色
  └── 深度测试 + 深度写入 (opaqueFb.depth)
  输出: opaqueFb.color, opaqueFb.depth

Pass 2: 透明 OIT 累积
  ├── 顶点变换 (同 Pass 1)
  ├── 双面渲染 (不做背面剔除)
  ├── 三角形光栅化
  ├── 深度测试 (对比 opaqueFb.depth，透明物体必须在不透明前景前面)
  ├── Blinn-Phong 着色
  ├── 权重计算 w(z, alpha)
  ├── 累积: oit.accum += litColor * alpha * w
  ├── 累积: oit.accum.a += alpha * w
  └── 透射率: oit.reveal *= (1 - alpha)
  输出: oit.accum, oit.reveal

Pass 3: 合成
  ├── 对每个像素:
  │     avgColor = oit.accum.rgb / oit.accum.a
  │     reveal = oit.reveal
  │     finalColor = mix(bg, avgColor, 1 - reveal)
  └── 输出最终图像

输出: finalPixels → PPM → PNG
```

### 3.2 关键数据结构

**OITBuffers（核心缓冲区）**：

```cpp
struct OITBuffers {
    std::vector<Vec4> accum;   // .xyz = 加权颜色累积, .w = 加权 alpha 累积
    std::vector<float> reveal; // 透射率乘积 Π(1-α_i)，初始为 1.0
};
```

设计理由：
- `accum` 用 `Vec4` 存储两类信息：RGB 累积和 alpha 累积，这样可以一次遍历完成
- `reveal` 用单独的 float 数组，因为它是乘法累积（不是加法），需要初始化为 1.0
- 两个数组大小都是 W×H（每像素一个元素），这是 GPU 端 MRT（Multiple Render Targets）的 CPU 模拟

**Framebuffer（不透明缓冲区）**：

```cpp
struct Framebuffer {
    std::vector<Vec3> color;  // 最终颜色
    std::vector<float> depth; // 深度缓冲（NDC z 值）
};
```

这个深度缓冲在 Pass 1 中被写入，在 Pass 2 中被读取用于透明物体的深度测试。

### 3.3 几何预处理

所有几何体在构建时就被变换到**世界空间**：

```cpp
Mesh buildMesh(const std::vector<Triangle>& src, const Mat4& model,
               Vec3 color, float alpha, bool opaque) {
    // 将模型空间三角形 × model 矩阵 → 世界空间
    // 法线也做相同变换（假设无非均匀缩放时，法线矩阵 = 模型矩阵的旋转部分）
}
```

在光栅化时，再乘以 `VP = proj * view` 变换到裁剪空间。这样把模型变换计算从每帧每像素移到了构建阶段，减少运行时计算量。

---

## 四、关键代码解析

### 4.1 顶点变换与三角形投影

```cpp
ScreenTri projectTriangle(const Triangle& tri, const Mat4& VP) {
    ScreenTri st;
    st.valid = true;
    for (int i = 0; i < 3; i++) {
        // 世界空间坐标 → 裁剪空间
        Vec4 clip = VP * Vec4(tri.v[i].pos, 1.0f);
        
        // 防止 w 为 0（相机后方或在近平面上的退化情况）
        if (std::abs(clip.w) < 1e-7f) { st.valid = false; return st; }
        
        // 透视除法：裁剪坐标 → NDC（Normalized Device Coordinates）
        st.ndc[i] = clip.xyz() / clip.w;
        
        // 简单视锥体裁剪（只检查深度方向，x/y 方向用 bbox 裁剪）
        if (st.ndc[i].z < -1.0f || st.ndc[i].z > 1.0f) { st.valid = false; return st; }
        
        // NDC → 屏幕像素坐标
        // NDC.x ∈ [-1,1] → screen.x ∈ [0, W-1]
        st.sx[i] = (st.ndc[i].x * 0.5f + 0.5f) * (W - 1);
        // 注意 y 轴翻转！NDC.y 向上为正，屏幕像素 y 向下为正
        st.sy[i] = (1.0f - (st.ndc[i].y * 0.5f + 0.5f)) * (H - 1);
        
        // 世界空间法线直接存储（用于 Blinn-Phong 光照）
        st.norm[i] = tri.v[i].normal;
    }
    return st;
}
```

这里最重要的是 **y 轴翻转**：`sy = (1 - (ndy*0.5+0.5)) * H`。这是因为 OpenGL/NDC 中 y 轴向上（+1 在顶部），而屏幕像素坐标 y 轴向下（0 在顶部，H-1 在底部）。如果不翻转，整个渲染结果会上下颠倒。

### 4.2 屏幕空间 Winding 判断与 Edge Function

这是整个 rasterizer 中最容易出 bug 的地方：

```cpp
// Edge Function: 判断点 p 在有向边 a→b 的哪一侧
// 返回值为正：p 在 a→b 的左侧
// 返回值为负：p 在 a→b 的右侧
inline float edgeFunc(float ax, float ay, float bx, float by, float px, float py) {
    return (px - ax) * (by - ay) - (py - ay) * (bx - ax);
}
```

这个函数本质上是 2D 叉积：`(b-a) × (p-a)` 的 z 分量。

**关键：屏幕空间的 winding 与 NDC 相反！**

由于 y 轴翻转，NDC 空间中逆时针（CCW）的三角形，投影到屏幕空间后变成顺时针（CW）。这意味着：

```cpp
float area = (sx[1]-sx[0])*(sy[2]-sy[0]) - (sy[1]-sy[0])*(sx[2]-sx[0]);
// area < 0 表示屏幕空间 CCW（即 NDC 空间 CW，即：从相机看是背面）
// area < 0 即为"正面"（因为 NDC CCW → 屏幕 CW → area > 0 是相机背面）
```

经过我的调试验证：**在 y 轴翻转之后，屏幕空间 CCW（area < 0）的三角形是正面**。

更重要的是，对于任意 winding 的三角形，需要统一化 edge function 的符号：

```cpp
// signFactor 使内部像素的 barycentric 坐标始终为正
// 屏幕空间 CCW（area<0）时，ef() 对内部点返回正值 → signFactor = 1
// 屏幕空间 CW（area>0）时，ef() 对内部点返回负值 → signFactor = -1
float signFactor = (area > 0) ? -1.0f : 1.0f;

// 归一化后，内部点的 w0, w1, w2 全部 >= 0
float w0 = edgeFunc(sx[1], sy[1], sx[2], sy[2], px, py) * signFactor;
float w1 = edgeFunc(sx[2], sy[2], sx[0], sy[0], px, py) * signFactor;
float w2 = edgeFunc(sx[0], sy[0], sx[1], sy[1], px, py) * signFactor;
bool inside = (w0 >= 0 && w1 >= 0 && w2 >= 0);
```

### 4.3 不透明三角形光栅化

```cpp
void rasterizeOpaqueTriangle(const ScreenTri& st, Vec3 color, bool backfaceCull,
                              Framebuffer& fb, const Light& light, Vec3 viewDir) {
    float e1x = st.sx[1]-st.sx[0], e1y = st.sy[1]-st.sy[0];
    float e2x = st.sx[2]-st.sx[0], e2y = st.sy[2]-st.sy[0];
    float area = e1x * e2y - e1y * e2x;
    
    // 背面剔除：跳过 area > 0（屏幕空间 CW = NDC 中的背面）
    if (backfaceCull && area > 0) return;
    if (std::abs(area) < 1e-8f) return;
    
    float signFactor = (area > 0) ? -1.0f : 1.0f;
    
    // AABB bounding box 加速，只遍历三角形覆盖的像素范围
    int minX = (int)max(0.0f, min({sx[0],sx[1],sx[2]}));
    int maxX = (int)min((float)(W-1), max({sx[0],sx[1],sx[2]}));
    // ... 同理 y

    for (int py = minY; py <= maxY; py++) {
        for (int px = minX; px <= maxX; px++) {
            float w0 = edgeFunc(st.sx[1], st.sy[1], st.sx[2], st.sy[2], px, py) * signFactor;
            float w1 = edgeFunc(st.sx[2], st.sy[2], st.sx[0], st.sy[0], px, py) * signFactor;
            float w2 = edgeFunc(st.sx[0], st.sy[0], st.sx[1], st.sy[1], px, py) * signFactor;
            if (w0 < 0 || w1 < 0 || w2 < 0) continue;  // 不在三角形内
            
            float denom = w0 + w1 + w2;
            if (denom < 1e-8f) continue;
            
            // 重心坐标：三个 ef 值归一化后即为顶点权重
            float b0=w0/denom, b1=w1/denom, b2=w2/denom;
            
            // 插值深度
            float z = b0*ndc0.z + b1*ndc1.z + b2*ndc2.z;
            
            int idx = py * W + px;
            if (z >= fb.depth[idx]) continue;  // 深度测试（比较 NDC z 值）
            fb.depth[idx] = z;  // 写入深度缓冲
            
            // 插值法线并归一化
            Vec3 n = (st.norm[0]*b0 + st.norm[1]*b1 + st.norm[2]*b2).normalize();
            
            // Blinn-Phong 着色
            fb.color[idx] = shade(color, n, viewDir, light);
        }
    }
}
```

深度比较 `z >= fb.depth[idx]` 中：NDC z 值越小表示越靠近相机（NDC z ∈ [-1, 1]，-1 是近平面，1 是远平面）。因此用 `<` 更新——越小的 z 越近，应该覆盖。

### 4.4 OIT 透明片段累积

```cpp
void rasterizeTransparentTriangle(const ScreenTri& st, Vec3 color, float alpha,
                                   const Framebuffer& opaqueFb, OITBuffers& oit,
                                   const Light& light, Vec3 viewDir) {
    // 透明物体：双面渲染（不剔除背面，产生玻璃感）
    float area = (sx[1]-sx[0])*(sy[2]-sy[0]) - (sy[1]-sy[0])*(sx[2]-sx[0]);
    if (std::abs(area) < 1e-8f) return;
    float signFactor = (area > 0) ? -1.0f : 1.0f;

    for (int py = minY; py <= maxY; py++) {
        for (int px = minX; px <= maxX; px++) {
            // ... 重心坐标计算（同不透明版本）

            // 深度测试：透明片段只有在不透明几何前面才需要渲染
            // （被不透明物体遮挡的透明片段不可见）
            float z = /* 插值得到 */ ;
            if (z >= opaqueFb.depth[idx]) continue;  // 被不透明物体遮挡，跳过
            // 注意：透明物体不写入深度缓冲！这允许多个透明层叠加

            // 对法线方向（背面渲染时翻转法线）
            Vec3 n = (st.norm[0]*b0 + st.norm[1]*b1 + st.norm[2]*b2).normalize();
            if (signFactor < 0) n = n * -1.0f;  // 背面法线翻转

            Vec3 litColor = shade(color, n, viewDir, light);

            // NDC z ∈ [-1,1] → linearZ ∈ [0,1]
            float linearZ = (z + 1.0f) * 0.5f;
            
            // 计算 OIT 权重
            float w = oitWeight(linearZ, alpha);

            // ★ 核心：OIT 累积 ★
            oit.accum[idx].x += litColor.x * alpha * w;  // 加权颜色
            oit.accum[idx].y += litColor.y * alpha * w;
            oit.accum[idx].z += litColor.z * alpha * w;
            oit.accum[idx].w += alpha * w;               // 加权 alpha

            // 透射率：乘法累积，顺序无关！这正是 OIT 的魔法所在
            // 传统方法：只有严格排序才能正确计算透射率
            // WBOIT：每层独立乘以 (1-α)，顺序无关（因为乘法满足交换律）
            oit.reveal[idx] *= (1.0f - alpha);
        }
    }
}
```

`reveal` 的乘法 `*= (1 - alpha)` 是整个算法的精妙之处：`Π(1-α_i)` 无论 i 的顺序如何都相同（乘法交换律），因此这个值天然就是顺序无关的！

### 4.5 合成 Pass

```cpp
Vec3 compositeOIT(int idx, const Framebuffer& opaqueFb, const OITBuffers& oit) {
    Vec3 bg = opaqueFb.color[idx];  // 不透明背景颜色
    float revealage = oit.reveal[idx];  // 透射率（能看穿所有透明层的比例）

    // 如果没有透明层（accum.w ≈ 0），直接返回背景
    float accumW = oit.accum[idx].w;
    if (accumW < 1e-5f) return bg;

    // 计算加权平均颜色：除以权重总和归一化
    Vec3 accumColor = {oit.accum[idx].x, oit.accum[idx].y, oit.accum[idx].z};
    Vec3 avgColor = accumColor / accumW;

    // 最终合成：
    // - (1 - revealage) 是总遮挡度（所有透明层叠加遮挡背景的程度）
    // - revealage 是透过所有透明层能看到背景的比例
    // - mix(bg, avgColor, 1 - revealage)
    //   = avgColor * (1 - revealage) + bg * revealage
    return mix(bg, avgColor, 1.0f - revealage);
}
```

`mix(bg, avgColor, t)` 中 `t = 1 - revealage`：当 revealage=0（完全不透明）时，t=1，全部显示透明层颜色；当 revealage=1（完全透明）时，t=0，全部显示背景颜色。

### 4.6 Blinn-Phong 光照模型

```cpp
Vec3 shade(Vec3 albedo, Vec3 normal, Vec3 viewDir, const Light& light) {
    Vec3 n = normal.normalize();
    
    // Lambertian 漫反射
    float NdotL = max(0.0f, n.dot(light.dir));
    Vec3 diffuse = albedo * light.color * NdotL;
    
    // 环境光（防止暗面全黑）
    Vec3 ambient = albedo * light.ambient;
    
    // Blinn-Phong 镜面高光：使用半向量 h = normalize(L + V)
    // 相比 Phong 模型（用反射向量），Blinn-Phong 在掠射角时更自然
    Vec3 h = (light.dir + viewDir).normalize();
    float spec = pow(max(0.0f, n.dot(h)), 32.0f) * 0.4f;
    Vec3 specular = light.color * spec;
    
    return clamp3(ambient + diffuse + specular, 0, 1);
}
```

光照计算在世界空间进行。`light.dir` 是从片段指向光源的归一化向量，`viewDir` 是从片段指向相机的归一化向量。

---

## 五、踩坑实录

### Bug 1：透明像素全为 0（最关键的 Bug）

**症状**：程序运行正常，输出 `Transparent pixels: 0 / 480000 (0.0%)`，图像看不到任何透明效果。

**初步排查**：先检查了深度测试逻辑、透明几何体的位置是否在不透明物体前面。调试工具确认各个测试点的 NDC 坐标都在合理范围（~0.97），深度测试条件应该通过。

**发现问题**：写了一个最小调试程序，手动测试一个三角形的三个顶点屏幕坐标，然后在三角形中心点测试 edge function 值。

```
area = -29422 (area < 0，screenspace CCW)
centroid 的 ef 结果: w0=9815, w1=9815, w2=9815 (全为正！)
```

原代码逻辑：`signFactor = (area > 0) ? 1.0f : -1.0f`，于是对 area < 0 的三角形，`signFactor = -1`。

这意味着：ef 值（正的 9815） × signFactor（-1） = **-9815**，判断 `w >= 0` 失败，认为点在三角形外面！

**真实原因**：我混淆了屏幕空间 winding 与 edge function 符号的关系。

在屏幕空间（y轴向下）：
- **CCW 三角形（area < 0）**：内部点的 ef 值**为正**
- **CW 三角形（area > 0）**：内部点的 ef 值**为负**

这与通常在 y向上 坐标系中的直觉**相反**！因为 y 轴翻转导致了 winding 的定义颠倒。

正确的 signFactor：
```cpp
// CCW (area<0): ef 为正 → signFactor = 1，内部点仍 >= 0
// CW  (area>0): ef 为负 → signFactor = -1，乘后变正，内部点 >= 0
float signFactor = (area > 0) ? -1.0f : 1.0f;  // 注意：和直觉相反！
```

**修复**：交换 signFactor 的条件，立即解决，透明像素从 0 变为 96886（20.2%）。

---

### Bug 2：lookAt 函数返回类型错误

**症状**：编译报错 `could not convert 'r' from 'Vec3' to 'Mat4'`

**原因**：重写代码时，保留了旧版 `lookAt` 函数（从历史代码复制），该函数末尾 `return r` 中的 `r` 是一个 `Vec3` 而不是 `Mat4`。

```cpp
// 错误版本（保留了旧代码的残留）：
Mat4 lookAt(Vec3 eye, Vec3 center, Vec3 up) {
    Vec3 f = (center - eye).normalize();
    Vec3 r = f.cross(up).normalize();  // r 是 Vec3，不是 Mat4!
    // ... 
    return r;  // 编译错误！
}
```

因为在 main() 中已经内联实现了新的 lookAt 逻辑，直接删除旧函数即可。

**教训**：大规模重写代码时，要检查是否有残留的同名函数或变量，避免命名冲突或类型不匹配。

---

### Bug 3：背面剔除方向错误

**症状**：不透明物体（背景墙、柱子）要么完全不渲染，要么正背面都渲染（无剔除）。

**原因**：最初的背面剔除条件是 `if (backfaceCull && area <= 0) return`，基于"area>0 是正面"的错误假设。

由于 y 轴翻转，正确逻辑是：
- area < 0：屏幕空间 CCW = NDC 空间中面向相机的正面
- area > 0：屏幕空间 CW = NDC 空间中背对相机的背面

```cpp
// 错误：
if (backfaceCull && area <= 0) return;  // 剔除了正面！

// 正确：
if (backfaceCull && area > 0) return;   // 剔除 area>0 的背面
```

**教训**：涉及坐标系翻转时，不能凭直觉推断 winding 方向，必须用具体数值（比如一个已知的正面三角形的坐标）来验证。

---

### Bug 4：透明物体用了错误的 MVP 矩阵（早期版本）

**症状**（早期版本）：透明物体完全不可见，但不透明物体正常。

**原因**：早期版本中，几何体存储的是**模型空间坐标**，MVP = proj × view × model。但透明物体的 `model` 矩阵使用了 `proj * view`（没有 model），导致场景中的透明面板被投影到错误位置，完全超出屏幕范围。

**修复**：改为将所有几何体在构建时就用 `buildMesh(tris, model, ...)` 预变换到世界空间，光栅化时统一用 `VP = proj * view` 即可。

---

## 六、效果验证与数据

### 6.1 渲染结果

![OIT渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-27-oit/oit_output.png)

场景包含：
- **背景**：深蓝到暖棕的渐变（天空到地面）
- **不透明物体**：背景墙、地板、中央基座、左右柱子
- **透明物体**：6个彩色面板（红/绿/蓝/黄/青/品红）+ 2个玻璃箱 + 1块蓝色薄板，共9个透明网格

### 6.2 量化统计数据

运行程序输出的统计信息：

```
Statistics:
  Transparent pixels: 96886 / 480000 (20.2%)
  Average revealage: 0.8497
  Opaque meshes: 5, Transparent meshes: 9
```

- **透明像素占比**：20.2%，说明 OIT 效果覆盖了图像的五分之一区域
- **平均透射率 0.8497**：意味着平均来看，背景能透过透明层的比例约为 85%，这对于多个半透明层叠加来说是合理的

### 6.3 像素采样验证

```python
Image size: 800x600
Pixel mean: 53.5  std: 28.2
Min: 14  Max: 178
Channel means: R=53.5 G=55.2 B=51.9

Top 50 rows: R=21.2 G=25.7 B=37.9  (蓝色调天空)
Bot 50 rows: R=49.4 G=39.8 B=33.0  (暖色调地面)

Colored pixels (saturation>20): 20.4%  (彩色透明层)
```

- **均值 53.5**：在 [10, 240] 范围内 ✅（非全黑/全白）
- **标准差 28.2**：> 5 ✅（图像有丰富内容变化）
- **坐标系验证**：顶部偏蓝（天空）、底部偏暖（地面） ✅（无上下颠倒）
- **文件大小**：PNG 15.9KB > 10KB ✅

### 6.4 与无透明版本的对比

如果删除 Pass 2（OIT 累积）和 Pass 3（合成），只保留不透明几何：

| 指标 | 无透明 | OIT 透明 |
|------|--------|---------|
| 彩色像素 | ~0% | 20.4% |
| 图像标准差 | ~10 | 28.2 |
| 透明像素 | 0 | 96,886 |

OIT 使图像标准差提升近3倍，带来了丰富的透明颜色混合效果。

### 6.5 性能说明

在 800×600 分辨率下，完整渲染时间约 1-3 秒（CPU 软光栅化，无 SIMD 优化）。

在实际 GPU 实现中：
- Pass 1（不透明）：单次 forward pass，成本与普通渲染相同
- Pass 2（OIT 累积）：单次 forward pass，每片段额外执行权重计算和原子加操作
- Pass 3（合成）：全屏 quad pass，成本极低

相比精确排序方案（需要 GPU 排序或逐像素链表），WBOIT 的额外开销基本可以忽略不计。

---

## 七、总结与延伸

### 7.1 技术成果

本次实现了完整的 Weighted Blended OIT 渲染器，包括：
- 三 Pass 渲染管线（不透明 → OIT 累积 → 合成）
- OIT 权重函数（基于深度和 alpha）
- 正确的屏幕空间光栅化（含 y 轴翻转后的 winding 判断）
- 双面渲染（透明物体无背面剔除）

### 7.2 WBOIT 的局限性

1. **高度重叠时失真**：5+ 个高 alpha（>0.8）层重叠时，加权平均可能偏离真实混合。
2. **权重函数调参**：不同场景可能需要不同的权重函数参数，没有通用最优解。
3. **不处理折射**：纯颜色混合，无法模拟玻璃的折射偏移效果（需要额外 Pass）。
4. **不支持自定义混合顺序**：无法实现"某些特定物体必须最后渲染"的需求。

### 7.3 可优化方向

1. **Per-pixel Linked List（PPLL）**：精确 OIT，每个像素存储一个透明片段链表，排序后合成。质量最高，内存开销大。
2. **Depth Peeling**：多 Pass 逐层剥离，每 Pass 渲染"当前最近的未处理透明层"，精确但需要多个 Pass。
3. **Moment-Based OIT（MBOIT）**：使用统计矩重建透射率函数，比 WBOIT 精度更高，比 PPLL 内存更小。
4. **Stochastic Transparency**：基于随机采样的 OIT，结合 TAA 可以收敛到精确结果。

### 7.4 与本系列的关联

- **Day 25（SDF Font Rendering）**：后处理合成技术
- **Day 21（Bloom & Lens Flare）**：透明效果常与 Bloom 结合，Bloom 需要对半透明物体的高亮进行特殊处理
- **Day 18（Screen Space Reflections）**：反射与透明物体的交互是高级渲染话题
- **Day 9（SPH Fluid）**：流体渲染经常需要 OIT 处理半透明水体

透明渲染是现代实时图形管线中不可或缺的部分，WBOIT 以其简单高效的特性成为了当前的工业标准近似方案。理解它的原理和局限，才能在实际项目中合理选择透明渲染策略。

---

*代码仓库：[daily-coding-practice/2026/04/04-27-oit](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-27-oit)*
