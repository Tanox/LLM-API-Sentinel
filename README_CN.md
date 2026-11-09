# LLM API Sentinel v2.10.35

[English](README.md) | [中文](README_CN.md)

> 适用版本：v2.10.35 · 最近校对：2026-11-09 · 版本单一真源：`package.json`

全球 29 个大模型 API 端点的连通性、延迟实时监控与历史趋势追踪系统。

## 设计系统

- **主题**：浅色/深色双主题，由 `app/style.css` 的 `:root` / `.light` / `.dark` CSS 变量驱动；`next-themes` 使用 `defaultTheme="system"` / `enableSystem`，默认跟随系统。深色档使用靛蓝（`#6366f1`）+ 紫罗兰（`#8b5cf6`）强调色。
- **组件**：`app/components/ui/` 下 16 个 shadcn/ui 风格基础组件，`app/components/` 下为仪表盘业务组件。
- **视觉**：三层径向渐变背景、状态点光晕阴影、`fade-in-up` / `slide-in-right` / `shimmer` / `glow-pulse` 动画，并通过 `motion-safe:` 适配减少动态效果偏好。
- **响应式**：`grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4`；主容器 `max-w-[1400px]`。
- **原型**：`prototype/prototype.html`（高保真）、`prototype/wireframes.html`、`prototype/components.html`、`prototype/index.html`。

## 核心功能

- **全球监控**：29 个端点（美国/全球 16 家 + 中国 13 家）。`app/constants/apis.json` 为单一真源，`scripts/sync-apis.mjs` 同步到 `supabase/functions/monitor/apis.json`；`pnpm build` 与 CI 会校验两者一致。
- **历史图表**：使用 Recharts `AreaChart`（非手写 SVG），数据点上限 `CHART_DATA_LIMIT = 50`；时间范围 1H / 6H / 24H。
- **自适应 UI**：浅/深双主题、1/2/3/4 列响应式网格，状态网格为扁平布局（不按供应商分组）。
- **状态更新**：首屏从 Supabase 读取 `api_status`，随后由浏览器每 5 分钟主动探测（`CHECK_INTERVAL`）；仅 `alerts` 表使用 Supabase Realtime 订阅。手动检查对访客开放，无需登录。
- **身份验证**：Supabase Auth（Google OAuth）仅用于登录，不限制只读探测。
- **智能告警**：`createAlertForResult()` 生成 `critical`（离线）或 `high`（延迟 > 1500ms）告警，按 `api_id` + `resolved = false` 去重。Webhook 推送（Slack / Teams / Discord / 钉钉 / 飞书 / 通用 Webhook）默认关闭、按需启用。
- **可选自建服务器**：`examples/self-host-server.ts` 提供 Helmet CSP 安全头、`Permissions-Policy`、全局按 IP 限流（100 次 / 60 秒，内存或 Redis）以及可选的 5 分钟后台巡检。
- **性能优化**：图表 `React.memo` 浅比较、`useMemo`、`MAX_CONCURRENT_REQUESTS = 5`、失败重试 2 次且固定 1000ms 延迟（非指数退避）。
- **地理位置**：用户明确授权（opt-in）后才获取，本地缓存 24 小时。
- **缓存系统**：内存 + `localStorage` + `sessionStorage` 三层缓存（`CACHE_EXPIRY` 默认 30 秒，范围 5 秒–60 秒）。
- **国际化**：16 种语言（en、zh-cn、zh-tw、ar、cs、es、hi、id、it、nl、pl、sv、th、tr、ru、vi），自动检测浏览器语言并支持手动切换。注意：`<html lang>` 当前固定为 `en`，切换仅影响界面文案。
- **SEO / 统计**：`app/robots.ts` / `app/sitemap.ts` 元数据路由、JSON-LD 结构化数据、GA4（`G-0WKWY6YELE`）。

## 技术栈

| 层次 | 技术 |
|------|------|
| 框架 | Next.js `^14.2.13`（App Router，`output: 'export'`）、React `^18.2.0` |
| 运行时 / 包管理器 | CI 使用 Node.js 22；pnpm `9.15.0`，由 `preinstall`（`only-allow`）与 `.npmrc` 强制约束 |
| 样式 | Tailwind CSS `4.1.11` + shadcn/ui 风格组件、Lucide React、`motion ^12.23.24` |
| 图表 | Recharts `^3.8.0` |
| 状态 / 工具 | Zustand `^5.0.12`、`date-fns ^4.1.0` |
| 后端 | Supabase（`@supabase/supabase-js ^2.107.0`）：PostgreSQL、Auth（Google OAuth）、Realtime、Cron + Deno Edge Function `monitor` |
| 可选服务器 | Express `^5.2.1` + Helmet `^8.3.0`（仅 `examples/`） |
| 测试 / 类型 | Jest `^30.3.0` + ts-jest + Testing Library，TypeScript `5.9.3`（strict） |

## 系统架构

默认形态为 **静态前端 + Supabase 后端**：`next.config.mjs` 导出到 `out/`，不需要 Next.js 自定义服务器，也没有 Next.js API 路由。

```
┌──────────────────┐        ┌───────────────────────────────────────┐
│   静态托管服务    │───────▶│              Supabase                 │
│   (out/ 导出)    │        │  PostgreSQL（api_status / history /   │
│  Vercel/EdgeOne  │        │  alerts）+ Auth + Realtime（alerts）  │
└──────────────────┘        └───────────────┬───────────────────────┘
         ▲                                  │ pg_cron 每 5 分钟
         │                          ┌──────▼────────────────┐
         │                          │ Edge Function `monitor`│
         │                          │ POST /monitor → 29 探测│
         │                          └────────────────────────┘
         │  可选自建模式
    ┌────┴─────────────────────────────┐
    │ examples/self-host-server.ts     │
    │ Express + Helmet + 全局限流      │
    └──────────────────────────────────┘
```

- **前端**：静态导出到 `out/`；`useApiMonitor` 挂载时先探测一次，之后每 5 分钟定时探测；`alerts` 通过 Realtime 订阅。
- **后端（可选）**：`supabase/functions/monitor/index.ts` 由 `supabase/cron.sql`（`pg_cron`，每 5 分钟，`POST /monitor`）触发，返回 `{ ok, checked, offline, degraded, at }`。该示例需手动部署并注册定时任务。
- **身份验证**：Supabase Auth + Google OAuth；查看状态或手动探测均不需要登录。

## 快速开始

### 前置条件

- 推荐 Node.js 22（CI 使用 22；`package.json` 未声明 `engines.node`，一般 Node 18+ 亦可用）。
- pnpm 9.15.0（`preinstall` 会拒绝 `npm` / `yarn` 安装）。
- Supabase 项目（仅在需要登录功能时才需 Google OAuth 凭证）。

### 环境配置

```bash
cp .env.example .env.local
```

```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project-id.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

> 完整变量说明见 [docs/env.md](docs/env.md)，包含 `NEXT_PUBLIC_SITE_URL`、`NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION`、`PORT`、`RATE_LIMIT_REDIS_*` 及 Edge Function secrets。Firebase 变量已废弃，仅保留迁移注释。

### 数据库（本地探测可选）

1. 在 [supabase.com](https://supabase.com) 创建项目。
2. 运行 `supabase/schema.sql`（建表 + RLS 策略）。
3. 如需登录，在 Supabase Auth 中启用 Google OAuth。
4. 可选：运行 `supabase/cron.sql` 并部署后台监控：
   ```bash
   supabase functions deploy monitor --no-verify-jwt
   ```

### 安装与运行

```bash
pnpm install
pnpm dev          # http://localhost:3000
pnpm preflight    # 校验 apis.json 单一真源一致性
pnpm test         # Jest（含覆盖率）
pnpm lint
pnpm build        # 先执行 preflight，再静态导出到 out/
```

### 部署

已配置为 **静态导出**（`next.config.mjs` 中 `output: 'export'`）：

- **腾讯云 EdgeOne Pages** — `edgeone.config.js` + `edgeone.json`（安装/构建/输出目录）。
- **Vercel** — `vercel.json`。
- **Netlify** — 构建 `pnpm build`，发布目录 `out/`。

```bash
pnpm build   # 静态文件输出到 out/
```

如需自托管并启用 Helmet 安全头与限流，改用 `examples/self-host-server.ts`（二选一，不可同时静态托管）：

```bash
pnpm build
node examples/self-host-server.ts
```

> 该示例以自定义服务器模式包装 Next.js，与 `output: 'export'` **不兼容**，需先把 `next.config.mjs` 改为 `standalone`（或移除 `output`）。文件为 TypeScript，需 Node 22+ 类型剥离或自行编译为 JS；`package.json` 未声明 `tsx` / `ts-node` 运行器。

## API 监控配置

| 常量（`app/constants/index.ts`） | 值 | 说明 |
|----------------------------------|-----|------|
| `LATENCY_THRESHOLD` | 1500ms | 超过此值生成 `high` 告警 |
| `DEGRADED_THRESHOLD` | 1000ms | 超过此值标记为 `degraded` |
| `MAX_RETRIES` | 2 | 单次探测失败重试次数 |
| `RETRY_DELAY` | 1000ms | 固定延迟（非指数退避） |
| `MAX_CONCURRENT_REQUESTS` | 5 | 并发上限 |
| `CHECK_INTERVAL` | 5 分钟 | 前端定时探测间隔 |
| `CACHE_EXPIRY` | 30 秒（5 秒–60 秒） | 默认缓存有效期 |
| `CHART_DATA_LIMIT` | 50 | 图表数据点上限 |
| `GEO_INFO_EXPIRY` | 24 小时 | 地理位置缓存 |

探测语义：`response.status < 500` 即视为端点可达（`isOnline`），因此返回 401/403 的需鉴权端点仍可能被判为在线。

## 支持的 API

完整清单以 `app/constants/apis.json` 为准（29 条），并由 `pnpm sync:apis` 同步至 `supabase/functions/monitor/apis.json`。

### 美国 / 全球供应商（16）

- OpenAI GPT-4o
- Anthropic Claude 3.5
- Google Gemini 1.5
- Meta Llama 3 (Groq)
- Mistral Large
- xAI Grok
- Cohere Command R
- Perplexity Sonar
- Together AI Llama
- Replicate
- Stability AI
- Hugging Face
- OpenRouter
- Fireworks AI
- NVIDIA NIM
- AI21 Labs

### 中国供应商（13）

- Moonshot Kimi V1
- 智谱 GLM-4
- 百川 2
- 阿里通义千问 Max
- 腾讯混元
- 百度文心一言 4.0
- DeepSeek V3
- MiniMax
- 讯飞星火
- 字节豆包
- 零一万物 Yi
- SiliconFlow
- StepFun（阶跃星辰）

## 国际化

支持语言：English (en)、简体中文 (zh-cn)、繁體中文 (zh-tw)、العربية (ar)、Čeština (cs)、Español (es)、हिन्दी (hi)、Bahasa Indonesia (id)、Italiano (it)、Nederlands (nl)、Polski (pl)、Svenska (sv)、ไทย (th)、Türkçe (tr)、Русский (ru)、Tiếng Việt (vi)。语言文件位于 `app/locales/`；缺失词条回退英文，再回退词条键本身。语言偏好经白名单校验后持久化到 `localStorage.locale`。

## 项目结构

```
├── app/
│   ├── components/       # 26 个仪表盘组件 + components/ui/（16 个基础组件）
│   ├── constants/        # apis.json（单一真源）+ index.ts（阈值常量）
│   ├── hooks/            # 9 个 hooks（监控、告警、鉴权、地理、i18n、统计…）
│   ├── lib/              # 33 个模块（探测、缓存、指标、i18n、Webhook…）
│   ├── locales/          # 16 个 JSON 语言包
│   ├── store/            # Zustand 分片（api/auth/alerts/error/geo）
│   ├── types/            # TypeScript 类型定义
│   ├── layout.tsx / page.tsx / robots.ts / sitemap.ts
│   └── globals.css / style.css
├── docs/                 # 环境变量、部署、安全、贡献指南、路线图
├── openspec/             # 架构、数据、设计系统、功能、逻辑、项目规范、UI
│   ├── specs/            # 规范增量（当前为空）
│   └── changes/          # 历史变更提案
├── prototype/            # 4 个 HTML 原型 + assets/
├── public/               # manifest.json、robots.txt、sitemap.xml
├── scripts/              # sync-apis.mjs、add_missing_translations.cjs
├── supabase/             # schema.sql、cron.sql、functions/monitor/{index.ts, apis.json}
├── examples/             # self-host-server.ts（可选 Express 服务器）
├── .github/              # workflows/{ci,release}.yml + 社区健康文件
├── .env.example
├── next.config.mjs / edgeone.config.js / edgeone.json / vercel.json
├── jest.config.cjs / eslint.config.mjs / tsconfig.json / postcss.config.mjs
├── components.json / package.json
└── CHANGELOG.md / README.md / README_CN.md / LICENSE
```

## 相关文档

- [openspec/](openspec/) — 架构、数据模型、UI、功能、逻辑、项目规范与变更提案
- [docs/env.md](docs/env.md) — 环境变量参考
- [docs/deployment.md](docs/deployment.md) — 部署指南（Vercel / EdgeOne / Netlify / 自托管）
- [docs/security.md](docs/security.md) — 安全架构与最佳实践
- [docs/contributing.md](docs/contributing.md) — 贡献指南
- [docs/roadmap.md](docs/roadmap.md) — 发展路线与改进建议
- [CHANGELOG.md](CHANGELOG.md) — 版本变更记录

## 许可证

MIT
