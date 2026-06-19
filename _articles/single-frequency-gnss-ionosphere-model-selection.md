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

- 基础广播模型：利用 GPS、BDS、QZS-Wide、QZS-Japan 四种 Klobuchar 类模型的误差统计，确定单频接收机的基础改正候选。
- SBAS 格网增强：利用 DPS 与 SPS 的定位结果统计，评估 SBAS IGP 可用时的定位增益和工程风险。

<div class="lead-card">
  <strong>结论先行：</strong>Klobuchar 类模型适合作为单频接收机的连续基础改正。华东测站单日数据表明，QZS-Wide 可作为低高度角候选，BDS 在总体 RMS 和平均偏差上表现更稳；SBAS IGP 可用、完整性标志有效且残差监控通过时，可作为增强候选纳入定位解算，但需要保留 Klobuchar/SPS 回退路径。
</div>

## 1. 评估口径：围绕单频接收机选型

两个数据集分别对应模型改正量和最终定位结果，用于回答选型中的两个关键问题：基础广播模型如何选，具备 SBAS 条件时是否值得启用格网增强。

| 目录 | 评估对象 | 参考/输出 | 对选型的作用 |
|---|---|---|---|
| `tmp/Klobuchar` | GPS / BDS / QZS-Wide / QZS-Japan 四类 Klobuchar 改正 | 以 GIM slant 改正量为参考 | 形成单频基础改正模型的候选排序 |
| `tmp/analysis` | SPS 与 DPS 定位结果 | 输出 H / V / 3D 定位误差 | 评估 SBAS IGP 相对 Klobuchar SPS 的定位增益 |

Klobuchar 数据来自 2026-05-23 UTC 24 小时，测站约 31.16°N / 121.55°E，每个模型 2,446,538 个样本，总计 9,786,152 个观测。DPS/SPS 数据来自 2026-06-02 到 2026-06-03 的 24 小时静态定位测试，SPS 有 86,373 个有效历元，DPS 有 60,510 个有效历元。

<div class="data-note">
  数据源：<code>tmp/Klobuchar/analysis/overall.csv</code>、<code>hourly_rms.csv</code>、<code>elevation.csv</code>、<code>best_model.csv</code>、<code>tmp/analysis/stats_overall.csv</code>、<code>stats_hourly.csv</code>。DPS/SPS 统计按各自有效历元计算，并非严格同历元配对。
</div>

## 2. Klobuchar 维度：BDS 无偏，QZS-Wide 低仰角更稳

先看模型本身的误差。这里的误差可以理解为：

$$
e_{iono} = I_{model}^{slant} - I_{GIM}^{slant}
$$

GIM 在这里作为后处理参考产品，用于比较不同广播模型的相对误差，并不等同于真实电离层延迟。因此，文中“欠改正”“接近无偏”等表述均指相对 GIM slant 参考而言；负均值表示模型给出的电离层延迟小于 GIM 参考。

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
  <p class="caption">相对 GIM 参考，GPS、QZS-Wide、QZS-Japan 存在约 2 m 量级负偏差；BDS 的均值偏差接近 0。</p>
</div>

总体统计显示：BDS 的 RMS 最低，且相对 GIM 的均值接近零；QZS-Wide 与 GPS 的 RMS 差距不大，但二者都有明显负偏差；QZS-Japan 在该华东测站、该日数据下误差最大。由此可见，QZSS 针对日本区域优化的模型不宜在该类区域场景中默认使用，若要扩展到更大区域，仍需多站、多日和不同电离层活动条件下验证。

## 3. UTC 时段：不同模型的误差峰值并不同步

Klobuchar 类模型的误差具有明显日变化特征。单频接收机在进行模型选型和方差建模时，需要关注 UTC 时段、本地太阳时与模型播发区域之间的共同影响。对华东测站 2026-05-23 的数据而言，GPS、QZS-Wide、QZS-Japan 的 RMS 峰值集中在 UTC 07-08，对应北京时间 15-16 时；BDS 的峰值出现在 UTC 04，对应北京时间 12 时左右，较 GPS/QZS 系列提前约 3-4 小时。

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 2：不同 UTC 时段下的 Klobuchar RMS <span>GPS/QZS 下午峰值明显，BDS 峰值相位提前</span></div>
  <svg viewBox="0 0 760 390" role="img" aria-label="Klobuchar RMS by UTC hour chart">
    <rect x="0" y="0" width="760" height="390" rx="18" fill="#faf8f2"/>
    <line x1="80" y1="310" x2="710" y2="310" stroke="rgba(31,31,27,.22)"/>
    <line x1="80" y1="273" x2="710" y2="273" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="236" x2="710" y2="236" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="199" x2="710" y2="199" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="162" x2="710" y2="162" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="125" x2="710" y2="125" stroke="rgba(31,31,27,.08)"/>
    <line x1="80" y1="88" x2="710" y2="88" stroke="rgba(31,31,27,.08)"/>
    <text x="44" y="314" font-size="12" fill="#6d675d">0</text>
    <text x="44" y="240" font-size="12" fill="#6d675d">2</text>
    <text x="44" y="166" font-size="12" fill="#6d675d">4</text>
    <text x="44" y="92" font-size="12" fill="#6d675d">6m</text>
    <rect x="188" y="62" width="24" height="262" rx="12" fill="rgba(88,120,80,.08)"/>
    <rect x="266" y="62" width="50" height="262" rx="12" fill="rgba(173,107,53,.09)"/>
    <text x="200" y="52" text-anchor="middle" font-size="11" fill="#587850">BDS 峰值</text>
    <text x="291" y="52" text-anchor="middle" font-size="11" fill="#ad6b35">GPS/QZS 峰值</text>
    <polyline points="90,257.3 116,265.3 142,274.2 168,264.1 194,241.8 220,214.8 246,175.5 272,141.8 298,139.1 324,154.9 350,183.4 376,176.4 402,173.7 428,178.3 454,164.4 480,196.9 506,210.0 532,213.3 558,231.5 584,249.0 610,234.9 636,217.9 662,226.4 688,246.0" fill="none" stroke="#456f91" stroke-width="3"/>
    <polyline points="90,234.8 116,195.7 142,178.7 168,165.5 194,164.0 220,184.2 246,232.4 272,226.0 298,221.1 324,233.0 350,226.1 376,226.0 402,218.0 428,202.1 454,169.9 480,199.1 506,210.2 532,212.9 558,230.7 584,249.2 610,237.9 636,235.1 662,256.8 688,259.9" fill="none" stroke="#587850" stroke-width="3"/>
    <polyline points="90,259.2 116,273.0 142,269.0 168,276.5 194,249.9 220,211.2 246,165.1 272,140.1 298,146.0 324,178.2 350,202.7 376,194.0 402,189.7 428,188.7 454,166.3 480,197.3 506,210.0 532,213.3 558,231.5 584,249.2 610,236.0 636,223.8 662,235.8 688,259.3" fill="none" stroke="#ad6b35" stroke-width="3"/>
    <polyline points="90,238.7 116,241.5 142,230.8 168,223.0 194,195.3 220,152.7 246,105.7 272,85.2 298,98.8 324,159.2 350,190.5 376,191.1 402,195.3 428,202.1 454,176.7 480,197.6 506,210.0 532,213.3 558,231.5 584,249.0 610,239.7 636,236.4 662,240.9 688,259.6" fill="none" stroke="#9a4d3d" stroke-width="3"/>
    <text x="90" y="340" text-anchor="middle" font-size="11" fill="#6d675d">0</text>
    <text x="194" y="340" text-anchor="middle" font-size="11" fill="#6d675d">4</text>
    <text x="298" y="340" text-anchor="middle" font-size="11" fill="#6d675d">8</text>
    <text x="402" y="340" text-anchor="middle" font-size="11" fill="#6d675d">12</text>
    <text x="506" y="340" text-anchor="middle" font-size="11" fill="#6d675d">16</text>
    <text x="610" y="340" text-anchor="middle" font-size="11" fill="#6d675d">20</text>
    <text x="688" y="340" text-anchor="middle" font-size="11" fill="#6d675d">23 UTC</text>
    <circle cx="82" cy="26" r="5" fill="#456f91"/><text x="94" y="30" font-size="12" fill="#6d675d">GPS</text>
    <circle cx="148" cy="26" r="5" fill="#587850"/><text x="160" y="30" font-size="12" fill="#6d675d">BDS</text>
    <circle cx="214" cy="26" r="5" fill="#ad6b35"/><text x="226" y="30" font-size="12" fill="#6d675d">QZS-Wide</text>
    <circle cx="318" cy="26" r="5" fill="#9a4d3d"/><text x="330" y="30" font-size="12" fill="#6d675d">QZS-Japan</text>
  </svg>
  <p class="caption">GPS 与 QZS-Wide 在 UTC 07-08 达到约 4.4-4.6 m，QZS-Japan 同时段升至约 5.7-6.1 m；BDS 在 UTC 04 附近达到约 4.0 m，随后在 UTC 06-13 保持相对较低水平。</p>
</div>

逐小时统计对模型调度有两个启示。第一，该日该站在 UTC 06-09，即北京时间 14-17 时，GPS/QZS 系列 Klobuchar 的 RMS 明显升高，接收机可在残差监控中提高对这些模型的方差敏感度。第二，BDS 的峰值相位提前，说明固定采用“地方时下午统一放大所有模型方差”的策略可能掩盖模型间差异。更稳妥的做法是把 UTC、本地时、区域位置和模型来源作为方差建模变量，并通过多站多日数据验证具体门限。

## 4. 高度角比 CN0 更适合驱动模型选择

电离层延迟进入接收机观测时，斜距投影会放大低高度角方向的误差。因此，高度角天然会成为模型误差的主导解释变量。数据也支持这一点：`r(|error|, elev)` 约为 -0.40 到 -0.52，而 `r(|error|, cn0)` 只有 -0.04 到 -0.12。

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 3：不同高度角分箱下的 Klobuchar RMS <span>低高度角差异最大，高高度角逐渐收敛</span></div>
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
  <p class="caption">高度角低于 20° 时，QZS-Wide 在 GPS/BDS 卫星上表现较好；高度角升高后，各模型差异收敛，BDS 在多数分箱更稳。</p>
</div>

由 `best_model.csv` 可以得到更接近工程实现的候选矩阵。这里的“推荐”不宜理解为硬切换规则，更适合作为加权、监控或 A/B 验证的初始依据：

| 卫星系统 | 高度角 | 候选模型 | 解释 |
|---|---:|---|---|
| GPS / BDS | < 20° | QZS-Wide | 低高度角下 RMS 通常较低，可作为低仰角加权候选 |
| GPS / BDS | 20-45° | BDS | 中等高度角下 BDS 误差较小，平均偏差也更接近 GIM 参考 |
| GPS | 45-60° | GPS / BDS 并行验证 | 该分箱中 GPS 原生模型 RMS 最低，提示不宜用单一高度角阈值硬切换 |
| QZSS | ≥10° | BDS | 本测站下 BDS 在多数分箱 RMS 更低 |
| 华东测站该日策略 | 任意 | 不默认使用 QZS-Japan | 该站该日相对 GIM 负偏差和 RMS 较大，扩展结论需更多区域数据支撑 |

因此，单频接收机进行电离层模型调度时，不宜直接使用 CN0 作为主判据。CN0 主要反映信号质量、遮挡和多径，和电离层模型误差没有直接物理关系；但它仍然适合用于观测噪声建模、遮挡/多径识别和粗差检测。更可靠的电离层模型调度变量包括高度角、地方时段、区域经纬度，以及不同模型改正量之间的一致性。

## 5. 格网模型维度：DPS 有增强潜力，收益需按配对口径验证

第二组数据关注定位算法输出：SPS 使用 Klobuchar，DPS 使用 SBAS 播发的 IGP 格网模型。总体分布显示，DPS 在各自有效历元口径下的水平、垂直、3D 平均误差和 C95 均低于 SPS。不过，SPS 有 86,373 个有效历元，DPS 有 60,510 个有效历元，两者并非严格同历元配对；因此，这组总体分布更适合描述“可用解的表现”，不宜直接解释为 DPS 相对 SPS 的因果收益。

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 4：各自有效历元下 DPS 与 SPS 的总体差异 <span>正值表示 DPS 分布更低，非严格配对统计</span></div>
  <svg viewBox="0 0 760 360" role="img" aria-label="DPS and SPS non-paired distribution difference chart">
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
    <rect x="550" y="34" width="18" height="10" rx="3" fill="#587850"/><text x="574" y="43" font-size="12" fill="#6d675d">Mean 差异</text>
    <rect x="638" y="34" width="18" height="10" rx="3" fill="#ad6b35"/><text x="662" y="43" font-size="12" fill="#6d675d">C95 差异</text>
  </svg>
  <p class="caption">按各自有效历元统计，DPS 水平方向分布更低：Mean 从 1.27 m 降到 0.97 m，C95 从 2.91 m 降到 1.92 m。</p>
</div>

| 分量 | SPS Mean | DPS Mean | Mean 差异 | SPS C95 | DPS C95 | C95 差异 |
|---|---:|---:|---:|---:|---:|---:|
| 水平 H | 1.269 m | 0.967 m | 23.8% | 2.910 m | 1.923 m | 33.9% |
| 垂直 V | 1.706 m | 1.530 m | 10.3% | 4.377 m | 3.685 m | 15.8% |
| 三维 3D | 2.302 m | 1.941 m | 15.7% | 4.898 m | 3.968 m | 19.0% |

为了减少有效历元不同带来的选择偏差，还需要看同历元配对结果。只保留 DPS 与 SPS 同时有解的 60,510 个历元后，收益明显收窄：水平 Mean 改善约 5.6%，3D Mean 改善约 0.5%；垂直 Mean 从 1.475 m 增至 1.530 m，未体现平均误差改善。

| 分量 | 配对 SPS Mean | 配对 DPS Mean | Mean 变化 | 配对 SPS C95 | 配对 DPS C95 | C95 变化 |
|---|---:|---:|---:|---:|---:|---:|
| 水平 H | 1.024 m | 0.967 m | +5.6% | 2.154 m | 1.923 m | +10.7% |
| 垂直 V | 1.475 m | 1.530 m | -3.7% | 3.925 m | 3.685 m | +6.1% |
| 三维 3D | 1.951 m | 1.941 m | +0.5% | 4.128 m | 3.968 m | +3.9% |

格网模型带来的增益需要与工程可用性一起评估。这组数据里 DPS 有两个风险点：

- 有效历元为 60,510，而 SPS 为 86,373，DPS 覆盖率约 70%。
- DPS 的最大误差更大：3D 最大值 20.935 m，高于 SPS 的 14.517 m。

这说明 SBAS IGP 适合作为增强候选使用，前提是接收机具备质量控制和回退机制。DPS 的可用解分布较好，但配对口径下收益并非全维度稳定，尾部离群也需要被纳入完整性保护。

<div class="ion-chart" markdown="0">
  <div class="ion-chart-title">图 5：24 小时逐小时 3D 平均误差 <span>DPS 多数时段更低，但并非每小时都占优</span></div>
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
  <p class="caption">非配对逐小时统计中，DPS 在 15/24 个小时的 3D Mean 上低于 SPS；第 2、5、6、7、10、16、17、19、20 小时更高。逐小时样本量不同，需结合覆盖率解释。</p>
</div>

## 6. 工程选型：基础改正与格网增强

综合两组证据，可以形成一套面向单频接收机的分层策略：Klobuchar 类模型提供连续可用的基础改正；SBAS IGP 在可用、完整性标志有效且残差监控通过时，作为区域增强候选进入定位解算；增强解需要经过质量门限、覆盖率检查和异常保护。

<div class="strategy-grid" markdown="0">
  <div class="strategy-card">
    <h4>基础层：Klobuchar 改正</h4>
    <p>提供连续的一阶电离层改正。低高度角可评估 QZS-Wide 权重，中高高度角重点验证 BDS；QZS-Japan 在该华东测站数据下不宜默认采用。</p>
  </div>
  <div class="strategy-card">
    <h4>增强层：SBAS IGP 候选增强</h4>
    <p>SBAS IGP 可用且质量通过时，提高 DPS 解的候选权重。水平误差和 C95 具备改善空间，垂直方向仍需单独监控。</p>
  </div>
  <div class="strategy-card">
    <h4>保护层：异常与回退</h4>
    <p>DPS 缺格网、重收敛、残差异常或垂直离群时回退 SPS，并对电离层方差做自适应膨胀。</p>
  </div>
</div>

推荐逻辑可以写成两级：

```text
if sbas_igp_available and sbas_quality_passed:
    raise_weight(DPS_with_SBAS_IGP)
else:
    if elevation_deg < 20:
        evaluate(QZS_Wide_Klobuchar)
    else:
        evaluate(BDS_Klobuchar_like)
```

进一步，接收机里可以增加以下监控量：

| 监控量 | 用途 | 建议动作 |
|---|---|---|
| SBAS IGP 可用性 | 判断 DPS 是否具备空间覆盖 | 不可用时回退 Klobuchar |
| DPS/SPS 残差差异 | 捕捉格网异常或模型突变 | 差异过大时冻结 DPS 或降权 |
| BDS 与 QZS-Wide 改正差 | 低成本电离层异常探测 | 差值过大时膨胀电离层方差 |
| 高度角 | 模型调度与权重设置 | 低高度角更保守，必要时降权 |
| UTC / 地方时段 | 电离层日变化及模型峰值相位差 | 作为方差建模变量，具体门限需多站多日验证 |

## 7. 结论

对单频 GNSS 接收机来说，电离层模型选型应优先服务于伪距误差控制、连续可用性和异常保护。Klobuchar 类模型连续、低成本、易实现，适合作为基础改正来源，但不同广播源的区域适配性差异明显；SBAS IGP 格网模型具备增强潜力，同时需要处理覆盖率、收敛和尾部风险。

基于这两组数据，一个务实结论是：

- Klobuchar 维度：在华东测站单日数据中，BDS 相对 GIM 的总体 RMS 最低且均值接近零；低高度角时 QZS-Wide 可作为候选；QZS-Japan 在该站该日不宜默认采用。UTC 维度上，GPS/QZS 系列在 UTC 07-08 误差峰值明显，BDS 峰值提前到 UTC 04 左右，提示方差模型需要保留模型来源和时段信息。
- 格网模型维度：非配对总体分布中 DPS 低于 SPS，但同历元配对后，水平 Mean 改善约 5.6%，3D Mean 改善约 0.5%，垂直 Mean 未改善；C95 在水平、垂直和 3D 上分别改善约 10.7%、6.1% 和 3.9%。这说明 SBAS IGP 的收益需要按配对样本、覆盖率和实时质量共同评估。
- 接收机策略：Klobuchar 维持基础连续改正；SBAS IGP 在完整性、覆盖率和残差监控通过时作为增强候选；SPS/Klobuchar 回退机制应始终保留。

因此，推荐的工程路径是分层使用：基础模型保证可用性，格网模型作为精度增强候选，质量监控决定增强解的启用、降权和回退。
