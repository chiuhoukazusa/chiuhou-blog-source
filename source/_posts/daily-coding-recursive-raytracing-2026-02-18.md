---
title: 每日编程实践：递归光线追踪 - 镜面反射
date: 2026-02-18 11:35:00
categories:
  - 编程实践
  - 图形学
tags:
  - C++
  - Ray Tracing
  - 光线追踪
  - 镜面反射
  - Phong光照
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/reflection-raytracing-2026-02-18/reflection_output.png
---

# 递归光线追踪 - 镜面反射

**日期**: 2026-02-18  
**开发时间**: 约15分钟  
**迭代次数**: 2次  
**技术**: 递归光线追踪、镜面反射、Phong光照、阴影

## 项目目标

在之前的 Shadow Ray Tracing 基础上，实现**递归光线追踪**算法，支持镜面反射效果。通过不同的材质反射率，实现从完全漫反射到高度镜面的多样化材质表现。

## 核心算法

### 1. 递归光线追踪

```cpp
Vec3 trace(const Ray& ray, const Scene& scene, int depth) {
    if (depth <= 0) return Vec3(0, 0, 0); // 递归深度限制
    
    // ... 计算直接光照（环境光 + 漫反射 + 镜面高光 + 阴影）...
    
    // 递归反射
    if (material.reflectivity > 0.0) {
        Vec3 reflect_dir = ray.direction.reflect(normal);
        Ray reflect_ray(hit_point + normal * 1e-4, reflect_dir);
        Vec3 reflect_color = trace(reflect_ray, scene, depth - 1);
        
        // 混合漫反射和镜面反射颜色
        color = color * (1.0 - reflectivity) + reflect_color * reflectivity;
    }
    
    return color;
}
```

**关键技术点**：
- **递归深度限制**：防止无限递归（本项目最大深度5）
- **反射光线偏移**：`hit_point + normal * 1e-4` 防止浮点误差自相交
- **能量守恒混合**：`(1-r)` 部分漫反射 + `r` 部分反射

### 2. 镜面反射向量

反射公式：**R = V - 2(V·N)N**

```cpp
Vec3 reflect(const Vec3& normal) const {
    return *this - normal * (2.0 * this->dot(normal));
}
```

### 3. 材质系统

```cpp
struct Material {
    Vec3 color;          // 基础颜色
    double diffuse;      // 漫反射系数 [0,1]
    double specular;     // 高光系数 [0,1]
    double reflectivity; // 反射率 [0,1]
};
```

**场景材质配置**：
- 中心银球：`reflectivity = 0.7`, `diffuse = 0.3`（较强镜面 + 适量漫反射，避免过暗）
- 左侧红球：`reflectivity = 0.4`（半透半反）
- 右侧蓝球：`reflectivity = 0.2`（轻微反射）
- 地面：`reflectivity = 0.0`（纯漫反射）
- 顶部金球：`reflectivity = 0.6`（较强反射）

## 开发过程

### 迭代历史

| 版本 | 问题 | 解决方案 |
|------|------|---------|
| v1 | 编译错误：`Vec3 * Vec3` 未定义 | 添加逐分量乘法运算符 |
| v2 | ✅ 编译通过，运行成功 | 无需修改 |
| v3 | 中心银球过暗 | 增加 diffuse (0.1→0.3), 降低 reflectivity (0.8→0.7) |

**迭代次数**: 3次  
**v1错误原因**: 颜色混合需要向量逐分量相乘，但只定义了标量乘法  
**v3问题原因**: 过低的漫反射系数 (0.1) + 过高的反射率 (0.8) → 直接光照太弱 → 球体过暗

**v1修复方法**:
```cpp
Vec3 operator*(const Vec3& v) const { 
    return Vec3(x * v.x, y * v.y, z * v.z); 
}
```

**v3修复方法**: 调整材质参数平衡
```cpp
// 修改前: Material(Vec3(0.9, 0.9, 0.9), 0.1, 0.9, 0.8)
// 修改后: Material(Vec3(0.9, 0.9, 0.9), 0.3, 0.9, 0.7)
// 说明: 增加漫反射让球体更亮，稍降反射率保留更多直接光照
```

## 效果展示

![递归光线追踪 - 镜面反射](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/reflection-raytracing-2026-02-18/reflection_output.png)

**可观察到的效果**：
- ✅ 中心银球清晰反射其他球体
- ✅ 金球和红球在银球表面形成倒影
- ✅ 不同反射率材质对比明显
- ✅ 阴影与反射正确交互

## 技术进步

相比之前的 Shadow Ray Tracing 项目：

| 特性 | 之前 | 现在 |
|------|------|------|
| 递归追踪 | ❌ | ✅ |
| 镜面反射 | ❌ | ✅ |
| 材质系统 | 单一 | 多样化反射率 |
| 颜色混合 | 基础 | 向量逐分量运算 |

**完整光照模型**：
```
✅ 环境光 (Ambient)
✅ 漫反射 (Diffuse)
✅ 镜面高光 (Specular)
✅ 阴影 (Shadow Ray)
✅ 反射 (Reflection) ← 今日新增
⏳ 折射 (Refraction) ← 下一步
```

## 学习收获

1. **递归算法的应用** - 光线反射是经典的递归问题，理解了递归深度控制的重要性
2. **能量守恒原理** - 反射率决定了直接光照与反射光照的权重分配
3. **数值稳定性技巧** - 偏移量 `1e-4` 防止自相交，这是图形学中常见的数值技巧
4. **性能权衡** - 递归深度对渲染质量和速度有直接影响

## 下一步计划

- [ ] **折射效果** - 实现透明材质（玻璃球）
- [ ] **抗锯齿** - 多重采样提升图像质量
- [ ] **软阴影** - 面光源代替点光源
- [ ] **BVH加速** - 优化光线求交性能

## 代码仓库

GitHub: [daily-coding-practice/2026/02/18-recursive-raytracing-reflection](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/18-recursive-raytracing-reflection)

---

**项目状态**: ✅ 完成  
**输出**: 800×600 PNG图像  
**渲染时间**: 约2秒  
**代码行数**: 324行 C++
