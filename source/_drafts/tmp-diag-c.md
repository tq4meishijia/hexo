---
title: tmp-diag-c
date: 2026-11-17 12:11:24
---
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