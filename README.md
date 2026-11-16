# blog

个人博客，Hugo + PaperMod 主题，GitHub Pages 自动部署。

- 线上地址：<https://0863HY.github.io/blog/>
- 文章目录：`content/posts/`
- 技术栈：Hugo extended 0.170.0、GitHub Actions、GitHub Pages（build_type = workflow）

## 发布流程

1. 新文章放在 `content/posts/` 下
2. 提交 PR 到 `main`
3. CI 会跑 **Build check (PR)** —— 文章写坏了（frontmatter 错、日期在未来、模板报错）在这里就会红
4. 合并到 `main` 后，**Deploy Hugo site to Pages** 自动构建并发布

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

> 注意：`date` 不能是未来时间，Hugo 默认不构建未来文章（`buildFuture = false`）。
> 草稿用 `draft: true`，只有 `hugo server -D` 才看得到。

## 目录结构

```
archetypes/posts.md       新文章的 frontmatter 模板
content/posts/            文章
hugo.toml                 站点配置（baseURL / 主题 / 参数）
.github/workflows/        deploy.yml（main 部署）、pr-build.yml（PR 构建检查）
themes/PaperMod           git submodule，CI 用 submodules: recursive 拉取
```

## 本地预览

```bash
hugo server -D
# http://localhost:1313/blog/
```

需要 Hugo extended 0.170.0（或 `theme.toml` 要求的更高版本）。
