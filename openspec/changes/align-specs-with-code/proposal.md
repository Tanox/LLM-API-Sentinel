# Change: 对齐 openspec 规范与 app 代码实现

> **历史归档**：本提案保留提出时的版本号、路径与命令（如 v2.5.0 / `.trae/rules/` / `/workspace/` / npm 命令），仅作历史记录，不代表 v2.10.35 的当前状态。当前事实以双语 README、`docs/` 与 `openspec/*.md` 顶部的「代码对齐要点」为准。
**Status**: Approved
**Date**: 2026-08-16
**Version**: 2.10.0

## Why

`standardize-openspec-docs` change 曾声称「无过时信息」，但核对代码后发现
`openspec/features.md` 与 `openspec/ui.md` 仍大量引用早期实现，与 `app/`
实际代码存在 9 类偏差。本 change 将规范更新为代码的准确镜像，并同步
改造 `share-content.ts` 的分享键名以消除偏差根因。

## What Changes

### 规范文档修正（features.md / ui.md）
1. **API 监控范围**：12 → 29 个（与 `app/constants/index.ts` DEFAULT_APIS 一致），补全 §2.1 表格
2. **主题**：保持浅色/深色双主题（next-themes `defaultTheme="system"`），`:root`/`.light` 浅色、`.dark` 深色；顶栏 Sun/Moon 切换（**注**：v2.10.9 的「纯深色双档」描述与 `app/style.css` 实际不符，已于 v2.10.12 纠错）
3. **ApiStatusGrid**：按 provider 分组 → 扁平网格（grid 4 列 @ xl/2xl）
4. **图表库**：保持 Recharts 3.8.0 面积图（`ssr:false` 动态导入 + ChartSkeleton）；**注**：v2.10.9 误写为「手写 SVG」，已于 v2.10.12 纠错，以 `app/components/LatencyHistoryChart.tsx` 为准
5. **页面宽度**：max-w-7xl → max-w-[1600px]
6. **Alerts 色值**：medium=amber-500 / low=blue-400（对齐代码）
7. **分享键名**：明确 `share.promos` 数组语义（与代码改造后的实现一致）

### 代码改造
- `app/lib/share-content.ts`：`promo1..5` 平铺键 → 真正的 `share.promos` 数组，
  `getRandomPromo()` 随机取数组元素，消除与规范的键名偏差根因
- `app/locales/*.json`（16 语言包）：`share.promo1..5` → `share.promos` 数组
- `app/components/ApiStatusCard.tsx`：`containIntrinsicSize` 估算高度 168px → 200px，
  避免离线/错误率内容溢出导致滚动跳动

### 原型归档
- `prototype/` 顶部与 `data.js` 头部标注「历史原型归档」及与 app 的差异清单，
  明确规范以 openspec/ 为准、代码以 app/ 为准

## Impact

- 规范文档现准确反映 v2.10.0 代码实现，消除文档与代码不一致导致的误导
- 分享功能键名统一为数组语义，i18n 结构更简洁
- prototype/ 明确为历史参考，不再作为实现依据

## Tests

- `npm run test` i18n 完整性测试：16 语言包 key 一致（已含 promos 数组）
- 手动验证：分享按钮复制内容 = 链接 + 随机 promos 文案
