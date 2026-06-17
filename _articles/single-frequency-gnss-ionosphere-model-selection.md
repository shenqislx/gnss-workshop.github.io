---
title: "单频 GNSS 接收机电离层模型选型：从 Klobuchar 到 SBAS 格网"
layout: article
date: 2026-06-17
category: "GNSS 定位算法"
author: "Andy"
summary: "基于 24 小时实测数据，从广播 Klobuchar 类模型误差和 SBAS IGP 格网模型定位收益两个维度，讨论单频 GNSS 接收机的电离层改正模型选型策略。"
---

<style>
  .article-prose {
    --ion-accent: #ad6b35;
    --ion-ink: #1f1f1b;
    --ion-muted: #6d675d;
    --ion-line: rgba(31, 31, 27, 0.12);
    --ion-fill: #faf8f2;
    --ion-warm: rgba(173, 107, 53, 0.12);
    --ion-green: #587850;
    --ion-blue: #456f91;
    --ion-red: #9a4d3d;
  }

  .article-prose .lead-card,
  .article-prose .data-note,
  .article-prose .ion-chart,
  .article-prose .strategy-card,
  .article-prose .metric-card {
    border: 1px solid var(--ion-line);
    border-radius: 18px;
    background: linear-gradient(180deg, rgba(250, 248, 242, 0.98), rgba(255, 255, 255, 0.94));
    box-shadow: 0 8px 24px rgba(36, 30, 19, 0.05);
  }

  .article-prose .lead-card {
    padding: 1.1rem 1.2rem;
    margin: 1.2rem 0;
  }

  .article-prose .lead-card strong {
    color: var(--ion-accent);
  }

  .article-prose .metric-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
    gap: 0.85rem;
    margin: 1rem 0 1.2rem;
  }

  .article-prose .metric-card {
    padding: 0.95rem 1rem;
  }

  .article-prose .metric-label {
    color: var(--ion-muted);
    font-size: 0.86rem;
  }

  .article-prose .metric-value {
    margin-top: 0.25rem;
    color: var(--ion-ink);
    font-size: 1.55rem;
    line-height: 1.12;
    font-weight: 800;
  }

  .article-prose .metric-sub {
    margin-top: 0.35rem;
    color: var(--ion-muted);
    font-size: 0.84rem;
  }

  .article-prose .data-note {
    padding: 0.95rem 1rem;
    margin: 1rem 0;
    color: var(--ion-muted);
    font-size: 0.95rem;
  }

  .article-prose .ion-chart {
    padding: 1rem;
    margin: 1.2rem 0 1.5rem;
    max-width: 100%;
    overflow: auto;
  }

  .article-prose .ion-chart-title {
    display: flex;
    justify-content: space-between;
    gap: 1rem;
    margin-bottom: 0.75rem;
    color: var(--ion-ink);
    font-weight: 800;
  }

  .article-prose .ion-chart-title span {
    color: var(--ion-muted);
    font-size: 0.86rem;
    font-weight: 500;
  }

  .article-prose svg {
    display: block;
    width: 100%;
    min-width: 680px;
    height: auto;
  }

  .article-prose .caption {
    margin: 0.65rem 0 0;
    color: var(--ion-muted);
    font-size: 0.9rem;
  }

  .article-prose table {
    display: block;
    overflow-x: auto;
  }

  .article-prose .data-note code {
    overflow-wrap: anywhere;
  }

  .article-prose .strategy-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
    gap: 0.9rem;
    margin: 1rem 0;
  }

  .article-prose .strategy-card {
    padding: 1rem;
  }

  .article-prose .strategy-card h4 {
    margin: 0;
    padding: 0;
    border: 0;
    font-size: 1rem;
  }

  .article-prose .strategy-card p {
    margin: 0.6rem 0 0;
    color: var(--ion-muted);
  }

  .article-prose .decision-table td:first-child,
  .article-prose .decision-table th:first-child {
    width: 34%;
  }

  @media (max-width: 760px) {
    .article-prose .ion-chart {
      margin-left: -0.35rem;
      margin-right: -0.35rem;
      border-radius: 14px;
    }
  }
</style>

# 单频 GNSS 接收机电离层模型选型：从 Klobuchar 到 SBAS 格网

单频 GNSS 接收机无法通过双频组合消除一阶电离层延迟，电离层误差会直接进入伪距观测。在电离层活动较强、低高度角卫星较多或模型区域适配不足时，残余误差可能达到米级，进而影响水平定位、垂直定位和完整性监控。

工程实现中，Klobuchar 类广播模型最容易获取，计算量低、覆盖连续，适合作为单频接收机的基础电离层改正来源；具备 SBAS 能力时，IGP 格网模型可以提供更强的区域增强信息，但需要同时考虑可用性、收敛状态和异常保护。本文基于两个 24 小时数据集，从可获取性、改正精度和定位收益三个角度，给出单频接收机的模型选型建议。

- 基础广播模型：利用 GPS、BDS、QZS-Wide、QZS-Japan 四种 Klobuchar 类模型的误差统计，确定单频接收机的默认改正源。
- SBAS 格网增强：利用 DPS 与 SPS 的定位结果统计，评估 SBAS IGP 可用时的定位增益和工程风险。

<div class="lead-card">
  <strong>结论先行：</strong>Klobuchar 类模型适合作为单频接收机的基础改正方案。华东测站数据表明，低高度角优先 QZS-Wide，中高高度角优先 BDS；SBAS IGP 可用且质量通过时，DPS 在水平和 3D 定位上较 SPS 有更大收益，但需要配套覆盖率、重收敛和尾部离群监控。
</div>

## 1. 评估口径：围绕单频接收机选型

两个数据集分别对应模型改正量和最终定位结果，用于回答选型中的两个关键问题：基础广播模型如何选，具备 SBAS 条件时是否值得启用格网增强。

| 目录 | 评估对象 | 参考/输出 | 对选型的作用 |
|---|---|---|---|
| `tmp/Klobuchar` | GPS / BDS / QZS-Wide / QZS-Japan 四类 Klobuchar 改正 | 以 GIM slant 改正量为参考 | 确定单频基础改正模型的优先级 |
| `tmp/analysis` | SPS 与 DPS 定位结果 | 输出 H / V / 3D 定位误差 | 评估 SBAS IGP 相对 Klobuchar SPS 的定位增益 |

Klobuchar 数据来自 2026-05-23 UTC 24 小时，测站约 31.16°N / 121.55°E，每个模型 2,446,538 个样本，总计 9,786,152 个观测。DPS/SPS 数据来自 2026-06-02 到 2026-06-03 的 24 小时静态定位测试，SPS 有 86,373 个有效历元，DPS 有 60,510 个有效历元。

<div class="data-note">
  数据源：<code>tmp/Klobuchar/analysis/overall.csv</code>、<code>elevation.csv</code>、<code>best_model.csv</code>、<code>tmp/analysis/stats_overall.csv</code>、<code>stats_hourly.csv</code>。DPS/SPS 统计按各自有效历元计算，并非严格同历元配对。
</div>

## 2. Klobuchar 维度：BDS 无偏，QZS-Wide 低仰角更稳

先看模型本身的误差。这里的误差可以理解为：

$$
e_{iono} = I_{model}^{slant} - I_{GIM}^{slant}
$$

因此负均值意味着模型给出的电离层延迟小于 GIM 参考，也就是欠改正。

<div class="metric-grid" markdown="0">
  <div class="metric-card">
    <div class="metric-label">最佳总体 RMS</div>
    <div class="metric-value">2.66 m</div>
    <div class="metric-sub">BDS Klobuchar-like</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">BDS 平均偏差</div>
    <div class="metric-value">-0.03 m</div>
    <div class="metric-sub">接近无偏</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">GPS 欠改正</div>
    <div class="metric-value">-2.07 m</div>
    <div class="metric-sub">系统性偏小</div>
  </div>
  <div class="metric-card">
    <div class="metric-label">QZS-Japan RMS</div>
    <div class="metric-value">3.30 m</div>
    <div class="metric-sub">华东区域最差</div>
  </div>
</div>

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 1：四种 Klobuchar 类模型的总体误差 <span>RMS 越低越好；均值越接近 0 越无偏</span></div>
  <svg viewBox="0 0 760 360" role="img" aria-label="Klobuchar model overall RMS and mean bias chart">
    <rect x="0" y="0" width="760" height="360" rx="18" fill="#faf8f2"/>
    <line x1="90" y1="286" x2="700" y2="286" stroke="rgba(31,31,27,.22)"/>
    <line x1="90" y1="234" x2="700" y2="234" stroke="rgba(31,31,27,.08)"/>
    <line x1="90" y1="182" x2="700" y2="182" stroke="rgba(31,31,27,.08)"/>
    <line x1="90" y1="130" x2="700" y2="130" stroke="rgba(31,31,27,.08)"/>
    <line x1="90" y1="78" x2="700" y2="78" stroke="rgba(31,31,27,.08)"/>
    <text x="54" y="290" font-size="12" fill="#6d675d">0</text>
    <text x="54" y="238" font-size="12" fill="#6d675d">1</text>
    <text x="54" y="186" font-size="12" fill="#6d675d">2</text>
    <text x="54" y="134" font-size="12" fill="#6d675d">3</text>
    <text x="54" y="82" font-size="12" fill="#6d675d">4m</text>
    <rect x="130" y="134" width="82" height="152" rx="10" fill="#456f91"/>
    <rect x="270" y="148" width="82" height="138" rx="10" fill="#587850"/>
    <rect x="410" y="142" width="82" height="144" rx="10" fill="#ad6b35"/>
    <rect x="550" y="115" width="82" height="171" rx="10" fill="#9a4d3d"/>
    <text x="171" y="126" text-anchor="middle" font-size="14" font-weight="800" fill="#1f1f1b">2.92</text>
    <text x="311" y="140" text-anchor="middle" font-size="14" font-weight="800" fill="#1f1f1b">2.66</text>
    <text x="451" y="134" text-anchor="middle" font-size="14" font-weight="800" fill="#1f1f1b">2.77</text>
    <text x="591" y="107" text-anchor="middle" font-size="14" font-weight="800" fill="#1f1f1b">3.30</text>
    <rect x="130" y="230" width="82" height="24" rx="12" fill="rgba(255,255,255,.62)" stroke="rgba(31,31,27,.1)"/>
    <rect x="270" y="230" width="82" height="24" rx="12" fill="rgba(255,255,255,.62)" stroke="rgba(31,31,27,.1)"/>
    <rect x="410" y="230" width="82" height="24" rx="12" fill="rgba(255,255,255,.62)" stroke="rgba(31,31,27,.1)"/>
    <rect x="550" y="230" width="82" height="24" rx="12" fill="rgba(255,255,255,.62)" stroke="rgba(31,31,27,.1)"/>
    <text x="171" y="246" text-anchor="middle" font-size="11" fill="#6d675d">Mean -2.07</text>
    <text x="311" y="246" text-anchor="middle" font-size="11" fill="#6d675d">Mean -0.03</text>
    <text x="451" y="246" text-anchor="middle" font-size="11" fill="#6d675d">Mean -2.31</text>
    <text x="591" y="246" text-anchor="middle" font-size="11" fill="#6d675d">Mean -2.82</text>
    <text x="171" y="314" text-anchor="middle" font-size="13" fill="#1f1f1b">GPS</text>
    <text x="311" y="314" text-anchor="middle" font-size="13" fill="#1f1f1b">BDS</text>
    <text x="451" y="314" text-anchor="middle" font-size="13" fill="#1f1f1b">QZS-Wide</text>
    <text x="591" y="314" text-anchor="middle" font-size="13" fill="#1f1f1b">QZS-Japan</text>
    <circle cx="610" cy="36" r="5" fill="#ad6b35"/>
    <text x="622" y="40" font-size="12" fill="#6d675d">柱高 = RMS</text>
    <rect x="610" y="50" width="30" height="16" rx="8" fill="rgba(255,255,255,.7)" stroke="rgba(31,31,27,.1)"/>
    <text x="646" y="63" font-size="12" fill="#6d675d">标签 = Mean 偏差</text>
  </svg>
  <p class="caption">GPS、QZS-Wide、QZS-Japan 存在约 2 m 量级欠改正；BDS 的均值偏差接近 0。</p>
</div>

总体统计显示：BDS 的 RMS 最低，且均值几乎为零；QZS-Wide 与 GPS 的 RMS 差距不大，但二者都明显欠改正；QZS-Japan 在这个经度范围下表现最差。由此可见，QZSS 针对日本区域优化的模型不宜直接外推为中国区域的通用改正源。

## 3. 高度角比 CN0 更适合驱动模型选择

电离层延迟进入接收机观测时，斜距投影会放大低高度角方向的误差。因此，高度角天然会成为模型误差的主导解释变量。数据也支持这一点：`r(|error|, elev)` 约为 -0.40 到 -0.52，而 `r(|error|, cn0)` 只有 -0.04 到 -0.12。

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 2：不同高度角分箱下的 Klobuchar RMS <span>低高度角差异最大，高高度角逐渐收敛</span></div>
  <svg viewBox="0 0 760 390" role="img" aria-label="Klobuchar RMS by elevation bin chart">
    <rect x="0" y="0" width="760" height="390" rx="18" fill="#faf8f2"/>
    <line x1="86" y1="310" x2="704" y2="310" stroke="rgba(31,31,27,.22)"/>
    <line x1="86" y1="262" x2="704" y2="262" stroke="rgba(31,31,27,.08)"/>
    <line x1="86" y1="214" x2="704" y2="214" stroke="rgba(31,31,27,.08)"/>
    <line x1="86" y1="166" x2="704" y2="166" stroke="rgba(31,31,27,.08)"/>
    <line x1="86" y1="118" x2="704" y2="118" stroke="rgba(31,31,27,.08)"/>
    <line x1="86" y1="70" x2="704" y2="70" stroke="rgba(31,31,27,.08)"/>
    <text x="50" y="314" font-size="12" fill="#6d675d">0</text>
    <text x="50" y="266" font-size="12" fill="#6d675d">1</text>
    <text x="50" y="218" font-size="12" fill="#6d675d">2</text>
    <text x="50" y="170" font-size="12" fill="#6d675d">3</text>
    <text x="50" y="122" font-size="12" fill="#6d675d">4</text>
    <text x="50" y="74" font-size="12" fill="#6d675d">5m</text>
    <polyline points="116,102 216,87 316,144 416,185 516,210 616,232" fill="none" stroke="#456f91" stroke-width="3"/>
    <polyline points="116,86 216,115 316,164 416,199 516,223 616,237" fill="none" stroke="#587850" stroke-width="3"/>
    <polyline points="116,122 216,110 316,153 416,187 516,208 616,227" fill="none" stroke="#ad6b35" stroke-width="3"/>
    <polyline points="116,89 216,79 316,124 416,161 516,186 616,209" fill="none" stroke="#9a4d3d" stroke-width="3"/>
    <g fill="#456f91"><circle cx="116" cy="102" r="4"/><circle cx="216" cy="87" r="4"/><circle cx="316" cy="144" r="4"/><circle cx="416" cy="185" r="4"/><circle cx="516" cy="210" r="4"/><circle cx="616" cy="232" r="4"/></g>
    <g fill="#587850"><circle cx="116" cy="86" r="4"/><circle cx="216" cy="115" r="4"/><circle cx="316" cy="164" r="4"/><circle cx="416" cy="199" r="4"/><circle cx="516" cy="223" r="4"/><circle cx="616" cy="237" r="4"/></g>
    <g fill="#ad6b35"><circle cx="116" cy="122" r="4"/><circle cx="216" cy="110" r="4"/><circle cx="316" cy="153" r="4"/><circle cx="416" cy="187" r="4"/><circle cx="516" cy="208" r="4"/><circle cx="616" cy="227" r="4"/></g>
    <g fill="#9a4d3d"><circle cx="116" cy="89" r="4"/><circle cx="216" cy="79" r="4"/><circle cx="316" cy="124" r="4"/><circle cx="416" cy="161" r="4"/><circle cx="516" cy="186" r="4"/><circle cx="616" cy="209" r="4"/></g>
    <text x="116" y="340" text-anchor="middle" font-size="12" fill="#6d675d">0-10</text>
    <text x="216" y="340" text-anchor="middle" font-size="12" fill="#6d675d">10-20</text>
    <text x="316" y="340" text-anchor="middle" font-size="12" fill="#6d675d">20-30</text>
    <text x="416" y="340" text-anchor="middle" font-size="12" fill="#6d675d">30-45</text>
    <text x="516" y="340" text-anchor="middle" font-size="12" fill="#6d675d">45-60</text>
    <text x="616" y="340" text-anchor="middle" font-size="12" fill="#6d675d">60-90°</text>
    <circle cx="90" cy="26" r="5" fill="#456f91"/><text x="102" y="30" font-size="12" fill="#6d675d">GPS</text>
    <circle cx="158" cy="26" r="5" fill="#587850"/><text x="170" y="30" font-size="12" fill="#6d675d">BDS</text>
    <circle cx="226" cy="26" r="5" fill="#ad6b35"/><text x="238" y="30" font-size="12" fill="#6d675d">QZS-Wide</text>
    <circle cx="326" cy="26" r="5" fill="#9a4d3d"/><text x="338" y="30" font-size="12" fill="#6d675d">QZS-Japan</text>
  </svg>
  <p class="caption">高度角低于 20° 时，QZS-Wide 在 GPS/BDS 卫星上更有优势；高度角超过 20° 后，BDS 通常是更稳的选择。</p>
</div>

由 `best_model.csv` 可以得到更接近工程实现的模型矩阵：

| 卫星系统 | 高度角 | 推荐模型 | 解释 |
|---|---:|---|---|
| GPS / BDS | < 20° | QZS-Wide | 低高度角下 RMS 通常比 BDS/GPS 更低 |
| GPS / BDS | ≥ 20° | BDS | 中高高度角下 BDS 误差更小且无偏 |
| QZSS | 全高度角，尤其 ≥10° | BDS | 本测站下 BDS 明显优于 GPS 与 QZS-Japan |
| 中国大陆通用策略 | 任意 | 避免 QZS-Japan | 区域适配性差，系统性欠改正最大 |

因此，单频接收机进行电离层模型调度时，不宜直接使用 CN0 作为主判据。CN0 主要反映信号质量、遮挡和多径，和电离层模型误差没有直接物理关系。更可靠的调度变量包括高度角、地方时段、区域经纬度，以及不同模型改正量之间的一致性。

## 4. 格网模型维度：DPS 的主体分布更好，但尾部要保护

第二组数据关注定位算法输出：SPS 使用 Klobuchar，DPS 使用 SBAS 播发的 IGP 格网模型。总体结果表明，DPS 的水平、垂直、3D 平均误差和 C95 都优于 SPS。

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 3：DPS 相对 SPS 的定位误差改善 <span>正值表示 DPS 更优</span></div>
  <svg viewBox="0 0 760 360" role="img" aria-label="DPS improvement over SPS chart">
    <rect x="0" y="0" width="760" height="360" rx="18" fill="#faf8f2"/>
    <line x1="94" y1="286" x2="704" y2="286" stroke="rgba(31,31,27,.22)"/>
    <line x1="94" y1="226" x2="704" y2="226" stroke="rgba(31,31,27,.08)"/>
    <line x1="94" y1="166" x2="704" y2="166" stroke="rgba(31,31,27,.08)"/>
    <line x1="94" y1="106" x2="704" y2="106" stroke="rgba(31,31,27,.08)"/>
    <text x="54" y="290" font-size="12" fill="#6d675d">0%</text>
    <text x="54" y="230" font-size="12" fill="#6d675d">10%</text>
    <text x="54" y="170" font-size="12" fill="#6d675d">20%</text>
    <text x="54" y="110" font-size="12" fill="#6d675d">30%</text>
    <rect x="134" y="143" width="60" height="143" rx="9" fill="#587850"/>
    <rect x="202" y="82" width="60" height="204" rx="9" fill="#ad6b35"/>
    <rect x="326" y="224" width="60" height="62" rx="9" fill="#587850"/>
    <rect x="394" y="191" width="60" height="95" rx="9" fill="#ad6b35"/>
    <rect x="518" y="192" width="60" height="94" rx="9" fill="#587850"/>
    <rect x="586" y="172" width="60" height="114" rx="9" fill="#ad6b35"/>
    <text x="164" y="134" text-anchor="middle" font-size="13" font-weight="800" fill="#1f1f1b">23.8%</text>
    <text x="232" y="73" text-anchor="middle" font-size="13" font-weight="800" fill="#1f1f1b">33.9%</text>
    <text x="356" y="215" text-anchor="middle" font-size="13" font-weight="800" fill="#1f1f1b">10.3%</text>
    <text x="424" y="182" text-anchor="middle" font-size="13" font-weight="800" fill="#1f1f1b">15.8%</text>
    <text x="548" y="183" text-anchor="middle" font-size="13" font-weight="800" fill="#1f1f1b">15.7%</text>
    <text x="616" y="163" text-anchor="middle" font-size="13" font-weight="800" fill="#1f1f1b">19.0%</text>
    <text x="198" y="318" text-anchor="middle" font-size="13" fill="#1f1f1b">水平 H</text>
    <text x="390" y="318" text-anchor="middle" font-size="13" fill="#1f1f1b">垂直 V</text>
    <text x="582" y="318" text-anchor="middle" font-size="13" fill="#1f1f1b">三维 3D</text>
    <rect x="550" y="34" width="18" height="10" rx="3" fill="#587850"/><text x="574" y="43" font-size="12" fill="#6d675d">Mean 改善</text>
    <rect x="638" y="34" width="18" height="10" rx="3" fill="#ad6b35"/><text x="662" y="43" font-size="12" fill="#6d675d">C95 改善</text>
  </svg>
  <p class="caption">DPS 水平方向收益最大：Mean 从 1.27 m 降到 0.97 m，C95 从 2.91 m 降到 1.92 m。</p>
</div>

| 分量 | SPS Mean | DPS Mean | Mean 改善 | SPS C95 | DPS C95 | C95 改善 |
|---|---:|---:|---:|---:|---:|---:|
| 水平 H | 1.269 m | 0.967 m | 23.8% | 2.910 m | 1.923 m | 33.9% |
| 垂直 V | 1.706 m | 1.530 m | 10.3% | 4.377 m | 3.685 m | 15.8% |
| 三维 3D | 2.302 m | 1.941 m | 15.7% | 4.898 m | 3.968 m | 19.0% |

格网模型带来的增益需要与工程可用性一起评估。这组数据里 DPS 有两个风险点：

- 有效历元为 60,510，而 SPS 为 86,373，DPS 覆盖率约 70%。
- DPS 的最大误差更大：3D 最大值 20.935 m，高于 SPS 的 14.517 m。

这说明 SBAS IGP 适合作为增强解使用，前提是接收机具备质量控制和回退机制。主体误差分布改善明显，同时尾部离群也需要被纳入完整性保护。

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 4：24 小时逐小时 3D 平均误差 <span>DPS 多数时段更低，但并非每小时都占优</span></div>
  <svg viewBox="0 0 760 390" role="img" aria-label="Hourly 3D mean error for SPS and DPS">
    <rect x="0" y="0" width="760" height="390" rx="18" fill="#faf8f2"/>
    <line x1="80" y1="310" x2="708" y2="310" stroke="rgba(31,31,27,.22)"/>
    <line x1="80" y1="258" x2="708" y2="258" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="206" x2="708" y2="206" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="154" x2="708" y2="154" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="102" x2="708" y2="102" stroke="rgba(31,31,27,.08)"/>
    <text x="44" y="314" font-size="12" fill="#6d675d">0</text>
    <text x="44" y="262" font-size="12" fill="#6d675d">1</text>
    <text x="44" y="210" font-size="12" fill="#6d675d">2</text>
    <text x="44" y="158" font-size="12" fill="#6d675d">3</text>
    <text x="44" y="106" font-size="12" fill="#6d675d">4m</text>
    <polyline points="91,188 117,191 143,202 169,176 195,189 221,161 247,242 273,177 299,187 325,217 351,197 377,145 403,168 429,185 455,222 481,227 507,230 533,232 559,240 585,241 611,172 637,139 663,97 689,142" fill="none" stroke="#456f91" stroke-width="3"/>
    <polyline points="91,218 117,188 143,215 169,178 195,185 221,126 247,213 273,245 299,213 325,205 351,209 377,247 403,255 429,206 455,245 481,192 507,210 533,247 559,225 585,205 611,240 637,172 663,150 689,186" fill="none" stroke="#ad6b35" stroke-width="3"/>
    <text x="91" y="340" text-anchor="middle" font-size="11" fill="#6d675d">1</text>
    <text x="195" y="340" text-anchor="middle" font-size="11" fill="#6d675d">5</text>
    <text x="299" y="340" text-anchor="middle" font-size="11" fill="#6d675d">9</text>
    <text x="403" y="340" text-anchor="middle" font-size="11" fill="#6d675d">13</text>
    <text x="507" y="340" text-anchor="middle" font-size="11" fill="#6d675d">17</text>
    <text x="611" y="340" text-anchor="middle" font-size="11" fill="#6d675d">21</text>
    <text x="689" y="340" text-anchor="middle" font-size="11" fill="#6d675d">24</text>
    <line x1="536" y1="34" x2="570" y2="34" stroke="#456f91" stroke-width="3"/><text x="578" y="38" font-size="12" fill="#6d675d">SPS 3D Mean</text>
    <line x1="536" y1="56" x2="570" y2="56" stroke="#ad6b35" stroke-width="3"/><text x="578" y="60" font-size="12" fill="#6d675d">DPS 3D Mean</text>
  </svg>
  <p class="caption">DPS 在 15/24 个小时的 3D Mean 上优于 SPS；第 6、16、20 小时反而变差，说明还需要可用性和异常保护。</p>
</div>

## 5. 工程选型：基础改正与格网增强

综合两组证据，可以形成一套面向单频接收机的分层策略：Klobuchar 类模型提供连续可用的基础改正，SBAS IGP 在可用且质量通过时提供更高精度的区域增强，增强解进入定位前需要经过质量门限。

<div class="strategy-grid" markdown="0">
  <div class="strategy-card">
    <h4>基础层：Klobuchar 改正</h4>
    <p>保证任何时候都有电离层一阶改正。低高度角用 QZS-Wide，中高高度角用 BDS，避免在中国区域默认使用 QZS-Japan。</p>
  </div>
  <div class="strategy-card">
    <h4>增强层：SBAS IGP 优先</h4>
    <p>SBAS IGP 可用且质量通过时启用 DPS。水平和 3D 主体分布会明显改善，适合导航、授时和一般位置服务。</p>
  </div>
  <div class="strategy-card">
    <h4>保护层：异常与回退</h4>
    <p>DPS 缺格网、重收敛、残差异常或垂直离群时回退 SPS，并对电离层方差做自适应膨胀。</p>
  </div>
</div>

推荐逻辑可以写成两级：

```text
if sbas_igp_available and sbas_quality_passed:
    use DPS with SBAS IGP
else:
    if elevation_deg < 20:
        use QZS-Wide Klobuchar
    else:
        use BDS Klobuchar-like
```

进一步，接收机里可以增加以下监控量：

| 监控量 | 用途 | 建议动作 |
|---|---|---|
| SBAS IGP 可用性 | 判断 DPS 是否具备空间覆盖 | 不可用时回退 Klobuchar |
| DPS/SPS 残差差异 | 捕捉格网异常或模型突变 | 差异过大时冻结 DPS 或降权 |
| BDS 与 QZS-Wide 改正差 | 低成本电离层异常探测 | 差值过大时膨胀电离层方差 |
| 高度角 | 模型调度与权重设置 | 低高度角更保守，必要时降权 |
| 地方时段 | 电离层日变化 | 下午峰值时段提高过程噪声或观测方差 |

## 6. 结论

对单频 GNSS 接收机来说，电离层模型选型应优先服务于伪距误差控制、连续可用性和异常保护。Klobuchar 类模型连续、低成本、易实现，适合作为基础改正来源，但不同广播源的区域适配性差异明显；SBAS IGP 格网模型能带来更大的定位收益，同时需要处理覆盖率、收敛和尾部风险。

基于这两组数据，一个务实结论是：

- Klobuchar 维度：在华东测站，BDS 是总体最优且几乎无偏的模型；低高度角时 QZS-Wide 更值得优先使用；QZS-Japan 不适合作为中国区域通用模型。
- 格网模型维度：DPS 相比 SPS 的水平 Mean 改善 23.8%，3D Mean 改善 15.7%，C95 也同步改善；但 DPS 覆盖率约 70%，最大误差更大，必须有质量控制。
- 接收机策略：DPS 优先，SPS 保底；Klobuchar 内部再按高度角选择 QZS-Wide 或 BDS。

因此，推荐的工程路径是分层使用：基础模型保证可用性，格网模型提升精度，质量监控决定增强解的启用、降权和回退。
