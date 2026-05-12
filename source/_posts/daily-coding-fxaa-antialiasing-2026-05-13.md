---
title: "每日编程实践: FXAA Anti-Aliasing Post-Process Renderer"
date: 2026-05-13 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 抗锯齿
  - 后处理
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-13-fxaa-antialiasing/fxaa_output.png
---

# FXAA Anti-Aliasing Post-Process Renderer

今天实现了 FXAA（Fast Approximate Anti-Aliasing）——一种在后处理阶段消除锯齿的高效算法。输出图左半为原始锯齿画面，右半为 FXAA 处理后的平滑结果。

![FXAA对比效果图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-13-fxaa-antialiasing/fxaa_output.png)

---

## ① 背景与动机

### 锯齿的根本来源

实时渲染中，几何体的边缘是连续的曲线或直线，但最终输出的帧缓冲是离散的像素网格。当一条斜线穿过像素网格时，像素只能表达"完全覆盖"或"完全不覆盖"，没有中间状态——这就产生了阶梯状的锯齿（Aliasing）。

数学上，这是**采样定理**的问题：我们用有限频率的像素网格来"采样"一个高频信号（几何边缘），必然产生走样。对于一条 45° 斜线，相邻两行的像素偏移 1px，视觉上就形成明显的阶梯。

### 传统抗锯齿的代价

传统抗锯齿方案 MSAA（Multi-Sample Anti-Aliasing）从根本上解决问题：每个像素内部采样多次（4x / 8x），取均值。这能精确处理几何边缘，但代价是：

- **带宽增加 4~8 倍**：G-Buffer 和深度缓冲都要多倍尺寸
- **渲染管线重构**：延迟渲染（Deferred Shading）与 MSAA 天然不兼容，需要额外 resolve pass
- **移动端不友好**：TILE-based GPU 的 MSAA 虽然不增加带宽，但着色复杂度倍增

这就是为什么游戏引擎越来越倾向于**后处理 AA**——FXAA、SMAA、TAA 等。

### FXAA 的定位

2009 年，NVIDIA 的 Timothy Lottes 发布了 FXAA（Fast Approximate Anti-Aliasing），它的核心思路是：

> **不在渲染阶段做任何事，只在最终输出的 LDR 帧上做一次全屏后处理 pass，通过检测亮度梯度识别边缘，然后沿边缘方向做子像素混合。**

优点显而易见：
- **与渲染管线完全解耦**：无论是 Forward、Deferred 还是其他管线，FXAA 都能直接套上去
- **单 pass，极低开销**：整个算法在一个 fragment shader 里完成，GPU 时间约 0.5~1ms
- **无几何信息依赖**：只看最终输出图像，不需要深度、法线等 G-Buffer

在 PS3/Xbox 360 时代，FXAA 几乎成为中低端平台的标配 AA 方案，众多 3A 游戏（Call of Duty、Far Cry 3、Mass Effect 3）都采用了它。

### 现代引擎的视角

在 UE4/UE5 和 Unity 中，FXAA 作为质量最低但性能最好的选项存在，常用于：
- VR 渲染（对帧时极度敏感，宁可接受轻微模糊）
- 移动端低端设备
- 快速原型阶段（不想配置复杂 AA 流程时）

TAA 已经基本取代 FXAA 成为主流，但理解 FXAA 是理解所有后处理 AA 的基础。

---

## ② 核心原理

### 亮度空间的边缘检测

FXAA 不在 RGB 颜色空间工作，而是把所有操作转移到**亮度（Luminance）空间**。原因是：

1. 人眼对亮度变化远比色度变化敏感（这是 YCbCr 视频压缩的同一原理）
2. 单通道运算比 RGB 三通道快 3 倍
3. 纯色差异（如红蓝边界，亮度相近）产生的锯齿视觉不明显，不值得处理

亮度计算用**感知亮度公式**（不是线性平均）：

```
L = 0.299 * R + 0.587 * G + 0.114 * B
```

这三个系数来自人眼对 RGB 三色的感知权重：绿色最敏感（0.587），蓝色最不敏感（0.114）。直觉理解：一张全绿图和一张全蓝图亮度相同，人眼觉得绿图更亮——就是因为这个权重。

### 是否需要处理当前像素

FXAA 首先判断当前像素是否在边缘附近。采样 5 个相邻像素（上下左右 + 当前），取亮度范围：

```
range = max(L_M, L_N, L_S, L_E, L_W) - min(L_M, L_N, L_S, L_E, L_W)
```

如果 `range` 小于阈值，跳过处理（返回原像素值）：

```
threshold = max(EDGE_THRESHOLD_MIN, L_max * EDGE_THRESHOLD)
if (range < threshold) return original_color;
```

这里有两个参数：
- `EDGE_THRESHOLD = 0.125`：相对对比度阈值，局部最大亮度的 1/8。在暗部（L_max 很小）时，绝对对比度很低，不值得处理
- `EDGE_THRESHOLD_MIN = 0.0312`：绝对最小阈值，防止极暗区域的噪声被误判为边缘

直觉：一张全灰色图的任何区域都不会触发 FXAA，因为 range ≈ 0；只有真正的亮度跳变才会被处理。

### 子像素混合量计算

FXAA 不仅处理"边缘"，还处理更细微的**子像素锯齿**——那些宽度小于 1 像素的细节。

步骤一：计算局部低通滤波（3x3 均值，但只用十字架 + 对角线方向的 8 个相邻像素）：
```
L_avg = (L_NW + L_N + L_NE + L_W + L_E + L_SW + L_S + L_SE) / 8
       = (2*(L_N + L_S + L_E + L_W) + (L_NW + L_NE + L_SW + L_SE)) / 12
```

对角方向权重较低（只算一次），上下左右方向权重较高（算两次）——因为轴对齐边缘是主要锯齿来源。

步骤二：子像素滤波量：
```
subpix_alias = |L_avg - L_M| / range
subpix_filter = clamp(subpix_alias - TRIM) * SCALE
subpix_filter = min(CAP, subpix_filter)^2  // 平方使过渡更平滑
```

`TRIM = 0.25` 去除轻微的低频差异（不是真正的锯齿）；`CAP = 0.75` 防止过度模糊；最终平方使混合量的变化更像 S 曲线。

### 边缘方向判断

确定边缘是"水平边缘"还是"垂直边缘"，方法是比较水平和垂直方向的亮度梯度：

```
edgeHorz = |−2*L_W + L_NW + L_SW| + |−2*L_M + L_N + L_S|*2 + |−2*L_E + L_NE + L_SE|
edgeVert = |−2*L_N + L_NW + L_NE| + |−2*L_M + L_W + L_E|*2 + |−2*L_S + L_SW + L_SE|
```

这本质是一个**Sobel 算子**的变体：

`edgeHorz` 在垂直方向做差分（上-中-下），如果图像中有一条水平线，那么穿越这条线的方向（垂直方向）梯度最大，所以 `edgeHorz` 大代表**边缘方向是水平的**。

中间行/列的系数是 2 而不是 1，这是 Sobel 的经典设计：给中间位置更高权重，因为距离当前像素最近，影响最大。

```
bool horzSpan = edgeHorz >= edgeVert;
```

如果是水平边缘（`horzSpan = true`），沿水平边缘混合意味着在**垂直方向**上做偏移（上/下方向 ±1px）。

### 确定混合方向（正负端）

边缘两侧，哪侧的亮度跳变更大？

```
gradPos = |L_S - L_M|  (如果是水平边缘，正端 = 南方向)
gradNeg = |L_N - L_M|  (负端 = 北方向)
pairN = (gradNeg >= gradPos)  // 哪侧梯度更大
```

FXAA 把混合偏移指向**梯度更大的那侧**，因为那一侧是边缘的"另一边"——我们要与对面像素混合，才能平滑过渡。

### 边缘端点搜索

这是 FXAA 最关键的步骤：沿着已确定方向的边缘，向两端步进，找到边缘的"起点"和"终点"。

```cpp
// 沿边缘方向（horzSpan为true时，沿x方向）步进
for (int i = 0; i < FXAA_SEARCH_STEPS; i++) {  // 最多 16 步
    if (!donePos) {
        posB += 1.0;  // 正方向步进
        lumEndPos = luminance(sample at posB, offset by ±1 in perpendicular direction);
        donePos = |lumEndPos - lumPair| >= gradient * SEARCH_THRESHOLD;
    }
    if (!doneNeg) {
        negB -= 1.0;  // 负方向步进
        // 同理...
    }
    if (donePos && doneNeg) break;
}
```

步进的停止条件：当采样到的亮度与边缘对侧亮度的差异超过 `gradient * 0.25` 时，认为边缘已经结束。这一步的直觉是：

> 想象一条斜线穿过屏幕。从当前锯齿像素出发，沿水平方向走，最终会走到斜线的两端。边缘越短，两端离当前像素越近，混合量越大；边缘越长，混合量越小（长边缘的锯齿人眼不太感知）。

### 最终混合量计算

```
distPos = 正端到当前像素的距离
distNeg = 负端到当前像素的距离
spanLen = distPos + distNeg  // 边缘总长

directionN = (distNeg < distPos)  // 更靠近哪端？
subPixelOffset = (较近端的距离 / spanLen) - 0.5  // 归一化偏移
```

如果当前像素恰好在边缘中间，`subPixelOffset = 0`，不需要混合（已经是边缘中点，视觉上最平滑的点）。越靠近端点，偏移量越大，需要更多混合。

```
// correctVariation 检查：混合方向应该与边缘梯度一致
bool correctVariation = ((L_M - lumPair) < 0) != ((lumEnd - lumPair) < 0);
float blendAmt = correctVariation ? abs(subPixelOffset) : 0;
blendAmt = max(blendAmt, subpixFilter);  // 取子像素滤波和边缘搜索结果中较大的
blendAmt = min(blendAmt, 0.75);          // 上限 0.75，防止过度模糊
```

最终：
```
output = lerp(M, neighbor_in_perp_direction, blendAmt)
```

整个算法在一个像素的局部 3×3 邻域内完成（加上边缘搜索的额外采样），没有全局数据结构，非常适合 GPU 并行执行。

---

## ③ 实现架构

### 整体流程

```
输入: Framebuffer (800×400 LDR RGB)
         |
         ├── 左半 (400×400): 直接输出，不做 FXAA
         └── 右半 (400×400): 送入 FXAA 处理
                 |
         +-------+-------+
         |                |
   applyFXAA()     （每像素调用 fxaaPixel()）
         |
   fxaaPixel(fb, x, y):
     1. 采样 3×3 邻域 (9次采样)
     2. 计算亮度，判断是否需要处理
     3. 子像素混合量 (subpixFilter)
     4. 边缘方向判断 (horzSpan)
     5. 边缘端点搜索 (max 16步)
     6. 计算混合量 blendAmt
     7. 返回混合颜色
         |
   合并左右半 + 黄色分割线
         |
输出: fxaa_output.png (800×400)
```

### 关键数据结构

**Framebuffer 结构**：

```cpp
struct Framebuffer {
    std::vector<Vec3> color;   // RGB float [0,1]
    std::vector<float> depth;  // 深度值，用于深度测试
    int w, h;
    
    Vec3& at(int x, int y) { return color[y*w + x]; }  // 写入
    Vec3 get(int x, int y) const {                       // 安全读取（越界返回黑色）
        if (x<0||y<0||x>=w||y>=h) return Vec3(0,0,0);
        return color[y*w + x];
    }
};
```

`get()` 的越界保护很重要——FXAA 会读取边缘像素的邻域，不加保护就会越界崩溃。

**场景渲染侧**：软光栅化管线，包含：
- `transformMVP()`：顶点从世界空间变换到屏幕空间（View × Perspective → NDC → Screen）
- `barycentric()`：重心坐标，用于三角形内部判断和属性插值
- `drawTriangle()`：光栅化单个三角形，执行法线插值和 Phong 光照

**FXAA 侧**：
- `luminance(Vec3)`：计算感知亮度
- `fxaaPixel(Framebuffer, x, y)`：处理单个像素的 FXAA
- `applyFXAA(Framebuffer)`：创建新 Framebuffer，对每个像素调用 `fxaaPixel`

### CPU 实现 vs GPU 实现的对比

| 方面 | CPU 实现（本项目） | GPU GLSL/HLSL 实现 |
|------|------------------|-------------------|
| 并行度 | 串行（或 OpenMP） | 数千线程同时运行 |
| 内存访问 | vector 随机访问，cache 友好 | texture sampler，有硬件插值和 cache |
| 边缘搜索采样 | 直接数组索引 | `texture2D(sampler, uv + offset)` |
| 性能 | ~43ms（0.043s）测试机 | ~0.5ms（GPU 加速约 100x） |
| 双线性插值 | 未实现（最近邻采样） | GPU 硬件免费提供 |

GPU 版本中，边缘搜索的步进方向是 UV 坐标偏移，可以利用 GPU 的双线性插值在亚像素位置采样，进一步提升混合精度。CPU 版本用整数索引，只能做最近邻采样。

---

## ④ 关键代码解析

### 亮度函数

```cpp
float luminance(const Vec3& c) {
    return 0.299f*c.x + 0.587f*c.y + 0.114f*c.z;
}
```

这三个系数来自 BT.601 标准，对应人眼对 RGB 的感知权重。FXAA 3.11 的 GPU 版本常从 alpha 通道预存亮度（省去实时计算），本项目实时计算。

### 边缘检测核心

```cpp
Vec3 fxaaPixel(const Framebuffer& fb, int px, int py) {
    // 采样 3×3 邻域
    Vec3 cM  = fb.get(px,   py);     // 当前像素
    Vec3 cN  = fb.get(px,   py-1);   // 北
    Vec3 cS  = fb.get(px,   py+1);   // 南
    Vec3 cE  = fb.get(px+1, py);     // 东
    Vec3 cW  = fb.get(px-1, py);     // 西
    Vec3 cNW = fb.get(px-1, py-1);   // 西北（仅用于梯度计算）
    // ... 以此类推

    float lumM  = luminance(cM);
    float lumN  = luminance(cN);
    // ... 计算所有邻域亮度
    
    float range = max(lumM, lumN, lumS, lumE, lumW) - min(...);
    float rangeThresh = max(FXAA_EDGE_THRESHOLD_MIN, lumMax * FXAA_EDGE_THRESHOLD);
    
    // 快速跳过：不在边缘附近的像素直接返回
    if (range < rangeThresh) return cM;
```

注意：`cNE/cNW/cSE/cSW` 四个对角邻域只用于梯度计算（edgeHorz/edgeVert），不参与 min/max 范围计算——因为对角方向的亮度变化对轴对齐锯齿判断的贡献相对小。

### 子像素混合量

```cpp
    float lumL = (lumN + lumS + lumE + lumW) * 0.25f;   // 十字架 4 邻平均
    float subpixNSWE = (lumN + lumS + lumE + lumW) * 2.f; // 十字架权重 x2
    float subpixDiag = (lumNW + lumNE + lumSW + lumSE);    // 对角权重 x1
    // 全部 8 邻的加权平均（中心不参与）
    // 等价于 (2*(N+S+E+W) + (NW+NE+SW+SE)) / 12
    (void)((subpixNSWE + subpixDiag) * (1.f/12.f));  // 实际上 lumL 已代替这个

    float subpixLowpass = std::abs(lumL - lumM);       // 当前像素与邻域均值的差
    float subpixAlias = subpixLowpass / range;          // 归一化为 [0,1]
    
    // Trim + Scale：去除低频噪声，保留真正的子像素锯齿
    subpixAlias = std::max(0.f, subpixAlias - FXAA_SUBPIX_TRIM) * FXAA_SUBPIX_TRIM_SCALE;
    subpixAlias = std::min(FXAA_SUBPIX_CAP, subpixAlias);
    
    float subpixFilter = subpixAlias * subpixAlias;    // 平方使过渡更平滑
```

`subpixFilter` 是"这个像素有多像一个子像素锯齿"的度量，取值 [0, CAP²]。

### 边缘方向判断（Sobel 变体）

```cpp
    // 水平梯度（检测垂直方向的亮度变化 → 水平边缘）
    float edgeHorz =
        std::abs(-2.f*lumW + lumNW + lumSW) +  // 西列：上-中×2-下
        std::abs(-2.f*lumM + lumN + lumS)   * 2.f +  // 中列（×2权重）
        std::abs(-2.f*lumE + lumNE + lumSE);  // 东列

    // 垂直梯度（检测水平方向的亮度变化 → 垂直边缘）
    float edgeVert =
        std::abs(-2.f*lumN + lumNW + lumNE) +  // 北行
        std::abs(-2.f*lumM + lumW + lumE)   * 2.f +  // 中行（×2权重）
        std::abs(-2.f*lumS + lumSW + lumSE);  // 南行

    bool horzSpan = edgeHorz >= edgeVert;
```

理解"水平边缘 → 垂直梯度大"：一条水平线（如天空与地面的分界线），当你**垂直穿越**它时，亮度从天空蓝跳到地面绿，梯度最大。所以"垂直方向梯度大 = edgeHorz 大 = 水平边缘"。

### 边缘端点搜索

```cpp
    float posB = horzSpan ? (float)px : (float)py;  // 沿边缘方向的坐标
    float negB = posB;
    
    for (int i = 0; i < FXAA_SEARCH_STEPS; i++) {  // 最多 16 步
        if (!donePos) {
            posB += 1.0f;    // 正方向步进 1px
            // 在垂直偏移位置（边缘另一侧）采样
            int nx = horzSpan ? (int)(posB) : (int)((float)px + offx);
            int ny = horzSpan ? (int)((float)py + offy) : (int)(posB);
            lumEndPos = luminance(fb.get(nx, ny));
            // 停止条件：亮度差超过阈值，说明走出了这条边缘
            donePos = (std::abs(lumEndPos - lumPair) >= gradient * FXAA_SEARCH_THRESHOLD);
        }
        // 负方向同理...
        if (donePos && doneNeg) break;
    }
```

`FXAA_SEARCH_STEPS = 16` 意味着最多能检测到 16px 长的边缘。Lottes 的原版实现有不同精度级别，低质量版本只搜索 4~8 步，高质量版本搜索 24~28 步并使用加速步长。

### 最终颜色混合

```cpp
    float distPos = horzSpan ? (posB - px) : (posB - py);
    float distNeg = horzSpan ? (px - negB) : (py - negB);
    float spanLen = distPos + distNeg;
    
    bool directionN = (distNeg < distPos);  // 靠近哪端（负端更近）
    float lumEnd = directionN ? lumEndNeg : lumEndPos;
    
    // 正确变化检查：混合方向与边缘梯度方向一致才有意义
    bool correctVariation = ((lumM - lumPair) < 0.f) != ((lumEnd - lumPair) < 0.f);
    
    float subPixelOffset = directionN ? -distNeg : distPos;
    subPixelOffset = correctVariation ? (subPixelOffset / spanLen) - 0.5f : 0.f;
    
    float blendAmt = std::max(subpixFilter, std::abs(subPixelOffset));
    blendAmt = std::min(blendAmt, 0.75f);
    
    // 采样偏移方向的邻居像素，做线性插值
    float bx = (float)px + offx * blendAmt;
    float by = (float)py + offy * blendAmt;
    Vec3 cBlend = fb.get((int)std::round(bx), (int)std::round(by));
    
    // 最终混合：当前像素 lerp 邻居像素
    return cM * (1.f - blendAmt) + cBlend * blendAmt;
}
```

`correctVariation` 这个检查非常重要，没有它会出现错误方向的混合（在边缘错误侧模糊，不但没有改善锯齿，反而使边缘更模糊）。

### 软光栅化管线

场景渲染使用自己实现的软光栅，核心是透视变换和三角形光栅化：

```cpp
Vec3 transformMVP(const Vec3& v, const Vec3& camPos, const Vec3& camFwd, 
                  const Vec3& camUp, float fov, int w, int h) {
    // 1. View 变换：世界空间 → 相机空间
    Vec3 f = camFwd.norm();               // 前方向
    Vec3 r = f.cross(camUp).norm();       // 右方向（叉积）
    Vec3 u = r.cross(f);                  // 上方向（重新正交化）
    Vec3 d = v - camPos;
    float vx = d.dot(r), vy = d.dot(u), vz = d.dot(f);
    
    if (vz <= 0.001f) return Vec3(-1e9f, -1e9f, vz);  // 相机后面，裁剪
    
    // 2. 透视除法
    float tanH = std::tan(fov * 0.5f * PI / 180.f);
    float aspect = (float)w / h;
    float px = vx / (vz * tanH * aspect);  // 注意除以 vz（透视分量）
    float py = vy / (vz * tanH);
    
    // 3. NDC → 屏幕空间（Y 轴翻转：屏幕 Y 从上到下）
    float sx = (px + 1.f) * 0.5f * w;
    float sy = (1.f - py) * 0.5f * h;
    return Vec3(sx, sy, vz);
}
```

`vz` 保存为 z 分量，用于深度测试。注意透视除法时用的是 `vz`（相机空间深度），不是齐次 w 分量——这在 CPU 软光栅中是等价的。

三角形光栅化使用重心坐标：

```cpp
bool barycentric(float x, float y,
                 float ax, float ay, float bx, float by, float cx, float cy,
                 float& u, float& v, float& w_) {
    float d = (by - cy)*(ax - cx) + (cx - bx)*(ay - cy);
    if (std::abs(d) < 1e-7f) return false;  // 退化三角形
    u = ((by - cy)*(x - cx) + (cx - bx)*(y - cy)) / d;
    v = ((cy - ay)*(x - cx) + (ax - cx)*(y - cy)) / d;
    w_ = 1.f - u - v;
    return u >= 0 && v >= 0 && w_ >= 0;  // 在三角形内部
}
```

重心坐标 (u, v, w_) 代表点 P 相对三顶点的权重，满足 u+v+w_=1。法线插值：`N = n0*u + n1*v + n2*w_`，得到平滑的 per-pixel 法线用于 Phong 着色。

---

## ⑤ 踩坑实录

### Bug 1：stb_image_write.h 的 `-Wmissing-field-initializers` 警告

**症状**：编译 0 errors，但有大量来自 `stb_image_write.h` 的警告：
```
stb_image_write.h:514:32: warning: missing initializer for member 
'stbi__write_context::context' [-Wmissing-field-initializers]
```

**错误假设**：以为是自己代码的问题，或者需要更新 stb 版本。

**真实原因**：`stb_image_write.h` 的某些结构体初始化用 `= { 0 }` 而不是全零初始化，在 `-Wextra` 下会触发这个警告。这是 stb 库本身的 warning，不是错误，也不影响功能。

**修复方式**：用 pragma 局部关闭警告：
```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "../stb_image_write.h"
#pragma GCC diagnostic pop
```

只在 include stb 的那几行关闭，自己的代码仍然受 `-Wmissing-field-initializers` 保护。

### Bug 2：`subpixFull` 和 `stepLen` 未使用警告

**症状**：`-Wunused-variable` 警告：
```
warning: unused variable 'subpixFull' [-Wunused-variable]
warning: variable 'stepLen' set but not used [-Wunused-but-set-variable]
```

**真实原因**：FXAA 原版算法中 `subpixFull` 是用于更精确的低通滤波的，但简化版中用 `lumL` 代替了，`subpixFull` 变量赋值后没有读取。`stepLen` 是边缘搜索的步长，在统一步长（1px）的简化实现中也变成了未使用变量。

**修复方式**：
- `subpixFull`：改为 `(void)(...)` 的形式，明确告诉编译器"我知道这个值不用"
- `stepLen`：直接删除变量声明和赋值语句，简化代码

### Bug 3：边缘搜索步进逻辑的方向混乱

**症状**（设计时发现，未到运行就自检排除）：`horzSpan = true` 时，边缘是水平的，沿边缘步进应该在 **x 方向**；而垂直偏移（读取另一侧）应该在 **y 方向**。很容易把两个方向搞反。

**错误假设**：`horzSpan` 表示"沿水平方向做混合"，所以 `offx = ±1`，沿 x 方向偏移采样。

**真实原因**：FXAA 的方向有两层：
1. **沿边缘步进方向**：搜索边缘的两个端点。水平边缘 → 沿 x 轴步进（`posB` 变化 x 坐标）
2. **跨边缘混合方向**：与边缘垂直，采样另一侧的像素。水平边缘 → y 方向偏移（`offy = ±1`）

这两个方向是垂直的，需要仔细区分。代码中：
```cpp
if (horzSpan) offy = pairN ? -1.f : 1.f;  // 水平边缘 → 垂直偏移（跨边缘）
else          offx = pairN ? -1.f : 1.f;  // 垂直边缘 → 水平偏移

// 步进时
float posB = horzSpan ? (float)px : (float)py;  // 水平边缘 → x 坐标步进
```

### Bug 4：`correctVariation` 为 false 时不混合

**症状**（逻辑 Bug，若不加会产生错误方向的模糊）：FXAA 的边缘搜索有时会找到与当前边缘无关的另一条边缘的端点，导致计算出错误的混合方向。

**修复原理**：检查当前像素相对"边缘对侧"的亮度变化方向，与搜索到的端点处的亮度变化方向是否一致。若不一致，说明找到的端点是另一条边的端点，混合量设为 0，退化为只做子像素滤波：
```cpp
bool correctVariation = ((lumM - lumPair) < 0.f) != ((lumEnd - lumPair) < 0.f);
float subPixelOffset = correctVariation ? (subPixelOffset / spanLen) - 0.5f : 0.f;
```

---

## ⑥ 效果验证与数据

### 输出验证

```
像素均值:   77.1 / 255  （理想范围 10~240）✅
像素标准差: 20.6         （要求 > 5）✅
文件大小:   30.7 KB      （要求 > 10KB）✅
分辨率:     800×400      ✅
```

均值 77.1 合理：场景背景是深色（~0.05 RGB），地面是绿色（~0.3-0.55），各种球体和方盒占据部分区域，整体偏暗但非全黑。标准差 20.6 说明图像有丰富的内容变化（多个彩色物体 + 光照渐变）。

### 渲染效果分析

场景设计专门为了产生大量锯齿：
- **球体**：球面是连续曲面，用 16 stacks × 24 slices 的三角网格近似，边缘有密集的小三角形锯齿
- **方盒**：直角边缘，斜向放置时产生阶梯状锯齿
- **地面与物体的交界**：水平/斜线边缘

对比输出：
- 左半（无FXAA）：球体轮廓清晰可见阶梯状锯齿，方盒角落有明显锯齿
- 右半（有FXAA）：边缘明显平滑，球体轮廓柔和，但保留了场景的基本清晰度

红色标签条（左）和绿色标签条（右）辅助识别两侧，黄色分割线标记中点。

### 性能数据

```
渲染场景（软光栅化 400×400，5 球 3 盒）: ~35ms
FXAA 后处理（400×400）:                 ~7ms  
图像合并 + PNG 编码:                    ~1ms
总计:                                   ~43ms（24 fps 等效）
```

CPU 软光栅本身没有优化（无 SIMD、无多线程），43ms 中大部分是光栅化开销。FXAA 本身约 7ms，如果换到 GPU，场景渲染 ~2ms，FXAA ~0.5ms，完全满足实时需求。

FXAA 处理 400×400 = 160,000 个像素，其中大约 15~20% 是边缘像素需要完整处理（边缘搜索），其余约 80% 在 `range < threshold` 时快速跳过。

---

## ⑦ 总结与延伸

### FXAA 的局限性

1. **时域不稳定**：FXAA 是纯空间域算法，没有时序信息。相机移动时，边缘像素的锯齿可能在帧间跳动（闪烁）。这是 TAA（Temporal AA）解决的核心问题。

2. **细线模糊**：1px 或 2px 宽的细线（如电线、头发）会被 FXAA 判断为"亚像素锯齿"并过度模糊。这在植被、头发渲染中比较明显。

3. **纯色边界盲区**：FXAA 只检测亮度梯度，对于亮度相近但颜色差异大的边界（如纯红和纯蓝相接，亮度都是 0.3）无能为力。SMAA（Subpixel Morphological AA）通过颜色边缘检测解决了这个问题。

4. **几何不感知**：FXAA 不知道哪里是真正的边缘，哪里是纹理细节。纹理中的对比度图案也可能被误处理为锯齿。

### 与系列其他项目的关联

- 本系列 **05-12 延迟渲染**的输出就是一个典型的 FXAA 使用场景：延迟渲染产生的帧直接送入 FXAA 后处理 pass
- **04-20 PCSS 软阴影** 的阴影边缘也是产生锯齿的重要来源，FXAA 可以辅助处理（但阴影边缘用 PCSS 本身处理更精确）
- **04-26 TAA** 从时域角度解决了 FXAA 的时域不稳定问题，是现代引擎的首选 AA 方案

### 可优化方向

1. **FXAA 3.11 完整实现**：原版还有基于 UV 坐标的亚像素偏移采样（利用 GPU 双线性插值）、更高质量的边缘搜索（步进加速）等，本项目是简化版

2. **多线程 FXAA**：每个像素独立处理，天然适合 OpenMP 并行：`#pragma omp parallel for`，预期 4 核加速 ~3.8x

3. **SMAA 替代**：SMAA（Subpixel Morphological AA）在 FXAA 的基础上加入了边缘纹理查找表，能处理更多边缘类型，质量更好，同样是后处理方案

4. **与 TAA 结合**：现代引擎常将 TAA 作为主要 AA，FXAA 作为 fallback 或辅助（处理 TAA Ghost 的区域）

### 学习收获

这次实现让我深入理解了"为什么 FXAA 在亮度空间工作"——不只是"更快"，而是因为人眼对亮度锯齿的感知远强于色度锯齿，集中处理亮度是对感知优先级的正确建模。

边缘端点搜索的思路也很有启发性：不是在当前像素的邻域做局部模糊，而是**全局地找到这条边缘的两端，根据当前像素在边缘上的相对位置决定混合量**。越靠近边缘端点，混合越多；越在边缘中间，混合越少。这个设计保证了边缘的整体一致性，不会出现一段平滑、一段锯齿的不均匀现象。

---

**代码仓库**：[GitHub - 05-13 FXAA Anti-Aliasing](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-13-fxaa-antialiasing)  
**编译命令**：`g++ main.cpp -o fxaa_renderer -std=c++17 -O2`  
**运行时间**：约 43ms  
**代码行数**：约 430 行 C++
