---
title: "每日编程实践: Parallax Mapping（视差贴图）"
date: 2026-02-24 05:45:00
tags:
  - 每日一练
  - 图形学
  - 光线追踪
  - 纹理映射
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/02/02-24-Parallax-Mapping/parallax_output.png
---

# Parallax Mapping（视差贴图）

## 项目目标

实现**Parallax Mapping（视差贴图）**技术，这是在昨天Normal Mapping基础上的进阶效果。通过高度图偏移纹理坐标，模拟表面凹凸的深度效果，创造更真实的3D表面。

渲染**两个球体对比图**：
- **左球**：普通纹理映射（平面砖块）
- **右球**：视差贴图（立体砖块，带深度感）

直观展示视差贴图带来的立体效果差异。

## 实现过程

### 1. 视差贴图原理

视差贴图的核心思想是：**根据视角和表面高度，偏移纹理坐标**。

当从倾斜角度观察表面时：
- 高的部分（砖块）会"遮挡"后面的纹理
- 低的部分（灰浆）会"显露"更多纹理

通过计算视线方向和表面高度，我们可以预测这种"视差"效应：

```cpp
// 将视线方向转换到切线空间
Vec3 view_tangent = Vec3(view_dir.dot(T), view_dir.dot(B), view_dir.dot(N));

// 根据高度和视角偏移UV坐标
double parallax_scale = 0.05;  // 视差强度
double offset_x = view_tangent.x / view_tangent.z * height * parallax_scale;
double offset_y = view_tangent.y / view_tangent.z * height * parallax_scale;

// 应用偏移
u -= offset_x;
v -= offset_y;
```

**关键参数**：
- `height`：表面高度（0.0-1.0，从高度图采样）
- `view_tangent.z`：视线与表面的夹角（越小越倾斜）
- `parallax_scale`：控制效果强度

### 2. 程序化砖块纹理

生成带高度信息的砖块纹理：

```cpp
Vec3 brickTexture(double u, double v, double& height) {
    const double brick_width = 0.3;
    const double brick_height = 0.15;
    const double mortar_width = 0.02;
    
    // 交错排列（奇数行偏移半个砖块）
    double row = std::floor(v / brick_height);
    double offset = (int(row) % 2) * brick_width * 0.5;
    
    double x = std::fmod(u + offset, brick_width);
    double y = std::fmod(v, brick_height);
    
    // 判断是砖块还是灰浆
    bool is_mortar = (x < mortar_width || x > brick_width - mortar_width ||
                      y < mortar_width || y > brick_height - mortar_width);
    
    if (is_mortar) {
        height = 0.0;  // 灰浆深度为0（凹陷）
        return Vec3(0.5, 0.5, 0.5);  // 灰色
    } else {
        height = 0.1;  // 砖块凸起
        return Vec3(0.7, 0.3, 0.2);  // 红褐色砖块
    }
}
```

### 3. TBN切线空间

构建切线空间坐标系，用于将视线方向转换到表面局部坐标：

```cpp
void getTBN(const Vec3& point, Vec3& T, Vec3& B, Vec3& N) const {
    N = getNormal(point);  // 表面法线（Normal）
    
    // 计算切线（Tangent）
    Vec3 up = std::abs(N.y) < 0.999 ? Vec3(0, 1, 0) : Vec3(1, 0, 0);
    T = up.cross(N).normalize();
    
    // 计算副切线（Bitangent）
    B = N.cross(T).normalize();
}
```

**TBN矩阵**将世界空间向量转换到切线空间：
- **T (Tangent)**：沿着纹理U方向
- **B (Bitangent)**：沿着纹理V方向
- **N (Normal)**：垂直于表面

## 核心代码

```cpp
Vec3 parallax_mapping(const Vec3& point, const Sphere& sphere, 
                      const Vec3& view_dir, const Vec3& light_dir, 
                      bool use_parallax) {
    // 获取UV坐标
    double u, v;
    sphere.getUV(point, u, v);
    
    // 获取TBN矩阵
    Vec3 T, B, N;
    sphere.getTBN(point, T, B, N);
    
    // 如果使用视差贴图，偏移UV坐标
    if (use_parallax) {
        Vec3 view_tangent = Vec3(view_dir.dot(T), view_dir.dot(B), view_dir.dot(N));
        
        double height;
        brickTexture(u, v, height);
        
        double parallax_scale = 0.05;
        u -= view_tangent.x / view_tangent.z * height * parallax_scale;
        v -= view_tangent.y / view_tangent.z * height * parallax_scale;
        
        u = u - std::floor(u);  // 保持在[0,1]范围
        v = v - std::floor(v);
    }
    
    // 采样纹理颜色
    double height;
    Vec3 tex_color = brickTexture(u, v, height);
    
    // Phong光照
    return phong_shading(N, view_dir, light_dir, tex_color);
}
```

## 运行结果

![Parallax Mapping 对比效果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/02/02-24-Parallax-Mapping/parallax_output.png)

**对比说明**：
- **左半边**：普通纹理映射 - 砖块纹理看起来是平面的
- **右半边**：视差贴图 - 砖块有明显的立体感和深度

**观察要点**：
- 从倾斜角度看，右边的砖块边缘更清晰
- 高的砖块"遮挡"了后面的灰浆
- 整体表面看起来更有凹凸感

## 技术总结

### 学到的技术点

1. **视差贴图算法**
   - Simple Parallax Mapping（单次采样）
   - 基于视角和高度的UV偏移
   - 视差强度参数调节

2. **切线空间坐标系**
   - TBN矩阵的构建
   - 世界空间到切线空间的转换
   - 为什么需要切线空间（UV方向对齐）

3. **程序化纹理生成**
   - 砖块纹理算法（交错排列）
   - 同时输出颜色和高度信息
   - 模数运算实现纹理重复

### 技术对比

| 技术 | 原理 | 效果 | 性能 | 实现难度 |
|------|------|------|------|---------|
| **纹理映射** | 直接采样纹理 | 平面贴图 | ⭐⭐⭐⭐⭐ | ⭐ |
| **法线贴图** | 修改表面法线 | 光照变化 | ⭐⭐⭐⭐ | ⭐⭐ |
| **视差贴图** | 偏移UV坐标 | 深度感 | ⭐⭐⭐ | ⭐⭐⭐ |
| **位移贴图** | 修改几何形状 | 真实凹凸 | ⭐ | ⭐⭐⭐⭐ |

### 遇到的坑和解决方案

**问题1：编译警告 - 未使用的变量**
```
warning: variable 'light_tangent' set but not used
warning: variable 'view_tangent' set but not used
```

**原因**：初始代码计算了切线空间的光照向量，但最后在世界空间计算光照。

**解决**：移除多余的切线空间转换代码，保持简洁。

### 改进方向

当前实现是**Simple Parallax Mapping**，还有更高级的版本：

1. **Steep Parallax Mapping（陡峭视差贴图）**
   - 沿视线方向采样多次
   - 找到最接近表面的采样点
   - 效果更真实，但性能更慢

2. **Parallax Occlusion Mapping（POM）**
   - 使用二分查找精确定位交点
   - 添加自阴影效果
   - 质量最高，但性能开销大

3. **边缘处理优化**
   - 当前实现在边缘可能出现拉伸
   - 可以添加边缘检测和渐变混合

## 迭代历史

1. **初始版本**：单球体，左右半边切换
2. **用户反馈**："左右半边看起来一样"
3. **改进方案**：改为两个独立球体
   - 左球 (center: -1.5, 0, -3)：普通纹理映射
   - 右球 (center: 1.5, 0, -3)：视差贴图
4. **最终效果**：✅ 亮度差异 42.93，对比清晰可见

## 代码仓库

GitHub: https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/02-24-Parallax-Mapping

## 参考资料

- [Learn OpenGL - Parallax Mapping](https://learnopengl.com/Advanced-Lighting/Parallax-Mapping)
- GPU Gems 3: Chapter 18 - Relaxed Cone Stepping for Relief Mapping

---
**完成时间**: 2026-02-24 05:45  
**迭代次数**: 2 次  
**编译器**: g++ 12.3.1 -std=c++17 -O2  
**渲染时间**: ~3秒 (800x600)
