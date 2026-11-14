# 部署说明

## 现在的架构

```
GitHub  Zjxkkd/zjxkkd.github.io  main（只放源码）
        │  push 触发 .github/workflows/deploy.yml
        ▼
GitHub Actions：pnpm install → pnpm build → wrangler deploy
        │
        ▼
Cloudflare Worker  zjxkkd-github-io（静态资源，目录 ./dist）
        │  自定义域名
        ▼
https://cloudmoon7.top
```

要点：

- `main` 分支只存源码，**不再提交 `dist/`**。
- 构建与发布全部在 GitHub Actions 内完成，本地不需要装 wrangler 也能发版。
- `wrangler.jsonc` 指定 `assets.directory = ./dist` 与 `not_found_handling = 404-page`，未命中路径返回真正的 404。
- `public/_headers` 控制缓存：`/_astro/*` 与 `/fonts/*` 一年 immutable，HTML 不缓存。

## 一次性配置（只需要做一次）

### 1. 创建 Cloudflare API Token

Cloudflare 控制台 → 右上角头像 → **My Profile → API Tokens → Create Token**，
选模板 **Edit Cloudflare Workers**，并额外加上 **Zone · Cache Purge · Purge** 与 **Zone · Zone · Read**
（Account Resources 选自己的账号，Zone Resources 选 `cloudmoon7.top`）。

创建后复制 token（只显示一次）。

### 2. 写入仓库 Secrets

```bash
gh secret set CLOUDFLARE_API_TOKEN -R Zjxkkd/zjxkkd.github.io
gh secret set CLOUDFLARE_ACCOUNT_ID -R Zjxkkd/zjxkkd.github.io
```

`CLOUDFLARE_ACCOUNT_ID` 在 Cloudflare 控制台右侧栏（Workers & Pages 概览页）可以看到，是一串 32 位十六进制。

### 3. 关掉 Cloudflare 侧的 Git 自动构建（重要）

之前 Worker 是直接连着 GitHub 仓库、以仓库根目录当静态资源的。改用 Actions 之后，
请在 **Cloudflare Dashboard → Workers & Pages → zjxkkd-github-io → Settings → Build** 里
断开 Git 集成（Disconnect），否则每次 push 会有一条旧流程把源码目录当成站点发布，覆盖新版本。

自定义域名（cloudmoon7.top）不需要重绑，Worker 名称不变就会继续生效。

## 日常发布

```bash
git add -A
git commit -m "post: 新文章标题"
git push origin main
```

推送后到 **Actions** 页看 `Deploy` 工作流；成功后 https://cloudmoon7.top 即为新版本。
工作流最后会清一次 Cloudflare 缓存，避免删掉的文章在边缘继续被访问到。

### 切换顺序（重要，别颠倒）

从"main 放 dist"切到"main 放源码"必须按这个顺序，否则 Worker 会把源码目录当成站点发布：

1. 先确认 Actions 的 `Deploy` 能成功（Secrets 已配好）；
2. 到 Cloudflare 断开该 Worker 的 Git 自动构建；
3. 再把 `main` 换成源码、并关掉 GitHub Pages。

只做第 3 步而没做第 2 步，会出现"源码被当成静态站点"的 404 事故。

> 过渡期说明：在切换完成前，`main` 上放了一个只允许手动触发的占位 `deploy.yml`，
> 目的只是让 GitHub 注册 `Deploy` 这个工作流 ID（`workflow_dispatch` 要求工作流存在于默认分支）。
> 真正的 7 步流水线定义在 `source` 分支；`main` 换成源码后这份占位文件会被覆盖掉。

需要手动重发时：Actions → Deploy → Run workflow。

## 本地预览 / 手动发布

```bash
pnpm dev        # 开发
pnpm build      # 只构建到 dist/
pnpm preview    # 预览 dist/

# 本地手动发布（需要本机有 CLOUDFLARE_API_TOKEN 环境变量）
$env:CLOUDFLARE_API_TOKEN="..."
$env:CLOUDFLARE_ACCOUNT_ID="..."
pnpm deploy
```

## 回滚

Cloudflare Dashboard → Workers & Pages → `zjxkkd-github-io` → Deployments，
选中上一个版本 **Rollback** 即可；源码侧 `git revert` 后再推一次也能回到旧版本。

## 旧流程（已废弃）

以前的发布方式是：本地 `pnpm build` → 把 `dist/` 整个复制到 `main` 分支根 → push，
由 GitHub Pages / Cloudflare 直接吃仓库里的静态文件。这条路径的问题是仓库每次发布膨胀约 42 MB、
旧文件不会被清理、也无法回滚到某次构建。**现在请不要再向 main 提交 dist/**。
