---
title: "每日编程实践: VXGI 体素全局光照渲染器"
date: 2026-05-14 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 全局光照
  - 体素
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-14-vxgi/vxgi_output.png
---

## 背景与动机

全局光照（Global Illumination，GI）是真实感渲染中最核心也最昂贵的问题。光在现实世界中不只是从光源直射到表面，它还会在墙面、物体间多次弹射，产生漫反射互照（color bleeding）、焦散（caustics）、间接阴影和统一的光照感——这些效果让场景看起来"真实"，缺了它们场景就会显得塑料、干燥。

没有 GI 时：
- 所有阴影面都一片死黑（现实中背光面受到来自其他表面的漫射光）
- 红墙旁边的白墙不会泛红（color bleeding 缺失）
- 地面角落的细节完全丢失（缺少间接遮蔽感）

游戏引擎中 GI 的实现方案经历了漫长的演进：

- **烘焙光照贴图（Baked Lightmap）**：Unreal 3/Unity 时代的主流方案。离线计算后写入贴图，动态物体无法接受 GI，更新慢，占内存。
- **光照探针（Light Probe）**：Unreal 4 的 ILC，Unity 的 Probe Grid。低频近似，对小物体有伪影，不支持高频 specular GI。
- **SVOGI（Sparse Voxel Octree GI）**：Unreal 4 的实验性功能，在稀疏体素八叉树上做锥追踪，效果惊艳但实时代价极高。
- **VXGI（Voxel Cone Tracing GI）**：NVIDIA 2011 年由 Cyril Crassin 在 CryEngine 演示。将场景体素化到 3D 纹理（带 MIP），用少量锥追踪积分半球辐射，实现近似实时的动态 GI。
- **Radiance Cascade**：2024 年 Lumen 的后续演进方向，多级探针级联。

今天实现的是 VXGI 的软光栅版本：不使用 GPU，用纯 CPU 单线程实现，目标是搞清楚 VXGI 的核心思路而不是追求实时性能。

**VXGI 在工业界的实际使用**：
- CryEngine 3.8 引入 SVOGI，后因性能问题降格为预计算模式
- Unreal Engine 4.x 有 VXGI 插件（NVIDIA Gameworks），在 GPU 端实时跑
- Unity HDRP 的 Probe Volume（APV）借鉴了体素/探针思路
- 自研引擎（如 Naughty Dog、Guerrilla）用各种 GI Probes + 锥追踪组合

---

## 核心原理

### 渲染方程与 GI 的本质

渲染方程（Kajiya 1986）：

$$L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o) \cdot L_i(x, \omega_i) \cdot (\omega_i \cdot n) \, d\omega_i$$

其中：
- $L_o$ 是出射辐射（我们要求的量）
- $L_e$ 是自发光项
- $f_r$ 是 BRDF（双向反射分布函数）
- $L_i$ 是入射辐射——这里递归包含了间接光照
- $\omega_i \cdot n = \cos\theta_i$ 是余弦因子（Lambert 定律）

GI 的难点就在 $L_i$：它本身也是另一个表面的 $L_o$，形成无限递归。光线追踪用随机采样 + 俄罗斯轮盘赌截断这个递归。VXGI 则用体素预计算 + 锥追踪来近似积分。

### 体素化的本质

体素化把连续的三角形表面转化为离散的 3D 网格，每个体素存储该位置的：
- 平均反照率（albedo）
- 平均法向量（normal）
- 直接光照辐射（radiance，下一节注入）
- 不透明度（opacity，0=空气，1=表面）

体素化有多种方式：
1. **光栅化投影法**：对三角形分别投影到 XY/YZ/XZ 三个面，保守光栅化填充体素。GPU 可用 geometry shader 实现。
2. **射线检测法**（本实现）：从每个体素中心发射 6 方向射线，如果任何方向在 `VS*0.8` 距离内击中三角形，则认为该体素是表面体素。

射线法简单但保守性差：
```cpp
for(const auto& d : dirs[6]) {
    float bestT = VS * 0.8f;  // 只检测距离 < 0.8 个体素的碰撞
    for(const auto& tri : scene) {
        float t;
        if(rayTriangle(c, d, tri, t) && t < bestT) {
            bestT = t;
            sumAlb += tri.albedo;
            sumNorm += tri.normal();
            hits++;
        }
    }
}
```

`VS * 0.8`：为什么用 0.8 而不是 1.0？因为体素中心到体素表面的最短距离是 `VS/2 = 0.5*VS`，用 0.8 给了一点容差。

### 直接光注入

体素化完成后，对每个实体体素预计算一次直接光照，结果存入 `voxel.radiance`：

```cpp
void injectDirectLight(VoxelGrid& g, const std::vector<Triangle>& scene) {
    for each filled voxel {
        Vec3 c = grid.voxelCenter(ix, iy, iz);
        vox.radiance = directLight(c, vox.normal, vox.albedo, scene)
                     + vox.albedo * 0.05f;  // 小量环境光底色
    }
}
```

这里的直接光用的是解析点光（area light 近似）：

```
L_direct = L_emit * albedo * ndotl * (LAREA / (dist² + 0.1)) / π
```

分解理解：
- `L_emit * albedo`：光源强度乘以表面反照率（漫反射 BRDF 的 albedo/π，π 在分母）
- `ndotl`：Lambert 余弦因子（入射角越斜，受光越少）
- `LAREA / dist²`：面积光的立体角近似（距离越远，光强越弱）
- `/ π`：Lambertian BRDF 的归一化因子

**重要**：这里存的 `radiance` 是"该体素表面会向外辐射多少光"，用于后续 GI bounce 的积分源。

### 锥追踪（Cone Tracing）原理

传统光线追踪用无限细的射线（aperture=0），采样无穷多个方向求积分（蒙特卡洛）。锥追踪的关键洞察：**用一条有宽度的锥代替多条细射线的平均**。

一条半角为 $\alpha$（aperture）的锥，在距离 $d$ 处的"截面半径"为：
$$r(d) = d \cdot \tan(\alpha) \approx d \cdot \alpha$$

在距离 $d$ 处采样体素时，我们不只采样一个点，而是对半径 $r(d)$ 范围内的体素做加权平均——这等价于 MIP 采样。GPU 实现中这就是 3D 纹理的 MIP LOD，CPU 实现中我们用一个空间模糊核模拟：

```cpp
Vec4 sample(Vec3 p, float r) const {
    int kr = max(1, (int)ceil(r / VS));
    kr = min(kr, 3);  // 最大 3 体素半径的模糊
    // 以 p 为中心，在 kr 半径内做距离加权平均
    for each neighbor (dx, dy, dz) in [-kr, kr+1]³ {
        float w = 1.0f / (1.0f + dx² + dy² + dz²);  // 距离平方的倒数权重
        acc += voxel.radiance * voxel.opacity * w;
    }
}
```

锥追踪过程（前-后合成，Front-to-Back compositing）：

```
dist = VS * 2.0   // 起始距离（避开自身体素）
acc = (0,0,0,0)   // RGB + alpha

while dist < maxDist and acc.alpha < 0.95:
    p = start + dir * dist
    coneRadius = aperture * dist           // 锥半径随距离线性增大
    sample = grid.sampleWithRadius(p, coneRadius)
    
    alpha = sample.w * (1 - acc.alpha)    // 前-后合成的 alpha
    acc.rgb += alpha * sample.rgb
    acc.alpha += alpha
    
    step = max(VS, coneRadius * 0.5)      // 步长随锥的扩张而增大
    dist += step
```

**为什么前-后合成？**  
我们希望近处的体素遮挡远处的体素（有体积感）。前-后合成公式 `alpha_new = sample.w * (1 - acc.alpha)` 保证：
- 如果前面已经几乎不透明（`acc.alpha ≈ 1`），则后续采样对结果贡献很小
- 每次累加的 alpha 不超过 `1 - acc.alpha`（不会超过100%透明）

**步长自适应**：`step = max(VS, coneRadius * 0.5)`
- 近处：锥很细，步长接近一个体素宽度，精确采样
- 远处：锥很粗，步长可以更大（因为模糊核本身就很大了），提高效率

### 漫反射 GI：半球锥积分

对一个表面点，漫反射 GI 要积分整个上半球的入射辐射：

$$L_{diffuse-GI}(x) = \frac{1}{\pi} \int_{\Omega^+} L_{incoming}(x, \omega) \cdot (\omega \cdot n) \, d\omega$$

VXGI 用 **5 条固定方向的锥**来近似这个积分（类似 4-point Gauss quadrature + 1条法向）：

| 锥方向 | 余弦权重 | 积分权重 |
|--------|---------|--------|
| 沿法向 n | 1.0 | 0.28 |
| n 旋转±45° (切线方向) | cos45°=0.71 | 0.18 |
| n 旋转±45° (副切线方向) | cos45°=0.71 | 0.18 |

**5 个权重之和为 1.0**（0.28 + 4×0.18 = 1.0），满足归一化条件。

切线帧构建（需要一个与 n 不共线的向量做叉积）：
```cpp
Vec3 buildTangent(Vec3 n) {
    Vec3 up = abs(n.y) < 0.9 ? Vec3(0,1,0) : Vec3(1,0,0);
    return n.cross(up).normalized();
}
```

为什么要判断 `n.y < 0.9`？如果法向量恰好接近 (0,1,0)，叉积会产生接近零向量（数值不稳定），改用 (1,0,0) 避免这个问题。

### 环境光遮蔽（AO Cone Tracing）

VXGI AO 的思路：发射几条紧密的锥（大孔径 → 小 aperture），检测周围有多少体素遮蔽了上半球：

```cpp
float voxelAO(const VoxelGrid& g, Vec3 pos, Vec3 n) {
    // 5条紧凑锥（30°半角）
    Vec3 dirs[5] = {n, (n*ca+t*sa).norm(), ...};
    float aperture = 0.2f;    // 约 11°半角
    float maxDist  = 2.5f;    // 只看近处的遮蔽
    
    float occ = 0;
    for(int i=0; i<5; i++) {
        Vec4 cone = traceCone(grid, start, dirs[i], aperture, maxDist);
        occ += cone.w;  // cone.w 是积累的遮蔽 alpha
    }
    return 1.0f - min(occ/5.0f, 1.0f) * 0.65f;
}
```

`0.65f` 是遮蔽强度系数——即使完全遮蔽（occ=1），AO 最低只到 `1-0.65=0.35`，避免全黑阴影（现实中总有少量环境光）。

### 镜面 GI：反射锥

对于有粗糙度的表面，specular GI 用单条反射锥：

$$L_{specular}(x) = L_{incoming}(x, R(\omega_o))$$

其中 $R(\omega_o) = 2(n \cdot \omega_o)n - \omega_o$ 是反射向量。

锥的孔径和粗糙度成正比：`aperture = roughness * 0.3 + 0.02`

- 光滑表面（roughness=0）：aperture≈0.02，几乎是镜面反射
- 粗糙表面（roughness=1）：aperture≈0.32，积分较大角度范围

---

## 实现架构

整个渲染管线分 5 个阶段：

```
Scene Triangles
       │
       ▼ voxelizeScene()
   VoxelGrid (64³)
   [albedo, normal, opacity]
       │
       ▼ injectDirectLight()
   VoxelGrid
   [+ radiance (direct light)]
       │
       ▼ render() — for each pixel:
       ├─ Ray → Scene intersection
       ├─ Direct Light (shadow ray)
       ├─ diffuseGI (5 cones × traceCone)
       ├─ voxelAO (5 cones × traceCone)
       └─ specularGI (1 cone × traceCone)
              │
              ▼ toneMap (ACES) + gamma
              │
              ▼ writePNG (zlib compress)
              │
           vxgi_output.png
```

### 关键数据结构

**VoxelGrid**：
```cpp
static const int VR = 64;       // 64³ = 262144 个体素
static const float VS = 6.0/VR; // 每个体素 0.09375 世界单位

struct Voxel {
    Vec3 albedo;      // 表面颜色（diffuse reflectance）
    Vec3 normal;      // 平均法向量（用于光照计算）
    Vec3 radiance;    // 预注入的直接辐射（GI 的"能量源"）
    float opacity;    // 0=空气，1=表面体素
};
```

64³=262144 个体素 × 40字节/体素 ≈ 10MB，CPU 完全可以装下。

**VoxelGrid::sample()**：锥追踪的核心采样函数，用距离加权的空间模糊模拟 MIP LOD：

```cpp
Vec4 VoxelGrid::sample(Vec3 p, float r) const {
    // 计算体素坐标（使用 -0.5 偏移以对准体素中心）
    Vec3 lp = p - origin;
    float fx = lp.x/VS - 0.5f;
    int x0 = (int)floor(fx);
    // ...
    
    // kr = ceil(r/VS) 决定采样范围（最多3个体素半径）
    int kr = max(1, (int)ceil(r/VS));
    kr = min(kr, 3);
    
    // 遍历 (2kr+2)³ 邻域（注意是 -kr 到 kr+1 包含边界）
    for dz=-kr to kr+1:
      for dy=-kr to kr+1:
        for dx=-kr to kr+1:
          // 距离平方倒数权重（中心权重最大）
          float w = 1.0 / (1.0 + dx²+dy²+dz²);
          acc += voxel.radiance * voxel.opacity * w;
}
```

**场景构建**：Cornell Box = 5面墙 + 1个顶灯 + 2个盒子，共 32 个三角形。注意：**不需要前墙**，因为相机在场景外部朝内看。

---

## 关键代码解析

### 1. 体素化

```cpp
static void voxelizeScene(VoxelGrid& g, const std::vector<Triangle>& scene) {
    static const Vec3 dirs[6] = {
        {1,0,0},{-1,0,0},{0,1,0},{0,-1,0},{0,0,1},{0,0,-1}
    };
    
    for(int iz=0; iz<VR; iz++)
    for(int iy=0; iy<VR; iy++)
    for(int ix=0; ix<VR; ix++) {
        Vec3 c = g.center(ix, iy, iz);  // 体素中心世界坐标
        Vec3 sumAlb={}, sumNorm={}; int hits=0;
        
        for(const auto& d : dirs) {
            float bestT = VS * 0.8f;    // 只检测一个体素内的碰撞
            for(const auto& tri : scene) {
                float t;
                if(rayTriangle(c, d, tri, t) && t < bestT) {
                    bestT = t;
                    sumAlb += tri.albedo;    // 累积颜色（后续平均）
                    sumNorm += tri.normal(); // 累积法向（后续归一化）
                    hits++;
                }
            }
        }
        
        if(hits > 0) {
            Voxel& v = g.at(ix, iy, iz);
            v.albedo = sumAlb * (1.0f/hits);   // 平均颜色
            v.normal = sumNorm.normalized();    // 归一化平均法向
            v.opacity = 1.0f;
        }
    }
}
```

为什么对 6 方向各自找最近交点，而不是一次找全部？
- 我们想知道"这个体素是否在表面附近"，而不是要准确的交点
- 每个方向独立找最近交点，然后累加，可以更好地处理薄曲面（比如从多个方向都能检测到的角落体素）

时间复杂度：O(VR³ × 6 × N_triangles)，N_triangles=32 时约为 262144×6×32 ≈ 5000万次射线测试。用 BVH 可以大幅加速，这里为了简单直接暴力。

### 2. 锥追踪主循环

```cpp
static Vec4 traceCone(const VoxelGrid& g, Vec3 pos, Vec3 dir,
                       float aperture, float maxD) {
    float dist = VS * 2.0f;   // 初始步距：2个体素宽度（避开起点体素）
    Vec4 acc = {};             // (R, G, B, alpha)
    
    while(dist < maxD && acc.w < 0.95f) {
        Vec3 p = pos + dir * dist;
        float cr = max(aperture * dist, VS * 0.5f);  // 锥截面半径
        // 注意 min(cr, VS*0.5) 避免 cr 小于半个体素（无意义）
        
        Vec4 s = g.sample(p, cr);  // 在半径 cr 内采样
        
        // 前-后合成：近处先合成，远处被遮挡后贡献少
        float alpha = s.w * (1.0f - acc.w);
        acc.x += alpha * s.x;
        acc.y += alpha * s.y;
        acc.z += alpha * s.z;
        acc.w += alpha;
        
        // 自适应步长：锥越宽步长越大（减少远处的采样数）
        dist += max(VS, cr * 0.5f);
    }
    return acc;
}
```

**关键参数选择**：
- `aperture = 0.5`（漫反射）：约 26.5°，适中的扩散范围
- `aperture = 0.2`（AO）：约 11.3°，紧凑锥，只检测近处遮蔽
- `maxDist = 5.0`（GI）：5 个世界单位，约 53 个体素距离
- `maxDist = 2.5`（AO）：只看近处遮蔽，避免大范围假阴影

### 3. 漫反射 GI

```cpp
static Vec3 diffuseGI(const VoxelGrid& g, Vec3 pos, Vec3 n, float strength) {
    // 构建切线帧（正交基）
    Vec3 t = buildTangent(n);
    Vec3 b = n.cross(t).norm();  // 副切线 = n × t
    
    float sa = sin(0.7854f);  // sin(45°) = 0.707
    
    // 5 个半球采样方向：法向 + 4个斜向
    Vec3 dirs[5] = {
        n,
        (n + t*sa).normalized(),
        (n - t*sa).normalized(),
        (n + b*sa).normalized(),
        (n - b*sa).normalized()
    };
    float weights[5] = {0.28f, 0.18f, 0.18f, 0.18f, 0.18f};
    
    Vec3 gi = {};
    Vec3 start = pos + n*(VS*3.0f);  // 起点偏移 3 个体素（避开表面）
    
    for(int i=0; i<5; i++) {
        Vec4 cone = traceCone(g, start, dirs[i], 0.5f, 5.0f);
        float ndotl = max(0.0f, dirs[i].dot(n));  // 余弦权重
        gi += cone.xyz() * (ndotl * weights[i]);  // 加权积分
    }
    return gi * strength;
}
```

为什么 `start = pos + n*VS*3`？
- `VS*2` 是 traceCone 自己的初始步距
- 但起点本身如果在表面体素内，第一次 `g.sample()` 可能采到自己
- 用 `VS*3` 确保起点位于表面体素的上方（空气中），避免自遮挡

### 4. 直接光照

```cpp
static Vec3 directLight(Vec3 pos, Vec3 norm, Vec3 albedo,
                          const std::vector<Triangle>& scene) {
    Vec3 toL = LPOS - pos;
    float dist = toL.len();
    Vec3 lDir = toL / dist;
    
    // Lambert 余弦测试
    float ndotl = max(0.0f, norm.dot(lDir));
    if(ndotl < 1e-5f) return {};  // 背光面直接返回 0
    
    // 阴影测试：从表面沿法向偏移一点，避免自遮挡
    Vec3 sro = pos + norm * 0.005f;
    for(const auto& tri : scene) {
        if(tri.isLight) continue;  // 光源三角形不参与遮挡
        float t;
        if(rayTriangle(sro, lDir, tri, t) && t < dist - 0.01f)
            return {};  // 有遮挡
    }
    
    // 面积光衰减近似：LAREA / (dist² + 0.1)
    float attn = LAREA / (dist*dist + 0.1f);
    return LEMIT * albedo * (ndotl * attn / M_PI);
}
```

`dist*dist + 0.1` 中的 `0.1`：防止极近处距离趋零时能量爆炸（正则化项）。

### 5. ACES 色调映射

HDR 颜色值可能远大于 1.0（光源强度 12），需要 tone mapping 压缩到 [0,1]：

```cpp
float acesApprox(float x) {
    float a=2.51f, b=0.03f, c=2.43f, d=0.59f, e=0.14f;
    return max(0.0f, min(1.0f, (x*(a*x+b)) / (x*(c*x+d)+e)));
}
```

这是 ACES Filmic 的 Narkowicz 近似（2015）。直觉：
- 对小值（暗部）：近似线性（`x*b/e ≈ 0.214x`），保留细节
- 对大值（亮部）：趋向饱和（分子和分母都是二次，比值趋向 `a/c ≈ 1.033`），防止高光爆掉
- 整体 S 曲线：类似人眼感知的对数响应

### 6. PNG 写入（无第三方库）

```cpp
// 1. 构建 raw RGB 字节数组
// 2. 每行加滤波器字节（0=None）
// 3. zlib 压缩
// 4. 写 PNG 文件结构（IHDR + IDAT + IEND chunk）

// chunk 格式：[4B长度][4B标签][NB数据][4B CRC]
auto writeChunk = [&](const char* tag, const unsigned char* d, size_t l) {
    w32((unsigned)l);
    fwrite(tag, 1, 4, f);
    if(l > 0) fwrite(d, 1, l, f);
    // CRC 覆盖 tag+data
    unsigned crc2 = crc32(crc32(0L,Z_NULL,0), (Bytef*)tag, 4);
    if(l > 0) crc2 = crc32(crc2, d, (uInt)l);
    w32(crc2);
};
```

PNG 的 IEND 块长度为 0，CRC 只覆盖 "IEND" 四字节标签，这里容易写错（见踩坑记录）。

---

## 踩坑实录

### Bug 1：编译错误 `-ray.d` 无法取一元负号

**症状**：`error: no match for 'operator-' (operand type is 'Vec3')`

**错误假设**：以为所有 C++ 数学类型都支持一元负号。

**真实原因**：我的 `Vec3` 结构体只定义了二元 `operator-(Vec3 o)const`，没有定义一元 `operator-()const`。`-ray.d` 会被解析为一元负号。

**修复**：
```cpp
// 错误写法
Vec3 specGI = specularGI(..., -ray.d, ...);

// 修复：显式乘以 -1
Vec3 specGI = specularGI(..., ray.d * -1.0f, ...);
// 或者在 Vec3 加一元负号操作符
Vec3 operator-()const { return {-x,-y,-z}; }
```

### Bug 2：全图输出单一颜色 byte=16（最难的 Bug）

**症状**：渲染完成，PNG 写入成功，但图像所有像素均为 RGB(16,16,16)——一片灰色。

**调试过程**：
1. 首先验证 PNG 写入：手动解压缩 PNG 的 IDAT，检查 raw 字节，确认写入逻辑无误（1440600 字节 = 800×600×3+600 filter bytes）。
2. 验证 framebuffer：在 `writePNG` 前打印 `fb[0]`，得到 `(0.0068, 0.0068, 0.0068)`。
3. 验证 tone mapping：`aces(0.0068) ≈ 0.00222`，`gamma(0.00222) ≈ 0.062`，`0.062*255 ≈ 16`——确实是 16！
4. 奇怪：为什么所有像素都是 0.0068？初始化是 Vec3(0)，不是 0.0068。
5. 在 render 循环里打印：`fb[0,0]=(0,0,0)`——framebuffer 就是 0！
6. 那 0.0068 从哪来？原来是我检查的是 `writePNG` 执行**之后**的数据——那时图像已经写完了，显示的是磁盘上读回的值（图片解码有偏差），实际 fb 内容是全0。

**错误假设一**：以为是 PNG 写入有 bug（花了很多时间调试）。

**错误假设二**：以为是 framebuffer 索引错误。

**真实原因**：场景中有一堵**前墙**（`z = -2.5`，法向 (0,0,-1)）。相机在 `z=-4.5` 向 +Z 方向看，第一个击中的几何体就是前墙的**外侧**。前墙外侧的法向是 (0,0,-1)（指向-Z，朝相机），光源在 `(0,2.45,0)`，`ndotl = (0,0,-1)·lDir ≤ 0`，所以 `directLight` 返回零向量。扩散 GI 也为零（体素 radiance 是 0 的直接光×体素——前墙自身也被前墙遮挡）。

**修复**：删除前墙。Cornell Box 的设计本来就没有前墙——相机就是从"开口"处看进去的。

```cpp
// 删除这行
// addQuad({-s,-s,-s},{-s,s,-s},{s,s,-s},{s,-s,-s},{...}); // front wall
```

这是本次最有价值的教训：**调试时要验证渲染出来的颜色，不能只看文件大小和 PNG 头部结构**。

### Bug 3：voxelizeScene 填充数量从 38949 下降到 18472

这是删除前墙后自然减少的——前墙贡献了约 20000 个体素。对 GI 质量有轻微影响（少了前墙的反射），可以接受。

### Bug 4：render 函数调试耗时极长

`-O0` 编译后运行 800×600 需要超过 60 秒（每像素 10 条锥，每锥 14-19 步），调试时SIGTERM。

**经验**：VXGI 即使是 CPU 实现也需要 `-O2` 才能在合理时间内完成。调试单像素时直接写一个单独的 `single_pixel.cpp` 快速验证逻辑，不要用完整 800×600 分辨率。

---

## 效果验证与数据

### 量化像素统计

```bash
python3 -c "
from PIL import Image
import numpy as np

img = Image.open('vxgi_output.png')
pixels = np.array(img).astype(float)
mean = pixels.mean()
std = pixels.std()
top = pixels[:100,:].mean()   # 上部（靠近顶灯）
bottom = pixels[-100:,:].mean() # 下部（远离顶灯）
print(f'尺寸: {img.size}')
print(f'像素均值: {mean:.1f}')
print(f'像素标准差: {std:.1f}')
print(f'上部均值: {top:.1f}, 下部均值: {bottom:.1f}')
"
```

输出结果：
```
尺寸: (800, 600)
像素均值: 104.2    ✅（10~240之间，非全黑/全白）
像素标准差: 63.4   ✅（> 5，图像有内容变化）
上部均值: 122.7    ✅（顶灯附近更亮）
下部均值: 52.4     ✅（地板较暗，符合物理预期）
文件大小: 185KB    ✅（> 10KB）
```

### 渲染效果

![VXGI 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-14-vxgi/vxgi_output.png)

可以观察到的 GI 效果：
- **颜色渗透（Color Bleeding）**：左侧红墙旁的地板和后墙略带红色调，右侧绿墙旁略带绿色调——这是漫反射 GI 的体现
- **间接光照**：即使顶灯被两个盒子部分遮挡的区域，也不是纯黑，有来自墙面的间接光
- **环境光遮蔽**：盒子与地板的接触角落较暗（AO 效果）

### 性能数据

| 阶段 | 耗时 |
|------|------|
| 场景构建（32个三角形）| < 1ms |
| 体素化（64³×6方向×32三角形）| ~25s（CPU 暴力遍历）|
| 直接光注入（18472个体素×阴影测试）| ~2s |
| 渲染（800×600，10锥/像素）| ~4s |
| PNG 写入（zlib 压缩）| < 1s |

总耗时约 31 秒（`-O2`）。

GI 的主要开销：每像素 10 条锥（5 diffuse + 5 AO），每锥约 14-19 步，每步一次 `sample()` 调用（最多 8³=512 次体素查找）。GPU 实现中，体素化 + 渲染都在 GPU 上并行，帧时可以压缩到 16ms 以内（1080p 约 8ms on RTX 3080，参考 NVIDIA 文档）。

---

## 总结与延伸

### VXGI 的局限性

1. **体素分辨率限制细节**：64³ 的分辨率无法表现小于一个体素宽度（约 9cm）的几何细节，薄面体素化容易丢失。实际引擎用稀疏体素八叉树（SVO）或级联体素（Cascaded Voxels）来解决。

2. **动态更新代价高**：场景几何变化时需要重新体素化（本实现每帧体素化需要 25 秒）。GPU 实现用光栅化+几何着色器体素化，可以控制在 2-5ms，但对高频动态场景仍有挑战。

3. **漏光问题（Light Leaking）**：锥追踪越过薄墙时可能采到错误的辐射。缓解方式：减小体素尺寸，或对锥追踪加法向一致性检查（只采样法向相似的体素）。

4. **AO 精度有限**：5 条锥的半球采样太稀疏，拐角处 AO 可能不准确。实际引擎通常用 SSAO/HBAO 做屏幕空间的精细化，VXGI AO 只做低频遮蔽。

### 可优化方向

1. **BVH 加速体素化**：当前 O(N_voxels × 6 × N_tris)，加 BVH 后变成 O(N_voxels × log N_tris)。对 32 个三角形差别不大，但场景复杂时至关重要。

2. **GPU 并行体素化**：geometry shader + conservative rasterization，体素化速度提升 100x+。

3. **3D MIP Map**：真正的 MIP 纹理采样比当前的空间模糊核更准确，且 GPU 硬件支持（`tex3Dlod()`）。

4. **时序重用（Temporal Reprojection）**：帧间复用 GI 结果，用运动向量做反投影，减少每帧的锥追踪数量。

5. **Specular BRDF 改进**：当前用了固定的 roughness=0.7，实际应该从材质读取，并用 GGX 重要性采样替代锥孔径计算。

### 与本系列的关联

- **05-01 ReSTIR**：同样解决 GI 问题，但用的是 Reservoir 重要性采样，更适合多光源场景
- **05-03 PRT**：预计算辐射传输，适合低频环境光，VXGI 可以当作运行时的 PRT 替代方案
- **04-24 Bent Normal AO**：本实现的 `voxelAO()` 思路类似，Bent Normal AO 额外存储了主遮蔽方向
- **04-12 God Rays 体积光**：同样用光线步进（ray marching），本实现的 `traceCone` 是带孔径的广义版本

---

*今天调试了很长时间，但把前墙的坐标系问题、PNG写入验证方法、以及如何快速定位"全黑图"的根因都搞清楚了。每次踩坑都是值得记录的经验。*
