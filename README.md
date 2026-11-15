# hexo

基于 Hexo 的静态博客站点，主题为 Butterfly。

## 环境要求

- Node.js >= 20.19.0（Hexo 8.1.2 的最低要求）
- npm（随 Node 安装）

## 快速开始

```
npm install
npm run build      # hexo generate，产物输出到 public/
npm run server     # 本地预览 http://localhost:4000/
npm run clean      # 清除 public/ 与 db.json 缓存
```

## 技术栈

| 组件 | 版本 |
| --- | --- |
| Hexo / hexo-cli | 8.1.2 / 4.3.2 |
| Butterfly 主题 | 5.7.2 |
| hexo-renderer-marked | 7.0.1 |
| hexo-renderer-pug / stylus | 3.0.0 / 3.0.1 |
| hexo-generator-index | 4.0.0 |
| hexo-util / moment-timezone | 4.0.0 / 0.6.6 |

模板层为 Pug，样式层为 Stylus；代码高亮由 `_config.yml` 的 `syntax_highlighter: highlight.js` 控制。

## 配置

- `_config.yml` — 站点配置（`theme: butterfly`、永久链接、代码高亮）
- `_config.butterfly.yml` — 主题配置，Hexo 5+ 自动加载；升级主题时迁移此文件即可，不会被覆盖
- `themes/butterfly/` — 主题源码目录
- `source/_posts/` — Markdown 文章
- `scaffolds/` — 文章 / 页面 / 草稿模板

## 注意事项

- `node_modules/`、`public/`、`db.json` 已由 `.gitignore` 排除，请勿提交。
- `_config.yml` 中 `url` 仍是 `http://example.com` 占位、`deploy.type` 为空，正式发布前需按实际域名与部署方式配置。
- 升级主题前先备份 `_config.butterfly.yml`；Butterfly 5.x 与 4.x 的配置项名称有大量变更，不可直接沿用旧配置。
