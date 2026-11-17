---
title: tmp-diag-half
date: 2026-11-17 12:10:53
---
静态博客的写作体验，长期依赖本地环境：装 Node、配主题、记住几条 Hexo 命令，换一台电脑就得重新来一遍。Qexo 给出过一个答案——用 Django 搭一个 Web 后台，把写作搬进浏览器；代价是多了一个需要常驻的 Python 服务和一套数据库，以及随之而来的运维。

Qexo for Workers 换了一条路：把后台整体搬进 Cloudflare Workers，同时守住 Qexo 最核心的那个前提——**Git 仓库仍然是唯一内容源**。需要说明的是，它不是把 Django 代码逐行翻成 TypeScript：README 里明确写了，架构是被 Workers 的硬约束倒推出来的。

## 项目定位：给静态博客配一个几乎不用维护的后台

一句话概括：运行在 Cloudflare Workers 上的静态博客管理后台，向上兼容 Qexo 的管理语义与主题公开接口，向下把内容写回你自己的 Git 仓库。

面向的博客形态很明确：Hexo、Hugo、Valaxy、VuePress、VitePress、Nuxt、Astro 七套目录映射；内容源支持 GitHub、GitLab、Gitea 与 R2 四种。当前版本 1.5.0，上游基线 Qexo 4.3.0，许可证 GPL-3.0-or-later。

技术选择上有个细节值得单独提：**运行时 npm 依赖为零**，构建期只用到 wrangler、typescript、esbuild、yaml。生产包 470.85 KiB，gzip 后 100.11 KiB，代码约 8.6 千行。这个体量说明它没打算做通用 CMS，而是把「博客后台该有的那部分」做完整。

## 核心功能：日常写作的闭环

六组能力，基本覆盖从建站到日常维护：

- **内容**：文章、草稿、页面的增删改查；front-matter 可视化编辑；Markdown 实时预览；abbrlink（crc16 / crc32 复刻上游算法）；重命名、发布与转草稿。