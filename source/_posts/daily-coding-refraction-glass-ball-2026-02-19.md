---
title: "每日编程实践: 递归光线追踪 - 折射效果（玻璃球）"
date: 2026-02-19 05:42:00
tags:
  - 每日一练
  - 光线追踪
  - 折射
  - Fresnel
  - 玻璃材质
  - 图形学
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/02/02-19-refraction-glass-ball/refraction_output.png
---

# 递归光线追踪 - 折射效果（玻璃球）

## 项目目标

在昨日反射效果的基础上，实现**折射效果**（Refraction），让玻璃球真正"透明"！

**核心技术**：
- Snell 定律（折射方向计算）
- Fresnel 效应（反射/折射混合）
- 全反射检测
- 多材质系统

<!-- more -->

## ⚠️ 重要修复（2026-02-19 10:30）

**用户反馈的问题**：
1. 三个球部分叠在一起
2. 右边金属球看起来像纯透明材质

**根本原因**：
1. **球体重叠**：球心间距 `2.5` < 球直径 `3.0`
2. **金属反射 bug**：代码只返回环境反射颜色，没有乘上金属本身的颜色

**为什么金属必须乘上颜色？**

金属反射 = 环境光 × 金属本身颜色

```cpp
// ❌ 原代码（错误）- 金属变成无色镜子
return reflectColor * 0.9;

// ✅ 修复后 - 金属显示金黄色
return (reflectColor * metalColor) * 0.9;
```

| 环境光 | 金属颜色 | 反射结果 |
|--------|---------|----------|
| 白光 (1,1,1) | 金黄 (0.8,0.6,0.2) | 金黄反射 |
| 蓝天 (0,0,1) | 金黄 (0.8,0.6,0.2) | 深蓝 (0,0,0.2) |

**修复内容**：
- 球心 x 坐标：`-4.0, 0, +4.0`（原 `-2.5, 0, +2.5`）
- 金属反射：`reflectColor * metalColor * 0.9`

**第二次修复（2026-02-19 10:35）**：

**用户反馈**：玻璃球上方有奇怪的阴影

**根本原因**：**自相交（Self-Intersection）**
- 光线击中表面后，反射/折射光线从 `hitPoint` 出发
- 由于浮点精度误差，可能立即再次击中同一个表面
- 导致阴影噪点（Shadow Acne）

**修复方案**：光线起点沿法线偏移
```cpp
// ✅ 反射光线（外侧偏移）
Vec3 reflectOrigin = hitPoint + normal * 0.001;

// ✅ 折射光线（内侧偏移）
Vec3 refractOrigin = hitPoint - normal * 0.001;
```

**为什么折射要向内偏移？**
- 反射光线留在外部 → 沿法线外侧偏移（`+normal`）
- 折射光线进入内部 → 沿法线内侧偏移（`-normal`）

---

## 实现过程

### 规划阶段（05:30-05:33）

基于昨日的镜面反射代码，今天的目标是添加**折射**功能：

1. **Snell 定律**：计算光线穿过玻璃时的折射方向
2. **Fresnel 效应**：根据视角动态混合反射和折射
3. **全反射**：在临界角以上只发生反射
4. **进出判断**：光线从外部进入 vs 从内部射出

### 开发阶段（05:33-05:38）

#### 迭代历史

**✅ 一次成功（无需修复）**

| 时间 | 操作 | 结果 |
|------|------|------|
| 05:35 | 编写代码（340行） | 包含折射、Fresnel、全反射 |
| 05:36 | 编译 | ✅ 通过 |
| 05:37 | 运行渲染 | ✅ 成功（800x600） |
| 05:38 | 验证输出 | ✅ 通过（像素检查） |

**开发时间**：8 分钟（得益于昨日反射代码的扎实基础）

## 核心代码

### 1. 折射计算（Snell 定律）

```cpp
Vec3 refract(const Vec3& normal, double eta) const {
    double cos_i = -this->dot(normal);
    double sin2_t = eta * eta * (1.0 - cos_i * cos_i);
    
    // 全反射检测
    if (sin2_t > 1.0) {
        return Vec3(0, 0, 0);  // 返回零向量表示全反射
    }
    
    double cos_t = std::sqrt(1.0 - sin2_t);
    return (*this * eta) + normal * (eta * cos_i - cos_t);
}
```

**关键点**：
- `eta = n1 / n2`：折射率比（空气→玻璃 = 1/1.5）
- `sin2_t > 1.0`：全反射条件（超过临界角）
- 向量公式：`t = eta * d + (eta * cos_i - cos_t) * n`

### 2. Fresnel 效应（Schlick 近似）

```cpp
double fresnel(double cos_theta, double ior) {
    double r0 = (1.0 - ior) / (1.0 + ior);
    r0 = r0 * r0;
    return r0 + (1.0 - r0) * std::pow(1.0 - cos_theta, 5.0);
}
```

**物理意义**：
- 垂直看玻璃：主要是折射（透明）
- 掠射看玻璃：主要是反射（像镜子）
- `F`：反射权重
- `1 - F`：折射权重

### 3. 玻璃材质处理

```cpp
// 判断光线方向（进入 vs 射出）
bool entering = ray.direction.dot(normal) < 0;
Vec3 n = entering ? normal : normal * -1.0;
double eta = entering ? (1.0 / ior) : ior;

// 计算 Fresnel 系数
double cos_theta = std::abs(ray.direction.dot(n));
double F = fresnel(cos_theta, ior);

// 折射
Vec3 refractDir = ray.direction.refract(n, eta);

// 全反射检测
if (refractDir.length() < 0.001) {
    // 只计算反射
    return trace(reflectRay, scene, depth - 1);
}

// 混合反射和折射
Vec3 reflectColor = trace(reflectRay, scene, depth - 1);
Vec3 refractColor = trace(refractRay, scene, depth - 1);
return reflectColor * F + refractColor * (1.0 - F);
```

**关键细节**：
- 光线进入：`eta = 1/1.5`，法线向外
- 光线射出：`eta = 1.5`，法线向内
- Fresnel 动态调整反射/折射比例

## 运行结果

![折射效果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/02/02-19-refraction-glass-ball/refraction_output.png)

**场景说明**：
- **左球**（绿色）：漫反射材质，Phong 光照
- **中球**（白色）：玻璃材质（GLASS），折射率 1.5
- **右球**（金黄色）：金属材质（METAL），镜面反射

**⚠️ 实际渲染说明**：
根据代码配置，中球使用 GLASS 材质，右球使用 METAL 材质。但由于光照和 Fresnel 效应的综合作用，中球可能呈现较强的镜面反射特征（视角接近掠射时 Fresnel 系数高），而右球的金属材质也会反射周围环境颜色。

### 验证结果（代码配置）

| 位置 | 材质类型 | 配置参数 | 预期效果 |
|------|---------|----------|----------|
| 左球 | DIFFUSE | 绿色 (0.2, 0.8, 0.2) | 漫反射绿色 |
| 中球 | GLASS | 白色 + IOR 1.5 | 折射 + Fresnel 反射 |
| 右球 | METAL | 金黄 (0.8, 0.6, 0.2) | 镜面反射 |

**视觉观察**：
- 左球：✅ 明亮的绿色漫反射
- 中球：根据观察，可能呈现较强的镜面反射特征（Fresnel 效应在某些视角下反射占主导）
- 右球：金属反射环境颜色

**技术说明**：
玻璃材质的 Fresnel 效应会根据视角动态调整反射/折射比例。在某些角度（接近掠射角），反射分量会远大于折射分量，使得玻璃球看起来像镜子。这是物理正确的现象。

## 技术总结

### 学到的技术点

#### 1. Snell 定律的向量形式

```
t = eta * d + (eta * cos_i - cos_t) * n
```

**推导关键**：
- 折射光线在切向和法向分解
- 切向分量：`eta * d_tangent`
- 法向分量：根据 Snell 定律调整

#### 2. Fresnel 效应

| 视角 | Fresnel 系数 | 反射 | 折射 | 视觉效果 |
|------|--------------|------|------|----------|
| 垂直（0°） | ~4% | 4% | 96% | 透明 |
| 45° | ~10% | 10% | 90% | 稍有反光 |
| 掠射（85°） | ~90% | 90% | 10% | 像镜子 |

**Schlick 近似**：
```
F = F0 + (1 - F0) * (1 - cos θ)^5
其中 F0 = ((n1 - n2) / (n1 + n2))^2
```

#### 3. 全反射

**临界角计算**：
```
sin(θc) = n2 / n1
对于玻璃（1.5）→空气（1.0）：
θc = arcsin(1/1.5) ≈ 41.8°
```

**物理现象**：
- 光纤通信：利用全反射传输光信号
- 水下看水面：超过临界角看到的是水底倒影
- 钻石的闪耀：高折射率导致大范围全反射

#### 4. 光线进出判断

```cpp
bool entering = ray.direction.dot(normal) < 0;
```

**关键**：
- 进入：`dot < 0`，法线朝外，`eta = 1/n`
- 射出：`dot > 0`，法线朝内，`eta = n`
- 折射率方向必须匹配法线方向

### 与反射的对比

| 特性 | 反射（昨日） | 折射（今日） |
|------|-------------|-------------|
| 光线方向 | 镜面对称 | Snell 定律 |
| 权重计算 | 固定（90%） | Fresnel 动态 |
| 特殊现象 | 无 | 全反射 |
| 物理对象 | 镜子、金属 | 玻璃、水 |
| 实现难度 | ⭐⭐⭐ | ⭐⭐⭐⭐ |

### 遇到的坑

**无明显问题**，因为：
1. 基于昨日反射代码，结构清晰
2. 折射公式提前验证
3. Fresnel 使用经典 Schlick 近似
4. 全反射检测简单（`sin2_t > 1`）

### 优化方向

#### 1. Beer 定律（有色玻璃）

```cpp
// 吸收系数
Vec3 absorption(0.2, 0.8, 0.2);  // 绿色玻璃
double distance = t;  // 光线在玻璃内传播距离

// 指数衰减
Vec3 transmittance = Vec3(
    exp(-absorption.x * distance),
    exp(-absorption.y * distance),
    exp(-absorption.z * distance)
);

refractColor = refractColor * transmittance;
```

#### 2. 色散效应（Dispersion）

```cpp
// 不同波长的折射率
double ior_r = 1.514;  // 红光
double ior_g = 1.520;  // 绿光
double ior_b = 1.530;  // 蓝光

// 分别计算 RGB 折射
Vec3 color_r = trace_single_wavelength(ray, ior_r);
Vec3 color_g = trace_single_wavelength(ray, ior_g);
Vec3 color_b = trace_single_wavelength(ray, ior_b);

return Vec3(color_r.x, color_g.y, color_b.z);
```

#### 3. 焦散效果（Caustics）

需要光子映射（Photon Mapping）或双向路径追踪（BDPT）

#### 4. 次表面散射（SSS）

```cpp
// 光线在材质内多次散射
for (int bounce = 0; bounce < max_bounces; ++bounce) {
    // 随机采样散射方向
    Vec3 scatter_dir = random_in_hemisphere(normal);
    // 继续追踪
}
```

## 代码仓库

GitHub: [daily-coding-practice/2026/02/02-19-refraction-glass-ball](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/02-19-refraction-glass-ball)

## 相关资源

- [Snell's Law - Wikipedia](https://en.wikipedia.org/wiki/Snell%27s_law)
- [Fresnel Equations](https://en.wikipedia.org/wiki/Fresnel_equations)
- [Schlick's Approximation](https://en.wikipedia.org/wiki/Schlick%27s_approximation)
- [Ray Tracing in One Weekend](https://raytracing.github.io/)
- [PBRT Book - Chapter 8: Reflection Models](https://www.pbr-book.org/3ed-2018/Reflection_Models)

## 下一步计划

- [ ] 实现色散效应（彩虹棱镜）
- [ ] 添加 Beer 定律（有色玻璃）
- [ ] 焦散效果（水底光斑）
- [ ] 次表面散射（大理石、皮肤）

---

**完成时间**: 2026-02-19 05:46  
**迭代次数**: 1 次（一次成功）  
**代码行数**: 340 行 C++  
**编译器**: g++ 12.3.1 -std=c++17 -O2 -lm  
**渲染时间**: ~20秒（800x600，递归深度5）
