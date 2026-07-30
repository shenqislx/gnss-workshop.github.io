---
title: "多接收机钟差过程噪声的互相关：非 PSD 的 Q 如何搞垮 GNSS KF"
layout: article
date: 2026-07-30
category: "GNSS 定位算法"
author: "Andy"
summary: "从 GNSS KF 的 CV 模型出发推导过程噪声 Q，说明多系统/多接收机钟差共享晶振时互相关项不可省略，以及非正半定 Q 对滤波器的破坏；并给出单钟差+ISB、UDKF 多向量同时更新等规避方案。"
published: false
---

> 草稿：多系统或多接收机状态下，钟差过程噪声经常被写成对角阵。这篇文章先把 CV 模型的 Q 怎么来的说清楚，再说明共享晶振带来的互相关为什么不能省，以及 Q 一旦失去正半定性，KF 会怎样坏掉。

# 多接收机钟差过程噪声的互相关：非 PSD 的 Q 如何搞垮 GNSS KF

多系统 GNSS 滤波里，状态向量经常会挂上好几个接收机钟差：GPS 钟、Galileo 钟、BDS 钟，有时再加 GLONASS 频间偏差。工程实现里，最常见的偷懒写法是给每个钟差单独塞一个对角过程噪声：

$$
Q_{\mathrm{clk}}=\operatorname{diag}(q_G,q_E,q_C,\dots)
$$

看起来干净，也容易调。但只要这些钟差背后共享同一块接收机晶振，这个对角阵在物理上就不成立。更糟的是，一旦有人试图“补一点互相关”，却把相关系数写坏，得到的 $Q$ 不再正半定（non-PSD），Time Update 会把协方差矩阵一步步腐蚀掉，最后出现负方差、Cholesky 失败、增益异常，甚至直接 NaN。

这篇文章做三件事：

1. 推导 GNSS KF 里常用 CV 模型的过程噪声 $Q$。
2. 说明多钟差状态下，互相关项为什么不能忽略，以及非 PSD 的 $Q$ 如何破坏滤波器。
3. 给出几类工程上可落地的规避方案：单钟差 + ISB 重参数化、保证 PSD 的相关 $Q$ 构造，以及 UDKF 多向量同时更新等。

## 1. 先把 CV 模型的 Q 说清楚

以位置速度状态为例：

$$
x=
\begin{bmatrix}
p \\
v
\end{bmatrix}
$$

CV（Constant Velocity）模型假设速度近似常值，加速度当作过程噪声。连续时间形式可以写成：

$$
\dot{p}=v,\qquad \dot{v}=w(t)
$$

其中 $w(t)$ 是功率谱密度为 $q_a$ 的白噪声：

$$
E[w(t)w(\tau)]=q_a\,\delta(t-\tau)
$$

对应状态空间：

$$
\dot{x}=A x + G w,\quad
A=\begin{bmatrix}0&1\\0&0\end{bmatrix},\quad
G=\begin{bmatrix}0\\1\end{bmatrix}
$$

离散化后，状态转移矩阵是：

$$
F=e^{A\Delta t}=\begin{bmatrix}1&\Delta t\\0&1\end{bmatrix}
$$

过程噪声协方差由连续时间谱密度积分得到：

$$
Q=\int_0^{\Delta t} F(\tau)\,G\,q_a\,G^T F(\tau)^T\,d\tau
=q_a
\begin{bmatrix}
\dfrac{\Delta t^3}{3} & \dfrac{\Delta t^2}{2} \\
\dfrac{\Delta t^2}{2} & \Delta t
\end{bmatrix}
$$

三维位置速度只要对每个轴套同样结构，再 Kronecker 到 $I_3$ 即可。注意这里 $Q$ 本身就不是对角阵：位置噪声和速度噪声是相关的。这不是“可选优化项”，而是白噪声加速度假设下的必然结果。若只保留对角、丢掉交叉项，离散化后的噪声模型已经和连续模型不一致。

KF 的预测步：

$$
x_k^-=F x_{k-1}^+
$$

$$
P_k^-=F P_{k-1}^+ F^T + Q
$$

要求 $Q\succeq 0$。只要 $Q$ 正半定，$P^-$ 在精确算术下也会保持正半定；一旦 $Q$ 不是 PSD，后面的 $P$ 就不再有协方差的几何意义。

## 2. 接收机钟差：从单钟到双状态时钟

GNSS 伪距观测里，接收机钟差通常以 $c\delta t$ 进入状态。最简单的模型是随机游走：

$$
(c\delta t)_k=(c\delta t)_{k-1}+w_k,\quad
Q_{\delta t}=q_{\delta t}\Delta t
$$

更贴近晶振物理的做法，是双状态时钟：钟差 + 钟漂：

$$
x_{\mathrm{clk}}=
\begin{bmatrix}
c\delta t \\
c\dot{\delta t}
\end{bmatrix}
$$

其离散转移和 CV 同构：

$$
F_{\mathrm{clk}}=
\begin{bmatrix}
1 & \Delta t \\
0 & 1
\end{bmatrix}
$$

若连续时间噪声分别对应白频噪声谱密度 $S_g$ 和随机游走频噪声谱密度 $S_b$，则：

$$
Q_{\mathrm{clk}}=
\begin{bmatrix}
S_g\Delta t+S_b\dfrac{\Delta t^3}{3} &
S_b\dfrac{\Delta t^2}{2} \\
S_b\dfrac{\Delta t^2}{2} &
S_b\Delta t
\end{bmatrix}
$$

$S_g$、$S_b$ 可从 Allan 方差参数映射，例如常见近似：

$$
S_g\approx\frac{h_0}{2},\qquad
S_b\approx 2\pi^2 h_{-2}
$$

这里同样可以看到：钟差与钟漂的交叉项来自同一组驱动噪声，不能随手删掉。

到目前为止，$Q$ 都还容易保持 PSD，因为它们是从谱密度积分出来的，形式上天然是 Gram 矩阵。

## 3. 多钟差状态：互相关从哪里来

麻烦出在“多个钟差”同时进状态。多系统单点定位里，常见状态是：

$$
x=
\begin{bmatrix}
p & v &
c\delta t_G &
c\delta t_E &
c\delta t_C &
\dots
\end{bmatrix}^T
$$

但接收机通常只有一块主晶振。各系统“钟差”并不是彼此独立的振荡源，而是：

$$
\begin{aligned}
c\delta t_G &= c\delta t + b_G \\
c\delta t_E &= c\delta t + b_E \\
c\delta t_C &= c\delta t + b_C
\end{aligned}
$$

其中 $c\delta t$ 是公共接收机钟差，$b_E$、$b_C$ 是相对 GPS 的系统间偏差 ISB。ISB 通常慢变，过程噪声远小于公共钟差。

写成线性变换：

$$
\begin{bmatrix}
c\delta t_G \\
c\delta t_E \\
c\delta t_C
\end{bmatrix}
=
\begin{bmatrix}
1 & 0 & 0 \\
1 & 1 & 0 \\
1 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
c\delta t \\
\mathrm{ISB}_E \\
\mathrm{ISB}_C
\end{bmatrix}
=:T\,y
$$

若参数空间过程噪声为：

$$
Q_y=\operatorname{diag}(q_{\delta t},\,q_{\mathrm{ISB}_E},\,q_{\mathrm{ISB}_C})
$$

则绝对钟差空间的过程噪声必须是：

$$
Q_x=T Q_y T^T
=
\begin{bmatrix}
q_{\delta t} & q_{\delta t} & q_{\delta t} \\
q_{\delta t} & q_{\delta t}+q_{\mathrm{ISB}_E} & q_{\delta t} \\
q_{\delta t} & q_{\delta t} & q_{\delta t}+q_{\mathrm{ISB}_C}
\end{bmatrix}
$$

互相关项 $q_{\delta t}$ 不是装饰，它表达的是：三个钟差里那一大块共同漂移来自同一块晶振。

如果把 $Q_x$ 强行对角化成 $\operatorname{diag}(q,q,q)$，等于假设三块独立时钟。后果很直接：

- ISB 会被当成 $\sqrt{2q}$ 量级的随机游走，系统间偏差无端发散。
- 公共钟的可观性被拆碎，钟差估计互相拉扯。
- 位置解通过几何与钟差耦合，被错误的时间过程噪声间接污染。

多接收机联合滤波也是同一类问题。若若干天线/板卡共享同源时钟，或软件接收机里多通道共用同一时间基准，对应钟差状态的驱动噪声高度相关；忽略交叉项，就是在用错误的随机模型做 Time Update。

## 4. 非 PSD 的 Q：比“模型不准”更致命

忽略互相关，至少还可能得到一个 PSD 的对角 $Q$，滤波器“还能跑”，只是结果偏了。真正会把程序打崩的，是构造出一个非正半定的 $Q$。

常见踩坑方式：

1. **相关系数越界**  
   想表达强相关，写成
   $$
   Q=\sigma^2
   \begin{bmatrix}
   1 & \rho \\
   \rho & 1
   \end{bmatrix}
   $$
   却让 $|\rho|>1$。特征值出现负数，$Q\nsucceq 0$。

2. **只抄交叉项，不改对角线**  
   从 $T Q_y T^T$ 里“记得要加互相关”，却把对角线仍写成很小的 ISB 噪声，交叉项却用很大的 $q_{\delta t}$。例如
   $$
   Q=
   \begin{bmatrix}
   0.01 & 10 \\
   10 & 0.01
   \end{bmatrix}
   $$
   行列式为负，立刻非 PSD。

3. **离散化后再手工拼块**  
   位置 CV 的 $Q$、钟差 $Q$、多系统块各自推导后再拼接，单位不一致（米、米/秒、$c\delta t$ 秒或米混用），缩放错误也会破坏半定性。

4. **从相关观测反推过程噪声**  
   用经验相关阵去“修” $Q$，却没有保证对称化和特征值裁剪。

KF 预测

$$
P^-=FPF^T+Q
$$

在 $Q$ 有负特征值时，即使 $FPF^T$ 正定，$P^-$ 也可能失去正定性。后续测量更新

$$
K=P^-H^T(HP^-H^T+R)^{-1}
$$

会在新息协方差、增益和

$$
P^+=(I-KH)P^-
$$

上连锁放大。工程上常见症状：

- 某个状态方差变成负数。
- Joseph 形式也救不回来，因为病根在 $Q$。
- Cholesky / UD 分解直接失败。
- NIS、残差判据全面失真，FDE 跟着误判。
- 长时间运行后出现 NaN，且很难从哪一历元的观测异常反查。

平方根滤波对此更“敏感”：SRKF、UDKF 在注入过程噪声时往往需要对 $Q$ 做分解。$Q$ 非 PSD 时，分解阶段就会报错。这其实是好事——它比协方差形式默默算到崩溃更早暴露模型错误。

一句话：**模型不准会让估计变差；非 PSD 的 $Q$ 会让滤波器不再是滤波器。**

## 5. 如何正确构造多钟差 Q

原则很简单：先在物理独立的噪声源上定义 PSD 谱密度，再线性映射到状态空间。

以“公共钟 + ISB”为例：

$$
Q_y=\operatorname{diag}(q_{\delta t},q_{\mathrm{ISB}_E},q_{\mathrm{ISB}_C})\succeq 0
$$

$$
Q_x=T Q_y T^T\succeq 0
$$

任意 $T$ 下，这种 Gram 形式自动保证 PSD。若钟差还带钟漂，把 $y$ 扩成

$$
y=
\begin{bmatrix}
c\delta t &
c\dot{\delta t} &
\mathrm{ISB}_E &
\mathrm{ISB}_C
\end{bmatrix}^T
$$

对公共钟部分用第 2 节的双状态 $Q_{\mathrm{clk}}$，对 ISB 用小随机游走，再经 $T$ 映射即可。

数值上建议：

- 组装后检查 $\mathrm{eig}(Q)\ge -\epsilon$，必要时做对称化 $(Q+Q^T)/2$。
- 若必须裁剪负特征值，记录告警——说明建模路径有 bug，不应当作常规调参。
- 优先在 $y$ 空间调 $q_{\delta t}$ 和 $q_{\mathrm{ISB}}$，不要直接拧 $Q_x$ 的某个 off-diagonal。

## 6. 规避钟差互相关项的几类方案

互相关项“不能忽略”，不等于必须在绝对钟差参数化里硬扛一个稠密 $Q$ 块。更稳的做法是换参数、换更新顺序，或换滤波器形式，让问题在结构上消失。

### 6.1 单钟差 + ISB 重参数化

这是最推荐的默认方案。状态直接取：

$$
x=
\begin{bmatrix}
p & v &
c\delta t &
\mathrm{ISB}_E &
\mathrm{ISB}_C &
\dots
\end{bmatrix}^T
$$

过程噪声近似块对角：

$$
Q=\mathrm{blkdiag}(Q_{pv},\,Q_{\delta t},\,q_{\mathrm{ISB}_E},\,q_{\mathrm{ISB}_C},\dots)
$$

观测方程里，GPS 伪距对 $c\delta t$ 的系数为 1；Galileo / BDS 伪距对 $c\delta t$ 和对应 ISB 的系数都是 1。物理相关被参数化吸收，Time Update 不再需要钟差之间的交叉项。

好处：

- $Q$ 稀疏、易保证 PSD。
- ISB 过程噪声可以单独设得很小，符合“慢变偏差”直觉。
- 调试时公共钟抖动和系统偏差漂移不会缠在一起。

代价是观测矩阵 $H$ 稍改一下，以及要注意某系统卫星短暂不可见时 ISB 的可观性。

### 6.2 星间单差 / 接收机间单差消钟

若应用允许：

- 星间单差可消接收机钟差；
- 站间单差可消卫星钟，并削弱部分公共误差；
- 双差进一步消接收机钟。

状态里不再保留多套绝对钟差，自然没有多钟 $Q$ 相关问题。RTK、相对定位、部分 PPP-AR 预处理都会走这条路。代价是基准星切换、参考站依赖，以及单站绝对时间信息的丧失。

### 6.3 保留绝对多钟，但只用映射生成相关 Q

如果软件接口已经固定为 $(c\delta t_G,c\delta t_E,\dots)$，不要手填相关阵，而是内部维护 $y$ 空间噪声，每次 Time Update 前计算 $Q_x=T Q_y T^T$。这和 6.1 数学等价，只是把重参数化藏在 $Q$ 构造里。

### 6.4 UDKF：多向量同时更新

嵌入式 GNSS 里常用 UDKF（Bierman / Thornton 的 UD 分解滤波）。它维护

$$
P=UDU^T
$$

过程噪声注入和观测更新都在 UD 因子上进行，数值上比直接改 $P$ 稳。

和本文相关的两点：

1. **过程噪声注入要求 $Q$ 可分解。**  
   非 PSD 的 $Q$ 会在 UD / 秩一更新阶段失败，从而强制暴露钟差相关建模错误，而不是悄默污染 $P$。

2. **相关观测不要拆成标量序贯更新后假装独立。**  
   若测量噪声 $R$ 有互相关，或一组观测需要同时约束多个钟差/ISB，应做成多维观测向量，一次做多向量更新；或先对 $R$ 做白化/Decorrelation，再序贯更新。把相关观测强行当独立标量一个个灌进去，等价于用错 $R$，和用错 $Q$ 一样会破坏一致性。

多系统同时更新时，一组伪距往往同时看见公共钟和多个 ISB。UDKF 下把这些观测作为向量更新，或按白化后的等价序贯更新，能避免“先更新 GPS 钟、再更新 Galileo 钟”造成的人为信息顺序偏差。

### 6.5 SRKF / SRIF：用平方根形式做护栏

SRKF 对 $Q$ 做 Cholesky，SRIF 从信息阵一侧递推。它们不能魔法修复错误模型，但能：

- 在 $Q$ 非 PSD 时尽早失败；
- 降低长时间运行中 $P$ 因舍入失去对称正定性的概率；
- 在状态维数升高后更适合做数值审计。

若当前实现仍是普通 KF，至少在调试版对 $Q$ 和 $P$ 做周期性 PSD 检查。

### 6.6 工程上的折中：相关钟差合并，独立偏差留下

并非所有“多钟”都必须强相关。例如：

- 同源晶振的多系统接收机钟：强相关，优先单钟 + ISB；
- 真正独立的外接原子钟 / 另一板卡本地钟：可以独立 $Q$；
- GLONASS IFB、频点间偏差、硬件延迟：通常慢变，单独小过程噪声，不要并进公共钟的大 $q$。

把“共享驱动”和“独立慢变偏差”分开，是避免稠密病态 $Q$ 的关键。

## 7. 一个最小对照

假设只有 GPS / Galileo 两个钟，公共钟噪声 $q_{\delta t}=10$，ISB 噪声 $q_{\mathrm{ISB}}=0.01$。

正确相关 $Q$：

$$
Q_{\mathrm{good}}=
\begin{bmatrix}
10 & 10 \\
10 & 10.01
\end{bmatrix}
\succeq 0
$$

错误对角 $Q$：

$$
Q_{\mathrm{diag}}=
\begin{bmatrix}
10 & 0 \\
0 & 10
\end{bmatrix}
\succeq 0
\quad\text{但物理错误}
$$

错误相关 $Q$：

$$
Q_{\mathrm{bad}}=
\begin{bmatrix}
0.01 & 10 \\
10 & 0.01
\end{bmatrix}
\nsucceq 0
$$

$Q_{\mathrm{diag}}$ 会让 ISB 假发散；$Q_{\mathrm{bad}}$ 会让滤波器数值解体。看起来“加了互相关”并不自动等于正确，**互相关必须和对角线一起从同一组噪声源映射出来**。

## 8. 小结

GNSS KF 的 CV / 双状态时钟模型里，$Q$ 的交叉项来自连续白噪声驱动的离散化，不是可有可无的细节。推到多系统、多接收机钟差时，只要共享晶振，绝对钟差空间的 $Q$ 必然带互相关。

忽略互相关，滤波器还能跑，但 ISB 和钟差随机模型是错的；胡乱填写互相关，一旦得到非 PSD 的 $Q$，预测协方差会失去正定性，KF 的增益、残差检验和数值分解会一起垮掉。

更稳妥的路径通常是：

1. 用谱密度积分构造单块运动/时钟 $Q$；
2. 状态尽量选单钟差 + ISB，让过程噪声保持块对角；
3. 若必须保留多绝对钟，用 $Q_x=T Q_y T^T$ 生成相关阵，并做 PSD 检查；
4. 在嵌入式实现里用 UDKF / SRKF，相关观测做多向量同时更新或先白化，既护数值，也逼建模错误早点暴露。

钟差互相关不是矩阵里的装饰元素。它要么被正确的参数化消掉，要么被正确的映射写进 $Q$；唯独不能假装它不存在，更不能写成一个非 PSD 的“相关阵”。
