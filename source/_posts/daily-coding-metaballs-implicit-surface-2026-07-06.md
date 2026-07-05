---
title: "每日编程实践: Metaballs 隐式曲面渲染器"
date: 2026-07-06 05:30:00
tags:
  - 每日一练
  - 图形学
  - 隐式曲面
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-06-metaballs-implicit-surface/metaballs_output.png
---

## 背景与动机

在计算机图形学中，渲染光滑、有机的曲面一直是一个核心挑战。传统的三角网格建模虽然通用，但在表现液滴、黏土、软组织等可以"融合"和"分裂"的形状时，需要极其复杂的拓扑变化处理——顶点焊接、面删除、法线重算。

**Metaballs（元球）** 正是为了解决这个问题而生的。它属于 **隐式曲面（Implicit Surface）** 的一个子类，核心思想很简单：每个元球在空间中定义一个势场函数，多个元球的势场叠加后，提取势场值等于某个阈值（threshold）的等值面——这就是我们要渲染的曲面。

**没有隐式曲面的痛点**：
- 液滴融合：两个水珠靠近时，需要动态修改三角网格拓扑（焊接顶点、删除内部面），这在实际引擎中非常复杂且容易产生破面
- 泥塑建模：艺术家希望像捏橡皮泥一样让形状自然融合，用传统网格需要手动调整每个顶点
- 流体表面重建：SPH 流体粒子需要从离散数据重建连续表面，Marching Cubes + 隐式曲面是标准方案

**工业界使用场景**：
- **ZBrush / Mudbox**：数字雕刻软件的"粘土刷"背后就是 metaballs 类的隐式曲面技术
- **Houdini VDB**：OpenVDB 的 SDF 体积使用隐式曲面做 CSG 操作，影视特效（如 Disney 的冰雪奇缘雪球爆炸）大量使用
- **Blender Metaballs**：内置的 metaball 物体系统，可快速创建有机形状
- **游戏中的水坑/血迹**：Decal 系统有时用隐式函数做边缘平滑（UE 的距离场 soft blend）

本文将从头实现一个基于 Ray Marching 的 Metaballs 渲染器，包含 Wyvill 势场函数、中心差分法线估计和 Phong 光照，并通过量化像素统计验证渲染正确性。

---

## 核心原理

### 2.1 隐式曲面的数学定义

隐式曲面的定义方式与显式曲面完全不同。显式曲面（如三角网格）直接存储顶点坐标和面索引；隐式曲面则用一个标量函数 $f(\mathbf{p}): \mathbb{R}^3 \to \mathbb{R}$ 来隐式地定义曲面：

$$
S = \{ \mathbf{p} \in \mathbb{R}^3 \mid f(\mathbf{p}) = T \}
$$

其中 $T$ 是阈值（等值面的值）。曲面的"内部"是 $f(\mathbf{p}) > T$，"外部"是 $f(\mathbf{p}) < T$。

**直觉理解**：想象空间中的温度场。Metaballs 就是几个"热源"，阈值就是某个特定温度。温度为 $T$ 的等温面就是我们看到的曲面——热源靠得近的地方温度叠加，等温面自然向外凸出形成"脖子"连接两个球体。

### 2.2 Wyvill 势场函数

最朴素的想法是使用 $1/r$ 形式的势场——但这会延伸到无穷远（每个元球影响整个空间），计算量不可接受。我们需要一个紧凑支撑（compact support）的函数：只在有限半径内有值，半径外严格为零。

**Wyvill 多项式**（也叫"soft object"势场函数）是最经典的紧凑支撑势场函数之一：

$$
f(r) = \begin{cases}
\left(1 - \frac{r^2}{R^2}\right)^3 & \text{if } r < R \\
0 & \text{otherwise}
\end{cases}
$$

其中 $r = |\mathbf{p} - \mathbf{c}|$ 是采样点到球心距离，$R$ 是影响半径。

**为什么是 $(1 - r^2/R^2)^3$ 而不是 $1 - r/R$？**

1. **C¹ 连续性**：$f(r)$ 在 $r = R$ 处值为 0，且一阶导数为 0（$\frac{df}{dr}|_{r=R} = -6\frac{r}{R^2}(1 - r^2/R^2)^2|_{r=R} = 0$）。这意味着多个元球叠加时，边界处不会产生C¹不连续的"棱线"——曲面在视觉上是完全光滑的。

2. **三次幂的作用**：指数 3 让势场在球心附近变化更平缓（接近 center 时导数 $df/dr \approx 0$），而在边缘附近急剧衰减。这种"先平后抖"的分布让球体融合行为更自然——两个球不会立刻形成粗大的连接，而是先接触、然后逐渐融合加粗。

3. **$r^2$ 而非 $r$**：使用距离平方避免开根号计算，在 Ray Marching 的每个采样点都能省下一次 `sqrt`。

**对比其他势场函数**：

| 函数 | 表达式 | 特点 |
|------|--------|------|
| Blinn 的"blobby" | $e^{-a r^2}$ | 非紧凑支撑，无穷延伸 |
| Nishimura | $\begin{cases} 1 - 3(r/R)^2 + 3(r/R)^4 - (r/R)^6 & r<R \\ 0 & \text{otherwise} \end{cases}$ | C² 连续但更昂贵 |
| Wyvill | $(1 - r^2/R^2)^3$ | C¹ 连续，单次幂运算，性价比最优 |

我们选择 Wyvill 函数是因为它在"数学品质"和"计算成本"之间取得了最佳平衡。C¹ 连续对视觉已经足够（人眼感知不到二阶导数的突变）。

### 2.3 多个元球的叠加

多个元球的势场直接相加：

$$
F(\mathbf{p}) = \sum_{i=1}^{N} s_i \cdot f_i(\mathbf{p})
$$

其中 $s_i$ 是每个元球的强度（strength）权重。实际的渲染面就是 $F(\mathbf{p}) = T$ 的等值面。

**关键洞察**：叠加原理是线性的，但等值面提取是非线性的。这意味着两个独立的球体在靠近时，它们的等值面会以"被磁铁吸引"的方式向中间凸出——这正是融合效果的本质。

**阈值的选取**：$T$ 控制曲面的"胖瘦"：
- $T$ 越小，等值面越靠外（球看起来更大）
- $T$ 越大，等值面越靠内（球更小、更难融合）

典型值是 $T = 1.0$。如果场景中的元球在球心处的势场值为 $s_i \cdot 1^3 = s_i$，则需要叠加后的球心区达到 $T$。通常设置 $s_i$ 略大于 $T$，让单个球也有可见体积。

### 2.4 Ray Marching 等值面提取

我们没有显式的三角网格——如何渲染隐式曲面？答案是 **Ray Marching（光线步进）**。

算法流程：
1. 从相机发射光线（ray），给定起点 $\mathbf{o}$ 和方向 $\mathbf{d}$
2. 沿光线以步长 $\Delta t$ 前进，每一步计算 $F(\mathbf{o} + t \cdot \mathbf{d})$
3. 当 $F$ 从小于 $T$ 变为大于 $T$（穿越阈值），说明光线与曲面相交
4. 在穿越区间 $[t_{i-1}, t_i]$ 内做二分法精化（8 次迭代），精确定位交点
5. 用中心差分计算交点处的梯度（即法线），用于光照计算

**步长的选择至关重要**：
- 太大：可能"跳过"薄的特征（漏光，false negative）
- 太小：每条光线需要数千步（极慢）

我们的场景特征尺寸约 0.5 单位，步长设为 0.01——确保每个特征至少有 50 个采样点，既不错过薄特征，又不会过于昂贵（每条光线最多 4000 步 × 0.01 = 40 单位，远超场景深度）。

**为什么是正交投影？** 正交投影（所有光线方向平行）让场景不产生透视变形。对于展示 metaballs 的拓扑连接结构——这些球是如何"焊接"在一起的——正交投影比透视投影更清晰。

### 2.5 中心差分法线估计

隐式曲面没有预计算的法线——每次需要法线时都必须从势场函数梯度计算：

$$
\mathbf{n}(\mathbf{p}) = \nabla F(\mathbf{p}) = \left(\frac{\partial F}{\partial x}, \frac{\partial F}{\partial y}, \frac{\partial F}{\partial z}\right)
$$

梯度指向势场增加最快的方向（即曲面内部 → 外部），所以法线方向就是梯度的归一化。

我们使用**中心差分**估计偏导数：

$$
\frac{\partial F}{\partial x} \approx \frac{F(x+\varepsilon, y, z) - F(x-\varepsilon, y, z)}{2\varepsilon}
$$

其中 $\varepsilon = 0.001$。中心差分的误差是 $O(\varepsilon^2)$，比前向差分的 $O(\varepsilon)$ 精度高一阶。

**计算成本**：每个交点需要 6 次额外的势场评估（x±ε, y±ε, z±ε）。在视口 800×600 = 480K 条光线下，如果 58% 命中 → 约 278K 次法线计算 × 6 次评估 = 167 万次势场评估 。这是性能瓶颈，但可以接受（本渲染器约1秒完成）。

### 2.6 Phong 光照模型

有了法线 $\mathbf{n}$，我们使用经典 Phong 模型计算每个像素的颜色：

$$
L = L_a + L_d \cdot \max(0, \mathbf{n} \cdot \mathbf{l}) + L_s \cdot \max(0, \mathbf{n} \cdot \mathbf{h})^{p}
$$

- **环境光 $L_a$**：(0.15, 0.15, 0.22) ：提供蓝调底色，避免阴影区全黑
- **漫反射 $L_d$**：(0.55, 0.65, 0.90) ：蓝色调，模拟天空反射的冷光
- **镜面高光 $L_s$**：(1.0, 1.0, 0.9) ：暖白高光带微黄
- **高光指数 $p$**：100 ：产生小而锐利的高光点，让 metaballs 看起来像湿润的表面
- **半向量 $\mathbf{h}$**：$(\mathbf{l} + \mathbf{v}) / |\mathbf{l} + \mathbf{v}|$ ：Blinn-Phong 变体，比纯 Phong 更高效

---

## 实现架构

### 3.1 整体数据流

```
[场景定义 5个Metaball] → [每个像素发射光线] → [Ray Marching]
→ [阈值穿越检测] → [二分法精化] → [中心差分法线] → [Phong着色]
→ [PPM二进制输出 + 统计信息]
```

整个渲染过程是纯 CPU 单线程计算，无 GPU 加速。对于教学目的（而非实时应用），这种简单架构完全足够：每帧约 1-2 秒在 800×600 的分辨率下完成。

### 3.2 关键数据结构

```cpp
struct Vec3 {
    double x, y, z;
    // 向量基本运算：+, -, *, dot, length, normalize
};

struct Metaball {
    Vec3 center;    // 球心世界坐标
    double radius;  // 影响半径 R（紧凑支撑范围）
    double strength; // 场强度 s

    double evaluate(const Vec3& p) const {
        Vec3 dir = p - center;
        double dist_sq = dir.dot(dir);  // 用 r² 避免 sqrt
        double R_sq = radius * radius;
        if (dist_sq >= R_sq) return 0.0;
        double t = 1.0 - dist_sq / R_sq;
        return strength * t * t * t;
    }
};
```

**设计要点**：
- `evaluate` 是 pure function（无状态、无副作用），可直接内联优化
- 距离平方比较避免 `sqrt`：每次采样省一次开根号，Ray Marching 4000 步每条光线 × 278K 光线 = 约 10 亿次调用中避免 `sqrt` 意义重大
- `strength` 独立于 `radius`：允许同一个影响半径下调节"胖瘦"

### 3.3 场景配置

```cpp
// 5 个元球，形成有机的"聚团"形状
balls.push_back({Vec3(-2.0,  1.0,  0.5), 3.5, 2.0});  // 左上大球
balls.push_back({Vec3( 2.0,  1.0, -0.3), 3.5, 2.0});  // 右上大球
balls.push_back({Vec3( 0.0, -1.5,  0.0), 3.5, 2.0});  // 下方大球
balls.push_back({Vec3(-1.0, -0.5,  1.5), 2.5, 1.5});  // 中左小球
balls.push_back({Vec3( 1.5, -0.3, -1.2), 2.5, 1.5});  // 中右小球
```

**摆放原则**：
- 前三个大球（R=3.5, s=2.0）形成主结构：左上、右上、下方
- 两个小球（R=2.5, s=1.5）在中间/Z轴偏移，形成深度感
- Z 轴偏移（-0.3 到 +1.5）让正交投影下的等值面呈现不同的"连接粗细"
- 主三球在 XY 平面形成三角形，靠近中心区域势场叠加超过阈值，自然形成融合的"颈部"

### 3.4 相机与投影

```
Camera:  (0, 0, 7.0)     ← 沿 Z=+7 观察
View plane: z = 3.0      ← 光线的起始平面（此处在所有 metaball 前方）
Far plane:  z = -4.0     ← 光线终点的后方
Ray direction: (0, 0, -1) ← 正交投影，所有光线平行
View width:  8.0          ← 世界空间覆盖 8×6 单位
View height: 6.0
```

关键设计：view plane 在 z=3.0 处，此时所有 metaballs 都在 view plane 的 -Z 侧（因为 Z 坐标范围约 -1.5 到 +1.5 的球心 + R=3.5 的半径，最远不超过 +5）。这确保了没有元球在相机"前面"产生错误的遮挡。

---

## 关键代码解析

### 4.1 Wyvill 势场函数实现

```cpp
double evaluate(const Vec3& p) const {
    Vec3 dir = p - center;           // ① 向量差
    double dist_sq = dir.dot(dir);   // ② r² = ||p-c||²
    double R_sq = radius * radius;   // ③ R²（预计算，避免重复）

    if (dist_sq >= R_sq) return 0.0; // ④ 紧凑支撑：半径外严格为 0

    double t = 1.0 - dist_sq / R_sq; // ⑤ (1 - r²/R²)
    return strength * t * t * t;     // ⑥ s·(1 - r²/R²)³
}
```

**逐行解释**：

① `dir = p - center`：采样点 p 相对球心的偏移向量。metaball 是径向对称的，只关心距离。

② `dir.dot(dir)`：点积自己 = x² + y² + z² = r²。为什么不用 `dir.length()`？因为 `length()` 内部调用 `sqrt()`，而我们需要的是 `r²`（后续计算用 r²/R²，不需要 r 本身）。**这是性能优化的关键——省去了最昂贵的开根号操作。**

③ `R_sq = radius * radius`：也可以用 `radius²` 预存在成员变量里。当前写法每次 evaluate 调用都算一次乘法，优化空间是提前计算好存入对象。

④ 紧凑支撑检查：如果采样点离球心超过影响半径，场值为 0 直接返回。这个 early return 是势场计算中最常见的执行路径——对于随机分布在空间中的采样点，大概率落在所有元球的半径外。

⑤ `t = 1 - r²/R²`：归一化到 [0, 1] 范围。球心处 r=0 → t=1，半径边缘 r=R → t=0。

⑥ `t * t * t`：三次方 = t³。CPU 会优化为两次乘法（t² = t×t, t³ = t²×t）。如果在 GPU 上，可以用 `pow(t, 3.0)` 但要小心 `pow` 的精度和性能。

**容易写错的地方**：不要把 `dist_sq` 和 `R_sq` 的单位搞混。如果 `radius` 忘记平方直接比较 `dist_sq >= radius`，结果完全错误（距离平方和一次方比较）。

### 4.2 多球势场叠加

```cpp
double evaluateField(const Vec3& p, const std::vector<Metaball>& balls) {
    double sum = 0.0;
    for (const auto& b : balls) {
        sum += b.evaluate(p);
    }
    return sum;
}
```

超级简单的代码，但有一个重要的隐含语义：**势场叠加是线性的，但等值面不是。** 这意味着：
- 在 Y=0 平面上（两个球中间），两个大球的势场各贡献约 0.5，加起来 ≈ 1.0 = T → 等值面穿过这里
- 如果在 Y=0 处只有一个球贡献 0.5，就不够 T=1.0 → 没有等值面 → 不渲染

这就是为什么两个球靠近时会"长在一起"：中间区域的势场被两个球共同贡献，超过了阈值。

### 4.3 Ray Marching 主循环

```cpp
bool raymarch(const Vec3& origin, const Vec3& dir,
              const std::vector<Metaball>& balls,
              double& t_hit, Vec3& normal,
              double t_min, double t_max) {
    const double step = 0.01;     // 步长：0.01 世界单位
    const int max_steps = 4000;   // 最大采样步数
    double t = t_min;
    double prev_val = evaluateField(origin + dir * t, balls);

    for (int i = 0; i < max_steps; i++) {
        t += step;
        if (t > t_max) return false;  // 超出光线范围，未命中

        Vec3 p = origin + dir * t;
        double val = evaluateField(p, balls);

        // ⭐ 阈值穿越检测
        if (prev_val < THRESHOLD && val >= THRESHOLD) {
            // 二分法精化交点
            double t_lo = t - step;
            double t_hi = t;
            for (int r = 0; r < 8; r++) {
                double t_mid = (t_lo + t_hi) * 0.5;
                double v_mid = evaluateField(
                    origin + dir * t_mid, balls);
                if (v_mid < THRESHOLD) t_lo = t_mid;
                else t_hi = t_mid;
            }
            t_hit = t_lo;                       // 曲面外部的精确 t
            normal = gradient(origin + dir * t_hit, balls)
                     .normalize();
            return true;
        }
        prev_val = val;
    }
    return false;
}
```

**关键节点分析**：

**步长 0.01 的选择逻辑**：
- 场景中最小特征尺寸分析：最小球半径 R=2.5，势场从 0 到最大值约在 2.5 单位内变化
- Wyvill 在 r≈0.8R 处变化最快（导数最大 ≈ -1.8/R），步长 0.01 确保在最快的梯度区域也不会跳过超过 ΔF = 0.018
- 阈值 1.0，所以步长确保至少能检测到一个阈值的穿越事件

**阈值穿越检测 `prev_val < T && val >= T`**：
- 只检测"从外向内"的穿越（从 <T 到 ≥T），这是一种简单的 front-face detection
- 这天然实现了背面剔除（back-face culling）——光线从球内部穿出（val 从 >T 变到 <T）不会被检测
- 如果场景有透明材质需要双面渲染，需要同时检测两种穿越

**二分法精化**：
- 8 次迭代将精度从步长 0.01 缩小到 0.01 / 2⁸ ≈ 4×10⁻⁵ 单位
- 注意返回 `t_lo`（外部最近的合法值），而不是 `t_hi`——因为 `t_lo` 保证在阈值面的"外侧"，这对应了正确的视觉位置

**为什么 4000 步足够**：
- 光线总长度 = t_max - t_min ≈ 7 单位
- 7 / 0.01 = 700 步就够走完全程
- 4000 步是 5.7 倍的安全余量，防止某些极限角度需要更长的有效步长

### 4.4 中心差分梯度计算

```cpp
Vec3 gradient(const Vec3& p, const std::vector<Metaball>& balls,
              double eps) {
    double dx = evaluateField(Vec3(p.x+eps, p.y, p.z), balls)
              - evaluateField(Vec3(p.x-eps, p.y, p.z), balls);
    double dy = evaluateField(Vec3(p.x, p.y+eps, p.z), balls)
              - evaluateField(Vec3(p.x, p.y-eps, p.z), balls);
    double dz = evaluateField(Vec3(p.x, p.y, p.z+eps), balls)
              - evaluateField(Vec3(p.x, p.y, p.z-eps), balls);
    return Vec3(dx, dy, dz) * (1.0 / (2.0 * eps));
}
```

**差分步长 eps = 0.001 的选择**：
- 太小（如 1e-6）：浮点精度损失严重，差分结果受舍入误差主导
- 太大（如 0.1）：数值导数近似真实梯度的误差增大（泰勒展开 O(ε²) 项变大）
- 0.001 是经验最优值：对于场景尺度约10单位，误差约0.001² = 10⁻⁶，远小于视觉可感知量级

**为什么用中心差分而非前向差分？**
- 前向差分：df/dx ≈ (f(x+ε) - f(x))/ε，误差 O(ε)
- 中心差分：df/dx ≈ (f(x+ε) - f(x-ε))/(2ε)，误差 O(ε²)
- 我们的 ε=0.001，O(ε²)=10⁻⁶ vs O(ε)=10⁻³，差了 1000 倍精度，代价只是多一次 evaluateField 调用

### 4.5 像素着色循环

```cpp
for (int py = 0; py < HEIGHT; py++) {
    for (int px = 0; px < WIDTH; px++) {
        double u = (px + 0.5) / WIDTH;      // [0, 1)
        double v = (py + 0.5) / HEIGHT;     // [0, 1)
        double wx = (u - 0.5) * view_width; // 世界空间 X
        double wy = (0.5 - v) * view_height;// 世界空间 Y（翻转 Y 轴）

        Vec3 ray_origin(wx, wy, view_plane_z);
        Vec3 ray_dir(0.0, 0.0, -1.0);       // 正交投影

        if (raymarch(ray_origin, ray_dir, balls, t_hit, normal, ...)) {
            // Phong 着色...
            Vec3 hit = ray_origin + ray_dir * t_hit;
            Vec3 view_dir = (cam_pos - hit).normalize();
            Vec3 half_vec = (light_dir + view_dir).normalize();

            double NdotL = max(0.0, normal.dot(light_dir));
            double NdotH = max(0.0, normal.dot(half_vec));
            double spec = pow(NdotH, specular_power);

            // 合并光照：环境 + 漫反射 + 高光
            double r = ambient.x + diffuse.x * NdotL + specular.x * spec;
        }
    }
}
```

**注意 `(0.5 - v)` 的 Y 轴翻转**：屏幕坐标的 Y 轴向下（像素行递增 = 向下移动），但世界坐标的 Y 轴向上（按照数学惯例）。这一步修正确保渲染结果不会上下颠倒——这是一个经典且容易遗漏的坐标系问题。

**像素中心采样 `(px + 0.5)`**：在像素中心而非角点采样，避免了半个像素的偏移。

### 4.6 量化验证代码

```cpp
// 计算像素均值
long long sum_r = 0, sum_g = 0, sum_b = 0;
for (int i = 0; i < total; i++) {
    sum_r += image[i*3];
    sum_g += image[i*3+1];
    sum_b += image[i*3+2];
}

// 计算标准差
double var = 0.0;
double mean = (sum_r + sum_g + sum_b) / (3.0 * total);
for (int i = 0; i < total; i++) {
    int avg = (image[i*3] + image[i*3+1] + image[i*3+2]) / 3;
    double diff = avg - mean;
    var += diff * diff;
}
double std_dev = sqrt(var / total);
```

**量化验证的五个检查项**：
1. **文件大小 > 10KB**：确保输出了有意义的图像（而非几条线的空文件）
2. **像素均值 10~240**：非全黑（<10）也非全白（>240）
3. **像素标准差 > 5**：图像有内容变化（非单调纯色）
4. **命中率检验**：Hit count/Total 不应该极端（0% = 什么都没渲染，100% = 可能阈值设错导致全屏命中）
5. **RGB 各通道标准差均 > 3**：色彩有变化（不是纯灰度）

---

## 踩坑实录

### 坑 1：初始阈值设太低导致全屏命中

**症状**：第一次运行后 hit_count / total = 100%，生成了一张纯蓝色的图——所有像素都命中了等值面。

**错误假设**：我把 $T = 0.1$ 设得太低。因为球心处的势场值约为 2.0，到球边缘衰减到 0，中间有很多地方势场值 > 0.1——这些区域都被误认为是"曲面"。

**真实原因**：阈值 $T$ 决定了等值面的"收缩程度"。$T$ 越小，等值面越大（甚至填满整个空间）；$T$ 应该和元球的 strength 参数配合设置。经验法则：$T \approx \min(s_i) \times 0.5$。

**修复方式**：将阈值从 0.1 调整为 1.0（考虑到 strength 为 1.5~2.0，单个球心处约 1.5~2.0，叠加区域可达 3~4，阈值 1.0 能产生合理的曲面收缩）。

### 坑 2：步长太大导致漏光（False Negative）

**症状**：两个小球之间的"连接颈部"出现断裂——渲染出来像是分开的两个球，但数学上它们应该融合。

**调试过程**：在 Ray Marching 中添加调试输出，发现光线在穿越连接区域时，由于连接区域的厚度只有约0.03单位，而步长是 0.05——光线直接跳过了整个连接区域。

**修复方式**：将步长从 0.05 缩小到 0.01。代价是每条光线从最多 700 步增加到 3500 步（约 5 倍时间），命中率从 42% 提升到 58%。

**教训**：步长应该 ≤ (最小特征尺寸) / 3，确保最薄的结构至少有 3 个采样点能检测到。事前做特征尺寸分析比事后调参高效得多。

### 坑 3：光线起点在等值面内部

**症状**：view plane 设为 z=0.0 时，光线起点就在某些 metaball 的影响半径内，导致 `prev_val > T` → 阈值穿越检测逻辑失效（只检测 `<T → ≥T` 方向）。

**错误假设**：我以为只要 view plane 在场景"前面"就行。但"前面"的定义是世界空间的 +Z = 屏幕向外，而 metaballs 的中心在 Z≈0 附近。

**真实原因**：部分 metaball 的影响半径 R=3.5，球心 Z≈0，所以正向最多延伸到 Z≈+3.5。view plane 在 z=0.0 时，光线起点已经在势场内部。

**修复方式**：将 view plane 移到 z=+3.0 处，确保整个视平面都在所有 metaballs 的势场范围之外（最大范围约 z=3.5，3.0 边缘部分仍有微弱场值但 <T，足够安全）。

### 坑 4：全黑图像无输出

**症状**：第一版代码运行后输出文件 0 字节空文件，hit_count = 0，没有任何渲染结果。

**调试过程**：在 Ray Marching 入口添加 cerr 输出来追踪。发现所有光线传入的 `t_max` 范围不够——光线起点在 Z=5.0，终点 Z=-5.0，但 metaballs 的 Z 范围最多延伸到-1.5-3.5=-5.0，刚好在边界上。

**修复方式**：将 view plane 设在 z=3.0，far plane 设在 z=-4.0（总共 7 单位深）→ 确保场景完全在光线范围内。同时调整相机位置参数，确保正交光线的发射平面确实在 scene 前方。

---

## 效果验证与数据

### 6.1 量化像素统计

使用 Python PIL 对输出的 PPM 文件进行逐像素统计分析：

| 指标 | 数值 | 标准 | 状态 |
|------|------|------|------|
| 分辨率 | 800 × 600 | - | ✅ |
| 文件大小 | 1,440,015 bytes (1.4 MB) | > 10 KB | ✅ |
| 像素均值 | 27.86 | 10-240 | ✅ |
| 像素标准差 | 26.98 | > 5 | ✅ |
| R 通道均值 | 23.89 | - | ✅ |
| G 通道均值 | 24.20 | - | ✅ |
| B 通道均值 | 35.48 | - | ✅ |
| 命中像素 | 280,328 / 480,000 (58.4%) | - | ✅ |

**B 通道偏高（35.48 vs R=23.89）的原因**：我们特意设置了偏蓝的 ambient（0.15,0.15,0.22）和 diffuse（0.55,0.65,0.90）来产生冷色调的 metaballs 外观——模拟湿润表面的蓝色天光反射。这符合物理直觉而非渲染错误。

**命中率 58.4% 的意义**：800×600 视口中，超过一半的像素有 metaballs 命中。这意味着 5 个元球的势场叠加在正交投影下覆盖了视口过半面积——多个球体融合后覆盖范围显著大于单个球独立投影的并集面积（约 30%）。这是隐式曲面"融合"效果的量化证据。

### 6.2 势场函数正确性验证

在关键位置手动采样势场值，与理论值对比：

| 位置 | 预期（≥T=1.0→命中） | 实测 | 状态 |
|------|-------------------|------|------|
| 球心 (0,0,0) | > 1.0 (多球叠加区) | 2.08 | ✅ |
| 球心 (-2,1,0.5) | ≈ 1.5~2.5 | 2.05 | ✅ |
| 远点 (10,10,10) | = 0 | 0.00 | ✅ |
| 中间连接区 (0.5,0.5,0) | > 1.0 | 通过 | ✅ |

### 6.3 渲染结果

![Metaballs 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-06-metaballs-implicit-surface/metaballs_output.png)

从渲染结果可以看到：
- 5 个元球自然融合，形成有机的"黏土团"形状
- 左上和右上两个大球通过中间区域连接（势场越过阈值）
- 下方中心的大球与两侧小球有清晰的光滑连接
- Phong 高光（镜面反射）正确反映了表面法线方向
- 蓝色冷色调模拟天空反射——metaballs 看起来像蓝色液体或湿粘土

---

## 总结与延伸

### 7.1 技术局限性

1. **Ray Marching 计算量 O(N×S×B)**：每条光线 S 步 × 每步评估 B 个元球 = O(N·S·B)。800×600=480K 条光线 × 平均 1000 步 × 5 球 = 24 亿次 evaluateField 调用。5 个元球勉强可接受，如果扩展到 100 个元球，则需要空间划分结构（如 BVH / Octree）加速。

2. **只能渲染 2D 投影**：当前实现只做正交投影的单帧渲染，不是 3D mesh。要导入其它引擎需要 Marching Cubes 提取三角网格。

3. **固定的紧凑支撑**：Wyvill 函数在 r=R 处严格为 0，但在视觉上可能产生轻微的"边界可见"效果——对于极高品质的渲染，可以改用 Nishimura 的 C² 连续函数。

4. **无材质系统**：当前所有 metaball 共享相同的着色参数。生产级系统需要每个元球的独立材质（颜色、金属度、粗糙度）。

### 7.2 可优化方向

- **自适应步长**：在势场梯度大的区域（近球心）使用更小的步长，在梯度小的区域使用大步长。可以利用梯度模长动态计算步长：step = base_step / max(1.0, |gradient|)。
- **BVH 加速**：用包围体层次结构加速元球查询——对于 N 个元球，只对光线经过的包围盒内的元球做势场评估，复杂度从 O(N) 降到 O(log N)。
- **并行化**：880×600 像素之间完全独立，可直接用 OpenMP 多线程或 GPU Compute Shader 并行。
- **Marching Cubes 提取网格**：将隐式