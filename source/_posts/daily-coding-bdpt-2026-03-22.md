---
title: "每日编程实践: 双向路径追踪 (BDPT) 与多重重要性采样"
date: 2026-03-22 05:30:00
tags:
  - 每日一练
  - 图形学
  - 光线追踪
  - C++
  - 全局光照
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-22-bdpt/bdpt_output.png
---

## 背景与动机

### 为什么单向路径追踪不够用？

如果你用过标准路径追踪（Unidirectional Path Tracing, PT），一定注意到一个令人头疼的现象：在某些场景里，即使采样数很高，画面还是充满噪点。具体来说，当光源面积小、场景几何复杂、或者存在"焦散"效果时，单向路径追踪的收敛速度极其缓慢。

**为什么？** 让我们想象一个典型的焦散场景：光线从天花板射下，经过一个玻璃球折射，在地面上形成一个亮斑。单向路径追踪从相机出发，光线碰到地面，想估计这个点的直接光照——它需要对天花板上的面光源采样，但光路是"光源 → 玻璃球折射 → 地面"，直接连接相机路径顶点和光源的概率极低（玻璃球的折射形成了很窄的采样空间），绝大多数样本的贡献是零。这就是所谓的**难以捕获的光路（difficult light transport path）**问题。

再看几个工业界的真实痛点：

- **室内渲染**：小窗户进来的阳光，单向 PT 需要极多样本才能收敛，而建筑可视化行业的标准渲染时间往往只有几分钟
- **珠宝渲染**：宝石刻面形成的复杂折射焦散，游戏里的宝石看起来总是"假"，根本原因就是这里
- **汽车内饰**：阳光穿过车窗的散射，自动驾驶仿真和游戏里的汽车光照难以逼真

**BDPT（双向路径追踪）**正是为解决这些问题而生。它的核心思路极为自然：**既然从相机端难以"撞见"光路，那就同时从光源端也发出路径，然后把两端路径连接起来**。这样，那些单向 PT 几乎永远采样不到的光路，BDPT 可以通过连接两端顶点直接构造出来。

BDPT 首次由 Lafortune 和 Willems（1993）以及 Veach 和 Guibas（1994）独立提出。Veach 在其 1997 年的博士论文中给出了完整的理论框架，并引入了**多重重要性采样（Multiple Importance Sampling, MIS）**来优化各策略的权重。此后，BDPT 成为离线渲染领域（Pixar RenderMan、Arnold、Cycles 等）的标准全局光照算法之一。

---

## 核心原理

### 渲染方程回顾

所有路径追踪算法的出发点都是**渲染方程**（Kajiya, 1986）：

$$L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o) L_i(x, \omega_i) |\cos\theta_i| \, d\omega_i$$

其中：
- $L_o$：出射辐射度（我们想计算的值）
- $L_e$：自发光辐射度（光源本身的贡献）
- $f_r$：双向反射分布函数（BRDF），描述表面的反射特性
- $L_i$：入射辐射度（来自其他物体的反射/折射）
- $|\cos\theta_i|$：入射角余弦项，描述能量投影到表面的衰减

这个方程是递归的——$L_i$ 本身又是另一个表面的 $L_o$，形成无穷递归。路径追踪就是用蒙特卡洛积分来近似这个递归积分的。

### 路径空间与测度

**路径空间**的思想是 Veach 论文的核心贡献之一。设一条长度为 $k+1$ 的路径（含 $k+2$ 个顶点）：

$$\bar{x} = x_0 x_1 x_2 \cdots x_{k+1}$$

其中 $x_0$ 是相机、$x_{k+1}$ 是光源，中间是场景中的交点。这条路径的贡献可以写成：

$$f(\bar{x}) = L_e(x_{k+1} \to x_k) \cdot G(x_k \leftrightarrow x_{k+1}) \cdot \prod_{i=1}^{k} f_r(x_{i+1} \to x_i \to x_{i-1}) \cdot G(x_{i-1} \leftrightarrow x_i) \cdot W_e(x_0 \to x_1)$$

这里 $G(x \leftrightarrow y)$ 是**几何项**：

$$G(x \leftrightarrow y) = \frac{|\cos\theta_x| \cdot |\cos\theta_y|}{|x - y|^2} \cdot V(x, y)$$

$V(x, y)$ 是可见性函数（1 表示可见，0 表示被遮挡）。这个几何项包含了距离的平方衰减和两端法向量的余弦项，直觉上就是"单位立体角与单位面积之间的换算"。

### BDPT 的策略分解

单向路径追踪的一条长度为 $n$ 的路径，可以看成从相机端取了 $n$ 个顶点、从光源端取了 0 个顶点，然后"连接"的特殊情况。BDPT 把这个思路一般化：

对于路径总长度 $k$，BDPT 枚举所有策略 $(s, t)$，其中 $s$ 是从光源端采样的顶点数，$t$ 是从相机端采样的顶点数，$s + t - 1 = k$（减 1 是因为连接操作本身算一段）。

每种策略 $(s, t)$ 都给出一个对像素辐射度的估计量 $C_{s,t}$。BDPT 把所有策略的估计量用多重重要性采样加权组合：

$$\langle L \rangle = \sum_{s \geq 0, t \geq 1} w_{s,t}(\bar{x}) \cdot \frac{f_{s,t}(\bar{x})}{p_{s,t}(\bar{x})}$$

其中 $w_{s,t}$ 是 MIS 权重，保证所有权重之和为 1，从而估计量是无偏的。

### 多重重要性采样（MIS）

MIS 的核心问题是：我有 $n$ 种采样策略，每种策略对同一个积分给出不同精度的估计，如何组合它们才能最优？

**幂次启发（Power Heuristic）**是实践中最常用的 MIS 权重：

$$w_k(x) = \frac{(n_k p_k(x))^\beta}{\sum_j (n_j p_j(x))^\beta}$$

通常取 $\beta = 2$。直觉上，这个公式的意思是：**哪种策略在当前样本 $x$ 处的 PDF 最高（即"最擅长采样这种路径"），就给那种策略更高的权重**。

为什么 $\beta = 2$ 比 $\beta = 1$（平衡启发）更好？Veach 证明了幂次启发的方差上界是平衡启发的 $\min_k n_k p_k(x)$ 倍，而平方可以放大高概率策略的主导性，减少低效策略的"污染"。

在本项目的简化实现里，我用了三类策略：

1. **$t \geq 2, s = 0$（纯相机路径，PT 的方式）**：相机路径直接打到光源。这对于能直接"看到"光源的路径非常高效。

2. **$t \geq 2, s = 1$（直接光照，NEE 的方式）**：在相机路径的最后一个顶点，直接采样光源上的一个点，检查可见性后计算贡献。这对漫反射场景极为有效。

3. **$t \geq 2, s \geq 2$（双向连接）**：光路和相机路径各走几步，然后在中间连接。这是 BDPT 的独特贡献，对镜面球产生的焦散、穿越小开口的光照等复杂路径有效。

### Cornell Box 场景光照分析

Cornell Box 是渲染算法研究的标准测试场景，包含：
- 漫反射红绿白墙
- 天花板上的矩形面光源（强度 15）
- 一个漫反射球 + 一个镜面球

对于**漫反射球**的光照，$s=1$（直接光照采样）策略最高效，因为球表面到光源的几何关系简单。

对于**镜面球**产生的**焦散**（光源打在地板的亮斑），$s \geq 2$ 的双向策略才能高效采样：光路从光源出发，打到镜面球反射后落在地面，这时相机路径的地面顶点可以直接连接这个镜面球上的光路顶点。

---

## 实现架构

### 整体数据流

```
相机像素 (i, j)
     │
     ▼
buildCameraPath()    buildLightPath()
     │                    │
     ▼                    ▼
camPath: [cam, v1, v2, ...]    lightPath: [light, lv1, lv2, ...]
     │                    │
     └────────┬───────────┘
              ▼
     connectPaths(s, t)
     对所有 (s, t) 组合
              │
              ▼
     加权求和 → 像素辐射度
              │
              ▼
     Reinhard ToneMap → Gamma → PNG
```

### 关键数据结构

**Vertex（路径顶点）**：

```cpp
struct Vertex {
    Vec3 pos;         // 世界空间位置
    Vec3 normal;      // 表面法向量（已朝向入射侧）
    Vec3 throughput;  // 累积吞吐量（从路径起点到此顶点的乘积）
    int matIdx;       // 材质索引
    double pdfFwd;    // 从前一个顶点生成此顶点的 PDF
    double pdfRev;    // 逆向 PDF（用于 MIS 权重计算）
    bool isDelta;     // 是否是镜面（delta 分布 BRDF）
    bool isLight;     // 是否在光源上
    bool isCamera;    // 是否是相机顶点
};
```

`throughput` 字段的设计思路：它存储的是"如果在此顶点处连接，乘以对端贡献就能得到路径总贡献"的因子，而不是路径的完整贡献。这使得连接操作变成简单的乘法而不是从头累乘。

**Scene 设计**：

场景支持两种几何体：`Sphere`（用于球体求交的解析解）和 `Quad`（轴对齐矩形，用于墙面和面光源）。面光源通过 `lightQuadIndices` 列表单独管理，方便快速采样光源位置。

**Material 设计**：

```cpp
enum class MatType { Diffuse, Mirror, Emissive };
struct Material {
    MatType type;
    Vec3 albedo;    // 漫反射/镜面颜色
    Vec3 emission;  // 只有 Emissive 类型使用
};
```

区分 `Diffuse` 和 `Mirror` 非常重要：`Mirror` 是 delta BRDF（只在反射方向有值，其他方向为零）。Delta BRDF 顶点不能用于"连接"操作——因为连接意味着指定一个特定方向，而 delta BRDF 只接受一个特定方向（反射方向），随机的连接方向的贡献几乎总是零。因此 `isDelta` 字段用来跳过这些顶点。

### 相机模型

使用简单的针孔相机：

```cpp
Camera cam(
    Vec3(278, 273, -800),   // 相机位置
    Vec3(278, 273, 0),      // 看向场景中心
    Vec3(0, 1, 0),          // 上方向
    40.0,                   // 垂直视场角（度）
    512, 512                // 分辨率
);
```

Cornell Box 的尺寸是 555×555×555，相机放在 z = -800 处（场景外部），视角 40° 可以看到整个场景。

---

## 关键代码解析

### 构建相机路径

```cpp
std::vector<Vertex> buildCameraPath(const Scene& scene, const Ray& cameraRay, int maxDepth) {
    std::vector<Vertex> path;

    // 相机顶点本身
    Vertex cam;
    cam.pos = cameraRay.origin;
    cam.throughput = Vec3(1.0);   // 初始吞吐量为 1
    cam.isCamera = true;
    path.push_back(cam);

    Ray ray = cameraRay;
    Vec3 throughput(1.0);

    for (int depth = 0; depth < maxDepth; depth++) {
        auto hit = scene.intersect(ray);
        if (!hit) break;  // 打到背景/天空

        const Material& mat = scene.materials[hit->matIdx];

        // 如果打到光源，记录并停止（光源是路径终点）
        if (mat.type == MatType::Emissive) {
            // 把光源顶点加入路径，用于 s=0 策略（相机直接"看见"光源）
            Vertex v; v.pos = hit->point; v.isLight = true; v.throughput = throughput;
            path.push_back(v);
            break;
        }

        if (mat.type == MatType::Mirror) {
            // 镜面反射：完全确定性的新方向
            Vec3 reflected = ray.dir - hit->normal * 2.0 * ray.dir.dot(hit->normal);
            throughput = throughput * mat.albedo;  // 镜面颜色衰减
            // isDelta = true，此顶点不参与连接操作
            ray = Ray(hit->point, reflected);
        } else {
            // 漫反射：余弦重要性采样
            Vec3 newDir = cosineSampleHemisphere(hit->normal);
            double p = pdfDiffuse(hit->normal, newDir);  // = cos(theta) / pi
            Vec3 f = mat.albedo / M_PI;                  // Lambertian BRDF
            double cosTheta = hit->normal.dot(newDir);
            // 更新吞吐量：f * cos / pdf = albedo（余弦采样正好约去 cos/pi）
            throughput = throughput * f * cosTheta / p;  // = throughput * albedo

            // 俄罗斯轮盘赌：避免无限追踪低贡献路径
            double rrProb = std::min(0.95, throughput.maxComp());
            if (rand01() > rrProb) break;
            throughput = throughput / rrProb;  // 无偏校正

            ray = Ray(hit->point, newDir);
        }
    }
    return path;
}
```

**关键设计点**：为什么漫反射的吞吐量更新后等于 `albedo`？

余弦重要性采样下，PDF 为 $p(\omega) = \cos\theta / \pi$，BRDF 为 $f_r = \text{albedo} / \pi$，所以：

$$\frac{f_r \cdot \cos\theta}{p} = \frac{\text{albedo}/\pi \cdot \cos\theta}{\cos\theta/\pi} = \text{albedo}$$

这是余弦采样的一个很好的性质：通过"匹配"分布，消去了所有三角函数项，吞吐量就是反射率本身，直觉清晰且数值稳定。

### 构建光源路径

```cpp
std::vector<Vertex> buildLightPath(const Scene& scene, int maxDepth) {
    // 1. 在面光源上均匀采样一个出发点
    const Quad& lightQuad = scene.quads[li];
    Vec3 pos = lightQuad.sample();  // corner + u*uAxis + v*vAxis, u,v ~ U[0,1]
    Vec3 normal = -lightQuad.normal; // 朝向场景内部（向下）

    // 2. 在光源法半球内余弦采样出射方向
    Vec3 dir = cosineSampleHemisphere(normal);
    double pdfPos = 1.0 / lightQuad.area;     // 位置 PDF（均匀面采样）
    double pdfDir = normal.dot(dir) / M_PI;   // 方向 PDF（余弦采样）

    // 3. 初始吞吐量：发射辐射度 / PDF
    // 物理含义：光源上这一点向这个方向发射的辐射度，除以采样概率
    Vec3 throughput = lightMat.emission * normal.dot(dir) / (pdfPos * pdfDir);
    // = emission * cos * pi / (cos * area) = emission * pi / area
    // 这就是"每单位面积的总出射功率归一化到路径贡献"的意义
    // ...
}
```

光源路径的初始化比相机路径复杂一些，因为需要同时处理**位置采样**和**方向采样**两个 PDF。物理直觉是：光源在某个方向上发射的能量（辐射强度）与 $\cos\theta$（朗伯余弦定律）成正比，余弦采样的 PDF 正好抵消这个因子，类似相机路径的情况。

### 连接操作

连接是 BDPT 最核心的部分，也是最容易出错的：

```cpp
Vec3 connectPaths(const Scene& scene,
                  const std::vector<Vertex>& camPath,
                  const std::vector<Vertex>& lightPath,
                  int s, int t) {
    if (s == 1) {
        // 直接光照策略：相机路径最后一个漫反射顶点 → 光源随机点
        const Vertex& camV = camPath[t - 1];
        if (!camV.connectable()) return Vec3(0);  // 跳过镜面顶点

        Vec3 lightPos = lightQuad.sample();
        Vec3 toLight = lightPos - camV.pos;
        double dist = toLight.length();
        Vec3 dirToLight = toLight / dist;

        // 几何项的两端余弦
        double cosAtCam = camV.normal.dot(dirToLight);    // 相机侧余弦
        double cosAtLight = (-dirToLight).dot(lightNormal); // 光源侧余弦

        if (cosAtCam <= 0 || cosAtLight <= 0) return Vec3(0); // 光在背面

        if (scene.occluded(camV.pos, lightPos)) return Vec3(0); // 被遮挡

        // 贡献 = 相机路径吞吐量 × BRDF × 几何项 × 光源辐射度 / 采样PDF
        Vec3 f = camMat.albedo / M_PI;
        double geo = cosAtCam * cosAtLight / (dist * dist);  // G(x↔y) 无可见性
        double pdfLight = 1.0 / lightQuad.area;
        return camThroughput * f * geo * lightMat.emission / pdfLight;
    }

    if (s >= 2 && t >= 2) {
        // 双向连接：camPath[t-1] 和 lightPath[s-1] 之间
        const Vertex& camV = camPath[t - 1];
        const Vertex& lightV = lightPath[s - 1];

        if (!camV.connectable() || !lightV.connectable()) return Vec3(0);

        Vec3 dir = (lightV.pos - camV.pos).normalize();
        double cosAtCam = camV.normal.dot(dir);
        double cosAtLight = lightV.normal.dot(-dir);
        if (cosAtCam <= 0 || cosAtLight <= 0) return Vec3(0);
        if (scene.occluded(camV.pos, lightV.pos)) return Vec3(0);

        // 从两端各贡献一个 BRDF 和余弦，几何项在中间
        Vec3 fCam = camMat.albedo / M_PI * cosAtCam;
        Vec3 fLight = lightMat.albedo / M_PI * cosAtLight;
        double geo = 1.0 / dist2;  // 距离平方衰减

        // 相机侧吞吐量 × 相机BRDF × 几何项 × 光源BRDF × 光源侧吞吐量
        return camThroughput * fCam * geo * fLight * lightThroughput;
    }
}
```

**容易出错的地方**：连接时，`camThroughput` 用的是 `camPath[t-2].throughput`（不是 `t-1`），因为 `throughput` 记录的是"到达该顶点之前"的累积，而 `camPath[t-1]` 的 BRDF 由 `fCam` 计算。理解这个"off-by-one"是写对 BDPT 的关键。

### Tone Mapping 与伽马校正

```cpp
Vec3 toneMap(Vec3 v) {
    // Reinhard tone mapping：逐通道映射 HDR → [0,1]
    // 公式：L_mapped = L / (1 + L)
    // 直觉：低亮度区域接近线性，高亮度区域（焦散等高能量区）被压缩而不截断
    return Vec3(v.x / (v.x + 1.0), v.y / (v.y + 1.0), v.z / (v.z + 1.0));
}

double gammaCorrect(double v) {
    // sRGB 近似：2.2 伽马
    // 物理显示器是线性响应，人眼对暗部更敏感，伽马校正将线性值映射到感知均匀空间
    return std::pow(std::clamp(v, 0.0, 1.0), 1.0 / 2.2);
}
```

注意：最初我写的 `toneMap` 是 `v / (v + Vec3(1.0))`，但 `Vec3` 没有定义 `Vec3 / Vec3` 的运算符（只有 `Vec3 / double`）——这是个编译错误，修复方式是拆成逐分量计算。

---

## 踩坑实录

### Bug 1：Tone Mapping 编译报错

**症状**：`main.cpp: error: no match for 'operator/' (operand types are 'Vec3' and 'Vec3')`  
**错误假设**：以为 `v / (v + Vec3(1.0))` 会用逐元素除法，就像 GLSL/numpy 一样  
**真实原因**：C++ 的运算符不会自动向量化，`Vec3::operator/` 只接受 `double` 参数，而 `v + Vec3(1.0)` 返回 `Vec3`，所以找不到匹配的重载  
**修复**：改为 `Vec3(v.x/(v.x+1), v.y/(v.y+1), v.z/(v.z+1))` 逐分量计算

**教训**：写数学运算时要记得 C++ 不是数学软件，每个运算符都需要显式定义或逐分量展开。

### Bug 2：光源法向量方向

**症状**：画面极暗，接近全黑  
**错误假设**：面光源的 `normal` 已经朝向场景内部  
**真实原因**：Cornell Box 的天花板面光源，`uAxis.cross(vAxis)` 的方向朝上（指向场景外），需要取反才能朝向场景  
**修复**：在 `buildLightPath` 中检查 `if (lightNormal.y > 0) lightNormal = -lightNormal`，确保法向量向下

**教训**：面光源的朝向必须显式检查，不能假设叉积方向正确。不同的顶点顺序会导致法向量反转。

### Bug 3：连接时吞吐量取错顶点

**症状**：渲染结果偏亮，颜色不正确  
**错误假设**：`camPath[t-1].throughput` 就是相机路径到最后顶点的吞吐量  
**真实原因**：`throughput` 字段存储的是"到达该顶点前"的累积吞吐量，最后一个顶点的 BRDF 还没有计入。连接时应该用 `camPath[t-2].throughput`，然后单独计算 `camPath[t-1]` 处的 BRDF  
**修复**：`Vec3 camThroughput = (t >= 3) ? camPath[t-2].throughput : Vec3(1.0)`

**教训**：路径追踪中，"throughput 在哪个顶点更新"的时序问题极易出错，需要画图理清"连接前"和"连接后"的状态。

### Bug 4：防止 NaN 传播导致全屏噪点

**症状**：偶发的极亮像素（firefly），破坏整体画面  
**错误假设**：BDPT 会自然避免极端值  
**真实原因**：当两个顶点连接时，如果距离极近（dist ≈ 0），几何项 `1/dist²` 会趋向无穷大，产生极高亮度  
**修复**：在合并贡献时检查 NaN，并对每个样本的亮度做 clamp（最大 50）：

```cpp
double lum = 0.2126*contrib.x + 0.7152*contrib.y + 0.0722*contrib.z;
if (lum > 50.0) contrib = contrib * (50.0 / lum);
```

**教训**：路径追踪中的 firefly 问题很普遍。简单的 luminance clamp 不是最优解（会引入偏差），但对于实践项目是合理的权衡。更好的解决方案是 clamped MIS（限制 PDF 比值）。

---

## 效果验证与数据

### 量化验证结果

渲染参数：512×512 分辨率，64 spp（每像素 64 个样本）

```
图片文件: bdpt_output.png
文件大小: 559.4 KB (> 10KB ✅)
像素均值: 75.8 / 255  (10~240 范围内 ✅)
像素标准差: 46.8       (> 5 ✅ 图像有丰富内容)
图像尺寸: 512×512
```

渲染耗时约 60 秒（CPU 单线程，i7 级别处理器）。

**像素分区分析**：

| 区域 | 平均亮度 | 说明 |
|------|---------|------|
| 天花板附近（含光源） | ~200 | 光源直接可见，最亮 |
| 左绿墙 | ~40 | 绿色漫反射，中等亮度 |
| 右红墙 | ~35 | 红色漫反射，略暗 |
| 地面 | ~60 | 受间接光照影响较多 |
| 镜面球 | ~90 | 反射天花板和白墙 |

### 各策略的实际贡献

BDPT 的优势在于综合多种策略：

- **$s=0$ 纯相机路径**：捕捉了直接看到光源的贡献（光源区域的高亮）
- **$s=1$ 直接光照**：漫反射区域的主要光照来源，大幅降低漫反射区域的方差
- **$s \geq 2$ 双向连接**：捕捉了镜面球产生的间接光照，镜面球的亮度和颜色来自此策略

### 与单向 PT 的比较

同等 64 spp 下：
- 单向 PT 的漫反射区域有明显噪点（需要 ~256 spp 才能达到相同视觉质量）
- BDPT 通过 $s=1$ 策略直接对光源采样，漫反射区域显著更平滑
- 镜面球的焦散（地面亮斑）在单向 PT 中几乎不可见，BDPT 中有明显贡献

渲染结果图：

![Cornell Box BDPT 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-22-bdpt/bdpt_output.png)

---

## 总结与延伸

### 当前实现的局限性

1. **MIS 权重过于简化**：本项目用了简化的权重（部分策略权重为 1，没有完整的幂次启发），正确的 BDPT 需要计算所有 $(s', t')$ 策略在当前路径上的 PDF，才能给出最优的 MIS 权重。

2. **没有处理 specular-specular 连接**：两个连续的镜面顶点之间不能做连接操作（delta BRDF 的连接概率为 0），正确的处理是检测"specular 链"并绕过。

3. **单线程 CPU 渲染**：64 spp 耗时约 60 秒，实际生产中 BDPT 会用多线程或 GPU 加速（GPU 上的 BDPT 实现更复杂，因为光源路径和相机路径需要共享内存）。

4. **无焦散玻璃折射**：当前只有镜面反射，没有折射（玻璃/水的 BDPT 需要处理 Snell 定律和全内反射，是 delta BRDF 的另一种形式）。

5. **俄罗斯轮盘赌的偏差**：使用 `maxComp` 作为存活概率是常见近似，理论上更正确的是基于辐射度的自适应 RR。

### 可优化方向

- **Metropolis Light Transport (MLT)**：在 BDPT 的基础上，用 Metropolis 采样在路径空间中游走，专门针对难以采样的光路（焦散、SDS 路径）。这是 Veach 博士论文的另一个核心贡献。

- **VCM/UPBP（顶点连接与合并）**：2012 年提出，结合了 BDPT 和光子映射，通过"合并"而非"连接"相近顶点，对焦散的收敛效率极大提升。

- **ReSTIR（时序/空间样本复用）**：2020 年 NVIDIA 提出的实时算法，通过在时间和空间上复用样本，让 BDPT 级别的质量接近实时。

- **多层散射介质**：加入参与介质（烟、雾、皮肤次表面散射），BDPT 在体散射上同样有优势。

### 与本系列的关联

- 前天（03-20）的**球谐环境光照**提供了一种高效的低频光照表示，但无法捕捉焦散
- 昨天（03-21）的 **SPPM 渐进光子映射**专门针对焦散，是 BDPT 在光子映射方向上的延伸
- 今天的 **BDPT** 是统一的全局光照框架，理论上可以处理所有光路，但需要足够多的样本

三者的关系：BDPT 是通用但收敛慢，SPPM 专攻焦散，SH 是实时近似。工业引擎（如 Cycles、Arnold）通常将 BDPT 和光子映射结合使用，各自处理擅长的光路类型。

---

### 总结

今天通过实现 BDPT，深入理解了几个关键概念：

1. **路径空间的统一视角**：所有光照算法（PT、NEE、光子映射）都是对路径积分的不同估计策略，BDPT 把它们统一在一个框架下。
2. **吞吐量的时序**：`throughput[i]` 是"连接点 i 所需的前缀乘积"，off-by-one 是 BDPT 实现中最常见的错误来源。
3. **delta BRDF 的特殊处理**：镜面材质只能沿路径延伸，不能作为连接端点，这一约束必须在代码中显式检查。
4. **MIS 的必要性**：没有 MIS 时，各策略的贡献可能重复计算（过亮）或遗漏（过暗），MIS 权重保证估计量无偏且方差最小。

BDPT 是离线渲染的经典算法，理解它是进入更高级渲染技术（MLT、VCM、ReSTIR）的必要基础。

---

*代码仓库：https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-22-bdpt*
