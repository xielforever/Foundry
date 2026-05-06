---
alwaysApply: true
description: Foundry v1 核心概念命名规范，所有对话始终生效，确保代码和文档中的概念命名统一
---

## 核心概念命名

以下核心概念在代码和文档中必须使用统一命名，禁止自创别名：

### Agent 类型与执行器

| 概念 | 英文标识 | 代码命名示例 | 说明 |
|------|----------|-------------|------|
| 本地 AI CLI Agent | `LocalAiCliAgent` / `local_ai_cli_agent` | `LocalAiCliExecutor` | Codex CLI、Claude Code 等 |
| 远程 API Agent | `RemoteApiAgent` / `remote_api_agent` | `RemoteApiExecutor` | 远程大模型 API |
| 传统 CLI Agent | `TraditionalCliAgent` / `traditional_cli_agent` | `TraditionalCliExecutor` | 标准 CLI 工具 |
| 人类工程师 Agent | `HumanGateAgent` / `human_gate_agent` | `HumanGateExecutor` | Gate 模式介入 |

### 注册与发现

| 概念 | 英文标识 | 代码命名示例 | 说明 |
|------|----------|-------------|------|
| Agent 注册信息（静态） | `AgentRegistrationSpec` / `agent_registration_spec` | `AgentRegistrationSpec` | Executor 提供的静态注册信息 |
| Agent 运行时状态 | `AgentRuntimeState` / `agent_runtime_state` | `AgentRuntimeState` | Registry 管理的运行时状态 |
| Agent 完整注册信息 | `AgentRegistration` / `agent_registration` | `AgentRegistration` | Spec + Runtime 组合 |
| Agent 注册中心 | `Registry` / `registry` | `Registry` | Agent 注册、发现、状态管理 |
| 能力注册表 | `CapabilityRegistry` / `capability_registry` | `CapabilityRegistry` | v1 预定义和自定义能力标识管理 |
| Agent 状态 | `AgentStatus` / `agent_status` | `AgentStatus` | Agent 运行时状态枚举 |

### 失败处理与人工介入

| 概念 | 英文标识 | 代码命名示例 | 说明 |
|------|----------|-------------|------|
| 人工介入记录 | `Intervention` / `intervention` | `Intervention` | Step 失败后等待人类工程师操作时创建 |
| 介入状态 | `InterventionStatus` / `intervention_status` | `InterventionStatus` | 介入记录状态枚举 |
| 回滚粒度 | `RollbackGranularity` / `rollback_granularity` | `RollbackGranularity` | Step/Stage/Pipeline 三种粒度 |
| 失败策略 | `OnFailureStrategy` / `on_failure_strategy` | `OnFailureStrategy` | Pipeline 级别 on_failure 配置 |

### 审计

| 概念 | 英文标识 | 代码命名示例 | 说明 |
|------|----------|-------------|------|
| 审计事件 | `AuditEvent` / `audit_event` | `AuditEvent` | 记录一次 Agent 执行的完整信息 |
| 审计事件类型 | `AuditEventType` / `audit_event_type` | `AuditEventType` | 15 种审计事件类型枚举 |

### Capabilities 能力标识命名

能力标识使用 `snake_case` 格式，v1 预定义以下 8 种（新增能力需在 Registry 中注册）：

| 能力标识 | 含义 | 典型 Agent |
|---------|------|-----------|
| `ai_reasoning` | AI 推理能力 | LocalAiCli, RemoteApi |
| `tool_use` | 外部工具调用 | LocalAiCli（MCP 客户端） |
| `code_generation` | 代码生成 | LocalAiCli, RemoteApi |
| `code_review` | 代码评审 | LocalAiCli, RemoteApi |
| `security_scan` | 安全扫描 | RemoteApi |
| `deterministic` | 输出确定 | TraditionalCli |
| `approval` | 审批决策 | HumanGate |
| `notification` | 通知发送（副作用型） | TraditionalCli |
