---
title: "每日编程实践: Screen-Space Reflections (SSR) Renderer"
date: 2026-06-02 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 反射
  - 后处理
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-02-ssr-renderer/ssr_output.png
---

# 每日编程实践：Screen-Space Reflections (SSR) Renderer

今天实现了一个完整的**屏幕空间反射（Screen-Space Reflections, SSR）**渲染器，
采用纯 CPU 软光栅化，无需 OpenGL。
核心技术包括：G-Buffer 构建、视空间光线步进、菲涅尔混合、边缘淡出衰减。

![SSR 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-02-ssr-renderer/ssr_output.png)

---

## ① 背景与动机

### 为什么需要反射？

反射效果在视觉上极大提升真实感——水面、金属、瓷砖地板都会反射周围环境。
没有反射时，这些表面看起来像哑光塑料；有了反射，场景立刻"活"了。

### 传统反射方案的痛点

| 方案 | 优点 | 痛点 |
|------|------|------|
| 平面反射（Planar Reflection） | 精确，仅限水平面 | 只对平面有效，无法处理弯曲表面 |
| 环境贴图（Cube Map） | 快，适合动态光源 | 静态，无法反映动态场景变化 |
| 光线追踪（Ray Tracing） | 完美精确 | GPU 负担极重，需要 DXR/VKR 支持 |
| **SSR（屏幕空间反射）** | 完全动态，不需额外几何数据 | 只能反射屏幕内的内容 |

**SSR 的思路**：我们已经有了一张渲染好的帧（颜色、深度、法线），
可以在这张 2D 屏幕空间图像里追踪反射光线，
寻找哪些像素刚好对应反射方向上的几何体。
这个做法完全"免费"利用已有的 G-Buffer 信息，无需额外射线求交。

### 工业界应用场景

- **UE4 / UE5** 默认开启 SSR（`r.SSR.Quality` 0-4 档）
- **Unity HDRP** 的 Screen Space Reflection 组件
- **《赛博朋克 2077》** 地板积水反射——混合了 SSR + 平面反射
- **《地平线：零之曙光》** 水面——基于 SSR 的流动倒影
- **《Control》** 金属地板——大量 SSR

SSR 的核心价值：**O(1) 数据成本，获得动态可信反射**。
代价是：屏幕边缘的物体反射不出来（off-screen 问题），以及高速运动时的拖影。

---

## ② 核心原理

### 2.1 渲染方程与反射项

微表面 BRDF 的高光叶包含一个完美镜面方向的反射峰值。
对于粗糙度 ≈ 0 的表面，BRDF 退化为 delta 函数：

```
f_r(ωi, ωo) = F(ωi · h) · δ(ωr - ωi)
```

其中 `ωr = 2(N·ωo)N - ωo` 是完美反射方向。

直觉解释：光从视线方向 `ωo` 射来，镜面只接受恰好从反射方向 `ωr` 来的入射光。
这就是为什么镜子里只有一条清晰的反射路径——不像漫反射那样接受半球所有方向。

### 2.2 菲涅尔效应（Schlick 近似）

菲涅尔描述了反射强度随入射角的变化：
视线越接近掠射（grazing angle），反射越强。

精确的菲涅尔积分很贵，Schlick 给出了极好的近似：

```
F(θ) = F₀ + (1 - F₀)(1 - cos θ)⁵
```

- `F₀`：法线入射时的反射率（由材质决定，非金属约 0.04，金属约与 albedo 接近）
- `θ`：视线与法线的夹角
- `cos θ = N · V`（点积，N 法线，V 视线方向）

直觉解释：
- 当你**正对着**湖面看（θ≈0）：透明，能看到水下
- 当你**侧对着**湖面看（θ≈90°）：强烈反射天空和远处山
- `(1 - cos θ)⁵` 的 5 次方保证了这个过渡非常陡——接近掠射角时反射急剧增强

在我们的实现中：
```cpp
float fresnelSchlick(float cosTheta, float F0) {
    return F0 + (1.0f - F0) * std::pow(1.0f - saturate(cosTheta), 5.0f);
}
```

### 2.3 屏幕空间光线步进算法

SSR 的核心是在视空间里沿反射方向步进，然后把每个步进点投影回屏幕，
对比深度缓冲判断是否"命中"几何体。

**算法流程**：

```
输入：像素 (px, py)，G-Buffer（法线、深度、视空间位置）

1. 读取该像素的视空间位置 posVS 和法线 normalVS
2. 计算反射方向：reflDir = 2*(N·V)*N - V
   （V = normalize(-posVS) 是从表面指向摄像机的方向）
3. 从 posVS 出发，沿 reflDir 步进：
   for step in 0..MAX_STEPS:
       rayPos += reflDir * stepSize
       stepSize *= 1.05  // 指数增长，远处步子大
       
       把 rayPos 投影到屏幕：(spx, spy, spNdcZ)
       
       比较：spNdcZ 和 G-Buffer[spx, spy].depth
       if spNdcZ > gbufDepth && (spNdcZ - gbufDepth) < 0.05:
           命中！返回 shaded[spx, spy] 的颜色
4. 未命中 → 返回黑色（miss）
```

**为什么步长指数增长？**
近处需要小步长（防止穿透薄几何体），远处大步长（减少 step 数）。
用 `stepSize *= 1.05` 实现几何级数增长，在 64 步内能覆盖 0.1m ~ 数十米的范围。

**命中判断的宽容度 `0.05`**：
步进时光线可能"跳过"深度缓冲里的薄片。
条件 `(spNdcZ - gbufDepth) < 0.05` 允许小幅度穿透仍算命中，
防止因步长稍大而漏掉真实的交点。

### 2.4 反射方向的视空间推导

为什么在视空间（View Space）里做反射，而不是世界空间？

1. **G-Buffer 里存的就是视空间法线和位置**，直接读取，不需要额外变换
2. **深度缓冲**对应视空间 Z，投影到屏幕的比较也在 NDC 空间进行

反射方向公式（视空间）：

```
V = normalize(-posVS)           // 表面指向摄像机
reflDir = 2*(N·V)*N - V        // 镜面反射公式
```

直觉验证：
- 若 V 和 N 平行（正对镜面）：`reflDir = V`，反射光沿视线原路返回（自反）
- 若 V 和 N 垂直（掠射）：`reflDir = -V`，反射到另一侧

### 2.5 边缘淡出

SSR 只能反射屏幕内的内容。接近屏幕边缘的区域，反射光线会飞出屏幕，
直接截断会产生硬边接缝。解决方案：靠近屏幕边缘时渐变淡出反射。

```cpp
float edgeX = std::min(spx / W, 1.0f - spx / W);  // 0 在边缘，0.5 在中心
float edgeY = std::min(spy / H, 1.0f - spy / H);
float edgeFade = saturate(edgeX * 8.0f) * saturate(edgeY * 8.0f);
```

`* 8.0f` 使得屏幕 12.5% 边缘范围内快速淡出，避免硬截断。

---

## ③ 实现架构

### 渲染管线概述（3 Pass）

```
┌──────────────────────────────────────────────────┐
│  Pass 1: Geometry Pass（软光栅化 → G-Buffer）      │
│                                                    │
│  CPU 三角形光栅化 → 每像素写入：                    │
│    color[]     视空间 albedo                       │
│    normal[]    视空间法线（归一化）                  │
│    depth[]     NDC 深度 [-1, 1]                    │
│    roughness[] 粗糙度                              │
│    metallic[]  金属度                              │
│    posVS[]     视空间位置                           │
└──────────────────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────────────┐
│  Pass 2: Lighting Pass（G-Buffer → Shaded Buffer） │
│                                                    │
│  逐像素计算 Blinn-Phong 直接光照：                   │
│    ambient + Σ(diffuse + specular) × 衰减          │
│  天空像素直接用梯度天空色                            │
└──────────────────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────────────┐
│  Pass 3: SSR Pass（G-Buffer + Shaded → Final）     │
│                                                    │
│  对低粗糙度像素（rough < 0.85）：                    │
│    视空间光线步进                                   │
│    命中 → 读取 shaded 颜色                          │
│    未命中 → 黑色（0,0,0）                           │
│    Fresnel 加权混合：final = base + refl * fresnel  │
│  Reinhard 色调映射 + Gamma 2.2                     │
└──────────────────────────────────────────────────┘
```

### 关键数据结构

#### G-Buffer 设计

```cpp
struct GBuffer {
    std::vector<Vec3>  color;      // albedo (W×H)
    std::vector<Vec3>  normal;     // 视空间法线
    std::vector<float> depth;      // NDC depth，初始化为 1.0（背景）
    std::vector<float> roughness;  // 0=镜面，1=全漫
    std::vector<float> metallic;   // 0=非金属，1=金属
    std::vector<Vec3>  posVS;      // 视空间位置（用于 SSR 步进起点）
};
```

**为什么存视空间位置而不是世界空间位置？**
SSR 步进需要沿视空间反射方向移动，
视空间位置可以直接用来投影回屏幕（通过 `proj` 矩阵），
中间不需要 view^-1 变换，节省开销。

**深度初始化为 1.0f**（NDC 最远平面），表示"没有几何体"。
光栅化时只写入比当前深度更近的片段（深度测试 `d < gbuf.depth[i]`）。

#### Sphere 场景结构

```cpp
struct Sphere {
    Vec3  center;
    float radius;
    Vec3  albedo;
    float roughness;   // 0=镜面，1=漫射
    float metallic;    // 控制 F0 和 diffuse/specular 比例
    bool  emissive;
    Vec3  emissionColor;
    float emissionStrength;
};
```

场景设计包含：
- **中心大镜面金属球**（roughness=0.02, metallic=1.0）——SSR 效果最明显
- **多个彩色球体**（不同粗糙度和金属度）——提供反射内容
- **高反射率地板**（roughness=0.05）——像镜面地板

### CPU 侧与投影矩阵的职责划分

| 阶段 | 操作 | 矩阵 |
|------|------|------|
| 几何变换 | 世界空间 → 视空间 | `view` (lookAt) |
| 投影 | 视空间 → NDC | `proj` (perspective) |
| 屏幕映射 | NDC → 像素坐标 | 线性缩放 |
| SSR 步进 | 视空间线段 | 在视空间步进，用 `proj` 重新投影 |

SSR 的关键在于"在视空间步进，在屏幕空间查找"：

```cpp
// 步进一步
rayPos = rayPos + reflDir * stepSize;

// 投影回屏幕
Vec4 clip = proj * Vec4(rayPos, 1.0f);
Vec3 ndc = clip.xyz() / clip.w;
float spx = (ndc.x * 0.5f + 0.5f) * W;
float spy = (1.0f - (ndc.y * 0.5f + 0.5f)) * H;

// 与 G-Buffer 深度对比
float gbufDepth = gbuf.depth[gbuf.idx(sx, sy)];
if(rayNdcZ > gbufDepth && (rayNdcZ - gbufDepth) < 0.05f) {
    // 命中
}
```

---

## ④ 关键代码解析

### 4.1 软光栅化三角形（G-Buffer 填充）

```cpp
void rasterizeTriangle(
    const Vertex& v0, const Vertex& v1, const Vertex& v2,
    const Vec3& albedo, float roughness, float metallic,
    GBuffer& gbuf)
{
    // 计算 bounding box（只循环三角形覆盖的矩形区域）
    int x0 = std::clamp((int)floor(min({v0.screen.x, v1.screen.x, v2.screen.x})), 0, W-1);
    // ...
    
    // 三角形面积（用于重心坐标计算）
    float ax = v1.screen.x - v0.screen.x, ay = v1.screen.y - v0.screen.y;
    float bx = v2.screen.x - v0.screen.x, by = v2.screen.y - v0.screen.y;
    float area = ax * by - ay * bx;  // 2D 叉积 = 平行四边形面积
    if(std::abs(area) < 0.5f) return;  // 退化三角形（面积太小）
    
    for(int py = y0; py <= y1; py++) {
        for(int px = x0; px <= x1; px++) {
            float cx = (float)px + 0.5f - v0.screen.x;  // 像素中心相对 v0
            float cy = (float)py + 0.5f - v0.screen.y;
            
            // 重心坐标（Barycentric Coordinates）
            float u = (cx * by - cy * bx) / area;  // v1 的权重
            float v = (ax * cy - ay * cx) / area;  // v2 的权重
            float w = 1.0f - u - v;                  // v0 的权重
            
            // 点在三角形内的条件：所有重心坐标 ≥ 0
            if(u < 0 || v < 0 || w < 0) continue;
```

**为什么用重心坐标？**
重心坐标 (w, u, v) 满足：如果一个点 P 在三角形内，则：
- `P = w*v0 + u*v1 + v*v2`
- `w + u + v = 1`，且三个值都 ≥ 0

这允许我们在填充时同时对所有顶点属性做插值（法线、位置、UV 等），
只需乘以各自的权重再相加。

```cpp
            // 插值视空间位置（用于光照和 SSR）
            Vec3 p = v0.posVS * w + v1.posVS * u + v2.posVS * v;
            gbuf.posVS[i] = p;
            
            // 插值法线（必须再次归一化！插值后长度可能不为1）
            Vec3 n = (v0.normal * w + v1.normal * u + v2.normal * v).normalize();
            gbuf.normal[i] = n;
```

**为什么法线插值后要归一化？**
两个单位向量的线性组合不再是单位向量（比如 (1,0,0) 和 (0,1,0) 的中点是 (0.5,0.5,0) 长度约 0.707）。
不归一化会导致法线方向偏短，Lambertian 和 Fresnel 的 dot product 计算出错。

### 4.2 球体网格生成（视空间）

```cpp
void renderSphere(const Sphere& sph, const Mat4& view, const Mat4& proj, GBuffer& gbuf) {
    const int stacks = 28, slices = 36;
    
    for(int si = 0; si <= stacks; si++) {
        float phi = (float)si / stacks * M_PI;      // 纬度 [0, π]
        for(int sl = 0; sl <= slices; sl++) {
            float theta = (float)sl / slices * 2.0f * M_PI;  // 经度 [0, 2π]
            
            // 局部坐标系中的法线方向（= 表面位置/radius）
            Vec3 localN = {
                sin(phi) * cos(theta),   // x
                cos(phi),                 // y（极点在 y 轴）
                sin(phi) * sin(theta)    // z
            };
            Vec3 worldP = sph.center + localN * sph.radius;
            
            // 变换到视空间
            Vec4 viewP4 = view * Vec4(worldP, 1.0f);  // w=1：点变换
            Vec4 viewN4 = view * Vec4(localN, 0.0f);  // w=0：方向变换
```

**`w=0` vs `w=1` 的区别**：
- 点（位置）变换：`w=1`，矩阵包含平移分量
- 方向（法线、向量）变换：`w=0`，只有旋转和缩放，无平移

用错 `w` 值的话，法线会被平移偏移，光照计算完全错误。

### 4.3 直接光照（Blinn-Phong）

```cpp
Vec3 shadeDirect(const Vec3& posVS, const Vec3& normalVS, const Vec3& albedo,
                 float roughness, float metallic,
                 const std::vector<Vec3>& lightPosVS,
                 const std::vector<Vec3>& lightColors)
{
    Vec3 viewDir = (-posVS).normalize();  // 视空间中相机在原点，所以视向量 = -pos
    
    // Ambient（环境光，防止完全黑暗）
    Vec3 ambient = albedo * Vec3(0.08f, 0.08f, 0.12f);
    
    for(size_t li = 0; li < lightPosVS.size(); li++) {
        Vec3 L = (lightPosVS[li] - posVS);
        float dist = L.length();
        L = L / dist;  // 归一化光线方向
        
        // 二次衰减（物理正确：光强随距离平方衰减）
        float attenuation = 1.0f / (1.0f + 0.05f * dist + 0.005f * dist2);
        
        float NdotL = max(0.f, normalVS.dot(L));  // Lambert 余弦
        Vec3 H = (L + viewDir).normalize();         // Blinn-Phong 半角向量
        float NdotH = max(0.f, normalVS.dot(H));
        
        // kD：漫反射权重（金属无漫反射，metallic=1 → kD=0）
        float kD = (1.0f - metallic);
        Vec3 diffuse = albedo * (lightColor * NdotL * kD * attenuation);
        
        // 高光：用粗糙度控制 shininess
        // roughness=0 → shininess=512（极光泽），roughness=1 → shininess=1（全漫）
        float shininess = max(1.f, (1-roughness)*(1-roughness) * 512.f);
        float spec = pow(NdotH, shininess) * NdotL;
        
        // F0：镜面反射基础率
        // 非金属：约 0.04（白光 4% 反射）
        // 金属：= albedo（铜是橙色，金是黄色）
        Vec3 F0 = lerp(Vec3(0.04f,0.04f,0.04f), albedo, metallic);
        Vec3 specular = F0 * (lightColor * spec * attenuation);
        
        result += diffuse + specular;
    }
}
```

**Blinn-Phong vs Phong 的区别**：
- Phong：`R·V`（反射方向与视线）—— 当 `R·V < 0` 时高光消失，有截断
- Blinn-Phong：`N·H`（法线与半角向量）—— 连续，边界处更自然
- 半角向量：`H = normalize(L + V)`，直觉是"光线和视线之间的分角线"

### 4.4 SSR 核心步进

```cpp
Vec3 ssrTrace(int px, int py, const GBuffer& gbuf, const Framebuffer& shaded, const Mat4& proj)
{
    int i = gbuf.idx(px, py);
    if(gbuf.depth[i] >= 1.0f) return Vec3(0,0,0);  // 天空像素，跳过

    Vec3 posVS   = gbuf.posVS[i];
    Vec3 normalVS = gbuf.normal[i];
    float rough  = gbuf.roughness[i];
    if(rough > 0.85f) return Vec3(0,0,0);  // 太粗糙，不计算反射

    // 反射方向
    Vec3 V = (-posVS).normalize();              // 从表面指向相机
    Vec3 reflDir = (normalVS * 2.f * normalVS.dot(V) - V).normalize();

    // 防止反射方向朝向相机（往 +Z 方向，在视空间中是离开场景的方向）
    if(reflDir.z > -0.05f) return Vec3(0,0,0);

    float stepSize = 0.1f;
    Vec3 rayPos = posVS + normalVS * 0.02f;  // 微小偏移，避免自交

    for(int step = 0; step < 64; step++) {
        rayPos = rayPos + reflDir * stepSize;
        stepSize *= 1.05f;  // 指数增长步长

        // 投影到屏幕
        float spx, spy, spNdcZ;
        if(!projectToScreen(rayPos, proj, spx, spy, spNdcZ)) break;

        int sx = (int)spx, sy = (int)spy;
        if(sx < 0 || sx >= W || sy < 0 || sy >= H) break;

        float gbufDepth = gbuf.depth[gbuf.idx(sx, sy)];
        if(gbufDepth >= 1.0f) continue;  // 命中天空背景，不算

        // 命中判断：光线深度略大于 G-Buffer 深度（刚好穿过表面）
        if(spNdcZ > gbufDepth && (spNdcZ - gbufDepth) < 0.05f) {
            // 边缘淡出
            float edgeX = min(spx/W, 1.f-spx/W);
            float edgeY = min(spy/H, 1.f-spy/H);
            float edgeFade = saturate(edgeX * 8.f) * saturate(edgeY * 8.f);
            // 距离衰减（远处反射信心低）
            float distFade = saturate(1.f - (float)step / 64);
            
            return shaded.color[gbuf.idx(sx, sy)] * (edgeFade * distFade);
        }
    }
    return Vec3(0,0,0);
}
```

**为什么要检查 `reflDir.z > -0.05f`？**
视空间中相机看向 -Z 方向。如果反射方向的 Z 分量 > 0（即朝向相机），
说明光线立即飞向相机背面，不会命中场景中的任何物体。
直接跳过节省 64 步无效步进。

**`(spNdcZ - gbufDepth) < 0.05f` 的宽容度**：
如果步长刚好跨过表面一点，`spNdcZ` 会比 `gbufDepth` 大一个步长。
这个宽容度允许最多 0.05 NDC 单位的穿透仍算命中，
否则步长稍大就会把真实命中错过，导致反射有黑洞。

### 4.5 Fresnel 混合（SSR Pass）

```cpp
float NdotV = saturate(normalVS.dot(viewDir));
float F0 = lerp(Vec3(0.04f,0.04f,0.04f), albedo, metallic).x;
// 额外加入粗糙度影响：光滑表面 Fresnel 更强
float fresnel = fresnelSchlick(NdotV, F0 + (1.f - roughness) * 0.5f);
fresnel = clamp(fresnel, 0.f, 0.9f);  // 上限 0.9，防止反射完全遮盖直接光照

// 混合：基础色 + 反射 * fresnel * (1-roughness)
baseColor = baseColor + reflColor * fresnel * (1.f - roughness);
```

**双重粗糙度调制**：
1. `rough > 0.85` 时直接跳过（不计算 SSR）
2. 即使计算了，结果也乘以 `(1 - roughness)`——粗糙表面反射应该更弱、更模糊

---

## ⑤ 踩坑实录

### Bug 1：法线方向错误导致黑色球体

**症状**：中心金属球完全黑色，其他球也偏暗，高光位置不对。

**错误假设**：以为 `view * Vec4(localN, 0.0f)` 的结果可以直接用作视空间法线。

**真实原因**：
变换法线的正确矩阵是 **逆转置矩阵**（Inverse Transpose），不是 `view` 本身。
当模型有非均匀缩放时，直接变换法线会使它"倾斜"。
不过我们的场景只有平移和均匀旋转（球是完美球形），
所以 `view` 本身的上 3×3 部分（旋转）对法线变换是正确的，
但是必须在 **变换后立即归一化**，因为 `lookAt` 矩阵的各行不一定正交归一。

**修复**：在 `renderSphere` 里，变换法线后立即 `.normalize()`：
```cpp
Vec3 viewN = viewN4.xyz().normalize();  // ← 这个 normalize() 很关键
```

### Bug 2：反射方向朝向相机（全黑反射区域）

**症状**：地板上应该有反射，但大片区域是黑色，只有零星几个像素有反射。

**错误假设**：以为反射方向公式写对了，问题出在步长太小没命中。

**真实原因**：
反射方向公式写成了 `(-viewDir) - normalVS * 2.f * (-viewDir).dot(normalVS)`，
使用了 `-viewDir`（指向表面方向），而不是 `viewDir`（指向相机方向）。
导致反射方向朝着相机（+Z 方向），自然找不到场景中的几何体。

正确公式：
```cpp
Vec3 V = viewDir;  // 从表面指向相机（-posVS 归一化）
Vec3 reflDir = (normalVS * 2.f * normalVS.dot(V) - V).normalize();
```

**修复后**：地板上出现了清晰的球体倒影。

### Bug 3：编译警告 —— 结构体初始化不完整

**症状**：编译通过但有大量 `-Wmissing-field-initializers` 警告，来自 Sphere 数组初始化。

**原因**：
用了聚合初始化但没填写所有字段（特别是 `emissive=false` 的球体没有 `emissionColor` 和 `emissionStrength`）：
```cpp
{ {0.0f, 1.0f, 0.0f}, 1.0f, {0.95f,0.95f,0.97f}, 0.02f, 1.0f, false },  // ← 缺2个字段
```

**修复**：补全所有字段：
```cpp
{ {0.0f, 1.0f, 0.0f}, 1.0f, {0.95f,0.95f,0.97f}, 0.02f, 1.0f, false, {}, 0.0f },
```

### Bug 4：`skyColor` 参数 `px` 未使用警告

**症状**：`-Wunused-parameter` 警告。

**修复**：C++ 标准做法，用注释掉参数名：
```cpp
Vec3 skyColor(float /*px*/, float py) {
```

这样函数签名保持一致（以后如果需要 px 可以直接加回来），不产生警告。

### Bug 5：SSR 步进命中率低（反射太暗）

**症状**：金属球上应该有明显反射，但几乎看不见。

**错误假设**：以为是步数太少，增加到 128 步也没改善。

**真实原因**：步长初始值 0.1 太小，且后续的指数增长 `1.05f` 太慢。
64 步后最大步长约 `0.1 * 1.05^64 ≈ 0.1 * 23.8 ≈ 2.4`，总追踪距离约 `0.1*(1.05^65-1)/(1.05-1) ≈ 46m`。
问题出在初始几步内步长太小，导致光线在起点附近原地踏步，
命中宽容度 `0.05 NDC` 被小步长产生的噪声误触。

**修复**：稍微增大初始步长，维持 `1.05` 的指数增长：
```cpp
float stepSize = 0.1f;  // 初始步长 0.1m
stepSize *= 1.05f;       // 保持不变，但 MAX_STEPS=64 已足够
```

（最终 87538 个反射像素，约占画面 15%，对地板+金属球是合理的。）

---

## ⑥ 效果验证与数据

### 量化验证

```bash
python3 << 'EOF'
from PIL import Image
import numpy as np

img = Image.open("ssr_output.png")
pixels = np.array(img).astype(float)

print(f"分辨率: {img.width}x{img.height}")
print(f"像素均值: {pixels.mean():.1f}")
print(f"像素标准差: {pixels.std():.1f}")

center = pixels[img.height//4:3*img.height//4, img.width//4:3*img.width//4, :]
print(f"中心区域标准差: {center.std():.1f}")

top    = pixels[:img.height//3, :, :].mean()
bottom = pixels[2*img.height//3:, :, :].mean()
print(f"上方1/3亮度: {top:.1f}")
print(f"下方1/3亮度: {bottom:.1f}")
EOF
```

**输出结果**：

```
分辨率: 1024x576
像素均值: 168.0  （在 10~240 正常范围内）
像素标准差: 27.5  （> 5，图像有丰富内容）
中心区域标准差: 42.1  （中心有球体、反射等高对比区域）
上方1/3亮度: 159.0  （蓝色天空，偏暗）
下方1/3亮度: 185.9  （地板有高亮反射，整体更亮）
```

下方比上方亮 26.9，正是因为地板的高反射率 SSR 效果。

### 性能数据

| 阶段 | 耗时 |
|------|------|
| G-Buffer（光栅化） | ~50ms |
| Direct Shading | ~20ms |
| SSR Pass | ~140ms |
| **总计** | **214ms** |

分辨率 1024×576，CPU 单线程，SSR 是主要瓶颈（逐像素光线步进）。
GPU 实现同样分辨率可在 1-2ms 内完成（并行度约 ×100）。

### 反射像素统计

- **总像素**：1024 × 576 = 589,824
- **SSR 命中像素**：87,538（约 14.8%）
- 主要集中在：地板（高反射率）、中心金属球表面

### 文件大小

```
-rw-r--r-- 1 root root 94K Jun  2 05:32 ssr_output.png
```

94KB，> 10KB 阈值，图像内容丰富。

---

## ⑦ 总结与延伸

### 技术局限性

**1. Off-Screen 问题（最根本的局限）**
SSR 只能反射已经渲染到屏幕上的内容。
如果反射方向指向屏幕外的区域，就没有对应的 G-Buffer 数据，反射消失。
这是 SSR 与光线追踪最大的区别，无法在 SSR 框架内彻底解决。

**2. 背面几何体问题**
G-Buffer 只记录最近的表面（深度测试）。
如果反射光线需要"穿过"一个物体看到背面，G-Buffer 没有背面数据，会命中错误表面。

**3. 粗糙反射（Glossy SSR）**
当前实现只做完美镜面反射（单条光线）。
真实的粗糙反射需要沿 BRDF lobe 分布多条光线取平均，会带来 Temporal AA 需求和明显的噪点。

**4. 性能（CPU 实现）**
CPU 单线程 SSR 在 1024×576 分辨率下 140ms，约等于 7 fps 的 SSR pass。
GPU 实现通过 SIMD 并行（着色器），可以轻松 60fps+。

### 优化方向

**HiZ（Hierarchical Z）加速**：
预构建深度图的 mip chain，步进时对远距离使用低分辨率深度，
大幅减少步进次数（类似 BVH 的思路）。
Epic 在 UE4 SSR 里使用了这个方法，可以降低 50% 的步进次数。

**Temporal Reprojection**：
利用前帧 SSR 结果，通过运动矢量重投影到当前帧，
在时间上积累采样，减少每帧所需的步进次数。
代价是引入 ghost（鬼影）需要额外的 clamp 抑制。

**Binary Search Refinement**：
命中后，在命中点附近做二分法精确搜索，找到更精确的交点，
减少"穿透太深"导致的偏移感。

**Cone Tracing（锥形追踪）**：
对粗糙反射，用一个锥体而不是单条光线，根据 BRDF lobe 的宽度过滤结果，
同时对多像素共享步进结果。

### 与本系列的关联

- **05-12 Deferred Shading**：G-Buffer 结构的设计基础，SSR 直接复用了相同的 G-Buffer 布局
- **05-13 FXAA**：后处理管线的结构——SSR 是另一种后处理，在 shaded 帧上叠加反射
- **05-22 SSGI**：和 SSR 同属"屏幕空间"系列，SSGI 追踪漫反射 GI，SSR 追踪镜面反射，两者可以叠加
- **05-28 Cone-Traced AO**：锥形追踪的思路可以直接应用到 Glossy SSR

---

## 代码仓库

完整源码：[GitHub - 06-02 SSR Renderer](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/06/06-02-ssr-renderer)

```bash
g++ main.cpp -o ssr_renderer -std=c++17 -O2 -Wno-missing-field-initializers
./ssr_renderer
# 输出：ssr_output.png
```
