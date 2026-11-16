---
title: "第一篇文章：博客搭好了"
date: 2026-11-16T10:00:00+08:00
tags: ["Hugo", "GitHub Pages"]
author: "随心6"
---

这里是博客的第一篇测试文章，用来确认整条发布链路是通的。

## 发布流程

1. 在 `content/posts/` 下新建 Markdown 文件
2. 提交 PR
3. 合并到 `main` 后，GitHub Actions 会自动构建 Hugo 并部署到 GitHub Pages

## 新建文章

本地装了 Hugo 的话：

```bash
hugo new posts/my-new-post.md
```

会自动按 `archetypes/posts.md` 生成带 `title` / `date` / `tags` / `author` 的 frontmatter。
