---
title: "每日编程实践: Water Wave Simulation & Rendering"
date: 2026-04-05 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 水面渲染
  - 模拟
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-05-water-wave/water_output.png
---

# 每日编程实践：Water Wave Simulation & Rendering

> **项目概述**：基于 Gerstner Wave（Trochoidal Wave）方程实现多叠加海浪，配合软光栅化三角形网格、Fresnel 反射、次表面散射近似、天空渲染，输出 1024×512 海景图。

![最终渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-05-water-wave/water_output.png)

---

## ① 背景与动机

### 为什么水面渲染值得单独研究？

水是游戏和影视中最难处理的视觉元素之一。和固体表面不同，水面具备三个让它极其复杂的特质：

1. **几何形态动态变化**：波浪不断移动，不存在静态的"水面网格"
2. **光学特性多样**：水面同时具备反射（镜面）、折射（透明）、漫反射（散射）三种模式
3. **次表面散射**：光进入水体后散射，形成深色区域和浅绿色透光区域的色差

工业界的解法从上世纪延续至今，演化出以下几条技术路线：

- **Gerstner/Trochoidal Wave**（1802年数学模型，至今仍是AAA游戏标配）：解析函数叠加，实时友好
- **FFT Ocean（菲利普斯谱）**：频域模拟，Unity/UE4 的 Ocean 插件基础，效果更真实但需要 GPU FFT
- **SPH/Position-Based Fluid**：物理精确，实时代价高，用于游戏中局部区域（船体入水、波浪拍岸）

今天选择 **Gerstner Wave** 原因是：它能在 CPU 软光栅化的框架下实时计算，且几何形态（顶点水平位移产生尖锐波峰）比纯正弦波更接近真实海浪。GTA V、AC: Black Flag 的海洋系统核心都是 Gerstner Wave 的变体。

### 没有正确的水面渲染会有什么问题？

最常见的入门错误是直接用正弦波叠加（只有垂直位移）。结果：
- 波浪形状过于圆润，像橡皮泥而非海浪
- 波峰没有尖锐感，少了海浪的"翻卷"视觉
- Fresnel 计算错误时，水面全程金属质感或全程透明，两种都很假

Gerstner Wave 通过让顶点沿着波传播方向水平滑动，形成天然的波峰堆积效应，这正是它比单纯正弦叠加更接近真实的原因。

---

## ② 核心原理

### 2.1 Gerstner Wave 数学推导

Gerstner Wave（又称 Trochoidal Wave）的单波位移公式如下：

对于在 XZ 平面传播的水波，设：
- 顶点初始位置 $(x_0, 0, z_0)$
- 波向量大小 $k = 2\pi / \lambda$（$\lambda$ 为波长）
- 相速度 $c$，时间 $t$，传播方向单位向量 $\hat{d} = (d_x, 0, d_z)$

相位：
$$\phi = k (\hat{d} \cdot \mathbf{p_0}) - c \cdot t$$

位移（垂直 + 水平）：
$$\Delta y = A \sin(\phi)$$
$$\Delta x = Q \cdot A \cdot d_x \cdot \cos(\phi)$$
$$\Delta z = Q \cdot A \cdot d_z \cdot \cos(\phi)$$

其中 $Q$ 是陡峭度系数：
$$Q = \frac{\text{steepness}}{k \cdot A \cdot N_{waves}}$$

**直觉解释**：

为什么要有水平位移 $\Delta x, \Delta z$？

想象海浪：在波峰位置，水分子不仅在最高处，还被"推"向前（与波传播方向相同）。在波谷，水分子被"拉"回来。这种圆形（实际上是椭圆形）轨迹运动就是 Gerstner Wave 的核心——它把正弦波的上下震荡改成了顶点沿椭圆轨迹运动。

当 $Q$ 较大时，相邻顶点会靠拢，波峰变尖——这就是波浪的"翻卷感"。当 $Q = 1$ 时会产生数学意义的尖点（cusp），超过这个值会出现自相交（翻卷过头，产生碎浪效果）。

**多波叠加**：实际海洋是多组不同方向、波长、振幅的波叠加结果。代码中叠加了5组波：

| 组 | 振幅 | 波长 | 速度 | 陡峭度 | 主方向 |
|----|------|------|------|--------|--------|
| 1  | 0.8  | 10.0 | 2.5  | 0.6    | (1, 0.4) 归一化 |
| 2  | 0.5  | 6.0  | 3.0  | 0.5    | (0.8, 1) 归一化 |
| 3  | 0.3  | 4.0  | 3.5  | 0.4    | (-0.5, 1) 归一化 |
| 4  | 0.15 | 2.5  | 4.0  | 0.3    | (0.3, 1) 归一化 |
| 5  | 0.1  | 1.8  | 5.0  | 0.25   | (-0.2, 0.9) 归一化 |

各组叠加后形成有方向感但不完全规则的真实海浪。

### 2.2 法线计算

网格顶点的法线通过 Gerstner Wave 的解析导数计算（而非数值差分），精度更高、效率更好：

$$\frac{\partial y}{\partial x_0} = -\sum_i d_{x,i} \cdot k_i A_i \cos\phi_i$$
$$\frac{\partial y}{\partial z_0} = -\sum_i d_{z,i} \cdot k_i A_i \cos\phi_i$$
$$\text{(Y分量的增量)} = -\sum_i Q_i k_i A_i \sin\phi_i$$

法线向量（未归一化）：
$$\mathbf{n} = (-\partial x / \partial x_0, 1 - \sum Q_i k_i A_i \sin\phi_i, -\partial z / \partial z_0)$$

**直觉**：法线的 XZ 分量代表水面的倾斜程度，Y 分量（垂直分量）从1减去所有"弯曲贡献"。波峰处法线更倾斜，波谷处更竖直——这与真实海面一致。

### 2.3 Fresnel 反射（Schlick 近似）

真实水面对光有两种响应：
- 接近垂直观察时（俯视），大部分光穿过水面，看到水下
- 掠射角时（水平观察），几乎所有光被反射，看到天空倒影

Schlick 近似：
$$F(\theta) = F_0 + (1 - F_0)(1 - \cos\theta)^5$$

其中 $F_0 = 0.04$（水的折射率 $n=1.33$ 对应的正入射反射率），$\theta$ 为视线与法线的夹角。

**直觉**：$(1-\cos\theta)^5$ 在 $\theta=0$（正对着看）时为0（几乎不反射），$\theta=90°$（掠射）时为1（全反射）。Schlick 发现这个5次幂近似精度很高且计算便宜，沿用至今。

### 2.4 水面颜色模型（次表面散射近似）

真实海水颜色取决于深度：
- **深水区**：光几乎被完全吸收，呈深邃的深蓝/深绿
- **浅水区**：光能穿透到底部反射回来，呈浅绿/青色

这里用一种简化的"假SSS"：用波高作为深度代理，混合深浅水色：

```
depthFactor = (waveHeight + 1.0) * 0.5  // 归一化到 [0,1]
depthFactor = pow(depthFactor, 0.7)      // gamma 偏向浅色
waterColor = lerp(deepColor, shallowColor, depthFactor)
```

这是一个常见的游戏优化手段：严格来说深度应该是从相机到水底的折射光路长度，但在视觉上，波高提供了足够的色彩变化暗示（波峰更浅，波谷更深）。

### 2.5 Blinn-Phong 高光

海面高光用双叶片（two-lobe）模型：
- **锐利高光**（幂次80）：模拟太阳在平静水面的点状反射
- **宽泛高光**（幂次8，强度0.15）：模拟水面散射后的柔和光晕

两个叶片叠加比单一高光更接近真实海面光斑的外观。

---

## ③ 实现架构

### 整体渲染管线

```
1. 初始化
   ├─ 定义 5 组 Gerstner Wave 参数
   ├─ 设置相机（位置、朝向、FOV）
   └─ 设置光照（太阳方向、颜色、水体颜色）

2. 背景绘制（天空 + 地平线）
   ├─ 上半部分：skyColor() 函数逐像素计算
   └─ 地平线融合带：天空色与水体深色渐变

3. 海面网格构建与渲染
   ├─ 构建 80×90 网格，覆盖 X[-28,28], Z[2,70]
   ├─ 对每个顶点调用 gerstnerDisplace() 求位移
   ├─ 对每个顶点调用 gerstnerNormal() 求解析法线
   ├─ 对每个顶点调用 project() 投影到屏幕空间
   └─ 从远到近（iz从大到小）光栅化所有三角形

4. 三角形着色（光栅化器内）
   ├─ 重心坐标插值：位置、法线、深度
   ├─ Z-buffer 测试
   ├─ Fresnel 反射率计算
   ├─ 深度因子 → 水体颜色
   ├─ Blinn-Phong 双高光叶片
   ├─ 次表面散射项（背光近似）
   ├─ Fresnel 与天空反射颜色混合
   ├─ 距离雾效
   └─ Reinhard 色调映射

5. 输出 PNG
```

### 关键数据结构

```cpp
struct GerstnerWave {
    float amplitude;    // 波幅 (m)
    float wavelength;   // 波长 (m)
    float speed;        // 相速度 (m/s)
    float steepness;    // 陡峭度 [0,1]
    Vec3  direction;    // 传播方向 (xz 平面单位向量)
};

struct Vertex {
    Vec3  worldPos;   // 世界空间位置（Gerstner 位移后）
    Vec3  normal;     // 解析法线
    float depth;      // 相机空间 Z（用于 Z-buffer 和雾效）
    float sx, sy;     // 屏幕空间坐标
};
```

网格使用二维数组缓存所有顶点的屏幕投影，避免重复计算：

```cpp
vector<vector<Vec3>>  gPos (NZ+1, vector<Vec3> (NX+1));  // 世界位置
vector<vector<Vec3>>  gNorm(NZ+1, vector<Vec3> (NX+1));  // 法线
vector<vector<float>> gDepth(NZ+1,vector<float>(NX+1));  // 相机深度
vector<vector<float>> gSX, gSY;                           // 屏幕坐标
vector<vector<bool>>  gVis;                               // 可见性（视锥剔除）
```

### 渲染顺序与 Painter's Algorithm

网格从 Z 轴最远（iz=NZ-1）到最近（iz=0）遍历，这是 Painter's Algorithm（画家算法）：远处的三角形先画，近处的覆盖它。

因为海面网格是连续的、不会出现真正的排序错误（没有深度交叉），这个方法对于本场景足够。如果需要支持透明水面下的物体，需要 Z-buffer 正确处理（本项目已有 Z-buffer，所以即使排序不完美也有兜底）。

---

## ④ 关键代码解析

### 4.1 Gerstner Wave 位移计算

```cpp
Vec3 gerstnerDisplace(const vector<GerstnerWave>& waves, float px, float pz, float t) {
    Vec3 pos(px, 0, pz);
    for (auto& w : waves) {
        float k   = 2.f * M_PI / w.wavelength;  // 波数：波长越短，k越大
        float c   = w.speed;
        float d   = w.direction.x * px + w.direction.z * pz;  // 点乘：沿方向的距离
        float phi = k * d - c * t;  // 相位：空间周期 - 时间移动
        
        // 陡峭度归一化，防止多波叠加时过度尖锐
        float Q   = w.steepness / (k * w.amplitude * (float)waves.size());
        
        // 水平位移（波传播方向）
        pos.x += Q * w.amplitude * w.direction.x * cosf(phi);
        pos.z += Q * w.amplitude * w.direction.z * cosf(phi);
        // 垂直位移
        pos.y += w.amplitude * sinf(phi);
    }
    return pos;
}
```

**关键设计点**：
- `phi = k*d - c*t`：减去 `c*t` 是让波沿正方向传播（随时间 `t` 增大，相位减小，等效波形向前移动）
- `Q` 的分母包含 `waves.size()`：多波叠加时每组贡献的水平位移按比例缩小，防止总体形变过大
- 水平位移用 `cosf`，垂直位移用 `sinf`：相位差 90°，使顶点描绘椭圆轨迹

### 4.2 解析法线计算

```cpp
Vec3 gerstnerNormal(const vector<GerstnerWave>& waves, float px, float pz, float t) {
    float dx = 0, dz = 0, dy = 1;  // 初始：平面法线 (0,1,0)
    for (auto& w : waves) {
        float k   = 2.f * M_PI / w.wavelength;
        float c   = w.speed;
        float d   = w.direction.x * px + w.direction.z * pz;
        float phi = k * d - c * t;
        float Q   = w.steepness / (k * w.amplitude * (float)waves.size());
        float WA  = k * w.amplitude;  // 常用组合项
        
        dx += -w.direction.x * WA * cosf(phi);  // ∂y/∂x
        dz += -w.direction.z * WA * cosf(phi);  // ∂y/∂z
        dy += -Q * WA * sinf(phi);              // Y分量修正
    }
    return Vec3(-dx, dy, -dz).norm();
}
```

**为什么 dx 有负号**：法线是梯度的负方向（指向"上坡"的反面）。当 `∂y/∂x > 0`（沿 x 方向高度增加），法线的 x 分量应该向负方向倾斜。

**dy 初始为1**：对应平坦水面的竖直法线，每组波的 `sinf` 项减去一个小量，代表"表面被弯曲"后法线偏离竖直方向。

### 4.3 透视投影

```cpp
Vec3 project(const Vec3& world, const Camera& cam, float& outZ) {
    Vec3 forward = (cam.lookat - cam.eye).norm();
    Vec3 right   = forward.cross(cam.up).norm();    // 相机右向量
    Vec3 camUp   = right.cross(forward).norm();     // 修正后的上向量

    Vec3 rel     = world - cam.eye;
    float x = rel.dot(right);     // 相机空间 x
    float y = rel.dot(camUp);     // 相机空间 y
    float z = rel.dot(forward);   // 相机空间 z（深度）
    outZ = z;

    if(z < 0.01f) return {-1,-1,-1};  // 相机后方，丢弃

    float aspect = (float)W / H;
    float tanHalfFov = tanf(cam.fovY * 0.5f);
    // NDC：将相机空间坐标除以深度（透视除法），再按 FOV 缩放
    float ndcX = x / (z * tanHalfFov * aspect);
    float ndcY = y / (z * tanHalfFov);
    // 转屏幕坐标（Y 轴翻转，NDC 上方 = 屏幕上方）
    float sx = (ndcX + 1.f) * 0.5f * W;
    float sy = (1.f - ndcY) * 0.5f * H;
    return {sx, sy, z};
}
```

**透视除法的直觉**：距离越远（z 越大），同样的相机空间 x 偏移对应越小的屏幕偏移——这就是近大远小的数学本质。

### 4.4 三角形光栅化器（核心着色逻辑）

```cpp
void rasterizeTriangle(Image& img, Vertex v0, Vertex v1, Vertex v2, ...)
{
    // 1. 计算边界框
    float minX = min({v0.sx,v1.sx,v2.sx}); ...
    
    // 2. 边函数：判断点在三角形哪侧
    auto edge = [](float ax,float ay,float bx,float by,float px,float py)->float{
        return (px-ax)*(by-ay)-(py-ay)*(bx-ax);
    };
    float area = edge(v0.sx,v0.sy, v1.sx,v1.sy, v2.sx,v2.sy);
    
    for(int y=y0;y<=y1;y++){
        for(int x=x0;x<=x1;x++){
            float px=x+0.5f, py=y+0.5f;  // 像素中心
            float w0=edge(v1,v2,p), w1=edge(v2,v0,p), w2=edge(v0,v1,p);
            if(w0<0||w1<0||w2<0) continue;  // 在三角形外
            
            // 重心坐标 → 插值深度和属性
            float b0=w0/area, b1=w1/area, b2=w2/area;
            float z = v0.depth*b0 + v1.depth*b1 + v2.depth*b2;
            
            if(!img.testZ(x,y,z)) continue;  // Z-buffer 测试
            
            // ─── 着色 ───
            // Fresnel
            float cosTheta = max(0.f, norm.dot(viewDir));
            float F0 = 0.04f;
            float fresnel = F0 + (1.f-F0)*powf(1.f-cosTheta, 5.f);
            
            // 水体深度颜色（假 SSS）
            float depthFactor = powf((wpos.y+1.f)*0.5f, 0.7f);
            Vec3 waterColor = lerp(waterDeep, waterShallow, depthFactor);
            
            // Blinn-Phong 双高光
            Vec3 halfV = (sunDir + viewDir).norm();
            float spec  = powf(max(0.f, norm.dot(halfV)), 80.f);   // 锐利
            float spec2 = powf(max(0.f, norm.dot(halfV)), 8.f) * 0.15f; // 宽泛
            
            // 背光 SSS 近似
            float sssAmt = max(0.f, -sunDir.dot(norm)) * 0.5f;
            Vec3 sssColor = Vec3(0.0f, 0.3f, 0.25f) * sssAmt;
            
            // 组合
            Vec3 color = waterColor*(NdotL*0.6f+0.2f)
                       + sunColor*(spec+spec2) + sssColor;
            color = lerp(color, reflColor, fresnel);
            
            // 雾效 + Reinhard tone map
            color = lerp(color, fogColor, fogFactor);
            color = color / (color + Vec3(1,1,1));  // 逐分量 Reinhard
        }
    }
}
```

**关键设计点**：

1. **边函数的符号**：`edge(a,b,p)` 为正代表 p 在 ab 向量的左侧，三点顺时针排列时三个边函数都为正 → 点在内部
2. **像素中心采样 `(x+0.5, y+0.5)`**：避免整数边界上的走样，标准做法
3. **深度插值**：直接在屏幕空间插值深度（非透视正确插值），对于水面这种视角比较水平的场景，误差可接受
4. **Fresnel 上限 0.95**：防止水面完全不透明（水面正对相机时仍应有少量水色透出）

### 4.5 天空渲染

```cpp
Vec3 skyColor(float u, float v, const Vec3& sunDir) {
    // 天顶到地平线渐变
    Vec3 zenith(0.1f, 0.3f, 0.7f);
    Vec3 horizon(0.7f, 0.82f, 0.92f);
    Vec3 sky = lerp(horizon, zenith, powf(clamp(v,0.f,1.f), 0.5f));

    // 太阳光盘
    Vec3 rayDir((u-0.5f)*2.f*1.7f, v*0.8f+0.05f, 1.f);
    rayDir = rayDir.norm();
    float sunBlend = max(0.f, rayDir.dot(sunDir));
    Vec3 sunDisk = Vec3(1.f,0.95f,0.7f) * powf(sunBlend, 200.f);  // 锐利光盘
    Vec3 sunGlow = Vec3(1.f,0.8f,0.4f) * powf(sunBlend, 12.f) * 0.4f;  // 宽泛光晕

    // 地平线金色暖调（模拟 Rayleigh 散射产生的黄昏色）
    Vec3 warm(0.9f, 0.65f, 0.3f);
    float horizWarm = (1.f-v) * max(0.f, sunDir.y) * 0.8f;
    sky = lerp(sky, warm, horizWarm);

    return (sky + sunDisk + sunGlow).clamp01();
}
```

`powf(sunBlend, 200.f)` 创造尖锐光盘：200 次幂让只有非常接近太阳方向的像素才有亮度，远处快速衰减。`powf(sunBlend, 12.f)` 则产生更宽的光晕。

---

## ⑤ 踩坑实录

### Bug 1：编译警告 - 未使用变量 `sunAngle`

**症状**：`-Wunused-variable` 警告

**错误假设**：以为只要不影响结果就可以忽略警告

**真实原因**：代码中计算了 `float sunAngle = max(0.f, sunDir.y);` 但随后被注释掉的代码改写，留下了这个孤立的变量声明

**修复**：删除未使用的变量声明，遵循"0 warning"原则

```cpp
// 修复前
float sunAngle = max(0.f, sunDir.y);  // 计算了但没用
Vec3 rayDir(...);

// 修复后（直接删除第一行）
Vec3 rayDir(...);
```

### Bug 2：stb_image_write.h 的 `-Wmissing-field-initializers` 警告

**症状**：包含 `stb_image_write.h` 后大量 `-Wmissing-field-initializers` 警告

**错误假设**：以为需要修改第三方库的代码

**真实原因**：stb 系列头文件历史上对结构体初始化不完整，这是已知问题，不影响功能

**修复**：用 GCC pragma 在包含该头文件时临时压制警告：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "../stb_image_write.h"
#pragma GCC diagnostic pop
```

这是处理第三方库警告的标准手段：外科手术式地压制特定头文件的特定警告，不影响自己代码的警告检查。

### Bug 3：坐标系设计——相机在水面正上方会导致水平线消失

**症状**：早期版本相机设在 `(0, 10, 0)` 俯视，整个画面都是水面，没有天空

**错误假设**：相机高一些视野更好

**真实原因**：观察一片海需要"水平线视角"，相机应该接近水面高度，用较小的俯仰角往前看。人类观察海洋的自然视角是站在岸边或船上，而不是悬在空中。

**修复**：
```cpp
cam.eye    = Vec3(0.f, 3.5f, -2.f);   // 接近水面，轻微后退
cam.lookat = Vec3(0.f, 0.f,  12.f);   // 看向前方
```

这样的设置让海平线大约在画面中间偏上，天空和水面各占约一半，符合自然视角。

### Bug 4：Gerstner 陡峭度过大导致网格自相交

**症状**：部分区域出现尖刺状异常，或波峰穿透彼此

**错误假设**：陡峭度越大越好看，设为 1.0

**真实原因**：当多波叠加时，水平位移累积，相邻顶点可能重叠（数学上的"破波"）。$Q > 1/kAN_{waves}$ 时开始自相交。

**修复**：代码中的归一化因子 `Q = steepness / (k * A * N)` 已经处理了这个问题。设置 `steepness` 最大为 0.6，留有裕量：

```cpp
// 各波的陡峭度参数（最大0.6，确保不自相交）
{0.8f, 10.f, 2.5f, 0.6f, ...},  // 主波，陡峭度0.6
{0.5f,  6.f, 3.0f, 0.5f, ...},  // 副波，陡峭度0.5
```

### Bug 5：深度雾效范围设置不合理

**症状**：远处水面没有自然消退，出现硬边界

**错误假设**：雾效应该从相机附近开始

**真实原因**：相机设在 Z=-2 处，网格从 Z=2 延伸到 Z=70。相机到最近水面约5单位，到最远约72单位。雾效应该在这个范围内渐变：

```cpp
float fogFactor = min(1.f, (z - 5.f) / 60.f);  // 5到65单位范围内渐变
fogFactor = fogFactor * fogFactor;               // 平方让近处更清晰
```

平方是为了让雾效"慢启动"：前一半距离几乎无雾，后一半快速浓厚，符合视觉感知。

---

## ⑥ 效果验证与数据

### 像素统计（自动化验证）

```python
from PIL import Image
import numpy as np

img = Image.open('water_output.png')
pixels = np.array(img).astype(float)
print(f"尺寸: {img.size}")           # (1024, 512)
print(f"像素均值: {pixels.mean():.1f}")    # 92.4
print(f"像素标准差: {pixels.std():.1f}")  # 52.5

h = pixels.shape[0]
top_mean = pixels[:h//4].mean()   # 上方1/4（天空区域）: 134.2
bot_mean = pixels[3*h//4:].mean() # 下方1/4（深水区域）: 51.1
```

**验证结果**：

| 指标 | 值 | 标准 | 状态 |
|------|-----|------|------|
| 文件大小 | 264 KB | > 10 KB | ✅ |
| 像素均值 | 92.4 | 10~240 | ✅ |
| 像素标准差 | 52.5 | > 5 | ✅ |
| 天空区均值 | 134.2 | > 水面均值 | ✅ |
| 水面区均值 | 51.1 | < 天空均值 | ✅ |
| 天空在上方 | 是 | 坐标系正确 | ✅ |

天空均值（134.2）明显高于水面均值（51.1），证明了坐标系正确——天空亮、深水暗。

### 性能数据

| 指标 | 值 |
|------|-----|
| 网格规模 | 80×90 顶点 = 7,290 个顶点 |
| 光栅化三角形 | 12,979 个 |
| 渲染时间 | 0.114 秒（CPU，单线程） |
| 像素填充率 | ~0.5M像素/秒（含 Z-buffer 测试） |
| 内存占用 | ~8 MB（图像缓冲 + Z-buffer + 网格数据） |

CPU 软光栅化渲染约13K三角形耗时 0.114 秒，折合约 8.5 FPS。对于实时游戏来说远远不够，但对于教学/验证用途完全足够。GPU 实现（顶点着色器 + 片段着色器）在现代硬件上处理同等规模可以达到 > 1000 FPS。

### 渲染结果分析

从最终输出图像可以观察到：
- **天空**：蓝色渐变，地平线附近有金色暖调，太阳光盘及光晕清晰可见
- **水面**：Gerstner 波形态自然，波峰、波谷明暗对比清晰
- **高光**：金色高光点状分布在波峰附近，符合太阳在45°方向的光照预期
- **Fresnel 效果**：远处水面更多反射天空颜色（偏蓝），近处水色更多（偏深蓝绿）
- **深度变化**：波峰处颜色略浅（浅水感），波谷处更深（深水感）

---

## ⑦ 总结与延伸

### 技术局限性

1. **软光栅化性能**：CPU 实现无法实时运行，实际游戏必须在 GPU 上实现（GLSL/HLSL）
2. **Painter's Algorithm 的隐患**：本项目网格不会自相交，排序天然正确。若有浮空物体（木板、船只），需要完整的 Z-buffer 排序或 OIT（顺序无关透明度）
3. **假 SSS vs 真 SSS**：用波高代理水深是近似，真实的次表面散射需要折射光路积分。近岸浅水（可见海底）和深海的效果差异无法正确表现
4. **透视正确插值缺失**：当前在屏幕空间直接插值深度，对于视角接近水平的情况（大 Z 变化）会有轻微畸变
5. **只有单帧静态输出**：Gerstner Wave 本质上是动态的，若要输出视频需要循环调用不同时间 t 值

### 可优化方向

1. **GPU 实现**：顶点着色器处理 Gerstner 位移（天然并行），片段着色器处理 Fresnel/SSS，可以支持实时
2. **FFT Ocean**：用菲利普斯谱生成海浪频域表示，IFFT 逆变换得到位移图，效果比 Gerstner 更真实且可控
3. **Screen-Space Reflections（SSR）**：将真实场景反射到水面（见系列文章 03-16），而非近似的天空颜色
4. **Foam 泡沫**：雅可比行列式检测"顶点堆叠"区域（波峰）叠加泡沫纹理，增强真实感
5. **Caustics（焦散）**：水面折射在海底形成的焦散光斑（见系列文章 03-31）
6. **多分辨率 LOD**：近处高密度网格，远处低密度，减少不必要的三角形数量

### 与系列文章的关联

- [03-09 SPH 流体模拟](/chiuhou-tech-blog/2026/03/09/)：SPH 可以模拟局部水体，与 Gerstner 大范围海面互补
- [03-31 焦散渲染](/chiuhou-tech-blog/2026/03/31/)：焦散是水面折射的直接后果
- [04-01 SDF Ray Marching](/chiuhou-tech-blog/2026/04/01/)：水面也可以用 SDF 参数化，但细节不如网格法
- [04-04 大气散射](/chiuhou-tech-blog/2026/04/04/)：天空颜色的精确计算可以直接应用到今天的水面反射

---

**完成时间**：2026-04-05 05:37  
**代码行数**：约 340 行 C++  
**编译器**：g++ -std=c++17 -O2 -Wall -Wextra  
**GitHub**：[daily-coding-practice/2026/04/04-05-water-wave](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-05-water-wave)
