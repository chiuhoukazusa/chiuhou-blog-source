---
title: Voronoi 图生成器 - 计算几何的暴力之美
date: 2026-02-11 10:00:00
tags:
  - C++
  - 图形学
  - 计算几何
  - Voronoi Diagram
categories:
  - 每日编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/voronoi-2026-02-11/voronoi.png
---

# Voronoi 图生成器 - 计算几何的暴力之美

今天实现了经典的 Voronoi 图生成器，虽然使用暴力法（O(N×W×H)），但展示了计算几何的直观本质。

## 本项目代码已托管至 GitHub

每日编程实践项目，记录每天的学习和实现。

https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/02-11-voronoi-generator

---

## 项目简介

**Voronoi 图**是计算几何中非常重要的一种空间划分方法，将平面划分为若干个区域，每个区域内点到该区域对应的种子点的距离比到其他种子点更近。

- **开发时间**: 约 13 分钟
- **代码量**: 75 行 C++
- **编译状态**: ✅ 一次编译成功（0 错误，0 警告）
- **运行结果**: ✅ 成功生成 800×600 像素的 Voronoi 图

## 技术要点

### Voronoi 图算法

**暴力法实现**（Brute-force）：
```cpp
for (int y = 0; y < height; y++) {
    for (int x = 0; x < width; x++) {
        double minDist = std::numeric_limits<double>::max();
        int closestSeed = -1;
        
        for (int i = 0; i < numSeeds; i++) {
            double dx = x - seeds[i].x;
            double dy = y - seeds[i].y;
            double dist = dx * dx + dy * dy;  // 平方距离（避免开方）
            
            if (dist < minDist) {
                minDist = dist;
                closestSeed = i;
            }
        }
        
        // 使用最近种子点的颜色着色
        image[y][x] = colors[closestSeed];
    }
}
```

### 技术特点

1. **空间分割**: 基于最近邻原则自动划分平面
2. **效率取舍**: 暴力法 O(N×W×H)，50个种子 × 800×600像素 = 2400万次距离计算
3. **颜色编码**: 每个种子点有一个随机生成的颜色，相邻区域颜色不同

### 应用场景

**Voronoi 图在实际中的用途广泛**：
. **游戏地图生成**（Roguelike、策略游戏）
. **程序化纹理合成**（细胞状纹理、岩石纹理）
. **空间优化问题**（设施选址、服务区划分）
. **程序化内容生成**（地形、生物群落）

## 生成效果

### Voronoi 图效果
![Voronoi 图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/voronoi-2026-02-11/voronoi.png)

图中清晰地展示了 50 个随机种子点划分的 800×600 空间，每个多边形区域内的所有点到该区域种子点的距离都比到其他种子点更近。

## 编译运行

```bash
# 编译
g++ -std=c++11 -Wall -Wextra -lm voronoi.cpp -o voronoi

# 运行（生成 voronoi.png）
./voronoi
```

### 优化方向

1. **性能优化**
   - Fortune 算法 O(N log N)（线扫描法）
   - 增量构建算法
   - 空间分割加速（网格、四叉树）

2. **功能扩展**
   - 不同距离度量（曼哈顿、切比雪夫）
   - 加权 Voronoi 图
   - 更高维度扩展（3D Voronoi）
   - Lloyd 松弛算法（生成更规则的多边形）

3. **可视化增强**
   - 种子点边界标记
   - 实时交互更新
   - 动画显示构建过程

## 核心代码

### 随机种子点生成

```cpp
std::mt19937 rng(std::random_device{}());
std::uniform_int_distribution<int> distX(0, width-1);
std::uniform_int_distribution<int> distY(0, height-1);

for (int i = 0; i < numSeeds; i++) {
    seeds.push_back(Point{distX(rng), distY(rng)});
}
```

### 图像生成与保存

```cpp
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"

int main() {
    // 生成图像数据
    generateVoronoi(image, seeds);
    
    // 保存为 PNG
    stbi_write_png("voronoi.png", width, height, 3, image.data(), width * 3);
    
    std::cout << "Voronoi 图已生成: voronoi.png" << std::endl;
    return 0;
}
```

## 技术栈

- **语言**: C++11
- **随机数**: Mersenne Twister 引擎 (std::mt19937)
- **图像处理**: stb_image_write.h（单文件库）
- **编译选项**: -Wall -Wextra（严格警告检查）

## 学习收获

1. **理解了空间划分的本质**：基于距离度量的区域分割
2. **体验了算法复杂度的影响**：暴力法虽然简单但运算量大
3. **掌握了随机图案生成方法**：可以扩展到更多程序化内容生成场景
4. **学习了计算几何的直观实现**：从数学概念到具体实现

## 下一步计划

- 实现 Fortune 算法（线性时间）
- 扩展到 3D Voronoi 图
- 应用于游戏地图生成系统
- 研究 Lloyd 松弛算法的实现

---

**技术参考**
- [Voronoi Diagram - Wikipedia](https://en.wikipedia.org/wiki/Voronoi_diagram)
- [Fortune's Algorithm for Voronoi Diagrams](https://en.wikipedia.org/wiki/Fortune%27s_algorithm)
- [stb 单文件库](https://github.com/nothings/stb)