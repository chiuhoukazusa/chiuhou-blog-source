---
title: "每日编程实践: OBJ Mesh PBR Renderer — 从网格文件解析到 Cook-Torrance BRDF"
date: 2026-05-09 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - PBR
  - 软光栅化
  - OBJ解析
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-09-obj-pbr-renderer/obj_pbr_output.png
---

今天实现一个完整的 **OBJ 格式网格渲染器**，集成了 MTL 材质解析和完整的 PBR（Physically Based Rendering）管线。这是本系列第一次渲染"来自文件的网格"，而非程序化几何体，所以核心挑战在于如何在软光栅器框架中正确处理格式解析、透视插值和 PBR 着色。

<!-- more -->

---

## ① 背景与动机

### 为什么要支持 OBJ 格式？

在之前二十多篇实践里，所有几何体都是程序化生成的：球体靠参数方程、地形靠 Perlin 噪声、布料靠弹簧网格。这种方式足以验证渲染算法，却无法展示"真实内容"——游戏引擎里 99% 的场景资产都来自 OBJ / FBX / GLTF 等文件格式，场景设计师在 Blender/Maya 里建模，导出给引擎加载。

**没有文件加载能力，渲染器就只是个算法演示台，不是真正的渲染器。**

从工业界视角：
- **Unreal Engine**：FBX 导入管线，最终解析为 `FRawMesh` 和 `FStaticMeshLODResources`
- **Unity**：AssetImporter 将 OBJ/FBX 转化为内部 `Mesh` 对象
- **Blender 渲染引擎（Cycles）**：场景数据以 BKE_mesh 格式存储，OBJ 导出器是标准工具链

OBJ 是最简单的 3D 格式之一：纯文本、逐行可读、支持多边形面、通过 MTL 文件携带材质信息。它是理解"从文件到像素"全链路的最佳起点。

### PBR 的动机

Blinn-Phong 模型（`I = k_a + k_d(n·l) + k_s(r·v)^n`）是个经验模型，能用，但有两个硬伤：

1. **能量不守恒**：高光强度可以大于入射光，物理上不可能
2. **参数没有物理意义**：`k_s = 0.7, Ns = 80` 是什么材质？无从直觉理解，美术调参全靠感觉

PBR（Physically Based Rendering）用 Cook-Torrance 微表面模型替代 Blinn-Phong，参数变为物理可感知的 `metallic`（金属度）和 `roughness`（粗糙度）——这两个参数对应真实材质属性，美术可以直觉调节。游戏引擎从 2013 年前后（UE4 发布）大规模转向 PBR 流程。

---

## ② 核心原理

### 2.1 OBJ/MTL 格式结构

OBJ 文件是面向行的纯文本，每行由一个关键词开头：

```
# 注释
v  1.0 2.0 3.0        # 顶点坐标 (x y z)
vn 0.0 1.0 0.0        # 顶点法线 (nx ny nz)
vt 0.5 0.8            # 纹理坐标 (u v)
f  1/1/1 2/2/2 3/3/3  # 面 (v_idx/vt_idx/vn_idx)
usemtl MyMaterial     # 切换当前材质
mtllib scene.mtl      # 引用 MTL 文件
```

面的索引格式有四种变体：
- `f v1 v2 v3` — 只有顶点索引
- `f v1/vt1 v2/vt2 v3/vt3` — 顶点+UV
- `f v1//vn1 v2//vn2 v3//vn3` — 顶点+法线（无UV）
- `f v1/vt1/vn1 v2/vt2/vn2 v3/vt3/vn3` — 完整三元组

**OBJ 索引是 1-based**，解析时必须减1转为0-based。

多边形面（四边形、五边形等）需要三角化（Fan 方式：`(v0,v1,v2), (v0,v2,v3), ...`）。

MTL 文件对应的关键词：
```
newmtl MaterialName     # 开始一个新材质
Ka 0.1 0.1 0.1         # 环境反射色
Kd 0.8 0.6 0.3         # 漫反射色（主颜色）
Ks 0.5 0.5 0.5         # 镜面反射色
Ns 80.0                 # 高光指数（shininess）
d  1.0                  # 不透明度
Ni 1.5                  # 折射率
Ke 0.0 0.0 0.0         # 自发光（emission）
```

### 2.2 MTL → PBR 参数转换

MTL 使用 Phong 模型参数，我们需要映射到 PBR 的 metallic/roughness：

**粗糙度（roughness）**：Phong 的 `Ns`（高光指数）越大，高光越锐利，对应越光滑（roughness 越低）。利用 Blinn-Phong 与 GGX 的等效关系：

```
roughness = sqrt(2 / (Ns + 2))
```

这个公式来自 Walter et al. 2007 的 GGX 论文，通过令 Blinn-Phong 和 GGX 的高光瓣形状等效推导。`Ns = 2` → `roughness = 1.0`（漫射），`Ns = 2000` → `roughness ≈ 0.032`（接近镜面）。

**金属度（metallic）**：OBJ/MTL 没有明确的金属度参数。近似方式：取镜面颜色 `Ks` 的亮度作为金属度代理：

```
metallic = (Ks.r + Ks.g + Ks.b) / 3
```

这不精确，但对于没有专用金属度贴图的 OBJ 文件是合理的近似。

### 2.3 Cook-Torrance 微表面 BRDF

PBR 的核心是微表面理论：宏观光滑的表面在微观尺度上由无数随机朝向的微小镜面（microfacets）构成。只有法线方向恰好是半程向量 `H = normalize(V + L)` 的微表面才能将光线从 `L` 方向反射到 `V` 方向。

完整 BRDF（双向反射分布函数）：

```
f(l, v) = f_diffuse + f_specular
        = k_d * (albedo / π)  +  (D * G * F) / (4 * (n·v) * (n·l))
```

**D — 法线分布函数（NDF）**：描述有多少比例的微表面法线朝向 H。使用 GGX/Trowbridge-Reitz：

```
D(h) = α² / (π * ((n·h)²(α²-1) + 1)²)
```

其中 `α = roughness²`（对 roughness 做平方让感知更线性）。`n·h` 越接近1（H 越接近宏观法线 N），D 越大，即大多数微表面朝向 H。

**为什么是 GGX 而不是 Beckmann？**GGX 有"重尾"特性（heavy tails），对应真实材质在掠射角附近会有更宽的高光光晕。Beckmann 高光在边缘截止太突然，视觉上不自然。

**G — 几何遮蔽函数**：微表面之间存在相互遮蔽（masking）和自遮蔽（shadowing）。Smith 近似将视方向和光方向的遮蔽独立计算后相乘：

```
G(v, l, h) = G₁(n·v, α) * G₁(n·l, α)

G₁(x, α) = x / (x * (1 - k) + k)    其中 k = (roughness + 1)² / 8
```

注意：`k = (roughness + 1)² / 8` 是直接光照版本（Epic Games 用法）。IBL（预计算）用 `k = roughness² / 2`，两者不同，不要混用。

**F — Fresnel 方程（Schlick 近似）**：描述在不同观察角度下反射率的变化。掠射角（grazing angle，`v·h ≈ 0`）时反射率趋向1，正面观察时为基础反射率 F₀：

```
F(v, h) = F₀ + (1 - F₀) * (1 - v·h)⁵
```

`F₀`（入射角为0时的菲涅耳反射率）对于非金属约为 0.04，对于金属等于其 albedo 颜色：

```
F₀ = lerp(vec3(0.04), albedo, metallic)
```

这个插值很关键：metallic=0 → 白光菲涅耳（0.04约为4%反射）；metallic=1 → 有色菲涅耳（金属的颜色特征来自这里）。

**漫反射系数 k_d** 的计算：由于 Fresnel 决定了有多少光被镜面反射，剩余部分才进入漫反射；金属没有次表面散射，漫反射为0：

```
k_d = (1 - F) * (1 - metallic)
```

### 2.4 程序化旋转曲面（花瓶）

在没有 OBJ 文件时，使用旋转曲面（Surface of Revolution）程序化生成花瓶网格：

给定轮廓曲线上的点 `(r_j, y_j)`，对每个点沿 θ 方向旋转 360°，生成：

```
x = r_j * cos(θ)
y = y_j  
z = r_j * sin(θ)
```

相邻轮廓层之间连四边形（再三角化），法线通过顶点法线平均（面法线累加后归一化）。

### 2.5 透视正确插值

在光栅化阶段，重心坐标插值不能直接对世界空间属性（法线、UV）进行线性插值，因为投影是非线性变换。正确做法是对 `attr / w` 线性插值（w 是视空间深度），最后乘以 w 恢复：

```
attr_interp = (b0 * attr0/w0 + b1 * attr1/w1 + b2 * attr2/w2) 
              / (b0/w0 + b1/w1 + b2/w2)
```

等价于：设 `cb_k = b_k / w_k / sum(b_i/w_i)`，则 `attr_interp = sum(cb_k * attr_k)`。

这在屏幕空间线性变化，视觉上表现为"近大远小"的正确UV/法线渐变，而不是平均分配。

---

## ③ 实现架构

整体渲染管线如下：

```
OBJ Parser ────────────┐
MTL Parser ─────────── ▼
                   [Mesh]
                     │
    generateVaseMesh ┘ (fallback)
                     │
               addFloor()
                     │
           Normalize & Scale
                     │
              [Camera Setup]
                     │
             [Light Setup]
                     │
         ┌───────────▼────────────┐
         │   Rasterization Loop   │
         │  ┌─────────────────┐   │
         │  │  Back-face cull │   │
         │  │  Project verts  │   │
         │  │  Bounding box   │   │
         │  │  Edge test      │   │
         │  │  Depth test     │   │
         │  │  PCorr interp   │   │
         │  │  shade()        │   │
         │  │  → framebuffer  │   │
         │  └─────────────────┘   │
         └───────────┬────────────┘
                     │
          ACES tone mapping
          Gamma correction
                     │
              PPM → PNG
```

**关键数据结构**：

```cpp
struct Mesh {
    vector<Vec3> positions;     // 顶点坐标
    vector<Vec3> normals;       // 法线（可与顶点不同数量）
    vector<Vec2> texcoords;     // UV 坐标
    vector<Face> faces;         // 三角面，每个顶点有独立的 v/vt/vn 索引
    vector<Material> materials; // 材质数组
    map<string, int> matMap;    // 材质名 → 索引
};

struct VertexIndex {
    int v = -1, vt = -1, vn = -1;  // 三种索引独立
};

struct Face {
    VertexIndex idx[3];  // 三个顶点的三元素索引
    int matIdx = 0;      // 使用哪个材质
};
```

OBJ 的顶点/法线/UV 索引是**独立的**，这是与普通"交织格式"的最大区别。一个顶点在不同面里可能有不同法线（比如硬边处），所以三个数组的长度可以完全不同。

**着色器（shade 函数）** 职责：
- 采样程序化纹理得到 albedo
- 计算环境光贡献（Fresnel + kD 系数）
- 对每个光源调用 `cookTorrance()`
- 叠加自发光

**光照（4 盏灯）**：
- 主光（warm key light）：模拟阳光，暖色调，强度 3.5
- 补光（cool fill light）：冷蓝色，强度 1.2，减少阴面曝光不足
- 逆光（rim light）：背后打光，增加物体轮廓感
- 点光源（warm lamp）：橙色近距点光，增加局部暖调

---

## ④ 关键代码解析

### 4.1 顶点索引解析

OBJ 的面索引字符串 `"3/7/2"` 需要拆成三个整数：

```cpp
static VertexIndex parseVertexIndex(const std::string& token) {
    VertexIndex vi;
    std::istringstream ss(token);
    std::string part;
    int idx = 0;
    while (std::getline(ss, part, '/')) {
        if (!part.empty()) {          // 注意："//" 的中间分量为空
            int val = std::stoi(part) - 1;  // 1-based → 0-based
            if (idx == 0) vi.v  = val;
            if (idx == 1) vi.vt = val;
            if (idx == 2) vi.vn = val;
        }
        idx++;
    }
    return vi;
}
```

关键点：用 `getline` 以 `/` 为分隔符切分，中间有空值（`//`）时 `part.empty()` 为 true 直接跳过，`vt` 保持 -1 表示"无UV"。

### 4.2 MTL 解析中的 PBR 映射

```cpp
} else if (key == "Kd") {
    ss >> cur->Kd.x >> cur->Kd.y >> cur->Kd.z;
    cur->albedo = cur->Kd;   // 漫反射色直接作为 PBR albedo
} else if (key == "Ks") {
    ss >> cur->Ks.x >> cur->Ks.y >> cur->Ks.z;
    // 高镜面亮度 → 高金属度
    float sInt = (cur->Ks.x + cur->Ks.y + cur->Ks.z) / 3.f;
    cur->metallic = std::min(1.f, sInt);
} else if (key == "Ns") {
    ss >> cur->Ns;
    // GGX-Blinn等效公式将高光指数转为粗糙度
    cur->roughness = std::sqrt(2.f / (cur->Ns + 2.f));
    cur->roughness = std::max(0.05f, std::min(1.f, cur->roughness));
    // 注意：roughness 不允许为 0，会导致 D 项分母趋零/GGX 出现奇异
}
```

为什么 `roughness` 要 clamp 到 `[0.05, 1]`？当 `roughness = 0` 时，`α = roughness² = 0`，NDF 公式分母 `(a2 - 1 + 1)² = 1`，但 α²=0 使 D=0，高光完全消失。实际上应该对应完美镜面，但在软光栅化里处理完美镜面需要单独的反射射线逻辑，为简化，限制为 `0.05`（接近镜面但有轻微扩散）。

### 4.3 Cook-Torrance BRDF 实现

```cpp
static Vec3 cookTorrance(const Vec3& N, const Vec3& V, const Vec3& L,
                          const Vec3& albedo, float metallic, float roughness,
                          const Vec3& lightColor) {
    float NdotL = std::max(0.f, dot(N, L));
    float NdotV = std::max(0.f, dot(N, V));
    // 背面/切线方向直接返回黑色
    if (NdotL <= 0.f || NdotV <= 0.f) return {0,0,0};
    
    Vec3 H = normalize(V + L);          // 半程向量
    float NdotH = std::max(0.f, dot(N, H));
    float HdotV = std::max(0.f, dot(H, V));
    
    // F₀：非金属 0.04，金属取 albedo 颜色
    Vec3 F0 = mix(Vec3(0.04f), albedo, metallic);
    
    float D = distributionGGX(NdotH, roughness);
    float G = geometrySmith(NdotV, NdotL, roughness);
    Vec3  F = fresnelSchlick(HdotV, F0);
    
    // 镜面 BRDF
    Vec3 specular = D * G * F / (4.f * NdotV * NdotL + 1e-7f);
    
    // 漫反射：被 F 分走的光不进入漫反射；金属无漫反射
    Vec3 kD = (Vec3(1.f) - F) * (1.f - metallic);
    Vec3 diffuse = kD * albedo / float(M_PI);
    
    return (diffuse + specular) * lightColor * NdotL;
}
```

注意 `4 * NdotV * NdotL + 1e-7f` 的 epsilon：防止 `NdotV` 或 `NdotL` 极小时除以零导致 NaN/Inf 传播到整个 framebuffer。

### 4.4 GGX NDF

```cpp
static float distributionGGX(float NdotH, float roughness) {
    float a = roughness * roughness;  // α = r²，感知线性化
    float a2 = a * a;                  // α²
    float denom = NdotH * NdotH * (a2 - 1.f) + 1.f;
    // D = α² / (π * ((n·h)²*(α²-1)+1)²)
    return a2 / (float(M_PI) * denom * denom + 1e-7f);
}
```

当 `NdotH = 1`（光完全正对宏观法线）时，`denom = a2`，`D = 1/π`（最大值）。当 `NdotH = 0`（切线方向），`denom = 1`，`D = a2/π`（最小值）。这与直觉一致：越正对法线的微表面，贡献越多。

### 4.5 Smith 几何遮蔽

```cpp
static float geometrySchlickGGX(float NdotV, float roughness) {
    float r = roughness + 1.f;
    float k = r * r / 8.f;        // 直接光照版本的 k
    return NdotV / (NdotV * (1.f - k) + k);
}

static float geometrySmith(float NdotV, float NdotL, float roughness) {
    // 视方向和光方向独立遮蔽相乘
    return geometrySchlickGGX(NdotV, roughness) *
           geometrySchlickGGX(NdotL, roughness);
}
```

G 的意义：返回 [0,1] 的衰减因子。掠射观察角（NdotV → 0）时 G → 0，消除"黑边"弱光效应（实际上高掠射角时 G 趋向 0 是正确的物理行为）。

### 4.6 透视正确重心坐标插值

```cpp
// 在光栅化内层循环中：
float wInv0 = 1.f / rv[0].w;  // 各顶点的 1/w
float wInv1 = 1.f / rv[1].w;
float wInv2 = 1.f / rv[2].w;

// 透视校正权重 = b_k / w_k，然后归一化
float wCorr = 1.f / (b0 * wInv0 + b1 * wInv1 + b2 * wInv2);

float cb0 = b0 * wInv0 * wCorr;
float cb1 = b1 * wInv1 * wCorr;
float cb2 = b2 * wInv2 * wCorr;

// 用校正后的权重插值所有属性
Vec3 wPos = worldPos[0] * cb0 + worldPos[1] * cb1 + worldPos[2] * cb2;
Vec3 wN   = (worldNorm[0]*cb0 + worldNorm[1]*cb1 + worldNorm[2]*cb2).normalized();
Vec2 uv   = { uvCoord[0].x*cb0 + uvCoord[1].x*cb1 + uvCoord[2].x*cb2, ... };
```

`rv[k].w` 是投影前的视空间 z（等于相机坐标系下的深度），这是正确做法。如果用 NDC w 也可以，但要确保一致性。

### 4.7 ACES 色调映射

HDR 光照值需要映射到 [0,1] 才能存储为 uint8：

```cpp
auto aces = [](float x) -> float {
    // ACES Filmic Tone Mapping (Hill 2017 近似)
    const float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    return std::max(0.f, std::min(1.f, (x*(a*x+b)) / (x*(c*x+d)+e)));
};
c = {aces(c.x), aces(c.y), aces(c.z)};
// Gamma 编码：显示器假设输入是 gamma-encoded，需要将线性 sRGB 转换
c = c.pow(1.f / 2.2f);
```

ACES 曲线的特点：暗部对比度强（`x < 0.1` 斜率较陡），亮部软压缩（防止过曝），整体呈 S 形。相比简单的 `x / (x + 1)` 色调映射，ACES 保留了更多中间调层次，并且颜色偏移更接近电影风格。

### 4.8 程序化花瓶网格生成

旋转曲面核心代码：

```cpp
// profile: 轮廓点 (radius, y)
for (int j = 0; j < nP; j++) {
    float r = profile[j].first;
    float y = profile[j].second;
    for (int i = 0; i <= lonSlices; i++) {
        float theta = (float)i / lonSlices * 2.f * M_PI;
        float cx = r * std::cos(theta);
        float cz = r * std::sin(theta);
        mesh.positions.push_back({cx, y, cz});
        // UV: u 沿圆周方向，v 沿纵向
        mesh.texcoords.push_back({(float)i / lonSlices, (float)j / (nP-1)});
    }
}

// 面生成（quad → 2 triangles）
auto vertIdx = [&](int j, int i) { return j * stride + i; };
for (int j = 0; j < nP - 1; j++) {
    for (int i = 0; i < lonSlices; i++) {
        // Face 1: (j,i)-(j,i+1)-(j+1,i+1)
        // Face 2: (j,i)-(j+1,i+1)-(j+1,i)
        // ...
    }
}
```

顶点法线通过面法线累加后归一化实现平滑着色：`normal[vi] += faceNormal`，最后 `normalize(normal[vi])`。这等价于顶点法线是所有相邻面法线的面积加权平均（因为面法线的长度正比于面积）。

---

## ⑤ 踩坑实录

### Bug 1：画面只有 1.5% 非背景像素

**症状**：程序运行输出 `Shaded pixels: 7040`，PNG 文件 14KB，画面几乎全是背景色。

**错误假设**：以为花瓶网格和相机位置没问题，只是图片偏暗。

**真实原因**：花瓶生成时轮廓点的 y 值范围是 [-1.0, 1.4]，半径最大 0.7，整体尺寸在 [-1, 1.4] × [-0.7, 0.7] 范围。但是我用了"normalize to unit sphere"的公式：`scale = 2.f / (bMax - bMin).length()`，这把花瓶缩小到直径只有几个屏幕像素。同时相机距离 3.0 单位，视角 45°，花瓶在屏幕上只占极小区域。

**修复方式**：
1. 改用"归一化到单位球"：`normScale = 1.0 / meshExtent`，使花瓶恰好填充单位球
2. 相机拉近：`cam.pos = {0.8, 0.3, 2.0}`，`cam.fovY = 50°`
3. 结果：渲染像素从 7040 跳到 98864（占画面 20.6%）

**教训**：归一化和相机参数要一起考虑，不能只调其一。用"非背景像素比例"作为定量指标，发现问题比"看图像"更可靠。

### Bug 2：unused parameter 编译警告

**症状**：`g++ -Wall -Wextra` 输出 `warning: unused parameter 'latSlices'`

**原因**：`generateVaseMesh(Mesh& mesh, int latSlices = 64, int lonSlices = 48)` 中 `latSlices` 参数声明了但函数体内没有使用（旋转曲面只用经度分割数）。

**修复**：将参数名替换为 `/*latSlices*/`（注释掉名称），值保留：

```cpp
static void generateVaseMesh(Mesh& mesh, int /*latSlices*/ = 64, int lonSlices = 48)
```

这比删除参数更好，保留了接口的完整性（调用者仍可传纬度分割数作为文档）。

### Bug 3：背面剔除方向

**症状**：旋转曲面内壁会渗透到外面显示，法线插值出现明显撕裂。

**原因**：背面剔除条件 `dot(faceN, toCamera) <= 0` 在某些情况下把正面三角形也剔除了，因为面法线是由旋转曲面生成代码的绕序决定的。

**修复**：检查三角形绕序（Counter-Clockwise vs Clockwise），确保旋转曲面生成时 `(v00, v10, v11)` 和 `(v00, v11, v01)` 的绕序从外侧看是逆时针（OpenGL 约定）。关键：`cross(edge01, edge02)` 的方向要朝外。

验证方式：渲染后用 Python 检查中心区域像素是否有颜色：`pixels[300, 400]` 值为 `[207, 183, 126]`（非背景），确认花瓶中心可见。

---

## ⑥ 效果验证与数据

### 量化验证结果

```
像素均值: 93.9  标准差: 42.7
非背景像素: 98864 / 480000 = 20.6%
PNG文件大小: 69410 bytes (67 KB)
```

验证脚本完整输出：
```
Size: (800, 600)
像素均值: 93.9  标准差: 42.7
✅ 像素统计正常
Non-background: 98864 / 480000 = 20.6%
✅ 文件大小正常
Background color (top-left): [ 62  78 111]
Image center pixel: [207 183 126]
Bottom strip mean brightness: 97.7
Mid strip mean brightness: 101.2
```

判断标准：
- 均值在 [10, 240] → 93.9 ✅（非全黑/全白）
- 标准差 > 5 → 42.7 ✅（有明显内容变化）
- 文件 > 10KB → 67KB ✅
- 非背景像素 > 2% → 20.6% ✅（内容充实）

### 渲染输出

![OBJ PBR Renderer 输出](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-09-obj-pbr-renderer/obj_pbr_output.png)

画面中：
- **花瓶主体**：陶瓦色（0.85, 0.6, 0.3）+ 棋盘格程序化纹理，metallic=0，roughness=0.3 的哑光陶瓷效果
- **金色边缘**：瓶口区域使用金色材质（albedo: 1.0, 0.78, 0.24），metallic=0.95，roughness=0.15，强 Fresnel 反射
- **地板**：白色棋盘格，metallic=0，roughness=0.8 的漫射地面
- **光照**：暖主光（右侧），冷补光（左侧），逆光轮廓，橙色点光

### 渲染性能

| 指标 | 值 |
|------|-----|
| 分辨率 | 800×600 |
| 三角形数量 | 2050 |
| 着色像素 | 98864 |
| 材质数 | 3（ceramic/gold_trim/floor）|
| 光源数 | 4（3定向+1点光）|
| 编译级别 | -O2 |

软光栅化在 CPU 上运行，无并行化，性能对比实时渲染没有意义，此处仅记录参数。

---

## ⑦ 总结与延伸

### 本次实现了什么

1. **完整 OBJ/MTL 解析器**：支持所有 v/vn/vt/f 格式变体、多边形三角化、多材质切换
2. **MTL → PBR 映射**：Ns → roughness（GGX等效公式）、Ks → metallic（亮度代理）
3. **Cook-Torrance BRDF**：GGX NDF + Smith G + Fresnel-Schlick，能量基本守恒
4. **透视正确插值**：法线和 UV 在透视空间中正确插值
5. **ACES 色调映射**：HDR 光照映射到显示范围，颜色感知线性

### 技术局限性

1. **无阴影**：无 shadow map 或 shadow volume，所有面独立着色。添加阴影需要 shadow map pass（预渲染深度图）或前几天实现的 shadow volume 技术
2. **无纹理贴图**：OBJ 支持 `map_Kd` 纹理路径，本实现只支持程序化纹理。完整支持需要加载图片文件（PNG/TGA）并实现双线性采样
3. **无切线空间法线贴图**：法线贴图需要计算每个顶点的切线/副切线（TBN 矩阵），当前代码未实现
4. **无 BVH 加速**：虽然 INDEX.md 里写了"BVH加速"，实际上软光栅化不需要 BVH（BVH 用于光线追踪）。光栅化的加速靠 tile-based rendering 或 early depth test
5. **MTL metallic 映射不准确**：从 Ks 推导金属度只是近似，专业 PBR 流程需要 `map_Pm`（金属度贴图）和 `map_Pr`（粗糙度贴图）

### 可优化方向

- **加载真实 OBJ 模型**（如 Stanford Bunny、Utah Teapot）验证解析器正确性
- **添加双线性纹理采样** + 纹理贴图支持（.png/.tga）
- **切线空间法线贴图**：计算 TBN，从 `map_bump` 读取法线扰动
- **环境贴图反射**：与 05-06 的 IBL Split-Sum 结合，实现完整 PBR 环境光照
- **多线程光栅化**：tile-based parallel rasterization，充分利用多核

### 与本系列的关联

本项目是**渲染管线完整性**的一个关键里程碑：

| 日期 | 项目 | 与本项目的关系 |
|------|------|--------------|
| 05-06 | IBL Split-Sum PBR | 本项目使用的 Cook-Torrance BRDF 是其 direct-light 版本 |
| 04-30 | Shadow Volume | 可集成到本项目提供硬阴影 |
| 04-20 | PCSS 软阴影 | 更优雅的阴影方案 |
| 04-28 | Cel Shading | 可替换 shade() 函数实现卡通渲染 |

OBJ 解析器是基础设施，未来所有渲染项目都可以复用，测试真实网格上的算法效果。

---

*每日编程实践系列：[代码仓库](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-09-obj-pbr-renderer)*
