---
title: "每日编程实践: SDF Ray Marching Renderer"
date: 2026-04-01 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - Ray Marching
  - SDF
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-01-sdf-ray-marching/sdf_output.png
---

## 背景与动机

在渲染领域，我们通常用三角形网格来表示几何体。一个球体需要几百个三角面才能看起来圆润，一个复杂的有机形状（角色的脸部、液体、云朵）则需要数万甚至数百万个三角形，且对它们做 CSG 布尔运算（切割、合并、交集）在实时场景中代价极高。

**有向距离场（Signed Distance Field, SDF）** 提供了一种完全不同的几何表示方式：不存储顶点，而是用一个数学函数 `f(p) → d` 来描述空间中任意点 `p` 到曲面的最近距离 `d`。当 `d < 0` 时点在物体内部，`d > 0` 时在外部，`d = 0` 恰好在表面。这个简洁的定义带来了一系列天然的好处：

- **精确的 CSG 操作**：两个 SDF 做并集只需 `min(d1, d2)`，交集用 `max`，差集用 `max(-d1, d2)`，数学上完全精确，没有多边形法的数值误差
- **平滑过渡**：引入平滑最小值函数可以让两个物体无缝熔合，一行代码就能做到有机体般的流动感
- **无限分辨率**：SDF 是解析函数，不管你离多近，表面永远是精确的数学曲线，不会出现网格锯齿
- **简化渲染管线**：不需要顶点缓冲、光栅化、深度测试——Ray Marching 算法直接从 SDF 读取距离信息，一个主循环就完成了整个渲染过程

**工业界实际应用**：Unity 的 Shader Graph 内置 SDF 节点用于 UI 抗锯齿文字渲染（GPU Gems 3 中 Valve 首先提出 SDF 文字）；Inigo Quilez 在 Shadertoy 上用纯 SDF 渲染出了令人叹为观止的程序化场景；游戏《Claybook》以 SDF 为核心数据结构实现可破坏的粘土世界；NVIDIA Omniverse 中的 SDF 碰撞检测替代传统凸包算法，精度提升显著。

传统 Ray Tracing 对每条光线要遍历 BVH 找三角形；而 Ray Marching 利用 SDF 的距离保证，每次步进至少等于当前到最近几何体的距离，绝对不会越过任何表面——这就是 **Sphere Tracing**，约翰·哈特（John Hart）在 1996 年论文《Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces》中首次系统提出。

今天的项目目标：在纯 CPU 软渲染器中实现 SDF Ray Marching，包含多种几何原语、CSG 操作、法线估计、Blinn-Phong 光照、软阴影、AO 遮蔽，输出一张 800×450 的渲染图。

---

## 核心原理

### 有向距离场的数学定义

对于曲面 S，空间点 p 的有向距离函数定义为：

```
SDF(p) = sign(p, S) × min_{q ∈ S} ‖p - q‖
```

其中 `sign(p, S)` 在 p 位于曲面外部时为 +1，内部为 -1。每种几何体都有其解析 SDF 公式，以下是本项目实现的几种：

**球体**（半径 r，中心原点）：
```
SDF_sphere(p, r) = |p| - r
```
直觉：点到球心的距离减去半径。距离为正 = 在球外，为负 = 在球内。

**轴对齐盒子**（半尺寸 b = (bx, by, bz)）：
```
q = (|px|-bx, |py|-by, |pz|-bz)
SDF_box(p, b) = |max(q, 0)| + min(max(qx, qy, qz), 0)
```
这个公式分两部分理解：
- `max(q, 0)` 是点在盒子外部时各轴方向的超出量（Euclidean 距离到最近面）
- `min(max(qx,qy,qz), 0)` 处理点在盒子内部时的情况（取最近面的负距离）

**圆环（Torus）**（大半径 R，小半径 r）：
```
q = (√(px²+pz²) - R, py)
SDF_torus(p, R, r) = |q| - r
```
直觉：先把点投影到圆环的中心圆（在 XZ 平面），然后计算到截面圆心的距离再减半径。

**胶囊体（Capsule）**（两端点 a、b，半径 r）：
```
t = clamp(dot(p-a, b-a) / dot(b-a, b-a), 0, 1)
SDF_capsule(p, a, b, r) = |p - a - t*(b-a)| - r
```
直觉：先找线段 ab 上距离 p 最近的点（通过投影 + clamp），再测量到该点的距离减半径。

### Sphere Tracing 算法

Ray Marching 的核心思想极简：

```
t = tMin
repeat:
    p = ray_origin + t * ray_dir
    d = SDF_scene(p)      // 到最近几何体的距离
    if d < epsilon: 命中
    t += d               // 安全地步进 d 距离
    if t > tMax: 未命中
```

**为什么 `t += d` 是安全的**：SDF 保证当前位置到任何几何表面的最近距离 ≥ d，因此沿射线方向步进 d 距离，绝对不会穿越任何表面。这是 Sphere Tracing 的核心保证，不同于固定步长的 Ray Marching（固定步长可能漏掉薄物体或浪费步数在空旷区域）。

每次步进都在当前点画一个"安全球"（半径 = SDF 值），光线最多只步进到这个球的边界。随着接近表面，SDF 值越来越小，步伐自动收窄，精确逼近交点。

收敛条件是 `d < 0.0005f`（0.5mm 精度），最多迭代 256 步，实践中绝大多数光线在 30-50 步内收敛。

### SDF 的 CSG 操作

**普通布尔操作**（精确）：
```
并集 union:           min(d1, d2)       → 取两者更靠近的表面
差集 subtraction:     max(-d1, d2)      → 从 d2 中挖去 d1 的内部
交集 intersection:    max(d1, d2)       → 只保留两者都覆盖的区域
```

**平滑并集**（inigo quilez 的经典公式）：
```
smoothUnion(d1, d2, k):
    h = clamp(0.5 + 0.5*(d2-d1)/k, 0, 1)
    return mix(d1, d2, h) - k*h*(1-h)
```
参数 `k` 控制融合半径。最后一项 `-k*h*(1-h)` 是"下凹补偿"，确保融合区域向内收缩，产生有机熔合的视觉效果。当 `k=0` 时退化为普通 `min`，`k` 越大融合区域越宽。

### 法线估计：中心差分法

有了 SDF，法线的计算无需存储顶点法线缓冲。利用梯度计算：

```
N ≈ normalize(∇SDF(p))
```

用中心差分近似梯度：
```
N.x = SDF(p + (ε,0,0)) - SDF(p - (ε,0,0))
N.y = SDF(p + (0,ε,0)) - SDF(p - (0,ε,0))
N.z = SDF(p + (0,0,ε)) - SDF(p - (0,0,ε))
N = normalize(N)
```

每次调用需要 6 次额外的 SDF 求值（本项目 ε = 0.001）。这是 SDF 渲染中计算量较大的步骤，优化方向是四面体差分法（tetrahedron normal，只需 4 次 SDF 调用）。

### 软阴影算法

在光线追踪中，硬阴影通过一次 shadow ray 判断"遮挡/不遮挡"。软阴影则需要区域光采样，代价高昂。SDF 提供了一种优雅的近似方案：在 shadow ray 步进过程中，记录所有步骤中"最小安全余量 / 当前距离"的比值：

```
softShadow(p, lightDir, tMin, tMax, k):
    result = 1.0
    t = tMin
    while t < tMax:
        d = SDF(p + t * lightDir)
        if d < 0.001: return 0  // 完全遮挡
        result = min(result, k * d / t)  // 近似半影
        t += d
    return saturate(result)
```

直觉：`d/t` 表示当前步骤中光线"险些擦过"障碍物的程度——t 大时分母大，意味着遮挡物离接收点近；`d` 小时分子小，意味着光线紧贴障碍物飞过。系数 `k` 控制软阴影的锐利度，`k=16` 产生比较锐利的软阴影，`k=4` 产生非常柔和的阴影。这个方法不需要任何额外采样，几乎免费得到软阴影。

### 环境光遮蔽（AO）

真实世界中，凹角、缝隙处的环境光被周围几何体遮挡，因此显得更暗。SDF AO 同样有解析近似：

```
calcAO(pos, nor):
    occ = 0, sca = 1
    for i in [0..4]:
        h = 0.01 + 0.12 * i / 4   // 沿法线方向采样距离
        d = SDF(pos + nor * h)
        occ += (h - d) * sca      // 若 h > d，说明该点被遮挡
        sca *= 0.95               // 远处遮挡权重指数衰减
    return saturate(1 - 3 * occ)
```

直觉：沿法线方向每隔一定距离采样 SDF，如果采样到的 SDF 值比采样距离小，说明前方有障碍物，产生遮蔽。权重随距离指数衰减，近处遮挡影响更大。5 次采样就能得到视觉上令人满意的 AO 效果。

---

## 实现架构

整个渲染器是单文件 C++17，约 500 行，无外部依赖（除 stb_image_write.h）。

### 数据流

```
相机参数（位置/朝向/FOV）
         ↓
对每个像素 (i, j)：
    getRay() → 生成世界空间射线 (ro, rd)
         ↓
    rayMarch(ro, rd) → 步进直到命中/超出范围
         ↓
    hit? YES:
        calcNormal(pos) → 中心差分法线
        getMaterial(matID, pos) → 材质参数
        shade(pos, nor, rd, matID):
            for each light:
                softShadow() → 软阴影系数
                diffuse = albedo * ndl * shadow
                specular = Blinn-Phong
            AO = calcAO(pos, nor)
            ambient = albedo * ambient_color * AO
         ↓
    hit? NO: skyColor(rd) → 天空渐变+太阳
         ↓
    acesFilmic(color) → HDR色调映射
    gammaCorrect(color) → sRGB gamma 2.2
         ↓
写入 PNG 像素缓冲
```

### 场景 SDF 设计

场景由 7 个几何体组成：

| 几何体 | SDF 类型 | 材质 | 特色 |
|--------|---------|------|------|
| 大球 | sdSphere(r=0.9) | 红色金属 | 左侧主球 |
| 小球 | sdSphere(r=0.6) | 蓝色光泽 | 中心球 |
| 盒子 | sdBox(0.6,0.7,0.6) | 金色 | 右侧 |
| 圆环 | sdTorus(R=0.7,r=0.25) | 绿色金属 | 背景，绕X轴旋转60° |
| 胶囊体 | sdCapsule | 紫色 | 前景左侧斜放 |
| Smooth Union | min(sphA, sphB, k=0.4) | 橙色 | 两球融合演示 |
| 地面 | sdPlane | 棋盘格 | 无限平面 |

**关键设计决策**：
- 场景 SDF 函数 `sceneMap(p)` 返回 `SDFResult{dist, matID}`，一次调用同时得到距离和材质 ID，避免二次查找
- 每种几何体用 `if (d < res.dist)` 实现场景并集，相当于多个物体的 `min`
- 圆环通过局部空间变换（绕 X 轴旋转）实现倾斜，不需要修改 sdTorus 公式
- Smooth Union 两个球紧邻（中心距 0.7，融合半径 0.4），可见明显熔合效果

### 材质系统

```cpp
struct Material {
    Vec3  albedo;     // 漫反射颜色
    float specular;   // 高光强度
    float shininess;  // Blinn-Phong 指数
    float roughness;  // 预留（未来 PBR 扩展）
};
```

地面材质额外实现棋盘格：根据 `floor(pos.x) + floor(pos.z)` 的奇偶性切换亮/暗颜色，利用世界空间坐标直接生成程序化纹理。

---

## 关键代码解析

### Vec3 数学库

```cpp
struct Vec3 {
    float x, y, z;
    // 运算符重载：+/-/*(标量)/*(向量)//(标量)
    float dot(const Vec3& o) const { return x*o.x + y*o.y + z*o.z; }
    Vec3 cross(const Vec3& o) const {
        return {y*o.z - z*o.y, z*o.x - x*o.z, x*o.y - y*o.x};
    }
    Vec3 normalize() const {
        float l = length();
        return l > 1e-8f ? (*this) / l : Vec3(0,1,0);  // 避免零向量除零
    }
};
```

注意 `normalize()` 的零向量保护：当向量长度极小时返回 `(0,1,0)` 而不是 NaN，这在 SDF 等值面附近做法线估计时很重要——SDF 梯度在极端角落可能接近零向量。

### SDF 盒子：分区域讨论

```cpp
float sdBox(const Vec3& p, const Vec3& b) {
    Vec3 q = {std::abs(p.x) - b.x,
              std::abs(p.y) - b.y,
              std::abs(p.z) - b.z};
    Vec3 qMax = {std::max(q.x, 0.0f), std::max(q.y, 0.0f), std::max(q.z, 0.0f)};
    return qMax.length() + std::min(q.maxComp(), 0.0f);
}
```

这个公式覆盖三种区域：
1. **盒子角部外**（qx,qy,qz 都 > 0）：`qMax.length()` = Euclidean 距离到最近角点
2. **盒子边/面外**（某些 q 分量 > 0，某些 ≤ 0）：`qMax.length()` = 距离到最近边/面
3. **盒子内部**（所有 q < 0）：`qMax.length() = 0`，距离 = `min(max_q, 0)` = 最近面的负距离

### 平滑并集实现

```cpp
float opSmoothUnion(float d1, float d2, float k) {
    float h = clamp(0.5f + 0.5f*(d2-d1)/k, 0.0f, 1.0f);
    return d1*(1.0f-h) + d2*h - k*h*(1.0f-h);
}
```

`h` 是混合权重：当 `d2 >> d1` 时 h≈1（选择 d1），当 `d1 >> d2` 时 h≈0（选择 d2），在过渡区域平滑插值。最后的 `-k*h*(1-h)` 在 h=0.5 时达到最大值 -k/4，将融合区域向内"推"，产生熔合下陷的视觉效果。

### Ray Marching 主循环

```cpp
MarchResult rayMarch(const Vec3& ro, const Vec3& rd,
                     float tMin = 0.01f, float tMax = 100.0f) {
    float t = tMin;
    for (int i = 0; i < 256; i++) {
        Vec3 p = ro + rd * t;
        SDFResult r = sceneMap(p);
        if (r.dist < 0.0005f) {       // 命中阈值
            return {t, r.matID, true};
        }
        t += r.dist;                   // Sphere Tracing 核心步进
        if (t > tMax) break;
    }
    return {tMax, MAT_NONE, false};
}
```

`tMin = 0.01f` 是"防自交偏移"：从表面发射 shadow ray 时，如果从 t=0 开始步进，第一步就会命中自身（SDF ≈ 0）。这是 SDF 渲染中最常见的 Bug 来源之一（见踩坑章节）。

命中阈值 `0.0005f`（半毫米）权衡了精度和迭代次数：太大 → 法线估计在凹处可能采样到错误几何；太小 → 对某些场景步数激增，接近数值精度极限。

### 软阴影实现细节

```cpp
float softShadow(const Vec3& ro, const Vec3& rd,
                 float tMin, float tMax, float k) {
    float res = 1.0f;
    float t = tMin;
    for (int i = 0; i < 64; i++) {
        float d = sceneMap(ro + rd * t).dist;
        if (d < 0.001f) return 0.0f;    // 完全遮挡
        res = std::min(res, k * d / t); // 记录"险些擦过"程度
        t += d;
        if (t > tMax) break;
    }
    return saturate(res);
}
```

调用时：`softShadow(pos + nor * 0.002f, lightDir, 0.02f, 20.0f, 16.0f)`

- `pos + nor * 0.002f`：沿法线偏移出发点，避免表面自遮挡
- `tMin = 0.02f`：额外安全距离，对弯曲表面更稳健
- `k = 16.0f`：较大的 k 使阴影较锐利；降低到 4~8 产生更柔和的漫射感阴影

### 天空和太阳渲染

```cpp
Vec3 skyColor(const Vec3& rd) {
    float t = 0.5f * (rd.y + 1.0f);              // 竖直方向线性映射到 [0,1]
    Vec3 sky = mix(skyHorizon, skyTop, t);        // 地平线到天顶渐变

    Vec3 sunDir = Vec3(0.6f, 0.5f, 0.5f).normalize();
    float sunDot = rd.dot(sunDir);
    if (sunDot > 0.998f)                           // 太阳核心（极细）
        sky += Vec3(2.0f, 1.8f, 1.2f);
    else if (sunDot > 0.99f)                       // 太阳晕圈（平方衰减）
        sky += Vec3(1.0f, 0.9f, 0.6f) * pow(...);

    return sky;
}
```

太阳方向与 key light 方向一致，保证光照和天空视觉上统一。`sunDot > 0.998f` 对应约 3.6° 锥角，对于 800px 宽的图像太阳大约占 50 像素，视觉上合理。

### ACES Filmic 色调映射

```cpp
Vec3 acesFilmic(const Vec3& x) {
    const float a=2.51f, b=0.03f, c=2.43f, d=0.59f, e=0.14f;
    Vec3 num = x * (x * a + Vec3(b));
    Vec3 den = x * (x * c + Vec3(d)) + Vec3(e);
    return {saturate(num.x/den.x), saturate(num.y/den.y), saturate(num.z/den.z)};
}
```

这是 Narkowicz 2015 年对 ACES 曲线的近似拟合，一个有理函数，计算代价很低。相比 Reinhard（`x/(1+x)`），ACES 在高亮区有更好的色相保持（不会因为过度压缩而让颜色偏黄/偏白），适合有彩色高光的场景。

---

## 踩坑实录

### Bug 1：圆环旋转后消失

**症状**：场景里明明写了圆环，渲染结果里就是看不到任何环形物体，只有平面和球体。

**错误假设**：认为 `sdTorus` 函数写错了，反复检查大半径 R 和小半径 r 参数。

**真实原因**：旋转变换矩阵写反了。SDF 中对几何体做变换，需要对**点 p** 做**逆变换**（将世界空间坐标变换到物体局部空间），而不是对几何体做正变换。我写的是绕 X 轴顺时针旋转 60°，但数学上用的是 `(cos, -sin; sin, cos)`（逆时针），结果圆环被旋转到了视锥体外面。

**修复**：检查旋转方向，确认 Y 轴分量的 `sinA` 和 `-sinA` 符号正确。修复后圆环出现在预期位置。

**教训**：SDF 中的变换是在点 p 上做**逆变换**，直觉上是"把世界坐标系变回到物体坐标系"。一个好的调试方法是先测试不旋转（identity transform）确认几何体在正确位置，再逐步添加旋转。

### Bug 2：unused-label 编译警告

**症状**：编译时报 `-Wunused-label` 警告：`label 'Vec2d' defined but not used`。

**原因**：在写 `sdTorus` 时，本想用 `Vec2d q = ...` 但实际上用手动计算替代了，原先写的 `Vec2d:;` 残留成了未使用的标签（C++ 中 `name:` 是标签语法，不是类型声明）。

**修复**：直接删除那行。

**教训**：`g++ -Wall -Wextra` 会捕获各种意外语法。这类错误通常是复制粘贴或重构残留，肉眼很难发现，编译器警告是第一道防线。

### Bug 3：Shadow Ray 自遮挡（黑色条纹）

**症状**：第一版代码中阴影区域出现不规则的黑色条纹，特别是球体顶部高光区域旁边。

**错误假设**：以为是软阴影的 k 值设置问题，调整了半天 k 参数。

**真实原因**：shadow ray 的起点就在几何表面，`sceneMap(pos)` ≈ 0，softShadow 的第一步就命中了自身，返回 0（完全阴影）。

**修复**：在 `shade()` 里调用 `softShadow` 时，起点沿法线偏移：`pos + nor * 0.002f`，同时 `tMin = 0.02f`。两层保护：
- `nor * 0.002f`：把起点抬离表面 2mm（法线方向）
- `tMin = 0.02f`：跳过 shadow ray 起始的 2cm，避免在弯曲面上仍然误判自交

**教训**：SDF 渲染中所有从表面发出的二次射线（shadow ray、AO 采样）都必须有出发偏移。偏移量太小会产生条纹；太大会漏掉距离非常近的遮挡物。通常法线偏移 0.001~0.005，tMin 0.01~0.05 是合理范围，根据场景尺度调整。

### Bug 4：AO 在地面棋盘格处出现噪点

**症状**：地面棋盘格的颜色过渡处出现小的深色噪点，破坏棋盘格的整洁感。

**原因**：棋盘格是通过 `floor(pos.x) + floor(pos.z)` 的奇偶性切换颜色实现的，本身没有几何变化。但 AO 采样时，沿法线（向上）采样若干距离后调用 `sceneMap`，该函数会查询地面 SDF（`sdPlane`），无论何处都正确。但材质判断在 AO 之后，AO 只依赖几何 SDF 数值，与棋盘格材质无关，所以实际上 AO 本身是正确的。

重新审视后发现噪点来自**法线偏移量过小**，导致法线估计在棋盘格边界（坐标整数值处）恰好采样到浮点边界，差分结果不稳定。

**修复**：将法线估计的 `eps` 从 `0.0005` 提高到 `0.001`，避免在数值边界采样。

### Bug 5：渲染速度预期偏差

**症状**：第一版 800×450 估计 30 秒完成，实际运行了 90 秒才输出。

**分析**：`sceneMap` 包含 7 个几何体 SDF，Ray Marching 最多 256 步，法线估计 6 次 SDF，每个像素还有 3 盏灯 × (1 次 softShadow × 64 步 + 5 次 AO 采样)。粗略估算每像素 ~500 次 `sceneMap` 调用，800×450 = 36 万像素，共约 1.8 亿次 `sceneMap`，每次 7 个 SDF 求值。

**优化方向**（本次未实施，留待后续）：
- 多线程（OpenMP `#pragma omp parallel for`）
- 降低 shadow ray 最大步数（现在 64，可改 32）
- BVH/场景分层，快速剔除不相关 SDF

---

## 效果验证与数据

### 像素统计（Python PIL 验证）

```
尺寸: 800×450
像素均值: 203.0  标准差: 25.6
顶部均值(天空): 218.7
底部均值(地面): 201.3
```

- 均值 203 落在 [10, 240] 范围内 ✅
- 标准差 25.6 > 5，图像有足够的明暗变化 ✅
- 顶部（天空）均值 > 底部（地面），坐标系正确（天空在上） ✅
- 文件大小 102KB > 10KB ✅

### 渲染参数

| 参数 | 值 |
|------|---|
| 分辨率 | 800×450 |
| Ray Marching 最大步数 | 256 |
| 命中阈值 | 0.0005 |
| Shadow Ray 最大步数 | 64 |
| AO 采样次数 | 5 |
| 总渲染时间 | ~90 秒（单线程 CPU） |

### 渲染结果

![SDF Ray Marching 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-01-sdf-ray-marching/sdf_output.png)

场景包含：
- 红色金属大球（左）和蓝色光泽小球（中）
- 金色盒子（右）
- 绿色圆环（背景，倾斜60°）
- 紫色胶囊体（前景左侧）
- 橙色平滑并集双球（右前，可见明显熔合效果）
- 棋盘格地面（软阴影清晰可见）
- 天空渐变 + 太阳（与 key light 方向一致）

### 技术效果对比

| 技术要素 | 效果描述 |
|---------|---------|
| 软阴影 (k=16) | 阴影边缘有柔和过渡，约 10-15px 半影区域 |
| AO | 球体底部与地面接触处可见明显暗化 |
| 平滑并集 | 两个橙球之间有流畅的有机熔合过渡 |
| 棋盘格 | 程序化纹理，无走样，近处清晰远处缩小 |
| 雾效 | 背景圆环比前景物体略显朦胧 |

---

## 总结与延伸

### 技术局限性

1. **性能**：纯 CPU 单线程，800×450 需要 90 秒。不适合实时渲染，但可以通过 GPU Shader（GLSL/HLSL）移植实现实时 60fps
2. **动态场景**：每帧重新计算所有 SDF，对于复杂场景变形（如角色动画）需要 SDF 场变形或骨骼绑定的 SDF，实现复杂度大幅提升
3. **无纹理贴图**：目前只有程序化材质（棋盘格）。SDF 配合 UV 展开需要额外处理，通常用 Triplanar Mapping 作为替代
4. **反射/折射**：当前实现没有光线递归（反射球、折射玻璃）。Ray Marching 支持递归，但每次递归把渲染成本乘以 256 步

### 可继续优化的方向

- **多线程**：OpenMP 可以零改动地并行化像素循环，预计 8 核 CPU 提速 6-7×
- **GPU 实现**：将 `sceneMap` 移植为 GLSL fragment shader，实现实时 Ray Marching
- **SDF 变形**：实现 twist、bend、displacement 等空间变形操作，产生更复杂的有机形状
- **材质扩展**：加入 GGX BRDF（替换 Blinn-Phong）实现物理正确的金属/非金属材质
- **次级光线**：添加反射（镜面球）和折射（玻璃球），展示 SDF 在复杂光传输中的优势
- **SDF 字体渲染**：将 SDF 技术用于文字渲染（Valve 的 MSDF），与本系列的实时渲染主题结合

### 与本系列其他文章的关联

本文的 Ray Marching + 软阴影技术与以下文章有直接关联：
- [03-10 PCSS 软阴影](./daily-coding-pcss-soft-shadow-2026-03-10)：PCSS 用 Shadow Map 近似软阴影，本文用 SDF 直接计算；两者效果相似但实现路径完全不同
- [03-24 次表面散射](./daily-coding-subsurface-scattering-2026-03-24)：SSS 的偶极子模型依赖精确的法线，SDF 的梯度法线可以直接服务于 SSS 计算
- [03-25 SSAO](./daily-coding-ssao-2026-03-25)：本文的 SDF AO 是 SSAO 的解析精确版本，不需要深度缓冲采样，零噪点

SDF 是程序化图形学的核心工具之一。从简单的球体到 Inigo Quilez 用 400 行 GLSL 写出的写实人脸，SDF + Ray Marching 的表达力令人叹服。对于对图形学感兴趣的开发者，Shadertoy（shadertoy.com）是学习和实验 SDF 技术的绝佳平台——所有代码都在浏览器里实时运行，即改即见。
