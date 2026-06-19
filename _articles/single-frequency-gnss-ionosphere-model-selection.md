---
title: "单频 GNSS 接收机电离层模型选型：Klobuchar 基础改正与 SBAS IGP 增强"
layout: article
date: 2026-06-19
category: "GNSS 定位算法"
author: "Andy"
summary: "围绕单频 GNSS 接收机的电离层改正选型，分析 Klobuchar 类广播模型的低成本价值与误差局限，并讨论 SBAS IGP 格网模型对单频定位服务稳定性的改善。"
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
    grid-template-columns: 112px minmax(260px, 1fr) 64px 64px;
    gap: 0.65rem;
    align-items: center;
    margin: 0.48rem 0;
    min-width: 560px;
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

# 单频 GNSS 接收机电离层模型选型：Klobuchar 基础改正与 SBAS IGP 增强

单频 GNSS 接收机无法通过双频无电离层组合消除一阶电离层延迟，伪距观测中会保留显著的电离层残差。工程上最容易获得的改正手段是广播 Klobuchar 类模型：它随导航电文播发，计算量很小，不依赖额外通信链路，适合作为单频接收机的基础电离层改正。代价也很清楚：Klobuchar 是低阶经验模型，改正效果有限，残余误差在低仰角、电离层活动较强或区域适配不足时仍可达到米级。

如果接收机具备 SBAS 能力，IGP 电离层格网模型能够提供区域化的格网延迟和质量信息。相比 Klobuchar 的少量广播参数，格网模型携带更直接的空间信息，通常更适合作为单频定位服务的增强改正来源。本文基于单测站单日数据，从 Klobuchar provider 差异、UTC 时段差异、低仰角误差以及 SBAS IGP 对定位服务稳定性的影响四个角度，讨论单频接收机的模型选型。

<div class="summary-panel" markdown="0">
  <p><strong>核心判断。</strong>Klobuchar 类模型的价值在于廉价、连续、易实现；它能给单频接收机提供基础改正，但残余误差仍然明显。该样本中四类 Klobuchar 模型全日 RMS 约 2.66-3.30 m，P95 约 5.35-6.50 m，说明它更适合作为基础层和回退层。</p>
  <p><strong>provider 与时段差异。</strong>不同 provider 的 Klobuchar 模型表现并不一致。按 UTC 小时统计，BDS 仅在 12/24 小时取得最低 RMS，QZS-Wide、QZS-Japan 和 GPS 也分别在若干小时占优；模型间最大小时 RMS 差值约 3.81 m。</p>
  <p><strong>格网增强。</strong>SBAS IGP 属于格网电离层模型。使用 SBAS 电离层改正的 DPS 相比使用 Klobuchar 的 SPS，在各自有效历元下 3D Mean 从 2.30 m 降至 1.94 m，3D C95 从 4.90 m 降至 3.97 m。它更适合作为具备服务覆盖和完整性监控时的增强解。</p>
</div>

## 1. 选型问题：先保证基础改正，再争取区域增强

单频接收机的电离层模型选型可以分成两层。第一层是基础改正，目标是用最低成本降低一阶电离层延迟的主要量级；第二层是增强改正，目标是在区域服务可用时进一步压低残余误差并改善定位服务的一致性。

Klobuchar 模型位于基础层。它的工程优势非常明确：导航电文即可获得，不增加接收机通信成本，算法实现简单，适合芯片、模组和低功耗设备长期保留。它的精度上限同样明确：模型参数少，空间分辨率有限，对低仰角和电离层时变结构的描述能力不足。

SBAS IGP 格网模型位于增强层。它把电离层延迟以格网点形式播发，并伴随完整性相关质量信息。对于单频定位服务，SBAS IGP 的价值不只在于降低某一时刻的误差，还在于减弱 Klobuchar 模型在 provider、UTC 时段和区域适配上的不稳定表现。

<div class="metric-grid" markdown="0">
  <div class="metric-card">
    <div class="metric-label">Klobuchar 成本</div>
    <div class="metric-value">最低</div>
    <div class="metric-note">随导航电文获得</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">Klobuchar 残差</div>
    <div class="metric-value">2.66-3.30 m</div>
    <div class="metric-note">四类模型全日 RMS</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">低仰角 RMS</div>
    <div class="metric-value">约 4 m</div>
    <div class="metric-note">0°-20° 分箱仍明显偏大</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">DPS 3D C95</div>
    <div class="metric-value">3.97 m</div>
    <div class="metric-note">低于 SPS 的 4.90 m</div>
  </div>
</div>

## 2. Klobuchar 是廉价基础模型，但改正效果有限

以 GIM 斜向电离层产品作为参考量，该样本中四类 Klobuchar 模型的全日 RMS 分布在 2.66-3.30 m 之间，95% 分位绝对误差分布在 5.35-6.50 m 之间。BDS Klobuchar-like 在全日 RMS、MAE 和 P95 上最低，QZS-Wide 的标准差最低但存在约 -2.31 m 的平均偏差，GPS 和 QZS-Japan 的残余误差更高。

这组数据说明 Klobuchar 模型可以显著优于完全不做电离层建模的单频伪距解算状态，但它并不能把电离层误差压到可忽略水平。对算法实现而言，Klobuchar 应承担“基础改正”和“连续回退”的职责；对精度敏感的单频定位服务，还需要额外的区域增强信息或更保守的残差定权。

<div class="chart-panel" markdown="0">
  <div class="chart-title">图 1  Klobuchar provider 的全日误差水平 <span>单位：m；条形为 RMS，右侧为 P95</span></div>
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
  <p class="caption">四类模型均保留米级残差。BDS 在该单站单日样本中综合误差最低，但这一排序需要在目标区域和目标时段复核。</p>
</div>

| Provider | Mean | STD | RMS | MAE | P95 |
|---|---:|---:|---:|---:|---:|
| GPS | -2.07 | 2.07 | 2.92 | 2.32 | 6.06 |
| BDS | -0.03 | 2.66 | 2.66 | 2.17 | 5.35 |
| QZS-Wide | -2.31 | 1.54 | 2.77 | 2.31 | 5.63 |
| QZS-Japan | -2.82 | 1.71 | 3.30 | 2.82 | 6.50 |

## 3. provider 与 UTC 时段会显著改变 Klobuchar 表现

Klobuchar provider 的差异不能只看全日平均值。按 UTC 小时统计，四类模型的误差峰值和低误差时段并不同步。BDS 在 12 个小时取得最低 RMS，QZS-Wide 在 5 个小时最低，QZS-Japan 在 4 个小时最低，GPS 在 3 个小时最低。UTC 7 h 的模型间 RMS 差值约 3.81 m，是全日差异最大的小时。

这意味着单频接收机如果可以获得多套 Klobuchar 参数，provider 的选择和定权应结合时间、区域和残差表现进行验证。全日综合最好的 provider，在特定 UTC 时段未必表现最好；某个 provider 在全日平均上落后，也可能在局部时段给出较低 RMS。工程上更稳妥的做法是把 provider 作为模型状态的一部分纳入监控，而不能只保存一个固定排序。

<div class="chart-panel" markdown="0">
  <div class="chart-title">图 2  每个 UTC 小时 RMS 最低的 Klobuchar provider <span>颜色表示该小时最低 RMS 模型</span></div>
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
  <p class="caption">该图只描述单站单日样本的时段差异。它支持“需要按 UTC/区域验证 provider 表现”的工程判断，不支持固定小时切换规则。</p>
</div>

| UTC 维度 | 数据表现 |
|---|---|
| 最低 RMS 小时数 | BDS 12 h，QZS-Wide 5 h，QZS-Japan 4 h，GPS 3 h |
| 最大模型间差值 | UTC 7 h，约 3.81 m |
| 模型峰值 | QZS-Japan UTC 7 h 约 6.09 m；GPS UTC 8 h 约 4.63 m；QZS-Wide UTC 7 h 约 4.60 m；BDS UTC 4 h 约 3.95 m |
| 工程含义 | provider、UTC、本地时、电离层活动水平和区域位置需要共同参与模型验证 |

## 4. 低仰角下 Klobuchar 残差明显偏大

低仰角观测的电离层斜向延迟更大，多路径和遮挡风险也更高。该样本中，0°-10° 仰角分箱的最佳 RMS 仍为 3.93 m，10°-20° 分箱的最佳 RMS 为 4.07 m；到 60°-90° 分箱，最佳 RMS 降至 1.53 m。这个变化符合斜向映射函数的基本规律，也说明低仰角下仅依赖 Klobuchar 模型很难获得稳定的伪距改正效果。

因此，低仰角观测的处理不应只依靠切换 provider。更合理的策略包括提高电离层残差方差、降低低仰角观测权重、结合残差一致性进行粗差检测，并在 SBAS IGP 可用时使用格网改正作为增强来源。

| 仰角分箱 | GPS RMS | BDS RMS | QZS-Wide RMS | QZS-Japan RMS | 分箱最低 |
|---|---:|---:|---:|---:|---|
| 0°-10° | 4.34 | 4.67 | <span class="heat-best">3.93</span> | 4.61 | QZS-Wide |
| 10°-20° | 4.65 | <span class="heat-best">4.07</span> | 4.17 | 4.82 | BDS |
| 20°-30° | 3.46 | <span class="heat-best">3.04</span> | 3.27 | 3.88 | BDS |
| 30°-45° | 2.60 | <span class="heat-best">2.31</span> | 2.57 | 3.10 | BDS |
| 45°-60° | 2.09 | <span class="heat-best">1.81</span> | 2.12 | 2.58 | BDS |
| 60°-90° | 1.63 | <span class="heat-best">1.53</span> | 1.72 | 2.10 | BDS |

## 5. SBAS IGP 是格网模型，增强效果明显高于 Klobuchar 基础改正

SBAS IGP 电离层模型以格网点形式描述区域电离层延迟，并提供与完整性相关的质量信息。它的信息结构比 Klobuchar 的少量经验参数更丰富，尤其适合服务覆盖区域内的单频定位增强。对于接收机算法，SBAS IGP 通常作为受控增强源使用，在格网覆盖、完整性标志和残差检查满足条件时提高电离层改正质量。

从定位结果看，使用 SBAS 电离层改正数据的 DPS 在各自有效历元下明显优于使用 Klobuchar 的 SPS。水平 Mean 从 1.27 m 降至 0.97 m，3D Mean 从 2.30 m 降至 1.94 m；水平 C95 从 2.91 m 降至 1.92 m，3D C95 从 4.90 m 降至 3.97 m。该统计口径反映了服务可用条件下的实际表现。考虑算法归因时，还需要关注同历元配对统计；配对后水平误差和高分位误差仍有改善，三维平均收益收窄。

<div class="chart-panel" markdown="0">
  <div class="chart-title">图 3  SPS 与 DPS 的全天定位误差对比 <span>单位：m；各自有效历元统计，数值越低越好</span></div>
  <div class="bar-row">
    <div>SPS 3D Mean</div>
    <div class="bar-track"><span class="bar-fill alt" style="width: 47.0%"></span></div>
    <div class="bar-value">2.30</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>DPS 3D Mean</div>
    <div class="bar-track"><span class="bar-fill" style="width: 39.6%"></span></div>
    <div class="bar-value">1.94</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>SPS 3D C95</div>
    <div class="bar-track"><span class="bar-fill alt" style="width: 100%"></span></div>
    <div class="bar-value">4.90</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>DPS 3D C95</div>
    <div class="bar-track"><span class="bar-fill" style="width: 81.0%"></span></div>
    <div class="bar-value">3.97</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>SPS 小时 STD</div>
    <div class="bar-track"><span class="bar-fill alt" style="width: 72.0%"></span></div>
    <div class="bar-value">0.72</div>
    <div class="bar-value"></div>
  </div>
  <div class="bar-row">
    <div>DPS 小时 STD</div>
    <div class="bar-track"><span class="bar-fill" style="width: 65.0%"></span></div>
    <div class="bar-value">0.65</div>
    <div class="bar-value"></div>
  </div>
  <p class="caption">DPS 的 3D Mean 和 C95 均低于 SPS；逐小时 3D Mean 的标准差也略低。DPS 可用历元约为 SPS 的 70.1%，服务启用仍需质量门限和回退策略。</p>
</div>

| 统计口径 | 分量 | SPS Mean | DPS Mean | Mean 差异 | SPS C95 | DPS C95 | C95 差异 |
|---|---|---:|---:|---:|---:|---:|---:|
| 各自有效历元 | H | 1.27 | 0.97 | +23.8% | 2.91 | 1.92 | +33.9% |
| 各自有效历元 | V | 1.71 | 1.53 | +10.3% | 4.38 | 3.68 | +15.8% |
| 各自有效历元 | 3D | 2.30 | 1.94 | +15.7% | 4.90 | 3.97 | +19.0% |
| 同历元配对 | H | 1.02 | 0.97 | +5.6% | 2.15 | 1.92 | +10.7% |
| 同历元配对 | V | 1.48 | 1.53 | -3.7% | 3.93 | 3.68 | +6.1% |
| 同历元配对 | 3D | 1.95 | 1.94 | +0.5% | 4.13 | 3.97 | +3.9% |

## 6. DPS 的全天服务表现更稳定，但仍需完整性和回退

Klobuchar 模型误差在 UTC 维度上呈现明显 provider 差异和时段差异。定位服务层面，DPS 引入 SBAS IGP 后，在有效服务历元下给出的全天误差分布更低：DPS 3D Mean 低于 SPS，3D C95 也低于 SPS；逐小时 3D Mean 的标准差从约 0.72 m 降至约 0.65 m，16/24 个小时的 DPS 3D Mean 低于 SPS。

这说明 SBAS IGP 有助于削弱 Klobuchar 基础模型在不同 UTC 时段、电离层活跃程度和 provider 适配差异下带来的服务波动。该结论仍然需要保留边界：DPS 并非每小时都优于 SPS，个别小时仍会出现退化；DPS 可用历元少于 SPS，也说明格网增强服务需要可用性、完整性标志、残差一致性和解算稳定性共同约束。

| 指标 | SPS | DPS | 解释 |
|---|---:|---:|---|
| 有效历元 | 86,373 | 60,510 | DPS 覆盖约为 SPS 的 70.1% |
| 3D Mean | 2.30 m | 1.94 m | DPS 有效历元下平均误差更低 |
| 3D C95 | 4.90 m | 3.97 m | DPS 尾部误差更低 |
| 逐小时 3D Mean 标准差 | 0.72 m | 0.65 m | DPS 小时波动略低 |
| 3D Mean 较低小时数 | - | 16/24 | DPS 多数小时低于 SPS |

## 7. 面向单频接收机的工程选型建议

单频接收机的电离层模型选型建议采用分层策略。

1. Klobuchar 作为基础改正长期保留。它成本最低、可获得性最好，适合所有单频接收机作为基础电离层改正和增强服务不可用时的回退模型。
2. Klobuchar provider 需要按区域和时段验证。不同 provider 在全日、UTC 小时和仰角分箱下表现不同，单站单日数据只能形成局部经验，产品策略需要多站、多日和不同电离层活动水平验证。
3. 低仰角观测需要更保守的随机模型。0°-20° 分箱下即使选择较优 provider，RMS 仍约 4 m；低仰角观测应降低权重，并结合残差检测和质量控制。
4. SBAS IGP 作为单频增强解优先考虑。格网模型在服务覆盖区内比 Klobuchar 提供更丰富的区域电离层信息，适合用于 DPS 一类单频增强定位服务。
5. DPS 需要完整性、可用性和回退机制。SBAS IGP 增强能够改善总体误差分布和全天一致性，但格网缺失、质量标志异常、残差不一致或几何条件变差时，应降权或回退到 Klobuchar/SPS。

## 8. 结论

Klobuchar 类模型是单频 GNSS 接收机改善电离层延迟最廉价、最容易获取的模型。它适合承担基础改正任务，但该样本中的 RMS 和 P95 表明其残余误差仍处于米级，低仰角下尤其明显。不同 provider 的表现差异也较大，UTC 小时统计显示最优模型随时段变化，单频接收机需要把 provider、时段、仰角和残差监控纳入模型选型。

SBAS IGP 属于格网电离层模型，提供了比 Klobuchar 更丰富的区域改正信息。使用 SBAS 电离层改正数据的 DPS，相比使用 Klobuchar 的 SPS，在有效服务历元下表现出更低的 3D Mean 和 3D C95，全天逐小时误差也略更收敛。工程上更合理的路径是：Klobuchar 保证低成本连续改正，SBAS IGP 在覆盖和质量满足条件时提供增强，质量监控决定启用、降权和回退。
