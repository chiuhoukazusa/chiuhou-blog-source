---
title: "每日编程实践: Lightmap Baker — 离线光照贴图烘焙器"
date: 2026-05-25 05:35:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 全局光照
  - 游戏开发
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-25-lightmap-baker/lightmap_baker_output.png
---

## 背景与动机

### 实时渲染的性能困境

现代游戏中，全局光照（Global Illumination, GI）始终是最烧 GPU 的特性之一。一个场景里可能有数十盏灯光，每盏灯都需要进行阴影计算、反射计算、间接光传播……如果全部实时计算，帧率会直接崩塌。

早期 Quake（1996）就面临这个问题：如何让室内场景有逼真的光影，又不破坏流畅的游戏体验？id Software 给出的答案是 **Lightmap（光照贴图）**——把光照信息预先计算好，存成一张贴图，实时渲染时直接采样贴图，代替实时光照计算。

这个技术在后来的 Quake 2、Half-Life、CS:GO，到 Unity Enlighten、虚幻引擎的 Lightmass，都是核心技术之一。游戏里那种"静态环境的柔和阴影"和"角落里的暗部渐变"，大多数时候是 Lightmap 的功劳，而不是实时 GI。

### 没有 Lightmap 的痛点

没有光照贴图时，常见的替代方案是：

1. **纯环境光**：全局加一个常数亮度，没有阴影，平坦无立体感
2. **实时阴影**：每帧计算 Shadow Map，开销随光源数量线性增长，多光源场景吃不消
3. **Screen Space GI（SSGI）**：仅能处理屏幕内的间接光，相机转动时光照突变，闪烁明显
4. **Voxel GI（VXGI）**：品质好，但 GPU 显存和计算开销很高，适合高端硬件

Lightmap 的核心优势：**离线烘焙，实时几乎零成本**。烘焙一次可能要几分钟甚至几小时（复杂场景），但运行时采样一张贴图只需要几个 texture fetch 指令，对任何硬件都友好。

### 工业界实际使用场景

- **Unity Enlighten**：Unity 的 CPU 端 GI 烘焙系统，支持静态物体的漫射 GI、AO、阴影烘焙
- **Unreal Engine Lightmass**：UE4/5 的离线烘焙系统，基于 Path Tracing，支持自发光材质、透明材质
- **Lumen 的前身**：UE5 的 Lumen 实时 GI 系统，对于静态物体仍然大量依赖预计算
- **《Minecraft》光照**：经典 Minecraft 的软阴影是通过简化版光照贴图实现的
- **《CS:GO》地图**：Source 引擎的 VRAD 工具专门用于烘焙 Lightmap，地图 compile 步骤之一

今天这个项目，我从头实现了一个简化的 Lightmap Baker，核心包括：UV Atlas 展开、辐照度采样（直接光 + 一次间接弹射）、Shadow Ray 遮挡测试，以及用烘焙后的光照贴图渲染 Cornell Box。

---

## 核心原理

### 渲染方程回顾

Lightmap Baker 的数学基础是**渲染方程**（Rendering Equation，Kajiya 1986）：

$$
L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_{\Omega} f_r(\mathbf{x}, \omega_i, \omega_o) L_i(\mathbf{x}, \omega_i) (\omega_i \cdot \mathbf{n}) \, d\omega_i
$$

其中：
- $L_o$ 是出射辐射率（radiance）
- $L_e$ 是自发光项
- $f_r$ 是双向反射分布函数（BRDF）
- $L_i$ 是入射辐射率
- $(\omega_i \cdot \mathbf{n})$ 是入射角余弦项
- 积分域 $\Omega$ 是上半球

对于漫射材质（Lambertian），BRDF 是常数：$f_r = \frac{\rho}{\pi}$，其中 $\rho$ 是 albedo。

**Lightmap 存的是什么？** 它存储的是 **辐照度（Irradiance）** $E$，即单位面积上接收到的总辐射通量：

$$
E(\mathbf{x}) = \int_{\Omega} L_i(\mathbf{x}, \omega_i) (\omega_i \cdot \mathbf{n}) \, d\omega_i
$$

有了辐照度，实时渲染时直接乘以 albedo 即可：

$$
L_o = \frac{\rho}{\pi} \cdot E
$$

### 为什么存辐照度而不存辐射率？

关键区别：辐照度 $E$ 是与视角**无关**的量（漫射情况下），而辐射率 $L_o$ 与视角有关。

只要材质是漫射（Lambertian），不管从哪个角度看，辐照度乘以 BRDF 得到的出射辐射率是一样的。这使得一次烘焙可以服务任意视角，完美适合预计算场景。

如果材质有高光项（Blinn-Phong、GGX），就不能只存辐照度了，需要 Irradiance Probe、球谐函数或者 Reflection Capture 等方案。

### 直接光照估计

对于一个表面点 $\mathbf{x}$（法线 $\mathbf{n}$），来自面光源的直接辐照度：

$$
E_{direct}(\mathbf{x}) = \int_{A_L} L_e \cdot \frac{(\omega_i \cdot \mathbf{n}) \cdot (\omega_o \cdot \mathbf{n}_L)}{|\mathbf{x} - \mathbf{y}|^2} \cdot V(\mathbf{x}, \mathbf{y}) \, dA_L
$$

其中：
- $A_L$ 是光源面积
- $\omega_i$ 是从 $\mathbf{x}$ 指向光源点 $\mathbf{y}$ 的单位向量
- $\mathbf{n}_L$ 是光源法线
- $|\mathbf{x} - \mathbf{y}|^2$ 是距离的平方（平方反比衰减）
- $V(\mathbf{x}, \mathbf{y})$ 是可见性函数（0=被遮挡，1=可见）

用蒙特卡洛估计，对光源面积均匀采样：

$$
E_{direct} \approx \frac{A_L}{N} \sum_{i=1}^{N} L_e \cdot \frac{(\omega_i \cdot \mathbf{n}) \cdot |\cos\theta_L|}{|\mathbf{x} - \mathbf{y}_i|^2} \cdot V(\mathbf{x}, \mathbf{y}_i)
$$

其中 $\theta_L$ 是光源面法线与 $\omega_i$ 的夹角。注意这里我用绝对值 $|\cos\theta_L|$，因为面光源是双面发光的。

### 间接光照（一次弹射）

一次间接弹射（One Bounce Indirect）：

1. 从表面点 $\mathbf{x}$ 按余弦加权半球采样一个方向 $\omega$
2. 沿 $\omega$ 追踪 Shadow Ray，找到交点 $\mathbf{x}'$
3. 在 $\mathbf{x}'$ 估计直接辐照度 $E_{direct}(\mathbf{x}')$
4. 计算贡献：$\Delta L = E_{direct}(\mathbf{x}') \cdot \rho(\mathbf{x}') / \pi \cdot (\omega \cdot \mathbf{n}) \cdot \pi$（余弦采样的 PDF 消掉了 $1/\pi$）

这对应路径追踪中的"L(S*)D"路径中的前两段。

### 余弦加权半球采样

半球均匀采样的 PDF 是 $p(\omega) = 1/(2\pi)$，余弦加权采样的 PDF 是 $p(\omega) = \cos\theta/\pi$。

余弦加权采样更高效，因为接近法线方向（$\cos\theta$ 大）的样本贡献更多，这与渲染方程的余弦项完全匹配，消除了重要性比值中的 $\cos\theta$ 因子。

实现：使用 Malley 方法，先在单位圆盘上均匀采样，再投影到半球上：

```cpp
Vec3 cosineHemisphere(const Vec3& N) {
    float u1 = next(), u2 = next();
    // 在单位圆盘上采样（r = sqrt(u1) 保证均匀分布面积）
    float r = std::sqrt(u1);
    float phi = 2.f * PI * u2;
    float lx = r * std::cos(phi);
    float lz = r * std::sin(phi);
    // 投影到半球（py 是余弦加权的结果）
    float ly = std::sqrt(std::max(0.f, 1.f - u1));
    // 构建 ONB（正交归一化基），将局部方向转到世界空间
    Vec3 up = std::abs(N.y) < 0.999f ? Vec3(0,1,0) : Vec3(1,0,0);
    Vec3 T = up.cross(N).normalized();
    Vec3 B = N.cross(T);
    return (T * lx + N * ly + B * lz).normalized();
}
```

这里 `std::sqrt(u1)` 而不是 `u1`，是因为圆盘面积采样需要对半径做非线性变换（反CDF法，保证面积均匀），否则会在圆心堆积更多样本。

### UV Atlas 展开

Lightmap 需要把每个三角面片映射到一张 2D 纹理的独立区域，这个过程叫 **UV Unwrapping**（UV 展开）。

真正的 UV 展开（如 xAtlas、UE4 的 Lightmass）会最小化拉伸、充分利用纹理空间，是一个复杂的优化问题。我们使用一个简化方案：

**规则网格分配（Grid Atlas）**：
- 设场景有 $n$ 个三角形
- 每行放 $\lceil\sqrt{n}\rceil$ 个三角形的 tile
- 每个 tile 大小为 $\frac{1}{\lceil\sqrt{n}\rceil} \times \frac{1}{\lceil\sqrt{n}\rceil}$（留 2 像素 padding 防止 UV 泄漏）
- 三角形顶点 UV：$v_0 \to (u_0, v_0)$，$v_1 \to (u_1, v_0)$，$v_2 \to (u_0, v_1)$

这样每个三角形都有独立的 UV 空间，不会互相干扰。缺点是利用率低（32个三角形放在 6×6 的格子里，最后两行只有 2 个 tile），实际生产中不用这种方案。

---

## 实现架构

### 整体数据流

```
Scene（三角形 + 材质 + 光源）
         |
         ↓  buildAtlas()
UV Atlas Layout（每个三角形分配纹素范围）
         |
         ↓  bakeLightmap()
LightmapAtlas（512×512 辐照度贴图）
  ├── 对每个三角形的每个 texel：
  │     ├── texel (px,py) → UV (u,v) → 重心坐标 (s,t)
  │     ├── 世界坐标 = wA*(1-s-t) + wB*s + wC*t
  │     ├── 直接光照：对面光源采样 + Shadow Ray
  │     └── 间接光照：余弦半球采样 1 次弹射
  └── 累积 irradiance，记录 sampleCnt
         |
         ↓  renderFinalImage()
输出图像（Cornell Box 视角，用 lightmap 替代实时光照）
```

### 关键数据结构

```cpp
// 三角形：包含几何数据和 UV Atlas 坐标
struct Triangle {
    Vec3 v[3];       // 世界空间顶点
    Vec3 normal;     // 面法线
    int matIdx;      // 材质索引
    int triId;       // 场景中的索引（用于 lightmap lookup）
    float lmu[3], lmv[3];  // 每个顶点在 atlas 中的 UV 坐标
};

// 光照贴图 Atlas：存储每个 texel 的累积辐照度
struct LightmapAtlas {
    std::vector<Vec3> data;      // 辐照度值（线性光照空间）
    std::vector<int>  sampleCnt; // 采样计数（用于均值计算）
    // 支持双线性插值采样，unfilled texel 返回 (0,0,0)
};
```

**为什么需要 sampleCnt？** 因为不同位置的 texel 可能被多次访问（来自不同的采样点），需要对所有采样取均值，而不是累加。

### 管线模块职责

| 模块 | 职责 | 输入 | 输出 |
|------|------|------|------|
| `buildScene()` | 构建 Cornell Box 场景 | - | `Scene`（三角形、材质、光源列表）|
| `buildAtlas()` | 分配 UV 坐标 | `Scene` | 修改三角形的 `lmu/lmv` 字段 |
| `bakeLightmap()` | 离线烘焙辐照度 | `Scene` | `LightmapAtlas`（512×512） |
| `renderFinalImage()` | 实时渲染 + lightmap 采样 | `Scene` + `Atlas` | 输出图像 |
| `renderAtlasViz()` | 可视化 Atlas | `Atlas` | Atlas 图像（调试用） |

### Cornell Box 场景构造

场景是经典 Cornell Box：红色左墙、绿色右墙、白色地板/天花板/后墙、顶部面光源，加上两个盒子（高蓝盒 + 矮黄盒）。

```cpp
// 以 B=1 为半尺寸，构造 [-1,1]^3 的 Cornell Box
float B = 1.0f;
addQuad(scene, Vec3(-B,-B,-B), Vec3(B,-B,-B), Vec3(B,-B,B), Vec3(-B,-B,B), 0); // 地板（白）
addQuad(scene, Vec3(-B,B,-B),  Vec3(-B,B,B),  Vec3(B,B,B),  Vec3(B,B,-B),  0); // 天花板（白）
addQuad(scene, Vec3(-B,-B,B),  Vec3(B,-B,B),  Vec3(B,B,B),  Vec3(-B,B,B),  0); // 后墙（白）
addQuad(scene, Vec3(-B,-B,-B), Vec3(-B,-B,B), Vec3(-B,B,B), Vec3(-B,B,-B), 1); // 左墙（红）
addQuad(scene, Vec3(B,-B,-B),  Vec3(B,B,-B),  Vec3(B,B,B),  Vec3(B,-B,B),  2); // 右墙（绿）
// 面光源（天花板中央，双三角形，朝下发光）
float ls = 0.32f, lh = 0.98f;
addQuad(scene, Vec3(-ls,lh,-ls), Vec3(-ls,lh,ls), Vec3(ls,lh,ls), Vec3(ls,lh,-ls), 3);
```

注意面光源的法线：由顶点绕序 `(-ls,lh,-ls) → (-ls,lh,ls) → (ls,lh,ls)` 决定，计算得到法线为 `(0,+1,0)`（朝上）。但发光方向应该是向下的。

这里我使用 `abs(lt.normal.dot(toLightN))` 处理，让面光源双面发光，避免因为法线朝向问题导致某些接收点计算出负值贡献。

---

## 关键代码解析

### UV Atlas 展开实现

```cpp
void buildAtlas(Scene& scene) {
    int n = (int)scene.tris.size();
    // 每行多少个 tile：ceil(sqrt(n))
    int tilesPerRow = (int)std::ceil(std::sqrt((float)n));
    float tileSize = 1.f / tilesPerRow;
    float pad = 2.f / LM_SIZE;  // 2像素的内边距，防止相邻 tile 的 texel 互相泄漏

    for (int i = 0; i < n; i++) {
        int col = i % tilesPerRow;
        int row = i / tilesPerRow;
        float u0 = col * tileSize + pad;
        float v0 = row * tileSize + pad;
        float u1 = (col + 1) * tileSize - pad;
        float v1 = (row + 1) * tileSize - pad;
        // 三角形三个顶点的 UV：v0→(u0,v0), v1→(u1,v0), v2→(u0,v1)
        // 这构成了一个直角三角形占据了 tile 左下角的三角区域
        scene.tris[i].lmu[0] = u0; scene.tris[i].lmv[0] = v0;
        scene.tris[i].lmu[1] = u1; scene.tris[i].lmv[1] = v0;
        scene.tris[i].lmu[2] = u0; scene.tris[i].lmv[2] = v1;
    }
}
```

`pad = 2.f / LM_SIZE` 是为了避免"UV 泄漏"（UV Bleeding）——如果 tile 之间没有间隔，双线性插值可能采到相邻 tile 的 texel，导致光照从一个面"渗入"另一个面。

### Texel 中心到世界坐标的映射

这是 baking 的关键步骤：把纹素坐标反向映射回世界空间的表面点。

```cpp
for (int py = py0; py <= py1; py++) {
    for (int px = px0; px <= px1; px++) {
        // 1. 纹素中心在 UV 空间的坐标（+0.5 取 texel 中心）
        float u = (px + 0.5f) / LM_SIZE;
        float v = (py + 0.5f) / LM_SIZE;

        // 2. UV → 重心坐标 (s, t)
        //    三角形 UV 布局: v0=(u0,v0), v1=(u1,v0), v2=(u0,v1)
        //    所以 s = (u - u0)/(u1-u0) 表示沿 v0→v1 方向的权重
        //         t = (v - v0)/(v2-v0) 表示沿 v0→v2 方向的权重
        float s = (u - u0) / du;
        float t = (v - v0) / dv;

        // 3. 边界处理：如果 s+t>1 则超出了三角形范围
        //    镜像翻转保持在三角形内（而不是直接 skip）
        s = std::max(0.f, std::min(1.f, s));
        t = std::max(0.f, std::min(1.f, t));
        if (s + t > 1.f) {
            s = 1.f - s;  // 镜像，等价于使用三角形的另一半
            t = 1.f - t;
            if (s < 0.f) s = 0.f;
            if (t < 0.f) t = 0.f;
        }

        // 4. 重心坐标 → 世界坐标
        Vec3 worldPos = wA * (1.f - s - t) + wB * s + wC * t;
```

为什么需要镜像处理？tile 里有些 texel 在三角形外面（右上角区域，因为三角形只占了正方形的左下半）。如果直接跳过，这些 texel 就是黑的，边缘处会有锯齿。镜像处理把它们映射到三角形内部，虽然不完全精确，但对视觉效果影响很小。

### 法线朝向修正

这是今天遇到的最大 Bug，专门提出来讲解：

```cpp
// 确保法线朝向房间内部（面向光源的那侧）
// 利用场景原点 (0,0,0) 在房间中央这一特性来判断
Vec3 centroid = (wA + wB + wC) * (1.f/3.f);
Vec3 toCenter = Vec3(0,0,0) - centroid;
if (N.dot(toCenter) < 0.f) N = -N;  // 翻转朝向内部
```

这段代码的逻辑：如果三角形面心到场景中心的向量与法线方向相反，说明法线朝外（背离房间内部），需要翻转。

### 直接光照采样

```cpp
Vec3 directLight(const Scene& scene, const Vec3& pos, const Vec3& N, RNG& rng) {
    Vec3 Lo(0,0,0);
    for (int li : scene.lightTriIds) {
        const Triangle& lt = scene.tris[li];
        const Material& lm = scene.mats[lt.matIdx];

        // 在光源三角形上均匀采样一个点
        // 用重心坐标：(1-r1-r2)*v0 + r1*v1 + r2*v2
        // 注意：直接用 r1,r2 ~ Uniform(0,1) 会导致在平行四边形（而非三角形）上采样
        // 正确方法：如果 r1+r2 > 1，折叠到三角形内（等效于镜像对称）
        float r1 = rng.next(), r2 = rng.next();
        if (r1 + r2 > 1.f) { r1 = 1.f - r1; r2 = 1.f - r2; }
        float r3 = 1.f - r1 - r2;
        Vec3 lpos = lt.v[0] * r3 + lt.v[1] * r1 + lt.v[2] * r2;

        // 几何项计算
        Vec3 toLight = lpos - pos;
        float dist2 = toLight.dot(toLight);
        float dist = std::sqrt(dist2);
        Vec3 toLightN = toLight / dist;

        float cosTheta = N.dot(toLightN);          // 接收点余弦（≥0 才有贡献）
        if (cosTheta <= 0.f) continue;

        // 面光源双面发光（abs 处理法线朝向问题）
        float cosThetaL = std::abs(lt.normal.dot(toLightN));
        if (cosThetaL <= 0.f) continue;

        // Shadow Ray：从接收点射向光源，判断是否被遮挡
        // 两端各留 2e-3 的偏移，避免自相交（self-intersection）
        if (scene.shadowBlocked(pos + N * 2e-3f, lpos - lt.normal * 2e-3f)) continue;

        // 计算面积（用于把面积采样的 PDF 转换回立体角）
        float area = scene.lightArea(li);

        // 渲染方程离散化：Le * cos(theta_recv) * |cos(theta_emit)| * A / (pi * r^2)
        // 其中 1/pi 来自 Lambertian BRDF，A/r^2 把面积采样转换为立体角
        Lo += lm.emission * (cosTheta * cosThetaL * area / (PI * dist2));
    }
    return Lo;
}
```

**关键细节：`lpos - lt.normal * 2e-3f`** 为什么 Shadow Ray 的终点要减去法线偏移？因为直接去光源中心点会与光源三角形自相交（光源自遮挡），需要稍微偏移到光源面外侧。

### Lightmap 双线性采样

渲染时，根据命中三角形的重心坐标计算 atlas UV，然后双线性采样：

```cpp
// 在 renderFinalImage 中
const Triangle& tri = scene.tris[rec.triId];
float bu = rec.bu, bv = rec.bv;
float bw = 1.f - bu - bv;

// 用重心坐标插值三个顶点的 UV
float u = tri.lmu[0]*bw + tri.lmu[1]*bu + tri.lmu[2]*bv;
float v  = tri.lmv[0]*bw + tri.lmv[1]*bu + tri.lmv[2]*bv;

Vec3 irr = atlas.sample(u, v);  // 双线性插值
```

双线性插值的实现：

```cpp
Vec3 sample(float u, float v) const {
    // 把 [0,1] UV 转换为浮点像素坐标
    float fx = u * LM_SIZE - 0.5f;
    float fy = v * LM_SIZE - 0.5f;
    int x0 = (int)std::floor(fx), y0 = (int)std::floor(fy);
    float tx = fx - x0, ty = fy - y0;  // 小数部分 = 插值权重

    // 四个邻近 texel 的值
    Vec3 c00 = get(clamp(x0,   ...), clamp(y0,   ...));
    Vec3 c10 = get(clamp(x0+1, ...), clamp(y0,   ...));
    Vec3 c01 = get(clamp(x0,   ...), clamp(y0+1, ...));
    Vec3 c11 = get(clamp(x0+1, ...), clamp(y0+1, ...));

    // 双线性插值：先沿 X 插值，再沿 Y 插值
    Vec3 c0 = c00 + (c10 - c00) * tx;  // X 方向
    Vec3 c1 = c01 + (c11 - c01) * tx;
    return c0 + (c1 - c0) * ty;         // Y 方向
}
```

### Tone Mapping

辐照度值是线性的高动态范围（HDR），直接截断会过曝。使用 Reinhard Tone Mapping：

```cpp
Vec3 toneMap(Vec3 c) {
    // Reinhard: x/(1+x) 把 [0,∞) 压缩到 [0,1)
    c.x = c.x / (1.f + c.x);
    c.y = c.y / (1.f + c.y);
    c.z = c.z / (1.f + c.z);
    // Gamma 2.2 矫正（线性光照 → sRGB 显示）
    c.x = std::pow(std::max(0.f, c.x), 1.f/2.2f);
    c.y = std::pow(std::max(0.f, c.y), 1.f/2.2f);
    c.z = std::pow(std::max(0.f, c.z), 1.f/2.2f);
    return c.clamp01();
}
```

---

## 踩坑实录

### Bug 1：地板全黑，法线朝向错误

**症状**：渲染出的 Cornell Box 中，地板、天花板、大部分墙面几乎全黑，只有高蓝盒的部分面片有光照。Atlas 可视化里第 0、1 行的 tile（对应地板和天花板）全是黑色。

**错误假设**：以为 `estimateIrradiance` 返回了接近零的值是正常的，可能是采样数太少。

**真实原因**：`addQuad` 函数按照右手螺旋定则计算法线，但 Cornell Box 的地板 `addQuad((-1,-1,-1), (1,-1,-1), (1,-1,1), (-1,-1,1))` 生成的法线是 `(0,-1,0)`（朝下）。

计算过程：
- e1 = (1,-1,-1) - (-1,-1,-1) = (2,0,0)
- e2 = (1,-1,1) - (-1,-1,-1) = (2,0,2)
- n = e1 × e2 = (0*2-0*2, 0*2-2*2, 2*0-0*2) = (0,-4,0) → 归一化 (0,-1,0)

所以当光从上方来时，`cosTheta = N·toLightN = (0,-1,0)·(0,1,0) = -1`，被 `max(0,...)` 截断成 0，整个地板的直接光照变成了 0。

**修复方式**：在 baking 循环开始时，根据三角形面心到场景中心的向量来判断法线是否朝向内部，如果朝外则翻转：

```cpp
Vec3 centroid = (wA + wB + wC) * (1.f/3.f);
Vec3 toCenter = Vec3(0,0,0) - centroid;
if (N.dot(toCenter) < 0.f) N = -N;
```

这个方法适用于凸场景（如 Cornell Box），利用了"所有面都应该朝向中心"的假设。对于复杂凹形场景则需要其他方案（如手动指定法线方向，或从顶点顺序约定法线）。

**经验**：在编写场景时，务必明确 `addQuad` 的顶点绕序约定，并且先验证法线方向（打印或可视化法线向量）再开始 baking。

### Bug 2：面光源 `cosThetaL` 为负

**症状**：修复 Bug 1 之后，场景有了基本光照，但高蓝盒的顶面和右上角的一些地方依然明显偏暗，不符合预期。

**错误假设**：以为是间接光照采样数不够，尝试增加采样数也无效。

**真实原因**：面光源的法线也是朝上 `(0,1,0)`（因为 addQuad 的顶点绕序），而光应该朝下发射。在代码中：

```cpp
float cosThetaL = (-lt.normal).dot(toLightN);  // 错误！
```

当接收点在光源正下方时，`toLightN = (0,-1,0)`（从接收点看向光源是向上的），`-lt.normal = (0,-1,0)`，dot 结果 = `(-1,0)*(-1) + 0 + 0 = 1`... 

等等，让我重算：`toLightN` 是从接收点 `pos` 指向光源点 `lpos` 的方向。`pos` 在地板（y=-1），`lpos` 在天花板（y=+1），所以 `toLightN = (0,+1,0)`（向上）。而 `-lt.normal = (0,-1,0)`，所以 `(-lt.normal)·toLightN = (0,-1,0)·(0,1,0) = -1`，被截断为 0！

**修复**：用绝对值让面光源双面发光：

```cpp
float cosThetaL = std::abs(lt.normal.dot(toLightN));
```

物理上，一个理想的面光源可以双面发光（就像 Blender 里的 Area Light 默认设置），所以用绝对值是合理的。

**经验**：面光源的法线方向是隐性约定。如果构建时没有统一绕序，应该用绝对值或 `std::max(0, dot)` 来避免符号问题。最好的做法是场景构建时就统一"法线朝向发光侧"，并加断言检查。

### Bug 3：图像大部分为黑色背景

**症状**：输出图像均值只有 2-4，远低于标准的 10-240。但 Atlas 里确实有亮的区域（某些盒子面片有光照）。

**错误假设**：以为是 Lightmap 数据不对。

**真实原因**：相机 FOV 太小（`fov=0.45`），Cornell Box 在图像中只占很小一块，大量像素是背景黑色（未命中任何三角形）。增大 FOV 后 Box 覆盖了更大区域，均值从 2.0 升到了 65.6。

另外：上述两个 Bug 组合导致真正亮的表面（地板、墙面）都是黑色，只有高蓝盒的几个面有正确光照，而盒子只占很小面积。

**修复**：`camPos=(0,0,-1.85)`, `fov=0.85`，让 Cornell Box 几乎填满画面。

**经验**：像素统计验证（均值 10-240）是个非常有用的早期预警。均值过低往往有两种原因：(1) 渲染结果错误（全黑），(2) 场景覆盖率不足（FOV 太小、相机位置不对）。两种情况的修复方式完全不同，需要通过分析亮像素数量和分布来区分。

---

## 效果验证与数据

### 编译与运行

```bash
g++ main.cpp -o lightmap_baker -std=c++17 -O2 -Wall -Wextra
# EXIT:0，0 errors，0 warnings（自己的代码）
./lightmap_baker
```

输出：
```
=== Lightmap Baker ===
Triangles: 32  Lights: 2
Building UV atlas...
Baking lightmap (512x512)...
Baking complete.
Filled texels: 208083 / 262144 (79.4%)
Rendering final scene (512x512)...
Saved: lightmap_baker_output.png
Rendering lightmap atlas visualization...
Saved: lightmap_atlas_output.png
Done!

real    0m3.174s
```

性能指标：
- 场景：32 个三角形，2 个面光源三角形
- 辐照度采样：6 samples/texel × 4 间接光 bounce = 最多 24 条光线/texel
- 填充率：79.4%（208083/262144），未填充的是 tile padding 和空白区域
- 烘焙时间：3.17 秒（单线程 CPU）
- 渲染时间：含于 3.17 秒内（最后的 render 步骤很快）

### 像素统计验证

```python
from PIL import Image
import numpy as np

img = Image.open('lightmap_baker_output.png')
pixels = np.array(img).astype(float)
print(f"均值: {pixels.mean():.1f}  标准差: {pixels.std():.1f}")
print(f"最大值: {pixels.max():.0f}  文件大小: {os.path.getsize(...)//1024}KB")
```

输出：
```
均值: 65.6  标准差: 53.8
最大值: 235  文件大小: 376KB
✅ 文件大小 > 10KB
✅ 像素均值在 10~240 之间（非全黑/全白）
✅ 像素标准差 > 5（图像有内容变化）
```

### 输出效果

**主渲染图（Cornell Box + Lightmap）**：

![Lightmap Baker 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-25-lightmap-baker/lightmap_baker_output.png)

左图是用 Lightmap 渲染的 Cornell Box，可以看到：
- 地板上有柔和的直接光照区域（光源正下方最亮）
- 红色左墙和绿色右墙的颜色溢色到相邻表面（间接光）
- 蓝色高盒和黄色矮盒都有正确的光照和阴影
- 盒子投射的软阴影（边缘渐变，因为面光源有面积）

**Lightmap Atlas 可视化**：

![Lightmap Atlas](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05-25-lightmap-baker/lightmap_atlas_output.png)

Atlas 图像中可以看到：
- 每个 tile 对应一个三角面片
- 靠近光源的面片（如天花板附近）最亮
- 角落处的面片因为受间接光较少而较暗
- 未填充的区域（tile 之间的 padding 和空行）呈黑色

---

## 总结与延伸

### 实现总结

今天实现了一个完整的 Lightmap Baker，覆盖了光照贴图烘焙的核心流程：

1. **UV Atlas 展开**：简单但可用的网格分配方案，为每个三角形分配独立的 UV 空间
2. **Texel 到世界坐标的映射**：通过重心坐标插值，精确还原表面点位置
3. **直接光照（Area Light Sampling）**：蒙特卡洛面积采样 + Shadow Ray 可见性检测
4. **间接光照（One Bounce）**：余弦加权半球采样 + 在弹射点递归计算直接光
5. **Lightmap 采样**：双线性插值避免块状走样

修复了三个关键 Bug：法线朝向错误、面光源 cosThetaL 符号问题、FOV 设置不合理。

### 技术局限性

1. **只支持静态物体**：Lightmap 预计算后场景就固定了，灯光或物体移动后需要重新烘焙
2. **低频光照**：Lightmap 分辨率有限，无法捕捉高频的阴影细节（如细小的遮挡）
3. **UV 展开简陋**：当前的网格分配浪费空间，实际引擎用 xAtlas 或 UVAuto 等工具做优化展开
4. **只有一次间接弹射**：真正的 GI 需要多次弹射。一次弹射已经能表现基本的颜色溢色，但二次、三次弹射才能让暗角更自然
5. **噪点问题**：6 samples/texel 不足以收敛，需要更多采样或去噪（OIDN 等）

### 可优化方向

- **多线程并行**：每个 texel 的 baking 完全独立，用 `std::thread` 或 OpenMP 可以轻松并行化，速度提升 4-8 倍
- **更好的 UV 展开**：集成 xAtlas 库，减少 UV 浪费
- **Light Tree / BLAS**：当场景有很多光源时，构建光源 BVH 加速采样决策
- **降噪**：使用 Intel Open Image Denoise（OIDN）对烘焙结果进行 AI 降噪，可以大幅降低所需采样数
- **多次弹射**：实现完整的路径追踪 baking（Lightmass 就是这么做的）
- **Lightmap Dilate**：对 UV seam 处进行 texel 扩展（dilate），避免边界漏光
- **Probe 辅助**：对于大场景，单张全局 Lightmap 分辨率不够，可结合 Irradiance Probe

### 与本系列的关联

- **05-24 Subsurface Scattering**：同样涉及光照传播，SSS 可以视为 Lightmap Baker 的材质扩展（非 Lambertian）
- **05-22 SSGI**：实时版本的间接光照，和 Lightmap 是互补的——Lightmap 离线烘焙静态光，SSGI 实时处理动态光
- **05-19 Atmospheric Scattering**：Lightmap 可以与大气散射结合，烘焙室外场景的天空光照
- **03-21 SPPM / 03-22 BDPT**：更高级的离线渲染算法，Lightmass 内部就使用了类似 BDPT 的双向路径追踪

离线烘焙技术在游戏引擎中经历了几十年的发展，从 Quake 的 BSP + 辐射度方法，到 Lightmass 的双向路径追踪，核心思想没变：用时间换取质量，把复杂光照预先计算好，让运行时可以"免费"享用。
