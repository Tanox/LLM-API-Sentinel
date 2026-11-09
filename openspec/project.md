# LLM API Sentinel 项目规范 (v2.10.35)

> 适用版本：v2.10.35 · 最近校对：2026-11-09 · 真源：`app/`、`supabase/` 与 `package.json`
>
> **代码对齐要点（下文旧表述与本表冲突时，以本表及源码为准）**：
> - 版本单一真源为 `package.json`（2.10.35）；依赖版本以 `^` 范围为准；包管理器 pnpm 9.15.0、CI Node.js 22；覆盖率门禁为 70 / 55 / 70 / 70（`jest.config.cjs`）。
> - 29 个 API 的权威清单为 `app/constants/apis.json`（16 美国/全球 + 13 中国），由 `scripts/sync-apis.mjs` 同步到 Edge Function；`app/lib/mock-data.ts` 已删除。
> - 默认后台为 Supabase Cron → `POST /monitor`（返回 `{ ok, checked, offline, degraded, at }`）；Express 为可选自建模式；前端每 5 分钟定时探测，Realtime 仅用于 `alerts`。
> - 组件 Props 以源码为准：`DashboardHeaderProps` 的 `geo` 为内联结构且 `isGeoLoading` / `refreshGeo` / `resolveAlert` 可选；`ApiStatusGridProps` 含 `isChecking?`；`StatusDotProps` 无 `size`；`ProgressBarProps` 的 `variant` 必填且有 `showLabel?`；`AlertsDropdownProps.resolveAlert` 返回 `void`。
> - 日志为结构化 JSON；错误码为 `SUPABASE_QUERY_ERROR`（无 `DATABASE_ERROR`）；`Alert.timestamp` 为 `Date | string | number`；重试为固定 1000ms 延迟；主题为浅色/深色双主题。
> - CI 步骤：`pnpm install --frozen-lockfile` → `node scripts/sync-apis.mjs --check` → `pnpm lint` → `pnpm test -- --coverage` → `pnpm build`（无独立 `tsc --noEmit` 步骤）；部署用 `supabase functions deploy monitor --no-verify-jwt`。

## 1. 项目概述

LLM API Sentinel 是一个全球主流大模型 API 实时监控与历史可用性追踪系统，旨在为开发者和企业提供可靠的 API 状态监控服务。

### 1.1 核心功能
- **全球监控**：追踪美国（OpenAI, Anthropic, Google, Meta/Groq, Mistral）和中国（Moonshot/Kimi, ZhipuAI, Baichuan, Alibaba/Qwen, Tencent/Hunyuan, Baidu/Ernie, DeepSeek）主流 AI 供应商的连通性与延迟，共 29 个核心 API
- **历史数据**：Recharts 面积图可视化性能趋势；支持 1H/6H/24H 时间范围（`TIME_RANGES: lastHour/last6Hours/last24Hours`），曲线末点锚定当前实时延迟（`app/components/LatencyHistoryChart.tsx`，静态导入（页面级 Suspense 骨架由 DashboardSkeleton 提供））
- **自适应 UI**：全响应式设计（1/2/3/4 列网格），采用浅色/深色双主题（next-themes `defaultTheme="system"`），浅色/深色双主题（默认跟随系统，深色档为 Dark Indigo）（靛蓝 #6366f1 + 紫色 #8b5cf6）
- **实时更新**：首屏读取 api_status + 前端 5 分钟定时探测；仅 alerts 使用 Realtime（默认 5 分钟后台检查周期）
- **安全访问**：手动健康检查对访客开放，无需登录（Google OAuth 仅用于登录）
- **智能告警**：自动检测 API 宕机（offline）、降级（degraded）和延迟过高（阈值 1500ms），并生成告警通知
- **数据缓存**：内存 + localStorage 双层缓存机制（默认 30 秒缓存，动态 5 秒- 1 分钟）
- **地理位置**：检测监控节点位置，24 小时本地缓存（ipapi.co）
- **多语言支持**：16 种语言国际化（en / zh-cn / zh-tw / ar / cs / es / hi / id / it / nl / pl / sv / th / tr / ru / vi），自动检测浏览器语言
- **SEO 优化**：Schema.org 结构化数据、多语言 SEO 支持、Open Graph / Twitter Card meta
- **性能优化**：Next.js App Router + Server Components、动态导入、React.memo、useMemo

### 1.2 技术栈
| 类别 | 技术 | 版本 |
|-----|------|------|
| 前端框架 | Next.js 14.2.13 (App Router, Static Export) | 14.2.13 |
| 后端服务器 | Express 5.2.1 | 5.2.1 |
| 数据库 | Supabase PostgreSQL | - |
| 身份验证 | Supabase Auth (Google OAuth) | - |
| 实时订阅 | Supabase Realtime | - |
| 样式 | Tailwind CSS 4.1.11 | 4.1.11 |
| 组件库 | shadcn/ui (基于 Tailwind) | - |
| 图表 | Recharts 3.8.0 面积图（静态导入） | 3.8.0 |
| 图标 | Lucide React | - |
| 状态管理 | Zustand 5.0.12 | 5.0.12 |
| 设计系统 | [design-system.md](design-system.md) | v2.10.35 |
| 国际化 | 自定义 i18n 系统 | - |
| 时间处理 | date-fns 4.1.0 | 4.1.0 |

### 1.3 关键常量
| 常量 | 值 | 含义 |
|-----|-----|-----|
| `LATENCY_THRESHOLD` | 1500ms | 延迟过高告警阈值 |
| `DEGRADED_THRESHOLD` | 1000ms | 降级状态阈值 |
| `MAX_RETRIES` | 2 | 离线 API 重试次数 |
| `MAX_CONCURRENT_REQUESTS` | 5 | 最大并发请求数 |
| `CHART_DATA_LIMIT` | 50 | 图表数据点限制 |
| `CHECK_INTERVAL` | 5 * 60 * 1000ms | 后台检查周期 |
| `GEO_INFO_EXPIRY` | 24 小时 | 地理位置缓存有效期 |
| `CACHE_EXPIRY` | 30 秒 | 响应数据默认缓存 |

### 1.4 系统架构
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Next.js App   │────▶│    Supabase     │◀────│  Express Server │
│  (Static HTML)  │     │(Auth + Postgres)│   │  (Background)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                      ▲                         │
        │                      │                         ▼
        └───── Realtime  ──────┘             ┌─────────────────┐
                                                │  External LLM  │
                                                │  APIs (29)     │
                                                └─────────────────┘
```

## 2. 代码规范

### 2.1 文件结构
```
├── app/
│   ├── components/
│   │   ├── ui/                   # shadcn/ui 基础组件 (16 个, new-york 风格)
│   │   │   ├── alert.tsx
│   │   │   ├── avatar.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── input.tsx
│   │   │   ├── label.tsx
│   │   │   ├── popover.tsx
│   │   │   ├── select.tsx
│   │   │   ├── separator.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── switch.tsx
│   │   │   ├── tabs.tsx
│   │   │   └── tooltip.tsx
│   │   ├── AlertsDropdown.tsx    # 告警下拉组件（shadcn Dialog）
│   │   ├── ApiConfig.tsx         # API 配置组件（localStorage 读写）
│   │   ├── ApiStatusGrid.tsx     # API 状态网格（主组件，扁平 <section> 网格直接渲染）
│   │   ├── ChartSkeleton.tsx     # 图表骨架屏占位（当前无引用）
│   │   ├── DashboardClient.tsx   # 仪表盘客户端主组件（'use client'）
│   │   ├── DashboardFooter.tsx   # 页脚组件（3 列 Feature 展示）
│   │   ├── DashboardHeader.tsx   # 头部组件（Logo/告警/主题/地理/语言切换/登录）
│   │   ├── LocaleSwitcher.tsx     # 语言切换器（DropdownMenuRadioGroup，16 语言即时切换 + 持久化）
│   │   ├── DashboardSkeleton.tsx # 仪表盘整体骨架屏
│   │   ├── GeoOptInDialog.tsx    # 地理位置授权弹窗
│   │   ├── LatencyHistoryChart.tsx # 延迟历史图表（Recharts 面积图，静态导入）
│   │   ├── ProgressBar.tsx       # 进度条组件（渐变 + shimmer）
│   │   ├── StatCard.tsx          # 统计卡片组件（在线/降级/离线/平均延迟）
│   │   ├── StatusDot.tsx         # 状态圆点组件（三色 + 光晕）
│   │   ├── StatusGrid.tsx        # 兼容层（转发给 ApiStatusGrid）
│   │   ├── StructuredData.tsx    # Schema.org 结构化数据（SEO）
│   │   └── ThemeProvider.tsx     # 主题提供者（next-themes）
│   ├── hooks/
│   │   ├── use-mobile.ts         # 移动端检测
│   │   ├── useAlerts.ts          # 告警管理（Supabase Realtime）
│   │   ├── useApiMonitor.ts      # API 监控逻辑（并发 + 重试 + 缓存）
│   │   ├── useAuth.ts            # 认证管理（Supabase Auth）
│   │   ├── useDashboardData.ts   # 仪表盘数据聚合（主 hook，组合各子 hook）
│   │   ├── useGeoLocation.ts     # 地理位置（ipapi.co + 24h 缓存）
│   │   └── useI18n.ts            # 国际化（16 语言，persistLocale 持久化 + 浏览器语言自动检测）
│   ├── lib/
│   │   ├── cache.ts              # 缓存逻辑（内存 + localStorage 双层 + 智能过期）
│   │   ├── concurrency.ts        # 并发控制（信号量，默认 5）
│   │   ├── error-handler.ts      # 错误处理（统一捕获 + 分类 + 日志）
│   │   ├── error.tsx             # 错误边界和通知组件
│   │   ├── error.test.ts         # 错误处理单元测试
│   │   ├── i18n.ts               # 国际化系统（语言资源加载）
│   │   ├── i18n.test.ts          # 国际化单元测试
│   │   ├── metrics.ts            # 指标计算（平均延迟、可用性等）
│   │   ├── metrics-storage.ts    # 累计指标持久化（可用性/延迟历史真实写入 localStorage，跨刷新累加）
│   │   ├── monitor.ts            # API 监控核心逻辑（检查 + 重试 + 指标）
│   │   ├── monitor.test.ts       # 监控逻辑单元测试
│   │   ├── notification.ts       # 通知处理
│   │   ├── supabase.ts           # Supabase 客户端配置
│   │   └── utils.ts              # 工具函数（cn / getApiColor / 等）
│   ├── store/                    # Zustand 状态管理 (5 个 store)
│   │   ├── index.ts              # Store 统一导出
│   │   ├── store.ts              # Store 类型定义
│   │   ├── api.ts                # API 状态 store
│   │   ├── auth.ts               # 认证 store
│   │   ├── alerts.ts             # 告警 store
│   │   ├── geo.ts                # 地理位置 store
│   │   └── error.ts              # 错误/通知 store
│   ├── locales/                  # 语言资源 JSON（16 种语言）
│   ├── types/index.ts            # 类型定义（按逻辑分组）
│   ├── constants/index.ts        # 常量与默认 API 配置（29 个 API）
│   ├── style.css                 # 全局样式（CSS 变量 + Tailwind @theme + 动画）
│   ├── globals.css               # 全局基础样式
│   ├── layout.tsx                # 根布局（Server Component，ThemeProvider + StructuredData）
│   └── page.tsx                  # 主页面（Server Component，Suspense + DashboardClient）
├── prototype/
│   ├── index.html                # 高保真可交互原型 · 仪表盘主页面（纯 HTML/CSS/JS）
│   ├── components.html           # 组件库规范展示页（色彩/字体/间距/组件）
│   └── assets/
│       ├── styles.css            # 设计 token + 动画（与 app/style.css 语义变量对齐）
│       ├── data.js               # 原型演示数据与 i18n（原型历史参考，非应用数据源）
│       └── app.js                # 共享逻辑（主题/语言/渲染/图表/交互）
├── openspec/
│   ├── README.md                 # OpenSpec 说明
│   ├── config.yaml               # OpenSpec 配置
│   ├── project.md                # 项目规范（当前文档）
│   ├── architecture.md           # 架构文档
│   ├── data.md                   # 数据模型与安全
│   ├── design-system.md          # 设计系统规范
│   ├── features.md               # 功能规格
│   ├── logic.md                  # 逻辑与服务
│   ├── ui.md                     # UI 组件规范
│   └── changes/                  # 变更提案目录
├── scripts/
│   └── add_missing_translations.cjs  # 翻译补全脚本
├── supabase/
│   └── schema.sql                # 数据库 Schema
├── public/
│   ├── manifest.json             # PWA manifest
│   ├── robots.txt                # 爬虫规则
│   └── sitemap.xml               # 站点地图
├── CHANGELOG.md                  # 版本变更记录
├── README.md                     # 项目说明（英文）
├── README_CN.md                  # 项目说明（中文）
├── package.json                  # 依赖与脚本
├── tsconfig.json                 # TypeScript 配置
├── next.config.mjs               # Next.js 配置
├── next-env.d.ts                 # Next.js 类型声明
├── components.json               # shadcn/ui 配置
├── eslint.config.mjs             # ESLint 配置
├── jest.config.cjs               # Jest 测试配置
├── jest.setup.cjs                # Jest 测试 setup
├── postcss.config.mjs            # PostCSS 配置
├── examples/self-host-server.ts  # Express 后台监控服务器（可选，自建模式）
├── vercel.json                   # Vercel 部署配置
├── edgeone.config.js             # EdgeOne 部署配置
└── .gitignore                    # Git 忽略规则
```

### 2.2 命名约定
| 类型 | 约定 | 示例 |
|-----|------|------|
| 文件命名 | 小写字母 + 连字符 | `api-status-grid.tsx` |
| 组件 | PascalCase | `ApiStatusGrid` |
| 函数 | camelCase | `performCheck` |
| 变量 | camelCase | `apiStatus` |
| 常量 | SCREAMING_SNAKE_CASE | `LATENCY_THRESHOLD` |
| 类型/接口 | PascalCase | `ApiStatus` |
| 翻译 Key | 小写 + 点分隔 | `dashboard.title` |

### 2.3 代码风格
- **缩进**：使用 2 个空格
- **分号**：使用分号结尾
- **引号**：使用单引号（JSX 中使用双引号）
- **行宽**：≤ 100 字符
- **注释**：
  - 文件头部必须包含单行版本信息（格式：`path/to/file.tsx vX.X.X`）
  - 函数必须有简洁的注释
  - 复杂逻辑必须有内联注释

### 2.4 类型定义
- 使用 TypeScript 类型系统
- 为所有函数参数和返回值添加类型注解
- 为复杂数据结构创建接口定义
- 禁止使用 `any` 类型

### 2.5 版本控制
- 使用 SemVer 2.0.0 版本规范
- 版本号格式：`主版本.次版本.修订号`（如 `2.6.2`）
- 每次发布时更新以下文件：
  1. 文件头部版本号（所有代码文件）
  2. HTML Title 标签版本号（`app/layout.tsx`）
  3. `package.json` 中的版本信息
  4. `CHANGELOG.md` 中的版本记录

## 3. API 接口文档

### 3.1 客户端 API

#### 3.1.1 useDashboardData Hook
**功能**：统一获取仪表盘所需的所有数据

**返回值**：
```typescript
{
  statuses: ApiStatus[];      // API 状态列表
  history: StatusHistory[];   // 历史数据
  alerts: Alert[];            // 告警列表
  user: User | null;          // 当前用户
  isChecking: boolean;        // 是否正在检查
  lastUpdate: Date | null;    // 最后更新时间
  geo: GeoLocation | null;    // 地理位置信息
  runCheck: () => Promise<void>;       // 执行检查
  resolveAlert: (id: string) => Promise<void>;  // 解决告警
  login: () => Promise<void>; // 登录
  logout: () => Promise<void>; // 登出
}
```

#### 3.1.2 useApiMonitor Hook
**功能**：管理 API 监控逻辑

**返回值**：
```typescript
{
  statuses: ApiStatus[];    // API 状态列表
  history: StatusHistory[]; // 历史数据
  isChecking: boolean;      // 是否正在检查
  runCheck: () => Promise<void>;     // 执行检查
}
```

#### 3.1.3 performCheck 函数
**功能**：执行批量 API 检查

**参数**：无

**返回值**：`Promise<ApiCheckResult[]>` - API 检查结果数组

#### 3.1.4 getCache 函数
**功能**：获取缓存的 API 检查结果

**参数**：
- `apiId: string` - API 标识

**返回值**：`ApiCheckResult | null`

#### 3.1.5 setCache 函数
**功能**：设置 API 检查结果缓存

**参数**：
- `apiId: string` - API 标识
- `result: ApiCheckResult` - 检查结果

**返回值**：`void`

### 3.2 后台监控 API

#### 3.2.1 runBackgroundMonitor 函数
**功能**：执行后台监控任务

**参数**：无

**返回值**：`Promise<void>`

#### 3.2.2 sendAlert 函数
**功能**：发送告警通知

**参数**：
- `alert: Alert` - 告警信息

**返回值**：`Promise<void>`

## 4. 开发流程

### 4.1 分支管理
| 分支 | 用途 |
|-----|------|
| **main** | 主分支，用于生产环境 |
| **develop** | 开发分支，用于集成新功能 |
| **feature/** | 功能分支，用于开发具体功能 |
| **bugfix/** | 修复分支，用于修复 bug |

### 4.2 提交规范
| 类型 | 描述 |
|-----|------|
| **feat** | 添加新功能 |
| **fix** | 修复 bug |
| **docs** | 更新文档 |
| **style** | 修改样式 |
| **refactor** | 重构代码 |
| **test** | 添加或修改测试 |
| **chore** | 构建或配置更改 |
| **perf** | 性能优化 |
| **ci** | CI/CD 配置变更 |
| **revert** | 回滚提交 |

**提交格式**：
```
<type>: <description>

[可选正文]

[可选页脚]
```

**示例**：
```
feat: 添加 API 配置功能

- 支持添加/删除自定义 API 端点
- 配置持久化到 localStorage
- 支持重置为默认配置
```

## 5. 组件规范

### 5.1 DashboardClient
**功能**：仪表盘客户端主组件（'use client'），整合所有子组件和数据 hooks

**Props**：无

**结构**：
- DashboardHeader + GeoOptInDialog + AlertsDropdown
- Hero 区域（标题 + 描述 + 4 个 StatCard）
- Alerts Banner（条件显示）
- API Status Grid 区域（含 ApiConfig）
- Latency History Chart 区域
- DashboardFooter

### 5.2 DashboardHeader
**功能**：显示品牌信息、告警铃铛、主题切换、地理位置、用户登录状态

**Props**：
```typescript
interface DashboardHeaderProps {
  user: User | null;
  alerts: Alert[];
  showAlerts: boolean;
  setShowAlerts: (show: boolean) => void;
  theme: string | undefined;
  setTheme: (theme: string) => void;
  geo: { city: string; country: string; ip?: string } | null;
  isGeoLoading?: boolean;
  refreshGeo?: () => void;
  login: () => Promise<void>;
  logout: () => Promise<void>;
  resolveAlert?: (id: string) => Promise<void>;
}
```

### 5.3 ApiStatusGrid
**功能**：以扁平网格展示所有 API 状态卡片（不按供应商分组，主组件）

**Props**：
```typescript
interface ApiStatusGridProps {
  statuses: ApiStatus[];
  isChecking?: boolean;
}
```

### 5.4 StatusGrid
**功能**：兼容层组件，转发给 ApiStatusGrid（向后兼容）

**Props**：
```typescript
interface StatusGridProps {
  statuses: ApiStatus[];
}
```

### 5.5 LatencyHistoryChart
**功能**：展示历史延迟趋势（Recharts AreaChart，静态导入（页面级 Suspense 骨架由 DashboardSkeleton 提供），React.memo 优化）

**Props**：
```typescript
interface LatencyHistoryChartProps {
  chartData: ChartDataPoint[];
  statuses: ApiStatus[];
  getApiColor: (id: string) => string;
}
```

### 5.6 AlertsDropdown
**功能**：以 Dialog 形式展示活跃告警列表，支持告警解决

**Props**：
```typescript
interface AlertsDropdownProps {
  alerts: Alert[];
  show: boolean;
  onClose: () => void;
  resolveAlert: (id: string) => void;
}
```

### 5.7 ApiConfig
**功能**：允许用户自定义 API 检查配置（添加/删除/编辑/重置），localStorage 持久化

**Props**：无

### 5.8 StatCard
**功能**：统计卡片组件，展示在线/降级/离线/平均延迟四项指标

**Props**：
```typescript
interface StatCardProps {
  icon: React.ReactNode;
  label: string;
  value: number | string;
  iconBgColor: string;
  iconTextColor: string;
  valueColor: string;
  hoverBorderColor: string;
  hoverShadowColor: string;
}
```

### 5.9 ThemeProvider
**功能**：管理浅色/深色双主题（next-themes `defaultTheme="system" enableSystem`），`<html>` 切换 `.light`/`.dark`，基于 next-themes

**Props**：继承 `NextThemesProvider` 的所有 props

### 5.10 GeoOptInDialog
**功能**：地理位置授权对话框，用户同意后获取地理位置信息

**Props**：无

### 5.11 StructuredData
**功能**：Schema.org 结构化数据组件（SEO），服务端渲染

**Props**：无

### 5.12 StatusDot
**功能**：状态圆点组件（在线/降级/离线三色 + 光晕效果）

**Props**：
```typescript
interface StatusDotProps {
  status: ApiStatus['status'];
}
```

### 5.13 ProgressBar
**功能**：进度条组件（渐变填充 + shimmer 扫光动画）

**Props**：
```typescript
interface ProgressBarProps {
  value: number;  // 0-100
  variant: 'success' | 'warning' | 'danger';
  showLabel?: boolean;
}
```

### 5.14 DashboardFooter
**功能**：页脚组件，三栏展示（Global Coverage / UI Tech Stack / Data Integrity）

**Props**：无

## 6. API 监控逻辑

### 6.1 监控的 API

> 完整清单（29 条）以 `app/constants/apis.json` 为单一真源，并由 `scripts/sync-apis.mjs` 同步到 `supabase/functions/monitor/apis.json`；本表不再重复逐条维护，避免双源漂移。

| 区域 | 数量 | 供应商 |
|-----|------|--------|
| 美国 / 全球 | 16 | OpenAI / Anthropic / Google / Meta(Groq) / Mistral / xAI / Cohere / Perplexity / Together AI / Replicate / Stability AI / HuggingFace / OpenRouter / Fireworks / NVIDIA / AI21 |
| 中国 | 13 | Moonshot / ZhipuAI / Baichuan / Alibaba / Tencent / Baidu / DeepSeek / MiniMax / iFlytek / ByteDance / 01.AI / SiliconFlow / StepFun |

### 6.2 监控配置
| 常量 | 值 | 说明 |
|-----|---|------|
| `LATENCY_THRESHOLD` | 1500ms | 延迟阈值，超过此值触发告警 |
| `DEGRADED_THRESHOLD` | 1000ms | 降级阈值，超过此值标记为降级 |
| `MAX_RETRIES` | 2 | 最大重试次数 |
| `RETRY_DELAY` | 1000ms | 重试间隔 |
| `MAX_CONCURRENT_REQUESTS` | 5 | 最大并发请求数 |
| `CHECK_INTERVAL` | 5 分钟 | 后台检查间隔 |
| `CACHE_EXPIRY` | 30 秒 | 默认缓存过期时间 |
| `GEO_INFO_EXPIRY` | 24 小时 | 地理位置缓存过期时间 |

### 6.3 后台监控流程
1. 浏览器端 useApiMonitor 挂载时先探测一次，之后按 CHECK_INTERVAL（5 分钟）定时探测
2. 服务端持久化（可选）：cron.sql 的 pg_cron 每 5 分钟 POST /monitor 触发 Edge Function
3. Edge Function 读取白名单 apis.json，探测 29 个端点（6 秒超时）
4. 结果写入 api_status（upsert）与 status_history（insert），返回 { ok, checked, offline, degraded, at }
5. 可选自建 Express 模式启动 10 秒后首检，并自行创建告警

## 7. 数据模型

### 7.1 Supabase 表结构
| 表名 | 用途 | 说明 |
|---------|------|------|
| `api_status` | API 当前状态 | 所有人可读，**仅认证用户可写入** |
| `status_history` | 历史性能数据 | 所有人可读，**仅认证用户可写入** |
| `alerts` | 系统告警 | 所有人可读，**仅认证用户可写入/解决** |
| `user_profiles` | 用户资料 | 关联 Supabase Auth 用户 |

### 7.2 数据结构

#### ApiStatus
```typescript
interface ApiStatus {
  id: string;                    // API 唯一标识
  name: string;                  // 显示名称
  provider: string;              // 提供商名称
  url: string;                   // 检查 URL
  status: 'online' | 'offline' | 'degraded';  // 状态
  latency: number;               // 延迟(ms)
  lastChecked: string;           // 最后检查时间
  error?: string;                // 错误信息
  retries?: number;              // 重试次数
  errorRate?: number;            // 错误率(%)
  availability?: number;         // 可用性(%)
  uptime?: number;               // 正常运行时间(%)
  averageLatency?: number;       // 平均延迟(ms)
  maxLatency?: number;           // 最大延迟(ms)
  minLatency?: number;           // 最小延迟(ms)
}
```

#### StatusHistory
```typescript
interface StatusHistory {
  id?: string;                   // 记录 ID
  apiId: string;                 // API 标识
  status: 'online' | 'offline' | 'degraded';  // 状态
  latency: number;               // 延迟(ms)
  timestamp: Date;               // 时间戳
  time: string;                  // 格式化时间
}
```

#### Alert
```typescript
interface Alert {
  id: string;                    // 告警 ID
  apiId: string;                 // API 标识
  apiName: string;               // API 名称
  type: 'downtime' | 'latency' | 'error';  // 告警类型
  severity: 'low' | 'medium' | 'high' | 'critical';  // 严重程度
  message: string;               // 告警消息
  timestamp: Date | unknown;     // 时间戳
  resolved: boolean;             // 是否已解决
  error?: string;                // 错误信息
  retries?: number;              // 重试次数
  latency?: number;              // 延迟值
  resolvedAt?: Date;             // 解决时间
  resolvedBy?: string;           // 解决人
}
```

## 8. 错误处理规范

### 8.1 错误分类
| 错误类型 | 代码 | 描述 |
|---------|------|------|
| 网络错误 | `NETWORK_TIMEOUT` | 网络请求超时 |
| 网络错误 | `NETWORK_OFFLINE` | 用户离线 |
| 网络错误 | `NETWORK_ERROR` | 网络连接错误 |
| API 错误 | `API_UNAVAILABLE` | API 服务不可用 |
| API 错误 | `API_ERROR` | API 请求失败 |
| API 错误 | `API_RATE_LIMITED` | API 请求被限流 |
| 认证错误 | `AUTH_REQUIRED` | 登录态相关（手动探测无需登录） |
| 认证错误 | `AUTH_FAILED` | 登录失败 |
| 认证错误 | `AUTH_EXPIRED` | 会话过期 |
| Supabase 错误 | `SUPABASE_ERROR` | Supabase 服务错误 |
| Supabase 错误 | `SUPABASE_QUERY_ERROR` | 数据库操作失败 |
| 验证错误 | `VALIDATION_ERROR` | 输入验证失败 |
| 未知错误 | `UNKNOWN_ERROR` | 未知错误 |

### 8.2 错误处理流程
1. **捕获错误**：使用 try-catch 捕获异步操作错误
2. **分类处理**：使用 `handleError` 函数将错误分类
3. **记录日志**：使用 `logError` 函数记录错误日志
4. **显示通知**：将错误信息展示给用户
5. **优雅降级**：提供备用方案或默认数据

### 8.3 错误边界
- 使用 `ErrorBoundary` 组件捕获 React 组件错误
- 在布局层面实现全局错误处理
- 提供用户友好的错误提示和刷新按钮

## 9. 日志规范

### 9.1 日志级别
| 级别 | 用途 | 示例 |
|-----|------|------|
| **DEBUG** | 调试信息（仅开发环境） | 变量值、函数调用 |
| **INFO** | 一般信息 | 服务启动、配置加载 |
| **WARN** | 警告信息 | 缓存过期、降级处理 |
| **ERROR** | 错误信息 | API 调用失败、数据异常 |
| **CRITICAL** | 严重错误 | 服务崩溃、认证失败 |

### 9.2 日志格式
```
[时间] [级别] [模块] 消息
```

**示例**：
```
2024-01-15 10:30:00 [INFO] [Monitor] Starting background check...
2024-01-15 10:30:05 [ERROR] [Monitor] API check failed for openai-gpt-4o
2024-01-15 10:30:05 [WARN] [Cache] Cache expired for anthropic-claude-3-5
```

### 9.3 日志实践
- 开发环境：输出所有级别的日志
- 生产环境：仅输出 WARN、ERROR、CRITICAL 级别日志
- 敏感信息：禁止记录密码、Token 等敏感数据
- 错误追踪：记录错误堆栈信息以便排查

## 10. 安全措施

### 10.1 认证
- 使用 Supabase Auth (Google OAuth)
- 敏感操作（如手动检查）需要用户登录
- 会话管理由 Supabase 自动处理

### 10.2 Row Level Security (RLS) 策略

> 与 `supabase/schema.sql` 保持一致：状态与告警**公开可读**，但**仅认证用户可写入**，避免匿名篡改监控数据。

```sql
-- API 状态：所有人可读，仅认证用户可写入
CREATE POLICY "api_status_read_all" ON api_status FOR SELECT USING (true);
CREATE POLICY "api_status_write_auth" ON api_status FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "api_status_update_auth" ON api_status FOR UPDATE USING (auth.role() = 'authenticated');

-- 告警：所有人可读，仅认证用户可写入/解决
CREATE POLICY "alerts_read_all" ON alerts FOR SELECT USING (true);
CREATE POLICY "alerts_insert_auth" ON alerts FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "alerts_update_auth" ON alerts FOR UPDATE USING (auth.role() = 'authenticated');
```

### 10.3 安全最佳实践
- 代码/日志中不存储密钥
- 输入验证防止 XSS 攻击
- 使用 HTTPS 进行通信
- 最小权限原则配置 RLS 策略
- 服务端使用 Service Role Key，前端使用 Anon Key

## 11. 性能优化

### 11.1 前端优化
| 优化项 | 实现方式 |
|-------|---------|
| 组件渲染 | 使用 React.memo 减少不必要重渲染 |
| 计算缓存 | 使用 useMemo 缓存计算结果 |
| 图表性能 | 限制数据点数量（最近 50 个） |
| 地理位置 | 本地缓存 24 小时 |
| API 缓存 | 内存 + localStorage 双层缓存 |

### 11.2 后端优化
| 优化项 | 实现方式 |
|-------|---------|
| 并发控制 | 限制最大并发请求数（5 个） |
| 重试机制 | 固定 1000ms 延迟重试 |
| 批量写入 | 批量更新 Supabase |
| 缓存策略 | 智能过期时间计算 |

## 12. CI/CD 流程

### 12.1 持续集成
```
代码提交 → GitHub Actions → pnpm install --frozen-lockfile → sync-apis --check → lint → test（覆盖率）→ build
```

### 12.2 检查步骤
| 步骤 | 命令 | 用途 |
|-----|------|------|
| 安装依赖 | `pnpm install --frozen-lockfile` | 安装依赖（pnpm 9.15.0） |
| ESLint | `pnpm lint` | 代码风格检查 |
| API 单一真源 | `node scripts/sync-apis.mjs --check` | 校验 apis.json 前后台一致 |
| 测试 | `pnpm test` | 运行单元测试 |
| 构建 | `pnpm build` | 构建生产版本 |

### 12.3 部署流程
```
main 分支更新 → GitHub Actions → 构建 → 部署
```

### 12.4 部署命令
| 命令 | 用途 |
|-----|------|
| `pnpm build` | 构建前端 |
| `supabase functions deploy monitor --no-verify-jwt` | 部署后台 Edge Function |
| `node scripts/sync-apis.mjs --check` | 校验 API 单一真源 |

## 13. 测试策略

### 13.1 测试类型
| 类型 | 描述 | 工具 |
|-----|------|------|
| 单元测试 | 测试单个函数和组件 | Jest |
| 组件测试 | 测试组件交互 | React Testing Library |

### 13.2 测试覆盖
- 覆盖率门禁：statements / functions / lines ≥ 70%，branches ≥ 55%（jest.config.cjs）
- 重点测试：监控逻辑、数据处理、错误处理

### 13.3 测试文件命名
- 单元测试：`*.test.ts`
- 组件测试：`*.test.tsx`

## 14. 国际化 (i18n)

### 14.1 支持的语言
| 语言代码 | 语言名称 | 本地名称 |
|---------|---------|--------|
| en | English | English |
| zh-cn | 简体中文 | 简体中文 |
| zh-tw | 繁体中文 | 繁體中文 |
| ar | 阿拉伯语 | العربية |
| cs | 捷克语 | Čeština |
| es | 西班牙语 | Español |
| hi | 印地语 | हिन्दी |
| id | 印度尼西亚语 | Bahasa Indonesia |
| it | 意大利语 | Italiano |
| nl | 荷兰语 | Nederlands |
| pl | 波兰语 | Polski |
| sv | 瑞典语 | Svenska |
| th | 泰语 | ไทย |
| tr | 土耳其语 | Türkçe |
| ru | 俄语 | Русский |
| vi | 越南语 | Tiếng Việt |

### 15.2 语言检测与切换
- **自动检测**：系统自动检测浏览器语言设置
- **持久化**：用户语言选择保存到 localStorage
- **回退机制**：未找到语言时回退到英文
- **翻译格式**：使用点分隔的翻译键（如 `dashboard.title`）

### 15.3 翻译文件格式
```json
{
  "errors": {
    "networkTimeout": "超时信息"
  },
  "dashboard": {
    "title": "标题"
  }
}
```

### 15.4 API 使用方式
```typescript
import { t, setLocale, getLocale, initLocale } from '@/lib/i18n';

// 获取翻译
const title = t('dashboard.title');

// 设置语言（持久化至 localStorage）
setLocale('es');

// 初始化（自动检测）
initLocale();
```

## 16. Supabase 配置

### 16.1 客户端配置
```typescript
// app/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL || '';
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY || '';

export const supabase = createClient(supabaseUrl, supabaseAnonKey);
```

### 16.2 服务端配置
```typescript
// examples/self-host-server.ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL || '',
  process.env.SUPABASE_SERVICE_ROLE_KEY || ''
);
```

### 16.3 数据表说明
| 表名 | 说明 |
|------|------|
| `api_status` | 存储 API 当前状态 |
| `status_history` | 存储历史性能数据 |
| `alerts` | 存储系统告警 |
| `user_profiles` | 用户资料表，自动关联 Auth 用户 |

### 16.4 实时订阅
Supabase 支持实时数据订阅，用于告警更新：
```typescript
const channel = supabase
  .channel('alerts_changes')
  .on('postgres_changes', {
    event: '*',
    schema: 'public',
    table: 'alerts'
  }, (payload) => {
    console.log('Alert changed:', payload);
  })
  .subscribe();
```

## 15. 部署

### 15.1 环境配置
- Supabase 项目配置：在 `.env.local` 文件中配置以下环境变量：
  - `NEXT_PUBLIC_SUPABASE_URL`: Supabase 项目 URL
  - `NEXT_PUBLIC_SUPABASE_ANON_KEY`: Supabase Anon Key（前端使用）
  - `SUPABASE_SERVICE_ROLE_KEY`: Supabase Service Role Key（服务端使用）
- 数据库迁移：运行 `supabase/schema.sql`；如需后台持久化，再执行 `supabase/cron.sql` 并部署 Edge Function `monitor`

### 15.2 启动命令
| 命令 | 用途 |
|-----|------|
| `pnpm dev` | 开发模式 |
| `pnpm build` | 构建生产版本 |
| `pnpm start` | 启动生产服务器（`next start`，静态导出场景请用静态托管） |
| `pnpm test` | 运行测试 |
| `pnpm lint` | 代码检查 |
| `pnpm sync:apis` | 同步 apis.json 单一真源 |
| `pnpm preflight` | 仅校验一致性（build 内置） |
| `pnpm test:watch` | 监听模式测试 |
| `pnpm clean` | next clean |