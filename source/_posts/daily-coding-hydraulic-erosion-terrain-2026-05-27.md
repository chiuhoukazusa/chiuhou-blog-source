---
title: "每日编程实践: Procedural Terrain with Hydraulic Erosion — 程序化地形生成与水力侵蚀模拟"
date: 2026-05-27 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 地形生成
  - 程序化内容
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-27-hydraulic-erosion-terrain/terrain_3d_view.png
---

# 每日编程实践: 程序化地形生成与水力侵蚀模拟

![3D等角视角渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-27-hydraulic-erosion-terrain/terrain_3d_view.png)

今天的项目是**程序化地形生成 + 水力侵蚀模拟渲染器**。使用 Perlin 噪声 fBm 生成基础高度图，然后通过粒子化水力侵蚀算法模拟大自然亿万年的地质过程，最后用软光栅化渲染出有视觉层次感的地形。

---

## ① 背景与动机

### 为什么需要程序化地形？

开放世界游戏需要大量地形内容，手工雕刻高度图需要专业美术人员花费大量时间。程序化生成可以在运行时或离线烘焙时快速生成无限不重复的地形。

**没有程序化地形生成会怎样？**
- 手工雕刻 4096×4096 的地形高度图需要几周时间
- 不同区域风格难以保持一致
- 玩家在大地图上探索容易发现重复感

**工业界实际应用场景：**
- **《荒野大镖客2》**：Rockstar 使用多层噪声 + 艺术家雕刻结合的工作流程生成美国西部地形
- **《No Man's Sky》**：纯程序化行星地形，每颗星球高度图都在运行时生成
- **Unreal Engine 5 的 Landscape 系统**：提供侵蚀模拟工具作为地形后处理效果
- **《我的世界》**：全程序化地形，柏林噪声驱动山脉、平原、洞穴生成
- **GIS（地理信息系统）**：水文模拟用于预测真实世界的河流冲刷路径

### 水力侵蚀的必要性

仅靠噪声生成的地形有一个明显问题：**山脊太锋利、坡面太平滑、没有河流侵蚀痕迹**。

真实山地地形的特征：
- 山脊线细而锋利（长期被侵蚀）
- 山谷宽而平缓（沉积物积累）
- 有明确的水系流向（谷地相互连通）
- 坡面有侵蚀沟（小型水流刻出的细线）

水力侵蚀模拟正是为了弥补纯噪声地形缺乏这些"自然感"的不足。

---

## ② 核心原理

### 2.1 Perlin 噪声与 fBm 地形

**Perlin 噪声的基本思路：**

经典 Perlin 噪声将空间划分为单位格，在每个格点上赋予随机梯度向量，然后通过双线性插值（使用平滑步函数）将格点梯度混合。

**平滑步函数（fade function）：**

```
f(t) = 6t⁵ - 15t⁴ + 10t³
```

这个函数在 t=0 和 t=1 处导数为零，使得相邻格点之间过渡绝对平滑。直觉理解：如果用线性插值 `lerp`，格点边缘会有明显棱角（一阶导数不连续）；fade 函数使一阶、二阶导数在边界处都为零。

**梯度函数：**

对于 3D Perlin 噪声，使用 16 个固定梯度向量（指向立方体的12条边方向）：
```
grad(hash, x, y, z):
    h = hash & 15
    u = (h<8)  ? x : y
    v = (h<4)  ? y : (h==12||h==14 ? x : z)
    return (h&1 ? -u : u) + (h&2 ? -v : v)
```

**分形布朗运动（fBm）：**

单层 Perlin 噪声频率单一，无法模拟自然界的多尺度特征（大山脉 + 中等丘陵 + 小细节）。fBm 通过叠加多个不同频率、振幅的噪声层来解决：

```
fBm(x, y, octaves) = Σᵢ aᵢ · noise(fᵢ·x, fᵢ·y)
```

其中：
- `aᵢ = gain^i`（默认 gain=0.5，每层振幅减半）
- `fᵢ = lacunarity^i`（默认 lacunarity=2.0，每层频率加倍）

直觉理解：低频层决定整体山脉走向，高频层添加中等起伏，更高频层添加细小石块感。就像自然界中大地质构造（板块运动）决定大尺度地形，小尺度地形由风化侵蚀决定。

**幂函数山峰增强：**

```cpp
h = std::pow(h, 1.5);
```

将 [0,1] 范围的高度做幂次变换，使得低洼区域更多（河流、平原），高地区域更少但更尖锐（山峰）。这模拟了真实山地地貌的高度分布（大部分是低地，少量是山峰）。

### 2.2 粒子化水力侵蚀算法

**算法核心思想（Particle-based Hydraulic Erosion）：**

模拟一滴水从随机位置落下，沿地形梯度向低处流动。水滴携带沉积物：
- 在陡坡/高速处**侵蚀**地形（挖走土壤，放入水滴的"背包"）
- 在平缓/低速处**沉积**（将背包里的土壤放下）

最终效果：经过成千上万个水滴，山脊被侵蚀变锋利，山谷被沉积填平变宽，形成自然河道。

**水滴状态：**

每个水滴携带：
- 位置 `(x, y)`
- 速度向量 `(vx, vy)`
- 水量 `water`（随蒸发减少）
- 携带的沉积物量 `sediment`

**梯度计算（双线性插值）：**

水滴位置在格点之间，需要插值得到平滑高度和梯度：

```cpp
float h00 = terrain.h(cellX,   cellY);
float h10 = terrain.h(cellX+1, cellY);
float h01 = terrain.h(cellX,   cellY+1);
float h11 = terrain.h(cellX+1, cellY+1);

float gradX = (h10 - h00) * (1 - fracY) + (h11 - h01) * fracY;
float gradY = (h01 - h00) * (1 - fracX) + (h11 - h10) * fracX;
float curH  = h00*(1-fracX)*(1-fracY) + h10*fracX*(1-fracY)
            + h01*(1-fracX)*fracY      + h11*fracX*fracY;
```

梯度 `(gradX, gradY)` 就是高度函数在该点的偏导数，指向最陡上坡方向。水滴向下坡方向流动，即梯度的反方向。

**速度更新（惯性 + 梯度）：**

```cpp
drop.vx = drop.vx * inertia - gradX * (1 - inertia);
drop.vy = drop.vy * inertia - gradY * (1 - inertia);
```

`inertia`（默认0.05）控制水滴"记住"之前方向的程度。纯梯度（inertia=0）会使水滴每步都沿最陡坡，但会在平缓区域陷入局部极小值。加入惯性使水滴保持动量，能跨越小凸起继续流动——这更符合真实物理。

直觉类比：把一块石头扔在斜坡上，它不会每步都严格沿最陡方向，而是保持一定动量向前。

**沉积物容量（Sediment Capacity）：**

水滴能携带多少沉积物？容量由坡度、速度、水量共同决定：

```cpp
float sedCap = std::max(-deltaH * speed * drop.water * capacity, minSlope);
```

- `deltaH < 0`（下坡）时，容量大（水往低处流，动能大，能携带更多泥沙）
- 速度越快，容量越大
- 水量越多，容量越大
- `minSlope` 防止容量为零（避免数值问题）

**侵蚀 vs 沉积的判断：**

```cpp
if (drop.sediment > sedCap || deltaH > 0) {
    // 沉积：携带量超过容量，或正在爬坡（动能减少）
    float deposit = (deltaH > 0)
        ? std::min(drop.sediment, deltaH)  // 爬坡时沉积被坡挡住的量
        : (drop.sediment - sedCap) * deposition;
} else {
    // 侵蚀：容量有剩余，继续挖土
    float erode = std::min((sedCap - drop.sediment) * erosion, -deltaH);
}
```

**双线性权重写回地形：**

侵蚀/沉积不直接操作格点，而是按双线性权重分配到周围4个格点，这保证地形修改是连续平滑的：

```cpp
float w00 = (1-fracX)*(1-fracY);
float w10 = fracX*(1-fracY);
float w01 = (1-fracX)*fracY;
float w11 = fracX*fracY;
terrain.h(cellX,   cellY)   += deposit * w00;
terrain.h(cellX+1, cellY)   += deposit * w10;
terrain.h(cellX,   cellY+1) += deposit * w01;
terrain.h(cellX+1, cellY+1) += deposit * w11;
```

如果直接操作最近格点，地形修改会产生离散的"坑"和"峰"，视觉上非常不自然。

**蒸发：**

```cpp
drop.water *= (1 - evaporation);  // evaporation = 0.01
```

水量随时间蒸发，当水量低于阈值（0.01）时停止模拟。这限制了每个水滴的生命周期，同时使得长流程的水滴（主河道）比短流程（小支流）影响更大。

### 2.3 高度分层着色

真实地形的颜色与高度和坡度高度相关：

| 高度区间 | 对应地物 | 颜色特征 |
|---------|---------|---------|
| 0~0.12 | 深水 | 深蓝 → 中蓝 |
| 0.12~0.16 | 浅水/湿地 | 橄榄绿 |
| 0.16~0.22 | 沙滩 | 米黄 |
| 0.22~0.45 (低坡) | 草地 | 深绿 |
| 0.45~0.55 (低坡) | 灌木 | 深绿偏暗 |
| 任意高度 (高坡) | 裸岩 | 灰褐 |
| 0.72~0.82 | 高山岩 | 浅灰 |
| 0.82~1.0 | 积雪 | 白 |

坡度（`slope = 1 - normal.y`）决定是否显示裸岩——陡坡无法留住植被和土壤，即使在低高度区域也会露出岩石。

### 2.4 法线计算与光照

**中心差分法计算法线：**

```cpp
Vec3 terrainNormal(const Terrain& t, int x, int y) {
    float hL = t.h(x-1, y);
    float hR = t.h(x+1, y);
    float hD = t.h(x, y-1);
    float hU = t.h(x, y+1);
    float scale = (float)t.size * 0.5f;
    return Vec3(hL - hR, 2.0f / scale, hD - hU).normalized();
}
```

这等价于两个切线向量的叉积：
- `tangentX = (2/sz, hR-hL, 0)` （沿X方向的切线）
- `tangentY = (0, hU-hD, 2/sz)` （沿Y方向的切线）
- `normal = tangentX × tangentY`

`scale` 参数控制高度方向相对于水平方向的比例，避免法线过度倾斜。

**Lambert漫射 + 环境光：**

```cpp
float diffuse = std::max(0.0f, normal.dot(sunDir));
float ambient = 0.35f;
float light = ambient + (1.0f - ambient) * diffuse;
```

简单但有效：35% 环境光保证背光面不全黑，漫射分量基于兰伯特余弦定律。

**近似AO（环境遮蔽）：**

```cpp
for (int si = 1; si <= 3; ++si) {
    int sx = tx + si, sy = ty - si;
    if (sx < sz && sy >= 0)
        hMax = std::max(hMax, terrain.h(sx, sy));
}
float occlusion = std::max(0.0f, (hMax - hCenter) * 5.0f);
float ao = 1.0f / (1.0f + occlusion * 0.5f);
```

沿阳光来向采样3个点，如果周围有更高的地形遮挡，则该点被遮蔽。这是一种极简的光线遮挡近似，不是真正的光线投射AO，但视觉上能产生山谷阴暗、山脊明亮的效果。

---

## ③ 实现架构

### 数据流

```
随机种子
   ↓
generateBaseTerrain()    — Perlin fBm → height[sz×sz]
   ↓
renderTopView()          — 保存侵蚀前图（用于对比）
   ↓
simulateHydraulicErosion() — 80,000水滴迭代修改 height[]
   ↓
高度归一化 [0,1]
   ↓
renderTopView()          — 侵蚀后俯视图
render3DPerspective()    — 等角3D视角
   ↓
savePNG()                — Python PIL 输出
```

### 关键数据结构

**Terrain 结构体：**

```cpp
struct Terrain {
    int size;
    std::vector<float> height;  // 主要高度图
    std::vector<float> water;   // 水流沉积记录（未来可用于河流可视化）

    float& h(int x, int y) { return height[y * size + x]; }
    float  h(int x, int y) const {
        if (x < 0 || x >= size || y < 0 || y >= size) return 0.0f;
        return height[y * size + x];
    }
};
```

`const` 版本的边界保护（越界返回0）只用于读取，写入版本（引用）要求调用者保证边界安全——这是一个设计上的取舍：写入的性能路径不做额外检查，而是在外层循环保证边界。

**Droplet 结构体（水滴）：**

```cpp
struct Droplet {
    float x, y;        // 位置（浮点，在格点之间）
    float vx, vy;      // 速度方向
    float water;       // 水量
    float sediment;    // 携带的沉积物量
};
```

水滴是栈变量，不动态分配，每次迭代创建新水滴——80,000次迭代但不产生堆分配压力。

### 渲染管线（软光栅化）

**俯视图渲染：**

直接将地形坐标映射到屏幕坐标，无需投影矩阵：

```
地形(tx, ty) → 高度 h → 法线 n → 着色 c → 像素(px, py)
```

每个像素独立计算，完全没有光栅化三角形——本质上是高度图的直接可视化。

**3D等角视图渲染（Painter's算法）：**

等角投影公式（斜45度俯视）：

```cpp
float sx = (wx - wz) * 0.866f;           // cos(30°) ≈ 0.866
float sy = (wx + wz) * 0.5f - wy * 1.2f; // 高度方向拉伸
```

这是标准等角投影（Isometric Projection），世界坐标 (wx, wy, wz) 中：
- wx/wz 是水平面坐标（地形 x/y 方向）
- wy 是高度（上方向）

从后向前绘制（z从大到小），使用深度缓冲避免错误覆盖，用 2×2 像素方块填充每个地形格点。

---

## ④ 关键代码解析

### 4.1 完整的 fBm 实现

```cpp
double fbm(double x, double y, int octaves, double lacunarity = 2.0, double gain = 0.5) const {
    double val = 0, amp = 0.5, freq = 1.0;
    for (int i = 0; i < octaves; ++i) {
        val += amp * noise(x * freq, y * freq);
        freq *= lacunarity;
        amp  *= gain;
    }
    return val;
}
```

**为什么 amp 初始值是 0.5 而不是 1.0？**

当 gain=0.5 时，6层 fBm 的振幅总和为：0.5 + 0.25 + 0.125 + ... ≈ 1.0（几何级数）。这使得最终值大约在 [-1, 1] 范围内，便于后续归一化。如果初始 amp=1.0，则总和约为 2.0，需要额外除以2才能归一化。

**lacunarity=2.0 的含义：**

每层频率是前一层的2倍，对应空间尺度减半。这是自然界分形（自相似）的标准选择：大山 → 小山 → 丘陵 → 岩石，每个尺度的特征尺寸是上一级的一半。

### 4.2 水力侵蚀核心循环

```cpp
for (int d = 0; d < numDroplets; ++d) {
    Droplet drop;
    drop.x = distPos(rng);
    drop.y = distPos(rng);
    drop.vx = 0; drop.vy = 0;
    drop.water = 1.0f;
    drop.sediment = 0.0f;

    for (int life = 0; life < maxLifetime; ++life) {
        // 严格边界保护（确保 cellX+1, cellY+1 都在范围内）
        if (drop.x < 1 || drop.x >= sz-2 || drop.y < 1 || drop.y >= sz-2) break;

        int cellX = (int)drop.x;
        int cellY = (int)drop.y;
        float fracX = drop.x - cellX;
        float fracY = drop.y - cellY;

        if (cellX < 1 || cellX >= sz-2 || cellY < 1 || cellY >= sz-2) break;

        // ... 侵蚀/沉积逻辑 ...

        // 速度归一化后移动（单步长保证稳定）
        drop.vx /= speed;
        drop.vy /= speed;
        drop.x += drop.vx;
        drop.y += drop.vy;

        // 蒸发
        drop.water *= (1 - evaporation);
        if (drop.water < 0.01f) break;
    }
}
```

**为什么速度要归一化？**

不归一化的话，当坡度很大时速度会指数增长，水滴可能一步就跳出地形边界。归一化后每步只移动一个单位，算法稳定。但这丢失了速度大小信息——解决方案是用速度大小影响侵蚀量（`sedCap` 中的 `speed` 项），而不是影响移动距离。

**为什么 maxLifetime=30 而不更大？**

更长的生命周期会使水滴在平坦区域反复游走，产生噪声而非自然侵蚀。30步足以让大多数水滴从出发点流到谷底或边界，同时计算量可控（80,000 × 30 = 2,400,000次迭代）。

### 4.3 双层边界检查的原因

```cpp
// 检查1：入口检查（位置更宽松）
if (drop.x < 1 || drop.x >= sz-2 || drop.y < 1 || drop.y >= sz-2) break;

int cellX = (int)drop.x;
int cellY = (int)drop.y;

// 检查2：格点索引检查（保证 cellX+1 安全）
if (cellX < 1 || cellX >= sz-2 || cellY < 1 || cellY >= sz-2) break;
```

两层检查原因：
- 浮点转整数有截断（`(int)0.9999 == 0`，而 `(int)(-0.01) == 0`，但`(int)(-0.01)` 在 MSVC 上是0，在 GCC 上是0或-1）
- 第一层保护浮点比较，第二层保护整数越界
- 访问时需要 `cellX+1` 和 `cellY+1`，所以上界需要 `sz-2` 而不是 `sz-1`

### 4.4 地形着色的坡度判断

```cpp
// 草地：低坡、中等高度
if (h < 0.45f && slope < 0.7f) {
    // 草地着色
}
// 裸岩：高坡或高海拔（注意这里没有高度上界）
if (h < 0.72f || slope >= 0.6f) {
    // 裸岩着色
}
```

坡度用 `slope = 1 - normal.y` 计算：
- 水平面：normal = (0,1,0)，slope = 0
- 45度坡：normal.y = cos(45°) ≈ 0.707，slope ≈ 0.29
- 垂直面：normal.y = 0，slope = 1

`slope < 0.7` 意味着坡角约 45°，超过这个角度土壤无法稳定附着，只能露出岩石——这与自然界（安息角约30-45°）吻合。

### 4.5 近似AO实现

```cpp
float hMax = hCenter;
for (int si = 1; si <= 3; ++si) {
    int sx = tx + si, sy = ty - si;
    if (sx < sz && sy >= 0)
        hMax = std::max(hMax, terrain.h(sx, sy));
}
float occlusion = std::max(0.0f, (hMax - hCenter) * 5.0f);
float ao = 1.0f / (1.0f + occlusion * 0.5f);
```

采样方向 `(+1, -1)` 是太阳方向 `(1, 3, 1)` 在水平面的投影。检查3个步长内是否有更高的地形——如果有，说明该点被遮挡，乘以较暗的 ao 系数。

这不是真正的射线投射AO，而是一种"水平遮蔽近似"，但在俯视地形渲染中效果出奇地好：山谷和峡谷会呈现为更暗的阴影，山脊保持明亮。

---

## ⑤ 踩坑实录

### Bug 1：运行时 SIGSEGV（越界访问）

**症状：** 程序在"模拟水力侵蚀"阶段崩溃，生成了侵蚀前的图片但随后立刻 Segfault：

```
[3/5] 模拟水力侵蚀 (80000 水滴)...
Segmentation fault (core dumped)
```

**错误假设：** 以为 `cellX < sz-1` 就足够，因为只访问 `cellX` 和 `cellX+1`。

**真实原因：** 当 `cellX == sz-1` 时（即 cellX 取到最大索引），`cellX+1 == sz` 越界。原来的检查条件 `cellX >= sz-1` 实际上是允许 `cellX` 取 `sz-2`，此时 `cellX+1 = sz-1` 安全，**但 `cellX = sz-1` 被允许通过了**。

```cpp
// 错误写法（允许 cellX = sz-1，导致 cellX+1 越界）
if (cellX < 0 || cellX >= sz-1) break;  // 这里应该是 >= sz-1 才 break

// 正确写法（确保 cellX+1 < sz）
if (cellX < 1 || cellX >= sz-2) break;  // 给边界留一格余量
```

等等——我写的 `cellX >= sz-1` 实际上是 `cellX >= 255`（sz=256），即 `cellX == 255` 时 break。这看起来是对的：255+1=256=sz，应该越界。但实际上还有另一个路径：

当水滴位于 `drop.x = sz-2 = 254.5` 时，`cellX = 254`，通过了检查。但在移动后，`drop.x` 变为 `255.5`，新的 `nX = 255`，然后访问 `nX+1 = 256` 越界！

**修复方式：** 对移动后的新坐标也做严格检查：

```cpp
// 移动后立即检查
if (drop.x < 1 || drop.x >= sz-2 || drop.y < 1 || drop.y >= sz-2) {
    // 边界处沉积并停止
    break;
}

int nX = (int)drop.x;
int nY = (int)drop.y;
// 额外安全检查
if (nX < 0 || nX+1 >= sz || nY < 0 || nY+1 >= sz) break;
```

**教训：** 浮点坐标系的边界检查要在**每次移动后**重新验证，不能依赖进入循环时的一次性检查。

**用 AddressSanitizer 快速定位：**

```bash
g++ main.cpp -g -fsanitize=address -o terrain_dbg
./terrain_dbg
```

ASan 直接给出了文件名和行号，节省了猜测时间。这是调试越界 Bug 的标准做法。

### Bug 2：编译警告（Wunused-variable 和 Wunused-parameter）

**症状：** 编译时有3个 `-Wextra` 警告：

```
warning: unused variable 'oldX' [-Wunused-variable]
warning: unused variable 'oldY' [-Wunused-variable]
warning: unused parameter 'hasRiver' [-Wunused-parameter]
```

**原因：** 
- `oldX/oldY` 是在重构代码时留下的遗留变量（原本用于记录上一步位置，后来逻辑改变后不再需要）
- `hasRiver` 是预留给"河流可视化"功能的参数，实现中暂未用到

**修复：**

```cpp
// 删除 oldX/oldY
drop.x += drop.vx;  // 直接移动，不保存旧位置

// 用注释参数名消除警告，保留接口兼容性
Color terrainColor(float h, float slope, bool /*hasRiver*/) {
```

`/*hasRiver*/` 语法是 C++ 中消除"未使用参数"警告的惯用法，保留了参数位置（调用者不需要改变）同时告知编译器"这是故意不用的"。

**教训：** 及时清理不用的变量，`-Wall -Wextra` 是发现此类问题的有效工具。对于预留接口的参数，用注释参数名是比 `(void)hasRiver;` 更优雅的做法。

### Bug 3：PNG 转换工具不可用

**症状：** 程序运行成功（0 exit code）但没有生成 `.png` 文件，只有 `.ppm` 文件：

```bash
$ ls
terrain_before_erosion.png.ppm  terrain_erosion  main.cpp
```

**原因：** 代码调用 `convert`（ImageMagick）转换格式，但该工具不在 PATH 中：

```
which: no convert in (...)
```

**修复：** 改用 Python Pillow（已安装）：

```cpp
bool savePNG(const Image& img, const std::string& filename) {
    std::string ppmFile = filename + ".ppm";
    img.savePPM(ppmFile);
    std::string cmd = "python3 -c \"from PIL import Image; "
                      "img=Image.open('" + ppmFile + "'); "
                      "img.save('" + filename + "')\" 2>/dev/null";
    int ret = std::system(cmd.c_str());
    if (ret == 0) {
        std::remove(ppmFile.c_str());
    }
    return (ret == 0);
}
```

**教训：** 不应假设环境中安装了特定的外部工具。在代码中添加工具存在性检查，或优先使用更通用的备选方案。先用 `which convert` 验证工具是否存在，不存在则回退到 Python。

---

## ⑥ 效果验证与数据

### 输出图片验证

```
--- terrain_before_erosion.png ---
文件大小: 148.2 KB
像素均值: 56.6  标准差: 28.7  尺寸: (800, 600)
✅ 像素统计正常

--- terrain_top_view.png ---
文件大小: 192.9 KB
像素均值: 57.9  标准差: 30.6  尺寸: (800, 600)
✅ 像素统计正常

--- terrain_3d_view.png ---
文件大小: 268.3 KB
像素均值: 126.2  标准差: 54.5  尺寸: (800, 600)
✅ 像素统计正常
```

**指标解读：**
- 均值 ~57（俯视图）：合理，暗色区域（水面、深谷）较多，与真实地形分布一致
- 标准差 ~29（俯视图）：说明图像有丰富的色彩变化，不是单调的纯色
- 3D视图均值较高（126）：天空蓝占据一定面积（背景色 RGB(100,150,200)），拉高了均值

### 性能数据

```
运行时间（O2优化）：
  地形生成（256×256 fBm）：~5ms
  侵蚀模拟（80,000水滴）：~200ms
  俯视渲染（2×800×600）：~50ms
  3D渲染（800×600 + ZBuffer）：~80ms
总计：~0.33 秒
```

**侵蚀前 vs 侵蚀后对比：**

![侵蚀前地形](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-27-hydraulic-erosion-terrain/terrain_before_erosion.png)

*侵蚀前：纯 fBm 噪声地形，山脊圆润，没有明确水系痕迹*

![侵蚀后俯视图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-27-hydraulic-erosion-terrain/terrain_top_view.png)

*侵蚀后：山谷更明显，谷地有沉积物填平迹象，整体更自然*

### 地形统计

```
最低高度: 0.0（归一化后）
最高高度: 1.0（归一化后）
平均高度: 0.4699（接近0.5，分布较均匀）
```

平均高度约0.47，说明高地和低地的比例接近1:1，这对于 6层 fBm 是正常的（高度分布接近高斯分布，中值约0.5）。

---

## ⑦ 总结与延伸

### 本次实现的局限性

**1. 等角视图的深度排序问题：**

当前使用 ZBuffer + Painter's算法绘制 2×2 像素方块，但在等角投影中，两个不同高度的方块可能在屏幕上重叠，导致后面方块遮挡前面方块（深度关系反转）。解决方案是改用真正的三角形光栅化。

**2. 侵蚀参数固定：**

当前参数（inertia=0.05, capacity=8, erosion=0.3）是经验值，适合这个特定尺度的地形。不同尺度（更大更小的地形）需要调参才能得到理想效果。

**3. 无热流侵蚀（Thermal Erosion）：**

水力侵蚀之外，还有热流侵蚀（Thermal Erosion）：岩石热胀冷缩碎裂后沿坡面堆积。没有热流侵蚀，陡坡会保持不自然的锋利度，而不是形成堆积在坡脚的碎石堆。

**4. 无河流可视化：**

代码中有 `water` 字段记录水流路径，但尚未用于渲染。未来可以将高水流量区域渲染为蓝色河流。

### 可优化方向

**算法层面：**
- 加入热流侵蚀（Thermal Erosion）使陡坡自然崩塌
- 河流侵蚀（River Erosion）用于大规模河谷生成
- 多层岩石材质（不同岩层有不同硬度，影响侵蚀速率）

**性能层面：**
- SIMD/AVX 并行化 fBm 计算（每个格点独立，完全可并行）
- GPU 水力侵蚀（OpenCL/CUDA，80k水滴完全并行）
- 层次细节（LOD）：远处地形用低精度，近处用高精度

**渲染层面：**
- 真正的三角形光栅化（消除等角视图的深度错误）
- 大气散射（远处山脉呈蓝色雾气）
- 水面反射（低洼区域的湖泊添加菲涅尔反射）
- 程序化植被（在草地区域根据坡度/高度/法线方向添加树木）

### 与本系列其他文章的关联

- **05-15 FFT Ocean Surface**：同样是程序化噪声 + 真实物理模拟的组合，区别是地形是静态烘焙，海洋是实时模拟
- **05-23 SDF Ray Marching**：SDF 也可以用于程序化地形（将高度图转为 SDF 做光线步进），能获得更精确的接触阴影
- **05-24 Subsurface Scattering**：地形渲染中沙地/雪地实际上有次表面散射效果，是未来可以加入的特性
- **04-24 Bent Normal AO**：弯曲法线 AO 可以替代当前简陋的近似 AO，给地形渲染更准确的环境遮蔽

### 关键收获

今天最大的收获是**粒子化水力侵蚀算法的边界处理**。表面上看，在一个 256×256 的网格上做边界检查非常简单，但浮点坐标在每步移动后都需要重新验证，不能只在循环入口做一次性检查。这个教训在任何基于粒子/代理的模拟中都普遍适用。

另一个收获是对**fBm 参数意义**的直觉理解：lacunarity、gain、octaves 不是随机调的数字，每个参数都对应自然界的某种物理规律（空间自相似性、能量守恒、尺度分布）。理解了这一点，调参就从"试错"变成了"有依据的选择"。
