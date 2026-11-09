# LLM API Sentinel v2.10.35

[English](README.md) | [中文](README_CN.md)

> Version baseline: v2.10.35 · Last audited: 2026-11-09 · Single source of truth: `package.json`

Real-time reachability and latency monitoring for 29 LLM API endpoints worldwide, with historical trend tracking.

## Design System

- **Theme**: Light/dark dual themes driven by CSS variables (`:root` / `.light` / `.dark` in `app/style.css`); `next-themes` with `defaultTheme="system"` / `enableSystem`. The dark tier uses indigo (`#6366f1`) + violet (`#8b5cf6`) accents.
- **Components**: 16 shadcn/ui-style base components in `app/components/ui/`, plus dashboard widgets in `app/components/`.
- **Visuals**: 3-layer radial-gradient body background, status pulse shadows, `fade-in-up` / `slide-in-right` / `shimmer` / `glow-pulse` animations, `motion-safe:` variants for reduced motion.
- **Responsive**: `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4`; main container `max-w-[1400px]`.
- **Prototypes**: `prototype/prototype.html` (high fidelity), `prototype/wireframes.html`, `prototype/components.html`, `prototype/index.html`.

## Features

- **Global Monitoring**: 29 endpoints — 16 US/global providers and 13 China providers. `app/constants/apis.json` is the single source of truth; `scripts/sync-apis.mjs` mirrors it to `supabase/functions/monitor/apis.json`, and `pnpm build` / CI verify the two are identical.
- **Historical Charts**: Recharts `AreaChart` (not hand-written SVG), capped at `CHART_DATA_LIMIT = 50` points; selectable ranges 1H / 6H / 24H.
- **Adaptive UI**: Light/dark dual theme, 1/2/3/4-column responsive grid, flat status grid (not grouped by provider).
- **Status Updates**: first-screen read of `api_status` from Supabase, then browser-side probing every 5 minutes (`CHECK_INTERVAL`). Only the `alerts` table uses Supabase Realtime. Manual checks are open to guests — no login required.
- **Authentication**: Supabase Auth with Google OAuth is used for signing in only; read-only probes are not gated.
- **Smart Alerts**: `createAlertForResult()` creates `critical` (offline) or `high` (latency > 1500ms) alerts, deduplicated by `api_id` + `resolved = false`. Webhook fan-out (Slack / Teams / Discord / DingTalk / Feishu / generic Webhook) is opt-in and disabled by default.
- **Optional Self-host Server**: `examples/self-host-server.ts` adds Helmet CSP/security headers, `Permissions-Policy`, a global per-IP rate limit of 100 requests / 60 seconds (memory or Redis), and an optional 5-minute background monitor.
- **Performance**: `React.memo` shallow comparison on the chart, `useMemo`, `MAX_CONCURRENT_REQUESTS = 5`, 2 retries with a fixed 1000ms delay (not exponential backoff).
- **Geolocation**: opt-in consent dialog, 24-hour local cache.
- **Caching**: three layers — memory + `localStorage` + `sessionStorage` (`CACHE_EXPIRY` 30s default, 5s–60s range).
- **Internationalization**: 16 locales (en, zh-cn, zh-tw, ar, cs, es, hi, id, it, nl, pl, sv, th, tr, ru, vi), automatic browser detection and manual switching. Note: `<html lang>` is currently fixed to `en`; switching updates UI copy only.
- **SEO / Analytics**: metadata routes `app/robots.ts` / `app/sitemap.ts`, JSON-LD structured data, GA4 (`G-0WKWY6YELE`).

## Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js `^14.2.13` (App Router, `output: 'export'`), React `^18.2.0` |
| Runtime / Package manager | Node.js 22 in CI; pnpm `9.15.0` enforced by `preinstall` (`only-allow`) and `.npmrc` |
| Styling | Tailwind CSS `4.1.11` + shadcn/ui-style components, Lucide React, `motion ^12.23.24` |
| Charts | Recharts `^3.8.0` |
| State / Utils | Zustand `^5.0.12`, `date-fns ^4.1.0` |
| Backend | Supabase (`@supabase/supabase-js ^2.107.0`): PostgreSQL, Auth (Google OAuth), Realtime, Cron + Deno Edge Function `monitor` |
| Optional server | Express `^5.2.1` + Helmet `^8.3.0` (examples only) |
| Tests / Types | Jest `^30.3.0` + ts-jest + Testing Library, TypeScript `5.9.3` (strict) |

## Architecture

Default shape: **static frontend + Supabase backend**. `next.config.mjs` exports to `out/`; no Next.js custom server and no Next.js API routes are required.

```
┌──────────────────┐        ┌───────────────────────────────────────┐
│  Static Hosting  │───────▶│              Supabase                 │
│  (out/ export)   │        │  PostgreSQL (api_status, history,     │
│  Vercel/EdgeOne  │        │  alerts) + Auth + Realtime (alerts)   │
└──────────────────┘        └───────────────┬───────────────────────┘
         ▲                                  │ pg_cron every 5 min
         │                          ┌──────▼────────────────────┐
         │                          │ Edge Function `monitor`    │
         │                          │ POST /monitor → 29 probes  │
         │                          └────────────────────────────┘
         │  optional self-host
    ┌────┴─────────────────────────────┐
    │ examples/self-host-server.ts     │
    │ Express + Helmet + rate limiting │
    └──────────────────────────────────┘
```

- **Frontend**: static export to `out/`; browser-side probing via `useApiMonitor` (initial probe on mount, then every 5 minutes); `alerts` via Realtime subscription.
- **Backend (optional)**: `supabase/functions/monitor/index.ts` is triggered by `supabase/cron.sql` (`pg_cron`, every 5 minutes, `POST /monitor`) and returns `{ ok, checked, offline, degraded, at }`. It is an example that must be deployed and scheduled manually.
- **Authentication**: Supabase Auth with Google OAuth; not required to view statuses or run a manual check.

## Getting Started

### Prerequisites

- Node.js 22 recommended (CI uses 22; `package.json` declares no `engines.node` constraint — Node 18+ generally works).
- pnpm 9.15.0 (`npm`/`yarn` installs are rejected by `preinstall`).
- A Supabase project (Google OAuth credentials only if you need sign-in).

### Environment Setup

```bash
cp .env.example .env.local
```

```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project-id.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

> See [docs/env.md](docs/env.md) for the full reference, including `NEXT_PUBLIC_SITE_URL`, `NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION`, `PORT`, `RATE_LIMIT_REDIS_*` and the Edge Function secrets. Firebase variables are deprecated and kept only as migration notes.

### Database (optional for local probing)

1. Create a Supabase project at [supabase.com](https://supabase.com).
2. Run `supabase/schema.sql` (tables + RLS policies).
3. Enable Google OAuth in Supabase Auth if sign-in is needed.
4. Optional: run `supabase/cron.sql` and deploy the background monitor:
   ```bash
   supabase functions deploy monitor --no-verify-jwt
   ```

### Install and Run

```bash
pnpm install
pnpm dev          # http://localhost:3000
pnpm preflight    # verify apis.json single-source consistency
pnpm test         # Jest with coverage
pnpm lint
pnpm build        # runs preflight, then static export to out/
```

### Deployment

Configured for **static export** (`output: 'export'` in `next.config.mjs`):

- **Tencent Cloud EdgeOne Pages** — `edgeone.config.js` + `edgeone.json` (install/build/output).
- **Vercel** — `vercel.json`.
- **Netlify** — build `pnpm build`, publish `out/`.

```bash
pnpm build   # outputs static files to out/
```

For optional self-hosting with Helmet headers + rate limiting, use `examples/self-host-server.ts` instead of static hosting:

```bash
pnpm build
node examples/self-host-server.ts
```

> This example wraps Next.js in custom-server mode and is **not compatible** with `output: 'export'` — switch `next.config.mjs` to `standalone` (or remove `output`) first. It is TypeScript, so it needs Node 22+ type stripping or a compiled JS build; no `tsx`/`ts-node` runner is declared in `package.json`.

## API Monitoring Configuration

| Constant (`app/constants/index.ts`) | Value | Meaning |
|-------------------------------------|-------|---------|
| `LATENCY_THRESHOLD` | 1500ms | latency above this generates a `high` alert |
| `DEGRADED_THRESHOLD` | 1000ms | latency above this marks a probe as `degraded` |
| `MAX_RETRIES` | 2 | retries per failed probe |
| `RETRY_DELAY` | 1000ms | fixed delay (no exponential backoff) |
| `MAX_CONCURRENT_REQUESTS` | 5 | concurrency cap |
| `CHECK_INTERVAL` | 5 minutes | browser-side re-probe interval |
| `CACHE_EXPIRY` | 30s (5s–60s) | default cache TTL |
| `CHART_DATA_LIMIT` | 50 | max chart points |
| `GEO_INFO_EXPIRY` | 24 hours | geolocation cache |

Probe semantics: `response.status < 500` counts as reachable (`isOnline`), so auth-required endpoints returning 401/403 may still be reported online.

## Supported APIs

The full list lives in `app/constants/apis.json` (29 entries). It is mirrored to `supabase/functions/monitor/apis.json` by `pnpm sync:apis`.

### US / Global providers (16)

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

### China providers (13)

- Moonshot Kimi V1
- ZhipuAI GLM-4
- Baichuan 2
- Alibaba Qwen Max
- Tencent Hunyuan
- Baidu Ernie 4.0
- DeepSeek V3
- MiniMax
- iFlytek Spark
- ByteDance Doubao
- 01.AI Yi
- SiliconFlow
- StepFun

## Internationalization

Supported locales: English (en), 简体中文 (zh-cn), 繁體中文 (zh-tw), العربية (ar), Čeština (cs), Español (es), हिन्दी (hi), Bahasa Indonesia (id), Italiano (it), Nederlands (nl), Polski (pl), Svenska (sv), ไทย (th), Türkçe (tr), Русский (ru), Tiếng Việt (vi). Locale files are in `app/locales/`; unknown keys fall back to `en`, then to the key itself. The preference is persisted in `localStorage.locale` with whitelist validation.

## Project Structure

```
├── app/
│   ├── components/       # 26 dashboard widgets + components/ui/ (16 base components)
│   ├── constants/        # apis.json (single source) + index.ts (thresholds)
│   ├── hooks/            # 9 hooks (monitor, alerts, auth, geo, i18n, stats...)
│   ├── lib/              # 33 modules (probe, cache, metrics, i18n, webhook...)
│   ├── locales/          # 16 JSON locale files
│   ├── store/            # Zustand slices (api/auth/alerts/error/geo)
│   ├── types/            # TypeScript type definitions
│   ├── layout.tsx / page.tsx / robots.ts / sitemap.ts
│   └── globals.css / style.css
├── docs/                 # env, deployment, security, contributing, roadmap
├── openspec/             # architecture, data, design-system, features, logic, project, ui
│   ├── specs/            # spec increments (currently empty)
│   └── changes/          # historical change proposals
├── prototype/            # 4 HTML prototypes + assets/
├── public/               # manifest.json, robots.txt, sitemap.xml
├── scripts/              # sync-apis.mjs, add_missing_translations.cjs
├── supabase/             # schema.sql, cron.sql, functions/monitor/{index.ts, apis.json}
├── examples/             # self-host-server.ts (optional Express server)
├── .github/              # workflows/{ci,release}.yml + community health files
├── .env.example
├── next.config.mjs / edgeone.config.js / edgeone.json / vercel.json
├── jest.config.cjs / eslint.config.mjs / tsconfig.json / postcss.config.mjs
├── components.json / package.json
└── CHANGELOG.md / README.md / README_CN.md / LICENSE
```

## Documentation

- [openspec/](openspec/) — Architecture, data model, UI, features, logic, project conventions and change proposals
- [docs/env.md](docs/env.md) — Environment variables reference
- [docs/deployment.md](docs/deployment.md) — Deployment guide (Vercel / EdgeOne / Netlify / self-host)
- [docs/security.md](docs/security.md) — Security architecture and best practices
- [docs/contributing.md](docs/contributing.md) — Contribution guidelines
- [docs/roadmap.md](docs/roadmap.md) — Roadmap and improvement proposals
- [CHANGELOG.md](CHANGELOG.md) — Release history

## License

MIT
