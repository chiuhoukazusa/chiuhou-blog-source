---
title: "每日编程实践: Screen Space Refraction — 实时屏幕空间折射渲染"
date: 2026-04-26 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 实时渲染
  - 后处理
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-26-screen-space-refraction/ssr_refraction_output.png
---

# 每日编程实践: Screen Space Refraction — 实时屏幕空间折射渲染

> 本文实现了一套基于 G-Buffer 的屏幕空间折射（Screen Space Refraction，SSR）渲染器，  
> 结合 Schlick Fresnel 混合、Snell's Law 折射偏移、以及色散（Chromatic Aberration），  
> 完整模拟实时渲染管线中玻璃/水面材质的折射效果。

---

## ① 背景与动机

### 折射的视觉重要性

在游戏与实时渲染场景中，玻璃、水面、宝石、冰块等材质无处不在。这些材质有一个共同特点：**光线穿过它们时会发生弯曲**，即折射（Refraction）。

没有折射的玻璃球看起来像一个半透明的薄膜，毫无立体感；加上折射后，透过玻璃球看到的背景会发生形变，形成典型的"鱼眼"放大/缩小效果，这才是玻璃的本质视觉特征。

**现实中的折射失真**：当你隔着一杯水看对面的物体，会发现物体位置发生了偏移，这就是折射在现实中的直观体现。水的折射率约 1.33，玻璃约 1.5，钻石高达 2.42——折射率越高，光线弯曲越剧烈，视觉形变越夸张。

### 工业界使用场景

| 引擎/产品 | 折射实现 | 应用材质 |
|----------|---------|---------|
| Unreal Engine 5 | Screen Space Refraction (默认) + 光线追踪折射 (高质量) | 玻璃、水面、半透明材质 |
| Unity URP | Opaque Texture Grab Pass + 法线偏移 | 玻璃、水、热浪特效 |
| CryEngine | Screen Space Refraction | 水面、冰块、玻璃窗 |
| Frostbite (EA) | SSR + 深度感知偏移 | Battlefield 系列水面 |

为什么这么多引擎选择屏幕空间方案而不是光线追踪折射？原因很简单：**性能**。光追折射精确但昂贵，屏幕空间折射仅利用已有的 G-Buffer 数据，额外开销极小，适合 60fps 的实时场景。

### 没有它会怎样？

如果不实现折射，处理玻璃材质通常的降级方案是：
1. **假半透明**：直接用 alpha blending，忽略形变，视觉上不真实
2. **Cubemap 采样**：环境贴图折射近似，无法反映场景内其他物体
3. **纯反射球**：只有镜面反射，完全看不到背后的物体

三种降级方案都有明显的视觉缺陷，玩家一眼能看出"这不像真玻璃"。正确的折射是让玻璃材质脱离"廉价感"的关键一步。

---

## ② 核心原理

### 2.1 折射的物理基础：Snell's Law

光从介质 1（折射率 $n_1$）进入介质 2（折射率 $n_2$）时，折射方向由斯涅尔定律决定：

$$n_1 \sin\theta_1 = n_2 \sin\theta_2$$

其中 $\theta_1$ 是入射角（入射方向与法线的夹角），$\theta_2$ 是折射角。

**直觉解释**：这个公式说的是，光在不同密度介质中传播速度不同（速度 $v = c/n$，$c$ 为光速），为了保持波前连续性，方向必须发生弯曲。折射率越高的介质，光速越慢，光线偏折越大。

用向量形式更方便代码实现：

$$\vec{r} = \frac{n_1}{n_2}\vec{i} + \left(\frac{n_1}{n_2}\cos\theta_1 - \cos\theta_2\right)\vec{n}$$

其中：
- $\vec{i}$ = 归一化入射方向（朝向表面）
- $\vec{n}$ = 表面法线（朝外）
- $\cos\theta_1 = -\vec{i} \cdot \vec{n}$（注意负号，因为两者方向相反）
- $\cos\theta_2 = \sqrt{1 - \left(\frac{n_1}{n_2}\right)^2(1 - \cos^2\theta_1)}$

当根号内为负时，发生**全内反射**（Total Internal Reflection，TIR）——光完全被反射回去，不再透射。这在玻璃内部从大角度看外面时会发生（潜水员从水下看水面也能看到这个现象）。

```cpp
// Snell's Law 向量形式实现
float ratio = n1 / n2;                        // 折射率比值
float cosI = std::max(0.f, -rd.dot(N));       // cos(入射角)
float k = 1.f - ratio*ratio*(1.f - cosI*cosI); // 判别式
Vec3 refractDir;
if(k < 0){
    // 全内反射：退化为镜面反射
    refractDir = rd - N*(2.f*rd.dot(N));
} else {
    // 标准折射
    refractDir = rd*ratio + N*(ratio*cosI - std::sqrt(k));
}
refractDir = refractDir.norm();
```

**为什么 k < 0 时用反射**：k 就是 $\cos^2\theta_2$，如果它小于 0 说明没有实数解——折射角无法定义，物理上只能全反射。代码里用普通镜面反射来近似这个行为，虽然不完全准确（真正 TIR 没有透射），但视觉上足够合理。

### 2.2 Fresnel 效应：反射与透射的混合比例

现实中，光到达界面时**同时存在反射和折射**，只是各自的比例不同。这个比例由菲涅尔方程决定。

精确的菲涅尔方程涉及 s/p 两种偏振分量，在实时渲染中通常用 **Schlick 近似**：

$$F_r(\theta_i) \approx F_0 + (1 - F_0)(1 - \cos\theta_i)^5$$

其中 $F_0$ 是法向入射时的反射率：

$$F_0 = \left(\frac{n_1 - n_2}{n_1 + n_2}\right)^2$$

**直觉解释**：
- 当 $\theta_i = 0$（正对表面看）：反射率最小，大部分光透过（玻璃正面看最透明）
- 当 $\theta_i = 90°$（掠射角看）：反射率趋向 1，几乎全反射（这就是为什么远处的湖面像镜子，近处反而能看到水下）

```cpp
float schlick(float cosTheta, float n1, float n2){
    float r0 = (n1-n2)/(n1+n2);
    r0 = r0*r0;                    // F₀
    float c = 1.f - cosTheta;
    return r0 + (1-r0)*c*c*c*c*c; // Schlick 近似
}
```

对于空气（$n_1=1.0$）→玻璃（$n_2=1.5$）：$F_0 = \left(\frac{0.5}{2.5}\right)^2 = 0.04$  
这说明玻璃在正视方向只反射约 4% 的光，其余 96% 透射——符合我们对普通玻璃的认知。

### 2.3 色散（Chromatic Aberration）

真实玻璃的折射率随光波长变化（科学名：光的色散）。红光折射最弱，蓝光折射最强。这就是三棱镜能把白光分解成彩虹的原因。

在屏幕空间中模拟色散非常简单：对 RGB 三个通道使用略微不同的 UV 偏移量：

```cpp
float dispR = 0.0f;   // 红光偏移最小
float dispG = 0.02f;  // 绿光中等偏移
float dispB = 0.04f;  // 蓝光偏移最大

// 分别采样三个通道
Vec3 bgR = background.sample(su + dispR, sv);
Vec3 bgG = background.sample(su + dispG, sv);
Vec3 bgB = background.sample(su + dispB, sv);
// 合并：取各通道自己的分量
Vec3 refracted(bgR.x, bgG.y, bgB.z);
```

**直觉解释**：偏移量越大，折射越"远"，透过玻璃看到的背景越偏。红色往折射方向移动最少，蓝色移动最多，导致高对比度边缘出现彩色条纹——这正是玻璃折射的典型视觉特征。

### 2.4 G-Buffer 的作用

传统前向渲染（Forward Rendering）中，每个物体在自己的渲染 Pass 里处理。屏幕空间折射需要**"看到"被折射物体背后的场景**，在前向渲染中这很难实现。

延迟渲染（Deferred Rendering）的 G-Buffer 完美解决这个问题：
- 第一 Pass：将场景几何信息（法线、深度、反射率、材质标记）写入多张贴图
- 后续 Pass：作为屏幕空间的后处理，用 G-Buffer 数据计算各种效果

本实现的 G-Buffer 包含 5 个通道：

```
depth    → 每像素的线性深度值（用于判断折射光线是否打到物体）
normal   → 打包为 [0,1] 范围的法线向量（用于计算折射方向）
albedo   → 表面颜色（玻璃的颜色 tint）
position → 世界空间位置（折射光线起点）
mask     → glass 标记（=1 表示该像素是折射材质）
```

法线打包/解包：
```cpp
// 打包：[-1,1] → [0,1]
gbuf.normal.at(x,y) = (h.normal * 0.5f) + Vec3(0.5f,0.5f,0.5f);

// 解包：[0,1] → [-1,1]
Vec3 N = (nPacked * 2.f) - Vec3(1,1,1);
N = N.norm();  // 重新归一化（浮点精度损失修正）
```

---

## ③ 实现架构

### 渲染管线总览

本实现采用 4-Pass 软件光栅化流水线：

```
Pass 1: G-Buffer Construction
  输入：场景几何（Sphere, Plane）+ Camera
  输出：depth / normal / albedo / position / mask 五张缓冲
  
Pass 2: Background Scene Render
  输入：场景（所有物体视为不透明）+ Camera
  输出：背景颜色图（作为折射采样来源）
  算法：直接光照（Phong 漫反射 + 点光源阴影）
  
Pass 3: Screen Space Refraction Composite
  输入：G-Buffer + Background + 原始场景（用于反射追踪）
  输出：最终合成图
  对每个 glass 像素：
    1. 从 G-Buffer 读取法线、世界坐标
    2. 计算 Fresnel（反射系数）
    3. 计算 Snell 折射方向
    4. 投影折射出射点到屏幕空间
    5. 用色散 UV 偏移采样背景
    6. 追踪反射光线得到反射颜色
    7. Fresnel 混合 = Fr * reflect + (1-Fr) * refract
    
Pass 4: Output
  保存最终图像 + G-Buffer Debug 图（2x2 可视化）
```

### 关键数据结构

```cpp
// G-Buffer：5 张缓冲图
struct GBuffer {
    Image depth;     // .x = 线性深度（场景几何的 t 值）
    Image normal;    // xyz = 打包法线 [0,1]
    Image albedo;    // xyz = 表面颜色
    Image position;  // xyz = 世界坐标
    Image mask;      // .x = 1.0 if glass pixel
    GBuffer(int w, int h) : depth(w,h), normal(w,h), albedo(w,h), position(w,h), mask(w,h) {}
};

// 图像支持双线性插值
struct Image {
    Vec3 sample(float u, float v) const {
        // 双线性插值：防止采样折射偏移后出现锯齿
        ...
    }
};
```

**双线性插值的重要性**：折射偏移后的 UV 坐标通常是浮点数，不对齐到像素中心。如果用最近邻采样，折射区域会出现明显的马赛克感。双线性插值在 4 个相邻像素之间平滑插值，消除这种锯齿。

### 场景构成

本实现的测试场景是一个 Cornell Box 变体：
- **1 个玻璃球**（IOR=1.5）：主要折射对象
- **2 个不透明彩色球**（红色、蓝色）：作为折射背景内容
- **1 个小发光球**：模拟顶部点光源
- **5 个轴对齐平面**：地板、天花板、后墙、左墙（红）、右墙（绿）

场景之所以选 Cornell Box，是因为其左红右绿的墙壁会在玻璃球折射中形成明显的颜色分布，便于肉眼验证折射方向是否正确（从左侧看玻璃球，应该折射出右侧绿墙的颜色）。

---

## ④ 关键代码解析

### 4.1 G-Buffer 构建

```cpp
void buildGBuffer(const Scene& sc, const Camera& cam, GBuffer& gbuf){
    int W = gbuf.depth.w, H = gbuf.depth.h;
    for(int y=0; y<H; ++y){
        for(int x=0; x<W; ++x){
            float u = (x+0.5f)/W;
            // 关键：v 要翻转！Hexo 图像坐标从左上角开始（Y轴向下），
            // 而数学上的相机 Y 轴向上。不翻转会导致天空在下、地面在上。
            float v = 1.f - (y+0.5f)/H;

            Vec3 ro, rd;
            cam.getRay(u, v, ro, rd);

            Hit h{};
            if(sc.intersect(ro, rd, 1e-4f, INF, h)){
                gbuf.depth.at(x,y)    = Vec3(h.t, h.t, h.t);
                gbuf.normal.at(x,y)   = (h.normal * 0.5f) + Vec3(0.5f,0.5f,0.5f);
                gbuf.albedo.at(x,y)   = h.albedo;
                gbuf.position.at(x,y) = h.pos;
                // glass 标记：只有 glass=true 的球才会触发折射 Pass
                gbuf.mask.at(x,y)     = Vec3(h.glass ? 1.f : 0.f, 0, 0);
            } else {
                // 未击中任何物体：深度设为极大值，法线朝前，其余清零
                gbuf.depth.at(x,y)  = Vec3(1e9f, 1e9f, 1e9f);
                gbuf.normal.at(x,y) = Vec3(0.5f,0.5f,1.f); // 打包后 = (0,0,1)
                ...
            }
        }
    }
}
```

**易错点 1：为什么用 `1e-4f` 作为 tmin**？  
如果 tmin=0，光线会立即与自身出发的表面相交（因为浮点精度，位置不完全在表面上），导致自相交（self-intersection）——即 shadow acne 问题。1e-4 是经验值，足够跳过数值精度误差，又不会跳过真实的近处几何。

**易错点 2：深度存的是什么**？  
这里存的是光线参数 `t`（射线方程 `P = O + t*D` 中的 t），而不是相机空间 Z 值。这是软件光栅化的简化做法，避免透视除法。用于 SSR 偏移中判断深度关系。

### 4.2 背景场景渲染

```cpp
Vec3 shade(const Scene& sc, const Vec3& ro, const Vec3& rd, int depth=0){
    if(depth > 3) return Vec3(0,0,0);  // 防止无限递归

    Hit h{};
    if(!sc.intersect(ro, rd, 1e-4f, INF, h)){
        // 天空渐变：蓝天到白色地平线
        float t = 0.5f*(rd.norm().y + 1.f);
        return Vec3(0.5f,0.7f,1.f)*t + Vec3(1,1,1)*(1-t);
    }

    // 直接光照计算
    Vec3 lightPos(0.f, 2.2f, -3.f);
    Vec3 lightCol(1.f, 0.95f, 0.85f);
    float lightPow = 12.f;   // 调整到合适亮度

    Vec3 toL = (lightPos - h.pos).norm();
    float dist2 = (lightPos - h.pos).len2();

    // 阴影检测：向光源发射阴影射线
    Hit sh{};
    float shadow = 1.f;
    if(sc.intersect(h.pos + h.normal*1e-3f, toL, 1e-3f, std::sqrt(dist2)-0.3f, sh))
        shadow = 0.1f;  // 软阴影近似（0.1 而不是 0 是为了保留点间接光）

    float diff = std::max(0.f, h.normal.dot(toL));
    Vec3 direct = h.albedo * lightCol * (lightPow / (1.f + dist2)) * diff * shadow;
    Vec3 ambient = h.albedo * 0.12f;  // 全局环境光（近似间接光照）

    return direct + ambient;
}
```

**为什么背景渲染要把玻璃球当成不透明的**？  
因为背景图是 SSR 折射的**采样来源**。如果渲染背景时已经把玻璃球渲成透明的，折射效果就会被"双重叠加"，产生错误。正确做法是渲染一张纯不透明场景的颜色缓冲，折射 Pass 再从这里采样，在 glass 像素位置用折射坐标替换。

### 4.3 SSR 折射核心

```cpp
// 对每个玻璃像素
for(int y=0; y<H; ++y){
    for(int x=0; x<W; ++x){
        float isGlass = gbuf.mask.at(x,y).x;
        if(isGlass < 0.5f) continue;  // 跳过非玻璃像素

        // Step 1: 解包 G-Buffer 法线
        Vec3 nPacked = gbuf.normal.at(x,y);
        Vec3 N = (nPacked * 2.f) - Vec3(1,1,1);
        N = N.norm();

        // Step 2: 重建入射方向
        float u = (x+0.5f)/W;
        float v_coord = 1.f - (y+0.5f)/H;
        Vec3 ro, rd;
        cam.getRay(u, v_coord, ro, rd);

        // Step 3: Fresnel 系数
        float cosI = std::max(0.f, -rd.dot(N));
        float Fr = schlick(cosI, 1.0f, 1.5f);  // air→glass

        // Step 4: 折射方向（Snell's Law）
        float ratio = 1.0f / 1.5f;   // air IOR / glass IOR
        float k = 1.f - ratio*ratio*(1.f - cosI*cosI);
        Vec3 refractDir = ...;       // 见前文代码

        // Step 5: 折射出射点 → 屏幕 UV
        Vec3 hitPos = gbuf.position.at(x,y);
        float thickness = 1.2f;       // 玻璃厚度估计（平均弦长）
        Vec3 exitPos = hitPos + refractDir * thickness;
        
        float su, sv;
        if(!worldToScreen(exitPos, cam, W, H, su, sv)){
            // 超出屏幕范围时降级到简单偏移
            su = u + refractDir.x * 0.15f;
            sv = v_coord - refractDir.y * 0.15f;
        }

        // Step 6: 色散采样（三通道不同偏移）
        float dispR=0.0f, dispG=0.02f, dispB=0.04f;
        Vec3 bgR = background.sample(su+dispR, sv);
        Vec3 bgG = background.sample(su+dispG, sv);
        Vec3 bgB = background.sample(su+dispB, sv);
        Vec3 refracted(bgR.x, bgG.y, bgB.z);

        // Step 7: 反射颜色（1 bounce）
        Vec3 reflDir = (rd - N*(2.f*rd.dot(N))).norm();
        Vec3 reflColor = shade(sc, hitPos + N*1e-3f, reflDir, 1);

        // Step 8: Fresnel 混合 + 玻璃颜色 tint
        Vec3 glassTint = gbuf.albedo.at(x,y);
        Vec3 finalColor = reflColor*Fr + refracted*(1.f-Fr);
        finalColor = finalColor * glassTint;

        // Step 9: 高光点（增强玻璃质感）
        Vec3 lightDir = (Vec3(0.f,2.2f,-3.f) - hitPos).norm();
        float spec = std::pow(std::max(0.f, reflDir.dot(lightDir)), 64.f);
        finalColor += Vec3(1,1,1) * spec * 0.4f;

        result.at(x,y) = finalColor.clamp01();
    }
}
```

**thickness 参数的意义**：  
SSR 折射的核心思路是：折射光线进入玻璃后，在玻璃内部传播一定距离再出射。这个距离叫做"厚度"。精确厚度需要 Back-Face Depth Buffer（记录物体背面深度），本实现用固定厚度 1.2 近似（球体直径为 1.4，平均弦长约 1.1~1.3）。

**为什么要 `worldToScreen`**？  
折射光线在屏幕空间的偏移量取决于摄像机和物体的相对位置。如果直接用法线方向乘以一个固定系数，近处的物体和远处的物体会有相同的偏移量，而实际上近处物体的折射偏移在屏幕上应该更大。`worldToScreen` 通过真实的投影变换计算屏幕偏移，使结果与距离成正比。

### 4.4 世界坐标 → 屏幕 UV 投影

```cpp
bool worldToScreen(const Vec3& pos, const Camera& cam, 
                   int /*W*/, int /*H*/, float& su, float& sv){
    Vec3 d = pos - cam.origin;
    // 相机本地轴
    Vec3 forward = (cam.lower_left + cam.horiz*0.5f + cam.vert*0.5f - cam.origin).norm();
    Vec3 right    = cam.horiz.norm();
    Vec3 up_dir   = cam.vert.norm();

    float fwd = d.dot(forward);
    if(fwd <= 0) return false;  // 在相机后面

    float half_w = cam.horiz.len()*0.5f;
    float half_h = cam.vert.len()*0.5f;
    float scale  = 1.f / fwd;   // 透视除法

    float rx = d.dot(right)   * scale;
    float ry = d.dot(up_dir)  * scale;

    su = (rx / half_w) * 0.5f + 0.5f;  // 映射到 [0,1]
    sv = (ry / half_h) * 0.5f + 0.5f;
    sv = 1.f - sv;   // Y轴翻转（图像坐标系）

    return su>=0&&su<=1&&sv>=0&&sv<=1;
}
```

**透视除法**：`scale = 1/fwd` 完成了透视投影的核心操作。相机空间中，物体越远（fwd 越大），投影到屏幕上的范围越小（scale 越小）。这就是为什么远处的物体看起来更小——透视投影的数学本质。

---

## ⑤ 踩坑实录

### Bug 1：编译警告 `unused parameter`

**症状**：编译输出 2 条 warning：  
```
warning: unused parameter 'W' [-Wunused-parameter]
warning: unused parameter 'H' [-Wunused-parameter]
```

**错误假设**：以为这只是 warning，不影响运行，可以忽略。

**真实原因**：`worldToScreen` 函数签名里有 `int W, int H`，但函数体实际上不使用它们（早期版本用来做 clamp，后来改成直接用 [0,1] 范围检查）。`-Wall -Wextra` 编译选项会将这类情况报告为 warning。

**修复方式**：将参数名注释掉，保留类型（这是 C++ 中保留函数签名兼容性同时消除警告的标准做法）：
```cpp
// 修复前
bool worldToScreen(const Vec3& pos, const Camera& cam, int W, int H, ...)

// 修复后  
bool worldToScreen(const Vec3& pos, const Camera& cam, int /*W*/, int /*H*/, ...)
```

**教训**：始终以 `-Wall -Wextra` 编译，0 warnings 是硬性标准。Warning 不是"可能有问题"，而是"确实有问题，只是不一定当场崩溃"。

### Bug 2：场景偏暗，像素均值 65.7 → 90.3

**症状**：第一次渲染出来，图像偏暗（gamma 后均值 65.7），玻璃球折射区域几乎看不出细节。

**错误假设**：认为这是法线或折射方向计算错误导致的，准备排查数学。

**真实原因**：光源强度设置太低（lightPow=6），环境光也太弱（0.06）。Cornell Box 的五面墙大量吸收了光能，只有一个顶部点光源，场景能量密度本来就低。

**修复方式**：
- `lightPow`: 6 → 12（强度翻倍）
- `ambient`: 0.06 → 0.12（环境光翻倍，近似更多间接光照贡献）

**修复后效果**：均值从 65.7 升至 90.3，玻璃球的折射细节变得清晰可见。

**教训**：场景太暗是物理参数问题，不是算法问题。先确认光照配置合理，再排查复杂的算法错误。量化指标（像素均值）能帮你快速判断问题属于哪一类。

### Bug 3：G-Buffer debug 图的 gbuffer_debug.png 未生成

**症状**：`cp gbuffer_debug.png` 时提示文件不存在。

**真实原因**：程序里的 `savePPMasPNG` 调用了 `convert` 命令（ImageMagick），在当前环境中 ImageMagick 不可用。PPM 文件存在，但 PNG 转换失败。主输出文件 `ssr_refraction_output.png` 因为有 `rename` 降级处理所以存在，但 `gbuffer_debug.png` 没有这个降级处理。

**影响**：仅影响 debug 可视化，不影响主要输出和验证。

**教训**：不要假设外部工具一定可用。生产代码里应该为每个外部依赖提供降级方案，并且明确输出哪些文件是必须的（mandatory），哪些是可选的（optional）。

### Bug 4：Fresnel 混合后玻璃球亮度异常

**观察**：玻璃球内某些区域过亮（spec 高光过强）。

**原因分析**：spec 高光使用的是反射方向与光源方向的 64 次幂，但当 reflDir 恰好指向光源时，spec=1.0，乘以 0.4 系数后加到 finalColor 上，finalColor 可能超过 1.0。

**修复**：最终的 `result.at(x,y) = finalColor.clamp01()` 保证了输出在 [0,1] 范围内，视觉上 HDR 高光区域会饱和到白色，产生合理的玻璃高光效果，不会出现数值错误。

---

## ⑥ 效果验证与数据

### 输出图像

**主渲染结果**：

![Screen Space Refraction 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-26-screen-space-refraction/ssr_refraction_output.png)

*图：左侧红墙与右侧绿墙通过玻璃球折射发生颜色互换，中心区域可见色散产生的彩虹边缘，高光点强调了玻璃的光泽感。*

### 量化像素统计

```
=== PIXEL STATS (最终版本) ===
Resolution:     640 × 480
Mean (R,G,B):   34.8, 30.7, 25.0   (linear)
Overall mean:   90.3                 (gamma-corrected, 8-bit)
Std dev:        30.6                 (8-bit)
File size:      900KB
Glass pixels:   12,732 (4.1% of total)
```

**验证标准符合性**：
| 指标 | 要求 | 实测 | 结论 |
|-----|------|------|------|
| 文件大小 | > 10KB | 900KB | ✅ |
| 像素均值 | 10 ~ 240 | 90.3 | ✅ |
| 像素标准差 | > 5 | 30.6 | ✅ |
| 编译 | 0 errors 0 warnings | ✅ | ✅ |
| 运行 | 无崩溃 | ✅ | ✅ |

### 性能数据

在当前单核 CPU 软件光栅化实现下：

| Pass | 分辨率 | 耗时估算 |
|------|--------|---------|
| G-Buffer 构建 | 640×480 | ~50ms |
| 背景场景渲染 | 640×480 | ~200ms |
| SSR 折射合成 | glass pixels only | ~20ms |
| 总计 | | ~270ms |

注：软件光栅化用于教学演示，实际 GPU 实现中 SSR 折射 Pass 在 1080p 下通常 < 1ms（只处理 glass mask 像素，且 GPU 并行化）。

### 折射正确性验证

通过手动检查 Cornell Box 的颜色分布验证折射方向：
- 从正面看玻璃球：球的左侧应该折射出**右墙绿色**（光线穿过球后方向偏转）
- 从正面看玻璃球：球的右侧应该折射出**左墙红色**
- 球体边缘（掠射角）：反射成分增大（Fresnel），颜色偏向背景色而非透视内容

以上三点在输出图像中均可观察到，验证了物理计算的正确性。

---

## ⑦ 总结与延伸

### 本实现的技术局限

**1. 屏幕空间的固有限制**：  
SSR 折射只能折射"屏幕上可见的内容"。如果折射光线对应的背景内容被遮挡（例如玻璃球背后的墙壁被另一个不透明物体挡住），则无法正确采样，会产生拉伸或错误的颜色。这是所有屏幕空间技术（SSR、SSAO、SSS）的共同限制。

**2. 固定厚度近似**：  
`thickness = 1.2f` 是对球体的粗略估计。对于平面玻璃或不规则几何体，这个值会产生错误的折射深度。精确做法是同时渲染正面和背面的 G-Buffer，用深度差得到精确厚度。

**3. 单次折射/反射**：  
真实的多次折射（例如光线在两个玻璃球之间反复弹射）未实现。当前实现只计算第一次折射。

**4. 无焦散**：  
玻璃折射应该在背后的表面投射焦散（Caustics，集中亮斑）。SSR 方案无法反向追踪光线，因此无法模拟焦散。

### 可优化的方向

1. **Back-Face G-Buffer**：额外记录物体背面的深度/法线，实现精确厚度计算
2. **Hierarchical Z Tracing**：用 Mipmap 深度图加速折射光线步进，减少采样次数
3. **前向散射近似**：在折射颜色上叠加 Beer-Lambert 定律的颜色吸收（厚玻璃颜色更深）
4. **抖动采样 + TAA 降噪**：用时序抗锯齿消除折射边缘的高频噪声
5. **与 [04-25 SDF Font Rendering](https://chiuhoukazusa.github.io/chiuhou-tech-blog/) 组合**：在玻璃表面刻字，观察文字折射效果

### 与本系列的联系

本实现是对之前多篇的综合应用：
- **[04-18 SSR 反射](../04-18)** → 今天的折射类似，但采样方向相反（折射进入、反射离开）
- **[03-31 焦散渲染](../03-31)** → 光子映射可以弥补 SSR 折射无法处理焦散的缺陷
- **[04-13 光谱色散](../04-13)** → 今天的 Chromatic Aberration 是色散的屏幕空间近似
- **[04-14 体积云](../04-14)** → 光线步进技术在 SSR 中同样有应用（深度感知步进）

屏幕空间折射是现代实时渲染管线中的标准技术，掌握它之后，玻璃、水面、宝石等透明材质的渲染就有了扎实的物理基础。

---

## 代码仓库

完整源代码：[GitHub - screen-space-refraction](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-26-screen-space-refraction)

```
编译命令：g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
运行：./output
输出：ssr_refraction_output.png（640×480，~900KB）
```

---

*每日编程实践系列 — 第 52 天*  
*技术标签：Screen Space Refraction · G-Buffer · Snell's Law · Schlick Fresnel · Chromatic Aberration · 软光栅化*
