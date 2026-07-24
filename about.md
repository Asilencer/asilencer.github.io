---
layout: page
title: ""
permalink: /about.html
---

<section class="about-intro">
  <div class="about-portrait">
    <div class="about-portrait-frame">
      <img src="{{ site.author.avatar | relative_url }}" alt="Asilencer">
    </div>
    <span>HELLO / 你好</span>
  </div>
  <div class="about-copy">
    <p class="eyebrow"><span></span> ABOUT THE AUTHOR</p>
    <h1>你好，<br>我是 Asilencer。</h1>
    <p class="about-lede">
      专注 AI 发展、大模型与全栈开发，也在这里整理阅读、实验与工程实践。
    </p>
    <p>
      我更关心技术为什么成立、边界在哪里，以及如何把复杂问题讲清楚。
      这个博客就是这些思考的长期存档。
    </p>
  </div>
</section>

<section class="about-grid">
  <article class="about-card about-principle">
    <span class="about-card-number">01</span>
    <p class="about-card-label">WRITING PRINCIPLE</p>
    <blockquote>把复杂问题拆开，<br>把关键脉络留下。</blockquote>
    <svg viewBox="0 0 240 80" aria-hidden="true">
      <path d="M3 48c47-41 99-43 148-17 29 15 53 14 85-5"/>
      <path d="M2 67c60-28 112-24 153 3 26 17 50 16 81-6"/>
    </svg>
  </article>

  <article class="about-card">
    <span class="about-card-number">02</span>
    <p class="about-card-label">TECH STACK</p>
    <h2>日常使用的工具</h2>
    <div class="tech-stack">
      <span>Go</span>
      <span>Kitex</span>
      <span>GORM</span>
      <span>Python</span>
      <span>MySQL</span>
      <span>Elasticsearch</span>
      <span>Kafka</span>
    </div>
  </article>

  <article class="about-card about-timeline">
    <span class="about-card-number">03</span>
    <p class="about-card-label">MILESTONES</p>
    <h2>一路走来的方向</h2>
    <ol>
      <li>
        <time>2026 — NOW</time>
        <strong>AI Agent 与大语言模型应用</strong>
      </li>
      <li>
        <time>2023 — 2025</time>
        <strong>全栈开发工程师 · Go / Kitex / Python</strong>
      </li>
    </ol>
  </article>

  <article class="about-card about-connect">
    <span class="about-card-number">04</span>
    <p class="about-card-label">CONNECT</p>
    <h2>保持联系</h2>
    <p>欢迎交流文章里的问题、勘误，或分享新的技术线索。</p>
    <div class="connect-links">
      <a href="https://github.com/{{ site.author.github }}" target="_blank" rel="noreferrer">
        GitHub <span>↗</span>
      </a>
      <a href="mailto:{{ site.author.email }}">
        Email <span>↗</span>
      </a>
    </div>
  </article>
</section>

<section class="github-strip">
  <div>
    <p class="about-card-label">GITHUB ACTIVITY</p>
    <h2>持续构建，也持续记录。</h2>
  </div>
  <img src="https://ghchart.rshah.org/24837B/Asilencer" alt="Asilencer 的 GitHub 活跃记录">
</section>
