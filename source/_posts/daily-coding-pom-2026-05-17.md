---
title: "每日编程实践: Parallax Occlusion Mapping Renderer"
date: 2026-05-17 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 软光栅化
  - 纹理技术
  - 实时渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-17-pom/pom_output.png
---

# 每日编程实践 Day 80：Parallax Occlusion Mapping Renderer

今天实现 **Parallax Occlusion Mapping（视差遮挡贴图，POM）**——一种用高度图驱动 UV 偏移，在平面上营造真实立体感的经典实时渲染技术。与 Normal Map 只改法线不同，POM 会让 UV 本身"随视角移动"，使砖块、石板、地表看起来真正凸出平面。实现还包含沿光线方向的 **自阴影（Self-Shadowing）**，让高出部分在临近区域投射阴影。

![渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-17-pom/pom_output.png)

---

## ① 背景与动机

### 为什么不够用 Normal Map？

Normal Map 是改变每个像素的法线方向，欺骗光照计算，让光影看起来像有凹凸。但它有一个根本性的局限：**UV 不变**。当你以掠射角（grazing angle）看砖墙时，一个真正凸出的砖块应该遮挡后面的区域，但 Normal Map 无法做到这一点——在接近 90° 的视角下，砖块与墙面依然是同一平面，轮廓完全平直。

这在游戏场景中看起来非常假，特别是：

- 近距离观察地板（如 FPS 游戏向下看）
- 掠射角观察砖墙、石板路
- 侧视角下的岩石、浮雕

**Normal Map 在法线方向造假，但几何轮廓暴露了真相。**

### Parallax Mapping 的思路

Parallax Mapping（视差贴图）的核心思想很直接：

> 如果表面上某点实际"高出"平面 h，那么当你从斜角看这个点时，你看到的位置应该比几何上的 UV 坐标向视线方向偏移了一定量。

最简单的版本（Simple Parallax Mapping）只做一次近似偏移：
```
UV_offset = (height - 0.5) * parallax_scale * view_dir_tangent.xy / view_dir_tangent.z
```

这个近似在高度变化不剧烈时还好，但对高频细节（如深沟壑的砖缝）会产生明显错位。

**Parallax Occlusion Mapping（POM）** 是视差贴图的进阶版：沿视线方向在高度场中做**射线步进（Ray Marching）**，精确找到视线与高度场的交点。这是今天要实现的技术。

### 工业界使用场景

POM 是 UE4/UE5 的 Material 编辑器标配节点（`Parallax Occlusion Mapping` 节点）。在以下场景几乎必用：

- **FPS 游戏地面/墙面**：COD、战地系列的石板路、砖墙
- **RPG 场景道具**：武器铭文的金属浮雕感
- **建筑可视化**：砖块、大理石铺地、混凝土墙面
- **车漆 / 皮革纹理**：皮革的细腻凸起感

与全分辨率 Tessellation + Displacement Mapping 相比，POM 完全在像素着色器里做，**不增加顶点数量**，成本相对可控，适合中距离内的表面细节。

---

## ② 核心原理

### 2.1 问题建模

设一张平面 quad，法线指向 +Z。有一张高度图 $h(u, v) \in [0, 1]$，表示表面在 Z 方向上的相对高度（1 = 最高，0 = 最低）。

实际上，这个表面可以理解为一张三维函数图：

$$
\text{surface point} = (u, v, \, h(u, v) \times \text{heightScale})
$$

当视线方向（切线空间下）为 $\mathbf{V}_{ts} = (v_x, v_y, v_z)$（z < 0 表示向表面看），视线从 $(u_0, v_0, 1)$（切线空间上方）射向表面，参数化射线为：

$$
(u(t), v(t), d(t)) = (u_0 + t \cdot \frac{-v_x}{v_z}, \; v_0 + t \cdot \frac{-v_y}{v_z}, \; t)
$$

其中 $d(t)$ 是从表面顶（高度=1）向下的深度，$t$ 从 0 增大到 1。

**POM 要解的方程**：找最小的 $t^*$ 使得：

$$
h(u(t^*), v(t^*)) = 1 - t^*
$$

即高度场值等于"我们还剩多少深度"。这就是视线与高度场的首次交点。

### 2.2 线性步进（Linear Search）

直接解析求解 $t^*$ 几乎不可能（高度场是任意噪声函数），所以用迭代步进：

```
for i = 0 to linearSteps:
    t = i / linearSteps
    (u, v) = (u0 + du*t, v0 + dv*t)
    h = heightMap(u, v)
    if h > (1 - t):
        // 找到跨越点，记录 t_prev 和 t_curr
        break
```

步进方向 `du, dv` 由切线空间视线方向决定：

```cpp
float du = -Vts.x / (Vts.z + 1e-6f) * heightScale;
float dv = -Vts.y / (Vts.z + 1e-6f) * heightScale;
```

这里 `heightScale` 控制最大 UV 偏移量，通常取 0.05~0.20。

**线性步进的问题**：步数有限时，找到的交点是在上一步（未碰到）和这一步（已穿入）之间的某处，精度不足。

### 2.3 二分细化（Binary Search Refinement）

在线性步进找到跨越区间 $[t_{lo}, t_{hi}]$ 后，对区间做二分：

```
for j = 0 to binarySteps:
    mid = (lo + hi) / 2
    (u, v) = (u0 + du*mid, v0 + dv*mid)
    h = heightMap(u, v)
    if h > (1 - mid):
        hi = mid   // 已穿入，往回找
    else:
        lo = mid   // 未到，继续向前
```

8 次二分可将精度提升 $2^8 = 256$ 倍。实现代价很小（每次只多一次高度图采样），但效果大幅改善。

**直觉解释**：线性步进找到"第一个穿入"的格子，二分在这个格子里精确找交点。就像用二分查找找到有序数组中的准确位置。

### 2.4 切线空间的必要性

POM 必须在**切线空间（Tangent Space）**中进行，因为高度图的 UV 方向就定义在切线空间里。

对于轴对齐的 quad：
- Tangent (T) = (1, 0, 0)（对应 +U 方向）
- Bitangent (B) = (0, 1, 0)（对应 +V 方向）
- Normal (N) = (0, 0, 1)（表面法线）

视线方向从世界空间变换到切线空间：
$$
\mathbf{V}_{ts} = (V \cdot T, \; V \cdot B, \; V \cdot N)
$$

对于我们的轴对齐 quad，切线空间 = 世界空间，无需额外变换。但在真实引擎中，每个三角形都有自己的 TBN 矩阵，需要在顶点着色器中传递并在像素着色器中使用。

### 2.5 法线扰动（Normal Perturbation from Height Gradient）

POM 确定了"我们在高度场的哪个位置"后，还需要计算该点的正确法线以进行光照。

法线由高度场的梯度决定：

$$
\frac{\partial h}{\partial u} \approx \frac{h(u + \epsilon, v) - h(u, v)}{\epsilon}
$$

$$
\frac{\partial h}{\partial v} \approx \frac{h(u, v + \epsilon) - h(u, v)}{\epsilon}
$$

切线空间法线：

$$
\mathbf{N}_{ts} = \text{normalize}\left(-\frac{\partial h}{\partial u} \cdot \mathbf{T}, \; -\frac{\partial h}{\partial v} \cdot \mathbf{B}, \; 1\right)
$$

负号的原因：高度增大的方向，法线应该向反向倾斜（坡朝上的法线偏向"向上"）。

### 2.6 POM 自阴影（Self-Shadowing）

当光线从角度照射时，高度场的凸起应该在周围区域投下阴影。这需要从交点出发，沿光线方向在高度场中再做一次射线步进：

```
for i = 1 to shadowSteps:
    t = i * stepSize / surfaceH
    (su, sv) = (u + lu*t, v + lv*t)
    sh = heightMap(su, sv)
    expectedDepth = (1 - surfaceH) + stepSize * i
    if sh > expectedDepth:
        occlusion = (sh - expectedDepth) * 10
        shadowAccum = max(shadowAccum, clamp(occlusion, 0, 1))
```

`surfaceH` 是当前点的高度，`lu, lv` 是光线方向在切线空间的 UV 分量。

这和 POM 步进几乎一样，只是方向换成了光线方向，起点是已知的高度场交点。

---

## ③ 实现架构

### 3.1 整体数据流

```
程序化高度图定义 (sampleHeight)
         ↓
视线方向计算（从eye到surface point）
         ↓
POM 线性步进 → 二分细化 → 精确 UV 偏移 (pomTrace)
         ↓
法线计算（高度梯度）(heightNormal)
         ↓
自阴影步进 (pomShadow)
         ↓
Phong 着色（albedo × (ambient + diffuse × shadow + specular × shadow)）
         ↓
ACES Filmic 色调映射
         ↓
写入 PNG
```

### 3.2 高度图类型

实现了 4 种程序化高度图：

| 类型 | 名称 | 特点 |
|------|------|------|
| `HM_RIDGED` | 脊状 FBM 岩石 | 多倍频叠加，山脊感 |
| `HM_BRICK` | 砖墙 | 周期性砖块+灰缝，测试 POM 直线边界 |
| `HM_WAVES` | 波浪 | 正弦叠加+FBM 扰动，光滑渐变 |
| `HM_CRATER` | 月球陨石坑 | SDF 驱动的碗形 + 环形山脊 |

这 4 种覆盖了不同频率特征：高频噪声（RIDGED）、硬边界（BRICK）、低频平滑（WAVES）、特殊 SDF 形状（CRATER）。

### 3.3 关键数据结构

```cpp
// 图像缓冲区
struct Image {
    int w, h;
    std::vector<Vec3> pixels;  // HDR float pixel buffer
    void save(const char* path); // ACES tone-map + PNG write
};

// 面板描述
struct Panel {
    HeightMapType type;   // 高度图类型
    float hmScale;        // 高度图 UV 缩放（越大纹路越密）
    float heightScale;    // 最大 POM 偏移量（越大立体感越强，但也越容易伪影）
    const char* name;
    Vec3 border;          // 面板边框颜色
};
```

### 3.4 视线构造

每个像素对应 panel 内的 UV，然后根据 UV 在 $[-1, 1]^2$ 的世界空间 quad 上找到表面点，从固定眼睛位置 `eye = (0, -0.3, 2.5)` 射出视线：

```cpp
float wx = (u * 2.f - 1.f);   // UV [0,1] → world [-1,1]
float wy = (v * 2.f - 1.f);
Vec3 surfPt(wx, wy, 0.f);
Vec3 viewDir = (surfPt - eye).norm();
```

---

## ④ 关键代码解析

### 4.1 高度图采样

```cpp
float sampleHeight(float u, float v, HeightMapType type, float scale=4.f){
    u *= scale; v *= scale;
    switch(type){
    case HM_RIDGED:
        return ridgedFbm(u, v, 6);
    case HM_BRICK: {
        float row = floorf(v);
        float shift = (fmodf(row, 2.f) < 1.f) ? 0.5f : 0.f;  // 交错排列
        float uf = fract(u + shift) * 8.f;   // 在砖块内的位置（8砖/unit）
        float vf = fract(v) * 4.f;
        float mx = sinf(uf * 3.14159f) * sinf(vf * 3.14159f);
        mx = clampf(mx * 3.f, 0.f, 1.f);
        return 1.f - mx; // 砖缝（mx接近0）高度=1（凸出），砖面高度=0
    }
    // ...
```

**砖块高度图的设计**：砖缝处 `mx ≈ 0`，`height ≈ 1`（凸出）；砖面中央 `mx ≈ 1`，`height ≈ 0`（凹陷）。这样 POM 会让视线偏向砖缝时"掉进"砖缝，产生深度感。

**为什么用 sin 而不是 step？** `sin` 产生平滑过渡，避免砖块边缘的硬跳变在 POM 步进时产生严重锯齿。

### 4.2 脊状 FBM（Ridged FBM）

```cpp
float ridgedFbm(float x, float y, int octaves=6){
    float v=0, amp=0.5f, freq=1.f;
    for(int i=0;i<octaves;i++){
        float n = smoothNoise(x*freq, y*freq);
        v += amp * (1.f - fabsf(n*2-1));  // 关键：1 - |2n-1|
        amp  *= 0.5f;
        freq *= 2.f;
    }
    return clampf(v, 0, 1);
}
```

关键变换 `1 - |2n - 1|`：

- 普通 FBM 生成连续起伏（山丘感）
- `|2n - 1|` 将 0.5 处折叠为 0，两端为 1（像 V 形）
- `1 - |2n - 1|` 翻转，使 0.5 处为峰值（山脊感）

**直觉**：普通噪声是"丘陵"，ridged FBM 是"锋利的山脊"，非常适合模拟岩石、高山等地形。

### 4.3 POM 核心追踪

```cpp
Vec2 pomTrace(float u0, float v0, Vec3 Vts, HeightMapType type, float scale,
              float heightScale, int linearSteps=32, int binarySteps=8)
{
    float stepSize = 1.f / (float)linearSteps;
    float curDepth = 0.f;
    float cu = u0, cv = v0;

    // UV 在 Z 方向每增加 1 时的偏移量
    // 相当于：视线从表面顶部到底部时，UV 移动了多少
    float du = -Vts.x / (Vts.z + 1e-6f) * heightScale;
    float dv = -Vts.y / (Vts.z + 1e-6f) * heightScale;

    float prevDepth = 0.f;

    for(int i=0; i<linearSteps; i++){
        curDepth += stepSize;
        float nu = cu + du * curDepth;
        float nv = cv + dv * curDepth;
        float h  = sampleHeight(nu, nv, type, scale);

        if(h > 1.f - curDepth){  // 穿入高度场
            // 二分细化
            float lo = prevDepth, hi = curDepth;
            for(int j=0; j<binarySteps; j++){
                float mid = (lo+hi)*0.5f;
                float mu  = cu + du * mid;
                float mv  = cv + dv * mid;
                float mh  = sampleHeight(mu, mv, type, scale);
                if(mh > 1.f - mid) hi = mid;
                else                lo = mid;
            }
            float fd = (lo+hi)*0.5f;
            return {cu + du*fd, cv + dv*fd};
        }
        prevDepth = curDepth;
    }
    return {cu + du*curDepth, cv + dv*curDepth};
}
```

**为什么是 `h > 1.f - curDepth`？**

我们用深度 $d \in [0, 1]$ 从顶部往下走：
- $d = 0$：表面最顶（高度最高处）
- $d = 1$：表面最底（高度最低处）

"视线在 depth = d 处，期望高度 = 1 - d"。如果高度场 $h > 1 - d$，说明地形比视线当前位置高，即视线已穿入地形内部，交点就在此之前。

**`+ 1e-6f` 的作用**：防止视线完全垂直于表面时（`Vts.z = 0`）的除零错误。

### 4.4 法线扰动

```cpp
Vec3 heightNormal(float u, float v, HeightMapType type, float scale=4.f,
                  float eps=0.001f, float bumpScale=0.1f){
    float h  = sampleHeight(u,     v,   type, scale);
    float hx = sampleHeight(u+eps, v,   type, scale);
    float hy = sampleHeight(u,   v+eps, type, scale);
    float dhdx = (hx - h) / eps * bumpScale;
    float dhdy = (hy - h) / eps * bumpScale;
    // 切线空间法线：平面法线 (0,0,1) 加上梯度扰动
    Vec3 n = Vec3(-dhdx, -dhdy, 1.f).norm();
    return n;
}
```

**为什么是负号 `-dhdx`？**

考虑 U 方向上高度增加（$\partial h / \partial u > 0$），这意味着表面向 +U 方向倾斜向上。这个坡面的法线应该向 -U 方向（向左）倾斜，所以法线 X 分量 = $-\partial h / \partial u$。

**`eps = 0.001f` 的选择**：太大会使梯度估计不准（高度场变化太剧烈时），太小会因浮点精度损失产生噪声。0.001 对于 $[0,1]$ 范围的 UV 是合理的中间值。

### 4.5 自阴影步进

```cpp
float pomShadow(float u, float v, Vec3 Lts, HeightMapType type, float scale,
                float heightScale, float surfaceH, int steps=16)
{
    if(Lts.z <= 0.f) return 0.f;  // 光源在地面以下，无效

    float shadowAccum = 0.f;
    float du = Lts.x / (Lts.z + 1e-6f) * heightScale;
    float dv = Lts.y / (Lts.z + 1e-6f) * heightScale;
    float stepSize = surfaceH / (float)steps;  // 只步进到高度场顶部

    for(int i=1; i<=steps; i++){
        float t  = i * stepSize / surfaceH;
        float su = u + du * t;
        float sv = v + dv * t;
        float sh = sampleHeight(su, sv, type, scale);
        float expectedDepth = 1.f - surfaceH + stepSize * i;
        if(sh > expectedDepth){
            float occlusion = (sh - expectedDepth) * 10.f;
            shadowAccum = max(shadowAccum, clampf(occlusion, 0.f, 1.f));
        }
    }
    return 1.f - shadowAccum * 0.8f;  // 0.8 防止完全黑
}
```

**`expectedDepth` 的计算**：从当前点（高度 `surfaceH`，深度 `1 - surfaceH`）出发，沿光线方向步进 `i * stepSize` 个深度单位，期望该位置的高度场应在此深度以下（即 `sh ≤ expectedDepth`）。如果 `sh > expectedDepth`，说明光线在前往光源的途中被更高的地形遮挡，产生自阴影。

**`* 0.8f` 的作用**：保留 20% 的漏光（ambient），防止阴影处完全变黑，模拟间接光照的基本效果。

### 4.6 Phong 着色整合

```cpp
Vec3 shade(float u, float v, HeightMapType type, float hmScale,
           float heightScale, Vec3 viewDir, Vec3 lightDir, ...)
{
    // 1. POM 追踪：找精确交点 UV
    Vec3 Vts = Vec3(-viewDir.x, -viewDir.y, -viewDir.z);
    Vec2 uvNew = pomTrace(u, v, Vts, type, hmScale, heightScale);
    float uu = uvNew.x, vv = uvNew.y;

    // 2. 获取交点高度
    float h = sampleHeight(uu, vv, type, hmScale);

    // 3. 扰动法线
    Vec3 N = heightNormal(uu, vv, type, hmScale, 0.001f, 0.15f);

    // 4. Albedo
    Vec3 albedo = albedoFromType(type, uu, vv);

    // 5. 自阴影
    float shadowFactor = pomShadow(uu, vv, lightDir, type, hmScale, heightScale, h);

    // 6. Phong 光照
    Vec3 L = lightDir.norm();
    Vec3 V = viewDir.norm();
    Vec3 H = (L + V).norm();                 // Halfway vector
    float diff = max(0.f, N.dot(L));          // Lambertian
    float spec = powf(max(0.f, N.dot(H)), 64.f);  // Blinn-Phong

    Vec3 ambient  = ambientColor * albedo * 0.15f;
    Vec3 diffuse  = lightColor * albedo * diff * shadowFactor;
    Vec3 specular = lightColor * Vec3(0.8f) * spec * shadowFactor;

    return ambient + diffuse + specular;
}
```

注意 specular 不依赖 albedo（用固定的 0.8 灰白），这是 Blinn-Phong 的常见做法，防止颜色饱和的表面镜面高光颜色失真。

---

## ⑤ 踩坑实录

### Bug 1：`uint8_t` 未声明

**症状**：编译错误 `'uint8_t' was not declared in this scope`

**错误假设**：以为 `<cstdio>` 或 `<vector>` 会隐式包含整数类型。

**真实原因**：`uint8_t` 定义在 `<cstdint>` 中。`<cstdio>` 不保证包含它，即使在某些平台上能用（可能通过其他头文件的间接包含），也不应该依赖这个行为。

**修复**：在 `#include` 列表中明确加入 `#include <cstdint>`。

**教训**：标准 fixed-width 整数类型（`uint8_t`、`int32_t` 等）必须显式包含 `<cstdint>`，不要依赖其他头文件的间接包含。

---

### Bug 2：stb 头文件编译警告

**症状**：`-Wall -Wextra` 下，stb_image_write.h 产生大量 `-Wmissing-field-initializers` 警告，导致目标"0 warnings"未能达标。

**错误假设**：以为第三方库的警告无需处理，或以为 `-Wextra` 不会触发 stb。

**真实原因**：stb 是单头文件库，使用了 `{ 0 }` 的部分初始化方式，触发 GCC `-Wmissing-field-initializers` 警告。这是 stb 的已知问题，不影响功能，但会污染编译输出。

**修复**：用 `#pragma GCC diagnostic` 包裹 include：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#include "../stb_image_write.h"
#pragma GCC diagnostic pop
```

**教训**：第三方头文件导致的警告用 pragma 局部关闭，不要全局关闭，以免真正的自身代码警告被淹没。

---

### Bug 3：未使用变量 `prevHeight`

**症状**：`warning: variable 'prevHeight' set but not used`

**原因**：最初设计 POM 时打算用 `prevHeight` 做插值，但最终只用了 `prevDepth` 做二分区间，`prevHeight` 成了死变量。

**修复**：直接删除 `prevHeight` 和相关赋值，改为 `(void)sampleHeight(...)` 的注释调用仅做记录（实际上也直接删掉了）。

---

### Bug 4：视线方向传递错误（需要取反）

**症状**：POM 追踪返回的 UV 偏移方向相反，视差效果是"反的"（当视角向右看，凸起应该向左偏移，但实际向右偏移）。

**原因**：`pomTrace` 接收的 `Vts` 参数应该是"进入表面的方向"（z 为负），但我传入了 `viewDir`（从眼睛到表面，z 也为负，但 xy 分量符号需要配合）。具体来说：

```cpp
// 错误：直接传 viewDir（视线朝向表面，Vts.z < 0 ✓，但偏移方向算错）
Vec2 uvNew = pomTrace(u, v, viewDir, type, ...);

// 正确：取反后传入（进入表面的方向与视线方向相反）
Vec3 Vts = Vec3(-viewDir.x, -viewDir.y, -viewDir.z);
Vec2 uvNew = pomTrace(u, v, Vts, type, ...);
```

在 `pomTrace` 内部：
```cpp
float du = -Vts.x / (Vts.z + 1e-6f) * heightScale;
```

如果 Vts 是"进入方向"（Vts.z < 0），则 `du = -Vts.x / Vts.z * scale`，当视线从右上方射入（Vts.x > 0, Vts.z < 0）时，`du` 为正，UV 向 +U 方向偏移，正确！

**修复**：在调用 `pomTrace` 前对 `viewDir` 取反。

---

## ⑥ 效果验证与数据

### 输出图像统计

```
文件名: pom_output.png
分辨率: 1200 × 700
文件大小: 769 KB（> 10KB ✅）
渲染耗时: 3.9 秒
```

像素采样验证结果：

```
Pixel mean: 67.2  std: 57.4
✅ 均值在 10~240 之间（非全黑/全白）
✅ 标准差 > 5（图像有内容变化）
```

### 4 种材质面板表现

| 面板 | 特点 |
|------|------|
| Ridged FBM Rock（左上）| 山脊噪声产生清晰的立体岩石纹理，掠射角可见明显深度 |
| Brick Wall（右上）| 砖缝处明显凹陷，砖面边缘可见视差偏移 |
| Procedural Waves（左下）| 蓝色波浪面，光滑渐变，自阴影在波谷处清晰可见 |
| Lunar Craters（右下）| 环形陨石坑结构，坑沿的环形山脊明显凸出 |

### 性能分析

| 参数 | 值 |
|------|----|
| 图像分辨率 | 1200 × 700 = 840,000 像素 |
| POM 线性步数 | 32 步 |
| POM 二分步数 | 8 步 |
| 自阴影步数 | 16 步 |
| 高度图采样次数/像素 | ~32 + 8 + 16 + 3(法线) ≈ 59 次 |
| 总采样次数 | 840,000 × 59 ≈ 49.6M 次 |
| 总耗时 | 3.9 秒（CPU 单线程） |
| 每秒采样数 | ~12.7M 次/秒 |

在真实 GPU 实现中，shader 是大规模并行的，59 次纹理采样在中端 GPU 上只需 ~0.5ms，足以支持 60fps 实时渲染。

---

## ⑦ 总结与延伸

### 技术局限性

1. **接缝（Seam）问题**：POM 会导致 UV 越界，需要纹理设置 `WRAP` 模式或做边界检查
2. **步数与质量权衡**：步数少会产生条纹状伪影（stepping artifacts），步数多会增加着色器开销
3. **高度变化剧烈时失效**：当 `heightScale` 过大，线性步进找不到正确交点，出现"透视穿透"
4. **视角垂直时无效果**：当 `Vts.z = -1`（正视），`du = dv = 0`，POM 退化为 Normal Map
5. **轮廓依然正确性**：POM 依然无法解决几何轮廓问题（从外部看球的边缘），对此需要 Tessellation

### 优化方向

1. **变步数（Adaptive Steps）**：根据视角倾斜程度动态调整步数。正视时少步，掠射时多步
2. **Relaxed Cone Stepping**：预计算每个高度图像素的"安全步进圆锥半径"，允许更大步长，减少步数
3. **Relief Mapping**：将 POM 推广到两次二分（内部反射也追踪），更适合凹槽较深的材质
4. **结合 Normal Map**：先用 Normal Map 做 coarse 估计，再用 POM 精确追踪
5. **GPU 并行**：在实际引擎中，此算法完全在 Pixel Shader 中执行，可以开全分辨率渲染

### 与本系列的关联

- **Normal Map（基础）** → **Parallax Mapping（UV 偏移）** → **POM（射线步进）** → **Relief Mapping（完整追踪）**：这是表面细节技术的渐进演化链
- 参考本系列第 29 天（PCSS 软阴影）：自阴影的步进逻辑与 PCSS 遮挡搜索非常相似
- 参考本系列第 14 天（SDF Ray Marching）：POM 是高度场上的特化 Ray Marching，思想相通

### 代码仓库

- GitHub: [daily-coding-practice/2026-05-17-pom](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-05-17-pom)
- 编译命令: `g++ main.cpp -o output -std=c++17 -O2 && ./output`
- 生成: `pom_output.png`（1200×700，4 种材质对比）

---

*下次实现方向：Cone Step Mapping（CSM，基于安全锥的加速 POM）或 Shell Texturing（毛皮/草地渲染）。*
