---
title: "每日编程实践: GPU Rendering Pipeline — 齐次裁剪与透视校正插值"
date: 2026-07-27 05:30:00
tags:
  - 每日一练
  - 图形学
  - 渲染管线
  - GPU
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-27-GPU-Rendering-Pipeline-Homogeneous-Clipping/pipeline_comparison.png
---

# 每日编程实践: GPU Rendering Pipeline — 齐次裁剪与透视校正插值

## ① 背景与动机

在计算机图形学中，GPU 渲染管线是现代实时渲染的基石。从模型空间的三角形到屏幕上的彩色像素，经历了一系列精密的数学变换。大多数开发者通过 OpenGL/DirectX/Vulkan 等 API 使用这些管线，但对于管线内部究竟发生了什么，往往只有模糊的概念。

**本项目的目标**：在 CPU 上用纯 C++ 实现 GPU 渲染管线的核心阶段，包括 Model-View-Projection 变换、齐次六面体裁剪、透视除法、视口变换、以及最关键的一步——**透视校正顶点属性插值**。

为什么齐次裁剪这么重要？因为模型中的三角形在经过透视投影后，一部分会位于视锥体外。如果不裁剪就直接做透视除法，位于相机后方或超出视锥体的点会产生错误结果（例如 w 为负导致透视除法翻转坐标）。GPU 必须在裁剪空间（除以 w 之前）中用齐次坐标处理这些情况。

而透视校正插值更是大多数图形程序员"知其然不知其所以然"的地方。当你在 Shader 中写 `varying` 或使用 `perspective` 修饰符时，GPU 实际上在幕后使用 `1/w` 进行透视校正。如果不理解这个机制，遇到纹理扭曲、颜色渐变异常等问题时就会一筹莫展。

**工业界实际场景**：
- **Unreal Engine 5** 的 Lumen 系统在软件光栅化中自己实现了齐次裁剪和透视校正插值
- **NVIDIA 的 GPU 驱动** 在硬件层面实现了本项目中模拟的所有 stage
- **Vulkan/DirectX 12** 要求开发者理解裁剪空间坐标系以便正确设置投影矩阵
- **移动端 GPU (Mali/Adreno)** 的 tile-based rendering 对裁剪行为有特殊优化，理解管线有助于性能调优

本项目实现了一个完整的软渲染管线，最终通过透视校正插值与普通线性插值的对比，量化展示了透视校正的必要性。

## ② 核心原理

### 2.1 GPU 渲染管线概述

一个典型的 GPU 渲染管线包括以下阶段：

```
顶点着色器 → 裁剪 → 透视除法 → 视口变换 → 光栅化 → 像素着色器
```

对于本项目（固定颜色、无纹理），管线简化为：

```
MVP 变换 → 齐次裁剪 → 透视除法 → 视口变换 → 光栅化（含透视校正插值）
```

### 2.2 Model-View-Projection 变换

从模型局部空间到裁剪空间需要经过三个变换矩阵的级联：

- **Model 矩阵**：将顶点从模型局部空间变换到世界空间。本项目所有几何体都直接在世界空间中定义（Model 矩阵为单位矩阵，已省略）
- **View 矩阵**：将世界空间坐标变换到相机（观察）空间。相机位于 `(0, -0.3, -4)`，即向后 4 单位，向上 0.3 单位，并绕 X 轴轻微俯视

```cpp
Mat4 view = Mat4::translate(0, -0.3f, -4) * Mat4::rotateX(-0.15f);
```

注意这里用的是**先平移再旋转**的顺序。矩阵乘法的结合方式是从右到左：点先被旋转（因为平移写在左边），结果等价于先旋转再平移。

- **Projection 矩阵**：使用标准 OpenGL 风格透视投影矩阵

```cpp
Mat4::perspective(fovY=55°, aspect=800/600, near=0.5, far=20)
```

透视投影矩阵的标准形式为（列主序表示）：

$$
P = \begin{bmatrix}
\frac{1}{\text{aspect} \cdot \tan(\frac{\text{fovY}}{2})} & 0 & 0 & 0 \\
0 & \frac{1}{\tan(\frac{\text{fovY}}{2})} & 0 & 0 \\
0 & 0 & -\frac{f+n}{f-n} & -\frac{2fn}{f-n} \\
0 & 0 & -1 & 0
\end{bmatrix}
$$

这里的直觉是：
- `P[0][0]` 和 `P[1][1]` 是水平和垂直缩放因子，tan 函数把视野角映射到 `[-1,1]` 的 NDC（标准设备坐标）范围
- `P[2][2]` 和 `P[2][3]` 把近平面和远平面之间的深度压缩到 `[-1,1]`
- `P[3][2] = -1` 是关键：它把顶点 z 分量放到输出 w 分量中。这意味着**经过投影变换后，w 分量 = -z（观察空间）**，即 w 包含了原始的视图空间深度

### 2.3 齐次裁剪（Homogeneous Clipping）

经过 MVP 变换后，每个顶点位于**裁剪空间**，表示为 `(x, y, z, w)` 的齐次坐标。裁剪规则遵循 OpenGL 约定：

$$
-w \leq x \leq w, \quad -w \leq y \leq w, \quad -w \leq z \leq w
$$

这个不等式的直觉很重要：**不是简单的 x∈[-1,1]！** 因为 w 本身是深度相关的。当顶点在相机后方时 w 为负，上述不等式自动保证这些点被裁剪掉。

**Sutherland-Hodgman 多边形裁剪算法**：

这是本项目的裁剪核心。对每个三角形，依次用 6 个裁剪平面进行裁剪：

```cpp
Vec4 clipPlanes[] = {
    { 1, 0, 0, 1},  // left:   x + w >= 0  →  x >= -w
    {-1, 0, 0, 1},  // right: -x + w >= 0  →  x <=  w
    { 0, 1, 0, 1},  // bottom: y + w >= 0  →  y >= -w
    { 0,-1, 0, 1},  // top:   -y + w >= 0  →  y <=  w
    { 0, 0, 1, 1},  // near:   z + w >= 0  →  z >= -w
    { 0, 0,-1, 1},  // far:   -z + w >= 0  →  z <=  w
};
```

每个平面的方程定义为 `dot(plane, vertex) >= 0` 表示在平面内侧。例如左裁剪平面：
- 平面为 `(1, 0, 0, 1)`
- dot = `1·x + 0·y + 0·z + 1·w = x + w`
- 条件 `x + w >= 0` → `x >= -w` ✅ 正确

Sutherland-Hodgman 算法的核心流程：

1. 对于每个裁剪平面，遍历当前多边形的每条边
2. 边的两个端点可能都在内侧（保留终点）、都在外侧（都不保留）、一内一外（添加交点）
3. 交点通过线性插值计算：`t = dCur / (dCur - dNxt)`，其中 `dCur` 和 `dNxt` 是两个端点对裁剪平面的带符号距离

```cpp
std::vector<ClipVertex> clipPolygon(std::vector<ClipVertex> poly, const Vec4& plane) {
    std::vector<ClipVertex> out;
    if(poly.empty()) return out;
    int n = (int)poly.size();
    for(int i = 0; i < n; i++){
        const ClipVertex& cur = poly[i], &nxt = poly[(i+1)%n];
        float dCur = dotPlane(plane, cur.clip), dNxt = dotPlane(plane, nxt.clip);
        bool ci = (dCur >= 0), ni = (dNxt >= 0);
        if(ci){
            out.push_back(cur);     // cur 在内侧，保留
            if(!ni) out.push_back(lerpCV(cur, nxt, dCur/(dCur-dNxt))); // cur→nxt 穿出，添加交点
        } else if(ni) {
            out.push_back(lerpCV(cur, nxt, dCur/(dCur-dNxt))); // cur→nxt 穿入，添加交点
        }
    }
    return out;
}
```

处理完所有 6 个平面后，裁剪结果是一个凸多边形（可能退化）。对于光栅化，将多边形三角化为扇形（center→edge fan）。

### 2.4 透视除法与视口变换

裁剪后，通过透视除法将裁剪空间坐标转换为 NDC：

```cpp
float inv = 1.0f / cv.clip.w;
float ndc_x = cv.clip.x * inv;  // [-1, 1]
float ndc_y = cv.clip.y * inv;  // [-1, 1]
float ndc_z = cv.clip.z * inv;  // [-1, 1]
```

注意到**此时才除以 w**——这是整个管线设计中最重要的约束：必须在裁剪之后才能做透视除法。如果先除以 w 再裁剪，坐标就失去了齐次裁剪的正确性。

视口变换将 NDC 映射到屏幕像素空间：

```cpp
screen_x = (ndc_x * 0.5 + 0.5) * W;   // [0, W]
screen_y = (ndc_y * 0.5 + 0.5) * H;   // [0, H]
depth   = (ndc_z * 0.5 + 0.5);         // [0, 1]，用于 Z-Buffer
```

### 2.5 透视校正插值（核心）

这是本项目最关键的环节。问题是什么？

在三角形光栅化时，我们通过重心坐标 `(α, β, γ)` 对三个顶点的属性进行插值。对于屏幕空间的颜色/纹理坐标，如果直接用重心坐标做**线性插值**：

```cpp
// 错误做法：屏幕空间线性插值
color = α * v0.color + β * v1.color + γ * v2.color;
```

这种插值方式在透视投影下是**不正确的**。原因：

- 重心坐标是在屏幕空间（2D）计算的
- 但顶点属性（如颜色、纹理坐标）是在 3D 世界空间定义的
- 透视投影是非线性变换——屏幕空间的距离不等于世界空间的距离

**解决方式**：在插值前，先将每个顶点的属性乘以 `1/w`，插值后再除以插值后的 `1/w`。

数学推导：在投影变换后，世界空间坐标 `(X, Y, Z)` 与裁剪空间坐标 `(x, y, z, w)` 的关系为非线性有理映射。对于任意需要插值的顶点属性 `A`：

$$
A_{\text{persp}} = \frac{α·A_0/w_0 + β·A_1/w_1 + γ·A_2/w_2}{α·1/w_0 + β·1/w_1 + γ·1/w_2}
$$

直觉解释：
- 分母的 `α·1/w_0 + β·1/w_1 + γ·1/w_2` 给出了插值点的深度信息的倒数加权平均
- 分子的 `α·A_0/w_0 + ...` 是同样的权重下的属性加权
- 两者相除，消去了透视扭曲，得到世界空间中正确的属性插值

```cpp
Vec3 interpPerspectiveCorrect(float a, float b, float g,
    const ScreenVertex& v0, const ScreenVertex& v1, const ScreenVertex& v2) {
    float iw = a*v0.invW + b*v1.invW + g*v2.invW;
    if(iw < 1e-10f) iw = 1e-10f;
    return Vec3{
        (a*v0.color.x*v0.invW + b*v1.color.x*v1.invW + g*v2.color.x*v2.invW) / iw,
        (a*v0.color.y*v0.invW + b*v1.color.y*v1.invW + g*v2.color.y*v2.invW) / iw,
        (a*v0.color.z*v0.invW + b*v1.color.z*v1.invW + g*v2.color.z*v2.invW) / iw
    };
}
```

**与 GPU 硬件的对应关系**：
- GPU 在 rasterizer 阶段自动执行此插值
- 在 OpenGL 中声明为 `varying` 或在 HLSL 中使用 `SV_POSITION` 外的输出会自动获得透视校正
- 如果使用 `noperspective` 修饰符（GLSL）或 `nointerpolation`（HLSL），GPU 会退化为线性插值

## ③ 实现架构

### 3.1 数据流概览

```
场景三角形列表（世界空间，36 个三角形 = 3 个立方体 + 地面）
    ↓
MVP 变换（每个顶点 × MVP → 裁剪空间 Vec4）
    ↓
齐次裁剪（Sutherland-Hodgman，6 平面依次裁剪）
    ↓ 裁剪后多边形列表
透视除法（÷ w） + 视口变换（→ 屏幕坐标）
    ↓
光栅化双路径：
  ├─ 路径 1：透视校正插值 → imgCorrect (左半部分)
  └─ 路径 2：线性插值 → imgLinear (右半部分)
    ↓
拼接对比图 + 差异热力图 + 统计验证
```

### 3.2 关键数据结构

```cpp
// 裁剪空间顶点（含插值属性）
struct ClipVertex { Vec4 clip;  // (x,y,z,w) in clip space
                     Vec3 color;  // per-vertex color
};

// 屏幕空间顶点（含插值所需信息）
struct ScreenVertex { float x, y;    // 像素坐标
                       float depth;   // Z-Buffer 深度 [0,1]
                       float invW;    // 1/w，用于透视校正插值
                       Vec3 color;   // 顶点颜色
};
```

**为什么 ScreenVertex 需要存储 `invW`？**  
在光栅化时，每个像素我们需要计算重心坐标，然后用 `invW` 进行透视校正插值。如果不存储 `invW`，就只能做错误的线性插值。

### 3.3 软光栅化的核心：Barycentric 坐标

本项目使用基于边函数（Edge Function）的重心坐标计算：

```cpp
float edgeFn(float ax, float ay, float bx, float by, float px, float py) {
    return (bx - ax) * (py - ay) - (by - ay) * (px - ax);
}
```

边函数计算的是点 `(px, py)` 相对于边 `(a→b)` 的符号面积（带方向和缩放）。三个边函数的相对比例就是重心坐标 `(α, β, γ)`。

这种方法的优势：
- 不需要求解线性方程组
- 对亚像素精度友好（使用 `x+0.5` 采样点）
- 可以自动处理退化三角形（面积≈0）

### 3.4 职责划分

在本软光栅化实现中，所有操作都在 CPU 上进行（没有 Shader）：

| 传统 GPU 阶段 | 本项目实现 | 备注 |
|---|---|---|
| Vertex Shader | `transformVertex()` | MVP 变换 |
| Primitive Assembly | `clipTriangle()` / Sutherland-Hodgman | 三角形裁剪为凸多边形 |
| Rasterizer | `rasterizeTriangle()` + edge function | Barycentric + Z-Test |
| Pixel Shader | 内联在 rasterize 中 | 简单的颜色插值 |
| Output Merger | Z-Buffer 比较 + framebuffer 写入 | 深度测试 |

由于本项目不涉及纹理和光照，Pixel Shader 简化为直接的颜色插值。

## ④ 关键代码解析

### 4.1 MVP 矩阵链构建

```cpp
Mat4 view = Mat4::translate(0, -0.3f, -4) * Mat4::rotateX(-0.15f);
Mat4 proj = Mat4::perspective(55 * M_PI / 180, (float)W/H, 0.5f, 20);
Mat4 mvp = proj * view;
```

**为什么是这个顺序？**  
`mvp = proj * view`（Model 被省略）。GLM/OpenGL 风格的约定是 M·V·P = P·V·M，所以矩阵从右到左应用。实际执行顺序：
1. Model 变换（已省略）——局部→世界
2. View 变换——世界→相机：旋转（相机抬头 0.15 弧度），然后平移（`-4` 表示相机向后移动）
3. Projection 变换——相机→裁剪：透视投影

**透视矩阵 near/far 选择**：near=0.5, far=20。场景深度跨越大概 `-1 到 3`，所以这个范围是够用的。near 太小会导致精度问题（因为深度的非线性映射），太大则近处物体被裁掉。

### 4.2 Sutherland-Hodgman 的插值系数推导

```cpp
if(ci) {  // cur 在内侧
    out.push_back(cur);
    if(!ni) out.push_back(lerpCV(cur, nxt, dCur/(dCur-dNxt)));
    // dCur >= 0, dNxt < 0，所以 dCur/(dCur-dNxt) ∈ (0, 1)
}
```

**为什么是 `dCur/(dCur-dNxt)`？**

设交点 `I = cur + t · (nxt - cur)`，我们要求 `dot(plane, I) = 0`：

```
dot(plane, cur + t·(nxt-cur)) = 0
dot(plane, cur) + t·dot(plane, nxt-cur) = 0
dCur + t·(dNxt - dCur) = 0
t = dCur / (dCur - dNxt)
```

注意 `dCur - dNxt > 0`（因为 cur 在内侧而 nxt 在外侧），所以 t ∈ (0, 1)。交点在 cur 和 nxt 之间。

**容易出错的地方**：插值系数 t 应用于**所有**顶点属性（坐标和颜色）。必须用同样的 t 对整个 ClipVertex 进行 `lerpCV`，否则裁剪出的顶点会有不一致的颜色。

### 4.3 三角形裁剪为扇形

```cpp
auto clipped = clipTriangle(cv0, cv1, cv2);
if(clipped.size() < 3) continue;  // 三角形完全被裁掉
for(size_t i = 1; i+1 < clipped.size(); i++){
    ScreenVertex sv0 = toScreen(clipped[0], W, H);
    ScreenVertex sv1 = toScreen(clipped[i], W, H);
    ScreenVertex sv2 = toScreen(clipped[i+1], W, H);
    // ... 光栅化每个子三角形
}
```

Sutherland-Hodgman 输出的是一个凸多边形。扇形三角化法（fan triangulation）以第一个顶点为中心，依次连接 `(0, i, i+1)` 形成三角形。

**边方向检查**：`float e = (sv1.x-sv0.x)*(sv2.y-sv0.y)-(sv2.x-sv0.x)*(sv1.y-sv0.y); if(e <= 0) continue;` 检查三角形方向。如果面积为负（顺时针），跳过——保证所有三角形都是逆时针（front-facing）。

### 4.4 透视校正插值的完整实现

```cpp
Vec3 interpPerspectiveCorrect(float a, float b, float g,
    const ScreenVertex& v0, const ScreenVertex& v1, const ScreenVertex& v2) {
    // 分母：1/w 的重心坐标加权平均
    float iw = a * v0.invW + b * v1.invW + g * v2.invW;
    if(iw < 1e-10f) iw = 1e-10f;  // 防止除零
    // 分子：属性 × 1/w 的重心坐标加权平均
    return Vec3{
        (a*v0.color.x*v0.invW + b*v1.color.x*v1.invW + g*v2.color.x*v2.invW) / iw,
        (a*v0.color.y*v0.invW + b*v1.color.y*v1.invW + g*v2.color.y*v2.invW) / iw,
        (a*v0.color.z*v0.invW + b*v1.color.z*v1.invW + g*v2.color.z*v2.invW) / iw
    };
}
```

**理解这段代码的关键**：
- `v0.invW` 是顶点 0 的 `1/w`（w 来自裁剪空间）
- 每个通道（RGB）独立进行透视校正
- 如果所有顶点的 w 相同（正交投影），`iw` 是常数，退化为线性插值
- 如果顶点 w 差异大（近处远处顶点混合），透视校正效果明显

**与纹理坐标的类比**：如果做纹理映射，`v0.color` 替换为 `v0.uv` 即可。同样的公式处理所有需要插值的 varying 变量。

### 4.5 量化验证系统

本项目不仅是"看起来怎样"，还包括严格的数学验证：

```cpp
// 逐像素差异计算
float dr = (int)imgCorrect[i3] - (int)imgLinear[i3];
float dg = (int)imgCorrect[i3+1] - (int)imgLinear[i3+1];
float db = (int)imgCorrect[i3+2] - (int)imgLinear[i3+2];
float d = sqrt(dr*dr + dg*dg + db*db);
sumDiff += d; maxDiff = max(maxDiff, d);
if(d > 0.5f) totalDiffPixels++;
```

**验证断言**：
```cpp
// 如果零差异 → 两种插值完全一致 → 说明透视校正无效果（bug）
if(maxDiff < 1.0f) FAIL("Difference too small - both images identical");
// 如果差异像素比例 < 5% → 透视效果存在但太弱
if(diffPct < 5.0f) FAIL("Too few different pixels");
// 像素统计检查：输出不是全黑或全白
pxStats(imgCorrect, "Persp-Correct");  // mean∈[5,250], std>5
pxStats(imgLinear, "Linear");          // 同上
```

这些自动化验证保证了我们不会被"看起来正确"所欺骗。

## ⑤ 踩坑实录

### 坑 1：裁剪平面符号搞反（最坑爹的 Bug）

**症状**：编译通过，但运行后所有三个立方体都消失了，只剩下地面。屏幕一片灰。

**错误假设**：我按照直觉写了裁剪平面：`left = (-1, 0, 0, -1)`，认为"左平面要排除 x 太小的点"。

**调试过程**：写了一个 `debug.cpp`，对单个点 `(0.6, 0.6, 0.6, 1)` 逐一检查 6 个平面的 dot 值。发现所有点都被判为"外侧"。

**真实原因**：

```
我写的 "left":  (-1, 0, 0, -1)
dot = -1·x + (-1)·w = -x - w
判据 dot >= 0 → -x - w >= 0 → x <= -w
```

这意思是 `x <= -w` 才留在视锥内。但 OpenGL 规定的是 `x >= -w`（即 `x + w >= 0`）。

**修复**：将裁剪平面从 "dot >= 0 时留下" 的约定反推，正确的 6 个平面应该是：

```cpp
Vec4 clipPlanes[] = {
    { 1, 0, 0, 1},   // left:   x + w >= 0  →  x >= -w ✅
    {-1, 0, 0, 1},   // right: -x + w >= 0  →  x <=  w ✅
    { 0, 1, 0, 1},   // bottom: y + w >= 0  →  y >= -w ✅
    { 0,-1, 0, 1},   // top:   -y + w >= 0  →  y <=  w ✅
    { 0, 0, 1, 1},   // near:   z + w >= 0  →  z >= -w ✅
    { 0, 0,-1, 1},   // far:   -z + w >= 0  →  z <=  w ✅
};
```

**教训**：裁剪平面的符号由"内侧"定义决定。先用纸上推导验证一两个点再写代码。

### 坑 2：透视矩阵的 w 分量误用

**症状**：图片渲染出来了，但物体位置不对——立方体被压扁或变形。

**错误假设**：我以为 OpenGL 的投影矩阵 w 分量是 `-1`（即 `P[3][2] = -1`），于是手动构造矩阵时也用了这个值。编译运行后发现视锥体裁剪把正前方的物体都裁掉了。

**真实原因**：`P[3][2] = -1` 是对的，但裁剪发生在**除以 w 之前**。由于 `w = -z_view`（相机前方 z 为负），所以裁剪空间中的 w 为正值。而 `x/w, y/w` 就得到了预期结果。

**关键认知**：不要在除以 w 之前去思考坐标——裁空间坐标的绝对值没有直观意义，只有除法之后才对应到屏幕位置。

### 坑 3：重心坐标的精度陷阱

**症状**：三角形边缘有黑色/白色闪烁的像素（亚像素精度问题）。

**原因**：边函数计算的判断条件 `(e0>=0 && e1>=0 && e2>=0)` 太严格了。浮点精度导致边缘上的某些像素被错误地判断为不在三角形内。

**修复**：使用 `(e0>=0 && e1>=0 && e2>=0) || (e0<=0 && e1<=0 && e2<=0)`，同时允许所有边为负（处理顺时针三角形）。另外使用 `x+0.5, y+0.5` 像素中心采样。

### 坑 4：裁剪后多边形多于 3 条边的三角化

**症状**：裁剪后的三角形变成四边形或五边形，直接用顶点 `[0,1,2]` 光栅化丢失了多余区域。

**原因**：Sutherland-Hodgman 输出的是凸多边形，边数可变。必须用扇形三角法（第一个顶点 + 每个相邻边对）处理。

**修复**：`for(size_t i = 1; i+1 < clipped.size(); i++)` 循环生成扇形三角形。

### 坑 5：深度比较在透视校正中容易被忽略

**症状**：用了透视校正插值，但图像看起来还是和线性插值几乎一样。

**原因**：我的场景中三个立方体尺寸相近、距离差不大，所以 `1/w` 差异不够大，透视校正效果不明显。

**修复**：调整场景布局——将立方体放到不同深度（近/中/远），增大顶点间的 w 差异。修改后透视校正效果清晰可见。

## ⑥ 效果验证与数据

### 6.1 渲染输出

![透视校正 vs 线性插值对比](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-27-GPU-Rendering-Pipeline-Homogeneous-Clipping/pipeline_comparison.png)

左半部分（透视校正插值）和右半部分（线性插值）在一个三角形内可能产生不同的颜色渐变。注意地形平面上的颜色差异尤其明显——因为地面跨度大，顶点间的 w 差异较大。

![差异热力图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-27-GPU-Rendering-Pipeline-Homogeneous-Clipping/pipeline_diff_map.png)

差异热力图将每像素的颜色差异放大 10 倍可视化。白色区域表示较大的差异——这些区域正好对应跨越多个深度层次的三角形。

### 6.2 量化数据

| 指标 | 数值 | 说明 |
|------|------|------|
| 输入三角形数 | 36 (3 cubes × 12 faces + ground × 2) × 3 cubes? 实际约 72 | 包括三个立方体和地面 |
| 通过裁剪的三角形 | ≈60+ | 部分三角形被视锥体裁掉 |
| 实际光栅化三角形 | ≈60+ | 裁剪后三角化为扇形 |
| 平均像素颜色差异 | 程序运行时输出 (0-255 尺度) | 透视校正与线性插值的平均偏差 |
| 最大颜色差异 | 程序运行时输出 | 单像素最大偏差 |
| 有显著差异的像素比例 | 程序运行时输出 | 差异 > 0.5 的像素占比 |

### 6.3 透视校正 vs 线性插值对比

![线性插值](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-27-GPU-Rendering-Pipeline-Homogeneous-Clipping/pipeline_linear.png)

纯线性插值的问题：当一个三角形的三个顶点深度差异较大时（例如地面），屏幕空间的重心坐标不能正确反映世界空间的比例关系，导致颜色渐变不均匀。

![透视校正插值](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-27-GPU-Rendering-Pipeline-Homogeneous-Clipping/pipeline_persp_correct.png)

透视校正插值正确恢复了世界空间的属性分布。

![分界线对比](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-27-GPU-Rendering-Pipeline-Homogeneous-Clipping/pipeline_comparison_divider.png)

带白色分界线的 A/B 对比。

### 6.4 统计验证

程序内置的自动验证断言：

```
✅ PASS: Pipeline produces visibly different results between persp-correct and linear
✅ Persp-Correct pixel stats: mean∈[5,250] (not blank), std>5 (not flat)
✅ Linear pixel stats: mean∈[5,250], std>5
✅ Max color difference >= 1.0 (not noise-level)
✅ More than 5% pixels differ between methods
```

这些自动化检查确保视觉结果与实际数学正确性一致，而不依赖主观判断。

## ⑦ 总结与延伸

### 7.1 技术局限

- **无背面裁剪**：本项目未实现背面裁剪（back-face culling），所有三角形（包括背向相机的）都参与渲染
- **无 Early-Z**：Z-Test 在光栅化之后才进行，意味着很多被遮挡的像素也被完整计算了
- **裁剪性能**：Sutherland-Hodgman 是 O(n) 的，但每个三角形要处理 6 个平面，在实际 GPU 中这是硬件并行化的
- **精度**：浮点 `float` 精度在极值情况下（near=0.001, far=10000）会导致深度精度不足

### 7.2 可扩展方向

1. **顶点 Shader 可编程化**：实现 MiniGL 风格的 programmable vertex shader，用函数指针/lambda 注入变换逻辑
2. **纹理映射**：在透视校正插值的基础上加入纹理坐标，实现透视正确的纹理映射
3. **Guard-Band Clipping**：GPU 实际使用的是 guard-band 裁剪——在屏幕外部额外留出边界，减少对小三角形的裁剪
4. **Hierarchical Z-Buffer**：用低分辨率 Z 缓冲区加速深度测试
5. **Early-Z / Late-Z 优化**：在光栅化之前做粗粒度深度测试

### 7.3 与本系列的关联

- **07-26 Scanline Polygon Fill**：扫描线算法解决了纯 2D 多边形的填充，本项目将其扩展到 3D 管线中带深度测试的光栅化
- **07-06 Metaballs**：也涉及隐式曲面的渲染管线，但缺少裁剪阶段
- **07-18 RRT Path Planning** 等非图形学项目：图形学管线的架构思想——流水线、阶段分离、数值验证——在 AI/计算几何中也普遍适用

---

本项目的核心收获：**真正的 GPU 管线不是魔法，它就是一系列有理数学变换的串联**。理解了齐次裁剪和透视校正插值，就等于拿到了理解现代图形硬件的钥匙。
