---
title: Perlin Noise 程序化纹理生成器
date: 2026-02-10 10:00:00
tags:
  - C++
  - 图形学
  - 程序化生成
  - Perlin Noise
categories:
  - 每日编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/perlin-noise-2026-02-10/output_marble.png
---

今天实现了经典的 Perlin Noise 算法，用于生成自然的程序化纹理。这是图形学中非常重要的技术，可以生成云层、大理石、木纹等多种自然纹理。

<!-- more -->

## 项目概述

- **项目名称**: Perlin Noise 程序化纹理生成器
- **开发时间**: 约 10 分钟
- **代码量**: 230 行 C++
- **编译状态**: ✅ 一次编译成功（0 错误 0 警告）
- **运行结果**: ✅ 成功生成 3 种纹理
- **项目地址**: [GitHub](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-02-10-perlin-noise)

## 技术要点

### Perlin Noise 算法

Perlin Noise 是由 Ken Perlin 在 1983 年发明的噪声生成算法，它能产生连续、自然的随机值。核心特点：

1. **梯度噪声**: 基于网格点的随机梯度向量
2. **平滑插值**: 使用 fade 函数（6t⁵ - 15t⁴ + 10t³）确保连续性
3. **可重复性**: 使用排列表实现伪随机

### Octave Noise（多层噪声）

通过叠加不同频率和振幅的噪声层，创造出更复杂的细节：

```cpp
double octaveNoise(double x, double y, int octaves, double persistence) {
    double total = 0;
    double frequency = 1;
    double amplitude = 1;
    double maxValue = 0;
    
    for (int i = 0; i < octaves; i++) {
        total += noise(x * frequency, y * frequency) * amplitude;
        maxValue += amplitude;
        amplitude *= persistence;
        frequency *= 2;
    }
    
    return total / maxValue;
}
```

### 三种纹理实现

1. **云层纹理**: 8 层噪声叠加，模拟自然云层的柔和渐变
2. **大理石纹理**: 噪声 + 正弦波扰动，产生大理石的纹理效果
3. **木纹纹理**: 径向距离 + 环状图案，模拟木材的年轮

## 生成效果

### 云层纹理
![云层纹理](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/perlin-noise-2026-02-10/output_clouds.png)

### 大理石纹理
![大理石纹理](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/perlin-noise-2026-02-10/output_marble.png)

### 木纹纹理
![木纹纹理](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/perlin-noise-2026-02-10/output_wood.png)

## 核心代码

### Fade 函数（平滑插值）

```cpp
double fade(double t) {
    return t * t * t * (t * (t * 6 - 15) + 10);
}
```

这个函数确保了噪声值的平滑过渡，避免了明显的网格痕迹。

### Perlin Noise 主函数

```cpp
double noise(double x, double y) {
    // 找到单位网格的整数坐标
    int X = (int)floor(x) & 255;
    int Y = (int)floor(y) & 255;
    
    // 获取相对坐标（0-1）
    x -= floor(x);
    y -= floor(y);
    
    // 计算 fade 曲线
    double u = fade(x);
    double v = fade(y);
    
    // 网格四个角的哈希值
    int A = p[X] + Y;
    int B = p[X + 1] + Y;
    
    // 双线性插值
    return lerp(v, 
        lerp(u, grad(p[A], x, y), grad(p[B], x - 1, y)),
        lerp(u, grad(p[A + 1], x, y - 1), grad(p[B + 1], x - 1, y - 1))
    );
}
```

## 编译运行

```bash
# 编译
g++ -std=c++17 -Wall -Wextra -O2 perlin_noise.cpp -o perlin_noise -lm

# 运行
./perlin_noise

# 输出文件
# - output_clouds.ppm (云层纹理)
# - output_marble.ppm (大理石纹理)
# - output_wood.ppm (木纹纹理)
```

## 学习收获

1. 理解了 Perlin Noise 的核心原理：梯度噪声 + 平滑插值
2. 掌握了 Octave Noise 技术：通过叠加不同频率的噪声创造细节
3. 实践了程序化纹理生成：用数学公式创造自然纹理
4. 优化了代码性能：使用排列表和位运算提升效率

## 下一步计划

- 扩展到 3D Perlin Noise
- 实现 Simplex Noise（更高维度的改进版）
- 尝试生成更多类型的纹理（如火焰、水面等）
- 集成到图形渲染管线中
