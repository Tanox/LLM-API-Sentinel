# 贡献指南 (Contributing)

> 适用版本：v2.10.35 · 最近校对：2026-11-09

感谢你为 LLM API Sentinel 做出贡献！请遵循以下规范，确保协作顺畅、历史清晰。

## 1. 开发环境

```bash
pnpm install
cp .env.example .env.local   # 填入你的 Supabase 凭证
pnpm dev                     # 启动开发服务器 http://localhost:3000
```

> 包管理器固定为 pnpm 9.15.0（`preinstall` + `.npmrc` 强制），CI 固定 Node.js 22。

常用脚本（`package.json`）：

| 命令 | 实际行为 |
|------|----------|
| `pnpm dev` | `next dev` |
| `pnpm build` | 先执行 `preflight`，再 `next build` 静态导出到 `out/` |
| `pnpm start` | `next start`（与静态导出模式不等价，见 [deployment.md](deployment.md)） |
| `pnpm lint` | `eslint .` |
| `pnpm test` | `jest --coverage`（已带覆盖率） |
| `pnpm test:watch` | `jest --watch` |
| `pnpm sync:apis` | 把 `app/constants/apis.json` 同步到 `supabase/functions/monitor/apis.json` |
| `pnpm preflight` | `node scripts/sync-apis.mjs --check`，只校验一致性、不写文件 |
| `pnpm clean` | `next clean` |

CI（`.github/workflows/ci.yml`）门禁顺序：`pnpm install --frozen-lockfile` → `node scripts/sync-apis.mjs --check` → `pnpm lint` → `pnpm test -- --coverage` → `pnpm build`。

## 2. 分支策略

| 分支 | 用途 |
|------|------|
| `main` | 生产环境代码 |
| `dev` | 集成分支 |
| `feature/*` | 新功能开发 |
| `fix/*` | 缺陷修复 |

## 3. 提交规范

提交信息遵循 **Conventional Commits**：

```
<type>: <description>

[可选正文]

[可选页脚]
```

类型：`feat` / `fix` / `docs` / `style` / `refactor` / `test` / `chore` / `perf` / `ci` / `revert`

- 描述简短（≤50 字符）、首字母小写、动词开头、无句号。
- 涉及版本变更时，在正文或页脚标注新版本号。

示例：

```
docs: 完善部署与环境变量文档

- 补充 Edge Function /monitor 的返回体说明
- 澄清静态导出与可选自建服务器两种运行模式

版本: v2.10.35
```

## 4. 版本号规则 (SemVer)

每次修改（新增/删除/修改代码或文档）按规则升级版本号：

- **patch**（x.y.Z+1）：修复、文档、重构、样式、配置、依赖等任意修改。
- **minor**（x.Y+1.0）：向后兼容的新功能。
- **major**（X+1.0.0）：破坏性变更。

需同步更新的展示位（版本单一真源为 `package.json` 的 `version`）：

- `package.json` 的 `version` 字段（唯一权威源）
- `openspec/config.yaml` 中 `context: |` 文本块内的 `Version:` 行（**不是**顶层 YAML 字段）
- 被改动文件的头注释 `// path vX.Y.Z`（仅改动的文件，禁止全仓库批量刷写）
- `CHANGELOG.md` 新增对应版本小节
- `README.md` / `README_CN.md` 的 **H1 标题版本号**（不是徽章）
- `openspec/` 各规范文档头部的适用版本（如已标注）
- `prototype/` 各页面标注的版本号（如本次涉及原型）

`openspec/changes/` 下的归档提案保留其历史版本号，不随当前版本更新。

## 5. 代码规范

- 关键逻辑添加中文注释。
- 函数超过 20 行考虑拆分；源文件超过 200 行按职责拆分为子模块（文档文件不拆分）。
- TypeScript 启用 strict 模式，避免 `any`。
- 正确捕获和处理 Promise 异常。
- 无 `console.log` / `debugger` 残留（日志走 `error-handler`）。
- 国际化文本提取到 `app/locales/` 翻译文件；新增词条需 16 语言齐全。
- 监控目标只改 `app/constants/apis.json`，随后运行 `pnpm sync:apis`。

## 6. 提交前检查

- [ ] `pnpm preflight` 通过（API 单一真源一致）
- [ ] `pnpm lint` 通过
- [ ] `pnpm test` 通过（覆盖率阈值：statements/functions/lines ≥ 70%，branches ≥ 55%）
- [ ] `pnpm build` 通过
- [ ] 版本号已同步升级
- [ ] `CHANGELOG.md` 已更新
- [ ] 文档与代码事实一致

## 7. 文档约定

本项目文档分两层：

- **规范文档** `openspec/`：架构、数据模型、UI、设计系统、功能、逻辑、项目规范；另有 `specs/`（规范增量，当前为空）与 `changes/`（历史变更提案）。
- **补充文档** `docs/`：`env.md`、`deployment.md`、`security.md`、`contributing.md`、`roadmap.md`。

修改架构或功能时，请同步更新对应 `openspec/` 文档与双语 README，保持规范与实现一致。历史变更记录只写入 `CHANGELOG.md`，不在其他文档另建任务清单。
