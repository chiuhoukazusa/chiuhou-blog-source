---
title: "每日编程实践: 纹理映射光线追踪器"
date: 2026-02-20 05:35:00
tags:
  - 每日一练
  - 图形学
  - 光线追踪
  - 纹理映射
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/02/02-20-纹理映射光线追踪器/texture_output.png
---

# 纹理映射光线追踪器

## 项目目标

实现完整的**球面UV映射**和**纹理采样**系统，支持在光线追踪器中渲染带纹理的球体。

关键技术：
- 球面坐标到UV坐标的转换
- 程序化纹理生成（棋盘格）
- 材质系统（纹理/纯色混合）
- 基于UV坐标的纹理采样

## 实现过程

### 迭代历史

1. **初始版本**: 一次性实现完成
   - 球面UV映射公式 (`sphereUV`)
   - 棋盘格纹理生成 (`checkerboardTexture`)
   - 材质系统 (`hasTexture` 标志)
   - 量化验证通过

**为什么一次成功？**
- 球面UV映射的数学公式已标准化
- 使用成熟的反三角函数库
- 严格的量化验证流程

## 核心代码

### 球面UV映射

```cpp
UV sphereUV(const Vec3& point) {
    // point 是单位球面上的点
    Vec3 p = point.normalize();
    
    // 经度 → u坐标 (使用 atan2 处理方位角)
    double u = 0.5 + atan2(p.z, p.x) / (2 * M_PI);
    
    // 纬度 → v坐标 (使用 asin 处理俯仰角)
    double v = 0.5 - asin(p.y) / M_PI;
    
    return UV(u, v);
}
```

**关键点**：
- `atan2(z, x)` 计算方位角，范围 [-π, π]
- `asin(y)` 计算俯仰角，范围 [-π/2, π/2]
- 归一化到 [0, 1] 范围

### 棋盘格纹理

```cpp
Vec3 checkerboardTexture(const UV& uv, int scale = 10) {
    int ui = static_cast<int>(floor(uv.u * scale));
    int vi = static_cast<int>(floor(uv.v * scale));
    
    // 奇偶判断
    bool isEven = ((ui + vi) % 2) == 0;
    
    // 白色 vs 蓝色
    return isEven ? Vec3(0.9, 0.9, 0.9) : Vec3(0.2, 0.2, 0.8);
}
```

### 材质系统

```cpp
struct Sphere {
    bool hasTexture;  // 是否使用纹理
    
    Vec3 getColor(const Vec3& point) const {
        if (!hasTexture) return color;  // 纯色球
        
        // 转换到局部坐标系
        Vec3 localPoint = (point - center) * (1.0 / radius);
        
        // 计算UV并采样纹理
        UV uv = sphereUV(localPoint);
        return checkerboardTexture(uv, 10);
    }
};
```

## 运行结果

![纹理映射结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/02/02-20-纹理映射光线追踪器/texture_output.png)

### 场景说明

- **中心大球**: 棋盘格纹理，蓝白相间
- **左侧小球**: 纯色红色（无纹理）
- **右侧小球**: 棋盘格纹理
- **地面**: 大球模拟平面，棋盘格纹理

### 量化验证结果

使用 ImageMagick 提取关键区域像素值：

```bash
# 中心球蓝色格
convert texture_output.png -crop 100x100+350+250 +repage -scale 1x1! \
    -format "RGB(%[fx:int(255*r)],%[fx:int(255*g)],%[fx:int(255*b)])" info:-
# 输出: RGB(83,83,129) ✅ 蓝色系，符合预期

# 左侧红球
convert texture_output.png -crop 80x80+180+280 +repage -scale 1x1! \
    -format "RGB(%[fx:int(255*r)],%[fx:int(255*g)],%[fx:int(255*b)])" info:-
# 输出: RGB(161,68,68) ✅ 红色系，符合预期

# 颜色统计
identify -verbose texture_output.png | grep -A 3 "Channel statistics"
# Red: min=10, max=254, mean=123.7 ✅ 正常分布
```

## 技术总结

### 学到的技术点

1. **球面UV映射**
   - 理解了球坐标系到UV坐标的转换
   - 掌握了 `atan2` 和 `asin` 的正确用法
   - 处理了坐标范围归一化

2. **程序化纹理**
   - 棋盘格纹理的实现原理（奇偶判断）
   - 纹理缩放（scale参数）

3. **材质系统**
   - 纹理/纯色混合渲染
   - 局部坐标系转换

### 遇到的坑和解决方案

**潜在问题**：UV坐标在球体两极可能不连续

**解决方案**：
- 使用 `normalize()` 确保输入是单位向量
- `atan2` 自动处理符号问题
- 对于特殊情况（如纹理接缝），可以使用纹理重复模式

### 性能优化

- **分辨率**: 800x600
- **渲染时间**: ~1秒
- **优化空间**: 
  - BVH加速结构（减少光线求交次数）
  - 多线程渲染（并行化像素计算）

## 代码仓库

GitHub: https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/02-20-纹理映射光线追踪器

## 下一步计划

- [ ] 支持图片纹理加载（stb_image）
- [ ] 法线贴图（Bump Mapping）
- [ ] 环境贴图（Reflection Mapping）
- [ ] 柱面/平面UV映射
- [ ] 纹理过滤（双线性插值）

---

**完成时间**: 2026-02-20 05:35  
**迭代次数**: 1 次  
**编译器**: g++ (GCC)
