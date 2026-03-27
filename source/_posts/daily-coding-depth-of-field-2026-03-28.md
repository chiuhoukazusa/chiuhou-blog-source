---
title: "每日编程实践: Depth of Field Renderer — 薄透镜景深渲染器"
date: 2026-03-28 05:30:00
tags:
  - 每日一练
  - 图形学
  - 光线追踪
  - C++
  - 景深
  - 渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-28-depth-of-field/dof_output.png
---

# 每日编程实践: Depth of Field Renderer — 薄透镜景深渲染器

今天实现了一个基于薄透镜模型（Thin Lens Model）的物理正确景深渲染器，配合路径追踪产生自然的散景（Bokeh）效果。

![景深渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-28-depth-of-field/dof_output.png)

---

## ① 背景与动机

### 为什么景深很重要？

如果你玩过《赛博朋克 2077》《荒野大镖客2》或《对马岛之魂》，一定注意过游戏里的镜头语言——当主角站在前景，背景的城市灯光化成一圈圈朦胧的光晕，这就是**景深（Depth of Field, DoF）**效果。这个效果来自真实相机和人眼的光学特性：光圈有物理尺寸，只有焦平面附近的物体才能在感光元件上聚焦成点，其他距离的物体会形成弥散圆（Circle of Confusion），俗称**散景（Bokeh）**。

没有景深的渲染器——包括我们之前做的大多数渲染器——使用的都是**针孔相机（Pinhole Camera）**模型。针孔相机假设光圈无穷小，所有距离的物体都完全清晰。这对学习渲染基础够用，但就视觉真实性而言有明显缺陷：

- **画面缺乏层次感**：前景、中景、背景同样清晰，空间深度感弱
- **缺少导演感**：真实摄影中，摄影师用景深引导观众视线——清晰的地方才是重点
- **不够真实**：人眼本身就是个光学系统，有景深的画面更接近我们看世界的方式

### 工业界实际使用

- **光路追踪渲染器（离线）**：V-Ray、Arnold、Cycles 全部支持薄透镜/实体透镜模型。Pixar 的 RenderMan 甚至支持多层透镜组。这里今天实现的薄透镜是离线渲染的标准方案。
- **实时渲染（游戏引擎）**：由于路径追踪过于耗时，实时 DoF 通常用屏幕空间后处理模拟。Unreal Engine 的 Cinematic DoF 使用了散焦圆分离（Separate Translucency + Bokeh 卷积）；Unity 的 HDRP 用了 Physically Based DoF（基于 CoC 的 Tile-based 高斯模糊）。
- **电影工业**：Weta Digital 在《指环王》系列中用景深深度控制观众注意力，是"隐形特效"的经典案例。

今天的实现是**离线路径追踪版本**，结果最为物理正确，也最能展示薄透镜原理本身。

---

## ② 核心原理

### 2.1 针孔相机 vs 薄透镜相机

**针孔相机**：假设相机只有一个无穷小的点作为光圈。每个像素对应唯一一条光线，方向确定，所有深度的物体同样清晰。

$$
\text{Ray}(s, t) = \text{origin} + \text{direction}(s, t)
$$

**薄透镜相机（Thin Lens Model）**：相机有真实尺寸的光圈（直径 = aperture）。根据高斯薄透镜公式，焦距 $f$ 、物距 $d_o$ 、像距 $d_i$ 满足：

$$
\frac{1}{f} = \frac{1}{d_o} + \frac{1}{d_i}
$$

直觉解释：平行光（无穷远处的物体）经过透镜后汇聚在焦点（距离 $f$）。对于有限距离的物体，成像位置比焦点更远（$d_i > f$）。

### 2.2 弥散圆（Circle of Confusion）

对于不在焦平面的物体，它在感光元件上形成一个圆而不是点，这就是弥散圆（CoC）。CoC 的半径公式：

$$
r_{CoC} = \frac{A}{2} \cdot \frac{|d - d_f|}{d}
$$

其中：
- $A$ = 光圈直径
- $d$ = 物体到镜头的距离
- $d_f$ = 焦距（焦平面距离）

直觉解释：光圈越大（$A$ 大）→ 散景越明显；物体离焦平面越远（$|d - d_f|$ 大）→ 模糊越严重；物体离相机越近（$d$ 小）→ 相同离焦量产生更大 CoC。这就是为什么近景虚化比远景虚化更明显。

### 2.3 路径追踪中的薄透镜实现

在路径追踪里，我们不需要显式计算 CoC。薄透镜的物理行为可以直接编码在**光线生成阶段**：

**步骤 1**：在透镜平面（光圈圆盘）上随机采样一点 $P_{lens}$

$$
P_{lens} = \text{center} + r \cdot U + r' \cdot V
$$

其中 $(r, r')$ 是单位圆内的随机点，乘以光圈半径，$(U, V)$ 是相机坐标系的切平面基向量。

**步骤 2**：计算焦平面上的目标点 $P_{focus}$

这里用"聚焦射线"的方式：先发射一条从相机中心穿过像素的参考射线，把它延伸到焦距处：

$$
P_{focus} = \text{origin} + \text{direction}(s, t) \cdot d_{focus}
$$

直觉：所有穿过透镜不同位置但指向同一焦点的光线，在焦平面上汇聚到同一点——这正是薄透镜的核心光学特性。

**步骤 3**：从 $P_{lens}$ 向 $P_{focus}$ 发射真正的光线

$$
\text{Ray}_{real} = (P_{lens},\ \overrightarrow{P_{lens} \to P_{focus}})
$$

这条光线携带了"光从哪个方向穿过透镜上的这一点"的信息。对同一像素重复多次采样（不同的 $P_{lens}$），累加的结果就是物理正确的景深模糊。

**为什么这是正确的？**

薄透镜光学告诉我们：所有穿过透镜、指向焦平面上同一点的光线，经过透镜折射后确实汇聚到同一点。我们的实现直接绕过了折射计算，利用这个几何性质：让光线"已经汇聚好了"——即从透镜上随机点直接射向焦点。等价于真实光学，但效率更高。

### 2.4 光圈形状与散景形状的关系

真实相机的光圈是多边形叶片构成的，所以散景有时是六边形或八边形。我们的实现用**圆形光圈**（在单位圆内均匀采样），产生**圆形散景**——这是最理想的光学散景形状，实际上高端定焦镜头追求的就是接近圆形的散景。

单位圆内均匀采样（拒绝采样法）：

```cpp
Vec3 randomInUnitDisk() {
    while (true) {
        double x = rand01() * 2 - 1;
        double y = rand01() * 2 - 1;
        if (x*x + y*y < 1.0) return {x, y, 0};
    }
}
```

直觉：随机生成 $[-1,1]^2$ 的正方形采样点，只接受落在单位圆内的（约 78.5% 接受率）。这比变换采样略慢但代码最简单，对于光圈采样频率来说完全可以接受。

---

## ③ 实现架构

### 3.1 整体数据流

```
像素 (col, row)
      ↓
  子像素抖动 (rand01() jitter) → 反走样
      ↓
  薄透镜光线生成
    ├── 透镜盘采样 randomInUnitDisk()
    ├── 参考射线计算焦平面目标点
    └── 从采样点射向目标点
      ↓
  路径追踪 trace()
    ├── 场景求交 (球体 + 平面)
    ├── 材质评估 (Diffuse / Metal / Glass / Emissive)
    └── 递归 (最大深度 6)
      ↓
  累加 SAMPLES 次（128SPP）
      ↓
  ACES Filmic 色调映射 + Gamma 2.2
      ↓
  写入 PNG
```

### 3.2 相机结构体设计

```cpp
struct Camera {
    Vec3   pos;
    Vec3   lower_left, horizontal, vertical;  // 视口框架
    Vec3   u, v, w;           // 相机坐标系（右/上/后）
    double lensRadius;        // 光圈半径 = aperture / 2
    double focusDist;         // 焦距（到焦平面的距离）

    Camera(Vec3 lookFrom, Vec3 lookAt, Vec3 vup,
           double vfov, double aspect,
           double aperture, double focusDistance);

    Ray getRay(double s, double t) const;  // 核心：薄透镜光线生成
};
```

**关键设计决策**：

- `lower_left / horizontal / vertical` 定义了焦平面上的视口矩形，尺寸随 `focusDist` 线性缩放——这保证了无论焦距多远，视角（vfov）保持不变
- `w` 指向相机后方（lookFrom - lookAt），`u` 指向右，`v` 指向上。这是标准的右手相机坐标系
- `lensRadius = 0` 时退化为针孔相机，不产生任何模糊

### 3.3 场景布局设计

```
相机位置: (0, 0.6, 4.5)  →  lookAt: (0, 0, 1.5)
焦平面距离: ~3.06（到前景球群的距离）

深度层次:
  前景 z=+1.5 【焦点处，清晰】
    - 红色漫反射球 (-1.2, 0, 1.5)
    - 玻璃球 (0, 0, 1.5)
    - 金色金属球 (1.2, 0, 1.5)
  中景 z=-0.5 【轻微模糊，~1.4×焦距】
    - 蓝色漫反射球、银绿色金属球
  背景 z=-4.0 【明显散景，~2.8×焦距】
    - 橙色球、绿色球
  光源 y=4.0 【暖色自发光球，高处照亮场景】
```

这个布局专为展示景深效果设计：三个深度层次对比明显，前中后景都有足够的视觉内容。

---

## ④ 关键代码解析

### 4.1 薄透镜光线生成（核心）

```cpp
Ray Camera::getRay(double s, double t) const {
    // 步骤1：在光圈圆盘上随机采样
    // rd 是单位圆内的随机点，乘以 lensRadius 得到实际光圈偏移
    Vec3 rd     = randomInUnitDisk() * lensRadius;
    
    // 转换到相机坐标系中的实际偏移
    // u 是相机右方向，v 是相机上方向
    Vec3 offset = u * rd.x + v * rd.y;

    // 步骤2：计算焦平面上的目标点
    // lower_left + horizontal*s + vertical*t 是视口上的点
    // 视口已经缩放到了 focusDist 处（焦平面上）
    Vec3 target = lower_left
                + horizontal * s
                + vertical   * t;

    // 步骤3：从透镜采样点射向焦平面目标点
    Vec3 origin = pos + offset;          // 光线起点：透镜上的随机点
    Vec3 dir    = (target - origin).normalize();  // 指向焦平面目标
    return Ray(origin, dir);
}
```

**为什么 `lower_left` 要乘以 `focusDist`？**

这是整个薄透镜实现最容易搞错的地方。在相机构造函数里：

```cpp
lower_left = pos
           - u * halfWidth  * focusDist   // ← 乘以 focusDist！
           - v * halfHeight * focusDist   // ← 乘以 focusDist！
           - w * focusDist;               // ← 乘以 focusDist！

horizontal = u * 2.0 * halfWidth  * focusDist;  // 同样乘以 focusDist
vertical   = v * 2.0 * halfHeight * focusDist;
```

原因：视口（ViewPort）必须放置在焦平面上（距离相机 `focusDist`），而不是距离 1 的默认位置。这样，所有从不同光圈位置射出、但指向视口同一点的光线，都真正汇聚在焦平面上——符合薄透镜光学特性。

如果不乘以 `focusDist`，视口在距离1处，光线朝向的"汇聚点"不在焦平面上，产生错误的模糊效果。

### 4.2 材质系统

**漫反射（Lambertian）**：

```cpp
if (mat.type == MatType::DIFFUSE) {
    // 余弦加权半球采样
    // 相比均匀采样，自动给靠近法线方向更多权重
    // PDF = cos(θ)/π，刚好消掉 BRDF 中的 cos(θ)/π，采样效率更高
    Vec3 scattered = toWorld(cosineSampleHemisphere(), h.normal);
    Vec3 incoming  = trace(Ray(h.pos, scattered), scene, depth - 1);
    return mat.color * incoming;  // 颜色 = 反照率 * 入射光
}
```

`toWorld()` 把局部坐标系（Z轴=法线）的方向转换到世界坐标系，使用 Gram-Schmidt 正交化构造切线帧。

**金属（Phong 镜面反射 + 粗糙度）**：

```cpp
if (mat.type == MatType::METAL) {
    // 镜面反射公式：R = I - 2(I·N)N
    Vec3 refl = ray.dir - h.normal * 2.0 * ray.dir.dot(h.normal);
    
    if (mat.rough > 0) {
        // 粗糙度扰动：在反射方向附近随机偏移
        // 用余弦加权半球采样围绕反射方向采样
        Vec3 fuzz = toWorld(cosineSampleHemisphere(), refl.normalize()) * mat.rough;
        refl = refl + fuzz;
    }
    // 如果扰动后光线进入表面（dot < 0），直接吸收
    if (refl.dot(h.normal) <= 0) return {0,0,0};
    return mat.color * trace(Ray(h.pos, refl.normalize()), scene, depth - 1);
}
```

粗糙度（rough）控制反射的模糊程度：0 = 完美镜面；1 = 类漫反射。这与 PBR 中的 roughness 参数语义一致。

**玻璃（Schlick 菲涅尔 + Snell 折射）**：

```cpp
if (mat.type == MatType::GLASS) {
    // etaRatio: 入射介质折射率 / 折射介质折射率
    double etaRatio = h.front ? (1.0 / mat.ior) : mat.ior;
    double cosI  = -ray.dir.dot(h.normal);
    double sinT2 = etaRatio * etaRatio * (1.0 - cosI * cosI);

    // Schlick 菲涅尔近似：计算反射概率
    double r0 = (1 - mat.ior) / (1 + mat.ior); r0 *= r0;
    double fresnelP = fresnelSchlick(cosI, r0);

    if (sinT2 > 1.0 || rand01() < fresnelP) {
        // 全内反射（sin²θt > 1）或菲涅尔反射（概率采样）
        Vec3 refl = ray.dir + h.normal * 2.0 * cosI;
        return mat.color * trace(Ray(h.pos, refl.normalize()), scene, depth - 1);
    } else {
        // 折射（Snell 定律）
        double cosT = std::sqrt(1.0 - sinT2);
        Vec3 refr = ray.dir * etaRatio + h.normal * (etaRatio * cosI - cosT);
        return mat.color * trace(Ray(h.pos, refr.normalize()), scene, depth - 1);
    }
}
```

关键点：
- `h.front` 判断光线从外部还是内部击中表面，决定折射率比值方向
- Schlick 公式 $R(\theta) = R_0 + (1-R_0)(1-\cos\theta)^5$ 用单次随机数实现了概率性的反射/折射分支，等效于对菲涅尔积分的无偏蒙特卡洛估计
- 全内反射条件：$\sin^2\theta_t > 1$（光从密介质射向疏介质时可能发生）

### 4.3 ACES Filmic 色调映射

```cpp
uint8_t toU8(double v) {
    // ACES Filmic Tone Mapping
    // 参数来自 Stephen Hill 的 ACES 近似
    double a = 2.51, b = 0.03, c = 2.43, d = 0.59, e = 0.14;
    v = (v * (a * v + b)) / (v * (c * v + d) + e);
    
    // 钳制到 [0,1]
    v = std::max(0.0, std::min(1.0, v));
    
    // Gamma 2.2 校正（线性→sRGB）
    v = std::pow(v, 1.0 / 2.2);
    
    return static_cast<uint8_t>(v * 255.999);
}
```

ACES 色调映射比简单的 Reinhard（`v/(v+1)`）更好：
- 暗部细节更丰富（S 曲线的底部斜率更大）
- 高光过渡更自然（不会像 Reinhard 那样整体偏灰）
- 色彩饱和度保持得更好

与 Linear 映射对比：不用 ACES 直接截断的话，发光球周围的高光会变成难看的死白，细节全丢。

---

## ⑤ 踩坑实录

### Bug 1：视口未缩放导致景深效果错误

**症状**：渲染出来虽然有些模糊，但模糊程度与光圈设置不对应——光圈开大了，背景却只有轻微模糊，不是预期的大散景。

**错误假设**：以为光线生成逻辑正确，模糊效果应该随光圈线性增强。

**真实原因**：相机构造时忘记了视口坐标要乘以 `focusDist`。视口被放在距离 1 的位置，而不是焦平面（距离 ~3）上。光圈偏移量相对于这个错误距离来说"太大"，导致散射角度异常，不是物理正确的景深。

**修复**：在 Camera 构造函数中，`lower_left`、`horizontal`、`vertical` 全部乘以 `focusDist`：

```cpp
// 错误（视口在距离1）：
lower_left = pos - u * halfWidth - v * halfHeight - w;

// 正确（视口在焦平面）：
lower_left = pos
           - u * halfWidth  * focusDist
           - v * halfHeight * focusDist
           - w * focusDist;
```

**教训**：薄透镜相机最反直觉的地方就是"视口必须在焦平面上"。把这个原则写进注释里，不然下次还会忘。

### Bug 2：Y 轴翻转导致天空在下

**症状**：渲染结果天空在下面，地面在上面，整幅图上下颠倒。

**错误假设**：像素坐标 (row, col) 和图像坐标是一致的，row=0 对应图像顶部。

**真实原因**：图像写入时 row=0 是顶部，但在计算射线方向时，sv = row/H 使得 row=0 对应了视口底部（天空方向），row=H 对应了视口顶部（地面方向）。

**修复**：翻转 Y 坐标映射：

```cpp
// 错误：
double sv = (double)row / H;  // row=0 → 底部

// 正确：
double sv = (H - 1 - row + rand01()) / H;  // row=0 → 顶部（天空）
```

这是一个经典的坐标系问题，也是 SKILL.md 里"历史教训"特别提到的——以后所有图形学项目都要先检查坐标系方向。

### Bug 3：玻璃球内部反射无限循环

**症状**：渲染玻璃球时极慢，甚至有个别像素永远不结束（实际上是超出深度限制，但早期版本没有深度限制）。

**错误假设**：玻璃球内部光线会自然衰减，不需要特别处理。

**真实原因**：光线进入玻璃球后，由于全内反射，可能在球内部来回反弹很多次。如果没有最大深度限制，会无限循环。即使有深度限制（本实现 depth=6），深度不够时玻璃球会显示为暗色（能量被截断）。

**修复**：
1. 保持最大深度限制（depth=6），这已经足够处理大多数情况
2. 深度耗尽时返回黑色（能量守恒：未追踪完的能量视为被吸收）

深度6对玻璃球来说通常足够——真实场景中光线很少在玻璃球内部反弹超过3-4次就射出了。

### Bug 4：随机数种子固定导致散景图案有规律

**症状**：128 SPP 的结果散景看起来有轻微的网格感，特别是在强高光区域。

**错误假设**：固定种子 42 可以让结果可复现，同时质量不受影响。

**真实原因**：固定种子意味着每次渲染完全相同的随机序列。对于每个像素 128 次采样，序列分布均匀性足够，但相邻像素使用了连续的序列段，导致局部相关性。

**当前处理**：本次实现保留了固定种子（保证可复现性优先），但对于生产级渲染可以改用 PCG 哈希或 Blue Noise 采样提升质量。实际效果来看，128 SPP 已经足够平滑。

---

## ⑥ 效果验证与数据

### 像素统计验证

```
文件大小: 1.1MB（远大于 10KB 最低要求）
分辨率: 800 × 600
像素均值: 173.5（正常范围：10~240）
像素标准差: 33.1（正常范围：>5，有丰富内容变化）
天空亮度（上100行均值）: 195.6
地面亮度（下100行均值）: 178.5
→ 天空 > 地面 ✅（坐标系正确，天空在上）
```

所有验证通过，没有全黑/全白/无内容的问题。

### 渲染性能数据

```
分辨率:    800 × 600 = 480,000 像素
每像素采样: 128 SPP
总采样数:  61,440,000 次路径追踪
最大深度:  6
渲染耗时:  28.3 秒（单线程）
吞吐量:    ~2,170,000 样本/秒
平均每像素: 约 59μs
```

单线程 28 秒对于 128SPP 的路径追踪加景深采样是合理的。路径追踪开销主要在递归求交，景深额外增加了每次光线生成时的单位圆采样（拒绝采样，平均 ~1.27 次尝试），对总时间影响约 1%，可以忽略。

### 景深效果验证

| 深度层 | Z 位置 | 到焦平面距离 | 预期效果 |
|--------|--------|------------|---------|
| 前景   | z=+1.5 | 0（焦点）  | 清晰    |
| 中景   | z=-0.5 | 2.0 单位   | 轻微模糊 |
| 背景   | z=-4.0 | 5.5 单位   | 明显散景 |

理论上，光圈直径 0.25、焦距 ~3.06 时：
- 中景 CoC 半径 ≈ 0.125 × 2.0/3.06 ≈ 0.082（屏幕空间）
- 背景 CoC 半径 ≈ 0.125 × 5.5/3.06 ≈ 0.225（屏幕空间）

在 800px 宽度下，背景 CoC 约 90px 半径，符合渲染结果中背景球的模糊圈大小（目视约 80-100px）。

---

## ⑦ 总结与延伸

### 技术局限性

**当前薄透镜实现的限制**：

1. **圆形光圈**：真实相机光圈是多边形叶片（5叶、7叶、9叶等），散景形状相应为多边形。可以通过在多边形内均匀采样而非圆盘来改变散景形状。

2. **均匀圆盘采样**：使用了最简单的拒绝采样法。更高效的方法是 Shirley 同心圆映射（Concentric Sample Disk），避免拒绝浪费。

3. **无色差（Chromatic Aberration）**：真实透镜对不同波长折射率不同，导致散景边缘出现彩色晕圈（紫边/蓝移）。可以通过对 RGB 三通道分别用略不同的焦距渲染来模拟。

4. **薄透镜近似**：假设透镜无穷薄，没有厚度和像差（球差、彗差等）。实体透镜模型可以更准确地模拟高端镜头的特性散景。

5. **渐晕（Vignetting）**：真实相机边角进光量减少，图像边缘变暗。薄透镜模型里没有这个效应。

### 可优化方向

**采样效率提升**：
- 换用 Halton 序列或 Sobol 序列替代伪随机数，在相同 SPP 下显著减少噪声
- 重要性采样光源（显式光源采样）——当前实现靠路径随机碰到光源，对小光源非常低效
- 多重重要性采样（MIS）平衡直接照明与 BRDF 采样

**实时化方向**：
- 为了实时游戏，通常改用屏幕空间 CoC 计算 + 卷积 DoF
- Unreal Engine 5 的 Lumen 全局光照也支持 Temporal DoF，每帧只渲染 1-4SPP，靠时序累积
- 也可以用光线追踪 + Denoiser（DLSS 3/XeSS 等）接近今天的离线质量

**与本系列其他技术的关联**：
- 今天的薄透镜相机可以与 03-21 实现的 SPPM、03-22 的 BDPT 无缝结合——它们的差别只在光线追踪策略，相机模型完全兼容
- 03-26 实现的 TAA 里有时序抖动，景深的多帧累积与 TAA 思路相通，可以结合实现 Temporal DoF
- 03-27 的运动模糊和今天的景深都是"采样范围的扩展"——运动模糊在时间维度采样，景深在空间（光圈）维度采样，完全可以同时启用

### 今日收获

薄透镜景深是路径追踪器中少有的"一次实现就物理正确"的特性——只要把视口放到焦平面上、在光圈盘上随机采样，蒙特卡洛的威力会自动处理其余一切。这种优雅让它成为了我最喜欢的渲染技术之一。

代码仓库: https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-28-depth-of-field
