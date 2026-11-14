# 部署说明

## 现在怎么发版

```
GitHub  Zjxkkd/zjxkkd.github.io   main（只放源码）
        │  push 触发 Pages 的 Git 集成
        ▼
Cloudflare Pages  zjxkkd-github-io
        │  pnpm install（自动）→ pnpm build → 发布 dist/
        ▼
https://cloudmoon7.top
```

Pages 项目设置（在 Cloudflare 控制台 **Workers & Pages → zjxkkd-github-io → Settings → Build** 可以看到同一份）：

| 项 | 值 |
|---|---|
| 生产分支 | `main` |
| Build command | `pnpm build` |
| Build output directory | `dist` |
| Root directory | （空） |
| 环境变量 | `NODE_VERSION=22`、`PNPM_VERSION=9.15.1` |

> 这两个环境变量是必须的：Pages 默认镜像的 Node 版本带不动 Astro 5 + Vite 7，
> 锁到 22 才能稳定构建。

## 日常发布

```bash
git add -A
git commit -m "post: 文章标题"
git push origin main
```

推送后到 Cloudflare 控制台 **Deployments** 看这次构建；也可以直接看 Pages 给的预览地址。
构建期间旧版本继续在线，构建失败也不会影响访问。

## 本地预览

```bash
pnpm dev      # 开发服务器
pnpm build    # 只构建到 dist/
pnpm preview  # 预览 dist/
```

## 缓存策略

`public/_headers` 由 Pages 读取：

- `/ _astro/*`、`/fonts/*` → 一年 `immutable`（文件名带内容哈希）
- `/img/*` → 一周
- `/pagefind/*` → 一小时
- HTML → `max-age=0, must-revalidate`

## 回滚

Cloudflare 控制台 → **Workers & Pages → zjxkkd-github-io → Deployments** →
选上一个成功的 production 部署 → **Rollback**。
源码侧 `git revert` 后重新 push 也可以回到旧版本。

## 关于 GitHub Pages

之前同一个 `main` 分支还被 GitHub Pages 当成站点发布（`https://zjxkkd.github.io`），
和自定义域名形成重复内容。现在站点只由 Cloudflare Pages 提供，
GitHub Pages 已在仓库 **Settings → Pages** 里关闭；如果以后又看到它上线，请再关一次。

## 历史：为什么没有用 Workers + GitHub Actions

一开始设想的是「Actions 里 `wrangler deploy` 把 `dist` 推到 Cloudflare Worker」，
排查后确认这个站实际托管在 **Cloudflare Pages**（`zjxkkd-github-io.pages.dev`），
账号里并没有同名 Worker，`cloudflare/workers-autoconfig` 分支只是当时没生效的自动配置残留。

因此最终采用 Pages 自带的 Git 构建：少一条 GitHub Actions 链路，构建缓存、预览部署都由 Pages 提供。
仓库里与 Workers 相关的 `wrangler.jsonc`、`.github/workflows/deploy.yml` 与 `wrangler` 依赖已移除。
