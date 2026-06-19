---
title: "单频 GNSS 接收机电离层模型选型：基于单站单日数据的复算分析"
layout: article
date: 2026-06-19
category: "GNSS 定位算法"
author: "Andy"
summary: "从 Klobuchar 类广播模型和 SBAS IGP 格网模型两个维度，基于单站单日数据复算误差统计，讨论单频 GNSS 接收机电离层改正模型的工程选型方法。"
---

<style>
  .article-prose {
    --ion-ink: #20211d;
    --ion-muted: #686a61;
    --ion-soft: #f7f4ed;
    --ion-line: rgba(32, 33, 29, 0.13);
    --ion-blue: #3c6f8f;
    --ion-gold: #b6812e;
    --ion-olive: #64784f;
    --ion-rose: #a85b56;
    --ion-teal: #3f8377;
  }

  .article-prose .summary-panel,
  .article-prose .metric-card,
  .article-prose .chart-panel,
  .article-prose .method-panel {
    border: 1px solid var(--ion-line);
    border-radius: 8px;
    background: var(--ion-soft);
  }

  .article-prose .summary-panel {
    padding: 1rem 1.1rem;
    margin: 1.1rem 0 1.4rem;
  }

  .article-prose .summary-panel p {
    margin: 0.55rem 0;
  }

  .article-prose .summary-panel strong {
    color: var(--ion-blue);
  }

  .article-prose .metric-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(145px, 1fr));
    gap: 0.75rem;
    margin: 1rem 0 1.25rem;
  }

  .article-prose .metric-card {
    padding: 0.85rem 0.9rem;
  }

  .article-prose .metric-label {
    color: var(--ion-muted);
    font-size: 0.84rem;
  }

  .article-prose .metric-value {
    margin-top: 0.2rem;
    color: var(--ion-ink);
    font-size: 1.45rem;
    line-height: 1.1;
    font-weight: 800;
  }

  .article-prose .metric-note {
    margin-top: 0.28rem;
    color: var(--ion-muted);
    font-size: 0.82rem;
  }

  .article-prose .chart-panel {
    padding: 1rem;
    margin: 1.15rem 0 1.45rem;
    overflow-x: auto;
  }

  .article-prose .chart-title {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 1rem;
    margin-bottom: 0.85rem;
    color: var(--ion-ink);
    font-weight: 800;
  }

  .article-prose .chart-title span {
    color: var(--ion-muted);
    font-size: 0.84rem;
    font-weight: 500;
  }

  .article-prose .bar-row {
    display: grid;
    grid-template-columns: 92px minmax(260px, 1fr) 58px 58px;
    gap: 0.65rem;
    align-items: center;
    margin: 0.48rem 0;
    min-width: 520px;
    color: var(--ion-muted);
    font-size: 0.9rem;
  }

  .article-prose .bar-track {
    position: relative;
    height: 12px;
    border-radius: 999px;
    background: rgba(32, 33, 29, 0.08);
    overflow: hidden;
  }

  .article-prose .bar-fill {
    display: block;
    height: 100%;
    border-radius: 999px;
    background: var(--ion-blue);
  }

  .article-prose .bar-fill.alt {
    background: var(--ion-gold);
  }

  .article-prose .bar-value {
    color: var(--ion-ink);
    font-variant-numeric: tabular-nums;
    text-align: right;
  }

  .article-prose .legend {
    display: flex;
    flex-wrap: wrap;
    gap: 0.65rem 1rem;
    margin: 0.3rem 0 0.8rem;
    color: var(--ion-muted);
    font-size: 0.86rem;
  }

  .article-prose .legend i {
    display: inline-block;
    width: 0.8rem;
    height: 0.8rem;
    margin-right: 0.28rem;
    border-radius: 2px;
    vertical-align: -0.1rem;
  }

  .article-prose .utc-strip {
    display: grid;
    grid-template-columns: repeat(24, minmax(32px, 1fr));
    gap: 3px;
    min-width: 0;
  }

  .article-prose .utc-cell {
    border: 1px solid rgba(32, 33, 29, 0.1);
    border-radius: 6px;
    padding: 0.35rem 0.2rem;
    text-align: center;
    color: var(--ion-ink);
    font-size: 0.72rem;
    line-height: 1.15;
  }

  .article-prose .utc-cell b {
    display: block;
    margin-bottom: 0.16rem;
    color: var(--ion-muted);
    font-weight: 600;
  }

  .article-prose .gps { background: rgba(60, 111, 143, 0.17); }
  .article-prose .bds { background: rgba(100, 120, 79, 0.2); }
  .article-prose .wide { background: rgba(182, 129, 46, 0.2); }
  .article-prose .japan { background: rgba(168, 91, 86, 0.18); }

  .article-prose .heat-table td,
  .article-prose .heat-table th,
  .article-prose .decision-table td,
  .article-prose .decision-table th {
    white-space: nowrap;
  }

  .article-prose .heat-best {
    font-weight: 800;
    color: var(--ion-blue);
  }

  .article-prose .method-panel {
    padding: 0.9rem 1rem;
    margin: 1rem 0;
  }

  .article-prose .method-panel p {
    margin: 0.45rem 0;
  }

  .article-prose .caption {
    margin: 0.65rem 0 0;
    color: var(--ion-muted);
    font-size: 0.88rem;
  }

  .article-prose table {
    display: block;
    overflow-x: auto;
  }

  @media (max-width: 760px) {
    .article-prose .chart-panel {
      margin-left: -0.35rem;
      margin-right: -0.35rem;
      border-radius: 8px;
    }
  }
</style>

# 单频 GNSS 接收机电离层模型选型：基于单站单日数据的复算分析

单频 GNSS 接收机无法通过双频无电离层组合消除一阶电离层延迟，电离层残差会直接进入伪距观测方程。在电离层活动增强、低仰角观测占比升高、区域模型适配不足或格网改正可用性不稳定时，残余伪距误差达到米级并不罕见。电离层模型选型因此属于单频接收机定位算法中的基础工程问题：可获取性、误差分布、异常保护和连续性需要同时评估。

本文仅使用单测站、单日数据重新计算统计量，忽略原分析材料中的结论性描述。文中的结果不用于推出区域、季节或全场景通用结论，重点是给出一套可复用的分析方法，并说明这些数据对单频接收机选型有哪些可靠启示。

<div class="summary-panel" markdown="0">
  <p><strong>技术摘要。</strong>在该样本中，BDS Klobuchar-like 模型的全日 RMS、MAE 和 95% 分位绝对误差最低，平均偏差接近 0 m；QZS-Wide 在最低仰角和部分 UTC 小时表现更好，但整体存在约 -2.31 m 的相对负偏差；GPS 与 QZS-Japan 在若干时段仍可能成为小时最优模型，说明全日均值不能替代分时段评估。</p>
  <p><strong>对单频接收机的含义。</strong>Klobuchar 类模型适合作为连续基础改正。若接收机同时获得多套广播模型，模型选择应按测站区域、卫星系统、仰角、UTC 时段和残差监控结果进行配置或加权，不宜把单日总体排序固化为跨区域、跨日期的通用规则。</p>
  <p><strong>SBAS 格网模型。</strong>在定位结果中，DPS 的各自有效历元总体分布优于 SPS；同历元配对后，水平误差仍有改善，垂直均值略有退化，三维均值仅小幅改善。SBAS IGP 可作为精度增强候选，启用条件应包含可用性、完整性标志、残差一致性和回退策略。</p>
</div>

## 1. 分析口径与指标定义

Klobuchar 维度评估四套广播电离层模型：GPS、BDS、QZS-Wide、QZS-Japan。误差定义为模型给出的斜向电离层改正量减去 GIM 斜向参考量：

$$
e_{iono}=I_{model}^{slant}-I_{GIM}^{slant}
$$

GIM 是后处理参考产品，适合用于模型间相对比较，但不能等同于真实电离层延迟。下文中“偏小”“偏大”“接近无偏”均指相对该参考量而言。该部分每套模型包含 2,446,538 个样本，并按总体、仰角、UTC 小时、卫星系统、CN0 和相关性进行复算。

SBAS 维度评估 SPS 与 DPS 的定位误差。SPS 在这里代表使用 Klobuchar 类模型的单点定位结果，DPS 代表引入 SBAS IGP 格网电离层改正后的定位结果。SPS 有 86,373 个有效历元，DPS 有 60,510 个有效历元；由于 DPS 可用历元少于 SPS，定位收益需要同时查看“各自有效历元总体统计”和“同历元配对统计”。

<div class="metric-grid" markdown="0">
  <div class="metric-card">
    <div class="metric-label">Klobuchar 样本</div>
    <div class="metric-value">244.7万</div>
    <div class="metric-note">每套模型样本数</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">DPS 可用率</div>
    <div class="metric-value">70.1%</div>
    <div class="metric-note">相对 SPS 有效历元</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">配对历元</div>
    <div class="metric-value">60,510</div>
    <div class="metric-note">SPS 与 DPS 同时有效</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">UTC 覆盖</div>
    <div class="metric-value">24 h</div>
    <div class="metric-note">单日连续统计</div>
  </div>
</div>

## 2. 全日总体排序：BDS 的综合误差最低，QZS-Wide 的离散度最低

全日总体统计显示，BDS Klobuchar-like 的 RMS 为 2.66 m、MAE 为 2.17 m、95% 分位绝对误差为 5.35 m，均为四套模型中最低。它的平均偏差为 -0.03 m，说明该样本中 BDS 的全日正负误差抵消较充分。不过 BDS 的标准差为 2.66 m，高于其他模型，表明“均值接近 0 m”并不等于单历元误差小。

QZS-Wide 的 RMS 为 2.77 m，略高于 BDS，但标准差只有 1.54 m，是四套模型中最低；同时平均偏差为 -2.31 m，说明它在该测站该日呈现较稳定的相对负偏差。GPS 的平均偏差为 -2.07 m，RMS 为 2.92 m；QZS-Japan 的 RMS、MAE 和 95% 分位绝对误差最高。

<div class="chart-panel" markdown="0">
  <div class="chart-title">图 1  全日总体误差对比 <span>单位：m；条形为 RMS，右侧为 P95</span></div>
  <div class="bar-row">
    <div>BDS</div>
    <div class="bar-track"><span class="bar-fill" style="width: 40.9%"></span></div>
    <div class="bar-value">2.66</div>
    <div class="bar-value">5.35</div>
  </div>
  <div class="bar-row">
    <div>QZS-Wide</div>
    <div class="bar-track"><span class="bar-fill alt" style="width: 42.7%"></span></div>
    <div class="bar-value">2.77</div>
    <div class="bar-value">5.63</div>
  </div>
  <div class="bar-row">
    <div>GPS</div>
    <div class="bar-track"><span class="bar-fill" style="width: 45.0%"></span></div>
    <div class="bar-value">2.92</div>
    <div class="bar-value">6.06</div>
  </div>
  <div class="bar-row">
    <div>QZS-Japan</div>
    <div class="bar-track"><span class="bar-fill alt" style="width: 50.7%"></span></div>
    <div class="bar-value">3.30</div>
    <div class="bar-value">6.50</div>
  </div>
  <p class="caption">P95 为绝对误差 95% 分位。BDS 的综合分位误差最低，QZS-Wide 的误差离散度最低但偏差方向明显。</p>
</div>

| 模型 | Mean | STD | RMS | MAE | P50 | P95 |
|---|---:|---:|---:|---:|---:|---:|
| GPS | -2.07 | 2.07 | 2.92 | 2.32 | 1.94 | 6.06 |
| BDS | -0.03 | 2.66 | 2.66 | 2.17 | 1.90 | 5.35 |
| QZS-Wide | -2.31 | 1.54 | 2.77 | 2.31 | 1.93 | 5.63 |
| QZS-Japan | -2.82 | 1.71 | 3.30 | 2.82 | 2.27 | 6.50 |

**选型含义：**全日总体指标支持把 BDS 作为该测站该日的主要候选模型，但不能只看平均偏差。QZS-Wide 的低离散度适合进入候选集，尤其需要结合低仰角和分时段表现进一步判断。QZS-Japan 在全日总体上不占优，但分时段仍有局部表现，直接排除会损失对模型时变性的认识。

## 3. 仰角维度：低仰角与中高仰角呈现不同模型排序

按仰角分箱后，模型排序出现明显变化。0°-10° 仰角内，QZS-Wide 的 RMS 为 3.93 m，低于 GPS、BDS 和 QZS-Japan；从 10° 开始，BDS 在所有仰角分箱中取得最低 RMS。误差随仰角升高整体下降，符合斜向电离层延迟映射函数的基本规律。

| 仰角分箱 | GPS RMS | BDS RMS | QZS-Wide RMS | QZS-Japan RMS | 分箱最低 |
|---|---:|---:|---:|---:|---|
| 0°-10° | 4.34 | 4.67 | <span class="heat-best">3.93</span> | 4.61 | QZS-Wide |
| 10°-20° | 4.65 | <span class="heat-best">4.07</span> | 4.17 | 4.82 | BDS |
| 20°-30° | 3.46 | <span class="heat-best">3.04</span> | 3.27 | 3.88 | BDS |
| 30°-45° | 2.60 | <span class="heat-best">2.31</span> | 2.57 | 3.10 | BDS |
| 45°-60° | 2.09 | <span class="heat-best">1.81</span> | 2.12 | 2.58 | BDS |
| 60°-90° | 1.63 | <span class="heat-best">1.53</span> | 1.72 | 2.10 | BDS |

把卫星系统纳入分箱后，结论更细。BDS 在 18 个“卫星系统 × 仰角”分箱中的 13 个分箱 RMS 最低，按样本量加权覆盖约 742 万个模型-观测组合；QZS-Wide 在 GPS 与 BDS 卫星的 0°-20° 低仰角分箱占优；GPS 模型只在 GPS 卫星 45°-60° 分箱占优。

**选型含义：**仰角是 Klobuchar 类模型选型与定权的核心变量。该样本支持一种工程做法：中高仰角以 BDS 作为主要候选，低仰角保留 QZS-Wide 候选，并对低仰角观测持续采用更保守的随机模型。该做法仍需要在目标区域、多日和不同电离层活动水平下复核。

## 4. UTC 维度：全日排序会掩盖明显的时段差异

分 UTC 小时统计后，四套模型没有形成单一支配关系。BDS 在 12 个小时 RMS 最低，QZS-Wide 在 5 个小时最低，QZS-Japan 在 4 个小时最低，GPS 在 3 个小时最低。模型间 RMS 差值在 UTC 6-8 h 附近达到 3 m 以上，而在 UTC 15-19 h 多数小时收敛到 0.1 m 以内。

<div class="chart-panel" markdown="0">
  <div class="chart-title">图 2  每个 UTC 小时的最低 RMS 模型 <span>颜色表示该小时 RMS 最低的模型</span></div>
  <div class="legend">
    <span><i class="gps"></i>GPS</span>
    <span><i class="bds"></i>BDS</span>
    <span><i class="wide"></i>QZS-Wide</span>
    <span><i class="japan"></i>QZS-Japan</span>
  </div>
  <div class="utc-strip">
    <div class="utc-cell wide"><b>00</b>Wide</div>
    <div class="utc-cell wide"><b>01</b>Wide</div>
    <div class="utc-cell gps"><b>02</b>GPS</div>
    <div class="utc-cell wide"><b>03</b>Wide</div>
    <div class="utc-cell wide"><b>04</b>Wide</div>
    <div class="utc-cell gps"><b>05</b>GPS</div>
    <div class="utc-cell bds"><b>06</b>BDS</div>
    <div class="utc-cell bds"><b>07</b>BDS</div>
    <div class="utc-cell bds"><b>08</b>BDS</div>
    <div class="utc-cell bds"><b>09</b>BDS</div>
    <div class="utc-cell bds"><b>10</b>BDS</div>
    <div class="utc-cell bds"><b>11</b>BDS</div>
    <div class="utc-cell bds"><b>12</b>BDS</div>
    <div class="utc-cell japan"><b>13</b>JPN</div>
    <div class="utc-cell japan"><b>14</b>JPN</div>
    <div class="utc-cell bds"><b>15</b>BDS</div>
    <div class="utc-cell bds"><b>16</b>BDS</div>
    <div class="utc-cell gps"><b>17</b>GPS</div>
    <div class="utc-cell wide"><b>18</b>Wide</div>
    <div class="utc-cell bds"><b>19</b>BDS</div>
    <div class="utc-cell japan"><b>20</b>JPN</div>
    <div class="utc-cell japan"><b>21</b>JPN</div>
    <div class="utc-cell bds"><b>22</b>BDS</div>
    <div class="utc-cell bds"><b>23</b>BDS</div>
  </div>
  <p class="caption">UTC 小时统计按同一参考口径计算。该图强调模型表现的时变性，不用于制定固定 UTC 切换门限。</p>
</div>

| 统计项 | 结果 |
|---|---|
| 最低 RMS 小时数 | BDS 12 h，QZS-Wide 5 h，QZS-Japan 4 h，GPS 3 h |
| 最大模型间 RMS 差值 | UTC 7 h，约 3.81 m |
| 高差值时段 | UTC 6-8 h 差值均超过 3 m |
| 高收敛时段 | UTC 15-19 h 多数小时模型间差值低于 0.1 m |
| 单模型峰值 | QZS-Japan 在 UTC 7 h 达到 6.09 m；GPS 在 UTC 8 h 达到 4.63 m；QZS-Wide 在 UTC 7 h 达到 4.60 m；BDS 在 UTC 4 h 达到 3.95 m |

**选型含义：**UTC 维度对模型选型有实质影响。单频接收机若具备多模型输入，不宜只保存全日或全局模型排序；更稳妥的做法是把 UTC、本地时、电离层活动指标、仰角和卫星系统纳入模型残差监控或随机模型。该样本只能说明时段差异存在，不能支持固定小时切换策略。

## 5. CN0 与相关性：信号强度不能单独解释模型误差

相关性统计显示，绝对误差与仰角呈中等负相关，相关系数约为 -0.40 到 -0.52；与 CN0 的相关性较弱，绝对值大多低于 0.13；与 GIM VTEC 的相关性在不同模型间差异较大，QZS-Japan 和 QZS-Wide 的绝对误差随 GIM VTEC 增大而上升更明显。

| 变量 | GPS r(|err|) | BDS r(|err|) | QZS-Wide r(|err|) | QZS-Japan r(|err|) |
|---|---:|---:|---:|---:|
| 仰角 | -0.42 | -0.52 | -0.40 | -0.44 |
| CN0 | -0.05 | -0.12 | -0.04 | -0.06 |
| GIM VTEC | 0.39 | -0.10 | 0.46 | 0.71 |

**选型含义：**CN0 更适合进入观测噪声、遮挡、多路径和粗差检测模型，不适合作为电离层模型本身的主要选择变量。电离层模型选择更应依赖区域验证、仰角、时间、电离层活动水平、残差一致性和完整性信息。

## 6. SBAS IGP 定位效果：非配对总体改善明显，配对收益更有限

各自有效历元总体统计中，DPS 的水平、垂直和三维误差均低于 SPS：3D Mean 从 2.30 m 降至 1.94 m，3D C95 从 4.90 m 降至 3.97 m。但这组统计混合了可用性差异，DPS 只覆盖 SPS 历元的 70.1%，因此它反映的是“各自有效条件下的运行表现”，不能直接解释为同一历元上的因果改善。

| 口径 | 分量 | SPS Mean | DPS Mean | Mean 差异 | SPS C95 | DPS C95 | C95 差异 |
|---|---|---:|---:|---:|---:|---:|---:|
| 各自有效历元 | H | 1.27 | 0.97 | +23.8% | 2.91 | 1.92 | +33.9% |
| 各自有效历元 | V | 1.71 | 1.53 | +10.3% | 4.38 | 3.68 | +15.8% |
| 各自有效历元 | 3D | 2.30 | 1.94 | +15.7% | 4.90 | 3.97 | +19.0% |
| 同历元配对 | H | 1.02 | 0.97 | +5.6% | 2.15 | 1.92 | +10.7% |
| 同历元配对 | V | 1.48 | 1.53 | -3.7% | 3.93 | 3.68 | +6.1% |
| 同历元配对 | 3D | 1.95 | 1.94 | +0.5% | 4.13 | 3.97 | +3.9% |

<div class="chart-panel" markdown="0">
  <div class="chart-title">图 3  同历元配对后 DPS 相对 SPS 的变化 <span>正值表示 DPS 误差统计更低</span></div>
  <div class="bar-row">
    <div>H Mean</div>
    <div class="bar-track"><span class="bar-fill" style="width: 28.0%"></span></div>
    <div class="bar-value">+5.6%</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>H C95</div>
    <div class="bar-track"><span class="bar-fill" style="width: 53.5%"></span></div>
    <div class="bar-value">+10.7%</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>V Mean</div>
    <div class="bar-track"><span class="bar-fill alt" style="width: 18.5%"></span></div>
    <div class="bar-value">-3.7%</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>V C95</div>
    <div class="bar-track"><span class="bar-fill" style="width: 30.5%"></span></div>
    <div class="bar-value">+6.1%</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>3D Mean</div>
    <div class="bar-track"><span class="bar-fill" style="width: 2.5%"></span></div>
    <div class="bar-value">+0.5%</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>3D C95</div>
    <div class="bar-track"><span class="bar-fill" style="width: 19.5%"></span></div>
    <div class="bar-value">+3.9%</div>
    <div class="bar-value"></div>
  </div>
  <p class="caption">配对统计更适合判断同一历元启用 SBAS IGP 后的变化。水平分量改善较清晰，垂直均值未改善，三维平均误差变化很小。</p>
</div>

小时级统计进一步显示，DPS 并非每个时段都改善 3D 平均误差。按各自有效历元统计，DPS 在 16/24 个小时 3D Mean 低于 SPS；按同历元配对统计，DPS 在 12/24 个小时 3D Mean 低于 SPS。第 19、20、16、21、6、4、15 小时改善较明显；第 0、3、23、13、5、14 小时出现明显退化。由于这些小时同时存在 DPS 可用样本数变化，解释时需要把格网可用性和定位误差一起看。

**选型含义：**SBAS IGP 在该样本中能够降低水平误差和高分位尾部误差，但单日配对结果不支持“启用后所有分量稳定改善”的结论。对单频接收机而言，SBAS IGP 更适合作为受控增强源：当 IGP 覆盖、完整性标志、观测残差和解算状态满足条件时提高权重；当格网不可用、残差异常或几何条件变差时，回退到 Klobuchar 类基础改正。

## 7. 面向单频接收机的可复用选型方法

电离层模型选型应从“可获取、可验证、可回退”三个层面实施。单站单日数据可以建立局部经验，但不能替代长期区域验证。

| 工程问题 | 建议分析维度 | 本样本给出的启示 |
|---|---|---|
| 基础模型如何选 | 总体 RMS、MAE、P95、平均偏差、标准差 | BDS 综合指标最低，但 QZS-Wide 低离散度值得保留为候选 |
| 低仰角如何处理 | 仰角分箱、卫星系统分箱、残差分布 | 0°-10° 内 QZS-Wide RMS 最低，低仰角仍应采用保守定权 |
| 是否需要分时段策略 | UTC 小时 RMS、模型间差值、峰值时段 | 最优模型随 UTC 小时变化，固定全日排序会丢失信息 |
| CN0 能否参与选择 | CN0 分箱、CN0 与绝对误差相关性 | CN0 与模型误差相关性弱，更适合观测定权和粗差检测 |
| SBAS IGP 是否启用 | 各自有效统计、同历元配对统计、小时覆盖率 | 有增强潜力，但配对收益有限且时段差异明显 |
| 如何控制风险 | 完整性标志、残差一致性、回退路径 | SBAS IGP 应受控启用，Klobuchar 基础改正需要持续保留 |

推荐的工程流程如下：

1. 以可持续获取的 Klobuchar 类模型作为基础电离层改正，保证单频接收机在无增强服务时仍有连续改正能力。
2. 在目标区域采集多日、多电离层活动水平的数据，按模型、卫星系统、仰角、UTC、本地时、VTEC 水平和定位状态复算误差。
3. 使用 RMS、MAE、P95、平均偏差和标准差共同评价模型。平均偏差用于识别系统性偏差，RMS 与 P95 用于评价定位误差风险，标准差用于评价模型稳定性。
4. 将模型选择与随机模型耦合。低仰角、高残差时段和模型分歧大的时段应提高电离层残差方差，不能只切换模型名称。
5. 具备 SBAS 能力时，把 IGP 格网改正纳入增强候选，并同时检查 GIVE/完整性标志、格网覆盖、残差一致性、解算稳定性和告警状态。
6. 对增强模型设置降权和回退机制。DPS 与 SPS 或 Klobuchar 解之间的差异可作为监控量，异常时回到基础改正或降低增强权重。

## 8. 结论

该单站单日数据支持以下较稳健的判断：Klobuchar 类模型仍是单频 GNSS 接收机最现实的基础电离层改正来源；在本样本中，BDS 的全日综合误差最低，QZS-Wide 在低仰角和部分 UTC 时段具有优势；UTC 时段、仰角和卫星系统会显著改变模型排序；CN0 不能单独解释电离层模型误差；SBAS IGP 具备增强潜力，但其收益需要按同历元配对和可用性共同评估。

因此，单频接收机的模型选型不宜依赖单一总体排名。更可靠的做法是建立可复算的本地验证流程：用 Klobuchar 类模型保证连续性，用分箱统计确定候选模型和随机模型，用 SBAS IGP 提供受控增强，并用完整性与残差监控决定启用、降权和回退。
