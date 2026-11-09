# 环境变量参考 (Environment Variables)

> 适用版本：v2.10.35 · 最近校对：2026-11-09

本文件说明 LLM API Sentinel 各运行形态读取的环境变量。前端/构建期变量在项目根目录的 `.env.local` 中配置（模板见 `.env.example`）；Edge Function 的变量由 Supabase 平台注入。

> `.env.local` 已被 `.gitignore` 忽略，切勿提交密钥。`.env.example` 仅作为模板，不含任何真实凭证，当前只列出 3 个 Supabase 变量，其余可选变量需按需自行追加。

## 1. 前端 / 构建期变量（`.env.local`）

| 变量名 | 必填 | 读取位置 | 用途 |
|--------|------|----------|------|
| `NEXT_PUBLIC_SUPABASE_URL` | 建议必填 | `app/lib/supabase.ts`、`app/layout.tsx` | Supabase 项目地址，浏览器可见；未配置时前端不连接数据库，仅做本地真实探测 |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | 建议必填 | `app/lib/supabase.ts` | 匿名公钥，受 RLS 约束；与 URL 同时缺失时 `isSupabaseConfigured` 为 false |
| `NEXT_PUBLIC_SITE_URL` | 可选 | `app/layout.tsx`、`app/robots.ts`、`app/sitemap.ts`、`app/components/StructuredData.tsx` | 站点基址；缺省时开发环境为 `http://localhost:3000`，生产为 `https://llmapi.ewuse.com` |
| `NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION` | 可选 | `app/layout.tsx` | Google Search Console 站点验证 metadata |
| `NODE_ENV` | 自动注入 | 多处 | `development` / `production`，由 Next.js 与 Node 自动设置，无需手动配置 |

## 2. 服务端 / 可选自建模式变量

| 变量名 | 必填 | 读取位置 | 用途 |
|--------|------|----------|------|
| `SUPABASE_SERVICE_ROLE_KEY` | 自建模式必填 | `examples/self-host-server.ts` | 绕过 RLS 的服务端写入密钥；自建服务器缺失时直接退出；纯静态导出 + 托管 Edge Function 时由平台注入，本地 `.env.local` 可留空 |
| `PORT` | 可选 | `examples/self-host-server.ts` | 自建服务器监听端口，默认 `3000` |
| `RATE_LIMIT_REDIS_URL` | 可选 | `examples/self-host-server.ts` | 启用 Redis 限流后端（替代内存限流） |
| `RATE_LIMIT_REDIS_TOKEN` | 可选 | `examples/self-host-server.ts` | Redis 访问令牌，与上一项配套 |

## 3. Edge Function secrets（Supabase 平台自动注入）

| 变量名 | 读取位置 | 说明 |
|--------|----------|------|
| `SUPABASE_URL` | `supabase/functions/monitor/index.ts` | Edge Runtime 自动注入，无需手动设置 |
| `SUPABASE_SERVICE_ROLE_KEY` | `supabase/functions/monitor/index.ts` | 同上；函数以 service_role 写入 `api_status` / `status_history` |

> Edge Function 只读白名单 `apis.json`，不接受外部 URL 参数；部署命令为 `supabase functions deploy monitor --no-verify-jwt`（详见 [deployment.md](deployment.md)）。

## 4. 配置步骤

```bash
cp .env.example .env.local
```

编辑 `.env.local`：

```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project-id.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# 以下按需追加
# NEXT_PUBLIC_SITE_URL=https://your-domain.example
# NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION=your-verification-token
# PORT=3000
```

变量取值可在 Supabase 项目控制台的 **Settings → API** 中获取。

## 5. 已废弃变量（迁移参考）

项目早期使用 Firebase 做后端，以下变量已废弃且**不被任何运行时代码读取**，仅供迁移参考：

```env
# FIREBASE_PROJECT_ID=
# FIREBASE_CLIENT_EMAIL=
# FIREBASE_PRIVATE_KEY=
```

请勿在新部署中使用。

## 6. 安全注意事项

- `SUPABASE_SERVICE_ROLE_KEY` 拥有数据库完全读写权限，**只能**出现在服务端代码或托管平台的环境变量中，绝不可加 `NEXT_PUBLIC_` 前缀。
- 前端通过 `NEXT_PUBLIC_` 变量访问 Supabase 时，所有数据访问都应受 **Row Level Security (RLS)** 策略保护（见 `supabase/schema.sql`）。
- 切勿将 `.env.local` 提交到版本库（已在 `.gitignore` 中排除）。
- 查看状态与手动探测对访客开放，Google OAuth 仅用于登录；不要把 `ANON_KEY` 当作访问控制手段。
