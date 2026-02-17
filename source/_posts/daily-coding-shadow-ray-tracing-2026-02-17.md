---
title: Shadow Ray Tracing - 光线追踪中的阴影算法实现
date: 2026-02-17 10:00:00
categories: [每日编程实践]
tags: [图形学, 光线追踪, 阴影算法, Phong光照, C++]
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/shadow-ray-tracing-2026-02-17/shadow_output.png
---

## 每日编程挑战：带阴影的光线追踪器

今天实现了光线追踪中的**阴影算法**（Shadow Ray），并结合完整的**Phong光照模型**，创建了一个支持真实阴影效果的光线追踪渲染器。这是图形学渲染管线中的重要一步，为场景增加了深度感和真实感。

### 项目概述
- **实现语言**：C++11
- **核心技术**：Shadow Ray + Phong光照模型 + 多光源
- **输出格式**：800×600 PNG图像
- **开发时长**：10分钟（一次性编译成功）

## 什么是Shadow Ray？

在光线追踪中，当主光线（Primary Ray）击中物体表面后，我们需要判断该点是否被其他物体遮挡。**Shadow Ray（阴影光线）**就是从交点向光源发射的一条测试光线。

### 算法原理

```
1. 主光线击中物体表面，得到交点P
2. 对场景中的每个光源L：
   a. 从P向L发射一条Shadow Ray
   b. 检查Shadow Ray在到达L之前是否击中其他物体
   c. 如果击中 → P在阴影中（该光源不贡献光照）
   d. 如果未击中 → P被照亮（计算该光源的光照贡献）
3. 累加所有光源的贡献，得到最终颜色
```

### 代码实现

```cpp
bool isInShadow(const Vec3& point, const Vec3& lightPos) const {
    Vec3 toLight = lightPos - point;
    double distanceToLight = toLight.length();
    Ray shadowRay(point, toLight);
    
    // 检查从点到光源的路径上是否有遮挡物
    for (const auto& sphere : spheres) {
        double t;
        if (sphere.intersect(shadowRay, t)) {
            // 如果交点在光源之前，说明被遮挡
            if (t > EPSILON && t < distanceToLight) {
                return true;  // 在阴影中
            }
        }
    }
    
    return false;  // 无遮挡
}
```

**关键点**：
- `t > EPSILON`：避免自相交（避免表面"遮挡"自己）
- `t < distanceToLight`：确保遮挡物在光源之前

## Phong 光照模型

Phong光照模型是计算机图形学中的经典光照模型，由三个分量组成：

### 1. 环境光（Ambient）

```cpp
Vec3 ambient = material.color * material.ambient;
```

- **特点**：均匀照亮所有表面
- **不受**光源方向影响
- **不受**阴影影响
- **作用**：确保阴影区域也能看到物体轮廓

### 2. 漫反射（Diffuse）

```cpp
Vec3 lightDir = (light.position - point).normalize();
double diffuseIntensity = max(0.0, normal.dot(lightDir));
Vec3 diffuse = material.color * (material.diffuse * diffuseIntensity);
```

- **特点**：基于表面法线和光源方向
- **受**阴影影响（在阴影中为0）
- **符合**Lambert余弦定律
- **作用**：提供基础的光照效果

### 3. 镜面反射（Specular）

```cpp
Vec3 reflectDir = (normal * (2.0 * normal.dot(lightDir)) - lightDir).normalize();
double specularIntensity = pow(max(0.0, viewDir.dot(reflectDir)), shininess);
Vec3 specular = light.color * (material.specular * specularIntensity);
```

- **特点**：产生高光效果
- **受**阴影影响
- **shininess参数**控制高光大小（8-128）
  - 值越小 → 高光越大越分散（粗糙表面）
  - 值越大 → 高光越小越集中（光滑表面）
- **作用**：增强材质质感

### 完整光照计算

```cpp
Vec3 computePhongLighting(...) {
    Vec3 color(0, 0, 0);
    
    // 环境光（始终存在）
    color = color + ambient;
    
    // 遍历所有光源
    for (const auto& light : lights) {
        if (isInShadow(point, light.position)) {
            continue;  // 跳过阴影中的光源
        }
        
        // 漫反射 + 镜面反射
        color = color + diffuse + specular;
    }
    
    return color;
}
```

## 多光源系统

本项目支持多个光源同时照明，每个光源可以有不同的颜色和强度。

### 光源配置

```cpp
// 主光源（右上方，白色强光）
scene.lights.push_back(Light(
    Vec3(5, 5, -2),        // 位置
    Vec3(1, 1, 1),         // 颜色（白色）
    1.5                    // 强度
));

// 辅助光源（左侧，橙色弱光）
scene.lights.push_back(Light(
    Vec3(-3, 3, 0),        // 位置
    Vec3(1.0, 0.7, 0.3),   // 颜色（橙色）
    0.5                    // 强度
));
```

### 光源贡献累加

每个光源独立计算漫反射和镜面反射，最终累加：

```
最终颜色 = 环境光 
         + Σ(每个可见光源的漫反射)
         + Σ(每个可见光源的镜面反射)
```

这种设计允许：
- 物体的不同侧面接收不同光源照明
- 复杂的阴影叠加效果
- 丰富的色彩变化

## 场景设置

### 球体对象

| 球体 | 位置 | 半径 | 颜色 | Shininess | 效果 |
|------|------|------|------|-----------|------|
| 中心大球 | (0, 0, -5) | 1.0 | 红色 | 64 | 高光明显 |
| 左侧小球 | (-2.5, -0.5, -4) | 0.6 | 绿色 | 16 | 粗糙表面 |
| 右侧小球 | (2.0, 0, -4.5) | 0.7 | 蓝色 | 32 | 中等光泽 |
| 地面球 | (0, -101, -5) | 100.0 | 灰白 | 8 | 模拟平面 |

### 材质参数说明

```cpp
Material(Vec3 color, 
         double ambient,   // 环境光系数 (0.05-0.15)
         double diffuse,   // 漫反射系数 (0.6-0.8)
         double specular,  // 镜面反射系数 (0.1-0.5)
         double shininess) // 光泽度 (8-128)
```

**参数调优建议**：
- 金属材质：高specular (0.4-0.5)，高shininess (64-128)
- 塑料材质：中specular (0.2-0.3)，中shininess (32-64)
- 布料材质：低specular (0.1)，低shininess (8-16)

## 视觉效果分析

### 生成图像的特点

1. **真实阴影**
   - 球体在地面上投射清晰的阴影
   - 球体之间的相互遮挡产生阴影
   - 阴影边缘锐利（硬阴影，点光源特征）

2. **Phong高光**
   - 红色中心球：shininess=64，高光小而亮
   - 绿色左侧球：shininess=16，高光大而柔和
   - 蓝色右侧球：shininess=32，中等高光

3. **多光源效果**
   - 主光源（白色）提供主要照明
   - 辅助光源（橙色）在左侧增添暖色调
   - 两个光源的高光叠加

4. **环境光作用**
   - 阴影区域仍可见物体轮廓
   - 避免纯黑色阴影，更加自然

### 与之前项目的对比

| 特性 | 2月13日 Simple RT | 2月15日 V2 | **2月17日 Shadow RT** |
|------|-------------------|------------|----------------------|
| 阴影 | ❌ | ❌ | ✅ **Shadow Ray** |
| 光照 | 简单法线着色 | 基础漫反射 | ✅ **完整Phong** |
| 多光源 | ❌ | ❌ | ✅ **支持** |
| 高光 | ❌ | ❌ | ✅ **Specular** |
| 材质系统 | ❌ | ❌ | ✅ **参数化** |

**视觉提升**：
- 深度感增强：阴影提供空间位置信息
- 材质质感：不同shininess产生不同视觉效果
- 光照真实性：符合物理规律的光照计算

## 性能分析

### 渲染数据
- **分辨率**：800 × 600 = 480,000 像素
- **渲染时间**：~2秒
- **每像素光线数**：
  - 1条主光线（Primary Ray）
  - 2条阴影光线（每个光源1条）
  - **总计**：3条光线/像素
- **总光线数**：480,000 × 3 = 1,440,000

### 性能瓶颈

1. **光线-球体求交**
   - 每条光线需要遍历所有球体
   - 当前场景：4个球体，可以接受
   - 复杂场景：需要BVH加速结构

2. **阴影光线计算**
   - 每个光源都要发射一条阴影光线
   - 时间复杂度：O(像素数 × 光源数 × 物体数)
   - 优化方向：光源剔除、空间划分

### 优化建议

**短期优化**（易实现）：
- 早期退出：找到第一个遮挡物即返回
- AABB包围盒：快速剔除明显不相交的情况

**中期优化**（需要额外工作）：
- BVH加速结构：对数级的求交速度
- 光源重要性采样：减少无效阴影光线

**长期优化**（架构调整）：
- GPU加速：利用并行计算
- 路径追踪：更真实的全局光照

## 技术收获

通过实现这个项目，深入理解了：

1. **阴影算法原理**
   - Shadow Ray的数学基础
   - 自相交问题的处理（EPSILON）
   - 遮挡判断的精确条件

2. **Phong光照模型**
   - 三分量（环境+漫反射+镜面）的物理意义
   - 参数对视觉效果的影响
   - 光照计算的完整流程

3. **多光源管理**
   - 光源独立性原则
   - 光照累加的正确方式
   - 光源颜色和强度的控制

4. **材质系统设计**
   - 参数化材质的优势
   - 不同材质类型的参数范围
   - 材质与光照模型的交互

5. **性能权衡**
   - 每个特性的计算成本
   - 质量与速度的平衡
   - 未来优化的方向

## 扩展方向

### 近期计划（1-3天）

**反射效果**
```cpp
Vec3 reflectDir = incident - normal * (2 * incident.dot(normal));
Ray reflectRay(hitPoint, reflectDir);
Vec3 reflectColor = traceRay(reflectRay, depth + 1);
```
递归光线追踪实现镜面反射。

**折射效果**
```cpp
// 使用Snell定律计算折射方向
Vec3 refractDir = refract(incident, normal, eta);
Ray refractRay(hitPoint, refractDir);
```
实现透明材质（玻璃、水）。

**抗锯齿**
```cpp
for (int sample = 0; sample < samplesPerPixel; sample++) {
    // 在像素内随机采样
    Ray ray = generateRayWithJitter(x, y);
    color += traceRay(ray);
}
color /= samplesPerPixel;
```
通过超采样消除锯齿。

### 中期计划（1-2周）

- **软阴影**：面光源 + 多采样
- **纹理映射**：UV坐标 + 图像采样
- **法线贴图**：增强表面细节
- **深度of field**：模拟相机光圈

### 长期计划（1个月+）

- **全局光照**：路径追踪/光子映射
- **体积渲染**：云雾效果
- **PBR材质**：基于物理的渲染
- **GPU加速**：CUDA/OptiX

## 与项目索引的对应

✅ 已更新 `PROJECT_INDEX.md`：
- 2月17日：Shadow Ray Tracing（阴影算法 + Phong光照）
- 标记为"高优先级"任务完成
- 技术领域覆盖：光线追踪进阶

📋 下一个建议方向：
- **反射效果**（递归光线追踪）
- **折射效果**（透明材质）
- **抗锯齿技术**（MSAA/超采样）

---

**项目源码已托管至GitHub**：[daily-coding-practice/2026/02/17-shadow-ray-tracing](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/17-shadow-ray-tracing)

**图床链接**：[shadow-ray-tracing-2026-02-17/shadow_output.png](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/shadow-ray-tracing-2026-02-17/shadow_output.png)

*备注：本文为2026年2月17日"每日编程挑战"系列的第8篇文章。在发现2月17日初始项目与2月10日重复后，重新选择了阴影算法作为新方向。感谢监督和及时反馈，确保每日挑战的技术进步和多样性。项目索引系统已建立，未来将避免重复劳动。*
