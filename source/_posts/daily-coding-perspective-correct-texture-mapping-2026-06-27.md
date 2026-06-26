---
title: "每日编程实践: 透视校正纹理映射"
date: 2026-06-27 06:30:00
tags:
  - 每日一练
  - 图形学
  - 光栅化
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-27-perspective-correct-texture-mapping/output.png
---

## 背景与动机

纹理映射（Texture Mapping）是实时渲染中最基础也最常用的技术之一。它的核心思想很简单：把一张二维图片"贴"到三维几何表面上，让几何体拥有丰富的表面细节而不需要额外的几何复杂度。

但这里有一个非常容易掉进去的坑——**透视校正**。

想象你正在开发一个游戏引擎的软件光栅化器。你已经实现了三角形光栅化、重心坐标插值、纹理双线性采样——一切看起来都很完美。你把一张棋盘格纹理贴到一个平放的四边形上，渲染结果完美无瑕。然后你把相机稍微倾斜一点……棋盘格开始变得奇怪，纹理会像融化的冰淇淋一样扭曲。

这就是经典的"Affine 纹理映射"问题：**在屏幕空间线性插值 UV 坐标，在存在透视投影时会导致严重的视觉错误**。

这个问题在工业界有真实的危害性：

- **PlayStation 1 (1994)** 的 GPU 只支持 Affine 纹理映射。你在 PS1 游戏中看到的纹理扭曲（特别是墙面和地板）就是这个原因。当时硬件没有透视校正能力，开发者需要将模型细分（tessellation）更多三角形来缓解问题。
- **PlayStation 2 / Dreamcast** 的 GPU 已经支持透视校正，但为了性能，开发者有时会选择关闭。PC 上的 3dfx Voodoo 系列（1996-2000）从 Voodoo2 开始支持透视校正。
- 即便是今天的 **Unity / Unreal Engine**，在 Shader 中错误地使用 `tex2D` 而不考虑透视校正，也会导致错误。好消息是现代 GPU 的硬件光栅化器默认就做透视校正，你不需要手动实现——但如果你写的是**软件光栅化器**（比如本文），就必须自己处理。
- **软件渲染的领域**：光线追踪通常不需要纹理透视校正（因为光线在三维空间采样），但光栅化需要。在移动端、嵌入式设备、或写教学用渲染器时，这个问题依然相关。

本文的目标是：

1. **直观理解**为什么会发生 Affine 扭曲
2. **数学推导**透视校正的公式（不只是给结论）
3. **实现**一个左右对比的渲染器：左边用 Affine（错误），右边用透视校正（正确）
4. **量化验证**两种方法的像素级差异

读完这篇文章，你将不仅会"用"透视校正，还会理解它**为什么**以及**何时**需要。

## 核心原理

### 透视投影的本质

在标准的光栅化管线中，顶点经过以下变换：

```
Model Space → World Space → View Space → Clip Space → NDC → Screen Space
```

其中与本文最相关的步骤是 **View Space → Clip Space**，即透视投影矩阵。

透视投影矩阵（本文使用 OpenGL 右手坐标系，-Z 朝向屏幕内）：

```
M_proj = | f/aspect    0         0          0      |
         |    0        f         0          0      |
         |    0        0   -(f+n)/(f-n)  -2fn/(f-n)|
         |    0        0        -1          0      |
```

其中 `f = 1/tan(fov/2)`, `n/f` 是近/远裁剪面。

关键点在于第四分量 `w`（齐次坐标）不再是 1。变换后：

```
clip_pos = M_proj * eye_pos

x_clip, y_clip, z_clip, w_clip
```

然后 perspective divide：

```
x_ndc = x_clip / w_clip
y_ndc = y_clip / w_clip
z_ndc = z_clip / w_clip
```

最后映射到屏幕空间：

```
screen_x = (x_ndc * 0.5 + 0.5) * screen_width
screen_y = (0.5 - y_ndc * 0.5) * screen_height
```

**关键洞察**：`w_clip = -z_eye`（在本文使用的投影矩阵下）。也就是说，**clip space 的 w 分量携带了相机空间中的深度信息**。处于不同深度的顶点，其 w 值不同，进而影响了 perspective divide 的结果。

### 为什么 Affine 插值是错误的

当我们光栅化一个三角形时，需要在三角形内部进行插值。考虑三个顶点的 UV 坐标 `(u0,v0), (u1,v1), (u2,v2)`。

**Affine 方法**（错误）：在屏幕空间中，直接用重心坐标对 UV 做线性插值。

```
u_pixel = α * u0 + β * u1 + γ * u2
v_pixel = α * v0 + β * v1 + γ * v2
```

其中 (α, β, γ) 是根据像素的屏幕坐标计算的重心坐标。

**为什么这么做在透视投影下是错误的？**

考虑一个简单的例子：一个矩形从左下角到右上角，z 坐标从近到远变化（即矩形倾斜于相机）。

- 近处的顶点在屏幕空间占据**更多**像素（更大面积），但它的 UV 覆盖范围与远处顶点相同
- 如果在屏幕空间线性插值 UV，就相当于假设 UV 在屏幕空间的变化是**匀速**的——但实际上它在**世界空间**才是匀速的
- 屏幕空间与世界空间的关系由透视投影弯曲了

用一个具体的类比来理解：你站在铁轨上看向远方。近处的铁轨看起来很宽（占据大量屏幕像素），远处的铁轨看起来汇聚到一点（只占据很少屏幕像素）。但是铁轨的实际宽度在三维空间中是恒定的。

Affine 插值就像你在"铁轨的屏幕空间图像"中线性插值——这在近处会过度采样纹理，在远处会采样不足。

**数学上**：假设我们要插值某个属性 `P`（比如 UV 坐标）。在世界空间中，`P` 应该随三维坐标（包括深度）线性变化。经过透视投影后，屏幕坐标 `(sx, sy)` 与三维坐标的关系是：

```
sx = (P_x * x/z + const)  # 除以 z（深度）
sy = (P_y * y/z + const)  # 除以 z（深度）
```

属性 `P` 的屏幕空间表现形式实际上需要除以深度 `z`。**在齐次空间中，需要在插值 `P/w` 和 `1/w`，然后除以 `1/w` 来还原 `P`。**

这就是透视校正的核心。

### 透视校正插值的推导

假设三角形三个顶点在世界空间中的属性为 `P0, P1, P2`，对应的逆深度 `w0, w1, w2`（注意这里的 w 是 clip space w ≈ 1/z）。

在**三维空间**（即透视除法之前），`P` 应该随三维坐标线性变化。但在三维空间中，`P` 与 `w` 的关系是：

**透视校正插值公式**：

```
对于屏幕空间像素的重心坐标 (α, β, γ)：

1. 计算 1/z：
   1/z_pixel = α * (1/z0) + β * (1/z1) + γ * (1/z2)

2. 计算 P/z：
   P_over_z = α * (P0/z0) + β * (P1/z1) + γ * (P2/z2)

   （实际上：P0/z0 = P0 * (1/z0)）

3. 还原 P：
   P_pixel = P_over_z / (1/z_pixel)
```

或者用 w（≈ 1/z）的表达方式：

```
// 预计算
Pw0 = P0 * w0
Pw1 = P1 * w1
Pw2 = P2 * w2

// 屏幕空间插值
Pw_pixel = α * Pw0 + β * Pw1 + γ * Pw2   // 插值 P/w
w_pixel  = α * w0  + β * w1  + γ * w2    // 插值 1/w

// 还原
P_pixel = Pw_pixel / w_pixel
```

**直觉解释**：我们不是在屏幕空间插值 `P`，而是在三维空间插值 `P/z` 和 `1/z`，然后用 `P/z / (1/z) = P` 还原原始值。这样插值是"透视正确"的，因为 `P/z` 在 clip 空间是线性的。

### 为什么这样可以工作？

从齐次坐标的角度看：

- `P * w` = `P * (1/z)` 在 clip space 中随三维坐标线性变化
- 在 clip space 做重心坐标插值是数学正确的（有投影矩阵前的空间）
- 插值之后再除以 `w`（即做 perspective divide）回到 NDC

这本质上是**在透视除法之前做插值，除完之后恢复属性**。

### 与 Affine 插值的直观对比

**Affine**：
```
u = α*u0 + β*u1 + γ*u2     ← 屏幕空间直接插值 → 错误！
```

**Perspective-Correct**：
```
u/z = α*(u0/z0) + β*(u1/z1) + γ*(u2/z2)
1/z = α*(1/z0) + β*(1/z1) + γ*(1/z2)
u   = (u/z) / (1/z)         ← 三维空间插值后恢复 → 正确！
```

Affine 插值相当于强制 `z0 = z1 = z2`——即假设三角形上所有点的深度相同。这个假设在三角形垂直于视线时成立，但在倾斜于视线时完全失效。这就是为什么 PS1 时代的 3D 游戏在处理倾斜墙面时纹理扭曲的根本原因。

透视校正的额外开销：每个像素多 1 次除法（`Pw_pixel / w_pixel`）。在现代 GPU 上这微不足道，但在 PS1 时代确实是个影响性能的操作。

## 实现架构

### 整体数据流

```
[3D 场景定义]
     │ 倾斜四边形（4个顶点，带UV）
     ├── 绕X轴旋转25°
     └── 投影矩阵变换
           │
     [Clip Space 顶点]
     sx,sy (screen), invW=1/cw, u,v
           │
           ├── 左半 (x=0..399): Affine 插值
           ├── 右半 (x=400..799): Perspective-Correct 插值
           └── 中线: 黑色分隔线
           │
     [800×600 帧缓冲]
     depth buffer (z-buffer)
     checkerboard 纹理 (256×256, 8色, 双线性采样)
           │
     [PPM 文件输出]
     output.ppm
     output.png (convert)
```

### 关键数据结构

**Image (帧缓冲)**：
```cpp
struct Image {
    int w, h;
    vector<unsigned char> rgb;  // RGB 逐字节存储
};
```
800×600 帧缓冲，存储最终颜色。RGB 三个通道各占 1 字节，共 3 字节/像素 = 1.44 MB。

**DepthBuf (深度缓冲)**：
```cpp
struct DepthBuf {
    int w, h;
    vector<double> z;  // 双精度深度值
};
```
使用 double 精度的 Z-buffer。初始值设为 `-1e30`（负无穷），深度值越大表示越近（屏幕空间约定，与 OpenGL 的 -Z 朝向对应）。用于在光栅化过程中做深度测试。

**Texture (纹理)**：
```cpp
struct Texture {
    int w, h;
    vector<unsigned char> rgb;
    void sample(double u, double v, unsigned char& r, unsigned char& g, unsigned char& b) const;
};
```
256×256 的棋盘格纹理，8 种颜色交替排列，每个格子 32×32 像素。格子之间有黑色网格线。采样方式为**双线性插值**（bilinear sampling），使用 `floor()/fraction()` 定位四个相邻纹素并混合。

**TriSetup (三角形设置)**：
```cpp
struct TriSetup {
    double sx[3], sy[3];  // 屏幕空间坐标
    double u[3], v[3];     // UV 坐标
    double iw[3];          // 逆深度 1/w
};
```
在光栅化之前做 CCW（逆时针）保证，确保扫描线内像素的重心坐标非负。

### 两个光栅化函数

**`rasterAffine`** — Affine 插值版本（错误）：
```
for 每个像素在边界框内:
    w0, w1, w2 = 重心坐标（边缘函数）
    if 在三角形内:
        (α, β, γ) = 重心坐标（归一化）
        u = α*u0 + β*u1 + γ*u2      # 直接插值 UV
        v = α*v0 + β*v1 + γ*v2
        depth_test → tex.sample(u, v)
```

**`rasterCorrect`** — 透视校正版本（正确）：
```
预计算: uw0 = u0*iw0, vw0 = v0*iw0  (对每个顶点)

for 每个像素在边界框内:
    w0, w1, w2 = 重心坐标
    if 在三角形内:
        iw = α*iw0 + β*iw1 + γ*iw2     # 插值 1/z
        uw = α*uw0 + β*uw1 + γ*uw2     # 插值 u/z
        vw = α*vw0 + β*vw1 + γ*vw2     # 插值 v/z
        u = uw / iw                     # 还原 u
        v = vw / iw                     # 还原 v
        depth_test → tex.sample(u, v)
```

**关键差异**：`rasterCorrect` 多了一步插值后的除法 `uw/iw`，用来从齐次坐标还原 UV。这是额外的计算开销，但保证了透视正确性。

## 关键代码解析

### 投影矩阵构建

```cpp
static void projMatrix(double fov, double aspect, double n, double f, double m[4][4]) {
    memset(m, 0, sizeof(double)*16);
    double tanHalf = 1.0 / tan(fov * 0.5);
    m[0][0] = tanHalf / aspect;     // 水平缩放
    m[1][1] = tanHalf;              // 垂直缩放
    m[2][2] = -(f + n) / (f - n);   // 深度映射（右手系，-Z方向）
    m[2][3] = -2.0 * f * n / (f - n);
    m[3][2] = -1.0;                 // 将 -z_eye 写入 w_clip
}
```

这里使用右手坐标系（OpenGL 风格）。`m[3][2] = -1.0` 是关键——它把 `clip.w = -z_eye`，使得 `w` 携带深度信息。在 NDC 中，near 平面映射到 z=0，far 平面映射到 z=1。

**容易出错的地方**：如果 `m[3][2]` 不设为 -1，w 就不会携带深度，透视校正就失去了信息来源。

### 顶点变换

```cpp
static void xform(const double m[4][4], double x, double y, double z,
                  double& cx, double& cy, double& cz, double& cw) {
    cx = m[0][0]*x + m[0][1]*y + m[0][2]*z + m[0][3];
    cy = m[1][0]*x + m[1][1]*y + m[1][2]*z + m[1][3];
    cz = m[2][0]*x + m[2][1]*y + m[2][2]*z + m[2][3];
    cw = m[3][0]*x + m[3][1]*y + m[3][2]*z + m[3][3];
}
```

然后将 clip space 结果转换到屏幕空间：

```cpp
invW[i] = 1.0 / cw;                           // 预存逆深度
sx[i] = (cx*invW[i]*0.5 + 0.5) * W;           // NDC → 屏幕 (x)
sy[i] = (0.5 - cy*invW[i]*0.5) * H;           // NDC → 屏幕 (y) - 注意 Y 翻转
```

**关键细节**：
- `invW` 存储的是**逆深度** `1/cw = 1/(-z_eye + offset)`。这个值将在光栅化中被插值
- Y 坐标翻转：`0.5 - cy*invW[i]*0.5` 是因为 PPM 格式 y=0 在顶部，而 OpenGL NDC 的 y=-1 在底部
- `uv` 和 `invW` 按照顶点的顺序一对一对齐

### CCW 保证

```cpp
static TriSetup ensureCCW(...) {
    double area = edge(x0,y0, x1,y1, x2,y2);
    if (area < 0) {
        // 交换 v1 和 v2 使三角形为 CCW
        t.sx[0]=x0; t.sx[1]=x2; t.sx[2]=x1;
        // ... 对应属性也交换
    }
    // ...
}
```

如果三角形面积为负（顺时针），交换两个顶点使之为 CCW。这保证了光栅化中重心坐标始终非负。

**为什么这样做？** 边缘函数 `edge(ax,ay, bx,by, cx,cy) = (cx-ax)*(by-ay) - (cy-ay)*(bx-ax)` 对于 CCW 三角形内部点返回正值。确保 CCW 可以简化内部测试（只需 `w>=0` 不是 `min(w0,w1,w2) >= 0`）。

### Affine 光栅化 — 错误版本

```cpp
static void rasterAffine(img, zbuf, sx0,sy0,u0,v0,invW0, sx1,sy1,u1,v1,invW1, sx2,sy2,u2,v2,invW2, tex, xMin, xMax) {
    // ... CCW 保证
    double area = edge(...);  // 三角形面积
    double invA = 1.0 / area;
    
    int bx0 = max(xMin, (int)min({sx0,sx1,sx2}));   // 边界框
    int bx1 = min(xMax, (int)max({sx0,sx1,sx2}));
    int by0 = max(0, (int)min({sy0,sy1,sy2}));
    int by1 = min(H-1, (int)max({sy0,sy1,sy2}));
    
    for (int y=by0; y<=by1; y++) {
        for (int x=bx0; x<=bx1; x++) {
            double w0 = edge(sx1,sy1, sx2,sy2, x+0.5, y+0.5);  // 重心坐标
            double w1 = edge(sx2,sy2, sx0,sy0, x+0.5, y+0.5);
            double w2 = edge(sx0,sy0, sx1,sy1, x+0.5, y+0.5);
            if (w0 < 0 || w1 < 0 || w2 < 0) continue;
            
            double A = w0*invA, B = w1*invA, C = w2*invA;  // 归一化重心坐标
            double u = A*u0 + B*u1 + C*u2;                  // ❌ 直接插值 UV
            double v = A*v0 + B*v1 + C*v2;
            double iw = A*iw0 + B*iw1 + C*iw2;              // 插值深度（用于Z-buffer）
            if (!zbuf.write(x, y, 1.0/iw)) continue;
            tex.sample(u, v, r, g, b);
            img.set(x, y, r, g, b);
        }
    }
}
```

**问题所在**：`u = A*u0 + B*u1 + C*u2`——这是屏幕空间直接插值。当三角形有深度变化时，这种插值不符合透视投影的几何性质。

### 透视校正光栅化 — 正确版本

```cpp
static void rasterCorrect(img, zbuf, sx0,sy0,u0,v0,invW0, sx1,sy1,u1,v1,invW1, sx2,sy2,u2,v2,invW2, tex, xMin, xMax) {
    // ... CCW 保证
    
    // 预计算：将 UV 乘以逆深度（准备做齐次插值）
    double uw0 = u0 * invW0;  // u0 / z0
    double vw0 = v0 * invW0;  // v0 / z0
    double uw1 = u1 * invW1;
    double vw1 = v1 * invW1;
    double uw2 = u2 * invW2;
    double vw2 = v2 * invW2;
    
    for (int y=by0; y<=by1; y++) {
        for (int x=bx0; x<=bx1; x++) {
            // 重心坐标（同上）
            double w0 = edge(sx1,sy1, sx2,sy2, x+0.5, y+0.5);
            double w1 = edge(sx2,sy2, sx0,sy0, x+0.5, y+0.5);
            double w2 = edge(sx0,sy0, sx1,sy1, x+0.5, y+0.5);
            if (w0 < 0 || w1 < 0 || w2 < 0) continue;
            
            double A = w0*invA, B = w1*invA, C = w2*invA;
            double iw = A*iw0 + B*iw1 + C*iw2;   // 插值 1/z
            double uw = A*uw0 + B*uw1 + C*uw2;    // 插值 u/z
            double vw = A*vw0 + B*vw1 + C*vw2;    // 插值 v/z
            double u = uw / iw;                    // ✅ 还原：u = (u/z) / (1/z)
            double v = vw / iw;                    // ✅ 还原：v = (v/z) / (1/z)
            
            if (!zbuf.write(x, y, 1.0/iw)) continue;
            tex.sample(u, v, r, g, b);
            img.set(x, y, r, g, b);
        }
    }
}
```

**正确的关键在于最后的除法**：`u = uw / iw = (u/z) / (1/z) = u`。先在 clip space 线性空间插值 `u/z` 和 `1/z`，然后做 perspective divide 还原。

**性能成本**：每个像素多 2 次乘法（`u0*iw0` 等是预计算的）+ 1 次除法（`uw/iw`）。在实际应用中，除法是最昂贵的，但现代硬件对此有充分优化。

### 双线性纹理采样

```cpp
void sample(double u, double v, unsigned char& r, unsigned char& g, unsigned char& b) const {
    u = u - floor(u); v = v - floor(v);  // 取小数部分（tiling/repeat）
    double fx = u * w - 0.5;              // 半像素偏移（center of texel）
    double fy = v * h - 0.5;
    int x0 = ((int)floor(fx) % w + w) % w;
    int y0 = ((int)floor(fy) % h + h) % h;
    int x1 = (x0 + 1) % w, y1 = (y0 + 1) % h;
    double tx = fx - floor(fx);           // 插值权重
    double ty = fy - floor(fy);
    
    r = (uchar)lerp(lerp(at(x0,y0,0), at(x1,y0,0), tx),
                    lerp(at(x0,y1,0), at(x1,y1,0), tx), ty);
    // 同理对 g, b ...
}
```

双线性采样的顺序：先在 x 方向做两次线性插值（上下各一条扫描线），然后在 y 方向做一次线性插值（垂直混合）。这样可以得到平滑的纹理过渡，避免最近邻采样的马赛克块效应。

## 踩坑实录

### 坑1：CCW 重排序时属性不同步

**症状**：Affine 输出正常，Correct 输出完全扭曲，UV 坐标错位。

**错误假设**：`ensureCCW` 只交换了 `sx/sy`，没有同步交换 `u/v/iw`。

**真实原因**：当检测到三角形为顺时针（面积 < 0）时，需要交换 `v1` 和 `v2` 的所有属性——包括屏幕坐标、UV 坐标、逆深度。如果不交换，顶点顺序是 CCW 了，但属性表和坐标表对不上。

**修复方式**：在 `ensureCCW` 中统一交换：
```cpp
t.sx[0]=x0; t.sy[0]=y0; t.u[0]=u0; t.v[0]=v0; t.iw[0]=iw0;
t.sx[1]=x2; t.sy[1]=y2; t.u[1]=u2; t.v[1]=v2; t.iw[1]=iw2;  // v2 → v1
t.sx[2]=x1; t.sy[2]=y1; t.u[2]=u1; t.v[2]=v1; t.iw[2]=iw1;  // v1 → v2
```

### 坑2：Affine 光栅化也在插值 depth

**症状**：`rasterAffine` 函数中其实也用了 `invW` 做 depth test。既然用了 `invW`，是不是说明它也在做透视校正？

**错误假设**：用了 `invW` 就是透视校正。

**真实原因**：在 `rasterAffine` 中，`invW` **只用于 depth test**（判断哪个像素在前面），**不用于 UV 插值**。UV 插值使用的是 `A*u0 + B*u1 + C*u2`（屏幕空间直接插值）。深度测试的正确性不等于 UV 插值的正确性——它们解耦了。

`rasterCorrect` 中 `invW` 被同时用于 depth test **和** UV 还原（通过 `uw/iw`），这才是完全的透视校正。

**区分这两个概念很重要**：
- Depth 的透视校正：保证了遮挡关系正确
- UV 的透视校正：保证了纹理映射的形状正确

### 坑3：Y 坐标翻转

**症状**：纹理在水平方向正确但在垂直方向"反了"，上下颠倒。

**错误假设**：NDC → 屏幕的映射中 y 不需要翻转。

**真实原因**：OpenGL NDC 中 y 轴向上（-1 在底部，+1 在顶部），而 PPM 格式（和大多数图像格式）y=0 在顶部。如果不翻转：
```cpp
sy[i] = (cy*invW[i]*0.5 + 0.5) * H;  // 错误：顶部变成底部
```

应该写为：
```cpp
sy[i] = (0.5 - cy*invW[i]*0.5) * H;  // 正确：翻转 y
```

### 坑4：透视矩阵 near 值太小

**症状**：near=0.1 时，近处顶点的 `invW` 值极大，导致浮点精度问题，纹理采样出现跳变。

**错误假设**：near 值越小越好，"离相机越近才能看到更多细节"。

**真实原因**：near 值直接影响 `1/z` 的数值范围。当 `near=0.1, far=100` 时，`1/z` 的范围是 `[10, 0.01]`——差了 1000 倍，浮点精度不够。当它被插值到远端的三角形部分时，`u/z` 和 `1/z` 的商会有显著的舍入误差。

**修复方式**：设置 `near=0.5`，这样 `1/z` 范围变成 `[2, 0.01]`——只有 200 倍，精度充足。对于距离相机 5 单位远的四边形来说，0.5 的近裁剪面完全够用。

### 坑5：debug 版本的纹理采样被量化

**症状**：在开发阶段，临时写了只取左上角纹素的简化采样（"快速实现原型"），看到的效果像是最近邻采样——纹理有明显的锯齿。花了至少 15 分钟反复检查 UV 插值逻辑，发现完全正确后才意识到是采样函数的问题。

**错误假设**：纹理采样的正确性不会影响透视校正的验证——先实现最近邻，后面再换成双线性。

**真实原因**：在对比 Affine 和 Correct 效果时，双线性采样中的模糊效应会"软化"两者的差异，让人更难看到问题的本质。实际上应该在最终版本使用双线性采样（这是本文的做法），但如果你想最清晰地看到透视校正的效果，最近邻采样 + 高对比度棋盘格是最直观的。

**结论**：在开发阶段用最近邻验证核心逻辑，在最终版用双线性采样——两个版本都要跑一遍，确认效果符合预期。

## 效果验证与数据

### 视觉对比

`output.png` 展示了左右对比图：
- **左半（x=0..399）**：Affine 纹理映射 — 棋盘格在远端明显扭曲，格子变大或变形
- **右半（x=400..799）**：透视校正纹理映射 — 棋盘格在整个四边形上均匀自然
- 中间有黑色分隔线（2 像素宽）

### 量化指标（来自程序自动输出）

```
全图统计:
  mean=62.32  std=62.15
  ✅ mean ∈ [10, 240] (非全黑/全白)
  ✅ std > 5 (图像有内容变化)

左半 (Affine):    mean=63.15 std=62.05
右半 (Correct):   mean=61.50 std=62.23

像素级左右差异 (x vs x+400):
  meanDiff=21.38  std=29.45  max=255
  ✅ meanDiff >= 5.0 (两种方法产生了显著差异)

上半差异 (y=0..299): 20.35
下半差异 (y=300..599): 22.41
  ✅ bottomDiff > topDiff (深度越大，透视效应越明显)

File: 1440015 bytes (1406.3 KB)
  ✅ >= 10KB
```

**关键指标解读**：

1. **meanDiff=21.38**：在 0-255 的 RGB 范围内，平均每通道差异为 21.38——这意味着肉眼可见的差异，不是噪声。如果 `meanDiff < 5`，说明两种方法差异太小，透视效果不够明显。

2. **maxDiff=255**：存在完全不同的像素（通道最大差值 255），这意味着在图像的某些位置，两种方法采样的是完全不同的纹理区域。这是棋盘格高对比度纹理 + 透视变形的直接证据。

3. **下半差异 (22.41) > 上半差异 (20.35)**：因为四边形绕 X 轴旋转 25°，下半更靠近相机（深度更浅，w 更小），上半更远离相机（深度更大，w 更大）。下半的透视效应更明显，所以两种方法的差异更大。这符合透视校正的理论预测。

4. **右半 std=62.23 vs 左半 std=62.05**：右半的纹理更均匀（透视校正），但 std 略高是因为棋盘格图案在透视校正下覆盖整个屏幕时，远近的格子大小变化带来了更多的高频细节。

5. **文件大小 1406 KB**：800×600 的 PPM 文件，足够大的像素数据支持统计。

### 性能成本

在 800×600 分辨率下，两个三角形（各约 9.6 万像素的光栅化面积），透视校正方式的运行时间与 Affine 几乎无差异。这是因为：
- 预计算 `uw, vw` 是在光栅化循环外做的，每个三角形多 6 次乘法
- 循环内的额外成本：3 次加法（插值 `uw, vw, iw` vs 插值 `u, v, iw` 各 3 次）+ 2 次除法（`uw/iw`, `vw/iw` vs 无除法）

在现代 CPU 上，这些成本微小到不可测量。

## 总结与延伸

### 技术局限性

1. **四边形拆分为两个三角形**：本文的四边形是两个三角形的组合，UV 在两个三角形内各自做透视校正。两个三角形的透视校正效果各自正确，但三角形共享边的纹理过渡可能不连续——因为 UV 在两个三角形中分别对应不同的透视属性（虽然三个顶点确定一个透视变换，但分成两个三角形意味着用两组不同的变换）。在实际引擎中，如果有顶点法线/UV 变化等数据需要跨三角形连续，需要使用共享的透视校正。

2. **仅适用于光栅化**：透视校正是光栅化方法特有的需求。光线追踪不需要，因为光线在三维空间计算交点，直接在三维空间查询 UV（不需要在屏幕空间插值）。

3. **纹理过滤的独立性**：透视校正解决的是"采样哪个位置"的问题，不涉及"采样多个 texel 如何混合"。Mipmap 和各项异性过滤需要额外的计算。

4. **浮点精度**：极端的透视角度（比如从几乎平行于四边形的方向观看）会导致 `1/z` 的数值范围极大（近处 `1/z` ≈ ∞），浮点除法丢失精度。实际引擎会限制视角或使用更高精度。

### 可优化的方向

1. **透视校正的双线性优化**：在透视校正后做双线性采样会产生额外的精度误差。更好的方式是**先做采样再做校正**，但这需要更复杂的着色模型。例如在 Shader 中使用 `tex2Dproj` 一次性完成校正和采样。

2. **Perspective-Correct 的插值优化**：本文使用了逐像素的重心坐标计算（`edge()` 函数），但可以使用扫描线逐步更新重心坐标来减少除法。例如：
   ```
   // 扫描线优化（未实现）
   w0_next = w0 + delta_w0_x;  // 步进，避免逐像素计算 edge()
   ```
   
3. **结合 Mipmap**：透视校正会产生远处的纹素占据更少屏幕像素的效果。结合 Mipmap 可以让远处的纹理使用低分辨率层级，进一步提升视觉质量并减少带宽。

4. **非线性投影的校正**：本文使用的是标准透视投影（针孔相机模型），对于鱼眼镜头或全景式投影需要不同的校正公式。

### 与本系列其他文章的关联

- **02-20 纹理映射光线追踪器**：在光线追踪中，UV 是在三维交点直接计算的，不需要透视校正——但需要解决纹理过滤和抗锯齿问题
- **02-24 Parallax Mapping**：视差贴图依赖于正确的纹理 UV 坐标。在光栅化管线中，如果 UV