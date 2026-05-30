---
title: "每日编程实践: Geometry Shader Effects Renderer"
date: 2026-05-31 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 软光栅化
  - 几何着色器
  - 法线可视化
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-31-Geometry-Shader-Effects-Renderer/geometry_shader_output.png
---

# 每日编程实践 Day 100+：Geometry Shader Effects Renderer

今天的项目是用 **纯 C++ 软光栅化** 模拟 GPU 几何着色器（Geometry Shader）的核心效果：
**线框叠加渲染（Wireframe Overlay）**、**法线可视化（Normal Visualization）** 以及 **切线空间（TBN）可视化**。

这三种效果是图形调试、学习渲染管线不可缺少的工具，在 Unity/Unreal 的 Gizmos、RenderDoc 帧分析器、以及各类自制引擎的 Debug View 中随处可见。

![渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-31-Geometry-Shader-Effects-Renderer/geometry_shader_output.png)

---

## ① 背景与动机

### 几何着色器是什么？

在现代 GPU 渲染管线中，几何着色器（Geometry Shader，GS）位于顶点着色器和片段着色器之间，是唯一一个能够**生成新图元**的可编程阶段。

```
顶点着色器 → [曲面细分] → 几何着色器 → 光栅化 → 片段着色器
```

GS 的独特能力：
- 接收一个完整图元（点/线/三角形）以及它的所有顶点
- 可以输出 0 到多个新图元
- 能访问同一三角形的所有 3 个顶点（这是 VS 无法做到的）

### 为什么软光栅化能模拟 GS 效果？

软光栅化的光栅化循环本质上就是"逐三角形处理"——我们完全可以在这个循环里做 GS 所做的事：

1. **线框叠加**：GS 的经典用法是发射三角形的 3 条边。但在软光栅里，我们直接用**重心坐标**计算像素到最近边的距离，再用这个距离做混合——效果等价，还没有 GS 的几何放大开销。

2. **法线可视化**：GS 会在每个顶点位置生成一条"小线段"表示法线方向。我们用 Bresenham 直线算法在世界坐标投影后绘制。

3. **切线空间**：对每个面生成 TBN 三元组（Tangent、Bitangent、Normal），用三色箭头（红/绿/蓝）绘制出来。

### 工业界实际使用场景

这些效果不只是"炫技"，在实际开发中有明确用途：

| 效果 | 工具 | 用途 |
|------|------|------|
| 线框叠加 | Unity Scene View / Unreal Wireframe Mode | 检查 LOD 三角形密度、发现穿插面 |
| 法线可视化 | MeshLab / RenderDoc Vertex Inspector | 排查法线朝向错误（法线朝内导致背面剔除错误）|
| 切线空间 | Substance Painter / Unreal Material Debug | 验证法线贴图的 TBN 切线空间是否正确（UV 接缝处最易出错）|

另一个重要场景是**渲染器调试**。当你做光照时发现某个面奇怪地偏暗，第一步就是打开法线可视化看法线是否反了。切线空间可视化则是排查法线贴图"石头感消失"或"接缝处有明显错误"的标准手段。

---

## ② 核心原理

### 2.1 线框叠加：重心坐标边缘距离法

**为什么 GPU 上经典的线框需要 GS？**

传统方法是用 GS 把三角形的每条边作为独立的线段发射出去，然后用 `GL_LINES` 画。但这有两个问题：
- 线的粗细无法做抗锯齿混合
- 需要 GS，对性能有冲击（尤其在低端 GPU）

**现代方案：无 GS 线框**

论文 *Solid Wireframe* (NVIDIA 2007) 提出了一个优雅的方法：

1. 在顶点着色器，为每个顶点附上一个"边权重"向量 $(1,0,0)$、$(0,1,0)$、$(0,0,1)$
2. 光栅化时，这三个权重会被插值
3. 在片段着色器，插值后的权重向量 $(w_0, w_1, w_2)$ 中最小分量就是到最近边的距离比例

**软光栅版本**：我们用重心坐标 $(\lambda_0, \lambda_1, \lambda_2)$ 做等价计算。

对于三角形内一点 $P$，设三个顶点的屏幕坐标为 $A$、$B$、$C$，该点的重心坐标为 $(\lambda_A, \lambda_B, \lambda_C)$，则：

$$d_{\text{edge}\_BC}(P) = \frac{|\vec{BC} \times \vec{BP}|}{|\vec{BC}|}$$

即点 $P$ 到边 $BC$ 的垂直距离（像素单位）。

三条边取最小值：

$$d_{\min}(P) = \min(d_{BC}, d_{AC}, d_{AB})$$

最终线框透明度：

$$\alpha_{\text{wire}} = \left(1 - \min\left(1, \frac{d_{\min}}{w}\right)\right)^2$$

其中 $w$ 是线框宽度参数（像素单位）。取平方是为了让边缘更锐利（线性插值太柔）。

**为什么要平方？**

线性衰减 $1 - d/w$ 会让线框看起来很"模糊"，边缘有一个宽宽的灰色过渡带。平方后，大部分区域的 $\alpha$ 接近 0（看不到线框），只有靠近边的区域才急剧升高——视觉上更像真实的线条。

混合公式：

$$C_{\text{final}} = C_{\text{shade}} \cdot (1 - \alpha_w) + C_{\text{wire}} \cdot \alpha_w$$

### 2.2 法线可视化：法线颜色映射

**法线到颜色的映射**

法线向量每个分量的范围是 $[-1, 1]$，我们需要映射到颜色范围 $[0, 255]$：

$$C_{RGB} = \left(\frac{n_x + 1}{2} \cdot 255, \quad \frac{n_y + 1}{2} \cdot 255, \quad \frac{n_z + 1}{2} \cdot 255\right)$$

这就是法线贴图的颜色编码方式。

**直觉解释**：
- 法线朝 $+X$（向右）：$R = 1.0$，颜色偏红
- 法线朝 $+Y$（向上）：$G = 1.0$，颜色偏绿  
- 法线朝 $+Z$（朝向屏幕）：$B = 1.0$，颜色偏蓝
- 所以典型的法线贴图呈现蓝紫色——因为法线大多朝 $+Z$（切线空间中"正前方"），$B$ 分量接近最大值

右侧球体用这种颜色编码渲染，可以直观看到球面各处的法线方向分布。

### 2.3 法线箭头绘制：Bresenham 算法

**世界坐标 → 屏幕坐标**

法线箭头的起点是顶点/面中心 $P_{\text{world}}$，终点是 $P_{\text{world}} + N_{\text{world}} \cdot L$（$L$ 为箭头长度）。

先用 MVP 矩阵将两点投影到 NDC，再转屏幕坐标：

$$P_{\text{screen}} = \left(\frac{x_{NDC} + 1}{2} \cdot W, \quad \left(1 - \frac{y_{NDC} + 1}{2}\right) \cdot H\right)$$

注意 Y 轴翻转：NDC $+Y$ 向上，屏幕 Y 向下。

**Bresenham 直线算法**

Bresenham 算法用纯整数运算绘制直线，避免浮点除法：

```
初始误差 err = dx - dy
循环：
  绘制像素 (x, y)
  e2 = 2 * err
  if e2 > -dy: err -= dy; x += sx
  if e2 < dx:  err += dx; y += sy
```

其中 $dx = |x_1 - x_0|$，$dy = |y_1 - y_0|$，$sx$/$sy$ 是步进方向。

**直觉**：误差 $err$ 追踪"真实直线"与"当前整数位置"的偏差。当偏差超过阈值，就步进对应轴。

### 2.4 切线空间 TBN 构建

对于一个面，设面法线为 $N$，切线 $T$ 和副切线 $B$ 满足：

$$T \perp N, \quad B \perp N, \quad B = N \times T$$

且 $T$、$B$、$N$ 构成右手系。

**从法线和 Up 向量构建 T**

给定 $N$ 和世界 Up = $(0, 1, 0)$：

$$T = \text{normalize}\left(\text{Up} - N \cdot (N \cdot \text{Up})\right)$$

这是格拉姆-施密特正交化：先取 Up 方向，减去它在 N 方向的投影，得到的就是与 N 垂直的、尽量朝 Up 的切线。

$$B = N \times T$$

当 $N$ 与 Up 平行时（$|N \cdot \text{Up}| \approx 1$），此公式退化（$T$ 接近零向量）。实际中改用 Right = $(1, 0, 0)$ 作备选。

可视化：
- **红色箭头**：切线 $T$（类似于 UV 的 $U$ 方向）
- **绿色箭头**：副切线 $B$（类似于 UV 的 $V$ 方向）
- **蓝色箭头**：法线 $N$

---

## ③ 实现架构

### 整体渲染管线

```
几何体生成
  ├── UV Sphere (16×24 stacks/slices)
  └── Cube (6 face × 2 tri = 12 tri)
          │
          ▼
变换矩阵构建
  ├── Model Matrix（旋转/平移）
  ├── View Matrix（lookAt）
  └── Projection Matrix（perspective）
          │
          ▼
逐三角形处理
  ├── 顶点变换（MVP → NDC → 屏幕坐标）
  ├── 背面剔除（叉积符号判断）
  └── 光栅化（重心坐标插值）
          │
          ▼
多 Pass 叠加
  ├── Pass 1：Phong 着色（实体颜色）
  ├── Pass 2：线框叠加（重心坐标边缘距离）
  └── Pass 3（独立）：法线/切线箭头（Bresenham）
          │
          ▼
像素输出 → PNG
```

### 关键数据结构设计

**Vertex**：每个顶点携带位置、法线、UV。UV 为切线空间计算预留（本次实现中用于扩展）。

```cpp
struct Vertex {
    Vec3 pos;
    Vec3 normal;
    Vec2 uv;
};
```

**ScreenVert**：投影后的顶点，同时保存世界坐标和法线（用于 Phong 着色），以及屏幕坐标和深度（用于光栅化）。

```cpp
struct ScreenVert {
    Vec3 world;     // 世界坐标（Phong 着色用）
    Vec3 normal;    // 世界法线（Phong 着色用）
    Vec2 screen;    // 像素坐标（光栅化用）
    float depth;    // NDC 深度（Z-Buffer 用）
};
```

**Framebuffer**：双缓冲（颜色 + 深度）。深度测试使用"越小越近"约定（NDC Z 范围 [-1, 1]，越小越近于摄像机）。

```cpp
struct Framebuffer {
    std::vector<Color> color;   // RGB 颜色缓冲
    std::vector<float> depth;   // 深度缓冲（初始化为 +∞）
    
    void setPixel(int x, int y, Color c, float d);     // 有深度测试
    void blendPixel(int x, int y, Color c, float a);   // 无深度测试（叠加）
};
```

**为什么 `blendPixel` 不做深度测试？**

线框和法线箭头是调试信息，需要"永远可见"，即使被物体遮挡也要显示（类似 Gizmos 的 XRay 模式）。如果做深度测试，被球体遮挡的法线箭头就看不到了。这是设计上的权衡：本次实现选择 XRay 风格，让调试信息始终可见。

**RenderOptions**：参数化渲染选项，方便调试时切换效果。

```cpp
struct RenderOptions {
    bool wireframe   = true;
    bool showNormals = true;
    float wireWidth  = 1.2f;   // 线框宽度（像素）
    float normalLen  = 0.25f;  // 法线箭头世界空间长度
    Vec3 baseColor   = {0.7f, 0.5f, 0.3f};
};
```

### 矩阵约定

- **列主序**（Column-major）：$v' = M v$
- **右手系**（Right-handed）：$+X$ 向右，$+Y$ 向上，$+Z$ 向屏幕外
- **NDC**：$[-1, 1]^3$，$+Y$ 向上
- **屏幕坐标**：$[0, W] \times [0, H]$，$Y$ 向下（因此有翻转）

---

## ④ 关键代码解析

### 4.1 边缘距离计算

```cpp
float edgeDistance(Vec3 bary, Vec2 va, Vec2 vb, Vec2 vc) {
    // 计算点 P 到直线 AB 的垂直距离
    // 用叉积：|AB × AP| / |AB|
    auto edgeDist = [](Vec2 p, Vec2 a, Vec2 b) -> float {
        Vec2 ab = {b.x-a.x, b.y-a.y};
        Vec2 ap = {p.x-a.x, p.y-a.y};
        float len = std::sqrt(ab.x*ab.x + ab.y*ab.y);
        if(len < 1e-6f) return std::sqrt(ap.x*ap.x + ap.y*ap.y);
        // 2D "叉积"：ab.x*ap.y - ab.y*ap.x = |ab||ap|sin(θ)
        // 除以 |ab| 得到垂直距离 = |ap|sin(θ)
        return std::abs(ab.x*ap.y - ab.y*ap.x) / len;
    };
    
    // 用重心坐标插值当前像素坐标
    Vec2 p = {va.x*bary.x + vb.x*bary.y + vc.x*bary.z,
              va.y*bary.x + vb.y*bary.y + vc.y*bary.z};
    
    // 三条边：对边 BC、对边 AC、对边 AB
    float d0 = edgeDist(p, vb, vc);  // 到 BC 的距离（顶点 A 的对边）
    float d1 = edgeDist(p, va, vc);  // 到 AC 的距离
    float d2 = edgeDist(p, va, vb);  // 到 AB 的距离
    return std::min({d0, d1, d2});
}
```

**为什么用叉积？**

2D 向量的"叉积"（$ab_x \cdot ap_y - ab_y \cdot ap_x$）等于以 $\vec{AB}$ 和 $\vec{AP}$ 为两边的平行四边形面积，即 $|\vec{AB}||\vec{AP}|\sin\theta$。除以 $|\vec{AB}|$ 就是 $|\vec{AP}|\sin\theta$——这正是点 $P$ 到直线 $AB$ 的垂直距离。

**为什么要用绝对值？**

叉积的符号表示方向（点在直线左侧或右侧），我们只关心距离大小，取 $\text{abs}$。

### 4.2 线框叠加混合

```cpp
if(opts.wireframe) {
    float ed = edgeDistance(bc, sv[0].screen, sv[1].screen, sv[2].screen);
    float wireAlpha = 1.f - std::min(1.f, ed / opts.wireWidth);
    wireAlpha = wireAlpha * wireAlpha;  // 平方使边缘更锐利
    if(wireAlpha > 0.05f) {
        Color wireColor = {230, 230, 255};  // 淡蓝白色
        shade.r = uint8_t(shade.r * (1-wireAlpha) + wireColor.r * wireAlpha);
        shade.g = uint8_t(shade.g * (1-wireAlpha) + wireColor.g * wireAlpha);
        shade.b = uint8_t(shade.b * (1-wireAlpha) + wireColor.b * wireAlpha);
    }
}
```

**阈值 0.05f 的作用**：避免在实体区域做大量无意义的颜色混合计算。当 wireAlpha < 0.05 时，混合影响不到 5%，视觉上不可见，跳过即可。

**线框颜色选择**：淡蓝白 (230, 230, 255) 在深色背景上清晰，又不会太刺眼，与蓝紫色球体主题呼应。

### 4.3 法线颜色球体（右侧球体核心）

```cpp
// 插值世界法线
Vec3 wnorm = (sv[0].normal*bc.x + sv[1].normal*bc.y + sv[2].normal*bc.z).normalized();

// 法线 [-1,1] → 颜色 [0,255]
Color normColor = {
    uint8_t((wnorm.x * 0.5f + 0.5f) * 255),
    uint8_t((wnorm.y * 0.5f + 0.5f) * 255),
    uint8_t((wnorm.z * 0.5f + 0.5f) * 255)
};

// 叠加线框
float wa = (1.f - std::min(1.f, ed / 1.2f));
wa = wa * wa;
if(wa > 0.05f) {
    normColor.r = uint8_t(normColor.r * (1-wa) + 230 * wa);
    normColor.g = uint8_t(normColor.g * (1-wa) + 230 * wa);
    normColor.b = uint8_t(normColor.b * (1-wa) + 255 * wa);
}

fb.setPixel(px, py, normColor, depth);
```

**注意法线必须插值后再归一化**：直接插值的法线不是单位向量，因为插值是线性的而球面是曲面。归一化消除插值误差，确保每个像素的法线颜色映射正确。

### 4.4 Phong 着色实现

```cpp
Color phongShading(Vec3 worldPos, Vec3 normal, Vec3 baseColor) {
    Vec3 lightPos   = {3.f, 5.f, 4.f};
    Vec3 lightColor = {1.f, 0.95f, 0.85f};   // 暖白光
    Vec3 ambient    = {0.1f, 0.1f, 0.15f};    // 冷色调环境光
    Vec3 eyePos     = {0.f, 2.f, 6.f};

    // Lambertian 漫反射
    Vec3 L = (lightPos - worldPos).normalized();
    float diff = std::max(0.f, normal.dot(L));

    // Blinn-Phong 镜面高光（这里用的是 Phong，不是 Blinn-Phong）
    Vec3 V = (eyePos - worldPos).normalized();
    Vec3 R = (normal * (2.f * normal.dot(L)) - L).normalized();  // 反射向量
    float spec = std::pow(std::max(0.f, R.dot(V)), 32.f);

    // 线性组合
    Vec3 col = {
        baseColor.x * (ambient.x + lightColor.x * diff) + lightColor.x * spec * 0.5f,
        // ...
    };
    return clamp_to_uint8(col);
}
```

**反射向量公式推导**：

设入射方向 $L$（指向光源），法线 $N$，则反射方向 $R$ 满足：

$$R = 2(N \cdot L) N - L$$

直觉：$N \cdot L$ 是 $L$ 在法线方向的投影长度，$2(N \cdot L)N$ 是 $L$ 在法线方向的完整反射分量，减去 $L$ 得到反射向量。

### 4.5 背面剔除

```cpp
Vec2 ab = {sv[1].screen.x - sv[0].screen.x, sv[1].screen.y - sv[0].screen.y};
Vec2 ac = {sv[2].screen.x - sv[0].screen.x, sv[2].screen.y - sv[0].screen.y};
if(ab.x*ac.y - ab.y*ac.x < 0) continue;  // 顺时针 → 背面
```

**原理**：在屏幕空间，三角形的有向面积（叉积）决定它是正面还是背面。
- 叉积 $> 0$：顶点逆时针排列（正面）
- 叉积 $< 0$：顶点顺时针排列（背面）

注意：这里的约定取决于坐标系和顶点生成顺序。UV Sphere 和 Cube 的顶点都按"正面逆时针"生成，所以叉积 < 0 表示背面。

### 4.6 切线空间构建与绘制

```cpp
// 构建切线（格拉姆-施密特）
Vec3 up = {0,1,0};
Vec3 tangent = (up - worldNorm * worldNorm.dot(up)).normalized();

// 副切线（右手定则）
Vec3 bitangent = worldNorm.cross(tangent).normalized();

// 绘制三色箭头
drawNormalArrow(fb, worldCentroid, worldNorm,  0.35f, {80,160,255}, VP);  // 法线：蓝
drawNormalArrow(fb, worldCentroid, tangent,    0.28f, {255,80,80},  VP);  // 切线：红
drawNormalArrow(fb, worldCentroid, bitangent,  0.28f, {80,220,80},  VP);  // 副切线：绿
```

**格拉姆-施密特正交化**：

$T = \text{Up} - N \cdot (N \cdot \text{Up})$

**直觉**：$N \cdot \text{Up}$ 是 $\text{Up}$ 在 $N$ 方向的投影量，$N \cdot (N \cdot \text{Up})$ 是这个投影的向量。$\text{Up}$ 减去它在 $N$ 上的投影，剩下的部分就与 $N$ 垂直了。

---

## ⑤ 踩坑实录

### Bug 1：generateCube 数组越界

**症状**：编译时出现 `-Warray-bounds` 警告：
```
main.cpp:166: warning: array subscript 4 is above array bounds of 'Vec3 [4]'
```

**错误假设**：以为后面的 `t1.v[0].pos=f.pts[0]` 等显式赋值会覆盖前面的循环赋值，所以那个"临时的"危险循环无关紧要。

**真实原因**：编译器的 `-Warray-bounds` 会分析每一行，即使后续有覆盖，也会对当前行发出警告。而且那个循环确实在某些下标组合下会访问 `f.pts[4]`（越界），只是被后面覆盖了——但 UB 就是 UB，即使"幸运地"没崩溃也不正确。

**修复方式**：直接删除那个多余的循环，只保留显式的 4 点赋值：
```cpp
// 修复前（有隐患的混合写法）
for(int k=0;k<3;k++) { t2.v[k].pos=f.pts[(k+2)%4==3?3:(k==0?2:k+2)]; ... }
t1.v[0].pos=f.pts[0]; t1.v[1].pos=f.pts[1]; t1.v[2].pos=f.pts[2];
t2.v[0].pos=f.pts[0]; t2.v[1].pos=f.pts[2]; t2.v[2].pos=f.pts[3];

// 修复后（干净，无 UB）
t1.v[0].pos=f.pts[0]; t1.v[0].normal=f.n; t1.v[0].uv={0,0};
t1.v[1].pos=f.pts[1]; t1.v[1].normal=f.n; t1.v[1].uv={1,0};
t1.v[2].pos=f.pts[2]; t1.v[2].normal=f.n; t1.v[2].uv={1,1};
t2.v[0].pos=f.pts[0]; t2.v[0].normal=f.n; t2.v[0].uv={0,0};
t2.v[1].pos=f.pts[2]; t2.v[1].normal=f.n; t2.v[1].uv={1,1};
t2.v[2].pos=f.pts[3]; t2.v[2].normal=f.n; t2.v[2].uv={0,1};
```

**教训**：`-Wall -Wextra` 发出的警告不是"可以忽略的建议"，是"有潜在 UB 的问题"。0 warnings 是硬指标。

### Bug 2：stb_image_write.h 外部库 warnings

**症状**：即使自己代码无误，编译仍有大量 `-Wmissing-field-initializers` warnings，全来自 `stb_image_write.h`。

**错误假设**：以为可以用 `-Wno-missing-field-initializers` 全局关闭这个 warning。但这样的话自己代码里的同类问题也会被屏蔽。

**真实原因**：stb 库是 C 风格代码，用 `struct s = { 0 }` 的写法初始化，在 C++ 编译器下触发 `-Wmissing-field-initializers`。这是 stb 库本身的问题，不是我们代码的问题。

**修复方式**：用 `#pragma GCC diagnostic push/pop` 局部屏蔽，只影响 include 这个头文件的范围：
```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

这样我们自己代码里的 `-Wmissing-field-initializers` 仍然会报出，只有 stb 库的被屏蔽。**局部 pragma 比全局关 warning flag 更精准、更安全**。

### Bug 3（设计层面）：法线可视化密度过大

**症状**：初版每个三角形都绘制法线箭头，导致球体表面被绿色箭头完全覆盖，看不清几何体本身。

**原因**：UV Sphere 16×24 共 16×24×2 = 768 个三角形，768 条箭头严重遮挡视线。

**修复方式**：用面朝摄像机的点积做过滤：
```cpp
Vec3 toEye = (eyePos - worldCentroid).normalized();
if(worldNorm.dot(toEye) > 0.1f)   // 只绘制朝向摄像机的面
    drawNormalArrow(...);
```

这样只有正对摄像机的约一半面会显示法线，密度大幅降低，画面清晰。

**扩展教训**：可视化工具的密度控制（LOD、距离剔除、角度剔除）和渲染效果本身一样重要。

---

## ⑥ 效果验证与数据

### 像素统计结果

```
分辨率: 800×600
像素均值: 38.8
像素标准差: 47.5
文件大小: 219.5 KB
```

**分析**：
- 均值 38.8（范围在 10~240 之间）：图像整体偏暗（深色背景 18, 18, 30），但有明亮的球体和法线箭头 ✅
- 标准差 47.5 > 5：图像内容丰富，不是单色平面 ✅
- 219.5 KB >> 10 KB：PNG 内容充实 ✅

### 运行性能数据

- 分辨率：800×600 = 480,000 像素
- 球体三角形数：16×24×2 = 768（中央球）+ 20×30×2 = 1200（右侧球）
- 立方体三角形数：6×2 = 12
- 总三角形：约 1980 个
- 运行时间：< 0.5 秒（瞬间完成，纯 CPU 单线程）

### 视觉效果验证

| 区域 | 预期效果 | 实际 |
|------|---------|------|
| 中央蓝紫球 | Phong 着色 + 白色线框 + 绿色法线箭头 | ✅ |
| 左侧橙色立方体 | Phong 着色 + 白色线框 + TBN 三色箭头 | ✅ |
| 右侧球体 | 法线颜色编码 + 白色线框（XYZ → RGB）| ✅ |
| 地面网格 | 蓝灰色等间距格线 | ✅ |
| 背面剔除 | 球体内部不可见 | ✅ |
| Z-Buffer | 深度测试正确，无穿插 | ✅ |

---

## ⑦ 总结与延伸

### 本次实现的局限性

1. **切线空间计算过于简化**：真实引擎中，切线 $T$ 是从 UV 坐标的微分计算得到的（$\partial P / \partial u$），而非从 Up 向量格拉姆-施密特。UV 坐标决定了法线贴图的对齐方式，用 Up-GS 只是近似。

2. **无反走样**：线框和箭头都是硬边，没有 MSAA 或 FXAA。在 GPU 实现中，线宽可以用 geometry shader 的"边带三角形"技术做亚像素精度的抗锯齿。

3. **法线箭头没有箭头帽**：真实的法线可视化工具会在箭头末端绘制一个小锥体（箭头帽），更容易看清方向。本次用 Bresenham 线代替，方向不够直观。

4. **XRay 模式的取舍**：法线/切线箭头采用无深度测试的叠加混合，使得被物体遮挡的箭头也可见。这在调试时有用，但不符合真实深度关系。可以加一个 `xray` 开关来切换。

### 可优化方向

1. **基于 UV 的真实切线计算**：
```cpp
// 从三角形位置和 UV 坐标计算切线
Vec3 edge1 = v1.pos - v0.pos, edge2 = v2.pos - v0.pos;
Vec2 dUV1  = v1.uv  - v0.uv,  dUV2  = v2.uv  - v0.uv;
float f = 1.f / (dUV1.x*dUV2.y - dUV2.x*dUV1.y);
Vec3 tangent   = (edge1*dUV2.y - edge2*dUV1.y) * f;
Vec3 bitangent = (edge2*dUV1.x - edge1*dUV2.x) * f;
```

2. **法线贴图采样**：有了真实的 TBN，就可以从法线贴图采样扰动法线，实现凹凸感。

3. **连接 POM（视差遮挡贴图）**：昨天系列里做过 POM，TBN 正确性是 POM 效果的前提。

4. **实例化渲染**：当前每个对象单独调用 rasterizeTriangle，可以用 SIMD 或多线程加速像素循环。

### 与本系列的关联

- **05-17 POM**：依赖正确的 TBN 矩阵，本次可视化是调试 POM 错误的工具
- **05-21 薄膜干涉**：法线朝向会影响 Fresnel 计算，法线可视化可以排查 Fresnel 消失的问题
- **05-12 延迟渲染（G-Buffer）**：G-Buffer 的法线通道本质上就是法线颜色编码，右侧球体展示的正是 G-Buffer 法线贴图的样子

---

## 代码核心片段汇总

```cpp
// 边缘距离（核心）
float edgeDistance(Vec3 bary, Vec2 va, Vec2 vb, Vec2 vc) {
    auto edgeDist = [](Vec2 p, Vec2 a, Vec2 b) -> float {
        Vec2 ab = {b.x-a.x, b.y-a.y}, ap = {p.x-a.x, p.y-a.y};
        float len = std::sqrt(ab.x*ab.x + ab.y*ab.y);
        if(len < 1e-6f) return std::sqrt(ap.x*ap.x + ap.y*ap.y);
        return std::abs(ab.x*ap.y - ab.y*ap.x) / len;
    };
    Vec2 p = {va.x*bary.x + vb.x*bary.y + vc.x*bary.z,
              va.y*bary.x + vb.y*bary.y + vc.y*bary.z};
    return std::min({edgeDist(p,vb,vc), edgeDist(p,va,vc), edgeDist(p,va,vb)});
}

// 线框叠加
float wireAlpha = std::pow(1.f - std::min(1.f, ed/wireWidth), 2.f);
shade.r = uint8_t(shade.r*(1-wa) + 230*wa);  // 混合到白色

// 法线颜色编码
Color normColor = {
    uint8_t((n.x*0.5f+0.5f)*255),
    uint8_t((n.y*0.5f+0.5f)*255),
    uint8_t((n.z*0.5f+0.5f)*255)
};

// TBN 构建
Vec3 tangent = (up - N*(N.dot(up))).normalized();   // GS
Vec3 bitangent = N.cross(tangent).normalized();       // 右手定则
```

完整代码：[GitHub](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-31-Geometry-Shader-Effects-Renderer)

---

*写于 2026-05-31 05:30 AM。每天一个小项目，积累图形学理解。*
