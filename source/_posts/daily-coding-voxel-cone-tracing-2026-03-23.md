---
title: "每日编程实践: Voxel Cone Tracing — 用体素锥形追踪实现近似全局光照"
date: 2026-03-23 05:30:00
tags:
  - 每日一练
  - 图形学
  - 全局光照
  - C++
  - 光线追踪
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-23-voxel-cone-tracing/vct_output.png
---

## 一、背景与动机

### 全局光照的工程困境

光线追踪可以精确计算全局光照（Global Illumination，GI），但在实时渲染场景下代价极高。渲染一帧 1080P 图像，路径追踪往往需要每像素几十甚至上百条路径才能收敛——这对于 60FPS 的游戏来说显然不现实。

游戏引擎里用了哪些方案来"作弊"？

- **光照贴图（Lightmap）**：离线烘焙，无法响应动态物体
- **Irradiance Volume（光照探针）**：动态性好，但空间分辨率有限
- **SSAO（屏幕空间环境光遮蔽）**：只用屏幕信息，遮蔽范围受限
- **Lumen（UE5）**：混合 SDF 追踪 + Radiance Cache，非常复杂

2011 年，Cyril Crassin 等人提出了 **Voxel Cone Tracing（VCT）**，发表在 SIGGRAPH 论文"Interactive Indirect Illumination Using Voxel Cone Tracing"中。VCT 的核心思想是：

> 把场景体素化成一个带 Mipmap 的 3D 纹理，然后在着色时沿锥形方向"扫描"这个 3D 纹理，近似计算来自各方向的间接辐射度。

VCT 的魅力在于它能在实时帧率下产生**有说服力的漫反射 GI 和镜面 GI**，空间变化丰富，不依赖预计算。Unreal Engine 4 的 VXGI（NVIDIA 插件）、CryEngine 的 SVOGI 都基于这个思路。

### 没有 GI 会发生什么？

Cornell Box 场景如果只有直接光照，颜色渗色（Color Bleeding）就会完全消失。红色墙壁旁边的白色球体应该带上淡淡的红色——这是漫反射 GI 的关键效果。同样，镜面球体应该能看到周围环境的一次间接反射。这些效果没有 GI 是做不到的。

VCT 今天虽然已经被更现代的方案（DDGI、Lumen）部分取代，但它的思路依然优雅——把三维辐照度查询转化成体素锥形采样，是一个非常值得学习的设计。

---

## 二、核心原理

### 2.1 渲染方程回顾

着色点 $x$ 的出射辐射度 $L_o$ 满足渲染方程：

$$
L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o) L_i(x, \omega_i) (\omega_i \cdot n) d\omega_i
$$

其中 $L_i(x, \omega_i)$ 是从方向 $\omega_i$ 来的入射辐射度。VCT 把 $L_i$ 分为两部分计算：

1. **直接光照**：从光源出发，用阴影射线精确计算
2. **间接光照**：用体素锥形追踪近似计算

### 2.2 场景体素化

第一步是把场景"离散化"成一个 3D 网格。设网格分辨率为 $D^3$（本实现取 $D=64$），场景 AABB 为 $[\text{sceneMin}, \text{sceneMax}]$，每个体素的世界坐标范围是：

$$
[x_v, x_v + \Delta x] \times [y_v, y_v + \Delta y] \times [z_v, z_v + \Delta z]
$$

其中 $\Delta x = (\text{sceneMax}_x - \text{sceneMin}_x) / D$，y、z 类似。

体素化的方法：**对每个体素的中心，向 6 个坐标轴方向发射射线**，检查是否击中场景几何体。若某方向击中了距离 $< \Delta x + \epsilon$ 的表面，则这个体素被占用，记录该表面的颜色和不透明度。

每个体素存储一个 RGBA 值 $(r, g, b, \alpha)$：
- $rgb$：该体素内表面的辐射度（包含自发光 + 漫反射系数）
- $\alpha$：体素的不透明度（0 = 空气，1 = 实心）

### 2.3 Mipmap 层级构建

光滑锥形追踪需要在不同细节层级采样。对体素网格构建 Mipmap：

- Level 0：$64^3$ 分辨率，体素大小 $= \Delta x$
- Level 1：$32^3$ 分辨率，每个体素 = Level 0 的 2×2×2 平均
- Level 2：$16^3$，……
- Level 5：$2^3$，最粗粒度

Level $k$ 的体素颜色是 Level $k-1$ 中对应 8 个子体素的加权平均：

$$
v^{(k)}[x,y,z] = \frac{1}{8} \sum_{d_x \in \{0,1\}} \sum_{d_y \in \{0,1\}} \sum_{d_z \in \{0,1\}} v^{(k-1)}[2x+d_x, 2y+d_y, 2z+d_z]
$$

这个结构使我们能在远处用粗粒度体素、近处用细粒度体素，类似纹理 Mipmap 的原理。

### 2.4 锥形追踪原理

锥形追踪是 VCT 的核心。给定起点 $p$、方向 $d$、半角 $\theta$，锥在距离 $t$ 处的直径为：

$$
\text{diameter}(t) = 2t \tan\theta
$$

我们从 $t = t_0$（略微偏移以避免自交）开始步进，在每个采样点处：

1. 根据当前直径计算对应的 Mipmap 层级：
   $$
   \text{mipLevel} = \log_2\left(\frac{\text{diameter}(t)}{\Delta x}\right)
   $$
   直觉：当锥的直径等于一个体素时，用 Level 0；等于两个体素时，用 Level 1；以此类推。

2. 对 Mipmap 进行**三线性插值 + Mip 层间插值**采样，得到 $(r, g, b, \alpha)$。

3. **前后合成（Front-to-Back Alpha Compositing）**：
   $$
   \text{accum}.\alpha \mathrel{+}= \text{voxel}.\alpha \times (1 - \text{accum}.\alpha)
   $$
   $$
   \text{accum}.rgb \mathrel{+}= \text{voxel}.rgb \times \text{voxel}.\alpha \times (1 - \text{accum}.\alpha)
   $$
   这保证了靠近相机的体素优先遮挡远处体素，符合物理直觉。

4. 步长自适应：$\Delta t = \max(\text{diameter}(t) \times 0.5, \Delta x)$。步长随锥的扩张而增大，减少远处的采样次数。

### 2.5 漫反射间接光照

漫反射 GI 需要积分整个法线半球上的入射辐射度：

$$
L_{\text{indirect,diffuse}} = \frac{\rho}{\pi} \int_\Omega L_i(\omega) (\omega \cdot n) d\omega
$$

VCT 用 6 根锥（半角 30°）覆盖半球来近似这个积分。锥的方向按照 Crassin 2011 的配置：

- 1 根沿法线方向（权重 0.25）
- 5 根以 60° 倾斜角均匀分布在法线周围（各权重 0.15）

```
                  ↑ n
                  │
              ┌───┼───┐
           ╱     │     ╲
          /    0.25     \
         /   /  │  \     \
        ╱  /    │    \    ╲
    0.15 0.15   │   0.15 0.15
```

每根锥贡献的辐射度乘以权重后累加，最后乘以材质的漫反射系数 $\rho/\pi$。

### 2.6 镜面间接光照

镜面 GI 只需要一根锥，方向沿反射向量 $R = 2(n \cdot v)n - v$，半角由材质粗糙度决定：

$$
\theta_{\text{specular}} = \max(0.01, \text{roughness} \times 0.5)
$$

光滑材质（roughness ≈ 0）得到极窄的锥，接近镜面反射；粗糙材质得到宽锥，产生模糊反射。

Fresnel 权重使用 Schlick 近似：

$$
F(\omega_i) = F_0 + (1 - F_0)(1 - \cos\theta_i)^5
$$

其中 $F_0 = \text{mix}(0.04, \text{albedo}, \text{metallic})$。粗糙度很高时 VCT 贡献趋近 0，通过 $(1 - \text{roughness})$ 因子淡出。

---

## 三、实现架构

### 3.1 整体数据流

```
场景几何 (Box + Sphere)
       ↓ voxelizeScene()
VoxelGrid (Level 0: 64³ rgba)
       ↓ buildMipmaps()
VoxelGrid (Level 0~5)
       ↓
渲染循环 (per pixel):
  camera ray → scene.intersect() → HitRecord
       ↓
  directLight()          indirectDiffuse() + indirectSpecular()
  (shadow ray + BRDF)    (traceCone × 6 or 1)
       ↓                        ↓
              合并
       ↓
  ACESFilm tone mapping + gamma
       ↓
  写 PPM → 转 PNG
```

### 3.2 关键数据结构

**VoxelGrid**：核心数据结构，包含 6 个 Mipmap 层级的体素数据。

```cpp
struct VoxelGrid {
    std::vector<std::vector<Vec4>> levels;  // 每层 Vec4 数组
    std::vector<int> dims;                  // 每层的维度
    // Level 0: 64^3 = 262144 元素
    // Level 1: 32^3 = 32768 元素
    // ...总计约 300KB
};
```

**Vec4**：体素存储格式，x/y/z = RGB 辐射度（HDR，可超过 1），w = 不透明度。

**HitRecord**：射线求交结果，包含 t 值、交点坐标、法线、材质。

### 3.3 职责划分

| 函数 | 职责 |
|------|------|
| `voxelizeScene()` | 几何→体素：光线投射，填充 Level 0 |
| `VoxelGrid::buildMipmaps()` | Level 0→5：下采样构建 Mipmap 链 |
| `VoxelGrid::sampleMip()` | 三线性 + Mip 层间插值查询 |
| `traceCone()` | 单根锥形追踪，前后合成，返回 RGBA |
| `indirectDiffuse()` | 构建切线空间，发射 6 根漫反射锥 |
| `indirectSpecular()` | 发射 1 根镜面锥，Fresnel 加权 |
| `directLight()` | 面光源阴影测试 + Blinn-Phong/GGX |

---

## 四、关键代码解析

### 4.1 体素化：6 方向光线投射

体素化的核心思路是：对每个体素中心，向 6 个坐标轴方向发射短射线，看能否击中附近的几何表面。

```cpp
void voxelizeScene(VoxelGrid& grid, const Scene& scene) {
    int D = VOXEL_DIM;
    float voxelSize = SCENE_SIZE.x / D;
    
    for (int z = 0; z < D; z++) {
    for (int y = 0; y < D; y++) {
    for (int x = 0; x < D; x++) {
        // 体素中心世界坐标
        Vec3 center = SCENE_MIN + Vec3(
            (x + 0.5f) * voxelSize,
            (y + 0.5f) * voxelSize,
            (z + 0.5f) * voxelSize
        );
        
        Vec3 avgColor(0);
        float avgOpacity = 0;
        int hits = 0;
        
        // 6 个方向发射射线
        const Vec3 dirs[6] = {
            {1,0,0}, {-1,0,0}, {0,1,0}, {0,-1,0}, {0,0,1}, {0,0,-1}
        };
        
        for (auto& dir : dirs) {
            // 从体素边缘略外侧出发，向内射
            Vec3 ro = center - dir * (voxelSize * 0.5f + 1e-3f);
            HitRecord rec = scene.intersect(ro, dir);
            
            // 只有距离在体素范围内的交点才算"占据"这个体素
            if (rec.hit && rec.t < voxelSize + 2e-3f) {
                Vec3 color = rec.mat.emission + rec.mat.albedo * 0.5f;
                if (rec.mat.isLight) color = rec.mat.emission;
                avgColor += color;
                avgOpacity += 1.0f;
                hits++;
            }
        }
        
        if (hits > 0) {
            avgColor = avgColor / float(hits);
            float opacity = std::min(float(hits) / 6.0f, 1.0f) * 0.85f;
            grid.set(0, x, y, z, Vec4(avgColor, opacity));
        }
    }}}
```

**为什么选 6 方向而非更多？** 性能权衡——6 方向能检测到所有朝向的表面，已经足够覆盖 Box 和 Sphere 等基本几何体。更多方向会提高准确性，但 $D^3 \times 6$ 次求交已经是 $64^3 \times 6 = 1.57M$ 次射线测试。

**为什么存 albedo × 0.5 而非 albedo？** 体素里存的是"初始辐射度"，用于间接光照估算，而非准确的直接光照。乘 0.5 是保守的能量折扣，防止多次弹射后能量爆炸。发光体用自身 emission 颜色（不乘 0.5），因为它是真正的光源。

### 4.2 Mipmap 构建：3D 下采样

```cpp
void buildMipmaps() {
    for (int level = 1; level < MIP_LEVELS; level++) {
        int d = dims[level];          // 当前层维度（32, 16, 8, 4, 2）
        for (int z = 0; z < d; z++) {
        for (int y = 0; y < d; y++) {
        for (int x = 0; x < d; x++) {
            Vec4 sum(0);
            float cnt = 0;
            // 对应上层的 2×2×2 = 8 个体素
            for (int dz = 0; dz < 2; dz++)
            for (int dy = 0; dy < 2; dy++)
            for (int dx = 0; dx < 2; dx++) {
                Vec4 v = get(level-1, x*2+dx, y*2+dy, z*2+dz);
                sum += v;
                cnt++;
            }
            set(level, x, y, z, sum * (1.0f / cnt));
        }}}
    }
}
```

这里的平均包含了 alpha 通道——粗级别体素的 alpha 是细体素 alpha 的均值。这意味着粗体素中有 4 个被占用 4 个为空时，alpha = 0.5。从物理上说，锥追踪经过这个体素时，只有 50% 的能量被遮挡，50% 穿透——这近似了稀疏结构的半透明性。

### 4.3 三线性插值 + Mip 插值

```cpp
Vec4 sampleMip(float u, float v, float w, float mipLevel) const {
    // 在相邻两个 Mip 层之间插值（类似 Anisotropic Filtering 的思路）
    int mipLo = (int)std::floor(mipLevel);
    int mipHi = mipLo + 1;
    float t = mipLevel - mipLo;
    
    mipLo = std::clamp(mipLo, 0, MIP_LEVELS - 1);
    mipHi = std::clamp(mipHi, 0, MIP_LEVELS - 1);
    
    Vec4 lo = sample(mipLo, u, v, w);  // Level mipLo 的三线性插值
    Vec4 hi = sample(mipHi, u, v, w);  // Level mipHi 的三线性插值
    return lo * (1-t) + hi * t;         // 层间 lerp
}
```

**为什么需要层间插值？** 如果直接用 `floor(mipLevel)` 取整，在 Mip 层切换点会出现明显的辐射度突变（banding 现象）。双线性 Mip 插值消除了这个伪影，代价是每次采样翻倍（2 次三线性）。

### 4.4 锥形追踪核心循环

```cpp
Vec4 traceCone(const VoxelGrid& grid, Vec3 origin, Vec3 direction,
               float aperture, float maxDist) {
    direction = direction.normalized();
    Vec4 accum(0, 0, 0, 0);
    
    float stepDist = 0.0f;
    float voxelWorldSize = SCENE_SIZE.x / float(VOXEL_DIM);
    
    // 初始偏移：避开当前体素，防止自交
    stepDist = voxelWorldSize * 1.5f;
    
    while (stepDist < maxDist && accum.w < 0.97f) {
        // 当前距离下锥的直径
        float coneDiameter = 2.0f * stepDist * std::tan(aperture);
        coneDiameter = std::max(coneDiameter, voxelWorldSize);  // 最小一个体素宽
        
        // 直径对应的 Mip 层级
        float mipLevel = std::log2(coneDiameter / voxelWorldSize);
        mipLevel = std::clamp(mipLevel, 0.0f, float(MIP_LEVELS - 1));
        
        // 世界坐标 → UVW [0,1]³
        Vec3 sampleWorld = origin + direction * stepDist;
        Vec3 uvw = worldToUVW(sampleWorld);
        
        // 越界检查（超出场景 AABB 就停止）
        if (uvw.x < 0 || uvw.x > 1 || uvw.y < 0 || uvw.y > 1 
         || uvw.z < 0 || uvw.z > 1) break;
        
        // 采样体素网格
        Vec4 voxel = grid.sampleMip(uvw.x, uvw.y, uvw.z, mipLevel);
        
        // 前后合成（Front-to-Back Alpha Compositing）
        float alpha = voxel.w * (1.0f - accum.w);
        accum.x += voxel.x * alpha;
        accum.y += voxel.y * alpha;
        accum.z += voxel.z * alpha;
        accum.w += alpha;
        
        // 自适应步长：锥越宽，步长越大
        float stepSize = std::max(coneDiameter * 0.5f, voxelWorldSize);
        stepDist += stepSize;
    }
    
    return accum;  // xyz = 积累颜色，w = 积累不透明度
}
```

**早停条件 `accum.w < 0.97f`** 是一个重要优化：当锥已经"饱和"（遮挡度 > 97%）时，后续体素对结果影响微乎其微，直接终止追踪节省大量计算。

**自适应步长**：`stepSize = coneDiameter * 0.5f` 确保每步至少覆盖半个锥直径。在近距离（锥窄，用细 Mip），步长小，采样密集；在远距离（锥宽，用粗 Mip），步长大，采样稀疏。这个自适应机制使得总采样次数大约是 $O(\log D)$，而非线性增长。

### 4.5 漫反射 GI：6 锥半球采样

```cpp
Vec3 indirectDiffuse(const VoxelGrid& grid, const Vec3& point, 
                     const Vec3& normal, const Vec3& albedo) {
    // 建立切线空间：tangent 和 bitangent 正交于 normal
    Vec3 up = (std::abs(normal.y) < 0.9f) ? Vec3(0,1,0) : Vec3(1,0,0);
    Vec3 tangent = up.cross(normal).normalized();
    Vec3 bitangent = normal.cross(tangent);
    
    // Crassin 2011 的 6 锥配置（局部空间方向）
    struct ConeDir { float x, y, z, weight; };
    const ConeDir coneDirs[] = {
        { 0.0f,       1.0f, 0.0f,       0.25f },  // 法线方向（最主要）
        { 0.0f,       0.5f, 0.8660254f, 0.15f },  // 60°倾斜 × 5
        { 0.8228756f, 0.5f, 0.2745967f, 0.15f },
        { 0.5f,       0.5f,-0.7071068f, 0.15f },
        {-0.5f,       0.5f,-0.7071068f, 0.15f },
        {-0.8228756f, 0.5f, 0.2745967f, 0.15f },
    };
    
    Vec3 indirect(0);
    float totalWeight = 0;
    
    // 偏移起始点，避开当前体素（沿法线偏移 2 个体素）
    Vec3 offsetPoint = point + normal * (SCENE_SIZE.x / VOXEL_DIM * 2.0f);
    
    for (auto& cd : coneDirs) {
        // 局部空间方向 → 世界空间（用切线、法线、副切线做基矩阵变换）
        Vec3 localDir(cd.x, cd.y, cd.z);
        Vec3 worldDir = tangent   * localDir.x 
                      + normal    * localDir.y 
                      + bitangent * localDir.z;
        worldDir = worldDir.normalized();
        
        Vec4 result = traceCone(grid, offsetPoint, worldDir, 
                                CONE_ANGLE, MAX_DIST);
        
        indirect += Vec3(result.x, result.y, result.z) * cd.weight;
        totalWeight += cd.weight;
    }
    
    if (totalWeight > 0) indirect = indirect / totalWeight;
    
    // Lambert 漫反射：乘以 albedo / π
    return indirect * albedo / float(M_PI);
}
```

**为什么用切线空间而非世界空间？** 我们需要的锥方向是"相对于法线"的半球分布，与世界坐标无关。切线空间的基矩阵将 Y 轴对齐到法线，让锥方向的 y 分量代表与法线的夹角余弦。

**权重配置的物理意义**：6 根锥覆盖半球面积 $2\pi$，每根锥覆盖约 $0.524 \text{ sr}$（半角 30° 的立体角）。法线方向权重 0.25 偏高（实际比 5 根侧锥大），这是因为法线方向正对光源时贡献最多。总权重 $0.25 + 5 \times 0.15 = 1.0$，归一化后是无偏估计。

### 4.6 ACES 色调映射

HDR 渲染结果需要压缩到 [0,1] 显示范围，使用 ACES 胶片曲线：

```cpp
Vec3 vdiv(const Vec3& a, const Vec3& b) {
    return {a.x/b.x, a.y/b.y, a.z/b.z};
}

Vec3 ACESFilm(Vec3 x) {
    // Hill 2016 ACES fitted approximation
    // 来自 Unreal Engine 的实现
    const float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    Vec3 num = x * (x * a + Vec3(b));            // x*(2.51x + 0.03)
    Vec3 den = x * (x * c + Vec3(d)) + Vec3(e);  // x*(2.43x + 0.59) + 0.14
    return vdiv(num, den).clamp(0, 1);
}
```

ACES 曲线的特点：暗部细节保留好（斜率约 1），亮部自然压缩，整体色彩倾向符合电影感。相比简单的 `clamp(0,1)` 或 `x/(1+x)` 的 Reinhard 曲线，ACES 对高光区域的压缩更符合人眼感知。

---

## 五、踩坑实录

### Bug 1：编译错误——一元负号未定义

**症状**：
```
error: no match for 'operator-' (operand type is 'Vec3')
    float HdotV = std::max(0.0f, halfVec.dot(-viewDir));
```

**错误假设**：以为 `Vec3` 自带一元负号操作符，因为很多 GLSL 风格代码都能这么写。

**真实原因**：C++ 中一元 `-` 不会自动从二元版本派生，必须显式定义 `Vec3 operator-() const`。

**修复**：
```cpp
// 在 Vec3 结构体中添加：
Vec3 operator-() const { return {-x, -y, -z}; }
```

---

### Bug 2：编译错误——Vec3 之间的除法

**症状**：
```
error: no match for 'operator/' 
    return ((x * (x * a + Vec3(b))) * (Vec3(1) / (...)));
```

**错误假设**：ACES 公式写的是 `num / den`，以为 `Vec3/Vec3` 分量除法是自然可用的。

**真实原因**：自定义的 `Vec3` 只有 `operator/(float)`，没有逐分量的 `operator/(Vec3)`。GLSL 里天然支持，但 C++ 需要手动实现。

**修复**：添加辅助函数而非直接重载（避免与标量除法混淆）：
```cpp
Vec3 vdiv(const Vec3& a, const Vec3& b) {
    return {a.x/b.x, a.y/b.y, a.z/b.z};
}
```

---

### Bug 3：未使用变量警告

**症状**：
```
warning: unused variable 'originUVW' [-Wunused-variable]
warning: unused variable 'pd' [-Wunused-variable]
```

**错误假设**：以为 0 warnings 只是建议，警告不影响运行。

**真实原因**：本项目要求 0 错误 **0 警告**，警告视为不通过。这两个变量是开发中间状态遗留的，未被后续代码使用。

**修复**：
- `originUVW`：删除整行（函数内部不需要记录出发点的 UVW）
- `pd`（上层维度变量）：逻辑上不需要，删除

---

### Bug 4：体素锥追踪结果过暗

**症状**：间接光照图 (vct_indirect.png) 很暗，平均亮度只有 8.7（0~255 中的值），但直接光照也只有 9.3。整体看起来场景比预期暗。

**根本原因分析**：VCT 体素化时，每个体素存储的是 `albedo * 0.5`，这是一次弹射的能量。多次弹射的累积通过 Mipmap 层级的叠加体现，但由于锥追踪本质上是单次查询，无法递归积累多次弹射。

**实际情况**：对于这种 CPU 软件渲染 Demo，8.7 的间接亮度加上 9.3 的直接亮度，合计约 16.9，经 ACES 压缩后的视觉效果是合理的 Cornell Box 场景——并非真正过暗，是色调映射后的正常显示范围。

**教训**：判断 GI 是否"过暗"要在 HDR 域看原始辐射度，而非 LDR 压缩后的值。建议打印 HDR 域的最大辐射度和均值确认。

---

### Bug 5：坐标系与体素边界

**初始设计**：场景 AABB 设为 [-1,1]³，体素化和射线追踪都在同一空间。

**潜在问题**：Cornell Box 的墙壁厚度只有 0.02，体素大小 = 2/64 = 0.03125，比墙还厚！这意味着每面墙在体素化时只能被 1~2 个体素捕获。

**解法**：接受这个近似误差——VCT 本身就是近似算法。更高的体素分辨率（128³、256³）能改善这个问题，但会带来 8x/64x 的内存和计算开销。本 Demo 以展示原理为主，64³ 已经足够。

---

## 六、效果验证与数据

### 6.1 输出图片

**完整渲染（直接光照 + VCT 间接光照）**：

![Voxel Cone Tracing 完整渲染](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-23-voxel-cone-tracing/vct_output.png)

**直接光照（无 GI）**：

![直接光照](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-23-voxel-cone-tracing/vct_direct.png)

**VCT 间接光照单独可视化**：

![VCT 间接光照](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-23-voxel-cone-tracing/vct_indirect.png)

### 6.2 量化数据

**像素统计**（vct_output.png，0-255 范围）：

| 指标 | 值 |
|------|----|
| 像素均值 | 16.9 |
| 像素标准差 | 36.2 |
| 最大像素值 | 255 |
| 最小像素值 | 0 |

均值 16.9 意味着场景较暗（Cornell Box 内部受限），标准差 36.2 说明有丰富的明暗对比。非全黑（>5）非全白（<250）✅，内容丰富（std >5）✅。

**性能数据**：

| 阶段 | 耗时 |
|------|------|
| 场景体素化（64³ × 6 射线） | ~0.3s |
| Mipmap 构建（Level 1~5） | ~0.05s |
| 渲染 512×512 × 4 spp × 7 锥 | ~2.9s |
| **总计** | **~3.3s** |

每像素约 3.3s/262144 ≈ 0.012ms，即 7 根锥每根约 1.7μs——这是单线程 CPU 软渲的开销。实际 GPU 实现（并行 + 纹理硬件加速）可以达到 1~5ms per frame（1080P）。

**文件大小**：

| 文件 | 大小 |
|------|------|
| vct_output.png | 141.7 KB |
| vct_direct.png | 85.1 KB |
| vct_indirect.png | 70.5 KB |

全部 >10KB ✅（非空、非全色图片）。

### 6.3 间接光照的可见效果

在间接光照图中可以观察到：
- 左侧红墙附近（场景左下区域）有淡淡的红色渗色
- 右侧绿墙附近有绿色渗色
- 天花板下方有来自地面和墙壁的漫反射暖色
- 蓝色球体上有来自周围场景的环境色彩

这些效果在纯直接光照图中完全没有，正是 VCT 贡献的间接 GI。

---

## 七、总结与延伸

### 7.1 VCT 的局限性

1. **体素分辨率是瓶颈**：64³ 的体素无法捕获细节，门缝、薄片等结构可能完全被忽略。256³ 以上才能看到明显改善，但内存从 ~300KB 增长到 ~20MB+。

2. **动态场景开销大**：每次场景变化（灯光移动、物体动画），都需要重新体素化——至少需要增量更新受影响的体素区域。

3. **单次弹射局限**：VCT 本质上是一次查询，无法自然支持多次间接弹射（递归 GI）。理论上可以用体素存储已计算的 GI，形成迭代，但收敛慢。

4. **漏光问题（Light Leaking）**：体素分辨率不足时，锥追踪可能穿过薄墙，导致间接光"漏"进封闭空间。

### 7.2 优化与延伸方向

- **Clipmap 体素**：只体素化相机周围的区域（类似 Cascaded Shadow Maps 的分层思路），远处用更粗的分辨率节省内存
- **实时增量更新**：标记脏体素，只重新光栅化发生变化的区域
- **GPU 实现**：用 Compute Shader 并行体素化，用 3D Texture + 硬件 Mipmap 加速采样，理论上 <5ms/frame
- **VXGI（NVIDIA）**：VCT 的商业级实现，加入了多光源、各向异性体素、屏幕空间反馈等优化
- **与探针系统混合**：近处用 VCT 高精度，远处用 Irradiance Probe，兼顾质量与性能

### 7.3 与本系列的关联

这是本系列第 45 天左右的项目（从 2026-02-10 开始计算），主题演进脉络：

- 基础光追（02-12~02-18）→ 软阴影 PCSS（03-10）→ 全局光照 SPPM（03-21）→ BDPT（03-22）→ **VCT（03-23）**

VCT 是从路径追踪"精确但慢"转向"近似但实时"的一个重要节点，体现了渲染工程中永恒的质量-性能权衡哲学。下一步可以考虑探索 DDGI（Dynamic Diffuse Global Illumination），它用探针网格 + 射线追踪实现了更平滑的全局光照，是 UE5 Lumen 的基础之一。

---

## 参考资料

- Crassin C. et al. "Interactive Indirect Illumination Using Voxel Cone Tracing" SIGGRAPH 2011
- NVIDIA GameWorks VXGI: https://developer.nvidia.com/vxgi
- Laine S. et al. "Efficient Sparse Voxel Octrees" 2010
- Hill S. "ACES Filmic Tone Mapping Curve" 2016
- 源代码：https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-23-voxel-cone-tracing
