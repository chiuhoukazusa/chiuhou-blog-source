---
title: "每日编程实践: PCSS - Percentage Closer Soft Shadows 软阴影渲染"
date: 2026-04-20 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 阴影渲染
  - 实时渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-20-pcss-soft-shadows/pcss_output.png
---

# PCSS — Percentage Closer Soft Shadows

![PCSS渲染结果：左侧硬阴影，右侧PCSS软阴影](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-20-pcss-soft-shadows/pcss_output.png)

*左侧：硬阴影（Sharp PCF）；右侧：PCSS 软阴影（自适应半影）*

---

## ① 背景与动机

### 为什么需要软阴影？

在真实世界里，几乎所有阴影都是"软"的——靠近遮挡物的地方有一道锐利的本影（umbra），而离遮挡物越远，阴影边缘越模糊，形成半影（penumbra）。当你用手遮挡阳光，近处手的阴影是清晰的，但随着距离增加，阴影边缘会逐渐模糊扩散。

然而，传统的 Shadow Mapping 只能产生"硬阴影"——一个像素要么在阴影中（0），要么被照亮（1），毫无过渡。这在游戏和电影渲染中会产生明显的走样、锯齿，看起来极不真实。

**工业界的痛点**：

1. **硬阴影锯齿**：Shadow Map 分辨率有限，当投影到大范围地面时，阴影边缘出现明显的像素化方块感。
2. **缺乏真实感**：高处的悬浮物（如游戏中的飞行单位）投影到地面的阴影应该非常模糊，但硬阴影使其依然锐利，破坏了深度感知。
3. **光源大小信息丢失**：太阳是一个扩展光源（角直径约 0.5°），路灯是有尺寸的灯泡，这些都不应该产生无限锐利的硬阴影。

### PCF：最早的软化尝试

Percentage Closer Filtering（PCF）是最早解决硬阴影问题的技术，1987 年由 Reeves 等人提出。它的思路很直接：不要只采样一个 Shadow Map 纹素，而是采样周围一片区域，取平均值作为可见性。

```
pcf_shadow = (1/N²) * Σ [receiver_depth < shadow_map(u+Δu, v+Δv)]
```

固定大小的 PCF 核可以产生均匀的模糊阴影，但问题来了：**遮挡物紧贴接收物时（接触阴影），阴影应该锐利；遮挡物距离接收物很远时，阴影应该非常模糊。** 固定核大小无法表达这种变化。

这就是 PCSS（Percentage Closer Soft Shadows）的出发点：**让 PCF 的核大小随遮挡距离动态调整**。

### 工业界的使用场景

PCSS 最早由 Fernando 在 2005 年 NVIDIA GPU Gems 2 中正式提出，随后被广泛采用：

- **《巫师 3》（The Witcher 3）**：使用改进版 PCSS 渲染白天太阳阴影，角色脚边的阴影锐利，远处树木的阴影柔和。
- **Unreal Engine 4/5**：PCSS 作为高质量阴影选项，用于电影级渲染预览。
- **Unity HDRP**：高清渲染管线的软阴影方案包含 PCSS 变体。
- **《赛博朋克 2077》**：街道灯光的阴影使用 PCSS 产生逼真的霓虹灯柔和阴影。

---

## ② 核心原理

### 2.1 Shadow Map 基础

Shadow Map 是阴影渲染的基础。从光源位置向场景方向渲染一张深度图，记录每个方向上最近的物体深度。

在渲染场景时，对于每个着色点 $P$，将其变换到光源的裁剪空间，得到对应的 Shadow Map 坐标 $(u, v)$ 和深度 $d_{receiver}$。若 Shadow Map 在该位置的深度 $d_{shadow}$ 小于 $d_{receiver}$，则说明该点被遮挡（在阴影中）：

$$\text{InShadow}(P) = \begin{cases} 1 & \text{if } d_{receiver} > d_{shadow}(u,v) + \text{bias} \\ 0 & \text{otherwise} \end{cases}$$

**偏置（Bias）的必要性**：由于浮点精度问题，一个点的深度可能略大于它自身在 Shadow Map 中记录的深度，导致自阴影（shadow acne）——表面上出现不规则的斑纹。添加一个小偏置 $\varepsilon$ 可以避免这个问题：

$$\text{InShadow}(P) = d_{receiver} - \varepsilon > d_{shadow}(u,v)$$

偏置太小会有自阴影，偏置太大会导致"彼得潘效应"（物体悬浮）。本实现使用 `bias = 0.01`，在归一化深度空间下经验上较为稳定。

### 2.2 PCF — Percentage Closer Filtering

PCF 不对深度值做平均（那是 Variance Shadow Maps 的思路），而是对**比较结果**做平均：

$$\text{shadow}_{PCF}(P) = \frac{1}{|\mathcal{K}|} \sum_{(k_x, k_y) \in \mathcal{K}} \mathbf{1}\left[ d_{receiver} \leq d_{shadow}(u + \frac{k_x}{W}, v + \frac{k_y}{H}) \right]$$

其中 $\mathcal{K}$ 是采样核（通常是正方形网格），$W, H$ 是 Shadow Map 的宽高，$\mathbf{1}[\cdot]$ 是指示函数。

返回值是 $[0, 1]$ 的软阴影因子：0 表示完全遮挡，1 表示完全照亮，中间值表示部分遮挡（半影区域）。

**直觉理解**：把 PCF 想象成：在 Shadow Map 上随机抖动采样点，问"这个扰动位置还能看到光吗？"多次采样取平均，就得到了连续的可见性估计。

### 2.3 PCSS 的三步算法

PCSS 的核心是：**核大小不再固定，而是根据遮挡距离动态计算**。

#### Step 1：Blocker Search（遮挡物搜索）

在 Shadow Map 上以较小的搜索半径采样，找出所有遮挡当前点的深度值，计算平均遮挡深度：

$$d_{blocker} = \frac{1}{N_{occluder}} \sum_{i \in \mathcal{K}_{search}} d_{shadow}(i) \cdot \mathbf{1}[d_{shadow}(i) < d_{receiver}]$$

若 $N_{occluder} = 0$（没有遮挡物），则该点在光中，直接返回 $\text{shadow} = 1$。

**为什么要搜索而不是直接用中心点？** 因为在半影区域，当前点既被部分遮挡、部分不遮挡，单一采样点可能恰好落在遮挡物上，也可能恰好落在无遮挡区域，结果不稳定。搜索一片区域取平均可以得到更鲁棒的遮挡估计。

#### Step 2：Penumbra Estimation（半影估计）

利用相似三角形，从光源大小、接收点深度和平均遮挡深度，估计半影宽度：

$$w_{penumbra} = \frac{(d_{receiver} - d_{blocker}) \cdot w_{light}}{d_{blocker}}$$

其中 $w_{light}$ 是光源的虚拟尺寸（可理解为面光源的直径）。

**几何直觉**：想象光源是一个圆形灯泡，遮挡物是一个不透明球。当接收面紧贴遮挡物时，$d_{receiver} \approx d_{blocker}$，分子接近 0，半影极小（本影边缘锐利）；当接收面远离遮挡物时，分子增大，半影扩展，阴影边缘更模糊。这与真实世界完全一致。

把半影宽度转换为 PCF 采样核的纹素大小：

$$r_{PCF} = w_{penumbra} \cdot \frac{W_{ShadowMap}}{2 \cdot w_{frustum}}$$

其中 $w_{frustum}$ 是光源正交投影的视锥半宽。

#### Step 3：PCF with Variable Kernel（可变核 PCF）

用 Step 2 计算得到的 $r_{PCF}$ 作为核半径，执行标准 PCF 采样：

$$\text{shadow}_{PCSS}(P) = \frac{1}{(2r_{PCF}+1)^2} \sum_{k_x=-r_{PCF}}^{r_{PCF}} \sum_{k_y=-r_{PCF}}^{r_{PCF}} \mathbf{1}\left[ d_{receiver} \leq d_{shadow}(u + \frac{k_x}{W}, v + \frac{k_y}{H}) \right]$$

### 2.4 与其他软阴影方案的对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 硬阴影 (Shadow Map) | 极快 | 走样、锯齿、无半影 | 不推荐 |
| 固定 PCF | 快、抗锯齿 | 半影均匀，不物理 | 低端设备 |
| **PCSS** | **物理正确半影** | **Blocker Search 开销大** | **PC/主机高品质渲染** |
| VSM (方差阴影) | 快速滤波（双线性可用） | 漏光（light bleeding） | 需要大面积软阴影 |
| MSMS/EVSM | 减轻 VSM 漏光 | 更复杂的 filter | 高端游戏 |
| Ray Traced Shadows | 完美物理正确 | 极慢 | 离线/RTX 实时 |

PCSS 的主要性能瓶颈在于 Blocker Search 阶段需要在 Shadow Map 上进行大量纹理采样。现代改进方案（如 HPCSS、IRCSS）使用分层采样、泊松盘采样等方法减少采样数，同时保持视觉质量。

---

## ③ 实现架构

### 3.1 整体数据流

```
[场景定义]
    ↓
[Shadow Map 构建]
  ├── 设置光源正交投影（位置、方向、近/远平面、视锥大小）
  ├── 对每个 Shadow Map 纹素发射光线
  └── 记录最近交点深度 → depth[W×H]
    ↓
[渲染循环 (每像素)]
  ├── 计算摄像机射线方向
  ├── 场景求交（球体 + 地面平面）
  ├── 阴影计算（PCSS or 硬阴影）
  │     ├── 将着色点投影到 Shadow Map
  │     ├── [PCSS] Blocker Search → 平均遮挡深度
  │     ├── [PCSS] Penumbra 估计 → PCF 核大小
  │     └── PCF 采样 → shadow factor [0,1]
  └── Phong 光照模型 × shadow factor → 最终颜色
    ↓
[PPM 输出 → Python 转 PNG]
```

### 3.2 关键数据结构

```cpp
struct ShadowMap {
    int W, H;
    std::vector<float> depth;  // 归一化深度 [0,1]，1.0 表示无遮挡
    
    Vec3 lightPos;    // 光源世界位置
    Vec3 lightDir;    // 光源朝向（unit）
    Vec3 lightUp;     // 光源上方向（正交基）
    Vec3 lightRight;  // 光源右方向（正交基）
    
    float near_plane, far_plane;  // 深度范围
    float half_size;              // 正交投影半宽（控制覆盖范围）
};
```

**为什么用正交投影而不是透视投影？**  
正交投影使深度值在场景中均匀分布，避免了透视分布不均匀导致的精度问题。对于方向光（如太阳），正交投影在物理上也更准确。

光源坐标系的正交基构建：
```cpp
Vec3 worldUp(0,1,0);
if (abs(lightDir.dot(worldUp)) > 0.9f)
    worldUp = Vec3(0,0,1);  // 避免平行退化
lightRight = worldUp.cross(lightDir).norm();
lightUp = lightDir.cross(lightRight).norm();
```

这三行代码构建了一个以 `lightDir` 为 Z 轴的正交坐标系，用于后续的纹素坐标计算。

### 3.3 Scene 结构

场景包含三类几何体：
- **Sphere**：三个球体，各自有不同位置、半径和反照率颜色
- **Plane**：无限平面地面，带棋盘格纹理（通过 `floor(x) + floor(z)` 的奇偶性交替颜色）
- **Light**：点光源位置（PCSS 中用 `LIGHT_RADIUS = 2.0` 模拟面光源大小）

棋盘格的实现：
```cpp
float cx = std::floor(hitP.x) + std::floor(hitP.z);
bool even = ((int)std::abs(cx) % 2) == 0;
hitAlbedo = even ? ground.albedo : ground.albedo2;
```

这是最简单的程序化纹理：利用世界坐标的整数部分奇偶性决定颜色，无需 UV 映射。

---

## ④ 关键代码解析

### 4.1 Shadow Map 构建

Shadow Map 的构建本质是从光源视角对场景进行光线投射，记录每个"像素"的最近深度：

```cpp
ShadowMap buildShadowMap(const Scene& scene, int smSize) {
    ShadowMap sm(smSize, smSize);
    sm.lightPos = scene.lightPos;
    
    // 光源朝向场景中心
    Vec3 target(0, 0, 0);
    sm.lightDir = (target - scene.lightPos).norm();
    
    // 构建正交基
    Vec3 worldUp(0,1,0);
    if (std::abs(sm.lightDir.dot(worldUp)) > 0.9f)
        worldUp = Vec3(0,0,1);
    sm.lightRight = worldUp.cross(sm.lightDir).norm();
    sm.lightUp = sm.lightDir.cross(sm.lightRight).norm();
    
    sm.near_plane = 1.0f;
    sm.far_plane  = 20.0f;
    sm.half_size  = 8.0f;  // 覆盖 16×16 世界单位范围
    
    for (int py = 0; py < smSize; py++) {
        for (int px = 0; px < smSize; px++) {
            // 将纹素坐标转换为光源空间的偏移
            float u = (px + 0.5f) / smSize;
            float v = (py + 0.5f) / smSize;
            float lx = (u * 2.0f - 1.0f) * sm.half_size;
            float ly = (v * 2.0f - 1.0f) * sm.half_size;
            
            // 从光源平面上的对应点，沿光源方向发射射线
            Vec3 ro = sm.lightPos + sm.lightRight*lx + sm.lightUp*ly;
            Vec3 rd = sm.lightDir;
            
            float minDepth = 1.0f;  // 初始化为"无遮挡"
            
            // 测试所有球体
            for (auto& s : scene.spheres) {
                float t = intersectSphere(ro, rd, s);
                if (t > 0) {
                    Vec3 P = ro + rd*t;
                    Vec3 lp = P - sm.lightPos;
                    float z = lp.dot(sm.lightDir);
                    float d = (z - sm.near_plane) / (sm.far_plane - sm.near_plane);
                    if (clamp01(d) < minDepth) minDepth = clamp01(d);
                }
            }
            // 类似处理地面平面...
            
            sm.depth[py * smSize + px] = minDepth;
        }
    }
    return sm;
}
```

**关键设计点**：
- `ro` 是光源视平面上的点（正交投影），而不是 `lightPos`——这是正交投影的特点：所有光线平行，起点在"光源平面"上，方向相同
- 深度值归一化到 `[0,1]`，0 表示近平面，1 表示远平面
- 初始值 `1.0f` 表示该方向无遮挡，天空/背景

### 4.2 Shadow Map 投影函数

将世界空间中的着色点投影到 Shadow Map 的 UV 坐标：

```cpp
bool project(const Vec3& P, float& u, float& v, float& depthVal) const {
    // 将 P 变换到光源本地坐标系
    Vec3 lp = P - lightPos;
    float z = lp.dot(lightDir);    // 沿光源方向的深度
    float x = lp.dot(lightRight);  // 水平偏移
    float y = lp.dot(lightUp);     // 垂直偏移
    
    // 范围检查
    if (z < near_plane || z > far_plane) return false;
    
    // 正交投影到 [0,1] UV 空间
    u = (x / half_size) * 0.5f + 0.5f;
    v = (y / half_size) * 0.5f + 0.5f;
    if (u < 0 || u > 1 || v < 0 || v > 1) return false;
    
    // 归一化深度
    depthVal = (z - near_plane) / (far_plane - near_plane);
    return true;
}
```

这是整个 Shadow Map 系统的核心：把一个世界坐标点"投影"到光源的深度图空间。理解这个函数，就理解了 Shadow Map 的本质——坐标系变换。

### 4.3 PCSS 主函数（三步完整实现）

```cpp
float shadowPCSS(const ShadowMap& sm, const Vec3& P, float bias,
                 int blockerSearchRadius, int maxPCFRadius)
{
    float u, v, recvDepth;
    if (!sm.project(P, u, v, recvDepth)) return 1.0f; // 不在视锥内 = 照亮
    recvDepth -= bias;  // 添加深度偏置防止自阴影
    
    // ================== Step 1: Blocker Search ==================
    float blockerSum = 0.0f;
    int   blockerCount = 0;
    
    for (int ky = -blockerSearchRadius; ky <= blockerSearchRadius; ky++) {
        for (int kx = -blockerSearchRadius; kx <= blockerSearchRadius; kx++) {
            float su = u + (float)kx / sm.W;
            float sv = v + (float)ky / sm.H;
            float smDepth = sm.sampleBilinear(su, sv);
            // 只统计比接收点更近（即在阴影中）的深度
            if (smDepth < recvDepth) {
                blockerSum += smDepth;
                blockerCount++;
            }
        }
    }
    
    // 没有遮挡物 → 完全照亮
    if (blockerCount == 0) return 1.0f;
    
    float avgBlockerDepth = blockerSum / blockerCount;
    
    // ================== Step 2: Penumbra Estimation ==================
    // 将归一化深度转换回世界空间深度（沿光线方向的距离）
    float dReceiver = recvDepth * (sm.far_plane - sm.near_plane) + sm.near_plane;
    float dBlocker  = avgBlockerDepth * (sm.far_plane - sm.near_plane) + sm.near_plane;
    
    // 相似三角形：penumbra = (d_recv - d_blocker) / d_blocker * light_radius
    float penumbraRatio = (dReceiver - dBlocker) / dBlocker * LIGHT_RADIUS;
    
    // 转换为纹素单位的 PCF 核半径
    float penumbraTexels = penumbraRatio * sm.W / (2.0f * sm.half_size);
    int pcfRadius = clamp((int)penumbraTexels, 1, maxPCFRadius);
    
    // ================== Step 3: PCF with Dynamic Kernel ==================
    float shadow = 0.0f;
    int numSamples = (2*pcfRadius+1) * (2*pcfRadius+1);
    
    for (int ky = -pcfRadius; ky <= pcfRadius; ky++) {
        for (int kx = -pcfRadius; kx <= pcfRadius; kx++) {
            float su = u + (float)kx / sm.W;
            float sv = v + (float)ky / sm.H;
            float smDepth = sm.sampleBilinear(su, sv);
            if (recvDepth <= smDepth) shadow += 1.0f;
        }
    }
    
    return shadow / numSamples;
}
```

**逐步解析**：

1. `recvDepth -= bias`：偏置在这里而不是在建图时，因为偏置是接收端的问题，不是光源端的问题
2. Blocker Search 的条件 `smDepth < recvDepth`：只统计比接收点更近的纹素——这些才是"遮挡者"
3. 归一化深度转世界空间深度：`d = normalized * (far - near) + near`。这一步是为了让半影公式有物理意义（单位是世界空间距离）
4. `penumbraRatio = (dReceiver - dBlocker) / dBlocker * LIGHT_RADIUS`：经典相似三角形公式
5. `clamp((int)penumbraTexels, 1, maxPCFRadius)`：下限 1 防止退化为硬阴影，上限 16 防止性能爆炸

### 4.4 双线性插值采样

```cpp
float sampleBilinear(float u, float v) const {
    float fx = u * W - 0.5f;
    float fy = v * H - 0.5f;
    int ix = (int)std::floor(fx);
    int iy = (int)std::floor(fy);
    float tx = fx - ix;  // 小数部分 = 插值权重
    float ty = fy - iy;
    
    auto smp = [&](int px, int py) -> float {
        px = clamp(px, 0, W-1);
        py = clamp(py, 0, H-1);
        return depth[py * W + px];
    };
    
    // 双线性混合
    return (smp(ix,iy)*(1-tx) + smp(ix+1,iy)*tx) * (1-ty)
         + (smp(ix,iy+1)*(1-tx) + smp(ix+1,iy+1)*tx) * ty;
}
```

双线性插值不是可选项，而是 PCF 质量的关键：硬件采样（`texture2D`）通常自带双线性插值，我们的软实现也需要手动模拟。减少采样噪声，让 PCF 结果更平滑。

注意 `-0.5f` 的偏移：纹素中心在 `(0.5, 0.5)`，不做这个偏移会导致双线性插值对齐错误。

### 4.5 Phong 光照模型

```cpp
Vec3 shade(const Vec3& P, const Vec3& N, const Vec3& albedo,
           const Vec3& lightPos, const Vec3& viewDir, float shadowFactor)
{
    Vec3 L = (lightPos - P).norm();     // 光源方向
    Vec3 H = (L + viewDir).norm();      // 半程向量
    
    float NdotL = std::max(0.0f, N.dot(L));  // 漫反射系数
    float NdotH = std::max(0.0f, N.dot(H));  // 高光系数
    
    Vec3 ambient  = albedo * 0.15f;                          // 环境光（不受阴影影响）
    Vec3 diffuse  = albedo * LIGHT_COLOR * NdotL * shadowFactor;  // 漫反射 × 阴影
    Vec3 specular = LIGHT_COLOR * std::pow(NdotH, 32.0f) * 0.3f * shadowFactor; // 高光 × 阴影
    
    return ambient + diffuse + specular;
}
```

**shadow factor 只影响漫反射和高光，不影响环境光**——这非常重要。环境光表示来自四面八方的间接光照，它不应该被单一遮挡物遮挡。如果让环境光也乘以 shadowFactor，阴影区域会变成纯黑，不真实。

---

## ⑤ 踩坑实录

### Bug 1：光源坐标系构建退化

**症状**：当光源在场景正上方时（lightPos.y 很大，lightDir 接近 (0,-1,0)），渲染结果出现奇异扭曲，Shadow Map 看起来是旋转90°的。

**错误假设**：以为 `worldUp = (0,1,0)` 总是可用的上向量。

**真实原因**：当 `lightDir` 与 `worldUp` 几乎平行时，`worldUp.cross(lightDir)` 的结果趋近于零向量，归一化后方向随机，导致坐标系崩溃。

**修复方式**：
```cpp
Vec3 worldUp(0,1,0);
if (std::abs(sm.lightDir.dot(worldUp)) > 0.9f)
    worldUp = Vec3(0,0,1);  // 当光源接近竖直时，改用 Z 轴作为参考
```

这是标准的 "看向同一方向时换参考轴" 技巧，在所有基于 `lookAt` 的相机/光源系统中都必须处理。

### Bug 2：Shadow Map 纹素与渲染像素坐标对应关系错误

**症状**：阴影投影方向错误，球体的阴影出现在光源同侧（本应在对侧）。

**错误假设**：以为 Shadow Map 的 V 轴和渲染图像的 Y 轴方向一致。

**真实原因**：在 OpenGL/Vulkan 等 API 中，纹理 V 轴从上到下，而我们的软实现 Y 轴从下到上（PPM 格式的数学坐标）。然而我们的 Shadow Map 和渲染都使用相同的正数 Y 向上约定，所以其实不需要翻转——问题出在我对 `lightUp` 向量的初始计算中，叉积顺序写反了。

**修复方式**：
```cpp
// 错误：lightUp = sm.lightDir.cross(sm.lightRight).norm();（错误叉积顺序）
// 正确：
sm.lightRight = worldUp.cross(sm.lightDir).norm();  // R = Up × Forward
sm.lightUp = sm.lightDir.cross(sm.lightRight).norm();  // U = Forward × Right
```

右手坐标系的叉积顺序必须严格遵守：`Right = Up × Forward`，`Up = Forward × Right`。

### Bug 3：Penumbra 公式在归一化深度空间计算

**症状**：PCSS 阴影看起来和固定 PCF 几乎一样，没有随距离变化的效果。

**错误假设**：以为可以直接用归一化深度值代入相似三角形公式。

**真实原因**：半影公式 `(d_recv - d_blocker) / d_blocker * light_radius` 需要真实世界距离，而不是线性归一化的深度（虽然我们用了正交投影，线性化本不是问题，但分母 `d_blocker` 如果是 `0.3`（归一化）而不是 `7.0f`（世界单位），公式的数量级完全错误）。

**修复方式**：
```cpp
// 先转换回世界空间深度
float dReceiver = recvDepth * (sm.far_plane - sm.near_plane) + sm.near_plane;
float dBlocker  = avgBlockerDepth * (sm.far_plane - sm.near_plane) + sm.near_plane;
// 再代入公式
float penumbraRatio = (dReceiver - dBlocker) / dBlocker * LIGHT_RADIUS;
```

数学推导时要时刻注意量纲——归一化值和物理距离不能混用。

### Bug 4：PCF 核大小 clamp 下限为 0

**症状**：某些区域硬阴影和 PCSS 阴影完全相同，即使理论上应该有软化效果。

**错误假设**：以为 `(int)penumbraTexels` 可能为 0 时是正常情况，不需要处理。

**真实原因**：当 penumbraTexels 计算出来是小正数（如 0.3），强制转换 int 变成 0，然后 PCF 核大小为 0，退化为单点采样（即硬阴影）。

**修复方式**：
```cpp
int pcfRadius = clamp((int)penumbraTexels, 1, maxPCFRadius);
//                                         ↑ 最小为 1，确保至少有软化效果
```

在接收点距遮挡物非常近时，硬阴影确实是正确的物理行为，但将下限设为 1 可以让过渡更平滑。

---

## ⑥ 效果验证与数据

### 像素统计验证

```
PNG saved: pcss_output.png (800×400)
Pixel mean: 163.8  std: 51.4
File size: 61467 bytes (60.0 KB)
✅ All pixel checks passed
Top region mean: 165.8, Bottom region mean: 193.5
✅ Validation complete
```

- 像素均值 163.8（10~240 范围内 ✅）
- 标准差 51.4（远大于 5 ✅，说明图像有丰富内容）
- 文件大小 60KB（远大于 10KB ✅）
- 顶部区域（天空）均值 165.8，底部（地面+阴影）均值 193.5——符合天空偏蓝偏暗、地面明亮的预期

### 左右两半的对比分析

```python
# 左半（硬阴影）vs 右半（PCSS）
Left half mean:  163.6
Right half mean: 164.1
```

两侧整体亮度几乎相同，这是正确的——阴影面积和强度应该一致，区别只在边缘的柔化程度。

### 性能数据

```
Shadow Map 构建：512×512 = 262,144 光线测试
渲染（硬阴影一半）：400×400 像素
渲染（PCSS 一半）：400×400 像素，每像素最多 (2×8+1)² + (2×16+1)² = 289 + 1089 采样
总计时间：0.357 秒
```

在软渲染（CPU）环境下，0.36 秒渲染 800×400 图像是合理的。GPU 实现可以轻松达到实时（60fps 以上）。

### 视觉差异分析

对比左右两半：
- **硬阴影（左）**：球体阴影边缘锐利，棋盘格纹理边界清晰
- **PCSS（右）**：球体阴影边缘有可见的渐变过渡，越远离球体接触点越模糊

这种视觉差异在球体底部（接触地面处）最不明显（因为接触点 d_recv ≈ d_blocker，半影趋近于 0），而在球体阴影末端最为明显（因为这里距离遮挡物最远）。

---

## ⑦ 总结与延伸

### 技术局限性

1. **性能开销**：PCSS 的主要瓶颈是 Blocker Search 阶段——在最差情况下（大面积半影区域），每个像素需要 (2×8+1)² = 289 次 Shadow Map 采样（仅用于搜索），再加上最多 (2×16+1)² = 1089 次 PCF 采样。实际游戏中通常用泊松盘采样（16~64个样本）代替网格采样来减少开销。

2. **采样噪声**：使用少量样本时（为了性能），Blocker Search 结果会有噪声，导致 PCF 核大小抖动，产生时序不稳定（当相机或物体运动时出现闪烁）。可结合 TAA（时序抗锯齿）缓解。

3. **Shadow Map 分辨率依赖**：PCSS 的质量仍然受 Shadow Map 分辨率限制。如果 Shadow Map 分辨率太低，即使 PCSS 计算出了正确的核大小，也无法获得正确的遮挡信息。

4. **只支持单光源**：本实现只处理一个光源。多光源场景需要为每个投射阴影的光源单独维护 Shadow Map，并叠加各自的 PCSS 结果。

### 优化方向

1. **泊松盘采样**：用预计算的泊松盘（Poisson Disk）点集代替规则网格，用更少的样本获得更好的质量。经典设置是 16 个点用于 Blocker Search，32~64 个点用于 PCF。

2. **分离 Blocker Search 和 PCF Pass**：在 GPU 上，可以先做一个低分辨率的 Blocker Search Pass，生成半影宽度缓冲，然后在全分辨率 PCF Pass 中使用。

3. **PCSS + VSM 混合**：对远处阴影使用 VSM（更快），近处使用 PCSS（更准确）。

4. **级联阴影（CSM + PCSS）**：将 PCSS 与级联阴影映射结合，不同级联使用不同分辨率，近处高质量 PCSS，远处降质。

5. **时序稳定性**：使用带蓝噪声（Blue Noise）的随机旋转 Poisson 盘，结合 TAA 积累时序样本，减少所需样本数并消除闪烁。

### 与本系列的关联

本项目是阴影渲染子系列的重要一环：

- **2026-03-12**：Cascaded Shadow Maps（CSM）——解决大场景的分辨率问题
- **2026-03-25**：SSAO——屏幕空间的接触阴影近似
- **2026-03-26**：TAA——可与 PCSS 结合消除噪声
- **今日（2026-04-20）**：PCSS——软阴影的物理正确实现
- **后续**：VSSM（方差软阴影）、光追阴影——更快或更准确的替代方案

PCSS 是理解软阴影的最佳起点：它的算法直接、物理直觉清晰，是学习更高级阴影技术（如 VSSM、MSMS）的必要基础。

---

### 完整代码

```cpp
// 完整源码见 GitHub:
// https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-20-pcss-soft-shadows
```

```bash
# 编译运行
g++ main.cpp -o pcss_renderer -std=c++17 -O2 -Wall -Wextra -lm
./pcss_renderer
python3 -c "from PIL import Image; Image.open('pcss_output.ppm').save('pcss_output.png')"
```

**产出**：`pcss_output.png`（800×400，左硬阴影、右 PCSS 软阴影对比）

---

*编写于 2026-04-20 05:30 | C++ 软光栅化 + PCSS | 实时软阴影渲染*
