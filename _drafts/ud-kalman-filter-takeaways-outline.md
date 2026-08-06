# UD Kalman Filter 对话总结提纲

> 状态：草稿提纲，仅用于后续扩写。文件位于 `_drafts`，并提交在独立分支 `agent/udkf-takeaways-outline`，不进入 Pages 发布分支。

## 一、文章目标与主线

### 1. 核心问题

- UD 形式与常规协方差形式在数学上是否等价。
- 为什么 UD 形式在有限精度、定点和低精度浮点下通常更稳定。
- `Cov` 数组中对角线存 \(D\)、上三角存 \(U\) 时，如何理解和重构真实协方差 \(P\)。
- UD time update 如何处理状态转移 \(\Phi\) 和过程噪声 \(Q\)。
- Agee–Turner rank-one 中的“rank-one”是什么意思。
- 如何把 \(Q\) 拆成若干 rank-one 项，尤其比较 \(Q_{ENU}\) 与 \(Q_{XYZ}\) 的实现代价。
- 原始 Agee–Turner、modified Agee–Turner 和 Bierman UD measurement update 的关系、区别和历史顺序。
- 从 UD 紧凑存储中删除一个状态时，为什么不能直接删除 \(U,D\) 的一行一列，以及为什么只有删除位置之前的因子需要重新更新。

### 2. 总体结论

- UD-KF 与常规矩阵 KF 在精确算术下完全等价。
- UD 的优势主要不是改变滤波理论，而是改变数值计算路径。
- measurement update 中，UD 形式把新息方差写成非负项累加，减少交叉项抵消。
- time update 中，UD 形式把完整协方差传播改造成因子传播、重新三角化和过程噪声注入。
- Agee–Turner 适合正 rank-one 加法，典型用于 time update 的 \(Q\) 注入。
- Bierman 标量 UD measurement update 使用针对 Kalman 负 rank-one 更新重新组织的稳定递推；现代资料常将其核心归入 modified Agee–Turner 一类。
- 删除状态时，对完整 \(P\) 删除对应行列是正确的；对 UD 因子直接删除行列则会错误丢失该 UD 随机方向对其他状态的协方差贡献。

---

## 二、UD 分解的基本定义与存储形式

### 1. 基本分解

采用约定：

\[
P=UDU^T,
\]

其中：

- \(U\) 为单位上三角矩阵；
- \(D=\operatorname{diag}(d_0,\ldots,d_{n-1})\)；
- \(d_i\ge 0\) 时，重构的 \(P\) 保持半正定。

### 2. 紧凑存储

一个二维数组 `Cov` 可同时存储：

- `Cov[i][i] = D_i`；
- `Cov[i][j] = U_{ij}`，其中 \(i<j\)；
- \(U_{ii}=1\) 不单独存储；
- 下三角通常不使用。

### 3. 必须避免的误解

- `Cov[i][j]` 不等于 \(P_{ij}\)。
- 对角线中的 \(D_i\) 通常也不等于真实方差 \(P_{ii}\)。
- 状态间相关性并非只存于某一个 \(U_{ij}\)，而是分散编码在多个后续 UD 方向中。
- UD 因子编号表示的是递归分解中的随机方向，不能简单视为原协方差矩阵中的独立行列。

---

## 三、为什么重构 \(P_{ij}\) 时要累加“后面的元素”

### 1. 元素公式

由矩阵乘法：

\[
P_{ij}=\sum_{k=0}^{n-1}U_{ik}d_kU_{jk}.
\]

由于 \(U\) 为上三角矩阵：

\[
U_{ik}=0\quad(k<i),\qquad U_{jk}=0\quad(k<j),
\]

因此：

\[
\boxed{
P_{ij}=\sum_{k=\max(i,j)}^{n-1}U_{ik}d_kU_{jk}
}
\]

若只计算上三角，即 \(i\le j\)：

\[
\boxed{
P_{ij}=\sum_{k=j}^{n-1}U_{ik}d_kU_{jk}
}
\]

### 2. 对角元素

\[
\boxed{
P_{ii}=d_i+\sum_{k=i+1}^{n-1}U_{ik}^2d_k
}
\]

所以一般有：

\[
D_i\ne P_{ii}.
\]

### 3. 三维例子

若：

\[
U=
\begin{bmatrix}
1&u_{01}&u_{02}\\
0&1&u_{12}\\
0&0&1
\end{bmatrix},
\]

则：

\[
P_{01}=u_{01}d_1+u_{02}u_{12}d_2.
\]

第二项说明：第 0、1 个状态共同受第 2 个 UD 方向影响，因此还要累加第 \(j\) 列之后的共同方向。

### 4. 一句话理解

\[
\boxed{
P_{ij}\text{ 是 }U\text{ 的第 }i,j\text{ 行在所有共同 UD 方向上的加权内积}
}
\]

---

## 四、标量 measurement update 中的新息方差 \(S\)

### 1. 常规形式

标量观测：

\[
z=hx+v,\qquad \operatorname{Var}(v)=r.
\]

新息方差：

\[
S=hPh^T+r.
\]

展开后：

\[
hPh^T=\sum_i\sum_j h_iP_{ij}h_j.
\]

非对角项允许为正或负，因此中间项可能出现大数正负抵消。

### 2. UD 形式

代入 \(P=UDU^T\)，定义：

\[
w=U^Th^T.
\]

则：

\[
\boxed{
S=r+w^TDw=r+\sum_i d_iw_i^2
}
\]

只要 \(d_i\ge0\)，每一项都非负。

### 3. 数值稳定性 takeaway

常规展开的求和条件可写为：

\[
\kappa_{sum}
=
\frac{\sum_k|a_k|}{|\sum_ka_k|}.
\]

若正负项严重抵消，则 \(\kappa_{sum}\gg1\)。

UD 最终求和中：

\[
b_i=d_iw_i^2\ge0,
\]

所以该累加阶段的求和条件数为 1。

核心结论：

\[
\boxed{
UD\text{ 消除了计算 }w^TDw\text{ 最终累加阶段的抵消}
}
\]

### 4. 必要限定

UD 并非消除一切数值抵消，因为计算：

\[
w=U^Th^T
\]

时仍可能发生正负项抵消。UD 的优势主要体现在：

- 最终二次型由非负项组成；
- 直接维护半正定因子；
- 避免完整协方差更新中的大矩阵相减；
- 更适合定点、单精度和有限字长实现。

---

## 五、UD time update 的本质

### 1. 常规协方差传播

\[
P_k^-=\Phi P_{k-1}^+\Phi^T+Q.
\]

normal form 的主要计算量通常在：

\[
\Phi P\Phi^T.
\]

而 \(Q\) 只需最后做矩阵加法。

### 2. UD 形式代入

\[
P^+=U^+D^+U^{+T},
\]

所以：

\[
P^-=(\Phi U^+)D^+(\Phi U^+)^T+Q.
\]

令：

\[
A=\Phi U^+.
\]

由于 \(A\) 一般不再是单位上三角矩阵，不能直接令 \(U^-=\Phi U^+\)。必须重新三角化或重新因子化。

### 3. 计算量“转移”的准确含义

不能简单说：

\[
\Phi\text{ 的计算转移给了 }Q.
\]

更准确是：

\[
\boxed{
\text{完整协方差矩阵传播}
\rightarrow
\text{因子传播、重新三角化和过程噪声注入}
}
\]

当 \(\Phi\) 很稀疏、结构简单，而 \(Q\) 需要拆成多个 rank-one 项时，运行时间会表现为主要消耗在 \(Q\) 注入。

### 4. GNSS 状态模型中的典型情况

GNSS CV 状态转移通常含：

- 位置—速度块；
- 钟差—钟漂块；
- 大量单位阵和零元素。

因此 \(\Phi U\) 可利用结构化行操作，不需要通用矩阵乘法。此时过程噪声方向数量、相关性和注入方法更容易成为主要成本。

---

## 六、Agee–Turner 中 Rank-one 的含义

### 1. 标准问题

已知：

\[
P=UDU^T,
\]

希望直接得到：

\[
\boxed{
P_{new}=P+caa^T
}
\]

对应的新 \(U,D\)。

### 2. 为什么 \(aa^T\) 是 rank-one

矩阵第 \(j\) 列为：

\[
(aa^T)_{:,j}=a_ja.
\]

所有列都是同一个向量 \(a\) 的倍数，因此：

\[
\operatorname{rank}(aa^T)=1\qquad(a\ne0).
\]

### 3. 不是“只更新一个元素”

即使 \(aa^T\) 的全部元素都非零，它仍只有一个独立方向。

\[
\boxed{
\text{Rank-one 描述新增随机方向的数量，而不是被修改矩阵元素的数量}
}
\]

### 4. 随机过程解释

若一个标量噪声 \(w\) 通过向量 \(a\) 作用于状态：

\[
\delta x=aw,\qquad \operatorname{Var}(w)=c,
\]

则：

\[
\operatorname{Cov}(\delta x)=caa^T.
\]

所以一次 rank-one update 对应一个独立标量随机源。

---

## 七、time update 中如何把 \(Q\) 拆成 rank-one 项

### 1. 一般形式

若：

\[
Q=GQ_wG^T,
\]

且：

\[
Q_w=\operatorname{diag}(q_1,\ldots,q_m),
\]

令 \(g_\ell\) 为 \(G\) 的第 \(\ell\) 列，则：

\[
\boxed{
Q=\sum_{\ell=1}^m q_\ell g_\ell g_\ell^T
}
\]

每一项调用一次原始 Agee–Turner positive rank-one update。

### 2. 一般稠密 \(Q\)

可先分解：

\[
Q=U_qD_qU_q^T
=\sum_i d_i^{(q)}u_i^{(q)}u_i^{(q)T}.
\]

实际需要的 rank-one 次数取决于 \(Q\) 的有效秩，而不一定等于状态维数。

### 3. 关键实现思想

不要先把 \(Q\) 当作完整矩阵；优先从独立过程噪声方向出发构造：

\[
\boxed{
Q=\sum_\ell c_\ell a_\ell a_\ell^T
}
\]

---

## 八、CV 模型中两种常见 \(Q\) 假设

### 1. 采样间隔内随机常加速度

状态：

\[
x=[p^T,v^T]^T.
\]

单轴噪声映射：

\[
a=
\begin{bmatrix}
\frac12\Delta t^2\\
\Delta t
\end{bmatrix}.
\]

单轴过程噪声：

\[
\boxed{
Q_{axis}=q_a aa^T
}
\]

其秩为 1，因此三轴共需 3 次 rank-one update。

### 2. 连续白噪声加速度

单轴离散 \(Q\)：

\[
Q_{axis}=q
\begin{bmatrix}
\frac{\Delta t^3}{3}&\frac{\Delta t^2}{2}\\
\frac{\Delta t^2}{2}&\Delta t
\end{bmatrix}.
\]

其行列式：

\[
\frac{\Delta t^4}{12}>0,
\]

所以单轴为 rank-two。

可拆为：

\[
\begin{bmatrix}
\frac{\Delta t^3}{3}&\frac{\Delta t^2}{2}\\
\frac{\Delta t^2}{2}&\Delta t
\end{bmatrix}
=
\frac{\Delta t^3}{12}
\begin{bmatrix}1\\0\end{bmatrix}
\begin{bmatrix}1&0\end{bmatrix}
+
\Delta t
\begin{bmatrix}\frac{\Delta t}{2}\\1\end{bmatrix}
\begin{bmatrix}\frac{\Delta t}{2}&1\end{bmatrix}.
\]

因此三轴共需 6 次 rank-one update。

### 3. 必须强调

以下两个模型不能混用：

- 随机常加速度：单轴 rank-one；
- 连续白噪声加速度：单轴 rank-two。

它们的 \(\Delta t\) 幂次和物理假设不同。

---

## 九、\(Q_{ENU}\) 与 \(Q_{XYZ}\) 的构造和代价比较

### 1. ENU 中的噪声定义

\[
\Sigma_a^{enu}=\operatorname{diag}(q_E,q_N,q_U).
\]

ENU 方向彼此独立，完整矩阵通常稀疏，rank-one 向量也很稀疏。

### 2. ENU 到 ECEF XYZ 的旋转

令：

\[
C=C_{xyz\leftarrow enu}=[e_{xyz},n_{xyz},u_{xyz}],
\]

位置和速度同时旋转：

\[
T=\begin{bmatrix}C&0\\0&C\end{bmatrix}.
\]

于是：

\[
Q_{xyz}=TQ_{enu}T^T.
\]

### 3. 不应显式形成完整 \(Q_{xyz}\)

若：

\[
Q_{enu}=\sum_\ell c_\ell a_\ell^{enu}a_\ell^{enuT},
\]

则：

\[
\boxed{
Q_{xyz}=\sum_\ell c_\ell
(Ta_\ell^{enu})(Ta_\ell^{enu})^T
}
\]

只需旋转每个 rank-one 方向：

\[
a_\ell^{xyz}=Ta_\ell^{enu}.
\]

### 4. Rank-one 调用次数比较

坐标旋转不改变秩，所以：

| 模型 | ENU | XYZ |
|---|---:|---:|
| 随机常加速度 | 3 次 | 3 次 |
| 连续白噪声加速度 | 6 次 | 6 次 |

### 5. 实际代价差异

- ENU 向量更稀疏，例如一个方向只影响对应位置和速度两个元素；
- 转到 XYZ 后，一个局部方向通常在 \(X,Y,Z\) 三轴都有非零分量；
- 若 Agee–Turner 实现能够利用零元素，ENU 状态下可更省；
- 若实现总是遍历完整状态，主要成本仍约为 \(O(mn^2)\)，两者差异不大；
- XYZ 的额外旋转代价通常远小于 rank-one 更新本身。

### 6. 各向同性特例

若：

\[
q_E=q_N=q_U=q,
\]

则：

\[
C(qI)C^T=qI.
\]

因此各向同性噪声不需要 ENU 到 XYZ 旋转，可直接沿 XYZ 三轴注入。

### 7. 推荐实现

对于 ECEF 状态、ENU 噪声参数：

1. 计算当地 E/N/U 在 XYZ 中的三个方向向量；
2. 按随机过程模型构造 rank-one 向量；
3. 直接调用 Agee–Turner；
4. 不显式构造和分解完整 \(Q_{xyz}\)。

---

## 十、原始 Agee–Turner

### 1. 适用问题

\[
P_{new}=P+caa^T,\qquad c>0.
\]

典型应用：time update 中加入过程噪声。

### 2. 递推特点

- 通常按上三角结构从后向前消元；
- 直接修改 \(U,D\)；
- 不重构完整 \(P\)；
- 正 rank-one 加法时容易保持 \(d_i\ge0\)。

### 3. 不适合直接做什么

不推荐简单将 \(c\) 改成负数，用于：

\[
P-|c|aa^T.
\]

原因是可能出现：

\[
d_j^+=d_j-|C_j|a_j^2,
\]

即两个接近正数的直接相减，导致严重消减误差和负 \(D\)。

---

## 十一、Modified Agee–Turner 的改进

### 1. 针对标量 Kalman measurement downdate

标量更新：

\[
P^+=P-\frac{Ph^ThP}{S},
\qquad
S=hPh^T+r.
\]

定义：

\[
w=U^Th^T.
\]

则：

\[
S=r+\sum_j d_jw_j^2.
\]

### 2. 部分创新方差

定义：

\[
\alpha_j=r+\sum_{i=1}^j d_iw_i^2.
\]

递推：

\[
\alpha_j=\alpha_{j-1}+d_jw_j^2.
\]

### 3. \(D\) 的稳定缩放

\[
\boxed{
d_j^+=d_j\frac{\alpha_{j-1}}{\alpha_j}
}
\]

因为：

\[
0<\alpha_{j-1}\le\alpha_j,
\]

所以：

\[
0\le d_j^+\le d_j.
\]

这比直接做正数相减更稳定，也符合观测后方差收缩的物理意义。

### 4. 主要改进

- 将危险的负 rank-one 直接相减重写为正数累加和比例缩放；
- 在同一递推中可同时得到 \(S\)、未归一化增益和 \(U,D\)；
- 提供清晰的数值不变量：
  - \(\alpha_j>0\)；
  - \(\alpha_j\ge\alpha_{j-1}\)；
  - \(0\le d_j^+\le d_j\)。

### 5. 实现注意事项

- 必须先计算 \(w=U^Th^T\)，不能直接用 \(h\)；
- 原地更新 \(U\) 时，递推增益必须使用旧列；
- 覆盖 \(D_j\) 前，应保存旧 \(d_j\)；
- 循环方向通常与原始 positive Agee–Turner 不同。

---

## 十二、Bierman UD measurement update

### 1. 定位

Bierman 给出的是完整的标量 UD Kalman 测量更新流程，而不仅是一个抽象 rank-one 因子修改接口。

典型流程包括：

1. 计算残差；
2. 计算 \(w=U^Th^T\)；
3. 计算 \(v=Dw\)；
4. 非负项累加得到 \(S\)；
5. 更新 \(D\)；
6. 更新 \(U\)；
7. 递推未归一化 Kalman gain；
8. 更新状态。

### 2. 与 modified Agee–Turner 的关系

对于标量、最优 Kalman measurement update：

\[
\boxed{
\text{两者通常不是竞争算法，而是“更新内核”和“完整流程”的关系}
}
\]

- modified Agee–Turner：强调特殊负 rank-one 的稳定因子更新；
- Bierman UD measurement update：将该类稳定递推与新息、增益和状态更新融合。

若两份实现采用相同的分解约定和递推公式，数值结果应只差浮点舍入。

### 3. 工程选择

对于 GNSS 逐标量伪距、多普勒更新，推荐 Bierman 融合式实现，因为：

- 不重构 \(P\)；
- 不显式形成 \(Ph^T\)；
- \(S\) 由非负项累加；
- 可直接输出标准化残差所需的 \(S\)；
- 避免中间量重复存储和重复遍历。

---

## 十三、删除一个状态：为什么不能直接删除 UD 的一行一列

### 1. 先区分“边缘化”与“条件化”

设要删除第 \(r\) 个状态。

若目标是保留其余状态的**边缘协方差**，则对完整协方差矩阵的正确操作是删除第 \(r\) 行和第 \(r\) 列：

\[
P_{\setminus r}=J_rPJ_r^T,
\]

其中 \(J_r\) 是删除第 \(r\) 个坐标的选择矩阵。

这不同于“已知 \(x_r\) 的精确值并对其余状态进行条件化”。后者对应 Schur 补：

\[
P_{aa\mid r}=P_{aa}-P_{ar}P_{rr}^{-1}P_{ra}.
\]

本节讨论的是前一种：从滤波状态中移除一个状态，但保留其余状态原有的边缘不确定性。

### 2. 将 UD 分解写成 rank-one 项之和

令 \(u_k\) 为 \(U\) 的第 \(k\) 列，则：

\[
\boxed{
P=\sum_{k=0}^{n-1}d_ku_ku_k^T
}
\]

第 \(r\) 列为：

\[
u_r=
\begin{bmatrix}
U_{0r}\\
U_{1r}\\
\vdots\\
U_{r-1,r}\\
1\\
0\\
\vdots\\
0
\end{bmatrix}.
\]

所以 \(d_r\) 对完整协方差的贡献不是只有 \(P_{rr}\)，而是：

\[
d_ru_ru_r^T.
\]

它同时影响所有 \(i,j\le r\) 的协方差元素。

### 3. 直接删除 UD 行列会丢掉什么

对完整 \(P\) 删除状态坐标后，第 \(r\) 个随机方向仍对前序状态产生贡献：

\[
d_r(J_ru_r)(J_ru_r)^T.
\]

但若直接删除 \(U\) 的第 \(r\) 行、第 \(r\) 列以及 \(D_r\)，就把整个第 \(r\) 个 rank-one 方向删除了。

设直接压缩后得到 \(\widetilde U,\widetilde D\)，并定义：

\[
a=J_ru_r
=
\begin{bmatrix}
U_{0r}\\
\vdots\\
U_{r-1,r}\\
0\\
\vdots\\
0
\end{bmatrix}.
\]

那么正确的边缘协方差满足：

\[
\boxed{
P_{\setminus r}
=
\widetilde U\widetilde D\widetilde U^T
+d_raa^T
}
\]

因此，正确实现是：

1. 保存 \(d_r\) 和第 \(r\) 列中 \(U_{0r},\ldots,U_{r-1,r}\)；
2. 删除并压缩 UD 存储中的第 \(r\) 行、第 \(r\) 列；
3. 将丢失的 \(d_raa^T\) 作为一次 positive rank-one 项加回；
4. 使用原始 Agee–Turner 更新压缩后的 UD 因子。

### 4. 为什么只有 \(i,j<r\) 的区域受影响

因为 \(U\) 为上三角矩阵：

\[
U_{ir}=0\qquad(i>r).
\]

删除第 \(r\) 个坐标后，向量 \(a=J_ru_r\) 只有原索引小于 \(r\) 的分量可能非零：

\[
a_i=
\begin{cases}
U_{ir},&i<r,\\
0,&i>r.
\end{cases}
\]

因此补偿项：

\[
\Delta P=d_raa^T
\]

满足：

\[
\Delta P_{ij}=d_ra_ia_j.
\]

只有当：

\[
i<r,\qquad j<r
\]

时才可能非零。

所以：

- 原索引 \(i,j<r\) 的左上区域需要重新吸收 \(d_r\) 的贡献；
- 与后续状态的交叉块不会因该补偿项改变；
- 原索引大于 \(r\) 的 UD 元素只需移动到压缩后的索引，不需要重新计算其数值。

### 5. 从 Agee–Turner 循环看为何后续列不变

压缩后的 rank-one 向量 \(a\) 在新索引 \(j\ge r\) 处均为 0。

原始 Agee–Turner 从后向前处理各列时，对于这些后续列：

- \(a_j=0\)；
- \(D_j\) 的修正项为 0；
- \(U_{ij}\) 的修正项也为 0；
- 工作向量不会因该列发生变化。

算法只有进入 \(j<r\) 的前序列后才产生实际更新。因此“只有 \(i,j<r\) 受影响”不仅来自矩阵结构，也能直接从 rank-one 更新循环看出来。

### 6. 三维例子

设：

\[
U=
\begin{bmatrix}
1&u_{01}&u_{02}\\
0&1&u_{12}\\
0&0&1
\end{bmatrix},
\qquad
D=\operatorname{diag}(d_0,d_1,d_2).
\]

删除状态 \(r=1\) 后，正确的边缘协方差为：

\[
P_{\setminus1}=
\begin{bmatrix}
d_0+u_{01}^2d_1+u_{02}^2d_2 & u_{02}d_2\\
u_{02}d_2 & d_2
\end{bmatrix}.
\]

若直接删除 UD 的第 1 行、第 1 列，只能得到：

\[
\widetilde U\widetilde D\widetilde U^T=
\begin{bmatrix}
d_0+u_{02}^2d_2 & u_{02}d_2\\
u_{02}d_2 & d_2
\end{bmatrix}.
\]

缺少的正是：

\[
d_1
\begin{bmatrix}u_{01}\\0\end{bmatrix}
\begin{bmatrix}u_{01}&0\end{bmatrix}.
\]

因此只需对删除位置之前的状态执行一次 positive rank-one 补偿。

### 7. 随机变量解释

可写：

\[
\delta x=U\xi,
\qquad
\operatorname{Cov}(\xi)=D.
\]

第 \(r\) 个独立随机分量 \(\xi_r\) 会影响：

\[
\delta x_0,\ldots,\delta x_{r-1},\delta x_r,
\]

但不会影响：

\[
\delta x_{r+1},\ldots,\delta x_{n-1}.
\]

删除状态坐标 \(x_r\)，并不表示 \(\xi_r\) 对前面状态造成的不确定性也应该消失。该贡献必须重新吸收到前序 UD 因子中。

### 8. 依赖分解约定

“只有删除位置之前的因子受影响”依赖于：

\[
P=UDU^T,
\qquad U\text{ 为单位上三角矩阵}.
\]

若代码采用：

\[
P=U^TDU,
\]

或使用下三角 \(L\)，受影响的方向可能相反。实现前必须先确认分解约定。

---

## 十四、Agee–Turner、Carlson、Bierman 的历史顺序

### 1. Agee–Turner

- 1972 年；
- 研究正定矩阵加一个 symmetric dyad 后的三角分解更新；
- 建立一般 positive rank-one UD 更新框架。

### 2. Carlson

- 1973 年；
- 发展三角平方根滤波和稳定的相关更新思想；
- 后世一些文献将 Carlson update 与 modified Agee–Turner 类更新并列讨论。

### 3. Bierman

- 工作形成于 1975 年前后，正式论文发表于 1976 年；
- 将 UD 因子化系统地组织为完整标量 Kalman measurement update；
- 明确融合新息方差、增益、因子和状态更新。

### 4. 发展脉络

\[
\boxed{
\text{Agee–Turner 1972}
\rightarrow
\text{Carlson 1973}
\rightarrow
\text{Bierman 1975/1976}
}
\]

---

## 十五、算法选型总表

| 场景 | 推荐算法 | 原因 |
|---|---|---|
| time update 中加入 \(qgg^T\), \(q\ge0\) | 原始 Agee–Turner | 标准 positive rank-one addition |
| 一般 \(Q\) 的低秩注入 | \(Q\) 分解后多次原始 Agee–Turner | 利用有效秩和独立噪声方向 |
| 标量 measurement update | Bierman integrated UD update | 同时稳定更新 \(S,K,U,D,x\) |
| 仅需要协方差因子的特殊负更新内核 | modified Agee–Turner | 利用 Kalman 负 rank-one 特殊结构 |
| 将原始 Agee–Turner 的 \(c\) 直接设为负 | 不推荐 | 直接相减可能破坏数值稳定性和正定性 |
| ECEF 状态、ENU 各向异性噪声 | 旋转 rank-one 方向后注入 | 不形成完整稠密 \(Q_{xyz}\) |
| 各向同性三轴噪声 | 直接沿 XYZ 三轴注入 | 旋转后仍为 \(qI\) |
| 删除第 \(r\) 个状态并保留边缘协方差 | 删除/压缩后补回 \(d_raa^T\) | 直接删除 UD 行列会丢失第 \(r\) 个方向对前序状态的贡献 |
| 已知被删状态精确值并条件化 | Schur 补或等价测量更新 | 与简单边缘化不是同一操作 |

---

## 十六、实现与调试检查清单

### 1. 分解约定

确认代码采用哪一种：

\[
P=UDU^T
\]

还是：

\[
P=U^TDU.
\]

两者对应的 \(H\) 变换、循环方向、状态删除方向和重构公式不同。

### 2. 协方差重构验证

随机抽取 \(i,j\)，比较完整重构结果与：

\[
\sum_{k=\max(i,j)}^{n-1}U_{ik}d_kU_{jk}.
\]

### 3. 新息方差交叉验证

对同一先验 \(P\)：

\[
S_{normal}=hPh^T+r,
\]

\[
S_{UD}=r+(U^Th^T)^TD(U^Th^T).
\]

两者应在舍入误差范围内一致。

### 4. Measurement update 不变量

每一步检查：

\[
\alpha_j>0,
\qquad
\alpha_j\ge\alpha_{j-1},
\qquad
0\le d_j^+\le d_j.
\]

### 5. Time update 不变量

- positive rank-one 注入后 \(d_i\ge0\)；
- rank-one 项数量应与噪声有效秩一致；
- ENU 到 XYZ 旋转不应改变理论秩；
- 连续白噪声模型不能误用成单轴一次 rank-one。

### 6. 状态删除验证

对随机正定 \(P\) 进行测试：

1. 对完整 \(P\) 直接删除第 \(r\) 行列，得到参考主子矩阵；
2. 对 UD 存储执行“压缩 + \(d_raa^T\) rank-one 补偿”；
3. 重构新的 \(P\)；
4. 比较两者误差。

并检查：

- 补偿向量在原索引 \(i>r\) 处应为 0；
- 后续 UD 列数值应保持不变，仅索引前移；
- 只有前 \(r\) 个状态的 UD 前导块发生数值更新；
- 删除后所有 \(d_i\) 仍应非负。

### 7. 原地覆盖风险

- 更新 \(U\) 前保留旧列；
- 更新 \(D_j\) 前保留旧值；
- 明确工作向量在每一步代表 \(k_{j-1}\) 还是 \(k_j\)；
- 删除状态前必须先保存 \(d_r\) 和 \(U_{0:r-1,r}\)；
- 定点实现中为平方和累加器保留保护位。

---

## 十七、最终 takeaway

1. **UD-KF 改变的是数值实现，不是 Kalman 理论。**
2. **UD 数组中的 \(U,D\) 是协方差因子，不是协方差元素本身。**
3. **真实 \(P_{ij}\) 必须累加两个状态共享的全部后续 UD 方向。**
4. **标量 \(S\) 在 UD 形式下变成非负项累加，显著降低最终求和的抵消风险。**
5. **time update 的核心从完整矩阵传播变为因子传播、重新三角化和 \(Q\) 注入。**
6. **rank-one 表示只有一个独立随机方向，不表示只改一个矩阵元素。**
7. **构造 \(Q\) 时应优先识别独立噪声方向，而不是先形成完整矩阵。**
8. **ENU 转 XYZ 不改变秩；旋转 rank-one 向量比构造稠密 \(Q_{xyz}\) 更合理。**
9. **随机常加速度单轴 rank-one，连续白噪声加速度单轴 rank-two。**
10. **原始 Agee–Turner 用于 positive rank-one；modified/Bierman 型递推用于稳定的标量 measurement downdate。**
11. **Bierman 更适合作为完整 GNSS 标量 measurement update 的工程实现。**
12. **对完整 \(P\) 删除状态行列是边缘化；对 UD 因子直接删除行列则会丢失一个仍影响其他状态的随机方向。**
13. **在 \(P=UDU^T\) 的上三角约定下，删除第 \(r\) 个状态只需把 \(d_ru_ru_r^T\) 在前序状态中的残余贡献重新吸收，因此只有 \(i,j<r\) 的前导块发生数值变化。**
14. **算法名称不如具体公式重要：应检查分解约定、循环方向、\(\alpha\) 递推和原地覆盖顺序。**

---

## 十八、后续扩写建议

### 1. 第一篇：UD 分解与协方差重构

- \(P=UDU^T\) 的随机方向解释；
- 紧凑存储；
- \(P_{ij}\) 重构；
- 与 LDL\(^T\)、Cholesky 的关系。

### 2. 第二篇：UD measurement update

- \(S\) 的非负求和；
- Bierman 标量递推；
- modified Agee–Turner 的稳定性来源；
- 与 normal/Joseph form 的数值实验。

### 3. 第三篇：UD time update

- \(\Phi U\) 的结构化传播；
- weighted Gram–Schmidt 或 Thornton 型 time update；
- Agee–Turner positive rank-one 注入；
- 稠密 \(Q\)、低秩 \(Q\)、分块 \(Q\) 的实现选择。

### 4. 第四篇：GNSS 中的过程噪声建模

- CV、随机常加速度、连续白噪声加速度；
- ECEF/ENU 转换；
- 水平/垂直各向异性；
- 多系统钟差、共享钟漂与互相关 \(Q\)；
- 计算量和定点实现对比。

### 5. 第五篇：动态状态管理

- UD 状态增广；
- UD 状态删除与边缘化；
- 边缘化和条件化的区别；
- 状态重排对 UD 稀疏性和计算量的影响；
- GNSS 卫星钟差、模糊度或动态 ISB 状态的增删策略。