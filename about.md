---
layout: page
title: 关于
permalink: /about.html
---

<div class="bento-grid">
  <!-- Profile Card -->
  <div class="bento-item col-span-2 row-span-2 bento-profile">
    <img src="{{ site.author.avatar | relative_url | default: '/assets/img/avatar.jpg' }}" alt="Asilencer">
    <h2>Asilencer</h2>
    <p>Backend Developer & Agent Configurator</p>
  </div>

  <!-- Tech Stack Card -->
  <div class="bento-item col-span-2">
    <div class="bento-title">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2L2 7l10 5 10-5-10-5z"></path><path d="M2 17l10 5 10-5"></path><path d="M2 12l10 5 10-5"></path></svg>
      Tech Stack
    </div>
    <div class="tech-stack-grid">
      <span class="tech-badge">Go</span>
      <span class="tech-badge">Kitex</span>
      <span class="tech-badge">GORM</span>
      <span class="tech-badge">Python</span>
      <span class="tech-badge">MySQL</span>
      <span class="tech-badge">Elasticsearch</span>
      <span class="tech-badge">Kafka</span>
    </div>
  </div>

  <!-- Contact & Links Card -->
  <div class="bento-item col-span-2">
    <div class="bento-title">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"></path><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"></path></svg>
      Connect
    </div>
    <div style="display: flex; gap: 1rem; flex-wrap: wrap;">
      <a href="https://github.com/{{ site.author.github }}" target="_blank" class="tech-badge" style="border-color: transparent;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M9 19c-5 1.5-5-2.5-7-3m14 6v-3.87a3.37 3.37 0 0 0-.94-2.61c3.14-.35 6.44-1.54 6.44-7A5.44 5.44 0 0 0 20 4.77 5.07 5.07 0 0 0 19.91 1S18.73.65 16 2.48a13.38 13.38 0 0 0-7 0C6.27.65 5.09 1 5.09 1A5.07 5.07 0 0 0 5 4.77a5.44 5.44 0 0 0-1.5 3.78c0 5.42 3.3 6.61 6.44 7A3.37 3.37 0 0 0 9 18.13V22"></path></svg>
        GitHub
      </a>
      <a href="mailto:{{ site.author.email }}" class="tech-badge" style="border-color: transparent;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 4h16c1.1 0 2 .9 2 2v12c0 1.1-.9 2-2 2H4c-1.1 0-2-.9-2-2V6c0-1.1.9-2 2-2z"></path><polyline points="22,6 12,13 2,6"></polyline></svg>
        Email
      </a>
    </div>
  </div>

  <!-- Timeline Card -->
  <div class="bento-item col-span-4">
    <div class="bento-title">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="10"></circle><polyline points="12 6 12 12 16 14"></polyline></svg>
      Milestones
    </div>
    <div class="timeline">
      <div class="timeline-item">
        <div class="timeline-date">2026 - Present</div>
        <div class="timeline-content">Focusing on AI Agent Configuration & Rule-based Systems</div>
      </div>
      <div class="timeline-item">
        <div class="timeline-date">2023 - 2025</div>
        <div class="timeline-content">Backend Developer (Go / Kitex / GORM / Elasticsearch)</div>
      </div>
    </div>
  </div>
  
  <!-- GitHub Stats Card -->
  <div class="bento-item col-span-4 github-stats">
    <div class="bento-title">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="16 18 22 12 16 6"></polyline><polyline points="8 6 2 12 8 18"></polyline></svg>
      GitHub Activity
    </div>
    <img src="https://ghchart.rshah.org/6366f1/Asilencer" alt="Asilencer's GitHub Chart">
  </div>
</div>

## 📝 关于本站

本站是我的数字花园，主要记录：
- **后端架构**与**微服务**的探索经验
- **AI Agent** 的前沿应用与配置心得
- 基于 **Go** 和 **Python** 的代码实践

博客使用 [Jekyll](https://jekyllrb.com) 构建，部署于 [GitHub Pages](https://pages.github.com)。追求极简、高性能与纯粹的阅读体验。
