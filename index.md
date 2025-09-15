---
layout: default
title: 首页
---

<!-- 英雄区域 -->
<section class="hero">
    <div class="hero-container">
        <h1>Andy的GNSS工坊</h1>
        <p>多年嵌入式实战经验，知识分享，拒绝重复造轮子！</p>
        <div class="hero-buttons">
            <a href="#latest-articles" class="btn btn-primary">浏览文章</a>
            <a href="https://github.com/shenqislx/gnss-workshop.github.io" class="btn btn-secondary" target="_blank">GitHub 仓库</a>
        </div>
    </div>
</section>

<!-- 最新文章 -->
<section class="main-content" id="latest-articles">
    <div class="section-title">
        <h2>最新文章</h2>
        <p>探索GNSS领域的前沿技术文章和深度解析</p>
    </div>
    
    <div class="article-grid">
        <article class="article-card">
            <div class="article-card-header">
                <h3><a href="articles/coarse-time-five-state-equation/">粗时导航五状态方程理论与应用</a></h3>
                <span class="article-category">GNSS接收机技术</span>
            </div>
            <div class="article-card-body">
                <p>粗时导航技术是一种在全球导航卫星系统（GNSS）接收机初始时间信息存在较大偏差（数量级为秒或分钟）时，仍能实现有效定位的算法。它通过扩展状态参数，将传统的四参数模型（三维位置与接收机钟差）增强为五参数模型，新增参数为接收机粗时间误差。</p>
                <div class="article-meta">
                    <span class="article-date"><i class="far fa-calendar-alt"></i> 2025-09-14</span>
                    <a href="articles/coarse-time-five-state-equation/" class="read-more">阅读全文 <i class="fas fa-arrow-right"></i></a>
                </div>
            </div>
        </article>
    </div>
</section>

<!-- 专栏区域 -->
<section class="columns-section">
    <div class="columns-container">
        <div class="section-title">
            <h2>专题专栏</h2>
            <p>按主题浏览GNSS相关技术文章</p>
        </div>
        
        <div class="columns">
            <div class="column">
                <div class="column-icon">
                    <i class="fas fa-satellite"></i>
                </div>
                <h3>GNSS接收机技术</h3>
                <p>探索GNSS接收机的设计、实现和优化技术，包括信号处理、定位算法和系统集成。</p>
                <ul class="column-articles">
                    <li><a href="articles/coarse-time-five-state-equation/">粗时导航五状态方程理论与应用</a></li>
                </ul>
                <div class="coming-soon">更多文章即将发布...</div>
            </div>
            
            <div class="column">
                <div class="column-icon">
                    <i class="fas fa-calculator"></i>
                </div>
                <h3>GNSS定位算法</h3>
                <p>深入研究GNSS定位算法，包括单点定位、差分定位、RTK和精密单点定位等技术。</p>
                <div class="coming-soon">更多文章即将发布...</div>
            </div>
            
            <div class="column">
                <div class="column-icon">
                    <i class="fas fa-microchip"></i>
                </div>
                <h3>嵌入式系统开发</h3>
                <p>分享嵌入式系统开发经验，包括硬件设计、软件架构和系统优化等方面的知识。</p>
                <div class="coming-soon">更多文章即将发布...</div>
            </div>
        </div>
    </div>
</section>