# Contributing to LLM API Sentinel

> 适用版本：v2.10.35 · 最近校对：2026-11-09

感谢你为 LLM API Sentinel 做出贡献！本文件说明如何参与开发与提交变更。更深入的规范细节见 [docs/contributing.md](../docs/contributing.md)。

## 行为准则

参与本项目即表示你同意遵守 [Code of Conduct](CODE_OF_CONDUCT.md)。

## 开发环境

```bash
pnpm install
cp .env.example .env.local   # 填入你的 Supabase 凭证
pnpm dev                     # 启动开发服务器 http://localhost:3000
```

> 包管理器固定为 pnpm 9.15.0（`preinstall` + `.npmrc` 强制），CI 固定 Node.js 22。

常用脚本：

| 命令 | 作用 |
|------|------|
| `pnpm dev` | 开发模式 |
| `pnpm build` | 先跑 `preflight`，再静态导出到 `out/` |
| `pnpm start` | `next start`（静态导出模式下请改用静态托管） |
| `pnpm lint` | ESLint 检查 |
| `pnpm test` | Jest 单元测试（含覆盖率） |
| `pnpm test:watch` | 监听模式测试 |
| `pnpm sync:apis` | 同步 `apis.json` 单一真源到 Edge Function |
| `pnpm preflight` | 仅校验 `apis.json` 一致性（CI 门禁） |
| `pnpm clean` | `next clean` |

## 分支策略

| 分支 | 用途 |
|------|------|
| `main` | 生产环境代码 |
| `dev` | 集成分支 |
| `feature/*` | 新功能开发 |
| `fix/*` | 缺陷修复 |

## 提交规范

提交信息遵循 **Conventional Commits**：

```
<type>: <description>

[可选正文]

[可选页脚]
```

类型：`feat` / `fix` / `docs` / `style` / `refactor` / `test` / `chore` / `perf` / `ci` / `revert`

- 描述简短（≤50 字符）、首字母小写、动词开头、无句号。
- 涉及版本变更时，在正文或页脚标注新版本号。

## 版本号规则 (SemVer)

每次修改（新增/删除/修改代码或文档）都需升级版本号：

- **patch**（x.y.Z+1）：修复、文档、重构、样式、配置、依赖等任意修改。
- **minor**（x.Y+1.0）：向后兼容的新功能。
- **major**（X+1.0.0）：破坏性变更。

需同步更新的展示位（版本单一真源：`package.json` 的 `version`）：

- `package.json` 的 `version` 字段
- `openspec/config.yaml` 中 `context: |` 文本块内的 `Version:` 行
- 被改动文件的头注释 `// path vX.Y.Z`（仅改动的文件）
- `CHANGELOG.md` 新增对应版本小节
- `README.md` / `README_CN.md` 的 **H1 标题版本号**
- `openspec/` 规范文档头部的适用版本（如已标注）

`openspec/changes/` 下的历史提案保留原版本号。

## 代码规范

- 关键逻辑添加中文注释。
- 函数超过 20 行考虑拆分；源文件超过 200 行按职责拆分为子模块（文档不拆分）。
- TypeScript 启用 strict 模式，避免 `any`。
- 正确捕获和处理 Promise 异常。
- 无 `console.log` / `debugger` 残留。
- 国际化文本提取到 `app/locales/` 翻译文件（16 语言齐全）。
- 监控目标只改 `app/constants/apis.json`，改后运行 `pnpm sync:apis`。

## Pull Request 流程

1. 从 `dev` 或 `main` 切出 `feature/*` 或 `fix/*` 分支。
2. 完成开发并通过 `pnpm preflight`、`pnpm lint`、`pnpm test`、`pnpm build`。
3. 升级版本号并补充 `CHANGELOG.md`。
4. 提交 PR 到 `main` 或 `dev`，PR 标题遵循提交规范。
5. PR 描述包含：变更总结 + 动机 + 测试说明。

## 提交前检查

- [ ] `pnpm preflight` 通过
- [ ] `pnpm lint` / `pnpm test` / `pnpm build` 通过
- [ ] 版本号已同步升级
- [ ] `CHANGELOG.md` 已更新
- [ ] 文档与代码事实一致

## 文档约定

规范文档在 `openspec/`（架构、数据模型、UI、设计系统、功能、逻辑、项目规范，另有 `specs/` 与 `changes/`），补充文档在
`docs/`（`env.md`、`deployment.md`、`security.md`、`contributing.md`、`roadmap.md`）。修改架构或功能时，请同步更新对应
`openspec/` 文档与双语 README。
