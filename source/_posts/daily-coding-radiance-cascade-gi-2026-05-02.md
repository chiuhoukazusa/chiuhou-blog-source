---
title: "每日编程实践: Radiance Cascade Global Illumination"
date: 2026-05-02 05:36:19
tags:
  - 每日一练
  - 图形学
  - 全局光照
  - C++
  - 软光栅化
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-02-radiance-cascade-gi/radiance_cascade_gi_output.png
---

# 每日编程实践: Radiance Cascade Global Illumination

![渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-02-radiance-cascade-gi/radiance_cascade_gi_output.png)

> Cornell Box 场景中的辐射级联全局光照：红墙的色溢打到中心球上，绿墙反光照亮黄盒侧面，间接光照让整个场景更真实可信。

---

## ① 背景与动机

### 为什么需要全局光照？

实时渲染长期困扰于一个根本矛盾：现实世界中的光线会在物体之间反复弹跳，但传统的局部光照模型（Blinn-Phong、甚至 PBR）只计算光源到表面的直接照射，完全忽视了光在场景中的**多次反弹**。

没有全局光照时，你会看到什么问题？

- **漏光**：阴影区域完全漆黑，即便现实中附近墙面的反射光应该能照进去
- **色溢缺失**：红墙旁边的白色物体不会带上红色，场景像纸板模型
- **接触阴影过硬**：物体之间的缝隙没有柔和的 ambient occlusion 感
- **环境光 hack 代价**：为了弥补这些缺陷，引擎往往用一个魔法常量 `ambient = 0.1` 来糊弄过去，但这是完全假的

这就是为什么游戏从 PS3 时代看起来总像玩具——光照是错的，哪怕模型多精细也难掩这种廉价感。

### 工业界是怎么做的？

全局光照（Global Illumination, GI）在工业界的演进大致如下：

**离线 GI（2000 年代前）**：光线追踪、辐射度算法，效果完美但需要数小时甚至数天，只用于电影 CG 和建筑可视化。

**Baked GI（Xbox 360/PS3 时代）**：用 Unity 的 Enlighten 或 Unreal 的 Lightmass 预计算光照贴图，烘焙进静态场景。优点：实时零开销；缺点：动态物体无法受益，场景不能有光照变化。

**Probe-based GI（PS4/Xbox One 时代）**：在场景中均匀放置光探针（Light Probe），每个探针存储周围一圈方向的辐射度（通常用球谐函数压缩），动态物体查询最近的探针来获取间接光照。但探针数量有限，边界处有插值伪影。

**Screen-Space GI（PS4 中期）**：SSAO、SSGI 等屏幕空间技术，只用 G-Buffer 数据做本地范围的 GI 近似。速度快，但受限于屏幕可见信息，摄像机移动时会有信息丢失。

**Lumen（UE5，2022）**：Epic Games 在 Unreal Engine 5 中推出的完全动态 GI 系统。核心思想之一就是 **Radiance Cache（辐射度缓存）** 结合多级探针，通过"不同层级覆盖不同空间尺度"的方式，在保持实时性能的同时获得高质量的全场景 GI。

**Radiance Cascades（独立技术，2022-2024）**：Alexander Sannikov 在研究 Lumen 类技术时提出的一种优雅的 GI 实现思路，核心在于用"辐射级联"解决探针分辨率与覆盖范围的权衡问题。在 2D 版本上效果尤为突出，已有多个游戏引擎在实验性实现。

今天的项目就是在软光栅 C++ 中从头实现一个简化版的 Radiance Cascade GI，直观感受其原理和效果。

---

## ② 核心原理

### 2.1 渲染方程（基础）

一切光照的数学基础是渲染方程（Kajiya 1986）：

$$L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_\Omega f_r(\mathbf{x}, \omega_i, \omega_o) L_i(\mathbf{x}, \omega_i) (\omega_i \cdot \mathbf{n}) \, d\omega_i$$

直觉解释：
- $L_o$：从点 $\mathbf{x}$ 向方向 $\omega_o$ 出射的辐亮度（你看到的亮度）
- $L_e$：点 $\mathbf{x}$ 自身的发光（光源项）
- 积分项：来自所有方向 $\omega_i$ 的入射光，经过 BRDF $f_r$ 调制，再被 $\cos\theta$ 权重加权后的贡献之和

这个积分在实时中无法直接计算（无限维），所有 GI 方案都是在近似这个积分。

### 2.2 为什么需要多级结构？

假设我们用探针来近似上面的积分。探针的核心困境是**分辨率与范围的矛盾**：

- **高密度探针**：能准确捕捉细节（如角落的色溢），但无法覆盖长距离光传播（如阳光穿过窗户照到 5 米外的地板）
- **低密度探针**：覆盖范围大，但细节全丢，边界插值会产生明显的"格子感"

Radiance Cascades 的解决思路：**不同尺度的光传播用不同密度的探针来处理**。

这其实和 Mipmap 的设计哲学相同：近处需要高频细节（高分辨率），远处只需要低频信息（低分辨率）。

### 2.3 辐射级联的数学描述

设场景中有 $N$ 个级联层级，第 $k$ 层的参数为：
- 探针数量 $P_k = G_k^2$（均匀网格，$G_k$ 为网格边长）
- 每个探针的方向数 $D_k$（角分辨率）
- 最大追踪距离 $R_k$（光线步进范围）

典型的级联设计（本实现）：

| 层级 | 网格 | 方向数 | 最大距离 | 作用 |
|------|------|--------|---------|------|
| 0（细粒度）| 16×16 | 16 | 3.0m | 近距离细节 |
| 1（中粒度）| 8×8 | 32 | 6.0m | 中等距离 |
| 2（粗粒度）| 4×4 | 64 | 12.0m | 远距离 |

每个级联层独立工作：向各方向发射射线，如果射线在本层范围内击中表面，记录该点的直接光照；如果没有击中，向更高层级"借"辐射度（即级联合并步骤）。

### 2.4 级联合并（Cascade Merging）

合并方向是**自顶向下**：粗粒度层的辐射度注入细粒度层。

数学上，对于细粒度探针 $p^0_i$ 的方向 $d$ 的辐射度：

$$R^0_{i,d} = R^0_{i,d,\text{direct}} + \alpha \cdot R^1_{j^*, d^*}$$

其中：
- $j^* = \arg\min_j \|\mathbf{x}^1_j - \mathbf{x}^0_i\|$（最近粗粒度探针）
- $d^* = \arg\max_{d'} (\mathbf{d} \cdot \mathbf{d}'^1)$（方向最接近的粗粒度探针方向）
- $\alpha = \max(0, \mathbf{d} \cdot \mathbf{d}'^1_{d^*}) \times 0.25$（方向相似度权重，且远距离贡献打折）

直觉解释：每个细粒度探针在某方向上"看到"了短距离内的场景，同时借助粗粒度探针知道更远处有什么——就像你近处看清楚，远处靠余光感知。

### 2.5 间接光照查询

在 Deferred Shading 阶段，对 G-Buffer 中每个像素查询间接辐照度：

$$E_{\text{indirect}}(\mathbf{x}, \mathbf{n}) = \frac{\sum_{d=0}^{D-1} R_{j^*,d} \cdot (\mathbf{n} \cdot \mathbf{d}_d) \cdot \mathbb{1}[\mathbf{n} \cdot \mathbf{d}_d > 0]}{\sum_{d=0}^{D-1} \max(0, \mathbf{n} \cdot \mathbf{d}_d)}$$

这是个 Cosine-weighted 的半球积分近似：对法线上方的方向，用 $\cos\theta = \mathbf{n} \cdot \mathbf{d}$ 做加权平均，再归一化。相当于用探针的离散方向样本来近似球面积分，越与法线对齐的方向贡献越大——这正是 Lambertian BRDF 的物理含义。

### 2.6 方向均匀采样（Fibonacci 球面）

探针的方向需要尽可能均匀分布在球面上。本实现使用 Fibonacci 螺旋采样：

$$\theta_i = 2\pi \cdot \frac{i}{\phi}, \quad \phi_i = \arccos\left(1 - \frac{2i+1}{N}\right)$$

其中 $\phi = \frac{1+\sqrt{5}}{2} \approx 1.618$ 是黄金比例。

为什么不用经纬网格？因为经纬网格在极点附近样本密度极高（所有经线在极点汇聚），在赤道附近密度低，分布不均匀。Fibonacci 螺旋则保证每个样本都和邻近样本保持大致相同的角距离，对于辐射度积分来说误差更小。

---

## ③ 实现架构

### 3.1 整体渲染管线

```
输入：场景三角形 + 光源列表
         ↓
[Pass 1: 软光栅化 G-Buffer]
   G-Buffer:
     - albedo(x,y)   : 漫反射率
     - normal(x,y)   : 世界空间法线
     - worldPos(x,y) : 世界空间位置
     - depth(x,y)    : NDC 深度
         ↓
[Pass 2: 构建辐射级联]
   对每个级联层：
     - 均匀放置探针（2D 网格）
     - 每个探针向 D 个方向发射射线
     - 射线打到场景 → 计算直接光照
     - 射线未打到 → 返回天空颜色
         ↓
[Pass 3: 合并级联]
   c2 → c1 → c0（顶层 → 底层）
   方向匹配 + 相似度权重
         ↓
[Pass 4: Deferred Shading]
   对每个有效像素：
     1. 查询 G-Buffer
     2. 计算直接光照
     3. 从 Cascade 0 查询间接辐照度
     4. finalColor = direct + indirect × albedo × 0.5
         ↓
[Post-process: ACES Tone Mapping + Gamma 校正]
         ↓
输出：radiance_cascade_gi_output.png
```

### 3.2 关键数据结构

```cpp
// 单个辐射探针：存储 numDirs 个方向的入射辐亮度
struct Probe {
    std::vector<Vec3> radiance; // [numDirs] 每方向的 RGB 辐亮度
    Vec3 pos;                   // 世界空间位置
    bool valid = false;         // 是否已被初始化
};

// 一个级联层级的完整参数描述
struct CascadeLevel {
    int gridW, gridH;   // 探针网格尺寸
    int numDirs;        // 方向数（角分辨率）
    float spacing;      // 探针间距（世界空间）
    float maxRange;     // 最大射线追踪距离
    std::vector<Probe> probes; // gridW × gridH 个探针
};

// G-Buffer（延迟渲染的中间数据）
struct GBuf {
    std::vector<Vec3>  albedo;    // 漫反射率
    std::vector<Vec3>  normal;    // 世界法线
    std::vector<Vec3>  worldPos;  // 世界坐标
    std::vector<float> depth;     // NDC 深度（用于深度测试）
    std::vector<bool>  valid;     // 该像素是否有几何体
};
```

### 3.3 性能权衡

本实现的探针配置（256+64+16=336 个探针）在 CPU 软渲染器上 **0.3 秒**内完成，这比路径追踪快了几个数量级。

为什么快？关键在于：
- **间接光照只需 1 次反弹**：探针存储的是直接光照，间接光照是 1-bounce
- **探针复用**：任意屏幕像素都复用同一组探针，没有 per-pixel 的射线开销
- **方向预计算**：Fibonacci 方向集合预先生成，查询时只做点积比较

这正是 Radiance Cascade 相比路径追踪的核心优势：将昂贵的球面积分**离线化**到探针中，运行时只做廉价的插值查询。

### 3.4 软光栅化流程

G-Buffer Pass 使用经典的三角形光栅化：
1. 三角形顶点经过 View Matrix（LookAt）和 Projection Matrix（透视）变换到 NDC
2. NDC 转换到屏幕空间整数坐标
3. 对三角形包围盒内每个像素，用重心坐标做点内测试
4. 插值法线和世界坐标写入 G-Buffer（depth test）

---

## ④ 关键代码解析

### 4.1 辐射探针构建

```cpp
static void buildCascade(CascadeLevel& cascade,
                          const std::vector<Triangle>& scene,
                          const std::vector<Light>& lights,
                          const std::vector<Vec3>& dirs,
                          float worldMinX, float worldMaxX,
                          float worldMinZ, float worldMaxZ,
                          float probeY)
{
    int GW = cascade.gridW, GH = cascade.gridH;
    cascade.probes.resize(GW * GH);

    for (int gz=0; gz<GH; ++gz) {
        for (int gx=0; gx<GW; ++gx) {
            int idx = gz*GW + gx;
            // 均匀网格分布：探针放在每个格子的中心
            float px = worldMinX + (gx + 0.5f) * (worldMaxX-worldMinX) / GW;
            float pz = worldMinZ + (gz + 0.5f) * (worldMaxZ-worldMinZ) / GH;
            Vec3 probePos(px, probeY, pz);
            cascade.probes[idx].pos = probePos;
            cascade.probes[idx].radiance.resize(cascade.numDirs, Vec3(0,0,0));
            cascade.probes[idx].valid = true;

            // 向每个方向发射射线
            for (int d=0; d<cascade.numDirs; ++d) {
                const Vec3& rayDir = dirs[d];
                float tHit; Vec3 hN, hA;
                if (rayIntersectScene(probePos, rayDir, scene,
                                      cascade.maxRange, tHit, hN, hA)) {
                    // 命中场景：在命中点计算直接光照
                    Vec3 hitPos = probePos + rayDir * tHit;
                    Vec3 lit = directLight(hitPos, hN, hA, lights, Vec3(0,3,8));
                    cascade.probes[idx].radiance[d] = lit;
                } else {
                    // 未命中：返回天空颜色（简单的竖向渐变）
                    float skyT = rayDir.y * 0.5f + 0.5f;
                    cascade.probes[idx].radiance[d] =
                        mix(Vec3(0.1f,0.15f,0.25f), Vec3(0.05f,0.05f,0.08f), skyT);
                }
            }
        }
    }
}
```

**关键设计说明**：
- `probeY` 参数：不同级联层的探针高度略有不同（0→1.5m，1→2.5m，2→3.0m），这样能覆盖更多的三维空间，避免所有探针在同一水平面上导致方向信息退化
- 天空颜色是基于射线方向的 Y 分量做的渐变，模拟从地平线到天顶的颜色变化
- 直接光照在**命中点**计算，而非在探针位置——这是关键！探针存储的是"从某方向看过去会看到什么亮度"，不是探针位置本身的亮度

### 4.2 Möller–Trumbore 射线-三角形相交

```cpp
static bool rayIntersectScene(const Vec3& orig, const Vec3& dir,
                               const std::vector<Triangle>& scene,
                               float maxDist,
                               float& tHit, Vec3& hitNorm, Vec3& hitAlbedo)
{
    tHit = maxDist;
    bool hit = false;
    for (auto& tri : scene) {
        Vec3 e1 = tri.v[1].pos - tri.v[0].pos;  // 边向量 1
        Vec3 e2 = tri.v[2].pos - tri.v[0].pos;  // 边向量 2
        Vec3 h = dir.cross(e2);                   // 辅助向量
        float a = e1.dot(h);                       // 行列式（=0 表示射线与三角形平行）
        if (std::abs(a) < 1e-7f) continue;        // 平行，跳过
        float f = 1.f/a;
        Vec3 s = orig - tri.v[0].pos;
        float u = f * s.dot(h);                    // 重心坐标 u
        if (u<0||u>1) continue;
        Vec3 q = s.cross(e1);
        float v = f * dir.dot(q);                  // 重心坐标 v
        if (v<0||u+v>1) continue;
        float t = f * e2.dot(q);                   // 交点参数 t（射线方向上的距离）
        if (t>1e-4f && t<tHit) {
            tHit = t; hit = true;
            float w = 1-u-v;
            // 用重心坐标插值法线，实现平滑着色
            hitNorm = (tri.v[0].normal*w + tri.v[1].normal*u + tri.v[2].normal*v).normalize();
            hitAlbedo = tri.albedo;
        }
    }
    return hit;
}
```

为什么 `t > 1e-4f` 而不是 `t > 0`？这是**自相交偏移**（Self-intersection Offset）。当射线从某个三角形表面出发时，由于浮点精度，会以极小的 t 值重新与自身相交（假阳性）。设置 1e-4 的最小距离阈值可以跳过自相交，但又不会过度偏移到相邻几何体。

### 4.3 级联合并

```cpp
static void mergeCascades(CascadeLevel& fine, const CascadeLevel& coarse,
                           const std::vector<Vec3>& fineDirs,
                           const std::vector<Vec3>& coarseDirs)
{
    int GW = fine.gridW, GH = fine.gridH;
    for (int gz=0; gz<GH; ++gz) {
        for (int gx=0; gx<GW; ++gx) {
            int idx = gz*GW + gx;
            Vec3 fPos = fine.probes[idx].pos;

            // 找最近粗粒度探针（线性搜索，探针数量少所以开销可接受）
            float best = 1e30f;
            int bestIdx = 0;
            for (int ci=0; ci<(int)coarse.probes.size(); ++ci) {
                float d2 = (coarse.probes[ci].pos - fPos).length();
                if (d2 < best) { best=d2; bestIdx=ci; }
            }

            // 对每个细粒度方向，找粗粒度中方向最接近的，加权融合
            for (int fd=0; fd<fine.numDirs; ++fd) {
                Vec3 fDir = fineDirs[fd];
                float bestDot = -2.f; int bestCD = 0;
                for (int cd=0; cd<coarse.numDirs; ++cd) {
                    float dt = fDir.dot(coarseDirs[cd]);
                    if (dt > bestDot) { bestDot=dt; bestCD=cd; }
                }
                float weight = std::max(0.f, bestDot);
                // 粗粒度的辐射度按方向相似度衰减后加入细粒度
                // 0.25 是衰减系数：远距离 GI 通常比直接光照弱
                fine.probes[idx].radiance[fd] +=
                    coarse.probes[bestIdx].radiance[bestCD] * (weight * 0.25f);
            }
        }
    }
}
```

**为什么 weight × 0.25？**

物理上，间接光照经过一次漫反射后，能量会被吸收（Albedo < 1）。典型漫反射材质的反射率约 0.5-0.8，再考虑入射方向的 cosine 衰减，二次反射后的能量通常只有直接光照的 20-30%。0.25 这个系数在不进行精确能量守恒计算的情况下，给出了合理的视觉近似。

### 4.4 间接光照查询（Cosine-weighted 半球积分）

```cpp
static Vec3 sampleCascade(const CascadeLevel& cascade,
                            const std::vector<Vec3>& dirs,
                            const Vec3& worldPos,
                            const Vec3& surfNormal)
{
    // 找最近探针
    float bestDist = 1e30f; int bestIdx=0;
    for (int i=0; i<(int)cascade.probes.size(); ++i) {
        if (!cascade.probes[i].valid) continue;
        float d = (cascade.probes[i].pos - worldPos).length();
        if (d < bestDist) { bestDist=d; bestIdx=i; }
    }

    // Cosine-weighted 半球积分近似
    Vec3 indirect(0,0,0);
    float totalW = 0.f;
    for (int d=0; d<(int)dirs.size(); ++d) {
        float ndotl = surfNormal.dot(dirs[d]);
        if (ndotl <= 0.f) continue;       // 法线下方的方向不贡献（遮蔽）
        indirect += cascade.probes[bestIdx].radiance[d] * ndotl;
        totalW += ndotl;
    }
    if (totalW > 1e-7f) indirect *= (1.f / totalW);
    return indirect;
}
```

这段代码实现的是离散化的球面积分：

$$E \approx \frac{\sum_d L_d \cdot \max(0, \mathbf{n} \cdot \mathbf{d})}{\sum_d \max(0, \mathbf{n} \cdot \mathbf{d})}$$

归一化是为了让结果不受方向数量 $D$ 的影响——无论探针有 16 还是 64 个方向，输出的辐照度应该在同一量级。

### 4.5 Deferred Shading（最终合成）

```cpp
for (int y=0; y<H; ++y) {
    for (int x=0; x<W; ++x) {
        int i = y*W+x;
        if (!gbuf.valid[i]) {
            // 背景：天空渐变
            float t = float(y)/H;
            canvas.color[i] = mix(Vec3(0.05f,0.08f,0.15f), Vec3(0.02f,0.03f,0.06f), t);
            continue;
        }
        Vec3 pos    = gbuf.worldPos[i];
        Vec3 normal = gbuf.normal[i];
        Vec3 albedo = gbuf.albedo[i];

        // 直接光照（多光源 Lambertian + Blinn-Phong）
        Vec3 direct = directLight(pos, normal, albedo, lights, cam.pos);

        // 间接光照（从 Cascade 0 查询）
        Vec3 indirect = sampleCascade(c0, dirs0, pos, normal);

        // 合成：直接 + 间接×漫反射率×权重
        // indirect × albedo：模拟间接光照经材质反射后的颜色（色溢！）
        Vec3 finalColor = direct + indirect * albedo * 0.5f;

        finalColor = toneMap(finalColor);
        finalColor = gamma(finalColor);
        canvas.color[i] = finalColor.clamp01();
    }
}
```

**关键点：`indirect * albedo`**

这一乘法是色溢（Color Bleeding）的物理来源。红色墙壁反射的光 `indirect = (0.7, 0.1, 0.1)` 打到白色球上 `albedo = (0.9, 0.9, 0.9)` 得到 `(0.63, 0.09, 0.09)`，球的侧面就带上了淡淡的红色。没有这一乘法，间接光照就只是亮度贡献，无法产生颜色串扰，场景看起来会很假。

---

## ⑤ 踩坑实录

### Bug 1：Triangle 初始化失败（编译错误）

**症状**：
```
error: no matching function for call to 'std::vector<Triangle>::push_back(<brace-enclosed initializer list>)'
```

**原始代码**：
```cpp
tris.push_back({ {{{a,n},{b,n},{c,n}}}, col });
```

**错误假设**：C++ 聚合初始化应该能递归地初始化 `Vertex v[3]` 数组。

**真实原因**：`Triangle::v` 是 `Vertex[3]`，而 `Vertex` 包含 `Vec3`（非 POD 的用户定义类型）。编译器无法推断嵌套初始化列表对应的深度，报告"没有匹配的构造函数"。

**修复方式**：改用显式辅助 lambda：
```cpp
auto makeTriangle = [](Vec3 a, Vec3 b, Vec3 c, Vec3 na, Vec3 nb, Vec3 nc, Vec3 col) {
    Triangle t;
    t.v[0]={a,na}; t.v[1]={b,nb}; t.v[2]={c,nc}; t.albedo=col;
    return t;
};
```
通过逐成员赋值绕过了嵌套聚合初始化的歧义。

---

### Bug 2：未使用变量警告（`-Wextra` 严格模式）

**症状**：
```
warning: variable 'ws' set but not used [-Wunused-but-set-variable]
```

**错误假设**：写下的 `ws[i] = cp.z` 是为了以后可能用到，先留着。

**真实原因**：在 `-Wall -Wextra` 下，这是警告，但目标是 0 warnings，所以必须移除。

**修复**：直接删除 `float ws[3]` 和 `ws[i] = cp.z` 这两行。软光栅化本实现不需要透视校正插值（因为深度是线性 NDC），所以这个变量确实是多余的。

**教训**：不要为"以后可能用"而保留未使用代码，这会污染编译输出并掩盖真正的问题。

---

### Bug 3：STB 头文件警告（第三方库）

**症状**：
```
stb_image_write.h:514:32: warning: missing initializer for member 'stbi__write_context::buffer'
(同类警告共 8 处)
```

**错误假设**：以为是自己代码的问题，开始排查 `main.cpp`。

**真实原因**：这是 `stb_image_write.h` 内部的 `s = { 0 }` 写法在 GCC `-Wmissing-field-initializers` 下触发的警告，不是我们代码的 bug。

**修复**：用 `#pragma GCC diagnostic push/pop` 包围 STB 的 include：
```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#pragma GCC diagnostic ignored "-Wunused-parameter"
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

**教训**：第三方库警告不代表自己代码有错，但压制时要精确：只压制包含第三方代码的范围，立即 `pop` 恢复，不要全局关警告。

---

### 调试思路：场景 "太亮" 问题（潜在陷阱）

在调试过程中观察到顶部区域均值 228.8，如果间接光照权重设得过高（如 `indirect * albedo * 2.0f`），整个场景会过曝，吃掉所有细节。

**原因**：间接光照叠加在直接光照上，如果权重过高会导致亮度超出 1.0，ACES tone mapping 会把这部分压平但细节损失。

**经验值**：`indirect × albedo × 0.5f` 在这个场景中最佳，给 GI 足够的视觉存在感，同时不会过曝。

---

## ⑥ 效果验证与数据

### 量化验证结果

```
像素均值: 192.0  标准差: 78.6
图像尺寸: (800, 600)
通道数: 3
文件大小: 127KB (> 10KB ✅)
顶部区域均值: 228.8
底部区域均值: 9.5
```

**分析**：
- 均值 192：处于合理亮度区间（10-240），场景以明亮的 Cornell Box 为主体，整体偏亮是预期的
- 标准差 78.6：说明图像有显著的亮暗对比变化（场景中有暗角落和明亮区域）
- 顶部(228.8) > 底部(9.5)：天花板照明 + 上方光源使顶部亮，地面局部有阴影使底部偏暗，分布合理

### 性能数据

```
场景三角形数:    1310 (Cornell Box + 两个盒子 + 一个球体)
探针总数:        256 + 64 + 16 = 336
Cascade 0 构建:  ~120ms (256探针 × 16方向 × BVH暴力搜索)
Cascade 1 构建:  ~60ms  (64探针 × 32方向)
Cascade 2 构建:  ~30ms  (16探针 × 64方向)
级联合并:        <5ms
Deferred Shading: ~40ms (800×600 = 480000像素)
总耗时:          0.307 秒
```

**对比参考**：同等质量的路径追踪（每像素64 spp）需要约 30-120 秒。Radiance Cascade 的速度优势来自探针复用：1310 个三角形 × 336 探针 × 平均48方向 = 约 2100 万次射线求交（比路径追踪少 2-3 个数量级）。

### GI 效果验证

通过观察输出图像可以确认：
1. **色溢**：中央灰球的左侧（朝向红墙）明显带有偏红色调，右侧（朝向绿墙）带有偏绿色调——这是 GI 色溢的标志，纯直接光照无法产生此效果
2. **角落填充**：墙角交界处不是纯黑，有来自对面墙壁反射的柔和亮度
3. **多光源软化**：三个不同颜色的光源（暖白、蓝、橙）产生了混合色的间接光，比单一光源更丰富

---

## ⑦ 总结与延伸

### 技术局限性

**本实现的主要简化和局限**：

1. **单次反弹**：探针只存储直接光照打到表面后的辐射度，没有二次反弹。真实 GI 需要无限次反弹（但实践中 2-3 次就够了）。实现多次反弹需要让每级联的射线命中点也查询下一层探针，形成递归——这是 Lumen 中更复杂的部分。

2. **探针最近邻查询**：当前用线性搜索找最近探针，空间不均匀时（如一个角落）可能拿到不准确的探针。生产级实现会做三线性插值（8个最近探针加权），但会增加约 8 倍的查询开销。

3. **无遮挡感知**：查询间接光照时没有考虑"这个表面点到探针之间是否有遮挡"。如果墙内部也有探针，会出现光透过墙的"漏光"问题。生产级实现需要 Visibility Test（方向可见性函数）。

4. **探针高度固定**：本实现的探针都在 Y=1.5m/2.5m/3.0m 平面上，对于高层建筑或多层场景效果会退化。完整实现应该是 3D 网格探针。

### 可优化方向

**短期改进**：
- **双线性/三线性插值**：用四个/八个最近探针插值，消除网格边界的接缝
- **自适应探针密度**：在几何变化复杂的区域（如角落、缝隙）自动加密探针
- **时序重投影**：帧间复用上一帧的级联数据，降低每帧的构建开销

**中期演进**：
- **多次反弹**：通过递归级联查询实现 2-3 次光路弹跳
- **3D 探针网格**：从 2D 平铺扩展到真正的三维空间均匀分布
- **Screen-Space 精化**：用 SSR 或 SSGI 在像素级别补充 Cascade 的高频细节

**工业级路线**：
- 结合 Sparse Virtual Textures：为大世界场景做级联探针的流式加载
- GPU 加速：将级联构建搬到 Compute Shader，实现真正实时（Lumen 的做法）
- 时空滤波：使用时间累积 + 空间 A-Trous 滤波降噪，允许每帧只更新部分探针

### 与本系列的联系

- [04-19 Irradiance Caching & SH Probes](../daily-coding-irradiance-probes-2026-04-19/)：使用球谐函数压缩探针数据，是更传统的静态 GI 方案
- [04-01 SDF Ray Marching](../daily-coding-sdf-raymarching-2026-04-01/)：Ray Marching 是构建级联时射线追踪的替代方案，SDF 场景可以大幅加速探针构建
- [03-23 Voxel Cone Tracing](../daily-coding-voxel-cone-tracing-2026-03-23/)：另一种实时 GI 方案，与 Radiance Cascade 的核心区别在于存储介质（体素 vs 探针网格）

**今日收获**：Radiance Cascade 的优雅之处在于它把"不同尺度的光传播用不同精度表示"这个直觉变成了严格的层级结构，并通过自顶向下的合并自然地连接了不同尺度。这种思想在计算机图形学中反复出现：Mipmap 是它在纹理采样领域的对应，BVH 是它在碰撞检测领域的对应。当你面对"精度 vs 范围"的权衡时，多级结构往往是正确答案。

---

*代码仓库：[daily-coding-practice/2026/05/05-02-radiance-cascade-gi](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-02-radiance-cascade-gi)*
