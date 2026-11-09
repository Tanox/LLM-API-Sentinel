# 发展路线与改进建议 (Roadmap & Improvement Proposals)

> 适用版本：v2.10.35 · 最近校对：2026-11-09

本文档基于项目当前状态（v2.10.35：静态前端 + Supabase 后端、29 个中美 API 真实探测、16 语言、Recharts 图表、浅色/深色双主题、可选 Express 安全服务器、CI/CD 已落地、无演示数据），提出务实的发展方向。按优先级分为短期、中期、长期三类，并单列工程质量项。

## 当前能力基线

- **监控对象**：29 个 LLM API 端点（美国/全球 16 + 中国 13），覆盖 OpenAI / Anthropic / Google / Meta / Mistral / xAI / Cohere / Perplexity / 智谱 / 阿里 / 腾讯 / 百度 / DeepSeek / Kimi / 火山等；单一真源 `app/constants/apis.json` 由 `sync-apis.mjs` 同步到 Edge Function。
- **核心功能**：连通性 + 延迟监控、扁平状态网格（不按供应商分组）、历史趋势（1H / 6H / 24H）、阈值告警（离线 `critical` / 高延迟 `high`）、Supabase Auth Google OAuth（仅登录）、`alerts` 表 Realtime 订阅 + 前端 5 分钟定时探测。**数据真实性**：自 v2.10.22 起移除演示数据注入，Supabase 未配置或加载失败时为空态（骨架占位）等待真实探测；累计指标经 `metrics-storage.ts` 写入 localStorage 跨刷新累加。
- **部署**：默认静态导出（`out/`）至 Vercel / EdgeOne / Netlify；可选 `examples/self-host-server.ts` 自建安全服务器（Helmet + 全局限流 100 次/60 秒）；后台持久化可选 `supabase/cron.sql` + `functions/monitor`。
- **工程现状**：Jest 覆盖率门禁（`jest.config.cjs`：statements/functions/lines ≥ 70%，branches ≥ 55%）；CI（`ci.yml`，含 `sync-apis.mjs --check`）+ Release（`release.yml`）已上线；多渠道告警已支持 Webhook / Slack / Teams / Discord / 钉钉 / 飞书（默认关闭）；Firebase 残留配置仅作迁移参考。

---

## 短期（稳定性与工程化）

### 1. 引入 CI/CD（✅ 已完成）
- `ci.yml`：PR/push 到 main/dev 触发 `pnpm install --frozen-lockfile` → `node scripts/sync-apis.mjs --check` → `pnpm lint` → `pnpm test -- --coverage` → `pnpm build`。
- `release.yml`：语义化版本 tag 触发静态产物构建与 GitHub Release。
- 门禁：测试/覆盖未过禁止合并（配合 PR 模板检查清单）。

### 2. 补全单元测试与覆盖率（🟡 进行中）
- 已覆盖纯逻辑模块：`app/lib/` 下 monitor / cache / cache-validation / cache-storage / concurrency / i18n / metrics / metrics-storage / notification / notification-platforms / webhook-formatter / utils / supabase-mapping 等，以及 `app/components/api-config-validation.test.ts`。
- 待补：为 `LatencyHistoryChart` 等 **Recharts** 组件补组件级测试；提升 branches 覆盖率。

### 3. 落地后台监控的官方示例（✅ 已完成）
- 已提供 `supabase/functions/monitor/index.ts` 示例 Edge Function（探测 29 个 API 并写入 `api_status` / `status_history`，含白名单 SSRF 防护与 6 秒超时中断）。
- 已提供 `supabase/cron.sql` 示例（pg_cron 每 5 分钟调度；90 天 `status_history` 保留策略为**注释掉的示例，默认未启用**）。

---

## 中期（功能增强）

### 4. 扩展监控维度
- **成本/配额感知**：除连通性与延迟外，记录 429 / 配额耗尽状态，区分「宕机」与「限流」。
- **模型级可用性**：部分供应商多模型端点，细化到模型粒度的健康度。
- **多地域探测**：同一 API 从多个地理节点探测，呈现区域差异延迟。

### 5. 历史 SLA 与公开状态页
- 基于历史数据计算**滚动可用性百分比**（如 30 天 uptime %），在卡片/状态页展示。
- 提供独立公开状态页（子路径 `/status` 或独立子域），无需登录即可查看。

### 6. 可配置监控目标
- 允许用户在 UI 中添加自定义 API endpoint（URL + 预期状态码 + 检查频率）。
- 当前 `ApiConfig` 仅支持本地 `localStorage.apiConfig` 配置与恢复默认；持久化到 Supabase、纳入统一监控与告警仍待实现。

### 7. 多渠道告警（✅ 基础能力已完成）
- 已支持 Webhook / Slack / Teams / Discord / 钉钉 / 飞书 / 通用 Webhook（`app/lib/notification-platforms.ts`），默认 `enabled: false`。
- 待办缩窄为：Email / Telegram 渠道、告警规则可配置（阈值、静默窗口、升级策略）。

---

## 长期（平台化）

### 8. 多语言公开状态页 + SEO
- 主站 SEO 已完成：`app/components/StructuredData.tsx`（JSON-LD）、`app/robots.ts` / `app/sitemap.ts` 与 `public/` 下的静态副本。
- 待办：独立状态页支持 16 语言切换，并为该页面补充专属结构化数据与 sitemap 条目。

### 9. 公开状态 API 与嵌入式组件
- 提供只读公开 API（带限流），供第三方查询状态。
- 提供可嵌入的轻量状态徽章/Widget（`<script>` 或 iframe）。

### 10. 移动端体验
- 当前为响应式 Web；可考虑 PWA（离线缓存状态快照 + 推送通知）或独立移动 App。

### 11. 周期报告与导出
- 自动生成周/月可用性报告（PDF / 邮件）。
- 支持历史数据 CSV 导出。

---

## 工程质量与技术债

### 12. 清理 Firebase 残留（✅ 已完成）
- `package.json` 已无 Firebase 依赖；仅 `.env.example` 与 `docs/env.md` 保留 3 个变量名作迁移注释，无运行时读取。

### 13. 统一运行模式叙事（✅ 路径已统一）
- 静态导出与自定义服务器互斥，现已在 README 双语、[deployment.md](deployment.md) 与 openspec 文档中统一使用 `examples/self-host-server.ts`（根目录 `server.ts` 已于 v2.10.29 移除）。
- 待办：将安全头/限流逻辑移植到 Edge 中间件，使静态托管也能获得同等加固。

### 14. 数据保留与聚合策略
- 定义历史数据保留窗口（如 90 天明细 + 聚合归档）；`cron.sql` 仅提供注释示例，`alerts` 无自动清理。

### 15. 可访问性 (a11y) 审计
- 对状态网格、图表、对话框做 WCAG AA 审计（对比度、键盘导航、ARIA）；浅色/深色双主题均需验证色盲友好性。

---

## 建议落地顺序

1. ~~**CI/CD + 测试覆盖**~~（✅ 已完成，持续提升覆盖率）
2. ~~**后台监控示例 + 多渠道告警基础能力**~~（✅ 已完成；Email/Telegram 与规则配置仍待办）
3. **公开状态页 + SLA**（对外价值最大化）
4. **可配置目标 + 平台化 API**（向多租户/生态演进）
5. 持续清理技术债

> 本路线为建议性文档，具体实现请以 `openspec/changes/` 下的变更提案与 PR 为准；已完成的提案保留历史版本号，不代表当前版本。
