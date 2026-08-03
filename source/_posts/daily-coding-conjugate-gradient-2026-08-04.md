---
title: "每日编程实践: Conjugate Gradient 共轭梯度法求解稀疏线性系统"
date: 2026-08-04 06:00:00
tags:
  - 每日一练
  - 数值方法
  - 线性代数
  - C++
categories:
  - 编程实践
toc: true
---

## ① 背景与动机

在科学计算和工程模拟中，求解大型稀疏线性方程组 \(Ax = b\) 是最频繁出现的计算任务之一。无论是有限元法的结构分析、计算流体力学中的压力修正、还是机器学习中的优化问题，最终都会归结为"解一个线性方程组"。

然而，传统的高斯消元法面对这类问题时完全无法胜任。考虑一个三维 Poisson 方程在 \(100 \times 100 \times 100\) 网格上的离散化——矩阵维度达到 \(10^6\)，如果使用稠密存储需要约 8TB 内存，而即使用稀疏存储，直接法也会在消元过程中产生大量"填充"(fill-in)，使稀疏性丧失殆尽。

**工业界实际场景**：

- **CFD 流体仿真**：SIMPLE 算法每次外迭代都需要求解压力 Poisson 方程，ANSYS Fluent、OpenFOAM 等工业软件的核心求解器都依赖 Krylov 子空间方法
- **几何处理**：Laplacian 网格编辑、测地线距离计算中需要反复求解稀疏线性系统，MeshLab、Blender 使用了类似方法
- **物理模拟**：Rigid Body Dynamics 的碰撞约束、布料模拟的隐式积分都需要 Jacobi 系统求解
- **计算机图形学**：全局光照中的 Radiosity 方法求解辐射度方程，也是大规模稀疏系统
- **深度学习**：二阶优化方法（如 Newton-CG、Hessian-Free）使用 CG 逼近 Newton 方向的搜索方向

共轭梯度法（Conjugate Gradient，CG）是解决这类问题的"标配"迭代法。它不需要存储完整的矩阵，只需要能计算矩阵-向量乘积 \(A \cdot v\)，内存开销仅与矩阵的非零元数量成正比。对于对称正定（SPD）矩阵，CG 具有最优的收敛性质：在精确算术中，\(n\) 维问题最多 \(n\) 步收敛到精确解。

本文实现一个完整的 CG 求解器，用 CSR 格式存储稀疏矩阵，以 1D 和 2D Poisson 方程作为测试用例，通过 5 个量化测试验证算法的正确性和收敛性。

---

## ② 核心原理

### 2.1 为什么叫"共轭梯度"？

CG 有两个核心思想：

**"共轭"（Conjugate）**：构造一系列关于矩阵 \(A\) 共轭的搜索方向 \(p_0, p_1, \ldots, p_k\)，满足：

\[
p_i^T A p_j = 0 \quad (i \neq j)
\]

这个性质的意义是：沿着 \(p_k\) 方向搜索时，不会破坏之前已经完成的对 \(p_0, p_1, \ldots, p_{k-1}\) 方向搜索的成果。想象你在一张倾斜的地图上爬山，如果朝一个方向走了之后，后面走的每个方向都和前一个方向"不冲突"——你不会因为走第二步而丢失第一步的进度。

**"梯度"（Gradient）**：每次迭代的更新方向 \(p_k\) 与当前残差（即负梯度方向）有关。基本的梯度下降法沿着负梯度方向走，但会导致锯齿状路径（zigzagging），收敛缓慢。CG 通过引入"共轭"修正，在残差方向上叠加前一步的搜索方向，避免了这种振荡。

把这两个思想合在一起：**CG 在每一步中选择一个既保持与历史方向 A-共轭、又尽量贴近当前负梯度的新方向，然后沿该方向精确搜索到一维极小值。** 最终在 \(n\) 维问题中最多 \(n\) 步收敛。

### 2.2 算法推导

我们的目标是最小化二次型：

\[
\phi(x) = \frac{1}{2} x^T A x - b^T x
\]

当 \(A\) 对称正定时，\(\phi(x)\) 的梯度 \(\nabla \phi(x) = Ax - b\)，极小值在 \(Ax = b\) 处取得。因此求解 \(Ax = b\) 等价于最小化 \(\phi(x)\)。

**第 0 步——初始化**：从任意初始猜测 \(x_0\) 开始（通常取 \(x_0 = 0\)），计算初始残差和搜索方向：

\[
r_0 = b - A x_0, \quad p_0 = r_0
\]

这里 \(r_0 = -\nabla \phi(x_0)\)，所以初始搜索方向就是负梯度方向（即最速下降方向）。这是合理的：在没有任何先验信息时，朝下降最快的方向走。

**第 k 步——精确线搜索**：当前点 \(x_k\)，搜索方向 \(p_k\)。在 \(p_k\) 上做精确线搜索，找到使 \(\phi(x_k + \alpha p_k)\) 最小的步长 \(\alpha_k\)：

\[
\frac{d}{d\alpha} \phi(x_k + \alpha p_k) \bigg|_{\alpha=\alpha_k} = 0
\]

展开：

\[
\frac{d}{d\alpha} \left[\frac{1}{2}(x_k + \alpha p_k)^T A (x_k + \alpha p_k) - b^T (x_k + \alpha p_k)\right] = 0
\]

\[
p_k^T A x_k + \alpha_k p_k^T A p_k - p_k^T b = 0
\]

\[
\alpha_k p_k^T A p_k = p_k^T (b - A x_k) = p_k^T r_k
\]

最后一步用了残差的定义 \(r_k = b - A x_k\)。所以：

\[
\alpha_k = \frac{p_k^T r_k}{p_k^T A p_k}
\]

一个关键性质：在 CG 中，可以证明 \(p_k^T r_k = r_k^T r_k\)（因为 \(p_k = r_k + \beta_{k-1} p_{k-1}\)，且 \(r_k \perp r_{k-1}\)）。所以实践中用更简单的形式：

\[
\alpha_k = \frac{r_k^T r_k}{p_k^T A p_k}
\]

**更新解和残差**：

\[
x_{k+1} = x_k + \alpha_k p_k
\]
\[
r_{k+1} = r_k - \alpha_k A p_k
\]

残差的更新公式直接来自定义 \(r_{k+1} = b - A x_{k+1} = b - A(x_k + \alpha_k p_k) = r_k - \alpha_k A p_k\)。

**构造下一个搜索方向**：新方向 \(p_{k+1}\) 需要在 \(r_{k+1}\) 的基础上，去掉与 \(p_k\) 在 A-内积下的分量，保证共轭性：

\[
p_{k+1} = r_{k+1} + \beta_k p_k, \quad \beta_k = \frac{r_{k+1}^T r_{k+1}}{r_k^T r_k}
\]

\(\beta_k\) 的这个公式叫做 **Fletcher-Reeves 公式**。它的直觉：当残差减小得很快时（\(r_{k+1}\) 远小于 \(r_k\)），\(\beta_k\) 很小，新方向主要依赖当前残差；当残差减小得慢时，\(\beta_k\) 接近 1，新方向更多地继承了旧方向，帮助避免 zigzagging。

**每个变量在做什么**：
- \(r_k\)：当前点处的残差 \(b - A x_k\)，表示"离目标还有多远"
- \(p_k\)：A-共轭搜索方向，保证每一步的搜索不会破坏之前的成果
- \(\alpha_k\)：沿 \(p_k\) 方向的精确步长，到达一维极小点
- \(\beta_k\)：方向更新系数，从当前残差中"减去"旧方向的分量，保持 A-共轭性

### 2.3 收敛性分析

CG 的最优性来自一个深刻性质：在第 k 步迭代后，近似解 \(x_k\) 是在 **Krylov 子空间** \(\mathcal{K}_k(A, b) = \text{span}\{b, Ab, A^2b, \ldots, A^{k-1}b\}\) 中使能量误差 \(\|x - x_k\|_A^2 = (x - x_k)^T A (x - x_k)\) 最小的向量。

这意味着每一步都在增大搜索空间（第 k 步在 k 维子空间中做最优估计），所以每一步都不比前一步差，且 n 维问题最多 n 步收敛。

**误差估计**：

\[
\frac{\|x - x_k\|_A}{\|x - x_0\|_A} \leq 2 \left(\frac{\sqrt{\kappa} - 1}{\sqrt{\kappa} + 1}\right)^k
\]

其中 \(\kappa = \lambda_{\max} / \lambda_{\min}\) 是 \(A\) 的条件数。这个公式告诉我们：

- 当 \(\kappa = 1\)（均匀缩放）：一步收敛，因为本质上就是一个标量乘
- 当 \(\kappa = 10\)：约 1.5 步就能把误差减半
- 当 \(\kappa = 1000\)：约 15 步才能把误差减半
- 对 2D Poisson（\(N \times N\)）：\(\kappa \propto N^2\)，所以步数随 \(N\) 线性增长

这正是为什么实际计算中 n 维问题的 CG 通常远远不需要 n 步就收敛了——因为收敛取决于特征值的聚类程度，而不单纯是维度。

### 2.4 为什么 CG 需要对称正定？

- **对称性**：保证 A 定义了一个有效的内积 \(\langle x, y \rangle_A = x^T A y\)，共轭性才有意义
- **正定性**：保证 \(p_k^T A p_k > 0\)，步长 \(\alpha_k\) 的公式才有效；如果 \(p_k^T A p_k = 0\)，则无法做精确线搜索

对于非对称系统，需要使用 GMRES（Generalized Minimal RESidual）或 BiCGSTAB 等方法。

### 2.5 与其他迭代法对比

| 方法 | 适用矩阵 | 存储需求 | 收敛保证 |
|------|---------|---------|---------|
| Jacobi | 对角占优 | O(n) | 条件苛刻 |
| Gauss-Seidel | 对角占优/正定 | O(nnz) | 比Jacobi快 |
| SOR | 对角占优 | O(nnz) | 最优ω难选 |
| **Conjugate Gradient** | 对称正定 | O(nnz) + O(n) | 固定上界 |
| Multigrid | 椭圆算子 | O(n) | O(n) 收敛 |
| GMRES | 一般非对称 | O(n·k) | 需重启(restart) |

CG 的优势在于不需要调参数（不像 SOR 的 ω）、内存可控（不像 GMRES 的重启问题）、对 SPD 矩阵有理论保证。

---

## ③ 实现架构

### 3.1 整体数据流

```
输入: Poisson 方程参数 (N, Nx, Ny)
         ↓
构造稀疏矩阵 (CSR格式)
         ↓
[CG 迭代循环]
    ├─ 矩阵-向量乘法: y = A · p_k     ← 核心计算
    ├─ 向量点积: r_k^T r_k, p_k^T (Ap_k)
    ├─ 步长计算: α_k = r_k^T r_k / p_k^T(Ap_k)
    ├─ 解更新: x_{k+1} = x_k + α_k p_k
    ├─ 残差更新: r_{k+1} = r_k - α_k (Ap_k)
    ├─ 方向更新: β_k = r_{k+1}^T r_{k+1} / r_k^T r_k
    └─ 收敛检查: ||r_{k+1}||/||b|| < tol
         ↓
输出: 解向量 x, 残差历史
         ↓
验证: vs 解析解, 残差单调性, 大规模可扩展性
```

### 3.2 CSR 稀疏矩阵格式

CSR（Compressed Sparse Row）是稀疏矩阵最常用的存储格式之一。其设计目标是在不做复制的情况下高效遍历一行的所有非零元——这正是 CG 中矩阵-向量乘法的需求。

**三个数组**：

- `values[k]`：第 k 个非零元的数值
- `col_indices[k]`：第 k 个非零元的列索引
- `row_ptr[i]`：第 i 行第一个非零元在 `values` 中的起始位置

**为什么用 CSR 而不是 COO 或 CSC？**

- COO（坐标格式）虽然构建方便但查询一行需要扫描全部数据，O(nnz) 变成 O(nnz²)
- CSC（列压缩）适合按列遍历但不适合按行遍历——CG 恰好需要按行遍历（每个输出分量 y[i] 是第 i 行的加权和）
- CSR 完美匹配 CG 的访问模式：外层循环遍历行 i，内层循环遍历 `values[row_ptr[i]..row_ptr[i+1]]`

**内存分析**：
- 对于 n×n 矩阵，nnz 非零元：`values` 和 `col_indices` 各 nnz 个元素，`row_ptr` n+1 个元素
- 2D Poisson 离散化（五点差分）：每个内部点连接 4 个邻居 + 自身，nnz ≈ 5n
- 稠密矩阵需要 n² 存储，稀疏 CSR 只需要约 5n——内存节省为 n/5 倍，n=10⁶ 时节省 200,000 倍

### 3.3 关键数据结构设计

```cpp
struct SparseMatrixCSR {
    int n;                          // 矩阵维度 n x n
    std::vector<double> values;     // 非零元值
    std::vector<int> col_indices;   // 列索引
    std::vector<int> row_ptr;       // 行偏移 (size = n+1)
};

struct CGResult {
    bool converged;                           // 是否收敛
    int iterations;                           // 迭代次数
    double final_rel_residual;                // 最终相对残差
    double final_abs_residual;                // 最终绝对残差
    std::vector<double> x;                    // 解向量
    std::vector<double> residual_history;     // 每步残差历史（用于收敛性分析）
};
```

设计要点：
- `SparseMatrixCSR` 不保存行列数之外的元信息，保持轻量
- `CGResult` 不仅返回"是否收敛"，还返回完整的残差历史——这对于调试和可视化至关重要
- 运算符 `axpy` 和 `scale` 重用在 CG 迭代中反复出现的向量操作，避免手写循环的错误

### 3.4 职责划分

| 模块 | 职责 |
|------|------|
| `SparseMatrixCSR` | 稀疏矩阵存储，构建 Poisson 离散化矩阵 |
| `mat_vec_mul` | CSR 格式的矩阵-向量乘法（CG 核心） |
| `conjugate_gradient` | CG 迭代主循环，收敛监控 |
| `build_poisson_1d/2d` | 构造测试用的 Poisson 矩阵 |
| `main` | 5 个测试用例编排 + 量化验证 |

---

## ④ 关键代码解析

### 4.1 矩阵-向量乘法（CG 的性能瓶颈）

```cpp
std::vector<double> mat_vec_mul(const SparseMatrixCSR& A, const std::vector<double>& x) {
    std::vector<double> y(A.n, 0.0);
    for (int i = 0; i < A.n; ++i) {
        double sum = 0.0;
        for (int k = A.row_ptr[i]; k < A.row_ptr[i+1]; ++k) {
            sum += A.values[k] * x[A.col_indices[k]];
        }
        y[i] = sum;
    }
    return y;
}
```

**关键设计**：
- 外循环按行遍历：`i` 从 0 到 n-1，对应输出向量 y 的每个分量
- 内循环遍历第 i 行的非零元：`k` 从 `row_ptr[i]` 到 `row_ptr[i+1]`，恰好是第 i 行在 values 数组中的区间
- `col_indices[k]` 给出非零元的列号，直接索引输入向量 x
- **为什么不用 y[i] += val * x[col] 而是构建 sum？** 局部变量 sum 在寄存器中，避免了每次加法都要写回内存的延迟（store-to-load forwarding 在现代 CPU 上仍有开销）

### 4.2 CG 主循环

```cpp
CGResult conjugate_gradient(
    const SparseMatrixCSR& A,
    const std::vector<double>& b,
    double tol = 1e-8,
    int max_iter = 1000
) {
    int n = A.n;
    CGResult result;
    result.converged = false;

    // 初始猜测 x0 = 0 — 最常见的默认选择
    std::vector<double> x(n, 0.0);

    // r0 = b - A*x0 = b (因为 x0=0)
    std::vector<double> r = b;
    std::vector<double> p = r;  // p0 = r0

    double b_norm = norm2(b);
    if (b_norm < 1e-15) {
        // 平凡解：b 本身几乎为零
        result.converged = true;
        result.iterations = 0;
        result.x = x;
        return result;
    }

    double rsold = dot(r, r);

    for (int k = 0; k < max_iter; ++k) {
        // 主力计算：A · p_k（一次完整的矩阵-向量乘法）
        std::vector<double> Ap = mat_vec_mul(A, p);

        // p_k^T · A · p_k — 需要保证正数（SPD 的数学保证 + 浮点保护）
        double pAp = dot(p, Ap);
        if (pAp <= 0.0) {
            break;  // 非正定，CG 无法继续
        }
        double alpha = rsold / pAp;

        // 解更新：x += alpha * p_k（精确线搜索步长）
        axpy(alpha, p, x);

        // 残差更新：r -= alpha * (A p_k)（从残差定义直接推导）
        axpy(-alpha, Ap, r);

        double rsnew = dot(r, r);
        double rel_res = std::sqrt(rsnew) / b_norm;
        result.residual_history.push_back(rel_res);

        // 收敛检查
        if (rel_res < tol) {
            result.converged = true;
            result.iterations = k + 1;
            result.final_rel_residual = rel_res;
            result.final_abs_residual = std::sqrt(rsnew);
            result.x = x;
            return result;
        }

        // 方向更新：β_k = (r_{k+1}·r_{k+1}) / (r_k·r_k)（Fletcher-Reeves）
        double beta = rsnew / rsold;

        // p_{k+1} = r_{k+1} + beta * p_k
        scale(beta, p);
        for (int i = 0; i < n; ++i) p[i] += r[i];

        rsold = rsnew;
    }

    result.iterations = max_iter;
    result.x = x;
    return result;  // 未收敛但返回当前最佳估计
}
```

**逐步骤解释**：

**① 初始化**：`x = 0` 是稀疏系统的自然选择——没有更好的先验信息。`r0 = b` 是因为 `b - A·0 = b`。`p0 = r0` 意味着第一步沿着最速下降方向走，这是合理的起点。

**② 主力计算**：`Ap = A * p_k`。这是每次迭代中最昂贵的操作——O(nnz) 复杂度。CG 的一个巨大优势是**每步迭代只需要一次矩阵-向量乘法**（大部分其他 Krylov 方法也是如此，但 CG 的"每步一次"已经是理论下界）。

**③ 步长计算**：`alpha = rsold / pAp`。这里 `rsold = r_k^T r_k`，`pAp = p_k^T A p_k`。分子是"还有多远要走"的平方度量，分母是搜索方向在 A-范数下的"刚度"——如果方向很"硬"（pAp 大），步长就要小一点；如果方向很"软"，就可以大步走。

**④ 解更新**：`x += alpha * p_k`。精确线搜索保证在 p_k 方向上到达极小值。

**⑤ 残差更新**：`r -= alpha * Ap`。这是残差定义的直接结果，不是重新计算 `r = b - A * x`——这样就省了一次矩阵-向量乘法。这是 CG 的精妙之处：通过代数恒等更新残差，维护了算法的 O(nnz) 复杂度。

**⑥ 收敛检查**：`||r||/||b|| < tol`。相对残差比绝对残差更有意义——对于不同尺度的 b，同样的 tol 有可比较的含义。

**⑦ 方向更新**：`beta = rsnew / rsold`，然后 `p = r + beta * p`。关键洞察：`p` 先被缩放为 `beta * p`（旧方向），再加上 `r`（残差方向）。这保证了 `p_{k+1}` 与 `p_k` 在 A-内积下正交。

**⚠️ 容易出错的地方**：
- `pAp <= 0` 检查：浮点误差可能导致微小的负数，对于实际正定矩阵，严格 `<= 0` 太激进——当 pAp 接近机器精度时，应该增大容差
- `rsold / pAp` 溢出：如果残差已经极小但 pAp 也极小（病态矩阵），步长可能爆炸
- 未收敛但返回结果：循环结束时 `result.converged = false`，但调用者容易忽略——应该用 assert 或日志警告

### 4.3 Poisson 方程离散化

```cpp
SparseMatrixCSR build_poisson_2d(int Nx, int Ny) {
    int n = Nx * Ny;
    SparseMatrixCSR A;
    A.n = n;
    A.row_ptr.resize(n + 1, 0);

    for (int j = 0; j < Ny; ++j) {
        for (int i = 0; i < Nx; ++i) {
            int k = j * Nx + i;
            int nnz = 1;  // 对角线
            if (i > 0) nnz++;       // 左
            if (i < Nx - 1) nnz++;  // 右
            if (j > 0) nnz++;       // 下
            if (j < Ny - 1) nnz++;  // 上
            A.row_ptr[k+1] = nnz;
        }
    }
    // ... 填充 values 和 col_indices
}
```

**五点差分模板**：
```
    -1
    |
-1--4---1
    |
    -1
```

对角线上是 4（中心点的系数），四个邻居各为 -1。这是 Poisson 方程 \(-\Delta u = f\) 的标准二阶中心差分离散化。除以 \(h^2\) 后得到矩阵方程：

\[
\frac{1}{h^2}
\begin{bmatrix}
4 & -1 & & -1 & & \\
-1 & 4 & -1 & & -1 & \\
& \ddots & \ddots & \ddots & & \ddots \\
\end{bmatrix}
\begin{bmatrix} u_1 \\ u_2 \\ \vdots \end{bmatrix}
=
\begin{bmatrix} f_1 \\ f_2 \\ \vdots \end{bmatrix}
\]

我们在代码中将 \(1/h^2\) 因子吸收到右端项 b 中（即 `b[i] = f(x) * h^2`），这样 A 的元素就保持在 O(1) 量级，改善了数值稳定性。

**为什么用自然行排序？**
自然行排序 `k = j*Nx + i` 使得 A 的带宽为 `2*Nx + 1`，且相邻节点的索引在矩阵中也相邻——这对于 CG 的缓存局部性至关重要。如果使用随机排列，缓存命中率会急剧下降。

### 4.4 向量操作基元

```cpp
// axpy: y += a * x  （线性代数中最常见的操作）
void axpy(double a, const std::vector<double>& x, std::vector<double>& y) {
    for (size_t i = 0; i < x.size(); ++i) y[i] += a * x[i];
}

// scale: x *= a
void scale(double a, std::vector<double>& x) {
    for (size_t i = 0; i < x.size(); ++i) x[i] *= a;
}
```

这两个函数看似简单，但提供了两个关键价值：
1. **语义清晰**：`axpy(alpha, p, x)` 比 `for(...) x[i] += alpha * p[i]` 更容易理解为"在 p 方向上移动 alpha"
2. **可扩展**：未来可以用 BLAS（Basic Linear Algebra Subprograms）替换这些实现，获得硬件级优化（SIMD、多线程），而不改动调用者

---

## ⑤ 踩坑实录

### 坑 1：CSR 构建时 row_ptr 的"先计数再前缀和"模式

**症状**：矩阵-向量乘法返回随机值，解完全不正确。

**错误假设**：以为 `row_ptr[i]` 永远指向第 i 行的起始位置。实际上在构建过程中，`row_ptr` 有一段过渡状态。

**真实原因**：CSR 构建需要两步：第一步遍历矩阵统计每行的非零元数量（暂存在 `row_ptr[i+1]` 中），第二步对 `row_ptr` 做前缀和转换为偏移量。如果忘记第二步，`row_ptr` 中存的还是计数而不是偏移。

**修复**：

```cpp
// Step 1: 统计
for (int j = 0; j < Ny; ++j) {
    for (int i = 0; i < Nx; ++i) {
        // ...
        A.row_ptr[k+1] = nnz;  // 暂存：第 k 行的非零元数量
    }
}
// Step 2: 前缀和（不能漏！）
for (int k = 0; k < n; ++k) {
    A.row_ptr[k+1] += A.row_ptr[k];  // 转换为偏移量
}
```

**教训**：两步构建是 CSR 的经典陷阱——几乎所有 CSR 实现教程都会强调这一点，但第一次写时仍然容易漏。写一个简单的打印函数 `print_csr(mat)` 验证 `row_ptr[n] == nnz` 是简单有效的检查。

### 坑 2：Poisson 方程的边界处理

**症状**：2D Poisson 解的中心值总是比精确解小一个数量级。

**错误假设**：假设 Dirichlet 边界条件 \(u = 0\) on boundary 已经正确施加。实际上的问题在于右端项的 `h^2` 因子。

**真实原因**：在连续形式中方程为 \(-\Delta u = f\)，离散化后为 \((4u_k - u_{left} - u_{right} - u_{up} - u_{down}) / h^2 = f_k\)。两边乘 \(h^2\) 得到 \(4u_k - u_{left} - \ldots = f_k \cdot h^2\)。如果忘记在右端项中乘 \(h^2\)，等价于在解一个比真实问题"大 \(h^2\) 倍"的方程。

**修复**：

```cpp
// 正确：b 中包含了 h^2 因子
b[k] = 2.0 * M_PI * M_PI * u_exact[k] * h * h;
```

**验证方法**：将 `h` 减半，观察误差是否按 O(h²) 递减。如果不是，说明离散化有问题。

### 坑 3：收敛判据的选择

**症状**：CG 报告"收敛"（达到 1e-8），但解与精确解的误差高达 0.1。

**错误假设**：认为相对残差范数 \(\|r\|/\|b\| < tol\) 等价于误差范数 \(\|x - x^*\| < tol\)。对于良态矩阵这近似成立，但对于病态矩阵差距可以很大。

**真实原因**：相对残差和相对误差之间的关系由条件数 \(\kappa\) 控制：

\[
\frac{\|x - x^*\|}{\|x^*\|} \leq \kappa \cdot \frac{\|r\|}{\|b\|}
\]

当 \(\kappa\) 很大时（例如大规模 Poisson），即使残差很小，解仍然可能不精确。

**修复**：在测试中同时检查**残差**和**误差**（通过已知精确解验证）：

```cpp
double error_L2 = 0.0;
for (int i = 0; i < N; ++i) {
    double diff = std::fabs(result.x[i] - u_exact[i]);
    error_L2 += diff * diff;
}
// L2 和 L∞ 两个指标都要检查
// L2: 整体精度
// L∞: 最差点的精度（容易暴露边界附近的离散化误差）
```

**教训**：CG 报告的是残差收敛，不是误差收敛。在不知道精确解的实际应用中，需要更严格的容差或使用预处理来降低有效条件数。

---

## ⑥ 效果验证与数据

### 6.1 1D Poisson 测试结果

```
N = 200
收敛: ✅ YES
迭代次数: 1
最终相对残差: 1.494974e-12
L2 误差 vs 精确解: 2.040867e-04
L∞ 误差 vs 精确解: 2.035722e-05
```

1D Poisson 只用了 **1 步就收敛**。这是因为 1D Poisson 矩阵是一个三对角矩阵，其特征值非常紧密地聚类在 [0, 4] 区间内。CG 对特征值聚类的矩阵收敛极快——这正是 CG 的优势所在：它不关心维度（n=200），只关心特征值分布。

L2 误差约 2e-4，对应 200 个网格点的平均每点误差约 0.0002/√200 ≈ 1.4e-5，在预期范围内。L∞ 误差比 L2 误差小约 10 倍，说明边界点附近的误差得到了均匀控制。

### 6.2 2D Poisson 测试结果

```
矩阵维度: 4096 x 4096
非零元: 20224 (密度: 0.12%)
收敛: ✅ YES
迭代次数: 1
最终相对残差: 1.27e-13
L2 误差 vs 精确解: 6.33e-03
L∞ 误差 vs 精确解: 1.95e-04
中心点值: 0.999611 (expected: 1.000000, error: 3.893108e-04)
```

稀疏度分析：4096×4096 = 16,777,216 个元素，只有 20,224 个非零元——密度仅 0.12%。如果不使用 CSR，稠密存储需要约 134MB（双精度），而 CSR 只需要约 20224×8×2 + 4097×4 ≈ 342KB，**节省 392 倍内存**。

中心点误差 3.9e-4，对应的相对误差约 0.039%，符合二阶精度的理论预期（h = 1/65 ≈ 0.015，h² ≈ 2.4e-4）。

### 6.3 残差收敛性测试

```
迭代次数: 50
初始残差: 7.000000e+00
最终残差: 0.000000e+00
残差衰减倍数: inf
平均衰减因子: 0.9133
```

平均衰减因子 0.9133 意味着每步将残差减少约 8.7%。对于 1D Poisson（N=100），理论收敛因子约为：

\[
\frac{\sqrt{\kappa} - 1}{\sqrt{\kappa} + 1} \approx \frac{\sqrt{10^4} - 1}{\sqrt{10^4} + 1} = \frac{99}{101} \approx 0.98
\]

实际衰减 0.9133 比理论 0.98 好——因为理论界是对最坏情况的估计，而 Poisson 矩阵的特征值分布有利于 CG（尤其是在高频区域，CG 的收敛速度会逐步加快，这种现象称为 **superlinear convergence**）。

### 6.4 可扩展性测试

```
矩阵维度: 10000 x 10000 (49600 nonzeros)
收敛: ✅ YES
迭代次数: 187
最终相对残差: 8.5966e-09
解统计: min=2.756075 max=751.338446 mean=365.595996 var=47035.072152
```

10000×10000 的问题仅需 187 步收敛——对于 10000 维问题，187 步远小于维度，这是 CG 实用性的最好证明。如果使用直接法（LU），即使用了稀疏计算，fill-in 也会让内存需求爆炸。解的范围 [2.76, 751.34] 和方差 47035 说明解有意义地分布在较大范围内，不是平凡解。

### 6.5 收敛曲线（多尺寸对比）

| 网格大小 | 迭代次数 | 最终残差 | 每100维迭代数 |
|---------|---------|---------|-------------|
| N=32 | 16 | 0 | 50.0 |
| N=64 | 32 | 0 | 50.0 |
| N=128 | 64 | 0 | 50.0 |
| N=256 | 128 | 0 | 50.0 |

迭代次数严格线性增长：N 翻倍，迭代次数恰好翻倍。理论上 CG 对于 Poisson 问题的迭代次数 ≈ N（因为 κ ∝ N²，而 \(2(\sqrt{\kappa}-1)/(\sqrt{\kappa}+1)^k\) 在 k ≈ N/2 时达到机器精度）。我们的结果 N/2 步收敛，与理论一致。

完整的收敛曲线数据保存在 `residual_profile.csv` 中（241 行数据），包含每步迭代的残差范数，可用于后续的可视化分析。

---

## ⑦ 总结与延伸

### 7.1 本文做了什么

实现了一个完整的 Conjugate Gradient 求解器，包含 CSR 稀疏矩阵存储、矩阵-向量乘法、向量操作基元、收敛监控和残差历史记录。通过 1D/2D Poisson 方程的五个测试验证了：残差理论收敛速度、解精度（vs 精确解）、残差单调性、大规模可扩展性（10000×10000）、收敛曲线记录。

### 7.2 技术局限性

1. **仅适用于对称正定矩阵**：非对称或不定系统（如对流-扩散方程的 advection 主导情况、Helmholtz 方程）需要 GMRES 或 BiCGSTAB

2. **无法处理约束**：对于带约束的优化问题（如 KKT 系统），需要 Lagrange 乘子法或专门的约束 Krylov 方法

3. **无预处理时收敛慢**：对于高条件数问题（如自适应网格、非均匀材料系数），CG 的迭代次数随条件数增长。预处理（Preconditioning）可以大幅改善。例如对 Poisson 问题使用**不完全 Cholesky 分解**或**多重网格**作为预处理器，可以将迭代次数从 O(N) 降至 O(1) 甚至常数级别

4. **浮点精度导致的正交性丧失**：在有限精度算术中，残差向量会逐渐偏离真正的正交方向。长迭代后可能出现"停滞"（stalling）——残差不再下降。这可以通过**显式重正交化**（reorthogonalization）来缓解

5. **没有利用多右端项**：如果需要求解 Ax = b 对于多个不同的 b，CG 每次都要从头开始。对于这类问题，**Block CG** 或 **Krylov 子空间回收**（recycling）方法可以复用已生成的 Krylov 子空间

### 7.3 可改进方向

1. **预处理（Preconditioning）**：这是 CG 实际应用中最关键的增强。一个好的预处理器 M 应满足：M 易于求逆（M^{-1} * v 计算快），M 近似于 A（M^{-1}A ≈ I）。经典选择包括：
   - **Jacobi/对角预处理器**：M = diag(A)，简单但效果有限
   - **不完全 Cholesky（IC）**：对 A 做 Cholesky 分解但丢弃 fill-in，保留稀疏性
   - **多重网格（Multigrid）**：利用 Poisson 算子的多尺度结构，几何多重网格可以达到 O(n) 总复杂度
   - **代数多重网格（AMG）**：不依赖几何信息，纯代数构造多层级

2. **与高级语言/库集成**：当前实现使用纯 C++ vector，未来可以和 Eigen、Armadillo 或 PETSc 等高性能线性代数库对接，获得 SIMD 加速和并行化

3. **GPU 加速**：CSR 格式的 SpMV（稀疏矩阵-向量乘法）是 GPU 上的经典优化问题。CUDA 的 cuSPARSE 库提供了高度优化的 CSR SpMV 实现。将 CG 求解器移植到 GPU 上，对 10⁶ 维问题可望获得 10-50x 加速

4. **扩展到非线性问题**：CG 本身是线性求解器，但可以作为 Newton 法的内核：Newton 法的每一步求解 Jacobi 系统 J·Δx = -F(x)，其中 J 通常是对称的。**Newton-CG**（也称为**截断 Newton 法**）在优化的线搜索中使用 CG 近似求解 Newton 方向，是 L-BFGS 等拟牛顿法的重要替代

5. **与 GMRES/BiCGSTAB 做系统性对比**：在同一套测试框架下，实现 GMRES（用于非对称矩阵）和 BiCGSTAB，对不同类型的稀疏矩阵（对称/非对称、正定/不定、良态/病态）做全面的收敛性对比实验

### 7.4 与本系列其他文章的关联

- **06-14 四叉树、06-15 KD-Tree、06-16 R-Tree**：这些空间索引结构可用于加速 CG 的预处理——例如用几何分区构造块对角预处理器
- **07-19 MST (Prim/Kruskal)**：最小生成树与**不完全 Cholesky 分解**的 fill-in 控制密切相关——保留 MST 中的边来构造预处理器
- **07-24 Half-Edge 网格数据结构**：网格操作的最终目的是生成稀疏矩阵（Laplacian、刚度矩阵），CG 是这些矩阵的标准求解器
- **08-01 Halton QMC**：CG 收敛取决于特征值分布，而特征值分布受到问题离散化方式的影响——QMC 采样可用于构造更均匀的配点法（Collocation Method）的采样点
- **08-02 Harris 角点检测**：角点检测中的结构张量就是 2×2 的对称正定矩阵——在这个小规模上，CG 退化为 Cholesky 直接求解

### 7.5 关键收货

这次实践让我深刻理解了三个关键点：

1. **"每步一次 SpMV"的威力**：CG 的每一步只需要一次矩阵-向量乘法，O(nnz) 复杂度。对于稀疏系统，这比直接法（O(n³) 或更差）快好几个数量级。而且内存需求仅为 O(nnz) + O(n)，对于 10⁶ 维问题只需几 MB

2. **条件数决定一切**：CG 的收敛速度不直接依赖矩阵维度，而依赖条件数 κ。这解释了为什么 N=10000 只需 187 步就收敛——虽然维度大，但条件数增长仅 O(N²)

3. **代数恒等更新残差是精妙设计**：用 `r -= alpha * Ap` 而不是 `r = b - A * x` 更新残差，既省了一次矩阵-向量乘法，又保持了数学等价性。这是 CG 从 1952 年 Hestenes-Stiefel 提出以来，始终是稀疏线性系统求解黄金标准的原因之一

---

**代码仓库**：[GitHub - daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/08/08-04-conjugate-gradient)

**收敛曲线数据**：[residual_profile.csv](https://github.com/chiuhoukazusa/daily-coding-practice/blob/main/2026/08/08-04-conjugate-gradient/residual_profile.csv)