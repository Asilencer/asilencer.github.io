---
layout: post
title: "使用 Jekyll + GitHub Pages 搭建个人博客"
date: 2026-04-17
categories: [技术]
tags: [Jekyll, GitHub Pages, 博客]
pin: true
excerpt: "本文介绍如何使用 Jekyll 静态站点生成器配合 GitHub Pages 免费搭建一个现代风格的个人博客。"
---

## 前言

搭建一个属于自己的博客是每个开发者的浪漫。本文将带你从零开始，使用 Jekyll 和 GitHub Pages 搭建一个免费、现代、可定制的个人博客。

## 为什么选择 Jekyll + GitHub Pages？

- **免费托管**：GitHub Pages 提供免费的静态站点托管
- **版本控制**：所有内容都在 Git 仓库中，天然具备版本管理
- **Markdown 写作**：用你熟悉的 Markdown 语法写文章
- **高度可定制**：完全掌控博客的外观和功能
- **自动部署**：推送到 GitHub 即自动构建和发布

## 快速开始

### 1. 创建仓库

在 GitHub 上创建一个名为 `username.github.io` 的仓库。

### 2. 克隆项目

```bash
git clone https://github.com/username/username.github.io.git
cd username.github.io
```

### 3. 本地预览

```bash
bundle install
bundle exec jekyll serve
```

访问 `http://localhost:4000` 即可预览。

### 4. 写文章

在 `_posts` 目录下创建 Markdown 文件，文件名格式为 `YYYY-MM-DD-title.md`：

```markdown
---
layout: post
title: "我的第一篇文章"
date: 2026-04-17
categories: [技术]
tags: [标签1, 标签2]
---

文章内容...
```

### 5. 部署

```bash
git add .
git commit -m "Add new post"
git push
```

推送后 GitHub Actions 会自动构建并部署。

## 自定义

- 修改 `_config.yml` 配置站点信息
- 编辑 `assets/css/style.css` 自定义样式
- 修改 `_layouts` 中的模板调整页面结构

> 开始写作吧，记录你的想法和成长！
