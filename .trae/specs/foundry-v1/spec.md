# Foundry v1 - 产品需求文档

| 属性 | 内容 |
|------|------|
| **文档标题** | Foundry v1 - 产品需求文档 |
| **文档作者** | Foundry Team |
| **文档日期** | 2026-05-04 |
| **文档版本** | v1.1 |
| **文档描述** | Foundry v1 产品需求定义，覆盖功能需求、非功能需求、验收标准和待决问题 |

---

## Overview

| 属性 | 内容 |
|------|------|
| **Summary** | Foundry v1 需要产出一套完整的工程设计文档，覆盖 Agent 抽象、Task/Artifact 规范、Harness 集成方案、流程模板定义等核心领域，为后续编码实现提供可审计、可评审、可执行的设计基线 |
| **Purpose** | 在编码之前，通过文档先行的方式，确保系统架构符合四大工程哲学，所有核心概念有明确定义，关键接口有清晰契约，避免实现阶段出现架构性返工 |
| **Target Users** | 软件架构师、DevOps 工程师、工程团队负责人、一线开发者（Agent 编写与使用者） |

## Goals

- 产出 Agent Executor 架构设计文档，覆盖四种 Agent 类型的统一抽象
- 产出 Task 与 Artifact 数据模型规范文档
- 产出 Harness 集成方案设计文档
- 产出 1-2 个可复用的软件研发流程模板文档
- 产出失败处理与人工介入机制设计文档
- 产出流程审计方案设计文档
- 产出 Agent 约束规范定义，明确 Agent 不具备项目视角、架构裁决权、需求扩展权、流程决策权
- 产出回滚机制设计文档，支持从任意失败点回退到已知安全状态
- 产出 Agent 注册与发现机制设计文档，支持 Agent 的注册、发现、配置管理

## Non-Goals (Out of Scope)

- 直接编写系统代码
- 自动规划研发流程的设计
- 自演化 / 自反思 Agent 的设计
- 端到端无人值守交付的设计
- Agent 之间智能协商的设计
- 自治 Agent、自我规划 Agent 的设计

## Background & Context

- Foundry v1 是一个全新项目，从空白开始构建
- 核心技术栈需在文档阶段确定（作为架构设计文档的一部分）
- Harness 作为流程引擎负责 Pipeline、Stage、Step、Gate、Audit
- Foundry 负责 Agent 抽象、Task/Artifact 规范、Agent 统一接入
- 协作模型：多 Agent「分工并行」，Agent 之间禁止直接通信，只通过结构化 Artifact 协作
- 本次产出物为设计文档，不涉及编码实现

## Terminology

| 术语 | 定义 |
|------|------|
| Flow | 流程，一切行为的附着载体，由 Pipeline / Stage / Step 构成 |
| Pipeline | 研发流程的顶层容器，由多个 Stage 组成 |
| Stage | 流程阶段，由多个 Step 组成，是阶段边界 |
| Step | 流程步骤，是 Agent 调度的最小单元 |
| Gate | 人类 / 规则介入点，用于审批、确认、修正 |
| Agent | 受控执行器（Executor），具备明确输入、输出和职责边界 |
| Task | Agent 的输入描述，包含任务描述、上下文、工作空间 |
| Artifact | Agent 的输出产物，结构化的工程产物，非无结构对话 |
| Executor | Agent 的执行器抽象，统一调度不同类型 Agent |
| Audit | 全流程追溯记录，包含每次 Agent 执行的完整信息 |
| Context | Agent 执行任务的上下文信息，包含环境变量、依赖关系、前置 Artifact 引用等 |
| Workspace | Agent 执行任务的工作空间，包含代码仓库路径、文件系统挂载、工作目录等 |

## Functional Requirements

- **FR-1**: Agent Executor 架构设计，支持四种 Agent 类型接入：本地 AI CLI 工具（Codex CLI、Claude Code 等）、远程大模型 API、传统 CLI 工具、人类工程师（Gate 模式）
- **FR-2**: Task 结构化规范定义，包含 Task 描述 + Context 上下文 + Workspace 工作空间
- **FR-3**: Artifact 结构化规范定义，作为 Agent 输出的核心资产，包含类型体系、内容格式、元数据规范
- **FR-4**: Harness 集成方案设计，支持 Pipeline/Stage/Step 调度 Agent
- **FR-5**: 流程审计方案设计，支持全流程追溯
- **FR-6**: 失败处理与人工介入机制设计，覆盖输出无效、格式错误、超时、人工中断等场景
- **FR-7**: Agent 约束规范定义，明确 Agent 不具备项目视角、架构裁决权、需求扩展权、流程决策权
- **FR-8**: 回滚机制设计，支持从任意失败点回退到已知安全状态
- **FR-9**: Agent 注册与发现机制设计，支持 Agent 的注册、发现、配置管理
- **FR-10**: 可复用流程模板设计，提供 1-2 个常见软件研发场景的 Pipeline 模板，包含多 Agent 分工并行示例和 Artifact 流转路径

## Non-Functional Requirements

- **NFR-1**: 所有设计文档必须体现可审计、可回滚、可替换、可失败四大属性
- **NFR-2**: 架构设计必须稳定、可预测，宁可保守也不允许不可控
- **NFR-3**: 设计中任意 Agent 都应可被替换或禁用，不依赖某个模型的「聪明程度」
- **NFR-4**: 失败是系统的一等公民，设计中必须支持定位、复现、人工修正
- **NFR-5**: 文档之间必须概念一致、术语统一、接口契约对齐

## Constraints

- **Technical**: 须遵循核心工程哲学（Flow First、Deterministic Over Smart、Artifact Over Conversation、Agent Is Replaceable）
- **Business**: v1 只设计明确列出的能力边界内的功能
- **Dependencies**: 依赖 Harness 作为流程引擎，需在文档阶段确定 Harness 版本和集成方式
- **Scope**: 本次产出物为设计文档，不涉及编码实现

## Assumptions

- Harness 作为流程引擎可用且符合需求
- 多 Agent 分工并行模型能够满足 v1 场景需求
- 结构化 Artifact 能够承载 Agent 协作所需信息
- 文档评审通过后，编码实现阶段可按文档直接推进

## Acceptance Criteria

> **编号约定**：AC 采用主编号 + 子编号方式，如 AC-6.1 表示 AC-6（失败处理）的子项（回滚机制），体现功能从属关系。

### AC-1: Agent Executor 架构设计文档
- **Given**: 需要接入四种类型的 Agent（本地 AI CLI 工具、远程大模型 API、传统 CLI 工具、人类工程师（Gate 模式））
- **When**: 完成 Agent Executor 架构设计文档
- **Then**: 文档包含统一的 Executor 接口定义、四种 Agent 类型的接入规范、每个 Agent 的输入/输出/职责边界定义，且 Agent 不具备项目视角/架构裁决权/需求扩展权/流程决策权
- **Verification**: `human-judgment`

### AC-2: Task 数据模型规范文档
- **Given**: 需要向 Agent 分配任务
- **When**: 完成 Task 数据模型规范文档
- **Then**: 文档包含 Task 完整字段定义（Task 描述、Context 上下文、Workspace 工作空间）、序列化格式规范、字段约束说明
- **Verification**: `human-judgment`

### AC-3: Artifact 数据模型规范文档
- **Given**: Agent 需要输出执行结果
- **When**: 完成 Artifact 数据模型规范文档
- **Then**: 文档包含 Artifact 类型体系定义、内容格式规范、元数据字段定义，且明确 Artifact 是结构化工程产物而非无结构对话
- **Verification**: `human-judgment`

### AC-4: Harness 集成方案设计文档
- **Given**: 需要在 Harness 流程中调度 Agent
- **When**: 完成 Harness 集成方案设计文档
- **Then**: 文档包含 Harness 版本选择说明、Step 插件集成方案、Pipeline/Stage/Step 到 Foundry 的映射关系、流程执行控制权归属说明（由 Harness 掌握）
- **Verification**: `human-judgment`

### AC-5: 流程审计方案设计文档
- **Given**: 需要设计流程审计方案，支持全流程追溯
- **When**: 完成流程审计方案设计文档
- **Then**: 文档包含审计日志数据结构定义、持久化方案、查询接口设计，支持按 Pipeline/Stage/Step 追溯每个 Agent 的执行情况
- **Verification**: `human-judgment`

### AC-6: 失败处理与人工介入机制设计文档
- **Given**: Agent 执行失败（输出无效、格式错误、超时、人工中断）
- **When**: 完成失败处理与人工介入机制设计文档
- **Then**: 文档包含失败检测规则定义、失败状态机设计、人工介入接口规范，确保失败可定位、可复现、可人工修正
- **Verification**: `human-judgment`

### AC-6.1: 回滚机制设计文档
- **Given**: 需要从任意失败点回退到已知安全状态
- **When**: 完成回滚机制设计文档
- **Then**: 文档包含回滚粒度定义、回滚触发条件、回滚后状态定义、回滚操作规范
- **Verification**: `human-judgment`

### AC-7: 工程哲学遵循验证
- **Given**: 所有设计文档已完成
- **When**: 对文档集合进行架构评审
- **Then**: 文档集合整体符合四大核心工程哲学——Flow First（一切行为附着在流程中）、Deterministic Over Smart（稳定可预测优先）、Artifact Over Conversation（核心资产是工程产物）、Agent Is Replaceable（Agent 可替换）
- **Verification**: `human-judgment`

### AC-8: 可复用流程模板文档
- **Given**: 需要为项目提供快速启动方案
- **When**: 完成流程模板文档
- **Then**: 至少有 1-2 个可复用的软件研发流程模板，包含多 Agent 分工并行示例、Artifact 流转路径说明、使用指南
- **Verification**: `human-judgment`

### AC-9: Agent 注册与发现机制设计文档
- **Given**: 系统中存在多种 Agent，需要统一管理
- **When**: 完成 Agent 注册与发现机制设计文档
- **Then**: 文档包含 Agent 注册规范、发现机制、配置管理方案，支持 Agent 的动态启用/禁用
- **Verification**: `human-judgment`

### AC-10: 文档间一致性
- **Given**: 所有设计文档已完成
- **When**: 对文档集合进行交叉审查
- **Then**: 所有文档术语统一、概念一致、接口契约对齐，无矛盾或遗漏
- **Verification**: `human-judgment`

## Open Questions

- [x] 具体技术栈选择（编程语言、框架等）——已解决：Go 1.22+、gRPC + Protobuf、容器化插件模型，详见 [tech_stack_and_architecture.md](../../docs/design/tech_stack_and_architecture.md)
- [x] Harness 具体版本和集成方式——已解决：Harness SaaS + Plugin Step，详见 [harness_integration.md](../../docs/design/harness_integration.md)
- [ ] 第一版流程模板的具体场景——需在 Task 8 流程模板文档中给出选择及理由

## 修订历史

| 版本 | 日期 | 修改内容 | 作者 |
|------|------|---------|------|
| v1.0 | 2026-05-04 | 初始版本 | Foundry Team |
| v1.1 | 2026-05-04 | 新增 FR-10（可复用流程模板）；统一 AC-1 Agent 类型命名；添加 AC 编号约定说明 | Foundry Team |
