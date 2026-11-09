# 安全架构 (Security)

> 适用版本：v2.10.35 · 最近校对：2026-11-09

本文档汇总 LLM API Sentinel 的安全设计与最佳实践。漏洞报告流程见 [.github/SECURITY.md](../.github/SECURITY.md)。

## 1. 认证与授权

- **登录方式**：Supabase Auth + **Google OAuth**，仅用于用户登录；查看状态、手动健康检查（只读探测）对访客开放，代码不校验登录态（`DashboardClient.tsx` 的 `canRunCheck = !isChecking`）。
- **客户端访问**：前端使用 `NEXT_PUBLIC_SUPABASE_ANON_KEY`（匿名公钥），数据库访问受 **Row Level Security (RLS)** 约束（`supabase/schema.sql`）。
- **服务端访问**：后台写入使用 `SUPABASE_SERVICE_ROLE_KEY`（Edge Function 由平台自动注入），绕过 RLS，**仅用于服务端**，绝不可暴露到浏览器或加 `NEXT_PUBLIC_` 前缀。

## 2. 数据保护

- **密钥隔离**：`.env.local` 已被 `.gitignore` 忽略；公开变量加 `NEXT_PUBLIC_` 前缀，私密变量不加。
- **无硬编码密钥**：代码库不包含真实 API Key / Token / 密码；`.env.example` 仅含占位值。
- **输入校验**：`ApiConfig`（自定义监控目标）与 `app/constants/index.ts` 的 `isValidApiConfigItem()` 采用手写白名单校验（`id/name/provider` 非空、`url` 必须以 `https://` 开头），**未使用 zod / `safeParse`**；Edge Function 只读取白名单 `apis.json`，不接受外部 URL 参数，以防 SSRF。
- **无演示数据**：Supabase 未配置或加载失败时保持空态并主动真实探测，不注入模拟数据。

## 3. 传输与响应头安全

`edgeone.config.js` 与 `vercel.json` 为静态资源附加安全响应头：

| 响应头 | 值 | 来源 |
|--------|-----|------|
| `X-Content-Type-Options` | `nosniff` | EdgeOne / Vercel |
| `X-Frame-Options` | `DENY` | EdgeOne / Vercel + `app/layout.tsx` meta |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | EdgeOne / Vercel |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | EdgeOne / Vercel |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | 可选 Express 示例与平台配置 |

`X-XSS-Protection` 未显式配置（现代浏览器已弃用该头），请勿按旧文档假设其存在。可选自建模式（`examples/self-host-server.ts`）额外通过 Helmet 设置 CSP（含 `frame-ancestors 'none'`）等头部。

## 4. 速率限制（自建模式）

`examples/self-host-server.ts` 对**全部请求**施加按 IP 的全局限流：默认 **100 次 / 60 秒**（`MemoryRateLimiter`，窗口 60 秒、上限 100）；设置 `RATE_LIMIT_REDIS_URL` / `RATE_LIMIT_REDIS_TOKEN` 时切换为 `RedisRateLimiter`。代码中没有 `/api/check` 路由，也没有针对手动检查的专用限流规则。

## 5. 静态导出与攻击面

- 默认静态导出（`output: 'export'`），前端不包含任何 Next.js 服务端 API 路由（仅 `robots.ts` / `sitemap.ts` 元数据路由）；攻击面限于静态资源与 Supabase 接口。
- 静态资源（JS/CSS）设置长缓存 `public, max-age=31536000, immutable`；EdgeOne 的 HTML 为 `public, max-age=0, must-revalidate`，Vercel 仅对 `/_next/static` 设置 immutable，两平台策略不同。
- 后台 Edge Function `monitor` 以 `--no-verify-jwt` 部署，属于示例配置：应仅由受信任的 `pg_cron` 调用，或补充签名校验后再对外开放。

## 6. 已废弃组件

- Firebase 后端已迁移至 Supabase；`.env.example` 与 [env.md](env.md) 仅保留 3 个 Firebase 变量名作为迁移参考，不再被任何运行时代码读取。
- `app/lib/mock-data.ts`（模拟数据）已删除；`netlify.toml` 与 `security_best_practices_report.md` 在仓库中不存在。

## 7. 安全清单（部署前自查）

- [ ] `.env.local` 未提交到版本库
- [ ] `SUPABASE_SERVICE_ROLE_KEY` 仅存在于服务端/托管环境变量
- [ ] Supabase RLS 策略已启用并覆盖 `api_status` / `status_history` / `alerts` / `user_profiles`
- [ ] Google OAuth 回调地址已加入 Supabase 白名单（如启用登录）
- [ ] 生产环境使用 HTTPS
- [ ] 若开放 `/monitor`，已限制调用来源或补充鉴权
