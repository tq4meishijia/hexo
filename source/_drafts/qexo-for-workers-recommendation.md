---
title: qexo-for-workers-recommendation
date: 2026-11-17 12:09:49
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
- **配置**：`_config.yml` 等文件在线编辑，带 YAML / JSON 合法性校验；七种博客平台的目录映射与自定义字段。
- **媒体**：图床支持 R2、GitHub、S3（SigV4）、自定义，以及「GitHub 内容源」——直接复用内容源的 token 与仓库，不另存一份密钥。
- **互动**：友链与前台申请、说说发布与点赞、PV / UV 统计与域名白名单、站内通知。
- **系统**：四步初始化向导（语言 → 管理员 → 内容源 → 部署）、八组设置、导入导出备份、缓存清理、WebHook 自动配置、部署触发（Pages / Vercel / Actions / 自定义）。
- **接口**：41 个管理 API、25 个 `/pub/*` 主题公开接口、18 个 MCP 工具，外加 7 种语言的界面词典。

## 主要亮点

### 主题不用改

`/pub/*` 的路径、token 鉴权方式和返回结构都保持上游语义，现有 Hexo 主题可以直接读取后台维护的数据，不需要改主题代码。对已经在用 Qexo 的人来说，这一条基本决定了迁移成本。

### 给 AI Agent 留了正式入口，而不是一个开关

MCP 服务端走 Streamable HTTP（`POST /mcp`），18 个工具按 10 只读 / 8 写入分组，read 与 write 令牌作用域分离；创建令牌时会同步生成一份自包含的接入提示词，包含端点、协议版本、客户端适配规则与排错说明，可以直接发给任意 agent。只支持 stdio 的客户端还有 `mcp/stdio-bridge.mjs` 桥接。

这比「支持 AI 写作」的口号更有价值：权限边界可配置，写操作有据可查，工具清单是明确的。

### 成本模型是照着免费额度设计的

- GitHub 侧：Trees API 一次拉全量目录，blob sha 复用，保存一篇文章只产生 1–2 次 Git 调用；
- D1 侧：内容索引落库，列表与搜索零 Git 调用；KV 免费额度只有 1,000 写/天，所以干脆不依赖 KV，缺失时回退 D1 的 `cache` 表；
- 静态资源：管理端 HTML / CSS / JS 由 Assets 直出，只有 `/api`、`/pub`、`/mcp` 才消耗 Worker 请求。

按项目自己的额度测算，日更 1–3 篇、日均 PV 5,000 的规模下余量在十倍以上。真正的瓶颈不在 Worker，而在 Pages 免费版每月 500 次构建。项目也已在 Workers Free 档位实际部署运行。

### 对「做不到的事」写得很清楚

迁移对照表列了 36 项：29 项原生实现、5 项等价替代、2 项平台不可行（云端 Python 脚本执行、依赖 TCP 的 FTP / 又拍 / Doge 图床）。本地磁盘 Provider 映射为 R2，Passkey 建议交给 Cloudflare Access。比起宣称「100% 兼容」，这种逐项标注意义的写法更值得信任——你在迁移之前就能知道哪一块会缺。

### 安全基线没有被省略

密码用 PBKDF2-SHA256（100,000 次迭代 + 随机盐）；会话 Cookie 是 HttpOnly + SameSite=Lax + HMAC 签名；写操作强制 CSRF token；登录按 10 次 / 10 分钟 / IP 限流；Provider 与图床 token 一律掩码回显；MCP 令牌与 WebHook APIKey 只存 SHA-256，可单独吊销；内容源默认引导申请只覆盖单个仓库、只含 `Contents: Read and write` 的 Fine-grained PAT。登录人机验证可选接 Turnstile，默认关闭，前置条件与实际生效状态在设置页可见。

## 适用场景

**适合：**

- 已经在跑 Hexo / Hugo 等静态博客，想要网页后台，但不想再养一台 VPS 或一个 Python 进程；
- 内容必须留在 Git，看重版本可追溯、可迁移、不被平台锁定；
- 经常在多台设备上写作，不想每次都装一遍 Node 环境；
- 希望 AI Agent 参与选题整理、草稿撰写或批量维护，但要求权限可控、写入有记录。

**需要权衡或不适合：**

- 工作流深度依赖 Django 插件生态的定制，本项目没有对应机制；
- 需要应用层 Passkey（可改用 Cloudflare Access）、FTP 类图床或云端脚本执行，这些在 Workers 上不可行；
- 日均 PV 很高、构建极频繁：Pages 免费版 500 次构建/月会先到顶；
- 对「后台托管在第三方云」有硬性合规要求，需要自建数据库审计。

## 上手成本

本地三步：`npm install` → `npm run db:local` → `npm run dev`，起在 8787，首次会自动创建管理员账号，登录后按向导配置内容源与部署方式。线上是创建 D1（可选 KV / R2）→ 填资源 ID → `npm run db:remote` → `npm run deploy` → 跑冒烟脚本。

三个容易踩的点，README 里都写了：

1. 初始密码是公开值，公网部署后第一件事是改密码，建议同时用 Cloudflare Access 保护 `/` 与 `/api/*`；
2. GitHub PAT 是必填项——Workers 出口 IP 与其它租户共享匿名配额，即使是公开仓库只读也可能被 403；
3. R2 未开通时不要绑定，删掉 `[[r2_buckets]]` 段，图床改用 GitHub / S3 / 自定义。

## 结论

如果博客已经是静态站点、内容已经在 Git 里，Qexo for Workers 是当前比较省心的后台方案之一：成本接近零，运维面几乎没有，日常写作功能完整，并给 AI 留了一个权限受控的入口。它并不试图替代上游 Qexo 的全部能力，而是把能在 Workers 上做好的部分做扎实，把做不到的部分明确标出来——这一点比功能数量更能说明项目的成熟度。

两种情况下建议先别选它：一是工作流深度依赖 Django 侧的扩展能力；二是构建频率已经逼近 Pages 免费额度，后台再省也要先解决构建瓶颈。

---

项目信息：v1.5.0 ｜ 上游 Qexo 4.3.0 ｜ GPL-3.0-or-later ｜ 仓库 `github.com/tq4meishijia/qexo4workers`