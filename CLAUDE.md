# CLAUDE.md

本文件为 Claude Code（及其他 AI 编程助手）在本仓库中工作时提供项目上下文指引。

> **职责边界**：本文件提供"项目是什么"的上下文；行为约束（"必须/禁止做什么"）由 `.trae/rules/` 下的规则文件定义。

---

## 项目定义

**Foundry** 不是业务应用，也不是 AI 编程助手，而是一个**用来构建软件的系统**——以 Harness 为流程引擎的工程级多 Agent 软件研发编排系统。

Foundry 的目标：
- 显式建模软件研发流程
- 在流程中调度多个职责单一的 Agent
- 所有行为必须：**可审计、可回滚、可替换、可失败**

---

## 核心工程哲学

Foundry 的设计遵循四项工程哲学原则，详细约束见 `.trae/rules/project_rules.md`（始终生效）：

| 原则 | 核心含义 |
|------|----------|
| **Flow First** | Agent 不拥有流程控制权，一切行为附着在流程中 |
| **Deterministic Over Smart** | 稳定可预测优先于聪明 |
| **Artifact Over Conversation** | 核心资产是工程产物，不是对话 |
| **Agent Is Replaceable** | 任意 Agent 都可被替换或禁用 |

设计或实现功能时，使用 `#Rule engineering_checklist` 逐项验证。

---

## 协作模型

Foundry v1 只支持一种协作模式：**多 Agent「分工并行」模型**

- 每个 Agent 只承担单一职责
- Agent 之间禁止直接通信
- 协作只通过结构化 Artifact 完成，由 Harness Step 负责传递
- 并行、顺序、失败处理全部由流程系统负责

**系统中不存在：** 自治 Agent、自我规划 Agent、Agent 之间的讨论或博弈。

---

## Agent 抽象

在 Foundry 中，**Agent = 受控执行器（Executor）**。

四种 Agent 类型（执行模板，FR-1）：
1. **本地 AI CLI 工具** — Codex CLI、Claude Code 等
2. **远程大模型 API** — 远程大模型服务
3. **传统 CLI 工具** — 标准命令行工具
4. **人类工程师（Gate 模式）** — 通过 Gate 介入

**Capabilities 能力声明**（与 AgentType 正交的第二维度）：

AgentType 定义"如何执行"，Capabilities 定义"能做什么"。新 Agent 形态通过"选择最接近的执行模板 + 声明能力"接入。

v1 预定义能力：`ai_reasoning`、`tool_use`、`code_generation`、`code_review`、`security_scan`、`deterministic`、`approval`、`notification`

详见 [agent_executor_architecture.md](docs/design/agent_executor_architecture.md)。

Agent 必须具备：
- 明确输入（Task + Context + Workspace）
- 明确输出（Artifact）
- 明确职责边界

Agent 不具备（FR-7）：
- 项目视角
- 架构裁决权
- 需求扩展权
- 流程决策权

**Agent 的失败是正常状态。Foundry 的价值在于管理失败，而不是避免失败。**

---

## Foundry 与 Harness 的关系

| 层面 | Harness 负责 | Foundry 负责 |
|------|-------------|-------------|
| 流程 | Pipeline / Stage / Step / Gate 执行 | — |
| Agent | — | Agent 抽象、Task/Artifact 规范、统一接入、调度执行 |
| 审计 | — | 审计结构定义、记录写入、查询追溯 |
| 边界 | 不理解 Agent 语义 | 不控制流程执行 |

用户与 Foundry 交互；Foundry 通过松耦合方式集成 Harness。

---

## v1 能力边界

### v1 实现（设计文档阶段）
- Agent Executor 抽象（4 种类型）
- Task 与 Artifact 结构化规范
- Harness 集成方案设计
- 失败处理与人工介入机制
- 回滚机制
- 流程审计与追溯设计
- Agent 注册与发现
- Agent 约束规范
- 5 个可复用的流程模板（覆盖开发、CI/CD、基础设施、安全、应急响应场景）

### v1 明确不做
- 自动规划研发流程
- 自演化 / 自反思 Agent
- 端到端无人值守交付
- Agent 之间智能协商
- 自治 Agent、自我规划 Agent

---

## 术语参考

> 完整定义见 `.trae/specs/foundry-v1/spec.md` 术语表章节。

核心术语：Flow、Pipeline、Stage、Step、Gate、Agent、AgentType、Capabilities、Task、TaskSpec、Artifact、ArtifactType、ArtifactRef、Executor、ExecutionStatus、AgentRegistrationSpec、AgentRuntimeState、AgentRegistration、Registry、CapabilityRegistry、AgentStatus、Intervention、InterventionStatus、RollbackGranularity、OnFailureStrategy、AuditEvent、AuditEventType、Audit、Context、Workspace

---

## 项目结构

```
Foundry/
├── README.md                                    # 项目定义与概述
├── CLAUDE.md                                    # AI 助手项目上下文（本文件）
├── PROGRESS.md                                  # 进度跟踪
├── docs/
│   ├── architecture.md                          # 业务架构图（Mermaid）
│   ├── pipeline_scenarios.md                    # 常见场景 Pipeline 模板
│   ├── design/                                  # 设计文档
│   │   ├── tech_stack_and_architecture.md       # 技术栈选型与项目架构
│   │   ├── task_artifact_data_model.md          # 核心数据模型（Task & Artifact）
│   │   ├── agent_executor_architecture.md       # Agent Executor 架构设计
│   │   ├── failure_handling_and_human_intervention.md  # 失败处理与人工介入机制
│   │   ├── audit_scheme.md                      # 流程审计方案
│   │   ├── agent_registry_and_discovery.md      # Agent 注册与发现机制
│   │   ├── harness_integration.md               # Harness 集成方案
│   │   └── flow_templates.md                    # 可复用流程模板
│   └── assets/
│       ├── foundry-banner.svg                   # 产品 Banner
│       └── foundry-architecture.svg             # 架构图
├── .trae/
│   ├── rules/                                   # 项目规则（按生效方式拆分）
│   │   ├── project_rules.md                     # 核心约束（始终生效）
│   │   ├── core_naming.md                       # 核心概念命名（始终生效）
│   │   ├── naming_conventions.md                # 命名规范（代码文件生效）
│   │   ├── documentation_standards.md           # 文档规范（Markdown 文件生效）
│   │   ├── design_doc_standards.md              # 设计文档规范（智能生效）
│   │   ├── git_conventions.md                   # 版本控制规范（智能生效）
│   │   ├── protobuf_schema_standards.md         # Protobuf 与 JSON Schema 规范（Proto/Schema 文件生效）
│   │   ├── review_process.md                    # 评审流程（手动触发）
│   │   └── engineering_checklist.md             # 工程哲学检查清单（智能生效）
│   └── specs/
│       └── foundry-v1/
│           ├── spec.md                          # 产品需求文档
│           ├── tasks.md                         # 实现计划
│           └── checklist.md                     # 验证清单
├── cmd/                                         # CLI 入口
│   ├── foundry/                                 # 主 CLI（foundry）
│   └── foundry-agent/                           # Agent 执行器入口（foundry-agent）
├── internal/                                    # 内部包（不可外部导入）
│   ├── agent/                                   # Agent Executor 实现
│   ├── scheduler/                               # Agent 调度层
│   ├── gateway/                                 # API Gateway（CLI/HTTP/gRPC）
│   ├── model/                                   # Task/Artifact 数据模型
│   ├── registry/                                # Agent 注册与发现
│   ├── failure/                                 # 失败处理与回滚
│   ├── audit/                                   # 流程审计
│   ├── harness/                                 # Harness 集成适配
│   └── config/                                  # 配置管理
├── proto/                                       # Protobuf 接口定义
├── gen/                                         # 生成的 gRPC 代码（不手动编辑）
├── configs/                                     # 配置文件
├── schemas/                                     # JSON Schema 文件
├── data/                                        # 运行时数据（Registry 快照、审计数据库等）
├── test/                                        # 集成测试
└── scripts/                                     # 构建/部署脚本
```

> 完整目录结构定义见 [tech_stack_and_architecture.md](docs/design/tech_stack_and_architecture.md)。

---

## 当前项目阶段

项目处于 **v1 设计文档阶段**，正在产出工程设计文档，为后续编码实现提供可审计、可评审、可执行的设计基线。

完整的任务分解、依赖关系和验收标准见 `.trae/specs/foundry-v1/tasks.md`。

---

## 开发工作流

### 设计文档阶段（当前）

1. 所有工作从 `.trae/specs/foundry-v1/tasks.md` 开始
2. **读取前置产出物**：根据 tasks.md 中当前 Task 的 `Required Reading` 字段，逐一读取前置 Task 的产出物文件，理解关键接口和设计决策
3. 开始任务前，更新 `PROGRESS.md` 标记为 `🔄 进行中`
4. 按照任务的描述和测试要求编写设计文档
5. 完成后，在 `PROGRESS.md` 中标记为 `✅ 已完成`，并在变更记录中补充关键决策摘要
6. 对照 `.trae/specs/foundry-v1/checklist.md` 逐项验证
7. 任何 checklist 🔴 项未通过时，不得标记任务为已完成

### 编码阶段（未来）

1. 编写代码前先阅读现有文档，验证所有假设
2. 所有代码必须追溯到对应的设计文档
3. 遵循 `.trae/rules/` 下的规则文件
4. 不得跳过测试或审计日志
5. 存在代码时必须运行 lint 和类型检查
6. 每个 Agent 集成必须实现 `proto/executor.proto` 中定义的 Executor gRPC 接口

---

## 语言策略

- 对话中跟随用户使用的语言
- 设计文档遵循 `.trae/rules/documentation_standards.md` 文档标准
- 代码注释跟随用户最新消息的语言，除非另有指示

---

## 待决问题（在指定任务中解决）

| 问题 | 解决任务 | 状态 |
|------|----------|------|
| 技术栈选型（编程语言、框架） | Task 1 | ✅ Go 1.22+，详见 [tech_stack_and_architecture.md](docs/design/tech_stack_and_architecture.md) |
| Harness 版本与集成方式 | Task 7 | ✅ Harness SaaS + Plugin Step，详见 [harness_integration.md](docs/design/harness_integration.md) |
| 第一版流程模板的具体场景 | Task 8 | ✅ 5 个模板，详见 [flow_templates.md](docs/design/flow_templates.md) |

---

## 关键文件速查

| 文件 | 用途 | 何时阅读 |
|------|------|----------|
| `README.md` | 项目定义、工程哲学、v1 范围 | 首次接触项目 |
| `CLAUDE.md` | AI 助手项目上下文（本文件） | 开始任何任务前 |
| `PROGRESS.md` | 当前任务状态与进度 | 工作前后 |
| `docs/architecture.md` | 业务架构图 | 理解系统结构 |
| `docs/pipeline_scenarios.md` | 常见场景 Pipeline 模板 | 理解 Foundry 编排能力 |
| `docs/design/tech_stack_and_architecture.md` | 技术栈选型（Go 1.22+）与项目架构 | 了解技术决策与目录规范 |
| `docs/design/task_artifact_data_model.md` | 核心数据模型（Task/Artifact/Context/Workspace） | 理解数据结构与接口契约 |
| `docs/design/agent_executor_architecture.md` | Agent Executor 架构设计 | 理解 Agent 接入规范与调度机制 |
| `docs/design/failure_handling_and_human_intervention.md` | 失败处理与人工介入机制 | 理解失败检测、状态机、人工介入、回滚机制 |
| `docs/design/audit_scheme.md` | 流程审计方案 | 理解审计日志结构、持久化、查询接口 |
| `docs/design/agent_registry_and_discovery.md` | Agent 注册与发现机制 | 理解 Agent 注册规范、发现算法、配置管理、动态启用/禁用 |
| `docs/design/harness_integration.md` | Harness 集成方案 | 理解 Harness 版本选型、Step 插件集成、概念映射、流程控制权归属 |
| `docs/design/flow_templates.md` | 可复用流程模板 | 理解模板格式规范、5 个预置模板、Artifact 流转路径、失败处理策略 |
| `.trae/specs/foundry-v1/spec.md` | 完整 PRD（FR、NFR、AC） | 理解要构建什么 |
| `.trae/specs/foundry-v1/tasks.md` | 任务分解与依赖关系 | 决定做什么 |
| `.trae/specs/foundry-v1/checklist.md` | 验证清单 | 标记工作完成前 |
| `.trae/rules/project_rules.md` | 核心约束（始终生效） | 任何对话 |
| `.trae/rules/naming_conventions.md` | 命名规范 | 编写代码时 |
| `.trae/rules/documentation_standards.md` | 文档规范 | 编写文档时 |
| `.trae/rules/design_doc_standards.md` | 设计文档规范 | 编写设计文档时 |
| `.trae/rules/git_conventions.md` | 版本控制规范 | Git 操作时 |
| `.trae/rules/protobuf_schema_standards.md` | Protobuf 与 JSON Schema 规范 | 编写/修改 proto 或 Schema 文件时 |
| `.trae/rules/review_process.md` | 评审流程 | 设计文档评审或代码评审时 |
| `.trae/rules/engineering_checklist.md` | 工程哲学检查清单 | 设计/实现功能时 |
