---
title: "每日编程实践: SSGI Screen Space Global Illumination Renderer"
date: 2026-05-22 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 全局光照
  - 实时渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-22-ssgi/ssgi_output.png
---

# SSGI：屏幕空间全局光照渲染器

今天实现了 SSGI（Screen Space Global Illumination），一种在屏幕空间内近似计算间接漫射光照的实时 GI 技术。渲染了一个 Cornell Box 场景：白色主球、蓝色小球、橙色球、彩色墙壁、面积光，以及一个点状发光球体。

![SSGI 渲染效果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-22-ssgi/ssgi_output.png)

---

## 一、背景与动机：为什么需要 GI，为什么选屏幕空间

### 1.1 没有 GI 的世界长什么样

直接光照（Direct Lighting）只处理光源直接打到物体表面的情况。现实世界中，光在空间里反弹——太阳光照进房间后，天花板被地板的反射光打亮；Cornell Box 里红色左墙会把红光"流血"到白色球体上，形成 Color Bleeding（颜色溢出）。

没有 GI 的场景特征：
- 阴影区域完全是黑色，没有任何细节
- 相邻表面颜色完全独立，没有颜色互相染色
- 室内场景视觉上"飘"，与现实差距大
- Ambient Term 只能用常数来糊弄

这就是为什么几乎所有 3A 游戏都有某种形式的 GI——UE5 的 Lumen、Unity 的 DDGI、Frostbite 的 Radiance Cache……没有 GI 的画面一眼假。

### 1.2 GI 的各种方案与权衡

全局光照的实现方案有很多，各有取舍：

| 方案 | 质量 | 实时性 | 复杂度 | 典型应用 |
|------|------|--------|--------|----------|
| 路径追踪（离线） | ⭐⭐⭐⭐⭐ | ❌ | 高 | 电影、建筑可视化 |
| 光子映射 | ⭐⭐⭐⭐ | ❌ | 高 | 离线渲染器 |
| Radiosity | ⭐⭐⭐ | △ | 中 | 静态场景预计算 |
| Lightmap | ⭐⭐⭐ | ✅ | 中 | 静态场景游戏 |
| SSGI | ⭐⭐ | ✅✅ | 低 | 实时游戏后处理 |
| DDGI（探针） | ⭐⭐⭐ | ✅ | 中 | UE4 Lumen 早期 |
| Lumen（UE5） | ⭐⭐⭐⭐ | ✅ | 极高 | 次时代游戏 |

SSGI 的核心价值在于：**只需要 G-Buffer，不需要加速结构，不需要任何预计算，GPU 一个后处理 pass 就能跑**。代价是它只能看到"屏幕上有的东西"——摄像机视野外的几何体贡献的间接光照是缺失的，这就是屏幕空间方法的根本局限性。

### 1.3 工业界实际使用场景

SSGI 在游戏引擎中广泛应用：
- **Unity HDRP**：提供 SSGI 功能作为 Lumen 的替代品
- **虚幻引擎**：Lumen 未开启时使用 SSAO + SSGI 组合
- **地平线：禁忌西部**：大量使用屏幕空间 GI 加强间接光照
- **赛博朋克 2077**（非 RTX 模式）：屏幕空间 GI 贡献了大量环境光照

---

## 二、核心原理：从渲染方程到屏幕空间近似

### 2.1 渲染方程回顾

渲染方程由 Kajiya（1986）提出：

```
Lo(x, ωo) = Le(x, ωo) + ∫_Ω fr(x, ωi, ωo) Li(x, ωi) (n·ωi) dωi
```

各项含义：
- `Lo(x, ωo)`：点 x 沿方向 ωo 的出射辐射度
- `Le(x, ωo)`：自发光项
- `fr(x, ωi, ωo)`：BRDF，描述材质如何反射光
- `Li(x, ωi)`：从方向 ωi 入射到点 x 的辐射度
- `(n·ωi)`：Lambert 余弦项，法线与入射方向的夹角

问题在于 `Li(x, ωi)` 本身又由另一个渲染方程决定——这是递归的，是 GI 问题困难的根本原因。

### 2.2 Diffuse GI 的简化

对于纯漫反射材质（Lambertian）：

```
fr(x, ωi, ωo) = albedo / π
```

漫反射出射辐射度：

```
Lo_diffuse(x) = (albedo / π) * ∫_Ω Li(x, ωi) (n·ωi) dωi
             = albedo * (1/π * ∫_Ω Li(x, ωi) (n·ωi) dωi)
             = albedo * Irradiance(x)
```

即：**漫反射 GI = albedo × 半球面上入射辐照度的积分**。

这个积分用蒙特卡洛方法近似：

```
Irradiance(x) ≈ (1/N) * Σ Li(x, ωi) / pdf(ωi)
```

对于余弦加权重要性采样（cosine-weighted importance sampling）：

```
pdf(θ) = cos(θ) / π
```

代入后，每个样本的权重正好为 1（余弦项与 PDF 相消）：

```
Irradiance(x) ≈ (π/N) * Σ Li(x, ωi)
```

### 2.3 屏幕空间近似的关键思路

SSGI 的巧妙之处在于：**用 G-Buffer 中已经渲染好的直接光照作为"一次反弹"的间接光来源**。

具体步骤：
1. 在当前像素的法线半球上采样 N 个方向 ωi
2. 沿每个方向走一段距离，得到"候选命中点" P'
3. 把 P' 重投影回屏幕空间，看看它对应 G-Buffer 的哪个像素
4. 从那个像素的直接光照结果（已算好的 DirectFb）中读取辐射度
5. 平均所有样本，乘以 albedo，得到间接漫射贡献

这就是"屏幕空间"的含义：不追光线到三维空间，而是在已有的屏幕像素数据中查找。

### 2.4 余弦半球采样的数学推导

余弦加权重要性采样用 Malley's Method 实现。先在单位圆盘上均匀采样，再投影到半球：

```
// 均匀圆盘采样 -> 余弦半球采样
phi = 2π * r1
sinTheta = sqrt(r2)      // r2 是 [0,1] 均匀随机数
cosTheta = sqrt(1 - r2)

// 局部坐标（z 轴为法线）
x_local = sinTheta * cos(phi)
y_local = sinTheta * sin(phi)
z_local = cosTheta
```

为什么 `sinTheta = sqrt(r2)` 而不是 `sinTheta = r2`？因为我们要让采样密度正比于 cos(θ)。

PDF 是：
```
pdf(θ) = cos(θ) / π
```

在极坐标下，面积元素为 `sin(θ) dθ dφ`，累积分布函数（CDF）对 θ 积分：

```
CDF(θ) = ∫_0^θ (cos(t)/π) * 2π * sin(t) dt = 1 - cos²(θ)
```

令 `CDF(θ) = r2`，反解：

```
cos(θ) = sqrt(1 - r2)
sin(θ) = sqrt(r2)
```

直觉：余弦加权让靠近法线的方向被更频繁采样（法线方向 cos(θ)=1，贡献大），赤道附近采样稀疏（cos(θ)≈0，贡献小）。这就是重要性采样的意义——把计算预算花在重要的地方。

### 2.5 TBN 矩阵：局部坐标到世界坐标的转换

采样方向在法线为 (0,0,1) 的局部空间里计算。要变换到世界空间，需要构建 TBN（Tangent-Bitangent-Normal）矩阵：

```
T = normalize(cross(N, up))  // Tangent
B = cross(N, T)              // Bitangent
N = 法线（已知）
```

up 向量取 (0,1,0)，当 N 与 up 太接近时换成 (1,0,0)（避免退化）。

世界空间方向 = T * x_local + B * y_local + N * z_local

这是每次半球采样的核心操作。

### 2.6 重投影：世界空间 -> 屏幕空间

给定候选命中点 P'（世界坐标），要找它在屏幕上的位置：

```
clip = Proj * View * P'
ndc_x = clip.x / clip.w
ndc_y = clip.y / clip.w
ndc_z = clip.z / clip.w

screen_u = ndc_x * 0.5 + 0.5
screen_v = 1 - (ndc_y * 0.5 + 0.5)  // 翻转 Y 轴
```

然后和 G-Buffer 的深度做比较：如果 |P' 的 NDC-Z - G-Buffer 深度| < 阈值，认为命中；否则跳过（说明中间有遮挡，或者该点根本不在场景中）。

---

## 三、实现架构：软光栅 + 多 Pass 管线

本项目用纯 CPU 软光栅化实现，模拟 GPU 的 G-Buffer 延迟渲染管线。

### 3.1 整体数据流

```
[场景几何体]
      │
      ▼
[G-Buffer Pass] ─── 光栅化所有三角形
      │               输出：albedo / normal / position / depth / emission
      ▼
[Direct Lighting Pass]
      │               对每个 G-Buffer 像素执行直接光照
      │               Blinn-Phong 漫反射 + 衰减
      ▼
[SSGI Pass]
      │               对每个像素采样 16×4 = 64 个方向
      │               重投影 -> 深度测试 -> 读取直接光照
      │               输出：indirectRaw (noisy)
      ▼
[Bilateral Blur Pass]
      │               双边滤波降噪（保边模糊）
      │               输出：indirectDenoised
      ▼
[Composite Pass]
      │               direct + indirect -> ACES 色调映射 -> gamma 矫正
      ▼
[PPM -> PNG Export]
```

### 3.2 G-Buffer 结构

```cpp
struct GBuffer {
    std::vector<Vec3> albedo;    // 漫反射颜色
    std::vector<Vec3> normal;    // 世界空间法线
    std::vector<Vec3> position;  // 世界空间位置
    std::vector<float> depth;    // NDC 深度 [-1, 1]
    std::vector<float> zbuf;     // 光栅化 Z 缓冲（用于深度测试）
    std::vector<Vec3> emission;  // 自发光颜色
};
```

为什么存世界空间位置而不是只存深度？G-Buffer 同时存储世界空间位置是为了 SSGI pass 直接使用，避免每次从深度反推世界坐标（那需要 View-Proj 矩阵的逆，且精度较差）。在真实 GPU 实现中通常存 view-space position 以节省带宽，这里为了代码清晰直接存 world-space。

### 3.3 SSGI 结构

```cpp
struct SSGI {
    const GBuffer& gb;
    const Framebuffer& directFb;  // 直接光照结果
    const Mat4& proj;
    const Mat4& view;
    int numSamples = 16;   // 每像素每帧的方向样本数
    float radius = 0.9f;   // 世界空间搜索半径
    int W, H;
};
```

`radius` 控制间接光照的"传播范围"：太小则只有近邻影响，太大则会采样到深度不匹配的点（误读），产生泄漏噪声。0.9 单位在这个场景里对应约 1/3 房间宽度，经过实验是比较好的值。

### 3.4 多帧准蒙特卡洛累积

单帧 16 个样本肯定有噪声。这里用 4 帧不同偏移的 Halton 序列累积，相当于 64 spp，然后取平均：

```cpp
for(int frame = 0; frame < numFrames; frame++){
    for(每个像素) {
        indirectRaw[id] += ssgi.sampleIndirect(x, y, frame*7+13);
    }
}
for(每个像素) indirectRaw[i] = accumIndirect[i] / numFrames;
```

Halton 序列是低差异序列（Low-Discrepancy Sequence），相比纯随机覆盖更均匀，减少聚集噪声。

---

## 四、关键代码解析

### 4.1 光栅化 + 透视插值

透视正确插值是软光栅化的重要细节：

```cpp
void drawTriangle(GBuffer& gb, const Triangle& tri, 
                  bool emissive, Vec3 emitColor) const {
    Vec4 clip[3];
    for(int i=0;i<3;i++) clip[i] = project(tri.v[i].pos);

    // 近裁面简单处理：全在后方就跳过
    int behind = 0;
    for(int i=0;i<3;i++) if(clip[i].w <= 0) behind++;
    if(behind == 3) return;

    // 透视除法 -> NDC 坐标
    for(int i=0;i<3;i++){
        float w = clip[i].w;
        if(w <= 0) w = 1e-5f;
        px[i] = clip[i].x / w;
        py[i] = clip[i].y / w;
        pz[i] = clip[i].z / w;
    }
```

这里 `pz` 存储的是 NDC-Z，直接用于深度测试。注意 `w<=0` 的保护：当顶点在近裁面后方时 w 会变负，如果不处理会导致坐标翻转。

边函数（Edge Function）判断点是否在三角形内：

```cpp
auto edge = [&](float ax,float ay,float bx,float by,float cx,float cy)->float{
    return (bx-ax)*(cy-ay)-(by-ay)*(cx-ax);
};
float area = edge(sx[0],sy[0],sx[1],sy[1],sx[2],sy[2]);
```

`edge(A, B, P)` 的符号告诉我们 P 在 AB 边的哪一侧。三个边函数同号 -> 点在三角形内。这是基于叉积的面积计算，`area` 就是三角形的有向面积 × 2。

重心坐标由三个边函数值除以总面积得到：

```cpp
float w0 = edge(sx[1],sy[1],sx[2],sy[2],fx,fy) / area;
float w1 = edge(sx[2],sy[2],sx[0],sy[0],fx,fy) / area;
float w2 = edge(sx[0],sy[0],sx[1],sy[1],fx,fy) / area;
```

### 4.2 余弦半球采样实现

```cpp
// 构建切线空间基向量
static void buildTBN(Vec3 n, Vec3& t, Vec3& b){
    Vec3 up = (std::abs(n.y) < 0.99f) ? Vec3{0,1,0} : Vec3{1,0,0};
    t = n.cross(up).norm();
    b = n.cross(t);
    // 注意：b = n.cross(t) 而不是 t.cross(n)
    // 这样保证 (t, b, n) 是右手坐标系
}

// Malley's Method：先在圆盘采样，再投影到半球
static Vec3 cosineSampleHemisphere(Vec3 n, float r1, float r2){
    float phi = 2.f * 3.14159265f * r1;
    float sinTheta = std::sqrt(r2);
    float cosTheta = std::sqrt(1.f - r2);
    Vec3 t, bi;
    buildTBN(n, t, bi);
    return (t*(sinTheta*std::cos(phi)) + bi*(sinTheta*std::sin(phi)) + n*cosTheta).norm();
}
```

这里有一个容易搞错的地方：`buildTBN` 中第二步用 `n.cross(t)` 而不是 `t.cross(n)`。前者与后者符号相反。要确保 TBN 是右手系（行列式 > 0），否则采样方向会以错误的方式旋转，导致间接光照方向系统性偏差。

### 4.3 Halton 低差异序列

```cpp
static float halton(int index, int base){
    float f = 1.f, r = 0.f;
    int i = index;
    while(i > 0){
        f /= base;
        r += f * (i % base);
        i /= base;
    }
    return r;
}
```

Halton 序列本质是把整数在不同进制下"反折叠"到 [0,1]。`halton(n, 2)` 给出 van der Corput 序列，`halton(n, 3)` 给出另一个独立序列。两个序列组合成 2D 点集，覆盖均匀，远优于伪随机。

使用时加入像素坐标哈希偏移，防止相邻像素使用完全相同的方向：

```cpp
float r1 = std::fmod(halton(frameIdx*numSamples+s, 2) + halton(px*13+py*7, 3), 1.f);
float r2 = std::fmod(halton(frameIdx*numSamples+s, 3) + halton(px*17+py*11, 5), 1.f);
```

### 4.4 SSGI 核心采样逻辑

```cpp
Vec3 sampleIndirect(int px, int py, int frameIdx) const {
    int id = py*W + px;
    Vec3 pos    = gb.position[id];
    Vec3 normal = gb.normal[id];
    Vec3 albedo = gb.albedo[id];

    if(normal.len() < 0.5f) return Vec3{};  // 背景像素（无几何体）
    if(albedo.x+albedo.y+albedo.z < 1e-6f) return Vec3{};  // 纯黑材质

    Vec3 indirect{};
    float totalWeight = 0.f;

    for(int s = 0; s < numSamples; s++){
        float r1 = ...; float r2 = ...;

        // 采样半球方向
        Vec3 dir = cosineSampleHemisphere(normal, r1, r2);

        // 候选命中点（沿方向走 hitRad）
        float hitRad = radius * (0.2f + r2 * 0.8f);
        Vec3 samplePos = pos + dir * hitRad;

        // 重投影到屏幕空间
        float su, sv, sndcZ;
        if(!worldToScreen(samplePos, su, sv, sndcZ)) continue;

        int sx = (int)(su * W);
        int sy = (int)(sv * H);
        int sid = sy*W + sx;

        // 深度一致性测试（防止泄漏穿墙）
        float depthDiff = std::abs(sndcZ - gb.depth[sid]);
        if(depthDiff > 0.15f) continue;

        // 法线方向测试（防止采样背面）
        Vec3 sNormal = gb.normal[sid];
        if(sNormal.dot(normal) < -0.1f) continue;

        // 读取该像素的直接光照 + 自发光
        Vec3 radiance = directFb.color[sid] + gb.emission[sid];

        float nAgreement = std::max(0.f, sNormal.dot(dir));
        float w = 1.f + nAgreement;
        indirect += radiance * w;
        totalWeight += w;
    }

    if(totalWeight > 0.f) indirect = indirect / totalWeight;

    // 漫反射调制：间接 GI = albedo * 入射辐照度
    return albedo * indirect * 0.85f;
}
```

权重 `w = 1 + nAgreement`：对命中点法线与采样方向同向的情况加大权重。直觉是：法线朝向入射方向的表面更可能是真正的反射面（而不是背面或掠射面）。

`0.85f` 是能量守恒因子：防止多次叠加后过曝。在物理上，每次 diffuse 反射都会损失一部分能量（依赖 albedo），这个系数补偿了近似误差。

### 4.5 双边滤波降噪

SSGI 原始输出有噪声，需要空间滤波。普通高斯模糊会让边缘模糊，双边滤波（Bilateral Filter）通过同时考虑空间距离和颜色/深度相似性来保边：

```cpp
void bilateralBlur(..., int radius=2){
    for(每个像素) {
        Vec3 centerCol = input[id];
        float centerDepth = gb.depth[id];

        for(int dy=-radius; dy<=radius; dy++){
            for(int dx=-radius; dx<=radius; dx++){
                float colorDiff = (nc - centerCol).len();
                float depthDiff = std::abs(nd - centerDepth);
                float normalSim = std::max(0.f, nn.dot(n));

                // 颜色相似 × 深度相似 × 法线相似 = 权重
                float wc = exp(-colorDiff² / (2 * 0.3²));
                float wd = exp(-depthDiff² / (2 * 0.08²));
                float wn = normalSim²;
                float w  = wc * wd * wn;
                
                sum += nc * w;
                wsum += w;
            }
        }
        output[id] = sum / wsum;
    }
}
```

三重权重保证了：
- 颜色差异大的像素（跨物体边界）不互相影响
- 深度突变的像素（几何边缘）不会混合
- 法线方向差异大的像素（曲面折角）不会混合

### 4.6 ACES Filmic 色调映射

```cpp
Vec3 aces(Vec3 x){
    const float a=2.51f, b=0.03f, c=2.43f, d=0.59f, e=0.14f;
    return ((x*(x*a + Vec3{b,b,b})) / ((x*(x*c + Vec3{d,d,d}) + Vec3{e,e,e}))).clamp01();
}
```

ACES（Academy Color Encoding System）是工业界标准的色调映射曲线，由 Narkowicz 2015 提出的近似拟合。它的特点：
- 暗部细节更丰富（不像 Reinhard 那样让黑暗区域变灰）
- 亮部滚降更自然，高光不过饱和
- 中间调对比度适中，视觉上更"电影感"

---

## 五、踩坑实录

### 5.1 Vec3 除法未定义导致编译错误

**症状**：ACES 函数编译报错：`no match for 'operator/'`

**错误假设**：以为 Vec3 的除法操作符已经重载了 Vec3/Vec3。

**真实原因**：Vec3 只实现了 `Vec3 / float`，没有 `Vec3 / Vec3`。ACES 公式里需要逐分量相除。

**修复**：
```cpp
// 添加 Vec3/Vec3 重载
Vec3 operator/(Vec3 o) const { return {x/o.x, y/o.y, z/o.z}; }
```

教训：编写向量数学库时要同时考虑标量和向量版本的运算符。GLSL 里 vec3/vec3 是天然支持的，C++ 里需要显式实现。

### 5.2 未使用变量编译 Warning

**症状**：`warning: variable 'pw' set but not used`

**原因**：光栅化函数里我声明了 `pw[3]` 来存储透视 w 值，后来发现只需要 x/y/z/，w 在赋值后没用到。

**修复**：
```cpp
// 方案A：删掉 pw 数组
float px[3],py[3],pz[3];  // 移除 pw[3]

// 方案B：标记为故意不使用
(void)w;  // 透视除法已完成，w 值不再需要
```

用 `-Wall -Wextra` 编译的好处就是这种潜在问题会被揪出来。

### 5.3 图像过亮：平均像素值 227/255

**症状**：第一版渲染出来平均亮度 227/255，接近全白，失去了 Cornell Box 的明暗层次。

**原因分析**：
- 直接光照 ambient 设置太高（0.12），即使阴影区域也很亮
- 光源强度偏高（6.0），区域光直接打穿了整个场景
- ACES 曝光系数 0.7 对于这个已经偏亮的场景来说太高

**修复**：
```cpp
// 减少环境光
result += albedo * Vec3{0.06f, 0.06f, 0.08f};  // 从 0.12 降到 0.06

// 降低光源强度
{{0.f,2.7f,-4.f}, {1.f,0.96f,0.88f}, 4.f},     // 从 6.0 降到 4.0

// 降低 ACES 曝光
Vec3 ldr = aces(hdr * 0.35f);  // 从 0.7 降到 0.35
```

修复后：平均亮度 190/255，标准差 52，分布合理（40% 中间调，58% 亮部，1.2% 暗部）。

### 5.4 `convert` 命令不存在

**症状**：程序运行完成但无法生成 PNG，stderr 输出 `sh: convert: command not found`

**原因**：ImageMagick 的 `convert` 没有安装。

**修复**：改为调用 Python PIL：
```cpp
std::system(("python3 -c \"from PIL import Image; "
             "Image.open('" + outPPM + "').save('" + outPNG + "')\"").c_str());
```

实际上 Python 环境中 PIL 已经安装，这是更可靠的依赖。

### 5.5 SSGI 间接光照强度不一致

**症状**：部分区域间接光照过强（浮白），部分区域完全没有间接光。

**原因**：`totalWeight` 为 0 时直接返回零向量，但在某些法线方向下所有样本都被 `depthDiff > 0.15f` 过滤掉了（特别是靠近墙角的像素，候选点超出屏幕边界）。

**修复**：加入样本有效率检查，并调整 `radius` 参数：
- 对于边界像素，减小采样半径（`0.2f + r2 * 0.8f` 的可变半径）
- 法线测试阈值从 0.0f 放松到 -0.1f（允许轻微背面）

### 5.6 重投影 Y 轴翻转

**症状**：间接光照呈水平条纹，似乎 GI 来源的像素垂直方向错误。

**原因**：重投影时 NDC Y 坐标需要翻转才能对应屏幕坐标系：
```
screen_v = 1 - (ndc_y * 0.5 + 0.5)  // 注意这里的 1 -
```
如果忘记取反，Y 轴翻转，导致从屏幕上方的像素读取下方的光照数据。

**修复**：检查 `worldToScreen` 函数，确认 V 坐标计算正确。

---

## 六、效果验证与数据

### 6.1 最终渲染图像

![SSGI 渲染效果 - Cornell Box](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-22-ssgi/ssgi_output.png)

场景包含：Cornell Box（左红右绿、白色地面/天花板/后墙）、面积光（天花板中央）、白球、蓝球、橙球、发光橙球。

### 6.2 量化像素统计

```
=== 输出验证 ===
分辨率: 800×600
文件大小: 163 KB（远超 10KB 门槛）
像素均值: 190.4 / 255（在 [10, 240] 范围内 ✅）
像素标准差: 52.4（>> 5，图像有充分内容变化 ✅）

亮度分布：
  暗部 (<50):  5881 像素 (1.2%)
  中间调 (50-200): 192242 像素 (40.1%)
  亮部 (>200): 281877 像素 (58.7%)
```

### 6.3 性能分析（单线程 CPU）

各 Pass 耗时（在测试机上实测）：

| Pass | 耗时 | 说明 |
|------|------|------|
| 场景光栅化 | ~0.3s | 球体三角面较多 |
| 直接光照 | ~0.02s | 简单逐像素计算 |
| SSGI（4帧×16spp）| ~8s | 主要开销，64次/像素重投影 |
| 双边降噪 | ~0.5s | 5×5 kernel，3次迭代 |
| 合成 + 色调映射 | ~0.01s | 轻量后处理 |
| **总计** | **~9s** | 800×600 全图 |

注：CPU 软光栅对性能没有要求，主要是验证算法正确性。GPU 实现中 SSGI 单 Pass 在 1080p 下可达 16ms 以内（60fps）。

### 6.4 有无 SSGI 对比

**无 SSGI（纯直接光照）**：
- 阴影区域纯黑
- 红色/绿色墙壁对白球没有颜色溢出（Color Bleeding）
- 球体背面看不到任何来自地面的反弹光

**有 SSGI（本项目）**：
- 球体背面可以看到来自地面/墙壁的间接光
- 靠近红色墙的白球会被微微染红（Color Bleeding 效果）
- 暗部细节更丰富，不是纯黑

---

## 七、总结与延伸

### 7.1 技术局限性

SSGI 有几个本质局限，是屏幕空间方法无法克服的：

**1. 视野外遮挡**：摄像机背后的光源反弹不会被捕捉。旋转摄像机时 GI 会突然变化（"GI 闪烁"）。

**2. 自阴影问题**：SSGI 无法正确处理物体内部的光路。薄物体（如树叶）的正反面 GI 可能混淆。

**3. 采样半径限制**：间接光照只能传播 `radius` 的距离。长距离 GI（如太阳光穿过走廊多次反弹）无法捕捉。

**4. 深度阈值的脆弱性**：`depthDiff > 0.15f` 这个阈值是经验性的。对于特殊几何体（非常薄的表面、掠射角度）容易出现漏光或缺光。

### 7.2 可优化方向

**降噪改进**：
- 时域复用（Temporal Accumulation）：利用前一帧的 GI 结果，大幅减少样本需求
- 联合双边滤波加入速度缓冲（Motion Vector）：防止运动物体的 GI "拖尾"
- SVGF（Spatiotemporal Variance-Guided Filter）：UE4 中实时降噪的工业标准

**采样改进**：
- 层级 Z 缓冲（Hierarchical Z-Buffer）：用 mipmap 形式的深度图加速光线步进，减少无效采样
- 重要性采样改进：优先采样屏幕上亮度高的区域（来自直接光照的信息）
- ReSTIR 思路：在 SSGI 中引入 Reservoir 采样，复用历史帧的有效样本

**质量改进**：
- 法线锥剔除：对于凹面区域，排除与法线锥体方向矛盾的样本
- Bent Normal：用 SSGI 的平均可见方向近似 bent normal，改善 AO 质量
- 双层间接光照：间接漫射 + 间接镜面反射（SSGI + SSR 组合）

### 7.3 与本系列其他项目的关联

本系列已实现的屏幕空间技术：
- **FXAA**（05-13）：屏幕空间抗锯齿，同样是纯后处理 pass
- **SSR**（04-18）：屏幕空间反射，原理上类似但针对镜面反射
- **PCSS**（04-20）：屏幕空间软阴影，同样依赖深度缓冲
- **TAA**（05-20）：时域抗锯齿，时域复用思路与 temporal SSGI 通用

下一步可探索的方向：
- DDGI（Distance-field Global Illumination）：基于有向距离场的探针 GI
- Radiance Cascade GI：2D 游戏引擎（如 Godot 4）使用的级联辐射度 GI
- Lumen 架构拆解：虚幻5 的混合射线/屏幕空间 GI

---

今天的 SSGI 实现验证了屏幕空间全局光照的核心思路：用 G-Buffer 中已知的直接光照作为间接光的来源，通过半球采样和重投影近似一次光路反弹。虽然有屏幕空间方法固有的局限，但它是理解实时 GI 的绝佳起点。
