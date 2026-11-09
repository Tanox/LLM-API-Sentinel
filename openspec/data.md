# 数据模型与安全文档

> 适用版本：v2.10.35 · 最近校对：2026-11-09 · 真源：`app/`、`supabase/` 与 `package.json`
>
> **代码对齐要点（下文旧表述与本表冲突时，以本表及源码为准）**：
> - 默认后台写入方为 Supabase Cron → Edge Function `monitor`（Deno），不是 Express Monitor；Express 仅见于可选的 `examples/self-host-server.ts`。
> - NaN
> - 仓库未启用自动数据清理：`supabase/cron.sql` 只有被注释掉的 `status_history` 90 天清理示例，`alerts` 无清理任务。
> - 写入 `api_status`（upsert）与 `status_history`（insert）是两次独立调用，非同一事务，不保证原子性/回滚。
> - 文档中的 Firestore 术语（`where()` / `getAll()` / Batch / 文档 / 集合）、`validate_api_status()` 触发器与 `cleanupOldData()` 均为历史示意，当前实现为 PostgreSQL + `CHECK` 约束 + `update_updated_at_column` 触发器，详见 `supabase/schema.sql`。

## 1. 数据模型

### 1.1 实体定义

LLM API Sentinel 使用 Supabase PostgreSQL 作为主要数据库，包含以下核心实体：

| 实体 | 描述 | 生命周期 |
|-----|------|---------|
| **ApiStatus** | API 的当前状态，包含基本信息、状态、延迟、指标等 | 实时更新 |
| **StatusHistory** | API 的历史性能数据，用于生成图表 | 持续追加 |
| **Alert** | 系统告警信息，包含告警类型、严重程度、解决状态等 | 创建→解决→保留 |

### 1.2 表结构概览

| 表名 | 用途 | 读取权限 | 写入权限 |
|-----|------|---------|---------|
| `api_status` | 存储每个 API 的当前状态 | 公开 | 认证用户 |
| `status_history` | 存储 API 状态历史记录 | 公开 | 认证用户 |
| `alerts` | 存储系统告警信息 | 公开 | 认证用户 |
| `user_profiles` | 用户资料表（关联 Auth） | 仅本人 | 仅本人 |

### 1.3 数据结构

#### ApiStatus
```typescript
interface ApiStatus {
  id: string;                    // API 唯一标识，格式: {provider}-{model}
  name: string;                  // 显示名称，如 "GPT-4o"
  provider: string;              // 提供商名称，如 "OpenAI"
  url: string;                   // 检查 URL
  status: 'online' | 'offline' | 'degraded';  // 状态
  latency: number;               // 延迟(ms)
  lastChecked: string;           // 最后检查时间 (ISO 字符串)
  error?: string;                // 错误信息
  retries?: number;              // 重试次数
  errorRate?: number;            // 错误率(%)
  availability?: number;         // 可用性(%)
  uptime?: number;               // 正常运行时间(%)
  averageLatency?: number;       // 平均延迟(ms)
  maxLatency?: number;           // 最大延迟(ms)
  minLatency?: number;           // 最小延迟(ms)
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| `id` | string | 是 | 唯一标识符 |
| `name` | string | 是 | 显示名称 |
| `provider` | string | 是 | 提供商名称 |
| `url` | string | 是 | 检查端点 URL |
| `status` | enum | 是 | 状态值 |
| `latency` | number | 是 | 延迟毫秒数 |
| `lastChecked` | string | 是 | ISO 时间戳 |
| `error` | string | 否 | 错误描述 |
| `retries` | number | 否 | 重试次数 |
| `errorRate` | number | 否 | 错误率百分比 |
| `availability` | number | 否 | 可用性百分比 |
| `uptime` | number | 否 | 正常运行时间百分比 |

#### StatusHistory
```typescript
interface StatusHistory {
  id?: string;                   // 记录 ID (自动生成 UUID)
  apiId: string;                 // 关联的 API ID
  status: 'online' | 'offline' | 'degraded';  // 状态
  latency: number;               // 延迟(ms)
  timestamp: Date;               // 时间戳
  time: string;                  // 格式化时间字符串（用于图表显示）
  error?: string;                // 错误信息
  retries?: number;              // 重试次数
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| `id` | string | 否 | UUID，自动生成 |
| `apiId` | string | 是 | 关联 API ID |
| `status` | enum | 是 | 状态值 |
| `latency` | number | 是 | 延迟毫秒数 |
| `timestamp` | Date | 是 | 时间戳 |
| `time` | string | 是 | 格式化时间字符串（图表 X 轴显示用） |
| `error` | string | 否 | 错误描述 |
| `retries` | number | 否 | 重试次数 |

**数据库对应字段（蛇形命名）**：
| TypeScript 字段 | 数据库字段 |
|----------------|-----------|
| apiId | api_id |
| lastChecked | last_checked |
| errorRate | error_rate |
| averageLatency | average_latency |
| maxLatency | max_latency |
| minLatency | min_latency |
| resolvedAt | resolved_at |
| resolvedBy | resolved_by |

#### Alert
```typescript
interface Alert {
  id: string;                    // 告警 ID
  apiId: string;                 // 关联的 API ID
  apiName: string;               // API 显示名称
  type: 'downtime' | 'latency' | 'error';  // 告警类型
  severity: 'low' | 'medium' | 'high' | 'critical';  // 严重程度
  message: string;               // 告警消息
  timestamp: Date | string | number;  // 时间戳
  resolved: boolean;             // 是否已解决
  error?: string;                // 错误信息
  retries?: number;              // 重试次数
  latency?: number;              // 延迟值(仅 latency 类型)
  resolvedAt?: Date;             // 解决时间
  resolvedBy?: string;           // 解决人(用户ID或邮箱)
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| `id` | string | 是 | 唯一标识符 |
| `apiId` | string | 是 | 关联 API |
| `apiName` | string | 是 | API 名称 |
| `type` | enum | 是 | 告警类型 |
| `severity` | enum | 是 | 严重程度 |
| `message` | string | 是 | 告警消息 |
| `timestamp` | Date | 是 | 创建时间 |
| `resolved` | boolean | 是 | 是否已解决 |
| `resolvedAt` | Date | 否 | 解决时间 |
| `resolvedBy` | string | 否 | 解决人 |

## 2. 数据流转

### 2.1 数据流向图

```mermaid
graph TD
    subgraph External[外部系统]
        Monitor[Supabase Cron / Edge Function]
        User[用户操作]
    end
    
    subgraph Database[Supabase PostgreSQL]
        API[api_status]
        History[status_history]
        Alerts[alerts]
    end
    
    subgraph Client[前端]
        Dashboard[Dashboard]
        Charts[Charts]
        AlertUI[Alert UI]
    end
    
    Monitor -->|每5分钟| API
    Monitor -->|每5分钟| History
    Monitor -->|异常时| Alerts
    
    User -->|手动检查| API
    User -->|解决告警| Alerts
    
    API -->|首屏查询 + 前端定时探测| Dashboard
    API -->|前端定时探测| Charts
    
    History -->|查询| Charts
    
    Alerts -->|Realtime| AlertUI
    Alerts -->|Realtime| Dashboard
```

### 2.2 数据写入流程

#### API 状态更新流程
```mermaid
flowchart TD
    A[Supabase Cron 触发 Edge Function] --> B[批量获取 API 列表]
    B --> C[并发执行 API 检查]
    C --> D{检查成功?}
    
    D -->|是| E[构建 ApiStatus 对象]
    D -->|否| F[记录错误信息]
    
    E --> G[更新内存缓存]
    F --> G
    
    G --> H[批量写入 Supabase]
    H --> I[触发实时更新]
    I --> J[客户端收到更新]
    
    J --> K[更新 Zustand Store]
    K --> L[重新渲染组件]
```

#### 告警创建流程
```mermaid
flowchart TD
    A[API 检查完成] --> B{状态异常?}
    
    B -->|是| C[检查是否已有同类告警]
    B -->|否| D[流程结束]
    
    C -->|已有| E[跳过，避免重复]
    C -->|无| F[创建 Alert 对象]
    
    F --> G[写入 Supabase]
    G --> H[触发实时推送]
    H --> I[客户端显示告警]
    
    E --> D
```

### 2.3 数据读取流程

#### 首页数据加载流程
```mermaid
flowchart TD
    A[用户访问首页] --> B[初始化 Supabase]
    B --> C[订阅 api_status 表]
    B --> D[订阅 alerts 表]
    
    C --> E[获取初始状态]
    D --> F[获取活跃告警]
    
    E --> G[更新状态 Store]
    F --> H[更新告警 Store]
    
    G --> I[渲染状态网格]
    H --> J[渲染告警横幅]
    
    I --> K[持续监听变更]
    J --> K
    
    K --> L[数据更新时自动刷新]
```

#### 历史数据查询流程
```mermaid
flowchart TD
    A[用户查看图表] --> B[获取时间范围]
    B --> C[构建查询条件]
    
    C --> D[查询 status_history]
    D --> E[按 apiId 分组]
    
    E --> F[计算统计指标]
    F --> G[构建图表数据]
    
    G --> H[渲染图表]
```

## 3. 索引策略

### 3.1 索引需求分析

| 查询场景 | 所需字段 | 排序 | 索引类型 |
|---------|---------|-----|---------|
| 获取所有 API 状态 | 无 | 无 | 集合扫描(小数据集) |
| 获取特定 API 状态 | `id` | 无 | 主键索引(默认) |
| 获取活跃告警 | `resolved` | `timestamp DESC` | 复合索引 |
| 查询历史数据 | `apiId`, `timestamp` | `timestamp DESC` | 复合索引 |
| 按提供商筛选 | `provider` | 无 | 单字段索引 |

### 3.2 数据库索引

实际使用 PostgreSQL 索引，定义在 `supabase/schema.sql` 中：

```sql
-- 历史数据索引：按 API 查询历史记录
CREATE INDEX IF NOT EXISTS idx_status_history_api_id ON status_history(api_id);
CREATE INDEX IF NOT EXISTS idx_status_history_timestamp ON status_history(timestamp DESC);

-- 告警索引：快速获取未解决告警
CREATE INDEX IF NOT EXISTS idx_alerts_api_id ON alerts(api_id);
CREATE INDEX IF NOT EXISTS idx_alerts_resolved ON alerts(resolved);
CREATE INDEX IF NOT EXISTS idx_alerts_timestamp ON alerts(timestamp DESC);
```

### 3.3 索引用途说明

| 索引 | 表名 | 用途 | 优先级 |
|-----|------|------|--------|
| `idx_status_history_api_id` | status_history | 按 API ID 过滤历史记录 | 高 |
| `idx_status_history_timestamp` | status_history | 按时间排序历史数据 | 高 |
| `idx_alerts_resolved` | alerts | 过滤已解决/未解决告警 | 高 |
| `idx_alerts_timestamp` | alerts | 按时间倒序排列告警 | 高 |
| `idx_alerts_api_id` | alerts | 按 API 查找告警 | 中 |

## 4. 数据生命周期管理

### 4.1 数据保留策略

| 数据类型 | 保留期限 | 清理策略 |
|---------|---------|---------|
| **ApiStatus** | 永久 | 不清理，持续更新 |
| **StatusHistory** | 30 天 | 定时清理过期记录 |
| **Alerts** | 90 天 | 解决后保留 90 天 |

### 4.2 清理流程

```mermaid
flowchart TD
    A[定时任务触发] --> B[计算过期时间]
    
    B --> C[清理 status_history]
    B --> D[清理 alerts]
    
    C --> E[删除 30 天前的记录]
    D --> F[删除已解决且超过 90 天的记录]
    
    E --> G[记录清理日志]
    F --> G
    
    G --> H[流程结束]
```

### 4.3 清理脚本示例

```typescript
import { supabase } from '../app/lib/supabase';
import { logError } from '../app/lib/error-handler';

async function cleanupOldData() {
  const thirtyDaysAgo = new Date();
  thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

  const ninetyDaysAgo = new Date();
  ninetyDaysAgo.setDate(ninetyDaysAgo.getDate() - 90);

  // 清理历史数据（删除 30 天前的记录）
  const { error: historyError } = await supabase
    .from('status_history')
    .delete()
    .lt('timestamp', thirtyDaysAgo.toISOString());

  if (historyError) {
    logError(historyError, 'Failed to clean up old status history');
  }

  // 清理已解决的告警（删除已解决超过 90 天的记录）
  const { error: alertsError } = await supabase
    .from('alerts')
    .delete()
    .eq('resolved', true)
    .lt('resolved_at', ninetyDaysAgo.toISOString());

  if (alertsError) {
    logError(alertsError, 'Failed to clean up old alerts');
  }
}
```

## 5. 备份策略

### 5.1 备份类型

| 类型 | 频率 | 保留 | 用途 |
|-----|------|-----|------|
| **自动备份** | 每日 | 7 天 | 日常恢复 |
| **手动备份** | 按需 | 永久 | 重大变更前 |
| **导出备份** | 每周 | 30 天 | 合规存档 |

### 5.2 备份流程

```mermaid
flowchart TD
    A[触发备份] --> B[导出数据]
    
    B --> C[生成备份文件]
    C --> D[上传到存储]
    
    D --> E[记录备份元数据]
    E --> F[验证备份完整性]
    
    F -->|成功| G[通知管理员]
    F -->|失败| H[重试或告警]
```

### 5.3 恢复流程

```mermaid
flowchart TD
    A[确认恢复需求] --> B[选择备份版本]
    
    B --> C[下载备份文件]
    C --> D[验证文件完整性]
    
    D -->|有效| E[停止写入操作]
    D -->|无效| F[选择其他备份]
    
    E --> G[清空/重建目标表数据]
    G --> H[批量导入数据]
    
    H --> I[恢复写入操作]
    I --> J[通知管理员]
    
    F --> B
```

## 6. 安全规则

### 6.1 Supabase RLS 策略

安全模型（自 v2.6.3 起的历史策略）：
- **READ (SELECT)**: 公开（所有用户，包括 anon）- 允许仪表盘数据访问
- **WRITE (INSERT/UPDATE)**: 仅认证用户 - 匿名用户不能修改数据
- **后台服务**: 使用 service_role 密钥，绕过 RLS

```sql
-- 启用 RLS
ALTER TABLE api_status ENABLE ROW LEVEL SECURITY;
ALTER TABLE status_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE alerts ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;

-- API Status: 所有人可读，仅认证用户可写
CREATE POLICY "api_status_read_all" ON api_status FOR SELECT USING (true);
CREATE POLICY "api_status_insert_authenticated" ON api_status FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "api_status_update_authenticated" ON api_status FOR UPDATE TO authenticated USING (true);

-- Status History: 所有人可读，仅认证用户可插入
CREATE POLICY "status_history_read_all" ON status_history FOR SELECT USING (true);
CREATE POLICY "status_history_insert_authenticated" ON status_history FOR INSERT TO authenticated WITH CHECK (true);

-- Alerts: 所有人可读，认证用户可更新(解决)和插入
CREATE POLICY "alerts_read_all" ON alerts FOR SELECT USING (true);
CREATE POLICY "alerts_update_authenticated" ON alerts FOR UPDATE TO authenticated USING (true);
CREATE POLICY "alerts_insert_authenticated" ON alerts FOR INSERT TO authenticated WITH CHECK (true);

-- User Profiles: 用户可读写自己的资料
CREATE POLICY "user_profiles_select_own" ON user_profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "user_profiles_insert_own" ON user_profiles FOR INSERT WITH CHECK (auth.uid() = id);
CREATE POLICY "user_profiles_update_own" ON user_profiles FOR UPDATE USING (auth.uid() = id);

-- 显式最小权限授权
GRANT USAGE ON SCHEMA public TO anon, authenticated;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO authenticated;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO anon;
```

### 6.2 安全策略说明

| 表名 | 读取权限 | 写入权限 | 说明 |
|-----|---------|---------|------|
| `api_status` | 公开 | 认证用户 | API 状态信息公开可读 |
| `status_history` | 公开 | 认证用户 | 历史数据公开可读 |
| `alerts` | 公开 | 认证用户 | 告警信息公开可读，认证用户可解决 |
| `user_profiles` | 仅本人 | 仅本人 | 用户资料仅本人可访问 |

### 6.3 自动创建用户资料

用户注册时自动创建 user_profiles 记录：

```sql
-- 触发器函数
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.user_profiles (id, email, display_name)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', NEW.raw_user_meta_data->>'name')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Auth 用户表触发器
DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

### 6.4 数据验证规则

#### ApiStatus 数据验证
```sql
-- 创建触发器验证数据格式
CREATE OR REPLACE FUNCTION validate_api_status()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.id IS NULL OR NEW.id = '' THEN
    RAISE EXCEPTION 'API ID is required';
  END IF;
  
  IF NEW.status NOT IN ('online', 'offline', 'degraded') THEN
    RAISE EXCEPTION 'Invalid status value';
  END IF;
  
  IF NEW.latency < 0 THEN
    RAISE EXCEPTION 'Latency cannot be negative';
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER api_status_validation
  BEFORE INSERT OR UPDATE ON api_status
  FOR EACH ROW EXECUTE FUNCTION validate_api_status();
```

#### Alert 数据验证
```sql
CREATE OR REPLACE FUNCTION validate_alert()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.type NOT IN ('downtime', 'latency', 'error') THEN
    RAISE EXCEPTION 'Invalid alert type';
  END IF;
  
  IF NEW.severity NOT IN ('low', 'medium', 'high', 'critical') THEN
    RAISE EXCEPTION 'Invalid severity value';
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER alert_validation
  BEFORE INSERT OR UPDATE ON alerts
  FOR EACH ROW EXECUTE FUNCTION validate_alert();
```

## 7. 数据一致性

### 7.1 批量写入流程

**写入操作分两步（非事务，但有错误处理）**：
```typescript
// examples/self-host-server.ts 中的后台监控写入逻辑
async function runBackgroundMonitor() {
  const results = await performCheck();

  // 第一步：Upsert API 状态
  const upsertData = results.map(result => ({
    id: result.id,
    name: result.name,
    provider: result.provider,
    url: result.url,
    status: result.status,
    latency: result.latency,
    last_checked: result.lastChecked,
    error: result.error || null,
    retries: result.retries || 0,
    error_rate: result.errorRate || 0,
    availability: result.availability || 100,
    uptime: result.uptime || 100,
    average_latency: result.averageLatency || null,
    max_latency: result.maxLatency || null,
    min_latency: result.minLatency || null,
    updated_at: new Date().toISOString(),
  }));

  const { error: upsertError } = await supabase
    .from('api_status')
    .upsert(upsertData, { onConflict: 'id' });

  if (upsertError) logError(upsertError, 'Failed to upsert API statuses');

  // 第二步：插入历史记录
  const historyData = results.map(result => ({
    api_id: result.id,
    status: result.status,
    latency: result.latency,
    error: result.error || null,
    retries: result.retries || 0,
    timestamp: new Date().toISOString(),
  }));

  const { error: historyError } = await supabase
    .from('status_history')
    .insert(historyData);

  if (historyError) logError(historyError, 'Failed to insert history records');
}
```

**注意**：当前实现中两步写入不是在同一个数据库事务中，失败时只记录错误日志，不回滚。

### 7.2 一致性保证

| 保证类型 | 实现方式 |
|-----|---------|
| **原子性** | Supabase 批量操作保证 |
| **隔离性** | PostgreSQL 内置事务支持 |
| **持久性** | PostgreSQL WAL 自动持久化 |

## 8. 数据迁移

### 8.1 迁移流程

```mermaid
flowchart TD
    A[准备迁移脚本] --> B[备份当前数据]
    
    B --> C[测试迁移脚本]
    C --> D{测试通过?}
    
    D -->|是| E[执行迁移]
    D -->|否| F[修复脚本]
    
    E --> G[验证迁移结果]
    G --> H{验证通过?}
    
    H -->|是| I[清理旧数据]
    H -->|否| J[回滚到备份]
    
    I --> K[完成迁移]
    F --> C
    J --> K
```

### 8.2 迁移注意事项

- 迁移前必须备份数据
- 迁移期间暂停写入操作
- 验证迁移后数据完整性
- 保留回滚方案

## 9. 性能优化

### 9.1 查询优化

| 优化项 | 实现方式 |
|-----|---------|
| **限制数据量** | 使用 `limit()` 限制返回数量 |
| **时间范围筛选** | 使用 `where()` 限制时间范围 |
| **复合索引** | 创建适当的复合索引 |
| **批量读取** | 使用 `getAll()` 批量获取文档 |

### 9.2 写入优化

| 优化项 | 实现方式 |
|-----|---------|
| **批量写入** | 使用 Batch 操作 |
| **减少写入频率** | 聚合更新而非单次更新 |
| **离线支持** | 使用 Supabase 离线模式 |

### 9.3 缓存策略

| 缓存层 | 数据类型 | 过期时间 |
|-----|---------|---------|
| **内存缓存** | API 状态 | 30 秒 |
| **LocalStorage** | 用户配置 | 永久 |
| **SessionStorage** | 会话数据 | 会话结束 |