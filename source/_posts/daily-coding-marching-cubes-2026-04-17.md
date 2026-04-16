---
title: "每日编程实践: Marching Cubes 等值面提取"
date: 2026-04-17 05:30:00
tags:
  - 每日一练
  - 图形学
  - 几何处理
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-17-marching-cubes/marching_cubes_output.png
---

## 一、背景与动机

在图形学、医学影像、科学可视化等领域，我们经常需要将**体积数据**（Volume Data）转化为可以被 GPU 渲染的**三角形网格**。想象一下 CT 扫描得到的骨骼数据——每个体素有一个密度值，我们希望提取出骨骼表面；或者流体模拟中粒子浓度场，我们希望看到水面的形状；又或者 SDF（有向距离场）隐式表达的几何体，需要显式化为网格送给光栅化管线。

### 没有等值面提取会怎样？

最朴素的方案是用体积光线步进（Ray Marching）直接渲染 SDF，这确实可行——本系列 04-01 那天就做过。但 Ray Marching 有几个硬伤：

- **每帧都要步进**：GPU 负担重，不适合实时场景的动态几何
- **无法与传统光栅化管线结合**：物理引擎需要三角形网格做碰撞检测，不能用 SDF 场
- **法线需要重新计算**：每次采样都要跑一次梯度，Ray Marching 本身不产生"顶点"

如果我们能把 SDF 或密度场预先转化为三角形网格，就能走标准的 VBO/EBO 路径，享受 GPU 硬件加速光栅化的所有好处。

### Marching Cubes 的工业地位

Marching Cubes（MC）是 1987 年由 Lorensen 和 Cline 在 SIGGRAPH 上发表的经典算法。尽管已经将近 40 年，它至今仍是：

- **医学影像软件**（如 ParaView、ITK-SNAP）生成器官网格的首选
- **游戏引擎体素地形**（Minecraft 以外的很多体素游戏用 MC 生成地形网格）
- **流体模拟**（SPH 模拟结束后，用 MC 从粒子密度场提取水面）
- **神经辐射场（NeRF）**：NeRF 训练完后提取 mesh，也会用 MC 或其变体（Marching Tetrahedra）

Unreal Engine 的 Geometry Script 插件、Houdini 的 VDB 工具链，背后都有 MC 的影子。

今天用 C++ 从零实现完整的 MC 算法：从 SDF 标量场到三角形网格，再到软光栅化渲染输出 PNG。

---

## 二、核心原理

### 2.1 问题定义

给定一个标量场 $f: \mathbb{R}^3 \to \mathbb{R}$，和一个等值阈值 $c$，我们希望提取满足 $f(\mathbf{x}) = c$ 的曲面。

对于 SDF，$c = 0$ 对应物体表面；$c > 0$ 对应外部某个偏移面；$c < 0$ 对应内部偏移面。

### 2.2 离散化：将空间切成格子

MC 的第一步是将三维空间划分为均匀的**立方体格子（Cubes/Voxels）**。每个立方体有 8 个顶点，在每个顶点上采样标量场值。

```
立方体顶点编号约定（Paul Bourke 标准）：

    7-------6
   /|      /|
  4-------5 |
  | |     | |
  | 3-----|-2
  |/      |/
  0-------1

顶点 0: (x0, y0, z0)
顶点 1: (x1, y0, z0)
顶点 2: (x1, y1, z0)
顶点 3: (x0, y1, z0)
顶点 4: (x0, y0, z1)
顶点 5: (x1, y0, z1)
顶点 6: (x1, y1, z1)
顶点 7: (x0, y1, z1)
```

### 2.3 等值面穿越哪条边？

对于每个立方体，将 8 个顶点的值与阈值 $c$ 比较：

$$\text{cubeindex} = \sum_{i=0}^{7} \mathbf{1}[f_i < c] \cdot 2^i$$

即：如果顶点 $i$ 的值小于阈值，就置对应的 bit。`cubeindex` 取值 0~255，共 256 种状态。

**直觉**：`cubeindex = 0` 表示全部顶点都在曲面外（$f \geq c$），没有等值面穿过；`cubeindex = 255` 表示全部顶点在曲面内（$f < c$），同样没有等值面穿过。中间的 254 种状态各有不同的等值面穿越模式。

### 2.4 edgeTable：哪些边被穿越？

一个立方体有 12 条边（编号 0~11）。`edgeTable[cubeindex]` 是一个 12-bit 的掩码，告诉我们哪些边被等值面穿过：

```
边 0:  顶点 0-1 (底面前)
边 1:  顶点 1-2 (底面右)
边 2:  顶点 2-3 (底面后)
边 3:  顶点 3-0 (底面左)
边 4:  顶点 4-5 (顶面前)
边 5:  顶点 5-6 (顶面右)
边 6:  顶点 6-7 (顶面后)
边 7:  顶点 7-4 (顶面左)
边 8:  顶点 0-4 (竖直左前)
边 9:  顶点 1-5 (竖直右前)
边 10: 顶点 2-6 (竖直右后)
边 11: 顶点 3-7 (竖直左后)
```

如果 `edgeTable[cubeindex] & (1 << k)` 为真，说明等值面穿过第 $k$ 条边，我们需要在这条边上用线性插值找交点。

### 2.5 线性插值找交点

如果边的两端点为 $\mathbf{p}_1, \mathbf{p}_2$，对应的场值为 $v_1, v_2$，等值面交点为：

$$\mathbf{p} = \mathbf{p}_1 + t(\mathbf{p}_2 - \mathbf{p}_1), \quad t = \frac{c - v_1}{v_2 - v_1}$$

这是 MC 的核心插值步骤。注意 $t \in [0, 1]$ 才有意义（实际上由 edgeTable 保证，因为只有一端在曲面内另一端在外时这条边才被选中）。

为什么是线性插值而不是其他方法？因为对于平滑的标量场，相邻两点之间值的变化可以近似为线性的，精度够用；更高阶的插值（如三次 Hermite）开销更大，在格子足够密的情况下提升有限。

### 2.6 triTable：如何连接三角形？

在被选中的边上得到交点后，需要把这些点连成三角形。`triTable[cubeindex]` 是一个最多 16 个整数的序列（每 3 个组成一个三角形，-1 结束），告诉我们用哪些边的交点连成三角形。

例如：

```
cubeindex = 1 (只有顶点 0 在曲面内)
edgeTable[1] = 0x109 = 0b0001_0000_1001 → 边 0, 3, 8 被穿越
triTable[1]  = {0, 8, 3, -1, ...}       → 一个三角形：edge0, edge8, edge3
```

这张表是 Lorensen & Cline 原始论文中手工列出的，共覆盖 256 种情况。

### 2.7 SDF 场景设计

为了展示有机复杂的等值面，我们构建了一个多形体 smooth-union 场景：

**smooth-min（平滑最小值）**：

$$\text{smin}(a, b, k) = \min(a, b) - \frac{\max(k - |a-b|, 0)^2}{4k}$$

参数 $k$ 控制融合范围：$k=0$ 退化为硬 $\min$，$k$ 越大融合越柔和。

这是 Inigo Quilez 推广的技巧，在 ShaderToy 社区极为流行。直觉上：两个球体的 SDF 取 min 会得到两球的并集（硬边界），用 smin 则会在交界处产生平滑的"溶合"，像两滴水银合并的效果。

场景包含：
- 中心大球（半径 1.2）
- 四个侧向小球（半径 0.6，绕 Y 轴均匀分布，高度随 Y 正弦波动）
- 顶部圆环（大半径 0.8，管半径 0.25）
- 底部胶囊体（连接 (-1,-1.8,0) 和 (1,-1.8,0)，半径 0.3）

所有形体用 smin 融合，$k$ 在 0.3~0.5 之间，形成有机的、类生物的形态。

### 2.8 法线计算

从 MC 得到三角形的顶点位置后，法线可以通过：

**方法1：面法线**：对每个三角形计算 $\mathbf{n} = (\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0)$，然后求顶点相邻三角形的平均。

**方法2：SDF 梯度**（我们用这个）：对每个顶点位置 $\mathbf{p}$ 计算 SDF 的数值梯度：

$$\mathbf{n} = \nabla f(\mathbf{p}) \approx \frac{1}{2\varepsilon}\begin{pmatrix} f(p_x+\varepsilon) - f(p_x-\varepsilon) \\ f(p_y+\varepsilon) - f(p_y-\varepsilon) \\ f(p_z+\varepsilon) - f(p_z-\varepsilon) \end{pmatrix}$$

用 SDF 梯度的优点是法线与 SDF 场完全一致，不依赖网格拓扑，且对于 smooth-union 形成的有机曲面能给出完美平滑的法线。代价是每个顶点要调用 6 次 SDF（每轴 +/- 各一次）。

$\varepsilon = 0.001$，对于我们的场景（空间范围约 ±3）来说足够精确，不会有太多数值误差。

---

## 三、实现架构

### 3.1 整体数据流

```
SDF 标量场（函数 sceneSDF）
        ↓
  Marching Cubes
  ┌─────────────────────────────────────────┐
  │ 遍历每个格子 (N×N×N)                    │
  │   → 采样 8 个顶点的 SDF 值              │
  │   → 计算 cubeindex                     │
  │   → 查 edgeTable，找被穿越的边          │
  │   → 线性插值得到 12 个候选交点          │
  │   → 查 triTable，连接成三角形          │
  │   → 计算每个顶点的 SDF 法线            │
  └─────────────────────────────────────────┘
        ↓
  三角形列表（Vec3 顶点 + Vec3 法线）
        ↓
  软光栅化渲染器
  ┌─────────────────────────────────────────┐
  │ 相机设置（透视投影矩阵）               │
  │ 背景填充（天空渐变）                    │
  │ 遍历每个三角形：                       │
  │   → 3 顶点透视投影到屏幕空间           │
  │   → AABB 裁剪                         │
  │   → 逐像素重心坐标插值（法线、深度）   │
  │   → Phong 着色（漫反射+高光+边缘光）  │
  │   → Z-buffer 深度测试                 │
  └─────────────────────────────────────────┘
        ↓
  Framebuffer（800×600 RGB）
        ↓
  Gamma 矫正（γ = 2.2）
        ↓
  手写 PNG 导出（zlib store + PNG chunks）
```

### 3.2 关键数据结构

```cpp
// 三角形：存储 3 个顶点位置 + 3 个顶点法线
struct Triangle {
    Vec3 v[3]; // 世界空间顶点
    Vec3 n[3]; // 从 SDF 梯度计算的顶点法线
};

// Framebuffer：颜色缓冲 + 深度缓冲
struct Framebuffer {
    std::vector<Vec3> color; // W×H，浮点 RGB
    std::vector<float> depth; // W×H，浮点深度（1/z）
};
```

### 3.3 格子分辨率的权衡

格子数 $N$ 决定了网格质量和计算量：

| N | 格子数 | 三角形数（约） | MC 耗时 | 网格质量 |
|---|--------|--------------|---------|--------|
| 32 | 32,768 | ~3,000 | <10ms | 粗糙，SDF 细节丢失 |
| 64 | 262,144 | ~15,000 | ~60ms | 较好 |
| **80** | **512,000** | **~25,000** | **~80ms** | ✅ 细节充分 |
| 128 | 2,097,152 | ~100,000 | ~500ms | 高质量，过慢 |

选 $N=80$，生成了 25,740 个三角形，MC 阶段 ~80ms，渲染阶段 ~200ms，总计 280ms，平衡了质量与速度。

### 3.4 模块划分

```
main.cpp
├── 数学工具：Vec3（基本向量运算）
├── SDF 场景：sceneSDF, calcNormal
│   ├── sdSphere, sdTorus, sdCapsule
│   └── smin（smooth union）
├── Marching Cubes 核心
│   ├── edgeTable[256]：12-bit 边掩码
│   ├── triTable[256][16]：三角形连接表
│   ├── vertexInterp()：边上线性插值
│   └── polygoniseCube()：单格子处理
├── 软光栅化渲染器
│   ├── Framebuffer：颜色+深度缓冲
│   ├── phongShading()：Phong 光照模型
│   ├── project()：透视投影
│   └── rasterizeTriangle()：三角形光栅化
└── PNG 导出（无第三方库）
    ├── crc32(), adler32()
    ├── zlibStore()：store 模式压缩
    └── savePNG()：写 PNG 文件
```

---

## 四、关键代码解析

### 4.1 polygoniseCube：单格子 MC 核心

```cpp
void polygoniseCube(
    std::array<Vec3, 8> pos,  // 8 个顶点的世界坐标
    std::array<float, 8> val, // 8 个顶点的 SDF 值
    float isoLevel,           // 等值阈值（0.0）
    std::vector<Triangle>& tris) // 输出三角形列表
{
    // Step 1: 计算 cubeindex（哪些顶点在曲面内）
    int cubeindex = 0;
    for (int i = 0; i < 8; i++)
        if (val[i] < isoLevel) cubeindex |= (1 << i);
    // 注意：SDF < 0 表示内部，我们的 isoLevel=0，
    // 所以 val[i] < 0 的顶点被认为在"内部"

    // 早退：如果没有边被穿越，跳过
    if (edgeTable[cubeindex] == 0) return;

    // Step 2: 在被穿越的 12 条边上插值
    Vec3 vertlist[12];
    if (edgeTable[cubeindex] & 0x001)
        vertlist[0] = vertexInterp(isoLevel, pos[0], val[0], pos[1], val[1]);
    // ... (12条边逐一处理)

    // Step 3: 查 triTable，组合三角形
    for (int i = 0; triTable[cubeindex][i] != -1; i += 3) {
        Triangle t;
        t.v[0] = vertlist[triTable[cubeindex][i  ]];
        t.v[1] = vertlist[triTable[cubeindex][i+1]];
        t.v[2] = vertlist[triTable[cubeindex][i+2]];
        // 每个顶点的法线由 SDF 梯度给出
        t.n[0] = calcNormal(t.v[0]);
        t.n[1] = calcNormal(t.v[1]);
        t.n[2] = calcNormal(t.v[2]);
        tris.push_back(t);
    }
}
```

**为什么 `val[i] < isoLevel` 而不是 `>`？**  
这是约定问题。SDF 符号：内部为负，外部为正。我们将"值小于阈值的顶点"定义为"内部"，cubeindex 按此构建。edgeTable 和 triTable 也遵循同样的约定。反过来用 `>` 会导致法线方向反转（外向变内向）。

### 4.2 vertexInterp：边上线性插值

```cpp
Vec3 vertexInterp(float isoLevel, Vec3 p1, float v1, Vec3 p2, float v2) {
    // 处理极端情况：如果某端点恰好在等值面上
    if (std::abs(isoLevel - v1) < 1e-5f) return p1;
    if (std::abs(isoLevel - v2) < 1e-5f) return p2;
    // 如果两端点值相等（极少发生），返回 p1
    if (std::abs(v1 - v2) < 1e-5f)       return p1;
    
    // 标准线性插值
    float t = (isoLevel - v1) / (v2 - v1);
    return p1 + (p2 - p1) * t;
}
```

**为什么要处理极端情况？**  
如果 `v1 == isoLevel`，按公式算出 `t=0`，结果是 `p1`，没有问题。但是如果不加这些 guard，`v1 == v2` 会导致除以零！浮点 NaN 会传染后续所有计算。这几行防御性代码是必须的，不是多此一举。

### 4.3 SDF 场景：smooth-union 有机融合

```cpp
// smooth minimum，融合两个距离场
inline float smin(float a, float b, float k) {
    float h = std::max(k - std::abs(a - b), 0.0f) / k;
    return std::min(a, b) - h*h*k/4.0f;
}

float sceneSDF(Vec3 p) {
    // 中心大球
    float d = sdSphere(p, Vec3(0,0,0), 1.2f);
    
    // 四个侧向小球，绕 Y 轴均匀分布
    // 高度 y 随角度正弦波动，形成非均匀布局
    for (int i = 0; i < 4; i++) {
        float a = i * (3.14159265f / 2.0f); // 0, 90, 180, 270 度
        Vec3 c(1.6f*std::cos(a),            // X
               0.4f*std::sin(a*2.0f),       // Y 正弦波动
               1.6f*std::sin(a));            // Z
        d = smin(d, sdSphere(p, c, 0.6f), 0.5f); // k=0.5，较大融合
    }
    
    // 顶部圆环
    Vec3 pt = p - Vec3(0, 1.5f, 0);
    float dt = sdTorus(pt, 0.8f, 0.25f);
    d = smin(d, dt, 0.3f); // k=0.3，中等融合
    
    // 底部胶囊
    float dc = sdCapsule(p, Vec3(-1.0f,-1.8f,0), Vec3(1.0f,-1.8f,0), 0.3f);
    d = smin(d, dc, 0.4f); // k=0.4
    
    return d;
}
```

**smin 公式推导直觉**：  
普通 min(a,b) 在 a=b 处不可微（导数跳变），造成硬边。smin 在 |a-b| < k 的范围内用二次函数"抬高"最小值，形成平滑过渡。k 的物理意义是融合影响的半径（以距离场单位计）。k=0.5 意味着两个 SDF 相距 0.5 个单位以内就会开始平滑融合。

### 4.4 透视投影

```cpp
bool project(const Vec3& v, const Vec3& camPos, const Vec3& camFwd,
             const Vec3& camRight, const Vec3& camUp,
             float fovY, float& sx, float& sy, float& depth_val)
{
    Vec3 d = v - camPos; // 相机到顶点的向量
    float z = d.dot(camFwd);    // 在相机前向的投影（相机空间 Z）
    if (z < 0.1f) return false; // 裁掉相机后方的点（near plane）
    
    float x = d.dot(camRight);  // 相机右向分量
    float y = d.dot(camUp);     // 相机上向分量
    
    float aspect = (float)W / H; // 800/600 ≈ 1.333
    float tanH = std::tan(fovY * 0.5f); // tan(FOV/2)
    
    // 透视除法：x/(z*tanH*aspect) 归一化到 [-1,1]
    sx = (x / (z * tanH * aspect)) * 0.5f + 0.5f; // [0,1]
    sy = (y / (z * tanH)) * 0.5f + 0.5f;           // [0,1]
    sy = 1.0f - sy; // 屏幕坐标 Y 向下，翻转
    
    depth_val = z; // 深度用相机 Z 值
    sx = sx * W;   // 转换为像素坐标
    sy = sy * H;
    return true;
}
```

**为什么 x 除以 `aspect`？**  
FOV 通常定义为垂直方向的张角。水平方向受屏幕宽高比影响，实际水平 FOV 是 `2*atan(tan(fovY/2)*aspect)`。如果不除 aspect，宽屏下物体会显得横向被压扁。

### 4.5 重心坐标插值光栅化

```cpp
for (int y = minY; y <= maxY; y++) {
    for (int x = minX; x <= maxX; x++) {
        float px = x + 0.5f, py = y + 0.5f; // 像素中心采样
        
        // 重心坐标（2D 三角形面积法）
        float w0 = ((sx1-px)*(sy2-py) - (sx2-px)*(sy1-py)) / area;
        float w1 = ((sx2-px)*(sy0-py) - (sx0-px)*(sy2-py)) / area;
        float w2 = 1.0f - w0 - w1;
        
        // 点在三角形外（任一权重为负）跳过
        if (w0 < 0 || w1 < 0 || w2 < 0) continue;
        
        // 插值深度和法线
        float depth = w0*d0 + w1*d1 + w2*d2;
        Vec3 interpN = (n0*w0 + n1*w1 + n2*w2).normalized(); // 法线重新归一化
        Vec3 interpP = p0*w0 + p1*w1 + p2*w2; // 世界空间坐标（用于着色）
        
        Vec3 vDir = (camPos - interpP).normalized(); // 视线方向
        Vec3 color = phongShading(interpP, interpN, vDir);
        fb.setPixel(x, y, color, depth); // Z-buffer 测试在内部
    }
}
```

**法线为什么要重新 normalized()?**  
重心坐标线性插值的是分量值，两个单位向量的加权平均不一定是单位向量（比如两个互相正交的单位向量平均后长度是 $\frac{\sqrt{2}}{2}$）。不归一化会导致 Phong 着色中 dot product 的值超出 [-1,1]，产生异常高亮或暗区。

### 4.6 Phong 着色 + 边缘光

```cpp
Vec3 phongShading(const Vec3& /*pos*/, const Vec3& normal, const Vec3& viewDir) {
    Vec3 lightDir1 = Vec3(1, 2, 1).normalized();  // 主光源（右上方）
    Vec3 lightDir2 = Vec3(-1, 1, 0.5f).normalized(); // 补光（左侧）
    Vec3 baseColor(0.4f, 0.6f, 0.9f); // 蓝紫色调
    
    float ambient = 0.15f; // 环境光
    float diff1 = std::max(0.0f, normal.dot(lightDir1)); // 主漫反射
    float diff2 = std::max(0.0f, normal.dot(lightDir2)) * 0.4f; // 补光漫反射
    
    // 镜面高光（Phong 模型）
    Vec3 reflDir1 = normal * (2.0f * normal.dot(lightDir1)) - lightDir1;
    float spec1 = std::pow(std::max(0.0f, viewDir.dot(reflDir1)), 32.0f) * 0.6f;
    
    // 边缘光（Rim Light）：法线与视线接近垂直时产生轮廓光
    float rim = std::pow(1.0f - std::max(0.0f, normal.dot(viewDir)), 3.0f) * 0.3f;
    
    Vec3 finalColor = baseColor * (ambient + diff1 * 0.7f + diff2);
    finalColor += Vec3(1,1,1) * spec1;              // 白色高光
    finalColor += Vec3(0.5f, 0.7f, 1.0f) * rim;    // 蓝色边缘光
    
    return finalColor;
}
```

**边缘光的物理直觉**：  
当法线 $\mathbf{n}$ 与视线方向 $\mathbf{v}$ 接近垂直时（即 $\mathbf{n} \cdot \mathbf{v} \approx 0$），这个像素位于物体边缘，从视线方向看几乎是切面。现实中边缘处往往接收到更多环境光（来自周围场景的间接光），显示为亮边。$1 - (\mathbf{n} \cdot \mathbf{v})$ 在边缘时趋近 1，三次幂使其衰减更陡峭，只有最边缘的部分才明显。

### 4.7 手写 PNG 导出（zlib store 模式）

PNG 格式本质上是：

```
PNG 签名（8字节）
→ IHDR chunk（图像宽高、位深等信息）
→ IDAT chunk（压缩的像素数据）
→ IEND chunk（结束标记）
```

像素数据用 zlib 压缩，我们使用最简单的 store 模式（压缩级别 0，不做任何压缩，只做包装）：

```cpp
std::vector<uint8_t> zlibStore(const std::vector<uint8_t>& data) {
    std::vector<uint8_t> out;
    out.push_back(0x78); out.push_back(0x01); // zlib header（CMF=0x78, FLG=0x01）
    
    size_t pos = 0, rem = data.size();
    while (rem > 0) {
        size_t blk = std::min(rem, (size_t)65535); // deflate 最大块 65535 字节
        bool last = (blk == rem);
        out.push_back(last ? 0x01 : 0x00); // BFINAL|BTYPE（store=00）
        out.push_back(blk & 0xff);
        out.push_back((blk >> 8) & 0xff);
        out.push_back((~blk) & 0xff);      // 长度的一补，deflate 规范要求
        out.push_back(((~blk) >> 8) & 0xff);
        out.insert(out.end(), data.begin() + pos, data.begin() + pos + blk);
        pos += blk; rem -= blk;
    }
    // Adler-32 校验和（zlib 结尾）
    uint32_t a = adler32(data.data(), data.size());
    out.push_back((a>>24)&0xff); // 大端序
    ...
    return out;
}
```

**为什么不用真正的 deflate 压缩？**  
Store 模式实现简单，不需要 Huffman 编码或 LZ77，代码量减少 90%。代价是文件更大（本项目输出 1.4MB vs. 压缩后约 200KB），但对于本地测试用途完全可以接受。PNG 规范允许 store 模式。

每行像素前加一个 Filter byte（0x00，表示 None 过滤），这是 PNG 规范要求的。不加会导致所有解码器报错。

---

## 五、踩坑实录

### Bug 1：未使用参数 Warning 导致 -Wextra 失败

**症状**：编译输出 `-Wextra` 警告，不满足 0 warning 要求。

**具体警告**：
```
warning: unused parameter 'pos' [-Wunused-parameter]
warning: variable 'viewDir' set but not used [-Wunused-but-set-variable]
```

**错误原因**：
1. `phongShading` 接收了 `pos` 参数（世界空间坐标，设计上留给后续扩展），但当前实现没用到
2. 光栅化循环里计算了 `viewDir = (camPos - p0).normalized()`，但随后在内层循环另外算了 `vDir = (camPos - interpP).normalized()`，外层那个被遗忘了

**修复**：
```cpp
// 1. 用 C++ 注释掉参数名（保留参数类型，去掉编译器能看到的名字）
Vec3 phongShading(const Vec3& /*pos*/, const Vec3& normal, const Vec3& viewDir)

// 2. 直接删掉外层多余的 viewDir 定义
```

**教训**：设计阶段保留的"未来用"参数，要么用注释参数名压制 warning，要么彻底删掉。不要留着悬空的变量声明。

### Bug 2：edgeTable 中间部分数据需要仔细核对

**背景**：Marching Cubes 的查找表有 256 个 cubeindex，手动录入极易出错。网上流传的各种 C 代码实现之间存在微妙差异（顶点编号顺序不同）。

**我采用的方案**：使用 Paul Bourke 的原始约定（8 顶点按 Z-底面→Z-顶面，前→右→后→左顺时针）。edgeTable 和 triTable 必须配套使用，不能混用不同来源的表。

**验证方法**：  
- 场景中心放一个球（已知正确形状），检查渲染结果是否圆滑
- 如果表格不匹配，会出现：平的大三角面片、锯齿破洞、法线反向（全黑）等症状

### Bug 3：坐标系翻转问题（Y 轴方向）

**症状**：如果不翻转 Y 轴，渲染出的图像天空在下、地面在上，模型看起来是颠倒的。

**原因**：数学坐标系 Y 轴向上，屏幕坐标 Y 轴向下（像素 (0,0) 在左上角）。透视投影后的 sy 值如果直接用作像素 y 坐标，不翻转的话 y 大的点（在数学空间中是上方）会被画到屏幕下方。

**修复**：
```cpp
sy = 1.0f - sy; // 翻转 Y：1-0=1(底部)→0(顶部)，0-1=0(顶部)→1(底部)
```

**历史教训**：本系列 2026-02-20 坐标系 Bug 的延续。Y 翻转是软光栅化中最高频的错误之一。

### Bug 4：PNG 文件生成但 Adler32 报错

**症状**：生成的 PNG 被 libpng 提示"adler32 校验失败"（在某些解码器下不报错，在 Python Pillow 下报错）。

**错误原因**：Adler32 的大端字节序写出顺序搞反了。正确是从最高字节到最低字节：

```cpp
// 错误：先写低字节
out.push_back(a & 0xff); out.push_back((a>>8) & 0xff); ...

// 正确：先写高字节（大端）
out.push_back((a>>24)&0xff); out.push_back((a>>16)&0xff);
out.push_back((a>>8)&0xff);  out.push_back(a&0xff);
```

**为什么是大端？**zlib/deflate 规范（RFC 1950）明确规定 Adler32 以大端序存储。PNG 的所有多字节字段也都是大端（包括 chunk 长度、CRC32）。但习惯写小端（x86 本机字节序）的人很容易犯这个错。

### Bug 5：渲染三角形数量为 0 的极端情况处理

设计了检查：

```cpp
if (triangles.empty()) {
    std::cerr << "ERROR: No triangles generated!" << std::endl;
    return 1;
}
```

实际上对于我们的场景（bounded SDF，`bound=2.8`，等值面在 ±1.5 范围内），这不会触发。但如果把 `bound` 改小到 1.0，等值面会跑到格子范围外，MC 产生 0 个三角形。加这个 guard 是为了提前发现这种配置错误。

---

## 六、效果验证与数据

### 6.1 像素统计验证

```
Pixel mean: 103.89  std: 43.83
```

对应 RGB 均值 103.9/255 ≈ 40.7%，标准差 43.8/255 ≈ 17.2%。
- 均值在合理范围（10~240）✅
- 标准差 > 5，图像有丰富的明暗变化 ✅

上半部分均值（113.9）> 下半部分均值（93.9）：说明天空（较亮的深蓝色）在上方，较暗的背景和模型底部在下方。坐标系翻转正确 ✅

### 6.2 文件信息

| 项目 | 数值 |
|------|------|
| 输出分辨率 | 800 × 600 |
| 文件大小 | 1.4 MB（zlib store，未压缩） |
| MC 格子数 | 80³ = 512,000 |
| 生成三角形数 | 25,740 |
| 总渲染时间 | ~280ms |
| MC 阶段 | ~80ms |
| 光栅化阶段 | ~200ms |

### 6.3 视觉效果

![Marching Cubes Isosurface Extraction](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-17-marching-cubes/marching_cubes_output.png)

图像展示：
- 中心主体（大球 + 4 个侧球 smooth-union 融合）呈现有机连接的形态
- 顶部圆环与主体平滑融合（k=0.3 的过渡带）
- 底部胶囊体延伸形成底座
- 蓝紫色 Phong 着色 + 白色镜面高光 + 蓝色边缘光
- 深色渐变背景（夜空色调）突出模型轮廓
- 25,000+ 三角形提供了足够的曲面细节

### 6.4 编译验证

```
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
```
结果：**0 errors, 0 warnings** ✅

---

## 七、总结与延伸

### 7.1 Marching Cubes 的局限性

**拓扑歧义（Ambiguous Cases）**：  
当某个格子的 cubeindex 使得某些面的连接方式不唯一时（共 6 种歧义情况），不同的三角化选择可能导致孔洞或自相交。原始 MC 不处理这个问题；Marching Cubes 33（Chernyaev 1995）和 Dual Marching Cubes 专门解决了歧义问题。

**各向异性**：  
MC 生成的网格沿格子轴方向有轻微偏差（"格子感"），对于高曲率区域表现差。解决方案：提取后运行 Laplacian Smooth 平滑，或用 Marching Tetrahedra（无歧义问题，但三角形数量更多）。

**内存开销**：  
80³ = 512,000 格子，每格需要采样 8 个 SDF 值。如果 SDF 很复杂（如带 BVH 的场景），内存访问模式较差。生产实现通常用分层（Octree）的 MC，只在等值面附近细化。

**非流形边**：  
某些 cubeindex 状态会产生非流形（non-manifold）顶点/边（多于 2 个三角形共享一条边），这对某些后处理操作（如内外判断、雕刻）有问题。

### 7.2 可优化方向

1. **共享顶点**：当前实现每个三角形存储独立的 3 顶点，相邻三角形的公共顶点被重复存储。建立顶点哈希表可以减少内存并支持顶点归并。

2. **并行 MC**：每个格子独立，天然适合 OpenMP 或 GPU 计算。CUDA 实现可以在 GPU 上并行跑 MC，速度提升 100x+。

3. **Dual Contouring**：MC 的改进版，能更好地重建尖锐特征（棱角、折痕），适合建筑体素、CAD 模型等。

4. **自适应分辨率**：对等值面曲率高的区域用更细的格子，平坦区域用粗格子，减少总体计算量同时保持细节。

5. **OpenVDB 整合**：OpenVDB 是工业级稀疏体素数据库（Houdini、Blender 均支持），其内置 MC 实现经过大量优化，是生产环境的首选。

### 7.3 与系列其他项目的关联

- **04-01 SDF Ray Marching**：隐式表面的另一种渲染方式，不生成网格，直接光线步进。本项目可以理解为"把隐式表面显式化"的工具
- **04-02 BVH Triangle Mesh Renderer**：本项目生成的三角形网格，可以直接输入那天的 BVH 加速光线追踪器
- **04-09 SPH 流体**：SPH 模拟完成后，可以用 MC 从粒子密度场提取水面网格，得到更好的渲染效果
- **04-11 PBD 软体**：软体形变后的 SDF 可以用 MC 可视化形变过程

Marching Cubes 是体积数据到网格的核心桥梁，掌握它意味着可以将本系列的隐式/体积方法与显式网格渲染管线连接起来，打通整个渲染技术栈。
