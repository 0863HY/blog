# blog

个人博客，Hugo + PaperMod 主题，GitHub Pages 自动部署。

- 线上地址：<https://0863HY.github.io/blog/>
- 每次推送到 `main`，GitHub Actions 自动构建并发布
- 文章目录：`content/posts/`

## 写一篇新文章

1. 新建 `content/posts/<slug>.md`（或本地 `hugo new posts/<slug>.md`）
2. frontmatter 固定四个字段：

```yaml
---
title: "文章标题"
date: 2026-11-16T15:00:00+08:00
tags: ["tag1", "tag2"]
author: "随心6"
---
```

3. 提交 PR，合并到 `main` 后自动部署

## 本地预览

```bash
hugo server -D
# http://localhost:1313/blog/
```

需要 Hugo extended 0.170.0（或 `theme.toml` 要求的更高版本）。
