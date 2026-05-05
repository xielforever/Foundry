# Foundry v1 - 流程审计方案设计文档

| 属性 | 内容 |
|------|------|
| **文档标题** | Foundry v1 - 流程审计方案设计文档 |
| **文档作者** | Foundry Team |
| **文档日期** | 2026-05-05 |
| **文档版本** | v1.1 |
| **文档描述** | Foundry v1 审计日志数据结构、持久化方案和查询接口设计，覆盖 AC-5 |

---

## 概述

本文档定义 Foundry v1 的流程审计方案。可审计性是 Foundry 的核心要求——所有 Agent 的执行行为必须可追溯、可查询、可复现。

本文档覆盖以下验收标准：
- **AC-5**：流程审计方案设计文档（审计日志数据结构、持久化方案、查询接口设计）
- **AC-7**：工程哲学遵循验证（审计设计符合四大工程哲学）

### 读者

- 软件架构师：理解审计日志的整体架构和存储选型
- 一线开发者：根据审计数据结构和查询接口实现 Foundry Core 的审计模块
- DevOps 工程师：理解审计日志的持久化方案和查询方式
- 安全合规团队：理解审计追溯能力和数据保护机制

### 前置依赖

- [task_artifact_data_model.md](task_artifact_data_model.md)：ArtifactMetadata 溯源字段（producer/pipeline/stage/step）、Task.created_by/created_at、Artifact 生命周期各状态、敏感数据处理约束
- [agent_executor_architecture.md](agent_executor_architecture.md)：Executor 接口、Agent 执行生命周期、ExecutionStatus 枚举、ExecutionMetrics、TaskSchedulingRecord
- [failure_handling_and_human_intervention.md](failure_handling_and_human_intervention.md)：失败状态机、Intervention 数据结构、回滚操作、ERROR_REPORT Artifact
- [spec.md](../../.trae/specs/foundry-v1/spec.md)：FR-5（流程审计方案设计）、NFR-1（可审计属性）

---

## 设计动机

### 为什么需要系统化的审计方案

Foundry 的核心价值主张之一是"可审计"——所有行为必须可追溯。在多 Agent 分工并行模型中，审计面临以下挑战：

| 挑战 | 说明 |
|------|------|
| Agent 类型多样 | 四种 Agent 类型的执行方式差异巨大，审计日志必须统一记录 |
| 执行并发 | 同一 Pipeline 内多个 Step 可能并行执行，审计日志必须支持时序查询 |
| 失败追溯 | 失败处理链路可能跨越重试、人工介入、回滚，审计日志必须记录完整链路 |
| 敏感数据 | Agent 输入/输出可能包含密钥、Token、漏洞详情，审计日志必须脱敏 |
| 合规要求 | 审计日志必须不可篡改，保留策略满足合规要求 |

系统化审计方案的核心价值：

1. **可追溯**：任意 Artifact 可追溯到产出它的 Agent、Task、Pipeline
2. **可查询**：支持按 Pipeline/Stage/Step、时间范围、Agent 类型等维度查询
3. **可复现**：审计日志包含足够的上下文信息，可复现执行场景
4. **可合规**：审计日志不可篡改，保留策略满足合规要求

### 设计约束

| 约束来源 | 约束内容 |
|---------|---------|
| Flow First | 审计日志附着在 Pipeline/Stage/Step 中，记录流程中每个节点的执行情况 |
| Deterministic Over Smart | 审计日志结构确定，查询结果确定，不依赖全文搜索的"相关性排序" |
| Artifact Over Conversation | 审计日志是结构化数据，不是无结构文本；审计查询结果是结构化的 Artifact |
| Agent Is Replaceable | 审计日志不依赖特定 Agent 类型的内部状态，通过统一的 AuditEvent 记录 |

---

## 详细设计

### 1. 审计日志数据结构

#### 1.1 AuditEvent 核心结构

审计日志的核心单元是 AuditEvent，记录一次 Agent 执行的完整信息：

```protobuf
message AuditEvent {
  string event_id = 1;
  AuditEventType event_type = 2;
  string pipeline_id = 3;
  string stage_id = 4;
  string step_id = 5;
  string task_id = 6;
  string executor_agent_id = 7;
  AgentType executor_agent_type = 8;
  ExecutionStatus execution_status = 9;
  AuditEventInput input = 10;
  AuditEventOutput output = 11;
  AuditEventContext context = 12;
  google.protobuf.Timestamp started_at = 13;
  google.protobuf.Timestamp completed_at = 14;
  int64 duration_ms = 15;
  map<string, string> labels = 16;
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `event_id` | `string` | 是 | UUID v4 格式，全局唯一 | 审计事件唯一标识 |
| `event_type` | `AuditEventType` | 是 | 枚举值 | 事件类型（见 1.2 节） |
| `pipeline_id` | `string` | 是 | 非空 | 所属 Pipeline 标识 |
| `stage_id` | `string` | 是 | 非空 | 所属 Stage 标识 |
| `step_id` | `string` | 是 | 非空 | 所属 Step 标识 |
| `task_id` | `string` | 是 | UUID v4 格式 | 关联的 Task 标识 |
| `executor_agent_id` | `string` | 是 | 非空 | 执行该 Task 的 Executor 标识 |
| `executor_agent_type` | `AgentType` | 是 | 枚举值 | Executor 的 Agent 类型 |
| `execution_status` | `ExecutionStatus` | 是 | 枚举值 | 执行结果状态 |
| `input` | `AuditEventInput` | 是 | 非空 | 执行输入摘要 |
| `output` | `AuditEventOutput` | 是 | 非空 | 执行输出摘要 |
| `context` | `AuditEventContext` | 是 | 非空 | 执行上下文 |
| `started_at` | `Timestamp` | 是 | RFC 3339 格式 | 执行开始时间 |
| `completed_at` | `Timestamp` | 否 | RFC 3339 格式 | 执行完成时间（未完成时为空） |
| `duration_ms` | `int64` | 否 | >= 0 | 执行耗时（毫秒） |
| `labels` | `map<string, string>` | 否 | key 非空 | 自定义标签 |

#### 1.2 AuditEventType 事件类型

| 枚举值 | Protobuf 数值 | 说明 | 触发时机 |
|--------|-------------|------|---------|
| `AUDIT_EVENT_TYPE_UNSPECIFIED` | 0 | 占位值 | — |
| `AUDIT_EVENT_TYPE_TASK_SCHEDULED` | 1 | Task 调度 | Scheduler 将 Task 分配给 Executor |
| `AUDIT_EVENT_TYPE_TASK_STARTED` | 2 | Task 开始执行 | Executor 开始执行 Task |
| `AUDIT_EVENT_TYPE_TASK_COMPLETED` | 3 | Task 执行完成 | Executor 返回 ExecuteResponse |
| `AUDIT_EVENT_TYPE_TASK_FAILED` | 4 | Task 执行失败 | Executor 返回 FAILED/TIMEOUT |
| `AUDIT_EVENT_TYPE_TASK_CANCELLED` | 5 | Task 被取消 | 用户取消执行 |
| `AUDIT_EVENT_TYPE_TASK_RETRIED` | 6 | Task 重试 | Scheduler 触发重试 |
| `AUDIT_EVENT_TYPE_ARTIFACT_VALIDATED` | 7 | Artifact 校验 | Foundry Core 完成 Artifact 校验 |
| `AUDIT_EVENT_TYPE_ARTIFACT_STORED` | 8 | Artifact 存储 | Artifact 写入 Artifact Store |
| `AUDIT_EVENT_TYPE_ARTIFACT_ROLLED_BACK` | 9 | Artifact 回滚 | 回滚操作标记 Artifact |
| `AUDIT_EVENT_TYPE_ARTIFACT_RESTORED` | 10 | Artifact 恢复 | 恢复操作还原 Artifact |
| `AUDIT_EVENT_TYPE_INTERVENTION_CREATED` | 11 | 人工介入创建 | Step 进入 AwaitingIntervention |
| `AUDIT_EVENT_TYPE_INTERVENTION_RESOLVED` | 12 | 人工介入解决 | 人类工程师完成介入操作 |
| `AUDIT_EVENT_TYPE_ROLLBACK_STARTED` | 13 | 回滚开始 | 回滚流程启动 |
| `AUDIT_EVENT_TYPE_ROLLBACK_COMPLETED` | 14 | 回滚完成 | 回滚流程完成 |
| `AUDIT_EVENT_TYPE_ROLLBACK_FAILED` | 15 | 回滚失败 | 回滚过程出错 |

#### 1.3 AuditEventInput 执行输入摘要

```protobuf
message AuditEventInput {
  string task_description = 1;
  AgentType agent_type = 2;
  repeated ArtifactType expected_artifact_types = 3;
  repeated ArtifactRef upstream_artifact_refs = 4;
  map<string, string> parameters = 5;
  repeated string required_capabilities = 6;
}
```

| 字段 | 类型 | 说明 | 脱敏规则 |
|------|------|------|---------|
| `task_description` | `string` | Task 描述 | 不脱敏 |
| `agent_type` | `AgentType` | 要求的 Agent 类型 | 不脱敏 |
| `expected_artifact_types` | `repeated ArtifactType` | 期望的 Artifact 类型 | 不脱敏 |
| `upstream_artifact_refs` | `repeated ArtifactRef` | 上游 Artifact 引用 | 不脱敏 |
| `parameters` | `map<string, string>` | Agent 参数 | 敏感字段脱敏（见 1.7 节） |
| `required_capabilities` | `repeated string` | Task 要求的 Agent 能力标识 | 不脱敏 |

#### 1.4 AuditEventOutput 执行输出摘要

```protobuf
message AuditEventOutput {
  repeated ArtifactRef produced_artifact_refs = 1;
  string error_message = 2;
  ValidationResult validation_result = 3;
  int32 retry_count = 4;
}
```

| 字段 | 类型 | 说明 | 脱敏规则 |
|------|------|------|---------|
| `produced_artifact_refs` | `repeated ArtifactRef` | 产出的 Artifact 引用列表 | 不脱敏 |
| `error_message` | `string` | 错误信息（失败时） | 不脱敏 |
| `validation_result` | `ValidationResult` | Artifact 校验结果 | 不脱敏 |
| `retry_count` | `int32` | 重试次数 | 不脱敏 |

> **设计决策**：AuditEventOutput 使用 ArtifactRef 引用而非内联完整 Artifact，原因：1) 避免审计日志与 Artifact Store 的数据冗余；2) Artifact 内容可能很大（如构建产物），内联会导致审计日志膨胀；3) 审计查询时通过 ArtifactRef 从 Artifact Store 获取完整内容。这符合 Artifact Over Conversation 原则——Artifact 作为独立工程产物存储和引用。

#### 1.5 AuditEventContext 执行上下文

```protobuf
message AuditEventContext {
  string workspace_id = 1;
  string repo_url = 2;
  string repo_branch = 3;
  string repo_commit = 4;
  string foundry_version = 5;
  string executor_version = 6;
  map<string, string> environment = 7;
}
```

| 字段 | 类型 | 说明 | 脱敏规则 |
|------|------|------|---------|
| `workspace_id` | `string` | Workspace 标识 | 不脱敏 |
| `repo_url` | `string` | 代码仓库地址 | 可能含凭证，脱敏 |
| `repo_branch` | `string` | 代码分支 | 不脱敏 |
| `repo_commit` | `string` | 代码提交哈希 | 不脱敏 |
| `foundry_version` | `string` | Foundry 版本 | 不脱敏 |
| `executor_version` | `string` | Executor 版本 | 不脱敏 |
| `environment` | `map<string, string>` | 环境变量 | 敏感字段脱敏 |

#### 1.6 审计事件与 Pipeline 层级的关系

```
Pipeline (pipeline_id)
├── AuditEvent: TASK_SCHEDULED (pipeline 级别)
├── Stage A (stage_id)
│   ├── AuditEvent: TASK_SCHEDULED
│   ├── Step A1 (step_id)
│   │   ├── AuditEvent: TASK_STARTED
│   │   ├── AuditEvent: TASK_COMPLETED
│   │   ├── AuditEvent: ARTIFACT_VALIDATED
│   │   └── AuditEvent: ARTIFACT_STORED
│   └── Step A2
│       ├── AuditEvent: TASK_STARTED
│       ├── AuditEvent: TASK_FAILED
│       ├── AuditEvent: TASK_RETRIED
│       ├── AuditEvent: TASK_STARTED (重试)
│       └── AuditEvent: TASK_COMPLETED (重试成功)
├── Stage B
│   └── Step B1
│       ├── AuditEvent: TASK_STARTED
│       ├── AuditEvent: TASK_FAILED
│       ├── AuditEvent: INTERVENTION_CREATED
│       ├── AuditEvent: INTERVENTION_RESOLVED
│       └── AuditEvent: TASK_COMPLETED (人工修正后)
└── AuditEvent: ROLLBACK_COMPLETED (如果触发回滚)
```

#### 1.7 敏感数据脱敏规则

审计日志记录时对敏感数据执行脱敏，确保密钥/Token 等信息不被明文记录：

| 数据类别 | 脱敏规则 | 示例 |
|---------|---------|------|
| API Key / Token | 替换为 `***` | `api_key=sk-abc123` → `api_key=***` |
| 密码 | 替换为 `***` | `password=secret` → `password=***` |
| 环境变量中的敏感值 | 替换为 `***` | `DATABASE_URL=postgres://user:pass@...` → `DATABASE_URL=***` |
| parameters 中的敏感值 | 替换为 `***` | `token_env=MY_TOKEN` → `token_env=***` |
| repo_url 中的凭证 | 替换为 `***` | `https://user:pass@git.example.com/...` → `https://***@git.example.com/...` |

**脱敏判定规则**：

1. **显式标记**：Task.labels 中 `sensitive=true` 的字段值脱敏
2. **键名匹配**：以下键名的值自动脱敏（不区分大小写）：
   - `password`、`passwd`、`pwd`
   - `token`、`api_key`、`apikey`、`access_key`、`secret_key`、`auth_token`、`auth_secret`、`auth_key`
   - `authorization`、`credential`
   - `private_key`、`privatekey`
   - 包含 `secret`、`password`、`token`、`key` 的键名
3. **URL 凭证**：URL 中包含 `user:pass@` 模式的自动脱敏

> **设计决策**：脱敏在审计日志写入时执行，而非查询时执行。原因：1) 审计日志一旦写入不可修改，脱敏必须在写入前完成；2) 查询时脱敏会增加查询延迟，且可能遗漏；3) 符合 Deterministic Over Smart 原则——脱敏规则确定，写入时即确定。

---

### 2. 审计记录持久化方案

#### 2.1 存储选型

| 候选方案 | 优势 | 劣势 | 评分 |
|---------|------|------|------|
| **SQLite** | 零部署、单文件、事务支持、Go 标准库支持 | 单写者限制、不支持高并发写入 | ⭐⭐⭐ |
| **PostgreSQL** | 成熟稳定、JSON 查询支持、高并发、扩展性好 | 需要独立部署、运维成本 | ⭐⭐⭐⭐⭐ |
| **文件系统（JSON Lines）** | 最简单、可追加写入、易于归档 | 查询性能差、无事务、并发写入需加锁 | ⭐⭐ |

**决策：v1 选择 SQLite 作为默认存储，PostgreSQL 作为可选存储。**

理由：

1. **Deterministic Over Smart**：SQLite 零部署，开箱即用，符合"稳定可预测优先"原则
2. **渐进式架构**：v1 使用 SQLite 满足单机部署需求，v2 可平滑迁移到 PostgreSQL
3. **Go 生态**：Go 标准库 `database/sql` + `modernc.org/sqlite`（纯 Go 实现，无 CGO 依赖）
4. **查询能力**：SQLite 支持 JSON 查询（`json_extract`），满足审计查询需求
5. **Artifact Store 复用**：Artifact Store 与审计日志可共享同一 SQLite 数据库，简化部署

**存储抽象层**：

```go
package audit

type AuditStore interface {
    WriteEvent(ctx context.Context, event *AuditEvent) error
    QueryEvents(ctx context.Context, query *AuditQuery) (*AuditQueryResult, error)
    GetEvent(ctx context.Context, eventID string) (*AuditEvent, error)
    Close() error
}

type AuditStoreFactory interface {
    Create(config AuditStoreConfig) (AuditStore, error)
    Type() string
}
```

通过 `AuditStoreFactory` 抽象，支持运行时切换存储实现，符合 Agent Is Replaceable 原则。

#### 2.2 数据库 Schema

```sql
CREATE TABLE IF NOT EXISTS audit_events (
    event_id TEXT PRIMARY KEY,
    event_type INTEGER NOT NULL,
    pipeline_id TEXT NOT NULL,
    stage_id TEXT NOT NULL,
    step_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    executor_agent_id TEXT NOT NULL,
    executor_agent_type INTEGER NOT NULL,
    execution_status INTEGER NOT NULL,
    input_json TEXT NOT NULL,
    output_json TEXT NOT NULL,
    context_json TEXT NOT NULL,
    started_at DATETIME NOT NULL,
    completed_at DATETIME,
    duration_ms INTEGER,
    labels_json TEXT,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_audit_pipeline_id ON audit_events(pipeline_id);
CREATE INDEX IF NOT EXISTS idx_audit_stage_id ON audit_events(pipeline_id, stage_id);
CREATE INDEX IF NOT EXISTS idx_audit_step_id ON audit_events(pipeline_id, stage_id, step_id);
CREATE INDEX IF NOT EXISTS idx_audit_task_id ON audit_events(task_id);
CREATE INDEX IF NOT EXISTS idx_audit_executor_agent_id ON audit_events(executor_agent_id);
CREATE INDEX IF NOT EXISTS idx_audit_event_type ON audit_events(event_type);
CREATE INDEX IF NOT EXISTS idx_audit_execution_status ON audit_events(execution_status);
CREATE INDEX IF NOT EXISTS idx_audit_started_at ON audit_events(started_at);
CREATE INDEX IF NOT EXISTS idx_audit_created_at ON audit_events(created_at);
```

**索引设计说明**：

| 索引 | 支持的查询场景 | 说明 |
|------|-------------|------|
| `idx_audit_pipeline_id` | 按 Pipeline 查询 | 最常用的查询维度 |
| `idx_audit_stage_id` | 按 Pipeline + Stage 查询 | 复合索引，支持 Stage 级追溯 |
| `idx_audit_step_id` | 按 Pipeline + Stage + Step 查询 | 复合索引，支持 Step 级追溯 |
| `idx_audit_task_id` | 按 Task 查询 | 支持从 Task 追溯执行历史 |
| `idx_audit_executor_agent_id` | 按 Agent 查询 | 支持按 Agent 维度统计 |
| `idx_audit_event_type` | 按事件类型查询 | 支持筛选特定事件（如失败事件） |
| `idx_audit_execution_status` | 按执行状态查询 | 支持筛选失败/超时事件 |
| `idx_audit_started_at` | 按时间范围查询 | 支持时间范围筛选 |
| `idx_audit_created_at` | 按创建时间查询 | 支持按写入时间排序 |

#### 2.3 保留策略

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `retention_days` | 90 | 审计日志保留天数 |
| `max_storage_mb` | 1024 | 审计日志最大存储空间（MB） |
| `cleanup_interval_hours` | 24 | 清理任务执行间隔（小时） |
| `archive_enabled` | false | 是否启用归档（将过期日志导出为文件） |
| `archive_path` | `/var/lib/foundry/audit-archive/` | 归档文件存储路径 |

**清理规则**：

1. 超过 `retention_days` 的审计日志标记为待清理
2. 清理前检查 `max_storage_mb`，如果未超限则保留
3. 清理时按 `created_at` 升序删除，优先删除最旧的记录
4. 如果 `archive_enabled = true`，清理前将待删除记录导出为 JSON Lines 文件
5. 清理操作记录自身的审计事件（元审计）

#### 2.4 性能考量

| 场景 | 预估量级 | 性能要求 | 方案 |
|------|---------|---------|------|
| 单次写入 | 1 条 AuditEvent | < 5ms | SQLite 单次 INSERT，WAL 模式 |
| 批量写入（Pipeline 完成） | 10~50 条 AuditEvent | < 50ms | 事务批量 INSERT |
| 按 Pipeline 查询 | 返回 10~100 条 | < 100ms | 索引命中 + LIMIT |
| 按时间范围查询 | 返回 100~1000 条 | < 500ms | 索引命中 + 分页 |
| 全量扫描 | 返回 10000+ 条 | < 5s | 分页 + 流式返回 |

**SQLite WAL 模式**：

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;
```

WAL（Write-Ahead Logging）模式允许并发读写，解决 SQLite 单写者限制对查询的影响。

**写入保障**：

1. 审计日志写入尝试同步完成——AuditEvent 写入成功后才返回执行结果
2. 若写入失败，记录错误日志并触发告警，不阻塞执行结果返回
3. 审计日志写入与 Artifact Store 写入在同一事务中（共享 SQLite 时）

---

### 3. 审计查询接口

#### 3.1 查询 API

**AuditService 部署架构**：

AuditService 由 Foundry Core 进程暴露，通过 Foundry Gateway 同时提供 gRPC 和 REST 访问：

```
┌─────────────────────────────────────────────────────────┐
│                    Foundry Gateway                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │ gRPC     │  │ REST API │  │ CLI (foundry audit)  │  │
│  └────┬─────┘  └────┬─────┘  └──────────┬───────────┘  │
│       │              │                    │              │
│       └──────────────┼────────────────────┘              │
│                      ▼                                   │
│            ┌──────────────────┐                          │
│            │  Foundry Core    │                          │
│            │  ┌────────────┐  │                          │
│            │  │AuditService│  │  ← gRPC Service 实现     │
│            │  └─────┬──────┘  │                          │
│            │        ▼         │                          │
│            │  ┌────────────┐  │                          │
│            │  │ AuditStore │  │  ← SQLite / PostgreSQL   │
│            │  └────────────┘  │                          │
│            └──────────────────┘                          │
└─────────────────────────────────────────────────────────┘
```

- **内部服务**：AuditService 是 Foundry Core 的内部 gRPC 服务，不直接暴露给外部
- **外部访问**：通过 Foundry Gateway 统一暴露为 REST API 和 gRPC 接口
- **CLI 访问**：`foundry audit` CLI 命令通过 Gateway REST API 查询审计日志
- **与 Executor Service 的关系**：AuditService 是只读查询服务，与 Executor Service（写操作）独立

```protobuf
service AuditService {
  rpc QueryEvents(AuditQueryRequest) returns (AuditQueryResponse);
  rpc GetEvent(AuditGetEventRequest) returns (AuditGetEventResponse);
  rpc GetPipelineTimeline(AuditPipelineTimelineRequest) returns (AuditPipelineTimelineResponse);
  rpc ExportEvents(AuditExportRequest) returns (stream AuditExportChunk);
}

message AuditQueryRequest {
  AuditQueryFilter filter = 1;
  AuditQuerySort sort = 2;
  int32 page_size = 3;
  string page_token = 4;
}

message AuditQueryFilter {
  repeated string pipeline_ids = 1;
  repeated string stage_ids = 2;
  repeated string step_ids = 3;
  repeated string task_ids = 4;
  repeated string executor_agent_ids = 5;
  repeated AgentType executor_agent_types = 6;
  repeated ExecutionStatus execution_statuses = 7;
  repeated AuditEventType event_types = 8;
  google.protobuf.Timestamp started_after = 9;
  google.protobuf.Timestamp started_before = 10;
  map<string, string> labels = 11;
}

message AuditQuerySort {
  string field = 1;
  bool ascending = 2;
}

`field` 合法值：`started_at`、`completed_at`、`duration_ms`、`event_type`、`execution_status`、`created_at`。传入非法值返回 INVALID_ARGUMENT 错误。

message AuditQueryResponse {
  repeated AuditEvent events = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message AuditGetEventRequest {
  string event_id = 1;
}

message AuditGetEventResponse {
  AuditEvent event = 1;
}

message AuditPipelineTimelineRequest {
  string pipeline_id = 1;
}

message AuditPipelineTimelineResponse {
  repeated AuditTimelineEntry entries = 1;
}

message AuditTimelineEntry {
  string stage_id = 1;
  string step_id = 2;
  AuditEvent event = 3;
  repeated AuditTimelineEntry children = 4;
}

message AuditExportRequest {
  AuditQueryFilter filter = 1;
  string format = 2;
}

message AuditExportChunk {
  bytes data = 1;
  bool is_last = 2;
}
```

#### 3.2 查询维度

| 查询维度 | API 参数 | 说明 |
|---------|---------|------|
| Pipeline | `filter.pipeline_ids` | 查询指定 Pipeline 的所有审计事件 |
| Stage | `filter.stage_ids` | 查询指定 Stage 的审计事件（需配合 pipeline_ids） |
| Step | `filter.step_ids` | 查询指定 Step 的审计事件（需配合 pipeline_ids + stage_ids） |
| Task | `filter.task_ids` | 查询指定 Task 的审计事件 |
| Agent | `filter.executor_agent_ids` | 查询指定 Agent 的审计事件 |
| Agent 类型 | `filter.executor_agent_types` | 按 Agent 类型筛选 |
| 执行状态 | `filter.execution_statuses` | 筛选成功/失败/超时/取消事件 |
| 事件类型 | `filter.event_types` | 筛选特定事件类型 |
| 时间范围 | `filter.started_after/before` | 按执行时间范围筛选 |
| 标签 | `filter.labels` | 按自定义标签筛选 |

#### 3.3 Pipeline 时间线查询

Pipeline 时间线查询是审计的核心场景——给定一个 Pipeline ID，返回该 Pipeline 的完整执行时间线：

```
Pipeline: code-review-pipeline-001
│
├── Stage: static-analysis
│   ├── Step: lint (TraditionalCli) ✅ 12s
│   │   └── Artifact: STATIC_ANALYSIS_REPORT
│   └── Step: security-scan (RemoteApi) ✅ 45s
│       └── Artifact: SECURITY_SCAN_REPORT
│
├── Stage: code-review
│   ├── Step: ai-review (LocalAiCli) ✅ 120s
│   │   └── Artifact: CODE_REVIEW_REPORT
│   └── Step: human-approval (HumanGate) ✅ 3600s
│       └── Artifact: APPROVAL_RECORD
│
└── Stage: patch
    └── Step: generate-patch (LocalAiCli) ❌ FAILED → RETRY → ✅ 60s
        └── Artifact: PATCH_DIFF
```

时间线查询返回嵌套结构，按 Stage → Step → Event 层级组织，支持前端渲染时间线视图。

#### 3.4 审计导出

审计导出支持将审计日志导出为外部格式，用于合规审计或离线分析：

| 导出格式 | 说明 | 适用场景 |
|---------|------|---------|
| `jsonl` | JSON Lines 格式，每行一条 AuditEvent | 数据分析、导入其他系统 |
| `csv` | CSV 格式，扁平化字段 | Excel 分析、报表生成 |

导出通过 gRPC Server Streaming 返回，支持大数据量导出：

```
1. 客户端发起 ExportEvents 请求
2. 服务端按 filter 查询匹配的 AuditEvent
3. 分批序列化为指定格式
4. 通过 stream 逐块返回
5. 最后一块 is_last = true
```

#### 3.5 REST API 映射

除了 gRPC 接口，审计查询也通过 Foundry Gateway 暴露为 REST API：

| REST API | gRPC 方法 | 说明 |
|----------|----------|------|
| `GET /api/v1/audit/events` | QueryEvents | 查询审计事件 |
| `GET /api/v1/audit/events/{event_id}` | GetEvent | 获取单个审计事件 |
| `GET /api/v1/audit/pipelines/{pipeline_id}/timeline` | GetPipelineTimeline | 获取 Pipeline 时间线 |
| `POST /api/v1/audit/events/export` | ExportEvents | 导出审计事件 |

**REST API 查询参数映射**：

```
GET /api/v1/audit/events?
  pipeline_id=xxx&
  stage_id=yyy&
  executor_agent_type=LOCAL_AI_CLI&
  execution_status=FAILED&
  started_after=2026-05-01T00:00:00Z&
  started_before=2026-05-05T00:00:00Z&
  page_size=50&
  page_token=abc123
```

---

### 4. 审计与 Artifact Store 的关系

#### 4.1 Artifact Store 复用审计存储

v1 选择让 Artifact Store 与审计日志共享同一 SQLite 数据库，简化部署：

```sql
CREATE TABLE IF NOT EXISTS artifacts (
    artifact_id TEXT PRIMARY KEY,
    artifact_type INTEGER NOT NULL,
    task_id TEXT NOT NULL,
    pipeline_id TEXT NOT NULL,
    stage_id TEXT NOT NULL,
    step_id TEXT NOT NULL,
    producer_agent_id TEXT NOT NULL,
    producer_agent_type INTEGER NOT NULL,
    content_format INTEGER NOT NULL,
    content_data BLOB,
    content_file_path TEXT,
    size_bytes INTEGER NOT NULL,
    checksum TEXT NOT NULL,
    schema_ref TEXT,
    validation_json TEXT,
    status TEXT NOT NULL DEFAULT 'Pending',
    labels_json TEXT,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_artifacts_pipeline_id ON artifacts(pipeline_id);
CREATE INDEX IF NOT EXISTS idx_artifacts_task_id ON artifacts(task_id);
CREATE INDEX IF NOT EXISTS idx_artifacts_producer_agent_id ON artifacts(producer_agent_id);
CREATE INDEX IF NOT EXISTS idx_artifacts_artifact_type ON artifacts(artifact_type);
CREATE INDEX IF NOT EXISTS idx_artifacts_status ON artifacts(status);
CREATE INDEX IF NOT EXISTS idx_artifacts_created_at ON artifacts(created_at);
```

**Artifact 状态字段**：

| 状态值 | 说明 | 对应 Task 2 生命周期状态 |
|--------|------|----------------------|
| `Pending` | 等待产出 | Pending |
| `Producing` | 正在产出 | Producing |
| `Produced` | 已产出 | Produced |
| `Validating` | 校验中 | Validating |
| `Valid` | 校验通过 | Valid |
| `Invalid` | 校验失败 | Invalid |
| `Stored` | 已存储 | Stored |
| `Referenced` | 已被引用 | Referenced |
| `Failed` | 失败 | Failed |
| `RolledBack` | 已回滚 | RolledBack（Task 5 新增） |

> **跨文档同步说明**：Artifact Store 的 `status` 字段值与 Task 2 定义的 Artifact 生命周期状态对应，并包含 Task 5 新增的 `RolledBack` 状态。

#### 4.2 审计与 Artifact 的关联查询

审计事件通过 `task_id`、`pipeline_id`、`stage_id`、`step_id` 与 Artifact 关联：

```sql
SELECT
    ae.event_id,
    ae.event_type,
    ae.execution_status,
    ae.started_at,
    ae.completed_at,
    a.artifact_id,
    a.artifact_type,
    a.status AS artifact_status
FROM audit_events ae
LEFT JOIN artifacts a ON ae.task_id = a.task_id
WHERE ae.pipeline_id = ?
ORDER BY ae.started_at ASC;
```

此查询可获取 Pipeline 中每个 Task 的执行情况和产出 Artifact，是审计追溯的核心查询。

---

### 5. 审计写入流程

```
1. Executor 返回 ExecuteResponse
2. Foundry Core 构造 AuditEvent
   a. event_type = TASK_COMPLETED / TASK_FAILED / TASK_CANCELLED
   b. 填充 input（从 Task 提取，执行脱敏）
   c. 填充 output（从 ExecuteResponse 提取）
   d. 填充 context（从 Workspace 和配置提取，执行脱敏）
   e. 计算 duration_ms = completed_at - started_at
3. 写入 AuditStore（同步写入）
4. 如果写入失败：
   a. 记录错误日志（标准日志，非审计日志）
   b. 触发告警（通知运维人员）
   c. 不阻塞 Agent 执行结果返回
5. 审计事件写入成功后返回执行结果
```

**审计事件写入时序**：

| 事件 | 写入时机 | 写入者 |
|------|---------|--------|
| TASK_SCHEDULED | Scheduler 分配 Executor 后 | Scheduler |
| TASK_STARTED | Executor 开始执行时 | Executor |
| TASK_COMPLETED | Executor 返回 SUCCESS 后 | Foundry Core |
| TASK_FAILED | Executor 返回 FAILED/TIMEOUT 后 | Foundry Core |
| TASK_CANCELLED | 用户取消后 | Scheduler |
| TASK_RETRIED | Scheduler 触发重试后 | Scheduler |
| ARTIFACT_VALIDATED | Foundry Core 校验完成后 | Foundry Core |
| ARTIFACT_STORED | Artifact 写入 Artifact Store 后 | Foundry Core |
| ARTIFACT_ROLLED_BACK | 回滚操作标记 Artifact 后 | Foundry Core |
| ARTIFACT_RESTORED | 恢复操作还原 Artifact 后 | Foundry Core |
| INTERVENTION_CREATED | 创建 Intervention 后 | Foundry Core |
| INTERVENTION_RESOLVED | 人类工程师完成介入后 | Foundry Core |
| ROLLBACK_STARTED | 回滚流程启动后 | Foundry Core |
| ROLLBACK_COMPLETED | 回滚流程完成后 | Foundry Core |
| ROLLBACK_FAILED | 回滚过程出错后 | Foundry Core |

---

## 接口定义

### 1. Protobuf 定义

#### 1.1 audit.proto

```protobuf
syntax = "proto3";

package foundry.v1;

option go_package = "github.com/foundry/foundry/gen/foundry/v1";

import "google/protobuf/timestamp.proto";
import "foundry/v1/common.proto";  // ExecutionStatus, AgentType 等枚举定义已移至 common.proto
import "foundry/v1/artifact.proto";

// 跨文档同步说明：
// - ExecutionStatus 枚举定义在 common.proto 中，与 agent_executor_architecture.md 保持同步
// - AgentType 枚举定义在 common.proto 中，与 agent_executor_architecture.md 保持同步
// - ArtifactRef, ArtifactType, ValidationResult 定义在 artifact.proto 中，与 task_artifact_data_model.md 保持同步

enum AuditEventType {
  AUDIT_EVENT_TYPE_UNSPECIFIED = 0;
  AUDIT_EVENT_TYPE_TASK_SCHEDULED = 1;
  AUDIT_EVENT_TYPE_TASK_STARTED = 2;
  AUDIT_EVENT_TYPE_TASK_COMPLETED = 3;
  AUDIT_EVENT_TYPE_TASK_FAILED = 4;
  AUDIT_EVENT_TYPE_TASK_CANCELLED = 5;
  AUDIT_EVENT_TYPE_TASK_RETRIED = 6;
  AUDIT_EVENT_TYPE_ARTIFACT_VALIDATED = 7;
  AUDIT_EVENT_TYPE_ARTIFACT_STORED = 8;
  AUDIT_EVENT_TYPE_ARTIFACT_ROLLED_BACK = 9;
  AUDIT_EVENT_TYPE_ARTIFACT_RESTORED = 10;
  AUDIT_EVENT_TYPE_INTERVENTION_CREATED = 11;
  AUDIT_EVENT_TYPE_INTERVENTION_RESOLVED = 12;
  AUDIT_EVENT_TYPE_ROLLBACK_STARTED = 13;
  AUDIT_EVENT_TYPE_ROLLBACK_COMPLETED = 14;
  AUDIT_EVENT_TYPE_ROLLBACK_FAILED = 15;
}

message AuditEvent {
  string event_id = 1;
  AuditEventType event_type = 2;
  string pipeline_id = 3;
  string stage_id = 4;
  string step_id = 5;
  string task_id = 6;
  string executor_agent_id = 7;
  AgentType executor_agent_type = 8;
  ExecutionStatus execution_status = 9;
  AuditEventInput input = 10;
  AuditEventOutput output = 11;
  AuditEventContext context = 12;
  google.protobuf.Timestamp started_at = 13;
  google.protobuf.Timestamp completed_at = 14;
  int64 duration_ms = 15;
  map<string, string> labels = 16;
}

message AuditEventInput {
  string task_description = 1;
  AgentType agent_type = 2;
  repeated ArtifactType expected_artifact_types = 3;
  repeated ArtifactRef upstream_artifact_refs = 4;
  map<string, string> parameters = 5;
  repeated string required_capabilities = 6;
}

message AuditEventOutput {
  repeated ArtifactRef produced_artifact_refs = 1;
  string error_message = 2;
  ValidationResult validation_result = 3;
  int32 retry_count = 4;
}

message AuditEventContext {
  string workspace_id = 1;
  string repo_url = 2;
  string repo_branch = 3;
  string repo_commit = 4;
  string foundry_version = 5;
  string executor_version = 6;
  map<string, string> environment = 7;
}

service AuditService {
  rpc QueryEvents(AuditQueryRequest) returns (AuditQueryResponse);
  rpc GetEvent(AuditGetEventRequest) returns (AuditGetEventResponse);
  rpc GetPipelineTimeline(AuditPipelineTimelineRequest) returns (AuditPipelineTimelineResponse);
  rpc ExportEvents(AuditExportRequest) returns (stream AuditExportChunk);
}

message AuditQueryRequest {
  AuditQueryFilter filter = 1;
  AuditQuerySort sort = 2;
  int32 page_size = 3;
  string page_token = 4;
}

message AuditQueryFilter {
  repeated string pipeline_ids = 1;
  repeated string stage_ids = 2;
  repeated string step_ids = 3;
  repeated string task_ids = 4;
  repeated string executor_agent_ids = 5;
  repeated AgentType executor_agent_types = 6;
  repeated ExecutionStatus execution_statuses = 7;
  repeated AuditEventType event_types = 8;
  google.protobuf.Timestamp started_after = 9;
  google.protobuf.Timestamp started_before = 10;
  map<string, string> labels = 11;
}

message AuditQuerySort {
  string field = 1;
  bool ascending = 2;
}

message AuditQueryResponse {
  repeated AuditEvent events = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message AuditGetEventRequest {
  string event_id = 1;
}

message AuditGetEventResponse {
  AuditEvent event = 1;
}

message AuditPipelineTimelineRequest {
  string pipeline_id = 1;
}

message AuditPipelineTimelineResponse {
  repeated AuditTimelineEntry entries = 1;
}

message AuditTimelineEntry {
  string stage_id = 1;
  string step_id = 2;
  AuditEvent event = 3;
  repeated AuditTimelineEntry children = 4;
}

message AuditExportRequest {
  AuditQueryFilter filter = 1;
  string format = 2;
}

message AuditExportChunk {
  bytes data = 1;
  bool is_last = 2;
}
```

### 2. Go 接口定义

```go
package audit

type AuditStore interface {
    WriteEvent(ctx context.Context, event *AuditEvent) error
    QueryEvents(ctx context.Context, query *AuditQuery) (*AuditQueryResult, error)
    GetEvent(ctx context.Context, eventID string) (*AuditEvent, error)
    Close() error
}

type AuditStoreFactory interface {
    Create(config AuditStoreConfig) (AuditStore, error)
    Type() string
}

type AuditStoreConfig struct {
    Driver              string `json:"driver"`
    DSN                 string `json:"dsn"`
    RetentionDays       int    `json:"retention_days"`
    MaxStorageMB        int    `json:"max_storage_mb"`
    CleanupIntervalHrs  int    `json:"cleanup_interval_hours"`
    ArchiveEnabled      bool   `json:"archive_enabled"`
    ArchivePath         string `json:"archive_path"`
}

type AuditQuery struct {
    Filter  AuditQueryFilter
    Sort    AuditQuerySort
    PageSize int
    PageToken string
}

type AuditQueryResult struct {
    Events       []*AuditEvent
    NextPageToken string
    TotalCount   int
}

type AuditQueryFilter struct {
    PipelineIDs        []string
    StageIDs           []string
    StepIDs            []string
    TaskIDs            []string
    ExecutorAgentIDs   []string
    ExecutorAgentTypes []foundryv1.AgentType
    ExecutionStatuses  []foundryv1.ExecutionStatus
    EventTypes         []foundryv1.AuditEventType
    StartedAfter       *time.Time
    StartedBefore      *time.Time
    Labels             map[string]string
}

type AuditQuerySort struct {
    Field     string
    Ascending bool
}
```

### 3. 脱敏工具

```go
package audit

var sensitiveKeyPatterns = []*regexp.Regexp{
    regexp.MustCompile(`(?i)password|passwd|pwd`),
    regexp.MustCompile(`(?i)token|api_key|apikey|access_key|secret_key|auth_token|auth_secret|auth_key`),
    regexp.MustCompile(`(?i)authorization|credential`),
    regexp.MustCompile(`(?i)private_key|privatekey`),
    regexp.MustCompile(`(?i)secret`),
}

func SanitizeMap(m map[string]string) map[string]string {
    result := make(map[string]string, len(m))
    for k, v := range m {
        if isSensitiveKey(k) {
            result[k] = "***"
        } else {
            result[k] = v
        }
    }
    return result
}

func SanitizeURL(rawURL string) string {
    u, err := url.Parse(rawURL)
    if err != nil {
        return "***"
    }
    if u.User != nil {
        u.User = url.UserPassword("***", "***")
    }
    return u.String()
}

func isSensitiveKey(key string) bool {
    for _, pattern := range sensitiveKeyPatterns {
        if pattern.MatchString(key) {
            return true
        }
    }
    return false
}
```

---

## 操作规范

### 1. 审计日志写入流程

```
1. 触发事件（Task 调度/执行/完成/失败/回滚等）
2. 构造 AuditEvent
   a. 生成 event_id (UUID v4)
   b. 填充 event_type
   c. 从 Task/ExecuteResponse/Intervention 中提取字段
   d. 对敏感字段执行脱敏
3. 序列化为 JSON（input_json, output_json, context_json, labels_json）
4. 写入 AuditStore
   a. SQLite: INSERT INTO audit_events
   b. PostgreSQL: INSERT INTO audit_events
5. 写入成功 → 继续
6. 写入失败 → 记录错误日志 + 触发告警 + 不阻塞主流程
```

### 2. 审计日志清理流程

```
1. 定时任务触发（每 cleanup_interval_hours 小时）
2. 计算过期时间 = now - retention_days
3. 查询 created_at < 过期时间 的记录数
4. 如果 archive_enabled:
   a. 导出待清理记录为 JSON Lines 文件
   b. 写入 archive_path
5. 删除过期记录
6. 检查存储空间是否超过 max_storage_mb
   a. 超过 → 按创建时间升序删除最旧记录，直到低于限制
7. 记录清理操作的元审计事件
```

### 3. 审计查询流程

```
1. 接收查询请求（gRPC / REST）
2. 解析查询参数为 AuditQueryFilter
3. 构造 SQL 查询
   a. WHERE 子句根据 filter 条件动态拼接
   b. ORDER BY 根据 sort 参数
   c. LIMIT = page_size + 1（多取一条判断是否有下一页）
4. 执行查询
5. 构造响应
   a. 如果结果数 > page_size → 生成 next_page_token
   b. total_count 通过 COUNT 查询获取
6. 返回响应
```

---

## 约束与限制

| 编号 | 限制项 | 说明 |
|------|--------|------|
| L-6.1 | **v1 不支持审计日志加密存储** | 审计日志以明文存储在 SQLite/PostgreSQL 中，加密存储在后续版本中实现 |
| L-6.2 | **v1 不支持审计日志不可篡改证明** | 审计日志的完整性依赖数据库的访问控制，不提供密码学不可篡改证明（如区块链锚定） |
| L-6.3 | **v1 查询性能受 SQLite 限制** | 单机部署下，SQLite 的查询性能满足 v1 需求；高并发查询场景需迁移到 PostgreSQL |
| L-6.4 | **v1 不支持实时审计流** | 审计事件通过查询 API 获取，不支持 WebSocket/SSE 实时推送 |
| L-6.5 | **v1 导出格式仅支持 JSONL 和 CSV** | 不支持 PDF、HTML 等可视化格式 |
| L-6.6 | **v1 审计日志不包含 Agent 内部日志** | Agent 的 stdout/stderr 日志由 Executor 收集，审计日志仅记录结构化的执行摘要 |
| L-6.7 | **Artifact Store 与审计日志共享 SQLite** | v1 简化部署，共享数据库；生产环境建议使用 PostgreSQL 独立部署 |
| L-6.8 | **Artifact content_data 大小限制** | `content_data BLOB` 仅用于小内容（< 1MB），大内容必须使用 `content_file_path` 引用文件系统。超过 1MB 的内容写入 content_data 会导致 SQLite 查询性能严重下降 |

---

## 待决问题

| 编号 | 问题 | 需要解决的任务 | 说明 |
|------|------|-------------|------|
| OQ-6.1 | Artifact Store 是否需要独立于审计存储 | 编码阶段 | v1 共享 SQLite 简化部署；生产环境是否需要独立 Artifact Store 取决于性能需求 |
| OQ-6.2 | 审计日志的合规标准 | 编码阶段 | 不同行业对审计日志的保留期限和不可篡改性有不同要求，需在编码阶段确认目标合规标准 |
| OQ-6.3 | 实时审计流的协议选择 | v2 | v2 可考虑 gRPC Server Streaming 或 SSE 推送实时审计事件 |
| OQ-6.4 | Agent stdout/stderr 的审计存储 | Task 7 | Agent 内部日志的收集和存储方式需在 Harness 集成方案中确定 |

**跨文档同步说明**：

| 同步项 | 目标文档 | 说明 |
|--------|---------|------|
| `ExecutionStatus` 移至 `common.proto` | Task 3（agent_executor_architecture.md） | `ExecutionStatus` 是跨多个 proto 文件共享的基础枚举（executor.proto / audit.proto / intervention.proto 均使用），应从 `executor.proto` 移至 `common.proto`，与 `AgentType`、`ArtifactType` 同级。Task 9 一致性审查时执行同步 |
| `AuditEventType` 枚举注册 | Task 2（task_artifact_data_model.md） | 新增 `AuditEventType` 枚举需在 common.proto 中注册，Task 9 一致性审查时执行同步 |
| `InterventionStatus`、`RollbackGranularity`、`OnFailureStrategy` 枚举注册 | Task 2（task_artifact_data_model.md） | Task 5 新增的三个枚举需在 common.proto 或 intervention.proto 中注册，Task 9 一致性审查时执行同步 |

---

## 修订历史

| 版本 | 日期 | 修改内容 | 作者 |
|------|------|---------|------|
| v1.0 | 2026-05-05 | 初始版本：覆盖审计日志数据结构（AuditEvent + 15 种事件类型）、持久化方案（SQLite 默认 + PostgreSQL 可选）、查询接口（gRPC + REST + 时间线 + 导出）、敏感数据脱敏规则、Artifact Store Schema、保留策略 | Foundry Team |
| v1.1 | 2026-05-05 | 评审修复：B-6.1 修正 audit.proto ExecutionStatus import 问题（声明移至 common.proto）；B-6.2 新增 AuditService 部署架构说明；S-6.1 AuditEventInput 新增 required_capabilities 字段；S-6.2 新增 L-6.8 content_data 大小限制；S-6.3 收窄脱敏规则 auth 模式；S-6.4 新增 AuditQuerySort.field 合法值约束；S-6.5 修正审计写入保障表述；S-6.6 修正 OQ-5.1 引用；新增跨文档同步说明 | Foundry Team |
