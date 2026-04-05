---
title: "每日编程实践: Procedural Terrain Renderer — 程序化地形渲染器"
date: 2026-04-06 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 渲染
  - 程序化生成
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-06-Procedural-Terrain-Renderer/terrain_output.png
---

# Procedural Terrain Renderer — 程序化地形渲染器

今天是每日编程实践系列第 48 天。主题是**程序化地形渲染**——从零用 C++ 实现一个完整的地形生成与渲染管线：Perlin 噪声高度图 + fBm 分形叠加 + 多生物群系着色 + Phong 光照 + 大气雾 + 软光栅化投影。

![渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-06-Procedural-Terrain-Renderer/terrain_output.png)

---

## ① 背景与动机：为什么要做程序化地形？

### 手工地形的局限

在游戏开发和影视特效中，场景中的地形是最占资源、最难手工制作的内容之一。一片 10km×10km 的游戏地图，如果依靠美术手工刷高度图，需要几周甚至几个月。而且手工地形有一个根本问题：**内容密度与规模不匹配**——想做大就细节少，想要细节就规模小。

程序化生成（Procedural Generation）的核心价值是：**用确定性算法无限扩展内容**，同时保持细节。一段几十行的噪声函数可以生成整个星球的地形，而且每次用不同种子（seed）都能得到独一无二的结果。

### 工业界应用场景

- **《No Man's Sky》**：整个游戏的 18 quintillion 颗行星地形全部程序化生成，包括地貌、植被、大气
- **《Minecraft》**：经典案例，Perlin 噪声叠加控制高度、洞穴、生物群系
- **《Flight Simulator 2020》**：用卫星高程数据 + 程序化细节，在全球范围内生成真实地形
- **UE5 的 Landmass 系统**：允许设计师用笔刷描述宏观地貌，细节由噪声自动填充
- **地形 LOD（Level of Detail）**：远处地形用低分辨率网格，近处用高精度——这是今天实现的核心优化思路之一

### 今天要解决的问题

不借助任何图形 API（OpenGL/Vulkan/DirectX），纯 CPU 软光栅实现：

1. **高度图生成**：用 Perlin 噪声 + fBm 生成自然感地形
2. **法线计算**：从高度图推导每顶点法线，用于光照
3. **生物群系着色**：按高度和坡度映射颜色（雪/岩石/森林/草地/沙滩/海水）
4. **Phong 光照**：环境光 + 漫反射 + 镜面反射
5. **大气雾**：指数衰减雾增加远景层次感
6. **透视投影 + 软光栅化**：投影到屏幕并填充三角形

---

## ② 核心原理：从噪声到地形的完整数学

### 2.1 Perlin 噪声：连续性随机的起点

纯随机数（white noise）不适合地形生成——相邻两点没有任何关联，结果是杂乱的"盐椒噪声"。Ken Perlin 1983 年设计的 **Perlin Noise** 解决了这个问题：

**基本思路**：
1. 把空间划分成整数格子
2. 在每个格子顶点存储一个**随机梯度向量**（不是随机值，是方向）
3. 对于空间中任意点 P，取其所在格子四个顶点的梯度向量
4. 计算 P 到各顶点的位移向量，与梯度做点积，得到影响值
5. 用 **smoothstep 曲线**插值合并四个影响值

核心公式（以 2D 为例）：

```
noise(x, y):
  ix, iy = floor(x), floor(y)          // 格子坐标
  fx, fy = frac(x), frac(y)           // 格子内偏移 [0,1]
  
  // 四顶点梯度点积
  v00 = grad(hash(ix,   iy  ), fx,   fy  )
  v10 = grad(hash(ix+1, iy  ), fx-1, fy  )
  v01 = grad(hash(ix,   iy+1), fx,   fy-1)
  v11 = grad(hash(ix+1, iy+1), fx-1, fy-1)
  
  // Fade 曲线插值
  u, v = fade(fx), fade(fy)
  return lerp(lerp(v00, v10, u), lerp(v01, v11, u), v)
```

**Fade 曲线** `fade(t) = 6t⁵ - 15t⁴ + 10t³`（而不是线性插值）：
- 这是 Ken Perlin 改进版（2002 年）的 Improved Perlin Noise 关键改动
- 原版用的是 `3t² - 2t³`（平滑步函数），一阶导数连续但二阶不连续
- 改进版的 6t⁵ 曲线在 t=0 和 t=1 处**一阶和二阶导数均为零**，消除了格子边界处的视觉瑕疵（"柱状感"）
- 直觉：插值时在端点处"刹车"，中间段加速，过渡更自然

**梯度哈希**的直觉：
- 不存储真实的随机梯度向量（内存开销大），而是用哈希映射到预定义的 16 个方向之一
- 这 16 个方向均匀分布在立方体 12 条边的中点方向上
- 既节省内存，又保证梯度的多样性

代码实现：

```cpp
struct PerlinNoise {
    std::vector<int> p;  // 置换表（permutation table）

    PerlinNoise(unsigned seed = 42) {
        p.resize(512);
        // 初始化 0-255 的排列
        std::iota(p.begin(), p.begin()+256, 0);
        // 用 seed 随机打乱
        std::mt19937 rng(seed);
        std::shuffle(p.begin(), p.begin()+256, rng);
        // 复制一份（避免取模，直接索引 256-511 == 0-255）
        for(int i=0;i<256;i++) p[256+i] = p[i];
    }

    // Ken Perlin 改进版 Fade 曲线
    static float fade(float t) { 
        return t*t*t*(t*(t*6-15)+10); 
    }

    // 梯度点积（15个方向哈希到12条棱方向）
    static float grad(int hash, float x, float y, float z) {
        int h = hash & 15;
        float u = h<8 ? x : y;          // 前 8 个选 x，后 8 个选 y
        float v = h<4 ? y : (h==12||h==14 ? x : z);
        return ((h&1) ? -u : u) + ((h&2) ? -v : v);
    }
    
    float noise(float x, float y, float z=0) const {
        // 格子坐标（& 255 取模）
        int X = (int)std::floor(x) & 255;
        int Y = (int)std::floor(y) & 255;
        int Z = (int)std::floor(z) & 255;
        // 格子内偏移
        x -= std::floor(x); y -= std::floor(y); z -= std::floor(z);
        // Fade 插值权重
        float u=fade(x), v=fade(y), w=fade(z);
        // 8个角的哈希值
        int A  = p[X]+Y,   AA = p[A]+Z,   AB = p[A+1]+Z;
        int B  = p[X+1]+Y, BA = p[B]+Z,   BB = p[B+1]+Z;
        // 三线性插值
        return lerp(
            lerp(lerp(grad(p[AA],   x,  y,  z), grad(p[BA],  x-1,y,  z), u),
                 lerp(grad(p[AB],   x,  y-1,z), grad(p[BB],  x-1,y-1,z), u), v),
            lerp(lerp(grad(p[AA+1], x,  y,  z-1),grad(p[BA+1],x-1,y,z-1),u),
                 lerp(grad(p[AB+1], x,  y-1,z-1),grad(p[BB+1],x-1,y-1,z-1),u),v), w);
    }
};
```

### 2.2 fBm（分形布朗运动）：多尺度叠加

单层 Perlin Noise 的频率是固定的，产生的地形过于平滑。真实地形同时有**大尺度**（山脉走向）和**小尺度**（石头纹理）的变化。**分形布朗运动（Fractional Brownian Motion, fBm）** 通过叠加不同频率和幅度的噪声模拟这种多尺度结构：

```
fbm(x, y, octaves=N):
  value = 0
  amplitude = 1.0   // 初始幅度（控制高度贡献）
  frequency = 1.0   // 初始频率（控制细节尺度）
  max_value = 0     // 用于归一化
  
  for i in range(N):
      value     += noise(x * frequency, y * frequency) * amplitude
      max_value += amplitude
      amplitude *= persistence    // 每倍频幅度衰减（通常 0.5）
      frequency *= lacunarity     // 每倍频频率增加（通常 2.0）
  
  return value / max_value  // 归一化到 [-1, 1]
```

**参数的物理意义**：

- **persistence（持续度）**：控制高频细节的"存在感"
  - 0.5 → 每多一倍频，幅度减半（自然感最强）
  - 0.7 → 细节更多，更崎岖
  - 0.3 → 细节很少，地形很平滑
  - 今天用 0.55，比标准 0.5 稍高，让地形更有棱角

- **lacunarity（空隙度）**：控制频率增长倍数
  - 2.0 是标准值（二倍频），模拟自然界的"1/f 噪声"特性
  - 1/f 噪声是大自然中随处可见的统计规律：海岸线、山脉轮廓、血管分布

- **octaves（倍频层数）**：7层意味着从大尺度到小尺度有 7 个不同的频率贡献

**为什么 fBm 像真实地形？**

地壳构造是多尺度物理过程的叠加：
- 板块运动（大尺度，低频）→ 山脉走向
- 火山侵蚀（中尺度，中频）→ 山体形状
- 风化侵蚀（小尺度，高频）→ 岩石纹理

fBm 的数学结构恰好模拟了这种层级叠加。这是为什么 Perlin Noise + fBm 在 1983 年一经提出就被电影工业广泛采用的原因。

代码实现：

```cpp
float fbm(float x, float y, int octaves=6, float persistence=0.5f, float lacunarity=2.0f) const {
    float val=0, amp=1, freq=1, maxVal=0;
    for(int i=0;i<octaves;i++){
        val += noise(x*freq, y*freq) * amp;
        maxVal += amp;
        amp  *= persistence;
        freq *= lacunarity;
    }
    return val / maxVal;  // 归一化到 [-1, 1]
}
```

地形高度计算：

```cpp
void build() {
    for(int j=0;j<N;j++){
        for(int i=0;i<N;i++){
            float nx = (float)i/(N-1) * 3.5f;  // 噪声坐标（控制地形密度）
            float ny = (float)j/(N-1) * 3.5f;
            float h = pn.fbm(nx, ny, 7, 0.55f, 2.0f);
            // 映射到 [0, 1]
            h = (h + 1.f) * 0.5f;
            // 幂次运算提高山峰尖锐度，降低低洼密度
            h = std::pow(h, 1.5f);
            heights[j*N+i] = h * heightScale;
        }
    }
}
```

**为什么要 `pow(h, 1.5)`？**

fBm 输出近似均匀分布在 [-1,1]。映射到 [0,1] 后，均值接近 0.5，地形整体平坦。`pow(h, 1.5)` 对高度做幂次运算：
- 小于 1 的值变得更小（海洋和平原变大）
- 接近 1 的值几乎不变（山峰保持高度）
- 整体效果：低地区域更多，山峰更突出，地形更有层次感

### 2.3 从高度图计算法线

地形法线不需要存储，可以从相邻高度差推导：

**中心差分法（Central Differences）**：

```
n.x = (h[i-1][j] - h[i+1][j]) / (2 * dx)  // 水平梯度
n.y = 2.0   // 这个系数控制法线"陡峭感"
n.z = (h[i][j-1] - h[i][j+1]) / (2 * dz)  // 垂直梯度
normalize(n)
```

**直觉**：地面法线应该指"向上"（+Y 方向）。平坦地面的法线完全是 (0,1,0)。有坡度时，法线向坡面的逆方向倾斜。高度差越大，倾斜越明显。

`n.y = 2.0` 这个常量是个权衡——较大的值让法线更倾向垂直（地形感觉更平），较小的值让坡度感更强。

```cpp
Vec3 normal(int i, int j) const {
    float hL = heightAt(i-1,j), hR = heightAt(i+1,j);
    float hD = heightAt(i,j-1), hU = heightAt(i,j+1);
    float dx = size/(N-1);  // 网格间距（世界空间）
    Vec3 n{ (hL-hR)/(2*dx), 2.0f, (hD-hU)/(2*dx) };
    return n.normalized();
}
```

### 2.4 生物群系着色：高度 + 坡度双维度映射

单纯按高度着色会显得假——山坡和峭壁是不同材质，但高度可能相同。加入**坡度**（slope = 1 - n.y）作为第二维度：

```cpp
Vec3 biomeColor(int i, int j) const {
    float h    = heightAt(i,j) / heightScale;  // 归一化高度 [0, 1]
    Vec3 n     = normal(i,j);
    float slope = 1.f - n.y;  // 0=水平面, 1=垂直面

    // 分层判断（从高到低）
    if(h > 0.72f) {  // 雪线以上
        float t = (h - 0.72f) / 0.12f;
        return lerp(rock_color, snow_color, t);  // 岩石→雪
    }
    if(slope > 0.25f || h > 0.50f) {  // 陡坡或中高海拔→岩石
        return lerp(dark_rock, light_rock, (h-0.4)/0.2);
    }
    if(h > 0.22f) { ... }  // 森林
    if(h > 0.14f) { ... }  // 沙滩过渡
    // 低于水线→海水颜色
}
```

**坡度检测 `slope > 0.25` 的意义**：
- 法线 Y 分量 = cos(θ)，θ 是坡角
- slope = 1 - cos(θ) ≈ θ²/2（小角度近似）
- slope = 0.25 ≈ θ ≈ 30°
- 超过 30° 的坡面显示为岩石而不是植被，这符合真实地貌

### 2.5 Phong 光照模型

经典的三分量光照：

```
color = material_color × (ambient + diffuse × max(0, N·L)) + specular_color × pow(max(0, N·H), shininess)
```

其中：
- `N` = 顶点法线（归一化）
- `L` = 指向光源的方向（归一化）
- `H` = 半程向量 = normalize(L + V)，V 是视线方向
- `ambient` = 0.15 × {1,1,1.2}，轻微偏蓝（模拟天光）
- `shininess` = 48，适中的镜面高光（岩石和水面有反光）

**为什么用半程向量 H 而不是反射向量 R？**

Blinn-Phong（使用 H）vs 原始 Phong（使用 R = 2(N·L)N - L）：
- R 是精确的反射方向，但计算成本高（需要 N·L 的乘法和两次向量运算）
- H 是近似，但视觉差异在 shininess < 100 时几乎不可见
- 更重要的是，Blinn-Phong 在掠射角（grazing angle）时**没有高光截断**，反而比原版 Phong 更物理正确

```cpp
auto shade = [&](const Vec3& n, const Vec3& bc, const Vec3& pos) -> Vec3 {
    float diff = std::max(0.f, n.dot(sunDir));
    
    Vec3 viewDir = (cam.pos - pos).normalized();
    Vec3 halfV   = (sunDir + viewDir).normalized();
    float spec   = std::pow(std::max(0.f, n.dot(halfV)), 48.f) * 0.20f;
    
    Vec3 col = bc * (ambient + lightColor*diff*0.90f) + lightColor*spec;
    
    // 大气雾（指数衰减）
    float dist = (pos - cam.pos).length();
    float fog  = std::exp(-dist * 0.025f);
    return col*fog + fogColor*(1-fog);
};
```

### 2.6 大气雾：指数衰减

`fog = exp(-dist * density)`

- density = 0.025，对应特征距离 1/0.025 = 40 单位
- 40 单位外的物体 fog=1/e≈0.37，也就是有 63% 的颜色被雾覆盖
- 远处地形渐渐融入天空色（{0.65, 0.75, 0.92}），创造景深层次

为什么是指数函数而不是线性？
- 真实大气的散射强度随距离指数积累（光子在每个微元被散射的概率相同）
- Beer-Lambert 定律：透射率 T = exp(-μ×d)，其中 μ 是散射系数

---

## ③ 实现架构

### 3.1 整体数据流

```
Perlin Noise (seed=42)
        ↓
  fBm(7 octaves)
        ↓
   高度图 heights[N×N]  →  worldPos(i,j)  →  3D世界坐标
        ↓                         ↓
   normal(i,j)              biomeColor(i,j)
        ↓                         ↓
   Phong 光照 shade()  ←  顶点颜色（光照 × 材质）
        ↓
   Camera::project()  →  屏幕坐标 (sx, sy, depth)
        ↓
   drawTriangle()  →  Image 深度缓冲 + 颜色写入
        ↓
   image.writePNG()  →  PPM 文件  →  Python PIL 转 PNG
```

### 3.2 关键数据结构

**Terrain（地形数据）**：
```cpp
struct Terrain {
    int N;               // 网格分辨率（200×200）
    float size;          // 世界空间尺寸（12×12）
    float heightScale;   // 高度缩放（3.5）
    std::vector<float> heights;  // 高度图 N×N 个浮点值
    PerlinNoise pn;      // 噪声生成器
};
```

高度图是核心，所有派生量（法线、颜色、3D坐标）都按需计算，不提前存储。这种**惰性求值**策略节省内存，代价是每次渲染都要重新计算法线——对离线软光栅来说这是合理权衡。

**Image（帧缓冲）**：
```cpp
struct Image {
    int w, h;
    std::vector<Vec3> pixels;  // 颜色缓冲
    std::vector<float> depth;  // 深度缓冲（z-buffer）
};
```

深度缓冲初始化为 `1e18f`（无穷远），每个三角形的片元只有深度小于当前值才会写入，自动处理遮挡。

**Camera（相机）**：
```cpp
struct Camera {
    Vec3 pos, target, up;  // 位置、目标点、上向量
    float fovY, aspect;    // 垂直视角、宽高比
    
    Vec4 project(const Vec3& world, int W, int H) const;
};
```

### 3.3 渲染循环：quad → 两个三角形

地形网格每个格子（quad）被拆成两个三角形：

```
v00 --- v10
 |  \    |
 |   \   |
v01 --- v11

三角形1: v00, v10, v01
三角形2: v10, v11, v01
```

这种对角线分割方式（右上到左下）在大多数情况下视觉效果较好，避免明显的"Z字形"伪影。

---

## ④ 关键代码解析

### 4.1 透视投影的正确实现

这是整个项目最容易出错的地方（也是我实际踩到的坑，见踩坑实录）。

**View Space 变换**：把世界空间的点转换到相机空间。相机坐标系由三个正交向量定义：

```cpp
Vec4 project(const Vec3& world, int W, int H) const {
    // 构建相机坐标系
    Vec3 f  = forward();             // 前方向（归一化）
    Vec3 r  = forward().cross(up).normalized();  // 右方向
    Vec3 u2 = r.cross(forward());    // 实际上方向（与 up 可能不完全相同）
    
    // 世界点相对于相机的偏移
    Vec3 d = world - pos;
    
    // 投影到相机坐标系三个轴
    float vx = d.dot(r);   // 相机空间 X（向右为正）
    float vy = d.dot(u2);  // 相机空间 Y（向上为正）
    float vz = d.dot(f);   // 相机空间 Z（向前为正！！）
    
    // 裁剪：深度小于近平面则丢弃
    if(vz < near_) return {0,0,0,-1};
    
    // 透视除法：除以深度 × tan(fov/2)
    float tanH = std::tan(fovY*0.5f * M_PI/180.f);
    float px = vx / (vz * tanH * aspect);  // NDC X ∈ [-1, 1]
    float py = vy / (vz * tanH);           // NDC Y ∈ [-1, 1]
    
    // NDC → 屏幕像素
    int sx = (int)((px + 1.f) * 0.5f * W);
    int sy = (int)((1.f - (py + 1.f)*0.5f) * H);  // Y 轴翻转！
    return {(float)sx, (float)sy, vz, 1.f};
}
```

**注意 `py → sy` 的翻转**：NDC 中 Y 轴向上为正，图像中 Y 轴向下为正（左上角是(0,0)）。公式 `(1 - (py+1)/2) * H` 完成这个映射：
- py=+1（顶部）→ sy=0（图像顶行）✓
- py=-1（底部）→ sy=H（图像底行）✓

### 4.2 重心坐标三角形光栅化

软光栅化的核心：对于屏幕上的每个像素，判断它是否在三角形内，并插值颜色和深度：

```cpp
bool bary(float ax,float ay, float bx,float by, float cx,float cy,
          float px,float py, float& u,float& v,float& w) {
    // 行列式（三角形有向面积的2倍）
    float d = (by-cy)*(ax-cx) + (cx-bx)*(ay-cy);
    if(std::abs(d) < 1e-6f) return false;  // 退化三角形
    
    // 各分量的重心坐标
    u = ((by-cy)*(px-cx) + (cx-bx)*(py-cy)) / d;
    v = ((cy-ay)*(px-cx) + (ax-cx)*(py-cy)) / d;
    w = 1.f - u - v;
    
    // 像素在三角形内：所有分量 ≥ 0
    return u >= -1e-4f && v >= -1e-4f && w >= -1e-4f;
}

void drawTriangle(Image& img, Vec4 p0, Vec4 p1, Vec4 p2,
                  Vec3 c0, Vec3 c1, Vec3 c2) {
    // 计算包围盒，限制扫描范围
    int minX = max(0, floor(min(p0.x, p1.x, p2.x)));
    int maxX = min(W-1, floor(max(p0.x, p1.x, p2.x)) + 1);
    // ... 同理 minY, maxY
    
    for(int y=minY; y<=maxY; y++) {
        for(int x=minX; x<=maxX; x++) {
            float u, v, w;
            // 注意：用像素中心 (x+0.5, y+0.5) 做测试
            if(!bary(p0.x,p0.y, p1.x,p1.y, p2.x,p2.y, 
                     x+.5f, y+.5f, u,v,w)) continue;
            
            // 重心插值深度
            float depth = u*p0.z + v*p1.z + w*p2.z;
            
            // 深度测试（Z-buffer）
            if(depth >= img.dep(x,y)) continue;
            img.dep(x,y) = depth;
            
            // 重心插值颜色
            Vec3 col = c0*u + c1*v + c2*w;
            img.set(x, y, col);
        }
    }
}
```

**为什么用 `x+0.5f`？** 像素代表的是一个有面积的方块，其中心在 (x+0.5, y+0.5)。用中心做采样测试避免了整数边界的歧义。

**`-1e-4f` 的容差**：浮点精度问题会导致恰好在三角形边上的像素被拒绝（0 变成了极小的负数）。加一点容差避免缝隙。

### 4.3 天空渐变 + 太阳光晕

```cpp
for(int y=0; y<H; y++){
    float t = (float)y/H;  // 0=顶部, 1=底部
    Vec3 topSky  {0.20f, 0.40f, 0.75f};  // 深蓝（天顶）
    Vec3 horizSky{0.75f, 0.82f, 0.95f};  // 淡蓝（地平线）
    Vec3 sunGlow {0.95f, 0.80f, 0.60f};  // 暖橙（太阳方向）
    
    Vec3 skyCol = topSky*(1-t) + horizSky*t;
    
    // 地平线附近（t > 0.7）叠加暖色调
    if(t > 0.7f) {
        float g = (t - 0.7f) / 0.3f;
        skyCol = skyCol*(1 - g*0.4f) + sunGlow*(g*0.4f);
    }
    for(int x=0; x<W; x++) img.set(x, y, skyCol);
}
```

**设计考量**：天空是全屏背景，地形三角形会覆盖天空区域。这种方式相当于在清屏时已经画好背景，比单独处理"未命中地形的像素"更简单。

### 4.4 后处理：饱和度增强

C++ 输出的 PPM 颜色偏淡（因为雾效积累），用 Python PIL 做一次饱和度 +50% 的后处理：

```python
from PIL import Image
import colorsys
import numpy as np

img = Image.open('terrain_output.ppm')
pixels = np.array(img).astype(float) / 255.0

boosted = np.zeros_like(pixels)
for y in range(pixels.shape[0]):
    for x in range(pixels.shape[1]):
        r, g, b = pixels[y, x]
        h, s, v = colorsys.rgb_to_hsv(r, g, b)
        s = min(s * 1.5, 1.0)  # 饱和度 ×1.5
        v = min(v * 1.15, 1.0) # 亮度 ×1.15
        boosted[y, x] = colorsys.hsv_to_rgb(h, s, v)

result = (np.clip(boosted, 0, 1) * 255).astype(np.uint8)
Image.fromarray(result, 'RGB').save('terrain_output.png')
```

---

## ⑤ 踩坑实录

### Bug 1：透视投影完全没有输出

**症状**：程序运行正常（0 warnings），但输出图片全是天空色，没有任何地形三角形。

**错误假设**：以为是相机朝向或位置设置错误，尝试了十几个相机参数组合，始终没有三角形被渲染。

**真实原因**：投影函数中的符号错误。原代码：
```cpp
float vz = -d.dot(f);  // ❌ 多了一个负号
```
我沿用了"OpenGL 风格"惯例：视图空间 Z 轴向后（相机看向 -Z）。在 OpenGL 中，`vz = -dot(d, forward)` 是正确的，因为 forward 定义为 -Z 方向。但我的代码中 `forward()` 返回的是从相机指向目标的方向（+Z），所以不需要取反。

**结果**：所有在相机前方的点，其 `vz` 计算出来是负数，被 `if(vz < near_) skip` 全部丢弃。

**修复**：
```cpp
float vz = d.dot(f);   // ✅ 去掉负号，前方点 vz > 0
```

**经验**：在实现坐标变换时，必须明确每个坐标系的**轴向约定**。我的系统用的是"右手系，Z 轴向前"，而 OpenGL 用"右手系，Z 轴向后"。混用约定是图形学新手最常见的错误。

**验证方法**：用 Python 手动计算几个已知世界坐标的点的 `vz` 值，确认在相机前方的点 `vz > 0`，然后再跑完整渲染：

```python
d = (0-0, 0-8, 0-(-8))  # target point at (0,0,0), camera at (0,8,-8)
forward = (0, -0.625, 0.781)  # normalized direction
vz = d[0]*forward[0] + d[1]*forward[1] + d[2]*forward[2]
# = 0 + (-8)*(-0.625) + 8*0.781 = 5 + 6.25 = 11.25  ✓ 正值
```

### Bug 2：地形渲染出来但颜色单调

**症状**：修复 Bug 1 后，地形出现了，但整体偏灰偏蓝，生物群系颜色几乎看不出差异。

**原因**：
1. 雾效密度 0.04 太高，远处地形（8-15 单位距离）被雾完全遮盖
2. 初始 heightScale 只有 2.2，地形起伏相对地形尺寸（12×12）太小
3. 相机角度太高（俯视 45°+），导致大部分画面是天空

**修复**：
- `heightScale`: 2.2 → 3.5（高度增加约 59%）
- 雾密度: 0.04 → 0.025（减弱雾效，让远景更清晰）
- 相机从正上方俯视改为低角度斜视（更像第一人称视角）

### Bug 3：输出 PNG 只有 2.9KB（图像内容极少）

**症状**：第一次成功生成 PNG 时文件只有 2.9KB（正常应 100KB+）。

**原因**：只有 100 个不同颜色（PNG 压缩极好），因为当时大部分像素都是天空色（Bug 1 的副作用），几乎没有地形三角形覆盖。

**识别方法**：
```python
print(len(np.unique(pixels.reshape(-1,3), axis=0)))
# 输出: 100  ← 正常渲染应该 > 5000
```

修复 Bug 1 后，独特颜色数从 100 → 17318，文件从 2.9KB → 251KB。

### Bug 4：后处理饱和度增强耗时过长

**症状**：Python 的像素级循环（双重 for loop）处理 800×600 图片需要 30+ 秒。

**修复**：这是开发阶段可以接受的，但记录下来：正确做法是用 numpy 向量化操作：

```python
# 慢（像素循环）
for y in range(H):
    for x in range(W):
        h,s,v = rgb_to_hsv(...)

# 应该用 numpy 批处理（改进建议）
from PIL import ImageEnhance
enhancer = ImageEnhance.Color(img)
enhanced = enhancer.enhance(1.5)  # 更快
```

---

## ⑥ 效果验证与数据

### 量化指标

```
像素均值:    178.1（∈ [5, 250] ✓）
标准差:      46.9  （> 5 ✓）
文件大小:    251KB（> 10KB ✓）
独特颜色数:  17,318
绿色像素占比: 46.2%（草地/森林区域）
蓝色像素占比: 48.9%（天空区域）
天空蓝色偏差(B-R): +140.5（上半部分，天空在上 ✓）
地形绿色偏差(G-B): +71.3（下半部分，地形在下 ✓）
```

### 渲染性能

```
编译: g++ -O2, 0 errors 0 warnings
运行时间: 0.042 秒（实时帧，200×200 网格，800×600 分辨率）
```

0.042 秒对于纯 CPU 软光栅来说相当快。这归功于：
- step=2 的网格简化（实际渲染 100×100 个 quad）
- O2 优化开启
- 简单的 BBox 扫描线（不是全屏扫描）

### 可视化验证

渲染结果包含：
- 深蓝色天顶 → 浅蓝地平线 → 暖橙太阳光晕的天空渐变
- 绿色草地/森林覆盖的丘陵地形
- 深色区域对应坡度较大的岩石面
- 指数雾效在远处地形上产生轻微朦胧感

---

## ⑦ 总结与延伸

### 今天学到了什么

1. **Perlin Noise 的哈希表设计**：置换表（permutation table）是一个简洁的哈希技巧，256个元素 + 复制一份 = 512个元素，避免所有取模运算
2. **fBm 参数直觉**：persistence 控制"崎岖程度"，lacunarity 控制"细节频率"，octaves 控制"细节层级数"
3. **坐标系约定的重要性**：每个子系统（世界、相机、NDC、屏幕）都有独立的坐标约定，混用是灾难
4. **软光栅的调试方法**：当看不到任何渲染结果时，先用 Python 手算几个顶点投影，确认投影函数正确

### 技术局限性

1. **无透视插值修正**：当前的重心坐标插值在大三角形上会产生"仿射扭曲"，应做透视除法修正（除以 1/w 后再插值）
2. **无光源阴影**：地形本身没有阴影投射，山的背光面只依靠 Phong 漫反射（N·L < 0 时为黑），不够真实
3. **法线不光滑**：相邻三角形的法线不共享（每个顶点只按高度差计算），会有轻微的分块感
4. **无 LOD**：当前 step=2 是全局固定的，真实 LOD 应该根据相机距离动态调整网格密度
5. **无背面剔除**：当前渲染了所有朝向的三角形，应该跳过面朝背面的三角形（N·ViewDir < 0）

### 可优化方向

1. **Brunton LOD（CDLOD）**：Florian Bösch 的 Continuous Distance-Dependent LOD 是地形渲染的工业标准，用 quadtree 动态调整网格分辨率
2. **噪声函数改进**：Simplex Noise（同是 Ken Perlin 设计）在高维度比 Perlin 更快，且没有轴对齐方向上的视觉伪影
3. **侵蚀模拟**：Hydraulic erosion（水力侵蚀）算法可以在噪声地形上模拟真实的河谷、山脊
4. **大气散射集成**：上一次实现了 Rayleigh + Mie 散射大气渲染，今天的地形场景可以完美集成
5. **GPU 加速**：相同的渲染管线用 GLSL Compute Shader 实现，可以达到实时渲染（60fps+ 对百万面地形）

### 与本系列的关联

这个项目建立在之前多个主题的基础上：
- **大气散射（04-04）**：今天的天空是简化版，可以集成完整的 Rayleigh-Mie 大气
- **SDF Ray Marching（04-01）**：Ray Marching 方法也可以渲染隐式地形，与今天的显式高度图方法形成对比
- **BVH 加速（04-02）**：地形三角形数量大时，BVH 可以加速可见性判断
- **Disney BRDF（04-03）**：地形材质（岩石、雪、草）可以用 PBR 参数化，替换今天的固定颜色映射

---

*代码仓库：[github.com/chiuhoukazusa/daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-06-Procedural-Terrain-Renderer)*
