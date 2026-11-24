---
title: "开张：博客上线"
date: 2026-11-24T16:30:00+08:00
tags: ["Hugo", "GitHub Pages", "博客"]
author: "clike-peng"
---

这是这个博客的第一篇文章，站点用 **Hugo** 生成，托管在 **GitHub Pages** 上。

## 写作约定

- 文章都是 Markdown，放在 `content/posts/` 目录下。
- frontmatter 固定四个字段：`title`、`date`、`tags`、`author`。
- 草稿成稿后开 PR，合并进 `main` 分支，GitHub Actions 自动构建并发布。

## 发布流程

1. 新建分支：`git switch -c post/文章名`
2. 在 `content/posts/` 下新增 Markdown 文件并提交
3. 推分支并开 PR
4. 合并到 `main`，等 Actions 跑绿，页面即上线

> 现在你正在读的就是这套流程产出的第一页。
