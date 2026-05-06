---
title: "每日编程实践: Photon Mapping Renderer（光子映射渲染器）"
date: 2026-05-07 05:30:00
tags:
  - 每日一练
  - 图形学
  - 全局光照
  - C++
  - 光线追踪
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-07-photon-mapping/photon_map_output.png
---

# 每日编程实践: Photon Mapping Renderer

![光子映射渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-07-photon-mapping/photon_map_output.png)

> 经典两阶段全局光照算法：光子发射 + 辐射度估计，在 Cornell Box 中同时渲染间接光照、镜面反射和玻璃焦散。

---

## ① 背景与动机

### 为什么需要光子映射？

在实时渲染和离线渲染中，"全局光照"（Global Illumination，GI）是最难实现又最能提升画面真实感的特性之一。如果没有全局光照，场景会显得非常生硬：

- **间接光照缺失**：光线只有一次弹射，暗角完全是黑色，不自然
- **焦散（Caustics）缺失**：玻璃球或水面聚焦后的明亮光斑根本不存在
- **色溢（Color Bleeding）缺失**：红墙旁边的白墙看不出一点红色反射

传统路径追踪（Path Tracing）虽然能产生正确的全局光照，但它的问题在于：
- **焦散收敛极慢**：从相机出发的路径很难恰好穿过折射路径到达光源，需要数百万 spp 才能消除噪声
- **每帧计算量巨大**：不适合场景固定、只有视角变化的情况

**光子映射（Photon Mapping）** 由 Henrik Wann Jensen 在 1996 年提出，专门解决这两个问题：

1. 将光源的能量以"光子"为单位预先射入场景，构建空间加速结构（kd-tree）
2. 渲染时从相机出发，在漫反射面附近查询光子密度，估计辐射度

这个两阶段设计让焦散的表示变得高效——因为光子是从光源出发，自然会沿折射路径聚集在焦散区域，查询时只需要一个小半径就能采集到足够多的焦散光子。

### 工业界的实际应用

光子映射广泛应用于**离线渲染器**：

- **Arnold（Autodesk）**：用于电影特效，焦散和体积光照处理
- **Luxrender / LuxCore**：开源物理渲染器，核心算法之一
- **PBRT（Physically Based Rendering Toolkit）**：教学渲染器参考实现
- **游戏引擎烘焙工具**：Unreal Engine 的 Lightmass 借鉴了光子映射思路，用于静态场景的间接光照烘焙
- **建筑可视化**：3ds Max 的 Mental Ray 渲染器默认使用光子映射处理室内光照

现代实时渲染（如 Lumen）虽然不直接用光子映射，但其"辐射缓存"、"探针重用"的思路都深受光子映射影响——先预计算、再查询的两阶段范式正是光子映射最核心的遗产。

---

## ② 核心原理

### 渲染方程回顾

光子映射本质是对渲染方程的一种数值求解方法。渲染方程是：

$$L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o) \cdot L_i(x, \omega_i) \cdot \cos\theta_i \, d\omega_i$$

- $L_o$：从 $x$ 点沿 $\omega_o$ 方向出射的辐射亮度
- $L_e$：自发光项
- $f_r$：BRDF（双向反射分布函数）
- $L_i$：从 $\omega_i$ 方向入射的辐射亮度
- $\cos\theta_i$：入射角余弦

在漫反射表面（Lambertian），$f_r = \rho / \pi$（$\rho$ 是反射率），方程化简为：

$$L_o(x) = L_e(x) + \frac{\rho}{\pi} \int_{\Omega} L_i(x, \omega_i) \cdot \cos\theta_i \, d\omega_i$$

光子映射把这个积分分成两部分：
- **直接光照**：光源直接照射的贡献，用 Next Event Estimation（NEE）精确采样
- **间接光照**：一次及以上弹射后的光照，用光子图估计

### 辐射度估计（Radiance Estimation）

光子图中每个光子携带功率（Power，单位 W），而渲染方程需要的是辐射亮度（Radiance，单位 W/m²·sr）。需要从功率估计辐射亮度。

设在点 $x$ 附近半径 $r$ 球形区域内有 $n$ 个光子，第 $k$ 个光子功率为 $\Phi_k$，则辐射亮度估计为：

$$\hat{L}(x, \omega_o) \approx \frac{1}{\pi r^2} \sum_{k=1}^{n} f_r(x, \omega_k, \omega_o) \cdot \Phi_k$$

分母 $\pi r^2$ 是投影面积（对 Lambertian 表面，上半球面积投影到表面是 $\pi r^2$）。

**直觉解释**：功率密度（单位面积的功率）= 总功率 / 面积。在漫反射面上，辐射亮度正比于功率密度除以 $\pi$（因为 Lambertian 的 BRDF $= 1/\pi$）。

这个估计是有偏的（biased）——当 $r \to 0$ 且光子数 $\to \infty$ 时才趋近真值。但有偏不是问题，只要偏差随分辨率提升而减小（consistent），结果就是可接受的。

### 光子追踪：Russian Roulette

发射光子时，光子在每次漫反射面上都要决定是否继续传播。如果每次都衰减功率，光子功率会趋向 0，数值精度损失。

**俄罗斯轮盘（Russian Roulette）** 是解决方案：

- 设漫反射率最大分量为 $p = \max(r, g, b)$
- 以概率 $p$ 继续传播，光子功率不变
- 以概率 $(1-p)$ 终止传播

这样期望功率保持不变，但光子数减少了。数学上：

$$E[\Phi_{continue}] = p \cdot \frac{\Phi}{p} = \Phi$$（此处我们选择不归一化，只控制概率）

在代码中：
```cpp
float prob = std::max({mat.color.x, mat.color.y, mat.color.z});
prob = std::min(prob, 0.95f);  // 限制最大概率，防止无限递归
if (randf() > prob) break;     // 以 1-prob 概率终止
power = power * mat.color * (1.f / prob);  // 功率除以 prob 保持期望不变
```

### kd-tree 最近邻搜索

光子图中有数十万个光子，每个着色点需要查询附近的所有光子。如果暴力遍历是 $O(n)$，渲染 512×512 图像需要 $2.6 \times 10^{10}$ 次比较，完全不可接受。

**kd-tree（k 维树）** 是一种二叉空间划分数据结构：

- **构建**：递归地找到当前点集的最大范围轴（比如 X、Y、Z 中最大跨度的那个），取中位数点作为分割，左子树放坐标小的点，右子树放坐标大的点
- **查询**：对于查询球（中心 $q$，半径 $r$），先检查当前节点是否在球内，然后判断查询球与分割平面的关系，只递归进相关子树

查询复杂度平均为 $O(\log n)$，构建复杂度 $O(n \log n)$。

**分割轴选择的直觉**：选最大范围轴保证树尽量平衡，避免退化为线性链表。

```cpp
// 选择最大范围轴
Vec3 span = mx - mn;
int axis = 0;
if (span.y > span.x) axis = 1;
if (span.z > (axis==0?span.x:span.y)) axis = 2;
```

### 焦散（Caustics）的表示

焦散是光线经过镜面反射或折射后聚焦在漫反射面上的高亮图案。它的特点是：
- 能量高度集中在小区域
- 路径：光源 → 一次或多次镜面/折射 → 漫反射面

光子映射用**独立的焦散光子图**来表示焦散：
- 只存储经过至少一次镜面/折射弹射后打到漫反射面的光子
- 查询时用更小的半径（$r = 0.06$，而全局图 $r = 0.12$），获得更锐利的焦散

```cpp
bool causticPath = false;
// 经过镜面或折射后，标记为焦散路径
if (mat.type == MIRROR || mat.type == GLASS) {
    causticPath = true;
}
// 打到漫反射面且是焦散路径时，存入焦散图
if (mat.type == DIFFUSE && causticPath) {
    causticPhotons.push_back(ph);
}
```

### Schlick Fresnel 近似

玻璃材质同时有反射和折射，分量比例由 Fresnel 方程决定。精确 Fresnel 计算繁琐，Schlick 近似是业界标准：

$$F(\theta_i) = F_0 + (1 - F_0)(1 - \cos\theta_i)^5$$

其中 $F_0 = \left(\frac{n_1 - n_2}{n_1 + n_2}\right)^2$ 是法线入射时的反射率。

**直觉**：掠射角（$\theta_i \to 90°$）时，$(1-\cos\theta_i)^5 \to 1$，反射率趋向 1——这正是我们日常观察玻璃的体验：正面看几乎全透，侧面看几乎全反射。

---

## ③ 实现架构

### 整体数据流

```
光源
  ↓ 随机方向发射 200,000 光子
光子追踪
  ├─ 漫反射面 → 存入全局光子图
  └─ 焦散路径漫反射 → 存入焦散光子图
         ↓
kd-tree 构建（两棵树独立）
         ↓
相机出发，逐像素采样
  每像素 32 条路径
    ├─ 打到漫反射面
    │   ├─ 直接光照（NEE）
    │   ├─ 全局光子图查询（r=0.12）→ 间接光照
    │   └─ 焦散光子图查询（r=0.06）→ 焦散
    ├─ 打到镜面球 → 递归反射
    └─ 打到玻璃球 → Schlick + 折射
         ↓
ACES 色调映射 + sRGB Gamma 校正
         ↓
写出 PNG
```

### 关键数据结构

**`Photon` 结构体**（24 字节）：
```cpp
struct Photon {
    Vec3 pos;    // 光子打到的位置（12 字节）
    Vec3 power;  // RGB 功率（12 字节）
    Vec3 dir;    // 入射方向（12 字节）
};
```

为什么存 `dir`？计算辐射度时需要判断光子是否来自表面正面（背面光子不应该贡献）：
```cpp
if (ph->dir.dot(normal) >= 0) continue;  // 剔除背面光子
```

**`KdNode` 结构体**：
```cpp
struct KdNode {
    Photon photon;  // 节点存储的光子
    int left, right; // 子节点索引（-1表示叶节点）
    int axis;        // 分割轴（0=X, 1=Y, 2=Z）
};
```

用整数索引代替指针，避免内存碎片，对 cache 更友好。

**`PhotonMap` 类**：
```cpp
class PhotonMap {
    std::vector<KdNode> nodes;  // kd-tree 节点数组
public:
    void build(std::vector<Photon>& photons);
    void query(const Vec3& pos, float r, std::vector<const Photon*>& result) const;
};
```

**两个 PhotonMap 实例**：
```cpp
PhotonMap globalMap;   // 全局间接光照（查询半径 0.12）
PhotonMap causticMap;  // 焦散效果（查询半径 0.06）
```

### 材质系统

```cpp
enum MatType { DIFFUSE, MIRROR, GLASS };

struct Material {
    Vec3    color;    // 漫反射颜色 / 镜面颜色 / 玻璃颜色
    Vec3    emission; // 自发光（光源用）
    MatType type;
    float   ior;      // 折射率（GLASS 用，默认 1.5）
};
```

Cornell Box 使用 7 种材质：
- 白色漫反射（地板/天花板/后墙）
- 红色漫反射（左墙）
- 绿色漫反射（右墙）
- 完美镜面（左球，产生镜像反射）
- 玻璃（右球，IOR=1.5，产生焦散）
- 黄色漫反射（小球）
- 高亮自发光（面光源）

### 场景求交

场景包含 Sphere 和 Plane 两种几何体：

```cpp
// 球体求交（二次方程）
bool Sphere::intersect(const Ray& ray, float tmin, float tmax, HitInfo& h) const;

// 平面求交（ray-plane，解析解）
bool Plane::intersect(const Ray& ray, float tmin, float tmax, HitInfo& h) const;
```

`sceneIntersect` 遍历所有几何体，取最近交点：
```cpp
bool sceneIntersect(const Ray& ray, HitInfo& h, bool skipLight=false);
// skipLight=true: 光子追踪时跳过光源球体本身
```

---

## ④ 关键代码解析

### kd-tree 构建

```cpp
int buildRecursive(std::vector<Photon>& ph, int l, int r) {
    if (l >= r) return -1;

    // 1. 找当前点集的 AABB（轴对齐包围盒）
    Vec3 mn = ph[l].pos, mx = ph[l].pos;
    for (int i = l+1; i < r; i++) {
        mn.x = std::min(mn.x, ph[i].pos.x); mx.x = std::max(mx.x, ph[i].pos.x);
        // ... Y, Z 同理
    }

    // 2. 选最大范围轴
    Vec3 span = mx - mn;
    int axis = 0;
    if (span.y > span.x) axis = 1;
    if (span.z > (axis==0?span.x:span.y)) axis = 2;

    // 3. std::nth_element：O(n) 时间找中位数并部分排序
    int mid = (l + r) / 2;
    std::nth_element(ph.begin()+l, ph.begin()+mid, ph.begin()+r,
        [axis](const Photon& a, const Photon& b){
            return (&a.pos.x)[axis] < (&b.pos.x)[axis];
        });

    // 4. 插入当前节点
    int idx = (int)nodes.size();
    nodes.push_back({});
    nodes[idx].photon = ph[mid];
    nodes[idx].axis   = axis;

    // 5. 递归左右子树
    int leftIdx  = buildRecursive(ph, l, mid);
    int rightIdx = buildRecursive(ph, mid+1, r);
    nodes[idx].left  = leftIdx;
    nodes[idx].right = rightIdx;
    return idx;
}
```

**关键：`std::nth_element` 而不是完整排序**。`nth_element` 只保证第 mid 个元素是"如果完整排序时应该在那里的值"，左边的都比它小，右边的都比它大，但左右各自内部不排序。这比 `std::sort` 快一个数量级（$O(n)$ vs $O(n \log n)$），对大量光子非常重要。

### kd-tree 球形查询

```cpp
void queryRecursive(int idx, const Vec3& pos, float r2,
                    std::vector<const Photon*>& result) const {
    if (idx < 0) return;

    const KdNode& node = nodes[idx];

    // 1. 检查当前节点是否在查询球内
    float d2 = (node.photon.pos - pos).len2();
    if (d2 <= r2) result.push_back(&node.photon);

    // 2. 计算查询点到分割平面的距离
    int axis = node.axis;
    float diff = (&pos.x)[axis] - (&node.photon.pos.x)[axis];

    // 3. 优先递归查询点所在的子树
    int first  = diff < 0 ? node.left  : node.right;
    int second = diff < 0 ? node.right : node.left;

    queryRecursive(first, pos, r2, result);

    // 4. 只有当查询球可能跨过分割平面时，才递归另一子树
    if (diff*diff <= r2)
        queryRecursive(second, pos, r2, result);
}
```

**关键剪枝**：`diff*diff <= r2` 判断查询球是否与分割平面相交。`diff` 是查询点到分割平面的有符号距离，`diff*diff` 是平方距离。只有当查询球足够大、能跨越分割平面时，才需要递归另一侧——这是 kd-tree 查询效率的核心所在。

注意：我传入的是 `r2`（半径平方）而不是 `r`，避免每次计算平方根。

### 光子发射核心逻辑

```cpp
void emitPhotons(int numPhotons) {
    Vec3 totalPower = materials[lightMatId].emission * (4*PI*lightRadius*lightRadius * PI);
    Vec3 photonPower = totalPower * (1.f / numPhotons);

    for (int i = 0; i < numPhotons; i++) {
        // 1. 从球形光源表面均匀采样
        Vec3 lightNorm = randomSphere();
        Vec3 origin    = lightCenter + lightNorm * (lightRadius + EPS);

        // 2. 从半球（朝外）采样发射方向
        Vec3 dir = randomHemisphere(lightNorm);
        Ray  ray(origin, dir);
        Vec3 power = photonPower;

        bool causticPath = false;
        bool prevSpecular = true;  // 来自光源，视为特殊

        for (int depth = 0; depth < 8; depth++) {
            HitInfo h;
            // skipLight=true 防止光子立刻打回光源
            if (!sceneIntersect(ray, h, true)) break;

            const Material& mat = materials[h.matId];
            if (mat.emission.len2() > 0) break;  // 自发光体不存储

            if (mat.type == DIFFUSE) {
                // 非直接（至少弹射一次后）才存入全局图
                if (!prevSpecular) {
                    globalPhotons.push_back({h.pos, power, ray.d});
                }
                // 焦散路径（经过镜面/折射）才存入焦散图
                if (causticPath) {
                    causticPhotons.push_back({h.pos, power, ray.d});
                    causticPath = false;
                }

                // Russian Roulette
                float prob = std::min(std::max({mat.color.x,mat.color.y,mat.color.z}), 0.95f);
                if (randf() > prob) break;
                power = power * mat.color * (1.f / prob);
                ray = Ray(h.pos + h.normal*EPS, randomHemisphere(h.normal));
                prevSpecular = false;

            } else if (mat.type == MIRROR) {
                Vec3 refl = ray.d - h.normal * 2.f * ray.d.dot(h.normal);
                power = power * mat.color;
                ray = Ray(h.pos + h.normal*EPS, refl);
                causticPath = true;
                prevSpecular = true;

            } else { // GLASS
                // Snell 定律 + Schlick Fresnel（代码略，见下方折射章节）
                causticPath = true;
                prevSpecular = true;
            }
        }
    }
}
```

**为什么 `prevSpecular` 初始为 `true`？**  
光源直接照射到漫反射面是直接光照，我们用 NEE 处理直接光照，不需要光子图存这部分。`prevSpecular=true` 表示"上一次弹射是镜面类型（或光源出发）"，直接命中漫反射面时跳过存储。

### 玻璃折射（含全内反射）

```cpp
} else { // GLASS
    float ni = 1.0f, nt = mat.ior;
    Vec3 n = h.normal;
    float cosi = -ray.d.dot(n);

    // 判断光线从内部还是外部射入
    if (cosi < 0) {
        std::swap(ni, nt);   // 交换折射率
        n = n * -1.f;        // 翻转法线
        cosi = -cosi;
    }

    float eta = ni / nt;
    float k   = 1.f - eta*eta*(1.f - cosi*cosi);  // k < 0 表示全内反射

    if (k < 0) {
        // 全内反射：只有反射，无折射
        Vec3 refl = ray.d + n * 2.f*cosi;
        ray = Ray(h.pos + n*EPS, refl);
    } else {
        // Schlick Fresnel
        float r0 = (ni-nt)/(ni+nt); r0 *= r0;
        float fr  = r0 + (1-r0)*std::pow(1.f-cosi, 5.f);

        if (randf() < fr) {
            // 反射
            Vec3 refl = ray.d + n * 2.f*cosi;
            ray = Ray(h.pos + n*EPS, refl);
        } else {
            // 折射：Snell 定律向量形式
            Vec3 refr = ray.d*eta + n*(eta*cosi - std::sqrt(k));
            ray = Ray(h.pos - n*EPS, refr.norm());  // 注意是 -n*EPS，从另一侧出发
        }
    }
}
```

**`ray.d*eta + n*(eta*cosi - std::sqrt(k))` 的推导**：

Snell 定律标量形式：$n_i \sin\theta_i = n_t \sin\theta_t$

向量形式设法线方向 $\hat{n}$，入射方向 $\hat{d}$（朝内），分解：
- 切向分量：$\hat{d}_t = \eta \hat{d} + (\eta \cos\theta_i - \cos\theta_t)\hat{n}$
- 其中 $\cos\theta_i = -\hat{d} \cdot \hat{n} = \text{cosi}$，$\cos\theta_t = \sqrt{k}$

这就是代码中折射方向的来源。

### 直接光照（Next Event Estimation）

```cpp
Vec3 directLight(const Vec3& pos, const Vec3& normal, const Vec3& color) {
    // 1. 在球形光源表面均匀采样一个点
    Vec3 lightNorm = randomSphere();
    Vec3 lightPoint = lightCenter + lightNorm * lightRadius;

    Vec3 toLight = lightPoint - pos;
    float dist2 = toLight.len2();
    float dist  = std::sqrt(dist2);
    Vec3 toL = toLight * (1.f / dist);

    // 2. 几何因子：两端法线与连线的夹角
    float cosLight   = std::max(0.f, (-lightNorm).dot(toL));  // 光源端
    float cosSurface = std::max(0.f, normal.dot(toL));         // 着色点端
    if (cosLight < EPS || cosSurface < EPS) return Vec3(0);

    // 3. 阴影光线测试
    Ray shadowRay(pos + normal*EPS, toL);
    HitInfo sh;
    if (sceneIntersect(shadowRay, sh, false)) {
        // 如果遮挡物不是光源本身，则被遮挡
        if (sh.t < dist - EPS && sh.matId != lightMatId) return Vec3(0);
    }

    // 4. 渲染方程：Lo = BRDF × Li × G × Area / PDF
    float lightArea = 2.f * PI * lightRadius * lightRadius;
    Vec3  Le = materials[lightMatId].emission;
    float G  = cosSurface * cosLight / dist2;
    return Le * color * (G * lightArea / PI);
}
```

**几何因子的意义**：从面光源的一个点采样，样本的概率密度正比于 $1/\text{Area}$，而贡献正比于两端夹角余弦除以距离平方（立体角转换）。分母的 $\pi$ 来自 Lambertian BRDF $= \rho/\pi$，即 $f_r = \text{color}/\pi$，所以 $f_r \cdot \pi = \text{color}$。

---

## ⑤ 踩坑实录

### Bug 1：Vec3 缺少一元负号运算符

**症状**：编译报错 `error: no match for 'operator-' (operand type is 'Vec3')`

**错误位置**：
```cpp
float cosLight = std::max(0.f, (-lightNorm).dot(toL));
//                               ^^^^^^^^^^
```

**错误假设**：以为 `-lightNorm` 会调用二元减法 `operator-(Vec3)`，但二元减需要两个操作数。

**真实原因**：`-lightNorm` 是一元取反，需要 `operator-()` 无参版本，但 Vec3 只定义了二元 `operator-(const Vec3& o)`。

**修复**：
```cpp
// 在 Vec3 中添加一元取反
Vec3 operator-() const { return {-x, -y, -z}; }
```

**教训**：写轻量 Vec3 时容易漏掉一元取反，但在光照计算中经常需要反转法线方向。

---

### Bug 2：光子从光源内部出发导致立刻自交

**症状**：运行时光子全部在第一步就碰到光源球体本身，大量无效光子。

**错误原因**：发射光子的起点设置为 `lightCenter + lightNorm * lightRadius`，但求交时没有跳过光源几何体本身，导致光子在半径 EPS 内就与光源球体相交。

**修复**：`sceneIntersect` 增加 `skipLight` 参数，光子追踪时传 `true`，跳过光源球体的求交：
```cpp
if (!sceneIntersect(ray, h, true)) break;  // skipLight=true
```

---

### Bug 3：全局光图存储了直接光，导致直接光照双倍

**症状**：着色点附近的直接光照区域过曝，特别是地板上靠近光源的区域。

**错误原因**：最初代码在每次漫反射面命中时都存储光子，包括了光源直接打到漫反射面的第一次弹射（直接光照）。而渲染时又通过 NEE 计算了一次直接光，导致双重计数。

**修复**：用 `prevSpecular` 标志，只有当光子经过至少一次漫反射弹射（非直接来自光源或镜面）时才存入全局图：
```cpp
if (!prevSpecular) {  // 不是直接来自光源/镜面
    globalPhotons.push_back({h.pos, power, ray.d});
}
```

---

### Bug 4：kd-tree 中 nodes vector 扩容导致悬空指针

**症状**：偶发性崩溃（AddressSanitizer 报 use-after-free），表现为 `nodes[idx]` 访问了无效内存。

**原因**：`buildRecursive` 是递归的，在递归调用期间，`nodes.push_back()` 可能触发 vector 扩容，导致 `nodes.data()` 指向新地址，但外层调用中保存的 `nodes[idx]` 引用仍指向旧地址。

**修复**：不使用引用保存节点，改为先保存索引，递归完成后再更新左右子节点：
```cpp
int idx = (int)nodes.size();
nodes.push_back({});          // 先占位
nodes[idx].photon = ph[mid];  // 通过索引访问（安全）
nodes[idx].axis   = axis;

int leftIdx  = buildRecursive(ph, l, mid);
int rightIdx = buildRecursive(ph, mid+1, r);
nodes[idx].left  = leftIdx;   // 递归完成后通过索引更新（安全）
nodes[idx].right = rightIdx;
```

**教训**：向 `std::vector` 插入元素后，所有之前获取的迭代器、指针、引用都可能失效。递归数据结构构建时要特别小心。

---

### Bug 5：阴影测试未排除光源本身

**症状**：直接光照几乎全为零，场景异常黑暗，只有间接光照（微弱）。

**原因**：阴影光线射向光源，路径上肯定会先碰到光源球体（距离比目标点稍近），直接被判定为遮挡。

**修复**：阴影测试时，判断相交点不是光源材质，或相交距离比光源距离远：
```cpp
if (sceneIntersect(shadowRay, sh, false)) {
    // 只有不是光源本身、且距离更近时才算遮挡
    if (sh.t < dist - EPS && sh.matId != lightMatId) return Vec3(0);
}
```

---

## ⑥ 效果验证与数据

### 输出图像

![光子映射渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-07-photon-mapping/photon_map_output.png)

图像显示 Cornell Box 场景：
- **间接光照**：地板和墙面有均匀的漫反射光照，阴影不会是纯黑色
- **色溢**：白色墙面靠近红墙处可见轻微红色色溢，靠近绿墙处可见绿色色溢
- **镜面球**：左侧球体清晰反射了周围场景
- **玻璃球焦散**：右侧玻璃球下方地板上可见焦散光斑（由折射聚焦产生）

### 量化验证数据

```
像素统计结果：
- 图片尺寸: 512×512
- 文件大小: 446 KB
- 像素均值: 196.0（范围 10~240 ✅）
- 像素标准差: 43.8（>5，图像有内容变化 ✅）
- 上半部分均值: 214.8（天花板/光源区域）
- 下半部分均值: 177.2（地板区域）
- 坐标系正确：天花板 > 地板 ✅
```

### 光子分布统计

```
全局光子: 517,507 个
焦散光子: 67,037 个
总发射光子: 200,000 个

实际存储比例：
- 全局光子 / 发射光子 ≈ 2.59（多次弹射）
- 焦散光子 / 发射光子 ≈ 0.34（约 34% 光子路径经过折射/镜面）
```

### 性能数据

| 阶段 | 时间 |
|------|------|
| 光子发射（20万光子）| ~0.5 秒 |
| kd-tree 构建 | ~0.2 秒 |
| 渲染（512×512，32 spp）| ~60 秒 |
| **总计** | **~61 秒** |

单线程渲染，无 SIMD 优化。如使用 OpenMP 并行，理论可达 8-16 倍加速（约 4-8 秒）。

每像素平均查询：
- 全局光子查询（r=0.12）：约 50-200 个光子
- 焦散光子查询（r=0.06）：约 0-20 个光子（焦散区域以外为 0）

---

## ⑦ 总结与延伸

### 本次实现的局限性

1. **有偏估计**：光子密度估计是有偏的，半径 $r$ 太大导致模糊，太小导致噪声。需要权衡 $r$ 和光子数量。现代方案（Progressive Photon Mapping）通过迭代缩小 $r$ 来获得无偏结果。

2. **单线程**：没有 OpenMP 并行，效率低。光子追踪和光栅化渲染两个阶段都容易并行化。

3. **光子数量有限**：20万光子在 Cornell Box 中够用，但对于更大场景或更复杂焦散，可能需要 100 万以上光子。

4. **固定查询半径**：全局图用 $r=0.12$、焦散图用 $r=0.06$，这两个值是手工调参的，不同场景需要重新调整。自适应半径（根据局部光子密度动态调整）是更鲁棒的方案。

5. **无次表面散射**：玻璃材质只有折射，没有实现体积散射（参与介质渲染）。

### 可优化的方向

- **Progressive Photon Mapping（PPM）**：Jensen & Christensen 2008。迭代渲染，每次发射新光子并缩小半径，最终收敛到无偏结果。
- **Stochastic Progressive Photon Mapping（SPPM）**：Hachisuka et al. 2009。处理焦散中光泽表面的问题。
- **VCM（Vertex Connection and Merging）**：结合 Bidirectional Path Tracing 和 Photon Mapping，取两者之长。
- **OpenMP 并行**：光子追踪阶段天然可并行，渲染阶段逐像素并行。
- **GPU 光子映射**：光子追踪可用 CUDA/RTX 加速，kd-tree 可改为 BVH（更适合 GPU）。

### 与本系列其他文章的关联

- **2026-05-01 ReSTIR**：也是处理多光源采样问题，但方向相反——ReSTIR 从相机出发、光子映射从光源出发
- **2026-05-03 PRT**：球谐函数预计算光照，与光子映射同属"预计算+实时查询"范式
- **2026-04-01 Path Tracing**：光子映射的渲染阶段借鉴了路径追踪的 Russian Roulette 和 NEE
- **2026-04-02 BVH**：本次使用 kd-tree，BVH 是更常见的实时渲染加速结构，各有优势

光子映射是理解全局光照的绝佳教学工具——它把"光是什么"（携带功率的粒子）和"渲染是什么"（估计辐射密度）的物理直觉直接映射到代码结构，是每位渲染工程师的必学算法之一。

---

*代码仓库: [daily-coding-practice/2026/05/05-07-photon-mapping](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-07-photon-mapping)*
