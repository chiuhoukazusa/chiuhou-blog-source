---
title: "每日编程实践: Cascaded Shadow Maps (CSM) Renderer"
date: 2026-05-08 05:30:00
tags:
  - 每日一练
  - 图形学
  - 实时渲染
  - 阴影
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-08-csm/csm_output.png
---

# 每日编程实践：Cascaded Shadow Maps (CSM) Renderer

> 本文是「每日编程实践」系列第 N 篇。  
> 技术栈：C++17 · 纯 CPU 软光栅化 · 无任何 GPU/OpenGL 依赖  
> 代码仓库：[daily-coding-practice/2026/05/05-08-csm](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-08-csm)

---

## ① 背景与动机

### 阴影的本质：可见性问题

在实时渲染中，阴影的本质是"光源视角下的可见性"：一个像素是否在阴影中，取决于从该像素向光源连线是否被遮挡。最朴素的做法是从光源向场景发出射线，但射线追踪对实时场景代价太高——于是 Shadow Map 技术应运而生（Williams, 1978）。

Shadow Map 的核心思路很简单：**先从光源视角渲染一张深度图，再从摄像机视角渲染时，将每个片元变换到光源空间，与深度图中的值比较，如果比深度图深，说明在阴影中。**

这个方法在理论上非常优雅，但有一个致命问题：**透视失真（Perspective Aliasing）**。

### 透视失真：Shadow Map 的天敌

考虑一个简单的例子：摄像机的视锥从 near=0.1 延伸到 far=1000。如果我们用一张 1024×1024 的 Shadow Map 来覆盖这整个视锥：
- 靠近摄像机的区域（z=1~10）占屏幕像素的绝大部分，但在 Shadow Map 上只占很小一块
- 远离摄像机的区域（z=900~1000）占屏幕像素的少数，但可能占 Shadow Map 的相当一部分

结果就是：**近处的阴影像素化严重（shadow acne、锯齿），远处反而浪费了大量 Shadow Map 分辨率。**

量化一下这个问题：假设视锥从 0.5 到 25，纵深比为 50x。如果用 512×512 的 Shadow Map 均匀覆盖，近处每个屏幕像素对应的 Shadow Map 纹素数量约为 1/50，远低于 1:1 的理想比例。

### 游戏引擎的标准答案：CSM

Cascaded Shadow Maps（级联阴影贴图）是现代游戏引擎中处理方向光阴影的**标准方案**，被 Unreal Engine、Unity、Godot、CryEngine 等所有主流引擎使用。

CSM 的核心思路：**将摄像机视锥沿 Z 轴分割为多个子视锥（cascade），每个子视锥单独生成一张 Shadow Map，近处用精度高的 Shadow Map，远处用精度低的 Shadow Map。**

在 Unreal Engine 5 中，CSM 与 Virtual Shadow Maps 并存：
- 动态场景用 CSM（低延迟，实时更新）
- 静态场景用 VSM（更高质量，允许预计算）

Unity URP/HDRP 中，CSM 分 4 级，可在 QualitySettings 中配置每级分辨率。

**今天的项目就是从零实现一个 3 级 CSM 渲染器，包含 PCF 软阴影和级联调试可视化。**

---

## ② 核心原理

### 2.1 Shadow Map 基础回顾

从光源位置生成深度图，光源 MVP 矩阵：

```
P_ndc = M_proj × M_view × M_model × P_world
d_light = P_ndc.z / P_ndc.w  ∈ [-1, 1]，映射到 [0, 1]
```

在摄像机渲染时，对每个 world-space 片元 P：

```
P_light = LightVP × P_world
u = P_light.x / P_light.w * 0.5 + 0.5  ∈ [0, 1]
v = P_light.y / P_light.w * 0.5 + 0.5  ∈ [0, 1]
d_pixel = P_light.z / P_light.w * 0.5 + 0.5  ∈ [0, 1]
d_shadow = ShadowMap.sample(u, v)

if d_pixel > d_shadow + bias:  # 比 Shadow Map 记录的更深
    in_shadow = true
```

**直觉理解**：`d_shadow` 是"光源能看到的最近物体深度"，`d_pixel` 是"当前片元的深度"。如果当前片元比光源能看到的最近物体更深，说明它在遮挡物背后，即在阴影中。

### 2.2 Practical Split Scheme：最优分割方案

视锥分割是 CSM 的核心设计决策。常见的分割方案有三种：

**方案 A：均匀分割（Uniform Split）**
```
z_i = z_near + (z_far - z_near) * i/N
```
优点：实现简单。缺点：近处精度不足，远处浪费。

**方案 B：对数分割（Logarithmic Split）**
```
z_i = z_near * (z_far / z_near)^(i/N)
```
优点：理论上与透视失真完美对应。缺点：相邻 cascade 大小差异过大，Shadow Map 利用率不均，容易产生 cascade 边界处的跳变。

**方案 C：Practical Split Scheme（实用分割，Engel 2007）**
```
z_log = z_near * (z_far / z_near)^(i/N)
z_uni = z_near + (z_far - z_near) * i/N
z_i = λ * z_log + (1 - λ) * z_uni
```

其中 λ（lambda）是混合系数，取值 [0, 1]：
- λ = 0：纯均匀分割
- λ = 1：纯对数分割
- λ = 0.5~0.7：实用推荐值（本项目用 0.6）

**直觉理解**：对数分割在近处切得很碎（好事），但远处 cascade 跨度太大导致单个 Shadow Map 覆盖范围过宽、精度又差了。均匀分割反过来。Practical Split 是两者的加权平均，在近处稍微比对数分割大一些（避免 cascade 过多切换），在远处稍微比均匀分割小一些（保证远处也有一定精度）。

以本项目的参数为例（near=0.5, far=25, N=3, λ=0.6）：
```
对数分割: [0.5, 1.97, 7.81, 25]
均匀分割: [0.5, 8.83, 17.17, 25]
实用分割: [0.5, 4.57, 10.80, 25]
```

Cascade 0 覆盖 0.5~4.57（近处，精细），Cascade 1 覆盖 4.57~10.80（中距离），Cascade 2 覆盖 10.80~25（远处）。

### 2.3 子视锥计算：从 NDC 反向推算世界空间角点

给定分割深度 z_near_i 和 z_far_i，我们需要知道这个子视锥的 8 个世界空间角点（用于计算紧凑的光源正交矩阵）。

方法：用这个子视锥的投影矩阵和视图矩阵的逆，将 NDC 空间的 8 个角点变换回世界空间：

```cpp
// NDC 8 角点：(±1, ±1, ±1)
for x in {-1, +1}:
  for y in {-1, +1}:
    for z in {-1, +1}:
      ndc = (x, y, z, 1)
      world = inv(Proj_i × View) × ndc
      world /= world.w  // 透视除法
```

其中 `Proj_i` 是子视锥的透视投影矩阵（用 z_near_i 和 z_far_i 构造）。

**为什么要计算角点**？因为光源正交投影矩阵需要"紧紧包裹"子视锥角点的 AABB（在光源空间中），这样才能最大化 Shadow Map 的分辨率利用率。如果光源投影范围设得过大，Shadow Map 的分辨率就会被稀释。

### 2.4 光源正交投影的 AABB 计算

将 8 个世界空间角点变换到光源视图空间（Light View Space），计算 AABB：

```
P_light = LightView × P_world

min_x = min(P_light.x)  max_x = max(P_light.x)
min_y = min(P_light.y)  max_y = max(P_light.y)
min_z = min(P_light.z)  max_z = max(P_light.z)
```

然后构造正交投影矩阵：
```
M_ortho = ortho(min_x, max_x, min_y, max_y, -max_z - margin, -min_z + margin)
```

注意 Z 方向需要加 margin（本项目用 ±2），因为场景中可能有在子视锥外但会投影到视锥内的遮挡物。忽略这个 margin 会导致遮挡物被 Shadow Map 剪裁掉，产生阴影缺失。

**直觉理解**：光源"看"的范围不能只包含摄像机子视锥的几何体，还要包含视锥外侧但会在视锥内产生阴影的遮挡物——典型例子是一棵在摄像机视锥外的大树，但它的影子投射到视锥内。这个 margin 就是为这类情况预留的空间。

### 2.5 PCF 软阴影：用滤波代替硬边

原始 Shadow Map 生成硬边阴影（Binary：要么在阴影里，要么不在），视觉上很生硬。PCF（Percentage Closer Filtering）通过在周围多个采样点比较，取平均值，产生软阴影过渡。

PCF 的关键：**过滤操作作用在深度比较结果上，而不是深度值本身**。

```
对 NxN 邻域中每个像素 (u+du, v+dv)：
    s = 1.0 if (pixel_depth <= shadow_map[u+du][v+dv] + bias) else 0.0
shadow_factor = sum(s) / (N*N)
```

本项目使用 3×3 PCF（9 个采样点）。shadow_factor ∈ [0, 1]，值越小越暗。

**直觉理解**：硬阴影是"要么 0 要么 1"的二值结果。PCF 在阴影边界附近，有些相邻 texel 在阴影里、有些不在，平均后得到 0.3、0.6 等中间值，从而形成软过渡。

注意 PCF 的 N×N 采样会带来 N² 倍的性能开销，在实际引擎中通常用泊松采样或随机旋转 Poisson Disk 来减少采样数同时保持软阴影质量。

### 2.6 Shadow Acne 与深度偏移（Bias）

一个常见问题：自阴影（Shadow Acne）。由于 Shadow Map 分辨率有限，一个片元可能被自己所在面的相邻 Shadow Map 纹素遮挡，产生条纹状阴影噪点。

解决方法：加深度偏移。本项目中：
```cpp
float shadow_factor = (pixel_depth <= shadow_map_depth + 0.001f) ? 1.0f : 0.0f;
```

偏移量 0.001 是个经验值。偏移太小：acne 消不掉。偏移太大：Peter Panning（物体"飘起来"，脱离地面阴影）。在实际引擎中，偏移量通常与 NdotL 相关（法线与光线夹角越大，偏移量越大），这就是 "slope-scale depth bias"。

---

## ③ 实现架构

### 3.1 整体渲染管线

```
1. 构建场景（Cornell Box 变体 + 11 个几何体）
   ↓
2. 计算 CSM 分割参数
   - computeSplits(near, far, N, λ) → 分割深度数组
   - 对每个 cascade：
     a. getFrustumCorners() → 8 个世界空间角点
     b. computeLightOrtho() → 紧凑正交投影矩阵
   ↓
3. Shadow Pass（3 次，对应 3 个 cascade）
   - 对每个 cascade 的 ShadowMap(512×512)：
     - rasterizeShadow() 绘制所有 mesh
   ↓
4. Main Pass（512×512）
   - 对每个 pixel：
     - 光栅化三角形（backface culling + z-buffer）
     - 找到对应 cascade（world pos → light NDC，判断是否在 [0,1]³ 内）
     - PCF 采样 shadow factor
     - Phong 光照着色 × cascade 调试着色
   ↓
5. 输出（676×512）
   - 左 512×512：主渲染
   - 右 164×512：3 个 Shadow Map 可视化（每个 160×160，带颜色边框）
```

### 3.2 关键数据结构

**ShadowMap 结构体**：
```cpp
struct ShadowMap {
    int res;                  // 分辨率（512）
    std::vector<float> depth; // 深度值 [0,1]，初始化为 1.0（最远）
    Mat4 lightVP;             // 光源 VP 矩阵（用于主 Pass 中坐标变换）
    
    void setDepth(int x, int y, float d);           // min 更新
    float sample(float u, float v, float compareDepth, int pcfN); // PCF 采样
};
```

为什么用 `min` 更新：同一个 texel 可能被多个三角形写入（前后遮挡关系），我们只关心最近的深度（离光源最近的遮挡物）。

**CascadeInfo 结构体**：
```cpp
struct CascadeInfo {
    float nearZ, farZ;    // 子视锥深度范围
    Mat4 lightVP;         // 该 cascade 的光源 VP 矩阵
    ShadowMap shadowMap;  // 该 cascade 的深度图
};
```

**Framebuffer 结构体**：
```cpp
struct Framebuffer {
    int width, height;
    std::vector<Color> pixels; // HDR color
    std::vector<float> depth;  // z-buffer
};
```

### 3.3 场景设计（11 个遮挡物 × 3 个距离层）

为了充分展示 CSM 效果，场景分 3 个距离层设计：

| 距离层 | z 范围 | 物体 | 目的 |
|--------|--------|------|------|
| Near（Cascade 0）| 0~5 | 2 个小盒子 | 展示近处精细阴影 |
| Mid（Cascade 1）| 5~11 | 3 个中型盒子 | 过渡区 |
| Far（Cascade 2）| 11~25 | 4 个大盒子 + 远墙 | 远处阴影覆盖 |
| 地面 | 整个 | 大平面 | 接受所有阴影 |

---

## ④ 关键代码解析

### 4.1 Practical Split Scheme 实现

```cpp
// lambda: 0=纯均匀, 1=纯对数, 推荐 0.5~0.7
std::vector<float> computeSplits(float near, float far, int n, float lambda = 0.5f) {
    std::vector<float> splits(n + 1);
    splits[0] = near;
    splits[n] = far;
    for (int i = 1; i < n; i++) {
        float f = (float)i / n;
        float log_split = near * std::pow(far / near, f);  // 对数分割
        float uni_split = near + (far - near) * f;          // 均匀分割
        splits[i] = lambda * log_split + (1 - lambda) * uni_split; // 混合
    }
    return splits;
}
```

为什么对数分割用 `near * pow(far/near, i/N)` 而不是别的形式？这来自于透视投影的深度分布：透视投影将 [near, far] 映射为非线性的 NDC，距离摄像机越近的区域在屏幕空间占据越多像素，比例大致与 ln(z) 成正比，所以对数分割才能与透视分布对齐。

### 4.2 子视锥角点计算

```cpp
std::array<Vec3, 8> getFrustumCorners(
    const Mat4& /*invVP*/,  // 实际用 sub-frustum 的逆
    float nearZ, float farZ,
    const Mat4& proj,       // 摄像机投影矩阵（用来提取 fov 和 aspect）
    const Mat4& view)       // 摄像机视图矩阵
{
    // 用 nearZ/farZ 构造子视锥的投影矩阵
    Mat4 subProj = Mat4::perspective(
        2.f * std::atan(1.f / proj.m[1][1]),  // fovY: 从 proj 矩阵中提取
        proj.m[1][1] / proj.m[0][0],           // aspect: 从 proj 矩阵中提取
        nearZ, farZ
    );
    Mat4 subVP = subProj * view;
    Mat4 invSubVP = subVP.inverse();

    std::array<Vec3, 8> corners;
    int idx = 0;
    // NDC 空间 8 个角点：(±1, ±1, ±1)
    for (float x : {-1.f, 1.f})
        for (float y : {-1.f, 1.f})
            for (float z : {-1.f, 1.f}) {
                Vec4 ndc(x, y, z, 1.f);
                Vec4 world = invSubVP * ndc;
                corners[idx++] = {world.x / world.w, world.y / world.w, world.z / world.w};
            }
    return corners;
}
```

**踩坑点**：从投影矩阵中提取 fovY 和 aspect 的公式：
- `fovY = 2 * atan(1 / proj.m[1][1])`（因为 proj.m[1][1] = 1/tan(fovY/2)）
- `aspect = proj.m[1][1] / proj.m[0][0]`（因为 proj.m[0][0] = 1/(aspect * tan(fovY/2))）

如果搞错了这两个提取公式，子视锥角点会完全错误，导致光源正交矩阵不正确，阴影消失或错位。

### 4.3 光源正交矩阵 AABB 计算

```cpp
Mat4 computeLightOrtho(const std::array<Vec3, 8>& corners, const Mat4& lightView) {
    float minX = 1e30f, maxX = -1e30f;
    float minY = 1e30f, maxY = -1e30f;
    float minZ = 1e30f, maxZ = -1e30f;
    
    for (auto& c : corners) {
        Vec4 lc = lightView * Vec4(c, 1.f);  // 变换到光源视图空间
        minX = std::min(minX, lc.x); maxX = std::max(maxX, lc.x);
        minY = std::min(minY, lc.y); maxY = std::max(maxY, lc.y);
        minZ = std::min(minZ, lc.z); maxZ = std::max(maxZ, lc.z);
    }
    
    // Z 方向扩展 margin，防止视锥外遮挡物被剪裁
    minZ -= 2.f;
    maxZ += 2.f;
    
    // 注意：OpenGL 约定中光源看向 -Z，所以 near = -maxZ, far = -minZ
    return Mat4::ortho(minX, maxX, minY, maxY, -maxZ, -minZ);
}
```

**为什么 near = -maxZ？** 在 lookAt 构造的视图矩阵中，看向方向是 -Z 轴（右手坐标系）。视图空间中，靠近摄像机的物体 Z 值为负数（越远越负）。所以 "Z 最大" 对应"最近的深度平面"（near），"Z 最小"对应"最远的深度平面"（far）。取负号是因为 ortho 函数约定 near < far 且都为正值（或至少 near < far）。

### 4.4 Shadow Map 光栅化（核心循环）

```cpp
void rasterizeShadow(ShadowMap& sm, const Mat4& lightMVP,
                     const Vec3& p0, const Vec3& p1, const Vec3& p2) {
    // Step 1: 投影到 Shadow Map 屏幕空间
    auto project = [&](const Vec3& p, float& sx, float& sy, float& sz) -> bool {
        Vec4 c = applyMat(lightMVP, p);
        if (c.w <= 1e-5f) return false;
        float iw = 1.f / c.w;
        sx = (c.x * iw * 0.5f + 0.5f) * sm.res;  // [0, res]
        sy = (1.f - (c.y * iw * 0.5f + 0.5f)) * sm.res;
        sz = c.z * iw * 0.5f + 0.5f;  // [0, 1]
        return sz >= 0.f && sz <= 1.f;
    };
    
    float sx0, sy0, sz0, sx1, sy1, sz1, sx2, sy2, sz2;
    if (!project(p0, sx0, sy0, sz0)) return;
    if (!project(p1, sx1, sy1, sz1)) return;
    if (!project(p2, sx2, sy2, sz2)) return;
    
    // Step 2: AABB 剪裁
    int minX = std::max(0, (int)std::min({sx0, sx1, sx2}) - 1);
    int maxX = std::min(sm.res - 1, (int)std::max({sx0, sx1, sx2}) + 1);
    int minY = ...;
    int maxY = ...;
    
    // Step 3: 逐像素重心坐标判断 + 深度写入
    for (int py = minY; py <= maxY; py++) {
        for (int px = minX; px <= maxX; px++) {
            float cx = px + 0.5f, cy = py + 0.5f;
            float denom = (sy1-sy2)*(sx0-sx2) + (sx2-sx1)*(sy0-sy2);
            if (std::abs(denom) < 1e-6f) continue;
            
            // 重心坐标
            float l0 = ((sy1-sy2)*(cx-sx2) + (sx2-sx1)*(cy-sy2)) / denom;
            float l1 = ((sy2-sy0)*(cx-sx2) + (sx0-sx2)*(cy-sy2)) / denom;
            float l2 = 1.f - l0 - l1;
            if (l0 < 0 || l1 < 0 || l2 < 0) continue;
            
            // 插值深度
            float depth = sz0 * l0 + sz1 * l1 + sz2 * l2;
            sm.setDepth(px, py, depth);  // min 更新
        }
    }
}
```

**为什么用 pixel center (px + 0.5, py + 0.5) 而不是 (px, py)？** 像素采样点应该在像素中心。使用 (px, py) 会导致光栅化偏移半个像素，虽然在低分辨率下误差不明显，但在精确渲染中会产生系统误差。

**为什么 setDepth 用 min？** 对于 Shadow Map 来说，我们只关心光源能看到的最近物体。如果多个三角形都覆盖了同一个 texel，min 保证记录最近的那个（被遮挡物的阴影应该来自最近的遮挡物）。

### 4.5 PCF 采样实现

```cpp
float sample(float u, float v, float compareDepth, int pcfN = 3) const {
    float sum = 0.f;
    float half = (pcfN - 1) * 0.5f;  // 偏移量：对于 N=3，half=1.0
    float texel = 1.f / res;           // 一个纹素的 UV 大小
    
    for (int dy = 0; dy < pcfN; dy++) {
        for (int dx = 0; dx < pcfN; dx++) {
            float su = u + (dx - half) * texel;
            float sv = v + (dy - half) * texel;
            int px = std::clamp((int)(su * res), 0, res - 1);
            int py = std::clamp((int)(sv * res), 0, res - 1);
            float sd = depth[py * res + px];
            // bias = 0.001f，防止 shadow acne
            sum += (compareDepth <= sd + 0.001f) ? 1.f : 0.f;
        }
    }
    return sum / (pcfN * pcfN);
}
```

**3×3 PCF 的效果**：9 个采样点，阴影边界处平均 3~6 个点在阴影内，产生 0.33~0.66 的中间值，形成约 2~3 个像素宽的软过渡带。如果想要更软的阴影，可以增大 pcfN，但代价是 N² 倍采样开销。

### 4.6 主 Pass 中的 Cascade 选择

```cpp
int cascadeIdx = -1;
float shadow = 1.f;
for (int i = 0; i < (int)cascades.size(); i++) {
    Vec4 lc = cascades[i].lightVP * Vec4(worldPos, 1.f);
    float iw = 1.f / lc.w;
    float u = lc.x * iw * 0.5f + 0.5f;
    float v = lc.y * iw * 0.5f + 0.5f;
    float d = lc.z * iw * 0.5f + 0.5f;
    
    // 判断是否在该 cascade 的 shadow map 范围内
    if (u >= 0 && u <= 1 && v >= 0 && v <= 1 && d >= 0 && d <= 1) {
        cascadeIdx = i;
        shadow = cascades[i].shadowMap.sample(u, v, d, 3);
        break;  // 优先用最小（最精细）的 cascade
    }
}
```

**为什么从小 cascade 开始遍历？** Cascade 0 是最近、最精细的，如果一个片元同时在 Cascade 0 和 Cascade 1 的范围内（overlap 区域），我们当然想用精度更高的 Cascade 0。

**关于 cascade overlap**：在实际引擎中，相邻 cascade 之间通常有一定的重叠区域（blend zone），在重叠区域内对两个 cascade 的 shadow factor 进行插值，避免 cascade 边界处的突变。本实现为了简洁没有加 blend zone，但用了颜色调试着色来可视化 cascade 边界。

---

## ⑤ 踩坑实录

### Bug 1：4x4 矩阵求逆错误 → 阴影完全消失

**症状**：运行后图像一片正常亮度，没有任何阴影，Shadow Map 右侧完全是黑色。

**错误假设**：以为是 PCF 偏移量设置问题，或者 cascade 边界判断有误。

**真实原因**：4x4 矩阵求逆函数中，行列式的符号计算出了问题——`cofactor` 的正负号模式写错了，导致求出的逆矩阵完全错误。这使得 `invSubVP` 计算出来的世界空间角点全是 NaN 或极大值，`computeLightOrtho` 得到一个极度扭曲的投影矩阵，Shadow Map 渲染范围完全错误。

**修复方式**：重写 4x4 逆矩阵函数，参考标准的 adjugate 方法，逐项验证 cofactor 的正负号矩阵（正确模式是棋盘格 +/-）：
```
+ - + -
- + - +
+ - + -
- + - +
```

**教训**：矩阵逆运算出错的症状往往是"什么都消失了"，容易被误认为是别的问题。在实现矩阵逆之后，应该立即用 `M * M^-1 = I` 验证一下。

### Bug 2：Shadow Map 上下颠倒 → 阴影位置对不上

**症状**：图像中有阴影，但阴影位置明显偏移，阴影在物体的"错误"方向。

**错误假设**：以为是光源方向向量 lightDir 和 lightPos 设置有问题。

**真实原因**：Shadow Map 光栅化中，Y 轴翻转逻辑不一致。主 Pass 渲染时 `sy = (1 - (cy/cw * 0.5 + 0.5)) * H`（Y 轴从屏幕上方到下方），但 Shadow Map 采样时用的 `v = lc.y*iw*0.5 + 0.5`（Y 轴方向相反），导致 Shadow Map 写入坐标和读取坐标 Y 轴方向不一致。

**修复方式**：确保 Shadow Map 光栅化的 Y 轴翻转与采样时的 Y 轴翻转保持一致。在本实现中，统一用"翻转"版本（`sy = (1 - ...)`），使得 Shadow Map 中纹素 (0, 0) 对应世界空间的"左上"方向，与主 Pass 的屏幕坐标系一致。

### Bug 3：unused variable 警告 → 编译标准要求 0 warnings

**症状**：编译有 7 个 `-Wunused-variable` 和 `-Wunused-parameter` 警告。

**原因**：在迭代重构代码时，留下了几个中间变量（如 `invArea`、`d`）和未使用的参数（如 `invVP`、`camPos`）。

**修复方式**：删除所有未使用的变量，对未使用的参数用 `/*注释掉*/` 形式保留函数签名但消除警告：
```cpp
// 方式 1：删除变量
// 方式 2：参数注释
void func(const Mat4& /*invVP*/, float nearZ) { ... }
```

**教训**：使用 `-Wall -Wextra` 强制零警告是好习惯，警告通常暗示了潜在的逻辑问题（即使在这个 case 里只是代码清洁问题）。

### Bug 4：near cascade 物体没有阴影

**症状**：Cascade 1 和 2 的物体有阴影，但 Cascade 0（近处两个小盒子）没有阴影。

**原因**：光源正交矩阵的 Z margin 太小（原来只有 ±0.5），导致近处物体的阴影遮挡面被剪裁掉了（它们在子视锥的 Z 范围之外但会投影到视锥内）。

**修复方式**：将 margin 从 ±0.5 增加到 ±2.0，覆盖足够多的视锥外遮挡物。

---

## ⑥ 效果验证与数据

### 渲染输出

![CSM Renderer 输出](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-08-csm/csm_output.png)

*左侧：主渲染（512×512），蓝天背景 + 11 个彩色几何体投射/接收阴影，三个 cascade 分别带红/绿/蓝色调。右侧：3 个级联的 Shadow Map 深度图可视化（160×160 each）*

### 像素统计验证

```
文件大小: 8.7 KB (PNG，因存在大块天空均色，合理)
图像尺寸: 676 × 512
像素均值: 146.2 / 255 = 57.3%  ✅（非全黑、非全白）
像素标准差: 91.1  ✅（> 5，图像有丰富内容变化）
唯一颜色数: 141  ✅（非纯色图像）

区域分析：
- 顶部(sky)均值: 142.3   → 天蓝色背景正常
- 中部(scene)均值: 170.2  → 彩色几何体 + 阴影对比
- 底部(ground)均值: 220.0 → 浅色地面（部分在阴影中会更暗）
- 阴影图区域均值: 53.4    → 深度图正常（近处黑/远处白）
```

### 性能数据（纯 CPU，单线程，开发机）

```
场景规模：11 个 mesh，约 66 个三角形
Shadow Pass：3 个 cascade × 512×512 Shadow Map
Main Pass：512×512 主帧缓冲

- Shadow Pass 总耗时：< 100ms（3 个级联各约 30ms，纯 CPU 软光栅）
- Main Pass 耗时：约 200~500ms（取决于三角形覆盖率）
- 总渲染时间：< 1 秒

注：这是纯 CPU 软光栅化，无 SIMD 优化，与 GPU 实时性能不可比。
GPU 实现中 CSM 3 个 Shadow Pass 通常仅需 0.5~2ms（取决于场景复杂度）。
```

### CSM 分割效果验证

```
near = 0.5, far = 25.0, N = 3, λ = 0.6

计算分割结果：
  splits[0] = 0.500   (near)
  splits[1] = 4.572   (Cascade 0/1 边界)
  splits[2] = 10.805  (Cascade 1/2 边界)
  splits[3] = 25.000  (far)

Cascade 0 覆盖 0.5~4.57   (近处 9.1x 深度范围)
Cascade 1 覆盖 4.57~10.81 (中距 2.4x 深度范围)
Cascade 2 覆盖 10.81~25.0 (远处 2.3x 深度范围)
```

Cascade 0 的 Shadow Map 覆盖最窄的深度范围（0.5~4.57），在 512×512 的纹理上获得最高的每单位深度分辨率，这正是 CSM 的精华所在。

---

## ⑦ 总结与延伸

### 技术局限性

**本实现的简化点**：

1. **无 cascade 边界混合（blend zone）**：在相邻 cascade 交界处，阴影会有轻微跳变（调试着色也会突然变色）。实际引擎中会在 overlap 区域用 `lerp(shadow0, shadow1, blend_factor)` 平滑过渡。

2. **光源正交矩阵不稳定（shimmer）**：当摄像机移动时，光源正交矩阵会连续变化，导致 Shadow Map 中的纹素对应世界空间位置漂移，产生阴影边缘闪烁（temporal shimmer）。解决方案是 "snap to shadow texel"：将正交矩阵的偏移量四舍五入到 Shadow Map 纹素大小的整数倍。

3. **固定 bias = 0.001**：在不同场景下、不同 cascade 中，固定 bias 可能太大（Peter Panning）或太小（acne）。实际引擎用 slope-scale depth bias：`bias = const * tan(acos(NdotL))`。

4. **无 PCSS 或 VSSM**：PCF 的软阴影大小是固定的（3×3 核），无法随遮挡物距离变化（现实中遮挡物越远阴影越软）。PCSS（Percentage Closer Soft Shadows）可以实现距离相关的软阴影。

### 与本系列相关项目

- [2026-04-20 PCSS](../daily-coding-pcss)：PCF 软阴影的进阶版本，实现了距离相关的软阴影半径
- [2026-04-30 Shadow Volume](../daily-coding-shadow-volume)：用 Stencil Buffer 的精确硬阴影，适合小场景

### 可优化方向

1. **Cascade Blend Zone**：在 cascade 边界附近对两个 cascade 的 shadow factor 插值，消除边界跳变
2. **Texel Snapping**：稳定化光源正交矩阵，消除 shimmer
3. **Slope-scale Bias**：用 NdotL 动态调整 bias
4. **PCSS 替代 PCF**：实现距离相关的软阴影
5. **MSAA for Shadow Maps**：减少 Shadow Map 边缘锯齿（虽然已有 PCF）
6. **Virtual Shadow Maps（VSM/UE5）**：为整个场景生成极高分辨率的虚拟 Shadow Map，按需分配物理内存页，解决 CSM 的透视失真和稳定性问题
7. **多线程光栅化**：当前单线程 CPU 渲染，可以用 std::thread 或 OpenMP 并行化

### 一点感悟

CSM 是阴影技术中"工程复杂度"和"可调参数"最多的方案之一。fovY、near/far、λ、bias、cascade 数量、每级分辨率……每个参数都直接影响视觉质量。这也是为什么游戏引擎都把这些参数暴露给美术人员调整——没有万能的"最佳参数"，只有针对特定场景调好的参数。

从零实现 CSM 让我更直观地理解了每个参数的物理含义，这是直接调 API 时难以获得的体验。

---

*代码仓库：[daily-coding-practice/2026/05/05-08-csm](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-08-csm)*
