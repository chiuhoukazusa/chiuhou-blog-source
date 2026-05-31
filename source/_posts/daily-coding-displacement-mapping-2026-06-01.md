---
title: "每日编程实践: Displacement Mapping & Tessellation Renderer"
date: 2026-06-01 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 光线步进
  - PBR
  - 地形渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-01-displacement-mapping/displacement_output.png
---

![渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-01-displacement-mapping/displacement_output.png)

今天实现了一个基于光线步进（Ray Marching）的位移贴图地形渲染器，将程序化高度场生成、精确法线重计算与 PBR 着色整合到一个约 550 行的 C++ 单文件里，0.57 秒渲染出 800×600 的地形图景。

<!-- more -->

---

## ① 背景与动机：为什么需要位移贴图？

### 法线贴图的极限

在实时渲染中，**法线贴图（Normal Mapping）** 是最常见的表面细节增强手段。它的核心思想是：不改变几何体的真实形状，只欺骗光照计算——用一张存储了扰动法线方向的纹理，让平坦的多边形在光照下"看起来"有凹凸感。

这在大多数时候足够用了。但法线贴图有一个根本局限：**轮廓（Silhouette）线依然是假的**。当你的视角和法线方向接近垂直时，表面轮廓处会显现出它本质上还是一个平面。在山脉的天际线、石头的边缘，这种瑕疵会显著破坏真实感。

**位移贴图（Displacement Mapping）** 则更进一步——它真正移动了顶点（或者在光线追踪/光线步进的情境下，直接改变了几何相交的判断依据），让表面在物理上就有了高低起伏。这带来了：

1. **正确的轮廓线**：山脊的天际线是真实的凸起，不再是假法线造的幻觉
2. **自遮挡（Self-Occlusion）**：山谷会真正遮挡来自侧面的光线
3. **视差位移**：从不同角度观察，高点真的"凸出来"了

### 工业界应用场景

- **Unreal Engine 5 的 Nanite**：将超精细几何体（数亿面）实时渲染，其背后的虚拟几何技术本质上就是不同分辨率的细分位移
- **地形系统（Landscape/Terrain）**：几乎所有 3A 游戏的地形都是高度图驱动的位移——一张 8192×8192 的 16-bit 高度图，加上四层细分，能生成极高密度的地形网格
- **海洋渲染（Ocean Surface）**：《Assassin's Creed: Black Flag》的海洋用 GerstnerWave + FFT 海浪高度图驱动顶点位移，兼顾实时性和视觉真实感
- **角色皮肤 / 织物**：Disney 的《冰雪奇缘》用位移贴图+细分曲面（Catmull-Clark）渲染雪的纹理和 Elsa 的裙摆褶皱

### 软光栅化 vs GPU Tessellation

真正的 GPU 实现会通过 Hull Shader + Domain Shader（DX11）或 Tessellation Control/Evaluation Shader（OpenGL）在硬件管线中完成细分和位移。但在这个 CPU 软光栅化项目里，我们用的方案是：

> **光线步进（Ray Marching）直接对高度场求交**

这等价于"无限细分"——光线与真实高度场相交，完全不受网格分辨率限制。代价是渲染速度较慢（每像素需要数十次迭代），但对于一个单文件 C++ 教学项目来说，这是最干净、最能体现位移贴图本质的做法。

---

## ② 核心原理

### 2.1 分形布朗运动（FBM）高度场

程序化地形生成的核心是 **分形布朗运动（Fractional Brownian Motion, FBM）**。一张真实感地形的高度图不是简单的 Perlin Noise，而是多个不同频率噪声的叠加：

```
H(x, y) = Σ_{i=0}^{N-1} amplitude_i * Perlin(x * frequency_i, y * frequency_i)
```

其中每一级（octave）的参数满足：
- `frequency_i = lacunarity^i`，lacunarity（间隙率）通常取 2.0，意味着每级频率翻倍
- `amplitude_i = gain^i`，gain（增益）通常取 0.5，意味着每级幅度减半

**直觉解释**：地球上的地形从宏观看是大陆尺度的隆起，从中观看是山脉起伏，从微观看是岩石纹理——每一个尺度都在独立贡献细节，且越小尺度对整体形状的影响越小（幅度 0.5^i 快速衰减）。FBM 正是在数学上模拟了这种多尺度叠加。

```cpp
static float fbm(float x, float y, int octaves=6, float lacunarity=2.0f, float gain=0.5f){
    float val=0, amp=0.5f, freq=1.0f, maxVal=0;
    for(int i=0;i<octaves;i++){
        val += perlin(x*freq, y*freq, 0.5f)*amp;  // 固定 z=0.5 取 2D 切片
        maxVal += amp;  // 记录最大可能幅度，用于归一化
        amp *= gain;    // 每级幅度减半
        freq *= lacunarity;  // 每级频率翻倍
    }
    return val/maxVal;  // 归一化到 [-1, 1]
}
```

本项目用了 **8 倍频**（比常见的 6 倍更多细节），并在 FBM 基础上叠加了一个高斯山峰：

```cpp
float base = fbm(fx, fy, 8, 2.0f, 0.5f);
// 中心高斯山峰
float cx=0.5f, cy=0.5f;
float d = sqrt((nx-cx)*(nx-cx)+(ny-cy)*(ny-cy));
float peak = exp(-d*d*8) * 0.4f;
h[y*sz+x] = base*0.6f + peak + 0.3f;
```

`exp(-d²*8)` 是一个以 `(0.5, 0.5)` 为中心的高斯函数，乘以 `0.4f` 控制山峰高度。这模拟了地质活动产生的"主峰 + 周边起伏"地形结构。

最后对整个高度场做**全局归一化**到 `[0, 1]`，方便后续颜色分层和光线步进时的高度比较。

### 2.2 光线步进（Ray Marching）与自适应步长

#### 基础光线-高度场相交

对于一条射线 `P(t) = O + t*D`，我们需要找最小的 `t` 使得：

```
P(t).y ≤ H(P(t).x, P(t).z)  ← 光线到达了地面或地面以下
```

暴力做法是固定步长遍历 t，但这很低效。**自适应步长**的思路来自于 Sphere Tracing 的精髓：

> 当前距地面的高度越大，我们可以迈的步子越大（因为不可能在这段空间内穿透地面）

```
dt = max(0.02, min(above * 0.5 + 0.02, 0.5))
```

`above = P.y - H(u,v)` 是当前点到地面的距离。当 `above` 很大时（高空中），步长接近 0.5；当接近地面时，步长缩小到 0.02，保证精度。

一旦检测到 `above < 0`（穿入地面），进入**二分细化**：

```cpp
for(int i=0;i<8;i++){
    t -= dt*0.5f; dt*=0.5f;  // 折半
    p = ray.o + ray.d*t;
    u = p.x/terrainW; v=p.z/terrainD;
    th = hf.sample(u,v)*terrainH;
    above = p.y - th;
    if(above>0) t+=dt;  // 在地面以上，往前走
    // 在地面以下，继续折半
}
```

8 次二分能将精度提升到 `0.02 / 2^8 ≈ 0.000078`，完全够用。

#### 双线性插值采样

高度场是离散的 256×256 网格，直接取整会产生明显的锯齿。双线性插值解决这个问题：

```cpp
float sample(float u, float v) const {
    float fx=u*(size-1), fy=v*(size-1);
    int ix=(int)fx, iy=(int)fy;
    float tx=fx-ix, ty=fy-iy;  // 小数部分 = 插值权重
    float a=get(ix,iy), b=get(ix+1,iy);
    float c=get(ix,iy+1), d=get(ix+1,iy+1);
    return lerp(lerp(a,b,tx), lerp(c,d,tx), ty);
}
```

**直觉**：`tx, ty` 是当前采样点在网格单元内的相对位置（0 到 1）。先在 x 方向插值两次，再在 y 方向插值一次，得到平滑的高度值。

### 2.3 法线重计算：差分法

**这是位移贴图最关键的步骤之一。** 顶点位移后，原始网格法线完全失效——我们必须根据新的几何形状重新计算法线。

**有限差分（Finite Difference）** 方法：在 UV 空间中，沿 u 和 v 方向各偏移一个微小量 ε，通过高度差近似切线方向：

```
∂P/∂u ≈ (P(u+ε, v) - P(u, v)) / ε = {ε*terrainW, (H(u+ε,v) - H(u,v)) * dispScale, 0}
∂P/∂v ≈ (P(u, v+ε) - P(u, v)) / ε = {0, (H(u,v+ε) - H(u,v)) * dispScale, ε*terrainD}
```

法线 = 两个切线向量的叉积（归一化）：

```cpp
Vec3 normal(float u, float v) const {
    float eps=1.0f/size;
    float h0=sample(u,v);
    float hx=sample(u+eps,v);
    float hy=sample(u,v+eps);
    Vec3 dx={eps*10, (hx-h0)*2.5f, 0};   // u 方向切线
    Vec3 dy={0, (hy-h0)*2.5f, eps*10};   // v 方向切线
    return dy.cross(dx).norm();
}
```

注意叉积的顺序 `dy × dx`（不是 `dx × dy`）——这确保法线指向地面的上方而不是下方。系数 `2.5f` 和 `10` 控制法线的"尖锐程度"，与位移比例 `dispScale=2.5f` 匹配。

**为什么不直接用高度场的梯度？**

梯度 `(∂H/∂u, ∂H/∂v)` 可以给你切线平面的方向，但叉积法更鲁棒——它天然处理了地形坐标系（XZ 平面）和法线方向的关系，不需要手动做坐标变换。

### 2.4 PBR 着色：Cook-Torrance BRDF

基于物理的渲染（PBR）的核心是 **Cook-Torrance 微面元模型**：

```
f_r(ω_i, ω_o) = f_diffuse + f_specular
             = (k_d * C_base / π) + (D * G * F) / (4 * (N·V) * (N·L))
```

三个核心项：

**D（Distribution）：微面元法线分布函数（GGX/Trowbridge-Reitz）**

```
D_GGX(m) = α² / (π * ((N·H)² * (α²-1) + 1)²)
```

`α = roughness²` 是粗糙度的平方（感知线性化）。`H` 是半程向量 `(V+L)/|V+L|`。

**直觉**：越粗糙（α 大），高光越散（分布越宽）；越光滑（α 小），高光越集中（只有 H ≈ N 时才有贡献）。

**G（Geometry）：微面元遮蔽-阴影项（Smith + Schlick）**

```
G_Smith = G_Schlick(N·V) * G_Schlick(N·L)
G_Schlick(x) = x / (x * (1-k) + k),  k = (roughness+1)² / 8
```

防止掠射角（grazing angle）时能量不守恒——当光线或视线几乎平行于表面时，大量微面元互相遮挡，实际出射能量减小。

**F（Fresnel）：菲涅尔反射率（Schlick 近似）**

```
F(θ) ≈ F₀ + (1 - F₀) * (1 - cos θ)⁵
```

`F₀` 是法向入射时的反射率，对非金属取 0.04（约 4%），对金属取基础色。Schlick 近似用 5 次方来拟合真实的菲涅尔曲线，误差 < 1%。

**直觉**：任何材质在掠射角（θ ≈ 90°）时都接近全反射——水面、玻璃、甚至皮肤在侧面都会显出高光，这就是菲涅尔效应。

```cpp
static Vec3 FresnelSchlick(float cosTheta, Vec3 F0){
    float t=std::pow(1-cosTheta,5);
    return F0 + (Vec3{1,1,1}-F0)*t;
}
```

### 2.5 软阴影：步进阴影射线

从地表交点向光源方向发射阴影射线，进行光线步进：

```
shadow(P, L) = min_{t∈[0.2,15]} ( k * above(t) / t )
```

`above(t)` 是光线路径中当前点到地面的高度，`k=8` 控制半影的柔化程度。

**直觉**：这是 Íñigo Quílez 的经典软阴影公式。当光线从 P 出发，在某个 t 处距地面只有很小的 `above(t)` 时，说明光线"险些被地面遮挡"——`k * above(t) / t` 会很小，产生软阴影。`t` 在分母是因为距离越远的遮挡影响越小。

```cpp
static float softShadow(const Vec3& origin, const Vec3& lightDir, const Heightfield& hf,
                         float dispScale=2.0f, float k=8.0f)
{
    float res=1.0f;
    float dt=0.1f;
    for(float t=0.2f; t<15.0f; t+=dt){
        Vec3 p = origin + lightDir*t;
        float u=p.x/terrainW, v=p.z/terrainD;
        if(u<0||u>1||v<0||v>1) break;
        float th = hf.sample(u,v)*terrainH;
        float above = p.y - th;
        if(above<0) return 0.05f;  // 完全遮挡
        res = std::min(res, k*above/t);  // 更新最小"余量比"
        dt = std::max(0.05f, above*0.4f);  // 自适应步长
    }
    return clamp(res, 0.05f, 1.0f);
}
```

---

## ③ 实现架构

### 整体渲染管线

```
┌─────────────────────────────────────────────────────┐
│  程序化高度场生成                                      │
│  Heightfield(256×256)                               │
│  ├── FBM Perlin Noise (8 octaves)                   │
│  ├── 高斯山峰叠加                                    │
│  └── 全局归一化 [0,1]                               │
└───────────────────┬─────────────────────────────────┘
                    │ 高度场数据
┌───────────────────▼─────────────────────────────────┐
│  像素级光线生成（每像素一条射线）                        │
│  Camera::getRay(px, py)                             │
│  └── 透视投影矩阵（FOV=50°, Aspect=4:3）             │
└───────────────────┬─────────────────────────────────┘
                    │ Ray(O, D)
┌───────────────────▼─────────────────────────────────┐
│  光线步进地形相交                                     │
│  intersectTerrain()                                 │
│  ├── 自适应步长前进                                   │
│  ├── 检测穿入地面                                    │
│  └── 8次二分细化精确交点                              │
└───────────────────┬─────────────────────────────────┘
                    │ (t, u, v) 或未命中
          ┌─────────┴──────────┐
     未命中│                    │命中
┌──────────▼──────┐  ┌──────────▼─────────────────────┐
│  天空着色        │  │  地形着色                        │
│  skyColor()     │  │  ├── 高度采样 h=sample(u,v)     │
│  ├── 渐变天空   │  │  ├── 法线计算 N=normal(u,v)     │
│  └── 太阳 + 光晕│  │  ├── 分层颜色 terrainColor()    │
└────────┬────────┘  │  ├── 软阴影 softShadow()        │
         │           │  ├── PBR 着色 pbr()             │
         │           │  └── 距离雾效                    │
         └──────────┬┘
                    │ Color3
┌───────────────────▼─────────────────────────────────┐
│  ACES Filmic Tone Mapping + Gamma 校正               │
│  → 写入 PNG (stb_image_write)                       │
└─────────────────────────────────────────────────────┘
```

### 关键数据结构

**Heightfield** — 高度场核心容器：

```cpp
struct Heightfield {
    int size;
    std::vector<float> h;  // 归一化高度值 [0,1]

    float get(int x, int y) const;       // 整数索引（边界 clamp）
    float sample(float u, float v) const; // 双线性插值采样
    Vec3  normal(float u, float v) const; // 差分法线
};
```

**Camera** — 透视相机：

```cpp
struct Camera {
    Vec3 origin, forward, right, up;
    float fov, aspect;
    struct Ray { Vec3 o, d; };
    Ray getRay(float px, float py) const;  // px,py 归一化到 [0,1]
};
```

**Color3 vs Vec3** — 故意分开：
- `Vec3`：方向、位置、法线，涉及几何运算（cross、normalize）
- `Color3`：颜色，R/G/B 的加法和缩放，有专用的 `mix()` 重载

两者独立的好处是：编译器能捕获到"把法线当颜色用"这类错误（类型不匹配），而不是默默算出错误结果。

### 地形 UV 映射

地形被定义在 `[0,10] × [0,10]` 的 XZ 平面上，Y 轴高度为 `[0, dispScale=2.5]`。

UV 坐标到世界坐标的映射：
```
world_x = u * terrainW  (terrainW = 10.0)
world_z = v * terrainD  (terrainD = 10.0)
world_y = h(u,v) * dispScale
```

相机位于 `(5, 6, -3)`，看向 `(5, 0.5, 5)` —— 从地形一端的高处，沿 Z 正方向斜向下俯瞰，能展示整个地形和远处的天际线。

---

## ④ 关键代码解析

### 4.1 Perlin Noise 实现细节

```cpp
static float grad(int h, float x, float y, float z){
    int hh=h&15;  // 取低 4 位，得到 0~15 共 16 种情况
    float u=hh<8?x:y;   // 前 8 种用 x，后 8 种用 y
    float v=hh<4?y:(hh==12||hh==14?x:z);  // 特殊处理均匀分布
    return ((hh&1)?-u:u)+((hh&2)?-v:v);   // bit1/bit2 决定正负
}
```

这是 Ken Perlin 的原始实现。每个格点映射到 16 个预设的梯度向量之一（这 16 个向量均匀分布在立方体的边上），通过位运算高效选取，然后计算梯度与距离向量的点积。

**关键**：`perm[]` 置换表经过 Fisher-Yates 随机打乱（`std::shuffle`），保证每次不同的种子产生不同的噪声模式，但对同一种子是确定性的（可重现）。

### 4.2 地形颜色分层

地形颜色基于两个维度：**绝对高度** h 和 **地形坡度** slope：

```cpp
static Color3 terrainColor(float h, float slope, const Vec3& /*normal*/){
    Color3 deepWater  {0.05f, 0.15f, 0.35f};  // 深海蓝
    Color3 shallowWater{0.1f, 0.4f, 0.6f};    // 浅水青蓝
    Color3 sand       {0.76f, 0.70f, 0.50f};  // 沙滩米黄
    Color3 grass1     {0.25f, 0.55f, 0.15f};  // 低草绿
    Color3 grass2     {0.20f, 0.45f, 0.10f};  // 高草深绿
    Color3 rock1      {0.45f, 0.40f, 0.35f};  // 浅岩石
    Color3 rock2      {0.35f, 0.30f, 0.25f};  // 深岩石
    Color3 snow       {0.90f, 0.92f, 0.95f};  // 雪顶冷白

    Color3 col;
    if(h < 0.12f) col = mix(deepWater, shallowWater, h/0.12f);
    else if(h < 0.18f) col = mix(shallowWater, sand, (h-0.12f)/0.06f);
    // ... 共 8 层
    
    // 坡度覆盖：陡坡用岩石颜色
    if(slope > 0.5f){
        float t=saturate((slope-0.5f)/0.3f);
        col = mix(col, rock2*Color3(1.1f,1.05f,1.0f), t*0.6f);
    }
    return col;
}
```

`slope = 1 - N.y`：法线 Y 分量越大（越竖直），坡度越小；法线 Y 越小（越水平），坡度越大。当坡度 > 0.5 时（大约 60° 以上的斜坡），渐变混入岩石颜色，因为陡坡上草不会生长，只有裸露岩石。

### 4.3 水面特殊处理

低高度区域（h < 0.15f）判定为水面，有额外的特殊处理：

```cpp
if(isWater){
    metallic  = 0.1f;
    roughness = 0.08f;  // 极低粗糙度 → 镜面反射
    baseColor = {0.05f, 0.25f, 0.45f};  // 深蓝基础色
    
    // Perlin 扰动模拟水波法线
    float wn = perlin(P.x*2+0.5f, P.z*2+0.3f, 0.5f)*0.12f;
    N = Vec3{N.x+wn, N.y, N.z+wn*0.7f}.norm();
}
```

粗糙度 0.08 会产生非常集中的高光，配合高菲涅尔反射率（随入射角变化显著），让水面在视线接近平行时出现强烈的镜面高光，模拟真实水面效果。

### 4.4 大气雾效

```cpp
float fogDist = t / 80.0f;
fogDist = saturate(fogDist*fogDist);  // 平方使近处雾效更淡
Color3 fogColor{0.65f, 0.78f, 0.92f};  // 带蓝调的雾色
col = mix(col, fogColor, fogDist);
```

`t / 80.0f` 将光线步进参数 t 归一化（80 是场景中能看到的最远距离），`t²` 的指数曲线让近处几乎无雾、远处快速变蓝——这符合大气散射的物理特性（蓝光散射更多，远处天空的蓝光"染色"地形）。

### 4.5 ACES Filmic Tone Mapping

```cpp
static Color3 aces(Color3 c){
    float a=2.51f, b=0.03f, cc=2.43f, d=0.59f, e=0.14f;
    auto f=[&](float x){ return saturate((x*(a*x+b))/(x*(cc*x+d)+e)); };
    return {f(c.r), f(c.g), f(c.b)};
}
```

这是 ACES（Academy Color Encoding System）Filmic 曲线的 Krzysztof Narkowicz 近似公式。分子 `x*(2.51x+0.03)` 是一个向上弯曲的曲线，分母 `x*(2.43x+0.59)+0.14` 压制高亮部分，使整体呈 S 形响应曲线：

- 暗部对比度略微提升（曲线斜率 > 1）
- 中间调色彩准确保留
- 高光柔和压缩（不会出现"高光爆掉一片白"）

在调用前乘以 `1.2f` 相当于调高曝光值：`aces(col * 1.2f)`，让整体更亮一些，避免场景看起来欠曝。

---

## ⑤ 踩坑实录

### Bug #1：Color3 与 Vec3 混用导致 8 个编译错误

**症状**：第一次编译，`terrainColor()` 和 `skyColor()` 函数里大量 `invalid initialization of reference of type 'const Vec3&' from expression of type 'Color3'` 错误，共 8 处。

**错误假设**：以为 `mix()` 函数只有一个重载接受 `Vec3`，而 `Color3` 应该能隐式转换（因为结构体成员都是 float）。

**真实原因**：C++ 没有隐式结构体转换。`Color3` 和 `Vec3` 是独立的结构体，即使成员相同也不能互转。`mix(const Vec3&, const Vec3&, float)` 完全无法接受 `Color3` 参数。

**修复方式**：为 `Color3` 单独提供 `mix()` 重载：
```cpp
inline Color3 mix(const Color3& a, const Color3& b, float t){ return a*(1-t)+b*t; }
```

但这个重载要放在 `Color3` 结构体定义**之后**，放在 `Vec3 mix()` 下方会导致 `Color3` 未定义报错。最终把 `Color3::mix` 放在结构体末尾的 `};` 之后，一次解决。

**教训**：涉及两个相似结构体的自由函数重载，要特别注意定义顺序。先定义结构体，再定义与其相关的自由函数。

### Bug #2：unused variable 警告

**症状**：`prevH` 变量被声明但只赋值、从未读取，产生 `-Wunused-but-set-variable` 警告。

**原因**：早期版本中曾用 `prevH` 做步长启发式计算，重构时忘记删除了。

**修复**：直接删除 `float prevH = 1e9f;` 和 `prevH = above;` 两行。

### Bug #3：unused parameter 警告

**症状**：`terrainColor(float h, float slope, const Vec3& normal)` 的 `normal` 参数从未使用，产生 `-Wunused-parameter` 警告。

**原因**：最初设计时打算用法线做一些各向异性着色（根据法线角度混合不同颜色），后来简化为只用高度和坡度，但参数没删。

**修复**：将参数名注释掉：`const Vec3& /*normal*/`。这保留了接口的语义（调用者知道可以传法线），但告诉编译器我们有意不使用它，消除警告。

### Bug #4：色调映射前未调整曝光导致场景偏暗

**症状**：首次渲染的原始图像颜色偏灰暗，天空蓝色不饱和，地形对比度低。

**原因**：PBR 计算中光源强度 `3.5` 是物理单位，经过多次能量守恒计算后，最终颜色值在 `[0, ~2]` 范围，而 ACES 曲线的"最佳输入范围"大约是 `[0, ~4]`。直接输入低值会让曲线工作在线性段（没有 S 形压缩效果）。

**修复**：在 tone mapping 之前乘以曝光值：`aces(col * 1.2f)`。这把输入信号整体上移，让暗部进入曲线的对比增强区，高光进入压缩区，整体视觉效果更接近摄影感。

---

## ⑥ 效果验证与数据

### 输出结果

![地形渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-01-displacement-mapping/displacement_output.png)

*800×600，0.57 秒渲染完成。可以看到前景的草地和远处的山峰，软阴影在山脉东侧形成了自然的渐变阴影，水面的低粗糙度PBR高光清晰可见，远处地形渐入蓝灰色雾效。*

### 量化数据

| 指标 | 数值 |
|------|------|
| 渲染分辨率 | 800 × 600 |
| 总耗时（real） | 0.570s |
| 用户态耗时 | 0.561s |
| 系统调用耗时 | 0.004s |
| 高度场分辨率 | 256 × 256 |
| 最大光线步进步数 | 约 50~200 步/像素 |
| 软阴影采样步数 | 约 20~50 步/像素 |
| 像素均值 R | 171 |
| 像素均值 G | 193 |
| 像素均值 B | 172 |
| 整体像素均值 | 179.1 |
| 像素标准差 | 47.7 |
| 上方均值（天空） | 215.1 |
| 下方均值（地形） | 155.3 |
| 输出文件大小 | 689 KB |

### 验证脚本结果

```python
from PIL import Image
import numpy as np

img = Image.open('displacement_output.png')
pixels = np.array(img).astype(float)

mean = pixels.mean()  # 179.1
std  = pixels.std()   # 47.7

# 验证结果
assert 5 < mean < 250   # ✅ 非全黑/全白
assert std > 5           # ✅ 图像有内容变化
assert top_half > bottom_half  # ✅ 天空（上方）比地形（下方）亮，坐标系正确

# ✅ 所有验证通过
```

天空区域均值（215.1）明显高于地形区域均值（155.3），确认：
1. 相机坐标系正确，Y 轴朝上，天空在图像上方
2. 地形着色分层正常，近景地形有正确的明暗变化
3. 雾效将远处地形颜色拉向天空色，符合预期的颜色渐变

### 性能分析

0.57 秒渲染 48 万像素（800×600）：
- 平均 **8.4μs/像素**
- 包含 Ray Marching（自适应步长）+ 软阴影射线（独立步进）+ PBR 计算
- 全部单线程，未使用 SIMD 或多线程

与同系列项目对比：
- 04-29 SSS Renderer：~2.1s（需要多次散射采样）
- 05-23 SDF Ray Marching：~1.8s（SDF 评估比高度场插值更贵）
- 本项目：0.57s（高度场双线性插值 + 自适应步长效率较高）

---

## ⑦ 总结与延伸

### 当前实现的局限性

1. **纯 CPU 单线程**：没有利用 SIMD 或多线程，实际 GPU 实现可以快 100~1000 倍
2. **无 LOD（细节层次）**：远处地形和近处用同样的光线步进精度，浪费计算
3. **软阴影过于简单**：只考虑了地形对地形的遮挡，没有做基于 PCF 深度图的方案
4. **固定天空模型**：没有用真正的大气散射（Rayleigh + Mie），天空是硬编码的颜色渐变
5. **水面缺乏反射**：水面只有高光，没有环境反射（实际上需要反射光线步进）

### 可优化的方向

**短期（难度★★）**：
- 多线程并行（OpenMP 加速，`#pragma omp parallel for` 一行搞定）
- 各向异性粗糙度：沿风化方向的岩石纹理有方向性高光

**中期（难度★★★）**：
- 大气散射天空（参考 05-19 的实现），让天空颜色随太阳位置变化
- 水面反射：在水面命中时反射光线，继续步进（类似镜面间接光照）
- 地形 LOD：近处用更小步长、更多精化迭代，远处用大步长粗略

**长期（难度★★★★）**：
- GPU 实现：用 GLSL/HLSL 的 Tessellation Shader，在 Hull/Domain Stage 做细分和位移
- 虚拟纹理（Virtual Texturing）：超大地形（8192²）按需流式加载细节
- 程序化材质：基于高度、坡度、曝光度（朝向太阳的面）动态混合材质

### 与本系列其他文章的关联

- **05-27 程序化地形 + 水力侵蚀**：同样做了高度场地形，但用的是软光栅化（网格三角形），本文用 Ray Marching 直接求交，视觉细节更丰富但速度更慢
- **05-23 SDF Ray Marching**：同样是光线步进，但 SDF 是隐式几何（符号距离场），高度场是参数化的显式几何，步长策略不同
- **05-19 大气散射**：天空颜色模型，可以直接移植到本项目中替换当前的线性渐变天空
- **05-06 IBL Split-Sum PBR**：PBR 着色核心完全兼容，本项目用了同样的 Cook-Torrance BRDF，只是简化了 IBL 部分（用固定环境光代替环境贴图）

这个项目是图形学渲染管线中"几何表示"和"光照计算"两条线的交汇点：高度场 Ray Marching 属于几何表示/求交，PBR 着色属于光照计算。理解两者的解耦方式，是构建更复杂渲染器的基础。

---

**代码仓库**：[GitHub - chiuhoukazusa/daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/06/06-01-displacement-mapping)
