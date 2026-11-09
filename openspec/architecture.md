# 架构设计文档

> 适用版本：v2.10.35 · 最近校对：2026-11-09 · 真源：`app/`、`supabase/` 与 `package.json`
>
> **代码对齐要点（下文旧表述与本表冲突时，以本表及源码为准）**：
> - 依赖版本以 `package.json` 的 `^` 范围为准（Next.js ^14.2.13 / React ^18.2.0 / Recharts ^3.8.0 / date-fns ^4.1.0 / Zustand ^5.0.12），Tailwind CSS 固定 4.1.11；CI 为 Node.js 22 + pnpm 9.15.0。
> - 图表 `LatencyHistoryChart` 为静态导入的 Recharts 面积图（无 `ssr:false`、无 `ChartSkeleton` 引用）；`DashboardClient` 仅动态导入 `ApiConfig` / `AlertsDropdown`。
> - 默认后台为 `supabase/cron.sql`（pg_cron 每 5 分钟）→ Edge Function `POST /monitor`；`examples/self-host-server.ts` 仅为可选自建模式，限流为全局 100 次 / 60 秒。
> - 状态数据 = 首屏 `api_status` 查询 + 前端 5 分钟定时探测；仅 `alerts` 表使用 Realtime。手动检查无需登录。
> - 重试为固定 1000ms 延迟（非指数退避），常量名为 `MAX_CONCURRENT_REQUESTS = 5`；告警由前端 `createAlertForResult` 生成，默认 Edge Function 不创建告警。

## 1. 架构概览

LLM API Sentinel 采用**静态前端 + Supabase 后端**架构，无需自定义 API 路由。前端直接与 Supabase PostgreSQL 进行实时数据同步。

### 运行模式

- **默认部署（静态导出）**：`next.config.mjs` 启用 `output: 'export'`，构建产物输出到 `out/`，由 Vercel / EdgeOne Pages / Netlify 等静态托管提供服务。后台监控通过 Supabase Cron / Edge Functions 运行，**此模式下无需自定义服务器**。
- **可选自建模式（Express 安全服务器）**：`examples/self-host-server.ts` 以自定义服务器模式包装 Next.js，新增 Helmet 安全响应头与全局按 IP 限流（100 次 / 60 秒）。该模式与静态导出互斥，仅用于 `node examples/self-host-server.ts` 自托管场景。默认后台持久化由 Supabase Cron 调用 Edge Function monitor 完成；可选自建 Express 模式另用 setInterval 每 5 分钟巡检。

### 1.1 技术栈
| 类别 | 技术 | 版本 |
|-----|------|------|
| 前端框架 | Next.js 14.2.13 (App Router) | ^14.2.13 |
| 后端服务器（可选） | Express 5.2.1（仅 `examples/self-host-server.ts` 自建模式） | ^5.2.1 |
| 数据库 | Supabase PostgreSQL | - |
| 身份验证 | Supabase Auth (Google OAuth) | - |
| 样式 | Tailwind CSS 4.1.11 | 4.1.11 |
| 图表 | Recharts 面积图（静态导入，animationDuration=1200） | ^3.8.0 |
| 图标 | Lucide React | - |
| 状态管理 | Zustand 5.0.12 | ^5.0.12 |
| 时间处理 | date-fns 4.1.0 | ^4.1.0 |

### 1.2 系统架构图

```mermaid
graph TD
    subgraph Frontend [Next.js 静态前端]
        Dashboard[Dashboard Page]
        Components[UI Components]
        Hooks[Custom Hooks]
        Store[Zustand Store]
        Lib[Utility Functions]
    end

    subgraph Backend [Supabase]
        Auth[Supabase Auth]
        DB[(PostgreSQL)]
    end

    subgraph Background [Express Server]
        Monitor[Background Monitor]
        Concurrency[Concurrency Manager]
    end

    subgraph External [External APIs]
        US[US APIs\nOpenAI, Anthropic, Google, Meta, Mistral]
        CN[China APIs\nMoonshot, Zhipu, Baichuan, Alibaba, Tencent, Baidu, DeepSeek]
    end

    Dashboard -->|Reads| Store
    Components -->|Uses| Hooks
    Hooks -->|Updates| Store
    Store -->|首屏查询 + 前端定时探测| DB
    Auth -->|Authenticates| Dashboard
    
    Monitor -->|Writes| DB
    Monitor -->|Checks| External
    Concurrency -->|Controls| Monitor
    
    US -->|Returns Status| Monitor
    CN -->|Returns Status| Monitor
    
    DB -->|alerts 表 Realtime| Dashboard
```

## 2. 前端架构

### 2.1 组件层级结构

```mermaid
graph TD
    RootLayout[RootLayout] --> ThemeProvider[ThemeProvider]
    ThemeProvider --> StructuredData[StructuredData SEO]
    ThemeProvider --> DashboardClient[DashboardClient]
    
    DashboardClient --> DashboardHeader[DashboardHeader]
    DashboardClient --> GeoOptInDialog[GeoOptInDialog]
    DashboardClient --> AlertsDropdown[AlertsDropdown Dialog]
    DashboardClient --> StatCardSection[统计卡片区域]
    DashboardClient --> AlertsBanner[告警横幅]
    DashboardClient --> StatusGridSection[状态网格区域]
    DashboardClient --> HistoryChartSection[历史图表区域]
    DashboardClient --> DashboardFooter[DashboardFooter]
    
    StatCardSection --> StatCard[StatCard × 4]
    
    StatusGridSection --> ApiConfig[ApiConfig]
    StatusGridSection --> ApiStatusGrid[ApiStatusGrid]
    ApiStatusGrid --> StatusDot[StatusDot]
    ApiStatusGrid --> ProgressBar[ProgressBar]
    
    HistoryChartSection --> DashboardSkeleton[页面级 Suspense 骨架]
    HistoryChartSection --> LatencyHistoryChart[LatencyHistoryChart]
    
    DashboardHeader --> AlertsDropdown[AlertsDropdown]
```

### 2.2 组件职责说明

| 组件 | 职责 | 状态依赖 |
|-----|------|---------|
| **RootLayout** | 根布局，设置主题和全局样式 | 无 |
| **ThemeProvider** | 主题管理，浅色/深色双主题（next-themes `defaultTheme="system"`），基于 next-themes | `theme` |
| **StructuredData** | SEO 结构化数据组件 | 无 |
| **DashboardClient** | 主页面客户端组件，整合所有功能 | 所有状态 |
| **DashboardHeader** | 头部导航，包含品牌、告警、主题切换、用户认证 | `user`, `alerts`, `theme`, `geo`, `isGeoLoading` |
| **GeoOptInDialog** | 地理位置授权对话框 | `geo` |
| **AlertsDropdown** | 告警下拉对话框 | `alerts` |
| **StatCard** | 统计卡片，显示在线/降级/离线/平均延迟 | `statuses` |
| **AlertsBanner** | 告警横幅，显示活跃告警数量 | `alerts` |
| **ApiStatusGrid** | API 状态卡片网格 | `statuses` |
| **StatusDot** | 状态指示点 | `status` |
| **ProgressBar** | 延迟进度条 | `latency` |
| **ApiConfig** | API 配置面板 | 本地存储 |
| **StatusGrid** | 兼容层组件，转发到 ApiStatusGrid | `statuses` |
| **LatencyHistoryChart** | 延迟历史图表 | `history`, `statuses` |
| **DashboardSkeleton** | 页面级加载骨架屏（ChartSkeleton 无引用） | 无 |
| **DashboardFooter** | 页脚信息 | 无 |

### 2.3 数据管理架构

```mermaid
graph TD
    subgraph Store [Zustand Stores]
        API[api.ts]
        Auth[auth.ts]
        Alerts[alerts.ts]
        Geo[geo.ts]
        Error[error.ts]
    end

    subgraph Hooks [Custom Hooks]
        useDashboard[useDashboardData]
        useApi[useApiMonitor]
        useAuthHook[useAuth]
        useAlertsHook[useAlerts]
        useGeo[useGeoLocation]
    end

    subgraph External [External Data Sources]
        Supabase[(Supabase PostgreSQL)]
        GeoAPI[Geo Location API]
        AuthAPI[Supabase Auth]
    end

    useDashboard --> API
    useDashboard --> Auth
    useDashboard --> Alerts
    useDashboard --> Geo
    
    useApi --> API
    useApi --> Supabase
    
    useAuthHook --> Auth
    useAuthHook --> AuthAPI
    
    useAlertsHook --> Alerts
    useAlertsHook --> Supabase
    
    useGeo --> Geo
    useGeo --> GeoAPI
    
    Supabase -->|首屏 select| API
    Supabase -->|Realtime| Alerts
```

### 2.4 状态管理模块

| Store | 职责 | 持久化 |
|-----|------|--------|
| **api.ts** | 管理 API 状态和历史数据 | 部分（statuses） |
| **auth.ts** | 管理用户认证状态 | 无（从 Supabase 获取） |
| **alerts.ts** | 管理告警列表 | 无（从 Supabase 获取） |
| **geo.ts** | 管理地理位置信息 | 是 |
| **error.ts** | 管理错误和通知 | 无 |

## 3. 后端架构

### 3.1 后台监控流程

```mermaid
flowchart TD
    A[Express Server 启动] --> B[延迟 10 秒]
    B --> C[执行首次检查]
    C --> D{检查成功?}
    
    D -->|是| E[批量写入 Supabase]
    D -->|否| F[记录错误日志]
    
    E --> G[等待 5 分钟]
    F --> G
    
    G --> H[执行定时检查]
    H --> D
    
    subgraph 检查流程
        I[获取 API 列表]
        J[并发管理器控制]
        K[执行单个 API 检查]
        L[检查缓存]
        M[发送 HTTP 请求]
        N[计算指标]
        O[更新缓存]
        
        I --> J
        J --> K
        K --> L
        L -->|有缓存| O
        L -->|无缓存| M
        M --> N
        N --> O
    end
    
    C --> I
    H --> I
```

### 3.2 并发控制机制

```mermaid
graph TD
    subgraph ConcurrencyManager
        Queue[请求队列]
        Active[活跃请求池]
        Limit[并发限制器\nMAX_CONCURRENT_REQUESTS=5]
        Processor[请求处理器]
    end
    
    subgraph NetworkQuality
        Excellent[Excellent\nlimit=8]
        Good[Good\nlimit=5]
        Fair[Fair\nlimit=2]
        Poor[Poor\nlimit=1]
    end
    
    API[API 请求] --> Queue
    Queue --> Limit
    Limit -->|<= limit| Active
    Limit -->|> limit| Queue
    
    Active --> Processor
    Processor -->|完成| Result[返回结果]
    Processor -->|完成| Limit
    
    NetworkQuality -->|动态调整| Limit
```

### 3.3 智能告警流程

```mermaid
flowchart TD
    A[API 检查完成] --> B{状态为 offline?}
    
    B -->|是| C[检查是否已有同类告警]
    B -->|否| D{延迟 > LATENCY_THRESHOLD?}
    
    C -->|有| E[跳过，不重复创建]
    C -->|无| F[创建 downtime 告警]
    
    D -->|是| G[计算严重程度]
    D -->|否| H[检查完成]
    
    G --> I[检查是否已有同类告警]
    I -->|有| E
    I -->|无| J[创建 latency 告警]
    
    F --> K[发送通知]
    J --> K
    
    K --> H
    E --> H
```

## 4. 数据流

### 4.1 实时数据同步流程

```mermaid
sequenceDiagram
    participant Client as Next.js Client
    participant Auth as Supabase Auth
    participant Supabase as Supabase
    participant Monitor as Supabase Cron / Edge Function monitor
    
    Client->>Auth: 用户登录请求
    Auth-->>Client: 返回用户信息
    
    Client->>Supabase: 订阅 api_status 表
    Client->>Supabase: 订阅 alerts 表
    
    Note over Monitor: 每 5 分钟执行一次
    Monitor->>Monitor: 执行 API 检查
    Monitor->>Supabase: 批量写入检查结果
    Supabase-->>Client: 实时推送更新
    
    Client->>Client: 更新 Zustand Store
    Client->>Client: 重新渲染组件
```

### 4.2 手动检查流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Client as Next.js Client
    participant Monitor as Monitor Logic
    participant Supabase as Supabase
    
    User->>Client: 点击"立即检查"按钮
    Client->>Client: 检查用户是否已登录
    
    alt 用户已登录
        Client->>Monitor: 调用 runCheck()
        Monitor->>Monitor: 执行批量 API 检查
        Monitor->>Supabase: 写入检查结果
        Supabase-->>Client: 实时同步更新
        Client->>Client: 更新界面显示
    else 用户未登录
        Client->>Client: 显示登录提示
    end
```

### 4.3 告警处理流程

```mermaid
sequenceDiagram
    participant Client as Next.js Client
    participant Supabase as Supabase
    participant Notification as Notification Service
    
    Supabase-->>Client: 推送新告警
    Client->>Client: 更新告警状态
    Client->>Client: 显示告警横幅/铃铛
    
    User->>Client: 点击查看告警
    Client->>Client: 打开告警下拉菜单
    
    User->>Client: 点击"解决"按钮
    Client->>Supabase: 更新告警 resolved=true
    Supabase-->>Client: 同步更新
    
    Client->>Client: 从界面移除告警
    
    Note over Notification: 可选：发送 Webhook 通知（默认 enabled: false）
    Supabase->>Notification: 新告警触发
    Notification->>Notification: 根据配置发送通知
```

## 5. 缓存架构

### 5.1 多层缓存系统

```mermaid
graph TD
    subgraph CacheLayers
        Memory[内存缓存]
        LocalStorage[localStorage]
        SessionStorage[sessionStorage]
    end
    
    subgraph CachePolicy
        Default[默认 30 秒]
        Online[在线 & 低延迟\n延长至 60 秒]
        Offline[离线\n缩短至 5 秒]
        Degraded[降级\n缩短至 15 秒]
    end
    
    subgraph Monitor[监控逻辑]
        CheckAPI[检查 API]
        GetCache[获取缓存]
        SetCache[设置缓存]
    end
    
    CheckAPI --> GetCache
    GetCache --> Memory
    Memory -->|命中| CheckAPI
    Memory -->|未命中| LocalStorage
    LocalStorage -->|命中| Memory
    LocalStorage -->|未命中| CheckAPI
    
    CheckAPI -->|成功| SetCache
    SetCache --> Memory
    Memory --> LocalStorage
    Memory --> SessionStorage
    
    CachePolicy -->|动态计算| SetCache
```

### 5.2 缓存过期策略

| 状态 | 过期时间 | 说明 |
|-----|---------|------|
| **online + latency < 100ms** | 60 秒 | 快速响应，延长缓存 |
| **online + latency 100-1000ms** | 30 秒 | 默认缓存时间 |
| **online + latency > 1000ms** | 15 秒 | 高延迟，缩短缓存 |
| **degraded** | 15 秒 | 降级状态，缩短缓存 |
| **offline** | 5 秒 | 离线状态，快速重试 |

## 6. 部署架构

### 6.1 部署组件

```mermaid
graph TD
    subgraph Deployment
        Vercel[Vercel / EdgeOne Pages]
        Express[Express Server]
    end
    
    subgraph Database
        Supabase[(Supabase PostgreSQL)]
        Auth[Supabase Auth]
    end
    
    subgraph CDN
        Static[静态资源 CDN]
    end
    
    User[用户] --> Static
    User --> Vercel
    Vercel --> Supabase
    Vercel --> Auth
    
    Express --> Supabase
```

### 6.2 部署选项对比

| 选项 | 适用场景 | 优势 | 劣势 |
|-----|---------|------|------|
| **Vercel** | 纯静态前端 | 自动 SSL、全球 CDN、无缝 Next.js 集成 | 无后端支持 |
| **EdgeOne Pages** | 中国区部署 | 国内加速、低成本 | 功能相对简单 |
| **Express Server** | 后台任务 | 可控性强、支持定时任务 | 需要服务器维护 |

## 7. 安全架构

### 7.1 安全层级

```mermaid
graph TD
    subgraph SecurityLayers
        AuthLayer[认证层]
        RLSPolicies[RLS 策略层]
        ClientValidation[客户端验证层]
        HTTPS[传输层]
    end
    
    subgraph ExternalAccess
        User[用户]
        Admin[管理员]
        Monitor[监控服务]
    end
    
    User --> AuthLayer
    Admin --> AuthLayer
    
    AuthLayer -->|认证用户| RLSPolicies
    AuthLayer -->|未认证| Deny[拒绝]
    
    RLSPolicies -->|读取| AllowRead[允许读取]
    RLSPolicies -->|写入| AdminCheck[管理员检查]
    
    AdminCheck -->|管理员| AllowWrite[允许写入]
    AdminCheck -->|非管理员| Deny
    
    Monitor -->|服务器端| RLSPolicies
    Monitor -->|Service Role| AllowWrite
    
    HTTPS -->|加密传输| SecurityLayers
```

### 7.2 安全规则矩阵

| 表名 | 读取权限 | 写入权限 | 说明 |
|-----|---------|---------|------|
| `api_status` | 公开 | 认证用户 | 状态信息公开可读，认证用户可写入 |
| `status_history` | 公开 | 认证用户 | 历史数据公开可读，认证用户可插入 |
| `alerts` | 公开 | 认证用户 | 告警信息公开可读，认证用户可更新(解决)和插入 |
| `user_profiles` | 仅本人 | 仅本人 | 用户资料仅本人可读写 |

## 8. 性能优化架构

### 8.1 前端优化策略

```mermaid
graph TD
    subgraph Optimizations
        Memo[React.memo]
        UseMemo[useMemo]
        ChartLimit[图表数据限制]
        GeoCache[地理位置缓存]
        APICache[API 缓存]
        CodeSplit[代码分割]
    end
    
    subgraph Components
        Grid[StatusGrid]
        Chart[LatencyHistoryChart]
        Header[DashboardHeader]
        Dropdown[AlertsDropdown]
    end
    
    Memo --> Grid
    Memo --> Chart
    
    UseMemo --> Grid
    UseMemo --> Chart
    
    ChartLimit --> Chart
    
    GeoCache --> Header
    
    APICache --> Grid
    APICache --> Chart
    
    CodeSplit --> Dropdown
```

### 8.2 后端优化策略

| 优化项 | 实现方式 | 收益 |
|-----|---------|------|
| **并发控制** | 限制最大并发数为 5 | 避免请求过多被限流 |
| **重试机制** | 固定 1000ms 延迟重试（RETRY_DELAY） | 提高成功率 |
| **批量写入** | Supabase 批量操作 | 减少网络往返 |
| **智能缓存** | 根据状态动态调整过期时间 | 减少不必要请求 |
| **超时控制** | 6 秒请求超时 | 防止长时间阻塞 |