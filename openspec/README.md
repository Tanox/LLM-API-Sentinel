# LLM API Sentinel 项目规范 (OpenSpec)

> 适用版本：v2.10.35 · 最近校对：2026-11-09 · 版本单一真源：`package.json`

本目录是项目的规范（ specification ）主目录，描述架构、数据模型、UI 组件、设计系统、功能、逻辑服务及开发约定。当前规范以平铺的 `*.md` 文件为准；`specs/` 为增量规范目录（目前为空），`changes/` 仅保存历史变更提案（保留其原始版本号）。

## 目录结构

| 文档 | 内容 |
|------|------|
| [架构概览](architecture.md) | 运行模式、技术栈、系统架构图、部署与数据流 |
| [数据模型与安全](data.md) | Supabase 表结构、类型定义、RLS 与写入流程 |
| [UI 组件规范](ui.md) | 组件 Props、布局、动效与主题变量 |
| [设计系统](design-system.md) | 色彩/字体/间距/动画 token 与原型对齐关系 |
| [功能规格](features.md) | 功能清单、交互行为与验收标准 |
| [逻辑与服务](logic.md) | 探测、缓存、指标、告警、i18n 的实现说明 |
| [项目规范](project.md) | 技术栈、目录结构、接口签名、CI 与版本规则 |
| [config.yaml](config.yaml) | OpenSpec 上下文与产出规则 |
| [specs/](specs/) | 规范增量目录（当前为空） |
| [changes/](changes/) | 历史变更提案（归档，版本号不代表当前版本） |

## 原型

- [高保真原型](../prototype/prototype.html)
- [线框图](../prototype/wireframes.html)
- [组件库示例](../prototype/components.html)
- [原型索引](../prototype/index.html)

> 原型为历史参考，React 实现的真实行为以 `app/` 下代码为准；两处不一致时以代码为准。

## 相关文档

- [README.md](../README.md) / [README_CN.md](../README_CN.md) — 项目概览与快速开始
- [docs/](../docs/) — 环境变量、部署、安全、贡献指南、路线图
- [CHANGELOG.md](../CHANGELOG.md) — 版本变更记录
