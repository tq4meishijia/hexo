# Handoff: Hexo 8.1.2 + Butterfly 5.7.2 全量升级（已推送）+ 分支清理

## Current state

- 仓库位于工作区根 `C:/Users/Administrator/Desktop/myshit/hexo`。升级已提交并推送：提交 `673c616`（`4da1805..673c616  main -> main`），本地与 `origin/main` 完全同步，`UNPUSHED_COUNT=0`。
- 提交规模：7204 个文件变更，12,134 行新增 / 961,194 行删除（主要是移出 `node_modules/`）。
- 已完成一次性全量升级并验证通过：Hexo 6.3.0 → 8.1.2、Butterfly 4.10.0 → 5.7.2，依赖全量对齐（详见 `package.json`）。
- 已补 `.gitignore` 并停止追踪 `node_modules/`、`public/`、`db.json`；远程提交树中已无这三项，文件仍保留在本地磁盘。
- 新增 `_config.butterfly.yml`（主题配置置于根目录），`_config.yml` 迁移为 `syntax_highlighter: highlight.js`；`README.md` 已补齐运行说明。
- 验证结果：`hexo generate` 退出码 0、25 个产物、无 WARN/ERROR；从零重建结果一致；`hexo server` 全路由 HTTP 200；`source/` 正文零改动。
- 已清理 3 条陈旧 Dependabot 分支（详见下方 Branch cleanup）。**当前远程实体分支仅剩 `main`，本地分支仅 `main`，`origin/HEAD` 为指向 `origin/main` 的符号引用。**
- 本文件（`docs/handoff.md`）与 `.workbuddy/` 未纳入提交，仅保留在本地；工作区仅有 `?? docs/`。
- 旧主题、旧 `db.json`、旧 `public/` 以「移动备份、零删除」方式保存在 `.workbuddy/backup/`。

## Branch cleanup

删除的 3 条分支（陈旧 1076–1121 天，各含 1 个仅改依赖清单的机器人提交，目标版本已被 main 超越）：

| 分支 | 原 tip | 关联 PR |
|---|---|---|
| `dependabot/npm_and_yarn/hexo-renderer-stylus-3.0.0` | `65e7cc5` | #1 → closed |
| `dependabot/npm_and_yarn/hexo-7.0.0` | `3debafc` | #2 → closed |
| `dependabot/npm_and_yarn/hexo-renderer-marked-6.2.0` | `7e61b26` | #3 → closed |

三个 tip commit 对象仍在本地（`git cat-file -t` 已确认），可精确恢复。

## Rollback

**升级提交**已公开到远程，不可再用 `git reset` 改写历史。优先用反向提交：

```
git revert --no-edit 673c616
git push origin main
```

这会恢复 4.10.0 主题与旧依赖清单，并把 `node_modules/`、`public/`、`db.json` 重新加回版本控制（约 96 万行）。若确需彻底抹除，须经明确确认后才用 `git reset --hard 4da1805 && git push --force-with-lease origin main`。

**已删除分支**的恢复（tip SHA 见 `.workbuddy/backup/branch-tips-20261115.txt`）：

```
git push origin 65e7cc51e10f0ee13e94faa73cdca87884abe866:refs/heads/dependabot/npm_and_yarn/hexo-renderer-stylus-3.0.0
git push origin 3debafcfbbef36c63784731d6a7757c87d83b6b9:refs/heads/dependabot/npm_and_yarn/hexo-7.0.0
git push origin 7e61b260309d7a176b57a9768e1ffea25baed409:refs/heads/dependabot/npm_and_yarn/hexo-renderer-marked-6.2.0
```

注意：重建分支不会自动重开已关闭的 PR（需另行处理）。

## Next steps

1. 按实际域名修正 `_config.yml` 的 `url`（当前 `http://example.com`），并配置 `deploy.type`（当前为空）后方可发布。
2. 审阅 `_config.butterfly.yml`（1144 行，来自 5.7.2 默认值），按需个性化菜单、头像、评论与搜索；Butterfly 5.x 配置结构与 4.x 不兼容。
3. `hexo-theme-butterfly-4.10/` 是旧主题源码副本，已无用途；确认后自行处理。
4. 如后续不需要 Dependabot 自动升级，可调整或关闭 `.github/dependabot.yml`（当前为 npm 每日、PR 上限 20）。

## Config audit (2026-11-15)

仅审计，未修改任何配置文件。发布前必须修复的 3 处：

- `_config.yml:16` `url: http://example.com` 是 Hexo 8 官方默认占位值，已泄漏到 `public/index.html` 的 `og:url` / `canonical`（首页出现 6 次）。
- `_config.yml:104` `deploy.type: ''` 为空，`hexo deploy` 无 deployer 可用。
- `_config.butterfly.yml:56` `favicon: /img/favicon.png` 指向不存在的文件（产物中只有 `favicon.ico`），实测 404；此值来自 Butterfly 5.7.2 官方 `default_config.js`，属主题默认值疏漏。

其余待办：主题 `menu`/`social`/`search.use`/`comments.use` 全为空、`subtitle.enable: false` 与站点 subtitle 不一致、`footer.owner.since: 2025` 为默认值；冗余的 `_config.landscape.yml`（0 字节）、`hexo-theme-landscape` 依赖、`hexo-theme-butterfly-4.10/` 副本、与根目录重复的 `themes/butterfly/_config.yml`；`build.sh` 依赖全局 hexo-cli；`dependabot.yml` 为 daily/limit 20 过频；根级 `img/`、`img-qexo/`、`imagine/` 不在 `source/` 下故不会被发布。

## Relevant artifacts

- `package.json`、`package-lock.json`、`_config.yml`、`_config.butterfly.yml`、`.gitignore`、`README.md` — 升级变更文件
- `.workbuddy/backup/themes-butterfly-4.10.0/`、`.workbuddy/backup/db.json.pre-hexo8`、`.workbuddy/backup/public.pre-hexo8` — 升级回滚资源
- `.workbuddy/backup/branch-tips-20261115.txt` — 已删除分支的 tip SHA 与恢复命令
- `.workbuddy/memory/2026-11-15.md` — 会话经验记录
- `https://github.com/tq4meishijia/hexo` — 远程仓库

## Suggested skills

- `handoff` — 若下一会话仍需继续交接
