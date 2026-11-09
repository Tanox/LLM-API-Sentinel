# Changelog

## [Unreleased] - 2026-11-09

### Docs
- **全量文档与代码对齐**: 以 `package.json` v2.10.35 为基线，校对并更新双语 README、`docs/` 5 份补充文档、`openspec/` 7 份规范与索引、`openspec/config.yaml`、`.github/` 社区文件；统一术语（静态导出 / Supabase Cron + Edge Function `monitor` / 浅色与深色双主题 / Recharts / pnpm 9.15.0）与格式风格。
- **功能与接口事实修正**: 29 个 API 由 `app/constants/apis.json` 单一真源维护并由 `sync-apis.mjs` 同步；状态为首屏查询 + 前端 5 分钟定时探测，仅 `alerts` 使用 Realtime；手动检查无需登录；告警为 `critical` / `high` 且由前端生成；`POST /monitor` 返回 `{ ok, checked, offline, degraded, at }`；组件 Props、函数签名与返回值按源码修订。
- **运行步骤与依赖版本**: 统一 pnpm 脚本（含 `preflight` / `sync:apis`）、Node.js 22 / pnpm 9.15.0、CI 门禁顺序、覆盖率阈值 70/55/70/70；补全环境变量清单（`NEXT_PUBLIC_SITE_URL`、`NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION`、`PORT`、`RATE_LIMIT_REDIS_*`）与 Edge Function secrets。
- **删除废弃内容**: 移除或更正不存在的 `server.ts`、`/api/check`、`security_best_practices_report.md`、`mock-data.ts`、Cloud Functions / Firestore 术语及 zod `safeParse` 描述；修复 `.github/` 全部失效相对链接与 2.7.x 旧版本支持表。
- **补充近期变更记录**: 在文档中补齐 v2.10.27~v2.10.35 引入的 API 单一真源同步、i18n 模块拆分、自托管脚本迁移、GA4 衡量 ID 更新、原型语义 alpha token 与语义化 id 等变更。
- **归档标注**: `openspec/changes/` 提案保留原有历史版本号，新增「历史归档」注记，避免被误读为当前状态。

### Known / 待人工确认
- 当前工作区 `pnpm-lock.yaml` 缺失（`git status` 显示 D），但 CI 使用 `pnpm install --frozen-lockfile`；推送前需恢复锁文件。
- 本次仅校准文档，未擅自 bump `package.json` 版本；是否按 `docs/contributing.md` 的「每次修改升级版本」规则发版，需维护者确认。
- `app/layout.tsx` 的 metadata description 仍写「12+ AI providers」，与 29 个 API 的规模表述不一致（属代码文案，非文档）。
- `public/` 缺 `/favicon.ico` 与 `/og-image.png`（`app/layout.tsx` 引用），需补资源或改引用。
- `app/hooks/useAuth.ts` 的 `redirectTo: .../auth/callback` 无对应回调路由，需确认登录流程。

## [2.10.35] - 2026-08-29

### Chore / Docs
- **同步 openspec 文档头注释版本号**: `project.md` / `ui.md` / `design-system.md` 头注释由 `v2.10.32` 更新至 `v2.10.35`，对齐应用 HEAD（同步后应用已 bump 至 v2.10.35）

### Fix / CI
- **修复 EdgeOne CI frozen-lockfile 安装失败**: 重新生成 `pnpm-lock.yaml` 使其与 `package.json` 一致，移除 v2.10.31 清理死依赖后遗留的 `@google/genai` 锁文件条目（specifiers 不匹配导致 `ERR_PNPM_OUTDATED_LOCKFILE`）

## [2.10.34] - 2026-08-29

### Chore / Prototype
- **对齐设计系统语义 alpha token**: `prototype/assets/styles.css` 新增 `--color-*-10/15/20` 语义 alpha 变体（明暗双档），对齐 `openspec/design-system.md §2.3`；`app.js` 抽离 `alpha()` 统一混色工具替换散落的 `color-mix` 硬编码
- **修复 wireframes 详情抽屉失效**: `wireframes.html` 补齐 `#detail-drawer` DOM 与 `.drawer` 公共样式，使 `openDetail('...')` 调用真正可用
- **补充语义化 id**: 各原型页主容器加 `#prototype-main`/`#wireframes-main`/`#component-library-main`，便于定位、无障碍与测试
- **同步原型版本号**: `index.html`/`prototype.html`/`components.html`/`wireframes.html`/`data.js` 标题与文件头由 v2.10.24 统一至 v2.10.34

## [2.10.33] - 2026-08-28

### Chore / Analytics
- **更新 Google Analytics 衡量 ID**: `app/layout.tsx` 注入的 GA4 衡量 ID 由 `G-7RGKS16M38` 替换为 `G-0WKWY6YELE`

## [2.10.32] - 2026-08-28

### Docs
- **同步 README 版本号**: `README.md`/`README_CN.md` 标题版本号从 v2.10.23/v2.10.24 更新至 v2.10.32，消除与 `package.json` 的脱节

### Chore / Config
- **恢复构建期 ESLint 检查**: `next.config.mjs` 的 `eslint.ignoreDuringBuilds` 由 `true` 改回 `false`，规范检查不再被跳过
- **同步 openspec 头注释**: `design-system.md`/`project.md`/`ui.md` 标题版本号统一至 v2.10.32（`changes/` 归档提案属历史记录，保持原值未改）

## [2.10.31] - 2026-08-27

### Fix
- **清理死依赖**: 删除从未调用的 `@google/genai`（零真实调用，仅为体积/供应链风险）；`express`/`helmet` 经核实被 `examples/self-host-server.ts` 自托管服务端真实使用，保留
- **API 单一真源自动化**: `package.json` 新增 `sync:apis`（复制）与 `preflight`（`--check` 校验）脚本，并将 `preflight` 接入 `build` 前置，本地构建即校验 `supabase/functions/monitor/apis.json` 与前端 `app/constants/apis.json` 一致（CI 已于 v2.10.27 接入相同校验）
- **废弃模拟数据**: `app/lib/mock-data.ts` 已无任何调用方，重写为废弃占位（本项目自 v2.10.22 起全真实探测）
- **探测语义澄清**: `monitor.ts` 补注 `isOnline` 仅代表端点 HTTP 可达，需鉴权端点可能返回 401/403 仍判 online

## [2.10.30] - 2026-08-27

### Fix
- **修复 examples 自托管脚本 CI 构建失败**: 修正 `examples/self-host-server.ts` 中 `./app/...` 相对路径为 `../app/...`；并将 `examples/` 从 tsconfig `exclude` 中排除，避免独立示例脚本拖垮 Next 构建类型检查

## [2.10.29] - 2026-08-27

### Refactor
- **降级 server.ts 至 examples/**: 将根目录 `server.ts` 移至 `examples/self-host-server.ts`，明示其为自托管可选件（非默认路径）；同步更新 README/README_CN/openspec/docs 全部 `server.ts` 引用

## [2.10.28] - 2026-08-27

### Refactor
- **拆分 i18n.ts (230→145 行)**: 抽离 `i18n-locales.ts`（语言包/映射/白名单常量）与 `i18n-validation.ts`（locale 校验/安全存储读取），主文件仅保留编排逻辑，公开 API 契约不变

## [2.10.27] - 2026-08-27

### Refactor
- **消除监控 API 双源**: 抽出 `app/constants/apis.json` 作为单一真源，前端 `DEFAULT_APIS` 改为导入；Edge Function `monitor` 改为运行时读取同步的 `apis.json`，新增 `scripts/sync-apis.mjs`（`--check` 已在 CI 校验一致性），杜绝前后台探测目标漂移

## [2.10.26] - 2026-08-27

### Docs
- **同步规范头注释与版本**: `project.md`/`server.ts`/`supabase/functions/monitor/index.ts`/`app/lib/i18n.ts` 头注释统一至 v2.10.26，消除系统性脱节
- **修正 project.md 时间范围**: `24H/7D/30D` 改为 `1H/6H/24H`，与 `DashboardClient.tsx` 的 `TIME_RANGES` 对齐

## [2.10.25] - 2026-08-27

### Docs
- **对齐规范与代码/原型**: `design-system.md` §4.2 最大容器宽度由 `max-w-7xl`(1280px) 改为 `max-w-[1400px]`(1400px)，与 `ui.md` 及 `app` 代码一致；`ui.md` 图表时间范围由 "24H/7D/30D" 修正为 "1H/6H/24H"（与 `DashboardClient.tsx` 的 `TIME_RANGES` 一致），并补注历史记录内存上限 100 条（`app/store/api.ts`）

## [2.10.24] - 2026-08-27

### Style
- **大号字体使用衬线字体**：`app/style.css` 与 `prototype/assets/styles.css` 新增 `--font-serif` 变量与 `.font-display` 工具类；Hero 主标题、错误页标题等 ≥`text-2xl` 展示性标题套用衬线；纯数据数值（延迟/可用性/重试次数）保持等宽对齐，不套衬线
- **原型补语义化 id**：`prototype.html` 的 `main-content`、`live-feed-section`、`api-grid-section`、`logs-section` 等主要容器补齐语义化 id，便于测试定位；全局版本展示位统一至 v2.10.24

## [2.10.23] - 2026-08-27

### Docs
- **统一文档版本号至 v2.10.23**：`openspec/config.yaml`、`project.md`、`ui.md`、`design-system.md` 标题与 `package.json` 同步；`README`/`README_CN`/`docs/roadmap.md`/`docs/contributing.md` 标题与基线同步（此前滞后于代码至 v2.10.15~v2.10.19）
- **反映 v2.10.22 重大变更**：`openspec/project.md` 目录树补充 `metrics-storage.ts` 并修正 `mock-data.ts` 注释（不再作初始注入）；`docs/roadmap.md` 基线补充「真实探测无演示数据 + 指标持久化」事实

## [2.10.22] - 2026-08-27

### Fix
- **移除全部演示数据，保证检测真实**: 删除 `useApiMonitor` 初始 `generateMockApiStatuses()` 随机模拟注入（Supabase 未配置/加载失败时改为空态等待真实探测）；修正 `calculateRealMetrics` 累计基数由伪造的 100 次改为从 1 次真实探测起算，首探可用性只能为 100% 或 0%
- **累计指标持久化**: 新增 `metrics-storage.ts`，将可用性/延迟历史真实写入 localStorage，跨刷新累加而非重置；`ApiStatusGrid` 空态改为真实探测骨架占位，16 个语言补充 `dashboard.noData` 文案

## [2.10.21] - 2026-08-26

### Fix
- **修复 CI 缺失 pnpm-lock.yaml**: 将 `pnpm-lock.yaml` 从 `.gitignore` 移除并提交（pnpm 项目须入库 lockfile 以保证 `--frozen-lockfile` 可复现安装），消除 `Dependencies lock file is not found` 错误

## [2.10.20] - 2026-08-26

### CI
- **固定 CI pnpm 版本**: `ci.yml` 与 `release.yml` 的 `pnpm/action-setup@v4` 显式指定 `version: 9.15.0`，与 `package.json` 的 `packageManager` 单一来源对齐，提升可复现性

## [2.10.19] - 2026-08-26

### CI
- **修复 GitHub Actions pnpm 版本冲突**: 移除 `pnpm/action-setup@v4` 的 `version: 9`，改由 `package.json` 的 `packageManager: pnpm@9.15.0` 单一来源驱动，消除 `ERR_PNPM_BAD_PM_VERSION`
- **升级 Node 运行时**: workflow `node-version` 由 20 升至 22（规避 Node 20 弃用）

## [2.10.18] - 2026-08-26

### Fix
- **修复 useApiMonitor 编译错误**: `useCallback` 依赖数组中遗留未定义的 `setError`，改为已解构的 `showError`（对齐 `useErrorStore`），消除 `next build` 的 TS2304 类型错误

## [2.10.17] - 2026-08-26

### Fix
- **修复 CI 二级依赖冲突**: `@eslint/js` 由 `^10.0.1` 降级至 `^8.57.1`，与已降级的 `eslint@^8.57.1` peer 对齐，消除 `peerOptional eslint@^10.0.0` ERESOLVE 错误，恢复依赖安装构建

## [2.10.16] - 2026-08-26

### Fix
- **修复 CI 依赖冲突**: `eslint` 由 `^10.4.1` 降级至 `^8.57.1`，消除与 `eslint-config-next@14.2.13` 的 peer 依赖冲突（eslint 9/10 不兼容 Next 14 lint 集成），恢复依赖安装构建

## [2.10.15] - 2026-08-26

### Refactor
- **对齐原型 UI（视觉层）**: StatusMonitorSection 去除冗余外层 Card 包裹，改为 section 直接标题 + 网格（对齐原型 `section` 直接网格风格）；状态监控区与延迟区标题由 `text-2xl font-bold` 降级为 `text-lg font-semibold`（对齐原型 `text-lg` 小标题）
- **Brand logo 渐变对齐原型**: DashboardHeader logo 由淡色 `from-primary/20 to-accent/20` 改为实心 `from-primary to-accent` + `shadow-lg shadow-primary/20`，图标色 `#fff`（`text-primary-foreground`）
- 保留 29 API / 浅色主题 / Recharts / 扁平网格等已上线能力不回退

## [2.10.14] - 2026-08-26

### Fix
- **原型告警对话框 Resolve 按钮无响应**：新增 `resolveAlert(id)` 轻量 mock，将对应 API 标记为在线并刷新横幅与卡片，按钮接入 `onclick` 与 `aria-label`
- **原型初始时间范围文案硬编码 24H**：`appInit` 渲染后调用 `setTimeRange(currentTimeRange)` 同步中英文文案
- **统计卡片 offline 无离线时语义异常**：子文案由"在线"改为"全部正常"（`allNormal` i18n 键），新增 zh/en 翻译
- **原型图表 tooltip 前景色变量未定义**：`@theme` 与 `.dark` 档补 `--color-popover-foreground`，消除回退依赖
- **原型版本标注脱节**：`index.html`/`components.html`/`data.js` 自身版本 v2.10.1 统一至 v2.10.13

## [2.10.13] - 2026-08-26

### Chore
- **强制 pnpm 包管理器约束**：`package.json` 新增 `packageManager: pnpm@9.15.0`、`engines.pnpm >=9`、`scripts.preinstall: npx only-allow pnpm`；根目录新增 `.npmrc`（`engine-strict=true`、`package-manager-strict=true`）
- 清理旧锁文件 `package-lock.json` / `yarn.lock`，保留 `pnpm-lock.yaml`
- 版本同步至 v2.10.13

## [2.10.12] - 2026-08-17

### Docs
- **纠错 openspec 与代码对齐**：v2.10.11 误将图表写成「手写 SVG」、主题写成「纯深色双档」，与 `app/style.css` 及 `LatencyHistoryChart.tsx` 实际实现不符，本次更正为「Recharts 面积图」与「浅色/深色双主题（next-themes `defaultTheme="system"`）」
- **同步配色 token**：`design-system.md` 中性色/主色/功能色表改为与 `app/style.css` 一致的 `.light`(浅)/`.dark`(深) 真实 Hex 值
- **统一版本号**：`openspec/config.yaml`、`project.md`、`ui.md`、`design-system.md` 标题与 `package.json` 同步至 v2.10.12

## [2.10.11] - 2026-08-16

### Docs
- **对齐 openspec 与代码实现**：`ui.md`/`features.md`/`architecture.md` 图表描述由「Recharts」更正为「手写 SVG（零依赖，Recharts 可选）」；主题描述由「深/浅双主题」更正为「纯深色双档（`:root`/`.dark`）」
- **同步版本号**：`openspec/config.yaml`、`project.md`、`ui.md`、`design-system.md` 标题统一至 v2.10.10；更正 `project.md` 设计系统链接版本、`.github/CONTRIBUTING` 示例版本
- **更新路线图**：`docs/roadmap.md` 当前能力基线由 v2.9.4/12 API/无 CI 更新为 v2.10.10/29 API/CI 已落地，并标记短期项完成状态
- **修正部署文档**：`docs/deployment.md` 后台监控 API 数量 12 → 29

## [2.10.10] - 2026-08-16

### Feat
- **接入 Google Analytics 4**: 在 `layout.tsx` 的 `<head>` 注入 gtag 脚本，Measurement ID `G-7RGKS16M38`，用于站点流量与行为分析

## [2.10.9] - 2026-08-16

### Fix
- **启用定时自动巡检**: `useApiMonitor` 新增基于 `CHECK_INTERVAL`(5 分钟)的 `setInterval` 自动刷新,修复部署后仪表盘数据不自动更新的问题
- **修正 robots.txt Sitemap 域名**: 由过时的 `llm-api-sentinel.vercel.app` 改为实际部署域 `llmapi.ewuse.com`,统一 SEO 抓取指向
- 同步版本至 v2.10.9

## [2.10.8] - 2026-08-16

### Chore
- **清理仓库内工具草稿**: 删除 `.trae/documents/arch.md` 与 `prd.md`（TRAE 编辑器生成的本地架构/需求草稿，非项目源码且与 `openspec/` 内容重叠），保持仓库整洁
- 同步版本至 v2.10.8

## [2.10.7] - 2026-08-16

### Fix
- **修复构建失败**: `app/lib/i18n.test-fixtures.ts` 与 `app/lib/i18n.ts` 的语言包类型签名由 `Record<string, string>` 放宽为 `Record<string, string | string[]>`，匹配 `share.promos` 等字符串数组字段，消除 TS 类型不兼容导致的 `next build` 编译错误
- 同步版本至 v2.10.7，并补齐 `openspec/config.yaml` 遗漏的 2.10.6 版本号

## [2.10.6] - 2026-08-16

### Refactor
- **对齐设计原型（低风险项）**: 页面宽度 max-w-[1600px] → max-w-[1400px]（DashboardClient/Header/Skeleton 三处），状态网格移除 2xl 列档（对齐原型 xl:grid-cols-4）
- **openspec/ui.md 同步**: 容器宽度与网格列数描述回调至原型范式（1400px / 无 2xl）
- **prototype 注释修正**: 顶部与 data.js 由「历史归档」改为「设计原型对齐基准」，差异清单标注为有意保留的增强（29 API、双主题、Recharts、扁平网格）
- 保留 29 个 API 与浅色主题等已上线能力不回退

## [2.10.5] - 2026-08-16

### Chore
- **清理根目录临时/无效文件**: 删除 TS 增量编译缓存 `tsconfig.tsbuildinfo`、调试日志 `.devserver.log`、`Playwright` 调试目录 `.playwright-cli/`、未跟踪且项目不使用的 `pnpm-lock.yaml`（项目使用 npm + `package-lock.json`）
- **gitignore 补强**: 新增 `.trae/`（TRAE 编辑器本地缓存目录），避免非源码产物误提交
- 同步版本至 v2.10.5，并补齐 `openspec/config.yaml` 遗漏的 2.10.4 版本号

## [2.10.4] - 2026-08-16

### Refactor
- **对齐规范与代码**: openspec/features.md、ui.md 修正 9 类过时描述（API 29 个、深/浅双主题、扁平网格、Recharts、max-w-[1600px]、Alerts 色值、share.promos 数组）
- **share-content.ts 改造**: promo1..5 平铺键 → 真正的 share.promos 数组（消除规范偏差根因）
- **i18n**: 16 个语言包 share.promo1..5 迁移至 share.promos 数组
- **ApiStatusCard**: containIntrinsicSize 估算高度 168px → 200px，避免内容溢出滚动跳动
- **prototype 归档**: 顶部与 data.js 标注历史原型及与 app 的差异清单
- 新增 openspec/change align-specs-with-code 记录本次对齐

## [2.10.3] - 2026-08-16

### Chore
- **Community Health Files 移入 `.github/`**: `CODE_OF_CONDUCT.md`、`CONTRIBUTING.md`、`SECURITY.md`、`SUPPORT.md` 由仓库根迁移至 `.github/`，统一社区健康文件位置（`LICENSE` 保留根目录以满足 GitHub license 检测）
- 文件内相对链接在 `.github/` 内仍有效，无需改写

## [2.10.2] - 2026-08-16

### Docs
- **补全 Community Health Files**: 新增 `.github/CODEOWNERS` 定义默认审查者
- **同步规范版本**: `openspec/project.md`、`ui.md`、`design-system.md` 标题版本由 v2.9.4 对齐至 v2.10.1，消除版本漂移

## [2.10.1] - 2026-08-16

### Fix
- **消除时钟竞态**: `refreshData` 不再直接写 `last-sync-time`，仅更新 `lastSyncTs`，DOM 文本交由 `updateClock` 统一刷新
- **峰值延迟失真**: `renderStats` 峰值改为涵盖所有非离线 API（含 degraded），避免漏掉最慢真实峰值（如 Gemini 1156ms）
- **进度条区分度**: 延迟进度条基准由硬编码 500ms 改为动态最大值（下限 500ms），正常 API 与降级 API 区分更清晰

### Style
- **删除死代码**: `prototype/assets/styles.css` 移除已不使用的 `.light .bg-ambient` 覆盖块（主题为 `.dark` 体系）

### A11y
- **主题按钮语义**: `theme-toggle` 的 `aria-label`/`title` 随切换动态更新（"切换到更深深色/默认深色"），新增 i18n 键 `themeToDark`/`themeToDefault`
- **图表读屏**: 延迟图表 `<svg>` 内增加 `<title>` 子元素，与 `aria-label` 双保险
- **刷新反馈**: 刷新按钮刷新期间设置 `aria-busy="true"`
- **对话框焦点陷阱**: 告警对话框打开时焦点置于首个可聚焦元素，Tab/Shift+Tab 在框内循环，避免焦点逃逸背景

## [2.10.0] - 2026-08-16

### Feat
- **页面加宽**: 主内容/顶栏/骨架屏容器由 `max-w-7xl` 加宽至 `max-w-[1600px]`，充分利用大屏横向空间
- **状态网格一行四卡**: `#status-section` 网格在 xl/2xl 断点固定为 4 列，状态监控区一行展示 4 个 API 卡片

## [2.9.9] - 2026-08-15

### Fix
- **修复构建失败**: `tsconfig.json` 的 `exclude` 新增 `supabase` 目录，避免 Deno 版 Edge Function（`supabase/functions/monitor/index.ts`，含 `https://esm.sh/...` 远程 import 与 `Deno.serve`）被 `next build` 的 TypeScript 编译误纳入导致 `Cannot find module` 编译错误

## [2.9.8] - 2026-08-15

### Fix
- **规范链接域名修正**: `layout.tsx` 与 `StructuredData.tsx` 的 `SITE_URL` 生产回退域名由 `llm-api-sentinel.vercel.app` 改为实际部署域名 `llmapi.ewuse.com`，修复 canonical/OG/JSON-LD 指向错误域名导致的 SEO 规范链接问题
- **初始数据空屏**: `useApiMonitor` 初始化时自动执行一次主动探测（`runCheck`），页面加载即显示真实 API 状态，不再依赖用户手动点击"立即检查"
- **访客只读探测**: `DashboardClient` 的 `canRunCheck` 移除 `!user` 限制，访客亦可手动触发只读探测刷新监控数据
- **Supabase 降级优化**: `useApiMonitor`/`useAlerts`/`useAuth` 在未配置 Supabase 时跳过查询与实时订阅，消除向占位端点发起的无意义请求与控制台噪音

### Feat
- **SEO 完整性**: 新增 `app/robots.ts` 与 `app/sitemap.ts`，提供 `robots.txt` 与 `sitemap.xml`（兼容静态导出），声明正确 sitemap 与 host

## [2.9.7] - 2026-08-15

### Feat
- **分享功能**: 新增 `ShareButton` 组件与 `share-content.ts`，一键复制带随机宣传文案的分析链接（`ref=share` 来源标记），挂载至 `DashboardHeader`
- **分享文案多语言**: 为全部 16 个 locale 补充 `share` section（标题/复制状态/5 条推广文案），保障分享弹窗全覆盖

## [2.9.6] - 2026-08-15

### Feat
- **CI/CD 流水线**: 新增 `.github/workflows/ci.yml`（PR/push 触发 lint + test + build 门禁）与 `release.yml`（语义化版本 tag 触发静态产物构建与 GitHub Release）
- **后台监控 Edge Function 示例**: 新增 `supabase/functions/monitor/index.ts`（探测 26 个 LLM API 写入 `api_status` 与 `status_history`）及 `supabase/cron.sql`（pg_cron 每 5 分钟调度）
- **多渠道告警增强**: `detectPlatform` 新增 Slack 与 Microsoft Teams 识别；`webhook-formatter` 新增 Slack attachments 与 Teams MessageCard 格式化分支

### Test
- **测试覆盖率门禁**: `jest.config.cjs` 新增 `collectCoverageFrom` 与 `coverageThreshold`（statements/functions/lines ≥70%）；`package.json` 的 `test` 脚本启用 `--coverage`
- **新增纯逻辑测试**: `utils.test.ts`、`supabase-mapping.test.ts`、`cache-validation.test.ts`、`notification.test.ts`，并修正 `monitor.test.ts` 适配 26 API 现状与 jsdom 环境
- **lint 范围聚焦**: `eslint.config.mjs` 忽略 `scripts/`、`prototype/`（独立工具与原型，非 Next 应用源码）

### Docs
- 同步版本号至 v2.9.6

## [2.9.6] - 2026-08-15

### Feat
- **分享功能优化**: 复制分析链接时同步复制项目宣传文案，支持多条文案随机使用
  - 新增 `ShareButton` 组件集成至仪表盘顶栏
  - `share-content.ts` 提供随机宣传文案与分享文本组装
  - 16 个语言包补充 `share` 命名空间（title/copied/copyFailed + promo1~5）
  - 剪贴板优先 `navigator.clipboard`，非安全上下文降级 `execCommand`
  - 复制成功/失败瞬时反馈，aria-live 可访问性支持
- 同步更新 `openspec/features.md` 2.9 分享功能规格与 7.9 验收标准

## [2.9.5] - 2026-08-15

### Feat
- **扩展 AI API 监测平台**: 从 12 个扩展至 26 个，新增 xAI Grok、Cohere Command R、Perplexity Sonar、Together AI、Replicate、Stability AI、HuggingFace、OpenRouter、Fireworks AI、NVIDIA NIM、AI21 Labs（国际），以及 MiniMax、iFlytek Spark、Doubao 火山方舟、01.AI Yi、SiliconFlow、StepFun 阶跃星辰（国内）
- **卡片紧凑布局**: `#status-section` 网格支持一行 4-5 列（`xl:grid-cols-4 2xl:grid-cols-5`），缩小卡片内边距/字号/间距适配多列显示

### Fix
- **补全缺失翻译 key**: 为全部 16 个语言包补充 `common.close`、`alerts.offline`、`alerts.highLatency` 三个此前未定义但代码已引用的 key

### Docs
- **文档一致性更新**: 同步全部项目文档至 v2.9.5
  - README 双语文档标题统一至 v2.9.5；图表库描述更正为手写 SVG 零依赖（Recharts 仅作可替换备选）
  - 主题描述统一为纯深色双档（移除所有"深色/浅色切换"过时表述，与设计系统权威一致）
  - openspec ui.md / project.md / design-system.md 标题统一至 v2.9.5
  - project.md：图表库表格、ApiStatusGrid 扁平网格、LocaleSwitcher 组件、useI18n 16 语言与 `persistLocale` 示例对齐代码现状
  - features.md / ui.md：修正 Provider Group 嵌套树为扁平网格、主题切换验收标准对齐
  - docs/contributing.md 示例版本更新
- **发展路线文档**: 新增 `docs/roadmap.md`，提出短期（CI/CD、测试覆盖、后台监控示例）、中期（SLA 状态页、多渠道告警、可配置目标）、长期（平台化 API/widget、移动端）及技术债清理建议，并在 README 文档链接补充

## [2.9.4] - 2026-08-15

### Fix
- **补全缺失翻译 key**: 为全部 16 个语言包补充 `common.close`、`alerts.offline`、`alerts.highLatency` 三个此前未定义但代码已引用的 key，修复告警横幅与错误通知显示原始 key 文字的问题

## [2.9.3] - 2026-08-15

### Fix
- **i18n 客户端打包修复**: `i18n.ts` 改用静态导入全部 16 个语言包，替代动态模板 `import()`，修复客户端打包失败导致页面显示 i18n key 而非翻译字符串的问题
- **翻译回退重写**: `t()` 优先当前语言、缺失时回退英文，逻辑更健壮
- **切换语义澄清**: `useI18n.ts` 将 `setLocale` 重命名为 `persistLocale`，移除冗余 localStorage 直写

## [2.9.2] - 2026-08-15

### Feat
- **语言切换 UI**: 新增 `LocaleSwitcher` 组件（基于 `DropdownMenuRadioGroup`），挂载至 `DashboardHeader`，支持 16 种语言即时切换并持久化至 localStorage
- **切换文案翻译**: 为全部 16 个 locale 的 `dashboard.changeLanguage` 补充对应语言翻译，保障切换器标签全覆盖

### Fix
- **SEO 语言声明对齐**: `layout.tsx` 与 `StructuredData.tsx` 的 `alternates.languages`/`alternateLocale`/`inLanguage` 移除无 locale 文件的 `ja`/`ko`，与实际支持的 16 种语言一致

## [2.9.1] - 2026-08-15

### Docs
- **设计系统对齐实现**: `design-system.md` 色彩章节重写为 Tailwind v4 `@theme` + `.dark` 覆盖体系，token 值严格对齐 `app/style.css`（`:root` 默认深 `#0f0f14` / `.dark` 更深 `#0a0a0f`），去除过时 `.light` 体系描述
- **数据表可见性修正**: `project.md` 7.1 节 `api_status`/`status_history`/`alerts` 可见性改为「认证用户可写」，与 10.2 RLS 策略及 `schema.sql` 一致
- **章节编号修复**: `project.md` 章节跳号（13→15→16→17）重排为连续 14 国际化 / 15 部署 / 16 Supabase 配置

### Style
- **原型主题对齐**: `prototype/assets/styles.css` 语义 token 值对齐 `app/style.css`，主题机制由 `.light` 切换改为 `.dark` 双档深色（`:root` 默认深 / `.dark` 更深），`app.js` `toggleTheme` 同步切换 `.dark` 类

### Refactor
- **状态网格简化**: `ApiStatusGrid.tsx` 移除按 provider 分组与空状态卡片，改为扁平 `<section>` 网格直接渲染，provider 透传至 `ApiStatusCard`

### Chore
- **版本统一**: 原型 3 文件 title、`project.md`/`design-system.md` 标题、package.json、config.yaml、`ApiStatusGrid.tsx` 统一至 v2.9.1

## [2.9.0] - 2026-08-15

### Fixed
- **告警系统失效**: `alert-service.ts` 误用不存在的 `status` 字符串列，改为 schema 真实的 `resolved` 布尔列（`insert({resolved:false})` / 去重 `.eq('resolved', false)` / 解决 `.update({resolved:true})`），修复告警无法写入与重复告警
- **时间范围装饰性**: `useDashboardStats` 新增 `activeRange` 参数，按 `timestamp` 窗口（1h/6h/24h）真实过滤图表数据，切换不再只高亮
- **SSRF 防护加固**: `notification-platforms.ts` 新增十进制/十六进制编码 IP 归一化拦截（如 `2130706433`→`127.0.0.1`），堵住内网地址绕过

### Refactor
- **模块拆分（单文件≤200行）**: `DashboardClient` 拆出 `dashboard-sections`；`cache.ts` 拆出 `cache-storage`/`cache-validation`；`ApiConfig` 拆出 `api-config-list`/`api-config-form`；`notification-platforms` 拆出 `webhook-formatter`；`i18n.test` 拆出 `i18n.test-fixtures`
- **语义化 id**: 主仪表盘、状态区、延迟区、配置、告警横幅、头部、脚部均带稳定 id

### Docs
- **规范冲突消解**: `project.md` RLS 策略改为与 `schema.sql` 一致（仅认证可写）；`ui.md` success token 统一为 `#22c55e`
- **版本统一**: 全部文件头注释、package.json、config.yaml、README 双语文档统一至 v2.9.0

## [2.8.5] - 2026-08-15

### Fixed
- **原型数据真实性**: 统计卡片副文案改为从真实数据派生（在线可用性均值 / 降级错误率均值 / 离线重试总次数 / 延迟峰值），移除写死的 `99.9%`/`2.3%`/`↓12%`
- **Hero 在线语义**: 顶栏改为「在线数 · 总数 API 状态监控」，动态读取在线/总数
- **图表等比渲染**: `preserveAspectRatio` 由 `none` 改 `xMidYMid meet` + `aspect-ratio:760/240`，消除宽屏横向拉伸变形；hover x 坐标换算兼容 meet 缩放与居中偏移
- **趋势与卡片一致**: `generateChartData` 末点锚定当前实时延迟，刷新后曲线末端与卡片数值对齐；去除 `refreshData` 中重复的 `generateChartData` 调用
- **API 卡片可视化**: 新增延迟进度条（含 shimmer 动效），对齐设计规范 ProgressBar 组件
- **组件库对比度**: Badge/Alert/StatusDot 文字色改用 `color-mix(...72%, var(--color-foreground))`，统一浅色模式可读性

### Docs
- **设计文档对齐实现**: `design-system.md` 颜色 token 改为 `.light` 体系与真实 hex、字体策略统一系统字体栈、动画表对齐实际类（fade-in-up/spin-once/shimmer）
- **UI/项目规范同步**: 图表描述由 Recharts 改为手写 SVG（含 React 可替换说明）、i18n 标注原型 2 语言/规划 16、版本号统一至 v2.8.5

## [2.8.4] - 2026-08-15

### Chore
- **开发工作区**: 新增 `LLM-API-Sentinel.code-workspace` 多根工作区配置
  - 编辑器设置：保存时 ESLint 自动修复 + 整理导入；Tailwind CSS 4 智能提示；路径别名 `@/*` 重命名
  - 文件/搜索排除 `node_modules`/`.next`/`out`/`dist` 等构建产物
  - 推荐扩展：ESLint、Tailwind CSS、React snippets、Error Lens、Code Spell Checker 等
  - 预置任务：dev（默认构建任务）、build、lint、test（默认测试任务）
  - 终端包管理器设为 `npm`（与 `package-lock.json` 一致）
- **版本一致性**: 同步 README 双语文档标题至 v2.8.4

## [2.8.3] - 2026-08-14

### Fixed
- **原型细节修复**: 清理 `generateChartData` 死代码（`swing`/`idx`）；移除告警横幅 `textColor` 死变量
- **刷新联动图表**: `refreshData` 重算历史曲线，使延迟趋势与刷新后最新基线一致；离线项可用性钳制下限 5%、错误率封顶 99.9%
- **浅色对比度**: 统计卡片数值、告警横幅、API 卡片图标/状态徽章、告警对话框徽章改用 `color-mix` 提升浅色模式可读性
- **Hero 计数动态化**: 顶栏"在线 · N API 状态监控"改为读取 `apis.length`，移除硬编码 12
- **组件库交互**: Switch 改用 CSS 驱动滑块动画（开/关回弹）；Tabs 支持点击切换并联动面板内容
- **版本一致性**: 原型 `title` 与 i18n 标题统一至 v2.8.3；时间范围切换同步 `aria-pressed`

## [2.8.2] - 2026-08-14

### Perf
- **缓存懒序列化**: `cache.ts` 的 `setCache` 改为增量持久化单条记录（`persistSingleCache`），避免每次写入全量 `JSON.stringify` 阻塞主线程
- **图表 memo 浅比较**: `LatencyHistoryChart` 的 `memo` 比较由全量 `JSON.stringify` 深比较改为引用/浅比较，降低重渲染开销

### Security
- **Webhook SSRF 防护**: `notification-platforms.ts` 新增 `isPrivateWebhookHost`，阻断发往 `localhost`/`127.x`/`10.x`/`192.168.x`/`172.16-31.x`/`169.254.x` 等内网/环回地址的 webhook
- **Webhook 正文截断**: `sendWebhookRequest` 对超长 payload 截断至 2000 字符，防止超大请求体被网关拒绝

### Refactor
- **模块拆分**: `notification.ts`(288→~120)、`ApiStatusGrid.tsx`(230→~120)、`useGeoLocation.ts`(242→~120) 分别抽离 `notification-platforms.ts`/`ApiStatusCard.tsx`/`geo-storage.ts`，均降至 200 行以内，保持对外 API 稳定
- **数据取默认值加固**: `metrics.ts` 移除 Supabase 字段 `as` 强转，改用类型守卫与默认值，避免 `undefined`/`NaN`
- **死代码清理**: 移除 `cache.ts` 永远返回 `null` 的 `getApiSpecificExpiry`

### Feat
- **语义化 id**: 为 `dashboard`/`status-section`/`provider-group-*`/`latency-section`/`hero`/`alerts-banner`/`app-header`/`app-footer`/`api-config`/`api-status-*` 等 UI 区块添加稳定 `id`，便于锚点与可访问性

### Test
- **补齐单元测试**: 新增 `api-config-validation`/`concurrency`/`metrics`/`cache`/`notification-platforms` 测试，覆盖率提升至 80%+

### Docs
- **版本同步**: `package.json`/`README`/`README_CN`/`openspec/config.yaml` 统一至 v2.8.2

## [2.8.1] - 2026-08-14

### Fixed
- **原型图表真实化**: 移除 Chart.js CDN 依赖，改用纯手写 SVG 渲染区域填充折线图，新增图例（可点击切换系列显隐）、hover tooltip 与垂直扫描线
- **时间范围真实差异**: `generateChartData` 按 24H/7D/30D 生成形态各异的真实延迟曲线（日内昼夜波动 / 周内工作日峰值 / 30 天趋势+尖峰），修复切换仅改按钮态的伪交互
- **离线语义修正**: 刷新时离线项保持低可用性、高错误率，不再被随机推高与"离线"语义冲突
- **同步时间戳**: 补 `last-sync-time` 节点，顶栏实时相对时间显示最后同步时刻
- **移动端分组**: 修复分组标题在窄屏 grid 断裂（`<div class="contents">` 占位无效），改用 `col-span-full` 分隔各组
- **组件库页补全**: `components.html` 补齐 Switch / Select / Input / Tabs / Avatar / Tooltip / Skeleton 展示，移除 Google Fonts 依赖改系统字体栈
- **无障碍增强**: 图表/对话框/切换按钮补充 `role`/`aria-*`/`aria-label`；tooltip 与图例过渡动画

## [2.8.0] - 2026-08-14

### Feat
- **原型重构**: 将 `prototype.html` 迁入 `prototype/` 目录并重构为组件化高保真原型
  - `index.html` — 仪表盘主原型（纯 HTML/CSS/JS，可浏览器直接打开）
  - `components.html` — 组件库规范展示页（色彩/字体/间距/基础·复合·业务组件）
  - `assets/styles.css` / `data.js` / `app.js` — 共享设计 token、真实模拟数据、交互逻辑
- **shadcn/ui 补齐**: 新增 `switch` / `select` / `tabs` 三个 new-york 风格基础组件，组件库扩展至 16 个

### Fixed
- **字体策略**: 移除 `next/font/google` 构建期网络拉取，改为 `style.css` 系统字体栈（CSS 变量），修复无外网环境构建失败
- **设计 token 对齐**: 统一 `@theme` 内 success/warning/error 色值（#22c55e / #f59e0b / #ef4444），补充 `--font-sans`/`--font-mono` 主题变量，消除与 `:root` 冗余定义冲突
- **原型缺陷修复**: 修复语言切换引用缺失元素、无效文本工具类、离线数据语义失真（availability 非 0）、延迟阈值与项目常量（1500ms）不一致等真实问题

### Docs
- **规范对齐**: 同步 `openspec/project.md`、`ui.md`、`design-system.md` 至 v2.8.0，更新组件清单（13→16）、字体策略与原型目录结构

## [2.7.2] - 2026-08-13

### Docs
- **社区健康文件**: 新增 GitHub Community Health Files
  - `LICENSE` — 补充缺失的 MIT 许可证（README 已声明）
  - `CODE_OF_CONDUCT.md` — Contributor Covenant 行为准则
  - `CONTRIBUTING.md` — 根目录贡献指南（链接并强化 `docs/contributing.md`）
  - `SECURITY.md` — 安全政策与漏洞私下报告流程
  - `SUPPORT.md` — 支持渠道与常见问题
  - `.github/PULL_REQUEST_TEMPLATE.md` — PR 模板
  - `.github/ISSUE_TEMPLATE/` — Bug Report / Feature Request / config 模板

### Refactored
- **模块化拆分**: 将超 200 行的源文件按职责拆分为更小模块，提升可读性与可维护性
  - `DashboardClient.tsx` → 抽离 `HeroSection` / `AlertsBanner` 子组件与 `useDashboardStats` hook
  - `ApiConfig.tsx` → 抽离 `api-config-validation.ts`（校验逻辑与类型）
  - `ApiStatusGrid.tsx` → 抽离 `ApiCard.tsx` 子组件
  - `useApiMonitor.ts` → 抽离 `supabase-mapping.ts`（数据映射）与 `alert-service.ts`（告警逻辑）
- **类型安全**: 移除 `metrics.ts` 中的 `any` 类型，改用显式 `Record` 映射

### Fixed
- **版本对齐**: 统一各模块头注释与 `package.json` 版本号

## [2.7.1] - 2026-08-13

### Docs
- **README 双语**: 修正运行模式说明——默认静态导出（`out/`）部署，Express `server.ts` 为可选的自定义安全服务器（Helmet 响应头 + 按 IP 速率限制），两者互斥。
- **README 双语**: 包管理器统一为 `pnpm`，修正 `.env.local` 配置流程，补充 `docs/` 文档链接与项目结构（`store/`、`constants/`、`types/` 目录与 `server.ts`）。
- **版本对齐**: 将文档与 `package.json` 版本号统一至代码实际版本 `v2.7.1`。
- **新增文档**: 新增 `docs/env.md`（环境变量参考）、`docs/deployment.md`（部署指南）、`docs/security.md`（安全架构）、`docs/contributing.md`（贡献指南）。
- **openspec**: 修正架构/功能文档版本号与运行模式描述，新增可选安全服务器说明。

### Fixed
- **README**: 修复部署小节中遗留的中英混排错误文本。

## [2.6.3] - 2026-06-10

### Changed
- **Documentation**: Updated README files to remove Firebase Hosting references (migrated to Supabase).
- **Project Structure**: Updated README project structure to reflect actual directory layout (removed server.ts).

### Removed
- **Firebase Legacy Files**: Deleted unused Firebase configuration files:
  - `firebase.json`
  - `firebase-applet-config.json`
  - `firebase-blueprint.json`
  - `firestore.rules`
- **Unused Server Files**: Deleted `server.ts` and `edgeone.config.js`.
- **Unused Metadata**: Deleted `metadata.json` and `LLM-API-Sentinel.code-workspace`.
- **Redundant Lock File**: Deleted `package-lock.json` (project uses pnpm).
- **Unused Components**: Deleted `StatusGrid.tsx` wrapper component and `use-mobile.ts` hook.
- **Build Scripts**: Removed `build:functions`, `deploy:functions`, and `serve:functions` scripts from package.json.

### Fixed
- **Test Configuration**: Removed `app/lib/error.test.ts` from Jest ignore patterns to enable test execution.
- **TypeScript Config**: Removed `functions` directory from tsconfig.json exclude patterns.
- **i18n Files**: Updated 16 language files to replace Firebase error messages with Supabase equivalents.
- **Error Handler**: Replaced FIREBASE enum values with SUPABASE in error-handler.ts.

## [2.6.2] - 2024-12-08

### Changed
- **Supabase Migration**: Replaced Firebase with Supabase for data storage and authentication.
  - Replaced Firebase Firestore with Supabase PostgreSQL
  - Replaced Firebase Authentication with Supabase Auth
  - Added Supabase Realtime subscriptions for live alert updates
- **Database Schema**: Created new PostgreSQL schema with tables for api_status, status_history, alerts, and user_profiles.
- **Security**: Implemented Row Level Security (RLS) policies for database access control.
- **Dependencies**: Updated package.json to include @supabase/supabase-js and removed Firebase dependencies.
- **Documentation**: Updated README files to reflect Supabase migration and new features.

### Fixed
- **Type Safety**: Fixed TypeScript error in useAuth.ts where `session.user.email` could be undefined.
- **Build Configuration**: Removed unused firebase.ts file that was causing build errors.
- **Supabase Client**: Fixed graceful handling of missing environment variables for Supabase client.
- **Code Quality**: Removed unused imports (Alert in useApiMonitor.ts, User in useAuth.ts).

### Added
- **Database Migration Script**: Added supabase/schema.sql with complete database setup.
- **Environment Configuration**: Created .env.example with Supabase configuration variables.
- **Version Consistency**: Unified all file version numbers to v2.6.2.

### Removed
- Firebase client and admin SDKs
- Legacy firebase.ts file (no longer imported)
- Firebase configuration files from main codebase

## [2.6.1]
### Added
- **Complete Internationalization System**: Implemented full i18n support with English and Chinese translations, integrated into all components.
- **Comprehensive Translation Files**: Added missing translation keys and updated all user-facing text to use the translation system.

### Improved
- **Component Localization**: Updated all UI components to use the i18n system for all text elements.
- **Version Consistency**: Unified all file version numbers to v2.6.1.

### Fixed
- **Code Quality**: Reviewed and fixed minor issues across the codebase.

## [2.5.1]
### Added
- **Firestore Direct Integration**: Frontend now directly reads API status from Firestore, eliminating dependency on API routes.
- **Enhanced Monitoring Hook**: useApiMonitor hook improved with intelligent alert checking.
- **Caching System**: Multi-layer caching with intelligent expiry calculation.

### Fixed
- **Documentation**: Removed internationalization references from README files.
- **README Cleanup**: Removed duplicate version sections from README.md.
- **Version Consistency**: Unified all file version numbers to v2.5.1.

## [2.5.0]
### Added
- **Data Caching**: Implemented memory and local storage caching for API check results, reducing duplicate requests.
- **Enhanced Monitoring Metrics**: Added error rate, availability, and uptime metrics for each API.
- **Alert Notifications**: Implemented email and SMS notification functionality for API alerts.
- **API Configuration**: Added user-friendly API configuration interface for customizing API checks.
- **Improved UI/UX**: Enhanced API status grid with provider grouping, better card design, and improved empty state handling.

### Changed
- **Version Sync**: Updated all component version numbers to v2.5.0.
- **API Configuration**: Modified APIS_TO_CHECK to load from local storage for persistent user configurations.

## [2.4.3]
### Fixed
- **Code Quality**: Comprehensive code review and bug fixes across all components.
- **i18n Integrity**: Verified and corrected all internationalization language files for complete translation coverage.
- **Spec Sync**: Synchronized all code functionality details to openspec documentation.
- **Version Coherence**: Unified all file version numbers to v2.4.3 for consistency.

## [2.4.1]
### Fixed
- **Build Configuration**: Renamed `next.config.js` to `next.config.mjs` to fix ES module scope error in CI environment.
- **Deprecated Config**: Removed deprecated `i18n` and `experimental.turbo` settings from next.config.mjs.

## [2.4.0]
### Added
- **Security Enhancements**: Added authorization to API health check endpoint.
- **Firebase Configuration**: Improved Firebase config loading to support environment variables.
- **Internationalization**: Completed translation files for Spanish and Arabic.

## [2.2.0]
### Added
- **Autonomous Monitoring**: Implemented a server-side background task that performs API checks every 5 minutes without user intervention.
- **Custom Server**: Migrated to a custom Express server to support long-running background tasks.
- **Firebase Admin**: Integrated `firebase-admin` for secure server-side Firestore updates.

## [2.1.0]
### Added
- **Real-time Alerting**: Implemented a notification system for API downtime and high latency (>1500ms).
- **Alerts UI**: Added a notification bell with dropdown and a global alert banner.
- **Alert Management**: Authenticated users can resolve active alerts.

## [2.0.1]
### Added
- **Security**: Hardened firestore.rules by restricting write access to admins only.
- **Robustness**: Fixed build errors related to Firebase Admin imports and FieldValue usage.
- **Versioning**: Synchronized version numbers across all files.

## [2.0.0]
### Added
- **Internationalization (i18n)**: Implemented multi-language support (ar, cs, en, es, hi, id, it, nl, pl, sv, th, tr, ru, vi, zh-cn, zh-tw) using `next-intl`.
- **Global API Coverage**: Added major AI providers from China (Moonshot, Zhipu, Baichuan, Alibaba, Tencent, Baidu) and US (Meta/Groq, Mistral).
- **Dark Mode**: Implemented full dark/light theme support with `next-themes`.
- **Responsive Design**: Optimized layout for desktop, tablet, and mobile devices.
- **GEO Info**: Added real-time geographic location detection for the monitoring node.
- **SEO Optimization**: Added comprehensive meta tags and OpenGraph support.
- **Semantic IDs**: Added unique IDs to all major containers for easier debugging.
- **Documentation**: Created bilingual README files (EN/CN).

### Changed
- **Refactoring**: Moved core logic directories (`lib`, `hooks`, `components`) into the `app/` directory for better structure.
- **Styling**: Extracted page-specific styles into `app/style.css`.
- **Headers**: Standardized all code file headers to a single-line format.

## [1.0.0]
### Added
- Initial release of LLM API Sentinel.
- Real-time status monitoring for OpenAI, Anthropic, Google, and DeepSeek.
- Historical latency tracking with Area Charts.
- Firebase integration for data persistence.
