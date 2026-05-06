# Foundry v1 - 实现计划（任务分解与优先级排序）

| 属性 | 内容 |
|------|------|
| **文档标题** | Foundry v1 - 实现计划 |
| **文档作者** | Foundry Team |
| **文档日期** | 2026-05-04 |
| **文档版本** | v1.2 |
| **文档描述** | Foundry v1 设计文档阶段的任务分解、依赖关系、优先级和验收标准 |

---

## [ ] Task 1: 技术栈选型与项目架构文档
- **Priority**: P0
- **Depends On**: None
- **产出物路径**: `docs/design/tech_stack_and_architecture.md`
- **Description**: 
  - 调研并确定核心技术栈（编程语言、框架、工具链），给出选型理由
  - 设计项目目录结构规范
  - 编写技术栈选型文档与项目架构文档
- **Acceptance Criteria Addressed**: [AC-7, AC-10]
- **Test Requirements**:
  - `human-judgement` TR-1.1: 技术栈选择理由充分，符合 Deterministic Over Smart 原则，优先考虑稳定性和可预测性
  - `human-judgement` TR-1.2: 项目目录结构设计清晰，模块划分符合工程化规范
  - `human-judgement` TR-1.3: 文档包含技术栈对比分析和最终决策理由
  - `human-judgement` TR-1.4: 技术栈选型解决了 Open Questions 中"具体技术栈选择"问题
- **Notes**: 这是所有后续文档任务的基础，技术栈选型直接影响后续设计决策

## [ ] Task 2: 核心数据模型规范文档（Task & Artifact）
- **Priority**: P0
- **Depends On**: [Task 1]
- **产出物路径**: `docs/design/task_artifact_data_model.md`
- **Required Reading**:
  - `docs/design/tech_stack_and_architecture.md` — 技术栈选型（Go 1.22+、gRPC + Protobuf、JSON Schema）、proto 目录结构、gen 代码路径约定
- **Description**: 
  - 定义 Task 数据结构规范（包含 Task 描述、Context 上下文、Workspace 工作空间）
  - 定义 Artifact 数据结构规范（类型体系、内容格式、元数据）
  - 定义序列化/反序列化格式规范
  - 明确 Artifact 是结构化工程产物而非无结构对话
- **Acceptance Criteria Addressed**: [AC-2, AC-3]
- **Test Requirements**:
  - `human-judgement` TR-2.1: Task 字段定义完整，包含 Task 描述、Context、Workspace，每个字段有类型和约束说明
  - `human-judgement` TR-2.2: Artifact 类型体系定义完整，包含类型枚举、内容格式、元数据字段
  - `human-judgement` TR-2.3: 序列化格式规范明确，支持 JSON Schema 或等价的机器可读定义
  - `human-judgement` TR-2.4: 文档明确声明 Artifact 是结构化工程产物，与无结构对话严格区分
- **Notes**: 这是系统的核心数据结构，所有其他文档都依赖此规范

## [ ] Task 3: Agent Executor 架构设计文档
- **Priority**: P0
- **Depends On**: [Task 2]
- **产出物路径**: `docs/design/agent_executor_architecture.md`
- **Required Reading**:
  - `docs/design/task_artifact_data_model.md` — Task/Artifact/Context/Workspace 完整字段定义、AgentType/ArtifactType 枚举、Protobuf 消息定义（common.proto/task.proto/artifact.proto）、校验错误码、Artifact 生命周期状态机
- **Description**: 
  - 设计 Agent Executor 统一接口
  - 设计四种 Agent 类型的接入规范：
    - 本地 AI CLI 工具（Codex CLI、Claude Code 等）
    - 远程大模型 API
    - 传统 CLI 工具
    - 人类工程师（Gate 模式）
  - 定义 Agent 约束规范（不具备项目视角、架构裁决权、需求扩展权、流程决策权）
  - 定义每个 Agent 的输入（Task + Context + Workspace）和输出（Artifact）
- **Acceptance Criteria Addressed**: [AC-1, AC-7] (覆盖 FR-1, FR-7)
- **Test Requirements**:
  - `human-judgement` TR-3.1: 统一接口定义完整，包含 execute 方法签名、输入输出类型、错误处理契约
  - `human-judgement` TR-3.2: 四种 Agent 类型各有独立的接入规范文档，包含配置示例
  - `human-judgement` TR-3.3: Agent 约束规范明确列出四项不具备的能力，并有设计机制保障约束不被违反
  - `human-judgement` TR-3.4: 架构设计符合 Agent Is Replaceable 原则，任意 Agent 可被替换或禁用
- **Notes**: 每个 Agent 必须职责单一，不拥有流程控制权

## [ ] Task 4: Agent 注册与发现机制设计文档
- **Priority**: P0
- **Depends On**: [Task 2, Task 3]（注册信息引用 Task/Artifact 结构，依赖 Executor 接口定义）
- **产出物路径**: `docs/design/agent_registry_and_discovery.md`
- **Required Reading**:
  - `docs/design/task_artifact_data_model.md` — Task/Artifact 数据模型、AgentType 枚举、ArtifactType 枚举、parameters 约定键
  - `docs/design/agent_executor_architecture.md` — Executor 统一接口、四种 Agent 类型接入规范
- **Description**: 
  - 设计 Agent 注册规范（注册信息、注册流程）
  - 设计 Agent 发现机制（如何查找可用 Agent）
  - 设计 Agent 配置管理方案（配置格式、动态启用/禁用）
- **Acceptance Criteria Addressed**: [AC-9]
- **Test Requirements**:
  - `human-judgement` TR-4.1: Agent 注册规范定义完整，包含注册信息字段和注册流程
  - `human-judgement` TR-4.2: 发现机制设计清晰，支持按类型/能力查找 Agent
  - `human-judgement` TR-4.3: 配置管理方案支持 Agent 的动态启用/禁用，无需重启系统
- **Notes**: 注册/发现是 Agent 可替换性的基础保障。**可与 Task 5、Task 6 并行执行**

## [ ] Task 5: 失败处理与人工介入机制设计文档
- **Priority**: P0
- **Depends On**: [Task 2, Task 3]
- **产出物路径**: `docs/design/failure_handling_and_human_intervention.md`
- **Required Reading**:
  - `docs/design/task_artifact_data_model.md` — Artifact 生命周期状态机（Pending→Producing→...→Failed）、ValidationResult 校验结果、校验错误码（9 个）、ArtifactRef 引用完整性约束、Task.timeout_seconds/retry_limit 字段
  - `docs/design/agent_executor_architecture.md` — Executor 接口错误处理契约、Agent 执行失败模式
- **Description**: 
  - 设计失败检测规则（输出无效、格式错误、超时、人工中断）
  - 设计失败状态机（正常→失败→人工介入→恢复/回滚）
  - 设计人工介入接口规范
  - 设计回滚机制（从任意失败点回退到已知安全状态）
- **Acceptance Criteria Addressed**: [AC-6, AC-6.1, AC-7]
- **Test Requirements**:
  - `human-judgement` TR-5.1: 失败检测规则覆盖四种场景（输出无效、格式错误、超时、人工中断），每种有明确的判定条件
  - `human-judgement` TR-5.2: 失败状态机设计完整，状态转换清晰，包含正常/失败/人工介入/恢复/回滚等状态
  - `human-judgement` TR-5.3: 人工介入接口规范定义完整，包含介入方式、操作类型、状态变更规则
  - `human-judgement` TR-5.4: 回滚机制设计完整，明确回滚粒度、回滚触发条件、回滚后状态定义
- **Notes**: 失败是一等公民，回滚机制从 NFR 提升为独立设计项。**可与 Task 4、Task 6 并行执行**

## [ ] Task 6: 流程审计方案设计文档
- **Priority**: P0
- **Depends On**: [Task 2, Task 3]
- **产出物路径**: `docs/design/audit_scheme.md`
- **Required Reading**:
  - `docs/design/task_artifact_data_model.md` — ArtifactMetadata 溯源字段（producer/pipeline/stage/step）、Task.created_by/created_at、Artifact 生命周期各状态、敏感数据处理约束
  - `docs/design/agent_executor_architecture.md` — Executor 接口、Agent 执行生命周期
- **Description**: 
  - 设计审计日志数据结构
  - 设计审计记录持久化方案
  - 设计审计查询接口
- **Acceptance Criteria Addressed**: [AC-5]
- **Test Requirements**:
  - `human-judgement` TR-6.1: 审计日志数据结构定义完整，包含 Agent 标识、输入、输出、时间戳、执行状态、所属 Pipeline/Stage/Step
  - `human-judgement` TR-6.2: 持久化方案设计明确，包含存储选型、保留策略、性能考量
  - `human-judgement` TR-6.3: 查询接口设计支持按 Pipeline/Stage/Step、时间范围、Agent 类型等维度查询
- **Notes**: 可审计性是核心要求。**可与 Task 4、Task 5 并行执行**

## [ ] Task 7: Harness 集成方案设计文档
- **Priority**: P0
- **Depends On**: [Task 3, Task 5, Task 6]
- **产出物路径**: `docs/design/harness_integration.md`
- **Required Reading**:
  - `docs/design/tech_stack_and_architecture.md` — gRPC 通信架构三层设计、容器化插件模型、Foundry CLI 与 Harness 通信方式待决问题
  - `docs/design/agent_executor_architecture.md` — Executor 接口、Agent 调度方式
  - `docs/design/failure_handling_and_human_intervention.md` — 失败状态机、人工介入接口
  - `docs/design/audit_scheme.md` — 审计日志结构、查询接口
- **Description**: 
  - 确定 Harness 版本选择，给出选型理由
  - 设计 Harness Step 插件集成方案
  - 设计 Pipeline/Stage/Step 到 Foundry 的映射关系
  - 明确流程执行控制权归属（由 Harness 掌握）
- **Acceptance Criteria Addressed**: [AC-4, AC-7]
- **Test Requirements**:
  - `human-judgement` TR-7.1: Harness 版本选择有明确理由，包含版本特性对比
  - `human-judgement` TR-7.2: Step 插件集成方案设计完整，包含插件接口、配置方式、错误处理
  - `human-judgement` TR-7.3: Pipeline/Stage/Step 映射关系定义清晰，包含映射规则和边界情况处理
  - `human-judgement` TR-7.4: 文档明确声明流程执行控制权由 Harness 掌握（Flow First 原则）
- **Notes**: Harness 不理解 Agent 语义，Foundry 不控制流程执行。此任务从 P1 提升为 P0

## [ ] Task 8: 可复用流程模板文档
- **Priority**: P1
- **Depends On**: [Task 7]
- **产出物路径**: `docs/design/flow_templates.md`
- **Required Reading**:
  - `docs/design/harness_integration.md` — Pipeline/Stage/Step 映射关系、Step 插件配置方式
  - `docs/design/task_artifact_data_model.md` — Task/Artifact 结构、ArtifactRef 引用传递流程
  - `docs/design/agent_executor_architecture.md` — 四种 Agent 类型、Artifact 流转路径
- **Description**: 
  - 设计 1-2 个常见软件研发场景流程
  - 定义流程模板规范（模板格式、参数化、复用方式）
  - 编写模板使用指南
  - 参考 [docs/pipeline_scenarios.md](../../docs/pipeline_scenarios.md) 中已有的 6 个场景模板作为输入
- **Acceptance Criteria Addressed**: [AC-8] (覆盖 FR-10)
- **Test Requirements**:
  - `human-judgement` TR-8.1: 至少有 1 个完整的流程模板，包含 Pipeline/Stage/Step 定义
  - `human-judgement` TR-8.2: 模板包含多个 Agent 分工并行的示例，体现 Artifact 流转路径
  - `human-judgement` TR-8.3: 模板使用指南清晰，包含参数说明、使用步骤、预期输出
  - `human-judgement` TR-8.4: 模板设计体现 Artifact Over Conversation 原则
- **Notes**: 模板是用户接触 Foundry 的入口，需精心设计。已有 [pipeline_scenarios.md](../../docs/pipeline_scenarios.md) 可作为场景参考

## [ ] Task 9: 文档间一致性审查与修订
- **Priority**: P1
- **Depends On**: [Task 8]
- **产出物路径**: `docs/design/consistency_review_report.md`
- **Required Reading**:
  - 所有 `docs/design/` 下的设计文档 — 全量交叉审查
- **Description**: 
  - 对所有设计文档进行交叉审查
  - 确保术语统一、概念一致、接口契约对齐
  - 修订发现的矛盾或遗漏
  - 编写文档间一致性审查报告
- **Acceptance Criteria Addressed**: [AC-10, AC-7]
- **Test Requirements**:
  - `human-judgement` TR-9.1: 所有文档使用统一术语，与术语表定义一致
  - `human-judgement` TR-9.2: 跨文档引用的接口契约对齐，无矛盾
  - `human-judgement` TR-9.3: 审查报告列出所有发现的问题及修订结果
  - `human-judgement` TR-9.4: 文档集合整体符合四大工程哲学
- **Notes**: 这是文档交付前的最终质量关卡

---

## 修订历史

| 版本 | 日期 | 修改内容 | 作者 |
|------|------|---------|------|
| v1.0 | 2026-05-04 | 初始版本 | Foundry Team |
| v1.1 | 2026-05-04 | Task 8 补充 FR-10 追溯 | Foundry Team |
| v1.2 | 2026-05-05 | 所有有依赖的 Task 增加 Required Reading 字段，显式列出前置产出物路径和关键接口摘要，解决跨会话上下文传递问题 | Foundry Team |
