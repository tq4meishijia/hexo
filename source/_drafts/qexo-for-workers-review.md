---
categories:
  - 开源推荐
date: 2026-11-17 20:02:42
excerpt: 把 Django 版 Qexo 重写成 Cloudflare Workers 原生应用：单个 Worker 承载管理 API、主题兼容接口与 MCP 服务端，文章仍提交到 Git，运行时零 npm 依赖。本文梳理它的定位、能力、亮点、适用边界，以及什么情况下不该选它。
tags:
  - Cloudflare Workers
  - Hexo
  - 开源
  - MCP
title: 一个 Worker 撑起整套博客后台：Qexo for Workers 推荐
---
静态博客的写作体验长期依赖本地环境：装 Node、配主题、记 Hexo 命令，换台电脑就得重来。Qexo 当年用 Django 做过一个 Web 后台解决这件事，代价是多一个常驻的 Python 服务与数据库。Qexo for Workers 换了条路：把后台整体搬进 Cloudflare Workers，同时保留「Git 仓库是唯一内容源」这个前提。它不是把 Django 逐行翻译成 TypeScript——README 里写得很清楚，架构是被 Workers 的硬约束倒推出来的。

## 项目定位：给静态博客配一个几乎不用维护的后台

一句话概括：跑在 Cloudflare Workers 上的静态博客管理后台，向上兼容 Qexo 的管理语义与主题公开接口，向下把内容写回你自己的 Git 仓库。支持 Hexo、Hugo、Valaxy、VuePress、VitePress、Nuxt、Astro 七套目录映射，内容源覆盖 GitHub、GitLab、Gitea 与 R2。当前版本 1.5.0，上游基线 Qexo 4.3.0，许可证 GPL-3.0-or-later。

值得一提的细节是运行时 npm 依赖为零，构建期只用到 wrangler、typescript、esbuild、yaml；生产包 470.85 KiB（gzip 后 100.11 KiB），代码约 8.6 千行。这个体量说明它不打算做通用 CMS，而是把「博客后台该有的那部分」做完整。

## 核心功能

- **内容**：文章、草稿、页面 CRUD，front-matter 可视化编辑，Markdown 实时预览，abbrlink（crc16 / crc32 复刻上游算法），重命名与发布/转草稿。
- **配置**：`_config.yml` 在线编辑与 YAML / JSON 校验，七种平台目录映射，自定义字段。
- **媒体**：R2、GitHub、S3（SigV4）、自定义图床，以及直接复用内容源凭据的「GitHub 内容源」。
- **互动**：友链与前台申请、说说发布与点赞、PV / UV 统计与域名白名单、站内通知。
- **系统**：四步初始化向导、八组设置、导入导出备份、缓存清理、WebHook 与部署触发。
- **接口**：41 个管理 API、25 个 `/pub/*` 主题接口、18 个 MCP 工具，界面提供 7 种语言。

## 主要亮点

**主题不用改。** `/pub/*` 的路径、鉴权方式与返回结构保持上游语义，现有 Hexo 主题可以直接读后台维护的数据，不需要改主题代码。

**AI 入口是正式的。** MCP 走 Streamable HTTP，10 个只读 / 8 个写入工具按令牌作用域隔离；创建令牌时生成一份自包含的接入提示词，只支持 stdio 的客户端还有桥接脚本。权限可配、写入可查，比一句「支持 AI 写作」实在。

**成本照着免费额度设计。** Trees API 一次拉全量目录、blob sha 复用，保存一篇只产生 1–2 次 Git 调用；列表与搜索走 D1 索引；KV 免费额度只有 1,000 写/天，于是干脆不依赖，缺失时回退 D1 的 `cache` 表；静态资源由 Assets 直出。按其测算，日更 1–3 篇、日均 PV 5,000 时余量在十倍以上，真正的瓶颈是 Pages 免费版每月 500 次构建。

**做不到的事写得清楚。** 36 项迁移对照里 29 项原生、5 项等价替代、2 项平台不可行（云端 Python 脚本、依赖 TCP 的图床），磁盘 Provider 映射为 R2，Passkey 交给 Cloudflare Access。这比宣称「100% 兼容」可信。

**安全基线没有被省略。** PBKDF2-SHA256（100,000 次迭代 + 随机盐）、HttpOnly + SameSite=Lax + HMAC 会话、写操作强制 CSRF、登录限流、凭据一律掩码、MCP 令牌与 WebHook Key 只存 SHA-256；Turnstile 可选且默认关闭。

## 适用场景

**适合：** 已在跑静态博客又不想再养 VPS 或 Python 进程；内容必须留在 Git、看重可迁移；经常换设备写作、不想每次装环境；希望 AI 参与整理与写作，但要求权限可控。

**需要权衡：** 工作流深度依赖 Django 插件生态；需要应用层 Passkey、FTP 类图床或云端脚本执行；日均 PV 很高、构建极频繁；对后台托管在第三方云有硬性合规要求。

## 上手成本

本地三步：`npm install` → `npm run db:local` → `npm run dev`，起在 8787，首次自动创建管理员账号。线上是创建 D1（可选 KV / R2）→ 填资源 ID → `npm run db:remote` → `npm run deploy`。三点注意：初始密码是公开值，公网部署后立刻改，并建议用 Cloudflare Access 保护 `/` 与 `/api/*`；GitHub PAT 必填，Workers 出口 IP 共享匿名配额；R2 未开通就不要绑定。

## 结论

如果博客已经是静态站点、内容已经在 Git 里，Qexo for Workers 是当前比较省心的后台方案之一：成本接近零，运维面几乎没有，写作功能完整，并给 AI 留了一个权限受控的入口。它不假装替代上游 Qexo 的全部能力，而是把能做的做扎实、做不到的标出来——这一点比功能数量更能说明项目的成熟度。两种情况下建议先别选它：工作流深度依赖 Django 侧的扩展能力；构建频率已经逼近 Pages 免费额度。

---

项目信息：v1.5.0 ｜ 上游 Qexo 4.3.0 ｜ GPL-3.0-or-later ｜ 仓库 `github.com/tq4meishijia/qexo4workers`