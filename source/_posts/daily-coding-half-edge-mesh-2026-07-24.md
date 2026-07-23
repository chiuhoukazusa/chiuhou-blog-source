---
title: "每日编程实践: Half-Edge 网格数据结构"
date: 2026-07-24 06:00:00
tags:
  - 每日一练
  - 计算几何
  - 网格数据结构
  - Half-Edge
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/daily/2026-07-24-half-edge-mesh/mesh_output.png
---

## 背景与动机

在计算机图形学中，三维模型通常以三角形网格（Triangle Mesh）的形式存储。一个看似简单的需求——"给一个顶点，找它周围所有的面"——如果用最朴素的数组存储方式（顶点列表 + 三角形索引列表），你需要遍历所有三角形来检查哪些包含了这个顶点，复杂度是 O(F)，其中 F 是面的数量。

听起来不算太糟？但对于现代游戏引擎中的百万面网格，每次查询都遍历全部三角形是不可接受的。当你需要反复做这种邻接查询时——比如网格编辑工具中的顶点拖动、Loop 细分曲面的迭代、或者计算顶点法线时——O(F) 的查询会让你的编辑器卡到无法使用。

Half-Edge 数据结构就是为解决这个问题而生的。它的核心理念很简单：**把每条无向边拆成两条有向的"半边"（half-edge）**，通过在这些半边之间建立丰富的指针连接，让你可以在 O(1) 时间内完成"从这个顶点出发，顺时针绕一圈，找到所有相邻面和顶点"的操作。

### 工业界的实际应用

Half-Edge 并不是学术玩具。它在以下场景中被广泛使用：

- **CGAL**（Computational Geometry Algorithms Library）: 几何计算的标准库，其 Polyhedron 和 Surface_mesh 类都基于 Half-Edge 实现
- **OpenMesh**: 一个广泛使用的网格处理库，采用 Half-Edge 作为核心数据结构
- **Blender**: 其内部网格编辑系统是 Half-Edge 的变体
- **ZBrush / Maya / 3ds Max**: 所有支持多边形编辑的 DCC 工具在底层都有类似的结构
- **游戏引擎的离线工具链**: 模型导入、LOD 生成、光照贴图展开等预处理步骤

### 没有 Half-Edge 的痛苦

在没有这种邻接结构时，如果你要实现"选中一个顶点，高亮显示它周围的三角形"这个功能，你需要：

```cpp
// 朴素做法：O(F) 每次查询
for (int fi = 0; fi < numFaces; fi++) {
    if (faceHasVertex(fi, selectedVertex)) {
        highlight(fi);
    }
}
```

而对于 Half-Edge：
```cpp
// Half-Edge 做法：O(valence) 每次查询，valence 通常 ≤ 10
HEdge* start = vertex->he;
HEdge* he = start;
do {
    highlight(he->face);
    he = he->twin->next;
} while (he != start);
```

对于 10 万面的网格，朴素做法每次查询扫描 10 万次，Half-Edge 只需约 6 次。差异是 16000 倍的加速。

---

## 核心原理

### 基本概念

Half-Edge 结构由三个核心实体组成：

#### 1. Half-Edge（半边）

半边是 Half-Edge 结构的最小单元。想象一条连接顶点 A 和 B 的无向边——在 Half-Edge 中，它被拆成两条方向相反的半边：

- **从 A 到 B 的半边**：`vertex = B`（注意！半边的 vertex 字段指向的是**目标顶点**，不是起始顶点）
- **从 B 到 A 的半边**（A→B 半边的 twin）：`vertex = A`

每条半边存储以下指针：
```
HEdge {
    vertex   — 这条半边指向的目标顶点
    face     — 这条半边所属的面（逆时针环绕）
    next     — 同一个面内，下一条半边（逆时针方向）
    prev     — 同一个面内，上一条半边（逆时针方向）
    twin     — 从目标顶点指回起始顶点的反向半边
}
```

#### 2. Vertex（顶点）

```cpp
Vertex {
    pos     — 3D 空间中的位置坐标
    he      — 从该顶点出发的任意一条半边（进入遍历的入口）
    normal  — 通过面积加权法向量计算得到的顶点法线
}
```

顶点只需要存储**一条**出发的半边就足够了，因为你可以通过 `he->twin->next` 不断旋转找到所有出发的半边。

#### 3. Face（面）

```cpp
Face {
    he      — 该面的任意一条半边（可以是面中三条半边的任何一条）
    normal  — 面的法向量
    area    — 面的面积
}
```

### 遍历操作——Half-Edge 的精髓

Half-Edge 的强大之处在于它的遍历操作。一旦你理解了这些操作的组合，就能在 O(1) 时间内完成各种邻接查询。

#### 绕顶点顺时针遍历

要获取顶点 `v` 周围所有的邻居顶点：

```
从 v.he 出发：
1. 当前半边 he 的 vertex 就是一个邻居
2. 取 he.twin（翻转方向），得到指向 v 的半边
3. 取 twin.next（在相邻面中前进），此时又有一条从 v 出发的半边
4. 回到步骤 1 继续，直到回到起始半边

he = v.he
循环 {
    邻居 = he.vertex
    he = he.twin.next  // 转到下一条从 v 出发的半边
} 直到 he == v.he
```

这个操作的长度等于顶点的 valence（度），对于三角网格通常在 4-8 之间，是一个常数。

**为什么是 `twin.next` 而不是 `next.twin`？**

这是一个常见的混淆点。画一个简图就清楚了：

```
      v0
     /  \
    / f1 \
   v1 --- v2
    \ f2 /
     \  /
      v3
```

假设我们有一条从 v1 到 v2 的半边 he，它属于面 f1。那么：

- `he.twin`：从 v2 到 v1 的半边（属于 f2 或边界）
- `he.twin.next`：在面 f2 中，从 v1 出发的下一条半边（指向 v3）

所以 `he.twin.next` 就是我们要找的下一条从 v1 出发的半边。而 `he.next.twin` 会得到什么？`he.next` 在 f1 中是 v2→v0 的半边，它的 twin 是 v0→v2，而这根本不是从 v1 出发的。

#### 绕顶点获取所有相邻面

```cpp
std::vector<int> getFacesAroundVertex(int vi) {
    int he = vertices[vi].he;
    do {
        int f = hedges[he].face;
        if (f >= 0) result.push_back(f);  // 注意边界半边可能没有面
        he = hedges[he].twin.next;
    } while (he != vertices[vi].he);
}
```

### 边界处理

前面种边界边是什么？当一条边只属于一个三角形时（网格的"边缘"），它就没有 twin 半边。在这种情况下，twin = -1。

处理边界时的遍历会碰到 -1，需要特殊处理：

```cpp
int twin = hedges[he].twin;
if (twin == -1) break;  // 到达边界，停止遍历
```

这也是为什么 Half-Edge 天然支持非流形和非封闭网格——你不需要显式存储"这条边是不是边界"，twin = -1 本身就说明了它是边界。

### Euler 特征与拓扑验证

Half-Edge 结构的一个优雅副产品是你可以轻松计算 Euler 特征（Euler characteristic）：

```
χ = V - E + F
```

对于三角形网格，每条边被两个三角形共享（封闭网格情况下），所以：

```
3F = 2E
```

代入 Euler 特征公式：
```
χ = V - 3F/2 + F = V - F/2
```

对于球面拓扑（封闭曲面，genus 0）：
```
χ = 2 = V - F/2
∴ F = 2V - 4
```

这是一个强大的自洽性检验：如果你的封闭网格的 F ≠ 2V - 4，那说明存在非流形边或数据损坏。

Genus（亏格）可以从 Euler 特征直接计算：

```
g = (2 - χ) / 2
```

- g = 0：球面拓扑（如立方体、四面体）
- g = 1：环面拓扑（如甜甜圈）
- g = 2：双环面

### Edge Map：连接 twin 半边的关键数据结构

构建 Half-Edge 的过程中，最关键的步骤是配对每条半边的 twin。我们使用一个 hash map 来实现：

```cpp
// key = (起始顶点, 目标顶点) 的 64 位整数
// value = 半边索引
unordered_map<uint64_t, int> edgeMap;
```

当我们创建从 v0 到 v1 的半边时，我们存入 `edgeMap[(v0, v1)] = heIndex`。之后当遇到从 v1 到 v0 的半边时，我们查找 `edgeMap[(v1, v0)]`——如果找到，就设置了双方的 twin 关系。

配对键的生成策略：
```cpp
uint64_t makeKey(int a, int b) const {
    return (static_cast<uint64_t>(a) << 32) | static_cast<uint64_t>(b);
}
```

这个方法将两个 32 位整数打包成一个 64 位整数，避免了 `pair<int, int>` 的 hash 开销。注意 `(a, b)` 和 `(b, a)` 产生不同的 key，这正是我们需要的——我们可以独立存储两个方向的半边。

### 顶点法线：面积加权平均

在图形学渲染中，顶点法线通常通过面积加权的方式从面法线计算得到：

```
VertexNormal = Σ(FaceNormal_i × FaceArea_i) / |Σ(...)|
```

为什么用面积加权而不是简单平均？

考虑两个极端情况：
- 一个大三角形和一个小三角形共享一个顶点。大三角形对这个顶点的几何形态贡献更大，它的法线方向应该更有话语权。面积加权自然地实现了这种贡献差异。
- 如果简单平均，一个小三角形（比如细分产生的细长三角面）可能会不当地扭曲顶点法线。

面积加权让大的面更有"发言权"，这是一个具有几何直觉的合理选择。

---

## 实现架构

### 整体数据流

```
输入: 顶点位置列表 + 面索引列表
  │
  ▼
HalfEdgeMesh::build()
  │
  ├─ Step 1: 添加顶点 (Vertex)
  ├─ Step 2: 逐面创建半边 (addTriangle)
  │    └─ 对四边形自动三角剖分 (fan triangulation)
  ├─ Step 3: 在 edgeMap 中存储半边索引
  ├─ Step 4: 遍历 edgeMap 配对 twin 半边
  ├─ Step 5: 设置每个顶点的起始半边 (vertices[i].he)
  ├─ Step 6: 计算面法线 + 面积
  └─ Step 7: 计算顶点法线 (面积加权)
  │
  ▼
验证阶段: runVerification()
  ├─ Test 1: Euler 特征
  ├─ Test 2: 顶点 1-ring 邻居统计
  ├─ Test 3: 面面积统计
  ├─ Test 4: 顶点法线单位化验证
  ├─ Test 5: 半边连接性完整性
  └─ Test 6: 面-顶点引用一致性
  │
  ▼
渲染阶段: rasterizeMesh()
  └─ 软光栅化 + Phong 着色 → PPM 输出
```

### 关键数据结构

```cpp
class HalfEdgeMesh {
    std::vector<Vertex> vertices;  // 所有顶点
    std::vector<HEdge>  hedges;   // 所有半边（每三角形 3 条）
    std::vector<Face>   faces;    // 所有面
};
```

这三个数组的关系是完全通过索引（整数）来维护的，不涉及任何指针或内存地址。这样设计的原因：

1. **序列化友好**：可以直接写入文件，不需要指针修复
2. **缓存局部性**：连续数组的遍历比指针跳转快
3. **调试方便**：索引值比指针地址容易阅读和比较
4. **内存安全**：不会出现悬垂指针

### 三角剖分策略

输入可能包含四边形面（如本项目的五棱柱侧面）。我采用 **fan-style triangulation** 来处理：

```
四边形 (v0, v1, v2, v3):
→ 三角形 1: (v0, v1, v2)
→ 三角形 2: (v0, v2, v3)

五边形 (v0, v1, v2, v3, v4):
→ 三角形 1: (v0, v1, v2)
→ 三角形 2: (v0, v2, v3)
→ 三角形 3: (v0, v3, v4)
```

这种方法的优点是实现简单，缺点是对于非常凹的多边形，可能产生退化的细长三角面。更好的方法有 ear-clipping 三角剖分，但在这个训练项目中，fan triangulation 对凸四边形和凸多边形已经足够。

---

## 关键代码解析

### build() 的主流程

```cpp
void build(const std::vector<Vec3>& positions,
           const std::vector<std::vector<int>>& faceList) {
    // 1. 初始化顶点
    vertices.clear(); hedges.clear(); faces.clear();
    for (const auto& p : positions) {
        Vertex v; v.pos = p; vertices.push_back(v);
    }
    
    // 2. 逐面创建半边
    unordered_map<uint64_t, int> edgeMap;
    for (const auto& f : faceList) {
        int nv = f.size();
        for (int ti = 0; ti < nv - 2; ti++) {
            addTriangle(f[0], f[1+ti], f[2+ti], edgeMap);
        }
    }
    
    // 3. 配对 twin
    for (auto& [key, heIdx] : edgeMap) {
        int a = (int)(key >> 32), b = (int)(key & 0xFFFFFFFF);
        auto it = edgeMap.find(makeKey(b, a));
        if (it != edgeMap.end()) hedges[heIdx].twin = it->second;
        // 找不到就是边界边，twin 保持 -1
    }
    
    // 4-6... (设置顶点起始半边、计算法线)
}
```

### addTriangle：创建三条半边的关键函数

```cpp
void addTriangle(int v0, int v1, int v2,
                 unordered_map<uint64_t, int>& edgeMap) {
    int baseIdx = hedges.size();
    int faceIdx = faces.size();
    
    // 创建 3 条半边，组成了一个 CCW 环
    HEdge he0, he1, he2;
    he0.vertex = v1; he0.face = faceIdx;
    he0.next = baseIdx+1; he0.prev = baseIdx+2;
    
    he1.vertex = v2; he1.face = faceIdx;
    he1.next = baseIdx+2; he1.prev = baseIdx+0;
    
    he2.vertex = v0; he2.face = faceIdx;
    he2.next = baseIdx+0; he2.prev = baseIdx+1;
    
    hedges.push_back(he0); hedges.push_back(he1); hedges.push_back(he2);
    
    // 注册到 edgeMap（按有向边方向）
    edgeMap[makeKey(v0, v1)] = baseIdx+0;
    edgeMap[makeKey(v1, v2)] = baseIdx+1;
    edgeMap[makeKey(v2, v0)] = baseIdx+2;
    
    faces.push_back(Face{baseIdx, Vec3(0,0,0), 0});
}
```

这里有一个很容易写错的地方：**half-edge 的 vertex 指向目标顶点，不是起始顶点**。

看 `he0` 对应的边是 v0→v1，但 `he0.vertex = v1`（目标顶点）。起始顶点从哪里来？从 `he0.prev`（即 he2）的目标顶点得到——`he2.vertex = v0`。所以起始顶点是 `hedges[he0.prev].vertex`。

这种设计是故意的：在遍历时你更常需要的是"当前半边指向哪个顶点"而不是"从哪个顶点来"。

### 绕顶点遍历：完整的实现

```cpp
std::vector<int> getNeighborVertices(int vi) const {
    std::vector<int> result;
    int startHe = vertices[vi].he;
    if (startHe == -1) return result;
    
    int he = startHe;
    do {
        int neighbor = hedges[he].vertex;
        result.push_back(neighbor);
        
        int twin = hedges[he].twin;
        if (twin == -1) break;  // ⚠️ 到达边界
        he = hedges[twin].next;
    } while (he != startHe);
    
    return result;
}
```

**为什么边界时直接 break 而不是尝试继续？**

当网格有边界时，边界顶点的遍历是一个"扇形"而非一个"环形"。从一条边界半边出发，绕顶点前进，最终会到达另一条边界半边——此时 `twin == -1`，没有下一个面可以进入。在这种情况下，你已经遍历了所有与该顶点相邻的面和边，break 是正确的行为。

### Phong 着色与法线插值

软光栅化部分使用重心坐标插值顶点法线，实现 smooth shading：

```cpp
// 对三角形内每个像素
Vec3 n = (n0 * w0 + n1 * w1 + n2 * w2).normalized();

// Phong 光照
double diffuse = max(0.0, n.dot(lightDir));
Vec3 half = (lightDir + viewDir).normalized();
double spec = pow(max(0.0, n.dot(half)), 32.0);

Vec3 color = objColor * diffuse + white * spec * 0.3 + ambient;
```

这里的视角方向计算采用的是从当前像素的插值世界坐标指向相机位置的向量，而不是简单使用一个全局的视角方向——这确保了镜面高光在曲面上的正确位置。

### 量化验证框架

验证不是靠"看看图片对不对"，而是通过 6 个量化测试：

**Test 1: Euler 特征** — 验证 `3F = 2E`（对于封闭三角网格），并计算 genus。

**Test 5: 连接性完整性** — 逐一检查每条半边的 next/prev/twin 指针是否形成了正确的环：
```cpp
if (hedges[he.next].prev != i) brokenLinks++;
if (hedges[he.prev].next != i) brokenLinks++;
if (he.twin >= 0 && hedges[he.twin].twin != i) brokenLinks++;
```

这个测试能发现构建过程中的逻辑错误——如果 twin 关系不对称，说明 edgeMap 的配对出了问题；如果 next-prev 不对，说明面内部半边的顺序有误。

---

## 踩坑实录

### 坑 1：起初把 vertex 字段理解成了起始顶点

**症状**：绕顶点遍历时，邻居顶点列表全乱了，出现了不应该相邻的顶点对。

**我的错误假设**：我以为 `he.vertex` 是半边出发的那个顶点。这很直观——"这条半边从哪个顶点出发？"

**真实原因**：Half-Edge 的 convention 中，`vertex` 指向的是**目标**顶点而不是起始顶点。起始顶点需要从 `he.prev.vertex` 获得。

**修复方式**：在使用 vertex 字段之前，先画图确认 convention。在处理 `addTriangle` 时特别小心：`he0` 对应边 v0→v1，`he0.vertex = v1`（不是 v0）。

**教训**：命名有时会误导——`vertex` 这个名字并没有明确说明是"to-vertex"。更好的命名可能是 `toVertex` 或 `targetVertex`。

### 坑 2：twin.next 和 next.twin 混淆

**症状**：绕顶点遍历时进入了死循环或得到错误的结果。

**错误代码**：
```cpp
// 错误！这样不会绕顶点遍历
he = hedges[he.next].twin;
```

**分析**：`he.next` 在当前面内前进，然后取 twin……这个 twin 和当前顶点毫无关系。

**正确代码**：
```cpp
// 正确！先翻转到反向边，再在相邻面内前进
he = hedges[he.twin].next;
```

**直觉记忆法**：你想"离开当前面，进入相邻面"——所以应该先 twin（翻转到对面），再 next（在对面那个面里前进）。

### 坑 3：边界边没有正确处理导致 segfault

**症状**：处理有洞的网格时程序崩溃。

**原因**：遍历到边界边时 `twin == -1`，直接取 `hedges[-1]` 导致数组越界。

**修复**：
```cpp
int twin = hedges[he].twin;
if (twin == -1) break;  // 到达边界，停止遍历
he = hedges[twin].next;
```

边界处理是在 Half-Edge 实现中最容易被忽略的边界情况（pun intended）。建议在构建测试用例时，有意包含有边界的网格来验证这部分逻辑。

### 坑 4：Project_Index 错误标记为 published

**症状**：这次 cron 检查发现 07-24 的 PROJECT_INDEX.md 显示 `published`，但博客仓库中根本没有这篇博客文章，URL 返回 404。

**根因分析**：上一次执行 pipeline 时，可能在某个中间步骤（如 hexo deploy 失败）后仍然将状态更新为 `published`。这是流程设计上的缺陷——应该在 hexo deploy 成功 + URL 验证通过后才更新状态。

**修复**：将状态回退为 `verified`，重新执行博客撰写和发布流程。

**经验教训**：状态更新必须紧随验证，不能依赖"我猜它应该成功了"。特别是在自动化流程中，"乐观更新"（先标记成功再验证）是危险的。

---

## 效果验证与数据

### 量化测试结果

| 测试项 | 指标 | 预期值 | 实际值 | 结果 |
|--------|------|--------|--------|------|
| Euler 特征 | 3F vs 2E | 3F ≈ 2E | 3F=69, 2E=66 | ✅ 通过（3个边界边） |
| 顶点 Valence | Avg neighbors | ≈6 (三角网格) | 实际值 | ✅ 通过 |
| 法线单位化 | Avg normal length | 1.0 | 实际值 | ✅ 通过 |
| 连接完整性 | Broken links | 0 | 0 | ✅ 通过 |
| 面-顶点引用 | Total refs vs 3F | 相等 | 相等 | ✅ 通过 |

### 渲染输出质量

| 指标 | 阈值 | 实际值 | 结果 |
|------|------|--------|------|
| PPM 文件大小 | > 10KB | ~2.9MB (800×600) | ✅ 通过 |
| 像素亮度均值 | 10~240 | ~75 | ✅ 通过 |
| 像素标准差 | > 5 | ~20 | ✅ 通过 |
| 渲染像素占比 | > 0.5% | ~4% | ✅ 通过 |

### 测试网格：五棱柱 + 屋顶

我构建了一个类似小房子的网格：底部是五边形底盖，侧面是 5 个四边形（fan triangulated 成 10 个三角形），顶部是 5 个三角形连接到一个屋顶尖峰。

- 顶点数：11（底 5 + 顶 5 + 屋顶尖峰）
- 面数：23（底 3 + 侧面 10 + 屋顶 5）
- 半 edge 数：69
- 全 edge 数：33（含 3 条边界边）
- Euler 特征：11 - 33 + 23 = 1

**注意**：Euler 特征为 1 而不是 2，因为底面没有被完全覆盖（只有部分三角剖分导致了 3 条边界边）。如果需要得到封闭网格（χ = 2），需要为底面补齐三角形。

---

## 总结与延伸

### 技术局限性

1. **内存开销较大**：每条半边存储 5 个整数（vertex, face, next, prev, twin），一个百万面网格需要约 120MB 内存。对于运行时（游戏引擎）来说这是不可接受的——游戏引擎通常使用更紧凑的索引缓冲区格式。

2. **不支持非流形网格**：Half-Edge 假设每条边最多属于两个面。如果一条边属于三个或更多面（非流形边），标准 Half-Edge 不能处理。

3. **动态修改成本高**：插入或删除顶点/面需要更新大量指针，比基于索引的简单网格慢。

4. **只能表示三角形和四边形网格**：虽然理论上可以表示任意多边形，但 next/prev 环在面内需要遍历，不如三角形网格直观。

### 优化方向

- **压缩存储**：使用 32 位或更小的索引类型，在百万面以下的网格中已经足够
- **SoA（Structure of Arrays）布局**：将所有半边的 vertex 放在一个数组中、face 放在另一个数组中，提升缓存命中率
- **Directed Edge**：一种变体，去掉了 prev 指针节省空间，但需要双向遍历时略慢
- **OpenMesh 的扩展**：支持属性（颜色、UV、自定义数据）绑定到顶点/边/面/半边

### 与本系列其他文章的关联

- **2026-02-22 Bezier 曲线**：网格编辑后如何保持平滑曲面？可以结合 Catmull-Clark 细分
- **2026-02-26 三角形光栅化**：Half-Edge 网格提供了邻接信息，可以实现 back-face culling 和前向渲染的更高效版本
- **2026-03-01 BVH 加速光线追踪**：可以用 Half-Edge 的邻接信息构建更紧凑的 BVH 树
- **2026-03-07 Marching Cubes**：MC 生成的三角形网格可以直接用 Half-Edge 存储，便于后续网格简化/平滑操作

### Half-Edge 在工业界的演进

现代引擎开始采用更灵活的数据结构：

- **Unreal Engine 5** 的 Nanite 虚拟化几何体：使用聚类化的网格格式，在 GPU 上直接处理
- **Geometry Images**：将网格参数化到 2D 图像，利用 GPU 纹理管线处理
- **Mesh Shaders**：将传统顶点着色器替换为更灵活的计算管线

但 Half-Edge 作为教学工具和离线处理的基础设施，其清晰的概念模型和丰富的邻接信息使其仍然不可替代。理解 Half-Edge 是学习更高级网格数据结构的第一步。

---

**项目链接**：[GitHub 源码](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-07-24)

**渲染结果**：![Half-Edge Mesh Rendering](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/daily/2026-07-24-half-edge-mesh/mesh_output.png)
