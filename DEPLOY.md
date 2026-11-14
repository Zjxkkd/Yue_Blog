# 部署说明

## 现在怎么发版

```
GitHub  Zjxkkd/Yue_Blog   main（只放源码）
        │  push
        ▼
GitHub Actions  Deploy
        │  pnpm install --frozen-lockfile → pnpm build → wrangler pages deploy dist
        ▼
Cloudflare Pages  zjxkkd-github-io（Direct Upload，生产分支 main）
        │  自定义域名
        ▼
https://cloudmoon7.top
```

关键事实（2026-11-14 实测）：

- 站点托管在 **Cloudflare Pages**，项目名 `zjxkkd-github-io`，不是 Workers（账号里没有同名 Worker）。
- 这个 Pages 项目走的是**直接上传（Direct Upload）**通道：`wrangler pages deploy` 可用，`/pages/projects/{name}/deployments` 接口要求传文件清单。
- 在 `main` 上 push **没有**自动触发 Pages 构建（观察了 10 分钟没有新 deployment），
  所以发布一律以 Actions 里的 `wrangler pages deploy` 为准。

## 日常发布

```bash
git add -A
git commit -m "post: 文章标题"
git push origin main
```

推送后到仓库 **Actions → Deploy** 看这次运行；成功后会打印 `Deployment complete!` 和预览地址。
构建期间旧版本继续在线，失败也不影响访问。

## 本地手动发版

```powershell
$env:CLOUDFLARE_API_TOKEN = "<token>"
$env:CLOUDFLARE_ACCOUNT_ID = "794c1c6d41415b258d3cc26c82c25282"
pnpm build
pnpm deploy        # = wrangler pages deploy dist --project-name zjxkkd-github-io --branch main
```

> `--branch main` 与项目的生产分支一致，这样上传的才是 production 部署；换成别的分支名只会得到预览部署。

## 需要的仓库 Secrets

| Secret | 说明 |
|---|---|
| `CLOUDFLARE_API_TOKEN` | 需要 `Account · Cloudflare Pages · Edit`、`Zone · Zone · Read`、`Zone · Cache Purge · Purge` |
| `CLOUDFLARE_ACCOUNT_ID` | `794c1c6d41415b258d3cc26c82c25282`（32 位十六进制） |

## 缓存策略

`public/_headers` 由 Pages 读取并应用到线上：

- `/_astro/*`、`/fonts/*` → 一年 `immutable`（文件名带内容哈希）
- `/img/*` → 一周
- `/pagefind/*` → 一小时
- HTML → `max-age=0, must-revalidate`

## 回滚

Cloudflare 控制台 → **Workers & Pages → zjxkkd-github-io → Deployments** →
选中一个历史 production 部署 → **Rollback**。
源码侧 `git revert` 后重新 push 也可以。

## 关于 GitHub Pages

仓库原名 `zjxkkd.github.io`（`<user>.github.io` 这种名字会被 GitHub 强制开启 Pages，API 无法关闭）。
2026-11-14 已把仓库改名为 **`Yue_Blog`**，改名后 Pages 才被真正关掉，`gh-pages` 跳转分支也已删除。

现在 `https://zjxkkd.github.io/` 返回 404，站点只由 `https://cloudmoon7.top/` 提供，不存在重复内容。
GitHub 会为旧仓库地址保留跳转，老的 clone 地址和网页链接仍然可用。

## 历史遗留

- `cloudflare/workers-autoconfig` 分支（早期 Workers 自动配置残留）已删除。
- 早期尝试过 Workers + `wrangler deploy`，方向错误（站点不在 Workers），相关文件已移除。
