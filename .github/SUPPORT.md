# Support

> 适用版本：v2.10.35 · 最近校对：2026-11-09

感谢使用 LLM API Sentinel！在遇到问题前，请先尝试以下自助渠道。

## 文档

- [README.md](../README.md) / [README_CN.md](../README_CN.md) — 项目概览与快速开始
- [docs/env.md](../docs/env.md) — 环境变量配置
- [docs/deployment.md](../docs/deployment.md) — 部署指南（Vercel / EdgeOne / 自托管）
- [docs/security.md](../docs/security.md) — 安全架构
- [docs/contributing.md](../docs/contributing.md) — 贡献指南
- [docs/roadmap.md](../docs/roadmap.md) — 发展路线
- [openspec/](../openspec/) — 架构、数据模型、UI 与功能规范
- [CHANGELOG.md](../CHANGELOG.md) — 版本变更记录

## 常见问题

**Q: 页面无法连接 Supabase？**
A: 确认 `.env.local` 中的 `NEXT_PUBLIC_SUPABASE_URL` 与 `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   已正确填写，且已运行 `supabase/schema.sql` 创建数据表。未配置 Supabase 时页面仍能
   进行浏览器端真实探测，但不会持久化历史数据。

**Q: 后台监控 / 历史数据不更新？**
A: 默认部署由浏览器端 `useApiMonitor` 每 5 分钟自动探测（数据仅在本地会话中累加）；
   若需要服务端持久化，请部署 `supabase/functions/monitor` 并执行 `supabase/cron.sql`
   注册 pg_cron 定时任务，同时确认 Edge Function 的 `SUPABASE_URL` /
   `SUPABASE_SERVICE_ROLE_KEY` 已由平台注入。

**Q: 构建报 API 清单不一致？**
A: 运行 `pnpm sync:apis` 把 `app/constants/apis.json` 同步到
   `supabase/functions/monitor/apis.json`，再执行 `pnpm build`。

## 获取帮助

- **Bug 报告 / 功能建议**：请在 GitHub 提交 Issue（使用对应模板）。
- **使用疑问**：可在 Issue 中描述，或在 Discussions（如已启用）中提问。
- **安全漏洞**：请勿公开提 Issue，请按 [SECURITY.md](SECURITY.md) 私下报告。

对于使用类问题，请在提交 Issue 前确认已阅读上述文档，并提供：

- 复现步骤
- 期望行为与实际行为
- 浏览器 / 运行环境信息
- 相关日志或截图
