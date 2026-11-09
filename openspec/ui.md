# UI 组件规范文档 (v2.10.35 - 浅色/深色双主题)

> 适用版本：v2.10.35 · 最近校对：2026-11-09 · 真源：`app/`、`supabase/` 与 `package.json`
>
> **代码对齐要点（下文旧表述与本表冲突时，以本表及源码为准）**：
> - 主题为浅色/深色双主题（`:root`/`.light` = `#f8fafc`，`.dark` = `#0a0a0f`），深色档使用 Indigo `#6366f1` + Violet `#8b5cf6`；不是纯 `#0f0f14` 深色。
> - 图表为静态导入的 Recharts 面积图（无 `ssr:false` / `ChartSkeleton` / `hiddenSeries` / `ReferenceLine`），图例为前 8 个 API 静态 Badge，Tooltip 使用 inline `contentStyle`。
> - `React.memo` 使用浅引用/逐项比较（明确避免全量 `JSON.stringify`）；`animationDuration={1200}`；`motion-safe:` 已适配 `prefers-reduced-motion`。
> - `GeoOptInDialog` 为自定义 `role="dialog"` 弹层，授权后调用 `enableGeoLocation()` 并刷新；主题切换按钮在 `DashboardHeader`，`ThemeProvider` 仅透传 props。
> - `StructuredData` 由 `app/layout.tsx` 注入；`app/page.tsx` 仅用 `Suspense` 包裹 `DashboardClient`。16 语言已全部落地；末尾版本历史止于 v2.7.0，属历史记录。

## 1. 设计原则

### 1.1 核心原则

参考现代仪表盘设计风格，强调数据可视化的清晰度和沉浸感：

| 原则 | 说明 |
|-----|------|
| **深色沉浸** | 浅色 `#f8fafc` / 深色 `#0a0a0f` 双主题，靛蓝与紫罗兰渐变作为氛围光 |
| **数据优先** | 图表、数字、状态指示器为核心视觉元素，留白辅助层级区分 |
| **清晰层级** | 通过边框、阴影、颜色透明度建立视觉层级关系 |
| **状态指示** | 所有动态数据必须有明确的在线/降级/离线三态视觉标识 |
| **一致性** | 所有组件使用统一的圆角 (rounded-xl)、间距、色彩与动画节奏 |
| **响应式** | 适配桌面、平板、移动端设备（1/2/3/4 列布局切换） |
| **可访问性** | 支持键盘导航和屏幕阅读器，按钮带 aria-label |
| **性能优先** | 使用 React.memo 等方式优化渲染性能，图表数据做 slice 优化 |
| **国际化** | 支持多语言切换（useI18n hook） |

### 1.2 设计系统

- **样式框架**: Tailwind CSS 4.1.11（`@theme` 块定义 CSS 变量桥接）
- **组件库**: shadcn/ui（基于 Radix UI primitives）
- **图标库**: Lucide React
- **图表库**: Recharts 3.8.0 面积图（`app/components/LatencyHistoryChart.tsx`），静态导入（无独立图表骨架屏）
- **颜色系统**: 深色紫色/靛蓝主题，CSS 变量驱动（详见 [design-system.md](design-system.md) 和 §5.1）
- **动画**: 自定义 keyframes（fade-in-up / spin-once / shimmer 等），遵循 `prefers-reduced-motion` 规范

## 2. 组件 API 规范

### 2.1 DashboardHeader

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
  geo: GeoLocation | null;
  isGeoLoading: boolean;
  refreshGeo: () => void;
  login: () => Promise<void>;
  logout: () => Promise<void>;
  resolveAlert: (id: string) => Promise<void>;
}
```

**设计特点**：
- 固定顶部 (sticky top-0)、80% 不透明背景、backdrop-blur-xl
- 左侧品牌标识：size-10 圆角方块 + primary/accent 渐变背景
- 右侧操作区：Bell 告警铃（含数字徽标 + critical 状态脉冲）、主题切换、地理位置胶囊、登录按钮
- 登录按钮带有 `shadow-lg shadow-primary/20` 光晕效果

### 2.2 ApiStatusGrid

**功能**：以网格形式展示所有 API 状态卡片（扁平网格，不按供应商分组）

**Props**：
```typescript
interface ApiStatusGridProps {
  statuses: ApiStatus[];
}
```

**内部结构**：
```
ApiStatusGrid
└── ApiCard[] (API 状态卡片扁平网格，交错入场动画，provider 透传至卡片)
```

**设计特点**：
- 响应式列数：sm:1 / sm+:2 / lg:3 / xl:4
- 每张卡片使用 `card-hover-lift`：悬停向上位移 8px 并放大 1.02，叠加 primary 紫色光晕
- StatusDot 状态圆点：带 `shadow-[0_0_12px_*]` 光晕，非在线态使用 `animate-pulse`
- 延迟进度条（ProgressBar）：渐变填充 + shimmer 扫光动画
- 离线卡片左侧添加 1px 渐变红色竖条（from-destructive via destructive/70）
- 入场动画：偶数 `animate-fade-in-up`，奇数 `animate-slide-in-right`，delay 交错

### 2.3 LatencyHistoryChart

**功能**：展示历史延迟趋势（基于 Recharts 面积图，静态导入（无独立图表骨架屏））

**Props**（当前 React 实现）：
```typescript
interface LatencyHistoryChartProps {
  chartData: ChartDataPoint[];
  statuses: ApiStatus[];
  getApiColor: (id: string) => string;
}
```

**设计特点**（与 `prototype/assets/app.js` 的 `renderMultiLineChart` 对齐）：
- 容器 `rounded-xl border bg-card`（shadcn Card）
- 容器 `rounded-xl border bg-card`（shadcn Card）
- 固定响应式高度：`h-[320px] md:h-[420px]`（Recharts `ResponsiveContainer`，自适应宽度）
- 网格线透明度 0.06，仅水平方向
- 每条 API 曲线：独立纯色 + 8% 不透明度区域填充（area fill）
- 当前未实现阈值线（无 ReferenceLine）
- 可交互：前 8 个 API 静态图例 Badge（不可点击切换）、hover 显示跟随鼠标的 tooltip（多系列数值 + 垂直扫描线）
- Tooltip 样式：`bg-popover border-border rounded-xl`，使用 CSS 变量，带 `inline contentStyle` 过渡动画
- 时间范围 1H / 6H / 24H 切换（与 `app/components/DashboardClient.tsx` 的 `TIME_RANGES` 一致），曲线形态各异（小时内密集波动 / 6 小时阶段性 / 24 小时昼夜波动），末点锚定当前实时延迟
- 历史数据上限：内存保留最近 100 条延迟记录（`app/store/api.ts` 的 `addHistoryEntry` 以 `.slice(-100)` 截断）

### 2.4 AlertsDropdown

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

**告警严重程度样式**：
| 严重程度 | 颜色 |
|---------|------|
| `critical` / `high` | text-destructive (#ef4444) |
| `medium` | text-amber-500 |
| `low` | text-blue-400 |

**空状态**：无告警时显示 `CheckCircle2` 图标 + 文字提示

### 2.5 ApiConfig

**功能**：允许用户自定义 API 检查配置（添加/删除/保存/重置）

**Props**：无

**设计特点**：
- 编辑态/查看态双模式切换
- 配置通过 localStorage 持久化
- 新 API 表单使用 dashed border 2 视觉区分
- 三个字段：name / provider / url
- 底部提供 reset 回退到默认（APIS_TO_CHECK 常量）

### 2.6 DashboardFooter

**功能**：展示项目技术栈和数据来源说明

**三个模块**：
1. Global Coverage（全球 AI 供应商监控）
2. UI 技术栈（Next.js + Tailwind CSS + shadcn/ui + Recharts 图表）
3. Data Integrity（Supabase PostgreSQL 持久化）

### 2.7 StatCard

**功能**：统计卡片组件，展示在线/降级/离线/平均延迟四项指标

**Props**：
```typescript
interface StatCardProps {
  icon: React.ReactNode;
  label: string;
  value: string | number;
  iconBgColor: string;
  iconTextColor: string;
  valueColor: string;
  hoverBorderColor: string;
  hoverShadowColor: string;
}
```

**设计特点**：
- 圆角卡片容器，半透明背景 + backdrop-blur-sm
- 左侧图标容器（w-8 h-8 rounded-lg）+ 文字标签
- 下方大数值（text-2xl font-bold），悬停时 scale-110
- 悬停时边框颜色变化 + 阴影增强

### 2.8 StatusDot

**功能**：状态圆点组件（在线/降级/离线三色 + 光晕效果）

**Props**：
```typescript
interface StatusDotProps {
  status: 'online' | 'offline' | 'degraded';
}
```

**设计特点**：
- size-2.5 rounded-full
- online → bg-emerald-500（静态）
- degraded → bg-amber-500 + animate-pulse
- offline → bg-destructive + animate-pulse
- 带 `getStatusPulseColor` 光晕阴影
- React.memo 优化性能

### 2.9 ProgressBar

**功能**：进度条组件（渐变填充 + shimmer 扫光动画）

**Props**：
```typescript
interface ProgressBarProps {
  value: number;  // 0-100
  variant: 'success' | 'warning' | 'danger';
  showLabel?: boolean;
}
```

**设计特点**：
- 容器 h-1.5 rounded-full bg-muted
- 填充条渐变 + transition-[width] duration-1000 ease-out
- success: from-emerald-500 to-emerald-400
- warning: from-amber-500 to-amber-400
- danger: from-red-500 to-red-400
- shimmer 扫光覆盖层（白色 0.3 opacity，2s infinite）
- React.memo 优化性能

### 2.10 DashboardClient

**功能**：仪表盘客户端主组件（'use client'），整合所有子组件和数据 hooks

**Props**：无

**内部结构**：
```
DashboardClient
├── DashboardHeader
├── GeoOptInDialog
├── AlertsDropdown (Dialog)
├── Hero Section
│   ├── 标题 + 描述
│   └── 4 × StatCard (online/degraded/offline/avgLatency)
├── Alerts Banner (条件显示)
├── API Status Grid Section
│   ├── Section Header (标题 + 描述 + 刷新按钮)
│   ├── ApiConfig
│   └── ApiStatusGrid
├── Latency History Chart Section
│   ├── Section Header (标题 + 描述)
│   └── LatencyHistoryChart
└── DashboardFooter
```

### 2.11 GeoOptInDialog

**功能**：地理位置授权对话框，用户同意后获取地理位置信息

**Props**：无

**设计特点**：
- 自定义 `role="dialog"` 弹层
- 用户同意后调用 `useGeoStore` 的 fetchGeoLocation
- localStorage 持久化用户选择（geoOptIn）

### 2.12 StructuredData

**功能**：Schema.org 结构化数据组件（SEO），服务端渲染

**Props**：无

**设计特点**：
- 使用 `<script type="application/ld+json">`
- 包含监控服务的结构化数据（名称、描述、URL、运营范围）
- 提升搜索引擎优化效果

### 2.13 ChartSkeleton（历史组件，当前无引用）

**功能**：图表骨架屏加载组件，数据加载时显示占位动画

**Props**：无

**设计特点**：
- 使用 `animate-pulse` 脉冲动画
- 模拟图表区域的骨架形状
- 提升感知加载速度

### 2.14 ThemeProvider

**功能**：管理浅色/深色双主题，基于 next-themes（`attribute="class"`，`defaultTheme="system"`，`enableSystem`），DashboardHeader 顶栏提供 Sun/Moon 切换按钮（ThemeProvider 仅透传 props），主题偏好持久化到 localStorage

**Props**：继承 `NextThemesProvider` 的所有 props

## 3. 组件库规范

### 3.1 基础组件（shadcn/ui）

所有基础组件均遵循相同规范：
- 统一 `cn()` 辅助函数（来自 `@/lib/utils`）拼接 class
- 支持 `cva` (class-variance-authority) 管理 variant
- 支持 `forwardRef`，符合 Radix 无障碍标准

#### Button

**变体**：`default` / `destructive` / `outline` / `secondary` / `ghost` / `link`

**尺寸**：`default` (h-9) / `sm` (h-8) / `lg` (h-10) / `icon` (size-9)

**默认样式**：
```
inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md
text-sm font-medium transition-colors
focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2
disabled:pointer-events-none disabled:opacity-50
[&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0
```

**default variant**: `bg-primary text-primary-foreground shadow hover:bg-primary/90`

#### Alert

**变体**：`default` / `destructive` / `warning` / `success`

**默认样式**：`relative w-full rounded-lg border px-4 py-3 text-sm`
- destructive: `border-destructive/50 text-destructive`
- warning: `border-amber-500/50 text-amber-700 dark:text-amber-400`
- success: `border-emerald-500/50 text-emerald-700 dark:text-emerald-400`

#### Badge

**变体**：`default` / `secondary` / `destructive` / `outline`

**默认样式**：
```
inline-flex items-center justify-center rounded-md border px-2.5 py-0.5
text-xs font-semibold transition-colors
focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2
```

#### Avatar

**结构**：`Avatar` (root) + `AvatarImage` (可选) + `AvatarFallback` (首字母)

**样式**：
- root: `relative flex h-10 w-10 shrink-0 overflow-hidden rounded-full`
- fallback: `bg-muted`，首字母大写
- DashboardHeader 中使用 size-8，添加 `border-border/30`

#### Card

**子组件**：`Card` / `CardHeader` / `CardTitle` / `CardDescription` / `CardContent` / `CardFooter`

**样式**：
- root: `rounded-xl border bg-card text-card-foreground shadow`
- header: `flex flex-col space-y-1.5 p-6`
- title: `font-semibold leading-none tracking-tight`
- content: `p-6 pt-0`
- footer: `flex items-center p-6 pt-0`

**业务层扩展**：ApiStatusGrid 使用 `card-hover-lift` 自定义 hover 效果

#### Input

**默认样式**：
```
flex h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-sm shadow-sm
transition-colors
placeholder:text-muted-foreground
focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring
disabled:cursor-not-allowed disabled:opacity-50
```

#### Switch

基于 `@radix-ui/react-switch`，受控开关。

**默认样式**：
```
peer inline-flex h-5 w-9 shrink-0 cursor-pointer items-center rounded-full border-2 border-transparent
transition-colors focus-visible:ring-2 focus-visible:ring-ring
data-[state=checked]:bg-primary data-[state=unchecked]:bg-input
```
- thumb: `h-4 w-4 rounded-full bg-background shadow-lg data-[state=checked]:translate-x-4`

#### Select

基于 `@radix-ui/react-select`，下拉选择器。

**组成**：`Select` / `SelectTrigger` / `SelectContent` / `SelectItem` / `SelectValue`

**Trigger 样式**：`flex h-9 w-full items-center justify-between rounded-md border border-input bg-transparent px-3 text-sm shadow-sm focus:ring-1 focus:ring-ring`

**Content 样式**：`z-50 min-w-[8rem] overflow-hidden rounded-md border bg-popover text-popover-foreground shadow-md data-[state=open]:animate-in data-[state=open]:fade-in-0 data-[state=open]:zoom-in-95`

**Item 样式**：`relative flex w-full cursor-default select-none items-center rounded-sm py-1.5 pl-2 pr-8 text-sm outline-none focus:bg-accent focus:text-accent-foreground`

#### Tabs

基于 `@radix-ui/react-tabs`，标签页切换。

**组成**：`Tabs` / `TabsList` / `TabsTrigger` / `TabsContent`

**List 样式**：`inline-flex h-9 items-center justify-center rounded-lg bg-muted p-1 text-muted-foreground`

**Trigger 样式**：`inline-flex items-center justify-center rounded-md px-3 py-1 text-sm font-medium transition-all data-[state=active]:bg-background data-[state=active]:text-foreground data-[state=active]:shadow`

### 3.2 复合组件（业务封装）

#### ApiStatusGrid

| 特征 | 规范 |
|------|------|
| 布局 | 扁平 `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4`，不按 provider 分组 |
| 卡片容器 | `Card` + `bg-card/80 backdrop-blur-sm` + 悬停 `border-primary/30` + `card-hover-lift` |
| 状态色带 | 离线态左侧红色竖条 `from-destructive via destructive/70 to-transparent` |
| 状态圆点 | size-2.5 rounded-full + `shadow-[0_0_12px_*]` 光晕，非 online 使用 `animate-pulse` |
| 进度条 | h-1.5 rounded-full，渐变填充 + shimmer 扫光覆盖层 |
| 空状态 | `border-dashed border-border/50 bg-secondary/30` + Server 图标脉冲 |

#### LatencyHistoryChart

| 特征 | 规范 |
|------|------|
| 容器 | `Card > CardContent p-4`；Recharts `ResponsiveContainer` + `AreaChart`，高度 `h-[320px] md:h-[420px]` |
| 网格 | `strokeDasharray="6 6"`，`opacity=0.06`，vertical=false |
| X 轴 | fontSize 11，opacity 0.4，dy=12，>20 点时 `preserveStartEnd` |
| Y 轴 | fontSize 11，opacity 0.4，insideLeft "ms" 标签 |
| Tooltip | `bg-card border-border rounded-xl`，boxShadow `0 8px 24px rgba(0,0,0,0.12)` |
| Area 曲线 | `strokeWidth=2.5`，linearGradient 5%→95% (0.15→0) 透明渐变填充 |
| activeDot | r=5, strokeWidth=0 |
| 动画 | animationDuration=1200 |
| 性能 | `React.memo` 包裹，JSON.stringify 做 props 比较 |

#### AlertsDropdown

| 特征 | 规范 |
|------|------|
| 容器 | shadcn `Dialog` + `max-w-lg` |
| 每条告警 | `Card > CardContent`，flex items-start justify-between gap-4 |
| 严重程度图标 | `AlertTriangle` size-5，使用 severityTone 着色 |
| 信息 | apiName (font-medium) + message + error (text-destructive) + latency |
| 操作 | `Button size="sm" variant="outline"` + "Resolve" |
| 最大高度 | `max-h-[60vh] overflow-y-auto` |

### 3.3 业务组件

#### DashboardHeader

| 区域 | 规范 |
|------|------|
| 容器 | `sticky top-0 z-50 border-b border-border/30 bg-background/80 backdrop-blur-xl h-16` |
| 内部宽度 | `max-w-[1400px] px-6 md:px-10 lg:px-16` |
| 左侧品牌 | size-10 rounded-xl + primary/accent 渐变，悬停 group-hover:scale-105 + blur-xl 光晕 |
| 右侧按钮 | variant="ghost" size="icon"，均带 aria-label |
| 告警徽标 | absolute -right-0.5 -top-0.5 size-5 rounded-full text-[10px] font-bold；critical→destructive+animate-pulse，其他→primary |
| 地理位置胶囊 | lg:flex only，`rounded-full bg-secondary/50 px-3 py-1.5 text-xs text-muted-foreground border border-border/20` |
| 登录按钮 | `bg-primary hover:bg-primary/90 shadow-lg shadow-primary/20 hover:shadow-primary/30 hover:scale-105` |
| 头像 | size-8 border-border/30，悬停 `border-primary/50 scale-105` |

#### DashboardFooter

| 区域 | 规范 |
|------|------|
| 顶部 | `pt-8` + `Separator` |
| 布局 | `grid-cols-1 md:grid-cols-3 gap-6` |
| 标题行 | flex items-center gap-2 text-sm font-medium + primary 色 icon |
| 描述行 | text-xs text-muted-foreground |

#### ApiConfig

| 区域 | 规范 |
|------|------|
| 容器 | 标准 Card |
| Header | flex-row items-start justify-between；title + description（显示 API 数量）；右侧 edit/save/cancel 按钮组 |
| 列表项 | `flex items-center justify-between gap-4 rounded-lg border p-3`；内容行 text-sm + provider text-xs muted + url text-xs font-mono |
| 编辑态 | 每项右侧显示 `Button variant=ghost size=icon` + Trash2（text-destructive） |
| 新 API 表单 | `rounded-lg border-2 border-dashed p-4`；grid sm:grid-cols-3 gap-3；每个子项 flex flex-col gap-1.5 + Label + Input |
| Footer | 编辑态时显示 ghost reset 按钮 |

### 3.4 组件使用规则

1. **统一入口**：所有 UI 组件从 `@/components/ui/*` 导入，业务组件从 `@/components/*` 导入
2. **图标一致性**：统一使用 Lucide React（size-4 默认 / size-5 强调 / size-8 头像）
3. **语义化 color**：使用 `text-primary` / `bg-card` / `border-border` 等语义化 token，禁止硬编码色值
4. **响应式优先**：移动端列数 1 → 2 → 3 → 4 渐进扩展，lg 以下隐藏次要信息（地理位置）
5. **动画克制**：入场动画仅在首屏渲染时触发；悬停动作用 cubic-bezier 缓出；禁用过度抖动动画
6. **无障碍**：所有图标按钮必须带 `aria-label`，焦点态使用 `focus-visible:ring-2 ring-ring`
7. **国际化**：所有用户可见文本必须通过 `useI18n()` 的 `t()` 函数获取
8. **性能**：列表/图表组件用 React.memo；数据量大时做 slice 优化；避免频繁重渲染

## 4. 交互标准

### 4.1 交互模式库

| 交互模式 | 典型组件 | 行为定义 |
|---------|---------|---------|
| **状态指示** | StatusDot、Badge、进度条 | 三色语义（online=emerald / degraded=amber / offline=destructive）；非在线态使用脉冲动画吸引注意 |
| **悬停反馈** | Button、Card、Avatar | 轻度位移（translateY -6~-8px）+ 阴影增强 + 颜色微变（primary/90） |
| **选中/展开** | Dialog（AlertsDropdown） | 背景遮罩 + 内容缩放淡入；ESC 键关闭；点击外部关闭 |
| **切换开关** | 主题切换按钮 | Sun/Moon 图标切换；悬停显示对应色调预览 |
| **表单输入** | ApiConfig Input | focus-visible:ring-1 ring-ring；placeholder text-muted-foreground；disabled:opacity-50 |
| **工具提示** | 图表 hover Tooltip（Recharts） | 内容卡片样式，阴影 boxShadow 0 8px 24px rgba(0,0,0,0.12)，跟手定位 + 过渡动画 |

### 4.2 状态指示器规范

#### 视觉层级（从强到弱）
1. **破坏性 (destructive / #ef4444)**：用于严重告警、离线、错误状态；可使用 `animate-pulse` 持续吸引注意
2. **警告 (warning / #f59e0b)**：用于降级状态、高延迟、重试；可使用脉冲动画
3. **成功 (success / #22c55e)**：用于在线状态、可用率、正常延迟；静态（无动画以避免视觉噪音）

#### 状态圆点规范
```
size-2.5 rounded-full + 12px 光晕 shadow
online → bg-emerald-500 + shadow-[0_0_12px_rgba(34,197,94,0.6)] （静态）
degraded → bg-amber-500 + shadow-[0_0_12px_rgba(245,158,11,0.6)] （animate-pulse）
offline → bg-destructive + shadow-[0_0_12px_rgba(239,68,68,0.6)] （animate-pulse）
```

#### 进度条规范
- 容器：`h-1.5 rounded-full bg-muted`
- 填充：`bg-gradient-to-r` + shimmer 扫光覆盖层（白色 0.3 opacity + animation-shimmer 2s infinite）
- 颜色变体：success (emerald 500→400) / warning (amber 500→400) / danger (red 500→400)

### 4.3 悬停效果规范

#### Button hover
```
bg-primary → bg-primary/90
shadow-lg shadow-primary/20 → hover:shadow-primary/30 + hover:scale-105（登录按钮强化版）
```

#### Card hover（card-hover-lift class）
```
transition-all duration-400 cubic-bezier(0.23, 1, 0.32, 1)
hover → translateY(-8px) scale(1.02)
       + boxShadow 0 25px 50px -12px rgba(0,0,0,0.4)
       + boxShadow 0 0 30px rgba(99,102,241,0.15)
       + border-primary/30
       + 内部渐变背景 from-primary/5 via-transparent to-accent/5 (淡入 opacity)
```

#### Avatar hover
```
border-border/30 → border-primary/50
scale-1 → scale-105
duration-300
```

#### Brand logo hover
```
group-hover:scale-105
+ 外层 -inset-1 rounded-xl bg-primary/10 blur-xl opacity-0 → opacity-100
```

### 4.4 错误处理规范

#### 视觉策略
- **内联错误**：在 ApiConfig 等表单中，错误字段使用 Input 标准样式（不做红色边框，保持简洁，由业务逻辑 block 提交）
- **告警 Banner**：AlertsDropdown Dialog 中使用 text-destructive 标注 error 字段
- **卡片级错误**：ApiStatusGrid 离线卡片左侧 1px 红色竖条 + `border-destructive/40 bg-destructive/5`
- **字段级错误**：errorRate > 1% → 容器 `border-amber-500/30 bg-amber-500/5`，数字 `text-amber-500`，进度条 variant="warning"

#### 数据降级规则
| 字段 | 缺省值 | 展示策略 |
|------|--------|---------|
| latency | 0 | 显示 "timeout"，text-destructive；进度条 100% danger |
| errorRate | undefined | 隐藏该指标 |
| availability | undefined | 隐藏该指标 |
| retries | 2 | 在离线卡片中显示 "Nx" 重试标记 |
| lastChecked | Date.now() | 显示为 `toLocaleTimeString` |

### 4.5 空状态设计规范

#### ApiStatusGrid 空状态
```
Card border-dashed border-border/50 bg-secondary/30
> CardContent flex-col items-center gap-4 py-16 text-center
  - size-16 rounded-full bg-muted animate-pulse 容器
  - Server size-8 text-muted-foreground 图标
  - h3 text-xl font-semibold 标题
  - p text-muted-foreground 辅助文字
```

#### AlertsDropdown 空状态
```
flex flex-col items-center gap-2 py-8 text-center
- CheckCircle2 size-8 text-emerald-500
- p text-sm text-muted-foreground "无活跃告警"
```

#### 设计原则
1. **不使用空空白**：任何空数据区域必须有视觉占位，避免界面坍塌
2. **语义化图标**：空数据使用对应语义图标（Server=无配置 / CheckCircle2=健康）
3. **操作引导**：空状态文字提示下一步操作（例如 "添加 API 开始监控"）
4. **动画克制**：使用 `animate-pulse` 做柔和的"等待/加载"暗示，不使用抖动动画

## 5. 样式规范

### 5.1 颜色系统（浅色 / 深色双主题）

#### CSS 变量（定义于 `app/style.css` 的 `:root` / `.light` 与 `.dark`）

| Token | 浅色（`:root` / `.light`） | 深色（`.dark`） | 用途 |
|-------|---------------------------|-----------------|------|
| `--background` | `#f8fafc` | `#0a0a0f` | 主页面背景 |
| `--foreground` | `#0f172a` | `#f4f4f5` | 主文字颜色 |
| `--card` / `--popover` | `#ffffff` | `#13131a` | 卡片 / 弹出层背景 |
| `--card-foreground` / `--popover-foreground` | `#0f172a` | `#f4f4f5` | 卡片 / 弹出层文字 |
| `--primary` | `#6366f1` | `#818cf8` | 主色（靛蓝） |
| `--primary-foreground` | `#ffffff` | `#0a0a0f` | 主色上的文字 |
| `--secondary` / `--muted` | `#e2e8f0` | `#1e1e28` | 次要 / 弱化背景 |
| `--secondary-foreground` | `#0f172a` | `#f4f4f5` | 次要背景上的文字 |
| `--muted-foreground` | `#64748b` | `#71717a` | 辅助文字 |
| `--accent` | `#8b5cf6` | `#a78bfa` | 强调色（紫色） |
| `--accent-foreground` | `#ffffff` | `#0a0a0f` | 强调色上的文字 |
| `--destructive` | `#ef4444` | `#f87171` | 错误 / 离线状态 |
| `--destructive-foreground` | `#ffffff` | `#0a0a0f` | 错误色上的文字 |
| `--border` / `--input` | `#e2e8f0` | `#242430` | 边框 / 输入框背景 |
| `--ring` | `#6366f1` | `#818cf8` | 焦点环 |

`@theme` 中的语义色不随主题切换：`--color-success` `#22c55e`、`--color-warning` `#f59e0b`、`--color-info` `#3b82f6`、`--color-error` `#ef4444`（另有 `--color-*-10` alpha 变体）。主题由 `next-themes` 的 `defaultTheme="system"` / `enableSystem` 切换，深色档沿用 Dark Indigo 主色。

#### Tailwind 桥接（`@theme` block）

CSS 变量通过 Tailwind 4.1 的 `@theme` 语法注册为 Tailwind token，例如：
```
@theme {
  --color-background: var(--background);
  --color-primary: var(--primary);
  --color-success: #22c55e;
  /* ... 其他 token ... */
}
```
这使得 `bg-primary` / `text-muted-foreground` / `border-border` 等 Tailwind class 自动映射到 CSS 变量。

#### 背景氛围光（Radial Gradient）

body 背景使用 `var(--background)` 并叠加三层径向渐变：
```
radial-gradient(circle at 20% 80%, rgba(99, 102, 241, 0.05) 0%, transparent 50%)
radial-gradient(circle at 80% 20%, rgba(139, 92, 246, 0.05) 0%, transparent 50%)
radial-gradient(circle at 40% 40%, rgba(59, 130, 246, 0.03) 0%, transparent 40%)
```
这营造出柔和的紫色光晕氛围，不干扰数据可读性。

### 5.2 圆角系统

| 尺寸 | Tailwind class | 像素 | 用途 |
|------|---------------|------|------|
| 小 | `rounded-md` | 6px | Button、Input、Badge（shadcn 默认） |
| 中 | `rounded-lg` | 8px | Alert 容器、信息卡片 |
| 标准 | `rounded-xl` | 12px | Card 主容器（shadcn 默认） |
| 大 | `rounded-2xl` | 16px | 品牌标识方块、分组图标容器 |
| 满圆 | `rounded-full` | 50% | Avatar、状态圆点、徽标胶囊 |

**深色主题边框/阴影层级**（按视觉深度从浅到深）：

| 层级 | 边框 | 阴影 | 典型组件 |
|------|------|------|---------|
| L0 背景 | — | — | body（纯背景 + 径向渐变） |
| L1 次要容器 | `border-border/30` | — | DashboardHeader border-b、Badge 默认透明 |
| L2 标准卡片 | `border-border/50` | `shadow` | 默认 Card、ApiCard |
| L3 强调卡片 | `border-primary/30`（hover） | `shadow-lg shadow-primary/20` | 主按钮、悬停中的 ApiCard |
| L4 警告卡片 | `border-destructive/40` | `shadow-[0_0_12px_rgba(239,68,68,0.6)]` | 离线 ApiCard、错误 Alert |
| L5 弹出层 | `border-border` | `boxShadow 0 8px 24px rgba(0,0,0,0.12)` | Dialog、Tooltip |

### 5.3 间距系统

| 规格 | Tailwind class | 像素值 | 用途 |
|------|---------------|--------|------|
| 微间距 | `gap-1.5` / `gap-2` | 6px / 8px | 图标与文字组合、行内元素间距 |
| 小间距 | `gap-3` / `p-3` | 12px | Input 内边距、紧凑信息块 |
| 默认间距 | `gap-4` / `gap-5` / `p-4` / `p-6` | 16px / 20px | 标准 Card 内边距 (p-6)、卡片间网格 gap-5 |
| 中等 | `gap-6` / `space-y-6` | 24px | Footer 三栏间距、Section 内部分组 |
| 大间距 | `gap-12` / `py-12` | 48px | Provider Group 间距、垂直 Section 分隔 |
| 页面边距 | `px-6 md:px-10 lg:px-16` | 24/40/64px | DashboardHeader 及主体横向边距 |

### 5.4 字体系统

**字体族**（通过 `app/style.css` 的 `@theme` 定义 CSS 变量，使用系统字体栈，**不依赖构建时网络拉取**，保证离线/无外网环境可构建）：
| 变量 | 字体 | 用途 |
|------|------|------|
| `font-sans` | "Inter", system-ui, -apple-system, "Segoe UI", Roboto, "PingFang SC", "Microsoft YaHei", sans-serif | 正文、标题、按钮（绝大多数场景） |
| `font-mono` | "JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, Consolas, monospace | URL 显示、latency 数值、标签 |
| `font-semibold` | 600 | CardTitle、section 标题 |
| `font-bold` | 700 | 大数值、品牌标题 |

> 说明：原先通过 `next/font/google` 在构建期拉取 Inter / JetBrains Mono，已改为 CSS 变量系统字体栈，规避无外网环境的构建失败；若部署环境有外网，可恢复 `next/font/google` 以获得更精确的字重控制。

**字体大小**：
| 大小 | Tailwind | 像素 | 用途 |
|------|---------|------|------|
| 极小 | `text-[10px]` | 10px | 微标签、时间戳辅助说明 |
| 小 | `text-xs` | 12px | CardDescription、辅助信息 |
| 正文 | `text-sm` | 14px | 按钮文字、列表项、描述文字（主文字大小） |
| 基础 | `text-base` | 16px | Input 文字 |
| 小标题 | `text-lg` | 18px | 空状态标题 |
| 标题 | `text-xl` | 20px | Provider Group 标题 |
| 大标题 | `text-2xl` | 24px | Latency 大数字显示 |

**数字规范**：所有 latency/errorRate/availability 数字使用 `tabular-nums`（等宽数字对齐）

### 5.5 阴影系统

| 阴影 | 值 | 用途 |
|------|----|------|
| 默认 | Tailwind `shadow` | Card 默认 |
| 主色光晕 | `shadow-lg shadow-primary/20` | 主按钮默认 |
| 悬停主色 | `hover:shadow-primary/30` | 主按钮 hover |
| 状态圆点光晕 | `shadow-[0_0_12px_rgba(34,197,94,0.6)]` 等 | StatusDot 的三色发光阴影 |
| 卡片悬停 | `0 25px 50px -12px rgba(0,0,0,0.4) + 0 0 30px rgba(99,102,241,0.15)` | card-hover-lift |
| Tooltip / Dialog | `0 8px 24px rgba(0,0,0,0.12)` | 弹出层 box-shadow |

## 6. 动画效果规范

### 6.1 动画核心 keyframes

#### pulse-gentle（柔和脉冲）
```css
@keyframes pulse-gentle {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(1.1); }
}
```
**用途**：StatusDot ::before 外层光晕扩展（opacity 0.25 inset -6px），2s ease-in-out infinite
**典型 class**：`animate-pulse-gentle`

#### fade-in-up（淡入上移）
```css
@keyframes fade-in-up {
  from { opacity: 0; transform: translateY(30px); }
  to { opacity: 1; transform: translateY(0); }
}
```
**用途**：ApiStatusGrid 偶数位置卡片入场动画
**典型 class**：`animate-fade-in-up`，配合 `animationDelay` 交错

#### scale-in-gentle（轻柔缩放入场）
```css
@keyframes scale-in-gentle {
  from { transform: scale(0.96); opacity: 0; }
  to { transform: scale(1); opacity: 1; }
}
```
**用途**：Dialog 内容、新卡片入场，营造"浮现"效果
**典型 class**：`animate-scale-in-gentle`

#### slide-in-right（右滑淡入）
```css
@keyframes slide-in-right {
  from { transform: translateX(20px); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}
```
**用途**：ApiStatusGrid 奇数位置卡片入场动画（与 fade-in-up 交替制造节奏）
**典型 class**：`animate-slide-in-right`

#### shimmer（扫光）
```css
@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
```
**用途**：ProgressBar 覆盖层（白色 0.3 opacity 线性渐变），制造数据"流动感"
**节奏**：2s infinite

#### glow-pulse（光晕脉冲）
```css
@keyframes glow-pulse {
  0%, 100% { box-shadow: 0 0 20px rgba(99, 102, 241, 0.3); }
  50% { box-shadow: 0 0 40px rgba(99, 102, 241, 0.6); }
}
```
**用途**：主操作按钮强化版的呼吸光晕（目前在 brand logo hover 使用类似 blur-xl 实现）

### 6.2 交错动画（Stagger）

使用方式：在循环渲染的卡片上，根据 index 设置 `animationDelay: \`${index * 0.08}s\``（在 style 内联中设置）

预定义 class（如需使用 Tailwind class 方式）：
```
stagger-1 → 0.05s
stagger-2 → 0.10s
stagger-3 → 0.15s
stagger-4 → 0.20s
stagger-5 → 0.25s
stagger-6 → 0.30s
stagger-7 → 0.35s
stagger-8 → 0.40s
```

### 6.3 过渡动画（Transition）

| 场景 | 时长 | 缓动函数 | class |
|------|------|---------|-------|
| 主题切换（背景色） | 500ms | `cubic-bezier(0.4, 0, 0.2, 1)` | body transition |
| 颜色过渡（按钮 hover） | 150-300ms | `ease-out` | `transition-colors` |
| 卡片位移 hover | 350-400ms | `cubic-bezier(0.23, 1, 0.32, 1)` | card-hover-lift |
| 入场动画 | 400-800ms | `cubic-bezier(0.25, 0.1, 0.25, 1)` | fade-in-up（原型统一使用 fade-in-up 交错，含 slide-in-right） |
| 图表动画 | 实时重绘 + 0.4-0.5s 透明度过渡 | Recharts `animationDuration` | 区域填充淡入、Tooltip 跟随、hover 高亮 |
| 缩放反馈 | 200ms | `ease-out` | apple-button:active scale(0.96) |

### 6.4 动画使用原则

1. **克制入场**：入场动画仅在首次渲染触发；滚动/筛选更新时不重复触发
2. **时长限制**：单体动画 ≤ 800ms，动画总时长（交错合计）≤ 1500ms
3. **动效为语义服务**：脉冲动画仅用于"需要关注"的状态（离线/告警），健康状态保持静默以减少视觉噪音
4. **硬件加速**：使用 `transform` / `opacity` 触发 GPU 合成，避免 `width` / `top` 等布局属性动画
5. **可配置**：尊重用户 `prefers-reduced-motion`（已通过 motion-safe: 支持）

## 7. 响应式规范

### 7.1 断点定义

| 断点 | Tailwind 前缀 | 屏幕宽度 | 设备类型 |
|------|--------------|---------|---------|
| sm | `sm:` | ≥ 640px | 手机横屏 |
| md | `md:` | ≥ 768px | 平板 |
| lg | `lg:` | ≥ 1024px | 小桌面 |
| xl | `xl:` | ≥ 1280px | 标准桌面 |

### 7.2 布局适配

#### ApiStatusGrid 列数

| 断点 | 列数 | class |
|------|------|-------|
| < 640px | 1 | `grid-cols-1` |
| 640-1023px | 2 | `sm:grid-cols-2` |
| 1024-1279px | 3 | `lg:grid-cols-3` |
| ≥ 1280px | 4 | `xl:grid-cols-4` |

#### DashboardFooter 列数

| 断点 | 列数 |
|------|------|
| < 768px | 1 |
| ≥ 768px | 3 |

#### ApiConfig 新 API 表单列数

| 断点 | 列数 |
|------|------|
| < 640px | 1 |
| ≥ 640px | 3 |

#### LatencyHistoryChart 高度

图表采用 Recharts 面积图，固定响应式高度 `h-[320px] md:h-[420px]`，自适应宽度（`ResponsiveContainer`）。

### 7.3 内容可见性适配

| 元素 | 移动端 | 桌面端 (≥ lg) |
|------|--------|--------------|
| 地理位置胶囊 | 隐藏 | 显示（`lg:flex`） |
| 副标题说明 | 显示 | 显示 |
| 图表完整刻度 | 简化（preserveStartEnd） | 完整（interval=0，>20 点时同上） |

### 7.4 页面边距

| 断点 | 横向边距 | class |
|------|---------|-------|
| 默认 | 24px | `px-6` |
| md | 40px | `md:px-10` |
| lg | 64px | `lg:px-16` |

## 8. 页面布局

### 8.1 主页面结构

```
<html lang>
└── <body> (bg: var(--background) + 三层径向渐变)
    ├── ThemeProvider (next-themes)
    ├── StructuredData (SEO - Server Component)
    └── DashboardClient (Client Component)
        ├── DashboardHeader (sticky top-0 z-50)
        │   ├── Brand Logo (size-10 rounded-xl + primary/accent gradient)
        │   ├── Title + Subtitle (hidden sm:block)
        │   ├── Right Actions (Bell / Theme / Geo / User / Login)
        │   └── AlertsDropdown (Dialog + DialogContent max-w-lg)
        ├── GeoOptInDialog (地理位置授权弹窗)
        ├── <main> (mx-auto max-w-[1400px] py-8 md:py-12)
        │   ├── Hero Section
        │   │   ├── 主标题 + 副标题描述
        │   │   └── 4 × StatCard (online/degraded/offline/avgLatency)
        │   ├── Alerts Banner（条件显示：当有活跃告警时）
        │   ├── Section: API Status Grid
        │   │   ├── Section Header (标题 + 描述 + 刷新按钮)
        │   │   ├── ApiConfig (自定义 API 配置)
        │   │   └── ApiStatusGrid (扁平卡片网格，provider 透传至 ApiStatusCard)
        │   └── Section: Latency History Chart
        │       ├── Section Header (标题 + 描述)
        │       └── LatencyHistoryChart（静态导入，无独立骨架屏）
        └── DashboardFooter
            ├── Separator
            └── Grid: Global Coverage / UI Tech Stack / Data Integrity
```

### 8.2 垂直间距

- main: `py-8 md:py-12`
- Section 间: `space-y-12`
- Hero 区域内: `space-y-8`
- Header 内部: 紧凑 flex items-center gap-2

### 8.3 最大宽度

- 所有内容容器：`max-w-[1400px]`
- AlertsDropdown Dialog 内容：`max-w-lg`
- ApiConfig 新 API 表单：`sm:grid-cols-3`

### 8.4 服务端/客户端组件划分

| 组件 | 类型 | 说明 |
|-----|------|------|
| app/page.tsx | Server Component | 页面入口，组合 StructuredData + DashboardClient |
| StructuredData | Server Component | SEO 结构化数据，无交互 |
| ThemeProvider | Client Component | 'use client'，管理主题状态 |
| DashboardClient | Client Component | 'use client'，主仪表盘，所有交互逻辑 |
| 所有子组件 | Client Component | 继承父组件的 'use client' |

## 9. 版本历史

### v2.7.0
- 🚀 Next.js 14.2 App Router 架构升级
- 🎯 新增 DashboardClient 主组件（客户端入口）
- 📊 新增 StatCard 统计卡片组件（4项核心指标）
- 💠 新增 StatusDot 状态圆点组件（三色 + 光晕 + React.memo）
- 📈 新增 ProgressBar 进度条组件（渐变 + shimmer + React.memo）
- 🗺️ 新增 GeoOptInDialog 地理位置授权对话框
- 🔍 新增 StructuredData SEO 结构化数据组件
- 💀 新增 ChartSkeleton 骨架屏加载组件（历史记录；当前实现无引用）
- ⚡ 新增 StatusGrid 兼容层（向后兼容 ApiStatusGrid）
- 🌍 国际化支持（useI18n hook）；原型已实现 中文/English 2 种，已支持 16 语言
- 🏪 5 个 Zustand Store（api/auth/alerts/geo/error）
- 🔒 Supabase 后端集成（Auth + PostgreSQL + Realtime）
- 📱 响应式优化（移动端 1 列 → 桌面端 4 列）
- ♿ 无障碍增强（语义化 HTML + ARIA + 键盘导航）
- ⚛️ React.memo 性能优化（LatencyHistoryChart/StatusDot/ProgressBar）
- 🎨 Tailwind CSS 4.1 + CSS 变量主题系统

### v2.6.3
- 🎨 Dark Indigo 主题全面落地（CSS 变量驱动）
- 🎯 ApiStatusGrid 增加 card-hover-lift 悬浮效果 + StatusDot 光晕 + shimmer 进度条
- 📊 LatencyHistoryChart 增加独立线性渐变填充 + 暗色图表样式
- 🔔 AlertsDropdown 从 dropdown 重构为 Dialog 组件
- ⚙️ ApiConfig 增加编辑/保存/重置完整流程 + localStorage 持久化
- 👤 DashboardHeader 增加品牌光晕 + 头像悬停效果
- ♿ 全面补充 aria-label 无障碍属性
- 🌍 useI18n 多语言支持

### v2.6.2
- 🐛 修复构建错误
- 📦 Supabase 迁移

### v2.6.1
- ✨ 初始版本
