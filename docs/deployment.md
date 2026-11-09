# 部署指南 (Deployment)

> 适用版本：v2.10.35 · 最近校对：2026-11-09

LLM API Sentinel 默认以 **静态导出** 方式构建（`next.config.mjs` 的 `output: 'export'`），产物输出到 `out/`。本文档覆盖静态托管、可选自建模式与后台监控三条路径。

## 1. 通用前置条件

1. Node.js 22（CI 固定值；`package.json` 未声明 `engines.node`，本地 Node 18+ 一般可用）；包管理器必须为 **pnpm 9.15.0** —— `preinstall`（`only-allow`）与 `.npmrc` 会拒绝 `npm` / `yarn` 安装。
2. 配置 Supabase 项目并运行 `supabase/schema.sql`（建表 + RLS）。
3. 准备 `.env.local`，至少包含 3 个 Supabase 变量，可选变量见 [env.md](env.md)。
4. 如需登录功能，在 Supabase Auth 中启用 Google OAuth，并将部署后的回调地址加入白名单（查看状态与手动探测不需要登录）。
5. CI 使用 `pnpm install --frozen-lockfile`，因此仓库必须包含与 `package.json` 一致的 `pnpm-lock.yaml`。

## 2. 静态托管部署（推荐）

### 2.1 Vercel

项目根已包含 `vercel.json`（`framework: nextjs`、构建 `pnpm build`、输出目录 `out/`、SPA 回退 rewrite 到 `/index.html`、安全响应头与 `_next/static` 长缓存）。

- 导入仓库后在 Vercel 控制台填入环境变量（至少 `NEXT_PUBLIC_SUPABASE_URL`、`NEXT_PUBLIC_SUPABASE_ANON_KEY`；`SUPABASE_SERVICE_ROLE_KEY` 仅服务端使用）。
- 触发部署即可。

### 2.2 腾讯云 EdgeOne Pages

项目根已包含 `edgeone.json`（`installCommand: pnpm install`、`buildCommand: pnpm build`、`outputDirectory: out`）与 `edgeone.config.js`（路由重写、全局安全头、JS/CSS/HTML 缓存策略）。

- 关联仓库后按上述配置构建，产物为 `out/`。
- EdgeOne 的 HTML 缓存头为 `public, max-age=0, must-revalidate`。

### 2.3 Netlify

- 构建命令：`pnpm build`；发布目录：`out`。
- 仓库未提供 `netlify.toml`，请在控制台将回退规则设为 `index.html`（或自行新增该配置文件）。

### 构建命令

```bash
pnpm install
pnpm preflight   # 可选：单独校验 apis.json 单一真源（build 已内置）
pnpm build       # preflight + 静态导出到 out/
```

## 3. 可选自建模式（Express 安全服务器）

`examples/self-host-server.ts` 以自定义服务器模式运行，提供额外安全加固：

```bash
pnpm build
node examples/self-host-server.ts
```

实际行为：

- Helmet 配置 CSP（`default-src 'self'`、`frame-ancestors 'none'`、GA4 / Google Fonts / Supabase 白名单）、`X-Frame-Options: DENY`、`X-Content-Type-Options`、`Referrer-Policy: strict-origin-when-cross-origin`，并手动补 `Permissions-Policy`；请求体限制 100kb。
- **全局**按 IP 限流，默认 100 次 / 60 秒（`MemoryRateLimiter`；若设置 `RATE_LIMIT_REDIS_URL` / `RATE_LIMIT_REDIS_TOKEN` 则改用 Redis）。代码里没有 `/api/check` 专用路由或专用限流。
- 所有请求经 `server.all(/.*/)` 交给 Next `handle`；启动 10 秒后执行首次后台巡检，之后每 5 分钟一次。
- 可选：`PORT`（默认 3000）、`SUPABASE_SERVICE_ROLE_KEY`（自建模式必填，缺失会退出）。

> ⚠️ 该模式与静态导出（`output: 'export'`）**不兼容**：需先把 `next.config.mjs` 改为 `standalone`（或移除 `output`）并重新构建。示例为 TypeScript，需要 Node 22+ 类型剥离或自行编译为 JS；`package.json` 未提供 `tsx` / `ts-node` 运行器。

## 4. 后台监控任务（可选）

默认部署下，浏览器端 `useApiMonitor` 每 5 分钟主动探测 29 个 API（数据仅在本地/页面会话中）；若需要服务端持久化，请在 Supabase 侧启用示例任务：

```bash
supabase functions deploy monitor --no-verify-jwt
```

再在 SQL 编辑器执行 `supabase/cron.sql`：注册 `pg_cron` 每 5 分钟 `net.http_post` 调用 `/monitor`，请求体 `{}`，带 `Authorization: Bearer <service_role_key>`。Edge Function 使用平台自动注入的 `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY`，以 service_role 写入 `api_status`（upsert）与 `status_history`（insert），返回：

```json
{ "ok": true, "checked": 29, "offline": 0, "degraded": 0, "at": "2026-11-09T00:00:00.000Z" }
```

`cron.sql` 中的 90 天 `status_history` 清理语句为**注释示例**，默认未启用；`alerts` 表没有自动清理任务。

## 5. 部署前检查清单

- [ ] `pnpm-lock.yaml` 已提交且与 `package.json` 一致（CI 使用 `--frozen-lockfile`）
- [ ] `pnpm preflight` 通过（`apis.json` 前后台一致）
- [ ] `pnpm lint` / `pnpm test` / `pnpm build` 通过
- [ ] `SUPABASE_SERVICE_ROLE_KEY` 未出现在任何 `NEXT_PUBLIC_` 变量中
- [ ] Supabase RLS 策略已启用；OAuth 回调地址已加白名单（如启用登录）
- [ ] 生产环境强制 HTTPS
