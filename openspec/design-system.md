# LLM API Sentinel 设计系统 (v2.10.35)

> 适用版本：v2.10.35 · 最近校对：2026-11-09 · 真源：`app/`、`supabase/` 与 `package.json`
>
> **代码对齐要点（下文旧表述与本表冲突时，以本表及源码为准）**：
> - 主题为浅色/深色双主题：`:root` / `.light` 浅色（`#f8fafc`）、`.dark` 深色（`#0a0a0f`），深色档使用 Indigo/Violet 主色；`next-themes` 默认 `system`。
> - `app/style.css` 的 `--font-sans` 以 `"Inter"` 开头；`prototype/assets/styles.css` 的字体栈细节与 React 应用并不完全一致，以 `app/style.css` 为准。
> - app 实际 keyframes：`pulse-gentle` / `fade-in-up` / `slide-in-right` / `scale-in-gentle` / `shimmer` / `float` / `glow-pulse` / `gradient-shift`；无 `spin-once` 与 `status-dot-*`（原型侧才有）。
> - Recharts 入场动画 `animationDuration={1200}`；文中 `file:///workspace/...` 绝对路径一律按仓库相对路径理解。

## 1. 设计哲学

### 1.1 核心理念
- **极简主义**: 去除一切非必要元素，每个像素都有其存在的意义
- **功能优先**: 数据可视化和状态监控是核心，UI 服务于信息传递
- **深度沉浸**: 浅色/深色双主题（默认跟随系统），深色档以 Dark Indigo 营造专注的监控环境

### 1.2 美学方向
- **风格**: 极简工业风 + 数据仪表盘美学
- **参考**: Vercel Dashboard + Linear App + Cloudflare Analytics
- **情绪**: 冷静、专业、可信、高科技

---

## 2. 色彩系统

> 色彩通过 Tailwind CSS 4.1 的 `@theme` 定义语义 token（如 `--color-background` → `bg-background`），由 `app/style.css` 落地为 CSS 变量。所有组件使用语义化 class（如 `bg-primary`、`text-muted-foreground`），禁止硬编码色值。下表 Hex 与 `app/style.css` 严格一致。

### 2.1 主题机制（浅色 / 深色双主题）
> 采用 next-themes（`attribute="class"`，`defaultTheme="system"`，`enableSystem`），在 `<html>` 上切换 `.light` / `.dark` 类：**浅色 `:root`/`.light`** 为默认浅色（Slate 系），**深色 `.dark`** 为 Dark Indigo 沉浸主题。两套 token 均在 `app/style.css` 显式定义，主题偏好持久化到 localStorage（`theme` key）。以 `app/style.css` 实现为准。

### 2.2 主色调 (Indigo)
| Token | Hex (`.light`) | Hex (`.dark`) | 用途 |
|-------|---------------|---------------|------|
| `--primary` | `#6366f1` | `#818cf8` | 主色、按钮、链接、高亮 |
| `--primary-foreground` | `#ffffff` | `#0a0a0f` | 主色上的文字 |
| `--accent` | `#8b5cf6` | `#a78bfa` | 强调色、渐变辅助色 |
| `--accent-foreground` | `#ffffff` | `#0a0a0f` | 强调色上的文字 |
| `--ring` | `#6366f1` | `#818cf8` | 焦点环 |

### 2.3 功能色 (语义色)
语义色额外提供 `-10` alpha 变体（如 `--color-success-10`），用于浅底填充、边框、光晕等低对比场景。

| Token | Hex (`.light`) | Hex (`.dark`) | 用途 |
|-------|---------------|---------------|------|
| `--success` | `#22c55e` | `#22c55e` | 在线、正常状态 |
| `--warning` | `#f59e0b` | `#f59e0b` | 降级、警告状态 |
| `--destructive` | `#ef4444` | `#f87171` | 离线、错误状态 |
| `--info` | `#3b82f6` | `#3b82f6` | 信息提示 |

### 2.4 中性色（浅色 / 深色双主题：`.light` / `.dark`）
| Token | Hex (`.light`) | Hex (`.dark`) | 用途 |
|-------|---------------|---------------|------|
| `--background` | `#f8fafc` | `#0a0a0f` | 页面背景 |
| `--foreground` | `#0f172a` | `#f4f4f5` | 主文字颜色 |
| `--card` | `#ffffff` | `#13131a` | 卡片背景 |
| `--card-foreground` | `#0f172a` | `#f4f4f5` | 卡片文字 |
| `--popover` | `#ffffff` | `#13131a` | 弹出层背景 |
| `--popover-foreground` | `#0f172a` | `#f4f4f5` | 弹出层文字 |
| `--secondary` | `#e2e8f0` | `#1e1e28` | 次要背景 |
| `--secondary-foreground` | `#0f172a` | `#f4f4f5` | 次要背景上的文字 |
| `--muted` | `#e2e8f0` | `#1e1e28` | 弱化背景 |
| `--muted-foreground` | `#64748b` | `#71717a` | 次要/辅助文字 |
| `--border` | `#e2e8f0` | `#242430` | 边框 |
| `--input` | `#e2e8f0` | `#242430` | 输入框背景 |

### 2.5 背景氛围光 (Radial Gradient)
body 背景在纯色基础上叠加三层径向渐变，营造柔和紫色光晕：
```css
radial-gradient(circle at 20% 80%, rgba(99, 102, 241, 0.05) 0%, transparent 50%)
radial-gradient(circle at 80% 20%, rgba(139, 92, 246, 0.05) 0%, transparent 50%)
radial-gradient(circle at 40% 40%, rgba(59, 130, 246, 0.03) 0%, transparent 40%)
```

### 2.6 渐变系统
```css
/* 主渐变（标题文字用） */
--gradient-primary: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);

/* 状态渐变（进度条用） */
--gradient-success: linear-gradient(135deg, #10b981 0%, #34d399 100%);
--gradient-warning: linear-gradient(135deg, #f59e0b 0%, #fbbf24 100%);
--gradient-destructive: linear-gradient(135deg, #ef4444 0%, #f87171 100%);
```

---

## 3. 字体系统

### 3.1 字体选择
- **Sans (正文/标题)**: `app/style.css` 的 `--font-sans` = `"Inter", system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, "PingFang SC", "Microsoft YaHei", sans-serif`；`prototype/assets/styles.css` 以 `system-ui` 开头（原型与应用语义变量对齐，字体栈细节以 `app/style.css` 为准）
- **Mono (数据/代码)**: `--font-mono` = `"JetBrains Mono", ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, "Liberation Mono", monospace`
- **Serif (展示性文字)**: `--font-serif` = `Georgia, "Times New Roman", "Songti SC", "SimSun", "Noto Serif CJK SC", serif`
- **Fallback**: `system-ui, -apple-system, sans-serif`

> 字体策略为 CSS 变量系统字体栈，**不依赖 `next/font/google` 构建期拉取**，规避无外网环境的构建失败，原型可完全离线打开预览。

### 3.2 字号层级 (Tailwind 体系)
| 级别 | Tailwind class | 像素 | 用途 |
|------|---------------|------|------|
| 极小 | `text-[10px]` | 10px | 微标签、徽标数字 |
| 小 | `text-xs` | 12px | 辅助说明、CardDescription |
| 正文 | `text-sm` | 14px | 按钮文字、列表项（主文字大小） |
| 基础 | `text-base` | 16px | Input 文字 |
| 小标题 | `text-lg` | 18px | 空状态标题 |
| 标题 | `text-xl` | 20px | Provider Group 标题 |
| 大标题 | `text-2xl` | 24px | Section 标题、Latency 数字 |
| Hero | `text-4xl ~ text-6xl` | 36~60px | 首页主标题（响应式缩放） |

### 3.3 字重
| Tailwind class | 字重 | 用途 |
|---------------|------|------|
| `font-normal` | 400 | 正文、辅助文字 |
| `font-medium` | 500 | 按钮、小标题 |
| `font-semibold` | 600 | CardTitle、section 标题 |
| `font-bold` | 700 | 大数值、品牌标题、Hero |

### 3.4 排版规则
- 标题使用负字距 (`tracking-tight`)
- 数据使用等宽字体 (`font-mono`)，确保数字对齐
- 所有数值使用 `tabular-nums`（等宽数字对齐）
- 中文内容行高增加至 1.75
- **大号字体（≥`text-2xl`，即 24px 及以上）的展示性标题/文字使用衬线字体（`.font-display`，变量 `--font-serif`）**，涵盖 Hero 主标题、页面主标题、Section 大标题、错误页标题等
- 例外：纯数据数值（如 StatCard 的延迟/可用性、ApiStatusCard 的重试次数）因依赖等宽对齐，仍使用 `font-mono`，不套用衬线

---

## 4. 间距系统

### 4.1 间距规格 (基于 Tailwind 默认 4px 单位)
| 规格 | Tailwind class | 像素值 | 用途 |
|------|---------------|--------|------|
| 微间距 | `gap-1.5` / `gap-2` | 6px / 8px | 图标与文字组合、行内元素 |
| 小间距 | `gap-3` / `p-3` | 12px | Input 内边距、紧凑信息块 |
| 默认间距 | `gap-4` / `gap-5` / `p-4` / `p-6` | 16px / 20px | 标准 Card 内边距 (p-6)、网格 gap-5 |
| 中等 | `gap-6` / `space-y-6` | 24px | Footer 三栏、Section 内部分组 |
| 大间距 | `gap-12` / `py-12` | 48px | Provider Group 间距、垂直 Section 分隔 |
| 页面边距 | `px-6 md:px-10 lg:px-16` | 24/40/64px | DashboardHeader 及主体横向边距 |

### 4.2 布局网格
- **桌面端 (xl)**: 4 列 API 卡片网格，列间距 20px (gap-5)
- **小桌面 (lg)**: 3 列 API 卡片网格
- **平板 (sm~lg)**: 2 列 API 卡片网格
- **移动端 (< sm)**: 1 列 API 卡片网格
- **最大容器宽度**: `max-w-[1400px]` (1400px，与 `ui.md` 及 `app` 代码一致)
- **Footer 网格**: 1 列 (移动端) / 3 列 (md+)

---

## 5. 圆角系统

| 尺寸 | Tailwind class | 像素 | 用途 |
|------|---------------|------|------|
| 小 | `rounded-md` | 6px | Button、Input、Badge（shadcn 默认） |
| 中 | `rounded-lg` | 8px | Alert 容器、信息卡片 |
| 标准 | `rounded-xl` | 12px | Card 主容器（shadcn 默认） |
| 大 | `rounded-2xl` | 16px | 品牌标识方块、分组图标容器 |
| 满圆 | `rounded-full` | 50% | Avatar、状态圆点、徽标胶囊 |

### 深色主题边框/阴影层级（按视觉深度从浅到深）
| 层级 | 边框 | 阴影 | 典型组件 |
|------|------|------|---------|
| L0 背景 | — | — | body（纯背景 + 径向渐变） |
| L1 次要容器 | `border-border/30` | — | DashboardHeader border-b、Badge 默认 |
| L2 标准卡片 | `border-border/50` | `shadow` | 默认 Card、ApiCard |
| L3 强调卡片 | `border-primary/30`（hover） | `shadow-lg shadow-primary/20` | 主按钮、悬停 ApiCard |
| L4 警告卡片 | `border-destructive/40` | `shadow-[0_0_12px_rgba(239,68,68,0.6)]` | 离线 ApiCard、错误 Alert |
| L5 弹出层 | `border-border` | `boxShadow 0 8px 24px rgba(0,0,0,0.12)` | Dialog、Tooltip |

---

## 6. 图标系统

### 6.1 图标库
- **Lucide React**: 主要图标库
- **默认尺寸**: `size-4` (16px) — 按钮内图标
- **强调尺寸**: `size-5` (20px) — 卡片标题图标
- **大尺寸**: `size-8` (32px) — 头像、空状态图标

### 6.2 图标规范
- 按钮内图标使用 `size-4`
- 装饰性图标必须添加 `aria-hidden="true"`
- 功能图标按钮必须添加 `aria-label`

---

## 7. 动效系统

### 7.1 核心动画 keyframes
定义于 [app/style.css](../app/style.css) 与 [prototype/assets/styles.css](../prototype/assets/styles.css)：

| 动画名 | 用途 | 时长 |
|-------|------|------|
| `fade-in-up`（`animate-fade-in-up`） | 卡片入场（淡入上移 30px） | 400-800ms cubic-bezier(0.25,0.1,0.25,1) |
| `slide-in-right` / `scale-in-gentle` | 卡片右侧滑入 / 轻缩放入场 | 400-600ms |
| `shimmer`（`shimmer-overlay` / `skeleton-shimmer`） | ProgressBar 扫光 / 骨架屏占位 | 1.2-2s infinite |
| `pulse-gentle` / `glow-pulse` | 状态点呼吸 / 强调元素光晕 | 2s ease-in-out infinite |
| `float` / `gradient-shift` | 装饰浮动 / 渐变位移 | 3-6s infinite |
| `pulse`（Tailwind 原生） | 骨架屏与状态点脉冲 | 2s ease-in-out infinite |

> `app/style.css` 实际 keyframes 为 `pulse-gentle` / `fade-in-up` / `slide-in-right` / `scale-in-gentle` / `shimmer` / `float` / `glow-pulse` / `gradient-shift`；`spin-once` 与 `status-dot-*` 仅存在于原型 `prototype/assets/styles.css`，React 组件未使用。

### 7.2 交错动画 (Stagger)
循环渲染的卡片根据 index 设置 `animationDelay`，步长约 0.08s，营造节奏入场感。

### 7.3 过渡动画
| 场景 | 时长 | 缓动 |
|------|------|------|
| 颜色过渡（按钮 hover） | 150-300ms | `ease-out` |
| 卡片位移 hover（card-hover-lift） | 400ms | `cubic-bezier(0.23, 1, 0.32, 1)` |
| 入场动画 | 400-800ms | `cubic-bezier(0.25, 0.1, 0.25, 1)` |
| 图表入场动画 | 1200ms | Recharts `animationDuration={1200}`（区域填充淡入、Tooltip 跟随、hover 高亮） |

### 7.4 动画使用原则
1. **克制入场**：入场动画仅首次渲染触发，滚动/筛选不重复触发
2. **语义驱动**：脉冲动画仅用于"需要关注"的状态（离线/告警），健康状态静默
3. **硬件加速**：仅使用 `transform` / `opacity` 触发 GPU 合成
4. **可配置**：支持 `prefers-reduced-motion`（通过 `motion-safe:` 前缀）

---

## 8. 响应式断点

| Breakpoint | Tailwind 前缀 | 屏幕宽度 | 典型布局变化 |
|------------|--------------|---------|-------------|
| sm | `sm:` | ≥ 640px | 手机横屏，2 列网格 |
| md | `md:` | ≥ 768px | 平板，Footer 变 3 列 |
| lg | `lg:` | ≥ 1024px | 小桌面，3 列网格，显示地理位置胶囊 |
| xl | `xl:` | ≥ 1280px | 标准桌面，4 列网格，最大容器宽度 |

---

## 9. 组件规范（摘要）

详细组件规范见 [ui.md](ui.md)。

### 9.1 按钮 (Button)
- **Primary**: `bg-primary text-primary-foreground shadow-lg shadow-primary/20 hover:bg-primary/90`
- **尺寸**: `sm` (h-8) / `default` (h-9) / `lg` (h-10) / `icon` (size-9)
- **圆角**: `rounded-md`
- **焦点**: `focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2`

### 9.2 卡片 (Card)
- **背景**: `bg-card/50 backdrop-blur-sm`
- **边框**: `border border-border/30`
- **圆角**: `rounded-xl`
- **业务扩展**: ApiStatusGrid 使用 `card-hover-lift` 自定义悬停效果（上移 8px + 紫色光晕）

### 9.3 状态点 (Status Dot)
- **在线**: `bg-emerald-500` + 光晕阴影（静态，无动画减少视觉噪音）
- **降级**: `bg-amber-500` + `animate-pulse`
- **离线**: `bg-destructive` + `animate-pulse`
- **尺寸**: `size-2.5` (10px) + 12px 光晕阴影

### 9.4 进度条 (Progress Bar)
- **高度**: `h-1.5` (6px)
- **圆角**: 全圆角
- **效果**: 渐变填充 + shimmer 扫光覆盖层

---

## 10. 文件位置

- 样式变量 + 主题定义: [app/style.css](../app/style.css)
- 全局基础样式: [app/globals.css](../app/globals.css)
- shadcn 配置: [components.json](../components.json)
- UI 组件: [app/components/ui/](../app/components/ui/)
- 业务组件: [app/components/](../app/components/)
