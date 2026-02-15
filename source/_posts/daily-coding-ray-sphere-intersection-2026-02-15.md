---
title: 光线追踪：球体相交检测实现 - 图形学每日挑战
date: 2026-02-15 10:00:00
categories: [图形学, C++编程, 每日挑战]
tags: [图形学, 算法, 光线追踪, 球体, 光线求交, 计算机图形学]
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026-02-15-ray-sphere/ray_sphere_intersection.png
---

## 每日编程挑战：光线与球体相交检测可视化

今天完成了光线追踪中的基础计算——**光线与球体相交检测**。这是光线追踪算法中最核心的几何计算之一，通过检测光线与球体的交点，为后续的材质、光照和阴影计算奠定基础。

### 项目概述
- **实现语言**：C++ (C++11标准)
- **算法核心**：光线-球体相交的几何计算
- **输出格式**：400×300像素PNG图像
- **场景内容**：单个球体，多方向光线可视化

### 核心算法：光线-球体相交

光线与球体的交点计算是光线追踪的基础问题。使用几何方法可以高效地计算交点。

#### 数学公式推导

设：
- 光线：$R(t) = O + t\vec{d}$（$O$为原点，$\vec{d}$为方向，$t$为参数）
- 球体：中心$C$，半径$r$，满足 $||P - C||^2 = r^2$

将光线方程代入球体方程：
$$ ||O + t\vec{d} - C||^2 = r^2 $$

展开得：
$$ (O - C + t\vec{d}) \cdot (O - C + t\vec{d}) = r^2 $$

令 $OC = O - C$，则：
$$ (OC + t\vec{d}) \cdot (OC + t\vec{d}) = r^2 $$
$$ OC \cdot OC + 2t(OC \cdot \vec{d}) + t^2(\vec{d} \cdot \vec{d}) = r^2 $$

整理得到一元二次方程：
$$ at^2 + bt + c = 0 $$

其中：
- $a = \vec{d} \cdot \vec{d}$
- $b = 2(OC \cdot \vec{d})$
- $c = OC \cdot OC - r^2$

#### 交点判定

通过判别式 $Δ = b^2 - 4ac$ 判断相交情况：
- $Δ < 0$：无交点
- $Δ = 0$：相切（一个交点）
- $Δ > 0$：相交（两个交点）

### 代码实现

#### 1. 向量类实现
```cpp
struct Vec3 {
    float x, y, z;
    
    Vec3 operator+(const Vec3& v) const { return Vec3(x + v.x, y + v.y, z + v.z); }
    Vec3 operator-(const Vec3& v) const { return Vec3(x - v.x, y - v.y, z - v.z); }
    Vec3 operator*(float s) const { return Vec3(x * s, y * s, z * s); }
    Vec3 operator/(float s) const { return Vec3(x / s, y / s, z / s); }
    
    float dot(const Vec3& v) const { return x * v.x + y * v.y + z * v.z; }
    float length() const { return sqrt(x * x + y * y + z * z); }
    Vec3 normalize() const { return (*this) / length(); }
};
```

#### 2. 相交检测函数
```cpp
bool raySphereIntersect(const Ray& ray, const Sphere& sphere, float& t) {
    Vec3 oc = ray.origin - sphere.center;
    float a = ray.direction.dot(ray.direction);
    float b = 2.0f * oc.dot(ray.direction);
    float c = oc.dot(oc) - sphere.radius * sphere.radius;
    
    float discriminant = b * b - 4 * a * c;
    
    if (discriminant < 0) {
        return false; // 没有交点
    }
    
    float sqrtDisc = sqrt(discriminant);
    float t1 = (-b - sqrtDisc) / (2.0f * a);
    float t2 = (-b + sqrtDisc) / (2.0f * a);
    
    // 选择最近的正值交点
    if (t1 > 0 && t2 > 0) {
        t = (t1 < t2) ? t1 : t2;
        return true;
    } else if (t1 > 0) {
        t = t1;
        return true;
    } else if (t2 > 0) {
        t = t2;
        return true;
    }
    
    return false;
}
```

#### 3. 光线生成与追踪
```cpp
// 球体定义
Sphere sphere(Vec3(0, 0, 0), 1.0f);

// 多方向光线测试
for (int i = 0; i < numRays; ++i) {
    float phi = (i * 2.0f * M_PI) / numRays;
    Ray ray(Vec3(2.0f, 0, 0), 
            Vec3(cos(phi), sin(phi), 0).normalize());
    
    float t;
    if (raySphereIntersect(ray, sphere, t)) {
        // 计算交点位置
        Vec3 hitPoint = ray.origin + ray.direction * t;
        // 计算表面法线
        Vec3 normal = (hitPoint - sphere.center).normalize();
        // 简单着色
        float intensity = fabs(normal.dot(ray.direction));
        Vec3 color = Vec3(0.5, 0.7, 1.0) * intensity;
        
        // 记录结果
        colors.push_back(color);
    } else {
        // 没有相交，使用背景色
        colors.push_back(Vec3(0.1, 0.1, 0.1));
    }
}
```

### 可视化效果

输出图像展示了多个方向的光线与球体的相交情况：

1. **相交区域**：蓝色区域表示光线与球体相交
2. **未相交区域**：深灰色表示光线未命中球体
3. **强度渐变**：根据交点的法线方向产生亮度变化

图像中可以看到：
- **中心区域**：光线与球体正面相交，颜色较亮
- **边缘区域**：光线与球体侧面相交，颜色较暗
- **外围区域**：光线完全错过球体，颜色最深

### 数学细节探讨

#### 交点选择策略

当存在两个交点时（$t_1$和$t_2$），选择策略很重要：

1. **最近交点**：选择较小的$t$值（$t_1$）
2. **外部进入**：$t > 0$表示光线从外部进入球体
3. **内部出发**：$t < 0$表示光线从球体内部出发

#### 数值稳定性优化

实际实现中需要考虑数值稳定性：

```cpp
// 避免除零错误
if (fabs(a) < EPSILON) {
    // 处理特殊情况
}

// 使用双精度计算避免精度丢失
double a_d = static_cast<double>(a);
double b_d = static_cast<double>(b);
double c_d = static_cast<double>(c);
```

#### 快速拒绝测试

对于复杂的场景，可以进行快速拒绝测试：

1. **包围盒测试**：先检测光线与球体包围盒的相交
2. **距离阈值**：预先排除过远的物体
3. **方向过滤**：排除方向错误的光线

### 性能分析

- **时间复杂度**：O(1) 每光线-球体对
- **空间复杂度**：O(1) 基本计算
- **浮点运算**：约20次乘加运算
- **分支预测**：1个主要分支（判别式判断）

优化后的近似计算：
```cpp
// 优化形式1：避免重复计算
Vec3 oc = ray.origin - sphere.center;
float b = oc.dot(ray.direction);
float c = oc.dot(oc) - sphere.radius * sphere.radius;
float discriminant = b * b - c;  // 当a=1时

// 优化形式2：预计算信息
float radiusSq = sphere.radius * sphere.radius;  // 预计算平方
```

### 实际应用场景

光线-球体相交计算在实际中有广泛应用：

1. **游戏引擎**：碰撞检测、拾取操作
2. **物理模拟**：粒子系统、刚体碰撞
3. **医学成像**：CT/MRI数据球体标记
4. **机器人学**：传感器距离计算

#### 扩展应用：多球体场景

```cpp
std::vector<Sphere> spheres = {
    Sphere(Vec3(-1.5, 0, 0), 0.5),
    Sphere(Vec3(0, 0, 0), 1.0),
    Sphere(Vec3(1.5, 0, 0), 0.7)
};

for (const auto& sphere : spheres) {
    float t;
    if (raySphereIntersect(ray, sphere, t)) {
        hits.emplace_back(t, &sphere);
    }
}

// 选择最近的交点
std::sort(hits.begin(), hits.end());
if (!hits.empty()) {
    // 处理最近交点
}
```

### 学习收获

通过实现光线-球体相交算法，深入理解了：

1. **几何推导**：从公式到代码的转换过程
2. **数值计算**：浮点运算的精度和稳定性
3. **算法优化**：计算简化与性能提升
4. **图形学基础**：光线追踪的核心构建块

### 下一步发展方向

基于此基础可以继续扩展：

1. **添加材质**：支持镜面反射、折射效果
2. **实现抗锯齿**：提升图像质量
3. **构建场景树**：支持复杂物体加速结构
4. **并行计算**：利用GPU加速光线追踪

---

**项目源码已托管至GitHub**：[daily-coding-practice/2026/02/15-ray-sphere-intersection](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/15-ray-sphere-intersection)

**图床链接**：[2026-02-15-ray-sphere/ray_sphere_intersection.png](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026-02-15-ray-sphere/ray_sphere_intersection.png)

*备注：本文为2026年2月15日"每日编程挑战"系列的第6篇文章。系列旨在通过每日小项目巩固计算机图形学基础知识。今天的项目基于实际的每日编程实践项目实现，展示了光线追踪的基础计算原理。*