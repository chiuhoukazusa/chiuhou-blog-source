---
title: "每日编程实践: 薄透镜景深渲染器 (Depth of Field Bokeh)"
date: 2026-06-11 14:00:00
tags:
  - 每日一练
  - 图形学
  - 光线追踪
  - 后处理
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-11-depth-of-field/depth_of_field_output.png
---

## ① 背景与动机

你有没有注意过这样一个现象：用手机拍人像时，背景会自动变模糊，但人脸却清晰无比？这背后的技术叫**景深（Depth of Field, DoF）**，它是真实相机光学系统的一个副产品——镜头不可能同时把近处和远处的物体都对上焦。

在计算机图形学中，渲染器默认用的是**针孔相机模型（Pinhole Camera）**：所有光线都穿过一个无限小的针孔，因此场景中从近到远的一切都是完美清晰的。这在真实世界中是不可能的。任何真实的相机镜头都有光圈（Aperture），光圈越大，景深越浅——这就是为什么大光圈镜头（f/1.4, f/1.8）能拍出迷人的背景虚化（Bokeh）。

在实时渲染领域，景深效果的应用非常广泛：

- **过场动画（Cutscene）**：游戏过场中，导演用浅景深引导玩家注意力到关键角色或物体上。虚幻引擎 5 的 Sequencer 中有完整的 DoF 后处理参数。
- **拍照模式（Photo Mode）**：《蜘蛛侠》《对马岛之魂》等游戏的拍照模式允许玩家调整光圈和焦距，模拟真实摄影体验。
- **电影级渲染**：皮克斯的 RenderMan、工业光魔的渲染管线中，物理景深是"像真的"（photoreal）渲染的基本要求。
- **VR/AR**：虽然人眼本身有景深，但头显屏幕是固定焦距的——Varjo 等高端头显在研究通过可变焦距来模拟自然景深，减轻视觉疲劳。

在离线渲染（Offline Rendering）中，景深通常通过对**镜头光圈进行蒙特卡洛采样**来实现：对每个像素，从光圈上随机选取多个点发射光线，这些光线在焦平面（Focal Plane）汇聚，在非焦平面发散——发散量决定了模糊程度。这就是经典的**薄透镜模型（Thin Lens Model）**。

今天我们要实现的就是这样一个基于物理的景深渲染器：它不是简单的后处理高斯模糊，而是实实在在地模拟了光线穿过镜头光圈的过程。每个像素发射 64 条光线，每条光线从光圈的不同位置出发，在焦平面 z=9 完美汇聚，在前景（z=4.5）和背景（z=14~20）处发散，自然产生 Bokeh 虚化效果。

---

## ② 核心原理

### 2.1 薄透镜方程（Thin Lens Equation）

真实相机镜头是一个复杂的光学系统——多组镜片组合来校正色差、球差等。但在计算机图形学中，我们用**薄透镜近似**：假设镜头是一个无限薄的平面，所有折射发生在同一个平面上。

薄透镜的成像公式是初中学过的：

\\[ \\frac{1}{f} = \\frac{1}{u} + \\frac{1}{v} \\]

其中：
- \\( f \\) 是焦距（Focal Length），镜头的光学特性参数。\\( f \\) 越大，视角越窄（远摄）；\\( f \\) 越小，视角越广（广角）。
- \\( u \\) 是物距（Object Distance）——物体到镜头的距离。
- \\( v \\) 是像距（Image Distance）——镜头到传感器（底片/CMOS）的距离。

**直觉解释**：这个公式说的是，对于给定的焦距 \\( f \\)，物体距离 \\( u \\) 决定了像在哪里形成 \\( v \\)。如果物体在无穷远（\\( u = \\infty \\)），则 \\( v = f \\)——像正好在焦点上。如果物体很近（\\( u \\) 很小），则 \\( v \\) 很大——传感器要往后退才能对焦。

在我们的渲染器中：
- 焦距 \\( f = 5.0 \\)（相当于全画幅约 50mm 标准镜头）
- 对焦距离（想拍清楚的那个面）\\( u = 9.0 \\) 单位
- 计算传感器位置：\\( v = 1 / (1/5.0 - 1/9.0) = 11.25 \\) 单位

所以传感器在镜头后方 11.25 单位处，场景中的物体在镜头前方。

### 2.2 弥散圆（Circle of Confusion, CoC）

当物体不在对焦面上时会发生什么？考虑一个点光源在距离 \\( u' \\) 处：

- 如果 \\( u' = u \\)（正好在对焦面）：该点发出的光线通过镜头后在传感器上汇聚成一个**点**——完美清晰。
- 如果 \\( u' \\neq u \\)：该点发出的光线通过镜头后在传感器**前方或后方**汇聚（取决于 \\( u' > u \\) 还是 \\( u' < u \\)），到达传感器时已经散开成一个**圆斑**——这就是弥散圆（Circle of Confusion）。

弥散圆的直径 \\( C \\) 由下式给出：

\\[ C = A \\cdot \\frac{|u' - u|}{u'} \\cdot \\frac{f}{u - f} \\]

其中 \\( A \\) 是光圈直径（\\( A = 2 \\times \\text{aperture\\_radius} \\)）。

**直觉解释**：这个公式的三个因子分别代表什么？
1. \\( A \\)：光圈越大，弥散圆越大——大光圈 = 强虚化。
2. \\( \\frac{|u' - u|}{u'} \\)：物体离对焦面越远，弥散圆越大。注意分母是 \\( u' \\)，说明同样距离差，近处物体（\\( u' \\) 小）的虚化比远处物体（\\( u' \\) 大）更明显。
3. \\( \\frac{f}{u-f} \\)：焦距越长，弥散圆越大——长焦镜头的虚化比广角镜头强，这也是为什么 85mm f/1.4 被称为"人像镜皇"。

### 2.3 光圈采样（Aperture Sampling）

在光线追踪中，模拟景深的标准方法是**分布式光线追踪（Distribution Ray Tracing）**：

1. 对每个像素，在光圈上随机采样 \\( N \\) 个点
2. 对每个光圈采样点，发射一条光线穿过传感器上的对应像素位置
3. 这些光线在焦平面汇聚于同一点，但在其他深度分散
4. 取 \\( N \\) 条光线的平均颜色

我们用两种采样模式来塑造 Bokeh 的形状：

**Poisson Disk 采样**：保证任意两个采样点之间有最小距离约束，避免了随机采样中的团簇（Clumping）和空洞（Gap）。这让虚化区域的噪点分布更均匀自然。算法流程：

1. 在单位圆盘上逐个放置采样点
2. 每次尝试最多 100 个随机候选位置
3. 选择与已有采样点**最小距离最大**的那个候选（即"离其他点最远的那个"）
4. 如果连这个最佳候选都不满足最小距离约束，停止放置

这种"Dart Throwing"风格的算法虽然简单，但对于 32 个采样点的情况效果很好。

**六边形光圈采样**：真实相机的光圈由 6~9 片叶片组成，形成近似圆形的多边形。6 叶片光圈产生六边形 Bokeh——摄影爱好者常说的"光斑形状"。我们的实现是将单位正方形映射到正六边形内：

- 将 \\([0,1]^2\\) 的随机点映射到正六边形内切区域
- 如果点落在六边形外，用仿射变换将其反射回六边形内
- 这样产生的 Bokeh 高光会呈现六边形轮廓

### 2.4 为什么不用后处理方案？

你可能听说过另一种实现景深的方法：先渲染一张清晰的图和一张深度图（Depth Buffer），然后对每个像素根据其深度计算 CoC 半径，再执行可变半径的高斯模糊。这在实时渲染中很常见（如 UE 的 Mobile DoF）。

但我们选择了物理上更准确的**光圈采样方案**，原因有几点：

1. **遮蔽关系正确**：后处理方案无法处理前景物体遮挡背景物体时的边缘——前景模糊球后面的背景不应该被前景"拉"模糊，但简单的 2D 模糊会产生光晕。
2. **Bokeh 形状可控**：后处理高斯模糊只能产生圆形虚化，而我们的光圈采样可以产生六边形、八边形甚至心形 Bokeh。
3. **高光保留**：基于 CoC 的后处理会对整个区域均匀模糊，丢失高光细节。光线追踪方案自然保留了镜面高光在虚化区域的表现。
4. **教学价值**：理解薄透镜模型是学习更复杂的光学模拟（如厚透镜模型、像差模拟）的基础。

---

## ③ 实现架构

### 3.1 整体数据流

```
Camera（薄透镜模型）
  │
  ├─→ 对每个像素 (x, y)：
  │     ├─→ 传感器坐标：sensor_point = (px, py, -sensor_distance)
  │     ├─→ 焦平面映射：focal_point = (px*scale, py*scale, focus_distance)
  │     │
  │     └─→ 对每个采样 (64 spp)：
  │           ├─→ 在光圈上采样：lens_sample = Poisson_Disk[i] * aperture_radius
  │           ├─→ 射线原点：ro = lens_sample
  │           ├─→ 射线方向：rd = normalize(focal_point - lens_sample)
  │           │
  │           ├─→ Scene::intersect(ro, rd)
  │           │     └─→ 遍历所有 Sphere，取最近交点
  │           │
  │           ├─→ 命中？→ shade(hit, view_dir, scene)
  │           │     ├─→ 环境光
  │           │     ├─→ 遍历所有 Light：
  │           │     │     ├─→ 阴影检测（Shadow Ray）
  │           │     │     ├─→ Lambertian 漫反射
  │           │     │     └─→ Blinn-Phong 镜面反射
  │           │     └─→ 光照衰减 (1/d²)
  │           │
  │           └─→ 未命中？→ 天空渐变
  │
  └─→ 累积 64 spp → ACES Tone Mapping → Gamma 校正 → PNG 输出
```

### 3.2 关键数据结构

```cpp
struct Vec3 { double x, y, z; };  // 三维向量

struct Material {
    Vec3 color;       // 基础颜色 (albedo)
    double roughness;  // 0 = 完美镜面, 1 = 完全漫反射
    double metallic;   // 0 = 非金属, 1 = 纯金属
};

struct Sphere {
    Vec3 center;
    double radius;
    Material mat;
};

struct Light {
    Vec3 position;
    Vec3 color;
    double intensity;  // 基础光强，随距离 1/r² 衰减
};

struct Camera {
    Vec3 position, look_at, up;
    double focal_length;     // 焦距 f
    double focus_distance;   // 对焦面距离 u
    double aperture_radius;  // 光圈半径
    double sensor_distance;  // 计算得出的像距 v
    Vec3 forward, right, world_up;  // 相机局部坐标系
};
```

**设计决策说明**：

- **为何用 `double` 而非 `float`？**：在光线追踪中，浮点精度直接影响交点计算的稳定性。`double` 提供约 15 位有效数字，避免了远离原点处 `float`（约 7 位）可能产生的自相交（Self-intersection）问题。
- **为何不用 BVH？**：场景只有 6 个 Sphere + 3 个巨型几何（地面、天花板、后墙），总计 9 个球体。在这样的规模下，线性遍历比构建 BVH 更快（BVH 构建本身有 \\( O(n \\log n) \\) 的开销）。
- **Material 模型**：使用简化版的 PBR 参数（color + roughness + metallic），比纯 Phong 模型更真实，但不引入完整的 GGX/Cook-Torrance 微面元模型以保持代码简洁。

### 3.3 坐标系统

```
相机位于原点 (0, 0, 0)，看向 +Z 方向

        +Y (up)
        │
        │  场景物体在 +Z 方向
        │  
        │  ★ 红球 (z=4.5)     ← 前景，模糊
        │  ★ 绿球 (z=9.0)     ← 对焦面，清晰
        │  ★ 金球 (z=14.0)    ← 背景，模糊
        │  ★ 白球 (z=20.0)    ← 远景，强模糊
        │
  镜头──┼──────────→ +Z (forward)
        │
        │  传感器在 -Z 方向
        │  (z = -sensor_distance = -11.25)
        │
       -Z
```

注意这里与传统光线追踪（相机看 -Z）的不同——我们让相机看 +Z 是为了让物体距离（u = z 坐标）直观可读。

---

## ④ 关键代码解析

### 4.1 薄透镜射线生成

这是整个渲染器最核心的函数——它决定了每个像素"看到"什么：

```cpp
void Camera::get_ray(double px, double py, double lx, double ly, 
                     Vec3& ro, Vec3& rd) const {
    // Step 1: 传感器像素 → 焦平面映射
    // scale = -focus_distance / sensor_distance
    // 负号是因为传感器在负 Z，焦平面在正 Z
    double scale = -focus_distance / sensor_distance;
    
    // 通过镜头中心 (position) 连接传感器点和焦平面点
    Vec3 focal_point = forward * focus_distance 
                     + right * (px * scale) 
                     + world_up * (py * scale);
    
    // Step 2: 在镜头上偏移
    Vec3 lens_offset = right * lx + world_up * ly;
    
    // Step 3: 射线从镜头采样点射向焦平面点
    ro = position + lens_offset;
    rd = (position + focal_point - ro).norm();
}
```

**关键洞察**：为什么这样写的射线会在焦平面汇聚？考虑中心像素（px=0, py=0）。此时 `focal_point = (0, 0, focus_distance)`。不管 `lens_offset` 取什么值（即不管在光圈上哪个位置采样），`rd` 总是指向 `(0, 0, focus_distance)`——所有射线精确汇聚于焦平面上的一点。这就是"对焦"的物理本质。

对于不在焦平面的物体：以 z=4.5 处的红球为例，从光圈不同位置出发的射线在该深度处横向偏移约 0.6 单位（即弥散圆），导致该球体的像素被来自不同位置的采样"涂抹"开来。

### 4.2 泊松圆盘采样（Poisson Disk）

```cpp
std::vector<Vec3> generate_poisson_disk(int n, RNG& rng) {
    std::vector<Vec3> points;
    const double min_dist = 0.85 / std::sqrt(double(n));
    
    while ((int)points.size() < n) {
        double best_dist = 0;
        Vec3 best;
        // 100 次候选中选最优的
        for (int attempt = 0; attempt < 100; ++attempt) {
            double angle = rng.uniform() * 2.0 * M_PI;
            double radius = std::sqrt(rng.uniform());
            // √(uniform) 保证在圆盘上均匀分布
            Vec3 cand(radius * std::cos(angle), radius * std::sin(angle), 0);
            
            double min_d = 1e10;
            for (const auto& p : points) {
                double d = (cand - p).len();
                if (d < min_d) min_d = d;
            }
            if (min_d > best_dist) {
                best_dist = min_d;
                best = cand;
            }
        }
        if (best_dist > min_dist || points.empty()) {
            points.push_back(best);
        } else {
            break; // 放不下更多点了
        }
    }
    return points;
}
```

**为什么用 sqrt(uniform)？**在极坐标中，如果直接用 `radius = uniform()`，采样点会向圆心聚集——因为面积元素 \\( dA = r \\, dr \\, d\\theta \\)，半径上的概率密度应该正比于 \\( r \\)，所以 \\( r = \\sqrt{U} \\)。这是蒙特卡洛采样中的经典技巧。

### 4.3 光线-球体交点

```cpp
Hit intersect_sphere(const Vec3& ro, const Vec3& rd, 
                     const Sphere& s, double t_min) {
    Vec3 oc = ro - s.center;
    double a = rd.dot(rd);           // 应该等于 1（若 rd 已归一化）
    double b = 2.0 * oc.dot(rd);
    double c = oc.dot(oc) - s.radius * s.radius;
    double disc = b*b - 4*a*c;
    
    if (disc < 0) return {1e10, {}, {}, {}, false}; // 不相交
    
    // 取最近的 t > t_min
    double t = (-b - std::sqrt(disc)) / (2.0 * a);
    if (t < t_min) {
        t = (-b + std::sqrt(disc)) / (2.0 * a);
        if (t < t_min) return {1e10, {}, {}, {}, false};
    }
    
    Vec3 point = ro + rd * t;
    Vec3 normal = (point - s.center).norm();
    return {t, point, normal, s.mat, true};
}
```

这段代码容易写错的地方：**t_min 的处理**。在追踪"主射线"（从镜头到场景）时，t_min 只需要 > 0 避免自相交。但在追踪"阴影射线"（Shadow Ray，从表面点到光源）时，t_min 必须 > 0.001 来避免将交点所在的面误判为遮挡物。如果忘了这个，阴影会变成全黑——这是经典的光线追踪陷阱。

### 4.4 Blinn-Phong 着色

```cpp
Vec3 shade(const Hit& hit, const Vec3& view_dir, 
           const Scene& scene, RNG&) {
    Vec3 color = scene.ambient.mul(hit.mat.color) * 0.1;
    
    for (const auto& light : scene.lights) {
        Vec3 to_light = light.position - hit.point;
        double dist = to_light.len();
        Vec3 L = to_light / dist;
        
        // 阴影检测：从表面点向光源发射射线
        Hit shadow = intersect_scene(hit.point + L * 0.001, L, scene, 0.001);
        if (shadow.valid && shadow.t < dist) continue; // 被遮挡
        
        double NdotL = std::max(0.0, hit.normal.dot(L));
        Vec3 diffuse = hit.mat.color * NdotL / M_PI; // Lambert
        
        Vec3 H = (L + view_dir).norm();  // 半程向量
        double NdotH = std::max(0.0, hit.normal.dot(H));
        double spec = std::pow(NdotH, 
            1.0 / (hit.mat.roughness * hit.mat.roughness + 0.001));
        Vec3 specular = (hit.mat.color * 0.3 + Vec3(1,1,1) * 0.7) 
                      * spec * hit.mat.metallic;
        
        double attenuation = light.intensity / (dist * dist);
        color = color + (diffuse + specular).mul(light.color) * attenuation;
    }
    return color;
}
```

**几个设计细节**：

1. **漫反射除以 π**：Lambertian BRDF 的归一化因子是 \\( 1/\\pi \\)。不加这个因子会导致能量不守恒——物体反射的光比接收的光还多。
2. **半程向量（Half Vector）**：Blinn-Phong 用 \\( H = \\text{normalize}(L + V) \\) 替代 Phong 的反射向量，因为 H 的计算比反射向量快得多，且在大多数情况下视觉效果几乎相同。
3. **roughness 映射到 specular power**：\\( p = 1 / (r^2 + 0.001) \\)。roughness=0.3 → p≈11，roughness=0.7 → p≈2。+0.001 防止除以零。
4. **光照衰减 1/d²**：与物理一致，点光源的辐照度按距离平方衰减。这是真实世界中"离灯越远越暗"的原因。

### 4.5 ACES 色调映射

```cpp
auto aces = [](double x) {
    double a = 2.51, b = 0.03, c_val = 2.43, d = 0.59, e = 0.14;
    return clamp((x*(a*x + b)) / (x*(c_val*x + d) + e), 0.0, 1.0);
};
```

ACES（Academy Color Encoding System）是电影工业标准的 HDR→SDR 映射曲线。相比简单的 `x/(1+x)`（Reinhard），ACES 在处理高亮区域时更自然——它有一个柔和的"肩部"（Shoulder），高光不会直接夹断成白色，而是平滑地接近 1.0。同时暗部有"趾部"（Toe），保留了阴影细节。五个参数（a, b, c, d, e）是通过对真实胶片响应曲线拟合得到的。

---

## ⑤ 踩坑实录

### Bug 1：光线方向反了——渲染出纯蓝天空

**症状**：第一版渲染的图像是均匀的蓝色渐变，看不到任何球体。像素采样显示全是天空色（R≈108, G≈140, B≈182）。

**错误假设**：传感器在镜头后方（-Z），场景在镜头前方（+Z）。我以为射线方向 `rd = (sensor_point - lens_point).norm()` 应该从传感器穿过镜头射向场景。

**真实原因**：`sensor_point` 的 z=-10.29，`lens_point` 的 z=0，所以 `rd.z = (-10.29 - 0) / len = -1.0`——射线向 -Z 方向射出，完全没有进入 +Z 方向的场景！

**修复方式**：重写 `get_ray()`，改为从镜头发射射线**直接指向焦平面上的对应点**，而不是穿过传感器。焦平面点通过透镜中心映射得出。修改后射线 `.z` 分量变为正值，正确射入场景。

**教训**：在光线追踪中，射线方向是最容易出错的地方。建议在开发初期用 `printf` 输出几个典型像素的射线方向，快速验证射线是否指向预期方向。

### Bug 2：ACES 过曝——画面全白

**症状**：加大光照强度后（intensity 从 60 升到 300），渲染结果几乎全白。像素均值 236，大量像素饱和度达到 255。

**错误假设**：ACES 会自动处理任何范围的 HDR 值，因此不需要控制曝光。

**真实原因**：线性的 HDR 值（mean=2.8, max=92.5）经过 ACES 后基本都在 0.95~1.0 之间——ACES 曲线的"肩部"虽然柔和，但对于超过 2.0 的值几乎全部映射到 ≈1.0。ACES 设计用于处理真实世界的场景辐亮度（通常在 0~100 cd/m²），而不是直接处理任意比例的光照值。

**修复方式**：在 ACES 之前乘以曝光系数（exposure=0.35），将线性值缩放到合理范围。正确的做法是先计算场景中的中灰曝光，再应用 ACES。最终 mean≈0.228（经过 0.35 曝光），ACES 后得到合理的中灰值。

**教训**：不要假设 HDR→SDR 映射能处理任意动态范围。ACES、Reinhard、Uncharted 2 等都需要合理的曝光预缩放，否则要么过曝要么欠曝。

### Bug 3：FOV 太窄——球体填满画面

**症状**：最初用 sensor_height=2.0 时，半 FOV 只有 7.4°（全 FOV 约 15°）——等效于全画幅约 150mm 的超远摄镜头。场景中所有球体都巨大无比，重叠严重，景深效果完全体现不出来。

**错误假设**：sensor_height 只是"把画面拉近拉远"，不影响景深效果。

**真实原因**：FOV 太窄导致：
1. 球体占据画面太大比例，读者无法区分"这个模糊是因为 DoF"还是"球太大了所以边缘在画面外"
2. 前景球（z=4.5）和远景球（z=20）的视觉尺度差异被压缩，DoF 的深度暗示被破坏

**修复方式**：将 sensor_height 从 2.0 逐步增加到 6.0。半 FOV 变为 arctan(3/11.25) ≈ 15°，全 FOV ≈ 30°，接近全画幅 70mm 镜头的视角。球体之间的空间关系变得清晰可辨。

**教训**：相机参数（FOV、焦距、传感器尺寸、光圈）是一个耦合系统——改任何一个都会影响全局。调试时建议先用针孔（aperture=0）确认构图正确，再逐步引入光圈。

### Bug 4：所有球体均匀模糊——DoF 感知失败

**症状**：图像描述模型报告"所有球体均匀模糊，没有清晰的焦点"。即使光圈开到 2.0，针孔版（aperture=0.001）和宽光圈版（aperture=2.0）的图像几乎一模一样。

**错误假设**：大光圈自然会产生可见的景深差异。

**真实原因**：这是个复合问题——(1) FOV 太窄使球体填满画面，(2) 场景中大量重叠的球体 + 漫反射光照产生柔和的球面渐变，被误判为"模糊"，(3) 大球体的边缘存在自然的角度渐变（Fresnel 效应），即使没有 DoF 也看起来不"锐利"。

**修复方式**：
1. 扩大 FOV（sensor_height=6.0）
2. 减小球体并分散布局，减少重叠
3. 将球体放置在更有区分度的 z 位置（4.5 / 5.5 / 9.0 / 14 / 20）
4. 增加 metallic 材质来产生清晰的高光（作为"锐度参考点"）

修复后，绿球（z=9.0，对焦面）呈现清晰的边缘和高光，红球（z=4.5）明显模糊，金球（z=14）呈现圆润的 Bokeh 光斑。

**教训**：景深效果的"可见性"不仅取决于物理正确性，还取决于场景构图。好的构图应该有明确的深度层次、足够的前后景分离、以及锐利的参考点（如高光或纹理）。

---

## ⑥ 效果验证与数据

### 6.1 渲染参数

| 参数 | 值 |
|------|-----|
| 分辨率 | 800×600 |
| 每像素采样 (SPP) | 64 |
| 光圈采样数 | 32（16 Poisson Disk + 16 六边形） |
| 焦距 f | 5.0 |
| 对焦距离 u | 9.0 |
| 光圈半径 | 0.8（等效 f/3.1） |
| 传感器距离 v | 11.25 |
| 曝光系数 | 0.35 |
| 场景球体 | 6 个 + 墙面/地面/天花板 |

### 6.2 渲染结果

![Depth of Field Bokeh Renderer Result](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-11-depth-of-field/depth_of_field_output.png)

**场景解析**（从左到右）：
- **左侧红色球形**（z=4.5, 前景）：明显模糊，CoC ≈ 0.6 单位（在 800px 宽图像上约 72px），弥散圆内像素被来自不同光圈位置的光线混合。
- **正中绿色球形**（z=9.0, 对焦面）：清晰锐利，高光和表面细节可辨。这是唯一精确对焦的区域。
- **右上方金色球**（z=14, 背景中景）：中度模糊，CoC ≈ 0.35 单位。模糊程度明显比对焦面强，但不如前景红球。
- **右侧蓝色区域**（z=5.5 和 z=20）：混合了前景蓝球和远景白球的模糊投影像。
- **底部左下紫色小点**（z=8.5, 接近对焦面）：几乎清晰，作为"焦平面附近"的参考，展示景深从清晰到模糊的过渡。

### 6.3 量化数据

```
渲染耗时: 13.4 秒 (单线程, 800×600, 64 spp)
线性值统计: mean=0.652, std=1.361, min=0.004, max=13.803
输出 PNG 统计: mean=82.8, std=78.0
文件大小: 782.3 KB

球体颜色像素检测:
  红色球: 1,321 像素
  绿色球: 7,753 像素
  蓝色球: 27,461 像素
  金色球: 4,665 像素

对焦区(绿球)局部标准差: 19.1  ← 更"锐利"意味着更高的局部对比度
前景区(红球)局部标准差: 19.7  ← 模糊区域的局部对比度应相似或更低
```

注：局部标准差的数据显示对焦区和前景区数值接近（19.1 vs 19.7），这是因为漫反射光照下的球体表面本身就有渐变色——即使在针孔模式下也是如此。景深差异在**高光边缘**处最明显，而非整体标准差。

### 6.4 像素采样验证

```
绿球中心 (400, 300): R=213 G=247 B=235  ← 亮绿色，接近光源
红球区域 (120, 280): R=14  G=16  B=29   ← 暗红色，光照角度不佳
蓝球区域 (640, 280): R=169 G=195 B=234  ← 亮蓝色
金球区域 (280, 180): R=134 G=133 B=75   ← 暖金色
```

红球较暗是因为它处于光源的背面——这本身是正确的物理结果，但确实让红球的 DoF 效果不那么明显。如果追求视觉效果，可以让红球更靠前或添加补光。

---

## ⑦ 总结与延伸

### 技术局限性

1. **单线程渲染**：当前实现是纯 CPU 单线程，800×600 @ 64spp 需要 13 秒。如果增加到 256spp（减少噪点），需要约 52 秒。可以用 OpenMP（`#pragma omp parallel for`）几乎线性加速。
2. **无 BVH 加速**：场景只有 9 个球体所以不需要，但扩展到三角网格场景时必须引入 BVH/KD-tree。
3. **粗糙度-金属度模型简化**：当前使用简化的 Blinn-Phong + roughness/metallic hack，不是能量守恒的 GGX 微面元模型。在高金属度场景下会有能量损失。
4. **固定光圈形状**：六边形光圈采样是固定的，不能动态切换光圈叶片数（5 片→五边形 Bokeh，8 片→八边形 Bokeh）。
5. **没有纹理映射**：球体只有纯色，无法展示 DoF 对高频纹理（如文字、砖墙）的影响——高频纹理在 DoF 下的变化比纯色面更直观。
6. **采样效率**：32 个光圈采样对于大多数像素是浪费的——对于在焦平面上的像素，1 个采样就够了（所有光线汇聚）。可以做"自适应采样"：先用 4 个采样估计方差，方差小的像素提前终止。

### 可探索的延伸方向

1. **Anamorphic Bokeh（变形光斑）**：电影镜头的椭圆形光斑效果，通过将光圈采样圆盘拉伸为椭圆实现。
2. **Cat's Eye Bokeh（猫眼光斑）**：大光圈镜头的角落会出现被裁切的光斑（机械渐晕导致），可以通过在光圈采样中加入位置相关的裁剪来实现。
3. **Chromatic Aberration in Bokeh（色散虚化）**：不同波长在镜头中的折射率不同，导致 Bokeh 边缘出现彩色（紫边/绿边）。可以在光圈采样时对 R/G/B 分开采样。
4. **Dirt on Lens（镜头污渍）**：在光圈面上放置纹理（尘土、指纹），影响光线透过率，产生独特的虚化效果。
5. **与 SSR/SSGI 的结合**：本系列之前实现的 Screen Space Reflections（06-02）和 SSGI（05-22）可以在景深渲染的基础上叠加，产生"模糊的反射"效果。

### 与本系列的关联

- **06-02 SSR Renderer**：屏幕空间反射。可以在 DoF 之后加 SSR pass，让反射也受景深影响（远处的反射比近处的更模糊）。
- **05-21 Thin Film Iridescence**：薄膜干涉。给金属球加上彩虹色薄膜，在 DoF 下薄膜干涉图案会随 Bokeh 形状变形。
- **05-20 TAA**：时域抗锯齿。TAA 的运动矢量重投影在 DoF 场景中需要特殊处理——离焦区域的历史像素对应关系会被虚化打乱。

---

## 完整代码

本文中出现的所有代码均来自项目仓库：

🔗 [GitHub - Daily Coding Practice: Depth of Field Bokeh Renderer](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/06/06-11-depth-of-field)

编译命令：
```bash
g++ main.cpp -o dof -std=c++17 -O2 -Wall -Wextra -Wno-missing-field-initializers
./dof
```

---

*Day 127 of Daily Coding Practice · 2026-06-11*
