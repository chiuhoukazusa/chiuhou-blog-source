---
title: "每日编程实践: Deferred Shading Multi-Light Renderer（延迟渲染多光源实现）"
date: 2026-05-12 05:30:00
tags:
  - 每日一练
  - 图形学
  - 渲染管线
  - C++
  - 延迟渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-12-deferred-shading/deferred_shading_output.png
---

## 背景与动机

### 为什么需要延迟渲染？

在传统前向渲染（Forward Rendering）管线中，每个几何物体在光栅化时就直接计算光照。这在光源较少时运作良好，但随着场景复杂度提升，问题会迅速爆炸式增长：

**前向渲染的痛点**：

假设场景有 N 个物体、M 个光源，最坏情况下的光照计算复杂度是 O(N × M)。对于一个游戏场景——100 个可见物体、64 个点光源（街灯、粒子效果、爆炸等）——就需要 6400 次光照计算调用，其中绝大多数是浪费的（因为很多光源对某个物体根本没有影响，或者该像素会被后来绘制的物体遮挡）。

更严重的问题是"OverDraw（过度绘制）"：如果场景中的物体从前到后排列，远处的物体会先被绘制，然后被近处的物体覆盖，那些被覆盖的像素的光照计算就完全浪费了。

**延迟渲染的思路**：

延迟渲染（Deferred Shading）的核心洞见是：**把几何信息的收集和光照计算这两件事分开做**。

第一遍（Geometry Pass）：只收集每个像素的几何属性——位置、法线、材质颜色——存到一组"中间缓冲区"（G-Buffer，Geometry Buffer）里。这一遍完全不计算光照，只是一个几何信息收集器。

第二遍（Lighting Pass）：遍历每个屏幕像素，从 G-Buffer 读取几何信息，对所有光源计算光照贡献，最终输出颜色。

这样，每个屏幕像素最终只会被光照计算一次（因为 G-Buffer 里已经是最终可见的表面），彻底解决了 OverDraw 问题。

### 工业界使用现状

延迟渲染已经是 AAA 游戏引擎的标配技术：

- **Unreal Engine 4/5**：默认渲染路径就是延迟渲染，G-Buffer 包含 BaseColor、Metallic、Roughness、WorldNormal、CustomData 等多个 RT。Lumen GI 系统也构建在这套 G-Buffer 体系之上。
- **Unity HDRP**（High Definition Render Pipeline）：也采用延迟渲染，提供精细化的 G-Buffer 布局控制。
- **id Tech 7**（Doom Eternal）：使用延迟渲染处理大量动态光源和爆炸特效。
- **Frostbite**（Battlefield 系列）：延迟渲染 + Tile-Based Deferred Shading（TBDS）进一步优化。

可以说，凡是支持"场景中几十上百个动态光源"的现代游戏，背后几乎都是延迟渲染或其变体（比如 Tile-Based 或 Clustered Deferred）。

### 本次实现的目标

用纯 C++ 软光栅化实现一个延迟渲染管线：
- G-Buffer 包含 albedo、world-space normal、world-space position
- 场景：8 个彩色球体 + 地面，共 ~13856 个三角形
- 17 个点光源（16 个彩色环绕 + 1 个顶部主光）
- Blinn-Phong 着色 + 逐通道 Reinhard 色调映射

---

## 核心原理

### G-Buffer 的设计

G-Buffer 本质上是一组"MRT（Multiple Render Targets）"，在渲染硬件上就是多张贴图同时作为输出对象。在软件实现中，就是几个并行的数组：

```
G-Buffer 布局（每像素）：
  gAlbedo    : vec3  漫反射颜色（0~1 归一化）
  gNormal    : vec3  世界空间法线（单位向量）
  gPosition  : vec3  世界空间坐标（未裁剪，真实世界坐标）
  gDepth     : float NDC 深度（-1~1 映射到 z-buffer 值）
  gValid     : bool  该像素是否有有效几何（背景 = false）
```

为什么存世界空间坐标而不是视空间坐标？

视空间坐标依赖于相机矩阵，如果相机移动则所有 G-Buffer 数据都失效。世界空间坐标是绝对坐标，对于 TAA（时间反走样）、ReSTIR 等需要利用历史帧数据的算法，世界空间坐标更易于处理。当然，存储世界空间坐标也有缺点——精度问题（大世界坐标范围导致 float 精度损失），这在 UE5 的 World Partition 中需要特别处理（使用 double 或相对坐标）。

**法线压缩**：在 GPU 上，世界空间法线通常压缩存储。由于法线是单位向量，三个分量满足 `x²+y²+z²=1`，理论上只需存 x 和 y，z 可以通过 `z = sqrt(1 - x² - y²)` 恢复（假设 z ≥ 0，即法线朝相机方向）。这把 24 bytes/pixel 降到 16 bytes/pixel。本实现为了简单，直接存 vec3。

### 点光源的光照衰减模型

物理上光的强度随距离平方衰减（平方反比定律）：

```
E = I / d²
```

其中 E 是照度（到达表面的光强），I 是光源发光强度，d 是距离。

但纯平方反比有一个问题：当 d → 0 时，E → ∞，在光源内部会产生无限亮度。游戏引擎通常加一个常数项来防止这种情况：

```
attenuation = 1 / (k_c + k_l × d + k_q × d²)
```

其中 k_c（常数）、k_l（线性）、k_q（二次）是可调参数。常用值参考 Ogre/Phong 的经验数据表：
- 7米范围：k_l = 0.7,  k_q = 1.8
- 13米范围：k_l = 0.35, k_q = 0.44
- 50米范围：k_l = 0.09, k_q = 0.032（本实现使用）

此外，点光源还需要一个"影响半径"概念。物理上光照无限延伸，但对于实时渲染，超出一定距离的贡献已经微乎其微，继续计算是浪费。本实现在衰减基础上加了一个"边缘平滑截断"：

```cpp
float edgeFade = 1.0f - (dist / light.radius);
edgeFade = edgeFade * edgeFade; // 平滑截断（二次方）
attenuation *= edgeFade;
```

当 dist = radius 时 edgeFade = 0，完全截断。相比硬截断，二次方渐变避免了光照边缘的突变现象（否则你会看到圆形光照边界）。

### Blinn-Phong 着色模型

本实现使用经典的 Blinn-Phong 模型，分为三个部分：

**环境光（Ambient）**：

```
L_ambient = k_a × I_a
```

不依赖光源位置，是对全局间接光照的粗糙近似。本实现设定为 `(0.03, 0.03, 0.05)`，轻微偏蓝以模拟天空反射。

**漫反射（Diffuse，Lambert）**：

```
L_diffuse = k_d × (N · L) × I_light × attenuation
```

`N` 是表面法线（单位向量），`L` 是从表面指向光源的方向向量（单位向量）。点积 `N·L` 给出光线与表面的夹角余弦值：
- 正对光源（法线与光源方向同向）：`N·L = 1`，最亮
- 掠射（光线擦过表面）：`N·L ≈ 0`，趋于零
- 背光（表面朝向背离光源）：`N·L < 0`，夹死到 0（clamp）

这体现的物理直觉是：同样面积的表面，在光线正射时截获的光子最多，在掠射时最少。

**镜面反射（Specular，Blinn-Phong）**：

Blinn-Phong 引入"半程向量"H，避免了 Phong 模型中需要计算完整反射向量的开销：

```
H = normalize(L + V)
L_specular = k_s × (N · H)^shininess × I_light × attenuation
```

`V` 是从表面指向相机的方向。`H` 是 `L` 和 `V` 的中间方向：当 H 与 N 完全对齐时，说明观察方向恰好是镜面反射方向，产生高光峰值。

Blinn-Phong 对比 Phong 的优势：
1. 半程向量计算比反射向量计算略快
2. 在掠射角时 Blinn-Phong 的高光更自然（Phong 在掠射时会出现断裂的高光轮廓）
3. 更接近物理（Blinn-Phong 是 Cook-Torrance BRDF 在某些参数下的近似）

`shininess` 控制高光的"锐利度"：越大高光越集中。本实现使用 64，属于中等光滑表面。

### 透视校正插值

这是软光栅化中最容易忽略的数学细节。在屏幕空间对属性（法线、世界坐标、UV等）进行线性插值是错误的，因为透视投影不保持线性。

设三角形三个顶点在裁剪空间中的 w 分量（即视空间深度）分别为 w₀, w₁, w₂，屏幕空间中的重心坐标为 λ₀, λ₁, λ₂，则透视正确的插值权重为：

```
p_i' = λ_i / w_i
归一化后: p_i = p_i' / (p_0' + p_1' + p_2')
```

直觉解释：靠近相机（w 小）的顶点在透视投影中贡献更大，需要对重心坐标按 1/w 加权。如果不做透视校正，远处的纹理会被拉伸变形，法线插值也会出错（靠近裁剪面的三角形特别明显）。

---

## 实现架构

### 整体管线数据流

```
三角形几何数据（顶点 + 法线 + 颜色）
         ↓
  [Geometry Pass]
    model → world → view → clip → NDC → screen
    背面剔除 → AABB Bounding Box → 光栅化
    重心坐标 → Z-Test → 透视校正插值
         ↓
  G-Buffer（albedo, normal, worldPos, depth, valid）
         ↓
  [Lighting Pass]
    逐像素: 读 G-Buffer → 累加所有光源光照
    Blinn-Phong(ambient + diffuse + specular)
    逐通道 Reinhard 色调映射
    Gamma 2.2 校正
         ↓
  Framebuffer (RGB float 0~1)
         ↓
  PPM 文件 → PIL 转 PNG
```

### 关键数据结构

**G-Buffer**（每像素 4 个 vec3 + 1 个 float + 1 个 bool，总约 6MB for 800×600）：

```cpp
struct GBuffer {
    Vec3 albedo[WIDTH * HEIGHT];    // 漫反射颜色
    Vec3 normal[WIDTH * HEIGHT];    // 世界空间法线
    Vec3 worldPos[WIDTH * HEIGHT];  // 世界空间位置
    float depth[WIDTH * HEIGHT];    // NDC 深度（z-buffer）
    bool valid[WIDTH * HEIGHT];     // 是否有有效几何
};
```

注意这里用 `static GBuffer gbuf;`——因为 G-Buffer 是 ~6MB 的大数组，放在栈上会栈溢出，必须是 static（全局内存）或 heap allocation。

**点光源**：

```cpp
struct PointLight {
    Vec3 position;    // 世界坐标
    Vec3 color;       // 光色（RGB，可超过 1.0 表示高强度）
    float intensity;  // 强度倍率
    float radius;     // 影响半径（用于剔除和边缘衰减）
};
```

**三角形**：直接存三个顶点（含世界坐标、法线、漫反射颜色），不使用索引缓冲。对于一次性渲染，这简化了代码；工业引擎中当然会用 VBO/IBO 优化。

### Geometry Pass vs Lighting Pass 的职责划分

**Geometry Pass** 完全不关心光照。它的工作就是：
1. 变换顶点到屏幕空间
2. 光栅化：对每个覆盖的像素做 Z-Test
3. 写入 G-Buffer（albedo, normal, worldPos）

**Lighting Pass** 完全不关心几何变换。它的工作就是：
1. 遍历每个有效像素
2. 从 G-Buffer 读取几何信息
3. 对所有光源求光照贡献（可以用光源范围剔除加速）
4. 色调映射 + Gamma 校正

这种分离使得以后替换光照模型（比如换成 PBR）不需要修改几何处理代码，反之亦然。

---

## 关键代码解析

### 顶点变换与屏幕坐标映射

```cpp
// 变换到裁剪空间（model → world 在本实现合并了）
Vec4 vw(tri.v[k].pos, 1.0f);  // 世界坐标扩展为 vec4（w=1）
Vec4 vc = view * vw;           // 视空间变换
clip[k] = proj * vc;           // 投影变换（生成裁剪坐标）

// 透视除法 → NDC（Normalized Device Coordinates, -1~1）
invW[k] = 1.0f / clip[k].w;
ndc[k] = clip[k].xyz() * invW[k];

// NDC → 屏幕像素坐标
// NDC x: -1(左) ~ +1(右) → 0 ~ WIDTH
// NDC y: -1(下) ~ +1(上) → HEIGHT ~ 0（Y轴翻转！屏幕坐标Y向下）
screen[k].x = (ndc[k].x * 0.5f + 0.5f) * WIDTH;
screen[k].y = (1.0f - (ndc[k].y * 0.5f + 0.5f)) * HEIGHT;
```

**Y 轴翻转**是一个常见的坑：OpenGL/NDC 中 Y 轴向上，但屏幕/图像坐标 Y 轴向下。`1.0f - (...)` 完成了这个翻转，确保天空（高 Y 值）出现在图像上方。

### 背面剔除与光栅化核心

```cpp
// 计算三角形在屏幕空间的"面积"（带符号）
float area = edgeFunc(screen[0], screen[1], screen[2]);
if(area <= 0) goto next_tri;  // 面积 ≤ 0 = 背面，剔除
```

`edgeFunc(a, b, p)` 计算点 p 相对于有向边 ab 的有符号距离：
```cpp
inline float edgeFunc(Vec2 a, Vec2 b, Vec2 p) {
    return (p.x - a.x) * (b.y - a.y) - (p.y - a.y) * (b.x - a.x);
}
```

当三角形按逆时针顺序（CCW）排列时，三个边函数对内部点都是正值。这也是为什么 `area = edgeFunc(v0, v1, v2) > 0` 表示正面——逆时针定向的三角形在屏幕空间面积为正。

重心坐标插值：

```cpp
// 重心坐标（归一化到 0~1）
float b0 = w0 / area;
float b1 = w1 / area;
float b2 = w2 / area;

// 透视校正
float pcb0 = b0 * invW[0];
float pcb1 = b1 * invW[1];
float pcb2 = b2 * invW[2];
float pcSum = pcb0 + pcb1 + pcb2;
pcb0 /= pcSum; pcb1 /= pcSum; pcb2 /= pcSum;

// 用透视校正后的重心坐标插值世界属性
Vec3 wp = worldPos[0]*pcb0 + worldPos[1]*pcb1 + worldPos[2]*pcb2;
Vec3 wn = (worldNormal[0]*pcb0 + worldNormal[1]*pcb1 + worldNormal[2]*pcb2).norm();
```

注意最后对法线要归一化（`.norm()`），因为两个单位向量的加权平均不再是单位向量。

### Lighting Pass 完整实现

```cpp
void lightingPass(const GBuffer& gbuf, const std::vector<PointLight>& lights,
                  const Vec3& cameraPos, Framebuffer& fb) {
    const Vec3 ambient(0.03f, 0.03f, 0.05f);

    for(int py = 0; py < HEIGHT; py++) {
        for(int px = 0; px < WIDTH; px++) {
            int idx = py * WIDTH + px;

            if(!gbuf.valid[idx]) {
                // 背景：从蓝紫到深蓝的渐变（代替天空盒）
                float t = (float)py / HEIGHT;
                fb.color[idx] = Vec3(0.05f, 0.05f, 0.15f) * (1.0f-t)
                              + Vec3(0.01f, 0.01f, 0.05f) * t;
                continue;
            }

            Vec3 albedo  = gbuf.albedo[idx];
            Vec3 N       = gbuf.normal[idx];        // 世界空间法线
            Vec3 fragPos = gbuf.worldPos[idx];       // 世界空间位置
            Vec3 V       = (cameraPos - fragPos).norm(); // 视线方向

            Vec3 result = ambient * albedo;  // 基础环境光

            for(const auto& light : lights) {
                Vec3 lightVec = light.position - fragPos;
                float dist = lightVec.len();

                // 光源范围剔除：距离超出影响半径则跳过
                // 这是延迟渲染的重要优化点！
                if(dist > light.radius) continue;

                Vec3 L = lightVec / dist;  // 归一化光源方向

                // 衰减：平方反比 + 边缘平滑截断
                float attenuation = 1.0f / (1.0f + 0.09f*dist + 0.032f*dist*dist);
                float edgeFade = (1.0f - dist/light.radius);
                edgeFade = edgeFade * edgeFade;
                attenuation *= edgeFade;

                // Diffuse (Lambert)
                float NdotL = std::max(0.0f, N.dot(L));
                Vec3 diffuse = albedo * light.color * (light.intensity * NdotL * attenuation);

                // Specular (Blinn-Phong)
                Vec3 H = (L + V).norm();
                float NdotH = std::max(0.0f, N.dot(H));
                float spec = powf(NdotH, 64.0f);  // shininess = 64
                Vec3 specular = light.color * (light.intensity * spec * attenuation * 0.3f);

                result += diffuse + specular;
            }

            // HDR 色调映射（逐通道 Reinhard）
            result.x = result.x / (result.x + 1.0f);
            result.y = result.y / (result.y + 1.0f);
            result.z = result.z / (result.z + 1.0f);

            // Gamma 校正 2.2
            result.x = powf(std::max(0.0f, result.x), 1.0f/2.2f);
            result.y = powf(std::max(0.0f, result.y), 1.0f/2.2f);
            result.z = powf(std::max(0.0f, result.z), 1.0f/2.2f);

            fb.color[idx] = clampVec(result);
        }
    }
}
```

**光源范围剔除**（`if(dist > light.radius) continue;`）是延迟渲染的核心优化。在工业引擎中，这一步可以做得更精细——Tile-Based Deferred Shading 把屏幕分成 16×16 的 Tile，对每个 Tile 预先计算哪些光源可能影响它（Tile-Light Culling），极大减少 Lighting Pass 的工作量。Clustered Deferred（如 Doom 2016 使用的方案）进一步在深度方向也做分割，处理深度范围跨度很大的场景。

### 球体网格生成

```cpp
// 经纬度参数化球体生成
// lat × lon 个小四边形，每个拆成两个三角形
for(int i = 0; i < lat; i++) {
    float t0 = (float)i     / lat * M_PI;  // 极角（0 ~ π）
    float t1 = (float)(i+1) / lat * M_PI;
    for(int j = 0; j < lon; j++) {
        float p0 = (float)j     / lon * 2.0f * M_PI;  // 方位角（0 ~ 2π）
        float p1 = (float)(j+1) / lon * 2.0f * M_PI;

        // 球面参数化：(θ, φ) → 单位球面法线
        // x = sin(θ)cos(φ)，y = cos(θ)，z = sin(θ)sin(φ)
        // θ=0 时 y=1（北极），θ=π 时 y=-1（南极）
        auto sph = [&](float t, float p) -> Vec3 {
            return {sinf(t)*cosf(p), cosf(t), sinf(t)*sinf(p)};
        };

        Vec3 n00 = sph(t0,p0), n10 = sph(t1,p0);
        Vec3 n01 = sph(t0,p1), n11 = sph(t1,p1);

        // 世界坐标 = 中心 + 法线 × 半径（法线即从球心到表面的单位向量）
        Vec3 p00 = center + n00 * radius;
        // ...
    }
}
```

球体的法线特别简单：对于单位球，法线就是从球心到表面点的向量，也就是参数化函数的直接输出（`sph(t, p)`），不需要额外计算。

---

## 踩坑实录

### Bug 1：Vec3 除法运算符缺失导致编译错误

**症状**：编译出现 `error: no match for 'operator/' (operand types are 'Vec3' and 'Vec3')`，错误发生在色调映射那行：
```cpp
result = result / (result + Vec3(1.0f,1.0f,1.0f));  // ❌ Vec3 / Vec3 未定义
```

**错误假设**：以为 Vec3 的 `/` 运算符支持 Vec3 / Vec3（逐分量除法）。

**真实原因**：Vec3 只定义了 `Vec3 / float`，没有定义 `Vec3 / Vec3`。这是对 Reinhard 色调映射公式的错误转写——公式 `x / (x + 1)` 本意是逐分量操作，但被错误地写成了 Vec3 整体除法。

**修复方式**：改为逐分量手动操作：
```cpp
result.x = result.x / (result.x + 1.0f);
result.y = result.y / (result.y + 1.0f);
result.z = result.z / (result.z + 1.0f);
```

或者添加 `Vec3 operator/(const Vec3& o) const { return {x/o.x, y/o.y, z/o.z}; }` 运算符重载。本次选择前者，更直观。

**教训**：C++ 的运算符需要显式重载，不要假设数学运算符"自动支持"各种操作数类型。

### Bug 2：未使用变量产生 Warning

**症状**：编译出现两个 warning：
```
warning: unused variable 'hw' [-Wunused-variable]
warning: unused variable 'hh' [-Wunused-variable]
```

**原因**：在 `rasterizeToGBuffer` 中定义了 `const float hw = WIDTH * 0.5f; const float hh = HEIGHT * 0.5f;`，原本计划用来做屏幕空间的中心偏移，但最终代码换了一种写法，这两个变量成了孤儿。

**修复方式**：直接删除这两行。目标是 0 warnings，所以必须处理。

**教训**：`-Wextra` flag 会捕获这类问题。养成"所有 warning 都当 error 处理"的习惯（可以用 `-Werror` 强制）。

### Bug 3：PPM 输出被误命名为 PNG

**症状**：程序输出了 `deferred_shading_output.png`，但 PIL 打开时报 `UnidentifiedImageError`，因为文件实际上是 PPM 格式（缺少 PNG 头）。

**原因**：在没有 ImageMagick 的备用路径里，`cp deferred_shading_output.ppm deferred_shading_output.png` 只是复制了文件并改了扩展名，内容还是 PPM 格式。`.png` 扩展名是谎言。

**修复方式**：在最终验证前，用 Python PIL 把 PPM 重新读入并以 PNG 格式保存：
```python
from PIL import Image
img = Image.open("deferred_shading_output.ppm")
img.save("deferred_shading_output.png")
```

PIL 的 `save()` 会根据文件扩展名选择正确的编码器，真正输出 PNG 格式。

**教训**：不要依赖文件扩展名判断格式，特别是在后处理管线中。

### Bug 4：大数组栈溢出（潜在 Bug，已通过 static 规避）

**G-Buffer 的内存占用**：
- 800 × 600 = 480,000 像素
- 每像素：3×vec3 + 1×float + 1×bool = 3×12 + 4 + 1 = 41 bytes
- 总计：~19.7 MB

Linux 默认栈大小约 8 MB，如果把 `GBuffer gbuf;` 放在 `main()` 的局部变量里，**必然栈溢出**（segfault）。

**修复方式**：使用 `static GBuffer gbuf;`（存放在 BSS 段）或 `new GBuffer()`（堆分配）。本实现选了 `static`，简洁。

**教训**：几MB以上的大数据结构永远不要放栈上。

---

## 效果验证与数据

### 渲染结果

主渲染图（延迟光照最终输出）：

![延迟渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-12-deferred-shading/deferred_shading_output.png)

G-Buffer 法线可视化（RGB 编码世界空间法线，向上 = 偏绿，向右 = 偏红）：

![G-Buffer 法线](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-12-deferred-shading/gbuffer_normal.png)

### 量化数据

**像素统计**（验证脚本输出）：
```
像素均值: 40.6  标准差: 54.9
图片尺寸: (800, 600)
PNG大小: 77.9 KB
上半部分亮度: 51.8, 下半部分亮度: 29.4
```

- 均值 40.6（约 16% 亮度）：说明场景整体偏暗但有大量高光区域（体现在高标准差 54.9）
- 标准差 54.9：图像有丰富的亮度变化，说明多光源产生了明显的明暗对比
- 上半（51.8）亮于下半（29.4）：符合预期——彩色点光源在中高度位置环绕，地面反射相对较弱

**G-Buffer 填充率**：
```
Valid G-Buffer pixels: 119196 / 480000 (24.8%)
```
即 75.2% 的像素是背景（未被任何几何遮挡），Lighting Pass 对这些像素只需填充背景渐变色，无需光照计算。延迟渲染的优势在几何覆盖率高（场景填满屏幕）时才会更显著。

**场景规模**：
```
Scene triangles: 13856
Lights count: 17
```

8 个球体（每个约 24×36×2 = 1728 个三角形）+ 地面（4×4×2 = 32 个三角形）。17 个光源。

在软件渲染（无 GPU 加速）的情况下，渲染时间约 1-2 秒（CPU 单线程）。GPU 延迟渲染管线中，这个规模的场景轻松达到 60fps 以上。

---

## 总结与延伸

### 延迟渲染的局限性

**1. 透明物体问题**

延迟渲染天然不支持透明物体——G-Buffer 只能存储每像素一个表面的信息，透明叠加需要多层信息。

解决方案：通常把透明物体单独用前向渲染处理，在延迟渲染结束后 Composite（合并）到画面上。这也是为什么游戏中的水面、玻璃往往是单独处理的。

**2. MSAA 难以支持**

MSAA（多重采样反走样）工作在几何边缘，需要在渲染三角形时同时写入多个采样点。G-Buffer 如果要支持 MSAA，存储成本乘以采样数（4x MSAA = 4 倍内存），代价很高。现代引擎主要使用 TAA（时间反走样）配合延迟渲染，避开这个问题。

**3. G-Buffer 带宽开销**

G-Buffer 的读写是 Lighting Pass 的瓶颈。在移动 GPU 上，DRAM 带宽非常宝贵，纯 Deferred 的 G-Buffer 带宽开销是严重问题，这催生了 Tile-Based Rendering（TBR）架构的 "Deferred Lighting"（只存法线+深度，保留前向渲染做材质）。

### 延伸方向

**Tile-Based Deferred Shading（TBDS）**：把屏幕分成 16×16 的 Tile，GPU Compute Shader 并行为每个 Tile 做 Light Culling，只处理对该 Tile 有影响的光源列表。这是 DICE/Frostbite 的关键优化，可以支持上千个动态光源。

**Clustered Deferred Shading**：在 TBDS 基础上，沿深度方向也做分割（Clusters），适合深度范围变化大的场景（比如透明物体堆叠、体积雾）。

**与本系列的关联**：
- 05-06 的 IBL Split-Sum PBR 可以作为 Lighting Pass 的升级版光照模型（把 Blinn-Phong 换成 Cook-Torrance）
- 05-08 的 CSM（级联阴影）可以集成到 Lighting Pass 中，为主方向光添加阴影
- 04-18 的 SSR（屏幕空间反射）天然配合延迟渲染——G-Buffer 中已有的法线和位置数据直接用于 SSR 的光线步进

延迟渲染是现代游戏引擎中几乎所有高级渲染特性的基础平台。掌握它，就有了通往 GI、体积光、SSS 等复杂效果的"脚手架"。

---

*代码仓库：[daily-coding-practice / 05-12-deferred-shading](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-12-deferred-shading)*
